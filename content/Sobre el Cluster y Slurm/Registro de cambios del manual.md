---
title: Registro de cambios del manual
tags:
  - manual
  - registro
---

> [!info] Para qué sirve esta nota
> Lleva el historial de cambios del manual sobre el clúster y Slurm: qué se agregó, qué se corrigió y qué falta verificar. Agrega una sección nueva **arriba** cada vez que se haga una revisión.

## 2026-10-09: Ampliación del manual de Slurm y del Manual del Administrador

### Notas nuevas

| Nota | Sección | Contenido |
|---|---|---|
| [[Guia rapida - Tu primer trabajo en Slurm]] | Usuario | Ruta corta de 9 pasos para encolar el primer trabajo, más una tabla de errores frecuentes y reglas de convivencia |
| [[Grupos de nodos y particiones]] | Usuario | Grupos de nodos (alta capacidad, GPU, ligero, master), tabla de decisión, advertencia sobre `--nodelist` con varios nodos |
| [[Archivo de trabajo y directivas SBATCH]] | Usuario | Reglas de las directivas, `--ntasks` vs `--cpus-per-task`, `--mem` vs `--mem-per-cpu`, variables `$SLURM_*`, cabecera estándar del lab |
| [[Ejemplo de flujo de trabajo completo]] | Usuario | Caso de principio a fin: dejar que Slurm elija el nodo, nodo específico o varios nodos; memoria, tiempo y ajuste con `sacct` |
| [[Protocolo SSH]] | Usuario | Llaves, `scp`/`rsync` y errores de conexión (antes era un link roto) |
| [[Plantillas de Slurm]] | Plantillas | Índice y forma de uso de las plantillas |
| [[Plantilla - Trabajo basico (CPU)]] | Plantillas | Script en serie en `node3` |
| [[Plantilla - Multihilo (cpus-per-task)]] | Plantillas | Python, ISCE2, MintPy, GDAL con varios núcleos |
| [[Plantilla - Nodo especifico]] | Plantillas | `--nodelist` y la pareja nodo ↔ partición |
| [[Plantilla - Grupo de nodos de alta capacidad]] | Plantillas | `node1` o `node4`, el que se libere primero |
| [[Plantilla - GPU (n2gpu)]] | Plantillas | GPU L4 con `--gres` |
| [[Plantilla - Arreglo de trabajos (job array)]] | Plantillas | Misma tarea sobre muchas entradas |
| [[Plantilla - Cadena de trabajos (dependencias)]] | Plantillas | Pasos encadenados con `--dependency` |
| [[Comando sbatch]] | Comandos | Formas de uso, opciones en terminal, formas cortas, errores al encolar |
| [[Comando squeue]] | Comandos | Lectura de la salida, `--start`, formato a la medida, alias |
| [[Comando sinfo]] | Comandos | Vista por nodo, CPUs y memoria libres, estados, columna `AVAIL` |
| [[Comando sacct]] | Comandos | Pasos `.batch`/`.extern`, `MaxRSS`, cómo ajustar recursos |
| [[Comando scancel y scontrol]] | Comandos | Cancelar, inspeccionar, pausar y entrar al nodo del trabajo |
| [[Manual del Administrador]] | Admin | Índice y tabla "tengo este problema → ¿a dónde voy?" |
| [[Uso de Clúster ADMIN]] | Admin | Integrada **sin cambios** (procedimiento verificado) |
| [[Comandos útiles en el manejo del instalador]] | Admin | Integrada **sin cambios** (procedimiento verificado) |
| [[Archivo de configuracion slurm conf]] | Admin | Qué es `slurm.conf`, parámetros clave, cómo aplicar cambios, propuestas de mejora |
| [[Recuperar nodos y servicios de Slurm]] | Admin | Recetas para `down`, `down*`, `drain`, `comp`, `inval`, mantenimiento y arranque tras un apagón |
| [[100 ACADEMIC/Servicio Social/40 Legend/BeeGFS]] | Admin | Componentes, comandos de estado (v7 y v8), cómo levantarlo, problemas frecuentes |
| [[BeeGFS - Pendientes por investigar]] | Admin | Lista de tareas para documentar la arquitectura real de BeeGFS |
| [[Registro de cambios del manual]] | — | Esta nota |

### Notas modificadas

| Nota | Cambio | Motivo |
|---|---|---|
| `index.md` | Se corrigió el link al manual (`inSAR` → `InSAR`) y se agregaron la guía rápida y el Manual del Administrador | El link podía romperse en Quartz |
| [[Manual sobre el Cluster del InSAR lab y Slurm]] | Reorganizado como ruta de aprendizaje en 5 bloques y con atajo a la guía rápida | Hacerlo más directo para quien empieza |
| [[Logica de trabajo en insarlab]] | "Sbre" → "Sobre"; encabezado "Sobre el almacenamiento"; explicación de BeeGFS y `/tmp`; link a [[Protocolo SSH]] | Corregir anclas rotas y explicar el almacenamiento compartido |
| [[Creacion de Usuario]] | Se agregaron `chmod 600` (y su equivalente en Windows) y el aviso del salto de línea final; el nombre de la llave se unificó como `insar_key` | Sin esos permisos `ssh` rechaza la llave; el nombre sugerido no coincidía |
| [[Uso del Cluster]] | Ejemplo `--ntasks=4` → `--ntasks=1 --cpus-per-task=4`; `--nodelist=node4` con advertencia; links nuevos | `ntasks` no es "número de núcleos" para programas multihilo |
| [[Manejo de Slurm]] | Links "A detalle" en cada comando, callout de atajos, link a admin, URL de exit codes corregida | Conectar con las notas nuevas |
| [[Arquitectura del Cluster del InSAR lab]] | Advertencias: `master` está en la partición por defecto `all` y `node2` está en dos particiones | Evitar que los trabajos caigan en el master y que se bloquee la GPU |
| [[Catalogo de estados de los nodos en sinfo]] | Reescrito: `drain` vs `drng`, más estados (`comp`, `resv`, `fail`, `inval`, `unk`), tabla de sufijos, aclaración de `up`, columna "¿qué hago?" | Corregir el significado de `drain` y completar el catálogo |
| [[Codigos de razon de un Job (Job Reason Codes)]] | Más razones: `DependencyNeverSatisfied`, `JobHeld*`, `PartitionDown`, `InvalidAccount`, etc. | Completar los casos frecuentes |
| [[Estados de un Job (Job State Codes)]] | Se agregaron el estado `COMPLETING (CG)` y frontmatter | Completar |
| [[Tabla de comandos de Slurm]] | Se agregaron la fila de `scontrol` y la columna "A detalle" | Conectar con las notas de comandos |
| [[Tabla de directivas más usadas]] | Reorganizada por grupos; se agregaron forma corta, `--chdir`, `--exclusive`, `--ntasks-per-node`, `--mem-per-cpu`, `--array`, `--dependency`, `--begin`; se corrigió la descripción de `--nodelist` | `--nodelist` con varios nodos pide todos a la vez |
| [[Ejemplos de script usando sbatch de Slurm]] | Callout al final con link a las plantillas | Conectar |
| [[Crear usuario (desde admin)]] | Callout de "método anterior" (el vigente es con Ansible), aviso `nodo<n>` → `node<n>`, ancla corregida | Evitar que se siga un procedimiento desactualizado |

### Notas sin cambios
[[Cluster Computing]] · [[Que es Slurm]] · [[Conexion al cluster]] · [[Forma general de un script sh para mandar un trabajo usando sbatch de slurm]] · [[Arquitectura del Cluster de InSAR lab (para admin)]]

### Pendientes de verificar en el clúster
- [ ] Sustituir el `slurm.conf` ilustrativo por el real (ver [[Archivo de configuracion slurm conf]]).
- [ ] Confirmar si `--mem` se aplica de verdad (cgroups / `SelectTypeParameters`).
- [ ] Confirmar la versión de BeeGFS y los subcomandos de la v8 (`beegfs --help`).
- [ ] Resolver la lista de [[BeeGFS - Pendientes por investigar]].
- [ ] Confirmar si hay correo para `--mail-type` y si está instalado `seff`.
