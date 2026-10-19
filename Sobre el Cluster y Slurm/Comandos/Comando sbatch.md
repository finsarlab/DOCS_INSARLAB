---
title: sbatch — encolar un trabajo
tags:
  - slurm
  - comandos
---

> [!summary]
> `sbatch` entrega un [[Archivo de trabajo y directivas SBATCH|archivo de trabajo]] a Slurm. Slurm lo pone en la cola, te devuelve un **Job ID** y lo ejecuta en segundo plano cuando haya recursos. Puedes cerrar tu terminal: el trabajo sigue.

## Sintaxis

```bash
sbatch [opciones] archivo.sh [argumentos para el script]
```

## Las 4 formas en que lo usarás

**1. Con una plantilla (la forma habitual en el InSAR lab)**
```bash
sbatch trabajo.sh
# Submitted batch job 10500
```

**2. Con una plantilla, cambiando recursos sin editar el archivo**
```bash
sbatch --mem=128G --time=2-00:00:00 trabajo.sh
```
Lo que escribes en la terminal **le gana** a las directivas `#SBATCH` del archivo.

**3. Pasándole argumentos al script**
```bash
sbatch trabajo.sh volcan_popo 2026-01-01
```
Dentro del script los recibes como `$1` (`volcan_popo`) y `$2` (`2026-01-01`). Así una sola plantilla sirve para varios casos.

**4. Una línea, sin archivo (`--wrap`)**: solo para algo muy rápido
```bash
sbatch -p extra -c 4 --mem=8G -t 00:30:00 -o logs/%j.out --wrap="python3 convertir.py datos.h5"
```
Útil para pruebas, pero **no queda registro** del comando en un archivo. Para trabajo real, usa plantillas.

## Opciones que conviene conocer en la terminal

| Opción | Qué hace |
|---|---|
| `--test-only` | **No encola.** Valida el script y te dice en qué nodo y a qué hora arrancaría. Ideal antes de lanzar algo grande. |
| `--parsable` | Imprime solo el Job ID (`10500`), útil para guardarlo en una variable. Ver [[Plantilla - Cadena de trabajos (dependencias)]]. |
| `--dependency=afterok:ID` | Espera a que otro trabajo termine bien. |
| `--array=1-20%4` | Crea un [[Plantilla - Arreglo de trabajos (job array)\|arreglo de trabajos]]. |
| `--hold` | Encola, pero en pausa. Se libera con `scontrol release ID`. |
| `--begin=now+2hours` / `--begin=22:00` | No arranca antes de esa hora. |
| `--chdir=/ruta` | Carpeta de trabajo (por defecto, desde donde ejecutas `sbatch`). |

### Formas cortas
| Larga | Corta | | Larga | Corta |
|---|---|---|---|---|
| `--job-name` | `-J` | | `--cpus-per-task` | `-c` |
| `--partition` | `-p` | | `--nodes` | `-N` |
| `--ntasks` | `-n` | | `--time` | `-t` |
| `--output` | `-o` | | `--nodelist` | `-w` |
| `--error` | `-e` | | `--exclude` | `-x` |

Todas las directivas están en la [[Tabla de directivas más usadas]].

## Errores frecuentes al encolar

| Mensaje | Significa |
|---|---|
| `Requested node configuration is not available` | Ningún nodo de esa partición tiene lo que pides (CPUs, RAM o GPU), o tu `--nodelist` no pertenece a la partición. |
| `invalid partition specified` | Nombre de partición incorrecto. |
| `Invalid generic resource (gres) specification` | `--gres` mal escrito o pedido en una partición sin GPU. |
| `Node count specification invalid` | `--nodes` no concuerda con `--nodelist`. |
| `This does not look like a batch script` | Falta `#!/bin/bash` en la primera línea. |

> [!tip] Sin `.out` ni `.err`
> Si `sbatch` aceptó el trabajo pero nunca aparecen los archivos de salida, casi siempre es porque la carpeta de `--output` no existe. Ver [[Archivo de trabajo y directivas SBATCH#Reglas de oro de las directivas|reglas de oro]].

Siguiente: [[Comando squeue]] · Volver a [[Manejo de Slurm]]
