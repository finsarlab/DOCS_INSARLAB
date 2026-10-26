# Fix subset.lalo fuera del swath en MiaplPy/MintPy

| Campo | Valor |
|---|---|
| Fecha | 2026-10-20 |
| Directorio | /raid/doki/workdir/PresaJales_SenDT27_202603_202606 |
| Repo / rama | rsmas_insar / main |
| Entorno | Skynet |
| Componentes | ISCE2, MiaplPy, MintPy, minsarApp |

## Objetivo
Correr `minsarApp.bash PresaJales_SenDT27_202603_202606.template --miaplpy --no-mintpy` sin error de coordenadas.

## Problema y causa raíz
- Síntoma: `RuntimeError: No corresponding coordinate found for y/x: 31.005/-112.915` en `coord.py::_get_lookup_row_col` durante `miaplpyApp.py::read_initial_info`.
- Causa: `miaplpy.subset.lalo` y `mintpy.subset.lalo` definían un rectángulo cuya esquina SW caía fuera del swath real (que es un paralelogramo inclinado por el ángulo del track descendente), por lo que no existe pixel radar correspondiente a esa lat/lon.
  - Verificado con `lat.rdr`/`lon.rdr` (merged/geom_reference, 838x2310 doubles): en lat=31.005 el dato solo llega a lon=-112.78 (se pedía -112.915); en lon=-112.915 el dato solo llega a lat=31.042 (se pedía 31.005).

## Cambios realizados
### Archivos modificados/creados
- `/raid/doki/workdir/TEMPLATES/PresaJales_SenDT27_202603_202606.template` — `miaplpy.subset.lalo` cambiado de `31.200429:31.005000,-112.915000:-112.624130` a `31.200429:31.050000,-112.915000:-112.624130`; `mintpy.subset.lalo` cambiado de `31.200429:30.994449,-112.927522:-112.624130` a `31.200429:31.050000,-112.915000:-112.624130`. Ambas cajas verificadas dentro del swath real.
- `/raid/doki/workdir/PresaJales_SenDT27_202603_202606/miaplpy/miaplpyApp.cfg` y `custom_smallbaselineApp.cfg` — edición de `subset.lalo` en caché del proyecto rechazada por el usuario (pendiente, no aplicada).

### Comandos relevantes
```bash
mkdir -p /raid/doki/docs-cluster/sesiones
```

## Pendiente
- Confirmar si se debe actualizar también `miaplpyApp.cfg`/`custom_smallbaselineApp.cfg` en el directorio del proyecto (caché de la corrida fallida) o si el re-run del template ya los sobreescribe.
