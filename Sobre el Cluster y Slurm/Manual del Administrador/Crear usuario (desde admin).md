> [!warning] Método anterior (manual)
> El procedimiento **vigente y verificado** para crear usuarios es con Ansible: [[Uso de Clúster ADMIN]]. Esta nota se conserva como referencia de lo que el *playbook* hace por dentro.
> Nota: los nodos se llaman `node1`…`node4` (no `nodo1`); en los comandos de abajo usa `ssh node<n>`.

> [!attention]
> Las siguientes indicaciones las debe ejecutar un usuario administrador de INSARLAB

Para crear un usuario $A$, y que este usuario $A$ pueda usar [[Que es Slurm|slurm]] para manejar el [[Arquitectura del Cluster del InSAR lab|cluster]], se deben seguir los siguientes pasos:

- Ingresar al cluster como el usuario `admin`[^1]
- [[#Crear al usuario desde `admin`|Crear al usuario]]

---

> [!important]
> En esta parte del manual, se plantea como ejemplo crear al usuario `usuarioA`, en todos los comandos siguientes se ocupa por ende, al usuario `usuarioA`.


## Crear al usuario desde `admin`
- Ingresar como `admin` al cluster
- Crear el usuario con
```bash
sudo useradd -K UID_MIN=5000 -K UID_MAX=5999 -m -d /mnt/beegfs/insarlab/usuarioA -g 5000 -s /bin/bash usuarioA
```

> [!important] 👀 
> Notese que en el anterior comando, se debe reemplazar `usuarioA` por el nombre que se le quiera poner al usuario que se está creando. 

Por ejemplo si se crea un usuario llamado `luis`, el comando anterior debería quedar:
```bash
sudo useradd -K UID_MIN=5000 -K UID_MAX=5999 -m -d /mnt/beegfs/insarlab/luis -g 5000 -s /bin/bash luis
```

## Anotar el id del usuario recién creado
Si el usuario creado tuvo por nombre `luis`
- Ejecutar `id luis`, regresará algo como lo siguiente
```
uid=5006(luis) gid=5000(insarlab) . . .
```

> [!warning]
> Se debe anotar o recordar el valor numérico de `uid`, ese es el id del usuario que se acaba de crear. En el ejemplo de arriba, dicho número es el `5006`

## Reflejar creación en cada nodo

En cada uno de [[Arquitectura del Cluster del InSAR lab#^fb106f|los nodos de cómputo (4)]]
- Ingresar a ellos, partiendo desde `admin`:
```bash
[admin@master ~]$ ssh nodo<n>
```
>$_\text{Se muestra el comando junto al prompt}$

> [!example]
> Si se quiere acceder al nodo 3:
> ```
> ssh nodo3
> ```

> [!hint]
> En el comando a continuación, reemplazar `<UID>` por el [[#Anotar el id del usuario recién creado|uid del usuario que se creo previamente]], así como `usuarioA` por el nombre del usuario creado.
> ```bash
> sudo useradd -M -u <UID> -g 5000 -d /mnt/beegfs/insarlab/usuarioA -s /bin/bash usuarioA
> ```
>La bandera `-M` hace que en ese nodo no intente crear una carpeta local, pues como BeeGFS ya está montado en los nodos y por ende ya existe dicha carpeta. 
>**La creación del usuario desde un nodo solo asegura que el nodo conozca de la existencia de dicho usuario y no haya errores de permisos**

### Alta en el gestor de trabajos 
```bash
sudo sacctmgr add user usuarioA account=insarlab_users
```

## Generar llaves para ingresar al cluster

#### Verificar que no exista una llave ya existente

#### Crear llave
En caso de que no haya una llave existente perteneciente al `usuarioA`
```bash
ssh-keygen -t ed25519 -C "usuario@correo.com"
```

>Modificar el correo `usuario@correo.com` por el correo real del `usuarioA`

Esto generará 2 archivos:
- `id_ed25519`
- `id_ed25519.pub`

```bash
cat ~/.ssh/id_ed25519.pub
```

Ese texto (que empieza con `ssh-ed25519...`) es lo único que debe compartirse. Su llave privada nunca debe salir del equipo local.

#### Crear carpeta oculta `.ssh`
```BASH
sudo mkdir -p /mnt/beegfs/insarlab/usuarioA/.ssh
```

Guardar la llave publica del usuario
```BASH
echo "PEGAR_EL_TEXTO_DE_LA_LLAVE_AQUI" | sudo tee /mnt/beegfs/insarlab/usuarioA/.ssh/authorized_keys
```

```bash
sudo chmod 700 /mnt/beegfs/insarlab/usuarioA/.ssh
sudo chmod 600 /mnt/beegfs/insarlab/usuarioA/.ssh/authorized_keys
```

### Asignarle propiedad al usuario
Pues se creo usando `sudo` (`root`)

```bash
sudo chown -R <UID_DEL_USUARIO>:5000 /mnt/beegfs/insarlab/usuarioA/.ssh
```
Se tiene que sustituir `<UID_DEL_USUARIO>` por el número que regrese:
```
id <NOMBRE_USUARIO>
```



---
## Crear usuario en la Base de Datos de [[Que es Slurm]]

==Aún no confirmado==:
```
sudo sacctmgr add user usuarioA account=insarlab_users
```

---




[^1]: Con el comando `ssh admin@132.248.59.7` e ingresando la contraseña proporcionada por el equipo de trabajo InSAR
