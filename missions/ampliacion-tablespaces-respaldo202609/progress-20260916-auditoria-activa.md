# Progreso — inventario de auditoría activa (2026-09-16)

## Contexto

La investigación de `SYS.AUD$` determinó que el volumen histórico está dominado por cargas temporales realizadas para elaborar informes a partir de datos importados desde servidores de producción. El usuario confirmó que, terminados esos trabajos, no existe necesidad funcional de conservar indefinidamente ni el universo importado ni 17 años de auditoría.

## Auditoría de sentencias actualmente activa

Se ejecutó, solo lectura:

```sql
SELECT user_name, proxy_name, audit_option, success, failure FROM dba_stmt_audit_opts ORDER BY user_name, proxy_name, audit_option;
```

Resultado: **28 opciones**.

En todas las filas observadas `USER_NAME` y `PROXY_NAME` aparecen vacíos, por lo que las opciones son globales, no específicas de un usuario.

Todas aparecen con:

```text
SUCCESS = BY ACCESS
FAILURE = BY ACCESS
```

Opciones observadas:

```text
ALTER ANY PROCEDURE
ALTER ANY TABLE
ALTER DATABASE
ALTER PROFILE
ALTER SYSTEM
ALTER USER
CREATE ANY JOB
CREATE ANY LIBRARY
CREATE ANY PROCEDURE
CREATE ANY TABLE
CREATE EXTERNAL JOB
CREATE PUBLIC DATABASE LINK
CREATE SESSION
CREATE USER
DATABASE LINK
DROP ANY PROCEDURE
DROP ANY TABLE
DROP PROFILE
DROP USER
EXEMPT ACCESS POLICY
GRANT ANY OBJECT PRIVILEGE
GRANT ANY PRIVILEGE
GRANT ANY ROLE
PROFILE
PUBLIC SYNONYM
ROLE
SYSTEM AUDIT
SYSTEM GRANT
```

## Auditoría por privilegios actualmente activa

Se ejecutó, solo lectura:

```sql
SELECT user_name, proxy_name, privilege, success, failure FROM dba_priv_audit_opts ORDER BY user_name, proxy_name, privilege;
```

Resultado: **23 privilegios**.

También aquí `USER_NAME` y `PROXY_NAME` aparecen vacíos en todas las filas observadas: la configuración es global.

Todas las filas aparecen con:

```text
SUCCESS = BY ACCESS
FAILURE = BY ACCESS
```

Privilegios observados:

```text
ALTER ANY PROCEDURE
ALTER ANY TABLE
ALTER DATABASE
ALTER PROFILE
ALTER SYSTEM
ALTER USER
AUDIT SYSTEM
CREATE ANY JOB
CREATE ANY LIBRARY
CREATE ANY PROCEDURE
CREATE ANY TABLE
CREATE EXTERNAL JOB
CREATE PUBLIC DATABASE LINK
CREATE SESSION
CREATE USER
DROP ANY PROCEDURE
DROP ANY TABLE
DROP PROFILE
DROP USER
EXEMPT ACCESS POLICY
GRANT ANY OBJECT PRIVILEGE
GRANT ANY PRIVILEGE
GRANT ANY ROLE
```

## Auditoría por objetos

Se ejecutó, solo lectura:

```sql
SELECT owner, object_name, object_type, alt, aud, com, del, gra, ind, ins, loc, ren, sel, upd, ref, exe FROM dba_obj_audit_opts WHERE alt <> '-/-' OR aud <> '-/-' OR com <> '-/-' OR del <> '-/-' OR gra <> '-/-' OR ind <> '-/-' OR ins <> '-/-' OR loc <> '-/-' OR ren <> '-/-' OR sel <> '-/-' OR upd <> '-/-' OR ref <> '-/-' OR exe <> '-/-' ORDER BY owner, object_name;
```

Resultado:

```text
no rows selected
```

Conclusión: **no existe auditoría explícita por objetos** actualmente.

## Composición de la auditoría reciente — últimos 12 meses

Se ejecutó:

```sql
SELECT action_name, returncode, COUNT(*) audit_rows FROM dba_audit_trail WHERE timestamp >= ADD_MONTHS(TRUNC(SYSDATE),-12) GROUP BY action_name, returncode ORDER BY audit_rows DESC;
```

Resultado:

```text
LOGON    0      256
LOGOFF   0      256
LOGON 1017       63
LOGON 28000       1
```

Interpretación:

- el volumen reciente es bajo: solo **576** eventos auditados en 12 meses;
- existen **256 conexiones exitosas** y sus **256 LOGOFF** correspondientes;
- hubo **63 intentos fallidos ORA-01017** (`invalid username/password; logon denied`);
- hubo **1 ORA-28000** (`the account is locked`);
- no aparecen en esta ventana otras acciones administrativas en volumen material;
- por tanto, el problema de espacio de `AUD$` es histórico, no crecimiento reciente;
- los fallos de autenticación sí tienen valor operativo y conviene preservarlos;
- las conexiones exitosas rutinarias son el principal candidato a dejar de auditar cuando se pase a ejecución, mientras se mantienen los fallos de sesión y las operaciones administrativas sensibles.

## Política preliminar — aún no ejecutada

La evidencia disponible sugiere como diseño futuro:

1. Mantener auditoría de privilegios/operaciones administrativas sensibles.
2. Mantener auditoría de `CREATE SESSION` **cuando falle**.
3. Evaluar dejar de registrar `CREATE SESSION` exitoso rutinario para evitar ruido innecesario.
4. Definir una ventana de retención limitada del audit trail en vez de conservar 17 años indefinidamente.
5. Antes de purgar, realizar respaldo/exportación del histórico que se decida conservar y documentar el procedimiento de recuperación.
6. La purga debe ser controlada y mediante mecanismo soportado; no `DELETE`/`TRUNCATE` directo sobre `SYS.AUD$`.

No se autoriza todavía ningún cambio: esta política sigue siendo preliminar hasta identificar el origen de los fallos de autenticación recientes.

## Siguiente diagnóstico autorizado

Identificar usuario, host y usuario de sistema operativo asociados a los `LOGON` fallidos de los últimos 12 meses:

```sql
SELECT NVL(username,'<NULL>') username, NVL(userhost,'<NULL>') userhost, NVL(os_username,'<NULL>') os_username, returncode, COUNT(*) audit_rows, MIN(timestamp) first_seen, MAX(timestamp) last_seen FROM dba_audit_trail WHERE timestamp >= ADD_MONTHS(TRUNC(SYSDATE),-12) AND action_name='LOGON' AND returncode <> 0 GROUP BY NVL(username,'<NULL>'), NVL(userhost,'<NULL>'), NVL(os_username,'<NULL>'), returncode ORDER BY audit_rows DESC;
```

Objetivos:

- comprobar si los 63 `ORA-01017` provienen de una aplicación/credencial antigua conocida;
- identificar el origen del único `ORA-28000`;
- descartar un patrón externo o inesperado antes de modificar la política de auditoría;
- cerrar el diagnóstico de auditoría activa y pasar a planificación de retención/purga.

Estado: **resultado pendiente**.

## Seguridad

Continúa prohibido por ahora ejecutar `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de `AUDIT_TRAIL`, movimientos de `AUD$`, cambios de tablespaces/datafiles o cambios de RMAN. La misión sigue en diagnóstico y planificación.
