---
title: Arquitectura del Clúster del InSAR lab
---

> [!info]
> El sistema operativo del Cluster es: `Rocky Linux`

El [[Cluster Computing|clúster]] del **InSAR lab** está diseñado para ofrecer una gran capacidad y flexibilidad a todos los miembros del equipo. Operando bajo el sistema operativo _Rocky Linux_, se compone de un nodo maestro y cuatro nodos de cómputo, donde cada equipo aporta características específicas para nuestros proyectos.

> [!summary] Nodos del Clúster de **InSAR lab**
> - Nodo `master`
> - Nodos de cómputo (nodos trabajadores) ^fb106f
> 	- `node1`
> 	- `node2`
> 	- `node3`
> 	- `node4`

Para facilitar la comprensión de los recursos con los que contamos, a continuación se detalla la configuración de nuestra arquitectura:

|**Nodo**|**Tipo**|**Núcleos (CPUs)**|**Memoria RAM**|**Características principales**|
|---|---|---|---|---|
|**master**|Maestro|40|~126 GB|Punto de acceso. Gestiona la cola y coordina a los demás nodos.|
|**node1**|Cómputo|56|~257 GB|Alta capacidad para procesos altamente demandantes.|
|**node2**|Cómputo|56|~240 GB|Equipado con una **GPU L4** para aceleración por hardware y tareas especializadas.|
|**node3**|Cómputo|24|~31 GB|Equipo de apoyo eficiente para tareas de menor exigencia.|
|**node4**|Cómputo|56|~257 GB|Alta capacidad para procesos altamente demandantes.|

Para organizar de manera inteligente el acceso a estos recursos, el InSAR lab utiliza el gestor de trabajos **Slurm**. 



# Estructura usando Slurm
> [!done]
> Se recomienda revisar primero el material:
> - [[Uso del Cluster]]
> - [[Manejo de Slurm]]
> antes de leer la siguiente configuración.

Este software es el encargado de administrar el flujo de tareas, agrupando los nodos en colas o secciones lógicas conocidas como _particiones_.

Actualmente, el clúster está dividido en tres particiones principales a las que podrás enviar tus trabajos:

- **all** (Partición por defecto): Agrupa los recursos del _master_ y el _node1_.
- **extra**: Consolida la potencia de los nodos de cómputo _node2_, _node3_ y _node4_ para trabajos que requieran dicha infraestructura.
- **n2gpu**: Partición de propósito específico destinada exclusivamente a encolar trabajos que necesiten aprovechar la tarjeta gráfica del _node2_.


> [!warning] Detalles que conviene saber
> - La partición por defecto `all` **incluye al master**: un trabajo sin `--partition` puede ejecutarse en el nodo coordinador. Declara siempre la partición.
> - `node2` pertenece a `extra` **y** a `n2gpu`: si no necesitas GPU, exclúyelo (`--exclude=node2`) para no bloquear la tarjeta.
>
> Guía práctica para elegir: [[Grupos de nodos y particiones]].

En las siguientes secciones de este manual, aprenderemos paso a paso cómo comunicarnos con Slurm para configurar y enviar nuestros trabajos a cada una de estas particiones de la manera correcta.