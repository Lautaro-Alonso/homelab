# QoS

Después de terminar la segmentación de la red quise solucionar otro problema: la latencia cuando la conexión está bajo carga.

La idea no era conseguir más velocidad, sino evitar que una descarga o subida saturara completamente el enlace y terminara aumentando demasiado el ping.

Para esto configuré traffic shaping en pfSense utilizando limiters con FQ_CoDel (Fair Queuing Controlled Delay).

## Limiters

Actualmente tengo dos limiters principales:

| Limiter | Ancho de banda |
|---------|----------------|
| `WANdown` | 290 Mbit/s |
| `WANup` | 105 Mbit/s |

Los configuré ligeramente por debajo de la capacidad real de la conexión.

La idea es que la congestión se produzca dentro de pfSense, donde puedo controlarla, en vez de dejar que se forme una cola más grande en el modem o dentro de la red del ISP.

## FQ_CoDel

Tanto la subida como la bajada utilizan FQ_CoDel (Fair Queuing Controlled Delay) como scheduler.

La configuración actual es:

| Parámetro | Valor |
|-----------|-------|
| target | `5 ms` |
| interval | `100 ms` |
| quantum | `1514` |
| limit | `10240` |
| flows | `1024` |
| Queue length | `1000` |
| ECN | Activado |

FQ_CoDel intenta separar los distintos flujos de tráfico y evitar que una única conexión termine ocupando toda la cola.

El objetivo principal es reducir el bufferbloat y mantener la latencia más estable cuando la conexión está siendo utilizada intensamente.

## ECN

También dejé habilitado ECN (Explicit Congestion Notification).

Cuando los dispositivos y protocolos involucrados lo soportan, ECN permite indicar que existe congestión sin depender únicamente de descartar paquetes.

No todos los flujos necesariamente van a aprovecharlo, pero decidí mantenerlo habilitado junto con FQ_CoDel.

## Colas

Dentro de cada limiter tengo una cola para el tráfico general y otra para la VLAN PERSONAL.

### Tráfico general

Para el tráfico normal utilizo:

- `WANdownQ`
- `WANupQ`

Ambas tienen un peso de:

`10`

### PERSONAL

Para mi VLAN PERSONAL creé:

- `WANdownPersonalQ`
- `WANupPersonalQ`

Estas colas tienen un peso de:

`100`

La intención es darle preferencia a mi red PERSONAL cuando existe congestión.

Esto no significa que PERSONAL tenga reservado diez veces más ancho de banda todo el tiempo. El peso entra en juego principalmente cuando varias colas están compitiendo por capacidad.

La estructura actual queda aproximadamente así:

```text
WANdown - 290 Mbit/s
├── WANdownQ - Weight 10
└── WANdownPersonalQ - Weight 100

WANup - 105 Mbit/s
├── WANupQ - Weight 10
└── WANupPersonalQ - Weight 100
