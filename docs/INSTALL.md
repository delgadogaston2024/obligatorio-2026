# Preparación inicial

### 1. Instalar Ansible en el nodo de control

El nodo de control es la **VM CENTOS / DB**, la misma que después será el
servidor web. Está en su propio inventario y se administra a sí misma.

```bash
sudo dnf install -y ansible-core git
ansible --version
```

### 2. Generar el par de claves SSH

```bash
ssh-keygen -t ed25519 -C "ansible@web" -N ""
```

La clave privada queda en `~/.ssh/id_ed25519` y **nunca** entra al repositorio.
`bootstrap.yml` instala la pública en los dos servidores.

### 3. Clonar el repositorio e instalar las colecciones

```bash
git clone https://github.com/delgadogaston2024/obligatorio-2026.git
cd obligatorio-2026
ansible-galaxy collection install -r requirements.yml
```

Esto es **una sola vez**. Si el repo ya está clonado y lo que se quiere es
traer cambios nuevos se ejecuta:

```bash
cd ~/obligatorio-2026
git pull origin main
```

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

El archivo de passphrase vive **fuera del repositorio**.

### 5. Poner las IPs en el inventario

Son los dos únicos valores que dependen del entorno.

```bash
nano inventory/hosts.ini
```

### 6. Correr el bootstrap

```bash
ansible-playbook bootstrap.yml -k -K -e bootstrap_ssh_user=sysadmin
```

- `-k` pide el password de SSH
- `-K` pide el password de `sudo`
- `bootstrap_ssh_user` es el usuario que ya existe en las VMs

Su primera tarea instala `sshpass` en el equipo WEB (necesario para el
`-k`/`-K` de este mismo comando) en un play local que no necesita SSH todavía.

Esta corrida es también la primera conexión real de Ansible a cada host, y de
paso funciona como chequeo de conectividad.

Antes de conectarse, también registra el fingerprint SSH de `web` y `db` en
`known_hosts`.

Deja en los dos servidores: el usuario `ansible`, la clave pública del nodo de
control y `/etc/sudoers.d/90-ansible` con `NOPASSWD`. Termina verificando que puede
escalar a root sin prompts.

> **Por qué pasamos el usuario con `-e` y no con `-u`:** `group_vars/all/main.yml`
> define `ansible_user: ansible`, y una variable de `group_vars` **le gana** a la
> opción `-u` de la línea de comandos. Las variables de `-e` son las de mayor
> precedencia.

### 7. Desplegar

```bash
ansible-playbook despliegue.yml
```

## Verificación manual

```bash
sudo ss -ltnp | grep 3306                        # en db: escucha en su IP, NO en 127.0.0.1
sudo ufw status verbose                          # en db: 3306 ALLOW IN desde la IP de web
sudo firewall-cmd --list-all                     # en web: solo http y ssh
getenforce                                        # en web: Enforcing
getsebool httpd_can_network_connect_db            # en web
```

La prueba importante es: abrir `http://IP-SERVIDOR-WEB/` (la IP de `web`) en el
navegador.

## Captura de evidencias

```bash
ansible-playbook site.yml                | tee docs/evidencias/01-primera-corrida.txt
ansible-playbook site.yml                | tee docs/evidencias/02-segunda-corrida.txt
ansible-playbook site.yml --check --diff | tee docs/evidencias/03-check-mode.txt
ansible-playbook validar.yml             | tee docs/evidencias/04-validaciones.txt
git log --oneline --graph    > docs/evidencias/git-log.txt
```

El `PLAY RECAP` de la segunda corrida tiene que mostrar `changed=0`,
`unreachable=0` y `failed=0` para **los dos** hosts, confirmando que si no se realizo ningun cambio nada cambia.
