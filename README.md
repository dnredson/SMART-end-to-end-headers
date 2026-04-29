# LoRaWAN/IoT End-to-End Capture — Sensor/RPi to IoT Platform Storage

This repository documents an end-to-end IoT data path captured across three machines:

1. a Raspberry Pi gateway/sensor host (`mist-5`);
2. a cloud host running ChirpStack/Mosquitto (`177.104.61.23`);
3. a cloud host running an MQTT-to-Magistrala adapter and Magistrala/SuperMQ in Docker (`177.104.61.21`).

The goal of this capture is to provide empirical packet-level evidence of how a sensor observation changes representation as it traverses different network and service domains. The capture was prepared to support discussion about end-to-end paths, protocol composition, and what the “headers” look like along the path.

The main takeaway is that there is no single universal end-to-end header from the physical sensor to the database. Instead, the observation is carried through a sequence of protocol contexts:

```text
RPi sensor/serial script
  → MQTT/TCP
  → Mosquitto/ChirpStack cloud host
  → MQTT/TCP
  → Python adapter
  → HTTP/SenML
  → Magistrala/SuperMQ internal services
  → NATS / writer services
  → TimescaleDB/PostgreSQL
```

## Repository contents

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

The actual packet captures are expected to be placed under:

```text
captures/
├── rpi/
├── chirpstack/
└── magistrala/
```

The light capture artifacts produced for this run are:

```text
lorawan-e2e-r04-20260429_rpi_LIGHT.tar.gz
lorawan-e2e-r04-20260429_chirpstack_LIGHT.tar.gz
lorawan-e2e-r04-20260429_magistrala_LIGHT.tar.gz
```

## Run ID

```text
lorawan-e2e-r04-20260429
```

## Capture hosts

| ID | Host | Role |
|---|---|---|
| `G1_RPI_GATEWAY` | `mist-5`, local IP `192.168.1.126` | Raspberry Pi sensor/gateway host |
| `C1_CHIRPSTACK_CLOUD` | `177.104.61.23` | ChirpStack/Mosquitto host |
| `C2_MAGISTRALA_ADAPTER` | `177.104.61.21` | Python adapter + Magistrala/SuperMQ Docker host |

## Validated path

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
  supermq-http → NATS/services → timescale-writer → timescale-db
```

A separate LoRaWAN packet forwarder process was also observed on the RPi, but it points to TTN (`au1.cloud.thethings.network`) rather than directly to `177.104.61.23`. For this capture, the platform-bound path is MQTT through `EnvData.py`.

## How to read this repository

Start with:

1. [`docs/01-experiment-overview.md`](docs/01-experiment-overview.md)
2. [`docs/02-system-topology.md`](docs/02-system-topology.md)
3. [`docs/04-protocol-flow-and-headers.md`](docs/04-protocol-flow-and-headers.md)
4. [`docs/06-analysis-guide.md`](docs/06-analysis-guide.md)

For collaboration purposes, [`docs/08-summary-for-collaboration.md`](docs/08-summary-for-collaboration.md) provides a compact explanation suitable for sharing.
# SMART-end-to-end-headers
