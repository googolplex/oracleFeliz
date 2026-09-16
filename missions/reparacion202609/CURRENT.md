# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Punto exacto de continuación

Oracle `orcl` en la VM `kanela` está `OPEN`, `ACTIVE`, `READ WRITE`, pero la tarea AWR `MMON_SLAVE / Auto-Flush Slave Action` sigue fallando al ejecutar `INSERT INTO wrh$_sql_plan` por `ORA-01578` / `ORA-26040` en el bloque 76214 del datafile 2 (`SYSAUX`). El bloque pertenece al LOB BasicFile `SYS.WRH$_SQL_PLAN.OTHER_XML`.

El LOB actual:

```text
SEGMENT_NAME  SYS_LOB0000006213C00038$$
INDEX_NAME    SYS_IL0000006213C00038$$
TABLESPACE    SYSAUX
CHUNK         8192
RETENTION     900
CACHE         NO
LOGGING       YES
IN_ROW        YES
SECUREFILE    NO
SIZE          ~19 MB
```

`WRH$_SQL_PLAN` no está particionada.

Índices actuales:

```text
SYS_IL0000006213C00038$$   LOB     VALID   SYSAUX   UNIQUE
WRH$_SQL_PLAN_PK           NORMAL  VALID   SYSAUX   UNIQUE
```

No se ha ejecutado todavía ningún `ALTER TABLE ... MOVE LOB`.

## Modo de archivado

`ARCHIVE LOG LIST`:

```text
Database log mode           No Archive Mode
Automatic archival          Disabled
Archive destination         USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence  15540
Current log sequence        15542
```

Conclusión: cualquier backup Oracle previo al DDL debe ser consistente. No hacer backup abierto de la base asumiendo que será recuperable. En NOARCHIVELOG, un backup RMAN válido debe hacerse después de shutdown consistente y con la base montada.

## Tamaño de la base

Total de datafiles:

```text
57.51 GB
```

Detalle:

```text
1  /home/oracle/app/oracle/oradata/orcl/system01.dbf    1.16 GB
2  /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf   1.19 GB
3  /home/oracle/app/oracle/oradata/orcl/undotbs01.dbf 13.06 GB
4  /home/oracle/app/oracle/oradata/orcl/users01.dbf    ~0 GB reportado por redondeo
5  /home/oracle/app/oracle/oradata/orcl/example01.dbf   0.10 GB
6  /ciruelas/oradata/tablas                            32.00 GB
7  /ciruelas/oradata/amada.dbf                        10.00 GB
```

`sysaux01.dbf` tiene 1220 MB y estado `AVAILABLE`.

## Espacio disponible en la VM

```text
/          23 GB libres
/ciruelas  51 GB libres
```

El total de 57.51 GB no cabe sin más en `/ciruelas`; por tanto no usar `/ciruelas` como único destino de un backup completo sin comprobar compresión y margen adicional.

## Estrategia de seguridad elegida antes del DDL

Como `kanela` es una VM libvirt/KVM alojada en `zapallo`, la opción preferida es inspeccionar y, si hay espacio suficiente, realizar un respaldo en frío a nivel de los discos virtuales con la VM apagada limpiamente. Deben respaldarse todos los discos de la VM, porque los datafiles están repartidos entre `/home/oracle/...` y `/ciruelas/...`.

Antes de apagar nada, ejecutar en `zapallo`:

```bash
sudo virsh domblklist kanela --details
df -hT
```

Con esa salida se decidirá el método exacto de copia según los discos sean archivos, volúmenes LVM u otro tipo, y según el espacio disponible en el host.

## No ejecutar todavía

- `ALTER TABLE SYS.WRH$_SQL_PLAN MOVE LOB (OTHER_XML) ...`
- `BLOCKRECOVER`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- apagado de la VM hasta definir el backup en `zapallo`
