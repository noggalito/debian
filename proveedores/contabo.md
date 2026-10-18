# Contabo: lo que deja la imagen de Debian 13

Complemento del [manual de despliegue de Debian 13 Trixie server](../13-server.md) para VPS de Contabo. Aplica la sección 5 de ese manual, la revisión del aprovisionamiento, a lo que se observó el 2026-10-16 en un VPS de Contabo con la imagen de Debian 13: cloud-init 25.1.4, datasource NoCloud desde una ISO `cidata` en `/dev/sr0` y red con netplan sobre `systemd-networkd`. Otras imágenes o fechas pueden diferir.

**Autor:** [Calú](https://github.com/calu777)
**Fecha de inicio:** 2026-10-18
**Versión:** v0.1

## 1. Cuentas y contraseñas

Contabo entrega acceso solo como root, con contraseña, y crea una cuenta `debian` (UID 1000) en el grupo `sudo` con `NOPASSWD:ALL` en `/etc/sudoers.d/90-cloud-init-users`. El usuario administrativo se crea con la sección 4.2.4 del manual.

Las contraseñas de root y de `debian` están en texto claro en `/var/lib/cloud/instance/user-data.txt` y en la ISO `cidata`, legible por el grupo `cdrom`. El panel no permite expulsar la ISO, así que la medida eficaz es cambiar las contraseñas y retirar la cuenta, no borrar archivos:

```bash
sudo passwd root
sudo deluser --remove-home debian
sudo rm /etc/sudoers.d/90-cloud-init-users
sudo visudo -c
getent group sudo
```

`getent group sudo` debe listar solo al usuario administrativo.

## 2. SSH

Un `bootcmd` del user-data escribe `PermitRootLogin yes` en `/etc/ssh/sshd_config`, y la imagen deja `50-cloud-init.conf` con `PasswordAuthentication yes`, que se lee antes que el archivo principal. Resultado: SSH con contraseña abierto a internet, root incluido. En el VPS observado hubo más de 160 intentos fallidos en las primeras horas.

El `01-hardening.conf` de la sección 10.1 anula los dos valores. Si no se va a aplicar enseguida, un drop-in provisional cierra la contraseña; requiere el acceso por llave ya probado (6.3):

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

## 3. Unidades de cloud-init en `failed`

Tras el primer arranque fallan `cloud-init-network.service` y `cloud-init-main.service`, por el `bootcmd`: termina en `pkill -HUP sshd` y se ejecuta antes de que arranque `sshd`, así que `pkill` devuelve 1. Las líneas anteriores, las que tocan `sshd_config`, sí se aplican. La causa se dedujo del script y del código de salida, sin trazar la ejecución. Desactivar cloud-init (5.1) elimina el error.

## 4. Red y ruta estática

La red está en `/etc/netplan/50-cloud-init.yaml`, con IPv4, IPv6 y los DNS de Contabo. La imagen añade `/etc/cron.d/staticroute`, que en cada arranque hace pasar la ruta de la subred por el gateway:

```text
@reboot root ip route replace $(ip route list dev eth0 scope link | head -n1 | awk '{ print $1 }') via $(ip route list dev eth0 | awk '/default/{ print $3 }') dev eth0 &>/dev/null
```

Por eso `ip route` muestra la subred `via` el gateway, igual que la ruta por defecto.

> **ATENCIÓN:** este cron no se borra al limpiar la imagen. Es la única pieza de red fuera de netplan; si la red se migra según 7.5, la ruta pasa a la configuración nueva.

La salida a internet no depende de esta ruta. Lo que cambia es el tráfico hacia la propia subred, que sin el cron se intenta por ARP; eso no se ha comprobado, porque hace falta un segundo equipo en la misma subred. Para probarlo sin reiniciar, con vuelta atrás programada:

```bash
GW="$(ip route show default dev eth0 | awk '{print $3; exit}')"
SUBRED="$(ip route show dev eth0 | awk -v gw="${GW}" '$2 == "via" && $3 == gw && $1 != "default" {print $1; exit}')"
: "${GW:?}" "${SUBRED:?}"
sudo systemd-run --on-active=5min --unit=restaurar-ruta ip route replace "${SUBRED}" via "${GW}" dev eth0
sudo ip route replace "${SUBRED}" dev eth0 scope link
ip route
ping -c3 1.1.1.1
getent hosts deb.debian.org
```

A los cinco minutos, `ip route` vuelve a mostrar la subred `via` el gateway. Si hay un segundo servidor en la subred, se le hace `ping` antes de que venza el temporizador. El gateway no responde a `ping` con ninguna de las dos rutas, así que su silencio no indica nada.[^ruta]

[^ruta]: En el VPS observado, sin la ruta del cron, el `ping` a `1.1.1.1` y la resolución de nombres funcionaron, y el temporizador restauró la ruta a los cinco minutos.

## 5. Otros archivos y paquetes

| Archivo | Contenido | Tratamiento |
|---|---|---|
| `/etc/cron.hourly/free` | `echo 1 > /proc/sys/vm/drop_caches` | Se retira |
| `/etc/cron.hourly/fstrim` | `fstrim /` | Se retira si `fstrim.timer` está activo |
| `/etc/sysctl.d/10-panic.conf` | `kernel.panic = 10` | Se revisa con la sección 13.1 |
| `/etc/cloud/cloud.cfg.d/99-ssh.cfg` | `disable_root: 0`, `ssh_pwauth: 1` | Sin efecto con cloud-init desactivado |
| `/etc/motd` | Banner de Contabo | Cosmético |

`free` vacía cada hora una caché que el kernel ya libera cuando hace falta, y `fstrim` repite lo que hace `fstrim.timer`. Se apartan para poder devolverlos:

```bash
systemctl is-enabled fstrim.timer              # debe responder enabled
sudo install -d -m 0700 /root/proveedor-ori
for f in free fstrim; do
  [ -e "/etc/cron.hourly/${f}" ] && sudo mv "/etc/cron.hourly/${f}" /root/proveedor-ori/
done
sudo run-parts --test /etc/cron.hourly
```

`run-parts --test` ya no debe listarlos, y tras la siguiente ejecución de `cron.hourly` (a los :17) la columna `búf/caché` de `free -m` debe seguir creciendo.

Además, en el primer arranque cloud-init:

- Hace un `full-upgrade`, que puede dejar un kernel pendiente de reinicio (8.4).
- Instala `cron`, que arrastra `exim4` y `bsd-mailx`; se retiran en la sección 21.
- Fija la zona horaria en `Europe/Berlin` (7.3) y un hostname `vmiNNNNNNN.contaboserver.net` (7.1 y 7.2).
- Deja las fuentes de APT en `debian.sources` (8.2).
- Deja `unattended-upgrades` activo con los orígenes de serie, antes de llegar a la sección 16.
