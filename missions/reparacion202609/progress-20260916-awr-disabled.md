# Progreso — AWR temporalmente deshabilitado

Fecha: 2026-09-16

## Estado

Se ejecutó:

```sql
exec dbms_workload_repository.modify_snapshot_settings(interval=>0);
```

Luego:

```sql
select snap_interval,retention from dba_hist_wr_control;
```

Resultado:

```text
SNAP_INTERVAL  +40150 00:00:00.0
RETENTION      +00008 00:00:00.0
```

Interpretación confirmada con documentación Oracle: al especificar `interval=>0`, Oracle deshabilita snapshots AWR automáticos y manuales y representa internamente el intervalo con un valor grande del sistema. La retención permanece en 8 días.

## Espacio en SYSAUX

```text
FREE_MB = 79.31
```

Segmentos relevantes:

```text
WRH$_SQL_PLAN                 TABLE       17 MB
WRH$_SQL_PLAN_PK              INDEX        9 MB
SYS_LOB0000006213C00038$$     LOBSEGMENT  19 MB
SYS_IL0000006213C00038$$      LOBINDEX     0.19 MB
```

Hay margen suficiente para recrear el LOB afectado dentro de SYSAUX.

## Seguridad

El usuario confirmó que existe un respaldo restaurable de la VM `kanela`, utilizable como punto de retorno si fuese necesario.

## Siguiente paso

Antes del `ALTER TABLE ... MOVE LOB`, confirmar el parámetro `db_securefile` para evitar una conversión implícita del LOB BasicFile durante la recreación.

Consulta recomendada:

```sql
select name,value from v$parameter where name='db_securefile';
```

No ejecutar todavía el `MOVE LOB` hasta verificar este valor.
