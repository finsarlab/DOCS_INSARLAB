---
title: Plantillas de Slurm
tags:
  - slurm
  - plantilla
---

> [!summary] ¿Qué es una plantilla?
> Un [[Archivo de trabajo y directivas SBATCH|archivo de trabajo]] ya armado y probado que copias, ajustas en 2 o 3 líneas y encolas con `sbatch`. En el InSAR lab **se trabaja casi siempre con plantillas**: es más rápido, más reproducible y más fácil de revisar entre compañeros que escribir opciones largas en la terminal.

## Elige tu plantilla

| Quiero… | Plantilla |
|---|---|
| Correr un script sencillo en un núcleo | [[Plantilla - Trabajo basico (CPU)]] |
| Correr Python, ISCE2, MintPy, GDAL… usando varios núcleos | [[Plantilla - Multihilo (cpus-per-task)]] |
| Que corra en un nodo en particular | [[Plantilla - Nodo especifico]] |
| Que corra en el nodo grande que se libere primero (`node1` o `node4`) | [[Plantilla - Grupo de nodos de alta capacidad]] |
| Usar la GPU L4 | [[Plantilla - GPU (n2gpu)]] |
| Lanzar la misma tarea sobre muchas fechas, escenas o archivos | [[Plantilla - Arreglo de trabajos (job array)]] |
| Ejecutar varios pasos en orden (paso 2 espera al paso 1) | [[Plantilla - Cadena de trabajos (dependencias)]] |

¿No sabes cuál? Lee primero [[Grupos de nodos y particiones]].

## Cómo usar una plantilla (siempre los mismos 5 pasos)

```bash
# 1. Ve a la carpeta de tu proyecto y crea la carpeta de registros
cd ~/proyectos/mi_proyecto
mkdir -p logs

# 2. Crea el archivo y pega la plantilla
nano trabajo.sh

# 3. Ajusta: --job-name, recursos (cpus, mem, time) y los comandos del final

# 4. (Opcional) Valida sin encolar: dice si es válido y cuándo arrancaría
sbatch --test-only trabajo.sh

# 5. Encola
sbatch trabajo.sh
```
Después sigue tu trabajo con [[Comando squeue|squeue]] y, cuando termine, revisa el resultado con [[Comando sacct|sacct]].

> [!tip] Una plantilla, muchos tamaños
> Lo que pongas en la terminal le gana al archivo. Puedes dejar la plantilla igual y cambiar recursos al encolar:
> ```bash
> sbatch --mem=128G --cpus-per-task=32 trabajo.sh
> ```

> [!info] Propuesta para el equipo
> Guardar estas plantillas en una carpeta común de BeeGFS (por ejemplo `/mnt/beegfs/insarlab/plantillas/`) para que cualquiera pueda hacer `cp /mnt/beegfs/insarlab/plantillas/multihilo.sh .` sin copiar y pegar desde la web. #Pendiente

## Convenciones de todas las plantillas
- `--output`/`--error` van a `logs/%x_%j.out|err` (nombre del trabajo + Job ID).
- Siempre declaran `--partition`, `--mem` y `--time`.
- Usan `set -euo pipefail` para detenerse ante el primer error.
- Imprimen nodo, Job ID y hora de inicio y fin, lo que facilita depurar.
