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

El listener está activo y escucha en:

```text
(PROTOCOL=tcp)(HOST=192.168.1.60)(PORT=1521)
```

pero `lsnrctl services` devuelve:

```text
The listener supports no services
```

La instancia reporta:

```text
instance_name  = orcl
service_names  = orcl.eukarya.adelantos.com.py
local_listener = <vacío>
```

`ALTER SYSTEM REGISTER;` no consiguió registrar el servicio.

Finalmente se comprobó que la resolución local de hostname está desactualizada:

```text
getent hosts "$(hostname)"
192.168.0.60    kanela.lcompras.biz kanela
```

mientras que el listener está vinculado a `192.168.1.60`.

El usuario confirmó que `/etc/hosts` todavía contiene la IP antigua `192.168.0.60` para `kanela`.

## Interpretación actual

La discrepancia entre la resolución local de `kanela` (`192.168.0.60`) y la IP vigente del listener (`192.168.1.60`) es una causa concreta y coherente con la ausencia de registro dinámico del servicio Oracle.

Además, el script sigue mostrando un segundo problema independiente en la etapa SSHFS hacia `zapallo`:

```text
read: Connection reset by peer
```

Ese problema de montaje se revisará después de restablecer el registro Oracle Net.

## Próximo paso

Respaldar `/etc/hosts`, corregir únicamente la IP de `kanela` de `192.168.0.60` a `192.168.1.60`, forzar nuevamente `ALTER SYSTEM REGISTER;` y validar con `lsnrctl services` antes de repetir el export full.
