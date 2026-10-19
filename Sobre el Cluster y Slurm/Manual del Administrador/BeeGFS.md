---
title: BeeGFS — almacenamiento compartido del clúster
tags:
  - beegfs
  - admin
  - almacenamiento
---

> [!warning] Nota en construcción
> Esta nota tiene lo **general** de BeeGFS y los comandos para revisarlo y levantarlo. La estructura **exacta** del InSAR lab (qué nodo aloja cada servicio, discos, versión, qué configuraciones se sincronizan) está pendiente de documentar: ver [[BeeGFS - Pendientes por investigar]]. #Pendiente

## ¿Qué es y para qué lo usamos?

**BeeGFS** es un sistema de archivos **paralelo y distribuido**: varios servidores aportan disco y todos los nodos lo ven como **una sola carpeta**. En el InSAR lab está montado en:

```
/mnt/beegfs
```

Lo usamos para dos cosas:
1. **Los `home` de los usuarios** viven en `/mnt/beegfs/insarlab/<usuario>` (ver [[Comandos útiles en el manejo del instalador]]). Por eso un script que escribes en `master` existe también en `node4` cuando Slurm lo ejecuta ahí, y por eso al crear un usuario en los nodos se usa `useradd -M` (no se crea carpeta local; ver [[Crear usuario (desde admin)]]).
2. **Sincronizar archivos entre nodos**, para no copiar configuraciones a mano en cada equipo. #Pendiente

> [!info] Para usuarios
> Lo único que necesitas saber: tu carpeta personal es la misma en todos los nodos. Lo que guardes ahí lo ve cualquier trabajo, en cualquier nodo. Ver [[Logica de trabajo en insarlab#Sobre el almacenamiento|lógica de almacenamiento]].

## Componentes

| Servicio | Función | Puerto por defecto (v7) | En el InSAR lab |
|---|---|---|---|
| `beegfs-mgmtd` | **Management**: registro central (quién es quién). Solo hay uno | 8008 | ❓ |
| `beegfs-meta` | **Metadata**: nombres, carpetas, permisos, dónde está cada fragmento | 8005 | ❓ |
| `beegfs-storage` | **Storage**: guarda el contenido de los archivos (en *targets*, que son discos o RAIDs) | 8003 | ❓ |
| `beegfs-client` | Módulo del kernel que **monta** `/mnt/beegfs` | 8004 | Todos los nodos |
| `beegfs-helperd` | Ayudante del cliente (solo **v7**; eliminado en v8) | 8006 | ❓ |

Orden de dependencia: **mgmtd → meta → storage → client**. Si cae el management o el metadata, `/mnt/beegfs` deja de responder en **todos** los nodos.

## Paso 0: ¿qué versión tenemos?

Los comandos cambian mucho entre la versión 7 y la 8:
```bash
rpm -qa | grep -i beegfs
```
- **7.x** → herramientas `beegfs-ctl`, `beegfs-df`, `beegfs-check-servers`, `beegfs-net`.
- **8.x** → una sola herramienta, `beegfs` (reemplaza a las anteriores). Consulta sus subcomandos con `beegfs --help`.

## Revisar el estado

### Desde cualquier nodo (cliente)
```bash
mount | grep beegfs                         # ¿está montado?
df -h /mnt/beegfs                           # ¿responde y cuánto espacio queda?
sudo systemctl status beegfs-client         # (v7: también beegfs-helperd)
timeout 10 ls /mnt/beegfs/insarlab > /dev/null && echo OK || echo "NO RESPONDE"
```
> [!tip] `ls` congelado
> Si `ls /mnt/beegfs` se queda colgado, **no es tu terminal**: el cliente no alcanza a un servidor (normalmente meta o storage). Usa siempre `timeout` al diagnosticar para no quedarte atorado.

### Estado de los servidores

| Pregunta | v7 (`beegfs-ctl`) | v8 (`beegfs`)[^1] |
|---|---|---|
| ¿Responden todos los servidores? | `beegfs-check-servers` | `beegfs health check` |
| Espacio libre por target | `beegfs-df` | `beegfs health df` |
| Lista de nodos de metadata | `beegfs-ctl --listnodes --nodetype=meta --nicdetails` | `beegfs node list` |
| Lista de nodos de storage | `beegfs-ctl --listnodes --nodetype=storage --nicdetails` | `beegfs node list` |
| Clientes conectados | `beegfs-ctl --listnodes --nodetype=client` | `beegfs node list` |
| Estado de los targets (online/offline, good/needs-resync) | `beegfs-ctl --listtargets --nodetype=storage --state --longnodes` | `beegfs target list` |
| Conexiones de este cliente | `beegfs-net` | `beegfs health net` |
| ¿En qué targets está un archivo o carpeta? | `beegfs-ctl --getentryinfo /mnt/beegfs/insarlab/<usuario>` | `beegfs entry info <ruta>` |

**Cómo leer el estado de un target:**
- *Reachability*: `Online` ✅ · `Probably-offline` ⚠️ (dejó de responder hace poco) · `Offline` ❌
- *Consistency*: `Good` ✅ · `Needs-resync` ⚠️ (solo si hay *mirroring*) · `Bad` ❌

### En los nodos servidores
```bash
sudo systemctl status beegfs-mgmtd beegfs-meta beegfs-storage --no-pager | grep -E "●|Active:"
sudo journalctl -u beegfs-meta -n 50 --no-pager
ls /var/log/beegfs-*.log 2>/dev/null          # (v7) registros en archivo
```

## Levantar BeeGFS después de una caída

> [!danger] Respeta el orden
> Levantar el cliente antes que los servidores solo genera montajes colgados.

```bash
# 1. En el nodo del management
sudo systemctl start beegfs-mgmtd

# 2. En el/los nodos de metadata
sudo systemctl start beegfs-meta

# 3. En el/los nodos de storage
sudo systemctl start beegfs-storage

# 4. Verificar servidores (v7 / v8)
beegfs-check-servers        # o: beegfs health check

# 5. En cada nodo cliente (v7: helperd primero)
sudo systemctl start beegfs-helperd     # solo v7
sudo systemctl restart beegfs-client
df -h /mnt/beegfs
```

Para revisar todos los clientes desde `master`:
```bash
for n in master node1 node2 node3 node4; do
  echo -n "$n: "; ssh $n 'timeout 10 df -h /mnt/beegfs | tail -1 || echo NO MONTADO'
done
```

> [!important] Después, revisa Slurm
> Si BeeGFS estuvo caído, es probable que haya nodos en `drain` con razón `Kill task failed` o `batch job complete failure`. Ver [[Recuperar nodos y servicios de Slurm]].

## Problemas frecuentes

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `/mnt/beegfs` no existe o está vacío en un nodo | El cliente no arrancó | `sudo systemctl restart beegfs-client`; revisar `journalctl -u beegfs-client` |
| El cliente no arranca **después de actualizar el kernel** (`dnf update`) | El módulo del cliente se compiló para el kernel anterior | Instalar `kernel-devel` de la versión actual (`uname -r`) y recompilar. En v7: `sudo /etc/init.d/beegfs-client rebuild` y luego `restart`. Después, **reiniciar `slurmd`** en ese nodo |
| `ls` se congela | Un servidor meta o storage no responde | `beegfs-check-servers` / `beegfs health check` desde otro nodo |
| Target `Offline` | El servicio de storage cayó o el disco tiene problemas | `systemctl status beegfs-storage` en ese servidor; revisar el disco (`dmesg`, `smartctl`) |
| Espacio lleno | Un target al 100% | `beegfs-df` / `beegfs health df`; buscar las carpetas más pesadas con `du -sh /mnt/beegfs/insarlab/* \| sort -h` |
| Errores de autenticación en los registros | La llave compartida (`connAuthFile`) difiere entre nodos | Comparar `md5sum` del archivo de autenticación en todos los nodos |
| Firewall tras una actualización | Puertos cerrados | `sudo firewall-cmd --list-all`; puertos 8003–8008 (v7) |

## Archivos de configuración (rutas habituales)

| Archivo | Para qué |
|---|---|
| `/etc/beegfs/beegfs-client.conf` | Cliente: dirección del management (`sysMgmtdHost`) |
| `/etc/beegfs/beegfs-mounts.conf` | Punto de montaje (`/mnt/beegfs`) |
| `/etc/beegfs/beegfs-meta.conf` | Metadata: `storeMetaDirectory` |
| `/etc/beegfs/beegfs-storage.conf` | Storage: `storeStorageDirectory` |
| `/etc/beegfs/beegfs-mgmtd.conf` (v7) o `.toml` (v8) | Management |
| `connInterfacesFile` / `connNetFilterFile` | Por qué red (VLAN 10/20, 1 Gb o 10 Gb) habla BeeGFS |

Ver también: [[BeeGFS - Pendientes por investigar]] · [[Manual del Administrador]]

[^1]: Los subcomandos de la v8 deben confirmarse con `beegfs --help` en el clúster. La [documentación oficial de la v8](https://doc.beegfs.io/8.0/release_notes.html) indica que `beegfs` reemplaza a `beegfs-ctl`, `beegfs-df`, `beegfs-check-servers` y `beegfs-net`, y que `beegfs-helperd` se eliminó.
