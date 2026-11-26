---
title: "Incidente en Skynet: RAID y BeeGFS sin montar"
description: Cómo se recuperó el nodo master cuando no arrancaba por un UUID incorrecto del RAID y cuando BeeGFS no se montó tras un reinicio.
tags:
  - cluster
  - skynet
  - raid
  - beegfs
  - incidentes
date: 2026-10-13
---

# Incidente en Skynet: RAID y BeeGFS sin montar

> [!summary] Resumen
> El nodo `master` de Skynet tuvo dos fallas relacionadas con el almacenamiento:
>
> 1. **El sistema no arrancaba** porque `/etc/fstab` apuntaba a un `UUID` del RAID que ya no existía. Se corrigió desde el modo recovery de Rocky Linux con una USB booteable.
> 2. **BeeGFS no se montó** después del reinicio del **9 de octubre de 2026**. El RAID y los servidores de BeeGFS estaban bien; solo falló `beegfs-client`. Se resolvió el **13 de octubre** con `sudo systemctl restart beegfs-client`, sin reiniciar el servidor.

| | Incidente 1: arranque | Incidente 2: BeeGFS |
| --- | --- | --- |
| **Síntoma** | El SO se quedaba esperando un disco al iniciar | `/mnt/beegfs` no aparecía montado |
| **Causa** | `UUID` incorrecto para `/raid` en `/etc/fstab` | `beegfs-client` arrancó antes que `beegfs-meta` y `beegfs-storage` |
| **Solución** | Corregir el `UUID` en `/etc/fstab` | `sudo systemctl restart beegfs-client` |

---

## Contexto: el RAID de Skynet

El RAID es **por hardware**, con una controladora Broadcom / LSI MegaRAID. El sistema lo ve como un solo disco, `sda` (32.7 TB), montado en `/raid`.

```bash
lspci | grep -i -E 'raid|storage|lsi|mega|perc|smart'
```

```text
6f:00.0 RAID bus controller: Broadcom / LSI MegaRAID 12GSAS/PCIe Secure SAS39xx
```

Rutas que usa BeeGFS:

| Componente | Ruta | Dónde vive |
| --- | --- | --- |
| Storage | `/raid/beegfs-storage` | RAID |
| Metadata | `/mnt/beegfs-meta` | NVMe del sistema (**no** está en el RAID) |
| Cliente (punto de montaje) | `/mnt/beegfs` | — |

> [!important] Antes de procesar
> El RAID tiene que estar montado para que los procesos (por ejemplo `run_workflow`) puedan leer y escribir datos. Si el cluster se cae, los discos dejan de poder escribir en `/raid`.

---

## Incidente 1: el sistema no arrancaba

![[Error discos del raid.png|400]]

### Causa

Al iniciar, el SO buscaba el disco del RAID por el `UUID` registrado en `/etc/fstab`. Ese `UUID` era incorrecto (no correspondía a ningún disco), así que el arranque se detenía.

### Recuperación desde el modo recovery

1. Crear una USB booteable con **Rocky Linux 10.1** y arrancar el `master` en **modo recovery**.
2. Abrir la terminal desde la USB.
3. Desmontar o deshabilitar los discos afectados para que el SO pueda iniciar.
4. Buscar el `UUID` real del RAID con `lsblk`, que lista todos los discos con su `UUID`:

   ```bash
   lsblk -f
   ```

   > [!warning] Dónde buscar el UUID
   > Al principio se buscó en una carpeta generada por la USB booteable, pero el `UUID` correcto estaba en **otro disco del cluster**. Hay que buscarlo en los discos del servidor, no en los de la USB.

5. Corregir la línea de `/raid` en `/etc/fstab`:

   ```bash
   sudo nano /etc/fstab
   ```

   ```text
   UUID=1bc201b2-17e5-4e07-b610-b9e0afe1c625 /          xfs   defaults                    0 0
   UUID=EA89-7C36                             /boot/efi  vfat  umask=0077,shortname=winnt  0 2
   UUID=089c1348-166e-426f-83ec-06d3fc51f5ad none       swap  defaults                    0 0
   #UUID=4ae5c637-136e-4459-be75-e6898bbfa506 /raid  xfs  defaults  0 0   # ❌ incorrecto
   UUID=ba36546f-838f-4cef-b48d-491c389d4308  /raid      xfs   defaults                    0 0   # ✅ correcto
   ```

6. Recargar systemd y montar para comprobar antes de reiniciar:

   ```bash
   sudo systemctl daemon-reload
   sudo mount -a
   df -h /raid
   ```

---

## Incidente 2: BeeGFS no se montó (9–13 oct 2026)

### Causa probable

El cliente intentó montar a las **17:22:04**, al mismo tiempo que arrancaban `beegfs-meta` y `beegfs-storage`, que todavía no estaban listos. El montaje falló con **código 32** y no se reintentó.

### Diagnóstico

1. **Comprobar que el RAID está detectado y montado.**

   ```bash
   lsblk
   df -h /raid
   ```

2. **Comprobar que los datos de BeeGFS están en el RAID.** La carpeta es de `root`; sin `sudo` da *Permission denied*.

   ```bash
   ls /raid
   sudo ls -la /raid/beegfs-storage
   ```

3. **Ver qué rutas usa BeeGFS.**

   ```bash
   grep -E 'storeStorageDirectory|storeMetaDirectory|storeMgmtdDirectory' /etc/beegfs/*.conf
   cat /etc/beegfs/beegfs-mounts.conf
   ```

4. **Revisar el estado de los servicios.** `beegfs-mgmtd`, `beegfs-meta` y `beegfs-storage` estaban activos; `beegfs-client` estaba en `failed` desde el arranque.

   ```bash
   systemctl status 'beegfs-*' --no-pager
   ```

> [!tip] Si el cliente vuelve a fallar
> La causa exacta aparece en:
>
> ```bash
> sudo dmesg | grep -i beegfs | tail -30
> journalctl -u beegfs-client -l --no-pager -n 50
> ```

### Solución aplicada

Con los servidores ya activos basta con reiniciar el cliente. **No hace falta reiniciar el servidor.**

```bash
sudo systemctl restart beegfs-client
systemctl status beegfs-client -l --no-pager
df -h /mnt/beegfs
```

**Resultado:** el servicio quedó `active (exited)` con estado `0/SUCCESS`, y `/mnt/beegfs` quedó montado con 33 TB (32 TB usados, 96 %).

Comprobar lectura y escritura:

```bash
ls /mnt/beegfs
touch /mnt/beegfs/prueba && rm /mnt/beegfs/prueba
```

> [!note] Nodos de cómputo
> Si un nodo de cómputo no ve `/mnt/beegfs`, se aplica el mismo `restart` del cliente **en ese nodo**.

---

## Prevención

### Que el cliente espere a los servidores

```bash
sudo systemctl edit beegfs-client
```

```ini
[Unit]
After=beegfs-mgmtd.service beegfs-meta.service beegfs-storage.service network-online.target
Wants=network-online.target
```

### Que un disco faltante no bloquee el arranque (recomendado)

Agregar `nofail` a la línea de `/raid` en `/etc/fstab`. Si el RAID no aparece, el sistema arranca igual y se puede entrar por SSH a revisar, en vez de necesitar la USB:

```text
UUID=ba36546f-838f-4cef-b48d-491c389d4308  /raid  xfs  defaults,nofail  0 0
```

---

## Pendientes

- [ ] Aplicar la dependencia de systemd en `beegfs-client` antes del próximo reinicio.
- [ ] Liberar espacio: el storage está al **96 %**. Para ver qué ocupa más:
  ```bash
  sudo du -sh /mnt/beegfs/* | sort -h | tail
  ```
- [ ] Revisar `beegfs-client` en los nodos de cómputo:
  ```bash
  systemctl is-active beegfs-client; df -h /mnt/beegfs
  ```
- [ ] Incluir `/mnt/beegfs-meta` en los respaldos (está en el NVMe del sistema, no en el RAID).
- [ ] Documentar cómo entrar al modo recovery **sin USB**.
- [ ] Homologar versión del SO: el **nodo 2** tiene Rocky Linux **10.2** y no 10.1.
- [ ] *(Opcional, SELinux)* `storeClientXAttrs = true` en `beegfs-meta.conf` y `sysSelinuxEnabled = true` en `beegfs-client.conf`.

> [!info] Hilos de `beegfs-meta`
> Unos 250 hilos en `beegfs-meta` son normales. Ese número ya aparecía antes de la solución y depende de `tuneNumWorkers` y de los núcleos de CPU.
