---
title: "Plantilla: multihilo (Python, ISCE2, MintPy, GDAL…)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** cuando tu programa es **un solo proceso que usa varios núcleos**: NumPy/SciPy, `multiprocessing`, ISCE2, MintPy, MiaplPy, GDAL, etc. Es la plantilla más usada en el procesamiento InSAR.

```bash
#!/bin/bash
#SBATCH --job-name=multihilo
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=extra
#SBATCH --exclude=node2              # deja libre el nodo con GPU
#SBATCH --ntasks=1                   # UN programa…
#SBATCH --cpus-per-task=16           # …que usa 16 núcleos
#SBATCH --mem=64G
#SBATCH --time=12:00:00

set -euo pipefail
echo "Job $SLURM_JOB_ID | nodo: $(hostname) | CPUs: $SLURM_CPUS_PER_TASK | inicio: $(date)"

# Que las librerías usen exactamente los núcleos reservados (ni más ni menos)
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OPENBLAS_NUM_THREADS=$SLURM_CPUS_PER_TASK
export MKL_NUM_THREADS=$SLURM_CPUS_PER_TASK
export NUMEXPR_MAX_THREADS=$SLURM_CPUS_PER_TASK

# ---- activa aquí el entorno de tu proyecto (pixi, conda, venv…) ----
# ejemplo con pixi:  cd ~/proyectos/mi_proyecto && pixi run python procesar.py
python3 procesar.py --workers "$SLURM_CPUS_PER_TASK"

echo "Fin: $(date)"
```

**Ajusta:** `--cpus-per-task` (máx. 56 en `node1/2/4` y 24 en `node3`), `--mem`, `--time` y el comando.

> [!tip] Pásale el número de núcleos a tu programa
> Si tu script tiene una opción como `--workers`, `-n` o `num_workers`, usa `$SLURM_CPUS_PER_TASK` en lugar de escribir un número fijo. Así, si cambias la directiva, el programa se ajusta solo.

> [!warning] Con `node3` en la partición
> `node3` solo tiene ~31 GB. Si pides `--mem=64G` en `extra`, Slurm **nunca** lo mandará ahí (lo cual está bien). Si pides poca memoria, sí puede caer en `node3`.

Ver también: [[Archivo de trabajo y directivas SBATCH|ntasks vs cpus-per-task]] · Volver a [[Plantillas de Slurm]].
