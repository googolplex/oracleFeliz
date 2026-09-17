# Progreso — DBMS_AUDIT_MGMT y estado de inicialización (2026-09-16)

## Estado confirmado

- `SYS.DBMS_AUDIT_MGMT` existe y está `VALID` tanto como `PACKAGE` como `PACKAGE BODY`.
- `DBA_AUDIT_MGMT_CONFIG_PARAMS` devuelve parámetros configurados para auditoría estándar, FGA, OS y XML.
- Para `STANDARD AUDIT TRAIL` se observa:
  - `DB AUDIT CLEAN BATCH SIZE = 10000`
  - `DB AUDIT TABLESPACE = SYSAUX`
- No aparece un parámetro `CLEANUP INTERVAL` para `STANDARD AUDIT TRAIL` en la salida observada.
- La tabla `SYS.AUD$` sigue físicamente en `SYSTEM` según el diagnóstico previo.

## Interpretación

La presencia de parámetros de configuración no demuestra por sí sola que `DBMS_AUDIT_MGMT.INIT_CLEANUP` haya sido ejecutado. La ausencia de un `CLEANUP INTERVAL` es un indicio de que la limpieza puede no estar inicializada, porque `INIT_CLEANUP` establece ese intervalo. Se verificará el estado mediante `DBMS_AUDIT_MGMT.IS_CLEANUP_INITIALIZED` antes de cualquier modificación.

No ejecutar todavía `INIT_CLEANUP`: Oracle documenta que, si el audit trail está en `SYSTEM`, la inicialización puede moverlo a `SYSAUX`. En esta base `SYSAUX` tenía aproximadamente 114 MB libres internos mientras `SYS.AUD$` ocupaba aproximadamente 250 MB, por lo que no debe iniciarse ese movimiento sin planificación de capacidad.

## Siguiente diagnóstico autorizado

Verificar explícitamente si la limpieza del audit trail estándar ya fue inicializada mediante `DBMS_AUDIT_MGMT.IS_CLEANUP_INITIALIZED(DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD)`.

## Seguridad

Sigue prohibido ejecutar `INIT_CLEANUP`, `SET_AUDIT_TRAIL_LOCATION`, `CLEAN_AUDIT_TRAIL`, `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de tablespaces/datafiles o cambios de RMAN hasta completar esta verificación y el plan de respaldo/retención.