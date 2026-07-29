# Obligatorio — Taller de Servidores Linux

Automatización con Ansible del despliegue de una aplicación web PHP sobre dos
servidores Linux: uno con **CentOS Stream** (Apache + PHP) y otro con **Ubuntu
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
    │  CentOS Stream 9       │  :3306     │  Ubuntu Server         │
    │  Apache + php-fpm      │ ─────────► │  MariaDB               │
    │  firewalld: ssh, http  │            │  UFW: ssh, 3306 (solo  │
    │  SELinux: enforcing    │            │  desde la IP de web)   │
    │  + nodo de control     │            │                        │
    └────────────────────────┘            └────────────────────────┘
```

El **nodo de control de Ansible es la propia VM CentOS**. Desde ahí se administra
a sí misma y al servidor de base de datos.

## Requisitos previos

Las dos máquinas virtuales tienen que existir, con sistema operativo instalado, IP
fija y `sshd` andando. Todo lo demás lo hace Ansible.

El detalle exacto de la preparación inicial —qué se hace a mano una sola vez y qué
queda del lado del playbook— está en [docs/INSTALL.md](docs/INSTALL.md). La regla
es sencilla: si se borran las dos VMs hasta el estado posterior al bootstrap, un
solo `ansible-playbook site.yml` reconstruye todo.

Colecciones de Ansible necesarias, declaradas en [requirements.yml](requirements.yml):

```bash
ansible-galaxy collection install -r requirements.yml
```

| Colección | Para qué se usa |
|---|---|
| `ansible.posix` | `firewalld`, `firewalld_info`, `seboolean`, `authorized_key` |
| `ansible.mariadb` | `mariadb_db`, `mariadb_user`, `mariadb_query` |
| `community.general` | `ufw`, `timezone` |

## Cómo se ejecuta

```bash
# 1. Poner las IPs reales de las dos VMs (para este despliegue y los siguientes)
nano inventory/hosts.ini

# 2. Cargar la password de la aplicación en el vault (una sola vez)
#    El procedimiento completo está en group_vars/all/vault.yml.example
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
nano group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml

# 3. Preparar el acceso de Ansible a los dos servidores (una sola vez)
ansible-playbook bootstrap.yml -k -K -e bootstrap_ssh_user=sysadmin

# 4. Desplegar todo
ansible-playbook site.yml
```

Reemplazar `sysadmin` por el usuario que ya existe en las VMs, **sin dejarlo
entre `< >`**: en bash esos caracteres son redirección de entrada/salida, así que
un valor literal como `<usuario_existente>` no da un error de Ansible sino
`bash: syntax error near unexpected token`. Detalle completo en `docs/INSTALL.md`.

La aplicación queda en `http://172.18.3.119/` (la IP de `web`).

Comandos auxiliares:

```bash
ansible-playbook site.yml --check --diff   # simulacro, sin tocar nada
ansible-playbook validar.yml               # solo las validaciones
ansible-playbook site.yml -t db            # solo el servidor de base de datos
ansible-playbook site.yml -t web           # solo el servidor de aplicación
```

## Estructura del repositorio

```
ansible.cfg              inventario, vault, pipelining
requirements.yml         colecciones necesarias
bootstrap.yml            preparación del acceso (se corre una sola vez)
site.yml                 playbook principal
validar.yml              validaciones (site.yml lo importa al final)
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
  FAST-INSTALL.md        lo mismo, como un solo comando encadenado
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
| `cumpleanios_seed` | 3 personas | Filas que se cargan en la tabla |

Las IPs no se repiten en ningún lado: `web_host` y `db_host` se derivan del
inventario con `hostvars`. Cambiar una IP se hace en un solo lugar.

## Reconciliación del código original

El código provisto por la cátedra (`dbappphp`) **no es consistente entre sus
propios archivos y el `.sql` no se puede ejecutar**. Documentarlo es parte del
trabajo, así que acá está el detalle.

| Dato | `cumple.php` | `cumpleanios.sql` | `README.md` de la cátedra | **Este proyecto** |
|---|---|---|---|---|
| Base de datos | `intranet` | `cumples` | `cumples` | **`cumples`** |
| Tabla | `cumpleaños` | `cumpleanios` | — | **`cumpleanios`** |
| Usuario | `intranet_user` | — | `intranet` | **`intranet_user`** |
| Password | `claveSegura` | — | `secureKey` | **en el vault** |

Con esos archivos tal cual, la aplicación no puede funcionar: consulta una tabla
que el script nunca crea, en una base que nunca existe.

Además:

1. La primera línea de `cumpleanios.sql` es `DROP DATABASE cumples IF EXISTS;`,
   que es **sintaxis inválida** en MariaDB. El orden correcto es
   `DROP DATABASE IF EXISTS cumples;`. Tal cual venía, el script fallaba en la
   línea 1.
2. Corregida la sintaxis, un `DROP DATABASE` en cada corrida es lo contrario de la
   idempotencia que pide la letra: borraría los datos y reportaría cambios para
   siempre. Se eliminó y el esquema pasó a
   `CREATE TABLE IF NOT EXISTS` + `INSERT IGNORE`, con una clave única sobre
   `(nombre, fecha)` para que el `INSERT IGNORE` tenga contra qué comparar.
3. La tabla con `ñ` es un identificador problemático entre PHP, MariaDB y dos
   sistemas operativos distintos. Se estandarizó a `cumpleanios`.
4. `cumple.php` traía las credenciales escritas en el código, algo que la letra
   prohíbe versionar. Ahora es un template y las credenciales salen de variables.

El `.php` se mantuvo lo más cerca posible del original: los únicos cambios son los
necesarios para que funcione y no filtre la password.

## Idempotencia

Requisito central de la letra: **la segunda corrida no reporta cambios**.
Los logs de las corridas van en [docs/evidencias/](docs/evidencias/README.md), que
además explica cómo se generan. Lo que se hizo para lograrlo:

- `state: present` en todos los paquetes, nunca `latest`.
- `owner`, `group` y `mode` explícitos en cada `copy` y `template` (el repositorio
  se edita en Windows, donde el bit de ejecución no se versiona).
- `changed_when: false` en toda tarea que solo lee.
- El `bind-address` de MariaDB se escribe como **drop-in propio**
  (`99-ansible.cnf`) y no con `lineinfile` sobre `50-server.cnf`, que es un
  conffile de dpkg.
- `mysql_db state=import` siempre reporta `changed`, porque el módulo no puede
  saber si el script hizo algo. Se lo puso detrás de una consulta a
  `information_schema.tables` y un conteo de filas: solo importa si falta algo.
- `firewalld` con `permanent: true` **y** `immediate: true`: la única combinación
  que es a la vez funcional e idempotente.
- `mysql_user` con `update_password: on_create`, para no reescribir el hash en
  cada corrida.
- `refresh` del índice de `apt` marcado como `changed_when: false`: actualizar un
  índice no modifica el estado del servidor.

Que `ok=` cambie entre corridas es normal. Lo que tiene que dar cero es `changed=`.

## Seguridad

La letra pide explícitamente que no se almacenen contraseñas reales ni claves
privadas. Cómo se cumple:

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
- El usuario de MariaDB se crea como `intranet_user@<IP de web>`, no `@'%'`, y con
  privilegio `SELECT` únicamente: la aplicación solo lee.
- **SELinux queda en `enforcing`.** No se desactiva: se habilita el booleano
  puntual `httpd_can_network_connect_db`, que es más restrictivo que
  `httpd_can_network_connect`.
- Los firewalls abren lo mínimo: en web solo `ssh` y `http` (se cierra `cockpit`,
  que CentOS abre de fábrica); en db solo `ssh` y el `3306` restringido por origen
  a la IP del servidor web.
- La clave privada SSH vive solo en la VM y nunca entra al repositorio.

## Validaciones

`site.yml` importa `validar.yml` al final, así que cada despliegue se verifica a
sí mismo. Todas las tareas son de solo lectura. Se comprueba:

- Que cada servidor tenga el sistema operativo que le corresponde según su grupo
  de inventario (si las IPs están invertidas, el playbook corta ahí con un mensaje
  claro en vez de fallar más adelante de forma confusa).
- Que `mariadb`, `php-fpm` y `httpd` estén corriendo **y habilitados al arranque**.
- Que MariaDB escuche en la IP del servidor y no en `127.0.0.1`.
- Que la tabla tenga exactamente las filas esperadas.
- Que `intranet_user` esté autorizado **solo** desde la IP del servidor web.
- Que SELinux siga en `enforcing` y que el booleano esté activo y persistido
  (leyendo `/sys/fs/selinux/booleans/`, sin usar `command`).
- Que el servidor web alcance el `3306` del servidor de base de datos, lo que
  atraviesa el UFW de verdad y prueba que la conexión es remota.
- Que en el servidor web estén abiertos `http` y `ssh`, y ningún puerto de base de
  datos.
- **Que la página responda por HTTP y liste las personas de la base.** Esta
  consulta se hace `delegate_to` al servidor de base de datos, no desde
  `localhost`: como el nodo de control **es** la VM CentOS, un pedido local
  saldría por la interfaz de loopback sin pasar por firewalld y daría un falso
  positivo con el puerto 80 cerrado.

Cuando una validación falla, el `fail_msg` dice qué revisar y en qué orden.

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

| Integrante | Commits |
|---|---|
| Gastón Delgado | *(ver `git log --author`)* |
| *(a completar)* | |

La modificación individual de cada integrante, que la letra evalúa por separado,
está guiada en [docs/modificacion-individual.md](docs/modificacion-individual.md).
