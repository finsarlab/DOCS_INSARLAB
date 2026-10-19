---
title: "Ejemplo de flujo de trabajo completo: de la necesidad al resultado"
tags:
  - slurm
  - ejemplo
---

> [!summary]
> Un caso realista contado de principio a fin. Una compañera tiene un script de procesamiento y quiere correrlo en el clúster. Verás **cómo decidir los recursos**, cómo elegir entre **dejar que Slurm escoja el nodo**, **fijar un nodo específico** o **pedir varios nodos**, y cómo revisar el resultado para afinar la siguiente corrida.

```mermaid
flowchart LR
    A[¿Qué necesita<br/>mi programa?] --> B[sinfo:<br/>¿qué hay libre?]
    B --> C[Elegir grupo<br/>de nodos]
    C --> D[Copiar plantilla<br/>y ajustar directivas]
    D --> E[Prueba chica]
    E --> F[sbatch]
    F --> G[squeue /<br/>scontrol]
    G --> H[Leer .out / .err]
    H --> I[sacct:<br/>¿pedí bien?]
    I -.ajustar.-> D
```

## El caso

**Ana** tiene `procesar_ifg.py`, que procesa un conjunto de interferogramas de una zona volcánica. En su laptop sabe que:
- Usa NumPy y `multiprocessing` y escala bien hasta **~16 núcleos**.
- Con el conjunto completo, la RAM llega a unos **40 GB**.
- Tarda unas **4 horas**.
- No usa GPU.

## Paso 1 — Traducir la necesidad a directivas

| Necesidad | Directiva | Razonamiento |
|---|---|---|
| Un programa con 16 hilos | `--ntasks=1 --cpus-per-task=16` | Es **un** proceso multihilo, no MPI ([[Archivo de trabajo y directivas SBATCH\|¿por qué?]]) |
| ~40 GB de RAM | `--mem=48G` | Lo estimado más ~20% de margen |
| ~4 horas | `--time=06:00:00` | Lo estimado más ~50% de margen la primera vez |
| Sin GPU | `--exclude=node2` | Deja libre la GPU |

48 GB descartan a `node3` (~31 GB). Quedan `node1` y `node4`, el [[Grupos de nodos y particiones#Los grupos de nodos|Grupo A]].

## Paso 2 — Ver el clúster

```bash
[ana@master ~]$ sinfo -N -o "%10N %10P %15C %10m %T"
NODELIST   PARTITION  CPUS(A/I/O/T)   MEMORY     STATE
master     all*       0/40/0/40       126000     idle
node1      all*       56/0/0/56       257000     allocated
node2      extra      0/56/0/56       240000     idle
node2      n2gpu      0/56/0/56       240000     idle
node3      extra      4/20/0/24       31000      mixed
node4      extra      8/48/0/56       257000     mixed
```
`node1` está lleno, pero `node4` tiene 48 núcleos libres. Más en [[Comando sinfo]].

## Paso 3 — Preparar la carpeta

```bash
mkdir -p ~/proyectos/volcan/logs
cd ~/proyectos/volcan
# aquí ya están procesar_ifg.py y la carpeta datos/
```

## Paso 4 — Escribir el archivo de trabajo

Ana tiene **tres opciones** para decir *dónde* correr. Las tres comparten el resto de la cabecera.

### Opción A — Dejar que Slurm elija entre los nodos grandes (recomendada)

`trabajo.sh`, basado en la [[Plantilla - Grupo de nodos de alta capacidad]]:
```bash
#!/bin/bash
#SBATCH --job-name=volcan_ifg
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=all,extra
#SBATCH --exclude=master,node2,node3
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=48G
#SBATCH --time=06:00:00

set -euo pipefail
echo "Job $SLURM_JOB_ID | $(hostname) | $SLURM_JOB_PARTITION | inicio: $(date)"
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

python3 procesar_ifg.py --entrada datos/ --salida resultados/ --workers "$SLURM_CPUS_PER_TASK"

echo "Fin: $(date)"
```
Arranca en `node1` o `node4`, el que tenga espacio primero. Hoy será `node4`.

### Opción B — Un nodo específico

Ana necesita comparar con una corrida anterior que se hizo en `node4` y quiere el mismo equipo. Cambia las dos líneas de "dónde":
```bash
#SBATCH --partition=extra          # node4 pertenece a extra
#SBATCH --nodelist=node4
```
y quita `--exclude`. Ventaja: es reproducible. Costo: si `node4` se llena, espera aunque `node1` esté libre. Ver [[Plantilla - Nodo especifico]].

### Opción C — Varios nodos a la vez (`--nodelist` con lista)

Solo si el programa **se reparte entre máquinas** (MPI, o lanzar copias con `srun`). Ejemplo: procesar dos zonas a la vez, una por nodo:
```bash
#!/bin/bash
#SBATCH --job-name=dos_zonas
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=extra
#SBATCH --nodes=2
#SBATCH --nodelist=node[2,4]       # pide AMBOS nodos (no "uno de los dos")
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=48G                  # por nodo
#SBATCH --time=06:00:00

set -euo pipefail
srun hostname                      # imprime node2 y node4: hay un proceso por nodo
srun bash -c 'python3 procesar_ifg.py --zona "zona_${SLURM_PROCID}" --workers "$SLURM_CPUS_PER_TASK"'
```
`srun` lanza **una copia por tarea** y cada una recibe su `$SLURM_PROCID` (0, 1, …). Sin `srun`, el script correría solo en el primer nodo y el segundo quedaría reservado sin usar.

> [!warning] Los nodos de un trabajo deben estar en la misma partición
> `node1` (en `all`) y `node4` (en `extra`) **no** pueden ir juntos en un mismo trabajo multinodo.

## Paso 5 — Prueba chica antes de la grande

Antes de pedir 6 horas, Ana prueba con 2 interferogramas en el nodo ligero, **sobrescribiendo recursos desde la terminal** sin tocar el archivo. Como lo de la terminal reemplaza a lo del archivo, `--exclude=node2` sustituye a la lista original y deja pasar a `node3`:
```bash
sbatch --partition=extra --nodelist=node3 --exclude=node2 --cpus-per-task=4 --mem=8G --time=00:20:00 \
       trabajo.sh
```
> Si su script aceptara un argumento como `--limite 2`, lo agregaría a la línea `python3` o se lo pasaría como argumento a `sbatch` (ver [[Comando sbatch#Las 4 formas en que lo usarás|argumentos]]).

Cuando la prueba termina bien (`COMPLETED`), sigue con la corrida grande. Antes valida que la petición sea correcta:
```bash
sbatch --test-only trabajo.sh
# sbatch: Job 10620 to start at 2026-10-09T12:41:03 using 16 processors on nodes node4 in partition extra
```

## Paso 6 — Encolar y seguir

```bash
$ sbatch trabajo.sh
Submitted batch job 10621

$ squeue -u $USER
JOBID PARTITION       NAME  USER ST   TIME NODES NODELIST(REASON)
10621     extra volcan_ifg   ana  R   0:42     1 node4
```
¿Usa de verdad los 16 núcleos?
```bash
srun --jobid=10621 --overlap --pty htop
```
¿Va avanzando?
```bash
tail -f logs/volcan_ifg_10621.out
```

Si hubiera quedado en `PD`, la última columna diría por qué (ver [[Codigos de razon de un Job (Job Reason Codes)]]).

## Paso 7 — Revisar el resultado y afinar

```bash
$ sacct -j 10621 --units=G --format=JobID,NodeList,AllocCPUS,ReqMem,MaxRSS,Elapsed,Timelimit,State,ExitCode
JobID         NodeList  AllocCPUS  ReqMem   MaxRSS    Elapsed  Timelimit      State ExitCode
------------ --------- ---------- ------- -------- ---------- ---------- ---------- --------
10621            node4         16     48G            03:51:10   06:00:00  COMPLETED      0:0
10621.batch      node4         16             38.7G  03:51:10             COMPLETED      0:0
```
Conclusiones de Ana para la siguiente vez:
- Usó 38.7 GB de 48 GB: el margen es correcto.
- Tardó 3:51. La próxima vez puede pedir `--time=05:00:00`.
- Lo anota en un comentario en la cabecera de su `trabajo.sh` para el siguiente compañero.

## ¿Y si algo falla?

| `State` en `sacct` | Qué hace Ana |
|---|---|
| `OUT_OF_MEMORY` | Sube `--mem` (si estaba en `node3`, cambia de grupo de nodos) |
| `TIMEOUT` | Sube `--time` |
| `FAILED` | Lee `logs/volcan_ifg_<JOBID>.err` |
| `NODE_FAIL` | No fue su culpa: vuelve a encolar y avisa al admin |
| `CANCELLED` | Alguien (ella o el admin) lo canceló |

## Variante: los tres pasos de un flujo InSAR encadenados

Si su flujo tiene etapas (preparar → procesar → series de tiempo), cada una con distintos recursos, se encadenan con dependencias. Ver [[Plantilla - Cadena de trabajos (dependencias)]].

---
Volver a [[Manejo de Slurm]] · [[Plantillas de Slurm]]
