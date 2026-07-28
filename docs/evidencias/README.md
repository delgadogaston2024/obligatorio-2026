# Evidencias

Salidas reales de las corridas contra las VMs. **Todavía vacío**: se completa al
desplegar.

La letra pide dos evidencias en particular: que la aplicación funciona, y que una
segunda ejecución es idempotente.

## Archivos que van acá

| Archivo | Cómo se genera |
|---|---|
| `01-primera-corrida.txt` | `ansible-playbook site.yml \| tee docs/evidencias/01-primera-corrida.txt` |
| `02-segunda-corrida.txt` | mismo comando; el `PLAY RECAP` debe dar `changed=0` en los dos hosts |
| `03-check-mode.txt` | `ansible-playbook site.yml --check --diff \| tee ...` |
| `04-validaciones.txt` | `ansible-playbook validar.yml \| tee ...` |
| `05-app-navegador.png` | Captura del navegador **en Windows** contra `http://172.18.3.119/` (la IP de `web`), con la barra de direcciones visible en el recorte: es lo que prueba que no es `localhost` |
| `06-recap-idempotencia.png` | Captura del `PLAY RECAP` de la segunda corrida |
| `07-estado-servidores.txt` | Salida de los comandos de verificación manual de `docs/INSTALL.md` |
| `git-log.txt` | `git log --oneline --graph > docs/evidencias/git-log.txt` |

## Qué tiene que mostrar la segunda corrida

```
PLAY RECAP ***********************************************************
db  : ok=NN  changed=0  unreachable=0  failed=0  skipped=N  rescued=0  ignored=0
web : ok=NN  changed=0  unreachable=0  failed=0  skipped=N  rescued=0  ignored=0
```

Que `ok=` difiera entre la primera y la segunda corrida es normal. Lo que tiene que
dar cero es `changed=`.

## Antes de armar el zip de la entrega

Los documentos de texto van en **PDF** dentro del zip, que no puede pasar de 40 MB.
Las capturas en PNG o JPG. Nada de video, ISOs ni discos virtuales.

Y el chequeo de secretos, que conviene hacer siempre:

```bash
git log --all -p | grep -i -n "$(cat ~/.ansible/vault_pass_cumples)" ; echo "---"
git ls-files | grep -E 'vault_pass|id_ed25519|id_rsa|\.pem$|\.key$'
head -1 group_vars/all/vault.yml    # $ANSIBLE_VAULT;1.1;AES256
```

Los dos primeros no tienen que devolver nada. Si alguna vez una password real
quedó en un commit, se **rota la password**; no se reescribe el historial.
