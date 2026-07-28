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

### 1. Instalar Ansible en el nodo de control

El nodo de control es la **VM CentOS Stream**, la misma que después será el
servidor web. Está en su propio inventario y se administra a sí misma.

```bash
sudo dnf install -y ansible-core git sshpass
ansible --version
```

`ansible-core` está en el repositorio AppStream; no hace falta EPEL. `sshpass` es
necesario para el paso 7 (`bootstrap.yml -k -K`): sin él, Ansible falla con
`to use the 'ssh' connection type with passwords ..., you must install the
sshpass program` apenas intenta conectarse con password.

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

Hay dos formas de dejarlas puestas, y no son excluyentes:

```bash
nano inventory/hosts.ini
```

Esto queda escrito en el repo, así que sirve también para el próximo despliegue
contra las mismas VMs. **Si el archivo ya tiene la IP real de cada servidor,
`bootstrap.yml`/`site.yml` no preguntan nada** al arrancar: la pregunta (la hace
`preguntar_ips.yml`, importado como primer play de los dos) sólo aparece cuando
el inventario todavía tiene el placeholder o quedó vacío -- por ejemplo, para
reusar este mismo repo contra **otro** par de VMs sin editar el archivo.

Para fijar otra IP sin tocar el inventario y sin que pregunte nada -- por
ejemplo, al capturar las evidencias, para que el log quede limpio -- se pasan
las IPs por línea de comandos, algo que Ansible entiende sin pedirlas de nuevo:

```bash
ansible-playbook site.yml -e ip_web=172.18.3.119 -e ip_db=172.18.3.120
```

Reemplazar esas dos IPs por las reales, **sin dejarlas entre `< >`**: en bash esos
caracteres son redirección de entrada/salida, así que un valor literal como
`<IP de la VM CentOS>` no da un error de Ansible sino
`bash: syntax error near unexpected token` antes de que el playbook llegue a
correr.

### 6. Comprobar la conectividad antes de seguir

```bash
ansible servidores -m ansible.builtin.ping -k -u sysadmin
```

Cambiar `sysadmin` por el usuario que ya existe en las VMs. **No dejar el nombre
entre `< >`**: en bash esos caracteres son redirección de entrada/salida, así
que pegar algo como `-u <usuario_existente>` literal no falla con un error claro
de Ansible sino con `bash: syntax error near unexpected token`, porque el shell
intenta abrir/crear archivos con esos nombres antes de que Ansible llegue a
correr.

Este chequeo no es una formalidad. `ansible-core` de AppStream puede ser una serie
más vieja que el Python que trae Ubuntu, y los módulos corren con el intérprete del
host administrado: el síntoma es un `SyntaxError` o `ModuleNotFoundError` en
cualquier módulo sobre `db`, mientras `web` funciona perfecto. Si el `ping` no
pasa contra los **dos** hosts, hay que resolverlo antes de desplegar.

También es la primera vez que Ansible se conecta a cada host, así que puede
preguntar si confiar en su fingerprint SSH (`Are you sure you want to continue
connecting...`). Es normal la primera vez; responder `yes` lo agrega a
`known_hosts` y no vuelve a preguntar en corridas posteriores.

### 7. Correr el bootstrap

```bash
ansible-playbook bootstrap.yml -k -K -e bootstrap_ssh_user=sysadmin
```

- `-k` pide el password de SSH
- `-K` pide el password de `sudo`
- `bootstrap_ssh_user` es el usuario que ya existe en las VMs (de nuevo, sin
  `< >`: mismo problema que en el punto 6)

Antes de pedir esos dos passwords, también resuelve la IP de cada VM: si
`inventory/hosts.ini` ya tiene la IP real no pregunta nada; si no, la pide. Para
fijar otra IP sin tocar el archivo: agregar `-e ip_web=172.18.3.119 -e
ip_db=172.18.3.120` al comando de arriba (IPs reales, no entre `< >`, por la
misma razón que arriba).

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

### 8. Desplegar

```bash
ansible-playbook site.yml -e ip_web=172.18.3.119 -e ip_db=172.18.3.120
```

(IPs reales, sin `< >`: en bash son redirección, no texto literal.) Este paso
con `-e` es opcional: si el inventario ya tiene las IPs correctas,
`ansible-playbook site.yml` a secas no pregunta nada tampoco.

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

Con `-e ip_web=... -e ip_db=...` en las cuatro corridas: así ninguna se detiene a
preguntar la IP y el log queda limpio de principio a fin.

```bash
IP_WEB=172.18.3.119
IP_DB=172.18.3.120

ansible-playbook site.yml    -e ip_web=$IP_WEB -e ip_db=$IP_DB | tee docs/evidencias/01-primera-corrida.txt
ansible-playbook site.yml    -e ip_web=$IP_WEB -e ip_db=$IP_DB | tee docs/evidencias/02-segunda-corrida.txt
ansible-playbook site.yml --check --diff -e ip_web=$IP_WEB -e ip_db=$IP_DB | tee docs/evidencias/03-check-mode.txt
ansible-playbook validar.yml -e ip_web=$IP_WEB -e ip_db=$IP_DB | tee docs/evidencias/04-validaciones.txt
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
