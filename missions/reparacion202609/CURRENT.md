# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Documentos de continuidad

Para retomar esta misión sin reconstruir el contexto, leer primero:

```text
missions/reparacion202609/HANDOFF.md
missions/reparacion202609/capacity-20260916.md
```

`HANDOFF.md` contiene el contexto consolidado de `zapallo`, KVM/libvirt, la VM `kanela`, CentOS 5.11, red/SSH, Oracle 11.2.0.1, datafiles, error AWR/LOB, reparación, RMAN, AWR, alert log y listener.

`capacity-20260916.md` contiene la medición exacta de espacio de datafiles y filesystems y su interpretación operativa.

## Estado operativo actual

**REPARACIÓN COMPLETADA Y ORACLE OPERATIVO.**

```text
Host KVM:        zapallo
IP zapallo:      192.168.1.71
VM Oracle:       kanela
SO kanela:       CentOS 5.11
IP kanela:       192.168.1.60
Gateway:         192.168.1.1
ORACLE_SID:      orcl
ORACLE_HOME:     /home/oracle/app/oracle/product/11.2.0/dbhome_1
Oracle:          11.2.0.1.0 64-bit
Instance:        OPEN / ACTIVE
Database ORCL:   READ WRITE
Archive mode:    NOARCHIVELOG
Listener:        FUNCIONANDO en 192.168.1.60
```

Existe un respaldo restaurable de la VM `kanela`.

## Listener

Después de cambiar la red de la VM se detectó que `listener.ora` conservaba la IP antigua `192.168.0.60`.

Se corrigió a:

```text
192.168.1.60
```

Archivo:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/network/admin/listener.ora
```

Se conservó/indicó backup `listener.ora.bak_20260916`, se reinició con `lsnrctl stop` / `lsnrctl start` y el usuario confirmó que el listener quedó funcionando correctamente.

## Reparación AWR/LOB

La causa del error repetitivo era:

```text
MMON_SLAVE / Auto-Flush Slave Action
INSERT INTO WRH$_SQL_PLAN
ORA-01578
ORA-01110
ORA-26040
file 2 / SYSAUX
LOB SYS.WRH$_SQL_PLAN.OTHER_XML
```

Bloques históricos `NOLOGGING`:

```text
76214
76228
76269
76273
```

Se pausó AWR, se recreó físicamente el LOB BasicFile mediante `MOVE LOB`, se validaron los índices y se confirmó que los cuatro bloques antiguos quedaron fuera de cualquier extent activo.

Huella posterior relevante:

```text
WRH$_SQL_PLAN                DATA_OBJECT_ID 658559 HEADER_BLOCK 99546
SYS_LOB0000006213C00038$$    DATA_OBJECT_ID 658560 HEADER_BLOCK 99522
WRH$_SQL_PLAN_PK             DATA_OBJECT_ID 6216   HEADER_BLOCK 4290
```

Índices:

```text
SYS_IL0000006213C00038$$   VALID
WRH$_SQL_PLAN_PK           VALID
```

## RMAN

Después de la reparación:

```text
VALIDATE DATAFILE 2
File Status = OK
Blocks Failing Data  = 0
Blocks Failing Index = 0
Blocks Failing Other = 0
```

`VALIDATE CHECK LOGICAL DATAFILE 2` también reportó `Blocks Failing = 0`.

Las cuatro entradas `NOLOGGING` pueden seguir figurando en `V$DATABASE_BLOCK_CORRUPTION`, pero están sobre bloques libres/no asignados. No ejecutar `BLOCKRECOVER` sobre ellas sin evidencia nueva.

## AWR

Estado final restaurado:

```text
SNAP_INTERVAL +00000 01:00:00.0
RETENTION     +00008 00:00:00.0
```

Snapshot manual de prueba exitoso:

```text
SNAP_ID 130356
BEGIN 16-SEP-26 12.08.23.777 PM
END   16-SEP-26 12.08.50.674 PM
```

## Alert log

El `alert_orcl.log`, que había alcanzado aproximadamente 541 MB, fue archivado y comprimido; el archivo activo fue truncado conservando el mismo archivo y Oracle continuó escribiendo normalmente después de `alter system switch logfile`.

## Capacidad actual de datafiles

Medición confirmada:

```text
FILE  TABLESPACE  SIZE_MB   USED_MB   FREE_MB   USED%   AUTOEXT  MAX_MB
1     SYSTEM       1190.00   1183.63      6.38   99.46   YES      32767.98
2     SYSAUX       1220.00   1105.88    114.13   90.65   YES      32767.98
3     UNDOTBS1    13370.00     26.75  13343.25    0.20   YES      32767.98
4     USERS           5.00      4.06      0.94   81.25   YES      32767.98
5     EXAMPLE        100.00     78.44     21.56   78.44   YES      32767.98
6     TABLAS       32767.98  14491.67  18276.31   44.23   YES      32767.98
7     AMANDA       10240.00    103.44  10136.56    1.01   YES      32767.98
```

Filesystems actuales:

```text
/          102G total, 75G usados, 23G libres, 77% usado
/ciruelas   98G total, 43G usados, 51G libres, 46% usado
```

### Interpretación

- No existe una emergencia general de espacio.
- `SYSTEM` tiene solo 6.38 MB libres dentro de su tamaño actual y depende de `AUTOEXTEND`; vigilar.
- `SYSAUX` tiene 114.13 MB libres internos y también depende de `AUTOEXTEND`; vigilar.
- Los datafiles alojados en `/` tienen un `MAXBYTES` teórico de ~32 GB por archivo, pero el margen físico conjunto real está limitado por los 23 GB libres del filesystem raíz y por el crecimiento del propio sistema operativo/ADR/logs.
- `UNDOTBS1` tiene ~13.34 GB libres internos y no presenta presión.
- `TABLAS` ya alcanzó el máximo individual del archivo (~32 GB), por lo que ese archivo no puede crecer más; sin embargo, conserva 18.28 GB libres internos (44.23% usado), por lo que no requiere ampliación ahora.
- `/ciruelas` conserva 51 GB físicos libres; `AMANDA` tiene ~10.14 GB libres internos y margen cómodo.

Detalle completo en `capacity-20260916.md`.

## Próximo control recomendado

Antes de cambiar tamaños o agregar datafiles, consultar:

1. `INCREMENT_BY` de cada datafile autoextensible;
2. tipo de tablespace `SMALLFILE`/`BIGFILE`, especialmente `TABLAS`;
3. crecimiento histórico si se desea estimar horizonte de capacidad.

No ampliar ni reducir datafiles todavía.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
