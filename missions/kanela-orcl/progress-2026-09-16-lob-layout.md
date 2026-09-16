# Progreso — definición del LOB `WRH$_SQL_PLAN.OTHER_XML`

Fecha: 2026-09-16

Consulta ejecutada:

```sql
select l.segment_name,l.index_name,l.tablespace_name,l.chunk,l.pctversion,l.retention,l.cache,l.logging,l.in_row,l.securefile,round(s.bytes/1024/1024,2) mb from dba_lobs l left join dba_segments s on s.owner=l.owner and s.segment_name=l.segment_name where l.owner='SYS' and l.table_name='WRH$_SQL_PLAN' and l.column_name='OTHER_XML';
```

Resultado interpretado:

- `SEGMENT_NAME`: `SYS_LOB0000006213C00038$$`
- `INDEX_NAME`: `SYS_IL0000006213C00038$$`
- `TABLESPACE_NAME`: `SYSAUX`
- `CHUNK`: `8192`
- `PCTVERSION`: nulo
- `RETENTION`: `900`
- `CACHE`: `NO`
- `LOGGING`: `YES`
- `IN_ROW`: `YES`
- `SECUREFILE`: `NO` (BasicFile LOB)
- Tamaño aproximado del segmento: `19 MB`

Contexto relacionado ya confirmado:

- El bloque 76214 del datafile 2 (`sysaux01.dbf`) pertenece a este LOB.
- `MMON_SLAVE`, acción `Auto-Flush Slave Action`, falla al ejecutar `INSERT INTO wrh$_sql_plan` con `ORA-01578` + `ORA-26040` sobre dicho bloque.
- `select count(*) from sys.wrh$_sql_plan where other_xml is not null;` devuelve `0`.
- RMAN `VALIDATE DATAFILE 2` y `VALIDATE CHECK LOGICAL DATAFILE 2` muestran `Blocks Failing = 0`, pero `V$DATABASE_BLOCK_CORRUPTION` conserva cuatro marcas `NOLOGGING` en bloques 76214, 76228, 76269 y 76273.

## Interpretación de trabajo

El LOB es BasicFile, pequeño (~19 MB), con LOGGING habilitado actualmente. El error reaparece durante el auto-flush AWR al insertar en `WRH$_SQL_PLAN`, lo que es compatible con reutilización de un bloque LOB previamente cargado con NOLOGGING.

## Regla de seguridad

No ejecutar todavía `ALTER TABLE ... MOVE LOB`, ni DDL directo sobre `SYS.WRH$_SQL_PLAN`, hasta confirmar si la tabla está particionada y si existen LOB partitions/subpartitions. La sintaxis de movimiento depende de esa topología.

## Próximo diagnóstico

Confirmar:

1. Si `SYS.WRH$_SQL_PLAN` está particionada.
2. Las entradas de `DBA_LOB_PARTITIONS`/`DBA_LOB_SUBPARTITIONS` asociadas a `OTHER_XML`.
3. Solo después diseñar una ventana breve de mantenimiento AWR y el DDL mínimo necesario para reconstruir el LOB en un segmento nuevo.
