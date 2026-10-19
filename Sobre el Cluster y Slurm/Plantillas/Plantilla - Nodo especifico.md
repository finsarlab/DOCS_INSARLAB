---
title: "Plantilla: nodo específico (--nodelist)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** cuando necesitas que el trabajo corra **en un nodo concreto**: para reproducir un resultado, depurar un error que solo pasa en un equipo o aprovechar una característica (por ejemplo, el enlace de 10 Gb de `node1`).

```bash
#!/bin/bash
#SBATCH --job-name=en_node4
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=extra            # ¡la partición debe contener al nodo!
#SBATCH --nodelist=node4
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=120G
#SBATCH --time=1-00:00:00            # 1 día

set -euo pipefail
echo "Job $SLURM_JOB_ID | nodo: $(hostname) | inicio: $(date)"
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

python3 procesar.py

echo "Fin: $(date)"
```

**Pareja nodo ↔ partición** (si no coinciden, `sbatch` da error):

| Nodo | `--partition` |
|---|---|
| `node1` | `all` |
| `node2` | `extra` (o `n2gpu` si usas GPU) |
| `node3` | `extra` |
| `node4` | `extra` |

> [!warning] El costo de fijar un nodo
> Si ese nodo está ocupado, tu trabajo espera aunque otros estén libres (razón `Resources` en [[Comando squeue|squeue]]). Si está caído o en mantenimiento, espera con la razón `ReqNodeNotAvail`. Revisa antes con:
> ```bash
> sinfo -n node4
> ```

> [!danger] Varios nodos en `--nodelist` = los quieres TODOS
> `--nodelist=node2,node4` pide **ambos nodos al mismo tiempo**. Para "cualquiera de estos" usa la [[Plantilla - Grupo de nodos de alta capacidad]].

Volver a [[Plantillas de Slurm]].
