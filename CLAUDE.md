# Contexto: documentación del InSAR lab

## Qué es este repositorio
- Vault de Obsidian dentro de `content/`, publicado con **Quartz 5** en GitHub Pages.
- Son los manuales del **equipo de cómputo del InSAR lab** (Facultad de Ingeniería, UNAM).
- Manual actual: `content/Sobre el Cluster y Slurm/` (usuario, plantillas, comandos y `Manual del Administrador/`).
- Historial de cambios: `content/Sobre el Cluster y Slurm/Registro de cambios del manual.md`. Agrega una sección nueva **arriba** en cada revisión.

## El clúster (Skynet)
- Rocky Linux, gestor de trabajos Slurm. Acceso: `ssh` a `132.248.59.7`.
- Nodos:

  | Nodo | Particiones | CPUs | RAM | Enlace | Nota |
  |---|---|---|---|---|---|
  | `master` | `all` (por defecto) | 40 | ~126 GB | 10 Gb | Coordinador |
  | `node1` | `all` | 56 | ~257 GB | 10 Gb | |
  | `node2` | `extra` y `n2gpu` | 56 | ~240 GB | 1 Gb | GPU NVIDIA L4: `--gres=gpu:l4:1` |
  | `node3` | `extra` | 24 | ~31 GB | 1 Gb | |
  | `node4` | `extra` | 56 | ~257 GB | 1 Gb | |

- Los `home` están en BeeGFS: `/mnt/beegfs/insarlab/<usuario>`. UIDs 5000–5999, grupo `insarlab` (GID 5000).
- Usuarios: se crean con Ansible en `/home/admin/ansible` (`playbooks/users.yaml`, variables en `roles/users/vars/main.yaml`).
- Hay dos flujos de procesamiento: el de este clúster y el de la supercomputadora Miztli (UNAM).

## Convenciones de escritura
- Español. Notas atómicas conectadas con wikilinks `[[ ]]`; Quartz los convierte en links.
- Nombres de archivo sin acentos (por ejemplo, `Creacion de Usuario.md`). El título con acentos va en el frontmatter `title:`.
- `#SBATCH` siempre entre backticks o dentro de bloques de código (fuera de código, Obsidian y Quartz lo toman como etiqueta).
- Un `title:` que contenga `#` va entre comillas en el YAML.
- Dentro de una tabla, el alias de un wikilink se escapa: `[[Nota\|alias]]`.
- No uses puntos en los nombres de nota (`slurm.conf.md` rompería la URL). La nota se llama `Archivo de configuracion slurm conf.md`.
- No crees carpetas llamadas `templates`: Quartz las ignora por defecto. Por eso la carpeta es `Plantillas/`.
- Al terminar, verifica que todos los wikilinks apunten a una nota existente.

## Commits de git
- **No agregues la línea `Co-Authored-By: Claude …`** (ni otro trailer de atribución) a los mensajes de commit. Los commits van solo a nombre del autor configurado en git.
- Mensajes de commit en español y descriptivos; si el cambio es en la documentación, actualiza también `Registro de cambios del manual.md`.

## No modificar sin permiso
- `Manual del Administrador/Uso de Clúster ADMIN.md` y `Manual del Administrador/Comandos útiles en el manejo del instalador.md`: son procedimientos verificados.

## Pendientes (tareas `- [ ]` y etiqueta `#Pendiente`)
- `Manual del Administrador/BeeGFS - Pendientes por investigar.md`: arquitectura real de BeeGFS. Al resolverla, reescribir `BeeGFS.md`.
- `Manual del Administrador/Archivo de configuracion slurm conf.md`: el `slurm.conf` que trae es **ilustrativo**; hay que reemplazarlo por el real. Incluye propuestas de mejora.
- `Registro de cambios del manual.md`: lista de verificaciones pendientes.
- Etiqueta `#Pendiente` en: `Creacion de Usuario.md` (instrucciones para Windows), `Plantillas/Plantillas de Slurm.md` (carpeta común de plantillas) y `BeeGFS.md`.
