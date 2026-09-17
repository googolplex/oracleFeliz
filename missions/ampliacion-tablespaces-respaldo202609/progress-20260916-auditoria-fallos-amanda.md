# Progreso — fallos recientes de autenticación y cierre de diagnóstico (2026-09-16)

## Resultado de fallos recientes

Se identificó el origen de los `LOGON` fallidos de los últimos 12 meses:

```text
USERNAME    = AMANDA
USERHOST    = suzuka.lcompras.biz
OS_USERNAME = tomcat6
RETURNCODE  = 1017
AUDIT_ROWS  = 63
FIRST_SEEN  = 21-SEP-25
LAST_SEEN   = 13-SEP-26

USERNAME    = AMANDA
USERHOST    = suzuka.lcompras.biz
OS_USERNAME = tomcat6
RETURNCODE  = 28000
AUDIT_ROWS  = 1
FIRST_SEEN  = 24-MAY-26
LAST_SEEN   = 24-MAY-26
```

Interpretación:

- todos los fallos recientes provienen del mismo origen conocido: `AMANDA` desde `suzuka.lcompras.biz`, proceso de sistema operativo `tomcat6`;
- `ORA-01017` corresponde a credenciales incorrectas;
- el único `ORA-28000` corresponde a una cuenta bloqueada en ese momento;
- no existe evidencia en esta ventana de múltiples orígenes externos o de un patrón distribuido de intentos fallidos;
- este tipo de evento sí tiene valor operativo y debe preservarse en la política futura de auditoría.

## Estado actual del usuario AMANDA

Se ejecutó:

```sql
SELECT username, account_status, lock_date, expiry_date, profile, created FROM dba_users WHERE username='AMANDA';
```

Resultado:

```text
USERNAME       = AMANDA
ACCOUNT_STATUS = OPEN
LOCK_DATE      = NULL
EXPIRY_DATE    = NULL
PROFILE        = DEFAULT
CREATED        = 09-OCT-12
```

Conclusiones:

- `AMANDA` está actualmente `OPEN`;
- no existe bloqueo actual ni expiración registrada;
- el `ORA-28000` del 24-MAY-2026 fue transitorio y ya no está vigente;
- los `ORA-01017` son compatibles con intentos ocasionales de `tomcat6` usando credenciales antiguas o incorrectas;
- no se modifica todavía la cuenta ni la aplicación; la investigación de auditoría puede cerrarse y pasar a planificación de retención/purga.

## Política preliminar consolidada — aún no ejecutada

1. Mantener auditoría de fallos de `CREATE SESSION`.
2. Mantener auditoría de operaciones/privilegios administrativos sensibles.
3. Evaluar dejar de auditar sesiones exitosas rutinarias para reducir ruido.
4. Definir una ventana limitada de retención del audit trail en lugar de conservar 17 años.
5. Realizar respaldo/exportación del histórico que se decida conservar antes de cualquier purga.
6. Usar un mecanismo soportado de Oracle para limpieza; no `DELETE` ni `TRUNCATE` directo sobre `SYS.AUD$`.

## Siguiente diagnóstico autorizado

Antes de diseñar comandos de purga, verificar si el paquete soportado de administración de auditoría está disponible y válido en esta instalación Oracle 11.2.0.1:

```sql
SELECT owner, object_name, object_type, status FROM dba_objects WHERE owner='SYS' AND object_name='DBMS_AUDIT_MGMT' ORDER BY object_type;
```

Estado: **resultado pendiente**.

## Seguridad

Continúan prohibidos por ahora `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de `AUDIT_TRAIL`, movimientos de `AUD$`, cambios de tablespaces/datafiles y cambios de RMAN. Se mantiene una consulta o comando por vez.
