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

Se inició formalmente la fase de diagnóstico de ocupación de `SYSTEM` y `SYSAUX`.

Primera consulta diagnóstica solicitada, exclusivamente de lectura, para identificar los 20 segmentos de mayor tamaño en cada uno de esos tablespaces:

```sql
SELECT tablespace_name, owner, segment_name, segment_type, ROUND(bytes/1024/1024,2) mb FROM (SELECT tablespace_name, owner, segment_name, segment_type, bytes, ROW_NUMBER() OVER (PARTITION BY tablespace_name ORDER BY bytes DESC) rn FROM dba_segments WHERE tablespace_name IN ('SYSTEM','SYSAUX')) WHERE rn <= 20 ORDER BY tablespace_name, mb DESC;
```

Estado actual: **consulta preparada / resultado pendiente de captura**.

Objetivo inmediato de esta evidencia:

- verificar qué segmentos explican la ocupación de `SYSTEM` y `SYSAUX`;
- detectar objetos impropios o inesperados en `SYSTEM`;
- distinguir crecimiento normal del diccionario/AWR frente a crecimiento anómalo;
- decidir la siguiente consulta de diagnóstico antes de pasar a `TABLAS`.

No se autoriza todavía ninguna modificación estructural.

## Regla de seguridad

No ejecutar todavía `ALTER DATABASE DATAFILE`, `ALTER TABLESPACE ... ADD DATAFILE`, reducción de datafiles, cambios de `AUTOEXTEND`, cambios de RMAN ni eliminación de respaldos. Esta misión comienza exclusivamente en modo diagnóstico y planificación.
