# Validación RMAN posterior al MOVE LOB

Fecha: 2026-09-16

## Estado previo

- AWR permanece deshabilitado temporalmente (`SNAP_INTERVAL = +40150 00:00:00.0`).
- `SYS.WRH$_SQL_PLAN` y su LOB `OTHER_XML` fueron físicamente recreados mediante `MOVE LOB`.
- Los índices `SYS_IL0000006213C00038$$` y `WRH$_SQL_PLAN_PK` están `VALID`.
- Los cuatro bloques históricos `76214`, `76228`, `76269`, `76273` ya no están asignados a ningún extent en `DBA_EXTENTS`.
- `V$DATABASE_BLOCK_CORRUPTION` todavía conserva cuatro entradas `NOLOGGING` para esos bloques.

## RMAN VALIDATE DATAFILE 2 posterior al MOVE

Se ejecutó:

```text
VALIDATE DATAFILE 2;
```

Resultado:

```text
Starting validate at 16-SEP-26
input datafile file number=00002 name=/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf
validation complete, elapsed time: 00:00:01

File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
2    OK     4              23492        156174          5967449698

Block Type Blocks Failing Blocks Processed
Data       0              49113
Index      0              51439
Other      0              32116

Finished validate at 16-SEP-26
```

## Interpretación

- `File Status = OK`.
- `Blocks Failing = 0` para Data, Index y Other.
- RMAN puede leer completamente `sysaux01.dbf`.
- Las cuatro entradas `NOLOGGING` continúan contabilizadas como `Marked Corrupt = 4`.
- Dado que ya se verificó que esos cuatro bloques no pertenecen a ningún extent, son bloques antiguos/no asignados y no corresponden al nuevo LOB ni a ningún segmento activo.
- No ejecutar `BLOCKRECOVER` ni `RECOVER CORRUPTION LIST` sobre estos bloques sin nueva evidencia.

## Próximo paso

Ejecutar una última validación lógica:

```text
VALIDATE CHECK LOGICAL DATAFILE 2;
```

Después:

1. volver a consultar `V$DATABASE_BLOCK_CORRUPTION`;
2. confirmar la configuración actual de `SYS.WRH$_SQL_PLAN.OTHER_XML`;
3. si no aparecen fallos nuevos, restaurar AWR a intervalo de 60 minutos;
4. crear un snapshot AWR manual de prueba;
5. revisar `alert_orcl.log` desde el momento de la reparación y confirmar que no reaparecen `ORA-01578`, `ORA-01110` ni `ORA-26040`.
