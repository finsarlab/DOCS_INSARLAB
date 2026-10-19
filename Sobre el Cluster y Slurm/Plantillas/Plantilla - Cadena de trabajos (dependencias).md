---
title: "Plantilla: cadena de trabajos (dependencias)"
tags:
  - slurm
  - plantilla
---

**Cuándo usarla:** flujos de varios pasos donde **cada paso necesita lo que produjo el anterior** (descarga → procesamiento → serie de tiempo → exportación). Cada paso puede pedir recursos distintos: la descarga necesita poco, el procesamiento mucho.

Necesitas un archivo de trabajo por paso (`paso1.sh`, `paso2.sh`, …, armados con cualquier otra plantilla) y un pequeño script que los encadene:

```bash
#!/bin/bash
# encadenar.sh — se ejecuta en el master, NO con sbatch:  bash encadenar.sh
set -euo pipefail
mkdir -p logs

J1=$(sbatch --parsable paso1_descarga.sh)
echo "Paso 1: $J1"

J2=$(sbatch --parsable --dependency=afterok:$J1 paso2_procesado.sh)
echo "Paso 2: $J2 (espera a $J1)"

J3=$(sbatch --parsable --dependency=afterok:$J2 paso3_series.sh)
echo "Paso 3: $J3 (espera a $J2)"
```

- `--parsable` hace que `sbatch` devuelva solo el número del Job ID, para guardarlo en una variable.
- `afterok:ID` → arranca solo si el trabajo `ID` terminó **bien** (`COMPLETED`).

| Tipo de dependencia | Arranca cuando el anterior… |
|---|---|
| `afterok:ID` | terminó con éxito |
| `afternotok:ID` | falló (útil para un paso de limpieza o aviso) |
| `afterany:ID` | terminó, sin importar cómo |
| `after:ID` | empezó a correr |

> [!warning] Si un paso falla, los siguientes se quedan esperando para siempre
> En `squeue` aparecerán como `PD` con razón `DependencyNeverSatisfied`. Cancélalos con `scancel` (ver [[Codigos de razon de un Job (Job Reason Codes)]]).

Volver a [[Plantillas de Slurm]].
