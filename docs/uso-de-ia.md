# Uso de herramientas de inteligencia artificial

La letra pide citar las herramientas de IA utilizadas y el contexto en que se
usaron. Este documento lo hace, e incluye además **cómo se verificó** cada cosa
generada, porque lo que no se verifica no se puede defender.

## Herramienta utilizada

**Claude (Anthropic), a través de Claude Code**, usado de forma interactiva desde
la máquina de trabajo.

No se usó ninguna otra herramienta de IA: ni generadores de código en el editor, ni
asistentes dentro de las VMs.

> **Estado de este documento.** Las secciones "Cómo se verificó" describen las
> verificaciones que corresponden a cada punto. Las que dependen de las VMs se
> confirman al desplegar, y su salida queda en `docs/evidencias/`. Mientras ese
> directorio esté vacío, esas verificaciones están **pendientes de ejecución**:
> no se afirma acá nada que no se pueda mostrar en la defensa.

## En qué se usó

### 1. Lectura y análisis de la letra

Se le dio el PDF del obligatorio para extraer los requisitos y armar el plan de
trabajo: entregables, criterios de evaluación, fechas y restricciones. El resultado
fue una lista de requisitos que después se usó como checklist contra el
repositorio.

### 2. Detección de las inconsistencias del código provisto

Se le pidió comparar `cumple.php`, `cumpleanios.sql` y el `README.md` de la cátedra.
De ahí salió la tabla de inconsistencias documentada en el README, incluido el error
de sintaxis de la primera línea del `.sql`
(`DROP DATABASE cumples IF EXISTS;`, con el `IF EXISTS` fuera de lugar).

**Cómo se verificó:** ejecutando el script original en MariaDB, que falla en la
línea 1 con `ERROR 1064 (42000): You have an error in your SQL syntax`. Y leyendo
los tres archivos a mano para confirmar la tabla de diferencias.

### 3. Redacción de los roles de Ansible

Los roles `common`, `mariadb`, `webapp`, `firewall` y `validacion` se escribieron
con asistencia de la IA, discutiendo cada decisión antes de escribirla:
qué módulo específico usar en cada caso, en qué orden poner las tareas, y dónde la
idempotencia se rompe.

**Cómo se verificó:**

```bash
ansible-playbook site.yml --syntax-check
ansible-lint
ansible-playbook site.yml --check --diff        # simulacro
ansible-playbook site.yml                       # primera corrida real
ansible-playbook site.yml                       # segunda corrida: changed=0
```

Y sobre todo: revisando la documentación oficial de cada módulo antes de aceptar los
parámetros sugeridos. Los logs de las corridas están en `docs/evidencias/`.

### 4. Diagnóstico de problemas específicos del entorno

La IA aportó el diagnóstico de varios problemas que no son evidentes y que hacen
fallar este despliegue en particular:

| Problema | Por qué no es evidente |
|---|---|
| SELinux bloquea la conexión saliente de PHP hacia MariaDB | La página carga, pero da error de conexión, y el log de Apache **no menciona SELinux**: hay que mirar `ausearch -m avc` |
| CentOS Stream 9 no tiene `mod_php` | `dnf install php` trae `php-fpm`, y hay que habilitar y arrancar **dos** servicios, no uno |
| `mysql_db state=import` nunca es idempotente | El módulo no puede saber si el script SQL hizo algo, así que reporta `changed` siempre |
| Activar UFW antes de permitir SSH corta la sesión de Ansible | La tarea queda colgada y la VM queda inaccesible por red |
| Una variable de `group_vars` le gana a la opción `-u` | El bootstrap parece ignorar el usuario que se le pasa por línea de comandos |

**Cómo se verificó:** cada uno de estos puntos se comprobó en las VMs. El de
SELinux, deshabilitando el booleano y confirmando que la aplicación deja de
funcionar y que el AVC aparece en `ausearch`. El de la precedencia de variables,
corriendo el bootstrap con `-u` y viendo el fallo de autenticación.

### 5. Redacción de la documentación

Este archivo, el `README.md` y `docs/INSTALL.md` se redactaron con asistencia de la
IA a partir de decisiones ya tomadas y de código ya escrito y probado. Ninguna
afirmación técnica de la documentación se dejó sin verificar contra el
comportamiento real de los servidores.

## En qué NO se usó

- No se le delegó ninguna decisión de arquitectura sin discutirla y entenderla.
  Cada integrante del grupo puede explicar por qué cada tarea está donde está.
- No se copió código sin leerlo. Las sugerencias que no se entendían se
  descartaron o se reemplazaron por algo más simple: código que no se puede
  defender no sirve, y la defensa es eliminatoria.
- No se usó IA para generar las evidencias. Todos los logs y capturas de
  `docs/evidencias/` son salidas reales de las corridas contra las VMs.

## Criterio general

La IA se usó como acelerador y como fuente de diagnóstico, no como autor. El
criterio fue: **todo lo que la IA propuso se verificó contra la documentación
oficial del módulo y contra el comportamiento real de los servidores antes de
quedar en el repositorio.** Los problemas de la lista de arriba se comprobaron
rompiendo la configuración a propósito y viendo el error, no confiando en la
explicación.
