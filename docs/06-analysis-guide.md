# 06 — Analysis guide

This guide explains how to inspect the packet captures.

## Do not use `cat` on `.pcap` files

Packet capture files are binary. Use `tcpdump`, Wireshark, or tshark.

For a quick text view:

```bash
tcpdump -nn -r file.pcap
```

For ASCII payloads:

```bash
tcpdump -nn -A -r file.pcap
```

## Locate marker packets

Markers use UDP port `9999`.

```bash
tcpdump -nn -A -r file.pcap 'udp port 9999'
```

In Wireshark:

```text
udp.port == 9999
```

Look for payloads such as:

```text
MARKER run=lorawan-e2e-r04-20260429 event=START ...
MARKER run=lorawan-e2e-r04-20260429 event=STOP ...
```

## Inspect RPi MQTT traffic

On the RPi capture:

```bash
tcpdump -nn -A -r lorawan-e2e-r04-20260429__G1_RPI_GATEWAY_ANY_MQTT_AND_UDP1700.pcap \
  'host 177.104.61.23 and tcp port 1883'
```

In Wireshark:

```text
ip.addr == 177.104.61.23 && tcp.port == 1883
```

Expected observations:

```text
192.168.1.126 → 177.104.61.23:1883
MQTT publish messages
sensor payloads such as Atmos41_WS_NSAAB and Teros12_S4_NSAAB
```

## Inspect ChirpStack/Mosquitto external traffic

On `C1A`:

```bash
tcpdump -nn -A -r lorawan-e2e-r04-20260429__C1A_CHIRPSTACK_EXTERNAL_ENS3__177.104.61.23.pcap \
  'tcp port 1883'
```

In Wireshark:

```text
tcp.port == 1883
```

Expected observations:

```text
MQTT traffic involving 177.104.61.23
traffic to/from 177.104.61.21
topics and payloads associated with the sensor stream
```

## Inspect adapter-side external MQTT traffic

On `C2A`:

```bash
tcpdump -nn -A -r lorawan-e2e-r04-20260429__C2A_MAGISTRALA_EXTERNAL_ENS3__177.104.61.21.pcap \
  'host 177.104.61.23 and tcp port 1883'
```

Expected observation:

```text
177.104.61.23:1883 → 177.104.61.21:<ephemeral-port>
```

This confirms that the Python adapter is consuming MQTT data from the cloud broker.

## Inspect HTTP/SenML ingestion into Magistrala/SuperMQ

On `C2B`:

```bash
tcpdump -nn -A -r lorawan-e2e-r04-20260429__C2B_MAGISTRALA_DOCKER_BRIDGE__br-2b65035ed9b3.pcap \
  'tcp port 8008'
```

In Wireshark:

```text
tcp.port == 8008
```

Look for:

```text
POST /m/<domain-id>/c/<channel-id>/ HTTP/1.1
Content-Type: application/senml+json
Authorization: Client ...
HTTP/1.1 202 Accepted
```

This is the transition where MQTT-originated data enters the IoT platform as HTTP/SenML.

## Inspect NATS/internal services

On `C2B`:

```bash
tcpdump -nn -A -r lorawan-e2e-r04-20260429__C2B_MAGISTRALA_DOCKER_BRIDGE__br-2b65035ed9b3.pcap \
  'tcp port 4222'
```

In Wireshark:

```text
tcp.port == 4222
```

Expected observation:

```text
NATS traffic involving 172.18.0.20
```

## Inspect TimescaleDB/PostgreSQL traffic

On `C2B`:

```bash
tcpdump -nn -r lorawan-e2e-r04-20260429__C2B_MAGISTRALA_DOCKER_BRIDGE__br-2b65035ed9b3.pcap \
  'host 172.18.0.3 or host 172.18.0.38 or tcp port 5432'
```

In Wireshark:

```text
tcp.port == 5432
```

Expected observation:

```text
magistrala-timescale-writer 172.18.0.3
magistrala-timescale        172.18.0.38
PostgreSQL protocol on 5432
```

## Suggested analysis order

1. Open the RPi pcap and inspect MQTT to `177.104.61.23:1883`.
2. Open C1A and inspect MQTT on `177.104.61.23`.
3. Open C2A and inspect MQTT from `177.104.61.23` to `177.104.61.21`.
4. Open C2B and inspect HTTP/SenML on port `8008`.
5. Continue in C2B with NATS `4222` and PostgreSQL `5432`.

This order follows the logical path of the observation.
