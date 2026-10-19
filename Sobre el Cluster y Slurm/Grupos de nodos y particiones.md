---
title: Grupos de nodos y particiones (¿a dónde mando mi trabajo?)
tags:
  - slurm
  - particiones
  - nodos
---

> [!summary] En una frase
> Antes de escribir tu script decide **qué tipo de máquina necesitas**. Esta nota traduce la [[Arquitectura del Cluster del InSAR lab|arquitectura del clúster]] en decisiones concretas: qué partición pedir, cuánta memoria puedes solicitar y qué nodo conviene evitar.

## Lo que hay en el clúster

| Nodo | Particiones | CPUs | RAM aprox. | Red hacia el clúster[^1] | Perfil |
|---|---|---|---|---|---|
| `master` | `all` (por defecto) | 40 | ~126 GB | 10 Gb | Coordinador. **No es para cómputo pesado.** |
| `node1` | `all` (por defecto) | 56 | ~257 GB | **10 Gb** | Alta capacidad. El mejor conectado. |
| `node2` | `extra`, `n2gpu` | 56 | ~240 GB | 1 Gb | Alta capacidad **+ GPU NVIDIA L4** |
| `node3` | `extra` | 24 | ~31 GB | 1 Gb | Ligero, de apoyo |
| `node4` | `extra` | 56 | ~257 GB | 1 Gb | Alta capacidad |

> [!warning] Los valores de RAM son aproximados
> Slurm no te deja pedir la RAM "física" del equipo, sino la que tiene registrada en su configuración (`RealMemory`), que siempre es un poco menor. Antes de pedir mucha memoria, consulta el valor real:
> ```bash
> sinfo -N -o "%N %P %c %m %G %T"
> ```
> La columna `MEMORY` está en **MB** (por ejemplo `257000` ≈ 251 GB). Si pides más de eso, `sbatch` rechaza el trabajo de inmediato. Ver [[Comando sinfo]].

## Los grupos de nodos

Desde el punto de vista de quien lanza trabajos, el clúster se divide en cuatro grupos:

> [!example] Grupo A — Alta capacidad CPU: `node1`, `node4`
> - Hasta 56 núcleos y ~250 GB por nodo.
> - Ideal para procesar *stacks* InSAR, inversiones de series de tiempo y cualquier cosa pesada en CPU y RAM.
> - `node1` está en la partición `all`, mientras que `node4` está en `extra`. Para que Slurm use **el que se libere primero**, pide las dos particiones y excluye los nodos que no quieres (ver [[Plantilla - Grupo de nodos de alta capacidad]]).

> [!example] Grupo B — GPU: `node2`
> - Única GPU del clúster (NVIDIA L4).
> - Se pide con `--partition=n2gpu` **y** `--gres=gpu:l4:1` (ver [[Plantilla - GPU (n2gpu)]]).
> - **Cortesía con el equipo:** `node2` también pertenece a `extra`. Si tu trabajo **no** usa GPU y lo mandas a `extra`, agrega `--exclude=node2` para no bloquear a quien sí necesita la tarjeta.

> [!example] Grupo C — Ligero: `node3`
> - 24 núcleos, pero solo ~31 GB de RAM.
> - Úsalo para pruebas, scripts cortos, descargas o conversiones de formatos.
> - Nunca le pidas más de ~28 GB: el trabajo no arrancará.

> [!example] Grupo D — `master`
> - Está en la partición por defecto `all`, así que **un `sbatch` sin `--partition` puede terminar corriendo en el master**.
> - Recomendación del equipo: declara siempre `--partition` y, si usas `all`, agrega `--exclude=master` o fija `--nodelist=node1`.

## Tabla de decisión rápida

| Lo que necesitas | Directivas sugeridas |
|---|---|
| Prueba rápida (minutos, poca RAM) | `--partition=extra --nodelist=node3 --mem=4G --time=00:30:00` |
| Proceso pesado CPU, cualquier nodo grande | `--partition=all,extra --exclude=master,node2,node3 --mem=200G` |
| Proceso pesado que lee/escribe muchísimo en disco | `--partition=all --nodelist=node1` (enlace de 10 Gb)[^2] |
| Un nodo específico (por reproducibilidad o depuración) | `--nodelist=node4 --partition=extra` |
| GPU | `--partition=n2gpu --gres=gpu:l4:1` |
| Varios nodos al mismo tiempo (MPI) | `--partition=extra --nodes=2 --nodelist=node[2,4]` |

> [!danger] `--nodelist` con varios nodos NO significa "cualquiera de estos"
> `--nodelist=node2,node4` le dice a Slurm "**necesito los dos nodos a la vez**". Si lo que quieres es "el que esté libre entre `node1` y `node4`", usa particiones + `--exclude`, como en el Grupo A.

> [!tip] Una partición no puede mezclarse con nodos que no le pertenecen
> `--partition=extra --nodelist=node1` es un error: `node1` no pertenece a `extra`. Revisa la columna *Particiones* de la tabla.

## Límites de tiempo

Hoy todas las particiones muestran `TIMELIMIT infinite` en `sinfo`. Aun así, **declara siempre `--time`**:
- Slurm puede meter tu trabajo en un "hueco" del calendario (*backfill*) si sabe cuánto va a durar. Los trabajos con tiempo declarado suelen arrancar antes.
- Evita que un proceso colgado ocupe un nodo durante días.

Formatos válidos: `30` (minutos), `04:00:00` (h:m:s), `2-00:00:00` (días-h:m:s).

---
Siguiente paso: [[Archivo de trabajo y directivas SBATCH]] o directo a las [[Plantillas de Slurm]].

[^1]: Según el [[Arquitectura del Cluster de InSAR lab (para admin)|diagrama de red]], `node1` y el master están conectados por los puertos de 10 Gb del switch; `node2`, `node3` y `node4` por los de 1 Gb.
[^2]: Pendiente de confirmar dónde viven físicamente los discos de [[BeeGFS]]. Si están en el master, `node1` tiene la ruta más rápida hacia tus archivos.
