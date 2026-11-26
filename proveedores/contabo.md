# Contabo: lo que deja la imagen de Debian 13

Complemento del [manual de despliegue de Debian 13 Trixie server](../13-server.md) para VPS de Contabo. Aplica la sección 5 de ese manual, la revisión del aprovisionamiento, a lo que se observó el 2026-10-16 en un VPS de Contabo con la imagen de Debian 13: cloud-init 25.1.4, datasource NoCloud desde una ISO `cidata` en `/dev/sr0` y red con netplan sobre `systemd-networkd`. Otras imágenes o fechas pueden diferir. La sección 6 aplica la sección 11.6 de ese manual, el firewall del proveedor, al firewall de red de Contabo.

**Autor:** [Calú](https://github.com/calu777)
**Fecha de inicio:** 2026-10-18
**Versión:** v0.2

## 1. Cuentas y contraseñas

Contabo entrega acceso solo como root, con contraseña, y crea una cuenta `debian` (UID 1000) en el grupo `sudo` con `NOPASSWD:ALL` en `/etc/sudoers.d/90-cloud-init-users`. El usuario administrativo se crea con la sección 4.2.4 del manual de Debian.

Las contraseñas de root y de `debian` están en texto claro en `/var/lib/cloud/instance/user-data.txt` y en la ISO `cidata`, legible por el grupo `cdrom`. El panel no permite expulsar la ISO, así que la medida eficaz es cambiar las contraseñas y retirar la cuenta, no borrar archivos:

```bash
sudo passwd root
sudo deluser --remove-home debian
sudo rm /etc/sudoers.d/90-cloud-init-users
sudo visudo -c
getent group sudo
```

`getent group sudo` debe listar solo al usuario administrativo. La contraseña nueva de root es provisional: la sección 9.6 del manual de Debian bloquea la cuenta (`passwd -S root` muestra `L`). Desde entonces la consola VNC del panel ya no admite entrar como root, y el rescate se hace con el usuario administrativo, que sí tiene contraseña.

### 1.1 La ISO `cidata`

La ISO `cidata` (`/dev/sr0`, sin montar) contiene `meta-data`, `network-config`, un `vendor-data` vacío y el `user-data`, idéntico a `/var/lib/cloud/instance/user-data.txt`. Ese `user-data` trae en texto claro la contraseña inicial de root (`chpasswd`) y otra en la clave `password` para el usuario por defecto, además del `bootcmd` de la sección 2, los archivos de la sección 5 y `ssh_deletekeys: 0`. El dispositivo es `root:cdrom` con modo 660, y el único miembro de `cdrom` es `debian`, así que esa cuenta puede leer la ISO sin montarla.

No se monta: cloud-init la lee directamente del dispositivo y, una vez desactivado (sección 5.1 del manual), nadie la necesita; el panel tampoco permite expulsarla. Con la contraseña de root cambiada y la cuenta `debian` eliminada, la ISO queda inerte, y `getent group cdrom` debe mostrar el grupo sin miembros. Por `ssh_deletekeys: 0`, se comprueba con `ls -l --time-style=long-iso /etc/ssh/ssh_host_*` que las claves de host tengan la fecha del primer arranque; si son anteriores, se regeneran con `sudo rm /etc/ssh/ssh_host_*` y `sudo dpkg-reconfigure openssh-server`. En el VPS de referencia (2026-11-19) tenían la fecha del primer arranque: se generaron en la instancia pese a `ssh_deletekeys: 0`. La contraseña inicial queda también copiada en `/var/lib/cloud/instances/<id>/`; se localiza y se borra con el procedimiento de la sección 5.1 del manual.

## 2. SSH

Un `bootcmd` del user-data escribe `PermitRootLogin yes` en `/etc/ssh/sshd_config`, y la imagen deja `50-cloud-init.conf` con `PasswordAuthentication yes`, que se lee antes que el archivo principal. Resultado: SSH con contraseña abierto a internet, root incluido. En el VPS observado hubo más de 160 intentos fallidos en las primeras horas; en otro, 5970 contraseñas fallidas desde 17 direcciones en seis horas y media.

El `01-hardening.conf` de la sección 10.1 del manual de Debian anula los dos valores. Si no se va a aplicar enseguida, un drop-in provisional cierra la contraseña; requiere el acceso por llave ya probado (6.3):

```bash
sudo tee /etc/ssh/sshd_config.d/00-provisional.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication)'
```

Las tres directivas deben salir en `no`. En la sección 10 se borran `00-provisional.conf` y `50-cloud-init.conf`, que no se regenera con cloud-init desactivado.

La línea del `bootcmd` en el archivo principal no tiene efecto, pero se devuelve al valor que trae Debian para que el archivo no engañe a quien lo lea:

```bash
sudo grep -n '^#\?PermitRootLogin' /etc/ssh/sshd_config
sudo sed -i 's/^PermitRootLogin yes$/#PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo sshd -t && sudo sshd -T | grep -i '^permitrootlogin'
```

Debe seguir mostrando `permitrootlogin no`. No hace falta recargar `sshd`.

La imagen añade además `ClientAliveInterval 120` al final del archivo principal. Tampoco tiene efecto, porque el `Include` de `sshd_config.d` va al principio y sshd se queda con el primer valor que lee, el de `01-hardening.conf`; se retira con el mismo criterio:

```bash
sudo sed -i '/^ClientAliveInterval 120$/d' /etc/ssh/sshd_config
sudo sshd -t && sudo sshd -T | grep -i '^clientaliveinterval'
```

Debe mostrar el valor de `01-hardening.conf`. Del primer arranque queda también `/etc/ssh/sshd_config.ucf-dist`, con el mismo `PermitRootLogin yes`. sshd no lo lee, así que es inerte; puede borrarse con `sudo rm /etc/ssh/sshd_config.ucf-dist`.

## 3. Unidades de cloud-init en `failed`

Tras el primer arranque falla `cloud-init-network.service`, y en algunos VPS también `cloud-init-main.service`, por el `bootcmd`: termina en `pkill -HUP sshd` y se ejecuta antes de que arranque `sshd`, así que `pkill` devuelve 1. `/var/log/cloud-init.log` lo registra como `Failed to run bootcmd module bootcmd`, con un script de tres órdenes que sale con `Exit code: 1` y sin nada en stdout ni stderr. Las líneas anteriores, las que tocan `sshd_config`, sí se aplican. La causa se dedujo del script y del código de salida, sin trazar la ejecución: el script temporal se borra al terminar. El módulo tiene frecuencia `always`, así que se repite en cada arranque. Desactivar cloud-init (sección 5.1 del manual de Debian) elimina el error, y en los arranques posteriores no vuelve a aparecer.

## 4. Red y ruta estática

La red está en `/etc/netplan/50-cloud-init.yaml`, completa y estática: IPv4, IPv6 con la ruta por defecto hacia `fe80::1` (on-link) y los DNS de Contabo. La aplica `systemd-networkd`, y con cloud-init desactivado (sección 5.1 del manual de Debian) el archivo ya no se regenera. La imagen añade `/etc/cron.d/staticroute`, que en cada arranque intenta hacer pasar la ruta de la subred por el gateway:

```text
@reboot root ip route replace $(ip route list dev eth0 scope link | head -n1 | awk '{ print $1 }') via $(ip route list dev eth0 | awk '/default/{ print $3 }') dev eth0 &>/dev/null
```

La ruta existe porque Contabo aísla a cada cliente en la capa 2: los demás VPS de la subred no responden al ARP, y solo el gateway los alcanza. Con la ruta `scope link` que crea el kernel para la subred, el tráfico hacia un vecino se intenta por ARP y no llega; con la ruta por el gateway, llega. La salida a internet no depende de esta ruta, porque va por la ruta por defecto.[^vecinos]

El cron no es fiable. `@reboot` corre en cuanto arranca `cron`, a menudo antes de que `systemd-networkd` levante `eth0`: `ip route list dev eth0 scope link` sale vacío, la orden falla y, como su salida va a `/dev/null`, no queda rastro. La línea, además, termina en CRLF y usa `&>`, que `/bin/sh` (dash) no entiende como redirección sino como `&` seguido de `>`, así que la orden corre en segundo plano. Cuál de los dos estados quedó en el último arranque se ve con:

```bash
ip -4 route show dev eth0 | grep -v '^default'
```

`via <gateway>` indica que el cron se aplicó; `proto kernel scope link`, que perdió la carrera.

> **ATENCIÓN:** este cron no se borra al limpiar la imagen. Es la única pieza de red fuera de netplan; si la red se migra según la sección 7.5 del manual de Debian, la decisión sobre esta ruta pasa a la configuración nueva.

Si el servidor no necesita llegar a otros VPS de su misma subred, la ruta no hace falta y el cron puede quedarse o retirarse sin efecto sobre el resto. Si la necesita, la ruta se declara en netplan, que la aplica cuando la interfaz ya está arriba, y el cron se aparta. Se añade al bloque `routes` de `eth0`, con la subred y el gateway que muestra `ip route`:

```yaml
      - to: "<subred>"
        via: "<gateway>"
```

```bash
sudo netplan try
ip -4 route show dev eth0 | grep -v '^default'
sudo mv /etc/cron.d/staticroute /root/
```

`netplan try` revierte el cambio si no se confirma a tiempo, y la segunda orden debe mostrar la subred `via` el gateway. Este caso está pendiente de validación: la ruta de netplan convive con la del kernel para la misma subred, y no se ha comprobado cuál prevalece tras un reinicio.

Para comprobar en un servidor concreto qué alcanza cada ruta, sin reiniciar y con vuelta atrás programada, se prueban unas cuantas direcciones de la subred con las dos rutas:

```bash
GW="$(ip -4 route show default dev eth0 | awk '{print $3; exit}')"
SUBRED="$(ip -4 route show dev eth0 | awk '$1 != "default" {print $1; exit}')"
SRC="$(ip -4 -o addr show dev eth0 | awk '{split($4, a, "/"); print a[1]; exit}')"
ORIG="$(ip -4 route show "${SUBRED}" dev eth0)"
: "${GW:?}" "${SUBRED:?}" "${SRC:?}" "${ORIG:?}"
VECINOS="<ip1> <ip2> <ip3> <ip4> <ip5>"     # direcciones de la misma subred
probar() { for ip in ${VECINOS}; do ping -c1 -W1 "${ip}" >/dev/null && echo "responde ${ip}"; done; }

sudo systemd-run --on-active=5min --unit=restaurar-ruta ip route replace ${ORIG} dev eth0
sudo ip route replace "${SUBRED}" dev eth0 proto kernel scope link src "${SRC}"
echo "== scope link"; probar; ip -4 neigh show dev eth0
sudo ip route replace "${SUBRED}" via "${GW}" dev eth0
echo "== por el gateway"; probar
sudo ip route replace ${ORIG} dev eth0
sudo systemctl stop restaurar-ruta.timer
ip -4 route show dev eth0
```

Con `scope link` no debe responder ninguna dirección, y la tabla de vecinos las deja en `FAILED` o `INCOMPLETE`; por el gateway responden las que aceptan ping. La última orden debe mostrar la ruta del principio. `ORIG` va sin comillas a propósito, para que `ip` reciba cada palabra de la ruta por separado. El gateway no responde a `ping` con ninguna de las dos rutas, así que su silencio no indica nada.

[^vecinos]: Validado el 2026-11-20 en un VPS con una /22 de Contabo: con la ruta `scope link`, 0 de 22 direcciones de la subred respondieron y todas quedaron en `FAILED` o `INCOMPLETE` en la tabla ARP; por el gateway respondieron 14 de 22, incluidas las dos contiguas. En otro VPS, con una /20, el ARP falló igual en las 10 direcciones probadas. En los dos, el cron había perdido la carrera con `systemd-networkd` en los arranques revisados, y la salida a internet, el correo y las actualizaciones funcionaban con la subred `scope link`.

## 5. Otros archivos y paquetes

| Archivo | Contenido | Tratamiento |
| --- | --- | --- |
| `/etc/cron.hourly/free` | `echo 1 > /proc/sys/vm/drop_caches` | Se retira |
| `/etc/cron.hourly/fstrim` | `fstrim /` | Se retira si `fstrim.timer` está activo |
| `/etc/sysctl.d/10-panic.conf` | `kernel.panic = 10` | Se conserva: tras un pánico del kernel el VPS se reinicia solo a los 10 s, en lugar de quedar colgado hasta que alguien entre al panel |
| `/etc/cloud/cloud.cfg.d/99-ssh.cfg` | `disable_root: 0`, `ssh_pwauth: 1`, `ssh_deletekeys: 0` | Sin efecto con cloud-init desactivado |
| `/etc/motd` | Banner de Contabo | Cosmético |

`free` escribe `1` en `drop_caches` cada hora, a los :17 de `cron.hourly`, y con eso descarta la caché de páginas: el contenido de archivos ya leídos que el kernel guarda en la RAM libre. Esa memoria no se le quita a ningún proceso; el kernel la recupera al instante cuando una aplicación la pide, por eso `free -m` la cuenta como disponible. Vaciarla solo hace que la columna `libre` se vea más alta y obliga a releer del disco bibliotecas, bases de datos y archivos que ya estaban en memoria, con un pico de E/S y respuestas más lentas después de cada vaciado. La documentación del kernel reserva `drop_caches` para pruebas y depuración, no para la operación normal. `fstrim`, por su parte, repite lo que ya hace `fstrim.timer` cada semana.

Primero se apartan, para poder devolverlos si algo falla:

```bash
systemctl is-enabled fstrim.timer              # debe responder enabled
sudo install -d -m 0700 /root/proveedor-ori
for f in free fstrim; do
  [ -e "/etc/cron.hourly/${f}" ] && sudo mv "/etc/cron.hourly/${f}" /root/proveedor-ori/
done
sudo run-parts --test /etc/cron.hourly
```

`run-parts --test` ya no debe listarlos, pero sí lo propio que haya en ese directorio, como el `server-check` de la sección 19.2 del manual; solo se retiran estos dos. Tras la siguiente ejecución de `cron.hourly` (a los :17) la columna `búf/caché` de `free -m` debe seguir creciendo.

Para devolverlos, se mueven de `/root/proveedor-ori/` a `/etc/cron.hourly/`. Tras un reinicio y unos días sin incidencias, se comprueba que no los volvió a crear la imagen y que la caché se mantiene. Si todo está bien, se borran definitivamente:

```bash
ls -l /etc/cron.hourly/                          # no deben estar free ni fstrim
systemctl list-timers fstrim.timer --no-pager    # debe tener próxima ejecución
free -m                                          # búf/caché acumulada desde el arranque
sudo rm -i /root/proveedor-ori/free /root/proveedor-ori/fstrim
sudo rmdir --ignore-fail-on-non-empty /root/proveedor-ori
```

Si `free` o `fstrim` reaparecen en `/etc/cron.hourly/` tras el reinicio, algo de la imagen los regenera en cada arranque. Se desactiva cloud-init (sección 5.1 del manual) y se repite el procedimiento.

Además, en el primer arranque cloud-init:

- Hace un `full-upgrade`, que puede dejar un kernel pendiente de reinicio (8.4).
- Instala `cron`, que arrastra `exim4` y `bsd-mailx`. `exim4` se retira en la sección 20.3.1 del manual; `bsd-mailx` se conserva como cliente `mail` (20.7).
- Fija la zona horaria en `Europe/Berlin` (7.3) y un hostname `vmiNNNNNNN.contaboserver.net` (7.1 y 7.2).
- Deja las fuentes de APT en `debian.sources` (8.2).
- Deja `unattended-upgrades` activo con los orígenes de serie, antes de llegar a la sección 16.
- La VM no tiene dispositivo vsock, así que `systemd-ssh-generator` registra `Failed to query local AF_VSOCK CID` en cada arranque y en cada `daemon-reload`. Es inofensivo; para que `journalctl -p err` quede limpio, se enmascara el generador según la sección 2.5.2 del manual.

## 6. Firewall de red

Contabo ofrece un firewall de red gratuito, que aplica a sus VPS la sección 11.6 del manual de Debian. Se crea en el panel y se asigna a una o varias instancias. Al activarlo descarta todo lo entrante sin regla y no filtra lo saliente. Sin una regla para ICMP descarta también el ping, por IPv4 y por IPv6, así que desde fuera el servidor se comprueba con una conexión al puerto de SSH y no con `ping`. Cada regla define protocolo, puertos de destino, acción y una lista de orígenes CIDR separada en IPv4 e IPv6, de modo que una sola regla admite todos los rangos de un borde. Un mismo firewall puede estar asignado a varias instancias, y entonces sus reglas son las de todas: cambiar una afecta a cada servidor que lo tenga asignado.

Una particularidad condiciona el procedimiento: la API no añade reglas sueltas, sino que reemplaza el conjunto completo. Cada cambio se hace leyendo las reglas actuales, modificando la que corresponde y enviando el conjunto entero. El cambio filtra sin un paso de sincronización posterior.

### 6.1 Firewall y regla de SSH

Se crean desde el panel, en el apartado Firewall. La regla de SSH, con el puerto de la sección 10.1 del manual y origen abierto, va antes de asignar el firewall a la instancia, o el firewall corta también la administración. El panel añade al final una regla `Block all traffic` con acción `drop`, que es la que descarta lo demás. Validado en el VPS de referencia el 2026-11-19 con la sección 11.6.3 del manual: el puerto SSH conecta y el 22 agota el tiempo.

### 6.2 Acceso a la API

Las órdenes se lanzan desde la estación de trabajo, con el CLI oficial `cntb`: unas credenciales de la API en el servidor darían control sobre todas las instancias de la cuenta. `cntb` se distribuye como ejecutable estático; se fija la versión en la variable y el bloque se repite para actualizarlo:

```bash
CNTB_VER="v1.7"
CNTB_ARCH="$(dpkg --print-architecture)"     # amd64 en una estación x86_64
CNTB_TMP="$(mktemp -d)"
curl -fsSL -o "${CNTB_TMP}/cntb.tar.gz" \
  "https://github.com/contabo/cntb/releases/download/${CNTB_VER}/cntb_${CNTB_VER}_linux_${CNTB_ARCH}.tar.gz"
tar -xzf "${CNTB_TMP}/cntb.tar.gz" -C "${CNTB_TMP}"
sudo install -m 0755 "${CNTB_TMP}/cntb" /usr/local/bin/cntb
rm -rf "${CNTB_TMP}"
cntb version
```

`cntb version` debe mostrar la versión fijada en `CNTB_VER`.[^cntb-instalacion]

`cntb` se autentica con cuatro datos de la sección Seguridad y acceso del panel, apartado API: el ID de cliente, el secreto de cliente, el nombre de usuario de la API (el correo de la cuenta) y la contraseña. El panel actual no tiene una contraseña de API propia: es la misma con la que se entra al panel, y funciona con el segundo factor (2FA) activo en la cuenta.

`cntb config set-credentials` guarda los cuatro datos en `~/.cntb.yaml`, aunque su ayuda menciona `~/.cntb.yml`. Para que los secretos no queden en el historial del shell, se leen primero en variables. Esta línea se ejecuta sola, porque cada `read` tomaría como respuesta la línea que viniera pegada a continuación:

```bash
read -rp 'ID de cliente: ' C_ID && read -rsp 'Secreto de cliente: ' C_SEC && echo && read -rp 'Usuario de la API: ' C_USER && read -rsp 'Contraseña: ' C_PASS && echo
```

Después, el siguiente bloque, una sola vez:

```bash
cntb config set-credentials --oauth2-clientid "${C_ID}" --oauth2-client-secret "${C_SEC}" \
  --oauth2-user "${C_USER}" --oauth2-password "${C_PASS}"
unset C_ID C_SEC C_USER C_PASS
chmod 600 ~/.cntb.yaml
sed -E -i "/^oauth2-(clientid|client-secret|user|password): [^']/{s/'/''/g; s/^([^:]+): (.*)$/\1: '\2'/}" ~/.cntb.yaml
```

**Si el bloque falla a medias, no se repite tal cual.** Tras el `unset` las variables están vacías, y un segundo `set-credentials` sobrescribe el archivo con las cuatro credenciales en blanco. Se vuelve a la línea de los `read`.

`set-credentials` escribe los valores sin comillas. Si alguno empieza por un carácter con significado en YAML, como `&`, `*` o `!`, el archivo deja de interpretarse y `cntb` responde `clientId is empty` aunque las credenciales estén escritas. El `sed` encierra los cuatro valores entre comillas simples, duplica las comillas simples que contengan y no actúa sobre los que ya están entrecomillados, de modo que puede repetirse.

Para comprobar el archivo no se usa `cntb config view`, que imprime los secretos; basta con ver qué claves tienen valor y listar los firewalls:

```bash
awk -F': *' '/^oauth2-(clientid|client-secret|user|password)/ {print $1, (length($2) > 2 ? "con valor" : "VACÍO")}' ~/.cntb.yaml
cntb get firewalls
```

Las cuatro claves deben aparecer `con valor`, y el listado debe mostrar los firewalls de la cuenta. Un error `invalid_grant` con `Invalid user credentials` indica usuario o contraseña incorrectos; con el ID o el secreto de cliente incorrectos, el error sería `invalid_client`.

Se identifica el firewall por el nombre que tiene en el panel y se guarda una copia de su estado antes de tocarlo:

```bash
FW_NOMBRE="primero"                          # nombre del firewall en el panel de Contabo
FW_DIR="${HOME}/contabo-firewall"
install -d -m 0700 "${FW_DIR}"
FW_ID="$(cntb get firewalls --name "${FW_NOMBRE}" -o json \
  | python3 -c 'import json, sys; f = json.load(sys.stdin); print(f[0]["firewallId"] if len(f) == 1 else "")')"
echo "FW_ID=${FW_ID}"
cntb get firewalls --firewallId "${FW_ID:?}" -o json > "${FW_DIR}/firewall-$(date +%F-%H%M%S).json"
chmod 600 "${FW_DIR}"/*.json
cntb get firewall-rules "${FW_ID:?}"
```

`FW_ID` debe mostrar un identificador y el listado final, la regla de SSH. Si `FW_ID` sale vacío, el nombre no corresponde a exactamente un firewall, y si falta la regla de SSH, no se continúa. `cntb get firewalls` muestra en `INSTANCES` a qué instancias está asignado.

### 6.3 Servicio web detrás de un borde

Las listas se descargan en la estación con la comprobación de la sección 14.4 del manual de Caddy, y van en una sola regla, `Web-borde`, con las dos familias de direcciones:

```bash
PUERTOS_BORDE="443"                          # "80 443" si el borde también llega por HTTP
rm -f /tmp/rangos-v4.txt /tmp/rangos-v6.txt
curl -fsS "https://<proveedor>/ips-v4" -o /tmp/rangos-v4.txt
curl -fsS "https://<proveedor>/ips-v6" -o /tmp/rangos-v6.txt
for F in /tmp/rangos-v4.txt /tmp/rangos-v6.txt; do
  if [ -s "${F}" ] && ! grep -qvE '^[0-9a-fA-F:.]+/[0-9]{1,3}$' "${F}"; then
    echo "✓ ${F}: $(grep -c . "${F}") rangos"
  else
    echo "✗ ${F}: vacío o con líneas que no son rangos"
  fi
done
```

Las dos líneas deben empezar por `✓`; para Cloudflare, las direcciones son las de la sección 14.4 del manual de Caddy. Luego se construye el conjunto nuevo de reglas a partir del actual, sin aplicarlo todavía:

```bash
cntb get firewalls --firewallId "${FW_ID:?}" -o json | python3 -c '
import json, sys
fw_id, puertos = sys.argv[1], sys.argv[2].split()
v4 = [l.strip() for l in open("/tmp/rangos-v4.txt") if l.strip()]
v6 = [l.strip() for l in open("/tmp/rangos-v6.txt") if l.strip()]
fws = json.load(sys.stdin)
assert v4 and v6, "lista del borde vacía"
assert len(fws) == 1 and fws[0]["firewallId"] == fw_id, "firewall no identificado"
reglas = [r for r in fws[0]["rules"]["inbound"] if r.get("displayName") != "Web-borde"]
nueva = {"action": "accept", "destPorts": puertos, "displayName": "Web-borde", "protocol": "tcp",
         "srcCidr": {"ipv4": v4, "ipv6": v6}, "status": "active"}
pos = next((i for i, r in enumerate(reglas) if r["action"] == "drop"), len(reglas))
reglas.insert(pos, nueva)
print(json.dumps({"inbound": reglas}, indent=2))
' "${FW_ID}" "${PUERTOS_BORDE}" > "${FW_DIR}/reglas-nuevas.json"

cat "${FW_DIR}/reglas-nuevas.json"
```

En el JSON deben aparecer todas las reglas que ya tenía el firewall, la de SSH incluida, más `Web-borde` justo antes de la regla con `"action": "drop"`. Si una comprobación falla, Python termina con un error y el archivo queda vacío o incompleto: no se aplica.

**Una regla de SSH perdida en el reemplazo cierra el acceso al servidor, y a todos los que tengan asignado el mismo firewall.** Se aplica con una sesión SSH abierta en cada uno. La consola VNC del panel sirve como vía de rescate: si está deshabilitada, se habilita antes en el panel, y con root bloqueada se entra con el usuario administrativo.

```bash
cntb update firewall-rules "${FW_ID:?}" --rules="$(cat "${FW_DIR}/reglas-nuevas.json")"
cntb get firewalls --firewallId "${FW_ID:?}" -o json | python3 -c '
import json, sys
f = json.load(sys.stdin)[0]
print(f["name"], f["status"], f["instanceStatus"])
for r in f["rules"]["inbound"]:
    print(r.get("displayName"), r["action"], r["destPorts"], len(r["srcCidr"]["ipv4"]), len(r["srcCidr"]["ipv6"]), r["status"])
'
```

El firewall debe quedar en `active`, cada instancia asignada en `ok`, y `Web-borde` con tantos rangos IPv4 e IPv6 como dieron las comprobaciones, igual que en UFW. Después se abre una sesión SSH nueva, sin cerrar la existente, y se comprueba desde fuera con la sección 11.6.3 del manual de Debian.

Dos comportamientos de `cntb` v1.7 confunden. Con `-f archivo.json`, `update firewall-rules` ignora el archivo y responde `Argument addRulesFirewallRules is empty`, por lo que las reglas se pasan como texto con `--rules`. Y el campo `updatedDate` del firewall no cambia al modificar las reglas, así que no sirve para confirmar el cambio.[^cntb-reglas]

### 6.4 Retiro de lo que sobra

Primero se aplican las reglas restringidas y después se retiran las abiertas, como la del 80 y el 443 cuando ya hay borde o un puerto abierto para una prueba. El retiro sigue el mismo camino que 6.3: se copia la última versión aplicada, se quita del JSON la regla que sobra, se revisa que la de SSH y la de bloqueo sigan presentes, y se aplica con `--rules`.

### 6.5 Mantenimiento

La revisión mensual es el listado de reglas de 6.2. Al terminar cada sesión con `cntb`, las credenciales se borran de la estación, porque dan control sobre todas las instancias de la cuenta; la próxima vez se vuelven a escribir con la línea de los `read`:

```bash
shred -u ~/.cntb.yaml
```

Las copias JSON de `${FW_DIR}` no tienen secretos y pueden conservarse como historial del firewall. Cuando la lista del borde cambia, en la misma sesión que UFW y `trusted_proxies` (sección 17.5 del manual de Caddy), se repite 6.3 completa: la regla `Web-borde` se reconstruye entera con la lista nueva, así que incluye los rangos nuevos y deja fuera los retirados en un solo cambio.

[^cntb-instalacion]: Validado con el archivo `cntb_v1.7_linux_amd64.tar.gz`, descargado y extraído a mano. El bloque completo, con la URL construida a partir de las variables, y el nombre del archivo para otras arquitecturas están pendientes de validación.

[^cntb-reglas]: Validado el 2026-10-18 con `cntb` v1.7: las credenciales con 2FA activo, el entrecomillado del archivo de credenciales, el formato `{"inbound": [...]}` con `--rules`, una regla con los quince rangos IPv4 y los siete IPv6 de Cloudflare, el fallo de `-f` y el `updatedDate` sin cambios. El 2026-11-20 se ejecutaron tal como aparecen aquí los bloques de 6.2 (credenciales escritas por la persona, comprobación de las claves, listado, `FW_ID`, copia y reglas) y de 6.3 la descarga con comprobación y la construcción de `reglas-nuevas.json`, que salió idéntica a la regla vigente; no se aplicó. Siguen pendientes de validación tal como aparecen aquí el `update firewall-rules` y el `chmod 600` de la copia.

## Referencias

- Contabo, [Contabo Firewall Is Here – Free with Every VPS and VDS](https://contabo.com/blog/contabo-firewall/).
- Contabo, [referencia de la API](https://api.contabo.com/).
- Contabo, [publicaciones de `cntb`](https://github.com/contabo/cntb/releases).
