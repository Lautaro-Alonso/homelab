# Proxmox

Proxmox es la base de mi homelab.

Lo instalé directamente sobre la mini PC y lo utilizo como hipervisor para correr pfSense y, más adelante, los distintos servidores y servicios del laboratorio.

Una de las cosas que más me interesaba aprender con este proyecto era cómo virtualizar un router/firewall sin perder la separación entre WAN, LAN y las distintas VLANs.

Actualmente estoy utilizando Proxmox VE 9.2.2.

## Interfaces físicas

La mini PC tiene dos interfaces Ethernet físicas:

| Interfaz | Uso |
|----------|-----|
| `nic0` | WAN |
| `nic1` | LAN / Trunk |

La idea es mantener separado el lado WAN del lado interno de la red.

`nic0` recibe la conexión que viene del modem del ISP.

`nic1` conecta el host con el switch Omada y transporta la red LAN junto con las VLANs del homelab.

## Bridges

Para conectar las interfaces físicas con las máquinas virtuales uso dos Linux Bridges.

### vmbr1 - WAN

`vmbr1` está conectado a:

`nic0`

Este bridge no tiene una dirección IP configurada en el host.

Su función es transportar la conexión WAN hasta la interfaz WAN de pfSense.

El recorrido queda así:

```text
Modem ISP
    |
    v
  nic0
    |
    v
  vmbr1
    |
    v
pfSense net0
    |
    v
vtnet0 - WAN
```

De esta forma Proxmox no necesita tener una IP propia sobre la red WAN.

---

### vmbr0 - LAN

`vmbr0` está conectado a:

`nic1`

Este bridge es VLAN-aware y funciona como bridge principal para la parte interna de la red.

Actualmente transporta las VLANs:

- VLAN 10 - MANAGEMENT
- VLAN 20 - HOME
- VLAN 30 - PERSONAL
- VLAN 40 - SERVERS
- VLAN 50 - DMZ

La interfaz LAN de pfSense está conectada a este bridge.

El recorrido general queda así:

```text
pfSense
   |
vtnet1
   |
   v
 vmbr0
   |
   v
 nic1
   |
   v
Switch Omada
```

`vmbr0` funciona entonces como punto de unión entre pfSense, las futuras VMs del homelab y la red física.

## Administración de Proxmox

Para administrar el propio host creé una interfaz VLAN sobre `vmbr0`:

`vmbr0.10`

Esta interfaz utiliza VLAN 10, que corresponde a MANAGEMENT.

Configuración actual:

| Parámetro | Valor |
|-----------|-------|
| Interfaz | `vmbr0.10` |
| VLAN | `10` |
| IP | `192.168.10.2/24` |
| Gateway | `192.168.10.1` |

El gateway `192.168.10.1` corresponde a pfSense.

De esta forma la interfaz web de Proxmox queda dentro de la VLAN de administración en lugar de depender de una red de uso normal.

Actualmente también existe una dirección:

`192.168.1.50/24`

configurada directamente sobre `vmbr0`.

Esta dirección pertenece a la LAN base y quedó de las primeras etapas de configuración del homelab.

Actualmente administro Proxmox principalmente desde:

`192.168.10.2`

dentro de MANAGEMENT.

Más adelante probablemente elimine la dirección de la LAN base y deje la administración del host solamente dentro de VLAN 10.

## VM de pfSense

pfSense corre como una máquina virtual dentro de Proxmox.

La configuración actual es:

| Recurso | Valor |
|---------|-------|
| vCPU | 2 cores |
| RAM | 3 GiB |
| Disco | 20 GB |
| NICs virtuales | 2 |
| Tipo de NIC | VirtIO |

La VM utiliza una interfaz para WAN y otra para LAN/trunk.

### net0 - WAN

`net0` está conectada a:

`vmbr1`

Dentro de pfSense esta interfaz aparece como:

`vtnet0`

y está asignada como WAN.

El recorrido queda:

```text
nic0
 |
 v
vmbr1
 |
 v
net0
 |
 v
vtnet0
 |
 v
WAN
```

El firewall de Proxmox está habilitado actualmente sobre esta interfaz.

---

### net1 - LAN / Trunk

`net1` está conectada a:

`vmbr0`

Dentro de pfSense aparece como:

`vtnet1`

y funciona como interfaz LAN y trunk para las VLANs.

Las VLANs permitidas actualmente son:

`10;20;30;40;50`

El recorrido queda:

```text
pfSense
   |
vtnet1
   |
 net1
   |
 vmbr0
   |
 nic1
   |
   v
Switch Omada
```

A partir de `vtnet1`, pfSense crea las distintas interfaces VLAN que después funcionan como gateway de cada red.

## VLANs virtuales

Una de las cosas que más me gusta de esta arquitectura es que no todas las VLANs necesitan salir físicamente del host.

Por ejemplo, SERVERS y DMZ están pensadas principalmente para máquinas virtuales o containers que viven dentro de Proxmox.

Una futura VM puede conectarse directamente a la VLAN correspondiente sin necesitar que ese tráfico salga hasta el switch físico y vuelva a entrar al host.

HOME y PERSONAL, en cambio, sí tienen dispositivos físicos conectados a través del switch.

## Firewall de Proxmox

Durante la configuración tuve un problema donde el firewall de Proxmox interfería con el tráfico etiquetado que pasaba por la interfaz LAN de pfSense.

Después de revisar el problema terminé dejando deshabilitado el firewall de Proxmox sobre `net1`.

La idea es que el filtrado y las reglas entre las distintas redes sean responsabilidad de pfSense.

Actualmente queda así:

| Interfaz VM | Bridge | Firewall Proxmox |
|-------------|--------|------------------|
| `net0` | `vmbr1` | Activado |
| `net1` | `vmbr0` | Desactivado |

Este fue otro de esos problemas que terminé entendiendo mejor después de tener que revisar cómo se mueve realmente el tráfico entre el host, los bridges y pfSense.

## Resumen

La arquitectura de red dentro de Proxmox queda aproximadamente así:

```text
                         Mini PC
                            |
            +---------------+---------------+
            |                               |
           nic0                            nic1
            |                               |
          vmbr1                           vmbr0
            |                               |
            |                         VLAN-aware
            |                               |
       pfSense net0                    pfSense net1
            |                               |
          vtnet0                          vtnet1
            |                               |
           WAN                       LAN / VLAN trunk
                                            |
                         +------------------+------------------+
                         |                  |                  |
                      VLAN 10            VLAN 40            VLAN 50
                         |                  |                  |
                    MANAGEMENT           SERVERS             DMZ
```

Todavía quiero seguir agregando VMs y servicios al host, así que esta parte del homelab seguramente vaya cambiando bastante con el tiempo.
