# Revisión de exports Oracle históricos

Fecha: 2026-09-16

## Objetivo

Revisar el mecanismo histórico de exports `.dmp` generado desde el propio CentOS de `kanela`, complementario a los respaldos automáticos de la máquina virtual entre `zapallo` y `cafe`.

## Hallazgo inicial

El `crontab` del usuario `oracle` contiene:

```text
0 2 * * 1-5 /home/oracle/exportar_kanela_full_v2.sh  > /home/oracle/exportar_kanela_full_v2.log  2>&1          #exportar kanela full
```

Interpretación confirmada:

- existe un mecanismo automatizado de export lógico Oracle;
- se ejecuta de lunes a viernes;
- horario programado: 02:00;
- script: `/home/oracle/exportar_kanela_full_v2.sh`;
- log: `/home/oracle/exportar_kanela_full_v2.log`;
- el comentario histórico lo identifica como `exportar kanela full`;
- la ejecución se realiza desde dentro de CentOS, bajo el entorno del usuario `oracle`.

## Estado

Revisión abierta. Próximo paso: inspeccionar en modo solo lectura el contenido de `/home/oracle/exportar_kanela_full_v2.sh` para identificar si utiliza `exp`, `expdp`, destino de los `.dmp`, política de nombres/rotación, compresión y eventual copia a otro host.
