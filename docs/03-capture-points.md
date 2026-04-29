# 03 — Capture points

The capture was split into multiple files to avoid a single noisy trace and to preserve the structure of the path.

## G1 — Raspberry Pi gateway/sensor host

### File

```text
lorawan-e2e-r04-20260429__G1_RPI_GATEWAY_ANY_MQTT_AND_UDP1700.pcap
```

### Interface

```text
any
```

### Main filter

```text
((tcp port 1883) or (udp port 1700) or (udp port 9999)) and not port 22
```

### Purpose

This capture shows the first production-visible network segment:

```text
RPi → 177.104.61.23:1883
```

It captures MQTT messages published by `EnvData.py` from the RPi to the cloud host.

It also includes UDP 1700 to observe any packet-forwarder traffic, although the relevant platform path is MQTT.

## C1A — ChirpStack/Mosquitto external interface

### File

```text
lorawan-e2e-r04-20260429__C1A_CHIRPSTACK_EXTERNAL_ENS3__177.104.61.23.pcap
```

### Interface

```text
ens3
```

### Main filter

```text
udp port 1700 or host 177.104.61.21 or tcp port 1883 or tcp port 8080 or udp port 9999
```

### Purpose

This capture shows:

1. incoming MQTT messages from the RPi;
2. MQTT communication involving the adapter host;
3. potential UDP 1700 LoRaWAN packet-forwarder traffic;
4. START/STOP markers.

This is the main cloud-side trace for the ChirpStack/Mosquitto host.

## C1B — ChirpStack loopback

### File

```text
lorawan-e2e-r04-20260429__C1B_CHIRPSTACK_LOOPBACK__lo.pcap
```

### Interface

```text
lo
```

### Main filter

```text
tcp port 1883 or tcp port 8080 or tcp port 5432 or tcp port 6379 or udp port 9999
```

### Purpose

This is an auxiliary capture for local interactions within the ChirpStack host, such as Mosquitto, ChirpStack, PostgreSQL, and Redis.

This file can be large and is not essential for the high-level header narrative.

## C2A — Magistrala/adapter external interface

### File

```text
lorawan-e2e-r04-20260429__C2A_MAGISTRALA_EXTERNAL_ENS3__177.104.61.21.pcap
```

### Interface

```text
ens3
```

### Main filter

```text
host 177.104.61.23 or tcp port 1883 or tcp port 8883 or tcp port 80 or tcp port 443 or udp port 9999
```

### Purpose

This capture shows the MQTT connection between the ChirpStack/Mosquitto host and the Python adapter host.

The important observed flow is:

```text
177.104.61.23:1883 → 177.104.61.21:<ephemeral-port>
```

This demonstrates the handoff from the ChirpStack/Mosquitto cloud host to the adapter.

## C2B — Magistrala Docker bridge

### File

```text
lorawan-e2e-r04-20260429__C2B_MAGISTRALA_DOCKER_BRIDGE__br-2b65035ed9b3.pcap
```

### Interface

```text
br-2b65035ed9b3
```

### Main filter

```text
host 172.18.0.21 or host 172.18.0.16 or host 172.18.0.45 or
host 172.18.0.3 or host 172.18.0.38 or host 172.18.0.20 or
host 172.18.0.5 or tcp port 1883 or tcp port 8008 or tcp port 9012 or
tcp port 5432 or tcp port 4222 or tcp port 5672 or udp port 9999
```

### Purpose

This is the most important trace for the platform internals.

It shows:

```text
adapter/host bridge side
  → supermq-http:8008
  → NATS/internal services
  → timescale-writer
  → timescale-db:5432
```

This capture demonstrates that the original MQTT event is transformed by the adapter into HTTP/SenML before entering the Magistrala/SuperMQ pipeline.
