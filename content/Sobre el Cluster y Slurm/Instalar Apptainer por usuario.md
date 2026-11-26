---
title: Instalar Apptainer por usuario (sin sudo)
tags:
  - contenedores
  - apptainer
  - guia-rapida
  - beegfs
  - pendiente
---

> [!summary] Para quién es esta guía
> Para cualquier usuario del clúster que quiera correr contenedores (por ejemplo, OpenDroneMap) **sin pedir permisos de administrador**. Al terminar tendrás Apptainer 1.5.5 instalado **solo en tu cuenta**, visible desde `master` y desde todos los nodos, sin tocar el software del sistema.
>
> **Necesitas:** una cuenta en el clúster y saber conectarte por SSH. Si aún no mandas trabajos a Slurm, empieza por [[Guia rapida - Tu primer trabajo en Slurm]].

## ¿Por qué por usuario y no para todo el clúster?

- Skynet **no tiene** ningún motor de contenedores instalado (ni Apptainer, ni Singularity, ni Docker, ni Podman).
- Instalarlo con `admin` y `sudo` lo instala para **todos los usuarios** de esa máquina y puede romper dependencias del software que ya usa el laboratorio.
- Instalado en tu `home`, Apptainer queda **aislado**: solo lo usa quien lo tenga en su `PATH`, y cada usuario puede tener su propia versión sin afectar a los demás.

Como tu `home` está en BeeGFS (`/mnt/beegfs/insarlab/<usuario>`), lo que instales ahí lo ven `master` y todos los nodos. Ver [[Logica de trabajo en insarlab]].

> [!important] Nunca uses `sudo` en esta guía
> Si algún paso te pide `sudo`, **detente**: significa que está intentando escribir en una ruta del sistema (`/usr`, `/opt`, `/etc`) y eso afectaría a todos.

### ¿Por qué Apptainer y por qué la versión 1.5.5?

| Decisión | Motivo |
|---|---|
| **Apptainer** en vez de Docker | Es el estándar en clústeres HPC: corre imágenes de Docker **sin privilegios de root** y no necesita un servicio corriendo en segundo plano |
| Versión **1.5.x** | Las versiones anteriores a la 1.5.1 tienen una vulnerabilidad crítica (CVE-2026-48785)[^1]. La 1.5 permite además crear imágenes `.sif` reproducibles, lo que sirve para homologar versiones entre Skynet y Miztli |
| Versión **1.5.5** exacta | Es la que publica EPEL 10.2 (el repositorio de paquetes extra de Rocky Linux). EPEL 10.1 ya está archivado y vacío |
| Modo **sin setuid** | Usa *user namespaces* del kernel en lugar de un binario con permisos de root. Es más seguro y no requiere administrador |

### Requisitos del clúster (ya verificados)

Estos requisitos los configura un administrador. Apptainer solo los **usa**; no los modifica.

| Requisito | Estado en Skynet | Cómo comprobarlo |
|---|---|---|
| *User namespaces* habilitados | ✅ en `master`, `node1` y `node2` | `sysctl user.max_user_namespaces` (debe ser mayor que 0) |
| Módulo `fuse` cargado | ✅ | `lsmod \| grep fuse` |
| cgroups v2 | ✅ | `stat -fc %T /sys/fs/cgroup` (debe decir `cgroup2fs`) |
| Internet | Solo en `master` | Por eso **toda la instalación se hace en `master`** |

## 1. Conéctate a master con tu usuario

Conéctate como tu usuario normal, **no como `admin`**.

```bash
whoami      # debe mostrar tu usuario, no admin
hostname    # debe mostrar master
```

Los nodos de cómputo no tienen internet, así que la descarga solo funciona en `master`.

## 2. Verifica las herramientas del instalador

```bash
which curl rpm2cpio cpio
```

El script de instalación necesita estas tres herramientas. Si falta alguna, avisa al equipo de cómputo.

## 3. Descarga el instalador y el paquete de Apptainer

```bash
mkdir -p ~/apps && cd ~/apps

# Script oficial de instalación sin privilegios
curl -sO https://raw.githubusercontent.com/apptainer/apptainer/main/tools/install-unprivileged.sh

# Paquete de Apptainer 1.5.5 de EPEL 10.2
curl -O https://dl.fedoraproject.org/pub/epel/10.2/Everything/x86_64/Packages/a/apptainer-1.5.5-1.el10_2.x86_64.rpm
```

> [!tip] Revisa el script antes de ejecutarlo
> `less install-unprivileged.sh`. Es buena práctica no ejecutar scripts de internet sin leerlos.

## 4. Instala Apptainer en tu home

```bash
bash install-unprivileged.sh -e -v $PWD/apptainer-1.5.5-1.el10_2.x86_64.rpm ~/apps/apptainer-1.5.5
```

| Parte del comando | Para qué sirve |
|---|---|
| `-e` | Muestra los errores de descarga si algo falla |
| `-v <ruta del .rpm>` | Usa el paquete que descargaste en vez de buscar uno automáticamente |
| `~/apps/apptainer-1.5.5` | Carpeta donde queda la instalación; incluir la versión en el nombre permite tener varias |

> [!warning] Pasa el `.rpm` con `-v`
> Si no le indicas el paquete, el script detecta la versión del sistema de `master` (Rocky 10.1) y lo busca en EPEL 10.1, que está vacío. Con el archivo `.rpm`, toma la versión del nombre del paquete.

## 5. Configura tu entorno

Agrega estas líneas al final de tu `~/.bashrc`:

```bash
export PATH=$HOME/apps/apptainer-1.5.5/bin:$PATH
export APPTAINER_CACHEDIR=$HOME/odm/cache
export APPTAINER_TMPDIR=$HOME/odm/tmp
```

Luego aplícalas y crea las carpetas:

```bash
source ~/.bashrc
mkdir -p ~/odm/{images,datasets,cache,tmp,logs,runs}
```

`APPTAINER_CACHEDIR` y `APPTAINER_TMPDIR` mandan la caché y los archivos temporales a carpetas propias. Las imágenes pesan varios GB y, sin estas variables, llenarían rutas que no controlas.

## 6. Verifica la instalación

```bash
apptainer --version
which apptainer
```

```bash
# Salida esperada (ilustrativa)
apptainer version 1.5.5-1.el10_2
/mnt/beegfs/insarlab/<usuario>/apps/apptainer-1.5.5/bin/apptainer
```

Si `which` apunta a tu carpeta `~/apps/...`, estás usando tu instalación personal. Si apuntara a `/usr/bin/...`, sería una de sistema.

Comprueba que los nodos también la ven:

```bash
srun -p all -w node1 apptainer --version
srun -p n2gpu -w node2 apptainer --version
```

## 7. Descarga tus imágenes en master

Las imágenes se descargan **solo en `master`** y se guardan en BeeGFS para que los nodos puedan leerlas. Usa siempre una **versión fija**, nunca `latest`, porque `latest` cambia sin aviso y rompe la reproducibilidad.

```bash
cd ~/odm/images
apptainer pull odm_3.6.2.sif     docker://opendronemap/odm:3.6.2
apptainer pull odm_3.6.2-gpu.sif docker://opendronemap/odm:3.6.2-gpu
```

La conversión tarda varios minutos. Es normal.

## 8. Prueba que el contenedor corre en un nodo

```bash
srun -p all -w node1 apptainer exec --cleanenv ~/odm/images/odm_3.6.2.sif cat /etc/os-release
```

Para probar la GPU de `node2` usa la opción `--nv`, que conecta la GPU del nodo con el contenedor:

```bash
srun -p n2gpu -w node2 --gres=gpu:l4:1 apptainer exec --nv --cleanenv ~/odm/images/odm_3.6.2-gpu.sif nvidia-smi -L
```

Si aparece la `NVIDIA L4`, la GPU está disponible dentro del contenedor. Ver [[Plantilla - GPU (n2gpu)]].

> [!tip] Usa siempre `--cleanenv`
> Evita que las variables de tu entorno (por ejemplo, de `pixi` o `conda`) se filtren al contenedor y provoquen conflictos de Python.

> [!success] Listo
> Ya puedes correr contenedores en el clúster con [[Comando sbatch|sbatch]]. Para medir tiempo y memoria de tus pruebas usa [[Comando sacct|sacct]].

## Si algo salió mal

| Síntoma | Causa probable | Solución |
|---|---|---|
| El instalador no descarga nada | Lo corriste en un nodo sin internet | Vuelve a `master` y repite el paso 4 |
| El instalador busca en EPEL 10.1 | No le pasaste el `.rpm` con `-v` | Usa el comando del paso 4 tal cual |
| `apptainer: command not found` en un trabajo de Slurm | El trabajo no cargó tu `~/.bashrc` | Agrega `export PATH=$HOME/apps/apptainer-1.5.5/bin:$PATH` dentro del script `.sbatch` |
| `which apptainer` apunta a `/usr/bin` | Hay una versión de sistema antes que la tuya en el `PATH` | Revisa el orden en `~/.bashrc`: tu ruta debe ir **antes** de `$PATH` |
| `apptainer pull` falla en un nodo | Los nodos no tienen internet | Haz todos los `pull` en `master` |
| Se llena tu `home` | Caché de Apptainer sin limpiar | `apptainer cache clean` |
| Algo pide `sudo` | El paso intenta escribir en una ruta del sistema | Detente y revisa que estés instalando en `~/apps` |

## Desinstalar

> [!warning] Borra solo tu instalación
> Revisa la ruta antes de ejecutar `rm -rf`.

```bash
rm -rf ~/apps/apptainer-1.5.5
```

Después quita las líneas de Apptainer de tu `~/.bashrc`. Tus imágenes `.sif` y tus datos se conservan y funcionarán con cualquier otra instalación de Apptainer.

## Pendientes por verificar

- [ ] Confirmar que todo el equipo completó la instalación con este procedimiento exacto (`apptainer --version` en cada cuenta) #Pendiente
- [ ] Confirmar si la GPU está declarada como GRES en Slurm (`scontrol show node node2 | grep -i gres`) #Pendiente
- [ ] Probar en `node3` y en `node4` cuando regrese a servicio (`srun -p extra -w node3 apptainer --version`) #Pendiente
- [ ] Decidir si se instalará una versión de sistema en todos los nodos; requiere actualizar `master` y `node1` a Rocky 10.2 #Pendiente
- [ ] Escribir las notas relacionadas: usuarios, máquinas y permisos en Skynet; OpenDroneMap en Skynet #Pendiente

---
Volver a [[Manual sobre el Cluster del InSAR lab y Slurm]]

[^1]: Fuentes: [instalación de Apptainer (documentación oficial)](https://apptainer.org/docs/admin/latest/installation.html), [script install-unprivileged.sh](https://raw.githubusercontent.com/apptainer/apptainer/main/tools/install-unprivileged.sh), [anuncio de Apptainer 1.5.0](https://apptainer.org/news/apptainer-1-5-0-20260506/).
