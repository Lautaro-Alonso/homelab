# VLANs

Una de las primeras cosas que quise hacer con el homelab fue dejar de tener toda la red de mi casa mezclada en el mismo segmento.

Con pfSense y el switch administrable empecé a separar los dispositivos según su función. Además de mejorar el aislamiento entre equipos, esto me sirve para aprender de forma práctica cómo funcionan las VLANs, el routing entre redes y las reglas de firewall.

Actualmente uso cinco VLANs.

## Resumen

| VLAN | Nombre | Subred | Gateway | DHCP |
|------|--------|--------|---------|------|
| 10 | MANAGEMENT | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.100 - 192.168.10.199` |
| 20 | HOME | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.100 - 192.168.20.199` |
| 30 | PERSONAL | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.100 - 192.168.30.199` |
| 40 | SERVERS | `192.168.40.0/24` | `192.168.40.1` | `192.168.40.100 - 192.168.40.150` |
| 50 | DMZ | `192.168.50.0/24` | `192.168.50.1` | `192.168.50.100 - 192.168.50.150` |

El servidor DHCP corre actualmente en pfSense utilizando ISC DHCP. (lo voy a tener que cambiar a kea pronto)

---

## VLAN 10 - MANAGEMENT

Esta VLAN está destinada exclusivamente a administrar la infraestructura de la red.

Acá viven equipos como:

- pfSense
- Proxmox
- TP-Link Omada ES206GP

Red:

`192.168.10.0/24`

Gateway:

`192.168.10.1`

DHCP:

`192.168.10.100 - 192.168.10.199`

La idea es mantener las interfaces de administración separadas de los dispositivos normales de la casa.

---

## VLAN 20 - HOME

Esta es la red de uso general de la casa.

Red:

`192.168.20.0/24`

Gateway:

`192.168.20.1`

DHCP:

`192.168.20.100 - 192.168.20.199`

Acá están principalmente los dispositivos del resto de la casa.

La intención es que teléfonos, televisores y otros dispositivos domésticos puedan usar Internet normalmente sin necesitar acceso a la infraestructura del homelab.

---

## VLAN 30 - PERSONAL

Esta es mi red personal.

Red:

`192.168.30.0/24`

Gateway:

`192.168.30.1`

DHCP:

`192.168.30.100 - 192.168.30.199`

Acá tengo mis dispositivos principales, entre ellos:

- PC de escritorio
- Notebook
- Archer AX1500

El Archer funciona como Access Point y conecta mis dispositivos a esta VLAN.

Esta es también la red desde la que normalmente administro el homelab.

---

## VLAN 40 - SERVERS

Esta VLAN está reservada para servidores y servicios internos.

Red:

`192.168.40.0/24`

Gateway:

`192.168.40.1`

DHCP:

`192.168.40.100 - 192.168.40.150`

A diferencia de HOME y PERSONAL, esta red vive principalmente dentro del entorno virtualizado de Proxmox.

La idea es utilizarla para servicios que solamente necesitan estar disponibles dentro de mi red.

---

## VLAN 50 - DMZ

Esta VLAN está destinada a servicios que eventualmente necesiten estar expuestos hacia Internet.

Red:

`192.168.50.0/24`

Gateway:

`192.168.50.1`

DHCP:

`192.168.50.100 - 192.168.50.150`

La intención es mantener cualquier servicio expuesto separado tanto de la red doméstica como de la infraestructura de administración.

Esto me permite experimentar con servicios públicos, port forwarding y reglas de firewall sin tener que exponer directamente el resto de mi red.


---

## Routing entre VLANs

pfSense funciona como gateway de todas las VLANs y se encarga del routing entre ellas.

Eso no significa que todas puedan comunicarse libremente.

El acceso entre redes está limitado mediante reglas de firewall según el propósito de cada VLAN.

Las reglas y el razonamiento detrás de ellas están documentados en:

[Firewall](firewall.md)
