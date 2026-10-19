---
title: Estados de un Job (Job State Codes)
tags:
  - slurm
---

| **Código Completo** | **Abreviatura** | **Significado**                                                                                                                            |
| ------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **PENDING**         | `PD`            | El trabajo está en la cola, esperando a que los recursos solicitados se liberen o que se cumpla su nivel de prioridad.                     |
| **RUNNING**         | `R`             | El trabajo tiene los recursos asignados y se está ejecutando actualmente en el nodo de cómputo.                                            |
| **COMPLETING**      | `CG`            | El trabajo terminó y Slurm está limpiando sus procesos en el nodo. Suele durar segundos.                                                   |
| **COMPLETED**       | `CD`            | El trabajo terminó exitosamente. Todos los comandos del script se ejecutaron sin devolver errores (Exit Code 0).                           |
| **FAILED**          | `F`             | El trabajo terminó de forma prematura porque ocurrió un error en el código ejecutado (Exit Code distinto de 0).                            |
| **CANCELLED**       | `CA`            | El trabajo fue abortado manualmente por el usuario (usando `scancel`) o por el administrador del sistema.                                  |
| **TIMEOUT**         | `TO`            | El trabajo superó el límite de tiempo (`--time`) que solicitaste en las directivas de `sbatch` y fue asesinado por Slurm.                  |
| **OUT_OF_MEMORY**   | `OOM`           | El trabajo intentó consumir más memoria RAM de la que le solicitaste a Slurm (con `--mem`), por lo que fue detenido para proteger el nodo. |
| **NODE_FAIL**       | `NF`            | El nodo de cómputo experimentó un fallo de hardware o pérdida de red mientras tu trabajo se estaba ejecutando.                             |
>Si algún código de estado no se encuentra documentado por nuestra parte, puedes consultar la [documentación oficial](https://slurm.schedmd.com/job_state_codes.html)