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

## Verificación de los cuatro bloques antiguos

Se comprobó si los bloques históricos problemáticos continuaban perteneciendo a algún extent:

```sql
select b.block#,e.owner,e.segment_name,e.segment_type,e.block_id,e.blocks
from (
  select 76214 block# from dual
  union all select 76228 from dual
  union all select 76269 from dual
  union all select 76273 from dual
) b
left join dba_extents e
  on e.file_id=2
 and b.block# between e.block_id and e.block_id+e.blocks-1
order by b.block#;
```

Resultado: para los cuatro bloques (`76214`, `76228`, `76269`, `76273`) las columnas de `DBA_EXTENTS` quedaron nulas.

Conclusión: los cuatro bloques antiguos ya no están asignados a ningún extent y, por tanto, ya no pertenecen al LOB nuevo ni a ningún otro segmento actualmente asignado. Este es un indicio fuerte de que el `MOVE LOB` liberó correctamente el espacio físico antiguo que contenía las marcas `NOLOGGING`.

## Punto exacto de continuación

AWR continúa pausado. Siguiente secuencia:

1. consultar `V$DATABASE_BLOCK_CORRUPTION` para comprobar si Oracle todavía conserva las cuatro marcas históricas;
2. ejecutar `RMAN VALIDATE DATAFILE 2`;
3. volver a consultar `V$DATABASE_BLOCK_CORRUPTION`;
4. ejecutar `RMAN VALIDATE CHECK LOGICAL DATAFILE 2` si todavía fuera necesario;
5. confirmar la configuración actual del nuevo LOB;
6. restaurar AWR a 60 minutos;
7. generar un snapshot AWR de prueba;
8. comprobar `alert_orcl.log` desde la reparación y verificar que no reaparezcan `ORA-01578`, `ORA-01110` ni `ORA-26040`.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
