---
title: "Plantilla: grupo de nodos de alta capacidad (node1 o node4)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** procesos pesados en CPU y RAM (hasta 56 núcleos y ~250 GB) donde **no te importa si corre en `node1` o en `node4`**, solo quieres que empiece lo antes posible.

Truco: `node1` está en la partición `all` y `node4` en `extra`. Se piden **ambas particiones** y se excluyen los nodos que no queremos. Slurm arranca el trabajo en la que primero tenga espacio.

```bash
#!/bin/bash
#SBATCH --job-name=pesado
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=all,extra                 # ambas particiones…
#SBATCH --exclude=master,node2,node3          # …pero solo node1 o node4
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=48
#SBATCH --mem=200G
#SBATCH --time=2-00:00:00                     # 2 días

set -euo pipefail
echo "Job $SLURM_JOB_ID | nodo: $(hostname) | partición: $SLURM_JOB_PARTITION | inicio: $(date)"
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

python3 procesar_stack.py

echo "Fin: $(date)"
```

**Ajusta:** `--cpus-per-task` (≤ 56), `--mem` (≤ `RealMemory` del nodo, consúltala con `sinfo -N -o "%N %c %m"`) y `--time`.

> [!note] ¿Y `node2`?
> Tiene la misma capacidad, pero es el único con GPU. Lo excluimos por cortesía. Si el clúster está vacío y tienes prisa, puedes quitarlo de `--exclude`.

Volver a [[Plantillas de Slurm]] · Ver [[Grupos de nodos y particiones]].
