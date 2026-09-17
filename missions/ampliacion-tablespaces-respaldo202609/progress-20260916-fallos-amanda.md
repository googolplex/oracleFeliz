# Progreso — fallos recientes de autenticación de AMANDA (2026-09-16)

## Evidencia

Se analizó la auditoría de los últimos 12 meses para `LOGON` con `RETURNCODE <> 0`.

Resultado:

```text
USERNAME     AMANDA
USERHOST     suzuka.lcompras.biz
OS_USERNAME  tomcat6
RETURNCODE   1017
AUDIT_ROWS   63
FIRST_SEEN   21-SEP-25
LAST_SEEN    13-SEP-26

USERNAME     AMANDA
USERHOST     suzuka.lcompras.biz
OS_USERNAME  tomcat6
RETURNCODE   28000
AUDIT_ROWS   1
FIRST_SEEN   24-MAY-26
LAST_SEEN    24-MAY-26
```

## Interpretación

- Los 64 fallos recientes de autenticación provienen del mismo origen conocido: usuario Oracle `AMANDA`, host `suzuka.lcompras.biz`, proceso/usuario de sistema operativo `tomcat6`.
- Los 63 `ORA-01017` son intentos con credenciales inválidas.
- El único `ORA-28000` corresponde a cuenta bloqueada.
- No aparece un patrón distribuido entre múltiples usuarios u hosts.
- Esta evidencia refuerza que conviene conservar auditoría de `CREATE SESSION` fallida, porque detecta problemas reales de credenciales/aplicación.
- Las conexiones exitosas rutinarias continúan siendo el principal candidato a dejar de auditar más adelante, mientras que los privilegios administrativos sensibles y los fallos de sesión deben preservarse.

## Siguiente diagnóstico autorizado

Verificar el estado actual de la cuenta `AMANDA` antes de modificar la política de auditoría:

```sql
SELECT username, account_status, lock_date, expiry_date, profile, created FROM dba_users WHERE username='AMANDA';
```

Objetivos:

- confirmar si `AMANDA` está actualmente abierta, bloqueada o expirada;
- relacionar el episodio `ORA-28000` con el estado real de la cuenta;
- decidir si existe una credencial antigua o configuración de aplicación que deba corregirse en `suzuka.lcompras.biz`;
- mantener el diagnóstico separado de cualquier cambio de auditoría o purga.

## Seguridad

Continúa prohibido ejecutar `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de `AUDIT_TRAIL`, movimientos de `AUD$`, cambios de tablespaces/datafiles o cambios de RMAN. La misión sigue en diagnóstico y planificación.
