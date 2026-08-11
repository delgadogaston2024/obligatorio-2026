# Obligatorio — Taller de Servidores Linux

Automatización con Ansible del despliegue de una aplicación web PHP sobre dos
servidores: uno con **CentOS Stream** (Apache + PHP) y otro con **Ubuntu
Server** (MariaDB). La aplicación lista los cumpleaños almacenados en la base de
datos, conectándose de forma **remota** desde el servidor web al servidor de base
de datos.

Todo el despliegue se hace desde un único comando. No hay pasos manuales
posteriores a la instalación de Ansible.

## Arquitectura

```
        Windows (navegador)
                 │  HTTP :80
                 ▼
    ┌────────────────────────┐            ┌────────────────────────┐
    │  web                   │  MySQL     │  db                    │
    │  CentOS Stream 10      │  :3306     │  Ubuntu Server 26      │
    │  Apache + php-fpm      │ ─────────► │  MariaDB               │
    │  firewalld: ssh, http  │            │  UFW: ssh, 3306 (solo  │
    │  SELinux: enforcing    │            │  desde la IP de web)   │
    │  + Ansible             │            │                        │
    └────────────────────────┘            └────────────────────────┘
```

El equipo con **Ansible es la propia VM CentOS**. Desde ahí se administra
a sí misma y al servidor de base de datos.

## Requisitos previos

Las dos máquinas virtuales tienen que existir, con sistema operativo instalado, IP, conectividad entre llas y `sshd` andando.

El detalle exacto de la preparación inicial está en [docs/INSTALL.md](docs/INSTALL.md).

Colecciones de Ansible necesarias, declaradas en [requirements.yml](requirements.yml):

```bash
ansible-galaxy collection install -r requirements.yml
```

## Cómo se ejecuta el despliegue

```bash
# 1. Ingresar las IPs reales de las dos VMs
nano inventory/hosts.ini

# 2. Cargar la password de la aplicación en el vault (unica vez)
#    El procedimiento completo está en group_vars/all/vault.yml.example
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
nano group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml

# 3. Preparar el acceso de Ansible a los dos servidores (una sola vez)
ansible-playbook requisito.yml -k -K -e requisito_ssh_user="usuario-existente-en-ambos-equipo"

# 4. Desplegar todo
ansible-playbook despliegue.yml
```

Reemplazar `"usuario-existente-en-ambos-equipo"` por el usuario que ya existe en las VMs, **sin dejarlo

La aplicación queda en `http://IP-DE-WEB/` (la IP de `web`).

Comandos auxiliares:

```bash
ansible-playbook despliegue.yml --check --diff   # simulacro, sin tocar nada
ansible-playbook validar.yml               # solo las validaciones
ansible-playbook despliegue.yml -t db            # solo el servidor de base de datos
ansible-playbook despliegue.yml -t web           # solo el servidor de aplicación
```

## Estructura del repositorio

```
ansible.cfg              inventario, vault, pipelining
requirements.yml         colecciones necesarias
requisito.yml            preparación del acceso (se corre una sola vez)
despliegue.yml            playbook principal
validar.yml              validaciones (despliegue.yml lo importa al final)
inventory/hosts.ini      hosts "web" y "db" dentro del grupo [servidores]
group_vars/
  all/main.yml           variables de la aplicación y de la base
  all/vault.yml          password real, cifrada con ansible-vault
host_vars/
  web.yml, db.yml        variables propias de cada servidor
roles/
  common/                validaciones tempranas, paquetes base, zona horaria, NTP
  mariadb/               instalación, bind-address, base, tabla, datos, usuario
  webapp/                httpd + php-fpm, SELinux, VirtualHost, despliegue
  firewall/              firewalld en web, UFW en db (una rama por SO)
  validacion/            comprobaciones del resultado
docs/
  INSTALL.md             preparación inicial y frontera con el playbook
  uso-de-ia.md           citación del uso de IA
  modificacion-individual.md  guía para la evaluación individual
  evidencias/            logs de las corridas y capturas
```

## Variables

Todo lo que cambia entre equipos está en `group_vars/`/`host_vars/`, no dentro
de los roles. Las principales, en `group_vars/all/main.yml`:

| Variable | Valor | Para qué |
|---|---|---|
| `db_name` | `cumples` | Nombre de la base de datos |
| `db_table` | `cumpleanios` | Nombre de la tabla |
| `db_app_user` | `intranet_user` | Usuario con que la app lee la base |
| `db_app_password` | *(desde el vault)* | Su password |
| `zona_horaria` | `America/Montevideo` | Hora de los dos servidores |
| `cumpleanios_seed` | Personas | Filas que se cargan en la tabla |

## Codigo (`dbappphp`)

El código provisto por la cátedra (`dbappphp`) **no es consistente entre sus
propios archivos y el `.sql` no se puede ejecutar**. Documentarlo es parte del
trabajo, así que acá está el detalle.

| Dato | `cumple.php` | `cumpleanios.sql` | `README.md` de la cátedra | **Este proyecto** |
|---|---|---|---|---|
| Base de datos | `intranet` | `cumples` | `cumples` | **`cumples`** |
| Tabla | `cumpleaños` | `cumpleanios` | — | **`cumpleanios`** |
| Usuario | `intranet_user` | — | `intranet` | **`intranet_user`** |
| Password | `claveSegura` | — | `secureKey` | **en el vault** |

## Idempotencia

Requisito principal: **la segunda corrida no reporta cambios**.

Los logs de las corridas van en [docs/evidencias/](docs/evidencias/README.md), que
además explica cómo se generan. Lo que se hizo para lograrlo:

- `state: present` en todos los paquetes, nunca `latest`.
- `owner`, `group` y `mode` explícitos en cada `copy` y `template` (el repositorio
  se edita en Windows, donde el bit de ejecución no se versiona).
- `changed_when: false` en toda tarea que solo lee.
- `mysql_db state=import` siempre reporta `changed`, porque el módulo no puede
  saber si el script hizo algo. Se lo puso detrás de una consulta a
  `information_schema.tables` y un conteo de filas: solo importa si falta algo.
- `firewalld` con `permanent: true` **y** `immediate: true`: la única combinación
  que es a la vez funcional e idempotente.
- `mysql_user` con `update_password: on_create`, para no reescribir el hash en
  cada corrida.
- `refresh` del índice de `apt` marcado como `changed_when: false`: actualizar un
  índice no modifica el estado del servidor.

## Seguridad

La letra pide explícitamente que no se almacenen contraseñas reales ni claves
privadas:

- La password de la aplicación vive en `group_vars/all/vault.yml`, cifrada con
  `ansible-vault`.
- La passphrase del vault vive **fuera del repositorio**, en
  `~/.ansible/vault_pass_cumples` con permisos `0600`, referenciada desde
  `ansible.cfg`. Además está en `.gitignore` como segunda red.
- `no_log: true` en toda tarea que toca la password, para que no aparezca en la
  salida del playbook, que es justamente lo que se entrega como evidencia.
- Las credenciales renderizadas quedan en `/var/www/private/cumples/`, **fuera del
  DocumentRoot**, con permisos `0640` y grupo `apache`: ninguna URL puede llegar a
  servirlas.
- El usuario de MariaDB se crea como `intranet_user@<IP de web>` con
  privilegio `SELECT` únicamente.
- **SELinux queda en `enforcing`.** No se desactiva: se habilita el booleano
  puntual `httpd_can_network_connect_db`, que es más restrictivo que
  `httpd_can_network_connect`.
- Los firewalls abren lo mínimo: en web solo `ssh` y `http` (se cierra `cockpit`,
  que CentOS abre de fábrica); en db solo `ssh` y el `3306` restringido por origen
  a la IP del servidor web.
- La clave privada SSH vive solo en la VM y nunca entra al repositorio.

## Validaciones

`despliegue.yml` importa `validar.yml` al final, así que cada despliegue se verifica a
sí mismo. Todas las tareas son de solo lectura. Se comprueba:

- Que cada servidor tenga el sistema operativo que le corresponde según su grupo
  de inventario.
- Que `mariadb`, `php-fpm` y `httpd` estén corriendo **y habilitados al arranque**.
- Que MariaDB escuche en la IP del servidor y no en `127.0.0.1`.
- Que la tabla tenga exactamente las filas esperadas.
- Que `intranet_user` esté autorizado **solo** desde la IP del servidor web.
- Que SELinux siga en `enforcing` y que el booleano esté activo y persistido.
- Que el servidor web alcance el `3306` del servidor de base de datos, lo que
  atraviesa el UFW de verdad y prueba que la conexión es remota.
- Que en el servidor web estén abiertos `http` y `ssh`, y ningún puerto de base de
  datos.
- **Que la página responda por HTTP y liste las personas de la base.**

## Uso de `shell` y `command`

La letra pide evitarlos cuando existe un módulo adecuado. En todo el proyecto hay
**dos** usos, los dos de solo lectura y con `changed_when: false`:

| Comando | Por qué no hay módulo |
|---|---|
| `httpd -t` | No se puede usar `validate: httpd -t -f %s` en el template del VirtualHost: eso lo valida como si fuera el `httpd.conf` principal, sin los módulos cargados, y falla siempre. |
| `ufw status verbose` | No existe un módulo `ufw_info`, a diferencia de `firewalld_info`. Sirve además como evidencia legible en el log. |

Todo lo demás sale por módulos específicos.

## Licencia

El código PHP y SQL original es de la cátedra y está bajo **GPL v2**; se conserva
el archivo [LICENSE](LICENSE) que lo acompaña. La automatización de este
repositorio se publica bajo la misma licencia.

## Uso de inteligencia artificial

Declarado en [docs/uso-de-ia.md](docs/uso-de-ia.md), como pide la letra.

## Integrantes

| Integrante |
|---|
| Gastón Delgado 293149 |
| Renzo Moretti 3567195  |
