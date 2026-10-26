# Diagnóstico error de coordenadas en subset.lalo (MiaplPy) — PresaJales SenDT27

| Campo | Valor |
|---|---|
| Fecha | 2026-10-20 |
| Directorio | /raid/doki/workdir/PresaJales_SenDT27_202603_202606 (plantilla en /raid/doki/workdir/TEMPLATES) |
| Repo / rama | rsmas_insar / main |
| Entorno | Skynet |
| Componentes | ISCE2, MiaplPy, MintPy, minsarApp, Slurm |

## Objetivo
Diagnosticar por qué `minsarApp.bash .../PresaJales_SenDT27_202603_202606.template --miaplpy --no-mintpy` fallaba durante la fase de preparación de MiaplPy.

## Problema y causa raíz
- Síntoma: `RuntimeError: No corresponding coordinate found for y/x: 31.005/-112.915`, lanzado en `mintpy/objects/coord.py` (`bbox_geo2radar` → `geo2radar` → `_get_lookup_row_col`), invocado desde `miaplpy/objects/utils.py:read_subset_box` durante `miaplpyApp.py: open()`.
- Causa: typo por transposición de dígitos en el límite sur de `subset.lalo` del template usado en esa corrida: `31.005000` en vez de `31.050000`. Todas las demás copias del template (`TEMPLATES/presas_jales/old/PresaJales_SenDT27_202603_202606.template`, `TEMPLATES/presas_jales/PresaJales_SenDT27_202603_202607.template`, y el backup del template de una corrida previa exitosa del mismo proyecto en `workdir/bkup/PresaJales_SenDT27_202603_202606/`) usan consistentemente `31.050000`.
- Efecto: el valor erróneo desplaza la esquina suroeste del AOI (`31.005, -112.915`) fuera de la huella real del swath SAR cubierta por `lat.rdr.full`/`lon.rdr.full`, por lo que no existe píxel de radar correspondiente y el lookup geo→radar falla.

## Corrección propuesta (no aplicada aún)
- Restaurar en el template usado:
  - `mintpy.subset.lalo  = 31.200429:31.050000,-112.915000:-112.624130`
  - `miaplpy.subset.lalo = 31.200429:31.050000,-112.915000:-112.624130`
- Verificar y limpiar el directorio de proyecto `PresaJales_SenDT27_202603_202606/miaplpy/` si quedó un `custom_smallbaselineApp.cfg` con el valor erróneo antes de volver a correr.
