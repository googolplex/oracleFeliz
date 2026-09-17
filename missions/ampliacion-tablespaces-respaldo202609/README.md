# Misión — ampliación de tablespaces y revisión de respaldo (2026-09)

## Objetivo

Planificar y ejecutar de forma conservadora la ampliación de capacidad de Oracle `orcl`, con atención prioritaria al tablespace `TABLAS`, y revisar/validar los procedimientos de respaldo antes de realizar cambios estructurales.

## Contexto de entrada

Esta misión continúa después de `missions/reparacion202609/`.

Estado de referencia al 2026-09-16:

- Oracle `orcl`: `OPEN`, `ACTIVE`, `READ WRITE`.
- Reparación AWR/LOB completada y validada.
- `SYSTEM`: 1190 MB, 1183.63 MB usados, 6.38 MB libres, `AUTOEXTEND YES`, incremento 10 MB.
- `SYSAUX`: 1220 MB, 1105.88 MB usados, 114.13 MB libres, `AUTOEXTEND YES`, incremento 10 MB.
- `TABLAS`: `SMALLFILE`, datafile actual de 32767.98 MB, ya en `MAXBYTES`, con aproximadamente 18.28 GB libres internos.
- `/`: aproximadamente 23 GB libres.
- `/ciruelas`: aproximadamente 51 GB libres.
- No ampliar, reducir ni agregar datafiles sin evidencia y planificación previa.

## Alcance

1. Revisar crecimiento y ocupación real de `SYSTEM`, `SYSAUX`, `TABLAS` y demás tablespaces relevantes.
2. Determinar necesidades futuras de capacidad y márgenes de seguridad.
3. Para `TABLAS`, evaluar la incorporación de un segundo datafile `SMALLFILE` en `/ciruelas` cuando la evidencia lo justifique; no intentar ampliar el datafile actual, que ya alcanzó su `MAXBYTES`.
4. Revisar la distribución física de datafiles y el espacio real disponible en los filesystems.
5. Revisar los procedimientos actuales de respaldo de la VM, Oracle y sus datafiles.
6. Verificar especialmente:
   - qué se respalda;
   - periodicidad;
   - ubicación y retención;
   - consistencia de los respaldos;
   - posibilidad real de restauración;
   - relación entre respaldo de VM y respaldo Oracle/RMAN;
   - implicaciones de que la base opere actualmente en `NOARCHIVELOG`.
7. Definir un procedimiento de respaldo/restauración documentado y verificable antes de cualquier ampliación estructural importante.

## Método de trabajo obligatorio

- Diagnóstico primero, solo lectura.
- Una consulta o comando por vez.
- SQL*Plus preferentemente en una sola línea terminada en `;`.
- No modificar ningún tablespace, datafile, configuración RMAN ni procedimiento de respaldo antes de analizar la evidencia.
- Mantener aislamiento entre diagnóstico, planificación y ejecución.
- Registrar en GitHub cada hallazgo relevante y cada decisión tomada.
- Antes de ejecutar cambios, disponer de un procedimiento de reversión/restauración adecuado y validado.

## Prioridad inicial

1. Terminar el análisis de ocupación y crecimiento de `SYSTEM` y `SYSAUX` iniciado en `reparacion202609`.
2. Levantar el estado exacto de `TABLAS`: segmentos, crecimiento, HWM, consumo histórico y proyección.
3. Inventariar y revisar el procedimiento de respaldo actual.
4. Solo después diseñar la ampliación de `TABLAS` y, si corresponde, de otros tablespaces.

## Estado de ejecución — 2026-09-16

### Diagnóstico de segmentos de `SYSTEM` y `SYSAUX`

Se ejecutó correctamente la consulta de los 20 segmentos de mayor tamaño por tablespace.

Resultado: 40 filas, 20 de `SYSAUX` y 20 de `SYSTEM`.

#### Hallazgos en `SYSTEM`

Principales segmentos observados:

- `SYS.AUD$` — TABLE — **250 MB**.
- `SYS.IDL_UB1$` — TABLE — **248 MB**.
- `SYS.SOURCE$` — TABLE — **72 MB**.
- `SYSTEM.SYS_LOB0000255229C00045$$` — LOBSEGMENT — **35 MB**.
- `SYS.IDL_UB2$` — TABLE — **33 MB**.
- `SYS.C_OBJ#_INTCOL#` — CLUSTER — **27 MB**.
- `SYS.C_TOID_VERSION#` — CLUSTER — **24 MB**.
- `SYS.C_OBJ#` — CLUSTER — **21 MB**.
- `SYS.I_SOURCE1` — INDEX — **15 MB**.
- `SYS.JAVA$MC$` — TABLE — **13 MB**.
- `SYS.OBJ$` — TABLE — **12 MB**.
- varios LOBSEGMENT del esquema `SYSTEM` — aproximadamente 12 MB cada uno.

Interpretación provisional:

- La mayor parte de los objetos grandes de `SYSTEM` corresponde al diccionario interno de Oracle.
- `SYS.AUD$` con **250 MB** es un consumidor relevante: aproximadamente una quinta parte del tamaño actual de `SYSTEM` (1190 MB).
- Antes de atribuir el problema a falta física de capacidad o ampliar `SYSTEM`, debe investigarse el volumen, antigüedad y configuración del audit trail.
- No purgar, mover ni modificar `SYS.AUD$` todavía.

#### Hallazgos en `SYSAUX`

Principales segmentos observados:

- `SYS.I_WRI$_OPTSTAT_H_OBJ#_ICOL#_ST` — INDEX — **60 MB**.
- `XDB.SYS_LOB0000056506C00025$$` — LOBSEGMENT — **57.13 MB**.
- `SYS.WRI$_ADV_MSG_GRPS_IDX_01` — INDEX — **54 MB**.
- `SYS.WRI$_ADV_MESSAGE_GROUPS_PK` — INDEX — **50 MB**.
- `SYS.SCHEDULER$_EVENT_LOG` — TABLE — **48 MB**.
- `SYS.WRI$_ADV_MESSAGE_GROUPS` — TABLE — **39 MB**.
- `SYS.WRI$_OPTSTAT_HISTGRM_HISTORY` — TABLE — **38 MB**.
- `SYS.I_WRI$_OPTSTAT_H_ST` — INDEX — **28 MB**.
- `SYS.WRI$_ADV_SQLT_PLANS` — TABLE — **21 MB**.
- LOBs internos SYS/MDSYS — aproximadamente 18–20 MB.
- objetos `WRH$_*` de AWR entre los mayores restantes.

Interpretación provisional:

- Los mayores consumidores observados en `SYSAUX` pertenecen a componentes internos esperables: optimizer statistics history, Advisor, Scheduler, XDB, MDSYS y AWR.
- En esta primera inspección no aparece un segmento de aplicación evidente ni un único objeto anómalo que justifique una modificación inmediata.
- Se mantiene la política de no borrar ni mover objetos internos de `SYSAUX` sin diagnóstico específico.

### Diagnóstico de `SYS.AUD$`

La consulta corregida confirmó:

```text
AUDIT_TRAIL = DB
AUDIT_ROWS  = 1216698
```

Luego se obtuvo la cronología mediante `DBA_AUDIT_TRAIL`:

```text
AUDIT_ROWS = 1216698
OLDEST_LOCAL = 15-AUG-09
NEWEST_LOCAL = 16-SEP-26
OLDEST_UTC = 15-AUG-09 03.25.27.182496 AM -04:00
NEWEST_UTC = 16-SEP-26 02.23.30.840963 AM -04:00
```

Hallazgos consolidados:

- el audit trail tradicional está configurado como `DB`;
- existen **1.216.698 filas** en `SYS.AUD$`;
- `SYS.AUD$` ocupa aproximadamente **250 MB** dentro de `SYSTEM`;
- la auditoría retenida abarca desde **15-AGO-2009** hasta **16-SEP-2026**, más de 17 años de historia;
- existe evidencia fuerte de acumulación histórica prolongada y la presión de espacio de `SYSTEM` no debe interpretarse únicamente como crecimiento normal del diccionario.

### Distribución anual del audit trail

Resultado por año:

```text
2009         12
2010     108921
2011      25755
2012      71626
2013     113500
2014     131065
2015      14220
2016      13939
2017      12556
2018       8764
2019     708292
2020       3704
2021       1598
2022        603
2023        597
2024        556
2025        582
2026        408
```

Interpretación:

- `2019` concentra **708.292 filas**, aproximadamente **58%** de las 1.216.698 entradas actuales.
- El volumen de auditoría cae de forma abrupta después de 2019.
- Desde 2020 el crecimiento anual es muy bajo; la ocupación actual está dominada por historia acumulada y no por crecimiento reciente acelerado.

### Distribución mensual de 2019

Resultado:

```text
2019-01    328344
2019-06    377205
2019-07       503
2019-08       475
2019-09       468
2019-10       327
2019-11       367
2019-12       603
```

Interpretación:

- enero de 2019 contiene **328.344** registros;
- junio de 2019 contiene **377.205** registros;
- juntos suman **705.549** registros de los **708.292** del año, aproximadamente **99,6%**;
- por tanto, el pico de 2019 no fue sostenido: estuvo concentrado casi por completo en dos episodios puntuales, enero y junio.

### Origen de los picos de enero y junio de 2019

Se agruparon los eventos por mes, usuario, acción y `RETURNCODE`.

Principales resultados:

```text
2019-06 ADMIN  LOGON   0   188646
2019-06 ADMIN  LOGOFF  0   188339
2019-01 ADMIN  LOGON   0   163935
2019-01 ADMIN  LOGOFF  0   163763
```

Interpretación:

- enero de 2019: `ADMIN` produjo **327.698** eventos `LOGON` + `LOGOFF` exitosos;
- junio de 2019: `ADMIN` produjo **376.985** eventos `LOGON` + `LOGOFF` exitosos;
- combinados, son **704.683 eventos**, aproximadamente **99,88%** de los 705.549 registros de ambos meses;
- `RETURNCODE=0` demuestra que se trató predominantemente de conexiones exitosas, no de errores masivos de autenticación.

La consulta por origen cliente confirmó como fuentes dominantes:

```text
2019-06  USERHOST=PARQUE\RODEOXP
         OS_USERNAME=caja
         TERMINAL=RODEOXP
         LOGON  = 188267
         LOGOFF = 188261

2019-01  USERHOST=GRUPO_TRABAJO\445F235B7F31467
         OS_USERNAME=xoldfusion
         TERMINAL=445F235B7F31467
         LOGON  = 163734
         LOGOFF = 163725
```

Los demás hosts/usuarios aparecen en cantidades marginales frente a esos dos orígenes.

### Confirmación funcional del usuario — causa conocida

El usuario confirmó que esos episodios correspondían a **imports/cargas temporales desde servidores de producción**. El propósito era disponer localmente de los datos necesarios para elaborar informes. Una vez terminados los uploads y el análisis, ya no existía necesidad funcional de conservar ese universo completo de datos ni una auditoría histórica tan extendida.

Conclusión de esta rama:

- el pico de 2019 tiene una explicación operativa conocida y coherente con la evidencia;
- no hace falta seguir investigando esos hosts como anomalía;
- el patrón de `LOGON`/`LOGOFF` masivo refleja el mecanismo de las cargas temporales realizadas entonces;
- desde 2020 la generación de auditoría es muy baja, por lo que el problema actual es principalmente **retención histórica acumulada**, no crecimiento reciente;
- no existe requerimiento funcional expresado por el usuario para conservar 17 años de audit trail;
- antes de purgar se debe definir qué auditoría debe seguir activa, qué ventana de retención conservar y qué respaldo/exportación se hará antes de eliminar histórico.

### Siguiente diagnóstico autorizado

La rama de atribución histórica queda cerrada. El siguiente paso es inventariar **qué auditoría de sentencias está habilitada actualmente**, en especial si continúa activo `AUDIT SESSION` u otras opciones que ya no sean necesarias.

Consulta de solo lectura:

```sql
SELECT user_name, proxy_name, audit_option, success, failure FROM dba_stmt_audit_opts ORDER BY user_name, proxy_name, audit_option;
```

Objetivos:

- identificar las opciones de auditoría estándar actualmente activas;
- verificar si `SESSION` sigue auditándose y bajo qué modalidad;
- distinguir auditoría global de auditoría específica por usuario;
- diseñar posteriormente una política de retención razonable;
- planificar, antes de cualquier purga, un respaldo/exportación del histórico que se decida conservar.

Estado: **resultado pendiente**.

No se autoriza todavía ninguna purga, movimiento de `AUD$`, modificación de auditoría ni cambio estructural de tablespaces.

## Regla de seguridad

No ejecutar todavía `ALTER DATABASE DATAFILE`, `ALTER TABLESPACE ... ADD DATAFILE`, reducción de datafiles, cambios de `AUTOEXTEND`, cambios de RMAN, `TRUNCATE`/`DELETE` sobre `SYS.AUD$`, comandos `NOAUDIT`, ni eliminación de respaldos. Esta misión continúa exclusivamente en modo diagnóstico y planificación.
