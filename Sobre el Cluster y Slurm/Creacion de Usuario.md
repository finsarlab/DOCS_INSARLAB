---
title: Creación de Usuario
---

Tu como usuario que utiliza el Cluster, NO puedes crear tu usuario por tu cuenta, para esto *contacta con el usuario administrador*, quién te proporcionará:
- Usuario
- Llave de acceso `ssh`

Una vez que tengas ambas, sigue leyendo y realizando lo que se solicita para poder tener acceso al cluster.

---
> [!attention]
> Para este punto, debes tener:
> - ***Nombre de Usuario***
> - ***Llave de acceso*** `ssh`
> proporcionados por el administrador al crear tu cuenta


> [!important]
> SI estás usando windows, utilizar la **terminal Powershell** #Pendiente 

## Crear archivo que guarde la llave

En tu equipo local[^1], dirígete a la ruta `~/.ssh`:
```bash
cd ~/.ssh
```
En caso de no existir el directorio, crearlo con:
```bash
mkdir ~/.ssh
```

Deberás crear un archivo que guarde ***la llave*** proporcionada por el administrador. 
Aunque puede nombrarse de cualquier forma, se sugiere `insar_key`

> [!note] 
> Crea el archivo con
> ```bash
> touch insar_key
> ```
> Abre con tu editor predeterminado, por ejemplo si usas `nvim`
> ```bash
> nvim insar_key
> ```
> 
> Una vez que abra la interfaz de nvim, pega la llave en dicho archivo

El archivo `insar_key` terminaría viéndose algo así:

```
-----BEGIN OPENSSH PRIVATE KEY-----
Aquí esta tu llave
-----END OPENSSH PRIVATE KEY-----
```

Guarda el contenido del archivo.

> [!danger] Protege la llave (paso obligatorio)
> `ssh` **se niega a usar** una llave privada que otros usuarios de tu computadora puedan leer (error `UNPROTECTED PRIVATE KEY FILE`). Ajusta sus permisos:
>
> **Linux / macOS**
> ```bash
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/insar_key
> ```
> **Windows (PowerShell)**
> ```powershell
> icacls "$env:USERPROFILE\.ssh\insar_key" /inheritance:r /grant:r "$($env:USERNAME):(R)"
> ```
>
> Además, asegúrate de que el archivo termine con un **salto de línea** después de `-----END OPENSSH PRIVATE KEY-----`; si no, `ssh` puede decir `invalid format`.

## Modificar el archivo `config` para poder ingresar bajo un *HostName*

En la misma ruta (`~/.ssh/`), modificar el archivo `config` o en su defecto crearlo y escribir dentro de él:
```
Host <nombreHost>
	HostName 132.248.59.7                                
	User <usuario>
	IdentityFile ~/.ssh/<ruta de la llave>
```

Reemplazar los valores entre `<>` por tus datos:
- `<nombreHost>`: El nombre que se usará para ingresar mediante `ssh`, se sugiere usar: `InsarNombreUsuario`
- `<usuario>`: Es el usuario que te proporcionó el administrador al momento de crear tu usuario
- `<ruta de la llave>`: El nombre del archivo con tu llave, se sugirió usar `insar_key`


> [!example] Un ejemplo real del archivo `config`
> ```
> Host insarFalcon
> 	HostName 132.248.59.7
> 	User falcon
> 	IdentityFile ~/.ssh/insar_key
> ```

Una vez hecho lo anterior, ahora podrás [[Conexion al cluster|conectarte al Cluster]]



[^1]: Recuerda la [[Logica de trabajo en insarlab#Sobre la conexión|lógica de equipos]]
