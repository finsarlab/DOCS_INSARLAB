---
title: Recuperar nodos y servicios de Slurm (down, down*, drain…)
tags:
  - slurm
  - admin
  - troubleshooting
---

> [!attention]
> Estos pasos requieren entrar como `admin` (o un usuario con `sudo`). Los usuarios normales solo pueden **ver** el estado con [[Comando sinfo|sinfo]]. El significado de cada estado está en el [[Catalogo de estados de los nodos en sinfo|catálogo de estados]].

## Mapa de servicios

| Servicio | Dónde corre | Qué hace | Si se cae… |
|---|---|---|---|
| `munge` | **Todos** los nodos | Autentica la comunicación entre nodos (todos comparten `/etc/munge/munge.key`) | Nodos `down*`, errores `Invalid authentication credential` |
| `slurmctld` | `master` | Controlador: cola, planificación, estado de los nodos | Nadie puede encolar (`Unable to contact slurm controller`) |
| `slurmd` | Cada nodo de cómputo **y `master`** (porque está en `all`) | Recibe y ejecuta los trabajos en ese nodo | Ese nodo pasa a `down*` |
| `slurmdbd` (+ `mariadb`) | `master` (a confirmar) | Historial y contabilidad (`sacct`, `sacctmgr`) | `sacct` no responde; la cola sigue funcionando |
| `chronyd` | Todos | Sincroniza el reloj | Si los relojes difieren, munge rechaza credenciales y los nodos caen |
| `beegfs-client` | Todos | Monta `/mnt/beegfs` (los `home`) | Los trabajos fallan al no encontrar archivos. Ver [[BeeGFS]] |

```bash
# Estado rápido de todo en un nodo
sudo systemctl status munge slurmd chronyd beegfs-client --no-pager | grep -E "●|Active:"
# En master, además:
sudo systemctl status slurmctld slurmdbd mariadb --no-pager | grep -E "●|Active:"
```

## Diagnóstico general (cualquier nodo que no acepta trabajos)

```bash
# 1. ¿Qué nodos tienen problema y por qué?
sinfo -R
# REASON               USER      TIMESTAMP           NODELIST
# Not responding       slurm     2026-10-09T08:12:40 node3
# Low RealMemory       slurm     2026-10-09T09:01:02 node2

# 2. Detalle del nodo
scontrol show node node3 | grep -E "State|Reason|RealMemory|CPUTot"

# 3. ¿Está vivo en la red?
ping -c 2 node3
ssh node3 uptime

# 4. Ya dentro del nodo: ¿están arriba los servicios?
ssh node3
sudo systemctl status slurmd munge
sudo journalctl -u slurmd -n 50 --no-pager
sudo tail -n 50 /var/log/slurm/slurmd.log      # ruta según SlurmdLogFile

# 5. ¿Funciona munge entre master y el nodo?
munge -n | ssh node3 unmunge                    # debe decir STATUS: Success (0)

# 6. ¿Coinciden los relojes?
date; ssh node3 date
```

Cuando se corrige la causa, se **regresa el nodo al servicio**:
```bash
sudo scontrol update nodename=node3 state=resume
sinfo -n node3          # debería verse idle (o mix/alloc)
```

## Recetas por estado

### `down*` — el nodo no responde
El asterisco significa que `slurmctld` **no recibe respuesta** de `slurmd` en ese nodo.

| Causa | Cómo detectarla | Solución |
|---|---|---|
| Nodo apagado o colgado | `ping` falla | Encenderlo o reiniciarlo físicamente; luego `state=resume` si no vuelve solo |
| `slurmd` caído | `systemctl status slurmd` → `inactive`/`failed` | `sudo systemctl restart slurmd` y revisar el `journalctl` |
| `munge` caído o con otra llave | `munge -n \| ssh nodeX unmunge` falla | `sudo systemctl restart munge`; comparar `md5sum /etc/munge/munge.key` con el de `master` |
| Reloj desfasado | `date` difiere más de unos segundos | `sudo systemctl restart chronyd && chronyc tracking` |
| Firewall | Hay ping y `slurmd` está activo, pero sigue `down*` | Abrir puertos 6817/tcp (slurmctld) y 6818/tcp (slurmd): `sudo firewall-cmd --list-all` |
| Red (cable o switch) | Sin ping, pero el nodo está encendido | Revisar el puerto del switch (ver [[Arquitectura del Cluster de InSAR lab (para admin)\|diagrama de red]]) |

Una vez que `slurmd` vuelve a responder, el `*` desaparece. Si queda en `down` (sin `*`), haz `state=resume`.

### `down` (sin asterisco) — fuera de servicio, pero responde
El nodo se comunica, pero alguien (un admin o Slurm) lo marcó como caído. Pasa por ejemplo después de un reinicio inesperado con `ReturnToService=0`.
```bash
scontrol show node node3 | grep Reason
sudo scontrol update nodename=node3 state=resume
```

### `drain` / `drng` / `drained` — drenado
`drng` = todavía termina trabajos, pero no acepta nuevos. `drain`/`drained` = ya vacío y sin aceptar nuevos.

| `Reason` típica | Qué pasó | Solución |
|---|---|---|
| Texto escrito por un admin (ej. `mantenimiento`) | Drenado a propósito | Terminar el mantenimiento y hacer `state=resume` |
| `Low RealMemory` | `slurm.conf` dice que el nodo tiene más RAM de la que detecta | Comparar con `slurmd -C` y bajar `RealMemory` en el [[Archivo de configuracion slurm conf\|slurm.conf]] |
| `Low socket*core*thread count` | Los CPUs declarados no coinciden con el hardware | Igual: alinear con `slurmd -C` |
| `Kill task failed` | Un proceso no murió al cancelar o terminar un trabajo (a menudo esperando E/S de BeeGFS) | Revisar procesos zombie: `ps aux \| grep <usuario>`; revisar [[BeeGFS]]; si hace falta, reiniciar el nodo |
| `batch job complete failure` / `Prolog error` | Falló el arranque de un trabajo en ese nodo (frecuentemente `/mnt/beegfs` no montado o usuario inexistente en el nodo) | `ssh nodeX 'df -h /mnt/beegfs; id <usuario>'` |
| `Not responding` | Estuvo `down*` y se drenó | Ver receta `down*` |

```bash
sudo scontrol update nodename=node2 state=resume
```

### `comp` / `completing` atorado
Un trabajo terminó, pero sus procesos no acaban de salir del nodo.
```bash
squeue -t CG                                   # ¿qué trabajo?
ssh node4 'ps -eo pid,user,stat,etime,cmd | grep -v root | head'
# si no se libera en unos minutos:
sudo scontrol update nodename=node4 state=down reason="CG atorado"
sudo scontrol update nodename=node4 state=resume
```
Si hay procesos en estado `D` (esperando disco), casi siempre el problema es BeeGFS.

### `inval` — registro inválido
El nodo se registró con **menos** recursos de los que declara `slurm.conf`, o con una configuración distinta. Alinear `slurm.conf` con `slurmd -C`, distribuirlo a todos los nodos, reiniciar `slurmd` en el nodo y `slurmctld`, y hacer `state=resume`.

### `unk` / `unknown`
Normal unos segundos después de reiniciar `slurmctld`. Si persiste, el nodo no se ha registrado: seguir la receta `down*`.

### `fail` / `failg`
Un admin marcó el nodo como fallado (`state=fail`). Se usa para retirarlo por hardware defectuoso. Se regresa con `state=resume` tras la reparación.

## Poner un nodo en mantenimiento (forma ordenada)

```bash
# Drenar: termina lo que tiene, no acepta nada nuevo
sudo scontrol update nodename=node3 state=drain reason="mantenimiento $(date +%F) - <tu nombre>"

# Esperar a que quede vacío
watch sinfo -n node3          # drng → drain

# … mantenimiento: actualizar, reiniciar, etc. …

# Regresarlo
sudo scontrol update nodename=node3 state=resume
```

Si el mantenimiento está programado para una fecha, una **reserva** evita que entren trabajos largos que no alcanzarían a terminar:
```bash
sudo scontrol create reservation reservationname=mant_node3 \
     starttime=2026-10-15T08:00:00 duration=04:00:00 \
     nodes=node3 users=root flags=maint,ignore_jobs
scontrol show reservation
sudo scontrol delete reservationname=mant_node3     # al terminar
```
Durante la reserva el nodo aparece como `maint`.

## Particiones (columna `AVAIL`)

```bash
sudo scontrol update partitionname=extra state=drain   # no acepta trabajos nuevos
sudo scontrol update partitionname=extra state=down    # acepta, pero no arranca
sudo scontrol update partitionname=extra state=up      # normal
```

## Arranque del clúster completo (después de un apagón)

El orden importa, porque cada capa depende de la anterior:

1. **Red y switch** encendidos.
2. **`master`** (y los nodos que alojan servicios de BeeGFS). Verificar que `chronyd` esté sincronizado.
3. **BeeGFS** del lado servidor (management → metadata → storage). Ver [[BeeGFS#Levantar BeeGFS después de una caída|BeeGFS]].
4. **Nodos de cómputo** → verificar que `/mnt/beegfs` esté montado en cada uno.
5. **`munge`** en todos los nodos.
6. **`mariadb` → `slurmdbd` → `slurmctld`** en `master`.
7. **`slurmd`** en todos los nodos (incluido `master`).
8. Verificar:
```bash
for n in master node1 node2 node3 node4; do
  echo "== $n"; ssh $n 'systemctl is-active munge slurmd beegfs-client; df -h /mnt/beegfs | tail -1'
done
sinfo -R
sinfo
```
9. Los nodos que sigan `down`: `scontrol update nodename=nodeX state=resume`.

> [!todo] Pendiente
> Confirmar en qué nodo corre cada servicio de BeeGFS y si los servicios arrancan solos (`systemctl is-enabled …`). Ver [[BeeGFS - Pendientes por investigar]].

## Registros (logs)

| Servicio | Ruta habitual | Confirmar con |
|---|---|---|
| `slurmctld` | `/var/log/slurm/slurmctld.log` | `scontrol show config \| grep SlurmctldLogFile` |
| `slurmd` | `/var/log/slurm/slurmd.log` | `scontrol show config \| grep SlurmdLogFile` |
| `slurmdbd` | `/var/log/slurm/slurmdbd.log` | `slurmdbd.conf` → `LogFile` |
| Cualquiera (systemd) | — | `journalctl -u <servicio> -n 100 --no-pager` |

Ver también: [[Archivo de configuracion slurm conf|slurm.conf]] · [[Manual del Administrador]]
