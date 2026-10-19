---
title: "El archivo de trabajo y sus directivas #SBATCH"
tags:
  - slurm
  - sbatch
  - directivas
---

> [!summary] En una frase
> El **archivo de trabajo** (*job script*) es un `.sh` con dos partes: arriba las **directivas** `#SBATCH`, que dicen qué recursos pides, y abajo los **comandos**, que dicen qué hacer con ellos. Se lo entregas a Slurm con [[Comando sbatch|sbatch]].

![[Forma general de un script sh para mandar un trabajo usando sbatch de slurm]]

## Dónde vive el archivo

Crea tus archivos de trabajo dentro de tu carpeta personal (`~`, que en realidad es `/mnt/beegfs/insarlab/<tu_usuario>`). Esa carpeta está en [[Logica de trabajo en insarlab#Sobre el almacenamiento|almacenamiento compartido]]: **todos los nodos ven exactamente los mismos archivos**, así que el script, tus datos y tus resultados están disponibles en el nodo que Slurm elija.

Una organización sugerida:
```
~/
├── plantillas/            ← tus copias de las plantillas
└── proyectos/
    └── volcan_2026/
        ├── trabajo.sh     ← el archivo de trabajo
        ├── datos/
        └── logs/          ← aquí caen los .out y .err
```

## Reglas de oro de las directivas

> [!important] 1. Van hasta arriba, justo después del `#!/bin/bash`
> Slurm deja de leer directivas en cuanto encuentra **el primer comando real**. Cualquier `#SBATCH` que esté debajo de un `echo`, `cd` o `export` se **ignora sin avisar**.
> ```bash
> #!/bin/bash
> #SBATCH --job-name=bien      # ✅ se lee
> echo "hola"
> #SBATCH --mem=64G            # ❌ ignorado: ya hubo un comando antes
> ```

> [!important] 2. Se escriben exactamente como `#SBATCH`
> Sin espacio entre `#` y `SBATCH`, y en mayúsculas. `# SBATCH` o `#sbatch` se tratan como comentarios normales.
> **Truco:** para "apagar" una directiva sin borrarla, agrégale otro `#`:
> ```bash
> ##SBATCH --nodelist=node4    # desactivada
> ```

> [!important] 3. No se pueden usar variables de bash dentro de una directiva
> `#SBATCH --output=$HOME/logs/x.out` **no funciona**: Slurm no expande `$HOME`. Usa rutas relativas (a la carpeta desde donde ejecutas `sbatch`), rutas absolutas o los comodines de Slurm (`%j`, `%x`, `%u`, `%N`; ver [[Manejo de Slurm#Nombrar los archivos de salida y error usando variables de entorno|variables de nombre]]).

> [!danger] 4. La carpeta de los `.out` y `.err` debe existir ANTES de encolar
> Si pones `--output=logs/%x_%j.out` y la carpeta `logs/` no existe, el trabajo **falla al instante y no deja ningún archivo** que te diga por qué. Siempre haz antes:
> ```bash
> mkdir -p logs
> ```

> [!note] 5. Lo que escribes en la terminal le gana al script
> Si en el script dice `--mem=32G` pero ejecutas `sbatch --mem=64G trabajo.sh`, se usan **64G**. Esto permite tener una sola plantilla y ajustar recursos al encolar sin editar el archivo.
> Orden de prioridad: **línea de comandos** > variables `SBATCH_*` del entorno > directivas `#SBATCH` del archivo.

> [!note] 6. ¿Dónde se ejecuta el script?
> Por defecto, en **la misma carpeta desde la que ejecutaste `sbatch`** (Slurm la guarda en `$SLURM_SUBMIT_DIR`). Si quieres otra, usa `--chdir=/ruta` o haz `cd` dentro del script.

## Las directivas, agrupadas por la pregunta que responden

| Pregunta | Directivas | Ejemplo |
|---|---|---|
| ¿Cómo se llama y dónde dejo los registros? | `--job-name`, `--output`, `--error` | `--output=logs/%x_%j.out` |
| ¿En qué máquinas? | `--partition`, `--nodelist`, `--exclude`, `--nodes` | `--partition=extra --exclude=node2` |
| ¿Cuántos núcleos? | `--ntasks`, `--cpus-per-task` | `--ntasks=1 --cpus-per-task=16` |
| ¿Cuánta memoria? | `--mem` (por nodo) o `--mem-per-cpu` | `--mem=64G` |
| ¿Cuánto tiempo? | `--time` | `--time=1-12:00:00` |
| ¿Necesito GPU? | `--gres` | `--gres=gpu:l4:1` |
| ¿Muchas corridas iguales? | `--array` | `--array=1-20%4` |
| ¿Debe esperar a otro trabajo? | `--dependency` | `--dependency=afterok:10500` |

La lista completa está en la [[Tabla de directivas más usadas]].

### `--ntasks` vs `--cpus-per-task` (la confusión más común)

| Tu programa es… | Pide | Por qué |
|---|---|---|
| Un solo proceso que usa varios hilos (Python con NumPy, ISCE2, MintPy, GDAL, `multiprocessing`) | `--ntasks=1 --cpus-per-task=N` | Es **un** programa que usa N núcleos. |
| Un programa MPI (se lanza con `srun`/`mpirun`) | `--ntasks=N` | Son N procesos que se comunican entre sí. |
| Un script en serie (un núcleo) | `--ntasks=1` (o nada) | Es el valor por defecto. |

> [!warning]
> Si pides `--ntasks=8` y ejecutas `python script.py` sin `srun`, Slurm reserva 8 núcleos pero tu programa corre **una sola vez** en el primero. Desperdicias 7 núcleos que otro compañero pudo usar.

Cuando uses `--cpus-per-task`, dile a tus librerías cuántos hilos pueden usar:
```bash
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OPENBLAS_NUM_THREADS=$SLURM_CPUS_PER_TASK
export MKL_NUM_THREADS=$SLURM_CPUS_PER_TASK
```

### `--mem` vs `--mem-per-cpu`
- `--mem=64G`: 64 GB **por nodo**, sin importar cuántos núcleos pidas. Es la forma más sencilla y la recomendada.
- `--mem-per-cpu=4G`: 4 GB **por núcleo** (con 16 núcleos son 64 GB).
- No uses las dos a la vez.
- `--mem=0` significa "toda la memoria del nodo". Úsalo solo si de verdad la necesitas, porque bloquea el nodo para todos los demás.

## Variables que Slurm te da dentro del script

Mientras tu trabajo corre, Slurm define variables de entorno que puedes usar en los **comandos** (no en las directivas):

| Variable | Contiene |
|---|---|
| `$SLURM_JOB_ID` | El [[Uso del Cluster#Job ID\|Job ID]] |
| `$SLURM_JOB_NAME` | El nombre de `--job-name` |
| `$SLURM_SUBMIT_DIR` | Carpeta desde donde ejecutaste `sbatch` |
| `$SLURM_JOB_NODELIST` | Nodo(s) asignados (ej. `node4`) |
| `$SLURM_CPUS_PER_TASK` | Núcleos por tarea que pediste |
| `$SLURM_NTASKS` | Número de tareas |
| `$SLURM_MEM_PER_NODE` | Memoria por nodo en MB (si usaste `--mem`) |
| `$SLURM_ARRAY_TASK_ID` | Índice dentro de un [[Plantilla - Arreglo de trabajos (job array)\|arreglo de trabajos]] |
| `$CUDA_VISIBLE_DEVICES` | GPU asignada (solo con `--gres=gpu`) |

## Cabecera estándar del InSAR lab

Copia esta cabecera como punto de partida. Llena **siempre** las 7 directivas marcadas como obligatorias aunque Slurm no las exija: así cualquiera que lea tu script sabe qué recursos usa.

```bash
#!/bin/bash
# ==================== DIRECTIVAS ====================
#SBATCH --job-name=NOMBRE                 # [obligatoria] sin espacios
#SBATCH --output=logs/%x_%j.out           # [obligatoria] la carpeta logs/ debe existir
#SBATCH --error=logs/%x_%j.err            # [obligatoria]
#SBATCH --partition=extra                 # [obligatoria] all | extra | n2gpu
#SBATCH --ntasks=1                        # [obligatoria]
#SBATCH --cpus-per-task=8                 # [obligatoria] núcleos para tu programa
#SBATCH --mem=32G                         # [obligatoria] RAM por nodo
#SBATCH --time=04:00:00                   # recomendada
##SBATCH --nodelist=node4                 # opcional (quita un # para activarla)
##SBATCH --exclude=node2                  # opcional

# ==================== COMANDOS ======================
set -euo pipefail                         # detente ante el primer error
echo "Job $SLURM_JOB_ID en $(hostname) - inicio: $(date)"

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

# ... tus comandos aquí ...

echo "Fin: $(date)"
```

> [!tip] `set -euo pipefail`
> Hace que el script se detenga en cuanto un comando falla. Sin esto, si un paso intermedio falla, bash sigue con los siguientes, el trabajo puede terminar como `COMPLETED` aunque los resultados estén mal, y el error queda enterrado en el `.err`.

---
Siguiente: copia una de las [[Plantillas de Slurm]] o mira el [[Ejemplo de flujo de trabajo completo]].
