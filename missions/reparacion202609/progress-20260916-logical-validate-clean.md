# Progreso — validación lógica RMAN post-MOVE

Fecha: 2026-09-16

Estado: **validación lógica limpia; AWR sigue temporalmente pausado.**

## Validación ejecutada

En RMAN se ejecutó:

```text
VALIDATE CHECK LOGICAL DATAFILE 2;
```

Resultado relevante:

```text
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
2    OK     4              23492        156174          5967449747

Block Type Blocks Failing Blocks Processed
Data       0              49113
Index      0              51439
Other      0              32116
```

Conclusiones:

- `sysaux01.dbf` continúa con `File Status = OK`.
- `Blocks Failing = 0` para Data, Index y Other.
- No se detectaron fallos físicos ni lógicos en bloques activos.
- Las cuatro marcas `NOLOGGING` continúan registradas como `Marked Corrupt = 4`.
- Previamente se confirmó que los bloques 76214, 76228, 76269 y 76273 ya no pertenecen a ningún extent después del `MOVE LOB`.
- Por tanto, no ejecutar `BLOCKRECOVER` ni `RECOVER CORRUPTION LIST` sobre esos bloques.

## Estado del objeto reparado

El `MOVE LOB` ya fue confirmado físicamente por cambio de `DATA_OBJECT_ID` y cabeceras:

```text
WRH$_SQL_PLAN       DATA_OBJECT_ID 658559, HEADER_BLOCK 99546
LOB OTHER_XML       DATA_OBJECT_ID 658560, HEADER_BLOCK 99522
WRH$_SQL_PLAN_PK    DATA_OBJECT_ID 6216,   HEADER_BLOCK 4290, VALID
```

Los cuatro bloques antiguos quedaron fuera de todos los extents.

## Próximo paso

1. Restaurar AWR a intervalo de 60 minutos preservando la retención actual de 8 días:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>60);
```

2. Verificar:

```sql
select snap_interval,retention from dba_hist_wr_control;
```

3. Solo después de confirmar `SNAP_INTERVAL = +00000 01:00:00.0`, crear un snapshot manual de prueba:

```sql
exec dbms_workload_repository.create_snapshot();
```

4. Revisar `DBA_HIST_SNAPSHOT` y el `alert_orcl.log` desde el momento de la prueba. El criterio funcional de éxito es que el snapshot termine sin `ORA-01578`, `ORA-01110` ni `ORA-26040`.
