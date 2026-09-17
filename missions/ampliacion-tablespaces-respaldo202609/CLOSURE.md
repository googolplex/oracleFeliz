# Cierre de misión — ampliación de tablespaces y revisión de respaldo

Fecha: 2026-09-16

## Decisión del usuario

La base `orcl` es una base histórica que ya no se utiliza operativamente. El objetivo principal era volver a ponerla en marcha, validar que pudiera operar nuevamente y, posteriormente, liberar espacio innecesariamente ocupado por auditoría histórica y evaluar si existen datafiles sobredimensionados que puedan devolver espacio al filesystem.

## Estado final / fase de optimización posterior

- Base `orcl` recuperada y operativa (`OPEN`, `ACTIVE`, `READ WRITE`).
- Listener funcional.
- Reparación AWR/LOB completada y validada.
- Datafile 2 validado física y lógicamente con `Blocks Failing = 0`.
- `SYSTEM`, `SYSAUX`, `TABLAS` y demás tablespaces diagnosticados.
- El gran volumen de auditoría histórica se explicó por imports/cargas temporales de 2019.
- Auditoría tradicional inventariada: sentencia y privilegio globales; sin auditoría por objetos.
- `DBMS_AUDIT_MGMT` está disponible y válido.
- `CLEANUP_INITIALIZED=FALSE`; no se ejecutó `INIT_CLEANUP`.
- No se modificó la política de auditoría.
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

### Medición posterior al `TRUNCATE`

La consulta global de capacidad devolvió:

```text
TABLESPACE  TOTAL_MB   USED_MB    FREE_MB   PCT_USED
AMANDA      10240.00    103.44   10136.56      1.01
EXAMPLE       100.00     78.44      21.56     78.44
SYSAUX       1220.00   1107.50     112.50     90.78
SYSTEM       1190.00    933.69     256.31     78.46
TABLAS      32767.98  14491.67   18276.31     44.23
UNDOTBS1    13370.00     28.75   13341.25      0.22
USERS           5.00      4.06       0.94     81.25
```

Totales actuales:

- capacidad asignada en datafiles: **58.892,98 MB** (~57,51 GiB);
- espacio usado: **16.747,55 MB** (~16,36 GiB);
- espacio libre interno: **42.145,43 MB** (~41,16 GiB);
- porcentaje libre interno global: **71,56%**.

Comparación de `SYSTEM` antes/después:

- libre antes: **6,38 MB**;
- libre después: **256,31 MB**;
- espacio liberado por `TRUNCATE SYS.AUD$`: **249,93 MB**.

Este espacio quedó reutilizable dentro de Oracle. El tamaño físico de `system01.dbf` no cambió, por lo que esta acción no devolvió todavía esos ~250 MB al filesystem.

## Candidatos para devolver espacio al filesystem

Los mayores márgenes internos son:

- `TABLAS`: **18.276,31 MB** libres (~17,85 GiB);
- `UNDOTBS1`: **13.341,25 MB** libres (~13,03 GiB);
- `AMANDA`: **10.136,56 MB** libres (~9,90 GiB).

No se debe inferir que todo ese espacio sea inmediatamente reducible: para devolver espacio físico al sistema operativo hay que verificar el high-water mark de cada datafile y, especialmente en `UNDOTBS1`, la distribución/estado de extents de undo.

El primer candidato para diagnóstico es `AMANDA`, porque el datafile asigna 10 GB y solo contiene ~103 MB usados. Antes de cualquier `RESIZE` se medirá su HWM exacto.

## Acciones explícitamente no realizadas todavía

- no ampliar `TABLAS`;
- no reducir datafiles aún;
- no mover `AUD$`;
- no ejecutar `DBMS_AUDIT_MGMT.INIT_CLEANUP`;
- no ejecutar `NOAUDIT`;
- no cambiar `AUDIT_TRAIL`;
- no cambiar configuración RMAN;
- no eliminar respaldos;
- no modificar `AUTOEXTEND`.

## Estado de la misión

**BASE RECUPERADA / FASE DE LIBERACIÓN DE ESPACIO ABIERTA.**

La base quedó nuevamente operativa, la auditoría histórica innecesaria fue eliminada de `SYS.AUD$`, se recuperaron **249,93 MB** dentro de `SYSTEM`, y se identificaron datafiles con potencial importante para devolver espacio físico al filesystem tras validar sus HWM de forma individual.
