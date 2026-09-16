# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Estado

**REPARACIÓN AWR/LOB COMPLETADA Y VALIDADA FUNCIONALMENTE.**

Oracle `orcl` en la VM `kanela` continúa `OPEN`, `ACTIVE`, `READ WRITE`.

La causa del error repetitivo quedó aislada en AWR: `MMON_SLAVE / Auto-Flush Slave Action` fallaba durante `INSERT INTO wrh$_sql_plan` con `ORA-01578`, `ORA-01110` y `ORA-26040` sobre bloques `NOLOGGING` del datafile 2 (`SYSAUX`) pertenecientes al LOB BasicFile `SYS.WRH$_SQL_PLAN.OTHER_XML`.

## Punto de retorno

Existe un respaldo restaurable de la VM `kanela`. Puede conservarse como punto de retorno histórico de esta intervención.

## Configuración AWR original y restaurada

Configuración original:

```text
SNAP_INTERVAL  +00000 01:00:00.0
RETENTION      +00008 00:00:00.0
```

Durante la reparación se pausó AWR con:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>0);
```

Se confirmó entonces:

```text
SNAP_INTERVAL  +40150 00:00:00.0
RETENTION      +00008 00:00:00.0
```

Después de las validaciones se reactivó AWR con:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>60);
```

Verificación posterior:

```text
SNAP_INTERVAL  +00000 01:00:00.0
RETENTION      +00008 00:00:00.0
```

Por tanto AWR quedó restaurado a su funcionamiento automático de una hora con retención de ocho días.

## LOB afectado antes de la reparación

```text
TABLE          SYS.WRH$_SQL_PLAN
COLUMN         OTHER_XML
SEGMENT        SYS_LOB0000006213C00038$$
LOB INDEX      SYS_IL0000006213C00038$$
TABLESPACE     SYSAUX
CHUNK          8192
RETENTION      900
CACHE          NO
LOGGING        YES
IN_ROW         YES
SECUREFILE     NO
DB_SECUREFILE  PERMITTED
SIZE LOB       ~19 MB
```

`WRH$_SQL_PLAN` no estaba particionada.

Espacio antes del cambio:

```text
WRH$_SQL_PLAN                 17.00 MB
WRH$_SQL_PLAN_PK               9.00 MB
SYS_LOB0000006213C00038$$     19.00 MB
SYS_IL0000006213C00038$$       0.19 MB
SYSAUX libre                  79.31 MB
```

## Huella física previa

```text
WRH$_SQL_PLAN
  OBJECT_ID       6213
  DATA_OBJECT_ID  6213
  HEADER_FILE     2
  HEADER_BLOCK    4266

SYS_LOB0000006213C00038$$
  OBJECT_ID       6214
  DATA_OBJECT_ID  6214
  HEADER_FILE     2
  HEADER_BLOCK    4274

WRH$_SQL_PLAN_PK
  OBJECT_ID       6216
  DATA_OBJECT_ID  6216
  HEADER_FILE     2
  HEADER_BLOCK    4290
```

## MOVE LOB — EJECUTADO

Se ejecutó la recreación/movimiento del LOB `OTHER_XML` en `SYSAUX`, conservándolo como BasicFile.

Una entrada posterior mal concatenada en SQL*Plus produjo `ORA-00911`, pero fue solamente un error de entrada de comandos; las consultas de control demostraron que el `MOVE` se había ejecutado correctamente.

Huella posterior:

```text
SYS_LOB0000006213C00038$$
  OBJECT_ID       6214
  DATA_OBJECT_ID  658560
  HEADER_FILE     2
  HEADER_BLOCK    99522
  SIZE            0.06 MB

WRH$_SQL_PLAN
  OBJECT_ID       6213
  DATA_OBJECT_ID  658559
  HEADER_FILE     2
  HEADER_BLOCK    99546
  SIZE            0.06 MB

WRH$_SQL_PLAN_PK
  OBJECT_ID       6216
  DATA_OBJECT_ID  6216
  HEADER_FILE     2
  HEADER_BLOCK    4290
  SIZE            9 MB
```

Conclusión: tabla y LOB fueron físicamente recreados. Los `OBJECT_ID` lógicos se conservaron, mientras que los `DATA_OBJECT_ID` y bloques de cabecera de tabla/LOB cambiaron.

Índices posteriores:

```text
SYS_IL0000006213C00038$$   LOB     VALID   SYSAUX
WRH$_SQL_PLAN_PK           NORMAL  VALID   SYSAUX
```

La PK no necesitó nueva reconstrucción porque permaneció `VALID`.

## Bloques NOLOGGING antiguos

Bloques históricos:

```text
76214
76228
76269
76273
```

Una consulta contra `DBA_EXTENTS` confirmó que los cuatro quedaron **sin asignación a ningún extent** después del `MOVE`. Por tanto ya no pertenecen al LOB nuevo ni a ningún segmento activo.

`V$DATABASE_BLOCK_CORRUPTION` continúa registrándolos como `NOLOGGING`, pero se trata ahora de bloques libres/no asignados. No ejecutar `BLOCKRECOVER` sobre ellos sin nueva evidencia.

## RMAN post-reparación

### VALIDATE DATAFILE 2

Resultado:

```text
File Status Marked Corrupt Empty Blocks Blocks Examined
2    OK     4              23492        156174

Data   Blocks Failing 0
Index  Blocks Failing 0
Other  Blocks Failing 0
```

### VALIDATE CHECK LOGICAL DATAFILE 2

Resultado igualmente limpio:

```text
File Status Marked Corrupt Empty Blocks Blocks Examined
2    OK     4              23492        156174

Data   Blocks Failing 0
Index  Blocks Failing 0
Other  Blocks Failing 0
```

Conclusión: el datafile 2 es físicamente y lógicamente legible; no hay bloques activos que fallen validación. Las cuatro marcas históricas permanecen sobre bloques ya no asignados.

## Prueba funcional AWR — EXITOSA

Después de restaurar AWR a una hora se ejecutó:

```sql
exec dbms_workload_repository.create_snapshot();
```

La ejecución terminó correctamente.

Verificación objetiva posterior en `DBA_HIST_SNAPSHOT`:

```text
SNAP_ID 130356
BEGIN_INTERVAL_TIME 16-SEP-26 12.08.23.777 PM
END_INTERVAL_TIME   16-SEP-26 12.08.50.674 PM

SNAP_ID 130355
BEGIN_INTERVAL_TIME 16-SEP-26 11.00.21.748 AM
END_INTERVAL_TIME   16-SEP-26 12.08.23.777 PM

SNAP_ID 130354
BEGIN_INTERVAL_TIME 16-SEP-26 10.41.16.000 AM
END_INTERVAL_TIME   16-SEP-26 11.00.21.748 AM
```

El nuevo `SNAP_ID 130356` confirma que AWR volvió a completar correctamente el flujo de snapshot que antes fallaba en `WRH$_SQL_PLAN`.

## Limpieza / rotación del `alert_orcl.log`

El `alert_orcl.log` histórico había crecido hasta aproximadamente 541 MB.

Se realizó una rotación manual conservadora manteniendo el archivo activo:

```bash
cd /home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace
cp -p alert_orcl.log alert_orcl_20260916_pre_limpieza.log
gzip alert_orcl_20260916_pre_limpieza.log
: > alert_orcl.log
```

Después se forzó una escritura Oracle con:

```sql
alter system switch logfile;
```

El usuario confirmó que la operación terminó correctamente y que el nuevo `alert_orcl.log` siguió recibiendo escritura normal. La historia previa quedó preservada en el archivo comprimido.

## Resultado final

La reparación puede considerarse **cerrada con éxito**:

- Oracle abre y permanece `OPEN`, `ACTIVE`, `READ WRITE`;
- AWR vuelve a operar cada 60 minutos;
- retención AWR permanece en 8 días;
- el LOB problemático fue físicamente recreado;
- índices involucrados permanecen `VALID`;
- los bloques históricos `NOLOGGING` quedaron fuera de cualquier extent activo;
- `VALIDATE DATAFILE 2` y `VALIDATE CHECK LOGICAL DATAFILE 2` reportan `Blocks Failing = 0`;
- el snapshot manual AWR generó correctamente el `SNAP_ID 130356`;
- el `alert_orcl.log` histórico fue archivado/comprimido y el archivo activo fue reiniciado correctamente.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
