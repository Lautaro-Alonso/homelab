# pfSense

pfSense es el router y firewall principal de mi red.

Lo tengo virtualizado dentro de Proxmox y se encarga de prácticamente toda la parte de red del homelab:

- Routing
- VLANs
- Firewall
- DHCP
- DNS
- NAT
- QoS

Antes de armar el homelab utilizaba principalmente el router del ISP. La idea de pasar estas funciones a pfSense fue tener mucho más control sobre la red y, principalmente, poder aprender cómo funcionan estas cosas de forma práctica.

## Interfaces

La VM de pfSense tiene dos interfaces VirtIO.

| Interfaz pfSense | Interfaz virtual | Uso |
|------------------|------------------|-----|
| WAN | `vtnet0` | Conexión hacia Internet |
| LAN | `vtnet1` | LAN y trunk de VLANs |

La WAN está conectada en Proxmox a `vmbr1`.

La LAN está conectada a `vmbr0`.

La arquitectura general queda:

```text
Internet
   |
   v
Modem ISP
   |
   v
vmbr1
   |
   v
vtnet0
   |
 pfSense
   |
 vtnet1
   |
   v
vmbr0
   |
   v
Switch / redes internas
```

## WAN

La interfaz WAN utiliza:

`vtnet0`

Actualmente obtiene su dirección mediante DHCP desde el ISP.

El modem del ISP está configurado en modo bridge, por lo que pfSense recibe directamente la conexión WAN.

## LAN

La interfaz LAN utiliza:

`vtnet1`

Su red base actual es:

`192.168.1.0/24`

Gateway:

`192.168.1.1`

Esta fue la red utilizada durante las primeras etapas del armado del homelab.

Actualmente la mayor parte de la infraestructura está segmentada utilizando VLANs.

## VLANs

Todas las VLANs están creadas sobre:

`vtnet1`

Actualmente tengo:

| VLAN | Nombre | Gateway |
|------|--------|---------|
| 10 | MANAGEMENT | `192.168.10.1` |
| 20 | HOME | `192.168.20.1` |
| 30 | PERSONAL | `192.168.30.1` |
| 40 | SERVERS | `192.168.40.1` |
| 50 | DMZ | `192.168.50.1` |

Dentro de pfSense aparecen como:

```text
vtnet1
├── vtnet1.10 -> MANAGEMENT
├── vtnet1.20 -> HOME
├── vtnet1.30 -> PERSONAL
├── vtnet1.40 -> SERVERS
└── vtnet1.50 -> DMZ
```

pfSense funciona como gateway de todas estas redes y se encarga del routing entre ellas.

La configuración y el objetivo de cada VLAN están documentados en:

[VLANs](vlans.md)

## DHCP

Actualmente pfSense también funciona como servidor DHCP.

Estoy utilizando:

`ISC DHCP`

aunque tengo pendiente migrarlo a Kea.

Los pools actuales son:

| Red | DHCP |
|-----|------|
| LAN | `192.168.1.100 - 192.168.1.199` |
| MANAGEMENT | `192.168.10.100 - 192.168.10.199` |
| HOME | `192.168.20.100 - 192.168.20.199` |
| PERSONAL | `192.168.30.100 - 192.168.30.199` |
| SERVERS | `192.168.40.100 - 192.168.40.150` |
| DMZ | `192.168.50.100 - 192.168.50.150` |

Una de las ventajas de manejar DHCP desde pfSense es que puedo tener un scope separado para cada VLAN y controlar el direccionamiento desde un único lugar.

## DNS

El hostname actual de pfSense es:

`firewall`

y utilizo como dominio interno:

`home.arpa`

Por lo tanto, el firewall queda identificado dentro de la red como:

`firewall.home.arpa`

En la configuración general tengo definidos como DNS externos:

- `1.1.1.1`
- `8.8.8.8`

La opción para aceptar automáticamente los DNS entregados por el ISP está deshabilitada.

La resolución está configurada para utilizar primero el DNS local de pfSense:

`127.0.0.1`

y utilizar los DNS externos como fallback.

Las VLANs aisladas tienen reglas específicas que permiten realizar consultas DNS hacia la dirección de pfSense en cada red.

Más adelante quiero seguir trabajando esta parte, probablemente agregando un servicio de filtrado DNS dentro del homelab.

## Firewall

pfSense también controla la comunicación entre las VLANs.

La idea general es que las redes no puedan hablar entre sí simplemente por estar conectadas al mismo router.

Por ejemplo:

- HOME tiene Internet pero no acceso al resto de mis redes internas.
- SERVERS puede salir a Internet pero no iniciar conexiones libremente hacia otras VLANs.
- DMZ está aislada de las redes internas.
- PERSONAL funciona actualmente como mi red de confianza y puede administrar el resto del homelab.

Las reglas están documentadas con más detalle en:

[Firewall](firewall.md)

## QoS

También utilizo pfSense para controlar la congestión del enlace.

Configuré limiters de subida y bajada utilizando FQ_CoDel y ECN.

Además tengo colas separadas para darle preferencia al tráfico de PERSONAL cuando existe congestión.

Toda esa configuración está documentada en:

[QoS](qos.md)

## VirtIO y velocidad mostrada

Dentro de pfSense las interfaces aparecen como:

`10Gbase-T <full-duplex>`

Esto inicialmente puede resultar confuso.

No significa que las interfaces físicas de la mini PC sean de 10 GbE.

pfSense está viendo interfaces virtuales VirtIO proporcionadas por Proxmox, por lo que esa velocidad corresponde al enlace virtual entre la VM y el host.

Las interfaces Ethernet físicas de la mini PC son de 2.5 GbE.

## Rol dentro del homelab

Hoy pfSense termina siendo una de las piezas centrales de todo el laboratorio.

El flujo simplificado es:

```text
                 Internet
                     |
                     v
                  pfSense
                     |
          +----------+----------+
          |          |          |
       Routing    Firewall     QoS
          |
          v
       VLANs
          |
   +------+------+------+------+------+
   |      |      |      |      |
 MGMT   HOME  PERSONAL SERVERS  DMZ
```
