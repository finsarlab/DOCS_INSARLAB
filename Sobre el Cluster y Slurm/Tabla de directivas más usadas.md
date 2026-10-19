---
title: Tabla de Directivas Principales de `sbatch`
tags:
  - slurm
  - directivas
---
El comando `sbatch` cuenta con una amplia variedad de opciones para gestionar con precisión cómo, cuándo y dónde se ejecutará tu código. Estas directivas se colocan al inicio de tu script de bash, siempre precedidas por la etiqueta `#SBATCH` (ver [[Archivo de trabajo y directivas SBATCH|reglas para escribirlas]]).

A continuación, se presenta una tabla de referencia con las directivas más utilizadas en el entorno de cluster computing:

## Identificación y registros

|**Directiva**|**Corta**|**Descripción**|**Ejemplo de uso**|
|---|---|---|---|
|**`--job-name=<nombre>`**|`-J`|Asigna un nombre personalizado al trabajo. Esto facilita su identificación al consultar la cola con `squeue`.|`#SBATCH --job-name=modelo_clima`|
|**`--output=<archivo>`**|`-o`|Ruta y nombre del archivo de salida estándar (stdout). Acepta comodines como `%j` (Job ID) y `%x` (nombre). **La carpeta debe existir.**|`#SBATCH --output=logs/%x_%j.out`|
|**`--error=<archivo>`**|`-e`|Ruta y nombre del archivo donde se registran exclusivamente los mensajes de error (stderr).|`#SBATCH --error=logs/%x_%j.err`|
|**`--chdir=<ruta>`**|`-D`|Carpeta en la que se ejecuta el script (por defecto, desde donde corriste `sbatch`).|`#SBATCH --chdir=/mnt/beegfs/insarlab/falcon/proy`|

## Dónde se ejecuta

|**Directiva**|**Corta**|**Descripción**|**Ejemplo de uso**|
|---|---|---|---|
|**`--partition=<nombre>`**|`-p`|Cola lógica a la que se envía el trabajo: `all`, `extra` o `n2gpu`. Admite varias separadas por coma; Slurm usa la que tenga espacio primero.|`#SBATCH --partition=extra`|
|**`--nodelist=<nodos>`**|`-w`|Obliga a ejecutar el trabajo en los nodos listados. ⚠️ Si listas **varios**, el trabajo los recibe **todos a la vez**, no "uno de ellos".|`#SBATCH --nodelist=node4`|
|**`--exclude=<nodos>`**|`-x`|Funciona al revés que `--nodelist`: indica los nodos en los que **no** debe ejecutarse el trabajo.|`#SBATCH --exclude=node2`|
|**`--nodes=<cantidad>`**|`-N`|Número de nodos físicos distintos que necesita el trabajo (solo para programas distribuidos o MPI).|`#SBATCH --nodes=2`|
|**`--exclusive`**| |Reserva el nodo completo para ti, aunque no uses todos sus recursos. Úsalo con moderación.|`#SBATCH --exclusive`|

## Cuántos recursos

|**Directiva**|**Corta**|**Descripción**|**Ejemplo de uso**|
|---|---|---|---|
|**`--ntasks=<cantidad>`**|`-n`|Número total de tareas (**procesos independientes**). Es para MPI o para lanzar varias copias con `srun`. Para un solo programa multihilo usa `1`.|`#SBATCH --ntasks=1`|
|**`--ntasks-per-node=<n>`**| |Tareas por nodo cuando usas varios nodos.|`#SBATCH --ntasks-per-node=1`|
|**`--cpus-per-task=<cantidad>`**|`-c`|Núcleos (CPUs) para cada tarea. **Es lo que normalmente quieres** para Python, ISCE2, MintPy, OpenMP o `multiprocessing`.|`#SBATCH --cpus-per-task=16`|
|**`--mem=<tamaño>`**| |Memoria RAM máxima **por nodo**, con sufijo `M` o `G`. `0` = toda la del nodo.|`#SBATCH --mem=64G`|
|**`--mem-per-cpu=<tamaño>`**| |Memoria por núcleo. Alternativa a `--mem` (no uses las dos).|`#SBATCH --mem-per-cpu=4G`|
|**`--gres=<recursos>`**| |Recursos genéricos del nodo. En el lab sirve para pedir la GPU L4 (obligatoria en `n2gpu`).|`#SBATCH --gres=gpu:l4:1`|
|**`--time=<tiempo>`**|`-t`|Tiempo máximo de ejecución; al excederlo, Slurm detiene el trabajo. Formatos: `minutos`, `horas:minutos:segundos`, `días-horas:minutos:segundos`.|`#SBATCH --time=12:30:00` _(12 horas y media)_|

## Flujos de trabajo

|**Directiva**|**Corta**|**Descripción**|**Ejemplo de uso**|
|---|---|---|---|
|**`--array=<rango>`**|`-a`|Crea un arreglo de subtrabajos idénticos; cada uno recibe `$SLURM_ARRAY_TASK_ID`. `%N` limita cuántos corren a la vez.|`#SBATCH --array=1-40%4`|
|**`--dependency=<tipo:ID>`**|`-d`|Espera a que otro trabajo cumpla una condición (`afterok`, `afterany`, `afternotok`).|`#SBATCH --dependency=afterok:10500`|
|**`--begin=<hora>`**| |No arranca antes de esa hora.|`#SBATCH --begin=22:00`|
|**`--mail-type=<eventos>`**| |Alertas por correo ante cambios de estado: `BEGIN`, `END`, `FAIL` o `ALL`. ⚠️ Solo funciona si el clúster tiene correo configurado (por confirmar).|`#SBATCH --mail-type=END,FAIL`|
|**`--mail-user=<correo>`**| |Dirección que recibe las alertas de `--mail-type`.|`#SBATCH --mail-user=usuario@dominio.com`|

> [!info]
> Ejemplos de uso combinado en las [[Plantillas de Slurm]]. Lista completa en la [documentación oficial de `sbatch`](https://slurm.schedmd.com/sbatch.html#SECTION_OPTIONS).
