# Instalación rápida

Todo el proceso de [INSTALL.md](INSTALL.md), pero como un solo comando
encadenado, para correr en el nodo de control (la **VM CentOS Stream**) con
una VM limpia:

```bash
sudo dnf install -y ansible-core git && \
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -C "ansible@web" -N "" && \
git clone https://github.com/delgadogaston2024/obligatorio-2026.git && cd obligatorio-2026 && \
ansible-galaxy collection install -r requirements.yml && \
mkdir -p ~/.ansible && nano ~/.ansible/vault_pass_cumples && chmod 600 ~/.ansible/vault_pass_cumples && \
cp group_vars/all/vault.yml.example group_vars/all/vault.yml && nano group_vars/all/vault.yml && ansible-vault encrypt group_vars/all/vault.yml && \
ansible-playbook bootstrap.yml -k -K -e bootstrap_ssh_user=sysadmin && \
ansible-playbook site.yml
```

Reemplazar `sysadmin` por el usuario que ya existe en las VMs (sin `< >`: en
bash son redirección, no texto literal). Los dos `nano` paran el comando para
escribir, a mano, la passphrase del vault y la password real de
`intranet_user`; guardar y cerrar continúa la cadena. Si `inventory/hosts.ini`
no tiene todavía la IP real de `web` y `db` (este repo ya las trae), editarlo
*antes* de correr el comando de arriba.

Si el repo ya está clonado y solo hace falta traer un fix, no se repite el
comando entero: alcanza con `git pull origin main` seguido de
`ansible-playbook site.yml` (y de `ansible-galaxy collection install -r
requirements.yml --force` si `requirements.yml` cambió).

## Qué hace cada paso

- **`dnf install ansible-core git`**: está en el repositorio AppStream de
  CentOS Stream, no hace falta EPEL.
- **`ssh-keygen`**: la clave privada queda en `~/.ssh/id_ed25519` y nunca entra
  al repositorio. `bootstrap.yml` instala la pública en los dos servidores,
  incluido el propio nodo de control (se administra a sí mismo, así que
  Ansible se conecta por SSH a su propia máquina).
- **`git clone` + `collection install`**: las colecciones quedan en
  `collections/` (lo define `ansible.cfg`), ignorado por git: son
  dependencias, no código propio.
- **Vault**: la passphrase vive fuera del repositorio, en
  `~/.ansible/vault_pass_cumples` con `chmod 600`; la password real de la app
  queda cifrada en `group_vars/all/vault.yml`. Verificar con
  `head -1 group_vars/all/vault.yml` (tiene que decir
  `$ANSIBLE_VAULT;1.1;AES256`).
- **`bootstrap.yml -k -K -e bootstrap_ssh_user=...`**: se corre una sola vez,
  con el usuario y password que ya existen en las VMs. Instala `sshpass` solo
  (no hace falta a mano), registra el fingerprint SSH de cada servidor,
  crea el usuario `ansible`, le instala la clave pública y le da sudo sin
  password. El usuario va con `-e` y no con `-u` porque una variable de
  `group_vars` (`ansible_user: ansible`) le gana a `-u`, y `-e` es lo único
  con más precedencia.
- **`site.yml`**: despliega todo y se valida a sí mismo al final, sin ningún
  prompt (usa la clave y el sudo que dejó el bootstrap) -- condición necesaria
  para que la evidencia de idempotencia sea un log limpio. No correrlo antes
  de que `bootstrap.yml` haya terminado sin errores en los dos servidores: sin
  la clave instalada, falla con `Permission denied`.

Para el detalle paso a paso, la verificación manual, la captura de evidencias
y qué hacer si algo sale mal, ver [INSTALL.md](INSTALL.md).
