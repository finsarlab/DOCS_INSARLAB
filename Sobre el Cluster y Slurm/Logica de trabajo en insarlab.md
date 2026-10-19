---
title: Lógica de trabajo en insarlab
---

## Sobre la conexión
Existen 2 equipos primordiales
- *Equipo local* (tu computadora personal)
- Equipo remoto (*[[Arquitectura del Cluster del InSAR lab|Cluster]]*)

Desde tu *equipo local* tienes que establecer una conexión al cluster mediante [[Protocolo SSH|el protocolo SSH]]. Esto permite que, mediante la consola de tu computadora puedas realizar trabajo de [[Cluster Computing]] usando todo el hardware disponible en el cluster del laboratorio.

Los resultados del [[Cluster Computing]] **no llegan a tu computadora**: se quedan en el clúster, en tu [[#Sobre el almacenamiento|carpeta personal]]. Si los necesitas en tu equipo local, los copias con `scp` o `rsync` (ver [[Protocolo SSH#Copiar archivos entre tu computadora y el clúster|copiar archivos]]).

## Sobre el almacenamiento
Cuando ingresas al cluster, tu directorio home (`~`) está vacío, listo para que puedas crear los ficheros y directorios necesarios en ese nivel, ==así como para crear tus scripts que enlistarás en las tareas para slurm.==[^2]

> [!important] Tu carpeta es la misma en todos los nodos
> Tu `home` no está en el disco del master: vive en un sistema de archivos compartido llamado **BeeGFS**, en la ruta
> ```
> /mnt/beegfs/insarlab/<tu_usuario>
> ```
> Todos los nodos (`master`, `node1`…`node4`) montan esa misma carpeta. Por eso:
> - El script que escribes en el master **existe también** en el nodo donde Slurm lo ejecute.
> - Los resultados que tu trabajo escribe en `node4` los ves de inmediato desde el master.
> - **No necesitas copiar nada** a los nodos.
>
> Lo que **no** se comparte es `/tmp`: cada nodo tiene el suyo. Úsalo solo para archivos temporales dentro de un mismo trabajo.

Puedes comprobarlo con:
```bash
pwd                    # /mnt/beegfs/insarlab/<tu_usuario>
df -h ~                # el sistema de archivos es beegfs
```



[^2]: [[Uso del Cluster]]
