---
title: scancel y scontrol — cancelar e inspeccionar
tags:
  - slurm
  - comandos
---

## `scancel`: cancelar trabajos

```bash
scancel 10500                    # un trabajo
scancel 10500_7                  # un subtrabajo de un arreglo
scancel -u $USER                 # TODOS mis trabajos (¡cuidado!)
scancel -u $USER -t PENDING      # solo mis pendientes
scancel --name=stack1            # por nombre
```
Solo puedes cancelar **tus** trabajos. El administrador puede cancelar cualquiera.

## `scontrol`: ver detalles y modificar

`scontrol` muestra **todo** lo que Slurm sabe de un trabajo, nodo o partición. Como usuario lo usarás sobre todo para **consultar**.

### Consultar
```bash
scontrol show job 10500          # recursos, nodo, razón de espera, rutas de .out/.err, WorkDir…
scontrol show node node4         # CPUs, RealMemory, carga, estado y Reason
scontrol show partition extra    # nodos, límites y si está UP
```

Campos que más ayudan en `show job`:
- `JobState` y `Reason`: en qué está y por qué espera.
- `StdOut` / `StdErr`: dónde quedarán los registros.
- `TRES=cpu=16,mem=64G,node=1`: lo que realmente se te asignó.
- `StartTime`: si está `PD`, la estimación de arranque.

### Modificar un trabajo pendiente
```bash
scontrol hold 10501              # pausar (no arrancará)
scontrol release 10501           # reanudar
scontrol update jobid=10501 TimeLimit=06:00:00     # un usuario solo puede BAJAR el tiempo
scontrol update jobid=10501 Partition=all
```

> [!info] Comandos de administrador
> `scontrol update nodename=… state=…`, `scontrol reconfigure` y similares requieren privilegios. Están en [[Recuperar nodos y servicios de Slurm]] y [[Archivo de configuracion slurm conf|slurm.conf]].

## Entrar al nodo donde corre tu trabajo

```bash
srun --jobid=10500 --overlap --pty bash
# ya dentro del nodo:
htop            # ¿están trabajando los núcleos que pedí?
free -g         # ¿cuánta memoria estoy usando?
exit
```
Esto **no** crea un trabajo nuevo: se une a tu reserva existente. Es la mejor forma de verificar que tu programa usa los recursos que pediste.

Anterior: [[Comando sacct]] · Volver a [[Manejo de Slurm]]
