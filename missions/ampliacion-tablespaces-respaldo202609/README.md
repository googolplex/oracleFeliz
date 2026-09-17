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

Se ejecutó correctamente la consulta de los 20 segmentos de mayor tamaño por tablespace:

```sql
SELECT tablespace_name, owner, segment_name, segment_type, ROUND(bytes/1024/1024,2) mb FROM (SELECT tablespace_name, owner, segment_name, segment_type, bytes, ROW_NUMBER() OVER (PARTITION BY tablespace_name ORDER BY bytes DESC) rn FROM dba_segments WHERE tablespace_name IN ('SYSTEM','SYSAUX')) WHERE rn <= 20 ORDER BY tablespace_name, mb DESC;
```

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
- Sin embargo, `SYS.AUD$` con **250 MB** es un consumidor particularmente relevante: representa aproximadamente una quinta parte del tamaño actual de `SYSTEM` (1190 MB).
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
- objetos `WRH$_*` de AWR entre los mayores restantes, incluyendo `WRH$_SYSMETRIC_HISTORY` y `WRH$_SQL_PLAN_PK`.

Interpretación provisional:

- Los mayores consumidores observados en `SYSAUX` pertenecen a componentes internos esperables: optimizer statistics history, Advisor, Scheduler, XDB, MDSYS y AWR.
- En esta primera inspección no aparece un segmento de aplicación evidente ni un único objeto anómalo que justifique una modificación inmediata.
- Se mantiene la política de no borrar ni mover objetos internos de `SYSAUX` sin diagnóstico específico.

### Siguiente diagnóstico autorizado

Prioridad inmediata: caracterizar `SYS.AUD$` antes de continuar con decisiones sobre `SYSTEM`.

Consulta solicitada, solo lectura:

```sql
SELECT (SELECT value FROM v$parameter WHERE name='audit_trail') audit_trail, COUNT(*) audit_rows, MIN(timestamp#) oldest_audit, MAX(timestamp#) newest_audit FROM sys.aud$;
```

Objetivos:

- confirmar la configuración actual de `AUDIT_TRAIL`;
- contar las filas almacenadas en `SYS.AUD$`;
- identificar la fecha de la auditoría más antigua y la más reciente;
- decidir si corresponde estudiar retención/limpieza del audit trail antes de cualquier ampliación de `SYSTEM`.

Estado: **resultado pendiente**.

No se autoriza todavía ninguna modificación estructural.

## Regla de seguridad

No ejecutar todavía `ALTER DATABASE DATAFILE`, `ALTER TABLESPACE ... ADD DATAFILE`, reducción de datafiles, cambios de `AUTOEXTEND`, cambios de RMAN ni eliminación de respaldos. Esta misión comienza exclusivamente en modo diagnóstico y planificación.
