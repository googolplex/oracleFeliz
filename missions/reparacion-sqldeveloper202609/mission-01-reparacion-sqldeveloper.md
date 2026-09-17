# Misión 01 — Reparar conexión de SQL Developer a `kanela`

Fecha de creación: 2026-09-16
Estado: **PENDIENTE / PROGRAMADA PARA MÁS ADELANTE**

## Objetivo

Reparar y validar la conexión de Oracle SQL Developer hacia la base histórica `orcl` alojada en la VM `kanela`.

Esta misión queda registrada como la **siguiente misión del proyecto `googolplex/oracleFeliz`**, posterior al cierre exitoso de la reparación del FULL export Oracle.

## Contexto técnico ya conocido

- VM Oracle: `kanela`
- IP vigente de `kanela`: `192.168.1.60`
- Oracle: 11.2.0.1.0 64-bit
- `ORACLE_SID=orcl`
- listener TCP confirmado en `192.168.1.60:1521`
- servicio confirmado: `orcl.eukarya.adelantos.com.py`
- estado del servicio en listener: `READY`
- `tnsping` desde `kanela` validado correctamente después de la reparación de red
- `/etc/hosts` de `kanela` ya fue corregido para la red vigente `192.168.1.x`

## Alcance futuro

Cuando se retome esta misión, revisar de manera controlada la conectividad de SQL Developer, sin asumir todavía la causa del fallo.

La revisión deberá cubrir, según la evidencia disponible en ese momento:

- host/IP configurado en SQL Developer;
- puerto `1521`;
- uso de `SERVICE_NAME` frente a `SID`;
- nombre de servicio `orcl.eukarya.adelantos.com.py`;
- resolución DNS/hosts desde la máquina cliente;
- alcance TCP hacia `192.168.1.60:1521`;
- configuración del listener si surgiera evidencia nueva;
- credenciales únicamente mediante prueba interactiva, sin documentarlas en GitHub;
- cualquier configuración histórica de SQL Developer que todavía apunte a la red antigua `192.168.0.x`.

## Regla de trabajo

No modificar listener, parámetros Oracle ni red sin evidencia previa. Empezar con pruebas de conectividad y con la configuración actual del cliente SQL Developer.

## Estado

**No ejecutar hoy.**

La misión se deja aparcada deliberadamente para continuar en otra sesión.
