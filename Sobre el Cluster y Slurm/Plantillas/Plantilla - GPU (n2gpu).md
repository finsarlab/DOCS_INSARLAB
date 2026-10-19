---
title: "Plantilla: GPU (partición n2gpu)"
tags:
  - slurm
  - plantilla
  - gpu
---

**Cuándo usarla:** cuando tu código usa la **GPU NVIDIA L4** de `node2`: PyTorch, TensorFlow, CuPy, modelos de HuggingFace, etc.

```bash
#!/bin/bash
#SBATCH --job-name=gpu
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=n2gpu
#SBATCH --gres=gpu:l4:1              # OBLIGATORIA: sin ella no tendrás acceso a la GPU
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=06:00:00

set -euo pipefail
echo "Job $SLURM_JOB_ID | nodo: $(hostname) | GPU: ${CUDA_VISIBLE_DEVICES:-ninguna} | inicio: $(date)"
nvidia-smi                           # deja registro del estado de la GPU en el .out

# Cachés en lugares seguros (evita errores de bloqueo de archivos en red)
export PIXI_CACHE_DIR=/tmp/pixi-cache-$USER
export HF_HOME="$HOME/.cache/huggingface"

cd "$SLURM_SUBMIT_DIR"
pixi run python inferencia.py

echo "Fin: $(date)"
```

> [!important]
> - Solo hay **una** GPU. Si otro compañero la está usando, tu trabajo esperará con la razón `Resources`.
> - Libérala en cuanto termines: no dejes sesiones interactivas abiertas con la GPU reservada.
> - Para verificar que tu código **sí** está usando la GPU, revisa la salida de `nvidia-smi` en el `.out` o, con el trabajo corriendo, entra al nodo con `srun --jobid=<JOBID> --overlap --pty nvidia-smi`.

Basada en el [[Ejemplos de script usando sbatch de Slurm#Ejemplo de un trabajo especializado (GPU)|ejemplo GPU original]] · Volver a [[Plantillas de Slurm]].
