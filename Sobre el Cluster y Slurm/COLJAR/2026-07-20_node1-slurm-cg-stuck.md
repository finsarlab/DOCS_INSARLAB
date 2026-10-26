# Nodo1 atascado en estado COMPLETING bloquea workers de MintPy/dask

| Campo | Valor |
|---|---|
| Fecha | 2026-07-20 |
| Directorio | /raid/doki/workdir/Amistad_SenDT12_201301_202607 |
| Repo / rama | rsmas_insar / main |
| Entorno | Skynet |
| Componentes | Slurm, MiaplPy (run_02_unpack_secondary_slc), MintPy (run_08_miaplpy_invert_network, dask/mintpy_bee) |

## Objetivo
Diagnosticar por qué node1 aparecía inactivo en btop/squeue durante el paso de inversión de red (L1/L2) de MintPy del proyecto Amistad_SenDT12_201301_202607.

## Problema y causa raíz
- Síntoma: 210 jobs `mintpy_bee` (workers dask) en PENDING; solo 11 corriendo, todos en `master`; node1 mostraba CPULoad ~166-168 con CPUAlloc=0.
- Causa: 56 jobs `run_02_unpack_secondary_slc_*` lanzados la noche anterior en node1 llegaron a TIMEOUT (walltime 6:10:00) y quedaron en estado CG (completing) sin limpiarse, dejando node1 en `IDLE+COMPLETING+PLANNED` durante 14+ horas. Mientras un nodo está en COMPLETING con jobs zombie, Slurm no le asigna nuevos jobs (partición `all` = master + node1), de ahí que los 210 workers quedaran en PENDING.
- Nota de corrección del usuario: node1 no estaba down ni con el montaje NFS colgado; simplemente no es accesible por ssh para este usuario. El diagnóstico inicial de "mount NFS colgado" fue descartado.
- minsar ya había recuperado los jobs de unpack automáticamente vía `rerun.log` (reintento con walltime doblado 6:10 → 13:33), completando en segundos en `master`; no hubo pérdida de datos, solo cuello de botella de scheduling.

## Cambios realizados
### Archivos modificados/creados
- Ninguno en el repo. Sesión de diagnóstico de cluster, sin cambios de código.

### Comandos relevantes
```bash
# Diagnóstico (solo lectura)
squeue -u $USER -o "%.10i %.20j %.10P %.2t %.10M %.6D %.4C %R"
sinfo -N -l
scontrol show node node1
sacct -u $USER -S now-12hours --format=JobID,JobName%30,State,Elapsed,Timelimit,ExitCode
sacct -j <jobid> --format=JobID,Start,End,State

# Fix propuesto para liberar node1 (bloqueado por el clasificador de permisos
# de Claude Code; el usuario debe ejecutarlo manualmente)
scontrol update nodename=node1 state=down reason="clear stuck CG jobs"
scontrol update nodename=node1 state=resume
```

## Pendiente / seguimiento
- Ejecutar manualmente el ciclo `state=down` → `state=resume` en node1 para purgar los jobs CG y permitir que los 210 workers `mintpy_bee` se programen en node1.
- Revisar en node1 (requiere acceso de quien sí tenga ssh) procesos huérfanos de `SentinelWrapper.py`/python que puedan quedar tras el purge, antes de confiar la carga de nuevos workers al nodo.
- Investigar por qué los jobs `run_02_unpack_secondary_slc` tardaron 6h+ (timeout) en node1 vs 18-33s en master con el mismo código/datos — causa aún no identificada tras descartar NFS colgado.
- node3 figura `down*` (Not responding) — pendiente de revisión aparte.
- No tocar node2 ni node4 (indicación explícita del usuario, no deben usarse).
