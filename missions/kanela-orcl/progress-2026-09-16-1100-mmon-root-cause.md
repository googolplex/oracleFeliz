# Progreso — 2026-09-16 11:00 — causa inmediata identificada

## Evidencia del trace
Archivo:
`/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc`

El trace confirma:
- `SERVICE NAME: SYS$BACKGROUND`
- `MODULE NAME: MMON_SLAVE`
- `ACTION NAME: Auto-Flush Slave Action`
- error sobre `file #2 / block #76214`
- `ORA-01578`
- `ORA-01110` sobre `sysaux01.dbf`
- `ORA-26040`
- sentencia que falla: `INSERT INTO wrh$_sql_plan ...`

Fragmento clave:
```text
*** 2026-09-16 11:00:22.233
*** SERVICE NAME:(SYS$BACKGROUND)
*** MODULE NAME:(MMON_SLAVE)
*** ACTION NAME:(Auto-Flush Slave Action)
...
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
*** KEWROCISTMTEXEC - encountered error
*** SQLSTR ... INSERT INTO wrh$_sql_plan ...
```

## Interpretación actual
La corrupción no es una marca histórica inactiva. El proceso de fondo AWR/MMON vuelve a encontrar el bloque durante el auto-flush horario al insertar nuevos planes en `SYS.WRH$_SQL_PLAN`.

Esto concuerda con el hecho ya confirmado de que:
- los cuatro bloques NOLOGGING persistentes pertenecen al LOB `OTHER_XML` de `SYS.WRH$_SQL_PLAN`;
- `select count(*) from sys.wrh$_sql_plan where other_xml is not null;` devuelve 0;
- RMAN valida el datafile física y lógicamente con `Blocks Failing = 0`;
- sin embargo, al intentar reutilizar espacio del LOB durante nuevos INSERT, se vuelve a tocar el bloque 76214 y aparece ORA-26040.

## Hipótesis técnica principal
El bloque NOLOGGING puede estar en espacio libre/reutilizable del LOB. Aunque ninguna fila actual tenga `OTHER_XML` no nulo, el LOB segment conserva los bloques afectados y el auto-flush de AWR intenta reutilizarlos para nuevos valores, provocando el error.

Una nota técnica de Oracle sobre ORA-1578/ORA-26040 en LOB explica este patrón: una vez liberado un LOB corrupto, los bloques pueden quedar en la freelist y volver a provocar el error cuando el segmento intenta reutilizarlos durante INSERT/UPDATE; en ese caso el remedio es mover el LOB a un segmento nuevo. No ejecutar todavía el MOVE sin caracterizar antes las propiedades actuales del LOB y pausar AWR de forma controlada.

## Próximo paso seguro
Antes de modificar nada:
1. consultar `DBA_LOBS` para `SYS.WRH$_SQL_PLAN.OTHER_XML` y registrar `SEGMENT_NAME`, `INDEX_NAME`, `TABLESPACE_NAME`, `CHUNK`, `PCTVERSION`, `RETENTION`, `CACHE`, `LOGGING`, `IN_ROW`, `SECUREFILE`;
2. consultar tamaño del LOB en `DBA_SEGMENTS`;
3. si se decide moverlo, deshabilitar temporalmente snapshots AWR con el mecanismo soportado `DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(interval=>0)`, realizar la operación controlada, restaurar `interval=>60` y `retention=>11520`, y validar nuevamente.

## No ejecutar aún
- `TRUNCATE TABLE SYS.WRH$_SQL_PLAN`
- `DELETE` directo sobre tablas `WRH$_*`
- `BLOCKRECOVER`
- `DROP` de objetos AWR
- `ALTER TABLE ... MOVE LOB` hasta confirmar atributos y preparar rollback/validación
