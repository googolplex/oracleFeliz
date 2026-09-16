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

## Estrategia de reparación prevista

Objetivo: recrear solamente el segmento LOB afectado de `SYS.WRH$_SQL_PLAN.OTHER_XML`, sin mover innecesariamente la tabla base.

Secuencia prevista, todavía no ejecutada:

1. Suspender temporalmente la generación automática de snapshots AWR para evitar que `MMON_SLAVE` intente insertar durante el cambio.
2. Ejecutar un `ALTER TABLE ... MOVE LOB (OTHER_XML) STORE AS ...` manteniendo el LOB como BasicFile en `SYSAUX` y conservando las características necesarias.
3. Confirmar que se creó un nuevo `SEGMENT_NAME`/`INDEX_NAME` para el LOB.
4. Confirmar que `WRH$_SQL_PLAN_PK` sigue `VALID`; reconstruirlo solo si fuera necesario.
5. Restaurar el intervalo AWR de 60 minutos.
6. Crear un snapshot AWR de prueba.
7. Revisar `alert_orcl.log` y el trace MMON.
8. Ejecutar nuevamente `RMAN VALIDATE DATAFILE 2` y consultar `V$DATABASE_BLOCK_CORRUPTION`.

## No ejecutar sin control

- `BLOCKRECOVER`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
