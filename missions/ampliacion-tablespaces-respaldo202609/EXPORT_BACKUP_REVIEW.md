# Revisión de exports Oracle históricos

Fecha: 2026-09-16

## Objetivo

Revisar el mecanismo histórico de exports `.dmp` generado desde el propio CentOS de `kanela`, complementario a los respaldos automáticos de la máquina virtual entre `zapallo` y `cafe`.

## Programación encontrada

El `crontab` del usuario `oracle` contenía inicialmente:

```text
0 2 * * 1-5 /home/oracle/exportar_kanela_full_v2.sh  > /home/oracle/exportar_kanela_full_v2.log  2>&1          #exportar kanela full
```

Interpretación confirmada:

- existe un mecanismo automatizado de export lógico Oracle;
- originalmente se ejecutaba de lunes a viernes;
- horario programado: 02:00;
- script: `/home/oracle/exportar_kanela_full_v2.sh`;
- log: `/home/oracle/exportar_kanela_full_v2.log`;
- el comentario histórico lo identifica como `exportar kanela full`;
- la ejecución se realiza desde dentro de CentOS, bajo el entorno del usuario `oracle`.

## Ajuste de frecuencia — 2026-09-16

Dado que `orcl` es actualmente una base histórica y el usuario considera suficiente un export semanal, se redujo la frecuencia sin modificar el script ni el mecanismo de export.

La entrada actual verificada con `crontab -l` es:

```text
0 2 * * 1 /home/oracle/exportar_kanela_full_v2.sh  > /home/oracle/exportar_kanela_full_v2.log  2>&1          #exportar kanela full
```

Por tanto, el export full queda programado **una vez por semana, todos los lunes a las 02:00**.

Antes del cambio se generó una copia del crontab en:

```text
/home/oracle/crontab_oracle_20260916.backup
```

## Contenido del script `exportar_kanela_full_v2.sh`

Se inspeccionó el script en modo solo lectura.

### Entorno Oracle

El script prepara explícitamente:

- `ORACLE_HOSTNAME=kanela.eukarya.adelantos.com.py`;
- `ORACLE_BASE=/home/oracle/app/oracle`;
- `ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1`;
- `ORACLE_SID=orcl`;
- `ORACLE_UNQNAME=orcl`;
- `NLS_LANG=AMERICAN_AMERICA.WE8MSWIN1252`;
- `PATH`, `LD_LIBRARY_PATH` y `CLASSPATH` compatibles con Oracle 11g.

### Tipo de export

El mecanismo usa el utilitario clásico de Oracle:

```text
exp ... parfile=/home/oracle/exportar_kanela_full_v2.sql
```

Por tanto, se trata de **Oracle Export clásico (`exp`)**, no Data Pump (`expdp`).

### Flujo de almacenamiento

El script utiliza `/picornavirales` como punto de montaje temporal/intermedio.

Si `/picornavirales` no está montado, ejecuta conceptualmente:

```text
sshfs root@zapallo:/picornavirales/images.backup /picornavirales
```

Luego elimina cualquier `export_kanela_full.dmp` previo y ejecuta el export. Esto indica que el `.dmp` intermedio se escribía sobre un filesystem accesible mediante SSHFS desde `kanela` hacia `zapallo`, en vez de quedar únicamente en el disco local de la VM.

Después del export, si existe `/picornavirales/export_kanela_full.dmp`, el script realiza una segunda copia mediante `scp` hacia:

```text
root@192.168.0.71:/camalote/images.backup/export_<timestamp>.dmp
```

La dirección `192.168.0.71` y su relación actual con `zapallo`, `cafe` u otro host todavía no se consideran confirmadas; no se debe inferir identidad sin evidencia adicional.

El nombre de la segunda copia incorpora fecha y hora mediante un sufijo `YYYYMMDD_HH_MM_SS`, por lo que esa etapa sí conserva versiones históricas diferenciadas.

Finalmente el script elimina el `.dmp` intermedio de `/picornavirales` y desmonta el filesystem.

### Notificaciones y control de ejecución

El script envía un correo al inicio de la tarea indicando `exportar_kanela_full`.

El control de éxito es simple:

- no se observa `set -e`;
- no se inspecciona explícitamente el código de retorno de `exp`;
- la decisión de copiar se basa en la existencia del archivo `.dmp`;
- la salida completa del cron queda redirigida a `/home/oracle/exportar_kanela_full_v2.log`.

Por tanto, para evaluar la calidad histórica real de los exports será útil revisar el log del cron y los `.dmp` que aún existan en los destinos.

### Credenciales

El script contiene una credencial Oracle embebida en texto plano dentro de la invocación de `exp`. **La credencial no se copia ni se documenta en GitHub.** Este hallazgo se registra únicamente como una debilidad histórica de seguridad del script.

## Parfile `exportar_kanela_full_v2.sql`

Se inspeccionó el archivo en modo solo lectura y contiene:

```text
full=y
file=/picornavirales/export_kanela_full.dmp
grants=y
rows=y
```

Esto confirma:

- `FULL=Y`: el objetivo era un **export lógico completo**;
- `ROWS=Y`: se incluían los datos de las tablas;
- `GRANTS=Y`: se incluían los grants exportables;
- el dump se generaba como `/picornavirales/export_kanela_full.dmp`, coincidiendo con el flujo del script;
- no se observan parámetros adicionales en el parfile para consistencia, compresión, buffers u otras opciones explícitas.

La ausencia de un parámetro explícito de consistencia queda anotada para evaluación técnica posterior; no se infiere todavía el comportamiento efectivo sin contrastarlo con la semántica exacta de `exp` en Oracle 11g y con los logs históricos.

## Lectura arquitectónica

El esquema histórico era de varias capas:

1. respaldo de la máquina virtual;
2. export lógico Oracle actualmente programado una vez por semana;
3. generación del `.dmp` sobre almacenamiento remoto montado por SSHFS;
4. segunda copia versionada por `scp` a otro destino.

Esto aporta independencia entre recuperación de VM y recuperación lógica de objetos/datos Oracle.

## Estado

Frecuencia del export ajustada y verificada. El mecanismo queda activo semanalmente los lunes a las 02:00. La revisión del log histórico y de los `.dmp` existentes puede hacerse más adelante si se considera necesario.
