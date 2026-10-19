---
title: "Plantilla: arreglo de trabajos (job array)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** cuando debes hacer **lo mismo muchas veces con distinto dato de entrada**: procesar 40 pares interferométricos, 12 fechas, 100 archivos… En lugar de encolar 40 scripts, encolas **uno** con `--array` y Slurm crea 40 subtrabajos.

Cada subtrabajo recibe un número distinto en `$SLURM_ARRAY_TASK_ID`, que usas para elegir su entrada.

```bash
#!/bin/bash
#SBATCH --job-name=arreglo
#SBATCH --output=logs/%x_%A_%a.out   # %A = Job ID del arreglo, %a = índice
#SBATCH --error=logs/%x_%A_%a.err
#SBATCH --partition=extra
#SBATCH --exclude=node2
#SBATCH --array=1-40%4               # 40 subtrabajos, máximo 4 al mismo tiempo
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G                    # memoria POR subtrabajo
#SBATCH --time=02:00:00              # tiempo POR subtrabajo

set -euo pipefail

# lista.txt tiene una entrada por línea (rutas, fechas, IDs de escena…)
ENTRADA=$(sed -n "${SLURM_ARRAY_TASK_ID}p" lista.txt)
echo "Subtrabajo $SLURM_ARRAY_TASK_ID de $SLURM_ARRAY_JOB_ID | nodo: $(hostname) | entrada: $ENTRADA"

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
python3 procesar_uno.py "$ENTRADA"
```

**Preparar la lista y encolar:**
```bash
ls datos/*.h5 > lista.txt          # una entrada por línea
wc -l lista.txt                    # ¿cuántas son? → ajusta --array=1-N
sbatch trabajo_arreglo.sh
```

**Seguimiento:**
```bash
squeue -u $USER                    # verás 10500_[5-40] PD y 10500_1 … 10500_4 R
sacct -j 10500 -X --format=JobID,State,ExitCode,Elapsed
scancel 10500_7                    # cancela solo el subtrabajo 7
scancel 10500                      # cancela todo el arreglo
```

> [!tip] El `%4` es cortesía
> Limita cuántos subtrabajos corren a la vez para no acaparar el clúster. Ajústalo según qué tan lleno esté (`sinfo`).

Volver a [[Plantillas de Slurm]].
