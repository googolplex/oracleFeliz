# Reglas del proyecto oracleFeliz

## Idioma y estilo
- Usar español neutral en todas las explicaciones, notas, diagnósticos y documentación.
- Evitar regionalismos, modismos locales y formas verbales regionales.
- Mantener instrucciones técnicas claras, directas y reproducibles.

## Política de diagnóstico y reparación
- Priorizar diagnóstico de solo lectura antes de cualquier cambio destructivo.
- No ejecutar ni recomendar operaciones de recuperación agresivas sin identificar antes el objeto afectado y el estado real de la base.
- Evitar `RESETLOGS`, recreación de controlfiles, borrado de datafiles, `DROP`, `DELETE` directo sobre tablas internas `SYS.WRH$_*`, `startup force` o recuperaciones invasivas salvo evidencia suficiente y plan explícito.
- Antes de modificar AWR, preferir APIs soportadas de Oracle como `DBMS_WORKLOAD_REPOSITORY`.
- Antes de recuperar bloques con RMAN, distinguir corrupción física real de bloques marcados `NOLOGGING`.
- Conservar siempre evidencia de comandos, salidas, rutas, versiones, bloques y objetos involucrados.

## Entorno principal conocido
- Host físico/KVM: `zapallo`.
- VM Oracle: `kanela`.
- `kanela` está conectada a bridge `br0` en `zapallo`.
- MAC de `kanela`: `54:52:00:13:4f:94`.
- IP actual de `zapallo`: `192.168.1.71`.
- IP actual de `kanela`: `192.168.1.60`.
- Gateway de la red: `192.168.1.1`.
- Para SSH desde `zapallo` a `kanela`, CentOS 5 requiere compatibilidad con `diffie-hellman-group14-sha1`; se configuró acceso por clave para evitar contraseña repetida.

## Regla de continuidad
- La misión maestra actual es `missions/reparacion202609/mission.md`.
- Antes de continuar esta reparación de Oracle, leer `missions/reparacion202609/mission.md` y también `missions/reparacion202609/CURRENT.md`.
- Continuar desde el punto exacto documentado allí, sin pedir nuevamente contexto ya registrado.
- La misión anterior `missions/kanela-orcl/mission-01-recovery.md` queda como antecedente histórico; no usarla como punto de continuidad si existe información más reciente en `reparacion202609`.
- Registrar en `missions/reparacion202609/mission.md` y/o `CURRENT.md` los nuevos hallazgos, cambios ejecutados, validaciones y estado final de la reparación.
