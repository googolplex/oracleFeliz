# Progreso — AWR reactivado, pendiente snapshot funcional

Fecha: 2026-09-16

## Estado previo consolidado

La reparación del LOB `SYS.WRH$_SQL_PLAN.OTHER_XML` fue ejecutada mediante `MOVE LOB` y se confirmó físicamente por cambio de `DATA_OBJECT_ID` y `HEADER_BLOCK` tanto en la tabla `SYS.WRH$_SQL_PLAN` como en el LOB `SYS_LOB0000006213C00038$$`.

Los cuatro bloques históricos problemáticos:

```text
76214
76228
76269
76273
```

ya no están asignados a ningún extent según `DBA_EXTENTS`.

Ambas validaciones RMAN posteriores a la reparación fueron satisfactorias:

```text
VALIDATE DATAFILE 2;
VALIDATE CHECK LOGICAL DATAFILE 2;
```

En ambas:

```text
File Status: OK
Blocks Failing Data: 0
Blocks Failing Index: 0
Blocks Failing Other: 0
Marked Corrupt: 4
```

Las cuatro entradas `NOLOGGING` permanecen registradas en `V$DATABASE_BLOCK_CORRUPTION`, pero corresponden a bloques ya no asignados a segmentos activos.

## AWR reactivado

Después de completar las validaciones iniciales se restauró el intervalo automático de AWR a 60 minutos.

Verificación:

```sql
select snap_interval,retention from dba_hist_wr_control;
```

Resultado:

```text
SNAP_INTERVAL  +00000 01:00:00.0
RETENTION      +00008 00:00:00.0
```

Por tanto:

- AWR automático está nuevamente activo;
- frecuencia: 1 hora;
- retención: 8 días;
- no se alteró la retención original.

## Punto exacto de continuación

La próxima acción es una prueba funcional controlada del mecanismo que anteriormente fallaba:

```sql
exec dbms_workload_repository.create_snapshot();
```

Después del snapshot manual se debe:

1. comprobar que se creó un nuevo `SNAP_ID` en `DBA_HIST_SNAPSHOT`;
2. revisar el `alert_orcl.log` desde el momento del snapshot;
3. confirmar ausencia de `ORA-01578`, `ORA-01110` y `ORA-26040`;
4. si la prueba es limpia, considerar la reparación funcionalmente exitosa;
5. mantener el respaldo de la VM hasta cerrar la validación final.
