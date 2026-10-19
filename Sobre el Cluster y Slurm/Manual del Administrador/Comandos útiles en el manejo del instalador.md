## uid Usuarios
```
[admin@master ansible]$ id toby
uid=5007(toby) gid=5000(insarlab) groups=5000(insarlab)
```

### Ejemplo
Sean los usuarios `fersa` y `toby`:
```sh
[admin@master vars]$ id fersa
uid=5006(fersa) gid=5000(insarlab) groups=5000(insarlab)
[admin@master vars]$ id toby
uid=5007(toby) gid=5000(insarlab) groups=5000(insarlab)
```


## Comando para listar los usuarios existentes:
```sh
getent passwd | awk -F: '$3 >= 5000 && $3 <= 5999 {print $3, $1}' | sort -n
```

Usuarios al momento:
```BASH
[admin@master vars]$ getent passwd | awk -F: '$3 >= 5000 && $3 <= 5999 {print $3, $1}' | sort -n
5000 dario
5001 daruma
5002 kam
5003 erick
5004 weile
5005 falcon
5006 fersa
5007 toby
[admin@master vars]$ 
```


## Listar información de un usuario
```
getent passwd
```
Retorna información de la forma `dato_1:dato_2: ... :dato_n`
1. Nombre del usuario
2. Contraseña (aparece con `x`)
3. UID (número de identificación del usuario)
4. GID (número de identificación del grupo principal)
5. Nombre completo o descripción
6. Directorio personal (su `home`)
7. Terminal por defecto (`shell`)
### Ejemplo
Sean los usuarios: `fersa`, `toby`:
```
[admin@master vars]$ getent passwd fersa toby
fersa:x:5006:5000::/mnt/beegfs/insarlab/fersa:/bin/bash
toby:x:5007:5000::/mnt/beegfs/insarlab/toby:/bin/bash
```


