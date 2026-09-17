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

## Interpretación consolidada

- La auditoría tradicional actual es global y relativamente amplia.
- Tanto operaciones exitosas como fallidas se registran `BY ACCESS`.
- `CREATE SESSION` está auditado y es coherente con el gran número histórico de eventos de conexión generados por los imports de 2019.
- Existen además múltiples privilegios administrativos sensibles auditados globalmente; no deben desactivarse de forma indiscriminada.
- No existe auditoría específica por objetos que deba preservarse o reconciliarse antes de simplificar la política.
- El problema actual sigue siendo principalmente la retención histórica acumulada, no una generación reciente masiva.
- El inventario de auditoría tradicional queda completo: sentencia, privilegio y objeto.
- Antes de proponer `NOAUDIT` o purga se debe observar la composición de la auditoría reciente para distinguir actividad rutinaria de eventos administrativos/fallidos realmente útiles.

## Siguiente diagnóstico autorizado

Medir la composición de la auditoría reciente por acción y código de retorno durante los últimos 12 meses disponibles:

```sql
SELECT action_name, returncode, COUNT(*) audit_rows FROM dba_audit_trail WHERE timestamp >= ADD_MONTHS(TRUNC(SYSDATE),-12) GROUP BY action_name, returncode ORDER BY audit_rows DESC;
```

Objetivos:

- identificar qué acciones explican el volumen reciente;
- comprobar si `LOGON`/`LOGOFF` exitosos siguen siendo la mayor fuente de ruido;
- separar eventos fallidos de eventos exitosos rutinarios;
- usar evidencia reciente, no solo la anomalía de 2019, para diseñar la política futura;
- definir posteriormente una combinación segura de auditoría, retención, archivado previo y purga controlada.

Estado: **resultado pendiente**.

## Seguridad

Continúa prohibido por ahora ejecutar `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de `AUDIT_TRAIL`, movimientos de `AUD$`, cambios de tablespaces/datafiles o cambios de RMAN. La misión sigue en diagnóstico y planificación.
