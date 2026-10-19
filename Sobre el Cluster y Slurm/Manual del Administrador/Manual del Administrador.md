---
title: Manual del Administrador del clúster
tags:
  - admin
---

> [!attention] Solo para el equipo de cómputo
> Estas notas requieren acceso como `admin` (o `doki`) y privilegios de `sudo`. Si solo quieres **usar** el clúster, ve al [[Manual sobre el Cluster del InSAR lab y Slurm|manual de usuario]].

## Usuarios

| Nota                                                                                                 | Estado                                                                |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| [[Uso de Clúster ADMIN]]: alta de usuarios con **Ansible** (`ansible-playbook playbooks/users.yaml`) | 🟩**Método vigente**🟩                                                |
| [[Comandos útiles en el manejo del instalador]]: UIDs, listar usuarios, `getent passwd`              | 🟩 Verificado                                                         |
| [[Crear usuario (desde admin)]]: alta manual nodo por nodo                                           | *Método anterior*. Sirve como referencia de lo que hace el *playbook* |

> [!info] Convenciones de usuarios
> - UIDs en el rango **5000–5999**, grupo principal `insarlab` (GID **5000**).
> - `home` en BeeGFS: `/mnt/beegfs/insarlab/<usuario>`.
> - Variables de Ansible: `/home/admin/ansible/roles/users/vars/main.yaml`.

## Slurm

| Nota | Para qué |
|---|---|
| [[Archivo de configuracion slurm conf\|slurm.conf]] | Qué es, dónde está, parámetros clave y cómo aplicar cambios |
| [[Recuperar nodos y servicios de Slurm]] | Nodos `down`, `down*`, `drain`, `comp`, `inval`; servicios; mantenimiento; arranque tras apagón |

## Almacenamiento

| Nota | Para qué |
|---|---|
| [[BeeGFS]] | Qué es, revisar estado, levantarlo tras una caída |
| [[BeeGFS - Pendientes por investigar]] | Lista de tareas para documentar la arquitectura real |

## Arquitectura

| Nota | Para qué |
|---|---|
| [[Arquitectura del Cluster de InSAR lab (para admin)]] | Diagrama de red (switch, VLANs, enlaces de 1 y 10 Gb) |
| [[Arquitectura del Cluster del InSAR lab]] | Nodos, hardware y particiones (vista de usuario) |

## Guía de Problemas (qué hacer cuando hay alguno)

| Problema                                           | Ir a                                                                                                                |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Alguien nuevo necesita acceso                      | [[Uso de Clúster ADMIN]]                                                                                            |
| `sinfo` muestra un nodo `down*`                    | [[Recuperar nodos y servicios de Slurm\|Recuperar nodos]] → sección *down\**                                        |
| Un nodo en `drain` con `Low RealMemory`            | [[Recuperar nodos y servicios de Slurm\|Recuperar nodos]] → sección *drain*                                         |
| `sbatch` dice `Unable to contact slurm controller` | [[Recuperar nodos y servicios de Slurm#Mapa de servicios\|Mapa de servicios]] (`slurmctld`)                         |
| `sacct` no funciona                                | [[Recuperar nodos y servicios de Slurm#Mapa de servicios\|Mapa de servicios]] (`slurmdbd`)                          |
| `ls /mnt/beegfs` se congela                        | [[BeeGFS#Problemas frecuentes\|BeeGFS → problemas frecuentes]]                                                      |
| Hubo un apagón                                     | [[Recuperar nodos y servicios de Slurm#Arranque del clúster completo (después de un apagón)\|Arranque tras apagón]] |
| Hay que dar mantenimiento a un nodo                | [[Recuperar nodos y servicios de Slurm#Poner un nodo en mantenimiento (forma ordenada)\|Mantenimiento]]             |
| Hay que cambiar particiones o recursos             | [[Archivo de configuracion slurm conf\|slurm.conf]]                                                                 |

## Comandos de administración más usados

```bash
sinfo -R                                                    # nodos con problema y por qué
sudo scontrol update nodename=nodeX state=resume            # regresar un nodo
sudo scontrol update nodename=nodeX state=drain reason="…"  # retirar un nodo ordenadamente
sudo scontrol reconfigure                                   # releer slurm.conf
sudo sacctmgr show assoc format=account,user                # usuarios dados de alta en la contabilidad
sudo sacctmgr add user <usuario> account=insarlab_users     # alta en la contabilidad
scancel -u <usuario>                                        # cancelar todo lo de un usuario
```
