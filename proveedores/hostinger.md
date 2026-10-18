# Hostinger: firewall del VPS y agente Monarx

Complemento del [manual de despliegue de Debian 13 Trixie server](../13-server.md) para VPS de Hostinger. Cubre su firewall, que aplica la sección 11.6 de ese manual, y el agente antimalware que trae la imagen.

**Autor:** [Calú](https://github.com/calu777)
**Fecha de inicio:** 2026-10-18
**Versión:** v0.1

## 1. Firewall del VPS

El firewall es un grupo de reglas que se crea en el panel y se activa sobre el VPS; una máquina tiene un solo grupo activo. Descarta todo lo entrante sin regla y alcanza a los puertos publicados por contenedores.

Dos particularidades condicionan el procedimiento. Las reglas no admiten listas de orígenes, así que hay una regla por rango. Y un cambio no filtra hasta que el grupo se sincroniza.

### 1.1 Grupo y regla de SSH

Se crean desde el panel, en el apartado de seguridad del VPS. La regla de SSH va antes de activar el grupo, o el grupo corta también la administración.

### 1.2 Acceso a la API

Las órdenes se lanzan desde la estación de trabajo. El token se genera en el apartado de API del panel, con la caducidad más corta posible, y se revoca al terminar. Se lee sin eco y llega a `curl` por la entrada estándar, de modo que no queda en el historial ni en la lista de procesos.

```bash
read -rsp "Token de la API de Hostinger: " HAPI; echo
export API=https://developers.hostinger.com/api/vps/v1
hapi() { printf 'Authorization: Bearer %s\n' "${HAPI:?}" | curl -sS -H @- -H 'Content-Type: application/json' "$@"; }
```

```bash
hapi "${API:?}/firewall" | python3 -m json.tool

export FW=<id-del-grupo>
hapi "${API:?}/firewall/${FW:?}" | python3 -m json.tool
```

La primera orden lista los grupos con su `id`; la segunda, las reglas del elegido, entre ellas la de SSH:

```json
{
    "id": 1391065,
    "action": "accept",
    "protocol": "TCP",
    "port": "17177",
    "source": "any",
    "source_detail": "any"
}
```

Si no hay grupos o falta la regla de SSH, no se continúa.

### 1.3 Servicio web detrás de un borde

Las listas se descargan con la comprobación de la sección 17 del manual de Caddy, y se crea una regla por rango:

```bash
rm -f /tmp/rangos-v4.txt /tmp/rangos-v6.txt
curl -fsS "https://<proveedor>/ips-v4" -o /tmp/rangos-v4.txt
curl -fsS "https://<proveedor>/ips-v6" -o /tmp/rangos-v6.txt
{ cat /tmp/rangos-v4.txt; echo; cat /tmp/rangos-v6.txt; echo; } | grep -v '^$' > /tmp/rangos-borde.txt
wc -l < /tmp/rangos-borde.txt

for R in $(cat /tmp/rangos-borde.txt); do
  printf '%-22s ' "${R}"
  hapi -o /dev/null -w '%{http_code}\n' -X POST "${API:?}/firewall/${FW:?}/rules" \
    -d "{\"protocol\":\"TCP\",\"port\":\"443\",\"source\":\"custom\",\"source_detail\":\"${R}\"}"
  sleep 1
done
```

El recuento debe coincidir con los rangos aplicados en UFW, y cada línea debe terminar en `200`. Si el borde también llega por HTTP, el bucle se repite con el puerto 80. Después se sincroniza y se comprueba:

```bash
hapi -X POST "${API:?}/firewall/${FW:?}/sync" | python3 -m json.tool

hapi "${API:?}/firewall/${FW:?}" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('sincronizado:', d.get('is_synced'))
for r in d.get('rules', []):
    print(r['id'], r['action'], r['protocol'], r['port'], r['source_detail'])"

unset HAPI
```

El listado debe mostrar `sincronizado: True`, la regla de SSH y una del 443 por rango. Se revoca el token en el panel y se comprueba que SSH sigue funcionando.

### 1.4 SSH o servicio web con origen abierto

```bash
export MIPUERTO=17177                           # definido en 10.1

hapi -o /dev/null -w '%{http_code}\n' -X POST "${API:?}/firewall/${FW:?}/rules" \
  -d "{\"protocol\":\"TCP\",\"port\":\"${MIPUERTO:?}\",\"source\":\"any\",\"source_detail\":\"any\"}"
```

### 1.5 Retiro de lo que sobra

Primero se crean y sincronizan las reglas restringidas, y después se borran las abiertas por su identificador. El borrado tampoco surte efecto hasta sincronizar:

```bash
hapi -o /dev/null -w '%{http_code}\n' -X DELETE "${API:?}/firewall/${FW:?}/rules/<id-de-la-regla>"
hapi -X POST "${API:?}/firewall/${FW:?}/sync" | python3 -m json.tool
```

> La API puede sustituir todas las reglas del grupo con un `PUT`. Si la lista enviada no incluye SSH, el acceso se pierde en la siguiente sincronización. No se usa aquí.

> Están verificados el listado, el alta por rango y la sincronización. El alta con origen abierto y el borrado salen de la referencia de la API, sin corrida propia.

### 1.6 Mantenimiento

La revisión mensual es el listado de 1.2, que además dice si el grupo quedó sin sincronizar. Cuando la lista del borde cambia, los rangos nuevos se dan de alta con el bucle de 1.3 sobre un archivo que contenga solo esos, se sincroniza, y al final se borran las reglas de los rangos retirados y se sincroniza otra vez.

## 2. Agente Monarx

La imagen de Debian 13 de Hostinger instala durante el aprovisionamiento, antes de que exista el usuario administrativo, el antimalware Monarx: `monarx-agent`, `monarx-protect` y `monarx-protect-autodetect`, desde `https://repository.monarx.com/repository/debian-trixie/`. El agente corre como root, incluye `/usr/bin/monarx-sample-upload` y escribe en `/var/log/monarx-agent.log`.

Es un componente de terceros con privilegios plenos y capacidad de enviar archivos fuera del servidor, así que mantenerlo es una decisión que se toma expresamente en la revisión de la sección 5.1 del manual. Para comprobar si está y de dónde viene:

```bash
dpkg -l 'monarx*' 2>/dev/null | grep '^ii'
apt-cache policy monarx-agent | head -8
cat /etc/apt/sources.list.d/monarx.list
zgrep -h 'monarx' /var/log/dpkg.log* | grep -E ' (install|upgrade) '
```

Su repositorio no figura entre los orígenes de `unattended-upgrades`, pero en el VPS observado el agente se actualizó solo. No se identificó el mecanismo.
