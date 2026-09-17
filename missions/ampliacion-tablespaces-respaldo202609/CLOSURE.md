# Cierre de misión — ampliación de tablespaces y revisión de respaldo

Fecha: 2026-09-16

## Decisión del usuario

La base `orcl` es una base histórica que ya no se utiliza operativamente. El objetivo principal era volver a ponerla en marcha, validar que pudiera operar nuevamente y, posteriormente, liberar espacio innecesariamente ocupado por auditoría histórica.

## Estado final

- Base `orcl` recuperada y operativa (`OPEN`, `ACTIVE`, `READ WRITE`).
- Listener funcional.
- Reparación AWR/LOB completada y validada.
- Datafile 2 validado física y lógicamente con `Blocks Failing = 0`.
- `SYSTEM`, `SYSAUX`, `TABLAS` y demás tablespaces diagnosticados.
- `TABLAS` tiene todavía ~18.28 GB libres internos; no requiere ampliación inmediata.
- El gran volumen de auditoría histórica se explicó por imports/cargas temporales de 2019.
- Auditoría tradicional inventariada: sentencia y privilegio globales; sin auditoría por objetos.
- `DBMS_AUDIT_MGMT` está disponible y válido.
- `CLEANUP_INITIALIZED=FALSE`; no se ejecutó `INIT_CLEANUP`.
- No se modificó la política de auditoría.
- No se modificaron tablespaces/datafiles.
- No se revisó ni modificó RMAN porque el usuario confirmó que la base es histórica y ya no se usa operativamente.

## Liberación de espacio en `SYSTEM`

`SYS.AUD$` ocupaba aproximadamente **250 MB** dentro de `SYSTEM` y contenía más de 1.2 millones de filas de auditoría acumuladas desde 2009.

Dado que el usuario confirmó que la base es histórica y que no necesitaba conservar esa auditoría extendida, se ejecutó:

```sql
TRUNCATE TABLE SYS.AUD$;
```

Resultado confirmado por SQL*Plus:

```text
Table truncated.
```

Consecuencia:

- se eliminaron las filas históricas almacenadas en `SYS.AUD$`;
- el espacio previamente ocupado por el segmento queda disponible para reutilización dentro del tablespace `SYSTEM`;
- no se ha afirmado todavía una cifra exacta de espacio libre posterior al `TRUNCATE`, porque falta una medición posterior;
- esta operación no implica por sí sola una reducción del tamaño físico de `system01.dbf` en el filesystem.

## Criterio de cierre

Dado el carácter histórico/no operativo de la base, no se justifica introducir cambios estructurales adicionales ni una reingeniería de respaldo. Se adopta una postura conservadora: dejar la base estable, funcional y con la auditoría histórica innecesaria eliminada.

## Acciones explícitamente no realizadas

- no ampliar `TABLAS`;
- no reducir datafiles;
- no mover `AUD$`;
- no ejecutar `DBMS_AUDIT_MGMT.INIT_CLEANUP`;
- no ejecutar `NOAUDIT`;
- no cambiar `AUDIT_TRAIL`;
- no cambiar configuración RMAN;
- no eliminar respaldos;
- no modificar `AUTOEXTEND`.

## Estado de la misión

**CERRADA / COMPLETADA**, con una acción posterior de liberación de espacio registrada.

La base quedó nuevamente operativa y se eliminó la auditoría histórica que ocupaba espacio innecesario en `SYSTEM`.
