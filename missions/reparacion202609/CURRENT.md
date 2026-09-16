# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Documentos de continuidad

Para retomar esta misión sin reconstruir el contexto, leer primero:

```text
missions/reparacion202609/HANDOFF.md
missions/reparacion202609/capacity-20260916.md
```

`HANDOFF.md` contiene el contexto consolidado de `zapallo`, KVM/libvirt, la VM `kanela`, CentOS 5.11, red/SSH, Oracle 11.2.0.1, datafiles, error AWR/LOB, reparación, RMAN, AWR, alert log y listener.

`capacity-20260916.md` contiene la medición exacta de espacio, `AUTOEXTEND`, `INCREMENT_BY`, tipo `SMALLFILE/BIGFILE`, límites por datafile y espacio real de filesystems.

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

## Reparación AWR/LOB

Reparación completada con éxito. El fallo original era `MMON_SLAVE / Auto-Flush Slave Action -> INSERT INTO WRH$_SQL_PLAN -> ORA-01578 / ORA-01110 / ORA-26040` sobre el LOB `SYS.WRH$_SQL_PLAN.OTHER_XML` en `SYSAUX`.

El LOB fue recreado, ambos índices quedaron `VALID`, los cuatro bloques históricos `NOLOGGING` quedaron fuera de cualquier extent activo y RMAN confirmó `Blocks Failing = 0` tanto en `VALIDATE DATAFILE 2` como en `VALIDATE CHECK LOGICAL DATAFILE 2`.

AWR quedó restaurado a una hora / ocho días y el snapshot manual de prueba `SNAP_ID 130356` se creó correctamente.

El `alert_orcl.log` histórico de ~541 MB fue archivado/comprimido y el archivo activo quedó operativo.

## Listener

`listener.ora` apuntaba a la IP antigua `192.168.0.60`. Fue corregido a `192.168.1.60`, se reinició el listener y el usuario confirmó que quedó funcionando correctamente.

## Capacidad de almacenamiento — VERIFICADA

### Filesystems

```text
/          102G total, 75G usados, 23G libres, 77% usado
/ciruelas   98G total, 43G usados, 51G libres, 46% usado
```

### Datafiles y espacio libre interno

```text
SYSTEM     1190.00 MB total,     6.38 MB libres, 99.46% usado
SYSAUX     1220.00 MB total,   114.13 MB libres, 90.65% usado
UNDOTBS1  13370.00 MB total, 13343.25 MB libres,  0.20% usado
USERS         5.00 MB total,     0.94 MB libres, 81.25% usado
EXAMPLE      100.00 MB total,    21.56 MB libres, 78.44% usado
TABLAS     32767.98 MB total, 18276.31 MB libres, 44.23% usado
AMANDA     10240.00 MB total, 10136.56 MB libres,  1.01% usado
```

Todos los tablespaces consultados son `SMALLFILE` (`BIGFILE=NO`), bloque de 8192 bytes.

### AUTOEXTEND real

```text
SYSTEM     NEXT 10.00 MB
SYSAUX     NEXT 10.00 MB
UNDOTBS1   NEXT  5.00 MB
USERS      NEXT  1.25 MB
EXAMPLE    NEXT  0.63 MB
TABLAS     NEXT 10 GB configurado, pero archivo ya en MAXBYTES
AMANDA     NEXT  2 GB
```

### Interpretación final

- No hace falta ampliar ningún datafile ahora.
- `SYSTEM` requiere vigilancia porque solo tiene 6.38 MB libres internos, pero crece automáticamente en pasos de 10 MB y `/` todavía dispone de 23 GB físicos libres compartidos.
- `SYSAUX` tiene 114.13 MB libres internos y autoextend de 10 MB.
- `UNDOTBS1` está ampliamente sobredimensionado respecto del uso actual; no reducir sin estudiar high-water mark y uso real de UNDO.
- `TABLAS` es `SMALLFILE`, su datafile ya está en ~32 GB, exactamente su `MAXBYTES`, por lo que no puede crecer más aunque `AUTOEXTENSIBLE=YES`. Sin embargo, todavía tiene 18.28 GB libres internos.
- Si `TABLAS` se acercara a agotarse, la acción correcta sería agregar un segundo datafile en `/ciruelas`, no intentar ampliar el archivo actual.
- `AMANDA` tiene margen muy amplio.
- `/ciruelas` dispone de 51 GB libres físicos.

Detalle completo: `missions/reparacion202609/capacity-20260916.md`.

## Estado de la misión

La reparación está cerrada. La capacidad fue revisada y no requiere cambios inmediatos. En futuras sesiones, empezar por `HANDOFF.md`, `CURRENT.md` y `capacity-20260916.md`; no es necesario volver a reconstruir el contexto histórico.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
- ampliar/reducir datafiles sin una nueva medición de espacio y crecimiento
