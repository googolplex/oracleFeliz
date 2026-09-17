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

El resto de combinaciones aparece solo en cantidades pequeñas comparadas con `ADMIN`.

Interpretación:

- enero de 2019: `ADMIN` produjo **327.698** eventos `LOGON` + `LOGOFF` exitosos, aproximadamente **99,8%** de todos los registros del mes;
- junio de 2019: `ADMIN` produjo **376.985** eventos `LOGON` + `LOGOFF` exitosos, aproximadamente **99,94%** de todos los registros del mes;
- combinados, son **704.683 eventos**, aproximadamente **99,88%** de los 705.549 registros de ambos meses;
- `RETURNCODE=0` demuestra que se trató predominantemente de conexiones exitosas, no de errores masivos de autenticación;
- los pocos `RETURNCODE=1017` observados son marginales y no explican la ocupación histórica;
- el patrón es consistente con un proceso o aplicación que abría y cerraba sesiones Oracle de forma extremadamente frecuente usando la cuenta `ADMIN`;
- no se infiere todavía qué aplicación o máquina era responsable: debe identificarse el origen mediante `USERHOST`, `OS_USERNAME` y `TERMINAL` antes de concluir la causa operativa.

### Siguiente diagnóstico autorizado

Consulta de solo lectura para identificar el origen cliente de los `LOGON`/`LOGOFF` exitosos de `ADMIN` en enero y junio de 2019:

```sql
SELECT * FROM (SELECT TO_CHAR(timestamp,'YYYY-MM') audit_month, NVL(userhost,'<NULL>') userhost, NVL(os_username,'<NULL>') os_username, NVL(terminal,'<NULL>') terminal, action_name, COUNT(*) audit_rows FROM dba_audit_trail WHERE username='ADMIN' AND returncode=0 AND action_name IN ('LOGON','LOGOFF') AND ((timestamp >= DATE '2019-01-01' AND timestamp < DATE '2019-02-01') OR (timestamp >= DATE '2019-06-01' AND timestamp < DATE '2019-07-01')) GROUP BY TO_CHAR(timestamp,'YYYY-MM'), NVL(userhost,'<NULL>'), NVL(os_username,'<NULL>'), NVL(terminal,'<NULL>'), action_name ORDER BY audit_rows DESC) WHERE ROWNUM <= 30;
```

Objetivos:

- identificar el host o hosts que originaron la avalancha de conexiones;
- identificar, si está disponible, el usuario del sistema operativo y terminal cliente;
- determinar si enero y junio provienen del mismo origen;
- separar el diagnóstico histórico del problema actual: desde 2020 el volumen de auditoría es bajo;
- solo después evaluar una política segura de retención/archivado/purga de `AUD$`.

Estado: **resultado pendiente**.

No se autoriza todavía ninguna purga, movimiento de `AUD$`, modificación de auditoría ni cambio estructural de tablespaces.

## Regla de seguridad

No ejecutar todavía `ALTER DATABASE DATAFILE`, `ALTER TABLESPACE ... ADD DATAFILE`, reducción de datafiles, cambios de `AUTOEXTEND`, cambios de RMAN, `TRUNCATE`/`DELETE` sobre `SYS.AUD$`, ni eliminación de respaldos. Esta misión continúa exclusivamente en modo diagnóstico y planificación.
