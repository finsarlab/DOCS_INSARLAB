---
title: sacct — historial de trabajos
tags:
  - slurm
  - comandos
---

> [!summary]
> `sacct` consulta la **base de datos histórica** de Slurm. Sirve para saber **cómo terminó** un trabajo que ya no aparece en `squeue`, cuánto tardó y cuánta memoria usó **de verdad**. Esto último te ayuda a pedir mejor los recursos la próxima vez.

## Lo esencial

```bash
sacct                                   # mis trabajos de hoy
sacct -j 10500                          # un trabajo
sacct -S 2026-10-01                     # desde una fecha
sacct -S now-7days -s F,TO,OOM          # los que fallaron, se quedaron sin tiempo o sin memoria esta semana
```

## El formato recomendado

```bash
sacct -j 10500 --units=G --format=JobID,JobName%20,Partition,NodeList,AllocCPUS,ReqMem,MaxRSS,Elapsed,Timelimit,State,ExitCode
```
```
JobID           JobName  Partition   NodeList  AllocCPUS  ReqMem   MaxRSS    Elapsed  Timelimit      State ExitCode
------------ ---------- ---------- ---------- ---------- ------- -------- ---------- ---------- ---------- --------
10500            stack1      extra      node4         16     64G            05:12:44   12:00:00  COMPLETED      0:0
10500.batch       batch                 node4         16             41.3G  05:12:44             COMPLETED      0:0
10500.extern     extern                 node4         16              0.0G  05:12:44             COMPLETED      0:0
```

**¿Por qué hay 3 renglones?** Cada trabajo se divide en *pasos* (*steps*):
- `10500`: el trabajo completo (la reserva).
- `10500.batch`: tu script. **Aquí aparece `MaxRSS`**, la memoria máxima que usó tu programa.
- `10500.extern`: un paso interno de Slurm. Ignóralo.

> [!tip] `-X` oculta los pasos
> `sacct -X` muestra un solo renglón por trabajo (más limpio), **pero pierdes `MaxRSS`**. Úsalo para listar muchos trabajos y quítalo para revisar la memoria.

## Ajusta tus recursos con `sacct`

Con el ejemplo de arriba:

| Pediste | Usaste | Conclusión |
|---|---|---|
| `ReqMem` = 64G | `MaxRSS` = 41.3G | La próxima vez pide ~48G (usado + 15–20% de margen). |
| `Timelimit` = 12:00:00 | `Elapsed` = 05:12:44 | Puedes pedir `--time=07:00:00`; arrancará antes por *backfill*. |

Si el clúster tiene instalado el complemento `seff`, `seff 10500` te da este resumen directamente (eficiencia de CPU y memoria).

## Columnas útiles para `--format`

| Columna | Qué es |
|---|---|
| `JobID`, `JobName` | Identificación |
| `State`, `ExitCode` | Cómo terminó. Ver [[Estados de un Job (Job State Codes)]] y [[Manejo de Slurm\|códigos de salida]] |
| `Submit`, `Start`, `End` | Cuándo se encoló, arrancó y terminó |
| `Elapsed`, `Timelimit` | Duración real vs. límite pedido |
| `AllocCPUS`, `TotalCPU` | Núcleos reservados vs. tiempo de CPU realmente usado |
| `ReqMem`, `MaxRSS` | Memoria pedida vs. memoria máxima usada |
| `NodeList` | En qué nodo corrió |
| `WorkDir` | Desde qué carpeta se lanzó (útil para encontrar sus `.out`) |

Para que una columna se vea más ancha, agrega `%N` al nombre: `JobName%30`.

> [!warning] Si `sacct` responde "accounting storage is disabled"
> Significa que la base de datos de contabilidad (`slurmdbd`) no está activa. Avisa al administrador (ver [[Recuperar nodos y servicios de Slurm]]).

Anterior: [[Comando sinfo]] · Siguiente: [[Comando scancel y scontrol]] · Volver a [[Manejo de Slurm]]
