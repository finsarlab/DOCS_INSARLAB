---
title: "Plantilla: trabajo básico (CPU, 1 núcleo)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** scripts en serie (un solo núcleo), pruebas, conversiones o descargas pequeñas. Va a `node3`, el nodo ligero, para no ocupar los grandes.

```bash
#!/bin/bash
#SBATCH --job-name=basico
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=extra
#SBATCH --nodelist=node3
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G
#SBATCH --time=01:00:00

set -euo pipefail
echo "Job $SLURM_JOB_ID | nodo: $(hostname) | inicio: $(date)"

# ---- tus comandos ----
python3 mi_script.py

echo "Fin: $(date)"
```

**Ajusta:** `--job-name`, `--mem` (máx. ~28G en `node3`), `--time` y la línea `python3 mi_script.py`.
**Si `node3` está ocupado o caído:** borra la línea `--nodelist` y Slurm elegirá cualquier nodo de `extra`.

Volver a [[Plantillas de Slurm]].
