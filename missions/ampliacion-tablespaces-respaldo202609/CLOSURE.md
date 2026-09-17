# Cierre de misión — ampliación de tablespaces y revisión de respaldo

Fecha: 2026-09-16

## Decisión del usuario

La base `orcl` es una base histórica que ya no se utiliza operativamente. El objetivo principal era volver a ponerla en marcha, validar que pudiera operar nuevamente y, posteriormente, liberar espacio innecesariamente ocupado por auditoría histórica y evaluar si existen datafiles sobredimensionados que puedan devolver espacio al filesystem.

### Preferencia de interacción

El usuario indicó expresamente que **puede ser tuteado** durante el trabajo del proyecto. Se puede usar trato de `tú` de forma natural en las respuestas y comandos explicados.

## Estado final / fase de optimización posterior

- Base `orcl` recuperada y operativa (`OPEN`, `ACTIVE`, `READ WRITE`).
- Listener funcional.
- Reparación AWR/LOB completada y validada.
- Datafile 2 validado física y lógicamente con `Blocks Failing = 0`.
- `SYSTEM`, `SYSAUX`, `TABLAS` y demás tablespaces diagnosticados.
- El gran volumen de auditoría histórica se explicó por imports/cargas temporales de 2019.
- Auditoría tradicional inventariada: sentencia y privilegio globales; sin auditoría por objetos.
- `DBMS_AUDIT_MGMT` está disponible y válido.
- `CLEANUP_INITIALIZED=FALSE`; no se ejecutó `INIT_CLEANUP`.
- No se modificó la política de auditoría.
- No se revisó ni modificó RMAN porque el usuario confirmó que la base es histórica y ya no se usa operativamente.

## Liberación de espacio en `SYSTEM`

`SYS.AUD$` ocupaba aproximadamente **250 MB** dentro de `SYSTEM` y contenía más de 1.2 millones de filas de auditoría acumuladas desde 2009.

Dado que el usuario confirmó que la base es histórica y que no necesitaba conservar esa auditoría extendida, se ejecutó:

```sql
TRUNCATE TABLE SYS.AUD$;
```

Resultado confirmado por SQL*Plus:

```text
Table truncated.
```

### Medición posterior al `TRUNCATE`

La consulta global de capacidad devolvió:

```text
TABLESPACE  TOTAL_MB   USED_MB    FREE_MB   PCT_USED
AMANDA      10240.00    103.44   10136.56      1.01
EXAMPLE       100.00     78.44      21.56     78.44
SYSAUX       1220.00   1107.50     112.50     90.78
SYSTEM       1190.00    933.69     256.31     78.46
TABLAS      32767.98  14491.67   18276.31     44.23
UNDOTBS1    13370.00     28.75   13341.25      0.22
USERS           5.00      4.06       0.94     81.25
```

Totales en ese momento:

- capacidad asignada en datafiles: **58.892,98 MB** (~57,51 GiB);
- espacio usado: **16.747,55 MB** (~16,36 GiB);
- espacio libre interno: **42.145,43 MB** (~41,16 GiB);
- porcentaje libre interno global: **71,56%**.

Comparación de `SYSTEM` antes/después:

- libre antes: **6,38 MB**;
- libre después: **256,31 MB**;
- espacio liberado por `TRUNCATE SYS.AUD$`: **249,93 MB**.

Este espacio quedó reutilizable dentro de Oracle. El tamaño físico de `system01.dbf` no cambió, por lo que esta acción no devolvió por sí sola esos ~250 MB al filesystem.

## Candidatos para devolver espacio al filesystem

Los mayores márgenes internos identificados fueron:

- `TABLAS`: **18.276,31 MB** libres (~17,85 GiB);
- `UNDOTBS1`: **13.341,25 MB** libres (~13,03 GiB);
- `AMANDA`: **10.136,56 MB** libres (~9,90 GiB).

No se debe inferir que todo ese espacio sea inmediatamente reducible: para devolver espacio físico al sistema operativo hay que verificar el high-water mark de cada datafile y, especialmente en `UNDOTBS1`, la distribución/estado de extents de undo.

## Diagnóstico de `UNDOTBS1`

Datafile:

```text
FILE_ID 3
/home/oracle/app/oracle/oradata/orcl/undotbs01.dbf
CURRENT_MB            13370
HWM_MB                  427
POTENTIAL_RECLAIM_MB  12943
```

La distribución observada de extents de undo fue:

```text
STATUS      EXTENTS   MB
EXPIRED          33   6.75
UNEXPIRED        14  21.00
```

No se observaron extents `ACTIVE`.

Interpretación previa a la ejecución:

- el datafile estaba fuertemente sobredimensionado para el uso actual;
- el HWM estaba en solo **427 MB**;
- no existían extents de undo activos al momento de la consulta;
- solo existían **21 MB UNEXPIRED** y **6,75 MB EXPIRED**;
- se eligió un objetivo conservador de **1024 MB (1 GiB)**, muy por encima del HWM observado.

### Reducción ejecutada con éxito

Se ejecutó:

```sql
ALTER DATABASE DATAFILE '/home/oracle/app/oracle/oradata/orcl/undotbs01.dbf' RESIZE 1024M;
```

Resultado confirmado por SQL*Plus:

```text
Database altered.
```

Consecuencia:

- `UNDOTBS1` pasó de **13.370 MB** a **1.024 MB**;
- se devolvieron aproximadamente **12.346 MB (~12,06 GiB)** al filesystem;
- la operación fue aceptada por Oracle sin error;
- el nuevo tamaño sigue dejando un margen amplio por encima del HWM observado de 427 MB.

## Diagnóstico ADR / logs del filesystem

Se consultó `V$DIAG_INFO` y se confirmó:

```text
ADR Base: /home/oracle/app/oracle
ADR Home: /home/oracle/app/oracle/diag/rdbms/orcl/orcl
Active Incident Count: 17217
Active Problem Count: 2
Diag Alert: /home/oracle/app/oracle/diag/rdbms/orcl/orcl/alert
Diag Incident: /home/oracle/app/oracle/diag/rdbms/orcl/orcl/incident
Diag Trace: /home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace
```

Luego se midió el ADR en filesystem:

```text
33M   alert
8.0K  cdump
8.0K  hm
41G   incident
8.0K  incpkg
16K   ir
168K  lck
21M   metadata
176M  stage
636K  sweep
979M  trace
```

Los homes conocidos por `adrci` son:

```text
diag/rdbms/orcl/orcl
diag/tnslsnr/kanela/listener
```

Para la base se fijó explícitamente `diag/rdbms/orcl/orcl` antes de cualquier purga.

### Política ADR actual

`show control` devolvió:

```text
SHORTP_POLICY = 720 horas   (30 días)
LONGP_POLICY  = 8760 horas  (365 días)
LAST_AUTOPRG_TIME = 2026-09-15 22:17:56 -04:00
```

Por tanto, el autopurge está activo y se ejecutó recientemente. La enorme ocupación no se explica por un mecanismo totalmente inactivo, sino por la política de retención larga y por incidentes que siguen dentro de esa ventana.

### Antigüedad de los directorios `incident`

Distribución por mes observada en filesystem:

```text
2025-09   358
2025-10   750
2025-11   743
2025-12   744
2026-01   755
2026-02   690
2026-03   695
2026-04   745
2026-05   781
2026-06   741
2026-07   790
2026-08   748
2026-09   375
```

Total de directorios observados: **8.915**. Prácticamente todo cae dentro de los últimos 12 meses, coherente con `LONGP_POLICY=8760` horas.

### Problemas registrados por ADR

`show problem` devolvió 10 problemas históricos. Los más recientes son:

```text
PROBLEM_ID 4   ORA 1578                       LAST_INCIDENT 1405071  2026-09-16 11:00:23 -04:00
PROBLEM_ID 10  ORA 353 [48544] [5946659360]  LAST_INCIDENT 1366317  2026-03-22 06:00:49 -04:00
```

Los demás corresponden a eventos históricos de 2011–2020 (`ORA-445`, `ORA-7445`, varios `ORA-600`, `ORA-3137`).

### Detalle del último `ORA-1578`

Se inspeccionó el incidente `1405071` con `adrci` y se obtuvo:

```text
INCIDENT_ID   1405071
STATUS        ready
CREATE_TIME   2026-09-16 11:00:23.415000 -04:00
PROBLEM_ID    4
ERROR_NUMBER  1578
ERROR_ARG1    ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
PROBLEM_KEY   ORA 1578
FIRST_INCIDENT 561807
FIRSTINC_TIME  2012-03-18 11:08:16.582000 -03:00
LAST_INCIDENT  1405071
LASTINC_TIME   2026-09-16 11:00:23.415000 -04:00
IMPACTS        0
```

Archivos asociados:

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_1405071/orcl_m000_3559_i1405071.trc
```

El incidente apunta a **datafile 2, bloque 76214**, es decir, al `SYSAUX` que fue objeto de la reparación AWR/LOB.

### Estado actual del bloque 2/76214

La consulta a `V$DATABASE_BLOCK_CORRUPTION` confirmó que Oracle aún registra:

```text
FILE#  BLOCK#  BLOCKS  CORRUPTION_CHANGE#  CORRUPTION_TYPE
2      76214   1       3227698275          NOLOGGING
```

Sin embargo, la búsqueda exacta en `DBA_EXTENTS` para `file_id=2` y bloque `76214` devolvió:

```text
no rows selected
```

Interpretación operativa actual:

- el bloque sigue apareciendo como `NOLOGGING` en la vista de corrupción;
- actualmente **no pertenece a ningún extent visible ni a ningún segmento asignado**;
- esto es coherente con la reparación/recreación previa del objeto afectado y reduce el riesgo operativo asociado a esa entrada residual;
- no se ejecutará `BLOCKRECOVER` sobre este bloque sin nueva evidencia que demuestre que pertenece a un objeto activo.

### Purga controlada de incidentes ADR — completada

Se decidió conservar los incidentes de los últimos **7 días** y purgar únicamente los anteriores mediante el mecanismo soportado de ADRCI:

```text
HOST $ORACLE_HOME/bin/adrci exec="set homepath diag/rdbms/orcl/orcl; purge -age 10080 -type incident"
```

`10080` corresponde a 7 días expresados en minutos.

Durante la ejecución se verificó desde otra sesión que el proceso estaba activo:

```text
oracle 8075 7679 65 21:03 pts/1 00:00:29 adrci ... purge -age 10080 -type incident
```

La purga terminó correctamente. La medición posterior fue:

```text
1013M  /home/oracle/app/oracle/diag/rdbms/orcl/orcl/incident
```

Comparación aproximada:

- `incident` antes: **~41 GB**;
- `incident` después: **1013 MB (~0,99 GiB)**;
- espacio recuperado en ese directorio: **aproximadamente 40 GB**.

No se borraron directorios manualmente con `rm`; la limpieza se realizó con ADRCI y mantuvo los incidentes de los últimos 7 días.

### Efecto final sobre el filesystem raíz

Antes de las principales acciones de liberación, `/` se encontraba aproximadamente en:

```text
102G total, 75G usados, 23G disponibles, 77% usado
```

Después de reducir `UNDOTBS1` y completar la purga ADR:

```text
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00
                      102G   23G   74G  24% /
```

Por tanto:

- espacio disponible pasó de **~23 GB a ~74 GB**;
- aumento observado de espacio disponible: **~51 GB**;
- uso del filesystem cayó de **~77% a 24%**;
- la diferencia es coherente, dentro del redondeo de `df -h`, con los **~12,06 GiB** recuperados al reducir `UNDOTBS1` más los **~40 GB** liberados del directorio ADR `incident`.

## Acciones no realizadas todavía

- no ampliar `TABLAS`;
- no reducir aún `AMANDA` ni `TABLAS`;
- no mover `AUD$`;
- no ejecutar `DBMS_AUDIT_MGMT.INIT_CLEANUP`;
- no ejecutar `NOAUDIT`;
- no cambiar `AUDIT_TRAIL`;
- no cambiar configuración RMAN;
- no eliminar respaldos;
- no modificar `AUTOEXTEND`;
- no se borraron manualmente archivos del ADR;
- no se ejecutó `BLOCKRECOVER` sobre el bloque residual 2/76214.

## Contexto histórico y lecciones de operación

Este sistema también conserva valor como ejemplo de longevidad de una plataforma tecnológica bien mantenida. `kanela` ejecuta **Oracle 11g sobre CentOS 5.11**, una combinación ya veterana que continúa arrancando, operando y permitiendo tareas de diagnóstico y recuperación muchos años después de su despliegue original.

El entorno se apoya en hardware y prácticas orientadas a la resiliencia:

- almacenamiento en **SSD configurados en RAID1**, para baja latencia y redundancia local;
- **backups automáticos abundantes entre `zapallo` y `cafe`**, independientes de la redundancia del RAID;
- uso de **fuentes de alimentación de buena calidad**;
- experiencia acumulada después de diversos problemas eléctricos;
- preferencia para servidores por hardware simple y durable, minimizando partes móviles innecesarias;
- evitar, cuando no son necesarios, dispositivos gráficos con pequeños ventiladores propios y sistemas de refrigeración líquida/hidrocoolers, privilegiando soluciones térmicas sencillas, robustas y de bajo mantenimiento.

El usuario tiene experiencia previa como **Oracle DBA** y dedicó, junto con otros profesionales de su época, una cantidad considerable de tiempo humano a aprender administración de bases de datos. Esa formación sigue influyendo directamente en la metodología aplicada actualmente: observar y diagnosticar antes de modificar, trabajar con evidencia, realizar cambios pequeños y controlados, preservar la recuperabilidad y validar siempre el resultado posterior.

La misma disciplina se reutiliza hoy en otros dominios, especialmente en proyectos **IoT y telemetría**: aislamiento estricto de dominios, identificación precisa de fuentes, Discovery antes de asignar semántica, trazabilidad, controles de consistencia y cambios reversibles. La experiencia histórica con Oracle y sistemas Linux/Unix no se considera solamente conocimiento legado, sino una base metodológica que continúa aportando valor en sistemas modernos y heterogéneos.

## Estado de la misión

**BASE RECUPERADA / LIBERACIÓN MAYOR DE ESPACIO COMPLETADA.**

La base quedó nuevamente operativa. La auditoría histórica innecesaria fue eliminada de `SYS.AUD$`, recuperando **249,93 MB internos en `SYSTEM`**; se devolvieron aproximadamente **12,06 GiB físicos** mediante la reducción controlada de `UNDOTBS1`; y la purga soportada de ADRCI redujo `incident` de **~41 GB a ~1 GB**, liberando aproximadamente **40 GB** adicionales. En conjunto, el filesystem raíz pasó de **~23 GB disponibles (77% usado)** a **~74 GB disponibles (24% usado)**, un aumento observado de aproximadamente **51 GB libres**.