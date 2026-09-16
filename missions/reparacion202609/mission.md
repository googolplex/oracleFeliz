# Misión `reparacion202609` — Recuperación y reparación de Oracle 11g en `kanela`

## Estado

**EN CURSO — Oracle está operativo, pero persiste un error AWR/LOB `NOLOGGING` que estamos aislando antes de cualquier DDL de reparación.**

Última actualización: **2026-09-16**.

## Regla de idioma

Toda la misión debe mantenerse en **español neutral**, sin regionalismos, modismos locales ni formas verbales regionales. Las instrucciones deben ser técnicas, directas, reproducibles y conservadoras.

---

# 1. Objetivo de la misión

Recuperar y estabilizar un Oracle 11g antiguo que estaba fuera de uso, preservando los datos de aplicación y evitando operaciones invasivas. La misión cubre tanto la puesta nuevamente en marcha de la VM y del sistema operativo como el diagnóstico y reparación del error residual de Oracle.

Principios aplicados:

- diagnóstico de solo lectura primero;
- una modificación a la vez;
- preservar los datos de aplicación;
- no usar `RESETLOGS`, recreación de controlfiles, borrado de datafiles, `DROP` de objetos internos ni `DELETE` directo sobre `SYS.WRH$_*` sin evidencia suficiente;
- distinguir corrupción física real de bloques marcados `NOLOGGING`;
- documentar cada hallazgo antes de reparar.

---

# 2. Infraestructura física / host KVM: `zapallo`

## 2.1 Identidad y red

Host de virtualización:

```text
zapallo
```

Datos confirmados:

```text
IP de zapallo: 192.168.1.71
Gateway:        192.168.1.1
Bridge KVM:     br0
Zona horaria:   -03
```

La VM Oracle `kanela` se ejecuta mediante libvirt/KVM en `zapallo`.

## 2.2 Identificación de la VM

Con `virsh list --all` se confirmó que `kanela` estaba definida y ejecutándose.

Con:

```bash
virsh domiflist kanela
```

se obtuvo:

```text
Interface: vnet3
Type:      bridge
Source:    br0
Model:     rtl8139
MAC:       54:52:00:13:4f:94
```

`virsh domifaddr kanela` no pudo obtener la IP del guest, comportamiento esperado para un CentOS 5 antiguo sin guest agent funcional.

---

# 3. Máquina virtual Oracle: `kanela`

## 3.1 Sistema operativo

Sistema operativo confirmado:

```text
CentOS release 5.11
```

Es un sistema muy antiguo; por tanto, muchas herramientas modernas requieren excepciones de compatibilidad.

## 3.2 Red original y corrección

La VM estaba configurada en una red antigua. Se reportó inicialmente una IP de la red anterior como:

```text
192.169.0.60
```

La configuración fue corregida a la red actual:

```text
IP actual de kanela: 192.168.1.60
Gateway:              192.168.1.1
```

La VM está conectada al bridge `br0` de `zapallo`.

### Problema detectado con archivos de configuración de red

Se creó una copia de seguridad llamada aproximadamente:

```text
ifcfg-eth0.bak
```

inicialmente dentro de:

```text
/etc/sysconfig/network-scripts/
```

En CentOS 5 esto es peligroso porque los scripts de red pueden intentar interpretar cualquier archivo `ifcfg-*` del directorio como una interfaz adicional.

La copia fue movida fuera de `/etc/sysconfig/network-scripts/` y luego se reinició la red correctamente.

Comando empleado:

```bash
/etc/init.d/network restart
```

Resultado final: conectividad de red funcional en `192.168.1.60`.

---

# 4. SSH desde `zapallo` a `kanela`

El OpenSSH moderno de `zapallo` rechazó inicialmente los algoritmos antiguos ofrecidos por CentOS 5:

```text
Unable to negotiate with 192.168.1.60 port 22: no matching key exchange method found.
Their offer: diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```

Se comprobó funcional el acceso con:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 kanela
```

Luego se configuró autenticación por clave SSH para evitar introducir la contraseña en cada acceso.

Regla importante: la excepción SHA1 debe quedar restringida solamente al host `kanela`, no habilitada globalmente.

---

# 5. Almacenamiento de CentOS (`df -vh`)

Estado observado:

```text
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00
                      102G   75G   23G  78% /
/dev/hda1              99M   42M   53M  45% /boot
tmpfs                 4.0G  1.1G  2.9G  26% /dev/shm
/dev/hdb1              98G   43G   51G  46% /ciruelas
```

Interpretación actual:

- `/`: 102 GB totales, 75 GB usados, 23 GB libres, 78% de uso.
- `/boot`: 99 MB totales, 45% de uso.
- `/dev/shm`: 4.0 GB, 26% de uso.
- `/ciruelas`: 98 GB totales, 43 GB usados, 51 GB libres, 46% de uso.
- No hay evidencia de falta crítica de espacio que explique el problema actual de Oracle.

---

# 6. Oracle instalado en `kanela`

## 6.1 Usuario y variables principales

Usuario del sistema operativo:

```text
oracle
```

Variables conocidas:

```text
ORACLE_SID=orcl
ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1
```

## 6.2 Versión

Comando:

```bash
sqlplus -v
```

Resultado:

```text
SQL*Plus: Release 11.2.0.1.0 Production
```

Base de datos:

```text
Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
```

Es una versión muy antigua y fuera de soporte; cualquier sintaxis de reparación debe validarse específicamente para 11.2.0.1 antes de ejecutarla.

## 6.3 `/etc/oratab`

Contenido relevante:

```text
orcl:/home/oracle/app/oracle/product/11.2.0/dbhome_1:Y
```

## 6.4 SPFILE

Existe:

```text
/home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/spfileorcl.ora
```

Tamaño observado:

```text
3584 bytes
```

No se necesitó crear un `initorcl.ora` para poner la instancia en marcha.

---

# 7. Puesta en marcha de la base

Luego de recuperar conectividad y reiniciar la VM, el `alert.log` mostró que Oracle realizó un apagado y arranque normales.

## 7.1 `alert.log`

Ruta:

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```

Tamaño observado en el momento del diagnóstico:

```text
aprox. 541 MB
```

## 7.2 Apagado limpio observado

Secuencia:

```text
16-SEP-2026 10:38:59  Shutting down instance (immediate)
ALTER DATABASE CLOSE NORMAL
ALTER DATABASE DISMOUNT
16-SEP-2026 10:39:40  Instance shutdown complete
```

## 7.3 Arranque normal observado

Secuencia:

```text
16-SEP-2026 10:41:15  Starting ORACLE instance (normal)
```

PMON y los demás procesos background arrancaron correctamente.

Luego:

```text
ALTER DATABASE MOUNT
ALTER DATABASE OPEN
Completed: ALTER DATABASE OPEN
```

La apertura terminó aproximadamente a las 10:41:21.

## 7.4 Estado confirmado desde SQL*Plus

Consulta:

```sql
select instance_name, status, database_status from v$instance;
```

Resultado:

```text
orcl  OPEN  ACTIVE
```

Consulta:

```sql
select name, open_mode from v$database;
```

Resultado:

```text
ORCL  READ WRITE
```

Conclusión: **la base está abierta, activa y en modo READ WRITE. El problema actual no es un fallo de arranque.**

---

# 8. Error Oracle que estamos resolviendo

Antes del reinicio, el `alert.log` mostraba errores repetidos aproximadamente cada hora:

```text
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

Se observaron ocurrencias alrededor de las 08:00, 09:00 y 10:00 del 16-SEP-2026.

El datafile afectado es:

```text
file #2
/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf
Tablespace: SYSAUX
```

---

# 9. Mapeo inicial del bloque 76214

Consulta de `DBA_EXTENTS`:

```sql
select owner,segment_name,partition_name,segment_type,tablespace_name,file_id,block_id,blocks
from dba_extents
where file_id=2
and 76214 between block_id and block_id+blocks-1;
```

Resultado:

```text
OWNER          SYS
SEGMENT_NAME   SYS_LOB0000006213C00038$$
SEGMENT_TYPE   LOBSEGMENT
TABLESPACE     SYSAUX
FILE_ID        2
BLOCK_ID       76160
BLOCKS         128
```

Ese extent cubre aproximadamente los bloques 76160–76287.

Luego:

```sql
select owner,table_name,column_name,segment_name
from dba_lobs
where segment_name='SYS_LOB0000006213C00038$$';
```

Resultado:

```text
OWNER       SYS
TABLE_NAME  WRH$_SQL_PLAN
COLUMN_NAME OTHER_XML
SEGMENT_NAME SYS_LOB0000006213C00038$$
```

Conclusión: el bloque 76214 pertenece al LOB de la columna:

```text
SYS.WRH$_SQL_PLAN.OTHER_XML
```

Esto es información interna de AWR, no una tabla de aplicación.

---

# 10. Inventario inicial de corrupción conocido por Oracle

Consulta:

```sql
select file#,block#,blocks,corruption_change#,corruption_type
from v$database_block_corruption
where file#=2;
```

Resultado inicial:

```text
FILE# BLOCK# BLOCKS CORRUPTION_CHANGE# CORRUPTION_TYPE
2     99009  2      3227705767         NOLOGGING
2     98812  4      3227705767         NOLOGGING
2     98808  3      3227703165         NOLOGGING
2     76273  1      3227698275         NOLOGGING
2     76269  1      3227698275         NOLOGGING
2     76228  1      3227698275         NOLOGGING
2     76214  1      3227698275         NOLOGGING
2     64812  1      0                  CHECKSUM
```

Total inicial:

- 8 rangos/entradas;
- 14 bloques en total;
- 13 marcados `NOLOGGING`;
- 1 marcado `CHECKSUM`.

---

# 11. Mapeo de todos los bloques a objetos

Consulta usada:

```sql
select c.file#,c.block#,c.blocks,c.corruption_type,
       e.owner,e.segment_name,e.segment_type,e.tablespace_name,
       e.block_id extent_start,e.blocks extent_blocks
from v$database_block_corruption c
left join dba_extents e
  on e.file_id=c.file#
 and c.block#<=e.block_id+e.blocks-1
 and c.block#+c.blocks-1>=e.block_id
where c.file#=2
order by c.block#;
```

Resultados:

```text
64812           CHECKSUM   SYS.WRH$_PARAMETER       TABLE PARTITION  SYSAUX
76214           NOLOGGING  SYS_LOB0000006213C00038$$ LOBSEGMENT     SYSAUX
76228           NOLOGGING  SYS_LOB0000006213C00038$$ LOBSEGMENT     SYSAUX
76269           NOLOGGING  SYS_LOB0000006213C00038$$ LOBSEGMENT     SYSAUX
76273           NOLOGGING  SYS_LOB0000006213C00038$$ LOBSEGMENT     SYSAUX
98808-98810     NOLOGGING  SYS.WRH$_PARAMETER       TABLE PARTITION SYSAUX
98812-98815     NOLOGGING  SYS.WRH$_PARAMETER       TABLE PARTITION SYSAUX
99009-99010     NOLOGGING  SYS.WRH$_LATCH           TABLE PARTITION SYSAUX
```

Todos los objetos afectados detectados pertenecían a componentes internos AWR/SYSAUX.

Hasta este punto **no se encontró corrupción en objetos de aplicación**.

---

# 12. Validaciones RMAN realizadas

## 12.1 `VALIDATE DATAFILE 2`

Dentro de RMAN:

```text
VALIDATE DATAFILE 2;
```

Salida resumida:

```text
Starting validate at 16-SEP-26
input datafile file number=00002 name=/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf
validation complete, elapsed time: 00:00:07

File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
2    OK     4              23492        156174          5967441630

Block Type Blocks Failing Blocks Processed
Data       0              49102
Index      0              51455
Other      0              32111
```

Interpretación:

- el datafile completo pudo leerse;
- `File Status = OK`;
- `Blocks Failing = 0`;
- permanecían 4 bloques marcados como corruptos por Oracle.

## 12.2 Efecto sobre `V$DATABASE_BLOCK_CORRUPTION`

Después del `VALIDATE DATAFILE 2`, la vista quedó reducida a:

```text
FILE# BLOCK# BLOCKS CORRUPTION_CHANGE# CORRUPTION_TYPE
2     76214  1      3227698275         NOLOGGING
2     76228  1      3227698275         NOLOGGING
2     76269  1      3227698275         NOLOGGING
2     76273  1      3227698275         NOLOGGING
```

Desaparecieron:

- el bloque 64812 `CHECKSUM`;
- los bloques de `WRH$_PARAMETER`;
- los bloques de `WRH$_LATCH`.

## 12.3 `VALIDATE CHECK LOGICAL DATAFILE 2`

Dentro de RMAN:

```text
VALIDATE CHECK LOGICAL DATAFILE 2;
```

Salida resumida:

```text
validation complete, elapsed time: 00:00:03

File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
2    OK     4              23492        156174          5967442388

Block Type Blocks Failing Blocks Processed
Data       0              49118
Index      0              51441
Other      0              32109
```

Conclusión:

- no hay bloques que fallen la lectura física;
- no hay bloques que fallen la validación lógica;
- persisten 4 marcas `NOLOGGING` dentro del mismo LOB AWR.

No se ha ejecutado `BLOCKRECOVER`.

---

# 13. Estado y configuración de AWR

## 13.1 Snapshots existentes

Consulta:

```sql
select min(snap_id),max(snap_id),min(begin_interval_time),max(end_interval_time),count(*)
from dba_hist_snapshot;
```

Resultado:

```text
MIN_SNAP_ID  130150
MAX_SNAP_ID  130354
MIN_BEGIN     07-SEP-26 10.00.33.553 PM
MAX_END       16-SEP-26 11.00.21.748 AM
COUNT         205
```

## 13.2 Retención y frecuencia

Consulta correcta para Oracle 11g:

```sql
select snap_interval, retention from dba_hist_wr_control;
```

Resultado:

```text
SNAP_INTERVAL  +00000 01:00:00.0
RETENTION      +00008 00:00:00.0
```

Interpretación:

- snapshot cada 1 hora;
- retención de 8 días.

## 13.3 Baselines

Consulta:

```sql
select baseline_id,baseline_name,baseline_type,start_snap_id,end_snap_id,expiration
from dba_hist_baseline
order by baseline_id;
```

Resultado único:

```text
BASELINE_ID    0
BASELINE_NAME  SYSTEM_MOVING_WINDOW
BASELINE_TYPE  MOVING_WINDOW
START_SNAP_ID  130163
END_SNAP_ID    130354
```

No se encontraron baselines AWR estáticas creadas por usuario.

---

# 14. Evaluación de `PURGE_SQL_DETAILS`

Se confirmó que existe en esta instalación:

```sql
select procedure_name
from all_procedures
where owner='SYS'
  and object_name='DBMS_WORKLOAD_REPOSITORY'
  and procedure_name='PURGE_SQL_DETAILS';
```

Resultado:

```text
PURGE_SQL_DETAILS
```

Se verificó si había filas huérfanas de `WRH$_SQL_PLAN`:

```sql
select count(*)
from sys.wrh$_sql_plan p
where not exists (
  select 1
  from sys.wrh$_sqlstat s
  where s.dbid=p.dbid
    and s.sql_id=p.sql_id
);
```

Resultado:

```text
COUNT(*) = 0
```

Conclusión:

**no ejecutar `DBMS_WORKLOAD_REPOSITORY.PURGE_SQL_DETAILS` en este punto**, porque no hay filas candidatas según ese criterio.

---

# 15. Hallazgo crítico sobre `OTHER_XML`

Consulta:

```sql
select count(*)
from sys.wrh$_sql_plan
where other_xml is not null;
```

Resultado:

```text
COUNT(*) = 0
```

Por tanto, actualmente ninguna fila de `SYS.WRH$_SQL_PLAN` contiene un valor `OTHER_XML` no nulo, aunque el segmento LOB físico de esa columna conserva los bloques `NOLOGGING` señalados.

Esto sugería inicialmente espacio LOB viejo o reutilizable, pero la prueba siguiente confirmó que el problema sigue activo.

---

# 16. Confirmación de que el error reaparece después del reinicio

Se tomó la última línea de arranque del `alert.log`:

```bash
ALERT=/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
START=$(grep -n "Starting ORACLE instance" "$ALERT" | tail -1 | cut -d: -f1)
```

Luego:

```bash
tail -n +$START "$ALERT" | egrep "ORA-01578|ORA-01110|ORA-26040"
```

El error volvió a aparecer después del reinicio:

```text
ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

Conclusión: el problema no es una marca histórica pasiva. Algún proceso de Oracle sigue intentando utilizar el bloque 76214.

---

# 17. Correlación con MMON/AWR

Se inspeccionó el contexto del error en el `alert.log`:

```bash
tail -n +"$START" "$ALERT" | grep -n -B 10 -A 6 "block # 76214"
```

Salida relevante:

```text
104-Wed Sep 16 11:00:22 2026
105-Errors in file /home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc  (incident=1405070):
106-ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
107-ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
108-ORA-26040: Data block was loaded using the NOLOGGING option
109-Incident details in: /home/oracle/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_1405070/orcl_m000_3559_i1405070.trc
```

El error ocurre inmediatamente después del cierre del snapshot AWR cuyo `MAX(END_INTERVAL_TIME)` era aproximadamente:

```text
16-SEP-26 11.00.21.748 AM
```

El error aparece a las:

```text
11:00:22
```

Esto hizo sospechar directamente de la tarea automática AWR/MMON.

---

# 18. Trace `orcl_m000_3559.trc`: causa inmediata confirmada

Ruta:

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc
```

Comando usado:

```bash
TRACE=/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc
egrep -n -B 20 -A 40 "Current SQL|SQL ID|sql_id|ORA-01578|ORA-26040|WRH._SQL_PLAN|OTHER_XML|Call Stack Trace" "$TRACE"
```

Salida relevante:

```text
*** 2026-09-16 11:00:22.233
*** SESSION ID:(90.48)
*** SERVICE NAME:(SYS$BACKGROUND)
*** MODULE NAME:(MMON_SLAVE)
*** ACTION NAME:(Auto-Flush Slave Action)

Byte offset to file# 2 block# 76214 is 624345088

ORA-01578: ORACLE data block corrupted (file # 2, block # 76214)
ORA-01110: data file 2: '/home/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option

*** KEWROCISTMTEXEC - encountered error

SQLSTR:
INSERT INTO wrh$_sql_plan sp (...)
```

Conclusión confirmada:

- el error ocurre dentro de un proceso `MMON_SLAVE`;
- la acción es `Auto-Flush Slave Action`;
- no es una consulta de usuario;
- el fallo ocurre durante un `INSERT INTO wrh$_sql_plan` ejecutado por AWR;
- el proceso intenta reutilizar/usar el LOB asociado y encuentra el bloque `NOLOGGING` 76214.

Esta es la causa inmediata del error repetitivo cada hora.

---

# 19. Definición actual del LOB afectado

Consulta:

```sql
select l.segment_name,l.index_name,l.tablespace_name,l.chunk,l.pctversion,
       l.retention,l.cache,l.logging,l.in_row,l.securefile,
       round(s.bytes/1024/1024,2) mb
from dba_lobs l
left join dba_segments s
  on s.owner=l.owner
 and s.segment_name=l.segment_name
where l.owner='SYS'
  and l.table_name='WRH$_SQL_PLAN'
  and l.column_name='OTHER_XML';
```

Resultado:

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

Interpretación:

- es un **BasicFile LOB** (`SECUREFILE=NO`);
- está en `SYSAUX`;
- usa `LOGGING=YES` actualmente;
- tamaño aproximado 19 MB;
- el segmento LOB físico contiene los bloques marcados `NOLOGGING`.

---

# 20. Particionamiento de `WRH$_SQL_PLAN`

Consulta:

```sql
select t.partitioned,p.partitioning_type,p.subpartitioning_type,p.partition_count
from dba_tables t
left join dba_part_tables p
  on p.owner=t.owner
 and p.table_name=t.table_name
where t.owner='SYS'
  and t.table_name='WRH$_SQL_PLAN';
```

Resultado:

```text
PARTITIONED = NO
```

Conclusión:

`SYS.WRH$_SQL_PLAN` **no está particionada**. No será necesario manejar `MOVE PARTITION` o particiones LOB individualmente si finalmente se decide reconstruir/mover el LOB.

---

# 21. Índices actuales de `WRH$_SQL_PLAN`

Consulta:

```sql
select index_name,index_type,status,tablespace_name,uniqueness
from dba_indexes
where table_owner='SYS'
  and table_name='WRH$_SQL_PLAN'
order by index_name;
```

Resultado:

```text
INDEX_NAME                    INDEX_TYPE  STATUS  TABLESPACE  UNIQUENESS
SYS_IL0000006213C00038$$      LOB         VALID   SYSAUX      UNIQUE
WRH$_SQL_PLAN_PK              NORMAL      VALID   SYSAUX      UNIQUE
```

Estado actual:

- índice LOB: `SYS_IL0000006213C00038$$` — `VALID`;
- índice normal/PK: `WRH$_SQL_PLAN_PK` — `VALID`;
- ambos están en `SYSAUX`;
- todavía no se ha ejecutado ningún `ALTER TABLE ... MOVE` ni `ALTER INDEX ... REBUILD`.

---

# 22. Diagnóstico consolidado actual

Hasta este punto, la evidencia permite afirmar lo siguiente:

1. La VM `kanela` está nuevamente operativa en la red actual.
2. CentOS 5.11 arranca y tiene almacenamiento suficiente.
3. SSH desde `zapallo` funciona con compatibilidad KEX limitada a `kanela` y autenticación por clave.
4. Oracle 11.2.0.1 abre normalmente.
5. La base está `OPEN`, `ACTIVE`, `READ WRITE`.
6. El datafile 2 `sysaux01.dbf` se valida completo con RMAN.
7. RMAN no encuentra bloques que fallen físicamente ni lógicamente.
8. El único problema persistente son cuatro bloques `NOLOGGING`:
   - 76214
   - 76228
   - 76269
   - 76273
9. Los cuatro pertenecen al LOB `OTHER_XML` de `SYS.WRH$_SQL_PLAN`.
10. `OTHER_XML IS NOT NULL` devuelve 0 filas actualmente.
11. No hay filas candidatas a `PURGE_SQL_DETAILS`.
12. El error reaparece después del reinicio.
13. Se dispara a la hora de auto-flush AWR.
14. El trace confirma `MODULE NAME: MMON_SLAVE` y `ACTION NAME: Auto-Flush Slave Action`.
15. El SQL que falla es un `INSERT INTO wrh$_sql_plan`.
16. El LOB es BasicFile, 19 MB, no particionado.
17. Los índices actuales están `VALID`.
18. No hay evidencia de corrupción activa en tablas de aplicación.

Hipótesis de trabajo actual:

> El segmento BasicFile LOB `SYS_LOB0000006213C00038$$`, correspondiente a `SYS.WRH$_SQL_PLAN.OTHER_XML`, contiene bloques históricos marcados `NOLOGGING`. Aunque no hay valores `OTHER_XML` activos, el auto-flush de AWR intenta insertar nuevas filas en `WRH$_SQL_PLAN`, reutiliza espacio del LOB y encuentra el bloque 76214, produciendo `ORA-01578/ORA-26040`.

---

# 23. Operaciones que NO se han ejecutado

No se ha ejecutado ninguna de las siguientes operaciones:

```text
BLOCKRECOVER CORRUPTION LIST
RECOVER DATAFILE
RESETLOGS
recreación de controlfiles
borrado o reemplazo de sysaux01.dbf
DELETE directo sobre SYS.WRH$_SQL_PLAN
DELETE directo sobre otras SYS.WRH$_*
DROP de objetos AWR
DROP_SNAPSHOT_RANGE
PURGE_SQL_DETAILS
ALTER TABLE SYS.WRH$_SQL_PLAN MOVE
ALTER TABLE ... MOVE LOB
ALTER INDEX ... REBUILD
```

Mantener esta política hasta que el plan DDL esté completamente definido y verificado para Oracle 11.2.0.1.

---

# 24. Próximo paso exacto

La misión está detenida justo antes de cualquier reparación DDL.

Ya sabemos:

```text
WRH$_SQL_PLAN no está particionada.
LOB OTHER_XML: BasicFile, ~19 MB, SYSAUX.
LOB index: SYS_IL0000006213C00038$$ VALID.
PK index: WRH$_SQL_PLAN_PK VALID.
```

El siguiente trabajo debe ser:

1. definir una estrategia soportada para Oracle 11.2.0.1 que cree un **segmento LOB nuevo** para `OTHER_XML` sin tocar datos de aplicación;
2. determinar si basta `ALTER TABLE ... MOVE LOB (OTHER_XML) STORE AS (...)` o si es necesario mover también la tabla;
3. comprobar el efecto esperado sobre `WRH$_SQL_PLAN_PK` y preparar `ALTER INDEX ... REBUILD` solo si realmente queda `UNUSABLE`;
4. considerar pausar temporalmente los snapshots automáticos de AWR durante la ventana de reparación para evitar que `MMON_SLAVE` escriba mientras se realiza el cambio;
5. después de cualquier reparación, validar en este orden:
   - `DBA_LOBS`: confirmar nuevo `SEGMENT_NAME` / `INDEX_NAME`;
   - `DBA_INDEXES`: todos `VALID`;
   - `V$DATABASE_BLOCK_CORRUPTION`;
   - RMAN `VALIDATE DATAFILE 2`;
   - RMAN `VALIDATE CHECK LOGICAL DATAFILE 2`;
   - forzar o esperar un snapshot AWR y comprobar que no reaparece `ORA-01578/ORA-26040`;
   - revisar `alert_orcl.log` solo desde el momento de la reparación.

**No ejecutar todavía el MOVE hasta validar la sintaxis exacta para esta versión.**

---

# 25. Comandos rápidos para retomar en un nuevo chat

## Estado de la base

```sql
select instance_name,status,database_status from v$instance;
select name,open_mode from v$database;
```

## Corrupción conocida

```sql
select file#,block#,blocks,corruption_change#,corruption_type
from v$database_block_corruption
where file#=2
order by block#;
```

## LOB afectado

```sql
select l.segment_name,l.index_name,l.tablespace_name,l.chunk,l.pctversion,
       l.retention,l.cache,l.logging,l.in_row,l.securefile,
       round(s.bytes/1024/1024,2) mb
from dba_lobs l
left join dba_segments s
  on s.owner=l.owner
 and s.segment_name=l.segment_name
where l.owner='SYS'
  and l.table_name='WRH$_SQL_PLAN'
  and l.column_name='OTHER_XML';
```

## Índices

```sql
select index_name,index_type,status,tablespace_name,uniqueness
from dba_indexes
where table_owner='SYS'
  and table_name='WRH$_SQL_PLAN'
order by index_name;
```

## Errores desde el último arranque

```bash
ALERT=/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
START=$(grep -n "Starting ORACLE instance" "$ALERT" | tail -1 | cut -d: -f1)
tail -n +$START "$ALERT" | egrep "ORA-01578|ORA-01110|ORA-26040"
```

## Trace del incidente conocido

```text
/home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_3559.trc
```

Incidentes conocidos:

```text
1405070
1405071
```

---

# 26. Regla de continuidad de esta misión

Al abrir un nuevo chat para continuar esta reparación:

1. leer `AGENTS.md`;
2. leer `missions/reparacion202609/mission.md`;
3. continuar desde la sección **Próximo paso exacto**;
4. no pedir nuevamente información ya documentada;
5. no asumir que el problema es corrupción de datos de aplicación;
6. mantener español neutral y evitar regionalismos;
7. registrar en esta misma misión cada nuevo hallazgo o cambio ejecutado.
