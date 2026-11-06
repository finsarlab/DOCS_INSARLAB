# Diagramas

Cinco diagramas en [Mermaid](https://mermaid.js.org/). GitHub y VS Code (con la extensión "Markdown Preview Mermaid Support") los muestran como gráficos. Cada uno responde una sola pregunta:

| # | Pregunta | Fuente |
|---|---|---|
| 1 | ¿Cuáles son las piezas grandes y quién usa a quién? | Lectura del código |
| 2 | ¿Qué pasos sigue un proyecto y qué produce cada uno? | Lectura + grafo, corregido con la revisión del 06-11-2026 |
| 3 | ¿Por qué con MintPy se puede correr MiaplPy, pero no al revés? | Lectura guiada por el grafo completo |
| 4 | ¿Qué módulos Python dependen de cuáles? (para modularizar) | Grafo, verificado contra la lectura |
| 5 | ¿Qué programas quedan escritos dentro de los `.job`? | Grafo |

Los números de línea de cada llamada están en [`resultados/HUECOS_REPORTE.md`](resultados/HUECOS_REPORTE.md#flujo-de-minsarappbash-por-paso). El grafo completo se explora en [`resultados/grafo.html`](resultados/grafo.html).

---

## 1. Componentes

El usuario solo interactúa con `minsarApp.bash`. Este llama a los **pasos** Python, que se apoyan en el **núcleo** compartido, y a la capa de **ejecución**, que envía trabajos al clúster. El cálculo científico ocurre en las herramientas de **terceros**. Las flechas punteadas son lectura de configuración o credenciales.

```mermaid
flowchart TB
    U(["Usuario"])

    subgraph CFG["Entorno y configuración"]
        TPL["Template del proyecto<br/>samples/*.template"]
        ENV["setup/<br/>environment.bash<br/>platforms_defaults.bash"]
        DEF["minsar/defaults/<br/>job_defaults.cfg, queues.cfg<br/>minsar_template*.cfg"]
        ACC["accounts/ y tools/SSARA/<br/>credenciales"]
    end

    APP["minsarApp.bash<br/>(orquestador)"]

    subgraph PASOS["Pasos: scripts Python"]
        DL["generate_download_command.py"]
        DEM["dem_rsmas.py"]
        RUN["create_runfiles.py"]
        H5["create_save_hdf5_jobfile.py"]
        PUB["upload_data_products.py<br/>create_insarmaps_jobfile.py"]
        FIN["create_html.py<br/>summarize_job_run_times.py"]
    end

    subgraph NUCLEO["Núcleo Python"]
        CORE["dataset_template.py (Template)<br/>process_utilities.py<br/>auto_defaults.py (PathFind)<br/>message_rsmas.py"]
        JS["job_submission.py<br/>(JOB_SUBMIT)"]
    end

    subgraph EJEC["Ejecución y vigilancia"]
        RW["run_workflow.bash"]
        SJ["submit_jobs.bash<br/>sbatch_conditional.bash"]
        CHK["check_job_outputs.py"]
    end

    subgraph TER["Terceros: rsmas_insar/tools"]
        SSARA["SSARA / ASF"]
        SARDEM["sardem"]
        ISCE["ISCE2 topsStack"]
        MINT["MintPy"]
        MIA["MiaplPy"]
        IM["insarmaps_scripts"]
    end

    SLURM[("Clúster SLURM")]

    U --> APP
    TPL --> APP
    ENV -.-> APP
    APP --> DL & DEM & RUN & H5 & PUB & FIN
    APP --> RW
    DL & DEM & RUN & H5 & PUB --> CORE
    RUN & H5 & PUB --> JS
    DEF -.-> CORE
    DEF -.-> JS
    DL --> SSARA
    ACC -.-> SSARA
    DEM --> SARDEM
    RUN --> ISCE
    RW --> SJ --> SLURM
    RW --> CHK
    SLURM --> ISCE & MINT & MIA & IM
```

---

## 2. Flujo por pasos

Cada caja es un paso de `minsarApp.bash` (un bloque `if [[ $<paso>_flag == "1" ]]`). Los paralelogramos son archivos. Lo importante: **el paso ifgram no siempre corre los mismos pasos de ISCE2**, y MintPy y MiaplPy **leen cosas distintas**. El diagrama 3 explica por qué.

```mermaid
flowchart TB
    T[/"Template *.template"/]

    subgraph S1["1. download"]
        GDC["generate_download_command.py"] --> SSC[/"ssara_command.txt"/]
        SSC -.->|"Bash lo ejecuta leyendo el archivo"| SFQ["ssara_federated_query.bash<br/>+ wget + check_download.py"]
    end

    subgraph S2["2. dem"]
        DEM["dem_rsmas.py + sardem"]
    end

    subgraph S3["3. jobfiles"]
        CR["create_runfiles.py<br/>stackSentinel.py (ISCE2) + JOB_SUBMIT"]
    end

    subgraph S4["4. ifgram: ISCE2 en SLURM vía run_workflow.bash"]
        P15["pasos 1-5<br/>desempaquetar y corregistrar"]
        P67["pasos 6-7<br/>región válida y unir SLC"]
        P8["paso 8"]
        P911["pasos 9-11<br/>unir, filtrar y desenrollar interferogramas"]
        P15 -->|"entre 5 y 6: puede cambiar la fecha de<br/>referencia y repetir 1-5 (solo Sentinel-1)"| P67 --> P8
        P8 -->|"solo si se pidió MintPy"| P911
    end

    SLC[/"SLC/*.zip"/]
    DEMF[/"DEM/*.dem"/]
    RF[/"run_files/*.job"/]
    MSLC[/"merged/SLC"/]
    MIFG[/"merged/interferograms"/]

    subgraph S5["5. mintpy"]
        MP["smallbaseline_wrapper.job<br/>check_timeseries_file.bash, smallbaselineApp.py, create_html.py"]
    end

    subgraph S6["6. miaplpy (opcional)"]
        MIA["miaplpyApp.py --jobfiles → sus 9 pasos<br/>(hace sus propios interferogramas)<br/>+ paso 10 create_save_hdf5_jobfile.py"]
    end

    OUT1[/"mintpy/*.he5"/]
    OUT2[/"miaplpy*/network_*/*.he5"/]
    UP["upload / insarmaps<br/>apagados (R3)"]
    FIN["7. finishup<br/>summarize_job_run_times.py"]

    T --> GDC
    SFQ --> SLC --> DEM --> DEMF
    SLC & DEMF --> CR --> RF --> P15
    P67 --> MSLC
    P911 --> MIFG
    MIFG --> MP --> OUT1
    MSLC --> MIA --> OUT2
    OUT1 & OUT2 -.-> UP
    OUT1 & OUT2 --> FIN

    classDef apagado fill:#eeeeee,stroke:#999999,color:#666666
    class UP apagado
```

- **Paso 8:** con `topsStack.workflow = interferogram` es `generate_burst_igram`; con `workflow = slc` es `grid_baseline`.
- **Omitido:** la variante `--chunks`, que divide la zona y lanza un `minsarApp.bash` por franja ([04 §4.10](04-flujo-de-datos.md#410-variante-procesamiento-por-trozos---chunks)).

---

## 3. El camino MintPy / MiaplPy

MiaplPy no usa los interferogramas de ISCE2: lee `merged/SLC` y hace los suyos. MintPy, en cambio, necesita `merged/interferograms`, que solo existe si ISCE2 corrió los pasos 9 a 11. Por eso **elegir MintPy deja abierto MiaplPy, pero elegir solo MiaplPy cierra MintPy**.

```mermaid
flowchart LR
    A["Camino A: MintPy<br/>--mintpy (valor por defecto)"] --> A1["ifgram: pasos 1-11"] --> A2[/"merged/SLC<br/>+ merged/interferograms<br/>+ smallbaseline_wrapper.job"/]
    A2 --> A3["MintPy ✔"]
    A2 --> A4["MiaplPy ✔<br/>(con --miaplpy, después)"]

    B["Camino B: solo MiaplPy<br/>--no-mintpy --miaplpy, --start miaplpy<br/>o workflow = slc"] --> B1["ifgram: pasos 1-8"] --> B2[/"solo merged/SLC"/]
    B2 --> B3["MiaplPy ✔"]
    B2 -.-x B4["MintPy ✘<br/>faltan pasos 9-11 y su job"]
```

Tres configuraciones deciden el camino, y las tres empujan en el mismo sentido:

| Configuración | Dónde se aplica | Camino A | Camino B |
|---|---|---|---|
| `--mintpy` / `--no-mintpy --miaplpy` / `--start miaplpy` | `minsarApp.bash:222-258`, `:441-444` y `:695-697` | ifgram hasta el paso 11 | ifgram hasta el paso 8 |
| `topsStack.workflow` (template) | `stackSentinel.py:1028-1043` y `minsarApp.bash:530-531` | `interferogram`: ISCE2 genera los pasos de interferogramas | `slc`: ISCE2 no los genera y MinSAR **apaga MintPy a la fuerza** |
| `processingMethod` (template) | `create_runfiles.py:239-254` | `smallbaseline`: escribe `smallbaseline_wrapper.job` | `miaplpy`: escribe `miaplpy_wrapper.job` y no el de MintPy |

Las excepciones y los puntos por confirmar están en [R23](riesgos.md#r23).

---

## 4. Dependencias entre módulos Python

Solo imports entre módulos propios: es el mapa para modularizar. **No se dibuja `message_rsmas.py`**: lo usan 30 archivos y llenaría el diagrama de flechas. Los nodos con más flechas entrantes (`process_utilities.py`, `auto_defaults.py`, `job_submission.py`) son los más delicados de cambiar.

Las flechas en **rojo** se agregaron al validar con el grafo ([validacion.md §1](validacion.md#1-fuentes-y-límites)). La de `process_utilities → job_submission` cierra una dependencia circular ([R21](riesgos.md#r21)).

```mermaid
flowchart LR
    subgraph PASOS["Scripts de cada paso"]
        GDC["generate_download_command.py"]
        DEMR["dem_rsmas.py"]
        CR["create_runfiles.py"]
        CSH["create_save_hdf5_jobfile.py"]
        CIJ["create_insarmaps_jobfile.py"]
        UDP["upload_data_products.py"]
        CJO["check_job_outputs.py"]
        SUM["summarize_job_run_times.py"]
        HTML["create_html.py"]
    end

    subgraph CORE["Núcleo"]
        JS["job_submission.py"]
        PU["utils/process_utilities.py"]
        SR["utils/stack_run.py"]
        US["objects/unpack_sensors.py"]
        TPL["objects/dataset_template.py"]
        AD["objects/auto_defaults.py"]
        RL["objects/rsmas_logging.py"]
    end

    KML["utils/get_boundingBox_from_kml.py"]
    PWD["password_config.py<br/>(credenciales)"]

    CR --> JS & PU & SR & US & AD
    CSH --> JS & PU & TPL & AD
    CIJ --> JS & PU & AD
    UDP --> JS & PU & HTML
    CJO --> JS & PU
    GDC --> TPL & PU & AD
    DEMR --> TPL & PU
    SUM --> PU

    JS --> PU & AD
    SR --> PU & AD
    PU --> TPL & AD

    %% Agregadas tras validar con el grafo
    PU -->|"import dentro de función<br/>L513"| JS
    JS --> TPL
    DEMR --> KML
    CIJ & UDP --> PWD
    UDP & SUM --> RL
    linkStyle 29,30,31,32,33,34,35 stroke:#d62728,stroke-width:2px
```

---

## 5. Qué se escribe dentro de los `.job`

Buena parte del pipeline no aparece como llamadas en el código: **se escribe como texto** dentro de archivos que después ejecuta SLURM. Todos pasan por `run_workflow.bash` ([04 §4.5](04-flujo-de-datos.md#cómo-se-ejecuta-cada-paso-en-el-clúster)). En gris, lo que se genera pero no se usa.

```mermaid
flowchart LR
    subgraph GEN["Quién lo escribe"]
        CR["create_runfiles.py"]
        SR["utils/stack_run.py"]
        JS["job_submission.py<br/>JOB_SUBMIT"]
        CSH["create_save_hdf5_jobfile.py"]
        CIJ["create_insarmaps_jobfile.py"]
        MAPP["miaplpyApp.py --jobfiles<br/>(MiaplPy)"]
    end

    subgraph ARCH["Archivos generados"]
        RF[/"run_files/run_01 … run_11<br/>(listas de comandos de ISCE2)"/]
        JOBF[/"run_files/run_XX_*.job"/]
        SW[/"smallbaseline_wrapper.job"/]
        MW[/"miaplpy_wrapper.job"/]
        MRF[/"miaplpy*/network_*/run_files<br/>run_01 … run_09"/]
        H5J[/"run_10_save_hdfeos5_radar_0.job"/]
        IMJ[/"insarmaps.job"/]
    end

    subgraph EJE["Lo que corre en el clúster"]
        ISCE["SentinelWrapper.py -c configs/… (ISCE2)"]
        MP["check_timeseries_file.bash<br/>smallbaselineApp.py (MintPy)<br/>create_html.py"]
        MIA["load_data, phase_linking, generate_ifgram,<br/>unwrap_ifgram, invert_network… (MiaplPy)"]
        MPH["spatial_filter.py · generate_mask.py<br/>save_hdfeos5.py · geocode.py · view.py (MintPy)"]
        INS["hdfeos5_2json_mbtiles.py<br/>json_mbtiles2insarmaps.py"]
    end

    CR --> SR
    SR -->|"stackSentinel.py"| RF
    RF -->|"write_batch_jobs"| JS --> JOBF
    CR -->|"si processingMethod = smallbaseline"| SW
    CR -->|"si processingMethod = miaplpy"| MW
    MAPP --> MRF
    CSH --> H5J
    CIJ --> IMJ

    JOBF -.->|contiene| ISCE
    SW -.->|contiene| MP
    MRF -.->|contiene| MIA
    H5J -.->|contiene| MPH
    IMJ -.->|contiene| INS

    classDef nousado fill:#eeeeee,stroke:#999999,color:#666666
    class MW nousado
```

`miaplpy_wrapper.job` se escribe pero `minsarApp.bash` nunca lo usa: lanza MiaplPy con `srun miaplpyApp.py --jobfiles` ([R23](riesgos.md#r23)).

---
Anterior: [04-flujo-de-datos.md](04-flujo-de-datos.md) · Siguiente: [guia-de-mantenimiento.md](guia-de-mantenimiento.md)
