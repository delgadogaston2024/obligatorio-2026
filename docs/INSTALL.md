# Preparación inicial

La letra pide que no haya **cambios manuales posteriores** al despliegue. Preparar
el sustrato para que Ansible pueda conectarse es legítimo, pero corresponde
documentarlo para que no parezca trabajo escondido. Eso es este archivo.

La regla que ordena todo: **si se borran las dos VMs hasta el estado posterior al
bootstrap, un solo `ansible-playbook site.yml` reconstruye el sistema completo.**

## Qué se hace una vez y qué hace el playbook

| Preparación inicial (una vez, documentada acá) | Lo hace el playbook, sin excepción |
|---|---|
| Las dos VMs existen, con SO, hostname e IP fija | `httpd`, `php`, `php-fpm`, `php-mysqlnd`, `mariadb-server` |
| `sshd` activo y un usuario con `sudo` | Habilitar y arrancar los servicios con systemd |
| `dnf install ansible-core git` en la VM CentOS | Reglas de firewalld y de UFW |
| `ansible-galaxy collection install -r requirements.yml` | Base de datos, tabla, datos iniciales y usuario de la app |
| Par de claves SSH en el nodo de control | `bind-address` de MariaDB |
| Archivo de passphrase del vault | Booleano de SELinux |
| `git clone` de este repositorio | Despliegue del `.php` desde template |
| `bootstrap.yml` (usuario `ansible`, clave, sudoers) | Zona horaria `America/Montevideo` y NTP |
| | Validaciones del resultado |

Notar que `bootstrap.yml` **también es Ansible**. Lo único verdaderamente manual es
instalar Ansible y generar la clave.

## Paso a paso

Los pasos van en orden y no son intercambiables: en particular, **no correr
`site.yml` (paso 7) antes de que el paso 6 (`bootstrap.yml`) termine sin
errores en los dos servidores**. `site.yml` sólo conecta por clave SSH, y esa
clave la instala `bootstrap.yml`; corrido antes, falla con `Permission denied`.

### 1. Instalar Ansible en el nodo de control

El nodo de control es la **VM CentOS Stream**, la misma que después será el
servidor web. Está en su propio inventario y se administra a sí misma.

```bash
sudo dnf install -y ansible-core git
ansible --version
```

`ansible-core` está en el repositorio AppStream; no hace falta EPEL.

`sshpass` (necesario para conectarse con password, el `-k`/`-K` del paso 6) **no
hace falta instalarlo acá a mano**: `bootstrap.yml` lo instala solo, como su
primera tarea, en un play local (`hosts: localhost`, `connection: local`) que
no necesita SSH todavía. Si en algún momento aparece igual el error
`to use the 'ssh' connection type with passwords ..., you must install the
sshpass program`, es señal de que esa tarea no llegó a correr en el nodo de
control -- revisar con `which sshpass` ahí.

### 2. Generar el par de claves SSH

```bash
ssh-keygen -t ed25519 -C "ansible@web" -N ""
```

La clave privada queda en `~/.ssh/id_ed25519` y **nunca** entra al repositorio.
`bootstrap.yml` instala la pública en los dos servidores.

> Esto incluye instalar la clave del servidor web **en sí mismo**. No es
> redundante: el nodo de control está en su propio inventario, así que Ansible le
> abre una conexión SSH a la misma máquina. Sin la clave en su propio
> `authorized_keys`, `site.yml` falla en `Gathering Facts` con
> `Permission denied (publickey)`, un error muy desconcertante justamente porque
> "es la misma máquina".

### 3. Clonar el repositorio e instalar las colecciones

```bash
git clone https://github.com/delgadogaston2024/obligatorio-2026.git
cd obligatorio-2026
ansible-galaxy collection install -r requirements.yml
```

Esto es **una sola vez**. Si el repo ya está clonado y lo que se quiere es
traer cambios nuevos (por ejemplo, después de un fix), no se clona de nuevo
adentro del mismo directorio -- eso deja un `obligatorio-2026/obligatorio-2026`
anidado y confunde cuál versión se está corriendo. Alcanza con:

```bash
cd ~/obligatorio-2026
git pull origin main
```

Las colecciones quedan en `collections/` dentro del proyecto (así lo define
`ansible.cfg`), directorio que está en `.gitignore`: son dependencias, no código
propio.

### 4. Configurar el vault

El procedimiento completo está en `group_vars/all/vault.yml.example`. Resumido:

```bash
mkdir -p ~/.ansible
printf 'una-passphrase-larga\n' > ~/.ansible/vault_pass_cumples
chmod 600 ~/.ansible/vault_pass_cumples

cp group_vars/all/vault.yml.example group_vars/all/vault.yml
nano group_vars/all/vault.yml        # poner la password real de intranet_user
ansible-vault encrypt group_vars/all/vault.yml
```

El archivo de passphrase vive **fuera del repositorio** a propósito, y además está
listado en `.gitignore` como segunda red.

Para verificar que quedó cifrado:

```bash
head -1 group_vars/all/vault.yml     # tiene que decir $ANSIBLE_VAULT;1.1;AES256
```

### 5. Poner las IPs en el inventario

Son los dos únicos valores que dependen del entorno. Importa no invertirlos: el
host `web` es la VM CentOS y el host `db` la VM Ubuntu. Si se invierten, el rol
`common` lo detecta en la primera tarea y corta con un mensaje explícito.

```bash
nano inventory/hosts.ini
```

Esto queda escrito en el repo, así que sirve también para el próximo despliegue
contra las mismas VMs. **Este paso 5 es sólo para dejar las IPs en el
inventario: todavía no hay que ejecutar ningún `ansible-playbook`.** Eso
empieza recién en el paso 6.

### 6. Correr el bootstrap

```bash
ansible-playbook bootstrap.yml -k -K -e bootstrap_ssh_user=sysadmin
```

- `-k` pide el password de SSH
- `-K` pide el password de `sudo`
- `bootstrap_ssh_user` es el usuario que ya existe en las VMs (de nuevo, sin
  `< >`: mismo problema que en el punto 5)

Su primera tarea instala `sshpass` en el nodo de control (necesario para el
`-k`/`-K` de este mismo comando) en un play local que no necesita SSH todavía,
así que no hay que instalarlo a mano en el paso 1 ni probarlo aparte.

Esta corrida es también la primera conexión real de Ansible a cada host, y de
paso funciona como chequeo de compatibilidad: `ansible-core` de AppStream puede
ser una serie más vieja que el Python que trae Ubuntu, y los módulos corren con
el intérprete del host administrado. Si `db` tiene un Python incompatible, el
síntoma es un `SyntaxError` o `ModuleNotFoundError` justo acá, mientras `web`
(que se administra a sí mismo) sigue funcionando perfecto -- no hace falta un
`ping` de prueba aparte, el error sale igual de claro en esta misma corrida.

Antes de conectarse, también registra el fingerprint SSH de `web` y `db` en
`known_hosts` (con el módulo `known_hosts`, corriendo `ssh-keyscan` desde el
propio nodo de control). No es cosmético: `sshpass` **no puede** responder el
prompt interactivo `Are you sure you want to continue connecting...` que SSH
hace la primera vez que ve un host nuevo -- si esa tarea no llegara a correr,
la conexión con `-k` falla directo con `Host Key checking is enabled and
sshpass does not support this`, sin llegar siquiera a pedir la password.

Deja en los dos servidores: el usuario `ansible`, la clave pública del nodo de
control y `/etc/sudoers.d/90-ansible` con `NOPASSWD`. Termina verificando que puede
escalar a root sin prompts.

> **Por qué se pasa el usuario con `-e` y no con `-u`:** `group_vars/all/main.yml`
> define `ansible_user: ansible`, y una variable de `group_vars` **le gana** a la
> opción `-u` de la línea de comandos. Las variables de `-e` son las de mayor
> precedencia, así que es la única forma limpia de que el primer acceso use el
> usuario viejo y todo lo posterior el nuevo.

Que `site.yml` corra sin ningún prompt no es cosmético: es condición necesaria para
que la evidencia de idempotencia sea un log limpio.

### 7. Desplegar

```bash
ansible-playbook site.yml
```

## Verificación manual

Además de las validaciones automáticas que corren dentro del playbook (los
comandos de abajo asumen que se corren directamente en cada VM, ya sea por
consola o por `ssh sysadmin@<IP>`):

```bash
sudo ss -ltnp | grep 3306                        # en db: escucha en su IP, NO en 127.0.0.1
sudo ufw status verbose                          # en db: 3306 ALLOW IN desde la IP de web
sudo firewall-cmd --list-all                     # en web: solo http y ssh
getenforce                                        # en web: Enforcing
getsebool httpd_can_network_connect_db            # en web
```

Y la prueba que importa: abrir `http://172.18.3.119/` (la IP de `web`) en el
navegador de Windows.

## Captura de evidencias

```bash
ansible-playbook site.yml                | tee docs/evidencias/01-primera-corrida.txt
ansible-playbook site.yml                | tee docs/evidencias/02-segunda-corrida.txt
ansible-playbook site.yml --check --diff | tee docs/evidencias/03-check-mode.txt
ansible-playbook validar.yml             | tee docs/evidencias/04-validaciones.txt
git log --oneline --graph    > docs/evidencias/git-log.txt
```

El `PLAY RECAP` de la segunda corrida tiene que mostrar `changed=0`,
`unreachable=0` y `failed=0` para **los dos** hosts.

## Si algo sale mal

**UFW cortó el acceso al servidor de base de datos.** Pasa si se activa UFW antes
de permitir SSH. Hay que entrar por la consola de la VM y correr `sudo ufw disable`.
El rol `firewall` está escrito con las reglas antes del `enable` justamente para
que esto no ocurra; mientras se lo modifique, conviene tener la consola abierta.

**La página carga pero dice "Error de conexión".** Casi siempre es SELinux
bloqueando la conexión saliente de PHP. El log de Apache **no lo menciona**:

```bash
sudo ausearch -m avc -ts recent
sudo getsebool httpd_can_network_connect_db
```

**Apache devuelve 503.** Falta `php-fpm`: en CentOS Stream 9 no existe `mod_php` y
`httpd` le hace proxy a su socket.

```bash
sudo systemctl status php-fpm
```
