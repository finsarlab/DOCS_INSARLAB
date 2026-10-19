---
title: Códigos de razón de un Job (Job Reason Codes)
tags:
  - slurm
---

Aparecen entre paréntesis en la columna `NODELIST(REASON)` de [[Comando squeue|squeue]] cuando un trabajo está en `PD` (pendiente).

|**Código de Razón**|**Descripción**|**Acción recomendada**|
|---|---|---|
|**Resources**|Es la razón más común. Los nodos que pediste están ocupados por los trabajos de otros compañeros.|Esperar tu turno o pedir menos recursos. `squeue --start -j <ID>` estima cuándo arrancará.|
|**Priority**|Hay recursos libres (o se están liberando), pero otros trabajos en la cola tienen mayor prioridad que el tuyo.|Esperar. Tu trabajo gana prioridad conforme pasa el tiempo.|
|**Dependency**|Tu trabajo espera a que termine otro trabajo del que depende.|Verificar que el trabajo del que depende no haya fallado.|
|**DependencyNeverSatisfied**|El trabajo del que dependías falló o se canceló, así que el tuyo **nunca** arrancará.|`scancel` y volver a encolar.|
|**ReqNodeNotAvail**|Pediste (con `--nodelist`) un nodo que está apagado, en mantenimiento, drenado o reservado.|Revisar con `sinfo -R` y quitar `--nodelist` o cambiar de nodo.|
|**BeginTime**|Usaste `--begin` y aún no llega la hora.|Ninguna.|
|**JobHeldUser**|Pusiste el trabajo en pausa (`--hold` o `scontrol hold`).|`scontrol release <ID>`.|
|**JobHeldAdmin**|Un administrador lo pausó.|Preguntar al equipo de cómputo.|
|**PartitionDown**|La partición está en `down` (por ejemplo, durante un mantenimiento).|Esperar o cambiar de partición.|
|**ReqNodeNotAvail, Reserved for maintenance**|Hay una reserva de mantenimiento en esos nodos.|Esperar o reducir `--time` para que termine antes del mantenimiento.|
|**InvalidAccount** / **InvalidQOS**|Tu usuario no está dado de alta en la contabilidad de Slurm.|Avisar al admin (falta `sacctmgr add user`, ver [[Manual del Administrador]]).|
|**QOSMaxCpuPerUserLimit** / **AssocGrp…Limit**|Alcanzaste un límite de recursos por usuario o grupo (solo si el admin configuró límites).|Esperar a que terminen tus trabajos actuales.|

>Si algún código de razón no está documentado aquí, puedes consultar la [documentación oficial](https://slurm.schedmd.com/job_reason_codes.html).
