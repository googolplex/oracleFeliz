# Huella física previa al MOVE LOB

Fecha: 2026-09-16

Estado: AWR pausado temporalmente (`SNAP_INTERVAL = +40150 00:00:00.0`), retención 8 días, `DB_SECUREFILE=PERMITTED`, respaldo restaurable de la VM `kanela` disponible.

Espacio en SYSAUX:

```text
FREE_MB = 79.31
```

Tamaños de los segmentos implicados:

```text
WRH$_SQL_PLAN                    TABLE       17 MB
WRH$_SQL_PLAN_PK                 INDEX        9 MB
SYS_LOB0000006213C00038$$        LOBSEGMENT  19 MB
SYS_IL0000006213C00038$$         LOBINDEX     0.19 MB
```

Huella previa capturada con `DBA_OBJECTS` + `DBA_SEGMENTS`:

```text
SYS_LOB0000006213C00038$$
  OBJECT_TYPE    LOB
  OBJECT_ID      6214
  DATA_OBJECT_ID 6214
  TABLESPACE     SYSAUX
  SEGMENT_TYPE   LOBSEGMENT
  HEADER_FILE    2
  HEADER_BLOCK   4274
  SIZE           19 MB

WRH$_SQL_PLAN
  OBJECT_TYPE    TABLE
  OBJECT_ID      6213
  DATA_OBJECT_ID 6213
  TABLESPACE     SYSAUX
  SEGMENT_TYPE   TABLE
  HEADER_FILE    2
  HEADER_BLOCK   4266
  SIZE           17 MB

WRH$_SQL_PLAN_PK
  OBJECT_TYPE    INDEX
  OBJECT_ID      6216
  DATA_OBJECT_ID 6216
  TABLESPACE     SYSAUX
  SEGMENT_TYPE   INDEX
  HEADER_FILE    2
  HEADER_BLOCK   4290
  SIZE           9 MB
```

El LOBINDEX interno no apareció como fila independiente en la salida de la consulta de `DBA_OBJECTS`, aunque previamente se confirmó en `DBA_INDEXES`/`DBA_SEGMENTS` como `SYS_IL0000006213C00038$$`, `VALID`, `SYSAUX`, ~0.19 MB.

Verificación documental final:

- En Oracle 11gR2, `ALTER TABLE ... MOVE LOB(...) STORE AS (...)` recrea el LOB y su LOBINDEX.
- Ask TOM reproduce específicamente que este comando también mueve la tabla base y deja el índice normal/PK `UNUSABLE`; debe reconstruirse después.
- Por ello, la secuencia controlada debe ser: MOVE LOB -> REBUILD PK -> verificaciones físicas/lógicas -> restaurar AWR -> snapshot de prueba.

Punto exacto: todavía no se ha ejecutado el `MOVE LOB`.
