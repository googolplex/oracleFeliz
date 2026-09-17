# Progreso — limpieza del ADR (2026-09-16)

## Objetivo

La base `orcl` es histórica/no operativa. Después de recuperar espacio interno y reducir `UNDOTBS1`, se inspeccionó el filesystem para identificar acumulación de logs, trazas e incidentes de Oracle ADR.

## Ubicación ADR confirmada

`V$DIAG_INFO` mostró:

```text
ADR Base      /home/oracle/app/oracle
ADR Home      /home/oracle/app/oracle/diag/rdbms/orcl/orcl
Diag Alert    /home/oracle/app/oracle/diag/rdbms/orcl/orcl/alert
Diag Cdump    /home/oracle/app/oracle/diag/rdbms/orcl/orcl/cdump
Diag Incident /home/oracle/app/oracle/diag/rdbms/orcl/orcl/incident
Diag Trace    /home/oracle/app/oracle/diag/rdbms/orcl/orcl/trace
Health Monitor /home/oracle/app/oracle/diag/rdbms/orcl/orcl/hm
```

La misma consulta mostró:

```text
Active Incident Count = 17217
Active Problem Count  = 2
```

## Uso del filesystem por componente ADR

Se ejecutó, solo lectura:

```text
HOST du -sh /home/oracle/app/oracle/diag/rdbms/orcl/orcl/* 2>/dev/null
```

Resultado:

```text
33M   alert
8.0K  cdump
8.0K  hm
41G   incident
8.0K  incpkg
16K   ir
168K  lck
21M   metadata
176M  stage
636K  sweep
979M  trace
```

Hallazgo principal: `incident` consume aproximadamente **41 GB**, muy por encima de cualquier otro componente. `trace` agrega ~979 MB y `stage` ~176 MB.

## ADR homes conocidos

`adrci show homes` devolvió:

```text
diag/rdbms/orcl/orcl
diag/tnslsnr/kanela/listener
```

Por tanto, cualquier `PURGE` deberá seleccionar explícitamente un único ADR home antes de ejecutarse.

## Política de retención ADR

Para `diag/rdbms/orcl/orcl`, `SHOW CONTROL` devolvió:

```text
SHORTP_POLICY   720
LONGP_POLICY    8760
LAST_AUTOPRG_TIME 2026-09-15 22:17:56.917299 -04:00
CREATE_TIME        2010-02-23 22:42:25.713476 -03:00
```

Interpretación conforme a Oracle 11g:

- `SHORTP_POLICY=720` horas = 30 días;
- `LONGP_POLICY=8760` horas = 365 días;
- los incidentes y sus dumps son contenido de larga vida, por lo que pueden mantenerse hasta 365 días antes de ser elegibles para purga automática;
- el autopurge se ejecutó el 15-Sep-2026, por lo que los ~41 GB restantes no deben interpretarse simplemente como ausencia de autopurge;
- antes de purgar se inspeccionará la antigüedad real de los directorios de incidentes.

## Seguridad

Todavía no se ejecutó `adrci purge` ni `rm`. La fase permanece en diagnóstico de solo lectura.
