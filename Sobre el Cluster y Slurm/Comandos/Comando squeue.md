---
title: squeue — ver la cola
tags:
  - slurm
  - comandos
---

> [!summary]
> `squeue` muestra los trabajos que están **esperando o corriendo ahora**. Cuando un trabajo termina, desaparece de aquí; para el historial usa [[Comando sacct|sacct]].

## Lo esencial

```bash
squeue                 # toda la cola (todos los usuarios)
squeue -u $USER        # solo mis trabajos
squeue --me            # igual que la anterior (versiones recientes de Slurm)
squeue -j 10500        # un trabajo en particular
squeue -p n2gpu        # quién está usando / esperando la GPU
squeue -t PD           # solo los pendientes
```

## Cómo leer la salida

```
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
10500     extra   stack1   falcon  R    3:12:44      1 node4
10501     extra   stack2   falcon PD       0:00      1 (Resources)
10502     n2gpu      gpu    fersa PD       0:00      1 (Priority)
```

| Columna | Significado |
|---|---|
| `JOBID` | El Job ID |
| `ST` | Estado: `R` corriendo, `PD` pendiente, `CG` terminando. Ver [[Estados de un Job (Job State Codes)]] |
| `TIME` | Cuánto lleva corriendo |
| `NODELIST(REASON)` | Si corre: en qué nodo. Si está `PD`: **por qué espera**. Ver [[Codigos de razon de un Job (Job Reason Codes)]] |

## Opciones útiles

```bash
squeue -u $USER --start          # hora ESTIMADA de arranque de tus pendientes
squeue -u $USER -l               # formato largo (incluye límite de tiempo)
watch -n 15 squeue -u $USER      # se refresca cada 15 s (salir con Ctrl+C)
```

### Formato a la medida
```bash
squeue -u $USER -o "%.10i %.9P %.25j %.2t %.10M %.10l %.4C %.8m %R"
```
| Código | Columna |
|---|---|
| `%i` | Job ID |
| `%P` | Partición |
| `%j` | Nombre |
| `%t` | Estado (corto) |
| `%M` | Tiempo corriendo |
| `%l` | Límite de tiempo |
| `%C` | CPUs |
| `%m` | Memoria pedida |
| `%R` | Nodo o razón |

> [!tip] Hazlo atajo
> Agrega a tu `~/.bashrc`:
> ```bash
> alias sq='squeue -u $USER -o "%.10i %.9P %.25j %.2t %.10M %.10l %.4C %.8m %R"'
> ```
> Luego `source ~/.bashrc` y basta con escribir `sq`.

## ¿Qué pasa dentro de mi trabajo mientras corre?

```bash
scontrol show job 10500                          # todos los detalles
srun --jobid=10500 --overlap --pty htop          # mira CPU y RAM en vivo en su nodo
tail -f logs/stack1_10500.out                    # sigue la salida en tiempo real
```
Ver [[Comando scancel y scontrol]].

Anterior: [[Comando sbatch]] · Siguiente: [[Comando sinfo]] · Volver a [[Manejo de Slurm]]
