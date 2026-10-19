---
title: sinfo — ver el estado de los nodos
tags:
  - slurm
  - comandos
---

> [!summary]
> `sinfo` muestra **el hardware**: qué particiones existen, qué nodos tienen y en qué estado está cada uno. Úsalo **antes de encolar** para decidir a dónde mandar tu trabajo.

## Lo esencial

```bash
sinfo                  # vista por partición
sinfo -N -l            # vista por nodo, formato largo (CPUs, memoria, estado)
sinfo -p extra         # solo una partición
sinfo -n node3         # solo un nodo
sinfo -R               # nodos caídos o drenados y POR QUÉ
sinfo -t idle          # solo nodos libres
```

La explicación de cada columna de la vista por defecto está en [[Manejo de Slurm#`sinfo`|Manejo de Slurm]].

## La consulta más útil: ¿cuánto hay libre en cada nodo?

```bash
sinfo -N -o "%10N %10P %15C %10m %10e %12G %T"
```
```
NODELIST   PARTITION  CPUS(A/I/O/T)   MEMORY     FREE_MEM   GRES         STATE
master     all*       0/40/0/40       126000     98000      (null)       idle
node1      all*       48/8/0/56       257000     40210      (null)       mixed
node2      extra      0/56/0/56       240000     230100     gpu:l4:1     idle
node2      n2gpu      0/56/0/56       240000     230100     gpu:l4:1     idle
node3      extra      0/0/24/24       31000      N/A          (null)       down*
node4      extra      0/56/0/56       257000     250300     (null)       idle
```
*(Los números son ilustrativos.)*

| Código | Significado |
|---|---|
| `%N` | Nodo |
| `%P` | Partición |
| `%C` | CPUs **A**signadas / **I**nactivas (libres) / **O**tras (no disponibles) / **T**otal |
| `%m` | Memoria configurada en Slurm, en MB. **Es el máximo que puedes pedir con `--mem`** |
| `%e` | Memoria libre real del sistema operativo, en MB |
| `%G` | Recursos genéricos (GPU) |
| `%T` | Estado |
| `%E` | Razón del estado (útil con nodos `down`/`drain`) |

**Cómo se lee el ejemplo:** `node1` tiene 8 núcleos libres. `node4` y `node2` están completamente libres. `node3` no responde (`down*`), así que un trabajo con `--nodelist=node3` se quedará esperando.

## Estados de los nodos

![[Catalogo de estados de los nodos en sinfo]]

> [!question] ¿Y el `up` de la columna `AVAIL`?
> `up` **no es un estado de nodo**: es el estado de la **partición**. `up` = acepta y arranca trabajos. `down` = acepta trabajos, pero no los arranca. `drain` = no acepta nuevos. `inactive` = ni acepta ni arranca.

Anterior: [[Comando squeue]] · Siguiente: [[Comando sacct]] · Volver a [[Manejo de Slurm]]
