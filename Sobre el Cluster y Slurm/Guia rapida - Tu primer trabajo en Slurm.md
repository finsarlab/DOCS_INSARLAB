---
title: "Guía rápida: tu primer trabajo en Slurm (10 minutos)"
tags:
  - slurm
  - guia-rapida
---

> [!summary] Para quién es esta guía
> Para quien ya tiene usuario y llave, y quiere **mandar su primer trabajo al clúster ya**. La teoría está en el [[Manual sobre el Cluster del InSAR lab y Slurm|manual completo]]; aquí solo están los pasos.
> ¿Aún no tienes acceso? Primero: [[Creacion de Usuario]].

## 1. Conéctate

```bash
ssh insarTuNombre          # el Host que pusiste en ~/.ssh/config
```
Verás algo como `[tu_usuario@master ~]$`. Estás en el **master**: aquí se *encolan* trabajos, **no se ejecutan** procesos pesados ([[Uso del Cluster|¿por qué?]]).

## 2. Mira cómo está el clúster

```bash
sinfo
```
```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
all*         up   infinite      1    mix node1
all*         up   infinite      1   idle master
extra        up   infinite      3   idle node[2-4]
n2gpu        up   infinite      1   idle node2
```
`idle` = libre, `mix` = parcialmente ocupado, `alloc` = lleno, `down`/`drain` = no disponible. Más en [[Comando sinfo]].

## 3. Prepara una carpeta de práctica

```bash
mkdir -p ~/practica_slurm/logs
cd ~/practica_slurm
```

## 4. Crea un programa de prueba

```bash
cat > hola.py <<'PY'
import os, socket, time
print(f"Hola desde {socket.gethostname()}")
print(f"Núcleos que Slurm me dio: {len(os.sched_getaffinity(0))}")
time.sleep(30)
print("Listo")
PY
```

## 5. Crea el archivo de trabajo (a partir de una plantilla)

```bash
cat > hola.sh <<'SH'
#!/bin/bash
#SBATCH --job-name=hola
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err
#SBATCH --partition=extra
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=1G
#SBATCH --time=00:05:00

echo "Job $SLURM_JOB_ID en $(hostname) - $(date)"
python3 hola.py
SH
```
Cada línea `#SBATCH` es una **directiva**: le pide recursos a Slurm. Están explicadas en [[Archivo de trabajo y directivas SBATCH]].

## 6. Encola

```bash
sbatch hola.sh
```
```
Submitted batch job 10601
```
Ese número es tu **Job ID**. Anótalo.

## 7. Síguelo

```bash
squeue -u $USER
```
```
JOBID PARTITION  NAME     USER ST  TIME NODES NODELIST(REASON)
10601     extra  hola  falcon  R  0:07     1 node3
```
`R` = corriendo, `PD` = esperando (la última columna dice por qué). Cuando termina, desaparece de `squeue`. Más en [[Comando squeue]].

## 8. Lee el resultado

```bash
cat logs/hola_10601.out
cat logs/hola_10601.err      # vacío = sin errores
```
```
Job 10601 en node3 - jue oct  9 12:00:01 CST 2026
Hola desde node3
Núcleos que Slurm me dio: 2
Listo
```

## 9. Confirma cómo terminó

```bash
sacct -j 10601 --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
```
`COMPLETED` y `0:0` = todo bien. Más en [[Comando sacct]].

🎉 **Listo.** Ya hiciste el ciclo completo: *escribir → encolar → seguir → revisar*. A partir de aquí, copia la [[Plantillas de Slurm|plantilla]] que se parezca a lo que necesitas.

---

## Si algo salió mal

| Síntoma | Causa probable | Solución |
|---|---|---|
| `sbatch: error: Batch job submission failed: Requested node configuration is not available` | Pediste más CPUs o RAM de las que tiene cualquier nodo de esa partición, o un `--nodelist` que no pertenece a la partición | Revisa [[Grupos de nodos y particiones]] |
| `sbatch: error: invalid partition specified` | Error de dedo en `--partition` | Usa `all`, `extra` o `n2gpu` |
| El trabajo desaparece y **no hay `.out`** | La carpeta `logs/` no existía | `mkdir -p logs` y vuelve a encolar |
| Se queda en `PD` con razón `Resources` | Nodos ocupados | Esperar, o pedir menos recursos |
| Se queda en `PD` con `ReqNodeNotAvail` | Pediste un nodo caído o en mantenimiento | `sinfo -R`, quitar `--nodelist` o avisar al admin |
| Estado `OUT_OF_MEMORY` | Tu programa usó más RAM que `--mem` | Sube `--mem` (mira `MaxRSS` en `sacct`) |
| Estado `TIMEOUT` | Se acabó el `--time` | Sube `--time` |
| Estado `FAILED` | Error en tu código | Lee el `.err` |
| `python3: command not found` o módulos que faltan | No activaste tu entorno dentro del script | Activa pixi/conda **en el script**, no solo en tu terminal |

## Reglas de convivencia del InSAR lab
1. **Nada pesado en el master.** Todo pasa por `sbatch` (o `srun` para pruebas interactivas).
2. **Pide lo que usas.** Pedir 56 núcleos para un script de un hilo deja a tus compañeros esperando.
3. **Declara siempre `--partition`, `--mem` y `--time`.**
4. **No uses `node2` si no necesitas GPU** (`--exclude=node2`).
5. **Cancela lo que ya no necesitas:** `scancel <JOBID>`.
