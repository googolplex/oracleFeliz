# Progreso — ADR y bloque registrado como NOLOGGING

Fecha: 2026-09-16

## Contexto

Durante la revisión del ADR se encontró que el directorio `incident` ocupa aproximadamente 41 GB y que `SHOW PROBLEM` mantiene, entre otros, el problema `ORA 1578` con un incidente creado el 2026-09-16.

El detalle del incidente `1405071` mostró:

```text
ERROR_NUMBER 1578
ERROR_ARG1 ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
PROBLEM_ID 4
PROBLEM_KEY ORA 1578
FIRSTINC_TIME 2012-03-18 11:08:16 -03:00
LASTINC_TIME 2026-09-16 11:00:23 -04:00
```

Se verificó posteriormente `V$DATABASE_BLOCK_CORRUPTION`:

```sql
SELECT file#, block#, blocks, corruption_change#, corruption_type
FROM v$database_block_corruption
WHERE file# = 2
  AND 76214 BETWEEN block# AND block# + blocks - 1;
```

Resultado:

```text
FILE#  BLOCK#  BLOCKS  CORRUPTION_CHANGE#  CORRUPTION_TYPE
2      76214   1       3227698275          NOLOGGING
```

## Interpretación actual

- el bloque sigue registrado en `V$DATABASE_BLOCK_CORRUPTION`;
- el tipo es `NOLOGGING`, no una corrupción física `CHECKSUM`/`FRACTURED`;
- no se debe asumir que el incidente ADR es puramente histórico mientras esa fila siga presente;
- antes de purgar los incidentes ADR se identificará qué segmento ocupa actualmente `file 2 / block 76214`;
- no ejecutar todavía `adrci purge` sobre incidentes.

## Siguiente diagnóstico autorizado

Consulta de solo lectura para mapear el bloque a un segmento:

```sql
SELECT owner, segment_name, segment_type, tablespace_name, file_id, block_id, blocks
FROM dba_extents
WHERE file_id = 2
  AND 76214 BETWEEN block_id AND block_id + blocks - 1;
```

Si no devuelve filas, el bloque no está actualmente asignado a ningún extent visible en `DBA_EXTENTS`; si devuelve una fila, se identificará el objeto exacto antes de decidir la purga ADR.
