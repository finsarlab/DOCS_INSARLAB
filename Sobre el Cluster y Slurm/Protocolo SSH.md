---
title: El protocolo SSH
tags:
  - ssh
  - conexion
---

**SSH** (*Secure Shell*) es el protocolo con el que tu computadora abre una terminal **cifrada** en otra máquina a través de la red. Es la única puerta de entrada al clúster del InSAR lab.

## Cómo funciona la autenticación con llaves
En lugar de una contraseña, el clúster usa un **par de llaves**:
- **Llave privada** (`insar_key`): la tienes **tú** en tu computadora. Es como la llave física de una puerta: **nunca se comparte ni se sube a ningún lado**.
- **Llave pública**: vive en el clúster, en `~/.ssh/authorized_keys` de tu usuario. Solo sirve para comprobar que quien se conecta tiene la privada.

Cuando haces `ssh`, el servidor reta a tu computadora a demostrar que tiene la llave privada correspondiente, sin que la llave viaje por la red.

La configuración paso a paso está en [[Creacion de Usuario]] y [[Conexion al cluster]].

## Comandos útiles

```bash
ssh insarTuNombre                 # conectarte (usa el Host de ~/.ssh/config)
ssh -v insarTuNombre              # modo detallado: muestra qué falla al conectar
exit                              # cerrar la sesión
```

## Copiar archivos entre tu computadora y el clúster
Se ejecutan **en tu computadora**, no en el clúster:

```bash
# De tu computadora → al clúster
scp datos.zip insarTuNombre:~/proyectos/volcan/
rsync -avP carpeta_local/ insarTuNombre:~/proyectos/volcan/datos/

# Del clúster → a tu computadora
scp insarTuNombre:~/proyectos/volcan/resultados/mapa.png .
rsync -avP insarTuNombre:~/proyectos/volcan/resultados/ ./resultados/
```
`rsync` es preferible para carpetas grandes: solo copia lo que cambió y puede reanudarse si se corta (`-P`).

## Errores comunes

| Mensaje | Causa | Solución |
|---|---|---|
| `UNPROTECTED PRIVATE KEY FILE` | Permisos de la llave demasiado abiertos | `chmod 600 ~/.ssh/insar_key` (ver [[Creacion de Usuario]]) |
| `Permission denied (publickey)` | Llave equivocada, usuario equivocado o la llave pública no está en el clúster | Revisa `User` e `IdentityFile` en `~/.ssh/config`; prueba con `ssh -v` |
| `Load key ... invalid format` | La llave se pegó incompleta o sin salto de línea final | Vuelve a pegarla completa, con las líneas `BEGIN` y `END` |
| `Could not resolve hostname insarTuNombre` | El `Host` no coincide con lo que escribiste | Revisa el nombre en `~/.ssh/config` |
| `Connection timed out` | Sin red hacia el clúster (o fuera de la red permitida) | Revisa tu conexión; avisa al equipo de cómputo |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | El servidor se reinstaló o cambió de llave | Confirma con el equipo de cómputo y luego `ssh-keygen -R 132.248.59.7` |

> [!tip] Que no se corte tu sesión
> Si tu conexión se cae, los comandos que corrían en tu terminal mueren. Los trabajos de Slurm **no** se afectan porque corren en los nodos. Aun así, puedes agregar esto a tu `~/.ssh/config` para mantener viva la sesión:
> ```
> Host insarTuNombre
>     ServerAliveInterval 60
> ```
