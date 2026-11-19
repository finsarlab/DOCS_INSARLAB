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
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): `master` usa Rocky Linux **10.1** y `node2` usa **10.2**, así que no están homologados. Falta la versión de `node1`, `node3` y `node4`, y la del kernel en todos.
- [ ] ¿Está fijado (`versionlock`) el kernel o BeeGFS para evitar que una actualización rompa el cliente? `dnf versionlock list`

## 2. ¿Qué servicio corre en qué nodo?
- [ ] Por nodo: `systemctl list-units 'beegfs*' --all --no-pager`
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): en `master` corren los cuatro servicios: `beegfs-mgmtd`, `beegfs-meta`, `beegfs-storage` y `beegfs-client`. Falta revisar los nodos de cómputo.
- [ ] ¿Qué servicios arrancan solos? `systemctl is-enabled beegfs-mgmtd beegfs-meta beegfs-storage beegfs-client`
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): en `master`, los cuatro arrancaron solos con el reinicio del 9 oct 2026. `beegfs-client` lo hizo **sin esperar** a `meta` y `storage`, y falló. Falta confirmar con `is-enabled` y revisar los nodos de cómputo.
- [ ] Nodo del **management**: `grep sysMgmtdHost /etc/beegfs/beegfs-client.conf`
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): `beegfs-mgmtd` corre en `master`. Falta confirmar que los clientes de los nodos de cómputo apuntan a él.
- [ ] Nodos de **metadata**: `beegfs-ctl --listnodes --nodetype=meta --nicdetails` (v8: `beegfs node list`)
- [ ] Nodos de **storage** y sus targets: `beegfs-ctl --listtargets --nodetype=storage --longnodes --state` (v8: `beegfs target list`)
- [ ] Clientes registrados: `beegfs-ctl --listnodes --nodetype=client`
- [ ] Dibujar el diagrama "servicio → nodo" y agregarlo a [[Arquitectura del Cluster de InSAR lab (para admin)]]

## 3. Discos
- [ ] Rutas de datos: `grep -E "storeStorageDirectory|storeMetaDirectory|storeMgmtdDirectory" /etc/beegfs/*.conf`
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): en `master`, storage está en `/raid/beegfs-storage`, metadata en `/mnt/beegfs-meta` y el cliente se monta en `/mnt/beegfs`. Falta la ruta de `mgmtd`.
- [x] Qué disco o RAID hay detrás de cada ruta: `lsblk -f`, `df -hT <ruta>`
  - ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]) `/raid/beegfs-storage` está en el RAID: `sda`, 32.7 TB, XFS, montado en `/raid` con `UUID=ba36546f-838f-4cef-b48d-491c389d4308`. `/mnt/beegfs-meta` está en el **NVMe del sistema**, no en el RAID.
- [ ] ¿RAID por hardware o por software (`cat /proc/mdstat`)? ¿Qué nivel?
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): es **por hardware**, con una controladora Broadcom / LSI MegaRAID 12GSAS/PCIe Secure SAS39xx. Falta el nivel de RAID (por ejemplo con `storcli /c0 /vall show`).
- [x] Capacidad total y uso: `beegfs-df` (v8: `beegfs health df`)
  - ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]) El 13 oct 2026, `df -h /mnt/beegfs` daba **33 TB en total y 32 TB usados (96 %)**. Hay que liberar espacio.
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
- [x] ¿Qué pasa con `/mnt/beegfs` si se reinicia `master`? (¿aloja el management o la metadata?)
  - ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]) `master` aloja el management, la metadata y el storage, así que **BeeGFS deja de estar disponible en todo el clúster** mientras `master` está apagado. Al volver, `beegfs-client` puede no montar si arranca antes que `meta` y `storage`. La solución es `sudo systemctl restart beegfs-client` en cada nodo afectado. Además, si falta el RAID en `/etc/fstab`, `master` no arranca.
- [ ] Orden real de encendido después de un apagón: validar y corregir la sección correspondiente en [[Recuperar nodos y servicios de Slurm]]
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): en `master` el orden debe ser RAID montado (`/raid`), luego `beegfs-mgmtd`, `beegfs-meta` y `beegfs-storage`, y al final `beegfs-client`. La dependencia de systemd que lo garantiza todavía no se aplica.

## 8. Respaldo y monitoreo
- [ ] ¿Existe respaldo de `/mnt/beegfs`? ¿Dónde, con qué frecuencia y quién lo revisa?
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): la metadata (`/mnt/beegfs-meta`) está en el NVMe, fuera del RAID, y hay que incluirla en los respaldos. No se sabe si existe algún respaldo.
- [ ] ¿Hay monitoreo (`beegfs-mon`, Grafana) o alertas de espacio?
- [ ] Rutas de los registros de cada servicio
  - *Parcial* ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]): para el cliente, `journalctl -u beegfs-client -l --no-pager -n 50` y `sudo dmesg | grep -i beegfs`. Faltan los de `mgmtd`, `meta` y `storage`.

## 9. Historial
- [ ] ¿Quién lo instaló, cuándo y con qué guía? ¿Hay notas del equipo anterior?
- [x] Incidentes conocidos (caídas, discos dañados) y cómo se resolvieron
  - ([[SSI. Como resolvimos el problema del cluster|incidente de oct 2026]]) Oct 2026: `master` no arrancaba por un `UUID` incorrecto del RAID en `/etc/fstab`, y después `beegfs-client` no montó tras el reinicio del 9 oct. Ambos casos están documentados ahí.

---
Cuando esta lista esté completa, se actualizará [[BeeGFS]] con la arquitectura real.
