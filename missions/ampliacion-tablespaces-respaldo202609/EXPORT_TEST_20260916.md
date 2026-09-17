# Prueba manual del export full — 2026-09-16

## Resultado de la prueba

Se ejecutó manualmente `/home/oracle/exportar_kanela_full_v2.sh`.

La primera ejecución falló con:

```text
EXP-00056: ORACLE error 12543 encountered
ORA-12543: TNS:destination host unreachable
```

Luego, tras corregir el destino Oracle Net hasta alcanzar el listener, la segunda prueba cambió a:

```text
EXP-00056: ORACLE error 12514 encountered
ORA-12514: TNS:listener does not currently know of service requested in connect descriptor
```

El listener estaba activo y escuchaba en:

```text
(PROTOCOL=tcp)(HOST=192.168.1.60)(PORT=1521)
```

pero inicialmente `lsnrctl services` devolvía:

```text
The listener supports no services
```

La instancia reportó:

```text
instance_name  = orcl
service_names  = orcl.eukarya.adelantos.com.py
local_listener = <vacío>
```

Se comprobó que la resolución local de hostname estaba desactualizada:

```text
getent hosts "$(hostname)"
192.168.0.60    kanela.lcompras.biz kanela
```

mientras que el listener estaba vinculado a `192.168.1.60`.

Se respaldó `/etc/hosts` y se corrigió la entrada de `kanela` para que apunte a `192.168.1.60`. Después se ejecutó nuevamente:

```sql
ALTER SYSTEM REGISTER;
```

La validación posterior con `lsnrctl services` confirmó que el registro dinámico quedó restablecido:

```text
Service "orcl.eukarya.adelantos.com.py" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s) for this service...
    "DEDICATED" ... state:ready
```

También quedó visible el servicio XDB:

```text
Service "orclXDB.eukarya.adelantos.com.py" has 1 instance(s).
```

La resolución Oracle Net fue validada con:

```text
tnsping orcl.eukarya.adelantos.com.py
```

resultado:

```text
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.60)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = orcl.eukarya.adelantos.com.py)))
OK (0 msec)
```

Por tanto, el frente Oracle/TNS quedó recuperado.

## Diagnóstico SSH/SSHFS hacia `zapallo`

El script también había fallado en la etapa SSHFS con:

```text
read: Connection reset by peer
```

Se comprobó que desde `kanela`, como usuario `oracle`, `zapallo` inicialmente no resolvía por nombre. El archivo `/home/oracle/.ssh/config` contiene:

```text
host zapallo
hostname zapallo
port 22
user root
```

Se agregó a `/etc/hosts` de `kanela` la resolución vigente:

```text
192.168.1.71 zapallo.lcompras.biz zapallo
```

Después de ese cambio, la conexión SSH desde `kanela` como `oracle` fue validada explícitamente con:

```text
ssh -v root@zapallo true
```

La salida confirmó:

```text
Connecting to zapallo [192.168.1.71] port 22.
Connection established.
...
Server accepts key: pkalg ssh-rsa
Authentication succeeded (publickey).
...
Sending command: true
...
Exit status 0
```

Por tanto:

- resolución `zapallo` correcta desde `kanela`;
- TCP/22 accesible;
- compatibilidad SSH entre OpenSSH 4.3 de CentOS 5 y OpenSSH 7.2 de `zapallo` confirmada para esta conexión;
- autenticación por clave RSA del usuario `oracle` hacia `root@zapallo` funcional;
- el problema ya no es un fallo SSH general.

## Estado actual

- Oracle Net/TNS: **RESUELTO**.
- Registro dinámico del servicio Oracle: **RESUELTO**.
- Resolución y autenticación SSH hacia `zapallo`: **RESUELTAS**.
- Falta validar específicamente el montaje `sshfs` del directorio remoto `/picornavirales/images.backup` sobre `/picornavirales` antes de repetir el export full completo.

## Próximo paso

Probar manualmente el mismo montaje SSHFS que utiliza el script, sin lanzar todavía `exp`, y verificar que `/picornavirales` quede correctamente montado.