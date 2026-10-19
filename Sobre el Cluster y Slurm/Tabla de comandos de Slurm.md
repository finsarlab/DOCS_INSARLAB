---
title: Tabla de comandos de Slurm
tags:
  - slurm
  - comandos
---

| **Comando** | **Descripción funcional** | **Ejemplo de ejecución** | **A detalle** |
| --- | --- | --- | --- |
| **`sbatch`** | Encola un script de trabajo (con las directivas de recursos y los comandos a ejecutar) para que Slurm lo procese de manera automatizada y en segundo plano. | `sbatch prueba.sh` | [[Comando sbatch]] |
| **`squeue`** | Muestra el estado actual de la cola: qué trabajos están pendientes (`PD`) o en ejecución (`R`) y en qué nodos. | `squeue -u $USER` | [[Comando squeue]] |
| **`scancel`** | Cancela un trabajo: lo saca de la cola si estaba pendiente, o lo detiene y libera los recursos si ya corría. | `scancel 10500` | [[Comando scancel y scontrol]] |
| **`sinfo`** | Reporta el estado, la configuración y la disponibilidad del hardware: qué nodos están libres (`idle`), ocupados (`alloc`) o caídos (`down`). | `sinfo -N -o "%N %C %m %T"` | [[Comando sinfo]] |
| **`sacct`** | Consulta el historial: estado final, duración, memoria usada y códigos de salida de trabajos que ya terminaron. | `sacct -j 10500 --format=JobID,State,MaxRSS` | [[Comando sacct]] |
| **`scontrol`** | Muestra todos los detalles de un trabajo, nodo o partición. También pausa o modifica trabajos pendientes. | `scontrol show job 10500` | [[Comando scancel y scontrol]] |
| **`salloc`** | Reserva recursos (núcleos, memoria, GPU) en tiempo real para pruebas interactivas, sin script. | `salloc --nodes=1 --partition=extra` | [[Manejo de Slurm#`salloc` (Allocation)\|Manejo de Slurm]] |
| **`srun`** | Lanza procesos dentro de recursos asignados. También abre una terminal en un nodo de cómputo. | `srun --pty /bin/bash` | [[Manejo de Slurm#`srun` (Run)\|Manejo de Slurm]] |
