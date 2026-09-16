# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Punto exacto de continuación

Oracle `orcl` en la VM `kanela` está `OPEN`, `ACTIVE`, `READ WRITE`, pero la tarea AWR `MMON_SLAVE / Auto-Flush Slave Action` sigue fallando al ejecutar `INSERT INTO wrh$_sql_plan` por `ORA-01578` / `ORA-26040` en el bloque 76214 del datafile 2 (`SYSAUX`). El bloque pertenece al LOB BasicFile `SYS.WRH$_SQL_PLAN.OTHER_XML`.

El LOB actual:

```text
SEGMENT_NAME  SYS_LOB0000006213C00038$$
INDEX_NAME    SYS_IL0000006213C00038$$
TABLESPACE    SYSAUX
CHUNK         8192
RETENTION     900
CACHE         NO
LOGGING       YES
IN_ROW        YES
SECUREFILE    NO
SIZE          ~19 MB
```

`WRH$_SQL_PLAN` no está particionada.

Índices actuales:

```text
SYS_IL0000006213C00038$$   LOB     VALID   SYSAUX   UNIQUE
WRH$_SQL_PLAN_PK           NORMAL  VALID   SYSAUX   UNIQUE
```

No se ha ejecutado todavía ningún `ALTER TABLE ... MOVE LOB`.

## Modo de archivado

`ARCHIVE LOG LIST`:

```text
Database log mode           No Archive Mode
Automatic archival          Disabled
Archive destination         USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence  15540
Current log sequence        15542
```

## Tamaño de la base

Total de datafiles: `57.51 GB`.

Datafiles relevantes:

```text
1  /home/oracle/app/oracle/oradata/orcl/system01.dbf    1.16 GB
2  /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf   1.19 GB
3  /home/oracle/app/oracle/oradata/orcl/undotbs01.dbf 13.06 GB
4  /home/oracle/app/oracle/oradata/orcl/users01.dbf    ~0 GB reportado por redondeo
5  /home/oracle/app/oracle/oradata/orcl/example01.dbf   0.10 GB
6  /ciruelas/oradata/tablas                            32.00 GB
7  /ciruelas/oradata/amada.dbf                        10.00 GB
```

Espacio conocido en la VM:

```text
/          23 GB libres
/ciruelas  51 GB libres
```

## Punto de retorno

El usuario confirmó que **ya existe un respaldo de la VM `kanela` y que es posible volver atrás en caso necesario**. Ese respaldo se toma como punto de retorno para esta reparación. Por tanto, no es necesario crear ahora otro backup local de 57.51 GB antes del DDL, siempre que se conserve ese respaldo hasta finalizar y validar la reparación.

## Espacio y tamaño de los segmentos implicados

Consulta realizada:

```sql
select segment_name,segment_type,round(bytes/1024/1024,2) mb
from dba_segments
where owner='SYS'
  and segment_name in ('WRH$_SQL_PLAN','WRH$_SQL_PLAN_PK','SYS_LOB0000006213C00038$$','SYS_IL0000006213C00038$$')
order by segment_type,segment_name;
```

Resultado:

```text
WRH$_SQL_PLAN_PK             INDEX       9.00 MB
SYS_IL0000006213C00038$$     LOBINDEX    0.19 MB
SYS_LOB0000006213C00038$$    LOBSEGMENT 19.00 MB
WRH$_SQL_PLAN                TABLE      17.00 MB
```

Espacio libre en `SYSAUX`:

```sql
select round(sum(bytes)/1024/1024,2) free_mb
from dba_free_space
where tablespace_name='SYSAUX';
```

Resultado:

```text
FREE_MB = 79.31
```

Conclusión: existe margen suficiente en `SYSAUX` para crear un nuevo segmento LOB de ~19 MB sin ampliar el tablespace.

## AWR temporalmente pausado

Se ejecutó:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>0);
```

Verificación posterior:

```sql
select snap_interval,retention from dba_hist_wr_control;
```

Resultado:

```text
SNAP_INTERVAL  +40150 00:00:00.0
RETENTION      +00008 00:00:00.0
```

Interpretación operativa: AWR quedó temporalmente deshabilitado; la retención continúa en 8 días. Mantener AWR pausado hasta terminar la recreación y validación inicial del LOB.

## Parámetro `DB_SECUREFILE`

Consulta:

```sql
select name,value from v$parameter where name='db_securefile';
```

Resultado:

```text
NAME           VALUE
db_securefile  PERMITTED
```

Implicación: Oracle permite SecureFile, pero no lo fuerza. El LOB afectado actual es `SECUREFILE=NO` (BasicFile). La reparación debe preservar explícitamente el formato BasicFile para evitar una conversión implícita no deseada durante el `MOVE LOB`.

## Estrategia de reparación prevista

Objetivo: recrear solamente el segmento LOB afectado de `SYS.WRH$_SQL_PLAN.OTHER_XML`, sin mover innecesariamente la tabla base ni tocar datos de aplicación.

Secuencia prevista:

1. AWR ya está temporalmente pausado.
2. Ejecutar un `ALTER TABLE ... MOVE LOB (OTHER_XML) STORE AS BASICFILE ...` manteniendo el LOB en `SYSAUX` y recreando un nuevo segmento LOB/LOBINDEX.
3. Confirmar que cambió el `SEGMENT_NAME` / `INDEX_NAME` del LOB.
4. Confirmar que `WRH$_SQL_PLAN_PK` sigue `VALID`; reconstruirlo solo si realmente fuera necesario.
5. Comprobar `V$DATABASE_BLOCK_CORRUPTION`.
6. Ejecutar `RMAN VALIDATE DATAFILE 2` y `VALIDATE CHECK LOGICAL DATAFILE 2`.
7. Restaurar el intervalo AWR a 60 minutos.
8. Crear un snapshot AWR de prueba.
9. Revisar `alert_orcl.log` desde el momento de la reparación para confirmar que no reaparecen `ORA-01578`, `ORA-01110` ni `ORA-26040`.

## Punto exacto para continuar

La reparación está detenida justo antes del DDL que recreará el LOB.

Estado confirmado:

```text
AWR:                  pausado
SNAP_INTERVAL:        +40150 00:00:00.0
RETENTION:            8 días
DB_SECUREFILE:        PERMITTED
LOB actual:           BasicFile
LOB tamaño:           ~19 MB
SYSAUX libre:         79.31 MB
WRH$_SQL_PLAN:        no particionada
LOB index:            VALID
WRH$_SQL_PLAN_PK:     VALID
Respaldo VM:          disponible y restaurable
```

Siguiente acción: validar y ejecutar la sintaxis exacta del `MOVE LOB` para Oracle 11.2.0.1, preservando explícitamente `BASICFILE` y `TABLESPACE SYSAUX`.

## No ejecutar sin control

- `BLOCKRECOVER`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- movimiento de la tabla completa `WRH$_SQL_PLAN`
