# Progreso 2026-09-16 — Preparación previa al MOVE LOB

## Estado confirmado

La base `orcl` continúa `OPEN`, `ACTIVE`, `READ WRITE` y está en `NOARCHIVELOG`.

Existe un respaldo restaurable de la VM `kanela` que cubre el entorno completo y se utilizará como punto de retorno en caso necesario. Por decisión del usuario, no se creará otro respaldo completo antes del DDL.

## Tamaño y ubicación de datafiles

Total de datafiles: **57.51 GB**.

```text
1  /home/oracle/app/oracle/oradata/orcl/system01.dbf    1.16 GB
2  /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf   1.19 GB
3  /home/oracle/app/oracle/oradata/orcl/undotbs01.dbf 13.06 GB
4  /home/oracle/app/oracle/oradata/orcl/users01.dbf    ~0 GB por redondeo
5  /home/oracle/app/oracle/oradata/orcl/example01.dbf   0.10 GB
6  /ciruelas/oradata/tablas                            32.00 GB
7  /ciruelas/oradata/amada.dbf                        10.00 GB
```

## Segmentos de `SYS.WRH$_SQL_PLAN`

```text
WRH$_SQL_PLAN                  TABLE       17.00 MB
WRH$_SQL_PLAN_PK               INDEX        9.00 MB
SYS_LOB0000006213C00038$$      LOBSEGMENT  19.00 MB
SYS_IL0000006213C00038$$       LOBINDEX     0.19 MB
```

Índices actuales:

```text
SYS_IL0000006213C00038$$   LOB     VALID   SYSAUX   UNIQUE
WRH$_SQL_PLAN_PK           NORMAL  VALID   SYSAUX   UNIQUE
```

`WRH$_SQL_PLAN` no está particionada.

## Espacio libre en SYSAUX

```text
FREE_MB = 79.31
```

Hay margen suficiente para crear un nuevo LOB de aproximadamente 19 MB en `SYSAUX`.

## Causa inmediata confirmada

El proceso `MMON_SLAVE`, acción `Auto-Flush Slave Action`, falla al ejecutar `INSERT INTO wrh$_sql_plan` y produce:

```text
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

El bloque pertenece al BasicFile LOB `SYS.WRH$_SQL_PLAN.OTHER_XML`.

`select count(*) from sys.wrh$_sql_plan where other_xml is not null;` devuelve 0, por lo que no existen LOBs activos que puedan vaciarse fila por fila. El error aparece cuando AWR intenta reutilizar espacio del LOB durante un INSERT.

## Evidencia técnica externa relevante

El patrón coincide con casos documentados de ORA-01578/ORA-26040 en LOBs marcados por NOLOGGING: después de liberar referencias, el bloque puede volver a fallar cuando se reutiliza para INSERT/UPDATE; en ese escenario se recomienda mover el LOB a un segmento nuevo.

## Próximo paso

1. Suspender temporalmente los snapshots AWR.
2. Verificar que el intervalo quedó deshabilitado.
3. Ejecutar el DDL de reconstrucción del LOB en `SYSAUX` usando sintaxis válida para Oracle 11.2.
4. Verificar nombres de segmento LOB/LOBINDEX y estado de `WRH$_SQL_PLAN_PK`.
5. Restaurar AWR a 60 minutos.
6. Generar snapshot de prueba.
7. Revisar `alert.log` y `V$DATABASE_BLOCK_CORRUPTION`.
8. Ejecutar `RMAN VALIDATE DATAFILE 2` y, si procede, `VALIDATE CHECK LOGICAL DATAFILE 2`.

## No ejecutar sin seguimiento

- `BLOCKRECOVER`
- `DROP_SNAPSHOT_RANGE`
- `DELETE`/`TRUNCATE` directo sobre `SYS.WRH$_SQL_PLAN`
- `RESETLOGS`
- recreación de controlfiles
