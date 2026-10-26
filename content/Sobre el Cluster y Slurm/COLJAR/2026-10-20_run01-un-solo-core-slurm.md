# Diagnóstico: run_01_unpack_topo_reference confinado a 1 core en Skynet

| Campo | Valor |
|---|---|
| Fecha | 2026-10-20 |
| Directorio | /raid/doki/workdir/PresaJales_SenDT27_202603_202606 |
| Repo / rama | rsmas_insar / main (HEAD 407ff73, con cambios locales sin commitear) |
| Entorno | Skynet |
| Componentes | ISCE2 (topsStack), Slurm, minsar/job_submission.py |

## Objetivo
Diagnosticar por qué `run_01_unpack_topo_reference` corre lento aunque se incrementen los cores del nodo (40 cores / 128 GB RAM disponibles).

## Problema y causa raíz
- Síntoma: step1 no acelera al añadir cores al nodo; mismo comportamiento observado en pasos posteriores (run_05) corriendo en paralelo pero cada tarea limitada a 2 CPUs.
- Causa: Slurm en Skynet usa `TaskPlugin=task/cgroup,task/affinity` + `SelectType=select/cons_tres` (`CR_CORE_MEMORY`) → confina estrictamente cada job a los cores solicitados vía `#SBATCH -n`.
- `job_submission.py` (rama `scheduler == 'SLURM'`, bloque `### HARDCODED ###`, líneas ~644-656) fuerza `number_of_nodes=1` y `number_of_tasks=1` sin emitir `-c`/`--cpus-per-task`, y el esquema `singleTask` (usado en Skynet, `JOB_SUBMISSION_SCHEME=singleTask`) tampoco exporta `OMP_NUM_THREADS` — ese export solo ocurre en las ramas launcher/multiTask (líneas 1250 y 1285).
- `configs/config_reference` pide `numProcess: 8` para `topo`, pero al estar el job confinado a 1 core, los 8 procesos internos compiten por time-slicing en un solo core (más lento que serial).
- `job_defaults.cfg` define `num_threads=8` para `unpack_topo_reference`, pero ese valor nunca se traduce a `-c`/`OMP_NUM_THREADS` en jobs SLURM singleTask.
- Confirmado con `Cpus_allowed_list` de procesos en vivo (run_05, jobs 115167-115181): cada tarea mostraba solo 2 CPUs asignadas pese a `num_threads=4` en `job_defaults.cfg` para ese step.
- Hallazgo secundario: `queues.cfg` declara `CPUS_PER_NODE=28` para `skynet`, pero los nodos reales (`sinfo -N`) tienen 40-56 CPUs (master=40, node1/node2/node4=56, node3=24).
- Hallazgo secundario: los jobs cayeron en el nodo `master` (~19 GB libres al momento del check) mientras node1/2/4 (56 CPUs, 240 GB) estaban mayormente libres; los jobs SLURM no solicitan memoria (`memory_option = False` en rama SLURM).

## Cambios realizados
Ninguno aplicado en esta sesión — fue una sesión de diagnóstico. Fix propuesto pendiente de aprobación del usuario:
- `minsar/job_submission.py`, rama `scheduler == 'SLURM'` en `get_job_file_lines`: agregar `#SBATCH -c {N}` usando `self.default_num_threads` (ya cargado desde `job_defaults.cfg` por step).
- `minsar/job_submission.py`, `write_single_job_file`: exportar `OMP_NUM_THREADS={N}` también para el esquema `singleTask`.
- Opcional: corregir `CPUS_PER_NODE` en `minsar/defaults/queues.cfg` para `skynet` (28 → valor real por nodo, o manejar heterogeneidad de nodos).
- Opcional: subir `num_threads` de `unpack_topo_reference` en `job_defaults.cfg` (corre sola en el nodo, puede usar más de 8).
