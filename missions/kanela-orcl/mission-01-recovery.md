# Misión 01 — Recuperación y diagnóstico de Oracle en `kanela`

## Estado
**EN CURSO — diagnóstico avanzado, base abierta y operativa.**

Última actualización: 2026-09-16.

## Objetivo
Diagnosticar y reparar con el menor riesgo posible un Oracle 11g antiguo que se consideraba fuera de servicio, preservando los datos de aplicación y evitando operaciones destructivas hasta identificar con precisión la causa.

---

## Infraestructura y red

### Host KVM
- Host: `zapallo`
- IP actual: `192.168.1.71`
- Bridge de la VM: `br0`
- Gateway confirmado desde `zapallo`: `192.168.1.1`

### VM Oracle
- Nombre libvirt: `kanela`
- MAC: `54:52:00:13:4f:94`
- Modelo NIC: `rtl8139`
- Bridge: `br0`
- IP antigua: estaba en la red anterior (`192.169.0.60` reportada inicialmente; la configuración de gateway también estaba en la red 0).
- IP actual configurada y activa: `192.168.1.60`
- Gateway actual: `192.168.1.1`
- La red se reinició correctamente en CentOS 5 mediante `/etc/init.d/network restart`.
- Importante: no guardar copias `ifcfg-*` dentro de `/etc/sysconfig/network-scripts/`, porque CentOS 5 puede intentar tratarlas como interfaces. El backup de `ifcfg-eth0` se movió fuera de ese directorio.

### SSH
El OpenSSH moderno de `zapallo` rechazaba inicialmente el KEX antiguo de CentOS 5:

`Unable to negotiate ... no matching key exchange method found`

Método confirmado funcional:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 kanela
```

Se configuró autenticación por clave SSH para evitar ingresar repetidamente la contraseña. Mantener la compatibilidad SHA1 restringida solo al host `kanela`, no de forma global.

---

## Sistema operativo de `kanela`
- CentOS release **5.11**.

### Filesystems (`df -vh`)

```text
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00
                      102G   75G   23G  78% /
/dev/hda1              99M   42M   53M  45% /boot
tmpfs                 4.0G  1.1G  2.9G  26% /dev/shm
/dev/hdb1              98G   43G   51G  46% /ciruelas
```

Observaciones:
- `/` tiene 102 GB, 75 GB usados, 23 GB libres, 78% de uso.
- `/ciruelas` tiene 98 GB, 43 GB usados, 51 GB libres, 46% de uso.
- No hay, por ahora, evidencia de falta crítica de espacio que explique la falla de Oracle.

---

## Oracle

### Identidad
- Usuario del SO: `oracle`
- `ORACLE_SID=orcl`
- `ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1`
- SQL*Plus: `11.2.0.1.0`
- Oracle Database 11g Enterprise Edition Release `11.2.0.1.0` 64-bit Production.
- Opciones reportadas: Partitioning, OLAP, Data Mining, Real Application Testing.

### `/etc/oratab`

```text
orcl:/home/oracle/app/oracle/product/11.2.0/dbhome_1:Y
```

### SPFILE
Existe:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/spfileorcl.ora
```

Tamaño observado: 3584 bytes.

### Estado actual de la base
Confirmado desde SQL*Plus:

```sql
select instance_name, status, database_status from v$instance;
```

Resultado:

```text
orcl  OPEN  ACTIVE
```

```sql
select name, open_mode from v$database;
```

Resultado:

```text
ORCL  READ WRITE
```

Conclusión: **la base está abierta, activa y en READ WRITE. El problema no es actualmente un fallo de arranque.**

---

## Evidencia del `alert_orcl.log`

Ruta:

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```

Tamaño observado: aproximadamente 541 MB.

### Antes del reinicio
Oracle reportaba repetidamente, aproximadamente cada hora:

```text
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

El mismo bloque 76214 aparecía, entre otros momentos, a las 08:00, 09:00 y 10:00 del 2026-09-16.

### Reinicio observado
El log muestra un apagado limpio:

- 10:38:59 — `Shutting down instance (immediate)`
- `ALTER DATABASE CLOSE NORMAL`
- `ALTER DATABASE DISMOUNT`
- 10:39:40 — `Instance shutdown complete`

Luego un arranque normal:

- 10:41:15 — `Starting ORACLE instance (normal)`
- PMON y demás background processes arrancaron normalmente.
- `ALTER DATABASE MOUNT` completado.
- `ALTER DATABASE OPEN` completado.
- Base abrió correctamente en secuencia redo 15542.

Conclusión: **el reboot no produjo un fallo de recuperación; Oracle abrió normalmente.**

---

## Datafile afectado

```text
file #2: /home/oracle/app/oracle/oradata/orcl/sysaux01.dbf
Tablespace: SYSAUX
```

### Primer mapeo del bloque 76214
Consulta sobre `DBA_EXTENTS` mostró:

```text
OWNER: SYS
SEGMENT_NAME: SYS_LOB0000006213C00038$$
SEGMENT_TYPE: LOBSEGMENT
TABLESPACE_NAME: SYSAUX
FILE_ID: 2
EXTENT: block 76160, 128 blocks
```

`DBA_LOBS` resolvió ese LOB como:

```text
TABLE: SYS.WRH$_SQL_PLAN
COLUMN: OTHER_XML
LOB SEGMENT: SYS_LOB0000006213C00038$$
```

Esto ubica el bloque original en **AWR (histórico de planes SQL)** y no en una tabla de aplicación.

---

## Inventario inicial de `V$DATABASE_BLOCK_CORRUPTION`

Antes de las validaciones RMAN se observaron 8 entradas que totalizaban 14 bloques:

```text
file 2 block 99009 blocks 2 NOLOGGING
file 2 block 98812 blocks 4 NOLOGGING
file 2 block 98808 blocks 3 NOLOGGING
file 2 block 76273 blocks 1 NOLOGGING
file 2 block 76269 blocks 1 NOLOGGING
file 2 block 76228 blocks 1 NOLOGGING
file 2 block 76214 blocks 1 NOLOGGING
file 2 block 64812 blocks 1 CHECKSUM
```

### Mapeo de esos bloques a objetos

- `64812 CHECKSUM` → `SYS.WRH$_PARAMETER`, TABLE PARTITION, SYSAUX.
- `76214 NOLOGGING` → `SYS.WRH$_SQL_PLAN.OTHER_XML` LOB.
- `76228 NOLOGGING` → mismo LOB `SYS_LOB0000006213C00038$$`.
- `76269 NOLOGGING` → mismo LOB.
- `76273 NOLOGGING` → mismo LOB.
- `98808-98810 NOLOGGING` → `SYS.WRH$_PARAMETER`.
- `98812-98815 NOLOGGING` → `SYS.WRH$_PARAMETER`.
- `99009-99010 NOLOGGING` → `SYS.WRH$_LATCH`.

Todos los objetos afectados detectados hasta ese momento eran **objetos internos de AWR en SYSAUX**; no se encontró corrupción en objetos de aplicación.

---

## Validaciones RMAN realizadas

### 1. `VALIDATE DATAFILE 2`

Ejecutado:

```text
VALIDATE DATAFILE 2;
```

Resultado resumido:

```text
File Status Marked Corrupt Empty Blocks Blocks Examined
2    OK     4              23492        156174

Data  Blocks Failing: 0
Index Blocks Failing: 0
Other Blocks Failing: 0
```

Elapsed: ~7 segundos.

Después de esta validación, `V$DATABASE_BLOCK_CORRUPTION` se redujo a solo cuatro bloques:

```text
76214 NOLOGGING
76228 NOLOGGING
76269 NOLOGGING
76273 NOLOGGING
```

Desaparecieron del inventario:
- el bloque `64812 CHECKSUM`,
- los bloques de `WRH$_PARAMETER`,
- los bloques de `WRH$_LATCH`.

Interpretación de trabajo: RMAN pudo releerlos correctamente y ya no quedaron registrados como corrupción activa.

### 2. `VALIDATE CHECK LOGICAL DATAFILE 2`

Ejecutado:

```text
VALIDATE CHECK LOGICAL DATAFILE 2;
```

Resultado:

```text
File Status Marked Corrupt Empty Blocks Blocks Examined
2    OK     4              23492        156174

Data  Blocks Failing: 0
Index Blocks Failing: 0
Other Blocks Failing: 0
```

Elapsed: ~3 segundos.

Conclusión actual:
- RMAN no detecta bloques fallando físicamente.
- RMAN no detecta bloques fallando lógicamente.
- Persisten solo 4 bloques marcados `NOLOGGING`, todos dentro del LOB `OTHER_XML` de `SYS.WRH$_SQL_PLAN`.

**No se ha ejecutado `BLOCKRECOVER`.**
**No se ha ejecutado ninguna recuperación agresiva.**

---

## AWR

### Snapshots actuales

```sql
select min(snap_id),max(snap_id),min(begin_interval_time),max(end_interval_time),count(*) from dba_hist_snapshot;
```

Resultado:

```text
MIN SNAP_ID: 130150
MAX SNAP_ID: 130354
MIN BEGIN:   07-SEP-26 10.00.33.553 PM
MAX END:     16-SEP-26 11.00.21.748 AM
COUNT:       205
```

### Configuración de retención

```sql
select snap_interval, retention from dba_hist_wr_control;
```

Resultado:

```text
SNAP_INTERVAL = +00000 01:00:00.0
RETENTION     = +00008 00:00:00.0
```

Interpretación:
- Snapshot AWR cada 1 hora.
- Retención de 8 días.

### Baselines

```sql
select baseline_id,baseline_name,baseline_type,start_snap_id,end_snap_id,expiration from dba_hist_baseline order by baseline_id;
```

Resultado único:

```text
BASELINE_ID:   0
BASELINE_NAME: SYSTEM_MOVING_WINDOW
TYPE:          MOVING_WINDOW
START_SNAP_ID: 130163
END_SNAP_ID:   130354
```

No se encontraron baselines AWR estáticas creadas por usuario.

### `PURGE_SQL_DETAILS`
Se comprobó que el procedimiento existe:

```text
SYS.DBMS_WORKLOAD_REPOSITORY.PURGE_SQL_DETAILS
```

Pero la consulta de candidatos huérfanos devolvió:

```sql
select count(*)
from sys.wrh$_sql_plan p
where not exists (
  select 1
  from sys.wrh$_sqlstat s
  where s.dbid=p.dbid and s.sql_id=p.sql_id
);
```

Resultado:

```text
COUNT(*) = 0
```

Por lo tanto, **no ejecutar `PURGE_SQL_DETAILS` actualmente**, porque no hay filas huérfanas según ese criterio.

### Nuevo hallazgo crítico: `OTHER_XML`

```sql
select count(*) from sys.wrh$_sql_plan where other_xml is not null;
```

Resultado:

```text
COUNT(*) = 0
```

Esto significa que actualmente **no existe ninguna fila de `SYS.WRH$_SQL_PLAN` con `OTHER_XML` no nulo**, aunque los cuatro bloques `NOLOGGING` persistentes pertenecen físicamente al LOBSEGMENT de esa columna.

Hipótesis de trabajo a verificar:
- los cuatro bloques marcados pueden pertenecer a espacio LOB antiguo/inactivo o a contenido que ya no está referenciado por filas actuales;
- no asumir todavía que sea necesario borrar snapshots ni reconstruir el LOB;
- este resultado hace menos probable que una lectura de una fila actual de `OTHER_XML` sea la que siga disparando la corrupción.

---

## Operaciones que NO deben ejecutarse sin nueva evidencia

No ejecutar por ahora:
- `BLOCKRECOVER CORRUPTION LIST`
- `RECOVER DATAFILE`
- `RESETLOGS`
- recreación de controlfiles
- borrado o reemplazo de `sysaux01.dbf`
- `DELETE` directo sobre `SYS.WRH$_SQL_PLAN`, `SYS.WRH$_PARAMETER`, `SYS.WRH$_LATCH` u otras `WRH$_*`
- `DROP` de objetos AWR internos
- `DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE` hasta justificar con evidencia qué rango eliminar
- `DBMS_WORKLOAD_REPOSITORY.PURGE_SQL_DETAILS` mientras siga habiendo 0 candidatos

---

## Punto exacto para continuar

Estado actual confirmado:
1. CentOS 5.11 operativo en `kanela`.
2. Red corregida a `192.168.1.60/24`, gateway `192.168.1.1`.
3. SSH desde `zapallo` operativo con KEX compatible y clave pública.
4. Oracle 11.2.0.1 `orcl` está `OPEN`, `ACTIVE`, `READ WRITE`.
5. `SYSAUX` valida `OK` física y lógicamente con RMAN.
6. `Blocks Failing = 0`.
7. Persisten 4 marcas `NOLOGGING`: 76214, 76228, 76269, 76273.
8. Las cuatro pertenecen al LOB `OTHER_XML` de `SYS.WRH$_SQL_PLAN`.
9. `PURGE_SQL_DETAILS` existe pero hay 0 filas huérfanas.
10. `select count(*) from sys.wrh$_sql_plan where other_xml is not null;` devuelve **0**.

### Próximo objetivo recomendado
Antes de modificar AWR:
- determinar por qué continúan marcados esos cuatro bloques si `OTHER_XML` no tiene valores no nulos actualmente;
- comprobar si las marcas siguen generando nuevos `ORA-01578/ORA-26040` después del reinicio y de las validaciones RMAN;
- correlacionar cualquier nuevo incidente con tarea AWR/MMON y hora exacta;
- considerar después mecanismos soportados para liberar/reorganizar AWR únicamente si se demuestra necesario.

Una comprobación muy útil al retomar será revisar solo los nuevos errores posteriores a 10:41 del 2026-09-16 en `alert_orcl.log`, sin reanalizar los 541 MB completos.

---

## Principio de seguridad de esta misión
La base está operativa. La prioridad es **preservar los datos de aplicación y eliminar únicamente la condición residual de AWR de forma soportada y verificable**. No convertir un problema confinado a AWR en una recuperación invasiva de toda la base.
