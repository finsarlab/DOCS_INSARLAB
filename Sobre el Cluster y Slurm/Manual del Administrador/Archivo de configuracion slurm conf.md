---
title: slurm.conf — el archivo de configuración de Slurm
tags:
  - slurm
  - admin
  - configuracion
---

> [!attention]
> Nota para administradores. Los usuarios no necesitan modificar este archivo, pero leer la sección [[#Cómo se ve la configuración desde un usuario|cómo se ve desde un usuario]] les ayuda a entender de dónde salen los límites del clúster.

## ¿Qué es?

`slurm.conf` es **el archivo que define el clúster para Slurm**: qué nodos existen y cuántos recursos tiene cada uno, cómo se agrupan en particiones, qué nodo es el controlador, cómo se asignan CPUs y memoria y dónde se guardan los registros.

Todo lo que ves en [[Comando sinfo|sinfo]] (particiones, `TIMELIMIT`, nodos, GPU) sale de aquí.

| Dato | Valor en el InSAR lab |
|---|---|
| Ruta habitual | `/etc/slurm/slurm.conf` (confírmala con `scontrol show config \| grep SLURM_CONF`) |
| Quién lo lee | `slurmctld` (en `master`) y **cada** `slurmd` (en cada nodo, incluido `master`, porque está en la partición `all`) |
| Regla de oro | **Debe ser idéntico en todos los nodos.** Si difiere, Slurm registra advertencias y los nodos pueden quedar en `inval` o `drain` |
| Cómo se distribuye | ⚠️ Por confirmar: ¿Ansible, una copia en [[BeeGFS]] con enlace simbólico o modo *configless*? (ver [[BeeGFS - Pendientes por investigar]]) |

### Archivos hermanos (en la misma carpeta)
| Archivo | Para qué | Dónde debe existir |
|---|---|---|
| `slurm.conf` | Configuración general, nodos y particiones | Todos los nodos |
| `gres.conf` | Describe la GPU de `node2` (`Name=gpu Type=l4 File=/dev/nvidia0` o `AutoDetect=nvml`) | `node2` (o todos, con líneas `NodeName=`) |
| `cgroup.conf` | Aislamiento: impide que un trabajo use más CPU o RAM de la pedida | Todos los nodos |
| `slurmdbd.conf` | Conexión a la base de datos de contabilidad (`sacct`). Permisos `600`, dueño `slurm` | Solo `master` |

## Ver la configuración ACTIVA (sin abrir el archivo)

```bash
scontrol show config                     # todos los parámetros vigentes
scontrol show config | grep -i -E "select|returntoservice|defmem|proctrack|taskplugin|accounting"
scontrol show partition                  # particiones tal como las entiende slurmctld
scontrol show node node2                 # un nodo: CPUs, RealMemory, Gres, State, Reason
```

Para documentar la configuración real, sin comentarios ni líneas vacías:
```bash
grep -v -E '^\s*(#|$)' /etc/slurm/slurm.conf
```

## Anatomía del archivo

> [!warning] Ejemplo ilustrativo, NO es la copia real
> Lo siguiente se reconstruyó a partir de lo que muestran `sinfo` y la [[Arquitectura del Cluster del InSAR lab|tabla de arquitectura]]. **Sustitúyelo por el contenido real** del clúster (comando de arriba) en cuanto se verifique. #Pendiente

```ini
# ---------- Identidad del clúster ----------
ClusterName=skynet
SlurmctldHost=master

# ---------- Seguridad y procesos ----------
AuthType=auth/munge                    # todos los nodos comparten /etc/munge/munge.key
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity

# ---------- Cómo se reparten los recursos ----------
SelectType=select/cons_tres            # reparte núcleos, memoria y GPU por separado
SelectTypeParameters=CR_Core_Memory    # la memoria cuenta como recurso consumible
GresTypes=gpu

# ---------- Recuperación ----------
ReturnToService=1                      # un nodo que estuvo "no responde" vuelve solo al reconectarse

# ---------- Contabilidad (sacct) ----------
AccountingStorageType=accounting_storage/slurmdbd
JobAcctGatherType=jobacct_gather/cgroup

# ---------- Registros ----------
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

# ---------- NODOS ----------
NodeName=master CPUs=40 RealMemory=126000                     State=UNKNOWN
NodeName=node1  CPUs=56 RealMemory=257000                     State=UNKNOWN
NodeName=node2  CPUs=56 RealMemory=240000 Gres=gpu:l4:1       State=UNKNOWN
NodeName=node3  CPUs=24 RealMemory=31000                      State=UNKNOWN
NodeName=node4  CPUs=56 RealMemory=257000                     State=UNKNOWN

# ---------- PARTICIONES ----------
PartitionName=all   Nodes=master,node1 Default=YES MaxTime=INFINITE State=UP
PartitionName=extra Nodes=node[2-4]                MaxTime=INFINITE State=UP
PartitionName=n2gpu Nodes=node2                    MaxTime=INFINITE State=UP
```

### Los parámetros que más importan

| Parámetro | Qué controla | Por qué importa en el lab |
|---|---|---|
| `SlurmctldHost` | Nodo que corre el controlador | Si `master` se cae, nadie puede encolar |
| `SelectType` / `SelectTypeParameters` | Si se reparten núcleos sueltos y si la memoria es "consumible" | Sin `CR_*_Memory`, `--mem` **no reserva** memoria y dos trabajos pueden saturar un nodo |
| `ProctrackType`, `TaskPlugin` + `cgroup.conf` | Aislamiento real de CPU y RAM | Sin cgroups, un trabajo puede usar más núcleos o RAM de los que pidió y no habrá `OUT_OF_MEMORY` |
| `ReturnToService` | Si un nodo `down` vuelve solo | `0` = siempre a mano; `1` = vuelve solo si cayó por no responder; `2` = vuelve solo siempre (si su configuración es válida) |
| `NodeName … RealMemory=` | Memoria que Slurm cree que tiene el nodo | Si es **mayor** que la real, el nodo se pone en `drain` con razón `Low RealMemory` |
| `PartitionName … Default=YES` | Partición a la que van los trabajos sin `--partition` | Hoy es `all`, que incluye a `master` |
| `MaxTime` / `DefaultTime` | Tiempo máximo y tiempo por defecto | Hoy `INFINITE` |
| `DefMemPerCPU` | Memoria que recibe un trabajo que no declara `--mem` | Si no existe, un trabajo sin `--mem` puede llevarse **toda** la RAM del nodo |

## Obtener los valores correctos de cada nodo

En **cada** nodo, Slurm puede decirte la línea de hardware exacta:
```bash
ssh node3 slurmd -C
# NodeName=node3 CPUs=24 Boards=1 SocketsPerBoard=2 CoresPerSocket=6 ThreadsPerCore=2 RealMemory=31842
# UpTime=...
```
Copia esa línea (sin `UpTime`) al `slurm.conf`. Conviene dejar `RealMemory` un poco **por debajo** del valor reportado (entre 2 y 5% menos) para reservar memoria al sistema operativo y a BeeGFS.

## Cómo aplicar un cambio (procedimiento seguro)

```bash
# 1. Respaldo con fecha
sudo cp /etc/slurm/slurm.conf /etc/slurm/slurm.conf.$(date +%F_%H%M)

# 2. Editar
sudo nano /etc/slurm/slurm.conf

# 3. Distribuir el MISMO archivo a todos los nodos   ← según el método del lab (Ansible / BeeGFS)

# 4. Aplicar
sudo scontrol reconfigure                 # para la mayoría de los cambios
```

| Tipo de cambio | Cómo se aplica |
|---|---|
| Particiones (crear o cambiar nodos, `MaxTime`, `Default`) | `scontrol reconfigure` |
| Parámetros generales (`ReturnToService`, `DefMemPerCPU`, registros…) | `scontrol reconfigure` |
| Añadir o quitar nodos, cambiar CPUs, `RealMemory` o `Gres` de un nodo | Reiniciar `slurmctld` en `master` **y** `slurmd` en todos los nodos |
| Cambiar plugins (`SelectType`, `ProctrackType`, `AuthType`…) | Reiniciar todo, con la cola vacía de ser posible |

```bash
sudo systemctl restart slurmctld                                  # en master
for n in master node1 node2 node3 node4; do ssh $n sudo systemctl restart slurmd; done
sinfo -R                                                          # verificar que nadie quedó en drain/inval
```

> [!tip] Versionar la configuración
> Se recomienda guardar `slurm.conf`, `gres.conf` y `cgroup.conf` en un repositorio git del equipo (por ejemplo, dentro del repositorio de Ansible). Así cada cambio queda documentado con fecha, autor y motivo, en línea con el objetivo de reproducibilidad del equipo.

## Cómo se ve la configuración desde un usuario

Un usuario sin privilegios **puede leer** la configuración vigente:
```bash
scontrol show partition extra     # nodos, MaxTime, State
scontrol show node node4          # CPUTot, RealMemory, Gres
```
De aquí salen los límites de la [[Grupos de nodos y particiones|guía de particiones]].

## Propuestas de mejora (para discutir en el equipo)
- [ ] Hacer `extra` la partición por defecto, o quitar `master` de `all`, para que los trabajos sin `--partition` no caigan en el nodo controlador.
- [ ] Definir `DefMemPerCPU` (por ejemplo, 2–4 GB) para que un trabajo sin `--mem` no se lleve toda la RAM de un nodo.
- [ ] Evaluar un `DefaultTime` razonable (por ejemplo, 1 día) manteniendo `MaxTime=INFINITE`, para favorecer el *backfill*.
- [ ] Decidir si `node2` debe seguir en `extra` o si los trabajos de CPU deben excluirlo por política.

## Pendientes por verificar en la arquitectura real
- [ ] Copiar aquí el `slurm.conf` real (sin comentarios) y retirar el ejemplo ilustrativo.
- [ ] Confirmar la versión de Slurm: `sinfo --version`.
- [ ] Confirmar `SelectTypeParameters` y si existe `cgroup.conf` (¿se aplica de verdad `--mem`?).
- [ ] Confirmar el valor de `ReturnToService`.
- [ ] Confirmar cómo se distribuye `slurm.conf` a los nodos.
- [ ] Confirmar si existe `gres.conf` en `node2` y su contenido.
- [ ] Confirmar si `slurmdbd` está activo (`systemctl status slurmdbd`) y el nombre de la cuenta (`insarlab_users`, según [[Crear usuario (desde admin)]]).
- [ ] Confirmar si hay correo configurado (`MailProg`) para que funcione `--mail-type`.

Ver también: [[Recuperar nodos y servicios de Slurm]] · [[Manual del Administrador]]
