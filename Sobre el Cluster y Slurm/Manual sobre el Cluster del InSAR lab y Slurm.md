---
title: Manual sobre el Clúster del InSAR lab y Slurm
tags:
  - manual
  - slurm
---
> [!summary] Bienvenid@
> Te damos la bienvenida al manual de usuario sobre el [[Arquitectura del Cluster del InSAR lab|cluster]] del **InSAR lab** y el uso de [[Que es Slurm|slurm]]

El laboratorio **InSAR** cuenta con un [[Cluster Computing|cluster]], en donde se realiza gran cantidad de procesamiento computacional.

Con el fin de poder darle uso y configuración a dicho cluster, se propone el siguiente manual, el cuál le permitirá conocer lo básico para poder manejar esta pieza de hardware, empezando con cómo conectarse a él y todas las configuraciones correspondientes, cómo usarlo mediante el software [[Que es Slurm|slurm]], así como las cosas a tomar en cuenta al momento de querer usar el cluster para acciones de computo.

> [!tip] ⚡ ¿Quieres empezar ya?
> Si ya tienes usuario y llave, sigue la [[Guia rapida - Tu primer trabajo en Slurm]]: en 10 minutos habrás mandado tu primer trabajo. Después regresa aquí para entender el porqué de cada paso.

---

## Ruta de aprendizaje

### 1. Entender el contexto
- [[Cluster Computing]]: qué es un clúster, un nodo y un *job scheduler*.
- [[Que es Slurm]]: el planificador que usa el lab.
- [[Arquitectura del Cluster del InSAR lab]]: nuestros nodos, su hardware y particiones.
- [[Logica de trabajo en insarlab]]: tu computadora vs. el clúster y dónde viven tus archivos.

### 2. Obtener acceso
- [[Creacion de Usuario]]: configurar la llave que te da el administrador.
- [[Conexion al cluster]]: entrar con `ssh`.
- [[Protocolo SSH]]: cómo funciona, copiar archivos y errores comunes.

### 3. Usar el clúster
- [[Uso del Cluster]]: el concepto de *trabajo* y la dinámica de colas.
- [[Grupos de nodos y particiones]]: **a dónde mando mi trabajo** y cuánto puedo pedir.
- [[Archivo de trabajo y directivas SBATCH]]: cómo se escribe el `.sh` y sus directivas.
- [[Plantillas de Slurm]]: scripts listos para copiar ⭐ (la forma de trabajo del lab).
- [[Manejo de Slurm]]: los comandos y cómo interpretar estados, razones y códigos de salida.
- [[Ejemplo de flujo de trabajo completo]]: un caso real de principio a fin.

### 4. Comandos a detalle
[[Comando sbatch]]
[[Comando squeue]]
[[Comando sinfo]]
[[Comando sacct]]
[[Comando scancel y scontrol]]

### 5. Referencia rápida
- [[Tabla de comandos de Slurm]]
- [[Tabla de directivas más usadas]]
- [[Estados de un Job (Job State Codes)]]
- [[Codigos de razon de un Job (Job Reason Codes)]]
- [[Catalogo de estados de los nodos en sinfo]]

---

> [!note] ¿Eres del equipo de cómputo?
> La administración del clúster (usuarios, `slurm.conf`, recuperación de nodos, BeeGFS) está en el [[Manual del Administrador]].
