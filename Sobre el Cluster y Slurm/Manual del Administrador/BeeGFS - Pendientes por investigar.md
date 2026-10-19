---
title: "BeeGFS: pendientes por investigar en la arquitectura"
tags:
  - beegfs
  - admin
  - pendiente
---

> [!todo] Objetivo
> Reunir lo necesario para documentar **exactamente** cómo está montado [[BeeGFS]] en el clúster del InSAR lab. Cada casilla incluye el comando que (probablemente) da la respuesta. Al resolverla, **pega la salida o la respuesta debajo de la tarea** y márcala.

## 1. Versión e instalación
- [ ] Versión de BeeGFS en cada nodo: `rpm -qa | grep -i beegfs` (¿7.x u 8.x? ¿la misma en todos?)
- [ ] Origen de los paquetes: `dnf repolist | grep -i beegfs`
- [ ] Versión de Rocky y del kernel en cada nodo: `cat /etc/rocky-release; uname -r`
- [ ] ¿Está fijado (`versionlock`) el kernel o BeeGFS para evitar que una actualización rompa el cliente? `dnf versionlock list`

## 2. ¿Qué servicio corre en qué nodo?
- [ ] Por nodo: `systemctl list-units 'beegfs*' --all --no-pager`
- [ ] ¿Qué servicios arrancan solos? `systemctl is-enabled beegfs-mgmtd beegfs-meta beegfs-storage beegfs-client`
- [ ] Nodo del **management**: `grep sysMgmtdHost /etc/beegfs/beegfs-client.conf`
- [ ] Nodos de **metadata**: `beegfs-ctl --listnodes --nodetype=meta --nicdetails` (v8: `beegfs node list`)
- [ ] Nodos de **storage** y sus targets: `beegfs-ctl --listtargets --nodetype=storage --longnodes --state` (v8: `beegfs target list`)
- [ ] Clientes registrados: `beegfs-ctl --listnodes --nodetype=client`
- [ ] Dibujar el diagrama "servicio → nodo" y agregarlo a [[Arquitectura del Cluster de InSAR lab (para admin)]]

## 3. Discos
- [ ] Rutas de datos: `grep -E "storeStorageDirectory|storeMetaDirectory|storeMgmtdDirectory" /etc/beegfs/*.conf`
- [ ] Qué disco o RAID hay detrás de cada ruta: `lsblk -f`, `df -hT <ruta>`
- [ ] ¿RAID por hardware o por software (`cat /proc/mdstat`)? ¿Qué nivel?
- [ ] Capacidad total y uso: `beegfs-df` (v8: `beegfs health df`)
- [ ] ¿Hay *buddy mirroring* (réplica) de metadata o storage? `beegfs-ctl --listmirrorgroups --nodetype=storage`
- [ ] Patrón de *striping* de las carpetas de usuarios: `beegfs-ctl --getentryinfo /mnt/beegfs/insarlab`

## 4. Red
- [ ] ¿Por qué interfaz o VLAN viaja el tráfico de BeeGFS? `cat /etc/beegfs/*interfaces*` y `connNetFilterFile` si existe
- [ ] Confirmar si `node2`, `node3` y `node4` (enlaces de 1 Gb) acceden a BeeGFS más lento que `node1` (10 Gb): `beegfs-net` en cada uno
- [ ] Puertos abiertos en el firewall de cada nodo: `sudo firewall-cmd --list-all`
- [ ] ¿Autenticación de conexión? `grep connAuthFile /etc/beegfs/*.conf` y dónde vive el archivo de la llave

## 5. Estructura de carpetas en `/mnt/beegfs`
- [ ] Árbol de primer y segundo nivel: `sudo tree -L 2 -d /mnt/beegfs` (o `ls -la`)
- [ ] ¿Qué hay además de `insarlab/<usuarios>`? (¿software compartido, datos del lab, configuraciones?)
- [ ] Permisos y dueños de cada carpeta de primer nivel: `ls -ld /mnt/beegfs/*`
- [ ] ¿Hay cuotas por usuario o grupo? `beegfs-ctl --getquota --gid 5000` (si están habilitadas)

## 6. Sincronización de configuraciones (lo que BeeGFS "reparte")
- [ ] Lista de archivos de configuración que viven en BeeGFS y se enlazan desde `/etc`: `find /etc -type l -lname '/mnt/beegfs/*' 2>/dev/null`
- [ ] ¿`slurm.conf`, `gres.conf` o `cgroup.conf` se leen desde BeeGFS? `ls -l /etc/slurm/`
- [ ] ¿`munge.key` está en BeeGFS? (por seguridad no debería ser legible por todos) `ls -l /etc/munge/`
- [ ] ¿El repositorio de Ansible (`/home/admin/ansible`) maneja BeeGFS? ¿Qué roles hay? `ls /home/admin/ansible/roles/`
- [ ] Procedimiento actual para cambiar una configuración "compartida" y que la tomen todos los nodos

## 7. Arranque y dependencias
- [ ] ¿`slurmd` espera a que `/mnt/beegfs` esté montado antes de arrancar? `systemctl cat slurmd | grep -i -E "after|requires"`. Si no, un nodo puede aceptar trabajos sin tener los `home`.
- [ ] ¿Qué pasa con `/mnt/beegfs` si se reinicia `master`? (¿aloja el management o la metadata?)
- [ ] Orden real de encendido después de un apagón: validar y corregir la sección correspondiente en [[Recuperar nodos y servicios de Slurm]]

## 8. Respaldo y monitoreo
- [ ] ¿Existe respaldo de `/mnt/beegfs`? ¿Dónde, con qué frecuencia y quién lo revisa?
- [ ] ¿Hay monitoreo (`beegfs-mon`, Grafana) o alertas de espacio?
- [ ] Rutas de los registros de cada servicio

## 9. Historial
- [ ] ¿Quién lo instaló, cuándo y con qué guía? ¿Hay notas del equipo anterior?
- [ ] Incidentes conocidos (caídas, discos dañados) y cómo se resolvieron

---
Cuando esta lista esté completa, se actualizará [[BeeGFS]] con la arquitectura real.
