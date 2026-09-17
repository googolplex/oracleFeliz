# Misión 01 — Reparación del FULL export Oracle de `kanela`

Fecha: 2026-09-16
Estado: **OPERATIVA / ACEPTADA POR EL USUARIO**

## Objetivo

Recuperar y validar el mecanismo histórico de FULL export lógico de Oracle ejecutado desde la VM CentOS 5.11 `kanela`, conservando el diseño existente y modificando solamente los puntos rotos por la migración/cambio de red.

Esta misión es independiente de la reparación AWR/LOB y complementa la misión `missions/ampliacion-tablespaces-respaldo202609/`, donde se había identificado el mecanismo histórico de exports `.dmp`.

## Entorno confirmado

- host KVM: `zapallo`
- IP vigente de `zapallo`: `192.168.1.71`
- VM Oracle: `kanela`
- IP vigente de `kanela`: `192.168.1.60`
- sistema operativo de `kanela`: CentOS 5.11
- Oracle: 11.2.0.1.0 64-bit
- `ORACLE_SID=orcl`
- `ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1`
- base: `OPEN`, `ACTIVE`, `READ WRITE`
- listener TCP: `192.168.1.60:1521`

## Mecanismo histórico identificado

El `crontab` del usuario `oracle` ejecutaba inicialmente:

```text
0 2 * * 1-5 /home/oracle/exportar_kanela_full_v2.sh > /home/oracle/exportar_kanela_full_v2.log 2>&1
```

Se decidió reducir la frecuencia porque `orcl` es actualmente una base histórica.

Antes del cambio se respaldó el crontab en:

```text
/home/oracle/crontab_oracle_20260916.backup
```

La programación quedó:

```text
0 2 * * 1 /home/oracle/exportar_kanela_full_v2.sh > /home/oracle/exportar_kanela_full_v2.log 2>&1
```

Por tanto, el FULL export queda programado **todos los lunes a las 02:00**.

## Script y parfile

Script:

```text
/home/oracle/exportar_kanela_full_v2.sh
```

Parfile:

```text
/home/oracle/exportar_kanela_full_v2.sql
```

Contenido funcional del parfile:

```text
full=y
file=/picornavirales/export_kanela_full.dmp
grants=y
rows=y
```

Se confirmó que el mecanismo utiliza el utilitario clásico Oracle `exp`, no Data Pump `expdp`.

El script contiene una credencial Oracle embebida en texto plano. **La credencial no se documenta ni se copia al repositorio.**

## Arquitectura del respaldo lógico

El flujo histórico es:

1. desde `kanela`, montar mediante SSHFS:

```text
root@zapallo:/picornavirales/images.backup -> /picornavirales
```

2. ejecutar `exp` FULL y generar:

```text
/picornavirales/export_kanela_full.dmp
```

3. copiar el dump mediante `scp` a un archivo versionado en:

```text
zapallo:/camalote/images.backup/export_<timestamp>.dmp
```

4. eliminar el dump intermedio y desmontar `/picornavirales`.

Este mecanismo complementa los respaldos de la VM y aporta una capa lógica independiente.

## Fallo inicial 1 — Oracle Net / TNS

La primera prueba manual del script produjo:

```text
EXP-00056: ORACLE error 12543 encountered
ORA-12543: TNS:destination host unreachable
```

Después de corregir el destino Oracle Net hasta alcanzar el listener, el error cambió a:

```text
EXP-00056: ORACLE error 12514 encountered
ORA-12514: TNS:listener does not currently know of service requested in connect descriptor
```

El listener estaba activo en:

```text
(PROTOCOL=tcp)(HOST=192.168.1.60)(PORT=1521)
```

pero inicialmente:

```text
lsnrctl services
```

reportaba:

```text
The listener supports no services
```

La instancia confirmaba:

```text
instance_name  = orcl
service_names  = orcl.eukarya.adelantos.com.py
local_listener = <vacío>
```

## Causa encontrada — resolución local antigua de `kanela`

Se comprobó que el hostname local todavía resolvía a la red anterior:

```text
192.168.0.60 kanela.lcompras.biz kanela
```

mientras el listener actual estaba en:

```text
192.168.1.60
```

Se respaldó `/etc/hosts` y se corrigió la entrada de `kanela` a:

```text
192.168.1.60 kanela.lcompras.biz kanela
```

Después se ejecutó:

```sql
ALTER SYSTEM REGISTER;
```

La validación con `lsnrctl services` mostró nuevamente:

```text
Service "orcl.eukarya.adelantos.com.py" has 1 instance(s).
  Instance "orcl", status READY
```

También quedó registrado el servicio XDB.

`tnsping orcl.eukarya.adelantos.com.py` pasó a contactar:

```text
HOST = 192.168.1.60
PORT = 1521
SERVICE_NAME = orcl.eukarya.adelantos.com.py
```

con resultado:

```text
OK (0 msec)
```

Resultado: **Oracle Net/TNS recuperado**.

## Fallo inicial 2 — SSHFS hacia `zapallo`

El script también producía:

```text
read: Connection reset by peer
```

Al probar desde `kanela` como usuario `oracle`:

```text
ssh -v root@zapallo true
```

se obtuvo inicialmente:

```text
ssh: zapallo: Temporary failure in name resolution
```

El archivo `/home/oracle/.ssh/config` ya contenía:

```text
host zapallo
hostname zapallo
port 22
user root
```

pero `/etc/hosts` de `kanela` no tenía entrada para `zapallo`.

Se agregó:

```text
192.168.1.71 zapallo.lcompras.biz zapallo
```

Después de la corrección, la prueba SSH confirmó:

```text
Connecting to zapallo [192.168.1.71] port 22.
Connection established.
...
Authentication succeeded (publickey).
...
Exit status 0
```

Se verificó además que el OpenSSH 4.3 de CentOS 5.11 puede autenticarse correctamente contra el OpenSSH 7.2 del `zapallo` actual utilizando la clave RSA histórica.

Resultado: **resolución, conexión y autenticación SSH recuperadas**.

## Validación específica de SSHFS

Se ejecutó manualmente:

```text
sshfs root@zapallo:/picornavirales/images.backup /picornavirales
```

sin error.

Posteriormente:

```text
mountpoint /picornavirales
```

respondió:

```text
/picornavirales is a mountpoint
```

Resultado: **SSHFS recuperado y validado**.

## Fallo inicial 3 — destino `scp` con IP antigua

El script todavía contenía como destino final:

```text
root@192.168.0.71:/camalote/images.backup/
```

Se probó explícitamente:

```text
ssh -o ConnectTimeout=5 root@192.168.0.71 true
```

con resultado:

```text
ssh: connect to host 192.168.0.71 port 22: No route to host
```

Se confirmó luego que el destino equivalente existe en el `zapallo` actual:

```text
ssh root@zapallo 'test -d /camalote/images.backup && echo OK || echo MISSING'
```

resultado:

```text
OK
```

Antes de modificar el script se creó:

```text
/home/oracle/exportar_kanela_full_v2.sh.20260916.backup
```

Luego se reemplazó únicamente el destino antiguo:

```text
root@192.168.0.71:
```

por:

```text
root@zapallo:
```

La línea final verificada quedó:

```text
scp /picornavirales/export_kanela_full.dmp root@zapallo:/camalote/images.backup/export_$sufijo.dmp
```

Usar el hostname `zapallo` evita volver a acoplar el script a una IP fija y aprovecha la resolución ya validada.

## Prueba final del FULL export

Después de recuperar Oracle Net, SSH, SSHFS y el destino final de `scp`, se relanzó:

```text
/home/oracle/exportar_kanela_full_v2.sh
```

La ejecución alcanzó correctamente la fase real de FULL export y comenzó a recorrer/exportar la base, mostrando entre otras etapas:

```text
About to export the entire database ...
. exporting tablespace definitions
. exporting profiles
. exporting user definitions
. exporting roles
. exporting resource costs
. exporting rollback segment definitions
. exporting database links
. exporting sequence numbers
. exporting directory aliases
. exporting context namespaces
. exporting foreign function library names
. exporting PUBLIC type synonyms
. exporting private type synonyms
. exporting object type definitions
```

El usuario confirmó visualmente que el export estaba efectivamente avanzando y, dada su duración, decidió dejarlo funcionando y considerar el mecanismo recuperado.

## Advertencia observada durante el export

Durante la ejecución apareció:

```text
EXP-00097: Object type "ADMIN"."JS_MENSAJE_TABLE" is not in a valid state, type will not be exported
```

Esto queda registrado como **pendiente menor de revisión**, separado de la recuperación del mecanismo de export. No se intentó recompilar el objeto mientras el FULL export estaba en ejecución.

## Estado final aceptado

A solicitud expresa del usuario, el mecanismo se considera **FUNCIONAL / RECUPERADO**.

Quedaron comprobados individualmente:

- listener Oracle accesible en `192.168.1.60:1521`;
- servicio `orcl.eukarya.adelantos.com.py` registrado y `READY`;
- `tnsping` correcto;
- resolución de `kanela` corregida a `192.168.1.60`;
- resolución de `zapallo` corregida a `192.168.1.71`;
- SSH `oracle@kanela -> root@zapallo` funcional por clave pública;
- SSHFS hacia `/picornavirales/images.backup` funcional;
- `/picornavirales` verificado como mountpoint;
- destino `/camalote/images.backup` existente en `zapallo`;
- IP obsoleta `192.168.0.71` eliminada del `scp` del script;
- script respaldado antes de la modificación;
- FULL export relanzado y observado exportando efectivamente la base;
- frecuencia del cron reducida a una vez por semana, lunes 02:00.

No se capturó en esta sesión el mensaje final de terminación del `exp` ni la confirmación final del `scp`, porque el FULL export es de larga duración. Por decisión del usuario, esto **no impide declarar recuperado el mecanismo**, aunque conviene revisar posteriormente el log o la presencia del dump versionado como control de cierre.

## Pendientes no bloqueantes

1. Verificar posteriormente `/home/oracle/exportar_kanela_full_v2.log` o el dump versionado más reciente en `zapallo:/camalote/images.backup/` para dejar evidencia de una ejecución completa de extremo a extremo.
2. Revisar el objeto inválido `ADMIN.JS_MENSAJE_TABLE` y sus dependencias antes de decidir si corresponde recompilarlo.
3. Mantener anotada como debilidad histórica la credencial Oracle embebida en el script; no se almacena su valor en GitHub.
4. El script histórico no valida rigurosamente códigos de retorno de `exp`/`scp`; una mejora futura podría robustecer esta lógica, pero no forma parte de la reparación mínima realizada hoy.

## Criterio de conservación

No rediseñar este flujo mientras siga siendo útil como respaldo lógico histórico. Las modificaciones realizadas fueron mínimas y dirigidas a restaurar compatibilidad con la red vigente, preservando el comportamiento original del sistema.
