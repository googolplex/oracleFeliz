# Estado actual — misión `reparacion202609`

Fecha: 2026-09-16

## Documento de continuidad

Para retomar esta misión sin reconstruir el contexto, leer primero:

```text
missions/reparacion202609/HANDOFF.md
```

Ese archivo contiene el detalle consolidado de `zapallo`, KVM/libvirt, la VM `kanela`, CentOS 5.11, red/SSH, Oracle 11.2.0.1, datafiles, error AWR/LOB, reparación, RMAN, AWR, alert log y listener.

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
Listener:        FUNCIONANDO en la red actual de kanela
```

Existe un respaldo restaurable de la VM `kanela`.

## Listener

Después de cambiar la red de la VM se detectó que `listener.ora` conservaba la IP antigua:

```text
192.168.0.60
```

Se corrigió a:

```text
192.168.1.60
```

Archivo:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/network/admin/listener.ora
```

Se indicó conservar backup `listener.ora.bak_20260916`, reiniciar con `lsnrctl stop` / `lsnrctl start` y verificar con `lsnrctl status`. El usuario confirmó que el listener quedó funcionando correctamente.

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

## Datafiles conocidos

Tamaño total observado: aproximadamente `57.51 GB`.

```text
1 /home/oracle/app/oracle/oradata/orcl/system01.dbf     ~1.16 GB
2 /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf    ~1.19 GB
3 /home/oracle/app/oracle/oradata/orcl/undotbs01.dbf  ~13.06 GB
4 /home/oracle/app/oracle/oradata/orcl/users01.dbf      pequeño
5 /home/oracle/app/oracle/oradata/orcl/example01.dbf   ~0.10 GB
6 /ciruelas/oradata/tablas                             ~32 GB
7 /ciruelas/oradata/amada.dbf                          ~10 GB
```

Último espacio de filesystem conocido:

```text
/          23 GB libres
/ciruelas  51 GB libres
```

## Próxima tarea

Revisar capacidad real de los datafiles y tablespaces:

1. tamaño actual;
2. espacio usado y libre;
3. porcentaje utilizado;
4. `AUTOEXTENSIBLE`;
5. `MAXBYTES` / margen potencial;
6. comprobar simultáneamente espacio real de `/` y `/ciruelas`.

## No ejecutar sin nueva evidencia

- `BLOCKRECOVER`
- `RECOVER CORRUPTION LIST`
- `DROP_SNAPSHOT_RANGE`
- `DELETE` directo sobre `SYS.WRH$_*`
- `RESETLOGS`
- recreación de controlfiles
- repetir el `MOVE LOB` sin necesidad
