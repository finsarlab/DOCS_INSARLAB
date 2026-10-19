
| Recursos                                        |     |
| ----------------------------------------------- | --- |
| [[Comandos útiles en el manejo del instalador]] |     |
## Añadir usuarios al Cluster

> [!info]
> Archivo donde se encuentra la configuración de usuarios y grupos en el clúster.
> ```
> /home/admin/ansible/roles/users/vars/main.yaml
> ```
> 
### Ingresar al usuario `doki` o `admin`
Primero se tiene que añadir los datos del usuario en el archivo
```
/home/admin/ansible/roles/users/vars/main.yaml
```
Después se tiene que posicionar en la ruta `~/ansible`
```sh
[admin@master ansible]$
```
Una vez allí ejecutar el comando:
```
ansible-playbook playbooks/users.yaml --ask-become-pass
```
Esto creará el usuario recien ingresado en el `main.yml`[^1]

Una vez creado el usuario, se generará un archivo `.ssh` el cuál podrás obtenerlo mediante:
```sh
sudo cat /mnt/beegfs/insarlab/USER/.ssh/id_ed25519
```
Este es el que tendrás que proporcionar a la persona dueña del usuario, junto con su nombre de usuario.
- Nombre de usuario
- Llave privada
 [[Creacion de Usuario|Para que pueda ingresar al cluster]]

```
minsar-cluster"·$
```

[^1]: `/home/admin/ansible/roles/users/vars/main.yaml`
