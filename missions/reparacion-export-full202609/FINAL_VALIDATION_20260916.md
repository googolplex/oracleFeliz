# Validación final — FULL export Oracle `kanela`

Fecha: 2026-09-16
Estado: **CERRADO / VALIDADO POR EL USUARIO**

## Confirmación final

El usuario confirmó después de la ejecución completa que el FULL export **funciona correctamente**, que la operación terminó bien y que además presentó un rendimiento satisfactorio, descrito como **muy rápido**.

Esta confirmación completa la validación práctica de extremo a extremo iniciada en `mission-01-reparacion-export-full.md`.

## Flujo validado

Queda considerado operativo el flujo completo:

1. resolución de `kanela` en la red vigente (`192.168.1.60`);
2. listener Oracle en `192.168.1.60:1521`;
3. registro dinámico del servicio `orcl.eukarya.adelantos.com.py` en estado `READY`;
4. resolución de `zapallo` desde `kanela` (`192.168.1.71`);
5. autenticación SSH por clave desde `oracle@kanela` hacia `root@zapallo`;
6. montaje SSHFS de `root@zapallo:/picornavirales/images.backup` sobre `/picornavirales`;
7. ejecución del Oracle classic `exp` con `FULL=Y`, `ROWS=Y` y `GRANTS=Y`;
8. copia versionada mediante `scp` hacia `zapallo:/camalote/images.backup/` usando el hostname vigente `zapallo` en lugar de la IP histórica `192.168.0.71`;
9. programación semanal mediante `cron`, todos los lunes a las 02:00.

## Cierre

La misión `reparacion-export-full202609` se considera **FINALIZADA CON ÉXITO**.

La advertencia observada durante una ejecución sobre el objeto inválido `ADMIN.JS_MENSAJE_TABLE` (`EXP-00097`) permanece documentada como asunto separado y no invalida la recuperación del mecanismo de FULL export.

No se almacena en el repositorio ninguna credencial embebida en el script histórico.
