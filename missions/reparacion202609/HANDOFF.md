# HANDOFF completo — misión `reparacion202609`

Fecha: 2026-09-16

Este documento es el punto de entrada recomendado para continuar la misión sin reconstruir el contexto desde conversaciones anteriores.

## 1. Estado final actual

La recuperación de la VM, la base Oracle y la reparación AWR/LOB fueron completadas con éxito.

Estado confirmado de Oracle:

```text
INSTANCE_NAME     orcl
STATUS            OPEN
DATABASE_STATUS   ACTIVE
DATABASE          ORCL
OPEN_MODE         READ WRITE
```

AWR volvió a funcionar normalmente y el listener fue corregido para la nueva red de `kanela` y está operativo.

## 2. Infraestructura

### Host KVM `zapallo`

```text
Host:       zapallo
IP:         192.168.1.71
Gateway:    192.168.1.1
Bridge KVM: br0
Timezone:   -03
```

La VM `kanela` está alojada en libvirt/KVM.

Interfaz confirmada con `virsh domiflist kanela`:

```text
Interface: vnet3
Type:      bridge
Source:    br0
Model:     rtl8139
MAC:       54:52:00:13:4f:94
```

`virsh domifaddr kanela` no devolvía IP, consistente con un CentOS 5 antiguo sin guest agent.

### VM `kanela`

```text
Sistema operativo: CentOS release 5.11
IP actual:          192.168.1.60
Gateway:            192.168.1.1
```

La red antigua fue sustituida por la actual. Se detectó que una copia `ifcfg-eth0.bak` colocada dentro de `/etc/sysconfig/network-scripts/` podía ser interpretada por CentOS 5 como configuración de interfaz; se retiró del directorio y la red fue reiniciada correctamente.

### SSH

OpenSSH moderno no negociaba inicialmente con el servidor SSH antiguo de CentOS 5. Funcionó usando:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 kanela
```

Luego se configuró una excepción específica por host y autenticación RSA/passwordless. No habilitar compatibilidad SHA1 globalmente.

## 3. Almacenamiento del sistema operativo observado

```text
Filesystem                              Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00         102G   75G   23G  78% /
/dev/hda1                                99M   42M   53M  45% /boot
tmpfs                                   4.0G  1.1G  2.9G  26% /dev/shm
/dev/hdb1                                98G   43G   51G  46% /ciruelas
```

Los datafiles están repartidos entre `/home/oracle/...` (filesystem raíz) y `/ciruelas`.

## 4. Oracle

```text
Oracle Database: Oracle 11g Enterprise Edition 11.2.0.1.0 64-bit
SQL*Plus:        11.2.0.1.0
OS user:         oracle
ORACLE_SID:      orcl
ORACLE_HOME:     /home/oracle/app/oracle/product/11.2.0/dbhome_1
```

`/etc/oratab`:

```text
orcl:/home/oracle/app/oracle/product/11.2.0/dbhome_1:Y
```

SPFILE:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/spfileorcl.ora
```

La base está en `NOARCHIVELOG`:

```text
Database log mode             No Archive Mode
Automatic archival            Disabled
Archive destination           USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence    15540
Current log sequence          15542
```

Tamaño total observado de datafiles: aproximadamente `57.51 GB`.

Datafiles conocidos:

```text
1 /home/oracle/app/oracle/oradata/orcl/system01.dbf     ~1.16 GB
2 /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf    ~1.19 GB
3 /home/oracle/app/oracle/oradata/orcl/undotbs01.dbf  ~13.06 GB
4 /home/oracle/app/oracle/oradata/orcl/users01.dbf      muy pequeño
5 /home/oracle/app/oracle/oradata/orcl/example01.dbf   ~0.10 GB
6 /ciruelas/oradata/tablas                             ~32 GB
7 /ciruelas/oradata/amada.dbf                          ~10 GB
```

Existe respaldo restaurable de la VM `kanela`; se usó como punto de retorno de seguridad durante la intervención.

## 5. Listener — estado final

Después de recuperar la red de la VM se detectó que Oracle Net Listener todavía apuntaba a la IP antigua:

```text
192.168.0.60
```

Se corrigió `listener.ora` para usar la IP actual:

```text
192.168.1.60
```

Ruta de configuración:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/network/admin/listener.ora
```

Antes del cambio se indicó conservar una copia:

```text
listener.ora.bak_20260916
```

Se reinició el listener con `lsnrctl stop` / `lsnrctl start` y el usuario confirmó que quedó funcionando correctamente. Por tanto, el estado operativo actual es:

```text
Base ORCL: OPEN / ACTIVE / READ WRITE
Listener:  funcionando en la red actual de kanela (192.168.1.60)
```

Si en el futuro cambia la IP, revisar tanto `listener.ora` como `LOCAL_LISTENER` antes de asumir un problema de base de datos.

## 6. Error AWR/SYSAUX original

El `alert.log` mostraba cada hora:

```text
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

El fallo se reprodujo en:

```text
SYS$BACKGROUND
MMON_SLAVE
Auto-Flush Slave Action
INSERT INTO WRH$_SQL_PLAN
```

El LOB afectado era:

```text
SYS.WRH$_SQL_PLAN.OTHER_XML
segment: SYS_LOB0000006213C00038$$
BasicFile
SYSAUX
CHUNK 8192
LOGGING YES
IN_ROW YES
SECUREFILE NO
```

Bloques persistentes marcados `NOLOGGING`:

```text
76214
76228
76269
76273
```

## 7. Diagnóstico y reparación

AWR se pausó temporalmente con:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>0);
```

Se confirmó:

```text
SNAP_INTERVAL +40150 00:00:00.0
RETENTION     +00008 00:00:00.0
```

El LOB fue recreado mediante `ALTER TABLE ... MOVE LOB`, preservando BasicFile y SYSAUX.

Huella previa:

```text
WRH$_SQL_PLAN                DATA_OBJECT_ID 6213   HEADER_BLOCK 4266
SYS_LOB0000006213C00038$$    DATA_OBJECT_ID 6214   HEADER_BLOCK 4274
WRH$_SQL_PLAN_PK             DATA_OBJECT_ID 6216   HEADER_BLOCK 4290
```

Huella posterior:

```text
WRH$_SQL_PLAN                DATA_OBJECT_ID 658559 HEADER_BLOCK 99546
SYS_LOB0000006213C00038$$    DATA_OBJECT_ID 658560 HEADER_BLOCK 99522
WRH$_SQL_PLAN_PK             DATA_OBJECT_ID 6216   HEADER_BLOCK 4290
```

Índices posteriores:

```text
SYS_IL0000006213C00038$$   LOB     VALID
WRH$_SQL_PLAN_PK           NORMAL  VALID
```

Los cuatro bloques antiguos se comprobaron contra `DBA_EXTENTS` y quedaron sin asignación a ningún extent. Siguen apareciendo como `NOLOGGING` en `V$DATABASE_BLOCK_CORRUPTION`, pero ya no pertenecen a ningún segmento activo.

No ejecutar `BLOCKRECOVER` sobre ellos sin nueva evidencia.

## 8. Validación RMAN post-reparación

`VALIDATE DATAFILE 2`:

```text
File 2 Status OK
Marked Corrupt 4
Blocks Failing Data  0
Blocks Failing Index 0
Blocks Failing Other 0
```

`VALIDATE CHECK LOGICAL DATAFILE 2` dio el mismo resultado limpio:

```text
File 2 Status OK
Blocks Failing = 0
```

Conclusión: `sysaux01.dbf` es legible física y lógicamente y no hay corrupción activa que falle validación.

## 9. AWR restaurado y probado

AWR se restauró con:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>60);
```

Estado final:

```text
SNAP_INTERVAL +00000 01:00:00.0
RETENTION     +00008 00:00:00.0
```

Se creó un snapshot manual con:

```sql
exec dbms_workload_repository.create_snapshot();
```

Resultado confirmado:

```text
SNAP_ID 130356
BEGIN 16-SEP-26 12.08.23.777 PM
END   16-SEP-26 12.08.50.674 PM
```

El snapshot manual confirma que el flujo AWR que antes fallaba volvió a funcionar.

## 10. `alert_orcl.log`

Ruta:

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```

Había crecido hasta aproximadamente 541 MB. Se archivó y comprimió el histórico y se truncó el archivo activo conservando su inode:

```bash
cd /home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace
cp -p alert_orcl.log alert_orcl_20260916_pre_limpieza.log
gzip alert_orcl_20260916_pre_limpieza.log
: > alert_orcl.log
```

Se forzó una nueva escritura Oracle con:

```sql
alter system switch logfile;
```

El usuario confirmó que el nuevo alert log sigue funcionando normalmente.

## 11. Reglas de seguridad que siguen vigentes

No ejecutar sin evidencia nueva:

- `BLOCKRECOVER` / `RECOVER CORRUPTION LIST` sobre los cuatro bloques libres;
- `RESETLOGS`;
- recreación de controlfiles;
- `DROP` de objetos AWR internos;
- `DELETE` directo sobre `SYS.WRH$_*`;
- `DROP_SNAPSHOT_RANGE` sin causa específica;
- repetir el `MOVE LOB` innecesariamente.

## 12. Próximo trabajo

La reparación está cerrada. El siguiente control solicitado es revisar capacidad de almacenamiento:

1. espacio libre actual dentro de cada datafile/tablespace;
2. `AUTOEXTENSIBLE`, `MAXBYTES` y margen potencial de crecimiento;
3. espacio libre real de los filesystems `/` y `/ciruelas` antes de permitir crecimiento adicional.

Este documento, junto con `CURRENT.md`, debe permitir continuar la administración de `orcl` sin volver a proporcionar el contexto histórico completo.
