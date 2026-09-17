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

## Interpretación

- La auditoría estándar actual es global y relativamente amplia.
- Tanto operaciones exitosas como fallidas se registran `BY ACCESS`.
- `CREATE SESSION` está entre las opciones activas y es consistente con la generación de registros de conexión/desconexión que llenaron `AUD$` durante las importaciones masivas de 2019.
- El problema actual no es una avalancha de auditoría reciente: desde 2020 el volumen anual es bajo. El problema principal es la retención histórica acumulada.
- Antes de ejecutar `NOAUDIT` o purgar `SYS.AUD$`, debe completarse el inventario de las demás categorías de auditoría tradicional.
- No se desactivará todavía ninguna protección útil; posteriormente se evaluará una política más selectiva, por ejemplo conservar auditoría de fallos y de operaciones administrativas sensibles mientras se evita registrar actividad rutinaria que no tenga valor operativo.

## Siguiente diagnóstico autorizado

Inventariar auditoría por privilegios del sistema:

```sql
SELECT user_name, proxy_name, privilege, success, failure FROM dba_priv_audit_opts ORDER BY user_name, proxy_name, privilege;
```

Objetivos:

- determinar si existen privilegios del sistema auditados adicionalmente a las 28 opciones de sentencia;
- distinguir auditoría global de auditoría por usuario/proxy;
- evitar desactivar accidentalmente controles de seguridad relevantes;
- completar el mapa de auditoría activa antes de diseñar retención, archivado y purga.

Estado: **resultado pendiente**.

## Seguridad

Continúa prohibido por ahora ejecutar `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, cambios de `AUDIT_TRAIL`, movimientos de `AUD$`, cambios de tablespaces/datafiles o cambios de RMAN. La misión sigue en diagnóstico y planificación.
