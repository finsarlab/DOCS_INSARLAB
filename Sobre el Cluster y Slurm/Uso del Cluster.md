---
title: Uso del Clúster
---
> [!important]
> Para poder utilizar el Cluster, [[Creacion de Usuario|necesitas un usuario]]

Una vez que has configurado exitosamente tu acceso SSH y te has conectado, notarás que el prompt de tu terminal cambia para indicarte tu ubicación dentro de la red del laboratorio. Retomando nuestro ejemplo anterior:
![[Conexion al cluster#^a045be]]
la línea de comandos se verá de la siguiente manera:

```
[falcon@master ~]$
```

Esta sencilla línea nos indica que **te encuentras dentro del nodo maestro**.

Como se mencionó en secciones anteriores[^1], el nodo maestro es exclusivamente el director de orquesta. Es muy común, especialmente cuando se viene de un entorno de desarrollo en una computadora personal, tener el impulso de ejecutar inmediatamente comandos como `python script.py` o compilar código pesado directamente en la terminal. Sin embargo, en un entorno de [[Cluster Computing|cluster computing]], *ejecutar procesos pesados en el nodo maestro* **está estrictamente prohibido**, ya que saturaría el sistema que coordina a los demás usuarios.

Para utilizar la verdadera potencia del **InSAR lab**, debemos cambiar nuestro enfoque de ejecución: 
> [!important]
> Pasar de "*correr comandos en la terminal*" a "==encolar trabajos en el clúster==".

## El concepto de Trabajo (Job) y el script que le indica a Slurm como procesarlo

En lugar de interactuar directamente con el hardware, empaquetaremos nuestras instrucciones en lo que llamaremos un **trabajo** o **job**. **Un trabajo no es más que un archivo de texto** (comúnmente un script de ``bash`` con extensión `.sh`) 

>Ejemplo de nombre del archivo situado en [[Logica de trabajo en insarlab#Sobre el almacenamiento|tu carpeta personal]]
```bash
~/prueba.sh
```

que contiene dos partes fundamentales:
1. [[#Las directivas de recursos]]
2. [[#Los comandos de ejecución]]
### Las directivas de recursos
Las instrucciones explícitas para el planificador indicando qué partición usar, cuántos núcleos se necesitan, si se requiere una GPU y dónde guardar los resultados. 
Las directivas se indican con ``#SBATCH`` al principio de la línea y normalmente hasta arriba del script

>*Ejemplos de directivas*
```
#SBATCH --job-name=prueba
#SBATCH --output=resultado_%j.out
#SBATCH --error=error_%j.err
```

>Puedes revisar [[Tabla de directivas más usadas|las directivas más usadas]]
>O en su defecto, revisar [la totalidad de las directivas en la documentación oficial de slurm](https://slurm.schedmd.com/sbatch.html#SECTION_OPTIONS)
### Los comandos de ejecución 
Los pasos exactos que el nodo de cómputo deberá seguir una vez que los recursos le sean asignados (por ejemplo, cargar un entorno virtual, moverse a un directorio específico y ejecutar un script de Python). 
Estos comandos no son necesariamente de slurm, sino que son del software que usarás para lo que necesites, naturalmente puedes ocupar comandos de bash. Se ponen después de las directivas de [[Que es Slurm|slurm]]

Ejemplo:
```bash
PROJECT_DIR="/mnt/beegfs/insarlab/fersa/prueba_gpu_l4"

# Evitar problemas de caché con Pixi y HuggingFace en red
export PIXI_CACHE_DIR=/tmp/pixi-cache-fersa
export HF_HOME="/mnt/beegfs/insarlab/fersa/.cache/huggingface"

cd "$PROJECT_DIR" || exit 1

echo "Iniciando proceso en el nodo: $(hostname)"

# Lanzar el modelo pesado
pixi run python inferencia.py
```

### Forma final del script
De tal forma que la forma general de un script para subir un [[#El concepto de Trabajo (Job) y el script que le indica a Slurm como procesarlo|trabajo]] se ve así:
![[Forma general de un script sh para mandar un trabajo usando sbatch de slurm]]

> [!info]
> Para ver más ejemplos de scripts completos armados con ambas partes, recurrir a [[Ejemplos de script usando sbatch de Slurm]]

Una vez que construyes este script, [[Manejo de Slurm#`sbatch`|se lo entregas a Slurm]]. A partir de ese momento, puedes cerrar tu terminal o apagar tu computadora; el clúster se encargará de formar tu trabajo en la cola, asignarle el nodo de cómputo adecuado (como `node1` o `node2`) y [[Manejo de Slurm#Archivos de Salida y Error (``.out`` y ``.err``)|guardar un registro de todo lo que tu programa imprima en pantalla o los errores que genere.]]

### Job ID
Al momento de entregarle el trabajo a slurm, este le asigna un JID (Job ID), que es un dígito que usa Slurm para poder identificar de manera única cada nuevo trabajo.
**Es importante tenerlo en mente y saber qué JID tiene un trabajo que subas, pues este sirve para ver su estado usando otros comandos de Slurm**


## La Dinámica de Colas y Asignación de Recursos

Una vez que entregas tu trabajo a Slurm mediante las directivas, este no se ejecuta de manera inmediata. En su lugar, entra a lo que llamamos la _dinámica de colas_. Puedes imaginar a Slurm como un administrador de recursos altamente eficiente que revisa constantemente dos cosas: 
- Qué es lo que pide tu trabajo.
- Qué partes del clúster están libres en ese momento.

El sistema mantendrá tu trabajo en un estado de espera (*pendiente*) hasta que encuentre que el clúster dispone exactamente de los recursos que indicaste en tus directivas. Para que esto funcione, es fundamental saber cómo pedir dichos recursos.

Por ejemplo, si necesitas una cantidad específica de poder de cómputo para un procesamiento en paralelo, tus directivas podrían verse así:

```SHELL
#SBATCH --ntasks=1          # Un programa…
#SBATCH --cpus-per-task=4   # …que usa 4 núcleos de procesamiento
#SBATCH --mem=32G           # Solicita 32 Gigabytes de memoria RAM
```

En este caso, Slurm buscará en la partición asignada algún nodo que tenga al menos 4 núcleos y 32 GB de memoria libres. Si no los hay, tu trabajo esperará en la cola.

> [!warning] `--ntasks` no es "número de núcleos"
> `--ntasks` es el número de **procesos independientes** (MPI). Para un programa de Python, ISCE2 o MintPy que usa varios hilos, lo correcto es `--ntasks=1 --cpus-per-task=N`. Ver [[Archivo de trabajo y directivas SBATCH]].

Por otro lado, habrá ocasiones en las que necesites que tu trabajo corra en un equipo muy particular, sin importar si otros están vacíos. Para indicarle a Slurm que se encole directamente hacia un nodo específico, usarías:

```shell
#SBATCH --nodelist=node4  # Fuerza a que el trabajo solo se ejecute en el nodo 4
```

> [!tip] ¿Qué nodo me conviene?
> Revisa [[Grupos de nodos y particiones]] para elegir partición y nodo según lo que necesita tu trabajo, y cuidado: `--nodelist` con **varios** nodos los pide **todos a la vez**.

### El sistema de prioridad de Slurm

Cuando hay muchos usuarios mandando trabajos al mismo tiempo y el clúster está lleno, Slurm debe decidir quién es el siguiente. Para ello, no utiliza un simple sistema de "el primero que llega, es el primero que se ejecuta" (FIFO), sino que calcula una **prioridad** matemática para cada tarea en la cola.

Esta prioridad sube o baja dependiendo de varios factores configurados en el sistema:

1. **Tiempo de espera (Age):** Entre más tiempo pase tu trabajo formado en la cola, mayor prioridad irá ganando.
2. **Tamaño del trabajo:** En algunas configuraciones, los trabajos que piden recursos pequeños pueden colarse en los "huecos" que dejan los trabajos más grandes, optimizando el uso del clúster.
3. **Uso justo (FairShare):** Slurm lleva un registro histórico. Si un usuario ha acaparado el clúster durante toda la semana, el sistema le dará menor prioridad a sus trabajos nuevos para permitir que otros miembros del InSAR lab, que no han usado el clúster recientemente, tengan la oportunidad de ejecutar sus proyectos.

> [!success]
> Una vez entiendas lo esencial del funcionamiento del cluster, puedes revisar el [[Manejo de Slurm]] o ir directo a las [[Plantillas de Slurm]].


[^1]: [[Cluster Computing#^DefinicionNodoMaestro|Definición de nodo maestro]]
