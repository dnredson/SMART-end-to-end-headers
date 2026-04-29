# 04 — Protocol flow and headers

This capture is best understood as a sequence of protocol transitions.

## Stage 1 — RPi to cloud broker

### Logical flow

```text
/home/mist/Software/EnvData.py
  → MQTT publish
  → TCP
  → IP
  → Ethernet/Wi-Fi
  → 177.104.61.23:1883
```

### Headers visible in the pcap

At this point, the capture shows:

```text
Link-layer header
  → IP header
    → TCP header
      → MQTT control packet
        → MQTT topic
        → MQTT payload
```

The important header/protocol fields are:

| Layer | Relevant fields |
|---|---|
| IP | source `192.168.1.126`, destination `177.104.61.23` |
| TCP | destination port `1883` |
| MQTT | publish topic, QoS, payload |
| Application payload | sensor data such as `Atmos41_WS_NSAAB` and `Teros12_S4_NSAAB` |

This segment is the first production-visible segment for the platform path.

## Stage 2 — ChirpStack/Mosquitto host to adapter host

### Logical flow

```text
Mosquitto / ChirpStack host
  → MQTT/TCP
  → Python adapter on 177.104.61.21
```

### Headers visible in the pcap

```text
IP header
  → TCP header
    → MQTT control packet
      → MQTT topic
      → MQTT payload
```

The important observed connection is:

```text
177.104.61.23:1883 → 177.104.61.21:<ephemeral-port>
```

This shows that the adapter is consuming MQTT events from the cloud broker.

## Stage 3 — Adapter to Magistrala/SuperMQ HTTP adapter

### Logical flow

```text
Python adapter
  → HTTP POST
  → SenML payload
  → supermq-http:8008
```

### Headers visible in the pcap

```text
IP header
  → TCP header
    → HTTP request
      → HTTP headers
        → Content-Type: application/senml+json
        → Authorization: Client ...
      → SenML JSON payload
```

The important observed HTTP request pattern is:

```text
POST /m/<domain-id>/c/<channel-id>/ HTTP/1.1
Content-Type: application/senml+json
Authorization: Client <token>
```

The platform responds with:

```text
HTTP/1.1 202 Accepted
```

This is one of the clearest transition points in the capture: the original MQTT-originated sensor event becomes an HTTP/SenML platform ingestion request.

## Stage 4 — Magistrala/SuperMQ internal pipeline

### Logical flow

```text
supermq-http
  → NATS / internal services
  → timescale-writer
  → timescale-db
```

### Headers visible in the pcap

Depending on the internal segment, the trace shows:

```text
IP/TCP
  → NATS messages on 4222
```

and:

```text
IP/TCP
  → PostgreSQL protocol on 5432
```

The important internal endpoints are:

```text
supermq-http:                 172.18.0.45
supermq-nats:                 172.18.0.20
magistrala-timescale-writer:  172.18.0.3
magistrala-timescale-db:      172.18.0.38
```

This demonstrates that once data enters the platform, the original MQTT headers are no longer present. The data is now represented by platform-internal service protocols.

## About LoRaWAN headers

The RPi also runs a LoRaWAN packet forwarder. However, for this specific production path, the packet forwarder points to TTN rather than directly to `177.104.61.23`.

Therefore, the captured end-to-end path to the platform does not include a direct LoRaWAN UDP 1700 segment from the RPi to the private ChirpStack host.

The LoRaWAN-related process remains relevant because it is part of the gateway environment, but the platform path captured here is:

```text
sensor/serial script → MQTT → cloud broker → adapter → HTTP/SenML → platform storage
```

## Main interpretation

The observation is not transported by a single end-to-end packet. Instead, it is successively re-encapsulated:

```text
Sensor value
  → MQTT payload
  → MQTT event
  → adapter-side decoded value
  → SenML JSON
  → platform message
  → database row/record
```

Thus, the “headers” are different at each boundary. This is the central empirical result of the capture.
