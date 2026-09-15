# Firewall

Después de separar la red en VLANs, el siguiente paso fue decidir qué redes podían comunicarse entre sí.

La idea no era solamente tener varias subredes distintas, sino aprovechar pfSense para aislarlas según su función.

Por ejemplo, los dispositivos de HOME no necesitan poder acceder a Proxmox, pfSense o mis servidores. En cambio, desde PERSONAL sí necesito poder administrar el resto del homelab.

Las reglas se aplican en pfSense sobre la interfaz por la que entra el tráfico.

## Política general

| Red | DNS pfSense | Internet | Otras redes internas |
|------|-------------|----------|-----------------------|
| MANAGEMENT | Permitido | Permitido | Bloqueado |
| HOME | Permitido | Permitido | Bloqueado |
| PERSONAL | Permitido | Permitido | Permitido |
| SERVERS | Permitido | Permitido | Bloqueado |
| DMZ | Permitido | Permitido | Bloqueado |

Para simplificar las reglas creé el alias `INTERNAL_NETS`, que agrupa las redes internas que no deberían ser accesibles desde las VLANs aisladas.

---

## VLAN 10 - MANAGEMENT

Aunque esta VLAN contiene las interfaces de administración de la infraestructura, no necesita iniciar conexiones hacia el resto de las redes internas.

Las reglas siguen este orden:

1. Permitir consultas DNS hacia pfSense por el puerto 53.
2. Bloquear tráfico hacia `INTERNAL_NETS`.
3. Permitir el resto del tráfico hacia Internet.

Esto mantiene MANAGEMENT aislada del resto de las VLANs.

El acceso para administrar dispositivos de esta red lo hago desde PERSONAL.

---

## VLAN 20 - HOME

HOME contiene los dispositivos de uso general de la casa.

No quiero que un dispositivo de esta red pueda acceder a la infraestructura del homelab o a otras VLANs.

Las reglas son:

1. Permitir DNS hacia pfSense.
2. Bloquear acceso hacia `INTERNAL_NETS`.
3. Permitir salida hacia Internet.

De esta manera los dispositivos de la casa pueden funcionar normalmente sin tener acceso directo a las otras VLANs.

---

## VLAN 30 - PERSONAL

PERSONAL es la excepción a la política de aislamiento.

Esta es mi red de confianza y es desde donde administro el homelab, por lo que actualmente tiene una regla que permite tráfico hacia cualquier destino.

Desde PERSONAL puedo acceder a:

- MANAGEMENT
- HOME
- SERVERS
- DMZ
- Internet

Esto me permite administrar pfSense, Proxmox, el switch, servidores y otros dispositivos desde mi PC o notebook sin tener que crear una excepción individual para cada servicio.

Más adelante probablemente vaya haciendo esta política más restrictiva a medida que el homelab crezca.

---

## VLAN 40 - SERVERS

Esta VLAN contiene los servicios internos del homelab.

Los servidores pueden necesitar acceso a Internet para descargar paquetes, actualizaciones o comunicarse con servicios externos, pero no deberían poder iniciar conexiones libremente hacia las demás redes internas.

Las reglas son:

1. Permitir DNS hacia pfSense.
2. Bloquear acceso hacia `INTERNAL_NETS`.
3. Permitir salida hacia Internet.

Esto también ayuda a limitar el alcance de un problema en caso de que alguno de los servidores se vea comprometido.

---

## VLAN 50 - DMZ

DMZ es la red que más me interesa mantener aislada.

Está pensada para servicios que eventualmente puedan quedar expuestos hacia Internet, por lo que no quiero que una máquina dentro de esta VLAN pueda usar ese acceso como puente hacia mi red interna.

Las reglas son:

1. Permitir DNS hacia pfSense.
2. Bloquear acceso hacia `INTERNAL_NETS`.
3. Permitir salida hacia Internet.

De esta forma, un servicio de la DMZ puede tener conectividad externa sin poder iniciar conexiones hacia MANAGEMENT, HOME, PERSONAL o SERVERS.

---

## Orden de las reglas

El orden es importante porque pfSense evalúa las reglas de arriba hacia abajo.

Por eso las VLANs aisladas siguen aproximadamente esta estructura:

`Permitir DNS -> Bloquear redes internas -> Permitir resto`

Si el bloqueo de `INTERNAL_NETS` estuviera después de la regla que permite cualquier destino, nunca llegaría a evaluarse.

Este fue uno de los conceptos que quise aplicar de forma práctica al armar la segmentación del homelab.
