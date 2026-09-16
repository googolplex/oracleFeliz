# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Estado

Oracle `orcl` en `kanela` continúa `OPEN`, `ACTIVE`, `READ WRITE`. La causa del error repetitivo fue aislada en AWR: `MMON_SLAVE / Auto-Flush Slave Action` fallaba durante `INSERT INTO wrh$_sql_plan` con `ORA-01578`, `ORA-01110` y `ORA-26040` sobre el bloque 76214 del datafile 2 (`SYSAUX`), perteneciente al LOB BasicFile `SYS.WRH$_SQL_PLAN.OTHER_XML`.

## Punto de retorno

Existe un respaldo restaurable de la VM `kanela`. Se mantiene como punto de retorno hasta completar toda la validación posterior a la reparación.

## AWR

AWR está temporalmente pausado mediante:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>0);
```

Estado confirmado:

```text
SNAP_INTERVAL  +40150 00:00:00.0
RETENTION      +00008 00:00:00.0
```

No reactivar AWR hasta terminar la validación inicial del LOB y de los bloques antiguos.

## Configuración del LOB antes de la reparación

```text
TABLE          SYS.WRH$_SQL_PLAN
COLUMN         OTHER_XML
SEGMENT        SYS_LOB0000006213C00038$$
LOB INDEX      SYS_IL0000006213C00038$$
TABLESPACE     SYSAUX
CHUNK          8192
RETENTION      900
CACHE          NO
LOGGING        YES
IN_ROW         YES
SECUREFILE     NO
DB_SECUREFILE  PERMITTED
SIZE LOB       ~19 MB
```

`WRH$_SQL_PLAN` no está particionada.

## Espacio antes del cambio

```text
WRH$_SQL_PLAN                 17.00 MB
WRH$_SQL_PLAN_PK               9.00 MB
SYS_LOB0000006213C00038$$     19.00 MB
SYS_IL0000006213C00038$$       0.19 MB
SYSAUX libre                  79.31 MB
```

## Huella física previa al MOVE

```text
WRH$_SQL_PLAN
  OBJECT_ID       6213
  DATA_OBJECT_ID  6213
  HEADER_FILE     2
  HEADER_BLOCK    4266
  SIZE            17 MB

SYS_LOB0000006213C00038$$
  OBJECT_ID       6214
  DATA_OBJECT_ID  6214
  HEADER_FILE     2
  HEADER_BLOCK    4274
  SIZE            19 MB

WRH$_SQL_PLAN_PK
  OBJECT_ID       6216
  DATA_OBJECT_ID  6216
  HEADER_FILE     2
  HEADER_BLOCK    4290
  SIZE            9 MB
```

Índices antes del cambio: `SYS_IL0000006213C00038$$` y `WRH$_SQL_PLAN_PK`, ambos `VALID`.

## MOVE LOB — EJECUTADO

La recreación del LOB fue realizada. Aunque una entrada posterior en SQL*Plus produjo `ORA-00911` al concatenarse accidentalmente sentencias (`...;select...`) en una misma línea, las consultas de control demuestran de forma inequívoca que el `MOVE` sí se ejecutó.

### Huella física posterior

```text
SYS_LOB0000006213C00038$$
  OBJECT_ID       6214
  DATA_OBJECT_ID  658560
  HEADER_FILE     2
  HEADER_BLOCK    99522
  SIZE            0.06 MB

WRH$_SQL_PLAN
  OBJECT_ID       6213
  DATA_OBJECT_ID  658559
  HEADER_FILE     2
  HEADER_BLOCK    99546
  SIZE            0.06 MB

WRH$_SQL_PLAN_PK
  OBJECT_ID       6216
  DATA_OBJECT_ID  6216
  HEADER_FILE     2
  HEADER_BLOCK    4290
  SIZE            9 MB
```

Conclusión: la tabla y el LOB fueron físicamente recreados/movidos. Los `OBJECT_ID` lógicos permanecieron iguales, pero cambiaron los `DATA_OBJECT_ID` y los bloques de cabecera de la tabla y el LOB. La PK permaneció en su segmento físico original.

### Índices después del MOVE

```text
SYS_IL0000006213C00038$$   LOB     VALID   SYSAUX
WRH$_SQL_PLAN_PK           NORMAL  VALID   SYSAUX
```

No es necesario reconstruir nuevamente `WRH$_SQL_PLAN_PK` mientras permanezca `VALID`.

## Punto exacto de continuación

AWR continúa pausado. Antes de reactivarlo:

1. comprobar si los antiguos bloques 76214, 76228, 76269 y 76273 continúan asignados a algún segmento;
2. consultar `V$DATABASE_BLOCK_CORRUPTION`;
3. ejecutar validación RMAN apropiada del datafile 2 / bloques afectados;
4. confirmar la configuración actual del nuevo LOB;
5. solo después restaurar AWR a 60 minutos;
6. generar un snapshot AWR de prueba;
7. comprobar `alert_orcl.log` desde la reparación y verificar que no reaparezcan `ORA-01578`, `ORA-01110` ni `ORA-26040`.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
