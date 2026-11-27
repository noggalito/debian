# Despliegue de Debian 13 Trixie server

Deja un Debian 13.8 Trixie server listo para operar, con énfasis en el endurecimiento. Es un manual base: agnóstico del proveedor y del rol, con pasos que se pueden repetir sin efectos. Cada rol añade lo suyo en su propio manual, y lo particular de cada proveedor está en [`proveedores/`](proveedores/).

Las subsecciones que empiezan por **Opcional** son recomendables pero no imprescindibles: dependen del entorno (físico o VPS), del rol o de los requisitos de auditoría. El resto es la base que necesita cualquier servidor.

![Debian](<./assets/logo debian.png>)

## Requerimientos previos

- Llave SSH Ed25519 para el punto "6.2 Copia de la llave pública del administrador". Manual para generarla disponible [aquí](https://github.com/noggalito/llave-ssh)
- Una cuenta de correo electrónico (a veces con contraseña por aplicación) desde donde se enviarán las notificaciones del servidor

---

## Índice

### Etapa I — Preparación

#### 1. Descarga y validación de la ISO

- [1.1 Variables de trabajo](#11-variables-de-trabajo)
- [1.2 Importación y verificación de la clave de firmado](#12-importación-y-verificación-de-la-clave-de-firmado)
- [1.3 Descarga de ISO y archivos de verificación](#13-descarga-de-iso-y-archivos-de-verificación)
- [1.4 Verificación de la firma GPG de los checksums](#14-verificación-de-la-firma-gpg-de-los-checksums)
- [1.5 Verificación del hash de la ISO](#15-verificación-del-hash-de-la-iso)

#### 2. Instalación del sistema base

- [2.1 Consideraciones previas al instalador](#21-consideraciones-previas-al-instalador)
- [2.2 Estrategia de particionado](#22-estrategia-de-particionado)
- [2.3 Selección de paquetes durante la instalación](#23-selección-de-paquetes-durante-la-instalación)
- [2.4 Unlock remoto de LUKS (opcional, solo si se usó cifrado)](#24-unlock-remoto-de-luks-opcional-solo-si-se-usó-cifrado)
- [2.5 Primer arranque y verificación básica](#25-primer-arranque-y-verificación-básica)

### Etapa II — Configuración base del sistema

#### 3. Plantillas y variables del manual

- [3.1 Convenciones del sistema de plantillas](#31-convenciones-del-sistema-de-plantillas)
- [3.2 Inventario de plantillas](#32-inventario-de-plantillas)
- [3.3 Resumen de variables del manual](#33-resumen-de-variables-del-manual)
- [3.4 Modificación local de plantillas](#34-modificación-local-de-plantillas)

#### 4. Preparación mínima para administración

- [4.1 Diagnóstico de instalación de `sudo`](#41-diagnóstico-de-instalación-de-sudo)
- [4.2 Instalación de `sudo`](#42-instalación-de-sudo)
- [4.3 Activación de la pertenencia al grupo](#43-activación-de-la-pertenencia-al-grupo)

#### 5. Revisión del aprovisionamiento del proveedor

- [5.1 Revisión tras el primer arranque](#51-revisión-tras-el-primer-arranque)
- [5.2 Reglas heredadas del aprovisionamiento (VPS con cloud-init)](#52-reglas-heredadas-del-aprovisionamiento-vps-con-cloud-init)
- [5.3 Cuentas creadas por la imagen](#53-cuentas-creadas-por-la-imagen)

#### 6. Acceso remoto inicial

- [6.1 Verificación del servidor SSH](#61-verificación-del-servidor-ssh)
- [6.2 Copia de la llave pública del administrador](#62-copia-de-la-llave-pública-del-administrador)
- [6.3 Verificación del acceso por llave](#63-verificación-del-acceso-por-llave)
- [6.4 Mantener el canal de rescate abierto](#64-mantener-el-canal-de-rescate-abierto)

#### 7. Configuración regional y de identidad

- [7.1 Hostname](#71-hostname)
- [7.2 Archivo `/etc/hosts`](#72-archivo-etchosts)
- [7.3 Zona horaria](#73-zona-horaria)
- [7.4 Locales](#74-locales)
- [7.5 Red](#75-red)
- [7.6 `machine-id`](#76-machine-id)

#### 8. Repositorios y primera actualización completa

- [8.1 Respaldo del `sources.list` original](#81-respaldo-del-sourceslist-original)
- [8.2 Edición del `sources.list`](#82-edición-del-sourceslist)
- [8.3 Actualización completa](#83-actualización-completa)
- [8.4 Reinicio si fue actualizado el kernel](#84-reinicio-si-fue-actualizado-el-kernel)

#### 9. Usuarios y privilegios

- [9.1 Verificación del usuario administrativo](#91-verificación-del-usuario-administrativo)
- [9.2 Grupos recomendados para el usuario administrativo](#92-grupos-recomendados-para-el-usuario-administrativo)
- [9.3 Hardening de sudo](#93-hardening-de-sudo)
- [9.4 Política de contraseñas](#94-política-de-contraseñas)
- [9.5 Creación de usuarios adicionales (si aplica)](#95-creación-de-usuarios-adicionales-si-aplica)
- [9.6 Bloqueo de la cuenta de root](#96-bloqueo-de-la-cuenta-de-root)

### Etapa III — Hardening

#### 10. Hardening de SSH

- [10.1 Archivo modular `/etc/ssh/sshd_config.d/01-hardening.conf`](#101-archivo-modular-etcsshsshd_configd01-hardeningconf)
- [10.2 Cambio de puerto, password auth y solo pubkey](#102-cambio-de-puerto-password-auth-y-solo-pubkey)
- [10.3 Restricción de usuarios (`AllowUsers`)](#103-restricción-de-usuarios-allowusers)
- [10.4 Límites de sesión y timeouts](#104-límites-de-sesión-y-timeouts)
- [10.5 Algoritmos criptográficos modernos](#105-algoritmos-criptográficos-modernos)
- [10.6 Validación y recarga segura](#106-validación-y-recarga-segura)

#### 11. Firewall (UFW)

- [11.1 Instalación y políticas por defecto](#111-instalación-y-políticas-por-defecto)
- [11.2 IPv6 (`/etc/default/ufw`)](#112-ipv6-etcdefaultufw)
- [11.3 Apertura selectiva de puertos según rol](#113-apertura-selectiva-de-puertos-según-rol)
- [11.4 Inspección antes de activar](#114-inspección-antes-de-activar)
- [11.5 Activación final y verificación](#115-activación-final-y-verificación)
- [11.6 Firewall del proveedor de nube](#116-firewall-del-proveedor-de-nube)
- [11.7 Baneo permanente de direcciones](#117-baneo-permanente-de-direcciones)

#### 12. Protección contra fuerza bruta (fail2ban)

- [12.1 Instalación](#121-instalación)
- [12.2 Configuración local (`jail.local`)](#122-configuración-local-jaillocal)
- [12.3 Activación](#123-activación)
- [12.4 Monitoreo de bans](#124-monitoreo-de-bans)
- [12.5 Límites del modelo por IP](#125-límites-del-modelo-por-ip)

#### 13. Hardening del kernel y del sistema

- [13.1 Parámetros sysctl de red y de kernel](#131-parámetros-sysctl-de-red-y-de-kernel)
- [13.2 Deshabilitar módulos de kernel innecesarios](#132-deshabilitar-módulos-de-kernel-innecesarios)
- [13.3 Protección de GRUB (solo bare metal)](#133-protección-de-grub-solo-bare-metal)
- [13.4 Servicios que escuchan en la red sin necesidad](#134-servicios-que-escuchan-en-la-red-sin-necesidad)

#### 14. Auditoría y detección

- [14.1 `auditd` — reglas mínimas](#141-auditd--reglas-mínimas)
- [14.2 `debsums` — integridad de paquetes](#142-debsums--integridad-de-paquetes)
- [14.3 AIDE — integridad de archivos del sistema](#143-aide--integridad-de-archivos-del-sistema)
- [14.4 Revisión periódica de logs y registros](#144-revisión-periódica-de-logs-y-registros)
- [14.5 `lynis` — auditoría periódica del baseline de seguridad](#145-lynis--auditoría-periódica-del-baseline-de-seguridad)

#### 15. AppArmor

- [15.1 Verificación del estado](#151-verificación-del-estado)
- [15.2 Perfiles en `enforce` vs `complain`](#152-perfiles-en-enforce-vs-complain)
- [15.3 Perfiles adicionales](#153-perfiles-adicionales)

#### [Cierre de la Etapa III](#cierre-de-la-etapa-iii)

### Etapa IV — Mantenimiento y actualizaciones

#### 16. Actualizaciones automáticas

- [16.1 Instalación](#161-instalación)
- [16.2 Configuración de orígenes](#162-configuración-de-orígenes)
- [16.3 Notificación por correo](#163-notificación-por-correo)
- [16.4 Reinicio automático controlado](#164-reinicio-automático-controlado)
- [16.5 Habilitación del flujo periódico](#165-habilitación-del-flujo-periódico)
- [16.6 Verificación de los timers](#166-verificación-de-los-timers)
- [16.7 Simulación antes de la primera ejecución real](#167-simulación-antes-de-la-primera-ejecución-real)
- [16.8 Revisión de logs tras la primera ejecución](#168-revisión-de-logs-tras-la-primera-ejecución)
- [16.9 Qué hacer después de un point release](#169-qué-hacer-después-de-un-point-release)

#### 17. Sincronización de tiempo

- [17.1 Migración de `systemd-timesyncd` a `chrony`](#171-migración-de-systemd-timesyncd-a-chrony)
- [17.2 Configuración de servidores NTP](#172-configuración-de-servidores-ntp)
- [17.3 Verificación](#173-verificación)
- [17.4 Comportamiento ante desviaciones grandes](#174-comportamiento-ante-desviaciones-grandes)
- [17.5 Verificación cruzada con `timedatectl`](#175-verificación-cruzada-con-timedatectl)

#### 18. Logs y rotación

- [18.1 Configuración de `journald`](#181-configuración-de-journald)
- [18.2 `logrotate` para logs de aplicaciones](#182-logrotate-para-logs-de-aplicaciones)
- [18.3 Verificación de uso de disco](#183-verificación-de-uso-de-disco)
- [18.4 Retención frente a necesidad forense](#184-retención-frente-a-necesidad-forense)

#### 19. Monitoreo básico

- [19.1 Herramientas de consulta puntual](#191-herramientas-de-consulta-puntual)
- [19.2 Alertas mínimas vía cron](#192-alertas-mínimas-vía-cron)
- [19.3 Estado de servicios críticos](#193-estado-de-servicios-críticos)

#### 20. Reenvío de correo del sistema con `msmtp`

- [20.1 Decisiones de diseño](#201-decisiones-de-diseño)
- [20.2 Requisitos previos](#202-requisitos-previos)
- [20.3 Instalación](#203-instalación)
- [20.4 Configuración global de msmtp](#204-configuración-global-de-msmtp)
- [20.5 Contraseña del relay](#205-contraseña-del-relay)
- [20.6 Aliases para destinatarios locales](#206-aliases-para-destinatarios-locales)
- [20.7 Cliente `mail` (`bsd-mailx`)](#207-cliente-mail-bsd-mailx)
- [20.8 Grupo `msmtp` para el usuario administrativo](#208-grupo-msmtp-para-el-usuario-administrativo)
- [20.9 Prueba de envío directo](#209-prueba-de-envío-directo)
- [20.10 Prueba de redirección de aliases](#2010-prueba-de-redirección-de-aliases)
- [20.11 Prueba con el flujo completo del sistema](#2011-prueba-con-el-flujo-completo-del-sistema)
- [20.12 Consideraciones de seguridad](#2012-consideraciones-de-seguridad)
- [20.13 Logs](#2013-logs)

#### 21. Respaldos

- [21.1 Estrategia 3-2-1](#211-estrategia-3-2-1)
- [21.2 Qué respaldar](#212-qué-respaldar)
- [21.3 BorgBackup](#213-borgbackup)
- [21.4 Verificación del repositorio](#214-verificación-del-repositorio)
- [21.5 Prueba de restauración](#215-prueba-de-restauración)
- [21.6 Restauración del servidor desde cero](#216-restauración-del-servidor-desde-cero)

#### 22. Documentación operativa

- [22.1 Inventario del servidor](#221-inventario-del-servidor)
- [22.2 Diagrama lógico](#222-diagrama-lógico)
- [22.3 Runbook básico](#223-runbook-básico)
- [22.4 Mantenimiento de la documentación](#224-mantenimiento-de-la-documentación)

#### [Cierre de la Etapa IV](#cierre-de-la-etapa-iv)

### Anexo A — Diagnóstico rápido post-incidente

- [A.1 Imagen general del sistema](#a1-imagen-general-del-sistema)
- [A.2 Síntoma: el servidor está lento o no responde](#a2-síntoma-el-servidor-está-lento-o-no-responde)
- [A.3 Síntoma: un servicio no funciona](#a3-síntoma-un-servicio-no-funciona)
- [A.4 Síntoma: el disco está lleno](#a4-síntoma-el-disco-está-lleno)
- [A.5 Síntoma: no puedo conectar por SSH](#a5-síntoma-no-puedo-conectar-por-ssh)
- [A.6 Síntoma: red sin conectividad](#a6-síntoma-red-sin-conectividad)
- [A.7 Síntoma: alerta de fail2ban con muchas IPs baneadas](#a7-síntoma-alerta-de-fail2ban-con-muchas-ips-baneadas)
- [A.8 Síntoma: el sistema no arrancó del todo](#a8-síntoma-el-sistema-no-arrancó-del-todo)
- [A.9 Recolección de evidencia para análisis posterior](#a9-recolección-de-evidencia-para-análisis-posterior)
- [A.10 Cuándo escalar](#a10-cuándo-escalar)

### [Referencias](#referencias)

### [Historial de cambios](#historial-de-cambios)

### [To-do](#to-do)

---

## Etapa I — Preparación

### 1. Descarga y validación de la ISO

Solo aplica cuando se instala desde ISO. Si el proveedor entrega Debian preinstalado, se pasa a la sección 2.5. La imagen es la [netinst](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/).

#### 1.1 Variables de trabajo

```bash
DEBIAN_VER="13.8.0"
DEBIAN_ARCH="amd64"
ISO_NAME="debian-${DEBIAN_VER}-${DEBIAN_ARCH}-netinst.iso"
WORKDIR="/tmp/iso-debian"
MIRROR="https://cdimage.debian.org/debian-cd/${DEBIAN_VER}/${DEBIAN_ARCH}/iso-cd"
CD_SIGNING_KEY="DF9B9C49EAA9298432589D76DA87E80D6294BE9B"  # gitleaks:allow

mkdir -p "${WORKDIR}"
cd "${WORKDIR}"
```

Se usa la ruta versionada (`/${DEBIAN_VER}/`) y no `current/`, para que la verificación no cambie si Debian publica otra versión durante el proceso. Si la ruta da 404 porque el point release aún no tiene imágenes, se espera unas horas o se instala con la versión anterior y se sube con `apt full-upgrade` en la sección 8.

#### 1.2 Importación y verificación de la clave de firmado

```bash
gpg --keyserver hkps://keyring.debian.org --recv-keys "${CD_SIGNING_KEY}"
gpg --fingerprint "${CD_SIGNING_KEY}"
```

La huella debe coincidir **exactamente** con la publicada en <https://www.debian.org/CD/verify>:

```text
DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

#### 1.3 Descarga de ISO y archivos de verificación

```bash
wget -c "${MIRROR}/SHA512SUMS"
wget -c "${MIRROR}/SHA512SUMS.sign"
wget -c "${MIRROR}/${ISO_NAME}"
```

`-c` reanuda descargas interrumpidas.

#### 1.4 Verificación de la firma GPG de los checksums

```bash
gpg --verify SHA512SUMS.sign SHA512SUMS
```

Resultado esperado (la fecha varía):

```text
gpg: Firmado el sáb 21 nov 2026 13:13:00 -05
gpg:                usando RSA clave DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Firma correcta de "Debian CD signing key <debian-cd@lists.debian.org>" [desconocido]
gpg: ATENCIÓN: ¡Esta clave no está certificada por una firma de confianza!
gpg:          No hay indicios de que la firma pertenezca al propietario.
Huellas dactilares de la clave primaria: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

El aviso "no está certificada por una firma de confianza" es normal; la huella ya se comprobó en 1.2. Lo que cuenta es "Firma correcta".

#### 1.5 Verificación del hash de la ISO

```bash
sha512sum --ignore-missing -c SHA512SUMS
```

Resultado esperado:

```text
debian-13.8.0-amd64-netinst.iso: La suma coincide
```

`--ignore-missing` evita errores por las demás ISO de `SHA512SUMS` que no se descargaron. Con la ISO verificada, se saca del directorio de trabajo (aquí, al directorio personal) y se borra el resto:

```bash
mv "${WORKDIR}/${ISO_NAME}" ~/
cd
rm -fR "${WORKDIR}"
```

### 2. Instalación del sistema base

Cubre tres escenarios: hardware físico, máquina virtual local y VPS. En un VPS el proveedor suele entregar el sistema instalado, y de esta sección solo aplican 2.2.3 y 2.5. El objetivo es un sistema que arranca, accesible por consola o SSH.

Lo que queda fuera: la IP estática definitiva (sección 7.5), los agentes de cada plataforma (QEMU guest agent, open-vm-tools) y el microcódigo, que llega con los repositorios de la sección 8. Los manuales de rol añaden sus propios ajustes de particionado, recursos y red.

#### 2.1 Consideraciones previas al instalador

##### 2.1.1 Modo de arranque: UEFI vs BIOS legacy

Para despliegues nuevos, UEFI: discos de más de 2 TiB, Secure Boot si hiciera falta y mejor soporte en hipervisores. En VPS lo fija el proveedor. En una VM local se activa en la definición (`virt-manager` → *Customize before install* → *Overview* → *Firmware: UEFI*); en bare metal, UEFI puro, sin CSM.

##### 2.1.2 Medio de instalación

En físico, un pendrive con la ISO verificada, grabado con `dd` (Rufus en modo ISO + partición puede dejar ambigüedades en GPT). En VM, la ISO como CD-ROM virtual. En VPS casi nunca aplica: si el proveedor no admite ISO propia, se parte del sistema entregado y se salta a 2.5.

##### 2.1.3 Recursos mínimos recomendados

| Recurso | Mínimo     | Recomendado base |
|---------|------------|------------------|
| CPU     | 1 vCPU     | 2 vCPU           |
| RAM     | 512 MB     | 1–2 GB           |
| Disco   | 8 GB       | 20 GB            |
| Red     | 1 interfaz | 1 interfaz       |

El dimensionamiento real lo fija el manual del rol.

#### 2.2 Estrategia de particionado

En VPS solo aplica 2.2.3. En instalaciones propias, el particionado se hace a mano o con el guiado revisado, porque es difícil de revertir.

##### 2.2.1 Esquema base recomendado (sin cifrado)

```text
/dev/sda1  →  /boot/efi   (EFI System Partition, 1 GiB, FAT32)
/dev/sda2  →  /boot       (1 GiB, ext4)
/dev/sda3  →  LVM PV
            ├─ vg-sistema/lv-root   →  /      (ext4, 10–20 GiB)
            ├─ vg-sistema/lv-var    →  /var   (ext4, 5–10 GiB)
            ├─ vg-sistema/lv-log    →  /var/log (ext4, 2–5 GiB)
            ├─ vg-sistema/lv-tmp    →  /tmp   (ext4, 2 GiB, opciones nodev,nosuid,noexec)
            ├─ vg-sistema/lv-home   →  /home  (ext4, el resto o según necesidad)
            └─ vg-sistema/lv-swap   →  swap   (igual a RAM hasta 4 GB; 4–8 GB para más)
```

`/boot` va fuera de LVM para que el cargador lea el kernel sin depender de él. `/var`, `/var/log` y `/tmp` separados evitan que un log desbordado o un `/tmp` lleno tumben el sistema, y `nodev,nosuid,noexec` en `/tmp` es endurecimiento gratuito. LVM permite redimensionar, hacer snapshots y ampliar con más discos.

##### 2.2.2 Esquema con cifrado LUKS (recomendado para datos sensibles o servidores físicos)

Cuando el servidor guarda datos sensibles o su ubicación física no está controlada:

```text
/dev/sda1  →  /boot/efi   (ESP, 512 MiB, FAT32, sin cifrar)
/dev/sda2  →  /boot       (1 GiB, ext4, sin cifrar)
/dev/sda3  →  LUKS2 → LVM PV
            └─ vg-sistema/[mismos LVs que arriba]
```

- `/boot` queda sin cifrar. GRUB 2.12 lee LUKS2 con PBKDF2 pero **no** con Argon2id, que es el KDF por defecto del instalador de Debian 13; con `/boot` dentro del contenedor, el sistema no arrancaría. Su contenido (kernels e initramfs) no es sensible.
- La frase de paso se pide en cada arranque; en un servidor remoto se resuelve con 2.4.
- LUKS2 con Argon2id, el valor por defecto, se mantiene.
- El swap va dentro del contenedor, como un LV más. Fuera de él filtra datos de la RAM.

##### 2.2.3 En VPS

Se acepta el particionado del proveedor, normalmente una sola `/`: modificarlo en caliente es arriesgado y rara vez se justifica. Los volúmenes de datos que se añadan sí pueden cifrarse con LUKS.

##### 2.2.4 Sistema de archivos

`ext4`, salvo que el manual del rol pida otro. `btrfs` y `xfs` añaden complejidad que una base común no necesita.

#### 2.3 Selección de paquetes durante la instalación

| Tarea                                | Marcar | Motivo                                                |
|--------------------------------------|--------|-------------------------------------------------------|
| Debian desktop environment           | ❌ No  | Un servidor no necesita GUI                           |
| ...y subvariantes (GNOME, KDE, etc.) | ❌ No  | Ídem                                                  |
| web server                           | ❌ No  | Se instala según rol en manuales específicos          |
| SSH server                           | ✅ Sí  | Acceso remoto desde el primer arranque                |
| standard system utilities            | ✅ Sí  | Herramientas básicas (`less`, `bzip2`, `wget`, etc.)  |

No se marca ningún rol aunque se conozca: `tasksel` instala combinaciones que luego estorban (Apache cuando se quería Nginx). Los roles se instalan en sus manuales.

Si el instalador pregunta, el kernel es el metapaquete `linux-image-amd64`, que sigue siempre a la última versión estable y permite que `unattended-upgrades` (sección 16) y `apt autoremove` gestionen los kernels solos. El initrd, `genérico`, para que el sistema arranque si cambia el hipervisor o el almacenamiento.

En VPS esta pantalla no aparece; los servicios sobrantes se revisan en 2.5 y en la sección 13.4.

#### 2.4 Unlock remoto de LUKS (opcional, solo si se usó cifrado)

**Opcional:** solo con LUKS en un servidor sin consola accesible. Un SSH mínimo dentro del initramfs permite introducir la frase de paso desde otra máquina. Se configura justo después de instalar, antes de cualquier reinicio en remoto. Si `sudo` todavía no está instalado (sección 4), las órdenes del servidor se ejecutan desde `su -`, sin el prefijo `sudo`.

##### 2.4.1 Instalación del paquete

```bash
sudo apt update
sudo apt install -y dropbear-initramfs
```

##### 2.4.2 Configuración de la llave pública autorizada

El SSH del initramfs solo acepta llave. Se usa una llave dedicada al desbloqueo (`~/.ssh/unlock_key` en el equipo del administrador), distinta de la personal y guardada en el gestor de contraseñas. Su parte pública se copia al servidor desde ese equipo:

```bash
IP_SERVIDOR="203.0.113.10"   # IP del servidor
MIUSUARIO=usuario
scp ~/.ssh/unlock_key.pub "${MIUSUARIO:?}@${IP_SERVIDOR:?}:/tmp/unlock_key.pub"
```

Y en el servidor se instala como llave autorizada del initramfs:

```bash
sudo install -d -m 0755 /etc/dropbear/initramfs
sudo install -m 0600 -o root -g root /tmp/unlock_key.pub /etc/dropbear/initramfs/authorized_keys
rm -f /tmp/unlock_key.pub
```

##### 2.4.3 Parámetros de red del initramfs

Con DHCP no hace falta nada. Con IP estática, en `/etc/initramfs-tools/initramfs.conf`:

```text
IP=192.168.1.50::192.168.1.1:255.255.255.0:servidor::off
```

Formato: `IP=cliente:server:gateway:netmask:hostname:interfaz:autoconf`, con `server` vacío.

##### 2.4.4 Puerto SSH del initramfs

Un puerto distinto del 22 evita confusiones en `known_hosts` entre el initramfs y el sistema arrancado:

```bash
# Secuencia robusta: funciona tanto si la línea está comentada
# como si ya existe descomentada o si la directiva no está en el archivo
if grep -q 'DROPBEAR_OPTIONS' /etc/dropbear/initramfs/dropbear.conf; then
    sudo sed -i 's/.*DROPBEAR_OPTIONS=.*/DROPBEAR_OPTIONS="-p 2222"/' \
        /etc/dropbear/initramfs/dropbear.conf
else
    echo 'DROPBEAR_OPTIONS="-p 2222"' \
        | sudo tee -a /etc/dropbear/initramfs/dropbear.conf > /dev/null
fi

# Verificar que la directiva quedó activa
grep 'DROPBEAR_OPTIONS' /etc/dropbear/initramfs/dropbear.conf
# Resultado esperado: DROPBEAR_OPTIONS="-p 2222"
```

##### 2.4.5 Regenerar el initramfs y probar

```bash
sudo update-initramfs -u
```

Se prueba con un reinicio, desde otra máquina:

```bash
ssh -p 2222 -i ~/.ssh/unlock_key root@"${IP_SERVIDOR:?}"
# Al entrar, ejecutar dentro del initramfs:
cryptroot-unlock
# Introducir la frase de paso de LUKS
# La conexión se cierra sola; el sistema continúa arrancando
```

Si el desbloqueo funciona, el servidor termina de arrancar y responde por su SSH normal.

#### 2.5 Primer arranque y verificación básica

Antes del endurecimiento se comprueba que el sistema quedó bien desplegado.

##### 2.5.1 Identificación del sistema

```bash
# Versión y nombre de release
cat /etc/os-release

# Kernel y arquitectura
uname -a

# machine-id (importante en VMs clonadas: debe ser único)
cat /etc/machine-id
```

> **VMs clonadas:** si el servidor viene de una plantilla, se regenera `machine-id` (más en 7.6):
>
> ```bash
> sudo rm /etc/machine-id /var/lib/dbus/machine-id
> sudo systemd-machine-id-setup
> sudo dbus-uuidgen --ensure
> sudo reboot
> ```
>
> Sin ese paso aparecen problemas con logs, concesiones DHCP duplicadas y systemd.

##### 2.5.2 Revisión del arranque

```bash
su -

# Log de arranque
journalctl -b --no-pager | head -100

# Errores y advertencias del arranque actual
journalctl -b -p err --no-pager

# Servicios que fallaron
systemctl --failed
exit
```

`systemctl --failed` debe devolver *"0 loaded units listed"* antes de continuar.

> En imágenes cloud root no tiene contraseña y `su -` no funciona; se usa `sudo -i`, aquí y en 2.5.3.
>
> En VMs sin dispositivo vsock del hipervisor (KVM sin `vhost-vsock`, aunque exista `/dev/vsock`; Hyper-V/Azure y muchos VPS) aparece `systemd-ssh-generator: Failed to query local AF_VSOCK CID` con `failed with exit status 1`. Es inofensivo (Debian #1137216), pero se repite en cada `daemon-reload` y deja `journalctl -p err` siempre sucio. Se enmascara el generador desde la shell de root, o con `sudo` tras la sección 4:
>
> ```bash
> mkdir -p /etc/systemd/system-generators
> ln -s /dev/null /etc/systemd/system-generators/systemd-ssh-generator
> systemctl daemon-reload
> systemctl stop sshd-unix-local.socket
> journalctl -p err --since "2 minutes ago"   # debe devolver "-- No entries --"
> ```
>
> El `stop` detiene el `sshd-unix-local.socket` que el generador ya creó; si sigue cargado, cada `daemon-reload` registra `Socket unit configuration has changed while unit has been running`. El SSH por red, que atiende `ssh.service`, no se ve afectado, y el acceso SSH local por socket AF_UNIX que se pierde no lo usa un servidor remoto. La alternativa por línea de kernel, `systemd.ssh_auto=no`, exige `update-grub` y reinicio.

##### 2.5.3 Estado del disco y particiones

```bash
su -

# Esquema de bloques
lsblk -f

# Espacio en cada punto de montaje
df -hT

# LVM (si aplica)
if lsblk -rno TYPE | grep -qx lvm; then
    pvs && vgs && lvs
else
    echo "Sin volúmenes LVM"
fi

# LUKS (si aplica)
CRYPT_DEVS=$(lsblk -rno NAME,TYPE | awk '$2 == "crypt" {print $1}' | sort -u)
if [ -n "${CRYPT_DEVS}" ]; then
    for dev in ${CRYPT_DEVS}; do cryptsetup status "${dev}"; done
else
    echo "Sin volúmenes LUKS"
fi

exit
```

Las comprobaciones usan el tipo de dispositivo que da `lsblk`. En un VPS lo normal es "Sin volúmenes"; si el esquema de 2.2 usa LVM o LUKS, no puede serlo.

##### 2.5.4 Red funcionando

```bash
# Interfaces e IPs
ip -brief address

# Ruta por defecto
ip route

# Resolución DNS
getent hosts deb.debian.org
```

Si `getent` falla pero hay gateway, se revisa `/etc/resolv.conf` o `systemd-resolved`.

##### 2.5.5 Acceso SSH funcional (desde otra máquina)

```bash
IP_SERVIDOR="203.0.113.10"   # IP del servidor
MIUSUARIO=usuario
ssh "${MIUSUARIO:?}@${IP_SERVIDOR:?}"
```

Todavía con la configuración del instalador (puerto 22, quizá con contraseña). Se endurece en la sección 10.

## Etapa II — Configuración base del sistema

Lleva el sistema de "arranca y es accesible" a "servidor administrable de forma estándar": herramientas, acceso remoto, identidad, repositorios y usuarios. El endurecimiento es la Etapa III.

---

### 3. Plantillas y variables del manual

Las configuraciones largas viven como plantillas en <https://github.com/noggalito/debian/tree/main/assets/13>, se descargan con `wget` y se completan con `sed`. Esta sección reúne cómo se usan, cuáles son y qué variables piden.

#### 3.1 Convenciones del sistema de plantillas

El nombre de cada plantilla es su ruta de destino con guiones en lugar de barras: `plantilla-etc-ssh-sshd_config.d-01-hardening.conf.txt` va a `/etc/ssh/sshd_config.d/01-hardening.conf`. Las variables llevan el prefijo `$` y se sustituyen antes de copiar:

```bash
# Patrón general
export PLANTILLA=plantilla-etc-ejemplo.conf.txt   # nombre en assets/13/
export DESTINO=/etc/ejemplo.conf                  # ruta de destino
export MIVARIABLE=valor

: "${ASSETS_BASE:?}" "${PLANTILLA:?}" "${DESTINO:?}" "${MIVARIABLE:?}" && \
wget "${ASSETS_BASE}/${PLANTILLA}" -O "/tmp/${PLANTILLA}" && \
sed -e "s|\$MIVARIABLE|${MIVARIABLE}|g" "/tmp/${PLANTILLA}" \
    | sudo install -m 0644 -o root -g root /dev/stdin "${DESTINO}"

# Comprobar que no quedó ningún marcador sin sustituir
sudo grep -n '\$MI[A-Z_]*' "${DESTINO}" || echo "✓ plantilla aplicada"
```

Guarda, descarga y despliegue van en una sola orden encadenada con `&&`, porque hay tres formas de dejar un archivo de sistema vacío sin ningún error visible:

- En una shell interactiva, un `${VAR:?}` en una línea aparte solo aborta esa línea; el resto del bloque pegado se ejecuta con la variable vacía.
- Si la guarda está dentro del `sed` de una tubería, el último eslabón (`install` o `tee`) se ejecuta igual y deja el destino con 0 bytes y código 0.
- `wget -O` crea el archivo aunque la descarga falle, y un `sed` o `cp` posterior lo instalaría.

El último eslabón es `install` con los permisos que corresponden al destino, para no depender de la `UMASK` (9.4); las plantillas sin variables lo usan con el archivo descargado como origen. Un heredoc sin comillas (`<<EOF`) es seguro por sí solo: si la expansión falla, `tee` no llega a ejecutarse. La comprobación final detecta marcadores `$MI` que el `sed` no contempla. El repositorio fuerza saltos de línea LF con `.gitattributes`.

##### 3.1.1 Modelo de seguridad: anclaje a commit SHA

La punta de una rama es mutable: si el repositorio se comprometiera, las instalaciones futuras recibirían otro contenido. Por eso las descargas se anclan a un commit (`SHA_COMMIT`). Se define una vez, antes de la primera plantilla:

```bash
# Obtener el SHA del commit que se desea anclar.
# Opción A — si se tiene git instalado localmente:
#   git ls-remote https://github.com/noggalito/debian/ HEAD | awk '{print $1}'
# Opción B — consultar la página de commits en GitHub y copiar el SHA completo (40 caracteres)
#   https://github.com/noggalito/debian/commits/main

export SHA_COMMIT="<REEMPLAZAR-POR-SHA-DE-40-CARACTERES>"
export ASSETS_BASE="https://raw.githubusercontent.com/noggalito/debian/${SHA_COMMIT}/assets/13"

# Verificar que la variable quedó definida
echo "SHA_COMMIT = ${SHA_COMMIT}"
echo "ASSETS_BASE = ${ASSETS_BASE}"
```

#### 3.2 Inventario de plantillas

| Plantilla                                               | Versión | Sección | Ruta destino                               | Variables                                                                              |
|---------------------------------------------------------|---------|---------|--------------------------------------------|----------------------------------------------------------------------------------------|
| `plantilla-etc-hosts.txt`                               | v1.1    | 7.2     | `/etc/hosts`                               | `$MIHOST`, `$MIFQDN`                                                                   |
| `plantilla-etc-ssh-sshd_config.d-01-hardening.conf.txt` | v1.4    | 10.1    | `/etc/ssh/sshd_config.d/01-hardening.conf` | `$MIPUERTO`, `$MIUSUARIO`                                                              |
| `plantilla-etc-fail2ban-jail.local.txt`                 | v1.0    | 12.2    | `/etc/fail2ban/jail.local`                 | `$MIPUERTO`                                                                            |
| `plantilla-etc-sysctl.d-99-hardening.conf.txt`          | v1.2    | 13.1    | `/etc/sysctl.d/99-hardening.conf`          | (sin variables)                                                                        |
| `plantilla-etc-modprobe.d-99-hardening.conf.txt`        | v1.2    | 13.2    | `/etc/modprobe.d/99-hardening.conf`        | (sin variables)                                                                        |
| `plantilla-etc-audit-rules.d-99-hardening.rules.txt`    | v1.1    | 14.1    | `/etc/audit/rules.d/99-hardening.rules`    | (sin variables)                                                                        |
| `plantilla-etc-cron.hourly-server-check.txt`            | v1.1    | 19.2    | `/etc/cron.hourly/server-check`            | (sin variables)                                                                        |
| `plantilla-etc-msmtprc.txt`                             | v1.3    | 20.4    | `/etc/msmtprc`                             | `$MIRELAY`, `$MIPUERTO_SMTP`, `$MIUSUARIO_SMTP`, `$MIFROM`                             |

#### 3.3 Resumen de variables del manual

| Variable          | Significado                                                       | Ejemplo                                       |
|-------------------|-------------------------------------------------------------------|-----------------------------------------------|
| `$MIHOST`         | Hostname corto del servidor                                       | `villonaco`                                   |
| `$MIFQDN`         | Nombre completo / FQDN del servidor                               | `villonaco.ejemplo.com`                       |
| `$MIUSUARIO`      | Usuario administrativo                                            | `usuario`                                     |
| `$MIPUERTO`       | Puerto SSH personalizado (también usado por fail2ban)             | `17177`                                       |
| `$MIRELAY`        | Hostname del relay SMTP saliente                                  | `smtp.gmail.com`                              |
| `$MIPUERTO_SMTP`  | Puerto SMTP del relay (sección 20 — distinto de `$MIPUERTO`)      | `587`                                         |
| `$MIUSUARIO_SMTP` | Usuario para autenticarse contra el relay                         | `cuenta@dominio.com`                          |
| `$MIFROM`         | Dirección "From" autorizada en el relay                           | `cuenta@dominio.com`                          |
| `$MIDESTINO`      | Dirección externa donde llega el correo del sistema               | `admin@dominio.com`                           |
| `$MIGRUPO`        | Grupo de activos: prefijo de sus respaldos Borg (21.1)            | `s.o`                                         |
| `$MIZONA`         | Zona horaria del servidor (7.3)                                   | `America/Lima`                                |
| `$MIIFAZ`         | Interfaz de la ruta por defecto (7.5.2)                           | `enp1s0`                                      |
| `$IP_SERVIDOR`    | IP del servidor, en el equipo del administrador (2.5.5, 10.2)     | `203.0.113.10`                                |
| `$MINOMBREDELLAVE`| Nombre de la llave SSH del administrador (6.2)                    | `id_ed25519`                                  |
| `$SHA_COMMIT`     | SHA del commit del repositorio de plantillas anclado              | `a1b2c3d4...` (40 chars)                      |
| `$ASSETS_BASE`    | URL base para descarga de plantillas (derivada de `SHA_COMMIT`)   | `https://raw.githubusercontent.com/.../assets`|

Los valores usados en cada servidor se anotan en su inventario (22.1).

#### 3.4 Modificación local de plantillas

Si hace falta un cambio que ninguna variable cubre, hay dos caminos. Un fork del repositorio, cuando el cambio es sostenido y vale para varios servidores; la descarga apunta entonces al fork. O editar el destino tras desplegarlo, cuando el cambio es de un solo servidor, anotándolo en el runbook (22.3). Las plantillas del repositorio original no se tocan sin coordinación, porque las usan todos los despliegues.

### 4. Preparación mínima para administración

Una instalación `netinst` con contraseña de root no trae `sudo`: Debian asume administración con `su -`. El manual entero usa `sudo`, así que se instala primero. En VPS y plantillas suele venir ya, y la sección se reduce a verificarlo.

#### 4.1 Diagnóstico de instalación de `sudo`

Desde la sesión del usuario administrativo:

```bash
sudo -V 2>/dev/null | head -1
```

| Resultado                                           | Estado                 | Acción              |
|-----------------------------------------------------|------------------------|---------------------|
| `Sudo version 1.9.x ...`                            | Instalado y funcional  | Saltar a 4.3        |
| `-bash: sudo: orden no encontrada`                  | No instalado           | Continuar con 4.2   |
| Instalado pero `usuario is not in the sudoers file` | Instalado sin permisos | Ir a 4.2.2          |
| Instalado, pero el acceso es solo como root         | Sin usuario propio     | Ir a 4.2.4          |

#### 4.2 Instalación de `sudo`

##### 4.2.1 Elevar privilegios temporalmente

```bash
su -
```

`su -` con guion carga el entorno completo de root; sin guion conserva parte del entorno del usuario y puede dar sorpresas.

##### 4.2.2 Instalar el paquete y añadir el usuario al grupo `sudo`

```bash
export MIUSUARIO=usuario
apt update
apt install -y sudo
usermod -aG sudo "${MIUSUARIO:?}"
```

En Debian, el grupo `sudo` da permisos completos porque `/etc/sudoers` incluye:

```text
%sudo   ALL=(ALL:ALL) ALL
```

`/etc/sudoers` no se edita; el endurecimiento de `sudo` es la sección 9.3.

##### 4.2.3 Salir de la shell de root

```bash
exit
```

##### 4.2.4 Crear el usuario administrativo (VPS que solo entregan root)

Algunos proveedores entregan el VPS solo con root y contraseña. Ahí `sudo` ya está y 4.1 parece correcto, pero solo porque lo ejecuta root, que no necesita el grupo. El manual asume la cuenta propia desde la sección 6, así que se crea desde la sesión de root:

```bash
export MIUSUARIO=usuario
adduser "${MIUSUARIO:?}"
usermod -aG sudo "${MIUSUARIO:?}"
```

`adduser` pide una contraseña y hay que darla, aunque el acceso vaya a ser por llave: es la que `sudo` pedirá tras la sección 9.3. Se revisa quién más está en el grupo:

```bash
getent group sudo
```

Las cuentas que haya creado la imagen se tratan en la sección 5. La sesión de root queda abierta como canal de rescate hasta que pase la verificación de 4.3, que se hace desde una conexión SSH nueva con el usuario recién creado.

#### 4.3 Activación de la pertenencia al grupo

Los grupos nuevos no se aplican a sesiones abiertas; hay que reconectar:

```bash
exit
# Reconectar por SSH o consola
```

Tras reconectar:

```bash
id
# La salida debe incluir ... grupos=...,27(sudo),...

sudo -v
# Pide la contraseña del usuario (no la de root). Si no devuelve error, sudo está operativo.
```

La verificación solo vale con la cuenta del usuario administrativo. Como root, `id` da `uid=0(root)` y `sudo -v` pasa sin pedir nada, sin comprobar nada.

---

### 5. Revisión del aprovisionamiento del proveedor

Un VPS arranca desde una imagen que el proveedor personaliza en el primer arranque, casi siempre con cloud-init: crea cuentas, asigna contraseñas, cambia SSH, instala paquetes y deja archivos en `/etc`. Nada de eso se ve en el panel, y parte contradice la Etapa III. Esta revisión se hace tras 2.5 y antes de la Etapa III. Lo encontrado en cada proveedor está en [`proveedores/`](proveedores/).

#### 5.1 Revisión tras el primer arranque

```bash
sudo cloud-init status --long
sudo cat /var/lib/cloud/instance/user-data.txt
```

> **ATENCIÓN:** el user-data puede contener contraseñas en texto claro (`chpasswd`, `password`). No se copia a tickets, chats ni documentación sin borrarlas. Si las contiene, se consideran expuestas y se cambian ahora, porque suelen seguir disponibles en el medio de arranque del proveedor.

Se revisan las cuentas con privilegios, las reglas de sudo heredadas, la configuración efectiva de SSH y los archivos que escribió la imagen (`write_files`):

```bash
getent group sudo
sudo ls -la /etc/sudoers.d/ /etc/ssh/sshd_config.d/
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication)'
sudo grep -E '^\s+path:' /var/lib/cloud/instance/user-data.txt
```

El grupo `sudo` debe contener solo al usuario administrativo; una cuenta del proveedor que no se use se elimina (5.3) junto con su regla de `/etc/sudoers.d/` (5.2). Si `sshd -T` muestra `passwordauthentication yes` o `permitrootlogin yes`, el servidor acepta contraseñas desde internet y conviene adelantar la sección 10. Se revisan también los agentes de terceros que traiga la imagen, como el antimalware de Hostinger; mantenerlos es una decisión expresa.

Una vez completado el primer arranque, cloud-init no aporta nada a un servidor administrado a mano, y se recomienda desactivarlo aquí, antes de la Etapa III. No todo lo que trae el user-data corre una sola vez: `bootcmd` y los módulos de frecuencia `always` se ejecutan en cada arranque. Mientras cloud-init siga activo, un reinicio puede deshacer en silencio lo que endurece este manual. Por ejemplo, el `bootcmd` de Contabo vuelve a escribir `PermitRootLogin yes` en `sshd_config` en cada arranque ([`proveedores/contabo.md`](proveedores/contabo.md), secciones 2 y 3). La frecuencia de cada módulo se ve en el log:

```bash
sudo grep -E "Running module .* with frequency always" /var/log/cloud-init.log | sed -E 's/.*Running module ([^ ]+).*/\1/' | sort -u
```

Si en la lista aparece `bootcmd`, se lee su contenido en el user-data de 5.1 para saber qué repetiría cada arranque.

Antes de desactivarlo se comprueba que la red vive en `/etc` y no se genera en cada arranque, como ocurre con un `/etc/netplan/50-cloud-init.yaml` completo (2.5.4). Si la red depende del datasource (DHCP con metadatos del proveedor, por ejemplo), primero se fija según 7.5. Luego:

```bash
sudo touch /etc/cloud/cloud-init.disabled
```

Tras el siguiente reinicio, `cloud-init status` debe dar `status: disabled` y la red debe seguir igual (2.5.4).

Con cloud-init desactivado, el user-data ya no hace falta, pero la contraseña inicial sigue copiada en varios archivos de `/var/lib/cloud/instances/<id>/`. Se localizan comparando el valor, sin mostrarlo, y se borran solo los archivos que lo contienen:

```bash
sudo bash -c '
D=$(readlink -f /var/lib/cloud/instance)
V=$(grep -oP "^\s*(-\s*)?(root|password)\s*:\s*[\"\x27]?\K[^\"\x27\s]{6,}" "$D/user-data.txt" | sort -u)
for f in $(find /var/lib/cloud /var/log/cloud-init*.log -type f 2>/dev/null); do
  n=0; for v in $V; do grep -qF -- "$v" "$f" && n=$((n+1)); done; [ "$n" -gt 0 ] && echo "$n $f"
done; true'
```

Solo se listan los archivos que contienen la contraseña. En un VPS de Contabo fueron `user-data.txt`, `user-data.txt.i`, `cloud-config.txt` y `obj.pkl`; los logs de cloud-init no la tenían. Se borran con `sudo rm -f` uno a uno. El medio de arranque del proveedor (por ejemplo, la ISO `cidata` en `/dev/sr0`) la conserva y no se puede modificar desde el servidor.

#### 5.2 Reglas heredadas del aprovisionamiento (VPS con cloud-init)

Las imágenes cloud suelen dejar `/etc/sudoers.d/90-cloud-init-users` con una regla `NOPASSWD`. Importa a quién beneficia:

```text
# User rules for <usuario>
<usuario> ALL=(ALL) NOPASSWD:ALL
```

Si beneficia al usuario administrativo, es grave. Las reglas se suman y, como `@includedir` va al final de `/etc/sudoers`, es la que sudo aplica: ningún comando pide contraseña, quien tenga la llave SSH obtiene root, y ni `pam_faillock` (9.3.3) ni `mail_badpass` (9.3.1) actúan nunca. Si beneficia a root, no tiene efecto práctico, pero se retira igual: lynis y las guías CIS la marcan.

Detección, desde la sesión del usuario administrativo:

```bash
ls -l /etc/sudoers.d/
sudo grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/
sudo -k
sudo true                      # si no pide contraseña, NOPASSWD está activo
sudo -l | grep -i nopasswd     # muestra "(ALL) NOPASSWD: ALL" si está activo
```

> **ATENCIÓN:** estas comprobaciones solo valen desde la sesión del usuario administrativo. Como root no detectan la regla, exista o no el archivo: `sudo true` nunca pide contraseña, y `sudo -l` muestra la regla `root ALL=(ALL:ALL) ALL` de `/etc/sudoers`, sin `NOPASSWD`. Y `sudo -v` tampoco sirve: con `verifypw=all` pide contraseña si alguna regla la exige, y `%sudo` la exige.

Antes de retirarla, el usuario debe tener contraseña o se quedará sin sudo:

```bash
sudo passwd -S "${MIUSUARIO:?}"
```

La segunda columna debe ser `P` (`usuario P 2026-09-26 1 90 14 -1`). Con `L` o `NP`, primero `sudo passwd "${MIUSUARIO:?}"`, y se confirma el grupo (9.1). Se lee el archivo completo, porque puede tener más de una regla:

```bash
sudo cat /etc/sudoers.d/90-cloud-init-users
```

Con una segunda sesión SSH abierta (6.4):

```bash
sudo rm /etc/sudoers.d/90-cloud-init-users
sudo visudo -c
sudo -k
sudo true      # ahora debe pedir contraseña
```

`visudo -c` va siempre después de tocar `/etc/sudoers.d/`: un error de sintaxis inhabilita sudo por completo y solo se repara por consola. La sesión de respaldo se cierra cuando `sudo true` pide la contraseña y la acepta.

El archivo no reaparece en reinicios normales, porque el módulo de usuarios de cloud-init corre una vez por instancia; sí al redesplegar o capturar la imagen. AIDE (14.3) lo detectaría en `/etc`. Desactivar cloud-init (5.1) elimina el riesgo.

Sin `NOPASSWD` se activan dos mecanismos dormidos. `pam_faillock` pasa a aplicarse a sudo: con `silent`, un bloqueo parece una contraseña incorrecta y dura `unlock_time` (10 minutos), sin afectar al acceso SSH por llave. Y la contraseña del usuario, que vence a los 90 días según 9.4, se convierte en la llave de sudo; conviene anotar su vencimiento con `sudo chage -l "${MIUSUARIO:?}"`.

#### 5.3 Cuentas creadas por la imagen

Además de root, algunas imágenes crean una cuenta propia del proveedor. Contabo, por ejemplo, crea `debian` (UID 1000) en los grupos `sudo` y `adm`, con `authorized_keys` vacío y la regla `NOPASSWD:ALL` de 5.2 ([`proveedores/contabo.md`](proveedores/contabo.md), sección 1). El manual no la usa: administra con la cuenta de 4.2.4. Una cuenta con sudo que nadie vigila es una puerta más, y si su contraseña viajó en el user-data, hay que darla por expuesta. Se elimina después de verificar el usuario administrativo (4.3), nunca antes.

Se identifican las cuentas humanas y los grupos con privilegios:

```bash
getent passwd | awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $7}'
getent group sudo adm
```

Solo deben aparecer el usuario administrativo y la cuenta del proveedor. Antes de borrar se comprueba que la cuenta no tenga sesiones, procesos, tareas ni archivos fuera de su directorio personal:

```bash
export CUENTA_PROV=debian
who | grep -w "${CUENTA_PROV:?}"
pgrep -l -u "${CUENTA_PROV:?}"
sudo crontab -l -u "${CUENTA_PROV:?}"
sudo find / -xdev -user "${CUENTA_PROV:?}" -not -path "/home/${CUENTA_PROV:?}/*" 2>/dev/null
```

En una imagen recién entregada, `who` y `pgrep` no devuelven nada, `crontab` responde `no crontab for debian` y `find` solo lista el propio `/home/debian`. Cualquier otro resultado se revisa antes de seguir. Después se elimina la cuenta con su directorio personal:

```bash
sudo deluser --remove-home "${CUENTA_PROV:?}"
getent group "${CUENTA_PROV:?}" && sudo delgroup --only-if-empty "${CUENTA_PROV:?}"
```

En Debian 13, `deluser` borra también el grupo personal si se queda sin miembros y no imprime nada; la segunda línea cubre las versiones que lo dejan y no hace nada si el grupo ya no existe. Si la regla de `/etc/sudoers.d/` de esta cuenta sigue ahí, se retira con el procedimiento de 5.2, que termina con `visudo -c`. Verificación:

```bash
getent passwd "${CUENTA_PROV:?}" || echo "cuenta eliminada"
getent group sudo
sudo grep -rn "${CUENTA_PROV:?}" /etc/sudoers /etc/sudoers.d/ || echo "sin reglas"
```

Debe responder `cuenta eliminada` y `sin reglas`, y el grupo `sudo` debe contener solo al usuario administrativo. El módulo de usuarios de cloud-init corre una vez por instancia, así que la cuenta no vuelve en los reinicios; solo volvería al redesplegar la imagen, y desactivar cloud-init (5.1) elimina ese riesgo.[^cuenta-prov]

[^cuenta-prov]: Validado en un VPS de Contabo el 2026-11-19, eliminando la cuenta `debian` de la imagen.

### 6. Acceso remoto inicial

Deja el acceso por llave pública funcionando, todavía con la configuración por defecto de SSH. En un VPS donde ya hay conexión, se reduce a formalizar la llave antes de desactivar las contraseñas en la sección 10.

#### 6.1 Verificación del servidor SSH

```bash
sudo systemctl status ssh --no-pager
```

Debe estar `active (running)`. Si no se instaló:

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

#### 6.2 Copia de la llave pública del administrador

Hace falta una llave Ed25519 ([manual](https://github.com/noggalito/llave-ssh)); `$MINOMBREDELLAVE` se define ahí o aquí. Desde el equipo del administrador, con las variables de 2.5.5:

```bash
ssh-copy-id -i ~/.ssh/"${MINOMBREDELLAVE:?}".pub "${MIUSUARIO:?}@${IP_SERVIDOR:?}"
```

Sin `ssh-copy-id`:

```bash
cat ~/.ssh/"${MINOMBREDELLAVE:?}".pub | ssh "${MIUSUARIO:?}@${IP_SERVIDOR:?}" '
  umask 077 &&
  mkdir -p ~/.ssh &&
  cat >> ~/.ssh/authorized_keys &&
  chmod 700 ~/.ssh &&
  chmod 600 ~/.ssh/authorized_keys
'
```

O añadiendo a mano `${MINOMBREDELLAVE}.pub` a `~/.ssh/authorized_keys` del servidor.

#### 6.3 Verificación del acceso por llave

```bash
ssh "${MIUSUARIO:?}@${IP_SERVIDOR:?}"
```

Debe entrar sin contraseña. Si la pide, la llave no se copió o los permisos están mal:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Permisos esperados:

```text
drwx------  .ssh
-rw-------  authorized_keys
```

#### 6.4 Mantener el canal de rescate abierto

Durante el resto del manual, la sesión que funciona no se cierra y los cambios se prueban desde una segunda. Si algo rompe SSH, la primera permite repararlo. `tmux` además sobrevive a un corte:

```bash
# Instalación de tmux en el servidor
sudo apt update
sudo apt install -y tmux

# Iniciar la sesión de tmux
tmux new -A -s admin
```

Tras un corte, se reconecta por SSH y se vuelve a la sesión:

```bash
tmux attach -t admin
```

Para salir de `tmux` sin cerrarlo: `Ctrl+b` y luego `d`.

### 7. Configuración regional y de identidad

#### 7.1 Hostname

```bash
sudo hostnamectl
```

Para fijarlo:

```bash
export MIHOST=villonaco
sudo hostnamectl set-hostname "${MIHOST:?}"
```

`hostnamectl` actualiza `/etc/hostname` en caliente, sin reiniciar.

#### 7.2 Archivo `/etc/hosts`

El hostname debe resolverse localmente, sin pasar por DNS:

```bash
# Respaldar el archivo original (solo la primera vez)
[ -e /etc/hosts.ori ] || sudo cp /etc/hosts /etc/hosts.ori

# Definir las variables $MIHOST y $MIFQDN
export MIHOST=villonaco
export MIFQDN=villonaco.ejemplo.com

# Descargar y aplicar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" "${MIHOST:?}" "${MIFQDN:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-hosts.txt" -O /tmp/plantilla-etc-hosts.txt && \
sed -e "s|\$MIFQDN|${MIFQDN}|g" -e "s|\$MIHOST|${MIHOST}|g" \
    /tmp/plantilla-etc-hosts.txt | sudo install -m 0644 -o root -g root /dev/stdin /etc/hosts

# Comprobar que no quedó ningún marcador sin sustituir
grep -n '\$MI[A-Z_]*' /etc/hosts || echo "✓ plantilla aplicada"
```

El aviso `sudo: unable to resolve host villonaco` que puede salir durante el cambio se ignora. Sin dominio, se usa `<hostname>.lan` o solo `127.0.1.1 <hostname>`.

```bash
hostname -f     # FQDN completo
hostname -s     # nombre corto
getent ahostsv4 "${MIHOST:?}"
```

`getent ahostsv4` debe mostrar `127.0.1.1`. `getent hosts` no sirve para esta comprobación: pregunta primero por IPv6 y, como `/etc/hosts` solo tiene la entrada IPv4 del nombre, responde `nss-myhostname` con las direcciones IPv6 de la interfaz.

#### 7.3 Zona horaria

```bash
export MIZONA=America/Lima                         # zona del servidor
timedatectl                                        # estado actual
timedatectl list-timezones | grep "^${MIZONA%%/*}/"   # zonas de la misma región
sudo timedatectl set-timezone "${MIZONA:?}"
```

```bash
date
```

La sincronización horaria se explica en la sección 17.

#### 7.4 Locales

En un servidor conviene `C.UTF-8` o `en_US.UTF-8` por defecto, con logs en inglés que se buscan fácil, y el locale regional como adicional.

```bash
sudo apt install -y locales
sudo dpkg-reconfigure locales
```

En el diálogo se marcan `en_US.UTF-8 UTF-8` y el regional (por ejemplo `es_PE.UTF-8 UTF-8`), y se elige el de defecto. Para dejarlo todo en el idioma regional:

```bash
sudo update-locale --reset LANG=es_PE.UTF-8
```

Basta con `LANG`. `LC_ALL` no se define en `/etc/default/locale`: tiene prioridad sobre todo e impide cambiar una categoría puntual, como `LC_MESSAGES=C` para buscar un error en inglés. `--reset` borra además un `LC_ALL` heredado.

```bash
locale
locale -a | grep -E 'es_PE|en_US'
```

El cambio se aplica al reabrir la sesión; hasta entonces `locale` sigue mostrando el valor anterior. SSH acepta además el `LANG` del cliente (`AcceptEnv LANG LC_*` en el `sshd_config` de Debian): si la estación usa un locale que el servidor no generó, la sesión muestra avisos de `setlocale`, así que conviene generar también ese.

#### 7.5 Red

La netinst deja la red en `ifupdown` (`/etc/network/interfaces`), sin `dhcpcd`, `NetworkManager` ni `systemd-networkd`. En VPS varía: muchos usan `ifupdown` con configuración estática y las imágenes con cloud-init suelen traer `systemd-networkd` o netplan. El objetivo es siempre una IP estable, gateway, DNS y persistencia. Si la red ya funciona como debe, de esta sección solo aplican 7.5.1, 7.5.2, 7.5.7 y 7.5.8; las opciones A, B y C son **opcionales**.

Con netplan (`/etc/netplan/*.yaml`, habitual en imágenes con cloud-init), las opciones A, B y C no aplican: la configuración se cambia en ese archivo y se aplica con `sudo netplan try`, que la revierte si no se confirma a tiempo. Este caso está pendiente de validación.

##### 7.5.1 Estado actual de la red

```bash
# Estado de IPs y rutas
ip -brief address
ip route

# Resolución DNS
cat /etc/resolv.conf
getent hosts deb.debian.org

# Qué paquetes de red están instalados
dpkg -l | grep -E '^ii  (ifupdown|systemd-networkd|netplan|network-manager|dhcpcd)' | awk '{print $2}'

# Qué servicios de red están activos
systemctl is-active networking systemd-networkd NetworkManager 2>/dev/null
```

- `networking` activo con `ifupdown`: configuración en `/etc/network/interfaces`, el caso de la netinst.
- `systemd-networkd` activo: configuración en `/etc/systemd/network/*.network`, típico en imágenes cloud.
- `NetworkManager` activo: raro en servidor.

Con IP del rango esperado, gateway y `getent hosts` resolviendo, solo queda decidir si la IP se fija.

##### 7.5.2 Identificar el nombre de la interfaz

Debian 13 usa nombres predecibles (`enp1s0`, `eno1`, `ens3`):

```bash
ip -brief address | grep -v '^lo'
```

Salida típica:

```text
enp1s0           UP             192.168.1.50/24 fe80::5054:ff:fe12:3456/64
```

Donde aparece `<interfaz>`, va el nombre real. Para guardarlo en una variable:

```bash
export MIIFAZ=$(ip -o -4 route show default | awk '{print $5}')
echo "Interfaz por defecto: $MIIFAZ"
```

Toma la interfaz de la ruta por defecto, la correcta en un servidor con una sola tarjeta.

##### 7.5.3 Opción A — DHCP con reserva por MAC (lo más simple)

Si el router permite reservar la IP por MAC, no se toca nada en el servidor:

1. Anotar la MAC: `ip link show $MIIFAZ` (línea `link/ether ...`).
2. Crear la reserva en el router.
3. Renovar la concesión:

   ```bash
   sudo systemctl restart networking
   ```

4. Verificar la IP:

   ```bash
   ip -brief address show $MIIFAZ
   ```

Depende del router y no sirve en redes sin control del DHCP.

##### 7.5.4 Opción B — IP estática con `ifupdown` (configuración por defecto)

La opción para IP estática manteniendo el backend de la netinst.

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.ori
```

```bash
cat /etc/network/interfaces
```

Salida típica con DHCP:

```text
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp1s0
iface enp1s0 inet dhcp
```

Se edita el archivo:

```bash
sudoedit /etc/network/interfaces
```

Y el bloque `iface ... inet dhcp` se sustituye por:

```text
# The primary network interface
allow-hotplug enp1s0
iface enp1s0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1 1.1.1.1
```

`address` admite CIDR. `dns-nameservers` necesita `resolvconf` para llegar a `/etc/resolv.conf` (`dpkg -l resolvconf`); sin él, se instala o se usa 7.5.6. Para IPv6 estática se añade un bloque `iface enp1s0 inet6 static`.

Lo más seguro es aplicar con `sudo reboot`, que además prueba la persistencia. En caliente:

```bash
sudo ifdown $MIIFAZ && sudo ifup $MIIFAZ
```

> **Atención:** `ifdown` por SSH sobre la propia interfaz corta la conexión unos segundos, y si la configuración nueva falla solo queda la consola. Por eso el `&&`, y mejor dentro de `tmux`.

```bash
ip -brief address show $MIIFAZ
ip route
cat /etc/resolv.conf
```

##### 7.5.5 Opción C — Migrar a `systemd-networkd` (solo si se justifica)

Solo aporta con bonding, VLAN, bridges o varias interfaces con políticas de routing; con una interfaz simple no mejora a `ifupdown`.

**1. Crear la configuración antes de desactivar `ifupdown`**, para no quedarse sin red:

```bash
sudo install -m 0644 -o root -g root /dev/stdin "/etc/systemd/network/10-${MIIFAZ}.network" <<EOF
[Match]
Name=${MIIFAZ}

[Network]
Address=192.168.1.50/24
Gateway=192.168.1.1
DNS=192.168.1.1
DNS=1.1.1.1
EOF
```

Para DHCP, el bloque `[Network]` es:

```text
[Network]
DHCP=yes
```

**2. Habilitar `systemd-networkd` y `systemd-resolved`:**

```bash
sudo apt install -y systemd-resolved
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
```

En Trixie `systemd-resolved` es un paquete aparte y su instalación convierte `/etc/resolv.conf` en un enlace al stub.

**3. Comentar en `/etc/network/interfaces`** las líneas `allow-hotplug` e `iface` de la interfaz, sin tocar `lo`:

```bash
sudoedit /etc/network/interfaces
```

**4. Reiniciar**, porque las dos pilas en caliente pueden pelear por la interfaz:

```bash
sudo reboot
```

**5. Verificar:**

```bash
networkctl status
networkctl status $MIIFAZ
resolvectl status
ip -brief address
ip route
```

`networkctl status <interfaz>` debe mostrarla `configured` con la IP correcta, y `resolvectl status` los DNS.

##### 7.5.6 DNS sin gestor automático (caso especial)

Si `/etc/resolv.conf` no se rellena solo:

```bash
sudo tee /etc/resolv.conf > /dev/null <<'EOF'
nameserver 1.1.1.1
nameserver 9.9.9.9
EOF
```

> **Atención:** `resolvconf`, `systemd-resolved` o el cliente DHCP lo sobrescribirán. Para fijarlo sin gestor:
>
> ```bash
> sudo chattr +i /etc/resolv.conf
> ```
>
> Ni root puede modificarlo hasta `sudo chattr -i /etc/resolv.conf`. Se anota en el inventario (22.1).

##### 7.5.7 Verificación final

```bash
ip -brief address           # IP correcta en la interfaz
ip route                    # gateway por defecto presente
getent hosts deb.debian.org # DNS funcional (resuelve nombre)
ping -c 3 1.1.1.1           # conectividad IP a internet
ping -c 3 deb.debian.org    # conectividad + DNS
```

Los cinco deben funcionar. Si `ping 1.1.1.1` va y `ping deb.debian.org` no, es DNS; sin ruta `default`, el gateway; sin IP en la interfaz, el backend (`journalctl -u networking` o `journalctl -u systemd-networkd`).

##### 7.5.8 Persistencia entre reinicios

Una configuración que funciona en caliente y no sobrevive al reinicio se detecta ahora, con la sesión de rescate abierta (6.4):

```bash
sudo reboot
```

Tras reconectar, se repite 7.5.7.

#### 7.6 `machine-id`

**Opcional:** solo en VMs clonadas de una plantilla, donde el identificador se duplica y causa concesiones DHCP repetidas y logs mezclados. En una imagen que arranca con `/etc/machine-id` vacío, systemd lo toma del Product UUID de SMBIOS (`cat /sys/class/dmi/id/product_uuid`), que el hipervisor asigna a cada VM: si coinciden, es único y no se regenera. Una instalación desde ISO lo genera al azar y no coincide con el Product UUID, sin que eso indique un clon; lo que delata un clon es el mismo `machine-id` en dos VMs.[^machine-id]

```bash
cat /etc/machine-id
```

Si es un clon:

```bash
sudo rm /etc/machine-id /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
sudo dbus-uuidgen --ensure
sudo reboot
```

[^machine-id]: Observado el 2026-11-26 en una instalación desde la ISO netinst 13.8.

### 8. Repositorios y primera actualización completa

La `netinst` instalada sin red deja el CD-ROM como único repositorio; con red, lo deja comentado y apunta a `deb.debian.org` por `http`, con `deb-src` y solo `main non-free-firmware`.[^netinst-sources] En los dos casos se pasa al contenido de 8.2 y se actualiza todo.

#### 8.1 Respaldo del `sources.list` original

```bash
[ -e /etc/apt/sources.list.ori ] || sudo cp /etc/apt/sources.list /etc/apt/sources.list.ori
```

#### 8.2 Edición del `sources.list`

```bash
sudoedit /etc/apt/sources.list
```

Contenido para Debian 13 Trixie:

```text
# Repositorio principal
deb https://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware

# Actualizaciones entre point releases
deb https://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware

# Parches de seguridad
deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

`main` es software libre soportado por Debian; `contrib`, libre con dependencias no libres; `non-free`, propietario, solo si hace falta; `non-free-firmware`, firmware separado en Trixie, que conviene mantener por el hardware moderno. `deb-src` se omite salvo que se compile desde fuente. La línea del CD-ROM se comenta:

```text
# deb cdrom:[...]  (línea comentada o eliminada)
```

Las imágenes cloud traen el repositorio en `/etc/apt/sources.list.d/debian.sources` (deb822, con `mirror+file:`, `deb-src`, backports y solo `main`). Con los dos activos, APT descarga los índices dos veces sin avisar. Antes de `apt update`:

```bash
ls -la /etc/apt/sources.list.d/
```

Si aparece `debian.sources`, se aparta como referencia:

```bash
sudo mv /etc/apt/sources.list.d/debian.sources /etc/apt/debian.sources.ori
```

También vale lo contrario: mantener el formato deb822 y llevar a `debian.sources` los componentes de arriba. Lo que no debe haber son las dos fuentes activas.

[^netinst-sources]: Observado el 2026-11-26 con la ISO netinst 13.8 y red durante la instalación.

#### 8.3 Actualización completa

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove --purge -y
sudo apt clean
```

`upgrade` no instala ni retira paquetes, así que deja sin actualizar lo que cambia de dependencias; `full-upgrade` sí lo hace. En producción se revisa antes con `apt full-upgrade --dry-run`; en una instalación nueva se aplica directamente.

#### 8.4 Reinicio si fue actualizado el kernel

```bash
ls /boot/vmlinuz-*
uname -r
```

Si `/boot` tiene una versión superior a `uname -r`:

```bash
sudo reboot
```

`needrestart`, opcional, avisa de qué necesita reinicio:

```bash
sudo apt install -y needrestart
sudo needrestart
```

`apt autoremove` conserva el kernel anterior a propósito, como respaldo de arranque. Cuando el nuevo ya arrancó sin problemas, se puede purgar:

```bash
uname -r                                     # kernel en ejecución, que no se toca
dpkg -l 'linux-image-[0-9]*' | awk '/^ii/{print $2}'
export KERNEL_VIEJO=linux-image-<versión>    # uno de la lista distinto del de uname -r
sudo apt purge "${KERNEL_VIEJO:?}"
```

El `postrm` del paquete regenera GRUB, que debe listar solo el kernel en ejecución.

### 9. Usuarios y privilegios

Grupos del usuario administrativo, endurecimiento de sudo, política de contraseñas y bloqueo de root.

#### 9.1 Verificación del usuario administrativo

```bash
export MIUSUARIO=usuario
id "${MIUSUARIO:?}"
```

Salida esperada:

```text
uid=1000(<usuario>) gid=1000(<usuario>) grupos=1000(<usuario>),27(sudo),...
```

Debe estar el grupo `sudo` (GID 27). Si falta:

```bash
sudo usermod -aG sudo "${MIUSUARIO:?}"
```

> **Nota:** las variables exportadas solo existen en su sesión. Tras reconectar, o dentro de `sudo -i` (que limpia el entorno), `MIUSUARIO` está vacía. `${MIUSUARIO:?}` convierte el olvido en un error visible.

#### 9.2 Grupos recomendados para el usuario administrativo

`adm` permite leer logs sin `sudo`:

```bash
sudo usermod -aG adm "${MIUSUARIO:?}"
```

Otros grupos dan privilegios equivalentes a root (`docker`) o solo sirven a roles concretos (`wireshark`, `kvm`, `libvirt`; `systemd-journal` sobra con `adm`). Se tratan como concesión de permisos y se añaden solo si el rol lo justifica.

#### 9.3 Hardening de sudo

Tickets largos, sesiones sin registro y entornos heredados son la superficie de `sudo`. La configuración va en `/etc/sudoers.d/`, sin tocar el archivo del paquete, y se ejecuta y verifica desde la sesión del usuario administrativo: root pasa cualquier comprobación de sudo aunque la configuración esté rota. La excepción es 9.3.3, que necesita una shell de root abierta como red de seguridad.

##### 9.3.1 Archivo de hardening de sudoers

```bash
tee /tmp/99-hardening > /dev/null <<'EOF'
# Limpiar el entorno heredado antes de ejecutar como root
Defaults env_reset

# Ruta segura de binarios (no heredar $PATH del usuario invocador)
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# El ticket de sudo expira a los 5 minutos de inactividad
Defaults timestamp_timeout=5

# Requiere TTY real. Descomentar solo en servidores sin automatización alguna;
# ver la nota de esta sección antes de activarla
#Defaults requiretty

# Registrar la sesión completa (output) de cada ejecución sudo en /var/log/sudo-io/
Defaults log_output

# No duplicar en sudo-io la salida de journalctl: el journal ya es el registro original
Defaults!/usr/bin/journalctl !log_output

# Notificar al administrador cuando se introduce una contraseña incorrecta en sudo
Defaults mail_badpass
EOF

# Validar fuera de /etc/sudoers.d/ e instalar solo si la sintaxis es correcta
sudo visudo -c -f /tmp/99-hardening && \
sudo install -m 0440 -o root -g root /tmp/99-hardening /etc/sudoers.d/99-hardening
rm -f /tmp/99-hardening
```

El archivo se escribe y se valida en `/tmp` porque todo lo que está en `/etc/sudoers.d/` entra en vigor al instante: con un error de sintaxis, sudo dejaría de funcionar antes de poder corregirlo. `install` lo coloca ya con `0440` y `root:root`, y el bloque se puede repetir.

`log_output` guarda en `/var/log/sudo-io/` lo que imprime cada orden. Para `journalctl` sería una copia del journal que crece con cada consulta, así que la excepción solo deja de guardar su salida; la línea `COMMAND=` del journal sigue registrando quién consultó y cuándo. Los `Defaults` por comando se aplican después de los globales, sin importar el orden. La retención de `/var/log/sudo-io/` es la 18.2.1.

> `requiretty` queda comentada a propósito: rompe sudo en cron, en units de systemd y en cualquier script sin TTY, con un `sorry, you must have a tty to run sudo` difícil de diagnosticar. Solo se activa en servidores operados exclusivamente a mano, anotándolo en el runbook (22.3). Una excepción puntual se declara así:
>
> ```text
> Defaults:miusuario_ansible !requiretty
> ```
>
> `mail_badpass` no envía correo con el msmtp de la sección 20. El sudo de Debian está compilado con `--disable-root-mailer` y lanza el envío como el usuario que falló, sin sus grupos suplementarios; sin el grupo `msmtp`, msmtp no puede leer `/etc/msmtprc` y termina con el código 78 sin dejar rastro en el journal (validado en un VPS de Contabo el 2026-11-19 con `setpriv --clear-groups`). La línea se mantiene por si el MTA cambia; los intentos fallidos quedan igualmente en el journal de sudo y en `pam_faillock` (9.3.3).

##### 9.3.2 Verificación

```bash
sudo visudo -c      # verifica la sintaxis de todo el árbol sudoers
sudo -l             # lista los privilegios del usuario actual con la nueva configuración
```

Como root, `sudo -l` describe a root, y una configuración rota para `$MIUSUARIO` pasaría. Para consultar a otro usuario sin abrir su sesión:

```bash
sudo -l -U "${MIUSUARIO:?}"
```

Cada archivo de `/etc/sudoers.d/` va con `0440` y `root:root`; si no, `visudo -c` falla con `bad permissions`. `visudo -f` no lo garantiza al crear, así que se aplica siempre:

```bash
export ARCHIVO_SUDOERS=99-hardening   # el archivo creado o editado
sudo chown root:root "/etc/sudoers.d/${ARCHIVO_SUDOERS:?}"
sudo chmod 0440 "/etc/sudoers.d/${ARCHIVO_SUDOERS:?}"
```

> **Nota:** `@includedir` ignora archivos con punto en el nombre o terminados en `~`. Un `99-hardening.conf` no tendría efecto, sin aviso.

Para comprobar el registro de E/S y la excepción de `journalctl`:

```bash
sudo true
sudo journalctl -n 1 > /dev/null
sudo journalctl -n 6 _COMM=sudo | grep COMMAND
```

La línea de `COMMAND=/usr/bin/true` debe llevar `TSID=`, el identificador de la sesión grabada. Las de `journalctl`, no.

##### 9.3.3 pam_faillock — lockout PAM para escalada local

**Opcional, recomendado.** fail2ban protege SSH, no la escalada local: con una sesión sin privilegios se pueden probar contraseñas de `sudo` o `su -` sin límite. `pam_faillock`, incluido en `libpam-modules`, bloquea la cuenta tras varios fallos en cualquier servicio que pase por `pam_unix`.

Es la subsección más delicada del manual. Un error en `/etc/pam.d/common-auth` deja a todos sin autenticación por contraseña, con el archivo válido y ningún error en los logs; con root bloqueado (9.6), solo queda la consola del proveedor.

El usuario administrativo necesita contraseña, porque es la que sudo pedirá:

```bash
sudo passwd -S "${MIUSUARIO:?}"
```

La segunda columna debe ser `P`; si no, `sudo passwd "${MIUSUARIO:?}"`.

Configuración:

```bash
sudo install -m 0644 -o root -g root /dev/stdin /etc/security/faillock.conf <<'EOF'
# Número de fallos antes de bloquear la cuenta
deny = 5

# Ventana de tiempo en segundos durante la que se cuentan los fallos (15 minutos)
fail_interval = 900

# Tiempo de bloqueo en segundos tras alcanzar el límite (10 minutos)
unlock_time = 600

# Registrar eventos en el subsistema de auditoría (auditd)
audit

# No revelar si la cuenta está bloqueada (mismo mensaje que contraseña incorrecta)
silent
EOF
```

Antes de editar PAM se abre una shell de root en la sesión paralela (6.4) y se deja abierta hasta terminar:

```bash
sudo -i
# No cerrar esta shell hasta que la verificación de más abajo haya pasado
export MIUSUARIO=usuario
```

Dentro de `sudo -i` se vuelve a definir `MIUSUARIO` (9.1); sin ella, `su -` y `faillock` fallan con `parameter null or not set`. Desde la sesión principal, un respaldo de la pila de fábrica por si hay que revertir:

```bash
sudo cp /etc/pam.d/common-auth /etc/pam.d/common-auth.ori
```

La pila no se edita a mano. `common-auth` lo gestiona `pam-auth-update`, que regenera el bloque entre sus marcas cada vez que se instala o actualiza un paquete `libpam-*`: una edición manual se pierde en silencio en la primera regeneración. La solución de raíz es declarar `pam_faillock` como perfiles en `/usr/share/pam-configs/`, de modo que cada regeneración la incluya:

<!-- markdownlint-disable MD010 -->

```bash
sudo install -m 0644 -o root -g root /dev/stdin /usr/share/pam-configs/faillock-preauth <<'EOF'
Name: Faillock pre-authentication check
Default: yes
Priority: 257
Auth-Type: Primary
Auth:
	required		pam_faillock.so preauth
Auth-Initial:
	required		pam_faillock.so preauth
EOF

sudo install -m 0644 -o root -g root /dev/stdin /usr/share/pam-configs/faillock-post <<'EOF'
Name: Faillock post-authentication failure handling
Default: yes
Priority: 255
Auth-Type: Primary
Auth:
	[default=die]		pam_faillock.so authfail
Auth-Initial:
	[default=die]		pam_faillock.so authfail
EOF

sudo install -m 0644 -o root -g root /dev/stdin /usr/share/pam-configs/faillock-authsucc <<'EOF'
Name: Faillock reset of failure count on success
Default: yes
Priority: 64
Auth-Type: Additional
Auth:
	sufficient		pam_faillock.so authsucc
EOF
```

Son tres perfiles porque `pam-auth-update` ordena por prioridad descendente y emite las líneas de cada perfil seguidas: uno solo no puede rodear a `pam_unix` (prioridad 256), que necesita `preauth` antes y `authfail` después. `authsucc` va en el bloque "Additional", al final de la pila: dentro del bloque principal, el salto de `pam_unix` se alargaría hasta `pam_permit` y el contador de fallos nunca se limpiaría con una autenticación exitosa; al final, corre después de `pam_permit`, limpia el contador y con `sufficient` cierra la pila.

```bash
sudo pam-auth-update --force
```

`--force` vuelve a habilitar los perfiles con `Default: yes` y regenera los `common-*`; el diálogo de selección muestra los tres de faillock ya marcados y se acepta. La pila resultante, sin comentarios ni líneas vacías:

```bash
grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth
```

```text
auth	required		pam_faillock.so preauth
auth	[success=2 default=ignore]	pam_unix.so nullok try_first_pass
auth	[default=die]		pam_faillock.so authfail
auth	requisite			pam_deny.so
auth	required			pam_permit.so
auth	sufficient		pam_faillock.so authsucc
```

<!-- markdownlint-enable MD010 -->

Respecto de la pila de fábrica, `pam_unix` salta ahora dos líneas en vez de una (`pam-auth-update` renumera el salto) y gana `try_first_pass` por dejar de ser el primer módulo; ambas diferencias son inofensivas. Si la autenticación quedó rota, desde la shell de root se restaura la pila de fábrica con `cp /etc/pam.d/common-auth.ori /etc/pam.d/common-auth`. Para retirar el faillock de forma limpia, ya con todo funcionando:

```bash
sudo rm -f /usr/share/pam-configs/faillock-preauth /usr/share/pam-configs/faillock-post /usr/share/pam-configs/faillock-authsucc
sudo pam-auth-update --force
```

Verificación desde la shell de root, invocando una sesión del usuario, para que un fallo no deje a nadie fuera:

```bash
su - "${MIUSUARIO:?}"
sudo -k
sudo true      # debe pedir la contraseña del usuario y aceptarla
exit
```

Si `sudo true` acepta la contraseña, la pila está bien. Si la rechaza, es el orden de PAM: se revierte y se revisa el `grep`. Después se provoca el bloqueo desde la sesión del usuario, con la shell de root todavía abierta:

```bash
sudo -k
sudo true      # 3 contraseñas incorrectas
sudo true      # otras 3 incorrectas: con deny = 5 la cuenta queda bloqueada
sudo true      # ahora la contraseña CORRECTA: debe rechazarla
```

`sudo -k` descarta el ticket. Con 3 intentos por ejecución, hacen falta dos rondas para llegar a `deny = 5`. La tercera es la que prueba algo: con la cuenta bloqueada, la contraseña correcta debe rechazarse. Que faillock cuente fallos no basta, porque también los cuenta con la pila rota. Por `silent`, el rechazo parece una contraseña incorrecta.

En la shell de root:

```bash
faillock --user "${MIUSUARIO:?}"
faillock --user "${MIUSUARIO:?}" --reset
```

Deben salir exactamente cinco fallos válidos (`V`):

```text
usuario:
When                Type  Source                                           Valid
2026-10-17 11:07:13 TTY   /dev/pts/0                                           V
2026-10-17 11:07:16 TTY   /dev/pts/0                                           V
2026-10-17 11:07:19 TTY   /dev/pts/0                                           V
2026-10-17 11:07:33 TTY   /dev/pts/0                                           V
2026-10-17 11:07:44 TTY   /dev/pts/0                                           V
```

Los intentos posteriores no aparecen porque el `preauth` los rechaza antes. Si las contraseñas se dieron sin terminal, por ejemplo con `sudo -S` desde un script, la columna `Type` dice `SVC` y `Source` dice `sudo`, en lugar de `TTY` y `/dev/pts/0`; la prueba vale igual. Una cuenta bloqueada no puede desbloquearse con su propio `sudo`, de ahí la shell de root previa.

Con las dos pruebas superadas, se guarda la pila verificada como referencia:

```bash
sudo install -m 0644 -o root -g root /etc/pam.d/common-auth /etc/pam.d/common-auth.faillock
```

> **ATENCIÓN.** Con los perfiles, el bloque de `common-auth` queda íntegramente gestionado: instalar o purgar cualquier paquete `libpam-*` regenera la pila con las líneas de faillock en su sitio, sin diálogo ni reedición (validado reinstalando y purgando paquetes `libpam-*` con los perfiles activos). El historial importa porque la edición manual usada hasta la versión 0.9 de este manual fallaba justo al revés: `pam-auth-update` detectaba los cambios locales y preguntaba si sobreescribirlos; con "Sí" — o con `--force` — regeneraba el bloque sin lo añadido a mano, el `pam_unix.so` reactivado dentro del bloque y el faillock fuera, sin que nada lo anunciara. Con "No", que es el default de una corrida no interactiva como la de `unattended-upgrades` (16), no escribía nada (`pam-auth-update: Local modifications to /etc/pam.d/common-*, not updating`) y el cambio pendiente quedaba en cola para la siguiente corrida; `--force` es la salida que el propio mensaje indica.
>
> Tras instalar o actualizar cualquier paquete `libpam-*`, la comprobación de que la pila sigue completa:
>
> ```bash
> diff <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth.faillock) \
>      <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth) && echo "✓ pila de faillock intacta"
> ```
>
> Si `diff` muestra diferencias, se revisa qué añadió `pam-auth-update`, porque puede ser un módulo nuevo legítimo (una línea `auth` de un perfil recién instalado); lo que no puede faltar son las tres líneas de `pam_faillock`.

Cuando sudo rechaza una contraseña que debería valer, el mensaje es siempre el mismo:

| Comprobación | Resultado | Causa |
| --- | --- | --- |
| `faillock --user <usuario>` | tabla con fallos registrados | bloqueo activo; esperar `unlock_time` o `--reset` |
| `faillock --user <usuario>` | tabla vacía tras varios fallos | la pila PAM se corta antes de llegar a faillock; revisar el orden de `pam_deny.so` |
| `passwd -S <usuario>` | segunda columna `L` o `NP` | la cuenta no tiene contraseña utilizable |
| `su - <usuario>` desde root | acepta la contraseña, sudo no | problema en `/etc/pam.d/sudo` o en `/etc/sudoers.d/` |

Una tabla vacía tras varios fallos significa que `authfail` nunca se ejecutó; el journal tampoco tendrá `authentication failure` de `pam_unix`:

```bash
sudo journalctl --since "10 minutes ago" | grep -iE 'pam|faillock'
```

> **Nota:** si queda un `NOPASSWD` para el usuario (5.2), sudo no lo autentica y la prueba no bloquea nada.

Operación:

```bash
# Ver el estado de bloqueo de cualquier usuario
sudo faillock --user "${MIUSUARIO:?}"

# Desbloquear manualmente
sudo faillock --user "${MIUSUARIO:?}" --reset

# Ver los eventos de fallo en auditd (requiere auditd activo, sección 14.1)
sudo ausearch -m USER_LOGIN -sv no | tail -20
```

#### 9.4 Política de contraseñas

**Opcional, recomendado.** Con SSH solo por llave, las contraseñas protegen sobre todo `sudo`.

```bash
sudoedit /etc/login.defs
```

Valores:

```text
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_MIN_LEN    12
PASS_WARN_AGE   14
UMASK           027
```

`PASS_MIN_LEN` y `UMASK` no vienen en el `login.defs` de Debian 13 y se añaden. `UMASK` la aplica `pam_umask` en cada sesión: sin ella vale 022 y los archivos nuevos son legibles por cualquiera. Con `USERGROUPS_ENAB yes`, los bits de grupo se igualan a los del propietario en usuarios con grupo privado, así que la máscara efectiva es 007; lo que importa es que el resto de usuarios queda sin acceso. Responde a AUTH-9328 de lynis (14.5) y rige desde el siguiente inicio de sesión.

También alcanza a lo creado con `sudo`, que combina su 022 con la de la sesión: `sudo mkdir` deja 0750 y `sudo tee` 0640. Los servicios que corren como root no lo notan, pero las herramientas que corren como usuario dejan de leer esos archivos sin avisar (`systemd-analyze cat-config` omite un drop-in ilegible). Por eso, desde aquí, los archivos bajo `/etc` se crean con `install` y permisos explícitos.

Lo mismo ocurre con las herramientas de debconf lanzadas con `sudo` (`debconf-set-selections`, `dpkg-reconfigure`): reescriben `/var/cache/debconf/config.dat` y `templates.dat` con 0640, y AIDE (14.3) lo marca como cambio. Se evita lanzándolas en una shell con `umask 022`, o se corrige después con `sudo chmod 0644 /var/cache/debconf/config.dat* /var/cache/debconf/templates.dat*`.

> **Nota:** con `pam_pwquality` activo, `PASS_MIN_LEN` se ignora y el mínimo es `minlen`. Se mantiene como respaldo.

Para aplicar la política al usuario actual:

```bash
sudo chage -M 90 -m 1 -W 14 "${MIUSUARIO:?}"
sudo chage -l "${MIUSUARIO:?}"
```

Contraseñas robustas con `libpam-pwquality`:

```bash
sudo apt install -y libpam-pwquality
```

> **ATENCIÓN.** Su postinst ejecuta `pam-auth-update`, y el efecto sobre la pila de faillock varía: en unas instalaciones la deja intacta y en otras la deshace, de modo que la contraseña correcta deja entrar a una cuenta bloqueada. Por eso, justo después se aplica la comprobación de la ATENCIÓN de 9.3.3.

El paquete añade `pam_pwquality.so retry=3` a `common-password`, dentro del bloque gestionado, donde cualquier parámetro se perdería. La política va en `/etc/security/pwquality.conf`, que `pam-auth-update` no toca:

```bash
[ -e /etc/security/pwquality.conf.ori ] || sudo cp /etc/security/pwquality.conf /etc/security/pwquality.conf.ori
grep -q '^# Política del manual' /etc/security/pwquality.conf \
  || sudo tee -a /etc/security/pwquality.conf > /dev/null <<'EOF'

# Política del manual, sección 9.4
minlen = 12
difok = 3
ucredit = -1
lcredit = -1
dcredit = -1
ocredit = -1
usercheck = 1
enforce_for_root
EOF
```

El respaldo y el bloque solo se escriben la primera vez, así que repetir la orden no duplica nada. `minlen = 12` es el mínimo efectivo; `difok = 3`, caracteres distintos de la anterior; los `*credit = -1` exigen mayúscula, minúscula, dígito y símbolo; `usercheck = 1` rechaza el nombre de usuario; `enforce_for_root` aplica la política también a root. La línea de `common-password` debe seguir siendo la del paquete:

```bash
grep -vE '^\s*#|^\s*$' /etc/pam.d/common-password
```

```text
password    requisite            pam_pwquality.so retry=3
password    [success=1 default=ignore]    pam_unix.so obscure use_authtok try_first_pass yescrypt
password    requisite            pam_deny.so
password    required            pam_permit.so
```

Se prueba con `passwd`: una contraseña corta debe rechazarse tras pedir la actual.

#### 9.5 Creación de usuarios adicionales (si aplica)

Solo si hay más administradores:

```bash
export NUEVO_USUARIO=nombre
sudo adduser "${NUEVO_USUARIO:?}"
sudo usermod -aG sudo,adm "${NUEVO_USUARIO:?}"
```

#### 9.6 Bloqueo de la cuenta de root

Con `sudo` funcionando, la llave SSH verificada y la sesión confirmada, root se puede bloquear. La cuenta sigue operativa para los procesos del sistema:

```bash
sudo passwd -l root
```

```bash
sudo passwd -S root
# Debe mostrar:  root L <fecha> ...   (L = locked)
```

`su -` deja de funcionar con la contraseña de root, pero `sudo -i` sigue disponible; root no inicia sesión por consola ni por SSH (`PermitRootLogin no`, sección 10), y cron y systemd siguen igual. Para recuperarla, sin conocer la contraseña anterior:

```bash
sudo passwd root
```

---

## Etapa III — Hardening

Reduce la superficie de ataque, dificulta el abuso de un punto de apoyo y deja rastro para el análisis forense. Es el mínimo razonable para cualquier servidor expuesto; cada rol añade sus capas en su manual. Tres reglas atraviesan la etapa: la sesión de rescate (6.4) sigue abierta antes de tocar SSH, firewall o red; los cambios van en archivos propios dentro de directorios `*.d/`, que se revierten borrando un archivo y sobreviven a las actualizaciones; y se valida antes de aplicar (`sshd -t`, `ufw show added`, `unattended-upgrades --dry-run`).

### 10. Hardening de SSH

SSH es la puerta principal: si cae, ninguna otra medida importa. La sección cambia el puerto, quita las contraseñas, restringe usuarios, acota sesiones, recorta la criptografía y recarga sin perder la sesión.

#### 10.1 Archivo modular `/etc/ssh/sshd_config.d/01-hardening.conf`

Desde OpenSSH 8.2 (Debian 11), `sshd_config` empieza con `Include /etc/ssh/sshd_config.d/*.conf`. Los archivos de ese directorio se leen en orden y antes que el principal, y gana la primera aparición de cada directiva: `01-hardening.conf` prevalece sobre `sshd_config` y sobre cualquier drop-in con número mayor, y el archivo del paquete queda intacto para las actualizaciones.

```bash
# Definir las variables
export MIPUERTO=17177
export MIUSUARIO=usuario # Definido en 4.2.2 o 4.2.4

# Descargar y aplicar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" "${MIPUERTO:?}" "${MIUSUARIO:?}" && \
id "${MIUSUARIO}" > /dev/null && \
wget "${ASSETS_BASE}/plantilla-etc-ssh-sshd_config.d-01-hardening.conf.txt" \
    -O /tmp/plantilla-etc-ssh-sshd_config.d-01-hardening.conf.txt && \
sed -e "s|\$MIPUERTO|${MIPUERTO}|g" \
    -e "s|\$MIUSUARIO|${MIUSUARIO}|g" \
    /tmp/plantilla-etc-ssh-sshd_config.d-01-hardening.conf.txt \
    | sudo install -m 0600 -o root -g root /dev/stdin /etc/ssh/sshd_config.d/01-hardening.conf
```

El `id` de la cadena es la guarda importante: la plantilla escribe `AllowUsers ${MIUSUARIO}` (10.3), y con un nombre mal escrito `sshd -t` acepta el archivo y tras recargar no entra nadie.

```bash
sudo grep -nE '^(Port|AllowUsers)' /etc/ssh/sshd_config.d/01-hardening.conf
sudo grep -n '\$MI[A-Z_]*' /etc/ssh/sshd_config.d/01-hardening.conf || echo "✓ plantilla aplicada"
```

```bash
sudo sshd -t && echo "✓ sintaxis válida"
```

#### 10.2 Cambio de puerto, password auth y solo pubkey

**Puerto.** Un puerto alto (17177) no es seguridad real, porque un escaneo dirigido lo encuentra, pero corta casi todo el ruido automatizado. Antes de recargar SSH, el puerto nuevo se abre en el firewall del proveedor (11.6) y en el router o NAT si los hay. Se comprueba desde el equipo del administrador, que no tiene las variables del servidor:

```bash
IP_SERVIDOR="203.0.113.10"   # IP pública del servidor
MIHOST=villonaco             # el mismo nombre corto de 7.1
MIPUERTO=17177
MIUSUARIO=usuario
nc -zv "${IP_SERVIDOR:?}" "${MIPUERTO:?}"
```

Antes de la recarga de 10.6 la respuesta esperada es `Connection refused`: los paquetes llegan y nada escucha todavía. Si agota el tiempo, algún firewall del camino lo cierra (11.6.3).

**Sin contraseñas.** Solo se acepta llave pública. Si la llave no se probó en 6.3, **no se aplica todavía**. `KbdInteractiveAuthentication no` es imprescindible: con `UsePAM yes`, PAM puede pedir la contraseña por el método keyboard-interactive aunque `PasswordAuthentication` esté apagado. `UsePAM yes` se mantiene para la gestión de sesión.

```bash
# Configuración efectiva tras fusionar sshd_config con todo sshd_config.d/
sudo sshd -T | grep -Ei "^(passwordauthentication|kbdinteractiveauthentication|usepam|pubkeyauthentication)"
```

Salida esperada:

```text
passwordauthentication no
kbdinteractiveauthentication no
pubkeyauthentication yes
usepam yes
```

Desde el cliente, sin llegar a autenticarse:

```bash
# Desde la estación del administrador, con las variables declaradas arriba.
# Debe responder que solo acepta publickey.
ssh -p "${MIPUERTO:?}" -o PreferredAuthentications=password \
    -o PubkeyAuthentication=no "${MIUSUARIO:?}@${IP_SERVIDOR:?}"
```

Debe responder `Permission denied (publickey)`. Si sale un prompt de contraseña, alguna directiva no surte efecto.

#### 10.3 Restricción de usuarios (`AllowUsers`)

Lista blanca: solo esos usuarios entran por SSH, aunque otra cuenta tenga una llave instalada por error.

```text
AllowUsers $MIUSUARIO
```

Para varios administradores:

```text
AllowUsers <usuario1> <usuario2>
```

Admite origen (`AllowUsers usuario2@192.168.1.0/24`). Con muchos usuarios, `AllowGroups` con un grupo como `ssh-admins` se mantiene mejor.

#### 10.4 Límites de sesión y timeouts

| Directiva                 | Valor | Efecto                                                                       |
|---------------------------|-------|------------------------------------------------------------------------------|
| `MaxAuthTries 3`          | 3     | El cliente se desconecta tras 3 intentos fallidos en una misma conexión.     |
| `MaxSessions 5`           | 5     | Máximo de sesiones (canales) por conexión TCP. Suficiente para un humano.    |
| `LoginGraceTime 30`       | 30 s  | Tiempo máximo para completar el login. Corta sesiones que solo abren TCP.    |
| `ClientAliveInterval 300` | 5 min | Cada 5 min envía un keepalive cifrado al cliente.                            |
| `ClientAliveCountMax 2`   | 2     | Tras 2 keepalives sin respuesta (10 min), cierra la sesión.                  |

`MaxAuthTries 3` cuenta cada llave que el cliente ofrece y se rechaza. Un cliente con varias llaves las prueba en orden y, si la buena no está entre las tres primeras, recibe `Too many authentication failures`; esos rechazos cuentan además para fail2ban. Se fija la llave en el `~/.ssh/config` del equipo del administrador:

```bash
[ -f ~/.ssh/config ] || install -m 600 /dev/null ~/.ssh/config
grep -qx "Host ${MIHOST:?}" ~/.ssh/config || cat >> ~/.ssh/config <<EOF

Host ${MIHOST:?}
    HostName ${IP_SERVIDOR:?}
    Port ${MIPUERTO:?}
    User ${MIUSUARIO:?}
    IdentityFile ~/.ssh/${MINOMBREDELLAVE:?}
    IdentitiesOnly yes
    ControlMaster auto
    ControlPath ~/.ssh/cm-%C
    ControlPersist 10m
EOF
chmod 600 ~/.ssh/config
```

Si el archivo no existe, se crea con modo 600; si ya tiene un `Host` con ese nombre, no se duplica y se revisa a mano. `IdentitiesOnly yes` limita el cliente a la llave de `IdentityFile`. Desde la recarga de 10.6, basta `ssh "${MIHOST}"`.

> GNOME Keyring puede responder `agent refused operation` si no logra mostrar el diálogo de passphrase, y ssh no reintenta con el archivo. `IdentityAgent none` en el bloque lo evita.

Las líneas `Control*` evitan el `limit` de UFW (11.3), que rechaza más de 6 conexiones en 30 segundos desde una misma IP, también las del administrador, y renueva la ventana con cada intento. Con `ControlMaster`, las conexiones siguientes viajan por la primera y no cuentan; `ControlPersist 10m` la mantiene diez minutos, y de paso las conexiones se abren al instante.

```bash
ssh -O check "${MIHOST}"
ssh -O exit "${MIHOST}"
```

`ClientAliveInterval` y `ClientAliveCountMax` mantienen viva la sesión frente a firewalls que cortan las inactivas y cierran las abandonadas.

El reenvío se desactiva por completo: de puertos hacia servicios internos, túneles SOCKS, agente y sockets Unix. Una sola directiva, desde OpenSSH 8.9:

```text
DisableForwarding yes
```

Sustituye a `X11Forwarding`, `AllowAgentForwarding`, `AllowTcpForwarding`, `AllowStreamLocalForwarding` y `PermitTunnel`, y según `sshd_config(5)` prevalece sobre todas ellas, así que ningún drop-in ni `Match` posterior reabre un reenvío.

> **Consecuencia operativa.** El servidor deja de servir de salto (`ssh -J`, `ProxyJump`) y no funcionan `ssh -L`, `-R` ni `-D`. Si el rol lo necesita, se decide aquí.

Para conceder un reenvío puntual, dentro del `Match` hay que desactivar `DisableForwarding` y luego permitir el tipo concreto:

```text
Match User operador-tuneles
    DisableForwarding no
    AllowTcpForwarding local
    PermitOpen 10.0.0.5:5432
```

`AllowTcpForwarding local` concede solo `ssh -L`, y `PermitOpen` acota el destino; `yes` a secas sería un proxy de propósito general.

```bash
sudo sshd -T | grep -Ei "^(disableforwarding|allowtcpforwarding|x11forwarding|permittunnel)"

# Configuración efectiva para un usuario concreto, resolviendo los bloques Match
sudo sshd -T -C user=operador-tuneles,host=localhost,addr=127.0.0.1 \
    | grep -Ei "^(disableforwarding|allowtcpforwarding|permitopen)"
```

`-C` simula una conexión y aplica los `Match`. Con la plantilla tal cual, la primera orden da:

```text
x11forwarding yes
allowtcpforwarding yes
disableforwarding yes
permittunnel no
```

`x11forwarding yes` y `allowtcpforwarding yes` no indican nada: `sshd -T` informa de cada opción por separado y `disableforwarding yes` prevalece sobre ambas. La segunda orden solo prueba algo si se añadió el `Match` del ejemplo.

> **PerSourcePenalties.** OpenSSH 10.0 rechaza durante un tiempo las conexiones de una dirección que acumula fallos o desconexiones sin autenticar. Es una capa interna de `sshd`, de memoria corta, que no sustituye a fail2ban (sección 12) pero explica rechazos antes de que fail2ban cuente nada. Se consulta con `sudo sshd -T | grep -i persource`. También penaliza agotar `LoginGraceTime` (10 s por defecto): si la llave tiene frase y se tarda más de 30 s en escribirla, o el diálogo de passphrase tarda en aparecer, sshd corta la conexión (`Connection closed by <ip> port <puerto>`) y las siguientes desde esa dirección se rechazan unos segundos. Reintentar seguido alarga la espera; basta esperar un minuto.

#### 10.5 Algoritmos criptográficos modernos

**Opcional, recomendado.** OpenSSH 10 ya negocia algoritmos modernos por defecto; recortar la lista cierra los ataques de downgrade cuando todos los clientes son conocidos.

**Key exchange (`KexAlgorithms`).**

```text
KexAlgorithms mlkem768x25519-sha256,curve25519-sha256,curve25519-sha256@libssh.org,sntrup761x25519-sha512@openssh.com
```

- `mlkem768x25519-sha256`: híbrido post-cuántico (ML-KEM con X25519), el valor por defecto desde OpenSSH 10.0; va primero.
- `curve25519-sha256`: clásico, para clientes anteriores a 9.0.
- `sntrup761x25519-sha512@openssh.com`: el híbrido anterior, por defecto entre 9.0 y 9.9. Los dos híbridos protegen frente a "harvest now, decrypt later".

Desde el equipo del administrador:

```bash
ssh -v "${MIHOST:?}" true 2>&1 | grep 'kex: algorithm'
```

Con un cliente OpenSSH 10.0:

```text
debug1: kex: algorithm: mlkem768x25519-sha256
```

**Cifradores (`Ciphers`).** Solo AEAD; fuera CBC y CTR.

```text
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
```

**MACs.** Solo Encrypt-then-MAC con SHA-2.

```text
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

**Algoritmos de host key (`HostKeyAlgorithms`).** Sin `ssh-rsa` (RSA con SHA-1), que solo afecta a clientes muy antiguos.

```text
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

> Antes de recargar, desde el cliente: `ssh -Q kex`, `ssh -Q cipher` y `ssh -Q mac` deben incluir al menos un algoritmo de cada lista.

#### 10.6 Validación y recarga segura

Antes de recargar: la sesión paralela (6.4) sigue conectada, la llave está probada y se conocen la IP y el puerto nuevo.

```bash
sudo sshd -t
```

Sin salida, la sintaxis es correcta. Con errores, no se recarga.

```bash
sudo systemctl reload ssh
```

`reload` no corta las sesiones abiertas, que siguen con la configuración anterior. Desde otra ventana:

```bash
ssh "${MIHOST:?}"
```

`MIHOST` es el alias del bloque `Host` de 10.4. Solo cuando la conexión nueva funciona se cierra la antigua.

> Si falla, **no se cierra la sesión antigua**. Desde ella: `journalctl -u ssh -n 50`, `sudo sshd -T | grep -E 'port|allowusers|passwordauthentication'` y `sudo ss -tlnp | grep ssh`.

La configuración efectiva, tras fusionar los drop-ins:

```bash
sudo sshd -T | grep -Ei \
  "^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers|maxauthtries|logingracetime|disableforwarding|kexalgorithms|ciphers|macs)"
```

Un valor por defecto donde debería estar el configurado indica que el archivo no se lee, que otro drop-in lo fijó antes o que la directiva está mal escrita.

### 11. Firewall (UFW)

UFW genera reglas de `iptables`, que en Debian 13 es `iptables-nft`: el kernel las aplica como nftables y aparecen en `nft list ruleset`. Basta para un servidor con pocos puertos; NAT, balanceo o reglas por interfaz piden `nftables` directo. Se deniega todo lo entrante salvo lo necesario, se permite lo saliente (restringirlo rompe DNS, NTP y repositorios a cambio de poco) y se filtra IPv6, porque apagarlo en UFW no lo apaga en el sistema, solo lo deja sin firewall. `ufw enable` va al final, con todas las reglas revisadas. En la nube, el firewall del proveedor (11.6) se abre a la vez y con las mismas reglas.

#### 11.1 Instalación y políticas por defecto

```bash
sudo apt install -y ufw
```

UFW queda desactivado tras instalarse.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

`deny routed` impide que el servidor se convierta en router abierto si algún día se activa el reenvío.

#### 11.2 IPv6 (`/etc/default/ufw`)

```bash
sudoedit /etc/default/ufw
```

Debe estar:

```text
IPV6=yes
```

Es el valor de Debian 13. Con `no` y una interfaz con IPv6, el tráfico IPv6 entrante queda sin filtrar por UFW.

#### 11.3 Apertura selectiva de puertos según rol

**SSH con `limit`.** Rechaza a una misma IP que supere 6 conexiones en 30 segundos:

```bash
export MIPUERTO=17177 # Definido en 10.1
sudo ufw limit ${MIPUERTO}/tcp comment 'SSH endurecido con rate limit'
```

Complementa a fail2ban (sección 12), que banea por patrones de log durante más tiempo.

**Servicios web.** Si el servicio irá detrás de un proxy de borde, estas órdenes no se ejecutan: los puertos se abren ya restringidos a los rangos del borde con la sección 17 del manual de Caddy. Si se publica directamente:

```bash
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
```

**Origen restringido**, cuando se sabe desde dónde se administra:

```bash
# SSH solo desde una red administrativa
sudo ufw limit from 192.0.2.0/24 to any port ${MIPUERTO} proto tcp comment 'SSH desde red admin'

# Base de datos accesible solo desde el servidor de aplicación
sudo ufw allow from 10.0.0.5 to any port 5432 proto tcp comment 'Postgres desde app'
```

En el manual base solo se abre SSH; el resto lo abre el manual del rol.

**Filtrado de egreso.** **Opcional.** Con el rol definido, limitar la salida a lo necesario dificulta la exfiltración y el contacto con infraestructura de mando de un servidor comprometido. Por ejemplo, para repositorios, NTP y un relay SMTP:

```bash
# Cambiar la política de egreso a deny y abrir solo lo necesario
sudo ufw default deny outgoing
sudo ufw allow out 53/udp comment 'DNS'
sudo ufw allow out 80/tcp comment 'HTTP (apt, mirrors)'
sudo ufw allow out 443/tcp comment 'HTTPS (apt, mirrors)'
sudo ufw allow out 123/udp comment 'NTP'
sudo ufw allow out 587/tcp comment 'SMTP relay'
```

#### 11.4 Inspección antes de activar

```bash
sudo ufw show added
```

Muestra las reglas registradas y aún inactivas; la de SSH debe tener el puerto correcto. Para borrar una mal escrita:

```bash
REGLA="allow 80/tcp"   # la regla tal como la muestra "ufw show added", sin "ufw"
sudo ufw delete ${REGLA:?}
```

`REGLA` va sin comillas a propósito, para que `ufw` reciba cada palabra como un argumento.

#### 11.5 Activación final y verificación

```bash
sudo ufw enable
sudo systemctl enable --now ufw
```

Ante *"Command may disrupt existing ssh connections"*, se confirma con `y` y se prueba **desde otra ventana** la conexión al puerto nuevo. Si falla, desde la sesión paralela:

```bash
sudo ufw disable
# Diagnosticar y volver a aplicar
```

```bash
sudo ufw status verbose
sudo ufw status numbered
```

`numbered` sirve para borrar una regla con `sudo ufw delete <n>`. Con solo SSH, `ufw status verbose` muestra:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
17177/tcp                  LIMIT IN    Anywhere                   # SSH endurecido con rate limit
17177/tcp (v6)             LIMIT IN    Anywhere (v6)              # SSH endurecido con rate limit
```

`disabled (routed)` aparece mientras `ip_forward` vale `0`, porque no hay tráfico enrutado que filtrar; la política `deny` queda guardada en `/etc/default/ufw` (`DEFAULT_FORWARD_POLICY="DROP"`). Con reenvío activo (`net.ipv4.ip_forward=1` y `net.ipv6.conf.all.forwarding=1`) pasa a `deny (routed)`.

**Desde fuera**, desde el equipo del administrador con las variables de 10.2:

```bash
nc -zv -w 5 "${IP_SERVIDOR:?}" "${MIPUERTO:?}"   # debe conectar
nc -zv -w 5 "${IP_SERVIDOR:?}" 22                # debe agotar el tiempo
nc -zv -w 5 "${IP_SERVIDOR:?}" 80                # debe agotar el tiempo
```

Un puerto no abierto debe agotar el tiempo. `Connection refused` significa que los paquetes llegan sin filtrar y que el puerto quedará expuesto cuando algo escuche en él.

Las reglas tal como las aplica el kernel:

```bash
sudo nft list ruleset | less
```

`nft` es del paquete `nftables`, que algunas imágenes cloud no traen (`nft: orden no encontrada`); lo instala la sección 12.1. Sin él, `sudo iptables -S` y `sudo ip6tables -S` muestran las mismas reglas.

#### 11.6 Firewall del proveedor de nube

En la nube hay dos firewalls en serie: el del proveedor, en su red, y UFW, en la máquina. Un puerto abierto en UFW y cerrado en el proveedor no recibe nada; uno cerrado en UFW y abierto en el proveedor puede quedar expuesto si lo publica un contenedor. Aplica a los grupos de seguridad de Azure y AWS, los firewalls de Hetzner, DigitalOcean o Hostinger y las listas de seguridad de Oracle. En físico o en una VM sin esa capa no hay nada que configurar. Las órdenes de cada proveedor están en [`proveedores/`](proveedores/).

##### 11.6.1 Dos firewalls, dos alcances

Los dos se mantienen con las mismas reglas, porque cada uno cubre un fallo del otro. El del proveedor filtra antes de que el tráfico llegue, sobrevive a un error dentro de la máquina y es el único que alcanza a los puertos publicados por contenedores: el demonio inserta sus reglas en cadenas que UFW no gobierna. UFW cubre lo que el proveedor no ve, como las otras máquinas de la misma red virtual o una regla abierta por error en el panel, y se respalda y audita con el resto de `/etc`.

> Sin firewall de proveedor, los puertos de contenedores quedan expuestos digan lo que digan las reglas de UFW. Filtrarlos en la cadena que consulta el propio demonio no está documentado ni verificado (To-do). Mientras tanto, no se publican puertos de contenedores en un servidor sin esa capa.

##### 11.6.2 Reglas según el momento del despliegue

| Momento | Reglas de entrada | Dónde se decide |
| --- | --- | --- |
| Despliegue base, este manual | Solo SSH en el puerto de la sección 10 | Sección 11.3 |
| Servicio web publicado directamente | SSH, más 80 y 443 desde cualquier origen | Manual del proxy inverso |
| Servicio web detrás de un borde | SSH, más 443 solo desde los rangos del borde | Manual de Caddy, sección 17 |

Tres criterios valen para cualquier proveedor. Se borra lo que sobre de lo que ya exista, empezando por la regla del puerto 22 que suelen dejar los asistentes. Las reglas temporales se anotan al crearlas, se limitan a la dirección del administrador y se borran en la misma sesión. SSH se restringe a la dirección del administrador solo si es estable; si cambia, la restricción acaba cortando el acceso, y la protección real ya la dan la llave y fail2ban.

##### 11.6.3 Verificación desde fuera

Desde una máquina que no sea el servidor, con las variables de 10.2, tras cada cambio de reglas aquí o en UFW:

```bash
PUERTO_CONTENEDOR=8080   # un puerto publicado por un contenedor, si los hay
nc -zv -w 5 "${IP_SERVIDOR:?}" "${MIPUERTO:?}"            # debe conectar
nc -zv -w 5 "${IP_SERVIDOR:?}" 22                         # debe agotar el tiempo
nc -zv -w 5 "${IP_SERVIDOR:?}" 443                        # según la fila de 11.6.2 que aplique
nc -zv -w 5 "${IP_SERVIDOR:?}" "${PUERTO_CONTENEDOR:?}"   # debe agotar el tiempo
```

Si conecta, el puerto está abierto en los dos firewalls y hay servicio. Si agota el tiempo, algún firewall lo descarta, lo deseado en todo puerto no público. `connection refused` significa que el camino está abierto y nada escucha, y quedará expuesto cuando algo lo haga. Detrás de un borde, el 443 debe agotar el tiempo desde fuera del borde, y un puerto de contenedor que conecta desde fuera está expuesto, diga lo que diga `ufw status`.

##### 11.6.4 Mantenimiento

Las reglas del proveedor no viajan con los respaldos del servidor, así que se documentan en el inventario (22.1) y el diagrama (22.2). La regla que no se explique con una fila de 11.6.2 se borra. La revisión y actualización de cada proveedor están en su documento de [`proveedores/`](proveedores/); cuando cambian los rangos del borde, se actualizan en la misma sesión que UFW (manual de Caddy, sección 17.5).

#### 11.7 Baneo permanente de direcciones

fail2ban (12) banea por un tiempo. Una dirección que se quiere bloquear para siempre, como la de un atacante persistente, se deniega en UFW por delante de las demás reglas:

```bash
export IP_BANEADA=203.0.113.50
sudo ufw insert 1 deny from "${IP_BANEADA:?}" to any comment "Fuerza bruta SSH $(date +%F) (baneo definitivo)"
sudo ufw status numbered | head -5
```

`insert 1` la coloca antes del `limit` de SSH, así que esa dirección no llega a ningún puerto. Una dirección IPv6 crea la regla solo para IPv6. Si el proveedor tiene firewall de red (11.6), la misma regla de bloqueo se añade allí, antes de la regla de SSH, para que el tráfico no llegue al servidor. Las reglas permanentes se anotan en el inventario (22.1) con su motivo.

### 12. Protección contra fuerza bruta (fail2ban)

**Opcional, recomendado.** Con SSH solo por llave, en un puerto alto, con `limit` en UFW y con `PerSourcePenalties`, un atacante no pasa de la autenticación. fail2ban lee el journal, banea las direcciones que insisten y reduce el ruido que tapa los eventos reales; es la capa que sirve además para los servicios que añadan los roles.

> **Relación con `pam_faillock` (§9.3.3):** fail2ban actúa sobre accesos por red; `pam_faillock`, sobre la autenticación local (`su`, `sudo`, consola), que no deja entradas que fail2ban pueda leer. No se solapan.

#### 12.1 Instalación

```bash
sudo apt install -y fail2ban nftables
```

En Debian 13 usa nftables y journald por defecto; no hay que tocar el backend. La acción `nftables-multiport` de la plantilla (12.2) ejecuta `nft`, que es del paquete `nftables`: el paquete `fail2ban` no lo instala, y en algunas imágenes cloud no viene. El servicio `nftables` queda desactivado; solo hace falta la herramienta.

#### 12.2 Configuración local (`jail.local`)

`jail.conf` se sobrescribe en las actualizaciones; lo propio va en `jail.local`:

```bash
export MIPUERTO=17177 # Definido en 10.1

# Descargar y aplicar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" "${MIPUERTO:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-fail2ban-jail.local.txt" \
    -O /tmp/plantilla-etc-fail2ban-jail.local.txt && \
sed -e "s|\$MIPUERTO|${MIPUERTO}|g" \
    /tmp/plantilla-etc-fail2ban-jail.local.txt \
    | sudo install -m 0640 -o root -g root /dev/stdin /etc/fail2ban/jail.local

# Comprobar que no quedó ningún marcador sin sustituir
sudo grep -n '\$MI[A-Z_]*' /etc/fail2ban/jail.local || echo "✓ plantilla aplicada"
```

`bantime` es la duración del baneo; `findtime`, la ventana en la que se cuentan `maxretry` fallos; `bantime.increment` dobla el baneo de los reincidentes hasta `maxtime`; `mode = aggressive` cuenta además desconexiones en preauth y protocolos viejos. `port = 17177` debe coincidir con el puerto de SSH.

> **`ignoreip`.** Las redes de administración evitan un autobaneo tras varios fallos legítimos, pero una red amplia o ajena anula la protección.

```ini
ignoreip = 127.0.0.1/8 ::1 192.0.2.0/24
```

#### 12.3 Activación

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
sudo systemctl status fail2ban --no-pager
```

El paquete arranca el servicio al instalarse, antes de que exista `jail.local`, y `enable --now` no lo reinicia: seguiría vigilando el puerto 22 con los valores de Debian. Por eso el `restart`. Para confirmar que el jail arrancó después del archivo:

```bash
sudo grep -E "Jail 'sshd' started" /var/log/fail2ban.log | tail -1
sudo ls -l --time-style=full-iso /etc/fail2ban/jail.local
```

La hora de arranque debe ser posterior a la del archivo. Si `status` muestra `failed`:

```bash
sudo journalctl -u fail2ban -n 50 --no-pager
```

#### 12.4 Monitoreo de bans

```bash
sudo fail2ban-client status
```

```bash
sudo fail2ban-client status sshd
```

En un servidor recién configurado, con un fallo:

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed:    1
|  |- Total failed:    1
|  `- Journal matches:    _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned:    0
   |- Total banned:    0
   `- Banned IP list:
```

Con `backend = systemd` el filtro dice `Journal matches`; un `File list` indica que `jail.local` no se cargó. El `+` es un O lógico: `_SYSTEMD_UNIT=ssh.service` cubre también las líneas de `sshd-session`, que registra la autenticación desde OpenSSH 9.8. Para comprobar el filtro sin esperar un ataque:

```bash
fail2ban-regex "2026-10-17T00:04:29-05:00 servidor sshd-session[7039]: Connection closed by authenticating user usuario 203.0.113.9 port 57178 [preauth]" \
    "sshd[mode=aggressive]" | grep -E '^Lines:'
```

```text
Lines: 1 lines, 0 ignored, 1 matched, 0 missed
```

Las líneas `srclimit_penalise` de `PerSourcePenalties` (10.4) no las reconoce el filtro, y no hace falta.

Para comprobar que el baneo llega a nftables, con una dirección de documentación que no se usa en Internet:

```bash
sudo fail2ban-client set sshd banip 192.0.2.1
sudo nft list table inet f2b-table
sudo fail2ban-client set sshd unbanip 192.0.2.1
```

La tabla debe incluir `192.0.2.1` en `addr-set-sshd` y una regla `tcp dport <puerto> ip saddr @addr-set-sshd reject`.

Operación:

```bash
# Desbanear una IP manualmente (por ejemplo, si fue uno mismo)
sudo fail2ban-client set sshd unbanip 1.2.3.4

# Banear manualmente una IP
sudo fail2ban-client set sshd banip 1.2.3.4

# Ver IPs que han sido baneadas alguna vez en este jail
sudo fail2ban-client get sshd banip --with-time
```

#### 12.5 Límites del modelo por IP

fail2ban cuenta fallos por dirección y funciona contra quien insiste desde una máquina, el caso habitual en SSH. No sirve contra una campaña distribuida: cientos de direcciones con un intento cada varios minutos, durante días, ninguna llega a `maxretry` y bajar el umbral solo banea a usuarios legítimos. La señal está en el agregado (peticiones por minuto contra una ruta y direcciones distintas por hora), que mide el síntoma B.13 del manual de Caddy.

Hay dos respuestas, ninguna de fail2ban: un límite de ritmo global por ruta en el proxy inverso, que cuenta todas las peticiones sin distinguir origen, y la reputación compartida, como CrowdSec, donde una dirección señalada por otro operador llega ya bloqueada. CrowdSec no se ha desplegado ni verificado (To-do). Nada de esto es motivo para retirar fail2ban, que sigue siendo la defensa contra el abuso concentrado.

### 13. Hardening del kernel y del sistema

Parámetros del kernel, módulos, cargador de arranque y servicios en escucha. Como en SSH, los cambios van en archivos propios de `/etc/sysctl.d/` y `/etc/modprobe.d/`.

#### 13.1 Parámetros sysctl de red y de kernel

```bash
# Descargar e instalar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-sysctl.d-99-hardening.conf.txt" \
    -O /tmp/plantilla-etc-sysctl.d-99-hardening.conf.txt && \
sudo install -m 0644 -o root -g root /tmp/plantilla-etc-sysctl.d-99-hardening.conf.txt /etc/sysctl.d/99-hardening.conf
```

- `rp_filter = 1`: descarta paquetes cuyo camino de vuelta saldría por otra interfaz, lo que frena el spoofing trivial.
- `tcp_syncookies = 1`: ante una cola de SYN llena responde con cookies en lugar de guardar estado.
- `accept_redirects = 0` y `send_redirects = 0`: los ICMP redirects son una vía clásica de intermediario local y solo tienen sentido en routers.
- `log_martians = 1`: registra paquetes con origen imposible.
- `fs.suid_dumpable = 0`: sin volcados de memoria de procesos setuid, que pueden contener credenciales.
- `dev.tty.ldisc_autoload = 0` y `net.core.bpf_jit_harden = 2`: añadidos en la v1.2 por KRNL-6000 de lynis (14.5). El primero impide a un usuario sin privilegios cargar disciplinas de línea de TTY; el segundo endurece el JIT de BPF también para root. De esa misma sugerencia se descarta `kernel.modules_disabled = 1`, que impide cargar módulos hasta reiniciar, y `kernel.unprivileged_bpf_disabled` se deja en el 2 de Debian, que ya bloquea BPF sin privilegios.

```bash
sudo sysctl --system
```

La primera vez conviene leer toda la salida: una clave rechazada aparece como `sysctl: setting key "...": Invalid argument` entre decenas de líneas correctas. `--system` recarga `/etc/sysctl.conf`, `/etc/sysctl.d/` y `/usr/lib/sysctl.d/`.

> `kernel.perf_event_paranoid = 3` solo existe en Debian, que lo trae por defecto; aquí se hace explícito.

```bash
sudo sysctl kernel.kptr_restrict
sudo sysctl net.ipv4.tcp_syncookies
```

Varios valores ya son los de Debian 13; fijarlos evita depender de futuros cambios de defaults.

> **Roles que reenvían paquetes.** Contenedores, virtualización y VPN necesitan `net.ipv4.ip_forward = 1`, que sus demonios activan al arrancar. Si algún archivo lo fija en `0`, un `sysctl --system` con el rol en marcha deja a los contenedores sin red sin que nada falle. Antes de instalar un rol así:
>
> ```bash
> grep -rn "ip_forward" /etc/sysctl.d/ /etc/sysctl.conf 2>/dev/null
> ```
>
> Si aparece en `0`, el manual del rol lo anula en un archivo que se lea después, como `/etc/sysctl.d/99-zz-reenvio.conf`.

#### 13.2 Deshabilitar módulos de kernel innecesarios

**Opcional, recomendado.** Cada módulo cargado es código en el kernel; los buses y sistemas de archivos que un servidor no usa son superficie gratuita.

```bash
# Descargar e instalar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-modprobe.d-99-hardening.conf.txt" \
    -O /tmp/plantilla-etc-modprobe.d-99-hardening.conf.txt && \
sudo install -m 0644 -o root -g root /tmp/plantilla-etc-modprobe.d-99-hardening.conf.txt /etc/modprobe.d/99-hardening.conf
```

`install <modulo> /bin/true` convierte cualquier carga en un éxito vacío, más firme que `blacklist`, que permite la carga explícita.

> **Contenedores.** Su red depende de módulos que una lista genérica podría bloquear, y un módulo bloqueado así no da ningún error. Antes de instalar un motor de contenedores:
>
> ```bash
> grep -nE 'overlay|br_netfilter|bridge|veth|vxlan|ip_vs' /etc/modprobe.d/99-hardening.conf \
>   || echo "✓ ninguno bloqueado"
> ```
>
> Bloquear `udf` impide montar ISOs. En un host de virtualización se comenta esa línea y se anota en el runbook (§22.3).
>
> **`usb-storage`.** En físicos donde se conectan discos de respaldo no se bloquea; en colocation o centro de datos, sí:
>
> ```text
> install usb-storage /bin/true
> ```
>
> En VMs y VPS no aplica.

Los `install` actúan en el siguiente intento de carga. Para descargar un módulo ya cargado:

```bash
sudo lsmod | grep firewire
sudo modprobe -r firewire-core   # si está cargado y no se usa
```

#### 13.3 Protección de GRUB (solo bare metal)

**Opcional:** solo en servidores físicos con consola accesible. En VMs y VPS la consola la controla el hipervisor o el proveedor, y **no se aplica**. Sin protección, editar la línea del kernel con `init=/bin/bash` da una shell de root. Se exige contraseña para **editar** entradas, no para arrancar, para que el servidor pueda reiniciarse sin intervención (WOL incluido).

```bash
# Generar hash de contraseña para GRUB
grub-mkpasswd-pbkdf2
# Pide contraseña dos veces, devuelve un hash que comienza con 'grub.pbkdf2.sha512.10000...'
```

Se crea `/etc/grub.d/40_password`:

```bash
sudoedit /etc/grub.d/40_password
```

```sh
#!/bin/sh
exec tail -n +3 $0
# Este archivo añade un usuario admin al menú de GRUB.
# El usuario admin puede editar entradas; los usuarios normales solo arrancan.
set superusers="admin"
password_pbkdf2 admin <hash-pegado-aqui>
```

```bash
sudo chmod +x /etc/grub.d/40_password
```

**En Debian 13 (grub 2.12) `10_linux` ya no añade `--unrestricted` por su cuenta**: el comportamiento de Debian 11/12 desapareció (no existe la lógica en `10_linux` ni en `grub-mkconfig_lib`). Con `superusers` definido y sin el parche, GRUB exigiría la contraseña también para **arrancar**, lo que rompe los reinicios desatendidos. Se añade `--unrestricted` a las dos líneas `menuentry` de `10_linux`:

```bash
sudo cp /etc/grub.d/10_linux /etc/grub.d/10_linux.ori
sudo sed -i 's/\${CLASS} \\\$menuentry_id_option/\${CLASS} --unrestricted \\\$menuentry_id_option/g' /etc/grub.d/10_linux
grep -n "unrestricted" /etc/grub.d/10_linux
# Resultado esperado: 2 líneas con --unrestricted (kernel y simple)
```

> El parche toca un archivo gestionado por el paquete: un upgrade de `grub-common` puede restaurar el original. Se re-aplica el `sed` y se regenera (o se protege con `dpkg-divert`). La comprobación tras cada upgrade de `grub-*` es el `grep -c` de abajo.

```bash
sudo update-grub
sudo grep -c "unrestricted" /boot/grub/grub.cfg
# Resultado esperado: al menos 3 (simple, kernel y recovery mode)
sudo grep -nE 'superusers|password_pbkdf2' /boot/grub/grub.cfg
```

Cada `menuentry` debe tener `--unrestricted` y las líneas de `superusers` deben estar presentes. Verificación con un reinicio: el arranque normal no pide nada; presionar `e` sobre una entrada pide usuario `admin` y la contraseña elegida.

#### 13.4 Servicios que escuchan en la red sin necesidad

Un servicio que escucha en todas las interfaces sin hacer falta queda expuesto el día que UFW se desactive para un diagnóstico o una regla se abra de más:

```bash
ss -tlnu
```

En un servidor base solo SSH debe aparecer en `0.0.0.0` o `[::]`; lo ligado a `127.0.0.1`, `::1` o `127.0.0.53` no es accesible desde fuera. El caso habitual con `systemd-resolved` es LLMNR, en el 5355 TCP y UDP de todas las interfaces, inútil en un servidor. Se desactiva con mDNS:

```bash
sudo install -d -m 0755 /etc/systemd/resolved.conf.d
printf '[Resolve]\nLLMNR=no\nMulticastDNS=no\n' | sudo install -m 0644 /dev/stdin /etc/systemd/resolved.conf.d/10-sin-llmnr.conf
sudo systemctl restart systemd-resolved
ss -tlnu | grep ':5355' || echo "✓ LLMNR desactivado"
getent hosts deb.debian.org
```

Los permisos van explícitos porque `systemd-resolved` corre como `systemd-resolve`: con la `UMASK 027` de 9.4, un directorio y un archivo creados con `mkdir` y `tee` quedarían ilegibles para él, y LLMNR seguiría activo. La última orden confirma que el DNS sigue funcionando. Sin `systemd-resolved` activo, no aplica.

### 14. Auditoría y detección

Las secciones anteriores previenen; esta deja constancia de lo que ocurre para poder investigar un compromiso. El alcance es modesto: reglas mínimas de auditoría e integridad de paquetes y archivos. Los marcos de cumplimiento (PCI-DSS, CIS nivel 2) tienen sus propios manuales.

#### 14.1 `auditd` — reglas mínimas

**Opcional, recomendado;** fundamental con requisitos de auditoría. Registra eventos del kernel en `/var/log/audit/audit.log`. Es verboso, así que vigila pocos archivos, los importantes.

```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

**Reglas básicas:**

```bash
# Descargar e instalar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-audit-rules.d-99-hardening.rules.txt" \
    -O /tmp/plantilla-etc-audit-rules.d-99-hardening.rules.txt && \
sudo install -m 0640 -o root -g root /tmp/plantilla-etc-audit-rules.d-99-hardening.rules.txt /etc/audit/rules.d/99-hardening.rules
```

**Recargar reglas:**

```bash
sudo augenrules --load
sudo systemctl restart auditd
sudo auditctl -l   # listar reglas activas
```

`augenrules` reubica las directivas globales: el último `-D` como primera línea, el último `-b` como segunda, el último `-f` como tercera y el último `-e` como última, vengan del archivo que vengan. Con `-e 2` las reglas quedan inmutables hasta reiniciar: un `augenrules --load` posterior falla y `auditctl -D` no vacía nada, y es la protección funcionando. Para cambiar reglas se edita el archivo y se reinicia.

La salida de `augenrules --load` empieza con `No rules`, la respuesta del `-D`, y repite `Old style watch rules are slower` por cada `-w`, que siguen siendo válidas. El bloque de estado que importa es el último, con `enabled 2`.

```bash
sudo head -4 /etc/audit/audit.rules   # -D, -b y -f reubicados al inicio
sudo tail -1 /etc/audit/audit.rules   # -e 2 al final
sudo auditctl -s                 # enabled=2, failure=1
```

**Consultas:**

```bash
# Eventos recientes con la clave 'identidad'
sudo ausearch -k identidad -ts recent

# Resumen por usuario
sudo aureport -u --summary

# Resumen de ejecuciones como root por usuarios normales
sudo ausearch -k root_exec | sudo aureport -x
```

**Retención.** Debian trae `max_log_file = 8` (MB) y `num_logs = 5`, unos 40 MB. Con `root_exec`, cada orden lanzada con `sudo` registra todo lo que ejecuta, y una tarde de `apt install` rota los 40 MB en minutos y borra el registro del despliegue.[^audit-rotacion] Para 320 MB (32 MB × 10):

```bash
sudo grep -E '^(max_log_file|num_logs|max_log_file_action) ' /etc/audit/auditd.conf
sudo sed -i -E 's/^max_log_file = .*/max_log_file = 32/; s/^num_logs = .*/num_logs = 10/' /etc/audit/auditd.conf
sudo systemctl restart auditd
sudo grep -E '^(max_log_file|num_logs) ' /etc/audit/auditd.conf
```

`max_log_file_action` debe seguir en `ROTATE`, o `num_logs` no se aplica. El total se ajusta al espacio de `/var/log`, porque al llenarse actúan `space_left_action` y `disk_full_action`. El `sed` se puede repetir. `auditd.conf` no admite drop-ins: es un conffile, y como `journald.conf` en 18.1.1, `debsums -ac` (14.2) lo lista y una versión nueva del archivo hace que `unattended-upgrades` aparte `auditd`.

[^audit-rotacion]: En un VPS de referencia, las instalaciones de las secciones 19 y 20 provocaron cuatro rotaciones en 18 minutos. Al terminar el despliegue, el registro más antiguo que quedaba era del mediodía de ese mismo día, posterior a casi todo el hardening.

#### 14.2 `debsums` — integridad de paquetes

**Opcional, recomendado.** Compara los archivos instalados con los hashes de sus paquetes.

```bash
sudo apt install -y debsums
```

**Escaneo manual:**

```bash
sudo debsums -ac
```

`-a` incluye los archivos de configuración y `-c` muestra solo los que difieren. Siguiendo este manual, salen solo los modificados a propósito:

```text
/etc/security/faillock.conf
/etc/security/pwquality.conf
/etc/login.defs
/etc/audit/auditd.conf
```

El primero es de 9.3.3, los dos siguientes de 9.4 y el último de la retención de 14.1. Los archivos de `/etc/default/` que se editan más adelante (`debsums` aquí mismo y `aide` en 14.3) es de esperar que aparezcan también en ejecuciones posteriores, pendiente de validación. Cualquier otro merece revisión.

> `sshd_config` y los `common-*` de PAM no aparecen porque no son conffiles registrados por `dpkg`: el primero lo gestiona `ucf` y los segundos `pam-auth-update`. Los vigila AIDE (14.3).

**Escaneo automático:**

```bash
sudoedit /etc/default/debsums
```

```text
CRON_CHECK=daily
```

El job de `/etc/cron.daily/debsums` usa `debsums -cs`, sin `-a`: solo escribe, y por tanto solo genera correo, cuando cambia un archivo de paquete. Los conffiles quedan para el `debsums -ac` manual y para AIDE.

#### 14.3 AIDE — integridad de archivos del sistema

**Opcional, recomendado.** Toma una foto criptográfica del sistema (hashes, permisos, propietarios) y avisa de cualquier cambio posterior, como un binario sustituido o una configuración alterada. A cambio, la base se rehace tras cada cambio legítimo; el caso más pesado es un point release (16.9). Un informe con miles de diferencias se ignora, y un control ignorado no existe.

##### Instalación

```bash
sudo apt install -y aide aide-common
```

`aide` es el binario; `aide-common` trae `/etc/aide/aide.conf`, `aideinit` y `dailyaidecheck`.

> **Contenedores.** Sus capas e imágenes cambian en cada despliegue y llenan el informe. Se excluyen antes de generar la base:
>
> ```bash
> printf '%s\n' '!/var/lib/docker' '!/var/lib/containerd' \
>   | sudo install -m 0644 -o root -g root /dev/stdin /etc/aide/aide.conf.d/70_local_contenedores
> sudo aide --config /etc/aide/aide.conf --config-check && echo "✓ configuración válida"
> ```
>
> El nombre del archivo solo admite letras, números, guiones y guiones bajos; con un punto, AIDE lo ignora sin avisar. La integridad de los contenedores se controla fijando cada imagen por su resumen, en el manual del rol.

##### Exclusión del ruido propio de este servidor

Debian ya trata como volátiles casi todas las rutas de `/var/log` y `/run`, pero no cinco que genera este manual ni el estado del cliente DHCP:

```bash
printf '%s\n' '!/var/log/sudo-io' '!/var/log/audit' '!/var/log/aide/aideinit\.log$' '!/var/log/sysstat' \
  | sudo install -m 0644 -o root -g root /dev/stdin /etc/aide/aide.conf.d/70_local_logs
printf '%s\n' '!/run/faillock' '!/run/dhcpcd' \
  | sudo install -m 0644 -o root -g root /dev/stdin /etc/aide/aide.conf.d/70_local_run
sudo aide --config /etc/aide/aide.conf --config-check && echo "✓ configuración válida"
```

- `/var/log/sudo-io`: un directorio nuevo por cada `sudo` (9.3.1), con retención propia en 18.2.1.
- `/var/log/audit`: crece sin parar y lo protege `auditd` (14.1).
- `/var/log/aide/aideinit.log`: se escribe después de la foto y, con `COPYNEWDB=no`, saldría todos los días. El punto escapado y el `$` limitan la exclusión a ese archivo, y los informes del mismo directorio siguen vigilados.
- `/var/log/sysstat`: la recogida histórica de 19.1 reescribe el archivo del día cada 10 minutos.
- `/run/faillock`: `pam_faillock` (9.3.3) lo actualiza en cada `sudo`.
- `/run/dhcpcd`: cuando la concesión DHCP la gestiona dhcpcd (`/etc/resolv.conf` empieza con `# Generated by dhcpcd`), cada renovación reescribe su estado. Sin dhcpcd, la exclusión no tiene efecto.

AIDE vigila también `/tmp`, `/root` y `/home`: un archivo de trabajo que exista al crear la base y desaparezca después, como un script o un informe guardado, sale en el informe como añadido o eliminado. Mientras se crea la base no se dejan archivos de trabajo en esas rutas, o se acepta esa diferencia en el primer check.

##### Inicialización de la base de datos

Recorre todo el sistema y tarda **entre 2 y 10 minutos**:

```bash
sudo aideinit
```

Salida esperada:

```text
Running aide --init...
```

La base queda en `aide.db.new` y se renombra para usarla como referencia:

```bash
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

> Con `gzip_dbout=yes`, la base sale comprimida aunque no lleve `.gz`; `sudo file /var/lib/aide/aide.db` dice `gzip compressed data`.

##### Verificación de la base de datos

```bash
sudo ls -la /var/lib/aide/
```

Salida esperada:

```text
drwx------  2 _aide root      4096 set 12 21:25 .
drwxr-xr-x 28 root  root      4096 set 12 21:19 ..
-rw-------  1 _aide _aide 12761927 set 12 21:25 aide.db
```

Directorio y base son de `_aide`, sin shell, con `700` y `600`.

##### Ejecución del primer check

```bash
sudo aide --config /etc/aide/aide.conf --check
```

Tarda lo mismo que la inicialización y parece colgado; no se interrumpe.

##### Salida esperada

**A) Sin diferencias:**

```text
AIDE 0.19.1 found NO differences between database and filesystem. Looks okay!!
```

**B) Con diferencias menores, habituales en la primera corrida:**

```text
Summary:
  Total number of entries:    XXXXX
  Added entries:              X
  Removed entries:            0
  Changed entries:            Y
```

En el despliegue de referencia, sin las exclusiones todavía, salieron 21 añadidas, 1 eliminada y 8 cambiadas, todas cubiertas por las exclusiones salvo esta:

```text
f------------------: /var/lib/aide/aide.db.new
```

Con las exclusiones aplicadas desde el principio (validado en VM), el primer check sale con 0 añadidas, 1 eliminada y unas pocas cambiadas, sin ninguna ruta de las excluidas. La eliminada es la misma de arriba: `aide --init` retrata a medias el `aide.db.new` que está escribiendo, el `mv` lo quita de encima y el check lo reporta. Tras regenerar (abajo), el check inmediato tampoco es limpio del todo: los timers de `apt` dejan stamps (`/var/lib/apt/periodic`, `daily_lock`, `/run/unattended-upgrades.lock`); es ruido operativo, no fallo de las exclusiones. Cualquier otra ruta, sobre todo en `/etc`, `/usr` o `/boot`, se revisa antes de regenerar, porque una base nueva sella como bueno lo que haya.

##### Regenerar el baseline tras el primer check

```bash
sudo aideinit -y -f
sudo aide --config /etc/aide/aide.conf --check
```

`-y` sobrescribe un `aide.db.new` previo y `-f` lo copia sobre `aide.db`. `aideinit`, el envoltorio de Debian, ejecuta AIDE como `_aide`, igual que el check diario, y registra en `/var/log/aide/aideinit.log`.

```text
-rw-------  1 root  root  11094453 oct 17 10:15 aide.db
-rw-------  1 _aide _aide 11094453 oct 17 10:15 aide.db.new
```

Quedan los dos archivos iguales, y `aide.db` pasa a root porque `aideinit` lo copia como root (con aide 0.19.1-2+deb13u5, en un VPS de Contabo, quedó de `_aide`; en ambos casos el check diario lo lee). El check diario lo lee igual gracias a `CAP_DAC_READ_SEARCH` y, con `COPYNEWDB=no`, nunca lo escribe. Desde aquí, toda diferencia es un cambio real.

Para probar un cambio sin esperar al check diario, `sudo systemctl start dailyaidecheck.service` lo lanza a mano. La unidad trae `StartLimitBurst=1` en 2 horas: un segundo arranque en ese plazo falla con `start-limit-hit` (se limpia con `sudo systemctl reset-failed dailyaidecheck.service`).

##### Programación del check diario automático

Lo lanza `dailyaidecheck.timer`, habilitado por el paquete. El `/etc/cron.daily/dailyaidecheck` termina sin hacer nada en sistemas con systemd, para no duplicar el check:

```bash
systemctl list-timers dailyaidecheck.timer
```

Debe aparecer la próxima ejecución:

```text
Sun 2026-10-18 02:15:07 -05       16h -                                      - dailyaidecheck.timer         dailyaidecheck.service
```

El timer está a las 01:50 con `RandomizedDelaySec=2h` y `Persistent=true`, así que la hora varía y un servidor apagado lo ejecuta al arrancar. `CRON_DAILY_RUN` viene comentada y vale `yes`; solo se descomenta para desactivar el check.

El servicio corre como `_aide`, con `COMMAND=update` y `COPYNEWDB=no`: compara contra `aide.db`, deja `aide.db.new` y escribe el informe completo en `/var/log/aide/aide.log`, rotado cada día; en el journal solo constan el inicio y el final de cada ejecución. `QUIETREPORTS=yes` limita el correo a los días con diferencias o errores; un informe diario que casi siempre dice «sin cambios» se deja de leer:

```bash
sudo sed -i 's|^#\?QUIETREPORTS=.*|QUIETREPORTS=yes|' /etc/default/aide
grep -n '^QUIETREPORTS' /etc/default/aide
```

```text
21:QUIETREPORTS=yes
```

`^#\?` cubre la línea comentada y la ya descomentada, y no hace falta reiniciar nada.

##### Envío del informe por correo

El envoltorio de `aide-common` solo llama a `mail` corriendo como root o desde cron: el servicio systemd corre como `_aide` y, sin ajustar nada, el informe no sale — en el journal queda `WARN: it is not possible to use mail(1) unless aide is run as root or as non-root with added capabilities` y el reporte solo existe en `/var/log/aide/aide.log`. Dos ajustes lo ponen en marcha (validado en VM con Gmail como relay). Se aplican después de la sección 20: el grupo `msmtp` lo crea la instalación de msmtp, y una unidad con `SupplementaryGroups` de un grupo inexistente no arranca:

```bash
# 1. Que el envoltorio use mail(1) sin comprobar el uid
grep -q '^MAILCMD=' /etc/default/aide || sudo tee -a /etc/default/aide >/dev/null <<'EOF'

# Enviar el informe con mail(1) aunque el servicio corre como _aide
MAILCMD="/usr/bin/mail"
EOF

# 2. Grupo suplementario para el servicio: msmtp comprueba que puede leer
#    /etc/msmtprc con el uid real del proceso — ignora la CAP_DAC_READ_SEARCH
#    de la unidad — y sin el grupo no encuentra la cuenta y falla con exit 78
sudo install -d -m 0755 /etc/systemd/system/dailyaidecheck.service.d
printf '[Service]\nSupplementaryGroups=msmtp\n' \
  | sudo install -m 0644 -o root -g root /dev/stdin \
      /etc/systemd/system/dailyaidecheck.service.d/grupo-msmtp.conf
sudo systemctl daemon-reload
```

La plantilla de `/etc/msmtprc` (20.4) trae además, desde la v1.2, `allow_from_override off`, que fija el remitente del sobre en el `from` de la cuenta aunque el programa que llama proponga otro a partir del usuario que envía, aquí `_aide`. Con `bsd-mailx` como cliente `mail` (20.7), el informe sale como `_aide` y llega con ese `from` (validado en un VPS de Contabo el 2026-11-19).

El destinatario es `root`, que `/etc/msmtp/aliases` traduce a `${MIDESTINO}` (20.6), y el correo empieza a salir cuando la sección 20 esté configurada; hasta entonces, y ante cualquier fallo posterior, el reporte completo sigue en `/var/log/aide/aide.log`. El envío queda registrado con `smtpstatus=250` en `journalctl -t msmtp` (20.9).

##### Mantenimiento tras cambios legítimos

Tras actualizar paquetes, cambiar la configuración o instalar software, el siguiente check muestra diferencias. Validadas, se regenera la base:

```bash
sudo aideinit -y -f
```

Se guarda además una copia de la base fuera del servidor o en un medio de solo lectura: un atacante con root puede alterar binarios y base a la vez.

#### 14.4 Revisión periódica de logs y registros

En Debian 13 `last` está en el paquete `wtmpdb`, con la base en `/var/log/wtmp.db`. `sshd` registra sus sesiones por su cuenta, y `libpam-wtmpdb` añade `su` y la consola con `skip_if=sshd` para no duplicarlas:

```bash
sudo apt install -y wtmpdb libpam-wtmpdb
diff <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth.faillock) \
     <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth) && echo "✓ pila de faillock intacta"
```

Al ser un `libpam-*`, su postinst ejecuta `pam-auth-update --package`. Con los perfiles de 9.3.3 no hay diálogo: la pila se regenera con las líneas de faillock en su sitio. El perfil de `wtmpdb` solo aporta líneas de sesión, así que `common-auth` no cambia y el `diff` de arriba debe salir limpio; si algún día marca una línea `auth` nueva, es un módulo legítimo recién instalado, y lo que no puede faltar son las tres líneas de `pam_faillock`.

```bash
# Últimos logins exitosos
last -a | head -20

# Logins exitosos
sudo wtmpdb last --fulltimes --service -n 20

# Intentos fallidos
sudo journalctl SYSLOG_FACILITY=10 -p warning --since '7 days ago' | tail -30

# Eventos SSH específicos de los últimos 7 días
sudo journalctl -u ssh --since '7 days ago' | grep -iE 'fail|invalid|preauth' \
  | grep -v 'Postponed publickey' | tail -20

# Warnings y errores del sistema del día actual
sudo journalctl --since today -p warning..emerg --no-pager
```

El segundo `grep` de la consulta de SSH quita los `Postponed publickey ... [preauth]`, un paso normal de cada inicio con llave que, sin filtrar, ocupaba 16 de 20 líneas. Lo que queda son conexiones cerradas antes de autenticarse, `Too many authentication failures` y escáneres que no completan el banner (`banner exchange: ... invalid format`).

#### 14.5 `lynis` — auditoría periódica del baseline de seguridad

**Opcional, recomendado** al menos una vez tras el despliegue. Audita la configuración contra buenas prácticas y da una puntuación con sugerencias; sirve para detectar derivas.

```bash
sudo apt install -y lynis
```

**Primera auditoría:**

```bash
sudo lynis audit system
```

`Hardening index` va de 0 a 100. El índice y el número de sugerencias varían entre despliegues: con el manual aplicado hasta 14.4 se obtuvieron `Great, no warnings`, índice 81 y 29 sugerencias, y hasta la sección 15, también sin warnings, 79 y 30. La puntuación no es un objetivo; conviene saber qué sugerencias no se persiguen.

No aplican, porque el manual ya lo cubre o lynis lo lee mal:

- SSH-7408 sobre `AllowTcpForwarding`, `X11Forwarding` y `AllowAgentForwarding`: `DisableForwarding yes` (10.4) prevalece sobre todas.
- FINT-4402, SHA256 o SHA512 en AIDE: `Checksums = H` los incluye.
- KRNL-5788, falta `/vmlinuz`: las imágenes cloud traen `do_symlinks = no`; los kernels están en `/boot`.
- HRDN-7222, compilador instalado: suele ser `as`, de `binutils`, llegado como dependencia; `dpkg -S /usr/bin/as` dice de dónde.
- "Non-native binary formats": el `binfmt` de `python3.13` para los `.pyc`.
- AUTH-9230, rondas de hash: `SHA_CRYPT_*` solo afecta a SHA-512, y Debian 13 usa yescrypt.
- BOOT-5180 y FIRE-4513: informativas.
- HRDN-7222 solo aparece si hay compilador; en una imagen cloud sin `binutils` no sale.
- `[LYNIS]`, versión antigua: ver más abajo.
- DEB-0831 y PKGS-7420, needrestart y actualizaciones automáticas: DEB-0831 desaparece al instalar needrestart, opcional en 8.4, y PKGS-7420 con la sección 16; antes de eso, las dos aparecen.

Se descartan a propósito:

- SSH-7408, `MaxSessions 2`: con `ControlMaster` (10.4) la tercera terminal fallaría.
- SSH-7408, `TCPKeepAlive no`: `ClientAlive*` ya detecta las conexiones muertas.
- FILE-7524, permisos 600/700 en `grub.cfg`, `crontab`, `sshd_config` y `/etc/cron.*`: no tienen secretos, y `update-grub` y `cron` podrían revertirlos (no comprobado).
- BOOT-5122 y FILE-6310, contraseña de GRUB y particiones propias: no compensan en un VPS de disco único.
- BOOT-5264, endurecer unidades con `systemd-analyze security`: trabajo por servicio, de cada manual de rol.
- DEB-0280, `libpam-tmpdir`: otro `libpam-*` por una mejora pequeña.
- USB-1000: un VPS no tiene USB.
- LOGG-2154, ACCT-9622, ACCT-9626, TOOL-5002, BANN-7126, BANN-7130, DEB-0810 y PKGS-7394: dependen del rol o de la política del cliente.

Aplicadas: `UMASK 027` (AUTH-9328, 9.4) y `ldisc_autoload` y `bpf_jit_harden` (KRNL-6000, 13.1).

**Auditoría mensual:**

```bash
sudo install -m 0700 -o root -g root /dev/stdin /etc/cron.monthly/lynis-audit <<'EOF'
#!/bin/bash
/usr/sbin/lynis audit system --quiet --cronjob 2>&1 \
  | { printf 'To: root\nSubject: Lynis audit %s %s\n\n' "$(hostname)" "$(date +%Y-%m)"; cat; } \
  | /usr/sbin/sendmail -t
EOF
```

El informe llega por correo a root y, con la sección 20, al destino externo. El paquete instala además un `lynis.timer` diario que deja el resultado en el log sin enviar nada; sobra junto al cron mensual:

```bash
sudo systemctl disable --now lynis.timer
systemctl is-enabled lynis.timer   # debe devolver "disabled"
```

> No se usa `mail -s asunto root`, aunque con `bsd-mailx` (20.7) funcionaría: el cron depende así solo de `sendmail`, que siempre existe con `msmtp-mta`, y no del cliente `mail` que haya instalado. Se construyen las cabeceras `To:` y `Subject:` y se pasan a `/usr/sbin/sendmail -t`, con ruta completa porque `/usr/sbin` no está en el `$PATH` de cron.

**Versión de lynis:**

```bash
sudo lynis update info
```

```text
  Version            : 3.1.4
  Status             : Outdated
  Installed version  : 314
  Latest version     : 316
```

`Outdated` es lo esperable: Debian estable mantiene la versión de trixie con correcciones de seguridad, y lynis se compara con la última de CISOfy; de ahí también la sugerencia `[LYNIS]`. Se actualiza con el sistema (16.9). lynis no modifica nada, solo audita.

### 15. AppArmor

AppArmor confina procesos por la ruta del binario: cada perfil declara qué puede leer, escribir y ejecutar un programa. En Debian 13 viene activo, pero en un servidor base no confina nada en marcha: sus perfiles son de escritorio y de herramientas sueltas, y los servicios de este manual (`sshd`, `fail2ban`, `cron`, `auditd`) no tienen perfil. El confinamiento llega con los roles, como Docker con `docker-default` o libvirt con cada VM. Aquí se confirma que está activo y se dejan las herramientas listas.

#### 15.1 Verificación del estado

```bash
sudo aa-status
```

Salida en el despliegue de referencia, recortada:

```text
apparmor module is loaded.
106 profiles are loaded.
7 profiles are in enforce mode.
   /usr/bin/man
   lsb_release
   ...
23 profiles are in complain mode.
   ...
76 profiles are in unconfined mode.
   ...
0 processes have profiles defined.
0 processes are in enforce mode.
0 processes are in complain mode.
```

- `enforce mode`: el perfil bloquea lo que no permite.
- `complain mode`: registra las violaciones sin bloquear.
- `unconfined mode`: perfiles con `flags=(unconfined)`, como `firefox` o `runc`, que AppArmor 4 usa para permitir espacios de nombres de usuario. No restringen nada.
- `processes have profiles defined`: lo que dice si algo está confinado. En el servidor base es 0. Para un servicio concreto, `cat /proc/$(pgrep -o -x sshd)/attr/current`.

Si AppArmor no está cargado:

```bash
sudo systemctl status apparmor --no-pager
sudo systemctl enable --now apparmor
```

Y en el kernel:

```bash
cat /sys/kernel/security/lsm
```

La lista debe incluir `apparmor`.

#### 15.2 Perfiles en `enforce` vs `complain`

`aa-complain`, `aa-enforce` y el resto vienen en `apparmor-utils`:

```bash
sudo apt install -y apparmor-utils
```

```bash
sudo aa-status | grep -A100 'profiles are in'
```

**A complain, para depurar:**

```bash
export PERFIL=usr.bin.msmtp   # archivo del perfil en /etc/apparmor.d/
sudo aa-complain "/etc/apparmor.d/${PERFIL:?}"
```

**De vuelta a enforce:**

```bash
sudo aa-enforce "/etc/apparmor.d/${PERFIL:?}"
```

**Recargar todos:**

```bash
sudo systemctl reload apparmor
```

#### 15.3 Perfiles adicionales

`apparmor-profiles` y `apparmor-profiles-extra` solo se instalan si el rol ejecuta alguno de sus programas; en un servidor genérico añaden perfiles sin confinar nada (en el despliegue de referencia, de 106 a 135 perfiles con `0 processes have profiles defined`). Para comprobar si alguno corresponde a un binario del servidor:

```bash
apt-get download apparmor-profiles apparmor-profiles-extra
for d in apparmor-profiles*.deb; do dpkg -c "$d" | grep -o 'etc/apparmor.d/[^/]\+$'; done
rm -f apparmor-profiles*.deb
```

Si se instalan, tras unos días de uso se buscan denegaciones:

```bash
if systemctl is-active --quiet auditd; then
  sudo grep -h 'apparmor="DENIED"' /var/log/audit/audit.log* | tail -50
else
  sudo journalctl -k | grep 'apparmor="DENIED"' | tail -50
fi
```

Con `auditd` activo, el kernel entrega los mensajes de AppArmor a `/var/log/audit/audit.log` y no los escribe en `journalctl -k`, donde solo quedan los del arranque previo a `auditd`. Un perfil en `complain` sin denegaciones legítimas puede pasar a enforce. Si se instalaron sin necesidad, se purgan y se descargan del kernel los perfiles que quedaron sin archivo:

```bash
sudo aa-remove-unknown -n   # dry run: solo lista lo que descargaría
sudo aa-remove-unknown      # los descarga, imprimiendo Removing '<perfil>'
```

Descarga todo perfil cargado cuyo archivo desapareció y excluye los dinámicos de libvirt (`libvirt-*`). El `-n` se corre primero para ver la lista sin actuar (validado en VM: con el archivo presente no lista nada, y sin él lista exactamente ese perfil). Los perfiles propios, con `aa-genprof`, son trabajo del manual del rol.

### Cierre de la Etapa III

- **Acceso (SSH):** puerto cambiado, solo llave pública, usuarios restringidos, criptografía moderna.
- **Red (UFW y proveedor):** denegación por defecto, solo SSH abierto, con rate limiting.
- **Fuerza bruta (fail2ban):** baneo por patrones de log, creciente para reincidentes.
- **Kernel (sysctl, modprobe):** anti-spoofing, ASLR, módulos innecesarios bloqueados.
- **Auditoría (auditd, debsums, AIDE):** registro y verificación de integridad.
- **MAC (AppArmor):** activo y listo para los perfiles que traigan los roles.

Al terminar el despliegue base, UFW y el firewall del proveedor admiten solo SSH. La restricción de los puertos web a un proxy de borde se aplica después, con la sección 17 del manual de Caddy. La Etapa IV mantiene este estado en el tiempo.

Antes de pasar a la Etapa IV se revisa el check de AIDE y se regenera la base (14.3, «Mantenimiento tras cambios legítimos»): los paquetes de 14.4, 14.5 y 15.2 se instalaron después de crearla y aparecen como cientos de entradas añadidas.

---

## Etapa IV — Mantenimiento y actualizaciones

Un servidor endurecido el primer día deja de estarlo en pocos meses si no se actualiza, si su reloj deriva, si los logs llenan el disco, si nadie se entera de una caída o si un fallo llega sin respaldo. Esta etapa lo mantiene: actualizaciones, hora, logs, monitoreo mínimo, correo, respaldos y documentación. La observabilidad avanzada y los respaldos empresariales son de los manuales de rol. Cada automatismo deja rastro en logs o en correo, el reinicio automático va en una ventana horaria predecible, y la restauración se prueba antes de necesitarla.

### 16. Actualizaciones automáticas

Actualizar a mano no escala: quien lo olvida unas semanas tiene un servidor con CVE sin parchear durante semanas. `unattended-upgrades` aplica solo los orígenes indicados, admite exclusiones, avisa por correo y puede reiniciar en una ventana.

#### 16.1 Instalación

```bash
sudo apt install -y unattended-upgrades apt-listchanges
```

Algunas imágenes los traen instalados y con el flujo ya activo (sección 5); entonces lo que sigue modifica un flujo que puede haber aplicado ya actualizaciones con los orígenes de serie. Se comprueba:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
sudo tail -n 5 /var/log/unattended-upgrades/unattended-upgrades.log
```

Si `unattended-upgrades` aún no se ha ejecutado, el log no existe y `tail` falla sin que sea un error; en la imagen de Contabo, `20auto-upgrades` trae solo dos líneas, que 16.5 completa.

`apt-listchanges` (con `which=news`, su valor de serie) envía las entradas de `NEWS.Debian`, donde los mantenedores avisan de cambios incompatibles, en la misma ejecución y segundos antes de instalar. No da margen para frenar nada, pero deja constancia de qué cambió.

#### 16.2 Configuración de orígenes

De serie, `/etc/apt/apt.conf.d/50unattended-upgrades` habilita el repositorio principal y seguridad y deja comentado `-updates`. Ese archivo no se edita: lo propio del servidor va en `/etc/apt/apt.conf.d/52unattended-upgrades-local`, que APT lee después. Las listas como `Origins-Pattern` se suman entre archivos, y en los valores simples (`Mail`, `Automatic-Reboot`) prevalece el del archivo que se lee después. Este manual añade `-updates`:

```bash
sudo install -m 0644 -o root -g root /dev/stdin /etc/apt/apt.conf.d/52unattended-upgrades-local <<'EOF'
// Configuración local de unattended-upgrades (manual de Debian 13 server, 16.2 a 16.4).
// 50unattended-upgrades queda como lo instala el paquete.
Unattended-Upgrade::Origins-Pattern {
        "origin=Debian,codename=${distro_codename}-updates";
};
EOF
```

Con las tres líneas que trae el `50`, la lista efectiva queda así:

```text
Unattended-Upgrade::Origins-Pattern {
    "origin=Debian,codename=${distro_codename},label=Debian";
    "origin=Debian,codename=${distro_codename},label=Debian-Security";
    "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
    "origin=Debian,codename=${distro_codename}-updates";
};
```

La primera es `trixie` (correcciones entre point releases), las dos siguientes `trixie-security` (dos formatos posibles de la metadata) y la cuarta `trixie-updates`. Con `-updates` el servidor recibe también correcciones funcionales, a cambio de más cambios entre reinicios. Donde pese más la estabilidad, basta con seguridad y el bloque de `Origins-Pattern` del `52` sobra. En el manual base no se excluye ningún paquete; si un rol lo necesita, lo hace en `Unattended-Upgrade::Package-Blacklist`, en el mismo `52`. La configuración efectiva, que suma los dos archivos, se lee con:

```bash
apt-config dump | grep -E '^Unattended-Upgrade'
```

> **ATENCIÓN.** `50unattended-upgrades` no es un conffile de dpkg, a diferencia de `journald.conf` (18.1.1): el paquete lo copia desde `/usr/share/unattended-upgrades/50unattended-upgrades` y lo gestiona `ucf` con fusión a tres vías, así que `debsums` no lo vigila y `ucfq /etc/apt/apt.conf.d/50unattended-upgrades` dice si está modificado (`Changed`). Si se edita, una versión nueva del paquete obliga a `ucf` a fusionar los cambios o a conservar el archivo local y dejar la versión nueva al lado; sin modificar, lo actualiza sin preguntar. Por eso lo propio va en el `52`. En un servidor que ya tiene editado el `50`, se pasa lo propio al `52` y se devuelve el `50` a la plantilla del paquete con `sudo install -m 0644 /usr/share/unattended-upgrades/50unattended-upgrades /etc/apt/apt.conf.d/`; `apt-config dump` debe mostrar los mismos valores antes y después.[^uu-local]

[^uu-local]: Validado el 2026-11-20 en un VPS de Contabo con unattended-upgrades 2.12: con el `50` devuelto a la plantilla (misma suma MD5 que la de `/usr/share/unattended-upgrades/`) y el `52` con `-updates`, el correo y `Automatic-Reboot "false"`, `apt-config dump` muestra los mismos orígenes y valores que con el `50` editado, `ucfq` pasa a `Changed No` y `unattended-upgrade --dry-run -d` lista las cuatro fuentes permitidas, `trixie-updates` incluida.

> **Los repositorios de terceros quedan fuera.** Un paquete de un repositorio externo (motor de contenedores, proxy inverso, base de datos del fabricante) no se actualiza solo, ni siquiera con parches de seguridad. Como varios reinician su servicio al actualizarse, se decide por paquete en el manual del rol: añadir su origen a la lista o anotar su actualización como tarea periódica con responsable. Los orígenes se leen así:
>
> ```bash
> apt-cache policy | grep -oE 'o=[^,]+' | sort -u
> ```
>
> Cada `o=<valor>` se añade como `"origin=<valor>";`.

#### 16.3 Notificación por correo

En el `52unattended-upgrades-local` de 16.2:

```bash
grep -q '^Unattended-Upgrade::Mail ' /etc/apt/apt.conf.d/52unattended-upgrades-local \
  || sudo tee -a /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
Unattended-Upgrade::Mail "root";
Unattended-Upgrade::MailReport "on-change";
EOF
```

`MailReport "on-change"` envía solo cuando hubo actualizaciones o errores (`always` es ruidoso y `only-on-error` demasiado silencioso). Hasta configurar la sección 20, el correo queda en `/var/mail/root`.

#### 16.4 Reinicio automático controlado

Kernel, libc, OpenSSL o systemd solo se aplican al reiniciar; hasta entonces sigue corriendo el código vulnerable. También en el `52`:

```bash
grep -q '^Unattended-Upgrade::Automatic-Reboot ' /etc/apt/apt.conf.d/52unattended-upgrades-local \
  || sudo tee -a /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "07:30";
EOF
apt-config dump | grep -E '^Unattended-Upgrade::(Mail|Automatic-Reboot)'
```

`Automatic-Reboot-WithUsers "false"` pospone el reinicio si hay sesiones, pero la espera es silenciosa e indefinida: cualquier sesión abierta lo bloquea noche tras noche, sea una consola que quedó con la sesión iniciada, una conexión SSH olvidada, la conexión maestra de `ControlPersist` (10.4) mientras dura o una sesión de `tmux` desasociada (6.4), que deja registro en `utmp`. `who` las muestra.

La hora va después de la actualización: `apt-daily-upgrade.timer` dispara a las 06:00 con hasta 60 minutos de retardo, y el reinicio espera a la siguiente `Automatic-Reboot-Time`. Con 04:00 se iría a la madrugada siguiente, con el kernel nuevo instalado y el viejo en marcha veintidós horas; con 07:00 pasa a veces; 07:30 deja margen. La hora es la de la zona de 7.3. Si un rol no admite el reinicio nocturno, se pone `Automatic-Reboot "false"` en el `52` y su manual documenta cuándo se reinicia.

Reinicio pendiente:

```bash
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs
```

El segundo comando dice qué paquetes lo piden.

#### 16.5 Habilitación del flujo periódico

`apt-daily.timer` actualiza las listas y descarga; `apt-daily-upgrade.timer` aplica lo que autoriza `unattended-upgrades`. Ambos necesitan `20auto-upgrades`:

```bash
sudoedit /etc/apt/apt.conf.d/20auto-upgrades
```

Contenido:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

Listas a diario, descarga, limpieza de la caché cada 7 días y aplicación diaria. `sudo dpkg-reconfigure -plow unattended-upgrades` crea el mismo archivo.

#### 16.6 Verificación de los timers

```bash
sudo systemctl status apt-daily.timer --no-pager
sudo systemctl status apt-daily-upgrade.timer --no-pager
```

Ambos `active (waiting)` con su `Trigger:`.

```bash
sudo systemctl list-timers --all
```

#### 16.7 Simulación antes de la primera ejecución real

```bash
sudo unattended-upgrades --dry-run --debug 2>&1 | less
```

Muestra qué se actualizaría, qué se omitiría y por qué, y si habría reinicio.

> La simulación escribe también en `/var/log/unattended-upgrades/unattended-upgrades.log`, con líneas `DEBUG` y `The list of kept packages can't be calculated in dry-run mode`, que no son de una ejecución real. Los `INFO` sobre `powermgmt-base` y `python3-gi` salen siempre y no aplican a un servidor.

#### 16.8 Revisión de logs tras la primera ejecución

Tras la siguiente activación de `apt-daily-upgrade.timer`:

```bash
sudo less /var/log/unattended-upgrades/unattended-upgrades.log
sudo less /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
```

El primero registra las decisiones; el segundo, la salida de `dpkg`.

#### 16.9 Qué hacer después de un point release

**Opcional, recomendado.** Un point release consolida lo que ya venía por `trixie-security` y `trixie-updates`, así que casi todo está instalado cuando se anuncia. Quedan varias tareas que ningún automatismo hace.

**Completar la actualización.** Un cambio de ABI del kernel crea un paquete nuevo en lugar de actualizar el existente, y solo `full-upgrade` lo instala:

```bash
sudo apt update
sudo apt full-upgrade
cat /etc/debian_version      # debe reflejar el point release nuevo
```

Antes, espacio en `/boot`: uno pequeño se llena con tres kernels y `update-initramfs` falla a medias:

```bash
df -h /boot
dpkg -l 'linux-image*' | grep ^ii
```

**Reiniciar, o al menos los servicios.** `glibc`, `openssl` o `systemd` nuevos no afectan a los procesos ya en marcha:

```bash
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs
cat /run/systemd/shutdown/scheduled 2>/dev/null   # ¿hay apagado ya programado?
sudo needrestart -u NeverRestart -r l             # listar sin actuar
sudo needrestart                                  # resolver
uname -r                                          # comparar con dpkg -l
```

Si `uname -r` es anterior al kernel más nuevo, se reinicia y `needrestart` sobra.

**Comprobar PAM.** Un `libpam-*` nuevo pasa por `pam-auth-update` y puede deshacer faillock (§9.3.3):

```bash
grep -E ' (libpam-[a-z0-9-]+|libpam-runtime):' /var/log/dpkg.log | grep ' upgrade \| install '
diff <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth.faillock) \
     <(grep -vE '^\s*#|^\s*$' /etc/pam.d/common-auth) && echo "✓ pila de faillock intacta"
```

Si hay diferencias, se corrige según la ATENCIÓN de §9.3.3 y se repiten sus pruebas.

**Rehacer la base de AIDE, después del reinicio.** El point release recompila binarios contra la `glibc` nueva, cambian sus hashes y la base de §14.3 queda desfasada: sin rehacerla, el informe trae miles de diferencias y se deja de leer.

```bash
sudo debsums -c                                       # 1. confirmar que nada quedó inconsistente
sudo aideinit -y -f                                   # 2. nueva base de AIDE (§14.3)
```

`debsums -c` va primero, porque una base nueva sobre un sistema alterado sella el compromiso. Y todo después del reinicio, nunca con el sistema a medio actualizar. Al día siguiente:

```bash
sudo aide --check | tail -20
```

**Revisar el anuncio.** En <https://www.debian.org/News/>, buscar los paquetes del manual (`openssh`, `sudo`, `auditd`, `chrony`, `fail2ban`, `apparmor`, `msmtp`) por si cambió su comportamiento:

```bash
timedatectl | grep -i "time zone"   # confirmar que tzdata no movió la zona
sudo journalctl -p warning -b | tail -30
```

**En una VM, parchear también el anfitrión**, con su propio ciclo: los point releases suelen corregir `qemu` y `libvirt`.

### 17. Sincronización de tiempo

Un reloj desviado rompe certificados, autenticaciones y la correlación de logs, y hace que fail2ban vea eventos "del futuro". Debian 13 trae `systemd-timesyncd`, un cliente SNTP suficiente. **Opcional, recomendado:** `chrony`, con NTP completo, mejor ajuste en redes inestables, NTS y métricas con `chronyc`.

#### 17.1 Migración de `systemd-timesyncd` a `chrony`

```bash
timedatectl
```

Salida típica con timesyncd:

```text
               Local time: ...
           Universal time: ...
                 RTC time: ...
                Time zone: America/Lima (-05, -0500)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

```bash
sudo apt install -y chrony
```

`chrony` declara `Conflicts: time-daemon` y `apt` retira timesyncd en la misma operación (`REMOVING: systemd-timesyncd`), dejándolo en `rc`. Para borrarlo del todo:

```bash
sudo apt purge -y systemd-timesyncd
```

El aviso `dpkg-statoverride: ... --update given but /var/log/chrony does not exist` es normal: el directorio lo crea systemd al arrancar el servicio (`LogsDirectory=chrony`, modo 0750), con los mismos permisos que registra el paquete.

```bash
sudo systemctl status chrony --no-pager
```

```bash
timedatectl
```

`NTP service: active` corresponde ahora a chrony.

#### 17.2 Configuración de servidores NTP

`/etc/chrony/chrony.conf` trae de serie el pool `2.debian.pool.ntp.org`:

```bash
grep -E '^(pool|server)' /etc/chrony/chrony.conf
```

```text
pool 2.debian.pool.ntp.org iburst
```

El pool se elige por la ubicación física, no por la zona horaria: un VPS con `America/Lima` en un centro de datos europeo se sincroniza mejor con servidores europeos. Las fuentes que elige el pool por defecto (`chronyc sources`, 17.3) dan una pista de dónde está. Para un servidor en Perú, el pool del país y el regional de respaldo:

```bash
sudoedit /etc/chrony/chrony.conf
```

La línea `pool` se sustituye por:

```text
# Pool por país (menor latencia, más precisión; ajustar según el país del servidor)
# Lista de pools por país: https://www.pool.ntp.org/zone/@
pool 0.pe.pool.ntp.org iburst maxsources 3
pool 1.pe.pool.ntp.org iburst maxsources 2

# Pool regional de Sudamérica como respaldo
pool 2.south-america.pool.ntp.org iburst maxsources 2

# Pool global como último respaldo
pool 2.debian.pool.ntp.org iburst maxsources 1
```

`iburst` acelera la primera sincronización y `maxsources` limita los servidores por pool. Otros países, en <https://www.pool.ntp.org/zone/@>, con el regional `north-america`, `europe`, `asia` u `oceania`. Para un VPS en un centro de datos de Francia, por ejemplo: `0.fr` y `1.fr` como pools de país, `2.europe` como regional y `2.debian` como último respaldo (validado en un VPS de Contabo el 2026-11-19).

> Con servidor NTP corporativo:
>
> ```text
> server ntp.empresa.local iburst
> ```

**NTS (RFC 8915).** NTP de los pools viaja sin autenticar; NTS lo autentica sobre TLS y chrony lo soporta desde 4.0:

```text
# Servidores con soporte NTS, delante de los pools
server time.cloudflare.com iburst nts
server ntppool1.time.nl iburst nts

# Pool estándar como respaldo sin NTS
pool 2.debian.pool.ntp.org iburst maxsources 2
```

> NTS necesita la hora aproximadamente correcta para negociar TLS; `makestep` (17.4) la corrige antes. Usa TCP 4460 de salida para el intercambio de claves (NTS-KE), que `allow outgoing` (11.1) permite.

Las líneas NTS se añaden delante de los pools, que quedan como respaldo. Tras reiniciar chrony, `sudo chronyc -n authdata` debe mostrar `NTS` en la columna `Mode` y un número de cookies distinto de cero para cada servidor NTS (validado en un VPS de Contabo el 2026-11-19 con chrony 4.6.1). Con `authselectmode mix`, el valor por defecto, chrony combina fuentes autenticadas y sin autenticar; `authselectmode require` solo usa las NTS, a costa de quedarse sin hora si todas fallan.

```bash
sudo systemctl restart chrony
```

#### 17.3 Verificación

Justo tras arrancar, `tracking` muestra `Skew : 1000000.000 ppm` y una `Root dispersion` de segundos: faltan muestras. Se espera unos minutos.

```bash
chronyc tracking
```

Salida típica:

```text
Reference ID    : C0A80101 (ntp.example.com)
Stratum         : 3
Ref time (UTC)  : Mon Apr 27 09:00:00 2026
System time     : 0.000123456 seconds slow of NTP time
Last offset     : -0.000045678 seconds
RMS offset      : 0.000234567 seconds
Frequency       : 12.345 ppm slow
Residual freq   : +0.012 ppm
Skew            : 0.456 ppm
Root delay      : 0.012345678 seconds
Root dispersion : 0.001234567 seconds
Update interval : 64.2 seconds
Leap status     : Normal
```

`Stratum` 2 a 4, `System time` en micro o milisegundos y `Leap status: Normal`.

```bash
chronyc sources -v
```

En `MS`, `^*` es la fuente seleccionada, `^+` las que se combinan con ella, `^-` válidas fuera de la combinación (lo normal en la mayoría), `^x` y `^~` descartadas por incoherentes o variables, y `^?` inutilizables o sin respuesta todavía.

#### 17.4 Comportamiento ante desviaciones grandes

chrony ajusta el reloj de forma gradual, porque un salto atrás rompe a las aplicaciones que asumen monotonía. Para el arranque tras un apagado largo o una VM suspendida, Debian trae:

```text
makestep 1 3
```

Si en alguna de las 3 primeras mediciones la desviación supera 1 segundo, salta; después, solo ajuste gradual. No se modifica.

#### 17.5 Verificación cruzada con `timedatectl`

```bash
timedatectl
```

`System clock synchronized: yes`, `NTP service: active` y la zona de 7.3.

### 18. Logs y rotación

Los logs son la primera fuente de diagnóstico y la única evidencia tras un incidente, y también pueden llenar `/var/log`. Debian 13 no instala `rsyslog`: no hay `/var/log/syslog` ni `auth.log`, y todo se consulta con `journalctl`. `logrotate` queda para los archivos planos de las aplicaciones. El reenvío a un servidor central depende de la infraestructura y tiene su propio manual (To-do).

#### 18.1 Configuración de `journald`

journald guarda en `/var/log/journal/` si existe (persistente) y en `/run/log/journal/` si no (se pierde al reiniciar). En Debian 13 el directorio ya existe tras la instalación, tanto en la imagen cloud como desde la ISO, y el journal es persistente desde el primer arranque.

```bash
sudo journalctl --disk-usage
ls -ld /var/log/journal/ 2>/dev/null && echo "Logs persistentes" || echo "Logs volátiles"
```

`--disk-usage` no distingue ubicación; lo concluyente es el directorio. Si no es persistente:

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

##### 18.1.1 Límites de tamaño y retención

Los valores van en un drop-in. `journald.conf` es un conffile de `systemd`: editado, `debsums -ac` (14.2) lo marca, y si una actualización trae una versión nueva, `unattended-upgrades` aparta el paquete (`Packages blacklist due to conffile prompts`) y systemd deja de actualizarse sin aviso.

```bash
sudo install -d -m 0755 /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\nCompress=yes\nSystemMaxUse=1G\nSystemKeepFree=512M\nSystemMaxFileSize=128M\nMaxRetentionSec=3month\n' \
  | sudo install -m 0644 /dev/stdin /etc/systemd/journald.conf.d/50-hardening.conf
```

Contenido resultante:

```ini
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=1G
SystemKeepFree=512M
SystemMaxFileSize=128M
MaxRetentionSec=3month
```

Los permisos explícitos evitan el efecto de `UMASK 027` (9.4), que dejaría el drop-in ilegible para `systemd-analyze cat-config`.

`Storage=persistent` crea el directorio si falta; `Compress=yes` comprime lo viejo; `SystemMaxUse=1G` acota el total, adecuado para un `/var/log` de 2 GiB (2.2) y ampliable a `2G` o `3G` con 5 GiB o más; `SystemKeepFree=512M` deja ese espacio libre; `SystemMaxFileSize=128M` rota cada archivo; `MaxRetentionSec=3month` borra lo anterior a tres meses, según la retención que exija la organización.

```bash
sudo systemctl restart systemd-journald
journalctl -u systemd-journald -n 1 -o cat
systemd-analyze cat-config systemd/journald.conf | grep -v '^#' | grep .
```

Salida de `journalctl` (el directorio es el machine-id y las cifras varían):

```text
System Journal (/var/log/journal/<machine-id>) is 16M, max 1G, 1007.9M free.
```

Salida de `systemd-analyze cat-config`:

```ini
[Journal]
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=1G
SystemKeepFree=512M
SystemMaxFileSize=128M
MaxRetentionSec=3month
[Journal]
ForwardToSyslog=yes
```

`max 1G` confirma el drop-in. El primer `[Journal]` vacío es `journald.conf` sin cambios, y el último bloque es el `syslog.conf` de Debian en `/usr/lib/systemd/journald.conf.d/`, que fija `ForwardToSyslog=yes` y gana a `journald.conf`. Sin `rsyslog` no hay quien reciba el reenvío; si se instala y se quiere evitar la duplicación, el valor se fija en un drop-in que se ordene después de `syslog.conf`.

##### 18.1.2 Comandos básicos de consulta

```bash
# Arranques registrados (requisito para usar -b -N)
sudo journalctl --list-boots

# Logs del arranque actual, saltando al final
sudo journalctl -b -e

# Logs del arranque anterior, saltando al final
sudo journalctl -b -1 -e

# Logs de un servicio específico
sudo journalctl -u ssh -n 100

# Logs en tiempo real (se interrumpe con Ctrl+C)
sudo journalctl -f

# Logs por periodo, con expresiones relativas
sudo journalctl --since today
sudo journalctl --since yesterday --until today
sudo journalctl --since "1 hour ago"

# Solo errores y peor, del arranque actual
sudo journalctl -b -p err

# Consistencia estructural de los archivos del journal
sudo journalctl --verify
```

- `-e` salta al final del paginador; en `-f` no hay paginador y se sale con Ctrl+C.
- `-b -1` necesita un arranque anterior registrado; si no lo hay, da `No journal boot entry found for the specified boot (-1)`. `--list-boots` lo dice antes.
- Una fecha sin hora es las 00:00: `--until "2026-09-26"` excluye ese día; para incluirlo, `--until "2026-09-26 23:59:59"`.
- Los meses siguen el locale (7.4): con `es_PE.UTF-8`, septiembre es `set`. Para scripts, `-o short-iso`.
- En un servidor recién desplegado, `-b -p err` solo debe mostrar entradas conocidas: el error de `systemd-ssh-generator` (2.5.2) hasta el siguiente arranque, y los fallos de autenticación de las pruebas de faillock (9.3.3) o de una conexión con demasiadas llaves (10.4) (`3 incorrect password attempts`, `a password is required`, `maximum authentication attempts exceeded`). Con dhcpcd en una red sin DHCPv6 aparece además, en cada arranque, `dhcpcd[...]: <interfaz>: failed to request DHCPv6 information`, que es inofensivo. Cualquier otra merece atención.
- Durante un recorrido intensivo del disco, como el de AIDE (14.3), puede salir `auditd: Error receiving audit netlink packet (No buffer space available)`: el kernel no pudo encolar más eventos en el búfer de recepción del socket netlink de `auditd` ante una ráfaga. Es un mecanismo distinto del contador `lost` (desbordes del `backlog_limit`) y del `backlog` momentáneo; si después `auditctl -s` sigue mostrando `lost 0`, como se observó, no se perdió ningún evento. No se reprodujo en cuatro recorridos de AIDE seguidos; si se repite fuera de esos momentos, se revisa `-b 8192` (14.1).
- Con el usuario en `adm` (9.2), `journalctl` lee todo sin `sudo`. Los ejemplos usan `sudo` para no depender de eso; la excepción de 9.3.1 evita que se dupliquen en `/var/log/sudo-io/`.

> **Nota:** `--verify` comprueba la estructura de los archivos, no que nadie los haya alterado. Para eso existe Forward Secure Sealing: `journalctl --setup-keys` genera una clave de sellado, que queda en el servidor, y una de verificación, que se guarda fuera, y `journalctl --verify --verify-key=<clave>` comprueba los sellos. `Seal=yes` es el valor por defecto, pero sin claves no hace nada. No se configura en el manual base (To-do).

La persistencia se demuestra cuando un arranque sobrevive al siguiente. Tras un reinicio controlado, `--list-boots` debe mostrar dos entradas y `sudo journalctl -b -1 -e` terminar con un apagado limpio:

```text
systemd-journald[...]: Received SIGTERM from PID 1 (systemd-shutdow).
systemd-journald[...]: Journal stopped
```

El mismo reinicio confirma que SSH, UFW y chrony sobreviven (7.5.8).

#### 18.2 `logrotate` para logs de aplicaciones

Para las aplicaciones que escriben archivos propios en `/var/log/`. Viene instalado.

```bash
cat /etc/logrotate.conf
```

Cada paquete deja su archivo en `/etc/logrotate.d/`. Para una aplicación nueva (`/etc/logrotate.d/miapp`):

```text
/var/log/miapp/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 0640 miapp adm
    sharedscripts
    postrotate
        systemctl reload miapp >/dev/null 2>&1 || true
    endscript
}
```

Rotación semanal con 8 archivos, comprimidos salvo el más reciente (`delaycompress`, por si la aplicación aún escribe en él), sin fallar si falta (`missingok`) ni rotar vacíos (`notifempty`), con el archivo nuevo creado con esos permisos y una recarga del servicio tras rotar (`sharedscripts` la hace una sola vez).

```bash
sudo logrotate -d /etc/logrotate.d/miapp     # debug, no aplica nada
sudo logrotate -f /etc/logrotate.d/miapp     # fuerza la rotación ahora
```

Lo ejecuta `logrotate.timer` a diario:

```bash
sudo systemctl status logrotate.timer --no-pager
```

##### 18.2.1 Sesiones de `sudo` en `/var/log/sudo-io/`

Lo que graba `log_output` (9.3.1) no lo rota nadie y crece sin límite. La retención se fija con `systemd-tmpfiles`:

```bash
echo 'e /var/log/sudo-io - - - 90d' | sudo install -m 0644 /dev/stdin /etc/tmpfiles.d/sudo-io.conf
sudo systemd-tmpfiles --clean /etc/tmpfiles.d/sudo-io.conf
sudo systemctl status systemd-tmpfiles-clean.timer --no-pager
```

El tipo `e` con `-` en modo, usuario y grupo solo borra por antigüedad; lo aplica `systemd-tmpfiles-clean.timer` a diario, y que `--clean` no devuelva nada es lo normal. Los 90 días coinciden con `MaxRetentionSec=3month` (18.1.1); si se exige más, se cambian ambos (18.4). `sudoreplay` sigue funcionando con las sesiones que quedan:

```bash
sudo sudoreplay -l | tail -5     # últimas sesiones grabadas
```

#### 18.3 Verificación de uso de disco

```bash
df -hT /var/log
sudo du -sh /var/log/* | sort -h
sudo journalctl --disk-usage
```

Espacio de `/var/log`, directorios más grandes (vigilando `/var/log/sudo-io`) y uso del journal. Si se llena: bajar `SystemMaxUse` y `MaxRetentionSec`, rotar menos archivos, acortar `/var/log/sudo-io/` o excluir de `log_output` otros comandos voluminosos (9.3.1), o ampliar el LV con `lvresize` (2.2).

#### 18.4 Retención frente a necesidad forense

La rotación está calibrada para que el disco no se llene y para diagnosticar lo que pasa hoy, no para reconstruir lo que pasó hace una semana. Un incidente se detecta tarde, y los registros de esos días pueden haber rotado aunque todo funcionara como se configuró. Se revisa la retención efectiva de cada fuente:

```bash
# journald: uso actual y límite configurado
journalctl --disk-usage
grep -rE '^\s*(SystemMaxUse|MaxRetentionSec)' /etc/systemd/journald.conf /etc/systemd/journald.conf.d/

# logs de aplicación: cuántos archivos hay y qué periodo cubren
SERVICIO=miapp   # directorio de logs de la aplicación
ls -lh --time-style=+%F "/var/log/${SERVICIO:?}/"
```

La fecha del archivo más antiguo debe coincidir con la retención pretendida. Con rotación por tamaño y número fijo de archivos, la retención depende del tráfico: cuanto más ocupado el servicio, menos historia, justo al revés de lo deseable, y los sondeos de salud consumen esa cuota sin aportar. Lo razonable es separar los dos usos: rotación normal para la operación y una copia comprimida de los logs de acceso al destino de respaldos (sección 21), con retención de meses.

> Los contenedores tienen su propia rotación, pensada para no llenar el disco y no para conservar historia. Se revisa con el mismo criterio.

### 19. Monitoreo básico

Un par de comprobaciones periódicas que envían correo cuando algo falla cubren la mayoría de los problemas operativos. Prometheus, Zabbix o Netdata son de manuales específicos.

#### 19.1 Herramientas de consulta puntual

**Opcional, recomendado.**

```bash
sudo apt install -y htop iotop iftop sysstat lsof
```

`htop` para procesos, CPU y memoria; `iotop`, qué procesos hacen E/S cuando el sistema va lento sin carga de CPU; `iftop`, qué conexiones consumen ancho de banda; `sysstat`, con `sar`, `iostat`, `mpstat` y `pidstat` para el histórico; `lsof`, qué proceso retiene un archivo o un puerto.

```bash
# Top 10 procesos por uso de memoria
ps aux --sort=-%mem | head -11

# Top 10 procesos por uso de CPU
ps aux --sort=-%cpu | head -11

# Espacio por filesystem
df -hT

# Inodos por filesystem (a veces el problema no es espacio sino inodos)
df -i

# Tamaño de directorios bajo /
sudo du -sh /* 2>/dev/null | sort -h

# Conexiones de red activas con proceso
sudo ss -tunap

# Puertos en escucha
sudo ss -tlnp

# Carga del sistema
uptime
w

# Memoria
free -h
```

La recogida histórica de `sysstat` se activa así:

```bash
sudo sed -i 's/^ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
sudo dpkg-reconfigure -f noninteractive sysstat
systemctl list-timers 'sysstat*' --no-pager
```

Con systemd recoge `sysstat-collect.timer`; el `/etc/cron.d/sysstat` llama a `debian-sa1`, que no hace nada con systemd. `ENABLED` solo lo lee la configuración del paquete, que habilita o deshabilita los timers, así que tras cambiarlo hace falta `dpkg-reconfigure`; reiniciar `sysstat.service` no toca los timers. Los dos comandos se pueden repetir.

`list-timers` debe mostrar `sysstat-collect.timer` y `sysstat-summary.timer`; sin ellos, `sar` no tendrá datos:

```text
NEXT                        LEFT LAST PASSED UNIT                  ACTIVATES
Sat 2026-10-17 19:10:00 -05 9min -         - sysstat-collect.timer sysstat-collect.service
Sun 2026-10-18 00:07:00 -05 5h   -         - sysstat-summary.timer sysstat-summary.service
```

#### 19.2 Alertas mínimas vía cron

**Opcional, recomendado.** Un script en `/etc/cron.hourly/` que comprueba condiciones críticas y avisa por correo:

```bash
# Descargar e instalar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §3.1.1)
: "${ASSETS_BASE:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-cron.hourly-server-check.txt" \
    -O /tmp/plantilla-etc-cron.hourly-server-check.txt && \
sudo install -D -m 0700 -o root -g root \
    /tmp/plantilla-etc-cron.hourly-server-check.txt /etc/cron.hourly/server-check
```

`install` fija propietario y permisos en un paso repetible, y `-D` crea el directorio si falta. El destino no lleva punto ni extensión, porque `run-parts` ignora esos nombres.

```bash
sudo run-parts --test /etc/cron.hourly
bash -n /tmp/plantilla-etc-cron.hourly-server-check.txt && echo "sintaxis correcta"
sed 's#| /usr/sbin/sendmail -t#| cat#' /tmp/plantilla-etc-cron.hourly-server-check.txt | bash
```

El primer comando debe listar `server-check`, y solo con `sudo`, porque el script es 0700. El tercero es una prueba en seco que cambia el envío por la pantalla; en un servidor sano no imprime nada. Los servicios críticos se adaptan al rol (`postfix` y `dovecot`, `nginx` o `apache2`). El correo a root llega fuera con la sección 20.

#### 19.3 Estado de servicios críticos

```bash
systemctl --failed
systemctl list-units --state=running --type=service | head -30
```

`systemctl --failed` debe estar vacío. Los servicios del manual:

```bash
for svc in ssh chrony fail2ban ufw apparmor unattended-upgrades; do
    printf "%-25s %s\n" "$svc" "$(systemctl is-active "$svc" 2>/dev/null)"
done
```

### 20. Reenvío de correo del sistema con `msmtp`

`unattended-upgrades` (16.3), `debsums`, AIDE (14) y las alertas (19.2) escriben a root, y sin reenvío se quedan en `/var/mail/root`. Esta sección envía el correo local a una dirección externa por un relay SMTP autenticado. El servidor solo envía. `msmtp` es solo un transmisor, sin demonio en escucha ni cola.

#### 20.1 Decisiones de diseño

- Aliases en `/etc/msmtp/aliases`: msmtp no lee `/etc/aliases`, y sin traducción el relay rechaza destinatarios como `root`.
- Logs en journald (18.1), sin archivo propio, que daría problemas de permisos y pediría logrotate.
- Outlook y Hotmail personales como remitentes quedan fuera porque exigen XOAUTH2.

#### 20.2 Requisitos previos

Un relay con autenticación estándar (corporativo, Mailgun, Brevo, Amazon SES, Gmail), su usuario y contraseña, una dirección `From` autorizada y la dirección de destino.

#### 20.3 Instalación

```bash
sudo apt install -y msmtp msmtp-mta bsd-mailx
```

`msmtp-mta` enlaza `/usr/sbin/sendmail` a msmtp, así que cron y cualquier programa envían por él. `bsd-mailx` aporta `mail`: ocupa 169 KB y solo depende de `libbsd0` y `liblockfile1`. Se prefiere a `mailutils`, cuya biblioteca `libmailutils9t64` arrastra como dependencias obligatorias los clientes de MariaDB y PostgreSQL, Guile y Python, de modo que `--no-install-recommends` no las evita.

##### 20.3.1 Retiro de `exim4` y `mailutils`

Si algún paquete trajo `exim4` por sus Recommends, como `cron` en algunas imágenes cloud, apt lo retira al instalar `msmtp-mta` pero lo deja en `rc`. Si `mailutils` estaba instalado, se purga con él: `bsd-mailx` ya sería `/usr/bin/mail` por prioridad (50 frente a 30), pero sus dependencias seguirían ocupando el sistema:

```bash
sudo apt -y purge exim4-base exim4-config exim4-daemon-light mailutils mailutils-common
sudo apt -y autoremove --purge
```

Antes de instalar `msmtp-mta`, `autoremove` no retira nada, porque `exim4` era el único MTA. El buzón `/var/mail/root` se conserva; conviene leerlo una vez con `sudo less /var/mail/root`. Si estas órdenes van dentro de un script que se pasa por la entrada estándar (`ssh servidor 'sudo bash -s' < script`), `apt` y los scripts de los paquetes leen esa misma entrada y se comen el resto del script: un paquete puede quedar desempaquetado sin configurar (`iU` en `dpkg -l`). Dentro de esos scripts, `apt` se ejecuta con `</dev/null`, y un paquete a medias se termina con `sudo dpkg --configure -a`.

```bash
dpkg -l 'exim4*' | grep -E '^(ii|rc)' || echo "exim4 eliminado por completo"
readlink -f /usr/sbin/sendmail
readlink -f /usr/bin/mail
sudo ss -tlnp | grep ':25\b' || echo "nada escucha en el puerto 25"
```

Salida esperada:

```text
exim4 eliminado por completo
/usr/bin/msmtp
/usr/bin/bsd-mailx
nada escucha en el puerto 25
```

#### 20.4 Configuración global de msmtp

`MIPUERTO_SMTP` no pisa `MIPUERTO`, el de SSH. `ASSETS_BASE` es el de la sección 3.

```bash
export MIRELAY=smtp.relay.example.com
export MIPUERTO_SMTP=587
export MIUSUARIO_SMTP=notificaciones@example.com
export MIFROM=notificaciones@example.com

: "${ASSETS_BASE:?}" "${MIRELAY:?}" "${MIPUERTO_SMTP:?}" "${MIUSUARIO_SMTP:?}" "${MIFROM:?}" && \
wget "${ASSETS_BASE}/plantilla-etc-msmtprc.txt" -O /tmp/plantilla-etc-msmtprc.txt && \
sed -e "s|\$MIRELAY|${MIRELAY}|g" \
    -e "s|\$MIPUERTO_SMTP|${MIPUERTO_SMTP}|g" \
    -e "s|\$MIUSUARIO_SMTP|${MIUSUARIO_SMTP}|g" \
    -e "s|\$MIFROM|${MIFROM}|g" \
    /tmp/plantilla-etc-msmtprc.txt \
    | sudo install -m 0640 -o root -g msmtp /dev/stdin /etc/msmtprc

# Comprobar que no quedó ningún marcador sin sustituir
sudo grep -n '\$MI[A-Z_]*' /etc/msmtprc || echo "✓ plantilla aplicada"
sudo ls -l /etc/msmtprc
```

`install` escribe con propietario y permisos definitivos, sin depender de la umask (9.4), y el bloque se puede repetir. El `ls` debe mostrar `-rw-r----- 1 root msmtp`.

Algunos programas pasan a sendmail su propio remitente (`-f usuario@FQDN`), que un relay estricto rechaza; `allow_from_override off` obliga a usar el `from` configurado. La plantilla lo trae desde la v1.2, y con ella este comando no hace nada:

```bash
sudo grep -q '^allow_from_override' /etc/msmtprc || sudo sed -i '/^aliases/a allow_from_override off' /etc/msmtprc
```

> `allow_from_override` fija el remitente del envelope. Si el relay valida también la cabecera `From:` (Amazon SES y algunos corporativos; Gmail la reescribe), se añade `set_from_header on` en `defaults`.

Con varios servidores avisando a la misma dirección, el nombre visible del remitente dice de cuál viene cada correo (`villonaco <cuenta@dominio.com>`):

```bash
sudo grep -q '^from_full_name' /etc/msmtprc || sudo sed -i "/^from /a from_full_name  $(hostname -s)" /etc/msmtprc
sudo grep -E '^(from|from_full_name) ' /etc/msmtprc
```

> **Ejemplo con Gmail.** Variables:
>
> ```bash
> export MIRELAY=smtp.gmail.com
> export MIPUERTO_SMTP=587
> export MIUSUARIO_SMTP=tu-cuenta@gmail.com
> export MIFROM=tu-cuenta@gmail.com
> ```
>
> Requiere la verificación en dos pasos (<https://myaccount.google.com/security>) y una app password generada en <https://myaccount.google.com/apppasswords>, que se muestra una sola vez y se usa en 20.5. Da acceso completo a la cuenta, así que conviene una cuenta dedicada a notificaciones; el límite de 500 destinatarios diarios no afecta a las alertas.

#### 20.5 Contraseña del relay

```bash
sudo install -d -o root -g msmtp -m 0750 /etc/msmtp
read -rsp 'Contraseña del relay: ' MIPASS_SMTP; echo
printf '%s' "${MIPASS_SMTP:?}" | sudo install -m 0640 -o root -g msmtp /dev/stdin /etc/msmtp/password
unset MIPASS_SMTP
sudo ls -la /etc/msmtp/
```

`read -rs` la pide sin eco y sin dejarla en el historial; `printf '%s'` evita el salto de línea final. Repetir el bloque la sustituye, que es lo que se quiere al rotarla. Salida esperada:

```text
drwxr-x--- 2 root msmtp 4096 ... .
-rw-r----- 1 root msmtp   XX ... password
```

#### 20.6 Aliases para destinatarios locales

```bash
export MIDESTINO=admin@example.com

sudo install -m 0644 -o root -g root /dev/stdin /etc/msmtp/aliases <<EOF
# Aliases de msmtp: <local>: <externa>
root: ${MIDESTINO:?}
default: ${MIDESTINO}
EOF

sudo grep '^aliases' /etc/msmtprc
```

`default:` recoge cualquier otro usuario local. El `grep` debe devolver `aliases         /etc/msmtp/aliases`; si no:

```bash
sudo grep -q '^aliases' /etc/msmtprc || sudo sed -i '/^timeout/a aliases         /etc/msmtp/aliases' /etc/msmtprc
```

El perfil de AppArmor de msmtp no permite leer `/etc/msmtp/aliases`: con el perfil en enforce y sin este override, todo envío a un destinatario local falla con `Permiso denegado` y exit 78. El instalador de msmtp pregunta si se activa el perfil (`msmtp/apparmor`), y en una instalación no interactiva queda desactivado, con un enlace en `/etc/apparmor.d/disable/`. Se comprueba y se activa así:

```bash
sudo aa-status | grep -E '^\s+msmtp' || echo "perfil de msmtp no cargado"
echo 'msmtp msmtp/apparmor boolean true' | sudo debconf-set-selections
sudo bash -c 'umask 022; dpkg-reconfigure -f noninteractive msmtp'
sudo aa-status | grep -E '^\s+msmtp'
```

Debe listar `msmtp` y `msmtp//helpers`. El `umask 022` evita el efecto de 9.4 sobre los archivos de debconf. Con el perfil en enforce, un envío como root desde un directorio ajeno (el `cwd` de `sudo` suele ser el home 0700 del administrador) deja en `audit.log` denegaciones `capable dac_read_search` y `dac_override` de `msmtp//helpers`; el correo sale igual.

```bash
sudo grep -qxF '/etc/msmtp/aliases r,' /etc/apparmor.d/local/usr.bin.msmtp \
  || echo '/etc/msmtp/aliases r,' | sudo tee -a /etc/apparmor.d/local/usr.bin.msmtp > /dev/null
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.msmtp
```

El `grep` evita duplicar la línea. El archivo local lo crea vacío el paquete y el perfil lo incluye, así que `tee -a` conserva sus permisos (0644).

#### 20.7 Cliente `mail` (`bsd-mailx`)

`bsd-mailx` no necesita configuración: entrega el mensaje a `/usr/sbin/sendmail` con los destinatarios como argumentos, y msmtp traduce `root` con los aliases de 20.6. No se crea `/etc/mail.rc`; si quedó uno de `mailutils`, con su sintaxis propia, se retira:

```bash
sudo rm -f /etc/mail.rc
readlink -f /usr/bin/mail
```

Debe devolver `/usr/bin/bsd-mailx` (validado en un VPS de Contabo el 2026-11-19).

#### 20.8 Grupo `msmtp` para el usuario administrativo

Permite enviar correo de prueba sin `sudo`. El grupo da lectura de `/etc/msmtprc` y `/etc/msmtp/password`, y msmtp exige verificarla con el uid real del proceso: una capacidad como la `CAP_DAC_READ_SEARCH` de alguna unidad no le basta, y sin acceso de grupo falla con `no se encuentra la cuenta default` y exit 78 (tabla de 20.9). El servicio de AIDE lo recibe como grupo suplementario, sin pasar por este paso (14.3).

```bash
sudo usermod -aG msmtp "${MIUSUARIO:?}"
exit
# Reconectar por SSH
id -nG | grep -qw msmtp && echo "✓ grupo msmtp cargado" || echo "✗ reabrir sesión o usar sudo"
```

#### 20.9 Prueba de envío directo

Primero msmtp solo y después con `mail`; si el primero funciona y el segundo no, el problema está en el cliente `mail` (20.7).

```bash
printf 'To: %s\nSubject: Prueba sendmail directo\n\nCuerpo de prueba.\n' "${MIDESTINO:?}" | /usr/sbin/sendmail -t; echo "exit=$?"
echo "Prueba desde $(hostname) a $(date)." | mail -s "Prueba de msmtp desde $(hostname)" "${MIDESTINO}"
sudo journalctl -t msmtp -n 2
```

Cada envío deja una línea con `smtpstatus=250`, `recipients=` con el destino y `from=` con el de `/etc/msmtprc`. Si no aparece nada, falló antes de conectar y el error está en la terminal.

| Mensaje | Causa probable | Solución |
| --- | --- | --- |
| `argumento inválido 587_SMTP para la orden port` | Sustitución incompleta de la plantilla | Revisar el `sed` de 20.4 y ejecutar su verificación |
| `mail: orden no encontrada` | `bsd-mailx` sin instalar o a medio configurar (estado `iU` en `dpkg -l`) | `sudo dpkg --configure -a` y comprobar `readlink -f /usr/bin/mail` (20.7) |
| `/etc/msmtp/aliases: Permiso denegado`, exit 78 | AppArmor bloquea la lectura de los aliases | Override de 20.6 |
| `no se encuentra la cuenta default`, o `dirección de correo inválida` con exit 78 | msmtp no puede leer `/etc/msmtprc`: usuario sin el grupo `msmtp` (20.8) o servicio sin el grupo suplementario (14.3); sin cuenta, msmtp intenta autoconfiguración SRV y produce el segundo mensaje | Añadir al grupo, o crear el drop-in de 14.3 |
| `authentication failed` | Usuario o contraseña incorrectos | Revisar `/etc/msmtp/password` y `user`; en Gmail, usar app password |
| `basic authentication is disabled` | El relay exige OAuth2 (Microsoft 365, Outlook) | Cambiar de relay o implementar XOAUTH2 |
| `connection refused` | Host, puerto o firewall | Revisar `host` y `port`; `nc -vz <host> <puerto>` |
| `sender address not allowed` | `from` no autorizado en el relay | Usar una dirección autorizada; confirmar `allow_from_override off` y, si aplica, `set_from_header on` |
| `TLS handshake failed` | Certificado del relay no confiable | Revisar `tls_trust_file` |
| `not a valid RFC 5321 address` | Destinatario local sin traducir | Revisar `/etc/msmtp/aliases` y la directiva `aliases` |
| `cannot log to /var/log/msmtp.log`, o el archivo existe pero sigue vacío | `logfile` activado sin el archivo creado, o sin la regla de AppArmor de 20.13 | Seguir los pasos de 20.13 o volver a comentar `logfile` |

#### 20.10 Prueba de redirección de aliases

```bash
echo "Prueba de redirección de root a $(date)" | mail -s "Test redirección root" root
sudo journalctl -t msmtp -n 1
```

`recipients=` debe ser la dirección de `MIDESTINO`, no `root`. Con `Permiso denegado`, falta el override de 20.6.

#### 20.11 Prueba con el flujo completo del sistema

Con el script de 19.2, bajando el umbral de disco:

```bash
df -h /
sudo sed -i 's/-ge 85/-ge 1/' /etc/cron.hourly/server-check
sudo /etc/cron.hourly/server-check
```

La alerta debe llegar en segundos. Se restaura:

```bash
sudo sed -i 's/-ge 1/-ge 85/' /etc/cron.hourly/server-check
sudo /etc/cron.hourly/server-check; echo "exit=$?"
```

La segunda ejecución debe terminar sin salida y con `exit=0`.

#### 20.12 Consideraciones de seguridad

- `/etc/msmtp/password` es lo más sensible: una credencial dedicada al servidor, que se rota al reinstalar o ante sospecha.
- Los correos pueden llevar información sensible (CVE pendientes, archivos cambiados, IPs baneadas): el destino debe ser una dirección controlada.
- Con `allow outgoing` (11) UFW no necesita reglas; si se filtra la salida, se abren 587 o 465 hacia el relay.
- OAuth2 o certificado de cliente, si el relay los admite, son preferibles a la contraseña.

#### 20.13 Logs

```bash
sudo journalctl -t msmtp -n 50              # últimos envíos
sudo journalctl -t msmtp -p err             # solo errores
sudo journalctl -t msmtp --since "1 hour ago"
```

La retención es la de journald (18.1). Solo para una depuración puntual se activa un log propio:

```bash
sudo sed -i 's|^#\s*logfile .*|logfile         /var/log/msmtp.log|' /etc/msmtprc
sudo install -o root -g msmtp -m 0660 /dev/null /var/log/msmtp.log
sudo grep -qxF '/var/log/msmtp.log wk,' /etc/apparmor.d/local/usr.bin.msmtp \
  || echo '/var/log/msmtp.log wk,' | sudo tee -a /etc/apparmor.d/local/usr.bin.msmtp > /dev/null
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.msmtp

sudo install -m 0644 -o root -g root /dev/stdin /etc/logrotate.d/msmtp <<'EOF'
/var/log/msmtp.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 0660 root msmtp
}
EOF
```

```bash
msmtp --pretend </dev/null | grep '^logfile ='
```

Debe mostrar `logfile = /var/log/msmtp.log`; si la línea lleva algo detrás, msmtp lo toma como parte del nombre. La regla de AppArmor hace falta porque el perfil solo permite escribir en `/var/log/msmtp`, sin extensión: sin ella el correo sale y llega al journal, pero el archivo queda vacío y la única huella es la denegación en `audit.log` (15.3):

```bash
sudo grep -h 'apparmor="DENIED"' /var/log/audit/audit.log* | grep 'msmtp.log'
```

```text
type=AVC msg=audit(...): apparmor="DENIED" operation="open" class="file" profile="msmtp" name="/var/log/msmtp.log" comm="sendmail" requested_mask="ac" denied_mask="ac" ...
```

Con la regla, cada envío deja una línea, lo lance root o un usuario del grupo `msmtp`:

```bash
echo "prueba" | sudo mail -s "log msmtp root" root
echo "prueba" | mail -s "log msmtp usuario" root
sudo tail -2 /var/log/msmtp.log
```

Para desactivarlo:

```bash
sudo sed -i 's|^logfile .*|# logfile         /var/log/msmtp.log|' /etc/msmtprc
sudo rm -f /var/log/msmtp.log /etc/logrotate.d/msmtp
sudo sed -i '\#^/var/log/msmtp.log wk,$#d' /etc/apparmor.d/local/usr.bin.msmtp
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.msmtp
msmtp --pretend </dev/null | grep '^logfile ='     # logfile = (no seleccionado)
```

Retirar la regla es opcional; en el archivo local debe quedar al menos la línea de los aliases de 20.6.

### 21. Respaldos

Sin respaldos, el servidor es un punto único de fallo, y un respaldo nunca restaurado es una hipótesis. Esta sección es el mínimo para un servidor base; los datos pesados (bases de datos, buzones) tienen estrategias propias en sus manuales.

> Validado en un VPS de Contabo el 2026-11-19 hasta la automatización (21.3.5), la verificación (21.4) y la prueba de restauración (21.5). El 2026-11-20 se validaron la cuenta de solo lectura (21.3.6), la copia en un servidor de respaldos físico con btrfs, con la restauración y `borg check --verify-data` desde la copia, y el encendido desde el apagado total por RTC (21.3.8), con un script que copiaba un solo repositorio. Quedan pendientes (To-do) la copia del directorio completo con `copia-borg.sh` (21.3.7), el ciclo semanal completo de encendido, copia y apagado, y la copia en un disco externo (21.3.9).

#### 21.1 Estrategia 3-2-1

Tres copias, en dos medios distintos y una fuera del sitio:

1. Un repositorio Borg local en el propio servidor, en `/var/backups/borg/<host>`. Protege contra borrados y cambios accidentales y es la fuente de las otras dos copias.
2. Una copia en un servidor de respaldos privado (`<servidor-respaldos>`), que puede pasar la mayor parte del tiempo apagado o sin acceso desde Internet.
3. Una copia en un disco externo cifrado con LUKS (`<disco-externo>`), conectado al servidor de respaldos. Puede haber más de uno, y se actualizan a mano.

Como el servidor de respaldos no siempre está en línea, el servidor no le envía nada: es el servidor de respaldos el que, al encenderse, trae la copia de `/var/backups/borg` (21.3.6 a 21.3.9). El servidor no guarda ninguna credencial hacia él, de modo que un compromiso del servidor no alcanza las otras dos copias. El servidor de respaldos tampoco guarda las passphrases de Borg: comprueba las copias sin descifrarlas.

El servidor tiene un solo repositorio, y lo que guarda se organiza en grupos de activos: cada respaldo se llama `<host>-<grupo>-<fecha>`, de modo que el grupo es el prefijo de sus archivos dentro del repositorio. El primero, `s.o.` (`s.o` en los nombres, sin el punto final), recoge el sistema operativo y la configuración del despliegue; los manuales de rol añaden los suyos, como el volcado y el filestore de una aplicación, en el mismo repositorio. Cada grupo tiene su propia frecuencia y su propia retención, porque `borg prune` filtra por prefijo. Un solo repositorio significa una passphrase, una clave exportada y una carpeta que sincronizar (21.3.6), y los bloques comunes entre grupos se guardan una vez; a cambio, un repositorio dañado afecta a todos los grupos, y como Borg lo bloquea mientras escribe, los grupos se respaldan uno detrás de otro.

Un rol crea su propio repositorio al lado, en `/var/backups/borg/<repositorio>`, solo cuando sus datos necesitan otra passphrase o la custodia de otra persona, como los datos de clientes de una aplicación. Ese repositorio sigue los mismos permisos de 21.3.6 y viaja en la misma copia, porque el servidor de respaldos trae el directorio entero.

#### 21.2 Qué respaldar

El grupo `s.o.` contiene:

| Ruta | Motivo |
| --- | --- |
| `/etc` | Toda la configuración del despliegue, incluidas las claves de host de SSH, las reglas de UFW, la credencial del relay (20.5) y los respaldos `.ori` |
| `/root` | Las listas de paquetes (21.2.1); se excluye `/root/.cache`, donde Borg guarda su caché |
| `/home` | Las llaves autorizadas y el entorno de los administradores |
| `/usr/local` | Los scripts propios, como el de respaldo |
| `/var/spool/cron` | Los crontabs, si un rol los añade |
| `/var/log` | Valor forense (`audit`, `sudo-io`, `aide`, `apt`); se excluye `/var/log/journal`, que tiene su propia retención (18.1.1) y es diagnóstico, no datos |

No se respalda lo que reinstalan los paquetes (`/usr`, `/bin`, `/sbin`, `/lib`, `/boot`), lo transitorio (`/var/cache`, `/var/tmp`, `/tmp`), el propio repositorio en `/var/backups/borg`, ni `/var/lib/cloud` en imágenes cloud, que guarda en claro la contraseña inicial del aprovisionamiento (5.1). Las bases de datos no se respaldan en bruto: se vuelcan con su herramienta (`pg_dump`, `mariabackup`, `mongodump`) y se respalda el volcado. En el despliegue de referencia, `s.o.` ocupó 65 MB, que quedaron en 3,3 MB comprimidos y deduplicados.

##### 21.2.1 Lista de paquetes instalados

```bash
sudo apt-mark showmanual | sudo tee /root/paquetes-manual.txt > /dev/null
sudo dpkg --get-selections | sudo tee /root/paquetes-completo.txt > /dev/null
```

Quedan en `/root/` y entran en el respaldo. Para restaurar:

```bash
sudo cat /root/paquetes-completo.txt | sudo dpkg --set-selections
sudo apt-get dselect-upgrade
```

La lista se lee con `sudo cat` y no con una redirección (`<`), porque la redirección la abre la shell del usuario, que no puede leer `/root/`.

#### 21.3 BorgBackup

`borg` deduplica por bloques, comprime, cifra de extremo a extremo y verifica integridad, sobre disco local, SSH o servicios compatibles (BorgBase, rsync.net).

##### 21.3.1 Instalación

```bash
sudo apt install -y borgbackup
borg --version
```

Debian 13 instala Borg 1.4. Por sus Recommends llegan también `fuse3` (con `fusermount3`, que es setuid) y `python3-pyfuse3`, que solo sirven para `borg mount`, la exploración de un respaldo como si fuera un directorio. Para restaurar basta `borg extract`; en un servidor mínimo pueden omitirse con `--no-install-recommends`.

##### 21.3.2 Passphrase y repositorio del servidor

El repositorio va cifrado con una passphrase que define el administrador, se guarda en su gestor de contraseñas y, en el servidor, en un archivo que solo lee root, porque los respaldos automáticos (21.3.5) la necesitan sin intervención:

```bash
sudo bash -c 'umask 077; read -rsp "Passphrase de Borg (no se mostrará): " PHRASE && echo && printf "%s" "$PHRASE" > /root/.borg-passphrase'
sudo ls -l /root/.borg-passphrase
```

`read -s` no hace eco ni deja el valor en el historial, y `printf` no añade salto de línea. El `ls` debe mostrar `-rw------- 1 root root`.

Se crea el repositorio, una sola vez, con el nombre corto del servidor (7.1):

```bash
export MIHOST=$(hostname -s)

sudo install -d -m 0700 -o root -g root /var/backups/borg
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" \
    borg init --encryption=repokey-blake2 "/var/backups/borg/${MIHOST:?}"
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" \
    borg info "/var/backups/borg/${MIHOST}" | grep -E 'Encrypted|Repository ID'
```

Debe mostrar `Encrypted: Yes (repokey BLAKE2b)`. El nombre del servidor distingue los repositorios de varios servidores en el servidor de respaldos y en el disco externo. `repokey-blake2` guarda la clave dentro del repositorio, protegida por la passphrase, así que las copias de 21.3.6 se restauran solo con ella; `keyfile-blake2` la deja fuera, para varios administradores o cuando la passphrase no debe estar en el servidor.

El cifrado se fija al crear el repositorio y no se cambia después. Aunque root ya puede leer los originales en el servidor, el repositorio viaja a las otras dos copias, y en un disco externo que se pierde o se roba el cifrado es la única protección de `/etc/shadow`, las claves de host y las credenciales. Sin cifrar, además, Borg tampoco autentica los datos y no detecta manipulaciones.

Se exporta la clave, para no depender de la copia que va dentro del repositorio:

```bash
sudo borg key export --paper "/var/backups/borg/${MIHOST:?}"
```

`borg key export` necesita la ruta del repositorio; sin ella falla con `Invalid location format: ""`. La salida sigue protegida por la passphrase y se guarda en el gestor de contraseñas junto a ella.

> **Importante:** sin la passphrase, o sin la clave si se daña la del repositorio, los respaldos son irrecuperables.

##### 21.3.3 Exclusiones en AIDE

El repositorio, la caché y el estado de Borg cambian con cada respaldo y llenarían el informe diario de AIDE (14.3):

```bash
printf '%s\n' '!/var/backups/borg' '!/root/\.cache/borg' '!/root/\.config/borg' \
  | sudo install -m 0644 -o root -g root /dev/stdin /etc/aide/aide.conf.d/70_local_backups
sudo aide --config /etc/aide/aide.conf --config-check && echo "✓ configuración válida"
```

Tras el primer respaldo se regenera la base (`sudo aideinit -y -f`).

##### 21.3.4 Primer respaldo del grupo `s.o.`

```bash
export MIHOST=$(hostname -s)
export MIGRUPO=s.o

sudo apt-mark showmanual | sudo tee /root/paquetes-manual.txt > /dev/null
sudo dpkg --get-selections | sudo tee /root/paquetes-completo.txt > /dev/null

sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg create \
    --stats --compression zstd,3 --exclude-caches \
    --exclude /root/.cache \
    --exclude /var/log/journal \
    "/var/backups/borg/${MIHOST:?}::${MIHOST}-${MIGRUPO:?}-$(date +%Y-%m-%dT%H%M%S)" \
    /etc /root /home /usr/local /var/spool/cron /var/log
```

El nombre del archivo lleva el host, el grupo y la fecha, para que la retención (21.3.5) pueda filtrar por grupo. `--stats` muestra el tamaño original, el comprimido y el deduplicado.

##### 21.3.5 Automatización

El script recibe el grupo como argumento y define en un `case` sus rutas, exclusiones y retención; un grupo nuevo, como el de un rol, es otra entrada del `case`:

```bash
sudo install -m 0700 -o root -g root /dev/stdin /usr/local/sbin/borg-backup.sh <<'EOF'
#!/bin/bash
# Respaldo Borg de un grupo de activos en el repositorio cifrado del servidor,
# /var/backups/borg/<host>. Uso: borg-backup.sh <grupo>
# Manual de Debian 13 server, sección 21. Un grupo nuevo es otra entrada del "case".
set -euo pipefail
umask 077
export BORG_PASSCOMMAND="cat /root/.borg-passphrase"
HOST=$(hostname -s)
REPO="/var/backups/borg/$HOST"
GRUPO="${1:?uso: borg-backup.sh <grupo>}"
FECHA=$(date +%Y-%m-%dT%H%M%S)

aviso_fallo() {
    printf 'To: root\nSubject: [FALLO] respaldo Borg %s en %s\n\nEl respaldo del grupo %s falló en %s a las %s (línea %s).\nRevisar: journalctl -u borg-backup@%s.service -n 50\n' \
        "$GRUPO" "$HOST" "$GRUPO" "$HOST" "$(date)" "$1" "$GRUPO" | /usr/sbin/sendmail -t || true
}
trap 'aviso_fallo $LINENO' ERR

# Listas de paquetes (21.2.1): se reescriben solo si cambian, para no ensuciar AIDE
lista() {
    local destino=$1 tmp; shift
    tmp=$(mktemp)
    "$@" > "$tmp"
    if cmp -s "$tmp" "$destino"; then rm -f "$tmp"; else install -m 0600 "$tmp" "$destino"; rm -f "$tmp"; fi
}

case "$GRUPO" in
    s.o)
        lista /root/paquetes-manual.txt apt-mark showmanual
        lista /root/paquetes-completo.txt dpkg --get-selections
        RUTAS=(/etc /root /home /usr/local /var/spool/cron /var/log)
        EXCLUIR=(--exclude /root/.cache --exclude /var/log/journal)
        RETENCION=(--keep-daily=7 --keep-weekly=4 --keep-monthly=6)
        ;;
    *)
        echo "grupo desconocido: $GRUPO" >&2
        false
        ;;
esac

# --lock-wait: si otro grupo está respaldando, esperar hasta 1 h en lugar de fallar.
# Código 1 de borg es un aviso (p. ej., un log que cambió durante la lectura): no es un fallo.
rc=0
borg create --lock-wait 3600 --stats --compression zstd,3 --exclude-caches \
    "${EXCLUIR[@]}" "$REPO::$HOST-$GRUPO-$FECHA" "${RUTAS[@]}" || rc=$?
[ "$rc" -le 1 ] || false

borg prune --lock-wait 3600 --list --glob-archives "$HOST-$GRUPO-*" "${RETENCION[@]}" "$REPO"
borg compact --lock-wait 3600 "$REPO"

# Lectura para la copia que trae el servidor de respaldos (21.3.6): Borg escribe en 0600 por
# el umask 077, y el grupo borg-lectura, heredado por el setgid de los directorios, necesita leer.
chmod -R g+rX "$REPO"
EOF
sudo bash -n /usr/local/sbin/borg-backup.sh && echo "✓ sintaxis"
```

- Las listas de paquetes se regeneran en cada ejecución, pero el archivo solo se reescribe si cambió: AIDE vigila `/root` y, de lo contrario, marcaría cambios cada día.
- `borg create` termina con código 1 cuando solo hubo avisos, como un log que cambió mientras se leía; tratarlo como fallo enviaría alertas falsas cada noche. Los códigos 2 o mayores sí son fallos.
- Ante cualquier fallo, `trap ERR` envía un correo a root (20) con el grupo y la línea.
- `borg prune` filtra por el prefijo del grupo, así que cada grupo conserva su propia retención (7 diarios, 4 semanales y 6 mensuales en `s.o`, unos 17 archivos) sin tocar los archivos de los demás grupos.
- `--lock-wait 3600` hace que, si dos grupos coinciden, el segundo espere al primero en lugar de fallar.

Las unidades son plantillas: una instancia por grupo, con el grupo como argumento.

```bash
sudo install -m 0644 -o root -g root /dev/stdin /etc/systemd/system/borg-backup@.service <<'EOF'
[Unit]
Description=Respaldo Borg del grupo %i

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/borg-backup.sh %i
Nice=19
IOSchedulingClass=idle
EOF

sudo install -m 0644 -o root -g root /dev/stdin /etc/systemd/system/borg-backup@.timer <<'EOF'
[Unit]
Description=Respaldo Borg diario del grupo %i

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=30m
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now borg-backup@s.o.timer
systemctl list-timers 'borg-backup@*' --no-pager
```

`Nice=19` e `IOSchedulingClass=idle` evitan competir con los servicios. El repositorio es local, así que el servicio no espera a la red. `RandomizedDelaySec=30m` reparte la carga y `Persistent=true` ejecuta al arrancar el respaldo que se perdió con el servidor apagado. Un grupo con otra frecuencia, por ejemplo cada 6 horas, cambia su `OnCalendar` en un drop-in de su instancia (`sudo systemctl edit borg-backup@<grupo>.timer`).

Prueba inmediata, sin esperar a la noche:

```bash
sudo systemctl start borg-backup@s.o.service
systemctl show borg-backup@s.o.service -p Result
sudo journalctl -u borg-backup@s.o.service -n 30 --no-pager | grep -E 'Archive name|This archive|Keeping|Pruning'
```

`Result=success` y un archivo nuevo en `borg list`. El aviso de fallo se prueba llamando al script con un grupo que no existe (`sudo /usr/local/sbin/borg-backup.sh grupo-inexistente`): debe terminar con error y llegar el correo `[FALLO]`.

##### 21.3.6 Cuenta de solo lectura para el servidor de respaldos

Las copias de 21.1 las trae `<servidor-respaldos>`. El servidor solo le abre una cuenta propia, con el nombre del servidor de respaldos, sin contraseña y con una única llave que solo puede leer `/var/backups/borg` mediante `rrsync -ro`: sin shell, sin reenvíos y sin escritura. Las copias son los mismos repositorios: solo el servidor escribe en ellos, y en las copias únicamente se lee y se restaura. Como se copia el directorio entero, un repositorio que añada un rol al lado del `<host>` (21.1) viaja sin tocar nada en el servidor de respaldos.

```bash
export MIHOST=$(hostname -s)
export MIRESPALDOS=<servidor-respaldos>   # nombre corto del servidor de respaldos

sudo apt install -y rsync
command -v rrsync

sudo groupadd --system borg-lectura
sudo useradd --system --create-home --home-dir "/var/lib/${MIRESPALDOS:?}" --shell /bin/sh \
    --user-group --groups borg-lectura --comment "Copia de los repositorios Borg" "${MIRESPALDOS}"
sudo install -d -m 0700 -o "${MIRESPALDOS}" -g "${MIRESPALDOS}" "/var/lib/${MIRESPALDOS}/.ssh"
```

`rsync` trae `rrsync` en `/usr/bin`. La cuenta necesita `/bin/sh` porque `sshd` ejecuta la orden forzada a través de la shell del usuario; su contraseña queda bloqueada (`!`), así que no admite otro acceso que la llave.

El grupo `borg-lectura` lee el directorio y el repositorio. Borg escribe sus archivos en `0600` por el `umask 077` de 21.3.5, así que el bit setgid de los directorios hace que lo nuevo herede el grupo, y la última línea del script de 21.3.5 devuelve la lectura al grupo después de cada respaldo:

```bash
sudo chgrp borg-lectura /var/backups/borg
sudo chmod 0750 /var/backups/borg
sudo chgrp -R borg-lectura "/var/backups/borg/${MIHOST:?}"
sudo find "/var/backups/borg/${MIHOST}" -type d -exec chmod g+rxs {} +
sudo chmod -R g+rX "/var/backups/borg/${MIHOST}"
sudo grep -n 'chmod -R g+rX' /usr/local/sbin/borg-backup.sh
```

Si el script se instaló antes de esta versión del manual y el `grep` no devuelve nada, se le añade al final la línea `chmod -R g+rX "$REPO"`. Solo root escribe en el repositorio; el grupo solo lee.

La llave pública la genera el servidor de respaldos (21.3.7). Se autoriza con la orden forzada y `restrict`, que retira la terminal y todos los reenvíos:

```bash
export LLAVE_PUB='<llave pública de root en el servidor de respaldos, de 21.3.7>'

printf 'restrict,command="/usr/bin/rrsync -ro /var/backups/borg/" %s\n' "${LLAVE_PUB:?}" \
  | sudo install -m 0600 -o "${MIRESPALDOS:?}" -g "${MIRESPALDOS}" /dev/stdin \
    "/var/lib/${MIRESPALDOS}/.ssh/authorized_keys"
```

Por último, la cuenta entra en `AllowUsers` (10.3) y se comprueba que lee el repositorio:

```bash
sudo sed -i "s/^AllowUsers .*/& ${MIRESPALDOS:?}/" /etc/ssh/sshd_config.d/01-hardening.conf
sudo grep '^AllowUsers' /etc/ssh/sshd_config.d/01-hardening.conf
sudo sshd -t && sudo systemctl reload ssh

sudo -u "${MIRESPALDOS}" find /var/backups/borg -type f ! -readable | wc -l   # 0
sudo aideinit -y -f
```

El `find` cuenta los archivos que la cuenta no puede leer y debe dar 0. La base de AIDE se regenera porque cambiaron `/etc/passwd`, `/etc/group`, la configuración de SSH y el script.

##### 21.3.7 Copia en el servidor de respaldos

El servidor de respaldos guarda, por cada servidor, una copia vigente de su `/var/backups/borg` y las copias anteriores como snapshots btrfs de solo lectura, hasta un límite de espacio. No guarda las passphrases de Borg: la comprobación es `borg check --repository-only`, que verifica la estructura y las sumas del repositorio sin descifrarlo. El destino necesita btrfs, tanto en el servidor de respaldos como en `<disco-externo>`.

```text
/mnt/backups/<host>/           subvolumen
├── vigente/                   subvolumen: copia de /var/backups/borg del servidor
│   ├── <host>/                repositorio del sistema (21.3.2)
│   └── <repositorio>/         repositorio de un rol, si existe
└── anteriores/
    └── <fecha>/               snapshot de solo lectura de vigente/ antes de cada copia
```

En el servidor de respaldos, una llave de root sin frase para cada servidor y su entrada en `/root/.ssh/config`. La clave de host se acepta solo si coincide con la huella tomada en el propio servidor (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`) o anotada en su bitácora:

```bash
export MIHOST=<host>          # nombre corto del servidor
export MIIP=<ip>              # dirección del servidor
export MIPUERTO=17177         # puerto SSH del servidor (10.1)
export MIRESPALDOS=$(hostname -s)

sudo apt install -y rsync borgbackup
sudo ssh-keygen -q -t ed25519 -N '' -C "root@${MIRESPALDOS}: copia de los repositorios Borg de ${MIHOST:?}" \
    -f "/root/.ssh/${MIHOST}"
sudo cat "/root/.ssh/${MIHOST}.pub"          # es la LLAVE_PUB de 21.3.6

ssh-keyscan -p "${MIPUERTO:?}" -t ed25519 "${MIIP:?}" 2>/dev/null | tee /tmp/kh | ssh-keygen -lf -
```

Si la huella coincide, se guarda y se crea el alias:

```bash
sudo bash -c 'cat /tmp/kh >> /root/.ssh/known_hosts' && rm /tmp/kh
sudo tee -a /root/.ssh/config > /dev/null <<EOF
Host ${MIHOST}
    HostName ${MIIP}
    Port ${MIPUERTO}
    User ${MIRESPALDOS}
    IdentityFile /root/.ssh/${MIHOST}
    IdentitiesOnly yes
    BatchMode yes
    StrictHostKeyChecking yes
    ConnectTimeout 30
EOF
sudo chmod 600 /root/.ssh/config
sudo rsync --list-only "${MIHOST}:"
```

`--list-only` debe listar los repositorios de `/var/backups/borg` del servidor. `BatchMode` y `StrictHostKeyChecking yes` hacen que una llave rechazada o una clave de host cambiada den un error inmediato en lugar de una pregunta que nadie contesta.

El script recibe el servidor y, para un disco externo (21.3.9), su nombre:

```bash
sudo install -m 0700 -o root -g root /dev/stdin /usr/local/sbin/copia-borg.sh <<'EOF'
#!/bin/bash
# Copia de los repositorios Borg de un servidor (manual de Debian 13 server, 21.3.7).
# Uso: copia-borg.sh <host>                   trae /var/backups/borg/ de <host> por rrsync de solo lectura
#      copia-borg.sh <host> <disco-externo>   copia en el disco, ya abierto y montado en /mnt/<disco-externo>,
#                                             la copia vigente de <host> en este servidor
# La copia vigente queda en <base>/<host>/vigente y las anteriores como snapshots btrfs de solo lectura
# en <base>/<host>/anteriores, hasta LIMITE_GB; al pasar AVISO_PCT de ese límite se avisa por correo.
# Sin passphrase de Borg: la comprobación es borg check --repository-only, que no la necesita.
set -euo pipefail
umask 077
export BORG_RELOCATED_REPO_ACCESS_IS_OK=yes
LIMITE_GB=100          # por servidor: copia vigente y copias anteriores
AVISO_PCT=85
LOCAL=/mnt/backups     # btrfs del servidor de respaldos
HOST="${1:?uso: copia-borg.sh <host> [<disco-externo>]}"
DISCO="${2:-}"
FECHA=$(date +%Y-%m-%dT%H%M%S)

if [ -z "$DISCO" ]; then
    DESTINO=$(hostname -s)
    BASE="$LOCAL/$HOST"
    ORIGEN="$HOST:"                      # alias de /root/.ssh/config; rrsync parte de /var/backups/borg/
else
    DESTINO=$DISCO
    BASE="/mnt/$DISCO/$HOST"
    ORIGEN="$LOCAL/$HOST/vigente/"
    mountpoint -q "/mnt/$DISCO" || { echo "$DISCO no está montado en /mnt/$DISCO" >&2; exit 1; }
fi
VIGENTE="$BASE/vigente"
ANTERIORES="$BASE/anteriores"

correo() { printf 'To: root\nSubject: %s\n\n%s\n' "$1" "$2" | /usr/sbin/sendmail -t || true; }
fallo() {
    correo "[FALLO] copia Borg de $HOST en $DESTINO" "La copia de $HOST en $DESTINO falló el $(date): $1.
Revisar: journalctl -u copia-borg@$HOST.service -n 80, o la salida de la ejecución manual."
    echo "FALLO: $1" >&2
    exit 1
}
trap 'fallo "error inesperado en la línea $LINENO"' ERR

# Primera vez en un destino: estructura de subvolúmenes
if [ ! -d "$BASE" ]; then
    btrfs subvolume create "$BASE" >/dev/null
    btrfs subvolume create "$VIGENTE" >/dev/null
    mkdir -m 700 "$ANTERIORES"
    chmod 700 "$BASE" "$VIGENTE"
fi

# 1. La copia vigente pasa a snapshot de solo lectura antes de tocarla
PREVIO=""
if [ -n "$(ls -A "$VIGENTE")" ]; then
    PREVIO="$ANTERIORES/$FECHA"
    btrfs subvolume snapshot -r "$VIGENTE" "$PREVIO" >/dev/null
fi

restaurar() {
    [ -n "$PREVIO" ] || return 0
    btrfs subvolume delete "$VIGENTE" >/dev/null
    btrfs subvolume snapshot "$PREVIO" "$VIGENTE" >/dev/null
}

# Cada repositorio de la copia debe pasar borg check; tiene que haber al menos uno
comprobar() {
    local n=0 c
    for c in "$VIGENTE"/*/config; do
        [ -f "$c" ] || continue
        borg check --repository-only "${c%/config}" || return 1
        n=$((n + 1))
    done
    [ "$n" -gt 0 ]
}

# 2. Traer y 3. comprobar. Si algo falla, vuelve la copia anterior: no se propaga
# un repositorio dañado ni uno copiado mientras el servidor escribía en él.
trap - ERR
if ! rsync -rlpt --delete "$ORIGEN" "$VIGENTE/"; then
    restaurar
    fallo "rsync no terminó; se conserva la copia anterior, si existía"
fi
if ! comprobar; then
    restaurar
    fallo "borg check rechazó la copia nueva; se conserva la copia anterior, si existía"
fi
trap 'fallo "error inesperado en la línea $LINENO"' ERR

# 4. Espacio: vigente y anteriores no pasan de LIMITE_GB; se borran los snapshots más viejos
uso_bytes() { btrfs filesystem du -s --raw "$BASE" | awk 'NR==2 {print $2 + $3}'; }
LIMITE=$((LIMITE_GB * 1024 * 1024 * 1024))
while [ "$(uso_bytes)" -gt "$LIMITE" ]; do
    mapfile -t LISTA < <(ls -1 "$ANTERIORES")
    [ "${#LISTA[@]}" -gt 0 ] || break
    btrfs subvolume delete "$ANTERIORES/${LISTA[0]}" >/dev/null
    btrfs subvolume sync "$BASE" >/dev/null
done

USO=$(uso_bytes)
PCT=$((USO * 100 / LIMITE))
mapfile -t LISTA < <(ls -1 "$ANTERIORES")
RESUMEN="Copia vigente de $HOST comprobada en $DESTINO; ${#LISTA[@]} copias anteriores; uso $((USO / 1024 / 1024)) MiB de ${LIMITE_GB} GiB (${PCT} %)."
echo "$RESUMEN"
if [ "$PCT" -ge "$AVISO_PCT" ]; then
    correo "[AVISO] copia Borg de $HOST en $DESTINO al ${PCT} %" "$RESUMEN
Superó el ${AVISO_PCT} % del límite: las copias anteriores más viejas se están descartando o se descartarán pronto."
fi
EOF
sudo bash -n /usr/local/sbin/copia-borg.sh && echo "✓ sintaxis"
```

- El snapshot de solo lectura se toma antes de copiar, así que una copia fallida o un repositorio dañado en el servidor no alcanzan a la anterior: el script la restaura y avisa con un correo `[FALLO]` a root (20).
- Si `rsync` coincide con un respaldo en curso, la copia puede quedar a medias y `borg check` la rechaza. Por eso la copia se programa fuera del horario de 21.3.5 y de los respaldos del rol.
- `btrfs filesystem du` suma lo exclusivo y lo compartido una sola vez, que es el espacio real: los snapshots comparten los bloques que no cambiaron. Cuando el conjunto pasa de `LIMITE_GB` se descartan los snapshots más viejos, y al superar `AVISO_PCT` llega un correo `[AVISO]`.
- `BORG_RELOCATED_REPO_ACCESS_IS_OK` evita la pregunta de Borg por un repositorio que conoce en otra ruta, porque la copia y sus snapshots tienen el mismo identificador.

Las unidades son plantillas, una instancia por servidor:

```bash
sudo install -m 0644 -o root -g root /dev/stdin /etc/systemd/system/copia-borg@.service <<'EOF'
[Unit]
Description=Copia de los repositorios Borg de %i
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/copia-borg.sh %i
Nice=19
IOSchedulingClass=idle
EOF

sudo install -m 0644 -o root -g root /dev/stdin /etc/systemd/system/copia-borg@.timer <<'EOF'
[Unit]
Description=Copia semanal de los repositorios Borg de %i

[Timer]
OnCalendar=Fri *-*-* 21:05:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now "copia-borg@${MIHOST:?}.timer"
sudo systemctl start "copia-borg@${MIHOST}.service"
sudo journalctl -u "copia-borg@${MIHOST}.service" -n 5 --no-pager | grep 'Copia vigente'
```

La primera ejecución deja la línea `Copia vigente de <host> comprobada…; 0 copias anteriores`, y la segunda crea el primer snapshot en `anteriores/`. Un servidor de respaldos que pasa el viernes apagado ejecuta la copia al encender, por `Persistent=true`. `/mnt/backups` y los discos externos se excluyen de AIDE (14.3), que no debe recorrer los repositorios; tras instalar el script y las unidades se regenera su base.

##### 21.3.8 Opcional: encendido programado del servidor de respaldos

Un servidor de respaldos físico puede pasar apagado toda la semana y encenderse solo para la copia, sin depender de otro equipo que le envíe un paquete Wake-on-LAN. El reloj de la placa (RTC) lo enciende desde el apagado total, y al terminar la copia se apaga. Antes se comprueba que la placa lo admite: se programa una alarma a cinco minutos, se apaga y se espera.

```bash
timedatectl | grep 'RTC in local TZ'          # no: el RTC está en UTC y rtcwake usa -u
sudo rtcwake -m no -u -t $(( $(date +%s) + 300 )) && sudo systemctl poweroff
```

Si no enciende, la opción de encendido por RTC de la BIOS está desactivada o la placa no lo soporta, y esta sección no aplica.

Un script programa la alarma del próximo encendido en cada apagado, y otro apaga al terminar la copia, solo si el arranque lo provocó esa alarma y nadie tiene sesión abierta:

```bash
sudo install -m 0700 -o root -g root /dev/stdin /usr/local/sbin/programar-encendido.sh <<'EOF'
#!/bin/bash
# Programa en el RTC el próximo encendido del servidor de respaldos (manual de Debian, 21.3.8).
# Lo ejecuta encendido-programado.service en cada apagado. El RTC está en UTC (-u).
set -euo pipefail
DIA=friday        # en inglés, como lo entiende date
HORA=21:00        # unos minutos antes del OnCalendar de copia-borg@.timer
ahora=$(date +%s)
objetivo=$(date -d "$DIA $HORA" +%s)    # es hoy si hoy es ese día
[ "$objetivo" -gt $((ahora + 300)) ] || objetivo=$(date -d "next $DIA $HORA" +%s)
rtcwake -m no -u -t "$objetivo"
install -d -m 700 /var/lib/copia-borg
echo "$objetivo" > /var/lib/copia-borg/proximo-encendido
echo "próximo encendido: $(date -d "@$objetivo")"
EOF

sudo install -m 0700 -o root -g root /dev/stdin /usr/local/sbin/apagar-si-programado.sh <<'EOF'
#!/bin/bash
# Apaga el servidor de respaldos al terminar la copia (manual de Debian, 21.3.8), solo si este
# arranque lo provocó la alarma del RTC (hasta 10 min de diferencia) y nadie tiene sesión abierta.
# Antes deja terminar apt y AIDE.
set -u
F=/var/lib/copia-borg/proximo-encendido
[ -r "$F" ] || { echo "sin alarma registrada: no se apaga"; exit 0; }
alarma=$(cat "$F")
arranque=$(( $(date +%s) - $(cut -d. -f1 /proc/uptime) ))
d=$((arranque - alarma))
[ "${d#-}" -le 600 ] || { echo "arranque no programado: no se apaga"; exit 0; }

for _ in $(seq 1 120); do   # hasta 60 min
    ocupado=0
    for u in apt-daily.service apt-daily-upgrade.service dailyaidecheck.service; do
        systemctl is-active --quiet "$u" && ocupado=1
    done
    [ "$ocupado" -eq 0 ] && break
    sleep 30
done

[ -z "$(who)" ] || { echo "hay sesiones abiertas: no se apaga"; exit 0; }
echo "apagando tras la copia programada"
systemctl poweroff
EOF
```

La alarma se programa al apagar y no al encender, para que la siguiente quede puesta aunque el equipo se apague a mano. Un encendido manual o por Wake-on-LAN no cae cerca de la alarma, así que el equipo no se apaga solo después de la copia; tampoco lo hace si hay alguien conectado. El chequeo diario de AIDE (14.3) y las actualizaciones automáticas (16) corren poco después de cada encendido por `Persistent=true`, y el script espera a que terminen.

```bash
sudo install -m 0644 -o root -g root /dev/stdin /etc/systemd/system/encendido-programado.service <<'EOF'
[Unit]
Description=Programa en el RTC el próximo encendido al apagar
After=local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/true
ExecStop=/usr/local/sbin/programar-encendido.sh

[Install]
WantedBy=multi-user.target
EOF

sudo mkdir -p "/etc/systemd/system/copia-borg@${MIHOST:?}.service.d"
sudo install -m 0644 -o root -g root /dev/stdin "/etc/systemd/system/copia-borg@${MIHOST}.service.d/apagado.conf" <<'EOF'
[Service]
ExecStopPost=/usr/local/sbin/apagar-si-programado.sh
TimeoutStopSec=90min
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now encendido-programado.service
```

`ExecStopPost` va en un drop-in del último servidor que se copia, no en la plantilla, para que con varios servidores el apagado espere a todos. Se ejecuta aunque la copia falle, porque el correo `[FALLO]` ya salió. La prueba completa es dejar el equipo apagado y comprobar al día siguiente en el journal el arranque a la hora de la alarma, la copia y el apagado, y en `/var/lib/copia-borg/proximo-encendido` la alarma de la semana siguiente.

##### 21.3.9 Copia en un disco externo

`<disco-externo>` es un disco USB con LUKS2 y btrfs, conectado al servidor de respaldos y montado solo bajo demanda. Puede haber varios, cada uno con su nombre; todos reciben la misma copia y se actualizan a mano, desde la copia vigente del servidor de respaldos. Se prepara una vez, con el disco identificado por su ruta estable:

```bash
export MIDISCO=<disco-externo>
export MIDISPOSITIVO=/dev/disk/by-id/<id>-part1   # de ls -l /dev/disk/by-id/, nunca /dev/sdX

sudo cryptsetup luksFormat --type luks2 "${MIDISPOSITIVO:?}"
export MIUUID=$(sudo cryptsetup luksUUID "${MIDISPOSITIVO}")
sudo cryptsetup open "/dev/disk/by-uuid/${MIUUID:?}" "${MIDISCO:?}"
sudo mkfs.btrfs -L "${MIDISCO}" "/dev/mapper/${MIDISCO}"
sudo install -d -m 0700 "/mnt/${MIDISCO}"
echo "${MIDISCO} UUID=${MIUUID} none luks,noauto" | sudo tee -a /etc/crypttab
echo "/dev/mapper/${MIDISCO} /mnt/${MIDISCO} btrfs noauto,noatime 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo cryptsetup luksHeaderBackup "${MIDISPOSITIVO}" --header-backup-file "/mnt/backups/luks-${MIDISCO}.img"
printf '%s\n' "!/mnt/${MIDISCO}" | sudo tee -a /etc/aide/aide.conf.d/70_local_backups
```

La contraseña de LUKS se guarda en el gestor de contraseñas y no en el servidor de respaldos. `noauto` evita que el arranque se detenga a pedirla cuando el disco no está conectado. Sin la cabecera de LUKS el disco es irrecuperable, de ahí su respaldo.

Cada vez que se conecta:

```bash
export MIHOST=<host>
export MIDISCO=<disco-externo>
export MIUUID=<uuid-luks>     # de /etc/crypttab

sudo cryptsetup open "/dev/disk/by-uuid/${MIUUID:?}" "${MIDISCO:?}"
sudo mount "/mnt/${MIDISCO}"
sudo /usr/local/sbin/copia-borg.sh "${MIHOST:?}" "${MIDISCO}"
sudo umount "/mnt/${MIDISCO}" && sudo cryptsetup close "${MIDISCO}"
```

La copia sigue el mismo esquema de 21.3.7, con su propia historia de snapshots y su propio límite en el disco. Conviene hacerla con el servidor de respaldos encendido a mano: si encendió por la alarma de 21.3.8, se apaga al terminar su copia programada.

Para restaurar desde una copia, en el servidor de respaldos o en otro equipo con el disco abierto, Borg pide la passphrase del repositorio, que sale del gestor de contraseñas; `sudo` limpia el entorno (9.3.1), así que la variable va en la propia orden. Las copias anteriores están en un snapshot de solo lectura, donde Borg no puede crear su bloqueo, así que se leen con `--bypass-lock`:

```bash
export MIHOST=<host>
export COPIA="/mnt/backups/${MIHOST:?}"

sudo BORG_RELOCATED_REPO_ACCESS_IS_OK=yes borg list "${COPIA}/vigente/${MIHOST}"
sudo BORG_RELOCATED_REPO_ACCESS_IS_OK=yes borg list --bypass-lock "${COPIA}/anteriores/<fecha>/${MIHOST}"
sudo BORG_RELOCATED_REPO_ACCESS_IS_OK=yes borg check --verify-data "${COPIA}/vigente/${MIHOST}"
```

La prueba de restauración de 21.5 se hace también, de vez en cuando, desde una copia: es la que queda si se pierde el servidor.

#### 21.4 Verificación del repositorio

```bash
export MIHOST=$(hostname -s)
export MIGRUPO=s.o
export BORG_REPO="/var/backups/borg/${MIHOST:?}"

# Archivos del repositorio, todos o solo los de un grupo
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list "${BORG_REPO}"
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list --glob-archives "${MIHOST}-${MIGRUPO:?}-*" "${BORG_REPO}"

# Detalle de un archivo
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg info "${BORG_REPO}::<archivo>"

# Integridad, leyendo todos los datos
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg check --verify-data "${BORG_REPO}"
```

`sudo` limpia el entorno (9.3.1), así que la ruta se pasa como argumento y `BORG_PASSCOMMAND` se declara en la propia orden. `borg check --verify-data` detecta corrupción silenciosa; en repositorios grandes tarda horas y se ejecuta una vez al mes.

#### 21.5 Prueba de restauración

Mensual y obligatoria. Se extraen archivos conocidos en un directorio temporal y se comparan con los originales:

```bash
export MIHOST=$(hostname -s)
export ARCHIVO="<archivo>"   # un archivo del grupo s.o, de borg list (21.4)

D=$(mktemp -d)
cd "${D}"
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg extract \
    "/var/backups/borg/${MIHOST:?}::${ARCHIVO:?}" \
    etc/ssh/sshd_config.d/01-hardening.conf etc/ufw/user.rules root/paquetes-manual.txt
sudo cmp etc/ssh/sshd_config.d/01-hardening.conf /etc/ssh/sshd_config.d/01-hardening.conf && echo "✓ SSH"
sudo cmp etc/ufw/user.rules /etc/ufw/user.rules && echo "✓ UFW"
sudo cmp root/paquetes-manual.txt /root/paquetes-manual.txt && echo "✓ paquetes"
cd / && sudo rm -rf "${D}"
```

Un archivo cambiado después del respaldo puede diferir sin que sea un fallo; lo que importa es que la extracción funcione y estén los archivos críticos. Se comprueba también que lo excluido no entró y que lo sensible sí:

```bash
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list \
    "/var/backups/borg/${MIHOST:?}::${ARCHIVO:?}" | grep -cE ' root/\.cache| var/log/journal'   # 0
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list \
    "/var/backups/borg/${MIHOST:?}::${ARCHIVO:?}" | grep -c 'etc/shadow$'                      # 1
```

El resultado se anota en la bitácora del servidor.

#### 21.6 Restauración del servidor desde cero

1. Instalar Debian 13 con las etapas I y II, hasta tener SSH.
2. Instalar borgbackup.
3. Recuperar del gestor de contraseñas la passphrase y la clave exportada (21.3.2).
4. Copiar el repositorio del servidor desde `<servidor-respaldos>` o `<disco-externo>`, de `vigente/<host>` o de un snapshot de `anteriores/` (21.3.7), y listarlo: `borg list <repositorio>`. Si la clave del repositorio está dañada, importar la exportada con `borg key import --paper <repositorio>`.
5. Restaurar las rutas del grupo `s.o.` (21.2) desde su archivo más reciente: `sudo borg extract <repositorio>::<host>-s.o-<fecha>`.
6. Reinstalar paquetes: `sudo cat /root/paquetes-completo.txt | sudo dpkg --set-selections && sudo apt-get dselect-upgrade` (21.2.1).
7. Restaurar los grupos del rol.
8. Verificar los servicios.

Este procedimiento se guarda donde siga accesible con el servidor caído (22.3.2).

### 22. Documentación operativa

Lo que se configuró, por qué y cómo operarlo, para que el servidor no dependa de quien lo desplegó. No se repite el manual: se documenta lo propio de cada servidor, es decir, su inventario, su diagrama y su runbook. La documentación vive fuera del servidor (repositorio, wiki o gestor de contraseñas), lo genérico queda en el manual base y lo del rol en el suyo, y se actualiza en el mismo momento que el sistema, porque la desactualizada es peor que ninguna.

#### 22.1 Inventario del servidor

| Campo                        | Ejemplo                                                             | Origen                                     |
|------------------------------|---------------------------------------------------------------------|--------------------------------------------|
| Hostname                     | `villonaco`                                                         | `hostnamectl`                              |
| FQDN                         | `villonaco.lan`                                                     | `hostname -f`                              |
| Rol                          | "Servidor base — sin rol específico aún"                            | manual / decisión del operador             |
| Ubicación física / proveedor | "VM local en estación de trabajo del admin" / "VPS en Hetzner FSN1" | conocida por el operador                   |
| Sistema operativo            | "Debian 13.8 Trixie"                                                | `cat /etc/os-release`                      |
| IP de gestión                | `192.168.1.10/24`                                                   | `ip -brief address`                        |
| Puerto SSH                   | `17177`                                                             | `/etc/ssh/sshd_config.d/01-hardening.conf` |
| Usuario administrativo       | `usuario`                                                           | manual                                     |
| Fecha de despliegue          | `2026-05-27`                                                        | conocida por el operador                   |
| Versión del manual seguido   | `v0.11`                                                             | historial del manual                       |

```bash
hostnamectl
cat /etc/os-release | grep -E '^(NAME|VERSION)='
ip -brief address
sudo grep -E '^(Port|AllowUsers)' /etc/ssh/sshd_config.d/01-hardening.conf
sudo apt-mark showmanual | wc -l    # número de paquetes instalados manualmente
uptime -s                            # fecha del último arranque
```

#### 22.2 Diagrama lógico

Responde de un vistazo qué puertos están expuestos, desde dónde se entra y qué corre. En un servidor base es casi trivial:

```text
            Internet / LAN
                  │
                  ▼
        ┌─────────────────────┐
        │ UFW (deny incoming) │
        │ Solo: SSH/17177     │
        └─────────┬───────────┘
                  │
        ┌─────────▼───────────┐
        │  villonaco          │
        │  Debian 13 Trixie   │
        │  192.168.1.10       │
        ├─────────────────────┤
        │ Servicios activos:  │
        │  - sshd (17177)     │
        │  - chrony           │
        │  - fail2ban         │
        │  - apparmor         │
        │  - msmtp (saliente) │
        └─────────────────────┘
                  │
                  ▼ (correo saliente)
            smtp.gmail.com:587
```

Excalidraw, draw.io o Mermaid sirven para versiones gráficas; importa más usar siempre la misma. Debe decir el hostname y la IP, los puertos expuestos y a qué redes, los servicios, con qué otros sistemas habla (relay, respaldos, monitoreo) y por dónde entra la administración. Lo demás vive en el manual o en el inventario de paquetes (21.2.1).

#### 22.3 Runbook básico

Tres operaciones que aplican a cualquier servidor; las del rol van en su manual.

##### 22.3.1 Reinicio controlado

```bash
# 1. Verificar que no hay actualizaciones a medio aplicar
sudo apt list --upgradable
ls /var/run/reboot-required 2>/dev/null && echo "Reinicio pendiente" || echo "Sin reinicio pendiente"

# 2. Notificar a usuarios conectados (si hubiese)
who
sudo wall "Reinicio en 1 minuto" 2>/dev/null || true

# 3. Verificar servicios críticos antes
sudo systemctl --failed
sudo systemctl is-active ssh chrony fail2ban

# 4. Ejecutar el reinicio
sudo reboot

# 5. Tras reconectar, verificar que todo arrancó bien
sudo systemctl --failed
sudo journalctl -b -p err --no-pager | head -20
```

Con LUKS y desbloqueo remoto (2.4), el desbloqueo va antes de que responda el SSH normal.

##### 22.3.2 Restauración tras desastre

Resumen de 21.6, para tenerlo a mano en mitad de un incidente:

1. Instalar Debian 13 con las etapas I y II.
2. Instalar borgbackup y recuperar del gestor de contraseñas la passphrase y la clave exportada.
3. Copiar el repositorio del servidor desde el servidor de respaldos o el disco externo y listarlo: `borg list <repositorio>`.
4. Restaurar las rutas del grupo `s.o.` (21.2): `borg extract <repo>::<host>-s.o-<fecha>`.
5. Reinstalar paquetes:

   ```bash
   sudo cat /root/paquetes-completo.txt | sudo dpkg --set-selections
   sudo apt-get dselect-upgrade
   ```

6. Restaurar los datos del rol.
7. Verificar los servicios.

El runbook dice dónde están la passphrase del repositorio y el acceso al servidor de respaldos; sin ellos no hay restauración.

##### 22.3.3 Acceso de emergencia

- Cómo entrar por la consola física o la del hipervisor o el proveedor, y cómo arrancar en modo rescate.
- Root está bloqueado (9.6): la emergencia se resuelve con `sudo` desde el usuario administrativo.
- IP propia baneada por fail2ban: desde la consola, `sudo fail2ban-client unban <ip>` o `sudo fail2ban-client set sshd unbanip <ip>`.
- Bloqueo de `pam_faillock` (9.3.3): con `silent` parece una contraseña incorrecta. SSH por llave sigue funcionando, pero sin root nadie puede ejecutar `faillock --reset` salvo desde una shell de root ya abierta, y la consola pasa por el mismo `pam_faillock`. Se espera a `unlock_time`; el modo rescate solo si el bloqueo persiste.

#### 22.4 Mantenimiento de la documentación

Cada cambio de IP, puerto, servicio o firewall actualiza inventario y diagrama en el momento. Las pruebas de restauración (21.5) sirven para comprobar que el runbook sigue valiendo. La versión del manual usada en el despliegue queda en el inventario, para saber más adelante qué decisiones se tomaron.

### Cierre de la Etapa IV

- Actualizaciones automáticas con aviso y reinicio controlado.
- Reloj sincronizado.
- Logs persistentes, acotados y retenidos tres meses.
- Alertas por correo cuando algo falla.
- Correo del sistema reenviado a una dirección que alguien lee.
- Respaldos diarios cifrados, locales y remotos, con prueba de restauración.
- Inventario, diagrama y runbook fuera del servidor.

Con esto termina el manual base. Los manuales de rol (correo, web, bases de datos, aplicaciones) heredan y referencian esta base: usuarios, SSH, firewall, fail2ban, sysctl, AppArmor, actualizaciones, hora, logs, monitoreo, correo saliente y respaldos.

---

## Anexo A — Diagnóstico rápido post-incidente

Comandos agrupados por síntoma para formarse una imagen del sistema y descartar las causas comunes. Se asume acceso por SSH o consola con `sudo`; si SSH falla, se entra por la consola física o la del proveedor (22.3.3).

### A.1 Imagen general del sistema

```bash
# Estado general en una pantalla
uptime
sudo systemctl --failed
free -h
df -hT
sudo ss -tlnp | head -20

# Kernel y arranque actual
uname -a
who -b                          # cuándo arrancó el sistema
last -a | head -10              # últimos logins

# Eventos recientes con prioridad warning o peor
sudo journalctl --since "1 hour ago" -p warning --no-pager | tail -50
```

Lo que salte a la vista (un servicio caído, un disco lleno, una carga alta) marca la pista.

### A.2 Síntoma: el servidor está lento o no responde

```bash
# Carga y uso de CPU
uptime                          # load average
top -bn1 | head -20
ps aux --sort=-%cpu | head -10

# Memoria y swap
free -h
ps aux --sort=-%mem | head -10

# I/O — frecuente causa real cuando "todo está lento" sin CPU alta
sudo iotop -bn1 -o | head -20   # paquete iotop
vmstat 2 5                      # 5 muestras a 2 segundos
sudo dmesg | tail -50           # mensajes del kernel recientes (OOM, errores de disco)
```

`iotop` muestra qué proceso hace E/S; su columna `IO>` es el porcentaje de tiempo que pasa bloqueado esperando al disco.

### A.3 Síntoma: un servicio no funciona

```bash
export SERVICIO=ssh   # nombre real del servicio (ssh, chrony, fail2ban...)
sudo systemctl status "${SERVICIO:?}" --no-pager
sudo journalctl -u "${SERVICIO:?}" -n 100 --no-pager
sudo journalctl -u "${SERVICIO:?}" --since "30 minutes ago" -p err

# Verificar dependencias del servicio
sudo systemctl list-dependencies "${SERVICIO:?}"

# Verificar configuración antes de recargar (cuando aplique)
sudo sshd -t                    # SSH
sudo nginx -t                   # nginx (si aplica)
sudo postfix check              # postfix (si aplica)
```

`journalctl -u` da el motivo. Lo habitual: sintaxis rota tras una edición, puerto ocupado o permisos.

### A.4 Síntoma: el disco está lleno

```bash
# Uso por filesystem
df -hT

# Inodos (a veces el problema no es espacio sino inodos agotados)
df -i

# Qué directorios consumen el espacio
sudo du -sh /* 2>/dev/null | sort -h
sudo du -sh /var/* | sort -h
sudo du -sh /var/log/* | sort -h

# Archivos grandes en /var/log o /tmp
sudo find /var/log -type f -size +100M -exec ls -lh {} \;
sudo find /tmp -type f -size +100M -exec ls -lh {} \;

# Espacio que journald está usando
sudo journalctl --disk-usage
```

Logs sin rotar, `/var/log/sudo-io/` sin retención (18.2.1), un proceso que escribe sin parar en `/var/log` o `/tmp`, un volcado olvidado o un respaldo a medias. Si es el journal, se baja `SystemMaxUse` en el drop-in de 18.1.1 y se libera en el momento con `sudo journalctl --vacuum-time=7d`.

### A.5 Síntoma: no puedo conectar por SSH

Desde otra máquina, con las variables de 10.2:

```bash
# Desde otra máquina, contra el servidor
MIFQDN=villonaco.ejemplo.com   # nombre DNS del servidor, si se usa
ping -c 3 "${IP_SERVIDOR:?}"
nc -vz "${IP_SERVIDOR:?}" "${MIPUERTO:?}"

# Resolución DNS si se usa el nombre
getent hosts "${MIFQDN:?}"

# Prueba SSH con verbose
ssh -vvv -p "${MIPUERTO:?}" "${MIUSUARIO:?}@${IP_SERVIDOR:?}"
```

Desde la consola del servidor:

```bash
# ¿Está SSH escuchando?
sudo systemctl status ssh --no-pager
sudo ss -tlnp | grep -E ":(22|${MIPUERTO})"

# ¿UFW está bloqueando?
sudo ufw status numbered
sudo ufw status verbose

# ¿fail2ban baneó la IP?
sudo fail2ban-client status sshd
IP_ADMIN=198.51.100.7   # IP pública del equipo del administrador
sudo fail2ban-client set sshd unbanip "${IP_ADMIN:?}"

# Logs de SSH
sudo journalctl -u ssh -n 50
sudo journalctl -u ssh --since "30 minutes ago" | grep -iE 'fail|invalid|preauth'
```

Lo habitual: un `01-hardening.conf` con sintaxis rota (`sudo sshd -t` antes de recargar), una regla de UFW borrada, la IP propia baneada por fail2ban o un puerto cambiado sin actualizar `~/.ssh/config`. `Too many authentication failures` sin haber escrito contraseña es un cliente que ofrece otras llaves y agota `MaxAuthTries`: falta `IdentitiesOnly yes` (10.4).

### A.6 Síntoma: red sin conectividad

```bash
# Interfaces e IPs
ip -brief address
ip route
cat /etc/resolv.conf

# Conectividad por capas
ping -c 3 127.0.0.1                                              # capa loopback
ping -c 3 "$(ip route show default | awk '{print $3; exit}')"    # capa enlace (gateway)
ping -c 3 1.1.1.1                                                # capa IP externa
ping -c 3 deb.debian.org                                         # capa IP + DNS

# Estado del backend de red activo
sudo systemctl status systemd-networkd --no-pager   # si se usa networkd
sudo systemctl status NetworkManager --no-pager     # si se usa NetworkManager
sudo systemctl status dhcpcd --no-pager             # si se usa dhcpcd

# Conexiones activas
sudo ss -tunap | head -30
```

Si responden las IP externas y no los nombres, es DNS; si no responde el gateway, es el enlace o la interfaz.

### A.7 Síntoma: alerta de fail2ban con muchas IPs baneadas

```bash
# Estado general
sudo fail2ban-client status

# Detalle del jail SSH
sudo fail2ban-client status sshd

# IPs actualmente baneadas
sudo fail2ban-client banned

# Últimos eventos en fail2ban
sudo journalctl -u fail2ban -n 100 --no-pager
sudo journalctl -u fail2ban --since "1 hour ago" | grep -i ban
```

Muchos baneos no alarman por sí solos: un SSH expuesto recibe escaneos constantes. Un cambio brusco, como cientos por hora, puede ser un ataque dirigido; se revisa también `sudo dmesg | grep -i 'syn flood'`.

### A.8 Síntoma: el sistema no arrancó del todo

```bash
# Servicios que fallaron en este arranque
sudo systemctl --failed

# Detalle de cada uno
for svc in $(sudo systemctl --failed --no-legend | awk '{print $2}'); do
    echo "=== $svc ==="
    sudo systemctl status "$svc" --no-pager | head -20
    echo
done

# Errores del arranque actual
sudo journalctl -b -p err --no-pager

# Comparar con el arranque anterior (si fue distinto)
sudo journalctl -b -1 -p err --no-pager
```

Si AppArmor, AIDE o auditd señalan algo tras el arranque:

```bash
sudo aa-status
sudo aide --config /etc/aide/aide.conf --check
sudo grep -h 'apparmor="DENIED"' /var/log/audit/audit.log* | tail -20   # con auditd activo (15.3)
sudo journalctl -k | grep 'apparmor="DENIED"' | tail -20                # sin auditd
```

### A.9 Recolección de evidencia para análisis posterior

Antes de tocar nada, cuando hará falta análisis forense, soporte externo o post-mortem:

```bash
# Crear directorio temporal con marca de tiempo
INCIDENT_DIR="/var/tmp/incident-$(date +%Y%m%dT%H%M%S)"
sudo mkdir -p "$INCIDENT_DIR"
cd "$INCIDENT_DIR"

# Capturar estado del sistema
sudo systemctl --failed > systemctl-failed.txt
sudo journalctl -b > journal-current-boot.txt
sudo journalctl -b -1 > journal-previous-boot.txt 2>/dev/null
sudo dmesg > dmesg.txt
sudo ss -tunap > ss-connections.txt
sudo ps auxf > processes.txt
sudo lsof > lsof.txt 2>/dev/null
ip -brief address > ip-address.txt
ip route > ip-route.txt
sudo ufw status verbose > ufw-status.txt
sudo fail2ban-client status > fail2ban-status.txt 2>/dev/null
sudo aureport --summary > aureport-summary.txt 2>/dev/null
df -hT > df.txt
free -h > free.txt
uptime > uptime.txt

# Comprimir todo y mover a /var/backups o copiar fuera del servidor
cd /var/tmp
sudo tar czf "incident-$(date +%Y%m%dT%H%M%S).tar.gz" "$(basename $INCIDENT_DIR)"
ls -la *.tar.gz
```

El paquete se copia fuera del servidor antes de investigar o corregir, porque cualquier cambio destruye la evidencia inicial. Ante sospecha de compromiso no se apaga el servidor, que perdería procesos, conexiones y lo que haya en `/tmp`: se aísla de la red (por ejemplo con `sudo ufw default deny outgoing`) y se captura primero lo volátil.

### A.10 Cuándo escalar

Si no se encuentra la causa o el incidente supera al operador: ticket en la organización, aviso al equipo de seguridad si se sospecha compromiso, y el servidor en estado controlado (servicios dañinos detenidos, red aislada) mientras llega ayuda. A quién se escala y por qué canal lo dice el runbook (22.3), escrito antes del incidente.

## Referencias

Ante una discrepancia, manda la página `man` del paquete instalado, que describe exactamente esa versión; la web del proyecto suele describir la última publicada.

### Documentación de Debian

- [Verificación de imágenes](https://www.debian.org/CD/verify), base de la sección 1.
- [Noticias del proyecto](https://www.debian.org/News/), donde se anuncian los point releases (16.9).
- [Debian Reference](https://www.debian.org/doc/user-manuals#quick-reference), empaquetada como `debian-reference-es` y `debian-reference-en`.
- [The Debian Administrator's Handbook](https://debian-handbook.info), empaquetado como `debian-handbook`.

### Componentes del manual

| Componente | Secciones | Páginas `man` | Proyecto |
| --- | --- | --- | --- |
| OpenSSH | 6, 10 | `sshd_config(5)`, `ssh_config(5)` | <https://www.openssh.com/> |
| sudo | 4, 9.3 | `sudoers(5)` | <https://www.sudo.ws/> |
| Linux-PAM, faillock | 9.3 | `pam_faillock(8)`, `faillock.conf(5)`, `pam-auth-update(8)`, `login.defs(5)` | <http://www.linux-pam.org/> |
| libpwquality | 9.4 | `pwquality.conf(5)` | <https://github.com/libpwquality/libpwquality> |
| UFW | 11 | `ufw(8)` | <https://launchpad.net/ufw> |
| fail2ban | 12 | `jail.conf(5)` | <https://www.fail2ban.org> |
| Kernel y módulos | 13 | `sysctl.d(5)`, `modprobe.d(5)` | — |
| systemd-resolved | 13.4 | `resolved.conf(5)` | <https://systemd.io> |
| auditd | 14.1 | `auditd.conf(5)`, `audit.rules(7)`, `auditctl(8)`, `augenrules(8)` | <https://people.redhat.com/sgrubb/audit/> |
| debsums | 14.2 | `debsums(1)` | — |
| AIDE | 14.3 | `aide(1)`, `aide.conf(5)` | <https://aide.github.io> |
| wtmpdb | 14.4 | `wtmpdb(8)` | <https://github.com/thkukuk/wtmpdb/> |
| Lynis | 14.5 | `lynis(8)` | <https://cisofy.com/lynis/> |
| AppArmor | 15 | `apparmor(7)`, `aa-status(8)`, `apparmor_parser(8)` | <https://apparmor.net/> |
| unattended-upgrades | 16 | `unattended-upgrade(8)` | — |
| apt-listchanges | 16 | `apt-listchanges(1)` | <https://salsa.debian.org/debian/apt-listchanges/> |
| chrony | 17 | `chrony.conf(5)`, `chronyc(1)` | <https://chrony-project.org> |
| systemd-journald | 18 | `journald.conf(5)`, `journalctl(1)`, `tmpfiles.d(5)` | <https://systemd.io> |
| logrotate | 18, 20.13 | `logrotate(8)` | <https://github.com/logrotate/logrotate> |
| sysstat | 19 | `sar(1)` | <https://sysstat.github.io/> |
| cron y run-parts | 19 | `crontab(5)`, `run-parts(8)`, `systemd.timer(5)` | — |
| BorgBackup | 21 | `borg(1)` | <https://borgbackup.github.io/borgbackup/> |
| msmtp | 20 | `msmtp(1)` | <https://marlam.de/msmtp/> |
| bsd-mailx | 20 | `mail(1)` | <https://packages.debian.org/trixie/bsd-mailx> |
| cryptsetup | 2 | `cryptsetup(8)` | <https://gitlab.com/cryptsetup/cryptsetup> |
| needrestart | 8.4 | — | <https://github.com/liske/needrestart> |
| Red | 7.5 | `interfaces(5)`, `systemd.network(5)` | — |

Las direcciones son las del campo `Homepage` de cada paquete (`apt-cache show PAQUETE`); sin ella, la referencia es la página `man` y `/usr/share/doc/PAQUETE/`.

## Historial de cambios

- [0.13] – 2026-11-26
  - Actualización de Debian 13.7 a 13.8, con las secciones 1 a 19 recorridas en una VM KVM instalada desde la ISO netinst.
  - Pasos idempotentes: los respaldos `.ori` de 7.2 y 8.1 se hacen solo la primera vez, `52unattended-upgrades-local` se crea con `install` y lo que añaden 16.3, 16.4 y el correo de AIDE (14.3) solo se escribe si falta.
  - Correcciones para la instalación desde ISO: `machine-id` frente al Product UUID (7.6), el `sources.list` que deja la netinst (8), el journal persistente (18.1), `PASS_MIN_LEN` ausente de `login.defs` (9.4) y el error de vsock aunque exista `/dev/vsock` (2.5.2).
  - AIDE excluye `/run/dhcpcd`, advierte de los archivos de trabajo en `/root` y `/home`, aplica el correo después de la sección 20 y se regenera al cerrar la Etapa III; la plantilla `modprobe` pasa a v1.2 por la referencia al runbook (§22.3).
  - Se redactan como variables el efecto de `libpam-pwquality` sobre faillock (9.4) y el índice de lynis (14.5), se amplían las sesiones que bloquean el reinicio automático (16.4) y los errores esperados del arranque (18.1.2), y el historial y el To-do pasan al final del manual.

- [0.12] – 2026-11-20
  - La §21.3.6 deja de ser un esquema pendiente y se divide en cuatro: cuenta de solo lectura con `rrsync -ro` y el grupo `borg-lectura` en el servidor (§21.3.6), copia del directorio `/var/backups/borg` completo en el servidor de respaldos con snapshots btrfs, límite de espacio y avisos por correo (§21.3.7), encendido por RTC y apagado tras la copia (§21.3.8) y copia manual en uno o varios discos externos con LUKS2 (§21.3.9).
  - Las plantillas `01-hardening.conf` y `msmtprc` pasan a v1.4 y v1.3: en la v0.11 solo cambiaron las referencias a secciones de sus comentarios, y la versión sube para que un servidor desplegado con la anterior se distinga.
  - El script de 21.3.5 devuelve la lectura del repositorio al grupo `borg-lectura` tras cada respaldo. La §21.1 admite un repositorio propio de un rol al lado del `<host>` y aclara que el servidor de respaldos no guarda las passphrases.
  - 16.2 a 16.4: la configuración local de unattended-upgrades pasa a `52unattended-upgrades-local` y el `50` queda como lo instala el paquete; validado en un VPS. El aviso de 16.2 se corrige: el `50` no es un conffile de dpkg sino un archivo de `ucf`, que `debsums` no vigila.
  - Sale del To-do la comprobación de `/etc/cron.d/staticroute`, que es del proveedor y no de Debian: el resultado, que el tráfico entre vecinos de una subred de Contabo sí depende de esa ruta y que el cron pierde la carrera con `systemd-networkd`, está en la sección 4 de [`proveedores/contabo.md`](proveedores/contabo.md), que pasa a v0.2 con lo aprendido al recorrerlo contra un segundo VPS: root bloqueada y consola VNC, restos de la imagen en `sshd_config`, el fallo de cloud-init en cada arranque, `kernel.panic` conservado, firewall compartido entre instancias, ping descartado, borrado de las credenciales de `cntb` y bloques de 6.2 y 6.3 validados tal como están escritos.

- [0.11] – 2026-11-19
  - Se intercambian las secciones 20 y 21: el reenvío de correo pasa a ser la §20 y los respaldos la §21; `bsd-mailx` sustituye a `mailutils` y msmtp añade su perfil de AppArmor y `from_full_name`.
  - La §21 se reescribe con un repositorio Borg cifrado por servidor, grupos de activos como prefijo de los archivos, script con argumento y unidades plantilla, y un esquema 3-2-1 con copia traída por el servidor de respaldos.
  - Se añaden el baneo permanente de direcciones (§11.7), `nftables` junto a fail2ban, NTS validado en chrony y el borrado de las copias de la contraseña inicial que deja cloud-init (§5.1).
  - Se documentan efectos laterales observados: `UMASK 027` con debconf, `mail_badpass` sin correo con msmtp, `LoginGraceTime` y `PerSourcePenalties`, y el ruido de sysstat y `/tmp` en AIDE.
- [0.10] – 2026-10-18
  - Reorganización y renumeración: plantillas en §3, aprovisionamiento del proveedor en §5, firewall del proveedor en §11.6, diagnóstico como único anexo y referencias al final
  - Lo propio de cada proveedor pasa a `proveedores/`, y la restricción a un CDN y el análisis de logs web al manual de Caddy. Se retiran rkhunter y las secciones redundantes
  - Correcciones de fondo en PAM y faillock (§9.3.3), SSH (§10), auditd y AIDE (§14), AppArmor con auditd (§15.3), journald (§18) y msmtp (§21)
- [0.9] – 2026-10-09
  - El Anexo C de reenvío de correo con `msmtp` pasa a ser la sección 19 de la Etapa IV, y la documentación operativa pasa a ser la sección 20. Todas las referencias internas quedan renumeradas
  - §9.6 reescrita: aviso de que no se ejecuta en el despliegue base, descarga de los rangos a un archivo con comprobación antes de tocar ninguna regla, retiro de las reglas abiertas por número y comprobación mensual de la lista publicada
  - Nuevo Anexo C con el firewall del proveedor de nube, que es la capa que filtra antes de UFW y la única que alcanza a los puertos publicados por contenedores. Incluye el procedimiento para Azure
  - §11.1, §11.3 y §12.4 advierten qué revisar cuando el rol del servidor incluye contenedores
  - §14.2 aclara que los repositorios de terceros quedan fuera de las actualizaciones automáticas
  - Anexo C: nueva C.4 con el firewall de VPS de Hostinger a través de su API, verificada en un despliegue detrás de un borde. La verificación desde fuera y el mantenimiento pasan a ser C.5 y C.6
- [0.8] – 2026-09-24
  - Cambio de la URL del manual y actualización de las URL de las plantillas
- [0.7] – 2026-09-13
  - Nueva §9.6 con la restricción del origen a los rangos de un proxy de borde o CDN
  - Nueva §10.5 sobre los límites del modelo de baneo por IP frente a campañas distribuidas
  - Nueva §16.5 sobre retención de logs frente a necesidad forense
  - Nuevo Anexo B.11 con el análisis de logs de acceso en formato JSON
- [0.6] – 2026-09-12
  - Arreglos menores en la plantilla `/etc/hosts` del apartado §5.2
  - Actualización de Debian 13.6 a 13.7
  - Se retiran los checklists de cierre de todas las secciones
  - Nueva §14.9 con el procedimiento posterior a un point release
- [0.5] – 2026-07-24
  - Modificación de las variables `ALLOW_SSH_ROOT_USER=unset` y `ALLOW_SSH_PROT_V1=2` en el apartado §12.2
- [0.4] – 2026-07-14
  - Actualización de Debian 13.5 a 13.6
- [0.3] – 2026-05-20
  - Seguridad: URLs de plantillas migradas a SHA anclado; nuevo hardening de sudo/pam_faillock; refuerzo SSH con   DisableForwarding; passphrase de Borg protegida con read -s.
  - Correcciones: SystemMaxUse de journald reducido a 1 G; nota sobre PASS_MIN_LEN ignorado con pam_pwquality; variable $MIPUERTO_SMTP para evitar colisión; LUKS2/Argon2id/GRUB aclarado; sed de dropbear reemplazado por secuencia robusta; fixes de numeración y typo.
  - Mejoras: nueva §12.6 con auditoría lynis; soporte NTS en chrony; pool NTP por país; nota de egreso en UFW; referencias cruzadas entre MTA, fail2ban y pam_faillock.
- [0.2] – 2026-05-16
  - Actualización de Debian 13.4 a 13.5
  - Inclusión del override en C.6.1
- [0.1] – 2026-04-28
  - Manual terminado para la versión 13.4.0 de Debian

## To-do

- Nuevo manual de reenvío de logs a un servidor central con `systemd-journal-upload` y `systemd-journal-remote`
- Evaluar y documentar CrowdSec como complemento o sustituto de fail2ban (§12.5). Requiere validación en entorno propio antes de escribirse como procedimiento: agente, escenarios, lista comunitaria y componente AppSec
- Validar las copias de la sección 21.3.6 (servidor de respaldos y disco externo); la copia local, la automatización y la restauración se validaron en un VPS de Contabo el 2026-11-19
- Inclusión de Ansible
- Evaluar Forward Secure Sealing (FSS) del journal para servidores con requisitos de auditoría (§18.1.2)
- Inclusión de Health Check, monitoreo y alerta, con uso de mensajería instantánea (Telegram) o correo electrónico
- Verificar en entorno propio las órdenes de `az` de [`proveedores/azure.md`](proveedores/azure.md). Están tomadas de la referencia de la CLI de Azure y no de una corrida completa
- Verificar en entorno propio las dos órdenes de [`proveedores/hostinger.md`](proveedores/hostinger.md) que no se han ejecutado contra la API de Hostinger: el alta de una regla con origen abierto y el borrado de una regla por su identificador
- Documentar el filtrado de puertos publicados por contenedores en el propio host, para servidores sin firewall de proveedor (11.6.1)
- Verificar sobre un servidor con contenedores la exclusión de AIDE de §14.3 y revisar el primer reporte diario posterior
- Diseñar una arquitectura de respaldo 3-2-1, donde 2 sea una unidad de almacenamiento adicional cifrada y conectada al servidor para ejecutar respaldos periódicos; y 3 sea Borg en un repo externo al servidor
- Auditar los servidores desplegados con versiones ≤ 0.9 de este manual (p. ej. `maracuya` y `arandano`): en el layout anterior de §9.3.3 las líneas de faillock iban debajo del bloque gestionado y `pam_deny` abortaba la pila antes del `preauth`, de modo que `pam_faillock` nunca contaba fallos. Se corrige aplicando la sección tal como está ahora

---

<!-- markdownlint-disable-next-line MD033 -->
[Despliegue de Debian 13 Trixie server](https://github.com/noggalito/debian) © 2026 by [Calú](https://github.com/calu777) in [noggalito](https://noggalito.com/) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">
