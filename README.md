# SMART End-to-End Headers — IoTinuum Capture

This repository documents an empirical packet capture of a smart-agriculture IoT data path. It was created to answer a practical question:

> What do the headers look like as a sensor observation moves from the field side to the IoT platform and storage?

The short answer is that there is no single universal end-to-end header from the sensor to the database. The same logical observation is carried through a sequence of communication segments, and each segment has its own communicating participants, protocols, and headers.

This README first explains the path using the conceptual language of Carlos Kamienski’s “Does the End-to-End Argument Still Matter?” presentation. The packet-capture details come afterward.

---

## 1. Conceptual view: many composed end-to-end communications

The captured path is best understood as a composition of smaller end-to-end communications across the IoTinuum, rather than as one direct network conversation from the physical sensor to the database.

In the presentation’s terms, the path crosses the thing/mist side and the cloud/platform side. Each step has a producer and a consumer for that specific communication segment.

```text
RPi sensor script
  → MQTT/TCP
  → Mosquitto/ChirpStack cloud host
  → MQTT/TCP
  → Python adapter
  → HTTP/SenML
  → Magistrala/SuperMQ platform
  → internal messaging/storage
  → TimescaleDB
```

The important distinction is:

```text
Network-level endpoint:
  The endpoint of one protocol conversation, such as an MQTT client/server pair
  or an HTTP client/server pair.

Application-level observation path:
  The larger logical path followed by the sensor observation from acquisition
  to platform storage.
```

Therefore, the endpoints in the packet captures are not always the final semantic endpoints of the IoT application. They are the participants that produce and consume headers in each local communication segment.

---

## 2. Participants and roles

| Participant | IoTinuum role | Role in this capture |
|---|---|---|
| Raspberry Pi `mist-5` | Thing/mist side | Local producer of sensor observations |
| `EnvData.py` script | Application/service component on the RPi | Reads local sensor/serial data and publishes MQTT messages |
| Mosquitto/ChirpStack host `177.104.61.23` | Cloud-side IoT infrastructure | Receives MQTT messages and exposes them to the next component |
| Python adapter on `177.104.61.21` | Cloud-side IoT service infrastructure | Consumes MQTT messages and converts them to HTTP/SenML |
| Magistrala/SuperMQ | IoT platform infrastructure | Receives platform ingestion requests and processes observations |
| TimescaleDB/PostgreSQL | Storage component | Stores the resulting observation records |

In base-Internet terms, these are ordinary IP hosts communicating with one another. In IoT-service terms, Mosquitto/ChirpStack, the adapter, Magistrala/SuperMQ, and TimescaleDB are infrastructure components of the IoT service overlay.

---

## 3. Communication segments and headers

### Segment 1 — RPi to Mosquitto/ChirpStack

```text
Producer:  RPi / EnvData.py
Consumer:  Mosquitto/ChirpStack host
Protocol:  MQTT over TCP/IP
```

Header stack visible in the capture:

```text
IP
  → TCP
    → MQTT
      → sensor payload
```

This segment is visible in the RPi capture and in the ChirpStack external-interface capture.

---

### Segment 2 — Mosquitto/ChirpStack to Python adapter

```text
Producer:  Mosquitto/ChirpStack host
Consumer:  Python adapter
Protocol:  MQTT over TCP/IP
```

Header stack visible in the capture:

```text
IP
  → TCP
    → MQTT
      → MQTT topic and payload
```

This shows the adapter consuming the MQTT stream from the cloud-side broker.

---

### Segment 3 — Python adapter to Magistrala/SuperMQ

```text
Producer:  Python adapter
Consumer:  Magistrala/SuperMQ HTTP service
Protocol:  HTTP with SenML payload
```

Header stack visible in the capture:

```text
IP
  → TCP
    → HTTP
      → HTTP headers
      → SenML JSON payload
```

This is the clearest transformation point. The data is no longer represented as an MQTT publication. It becomes an HTTP/SenML ingestion request to the IoT platform.

Observed request pattern:

```text
POST /m/<domain-id>/c/<channel-id>/ HTTP/1.1
Content-Type: application/senml+json
Authorization: Client <token>
```

The platform responds with:

```text
HTTP/1.1 202 Accepted
```

---

### Segment 4 — Magistrala/SuperMQ internal processing and storage

```text
Producer/consumer roles:  Internal Magistrala/SuperMQ services
Protocols:                NATS/internal service traffic and PostgreSQL/TimescaleDB
```

Header/protocol stacks visible in the Docker bridge capture:

```text
IP
  → TCP
    → NATS/internal service protocol
```

and:

```text
IP
  → TCP
    → PostgreSQL protocol
```

At this stage, the original MQTT headers are gone. The same logical observation is now represented as internal platform messages and database operations.

---

## 4. What was actually validated

The validated production path for this capture is:

```text
G1_RPI_GATEWAY
  /home/mist/Software/EnvData.py
  MQTT/TCP 1883
    ↓
C1_CHIRPSTACK_CLOUD
  Mosquitto / ChirpStack-side MQTT broker
  MQTT/TCP 1883
    ↓
C2_MAGISTRALA_ADAPTER
  /home/ubuntu/adapter/src/main.py
  HTTP/SenML
    ↓
Magistrala/SuperMQ Docker bridge
  supermq-http
    ↓
  NATS / internal services
    ↓
  timescale-writer
    ↓
  timescale-db
```

A separate LoRaWAN packet-forwarder process was observed on the RPi, but it points to TTN (`au1.cloud.thethings.network`) rather than directly to the private ChirpStack/Magistrala path captured here. For this run, the platform-bound production path starts with MQTT messages generated by `EnvData.py`.

---

## 5. Capture hosts

| Capture ID | Host | Role |
|---|---|---|
| `G1_RPI_GATEWAY` | `mist-5`, local IP `192.168.1.126` | Raspberry Pi sensor/gateway host |
| `C1_CHIRPSTACK_CLOUD` | `177.104.61.23` | ChirpStack/Mosquitto host |
| `C2_MAGISTRALA_ADAPTER` | `177.104.61.21` | Python adapter + Magistrala/SuperMQ Docker host |

---

## 6. Capture files

The light capture artifacts for this run are:

```text
lorawan-e2e-r04-20260429_rpi_LIGHT.tar.gz
lorawan-e2e-r04-20260429_chirpstack_LIGHT.tar.gz
lorawan-e2e-r04-20260429_magistrala_LIGHT.tar.gz
```

The actual packet captures are expected under:

```text
captures/
├── rpi/
├── chirpstack/
└── magistrala/
```

The most useful files for understanding the header transitions are:

```text
G1_RPI_GATEWAY_ANY_MQTT_AND_UDP1700.pcap
C1A_CHIRPSTACK_EXTERNAL_ENS3.pcap
C2A_MAGISTRALA_EXTERNAL_ENS3.pcap
C2B_MAGISTRALA_DOCKER_BRIDGE.pcap
```

The `C2B` Docker bridge capture is especially useful because it shows the transition from adapter-generated HTTP/SenML to the internal Magistrala/SuperMQ pipeline.

---

## 7. How to inspect the captures

Do not use `cat` on `.pcap` files. Use `tcpdump`, Wireshark, or tshark.

### Marker packets

Marker packets use UDP port `9999`.

```bash
tcpdump -nn -A -r file.pcap 'udp port 9999'
```

Wireshark filter:

```text
udp.port == 9999
```

### RPi MQTT publication

```bash
tcpdump -nn -A -r G1_RPI_GATEWAY_ANY_MQTT_AND_UDP1700.pcap \
  'host 177.104.61.23 and tcp port 1883'
```

Wireshark filter:

```text
ip.addr == 177.104.61.23 && tcp.port == 1883
```

### Adapter-side MQTT consumption

```bash
tcpdump -nn -A -r C2A_MAGISTRALA_EXTERNAL_ENS3.pcap \
  'host 177.104.61.23 and tcp port 1883'
```

### HTTP/SenML ingestion into Magistrala/SuperMQ

```bash
tcpdump -nn -A -r C2B_MAGISTRALA_DOCKER_BRIDGE.pcap \
  'tcp port 8008'
```

Look for:

```text
POST /m/<domain-id>/c/<channel-id>/ HTTP/1.1
Content-Type: application/senml+json
HTTP/1.1 202 Accepted
```

### Internal platform traffic

NATS/internal service traffic:

```bash
tcpdump -nn -A -r C2B_MAGISTRALA_DOCKER_BRIDGE.pcap \
  'tcp port 4222'
```

TimescaleDB/PostgreSQL traffic:

```bash
tcpdump -nn -r C2B_MAGISTRALA_DOCKER_BRIDGE.pcap \
  'tcp port 5432'
```

---

## 8. Main interpretation

The same logical sensor observation is successively re-encapsulated:

```text
sensor value
  → MQTT message
  → MQTT message consumed by adapter
  → HTTP/SenML request
  → internal platform message
  → database record
```

From the base-Internet viewpoint, the capture shows ordinary IP communication between hosts. From the IoT-service viewpoint, those individual communications compose one application-level path for the sensor observation.

This is the main empirical result of the capture: the end-to-end path exists at the level of the observation, but it is implemented as several composed end-to-end communications, each with its own protocol headers and participants.

---

## 9. Repository contents

```text
.
├── README.md
├── manifest.yaml
├── message-index-template.csv
└── docs/
    ├── 01-experiment-overview.md
    ├── 02-system-topology.md
    ├── 03-capture-points.md
    ├── 04-protocol-flow-and-headers.md
    ├── 05-how-to-reproduce.md
    ├── 06-analysis-guide.md
    ├── 07-sensitive-data-notes.md
    └── 08-summary-for-collaboration.md
```

For a compact collaborator-facing explanation, see:

```text
docs/08-summary-for-collaboration.md
```

For detailed packet analysis, see:

```text
docs/06-analysis-guide.md
```
