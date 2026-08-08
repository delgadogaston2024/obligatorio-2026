# Uso de herramientas de inteligencia artificial

La letra pide citar las herramientas de IA utilizadas y el contexto en que se
usaron. Este documento lo hace, e incluye además **cómo se verificó** cada cosa
generada, porque lo que no se verifica no se puede defender.

## Herramienta utilizada

**Gemini, a través de Visual Studio Code**, usado de forma interactiva desde
la máquina de trabajo.

No se usó ninguna otra herramienta de IA: ni generadores de código en el editor, ni
asistentes dentro de las VMs.

## En qué se usó

### 1. Lectura y análisis de la letra

Se le dio el PDF del obligatorio para extraer los requisitos y armar el plan de
trabajo: entregables, criterios de evaluación, fechas y restricciones. El resultado
fue una lista de requisitos que después se usó como checklist contra el
repositorio.

### 2. Redacción de los roles de Ansible

Los roles `common`, `mariadb`, `webapp`, `firewall` y `validacion` se escribieron
con asistencia de la IA, discutiendo cada decisión antes de escribirla:
qué módulo específico usar en cada caso, en qué orden poner las tareas, y dónde la
idempotencia se rompe.

**Cómo se verificó:**

```bash
ansible-playbook despliegue.yml --syntax-check
ansible-lint
ansible-playbook despliegue.yml --check --diff        # simulacro
ansible-playbook despliegue.yml                       # primera corrida real
ansible-playbook despliegue.yml                       # segunda corrida: changed=0
```

### 3. Diagnóstico de problemas específicos del entorno

La IA aportó el diagnóstico de varios problemas que no son evidentes y que hacen
fallar este despliegue en particular:

| Problema | Por qué no es evidente |
|---|---|
| CentOS Stream 9 no tiene `mod_php` | `dnf install php` trae `php-fpm`, y hay que habilitar y arrancar **dos** servicios, no uno |
| `mysql_db state=import` nunca es idempotente | El módulo no puede saber si el script SQL hizo algo, así que reporta `changed` siempre |
| Una variable de `group_vars` le gana a la opción `-u` | El despliegue parece ignorar el usuario que se le pasa por línea de comandos |

**Cómo se verificó:** cada uno de estos puntos se comprobó en las VMs.

### 4. Redacción de la documentación

Este archivo, el `README.md` y `docs/INSTALL.md` se redactaron con asistencia de la
IA a partir de decisiones ya tomadas y de código ya escrito y probado, mas que nada para poderlo hacer mas profesional y con formato markdown.
