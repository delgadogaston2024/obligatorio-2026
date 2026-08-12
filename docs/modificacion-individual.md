# Evaluación individual

La letra pide que **cada integrante realice una pequeña modificación** al proyecto
usando Ansible, manteniendo la idempotencia, en un **commit individual**. Son 30 de
los 100 puntos, y hay que poder explicarla en la defensa.

Reglas que se desprenden de la letra:

- La modificación tiene que hacerse **con Ansible**, no a mano en el servidor.
- Tiene que **mantener la idempotencia**: después de aplicarla, la segunda corrida
  sigue dando `changed=0`.
- Va en **un commit propio**, con el autor de cada integrante y una explicación
  breve en el mensaje.
- Cada integrante tiene que poder explicar **todo** el trabajo, no solo su parte.

## Cómo hacerla, sin romper nada

```bash
# 1. Configurar el autor propio en este repositorio
git config user.name  "Nombre Apellido"
git config user.email "correo@ort.edu.uy"

# 2. Hacer el cambio en el repositorio (no en el servidor)

# 3. Aplicarlo y verificar que sigue siendo idempotente
ansible-playbook despliegue.yml          # aplica el cambio: changed >= 1
ansible-playbook despliegue.yml          # verifica idempotencia: changed=0

# 4. Commit individual
git commit -am "individual: <qué hace y por qué mantiene la idempotencia>"
```

El paso 3 no es opcional. Una modificación que rompe la idempotencia cuesta más
puntos de los que suma.

## Opciones ordenadas por riesgo

### Muy bajo riesgo

**Agregar una persona a la lista de cumpleaños.** Se agrega una entrada a
`cumpleanios_seed` en `group_vars/all/main.yml` y listo. El rol `mariadb` ya está
escrito para esto: el conteo esperado se deriva de la variable, así que la próxima
corrida detecta que faltan filas, reimporta el esquema, y el `INSERT IGNORE` sobre
la clave única `(nombre, fecha)` carga solo la nueva. La validación también se
ajusta sola.

**Agregar una persona con extra-var (sin tocar el repo).** Pasar nombre y fecha
por `-e`; se suman a `cumpleanios_efectivos` y el mismo mecanismo de importación
controlada carga la fila nueva. Idempotente si se repite la misma corrida con los
mismos `-e`:

```bash
ansible-playbook despliegue.yml \
  -e cumpleanio_extra_nombre="Gaston Delgado" \
  -e cumpleanio_extra_fecha="1990-05-15"
ansible-playbook despliegue.yml \
  -e cumpleanio_extra_nombre="Gaston Delgado" \
  -e cumpleanio_extra_fecha="1990-05-15"   # changed=0
```

Si se corre después **sin** los `-e`, la validación esperará solo las filas del
seed base y fallará si la persona extra sigue en la base. Para dejarla permanente
sin pasar `-e` en cada corrida, agregarla a `cumpleanios_seed`.

### Bajo riesgo, más visible

**Agregar una columna a la tabla.** Por ejemplo `apodo VARCHAR(50)`. Hay que
agregarla al `CREATE TABLE` del template del esquema **y** a un `ALTER TABLE` que
sea idempotente, porque `CREATE TABLE IF NOT EXISTS` no modifica una tabla que ya
existe. Lo idempotente es consultar `information_schema.columns` primero y correr
el `ALTER` solo si la columna falta — el mismo patrón que ya usa el rol para
decidir si importa el esquema. Es una buena modificación para defender porque
obliga a entender por qué el `IF NOT EXISTS` no alcanza.

**Agregar una página de estado.** Un segundo template PHP que muestre la versión de
PHP y si la conexión a la base responde, desplegado por el rol `webapp` con los
mismos permisos y `setype` que la app. Sirve además como evidencia.

**Rotar la password de la aplicación.** Cambiarla en el vault y correr con
`-e db_app_user_recrear=true`, que el rol ya soporta. Demuestra manejo del vault,
que es un tema de defensa probable.

### Riesgo medio — pensarlo antes

**Habilitar HTTPS con un certificado autofirmado.** Requiere `mod_ssl`, generar el
certificado con `community.crypto`, abrir el servicio `https` en firewalld y
agregar un `VirtualHost *:443`. Es la más vistosa y la que más cosas puede romper:
`community.crypto` no está en `requirements.yml` (habría que agregarla) y generar un
certificado es idempotente solo si no se lo regenera en cada corrida.

**Redirigir logs de Apache a un directorio propio con rotación.** Toca
`logrotate`, que es fácil de dejar no idempotente si se usa `lineinfile`.

## Lo que NO conviene hacer

- Tocar el rol `firewall` sin tener la consola de la VM abierta. Un error ahí deja
  el servidor inaccesible por red.
- Cambiar el `bind-address` de MariaDB a `0.0.0.0`: es más permisivo de lo que
  pide la consigna y va en contra de "permitir únicamente lo necesario".
- Cualquier cosa que use `shell` o `command` donde exista un módulo: es un
  requisito explícito de la letra y se nota en la corrección.
- Modificar archivos directamente en los servidores. Además de ir contra la
  consigna, la próxima corrida del playbook lo revierte.

## Antes de la defensa

Ensayar la modificación completa al menos una vez: aplicarla, ver `changed=0` en la
segunda corrida, y poder explicar en una frase por qué es idempotente. Improvisar
esto en la defensa, que es eliminatoria, no es una buena idea.
