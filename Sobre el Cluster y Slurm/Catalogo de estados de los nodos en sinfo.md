---
title: Catálogo de estados de los nodos en sinfo
tags:
  - slurm
  - nodos
---
Los nodos del clúster pasan por distintos estados según la carga de trabajo de los usuarios o la condición de su propio hardware. El estado aparece en la columna `STATE` de [[Comando sinfo|sinfo]].

## Estados en los que el nodo SÍ puede recibir trabajos

| Estado | Significado | ¿Qué hago como usuario? |
|---|---|---|
| **`idle`** | Completamente libre: ningún trabajo, todos sus núcleos y su memoria disponibles. | Nada. Tu trabajo arrancará de inmediato si cabe. |
| **`mix`** _(mixed)_ | Parcialmente ocupado: algunos núcleos o parte de la memoria están en uso, pero quedan recursos libres. | Revisa cuánto queda con `sinfo -N -o "%N %C %e"`. Si tu petición cabe, arranca. |
| **`alloc`** _(allocated)_ | Completamente ocupado: todos sus núcleos (o su memoria o GPU) están reservados. | Tu trabajo esperará con la razón `Resources`. Considera otra partición o nodo. |
| **`comp`** _(completing)_ | Uno o más trabajos están terminando y liberando recursos. | Normalmente dura segundos. Si persiste varios minutos, avisa al admin. |

## Estados en los que el nodo NO recibe trabajos nuevos

| Estado | Significado | ¿Qué hago como usuario? | Admin |
|---|---|---|---|
| **`drng`** _(draining)_ | Termina los trabajos que ya tenía, pero **no acepta nuevos**. Normalmente es un mantenimiento programado. | No lo pidas con `--nodelist`. Usa otro nodo. | [[Recuperar nodos y servicios de Slurm\|receta drain]] |
| **`drain`** / **`drained`** | Ya **vacío** y sin aceptar trabajos. Puede ser por mantenimiento o porque Slurm detectó un problema (`sinfo -R` dice la razón). | Igual que arriba. | [[Recuperar nodos y servicios de Slurm\|receta drain]] |
| **`down`** | Fuera de servicio: falla, reinicio inesperado o retirado por el admin. | Avisa al admin si no hay un mantenimiento anunciado. | [[Recuperar nodos y servicios de Slurm\|receta down]] |
| **`down*`** | Fuera de servicio y **sin responder** al master (ver sufijo `*` abajo). | Avisa al admin. Los trabajos con ese nodo en `--nodelist` quedarán en `ReqNodeNotAvail`. | [[Recuperar nodos y servicios de Slurm\|receta down*]] |
| **`maint`** | Dentro de una **reserva de mantenimiento**. Solo los administradores pueden usarlo. | Espera al fin del mantenimiento (`scontrol show reservation`). | [[Recuperar nodos y servicios de Slurm\|mantenimiento]] |
| **`resv`** _(reserved)_ | Reservado para ciertos usuarios o un proyecto. | Solo puedes usarlo si la reserva te incluye (`--reservation=<nombre>`). | — |
| **`fail`** | Marcado como fallado por el admin (hardware defectuoso). | Usa otro nodo. | [[Recuperar nodos y servicios de Slurm\|receta fail]] |
| **`inval`** _(invalid)_ | El nodo se registró con una configuración que no coincide con `slurm.conf`. | Avisa al admin. | [[Recuperar nodos y servicios de Slurm\|receta inval]] |
| **`unk`** _(unknown)_ | Slurm aún no conoce su estado (normal unos segundos después de reiniciar el controlador). | Espera un minuto. | — |

## Sufijos que pueden aparecer pegados al estado

| Sufijo | Ejemplo | Significado |
|---|---|---|
| `*` | `down*`, `idle*` | El nodo **no responde** al master. Causas típicas: falla de red, `slurmd` caído o sistema operativo congelado. |
| `~` | `idle~` | Apagado por ahorro de energía. |
| `#` | `idle#` | Encendiéndose o configurándose. |
| `!` | `idle!` | Pendiente de apagarse. |
| `%` | `idle%` | Apagándose. |
| `$` | `idle$` | En una reserva de mantenimiento. |
| `@` | `mix@` | Reinicio pendiente (programado por el admin). |
| `^` | `idle^` | Reinicio en curso. |
| `-` | `idle-` | Reservado por el planificador para un trabajo de mayor prioridad. |

> [!question] ¿Y `up`?
> `up` **no es un estado de nodo**. Aparece en la columna `AVAIL` de `sinfo` y describe a la **partición**:
> - `up`: acepta y arranca trabajos (lo normal).
> - `down`: acepta trabajos en la cola, pero no los arranca.
> - `drain`: no acepta trabajos nuevos.
> - `inactive`: ni acepta ni arranca.

> [!tip] Para saber POR QUÉ un nodo no está disponible
> ```bash
> sinfo -R
> ```
> Muestra la razón, quién la puso y cuándo.

> Si algún estado no está documentado aquí, consulta la [documentación oficial de `sinfo`](https://slurm.schedmd.com/sinfo.html#SECTION_NODE-STATE-CODES).
