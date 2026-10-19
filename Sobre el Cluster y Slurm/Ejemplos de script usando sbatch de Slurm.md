### Ejemplo de un trabajo estándar (CPU)

A continuación, se muestra un script básico (`prueba.sh`) diseñado para ejecutarse en la partición general, ideal para tareas que requieren únicamente procesamiento tradicional.


```bash
#!/bin/bash

#SBATCH --job-name=prueba         # Nombre del trabajo para identificarlo en la cola
#SBATCH --output=resultado_%j.out # Archivo donde se guardará lo que el script imprima en pantalla (%j es el ID del trabajo)
#SBATCH --error=error_%j.err      # Archivo donde se guardarán los mensajes de error
#SBATCH --ntasks=1                # Número de tareas (procesos) a ejecutar
#SBATCH --partition=all           # Nombre de la partición a la cual enviar el trabajo

# --- Comandos a ejecutar ---
echo "Iniciando trabajo de Python en el nodo: $(hostname)"
echo "Fecha actual: $(date)"
echo "Directorio de trabajo: $(pwd)"

echo "Ejecutando proceso de prueba..."

python3 script.py
echo "--- Fin del trabajo ---"
```

Para enviar este trabajo a la cola, ejecutarías en la terminal del nodo maestro:
```bash
sbatch prueba.sh
```

### Ejemplo de un trabajo especializado (GPU)

Cuando tu trabajo requiere de computación con gpu, como lo pueden ser el uso de inteligencia artificial, modelos de lenguaje o simulaciones gráficas, debes ser más específico con tus directivas para acceder al `node2` y su acelerador gráfico.

El siguiente ejemplo (`gpu_test.sh`) muestra cómo el equipo del InSAR lab configura un trabajo para realizar inferencia con un modelo pesado de HuggingFace utilizando la GPU L4:


```bash
#!/bin/bash
#SBATCH --job-name=gpu_test
#SBATCH --partition=n2gpu         # Enviar a la partición exclusiva de GPU
#SBATCH --gres=gpu:l4:1           # Solicitud de Recurso Genérico: 1 GPU modelo L4
#SBATCH --output=/mnt/beegfs/insarlab/fersa/prueba_gpu_l4/resultadoURGENTE_%j.out
#SBATCH --error=/mnt/beegfs/insarlab/fersa/prueba_gpu_l4/errorURGENTE_%j.err

PROJECT_DIR="/mnt/beegfs/insarlab/fersa/prueba_gpu_l4"

# Evitar problemas de caché con Pixi y HuggingFace en red
export PIXI_CACHE_DIR=/tmp/pixi-cache-fersa
export HF_HOME="/mnt/beegfs/insarlab/fersa/.cache/huggingface"

cd "$PROJECT_DIR" || exit 1

echo "Iniciando proceso en el nodo: $(hostname)"

# Lanzar el modelo pesado
pixi run python inferencia.py
```

Para enviar este trabajo a la cola, ejecutarías en la terminal del nodo maestro:
```bash
sbatch gpu_test.sh
```

> [!info]
> Más casos (multihilo, nodo específico, arreglos, dependencias) en las [[Plantillas de Slurm]].
