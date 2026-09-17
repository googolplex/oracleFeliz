# Progreso — inventario de auditoría activa (2026-09-16)

## Contexto

La investigación de `SYS.AUD$` determinó que el volumen histórico está dominado por cargas temporales realizadas para elaborar informes a partir de datos importados desde servidores de producción. El usuario confirmó que, terminados esos trabajos, no existe necesidad funcional de conservar indefinidamente ni el universo importado ni 17 años de auditoría.

## Auditoría tradicional activa

### Auditoría de sentencias

`DBA_STMT_AUDIT_OPTS` devuelve **28 opciones globales**, todas con:

```text
SUCCESS = BY ACCESS
FAILURE = BY ACCESS
```

Incluye `CREATE SESSION` y múltiples operaciones administrativas sensibles.

### Auditoría por privilegios

`DBA_PRIV_AUDIT_OPTS` devuelve **23 privilegios globales**, también con `BY ACCESS` tanto para éxito como para fallo.

Incluye `CREATE SESSION`, `ALTER SYSTEM`, `ALTER DATABASE`, creación/eliminación de usuarios, grants y otros privilegios administrativos.

### Auditoría por objetos

La consulta sobre `DBA_OBJ_AUDIT_OPTS` devolvió:

```text
no rows selected
```

Conclusión: **no existe auditoría explícita por objetos** actualmente.

## Composición reciente — últimos 12 meses

Resultado:

```text
LOGON    0      256
LOGOFF   0      256
LOGON 1017       63
LOGON 28000       1
```

Interpretación:

- solo **576 eventos** auditados en 12 meses;
- el crecimiento reciente de `AUD$` es bajo;
- la presión actual de espacio es histórica, no crecimiento reciente acelerado;
- los fallos de autenticación sí tienen valor operativo;
- las conexiones exitosas rutinarias son el principal candidato a dejar de auditar cuando se pase a ejecución.

## Origen de los fallos recientes

Los 64 fallos recientes corresponden exclusivamente a:

```text
USERNAME    = AMANDA
USERHOST    = suzuka.lcompras.biz
OS_USERNAME = tomcat6
ORA-01017   = 63 eventos, 21-SEP-2025 a 13-SEP-2026
ORA-28000   = 1 evento, 24-MAY-2026
```

Estado actual de la cuenta:

```text
USERNAME       = AMANDA
ACCOUNT_STATUS = OPEN
PROFILE        = DEFAULT
CREATED        = 09-OCT-2012
LOCK_DATE      = NULL
EXPIRY_DATE    = NULL
```

Conclusión:

- el `ORA-28000` fue un bloqueo transitorio ya resuelto;
- los `ORA-01017` son compatibles con credenciales incorrectas/antiguas usadas intermitentemente por `tomcat6` en `suzuka.lcompras.biz`;
- conviene conservar auditoría de fallos de sesión.

## DBMS_AUDIT_MGMT

Se verificó:

```text
SYS.DBMS_AUDIT_MGMT PACKAGE      VALID
SYS.DBMS_AUDIT_MGMT PACKAGE BODY VALID
```

Por tanto existe el mecanismo soportado por Oracle para administrar y purgar el audit trail; no se utilizará `DELETE` ni `TRUNCATE` directo sobre `SYS.AUD$`.

`DBA_AUDIT_MGMT_CONFIG_PARAMS` muestra, entre otros:

```text
STANDARD AUDIT TRAIL
  DB AUDIT CLEAN BATCH SIZE = 10000
  DB AUDIT TABLESPACE       = SYSAUX
```

Sin embargo, `SYS.AUD$` se encuentra actualmente en `SYSTEM` y ocupa aproximadamente **250 MB**. `SYSAUX` tenía aproximadamente **114 MB libres internos** en el diagnóstico previo, por lo que no existe margen cómodo para provocar un traslado de `AUD$` hacia `SYSAUX` sin planificación.

Se verificó explícitamente:

```plsql
DBMS_AUDIT_MGMT.IS_CLEANUP_INITIALIZED(DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD)
```

Resultado:

```text
CLEANUP_INITIALIZED=FALSE
```

Conclusión consolidada:

- la limpieza del audit trail estándar **nunca fue inicializada**;
- no ejecutar todavía `DBMS_AUDIT_MGMT.INIT_CLEANUP`;
- no purgar ni mover `AUD$` todavía;
- antes de cualquier operación destructiva o estructural debe completarse el inventario y validación de respaldo/restauración exigido por la misión.

## Política preliminar — aún no ejecutada

La evidencia sugiere como diseño futuro:

1. Mantener auditoría de privilegios/operaciones administrativas sensibles.
2. Mantener auditoría de `CREATE SESSION` cuando falle.
3. Evaluar dejar de registrar `CREATE SESSION` exitoso rutinario.
4. Definir una ventana limitada de retención en vez de conservar 17 años indefinidamente.
5. Respaldar/exportar lo que se decida conservar antes de purgar.
6. Usar exclusivamente mecanismos soportados por Oracle para purga.
7. Definir un destino adecuado para `AUD$`; no asumir que `SYSAUX` es seguro con el espacio actual.

## Próxima fase

La rama de diagnóstico de auditoría queda suficientemente cerrada para pasar a **inventario de respaldo y restauración** antes de autorizar cambios.

Primer objetivo: inspeccionar la configuración RMAN actual en modo solo lectura.

## Seguridad

Continúa prohibido por ahora ejecutar `NOAUDIT`, `DELETE`/`TRUNCATE` sobre `SYS.AUD$`, `INIT_CLEANUP`, `CLEAN_AUDIT_TRAIL`, movimientos de `AUD$`, cambios de `AUDIT_TRAIL`, cambios de tablespaces/datafiles, cambios de configuración RMAN o eliminación de respaldos.
