# Azure: grupo de seguridad de red

Complemento del [manual de despliegue de Debian 13 Trixie server](../13-server.md) para servidores en Azure. Aplica la sección 11.6 de ese manual, el firewall del proveedor, al grupo de seguridad de red (NSG) de Azure.

**Autor:** [Calú](https://github.com/calu777)
**Fecha de inicio:** 2026-10-18
**Versión:** v0.1

> Estas órdenes están tomadas de la referencia de la CLI de Azure y no de una corrida completa. Antes de borrar ninguna regla, comprueba con la sección 11.6.3 del manual que las nuevas hacen lo que se espera.

## 1. Reglas de entrada

Puede haber un NSG asociado a la interfaz y otro a la subred, y el tráfico tiene que pasar los dos. Las reglas se evalúan por prioridad, de menor a mayor, y terminan en una regla implícita que deniega lo entrante que no venga de la propia red virtual o del balanceador.

Las órdenes se ejecutan desde la estación de trabajo: una sesión de `az` en el servidor es una credencial sobre toda la suscripción.

```bash
az login
export RG=<grupo-de-recursos>
az network nsg list -g "${RG:?}" -o table

export NSG=<nombre-del-nsg>
az network nsg rule list -g "${RG:?}" --nsg-name "${NSG:?}" -o table
```

Con `--include-default`, el listado añade las reglas implícitas.

### 1.1 SSH

Con la dirección del administrador fija:

```bash
export MIPUERTO=17177                           # definido en 10.1
export MI_IP=$(curl -s https://api.ipify.org)

az network nsg rule create -g "${RG:?}" --nsg-name "${NSG:?}" -n SSH-admin \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes "${MI_IP:?}" --destination-port-ranges "${MIPUERTO:?}"
```

Con una dirección que cambia, se omite `--source-address-prefixes`.

### 1.2 Servicio web publicado directamente

```bash
az network nsg rule create -g "${RG:?}" --nsg-name "${NSG:?}" -n Web-publico \
  --priority 200 --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 80 443
```

### 1.3 Servicio web detrás de un borde

Las listas se descargan en la estación, con la comprobación de la sección 17 del manual de Caddy, y van en una regla por familia de direcciones:

```bash
rm -f /tmp/rangos-v4.txt /tmp/rangos-v6.txt
curl -fsS "https://<proveedor>/ips-v4" -o /tmp/rangos-v4.txt
curl -fsS "https://<proveedor>/ips-v6" -o /tmp/rangos-v6.txt
cat /tmp/rangos-v4.txt; echo; cat /tmp/rangos-v6.txt; echo

az network nsg rule create -g "${RG:?}" --nsg-name "${NSG:?}" -n Web-borde-v4 \
  --priority 200 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes $(cat /tmp/rangos-v4.txt) --destination-port-ranges 443

az network nsg rule create -g "${RG:?}" --nsg-name "${NSG:?}" -n Web-borde-v6 \
  --priority 210 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes $(cat /tmp/rangos-v6.txt) --destination-port-ranges 443
```

Si una descarga falla, `az` rechaza la regla sin valores y no crea nada. La regla de IPv6 no hace daño aunque la máquina no tenga IPv6 pública. Si el borde también llega por HTTP, se añade el puerto 80.

### 1.4 Retiro de lo que sobra

Primero se crean las reglas restringidas y después se borran las abiertas: la del puerto 22 que deja el asistente, la de 80 y 443 abierta cuando ya hay borde, y cualquier puerto abierto para una prueba.

```bash
az network nsg rule list -g "${RG:?}" --nsg-name "${NSG:?}" -o table
az network nsg rule delete -g "${RG:?}" --nsg-name "${NSG:?}" -n <nombre-de-la-regla>
```

### 1.5 Los dos niveles

Si un puerto no responde, o responde cuando no debería, se mira el efecto combinado de los NSG de interfaz y de subred:

```bash
az network nic list -g "${RG:?}" -o table
az network nic list-effective-nsg -g "${RG:?}" -n <nombre-de-la-interfaz>
```

## 2. Mantenimiento

Revisión mensual, junto con la de los rangos del borde:

```bash
az network nsg rule list -g "${RG:?}" --nsg-name "${NSG:?}" -o table
```

Cuando la lista del borde cambia, se actualizan las reglas existentes en la misma sesión que UFW:

```bash
az network nsg rule update -g "${RG:?}" --nsg-name "${NSG:?}" -n Web-borde-v4 \
  --source-address-prefixes $(cat /tmp/rangos-v4.txt)

az network nsg rule update -g "${RG:?}" --nsg-name "${NSG:?}" -n Web-borde-v6 \
  --source-address-prefixes $(cat /tmp/rangos-v6.txt)
```

Cuando cambia la dirección del administrador:

```bash
az network nsg rule update -g "${RG:?}" --nsg-name "${NSG:?}" -n SSH-admin \
  --source-address-prefixes "$(curl -s https://api.ipify.org)"
```

> Si ya no se puede entrar por SSH, esta última orden es la vía de recuperación, porque se lanza contra la API y no contra el servidor.
