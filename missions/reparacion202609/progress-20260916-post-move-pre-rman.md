# Progreso 2026-09-16 — Post MOVE LOB, antes de RMAN

## Estado

AWR continúa temporalmente pausado.

El `MOVE LOB` de `SYS.WRH$_SQL_PLAN.OTHER_XML` se ejecutó correctamente. La tabla y el LOB fueron físicamente recreados:

```text
WRH$_SQL_PLAN
  DATA_OBJECT_ID 6213 -> 658559
  HEADER_BLOCK   4266 -> 99546
  SIZE           17 MB -> ~0.06 MB

SYS_LOB0000006213C00038$$
  DATA_OBJECT_ID 6214 -> 658560
  HEADER_BLOCK   4274 -> 99522
  SIZE           19 MB -> ~0.06 MB
```

Los índices `SYS_IL0000006213C00038$$` y `WRH$_SQL_PLAN_PK` están `VALID`.

## Bloques históricos

Los bloques:

```text
76214
76228
76269
76273
```

ya no están asignados a ningún extent según `DBA_EXTENTS`.

Sin embargo, `V$DATABASE_BLOCK_CORRUPTION` todavía conserva las cuatro entradas históricas:

```text
FILE# BLOCK# BLOCKS CORRUPTION_CHANGE# CORRUPTION_TYPE
2     76214  1      3227698275         NOLOGGING
2     76228  1      3227698275         NOLOGGING
2     76269  1      3227698275         NOLOGGING
2     76273  1      3227698275         NOLOGGING
```

Esto no contradice el éxito del `MOVE`: los bloques ya están libres, pero la vista aún conserva la marca registrada anteriormente.

## Próximo paso exacto

Conectar RMAN al target y ejecutar:

```text
VALIDATE DATAFILE 2;
```

Después volver a consultar:

```sql
select file#,block#,blocks,corruption_change#,corruption_type
from v$database_block_corruption
where file#=2
order by block#;
```

Si las entradas desaparecen, continuar con validación lógica opcional/final y luego restaurar AWR.

No ejecutar `BLOCKRECOVER` ni `RECOVER CORRUPTION LIST`: los bloques ya no pertenecen a ningún segmento.