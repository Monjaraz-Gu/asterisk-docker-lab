# Manual de procedimiento: laboratorio Asterisk con Docker

**Revisión del estado:** 6 de noviembre de 2026  
**Alcance:** configuración observada en este equipo. Este documento reconstruye el despliegue actual; no demuestra el orden exacto de todos los pasos históricos.

> Este manual contiene credenciales SIP. Se guardará con permisos restringidos; no lo publiques ni lo agregues a un repositorio. Las claves actuales son predecibles y deben cambiarse antes de usar el PBX fuera de un laboratorio.

## 1. Resumen del entorno actual

- **Host:** Omarchy 4.5.0, basado en Arch Linux. No es Ubuntu Server.
- **Docker:** versión 29.9.2; servicio activo.
- **Contenedor:** `asterisk-lab`, imagen `andrius/asterisk:latest`; Asterisk 23.5.0; estado `healthy`.
- **Digest de la imagen observada:** `sha256:921d1fdc86d328fa3b9e73aee13d40599fe335ce6edefd31c7874729c3582d95`.
- **Red del contenedor:** `host`. No hay publicación `-p`; Asterisk utiliza directamente la red del host.
- **Política de reinicio:** `no`. Docker no está configurado para reiniciar este contenedor automáticamente tras reiniciar el equipo.
- **Configuración persistida en el host:**
  - `~/asterisk/config/pjsip.conf` → `/etc/asterisk/pjsip.conf`
  - `~/asterisk/config/extensions.conf` → `/etc/asterisk/extensions.conf`
- El archivo local `~/asterisk/config/asterisk.conf` **no** está montado en el contenedor. El contenedor usa su propio `/etc/asterisk/asterisk.conf`, almacenado en el volumen Docker de `/etc/asterisk`.
- La imagen crea volúmenes Docker para `/etc/asterisk`, logs, spool, claves, phone provisioning y sonidos. Los archivos PJSIP y dialplan montados individualmente sustituyen esos dos archivos dentro del directorio de configuración.

### Red del host observada

| Interfaz | Dirección | Observación |
|---|---|---|
| `enp2s0` | `192.168.2.101/24` | La ruta al teléfono `192.168.2.71` sale por esta interfaz |
| `wlp3s0` | `192.168.2.108/24` | También conectada a `192.168.2.0/24` |
| Ruta por defecto preferida | vía `192.168.2.254`, `wlp3s0` | Hay otra ruta vía `192.168.2.1`, `enp2s0`, con métrica mayor |

El host tiene dos interfaces en la misma subred. Como el contenedor usa `host` y PJSIP escucha en `0.0.0.0:5060`, el servicio se liga a todas las interfaces; el firewall determina qué tráfico puede entrar.

## 2. Instalación y despliegue

Docker y UFW ya están instalados en este host. No se pudo comprobar el historial exacto de instalación. Para preparar un host Omarchy/Arch desde cero, los comandos habituales son:

```bash
sudo pacman -S docker ufw
sudo systemctl enable --now docker
```

La carpeta del proyecto y los archivos de configuración están bajo el home del usuario:

```bash
mkdir -p ~/asterisk/config
```

La inspección del contenedor permite reconstruir este comando equivalente para un despliegue nuevo. **No lo ejecutes sobre el contenedor existente**: el nombre ya está ocupado y recrearlo requiere planificar la conservación de sus volúmenes.

```bash
docker run -d \
  --name asterisk-lab \
  --network host \
  --restart=no \
  -v "$HOME/asterisk/config/pjsip.conf:/etc/asterisk/pjsip.conf" \
  -v "$HOME/asterisk/config/extensions.conf:/etc/asterisk/extensions.conf" \
  andrius/asterisk:latest
```

La imagen `latest` es mutable: una descarga posterior puede entregar una versión diferente. Para reproducibilidad, fija una versión o digest después de decidir y validar la actualización. El comando original documentado en `~/Downloads/resumen_instalaci_n_asterisk_docker.md` no coincide con el despliegue actual: allí se documentaron el montaje de toda la carpeta y `--restart=always`; el contenedor actual monta dos archivos por separado y tiene política `no`.

## 3. Configuración PJSIP actual

Archivo: `~/asterisk/config/pjsip.conf`.

```ini
[global]
type=global
user_agent=Asterisk PBX

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; --- Extensión 101 ---
[101]
type=endpoint
context=default
disallow=all
allow=ulaw,alaw
auth=auth101
aors=101
transport=transport-udp

[auth101]
type=auth
auth_type=userpass
password=ClaveSegura101
username=101

[101]
type=aor
max_contacts=2

; --- Extensión 102 ---
[102]
type=endpoint
context=default
disallow=all
allow=ulaw,alaw
auth=auth102
aors=102
transport=transport-udp

[auth102]
type=auth
auth_type=userpass
password=ClaveSegura102
username=102

[102]
type=aor
max_contacts=2
```

Las credenciales configuradas son:

| Extensión | Usuario SIP | Contraseña actual |
|---|---|---|
| 101 | `101` | `ClaveSegura101` |
| 102 | `102` | `ClaveSegura102` |

Las contraseñas están en texto plano en el archivo. Usa claves aleatorias y distintas para cada extensión. Mantén el archivo protegido y entrega cada credencial solamente al dispositivo correspondiente.

## 4. Plan de marcación

Archivo: `~/asterisk/config/extensions.conf`.

```ini
[default]
exten => 600,1,Answer()
exten => 600,n,Echo()
exten => 600,n,Hangup()

exten => _10X,1,Dial(PJSIP/${EXTEN})
exten => _10X,n,Hangup()
```

La extensión `600` es una prueba de eco: devuelve el audio recibido. El patrón `_10X` coincide con extensiones de `100` a `109`; actualmente solo están definidos los endpoints 101 y 102.

## 5. Estado de Asterisk observado

- Transporte PJSIP: UDP, `0.0.0.0:5060`.
- Rango RTP configurado: UDP `10000–20000`.
- Endpoint 101: tiene un contacto observado en `192.168.2.71:11457`; Asterisk lo mostró como `NonQual`.
- Endpoint 102: `Unavailable`, sin contacto registrado.
- También se observaron sockets UDP en `4520`, `4569` y `5000`, correspondientes a módulos DUNDi, IAX2 y UNISTIM cargados por la imagen. No son necesarios para las extensiones PJSIP descritas aquí; no los abras en el firewall salvo que vayas a usar esos protocolos.

Comandos de consulta:

```bash
docker ps --filter name=asterisk-lab
docker exec asterisk-lab asterisk -rx "core show version"
docker exec asterisk-lab asterisk -rx "pjsip show transports"
docker exec asterisk-lab asterisk -rx "pjsip show endpoints"
docker exec asterisk-lab asterisk -rx "dialplan show default"
docker exec asterisk-lab asterisk -rx "rtp show settings"
docker logs --tail 100 asterisk-lab
```

## 6. Firewall del host y permisos de red

El firewall encontrado en **este host Omarchy/Arch** es UFW, no el firewall del plan Ubuntu descrito más adelante:

- El servicio `ufw` aparece activo y `/etc/ufw/ufw.conf` tiene `ENABLED=yes`.
- `/etc/default/ufw` indica entrada predeterminada `DROP`, salida `ACCEPT`, reenvío `DROP` e IPv6 habilitado.
- Las reglas de usuario legibles permiten TCP y UDP en el puerto `53317`, UDP/53 hacia `172.17.0.1` desde redes privadas y tráfico de la interfaz `waydroid0`.
- No se encontró una regla explícita de UFW para SIP `5060/UDP` ni para RTP `10000–20000/UDP`.
- No se encontró una regla explícita de UFW para los sockets `4520`, `4569` o `5000`.

La consulta `ufw status verbose` necesita privilegios de administrador y no se pudo ejecutar sin contraseña. Por eso, los puntos anteriores describen los archivos de reglas y el servicio, pero **no confirman el ruleset efectivo ni los contadores en ejecución**. Compruébalo desde un terminal:

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo nft list ruleset
```

Como el contenedor usa red `host`, las conexiones entrantes se evalúan como tráfico dirigido al host; no pasan por el flujo habitual de puertos publicados de una red bridge de Docker. Si UFW confirma que el tráfico está bloqueado y los teléfonos deben conectarse desde la LAN `192.168.2.0/24`, las siguientes reglas limitan el acceso a esa subred. **Son recomendaciones; no se aplicaron durante esta revisión.**

```bash
sudo ufw allow from 192.168.2.0/24 to any port 5060 proto udp
sudo ufw allow from 192.168.2.0/24 to any port 10000:20000 proto udp
```

No habilites estos puertos globalmente hacia Internet. Para clientes fuera de esa LAN, define primero VPN, NAT y restricciones de origen apropiadas; no abras los puertos a `Anywhere` sin un diseño de seguridad.

## 7. El plan de Ubuntu es otro sistema

`~/Documents/planeacion-firewall.txt` describe un **diseño planeado** para Ubuntu Server 24.04 como gateway: WAN `enp0s3`, LAN `enp0s8`, red `10.77.77.0/28`, puerta de enlace `10.77.77.1`, `nftables`, NAT, `dnsmasq`, SSH y Cockpit limitados a la LAN.

Ese plan no coincide con el host Asterisk observado: el sistema actual es Omarchy/Arch, sus interfaces son `enp2s0` y `wlp3s0`, y sus direcciones son `192.168.2.101` y `192.168.2.108`. No hay evidencia en esta revisión de que las reglas nftables de aquel plan se hayan aplicado a un Ubuntu activo. No mezcles las interfaces, rutas o subredes del plan con las reglas UFW de este host.

## 8. Recarga, respaldo y mantenimiento

Antes de editar un archivo, crea una copia:

```bash
cp -a ~/asterisk/config/pjsip.conf \
  ~/asterisk/config/pjsip.conf.bak.$(date +%Y%m%d-%H%M%S)
cp -a ~/asterisk/config/extensions.conf \
  ~/asterisk/config/extensions.conf.bak.$(date +%Y%m%d-%H%M%S)
```

Recarga los archivos desde la CLI de Asterisk:

```bash
docker exec asterisk-lab asterisk -rx "pjsip reload"
docker exec asterisk-lab asterisk -rx "dialplan reload"
```

Después de recargar, vuelve a consultar endpoints y dialplan. Los cambios al transporte UDP pueden requerir reiniciar Asterisk; coordina ese reinicio porque interrumpe llamadas. Los dos archivos montados desde el host son los que se deben editar para PJSIP y el plan de marcación.

El contenedor actual tiene `restart=no`. Si se requiere que Docker lo inicie al arrancar el host, debe cambiarse deliberadamente la política y validarse primero que el contenedor inicia correctamente:

```bash
docker update --restart unless-stopped asterisk-lab
```

## 9. Acciones recomendadas pendientes

1. Consultar el estado efectivo de UFW con `sudo ufw status verbose` y decidir si hacen falta reglas SIP/RTP limitadas a la LAN.
2. Cambiar las contraseñas predecibles de ambas extensiones.
3. Decidir si se necesita el reinicio automático del contenedor y configurar la política correspondiente.
4. Deshabilitar los módulos/protocolos UDP que no se utilicen, o confirmar que UFW los bloquea.
5. Si se busca reproducibilidad, reemplazar `latest` por una versión o digest fijado.
