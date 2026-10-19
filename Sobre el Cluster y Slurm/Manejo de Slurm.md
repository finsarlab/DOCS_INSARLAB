> [!done] Previamente
> Para mandar un trabajo al cluster se debe escribir entonces un script `.sh` que ==le indique a slurm los recursos que sabes necesitará tu trabajo, y lo que hará tu trabajo== [^2].

> [!tip] ¿Con prisa?
> - Paso a paso en 10 minutos: [[Guia rapida - Tu primer trabajo en Slurm]]
> - Scripts listos para copiar: [[Plantillas de Slurm]]
> - Caso completo de principio a fin: [[Ejemplo de flujo de trabajo completo]]

# Archivos de Salida y Error (``.out`` y ``.err``)

Cuando ejecutas un programa en tu computadora personal o en el nodo maestro, estás acostumbrado a ver los resultados, las barras de progreso o los mensajes de error imprimiéndose en tiempo real en tu pantalla. Sin embargo, cuando envías un trabajo a Slurm, este se ejecuta de manera automatizada en un nodo de cómputo en el fondo (en _background_), lo que significa que no hay una terminal conectada para mostrarte esa información.

Para no perder estos datos vitales, Slurm captura todo lo que tu programa intente imprimir y lo guarda en archivos de texto. Es una buena práctica separar esta información en dos archivos distintos utilizando [[Uso del Cluster#Las directivas de recursos|directivas]]:

- **Archivo Output (`--output`):** Captura la "salida estándar" (stdout). Aquí se guardará todo lo que tu código imprima intencionalmente, como resultados numéricos, confirmaciones de `print()` o estados de progreso.
- **Archivo Error (`--error`):** Captura la "salida de error estándar" (stderr). Si tu código falla, le falta una librería o se interrumpe, el motivo del fallo se escribirá exclusivamente aquí.

>Tener estos archivos separados es indispensable en la automatización del clúster. Si tu trabajo finaliza pero no obtuviste los resultados esperados, el primer paso siempre será abrir el archivo `.err` para leer el diagnóstico del problema sin tener que buscarlo entre cientos de líneas de texto de resultados exitosos en el archivo `.out`.

Ejemplo de uso
```
#SBATCH --output=resultado.out
#SBATCH --error=error.err
```


## Nombrar los archivos de salida y error usando variables de entorno

Si corres el mismo script varias veces y siempre le llamas a tu archivo de salida `resultado.out`, cada nueva ejecución sobreescribirá la anterior y perderás tus datos históricos. Para evitar esto, Slurm nos permite usar **variables** en nuestras directivas, las cuales el sistema reemplazará por valores reales al momento de ejecutar el trabajo.

La variable más importante y utilizada es **`%j`**, la cual representa el [[Uso del Cluster#Job ID|Job ID]]

Si en tu script configuras la siguiente directiva:

```
#SBATCH --error=error_%j.err
```

Y al momento de encolarlo, Slurm decide que tu trabajo es el número **10452**[^3], el sistema creará automáticamente un archivo llamado: `error_10452.err`

De esta forma, cada vez que envíes tu tarea, se generará un archivo con un nombre único, permitiéndote llevar un registro ordenado de todas tus ejecuciones.

**¿Qué pasa si no declaro estas directivas?**
Si olvidas poner las directivas `--output` y `--error`, Slurm no descartará tu información. Por defecto, el sistema creará un único archivo llamado `slurm-%j.out` (por ejemplo, `slurm-10452.out`) en el directorio desde donde lanzaste el trabajo. Este archivo contendrá tanto los mensajes de salida (stdout) como los errores (stderr) mezclados.

>Las variables de entorno son una parte fundamental de Slurm, échale un vistazo al [conjunto de variables de entorno que puedes ocupar en tus scripts en la documentación oficial.](https://slurm.schedmd.com/sbatch.html#SECTION_FILENAME-PATTERN)

**Otras variables útiles** 
Además de `%j`, existen otras variables que puedes combinar para nombrar tus archivos de manera aún más descriptiva:

- **`%x` (Job Name):** Se sustituye por el nombre que le diste a tu trabajo en la directiva `--job-name`.
    - Ejemplo: `resultado_%x_%j.out` se convertiría en `resultado_prueba_10452.out`.
- **`%N` (Node Name):** Se sustituye por el nombre del nodo principal que ejecutó el trabajo (por ejemplo, `node2`). Es útil para rastrear si un error solo ocurre en un equipo en particular.
- **`%u` (User):** Se sustituye por tu nombre de usuario en el sistema.

---
# Comandos esenciales
Para interactuar con el sistema de colas del clúster, Slurm nos proporciona un conjunto de herramientas de línea de comandos. A continuación, documentamos el flujo de trabajo esencial y las directivas necesarias para solicitar recursos y ejecutar tareas en el **InSAR lab**.

## `sbatch`
> A detalle: [[Comando sbatch]] · Cómo escribir el archivo: [[Archivo de trabajo y directivas SBATCH]]

El comando `sbatch` es la **herramienta principal que utilizarás**. Sirve para enviar un script de trabajo a la cola de Slurm para su futura ejecución. Cuando ejecutas este comando, Slurm lee el archivo, reserva los recursos solicitados y te devuelve un número de identificación único (Job ID).

Para solicitar recursos, el script de bash debe contener líneas especiales al inicio que comiencen con `#SBATCH`.[^4] Aunque para bash estas líneas parecen comentarios, para Slurm son directivas de configuración obligatorias.

![[Ejemplos de script usando sbatch de Slurm]]


> [!important]
> La directiva `--gres=gpu:l4:1` es obligatoria cuando se utiliza la partición `n2gpu`, ya que le indica a Slurm que tu script requiere acceso físico a la tarjeta gráfica para poder ejecutar tu código, en este caso, el script de Python `inferencia.py`.

>Puedes revisar [[Tabla de directivas más usadas|las directivas más usadas]]
>O en su defecto, revisar [la totalidad de las directivas en la documentación oficial de slurm](https://slurm.schedmd.com/sbatch.html#SECTION_OPTIONS)

## `squeue`
> A detalle: [[Comando squeue]]

Una vez que has enviado tu trabajo con `sbatch`, querrás saber su estado. El comando `squeue` te permite visualizar la fila actual de trabajos en el clúster.

Al ejecutar simplemente `squeue` en la terminal, observarás una tabla con información valiosa:

- **JOBID:** El número de identificación de los trabajos.
- **PARTITION:** En qué partición están formados.
- **NAME:** El nombre que se le asignó con `--job-name`.
- **USER:** El usuario dueño del trabajo.
- **ST:** El estado actual del trabajo (`R` para _Running_ o ejecutándose, `PD` para _Pending_ o en espera de recursos).
- **NODELIST:** El nodo de cómputo exacto donde se está ejecutando la tarea (ej. `node2`).

Si deseas ver únicamente tus propios trabajos, puedes filtrar la salida usando tu nombre de usuario:

```bash
squeue -u <tu_usuario>
```

## `scancel`
> A detalle: [[Comando scancel y scontrol]]

Es completamente normal darse cuenta de que se ha cometido un error en el código fuente después de haber encolado el trabajo. Para detener la ejecución y liberar los recursos del clúster, utilizamos el comando `scancel` seguido del identificador del trabajo (**JOBID**) que nos proporcionó `sbatch` o que consultamos con `squeue`.

Al ejecutar este comando, Slurm interrumpirá el trabajo inmediatamente, deteniendo los procesos en el nodo de cómputo y marcando el trabajo como cancelado. Si el trabajo estaba escribiendo en sus archivos de `.out` o `.err`, la escritura se detendrá en ese punto exacto.


```bash
scancel 10452
```

## `sinfo`
> A detalle: [[Comando sinfo]] · ¿Qué nodo elijo? [[Grupos de nodos y particiones]]

El comando `sinfo` es una de las herramientas más importantes para entender el estado operativo de la infraestructura antes de enviar un trabajo. Mientras que `squeue` nos muestra el estado de las tareas de los usuarios, `sinfo` nos revela la salud y disponibilidad del hardware real del clúster.

Al ejecutar `sinfo` en nuestra terminal, el sistema nos devuelve una estructura organizada por filas que agrupa los nodos según su pertenencia a las particiones y su estado actual:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
all*         up   infinite      1    mix node1
all*         up   infinite      1   idle master
extra        up   infinite      1  down* node3
extra        up   infinite      2   idle node[2,4]
n2gpu        up   infinite      1   idle node2
```


### Significado de las Columnas

Para interpretar correctamente este reporte, es necesario desglosar cada uno de los encabezados que genera Slurm:
- **`PARTITION`**: Indica el nombre de la partición lógica del clúster.
    - _Nota sobre el símbolo asterisco (`*`):_ Si observas un asterisco junto al nombre (como `all*`), significa que esa es la **partición por defecto** del sistema. Si un miembro del laboratorio envía un script con `sbatch` sin incluir la directiva `--partition`, Slurm lo asignará automáticamente a esta cola.
- **`AVAIL`**: Muestra la disponibilidad de la partición para recibir nuevos trabajos. El valor `up` indica que la partición está activa y operativa. Si el administrador apagara una cola para mantenimiento masivo, este valor cambiaría a `down`.
- **`TIMELIMIT`**: El tiempo máximo de ejecución permitido para cualquier trabajo enviado a esa partición. En nuestro clúster el valor es `infinite`, lo que significa que el laboratorio no tiene configurada una política de interrupción por tiempo máximo, permitiendo procesamientos extensos.
- **`NODES`**: Es el recuento de nodos físicos que comparten exactamente la misma partición y el mismo estado operativo en esa fila específica.
- **`STATE`**: Indica la condición operativa actual en la que se encuentran los nodos de esa fila. Es la columna crucial para el diagnóstico.
- **`NODELIST`**: Lista los nombres específicos de los equipos que se encuentran bajo las condiciones de esa fila. Slurm utiliza una notación compacta entre corchetes para agrupar nodos; por ejemplo, `node[2,4]` representa de forma abreviada al `node2` y al `node4`.


### Catálogo de Estados de los Nodos (`STATE`)
![[Catalogo de estados de los nodos en sinfo]]

#### El Símbolo Asterisco (`*`) en la columna STATE

Un detalle de seguridad crítico que se puede observar en el reporte del clúster es el estado **`down*`** (con un asterisco al final) en el `node3`.

En la arquitectura de Slurm, un asterisco añadido al final de cualquier estado operativo significa que **el nodo no está respondiendo a las comunicaciones del nodo maestro** (_unresponsive_). Cuando el nodo maestro intenta enviar señales de control o verificar la salud del equipo y no recibe respuesta en un tiempo límite, el software marca el nodo automáticamente con esta bandera.

Las causas más comunes para que un nodo entre en estado `down*` o similar son:

1. **Fallo de Red:** Pérdida de conectividad física o de enrutamiento entre el nodo de cómputo y el nodo maestro.
2. **Servicio Caído:** El demonio secundario de Slurm (`slurmd`) se detuvo o experimentó un fallo dentro del nodo de cómputo, provocando que deje de reportarse con el maestro, aunque la computadora siga encendida.
3. **Congelamiento de Sistema:** El sistema operativo (_Rocky Linux_) del nodo de cómputo sufrió un colapso debido a una sobrecarga extrema o un fallo de hardware interno, requiriendo un reinicio físico por parte del administrador.

Como usuario del laboratorio, ver un nodo en `down*` te indica que cualquier trabajo que requiera obligatoriamente ese equipo se mantendrá en estado pendiente (`ReqNodeNotAvail`) hasta que el servicio sea reestablecido.

> [!info] Para administradores
> Cómo diagnosticar y regresar un nodo al servicio: [[Recuperar nodos y servicios de Slurm]].


---

# Comandos en tiempo real
Además de los comandos para enviar trabajos en segundo plano (como `sbatch`) y revisar la cola actual (`squeue`), Slurm ofrece herramientas para interactuar con el clúster en tiempo real y para consultar el historial de todo lo que se ha ejecutado. A continuación, integraremos estas herramientas al manual.

## Comandos de Interacción Directa: `salloc` y `srun`

Aunque la forma preferida y más eficiente de usar el clúster es mediante scripts automatizados con `sbatch`, habrá momentos durante el desarrollo de un proyecto en los que necesites hacer pruebas rápidas, depurar código o compilar software directamente en un nodo de cómputo. Para esto, utilizamos asignaciones interactivas.

### `salloc` (Allocation)

El comando `salloc` se utiliza para reservar recursos del clúster en tiempo real sin enviar un script. Al ejecutarlo, le pides a Slurm que te asigne núcleos, memoria o GPUs de forma inmediata. Si los recursos están disponibles, Slurm te otorgará la asignación; de lo contrario, tu terminal se quedará esperando hasta que se liberen.

Ejemplo de uso:

```bash
salloc --nodes=1 --ntasks=4 --partition=extra
```
>Este comando reserva los recursos, pero normalmente tu terminal seguirá estando en el nodo maestro. Para saltar al nodo asignado y usar esos recursos, requieres el siguiente comando

### `srun` (Run)

`srun` es el comando encargado de lanzar procesos en los recursos que Slurm te ha asignado. Tiene dos usos principales:

1. **Dentro de un script de `sbatch`:** Se utiliza para lanzar pasos de trabajo paralelos (job steps), especialmente en aplicaciones que usan múltiples nodos (como MPI).
2. **De forma interactiva:** Combinado con una asignación previa, o ejecutándolo por sí solo para obtener una terminal dentro del nodo de cómputo.

Para obtener una sesión interactiva en un nodo de cómputo y poder escribir comandos manualmente sin afectar al maestro, se utiliza:

```BASH
srun --pty /bin/bash
```
Esto te transportará directamente a la terminal del nodo asignado. Cuando termines tus pruebas, simplemente escribes `exit` para liberar los recursos.

## Monitoreo Histórico y Códigos de Estado `sacct`
> A detalle: [[Comando sacct]]
El comando `squeue` es excelente, pero tiene una limitación: en el momento en que tu trabajo termina (ya sea exitosamente o por un error), desaparece de la cola. Si dejaste un trabajo corriendo durante la noche y al día siguiente ya no está en `squeue`, ¿cómo sabes qué ocurrió con él?

Aquí entra el comando **`sacct`** (Slurm Accounting). Este comando consulta la base de datos histórica del clúster y te permite ver el estado final, el tiempo que tardó y los recursos consumidos de cualquier trabajo pasado.

### El ciclo de vida de un trabajo (Ejemplo de uso)

Imaginemos que has escrito tu script y decides encolarlo.

```bash
$ sbatch mi_modelo.sh
Submitted batch job 10500
```

Unas horas después, notas que el trabajo ya no está activo. Para consultar su estado, usas `sacct` seguido del Job ID y le pides un formato específico de columnas para facilitar la lectura:

```bash
$ sacct -j 10500 --format=JobID,JobName,Partition,State,ExitCode
```

La salida podría verse algo así:

```
JobID          JobName  Partition      State ExitCode 
------------ ---------- ---------- ---------- -------- 
10500        mi_modelo       extra    FAILED      1:0
```

Para entender por qué tu trabajo terminó de esa manera, Slurm utiliza tres tipos de códigos estandarizados que debes saber interpretar: Códigos de Estado, Códigos de Razón y Códigos de Salida.

# Códigos de Estado (Job State Codes)
Nos indican en qué fase del ciclo de vida se encuentra el trabajo o cómo terminó. Estos aparecen en la columna `State` de `sacct` o `ST` en `squeue`.
![[Estados de un Job (Job State Codes)]]
# Códigos de Razón (Job Reason Codes)
Cuando revisas `squeue` y ves que tu trabajo lleva mucho tiempo en estado `PENDING` (PD), la columna `NODELIST(REASON)` te dirá el porqué. Slurm te explica qué está deteniendo la ejecución.

![[Codigos de razon de un Job (Job Reason Codes)]]

# Códigos de Salida (Job Exit Codes)[^5]

En la columna `ExitCode` de `sacct`, verás números en el formato `Código:Señal` (ej. `0:0` o `1:0`).

Slurm no genera estos códigos; son los códigos de salida estándar del sistema operativo Linux y de la aplicación que estás corriendo (como Python o bash).

- **Exit Code 0:** Significa éxito absoluto. Si tu script de Python termina correctamente, envía un 0 al sistema, y Slurm marca el trabajo como `COMPLETED`.
- **Exit Code distinto de 0 (ej. 1, 2, 255):** Significa error. Puede ser un error de sintaxis en tu código, que no encontró un archivo, o que una librería falló. Si esto pasa, Slurm marca el trabajo como `FAILED`. Para descubrir exactamente qué falló, deberás revisar el archivo de texto `.err` que configuraste en tus directivas iniciales.

> [!info]
> [[Tabla de comandos de Slurm|Resumen en forma de tabla de los comandos vistos]]


---



[^2]: Refiriéndose a [[Uso del Cluster#Las directivas de recursos|las directivas de recursos]] y [[Uso del Cluster#Los comandos de ejecución|los comandos de ejecición]] respectivamente

[^3]: A manera de ejemplo, pero cada nuevo trabajo generado tiene un distinto JID

[^4]: [[Uso del Cluster#Las directivas de recursos|Directivas de recursos]]

[^5]: Puedes consultar la [documentación oficial](https://slurm.schedmd.com/job_exit_code.html) para saber más sobre los Job Exit Codes
