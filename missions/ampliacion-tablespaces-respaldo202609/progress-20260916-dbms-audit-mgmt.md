# Progreso — preparación de limpieza soportada de auditoría (2026-09-16)

## Estado confirmado

Se verificó la disponibilidad del paquete soportado por Oracle para administrar y limpiar el audit trail tradicional:

```sql
SELECT owner, object_name, object_type, status FROM dba_objects WHERE owner='SYS' AND object_name='DBMS_AUDIT_MGMT' ORDER BY object_type;
```

Resultado:

```text
SYS  DBMS_AUDIT_MGMT  PACKAGE       VALID
SYS  DBMS_AUDIT_MGMT  PACKAGE BODY  VALID
```

Conclusión:

- `DBMS_AUDIT_MGMT` está disponible y válido;
- la futura limpieza de `SYS.AUD$` debe diseñarse con este mecanismo soportado;
- no se utilizará `DELETE` ni `TRUNCATE` directo sobre `SYS.AUD$`;
- todavía no se ejecuta `INIT_CLEANUP`, `SET_LAST_ARCHIVE_TIMESTAMP`, `CLEAN_AUDIT_TRAIL`, `CREATE_PURGE_JOB` ni ninguna modificación de auditoría.

## Consideración importante

Antes de inicializar la limpieza debe verificarse la configuración existente de `DBMS_AUDIT_MGMT`. La inicialización es una operación modificadora y, según la configuración/versión, puede preparar la infraestructura de limpieza y afectar la ubicación del audit trail; por tanto, no debe ejecutarse a ciegas.

## Siguiente diagnóstico autorizado

Consultar únicamente la configuración actual del mecanismo de administración de auditoría:

```sql
SELECT parameter_name, parameter_value, audit_trail FROM dba_audit_mgmt_config_params ORDER BY audit_trail, parameter_name;
```

Objetivos:

- comprobar si ya existe configuración de cleanup para el audit trail estándar;
- identificar intervalos y parámetros actuales;
- evitar reinicializar o modificar una configuración ya existente;
- mantener la misión en modo diagnóstico hasta completar el procedimiento de respaldo/retención previo a la purga.

## Seguridad

Continúa prohibido por ahora ejecutar cambios sobre auditoría, purga de `SYS.AUD$`, cambios de tablespaces/datafiles o cambios de RMAN. Solo lectura.