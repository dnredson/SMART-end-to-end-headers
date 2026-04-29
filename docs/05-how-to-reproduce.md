# 05 — How to reproduce the capture

This document records the logic used to run the official capture.

## General approach

A common `RUN_ID` was used across all hosts:

```bash
RUN_ID="lorawan-e2e-r04-20260429"
```

Each host created the same directory structure:

```bash
CAP_ROOT="$HOME/iot-e2e-capture/$RUN_ID"

mkdir -p "$CAP_ROOT/captures" "$CAP_ROOT/logs" "$CAP_ROOT/notes"
```

Each host saved:

1. metadata;
2. clock synchronization data;
3. host/network inventory;
4. process/service inventory;
5. packet captures;
6. validation summaries;
7. SHA256 hashes of capture files.

## Clock synchronization

Before capture, each host recorded:

```bash
timedatectl status
chronyc tracking
chronyc sources -v
```

The captures are not intended for sub-millisecond timing claims. The timestamps are sufficient to correlate event order and approximate propagation across hosts.

## Marker packets

UDP marker packets on port `9999` were used to mark the start and stop of capture windows:

```bash
echo "MARKER run=$RUN_ID event=START host=<HOST_ID> point=<POINT_ID>" | nc -u -w1 <TARGET_IP> 9999
echo "MARKER run=$RUN_ID event=STOP host=<HOST_ID> point=<POINT_ID>"  | nc -u -w1 <TARGET_IP> 9999
```

These packets are artificial markers. They are not part of the IoT production path. Their purpose is to make packet traces easier to navigate.

## Recommended capture duration

For production systems where messages cannot be manually triggered, use a capture window long enough to catch naturally generated sensor updates.

The official run used:

```bash
DURATION_S=1800
```

which corresponds to 30 minutes.

## RPi capture

The RPi capture focused on:

```text
MQTT/TCP 1883
UDP 1700
UDP 9999 markers
```

Main filter:

```bash
"((tcp port 1883) or (udp port 1700) or (udp port 9999)) and not port 22"
```

## ChirpStack/Mosquitto host capture

The external interface capture focused on:

```text
UDP 1700
MQTT/TCP 1883
traffic to/from 177.104.61.21
UDP 9999 markers
```

Main filter:

```bash
"udp port 1700 or host 177.104.61.21 or tcp port 1883 or tcp port 8080 or udp port 9999"
```

The loopback capture was auxiliary and focused on:

```bash
"tcp port 1883 or tcp port 8080 or tcp port 5432 or tcp port 6379 or udp port 9999"
```

## Magistrala/adapter host capture

The external interface capture focused on MQTT exchange with the ChirpStack/Mosquitto host:

```bash
"host 177.104.61.23 or tcp port 1883 or tcp port 8883 or tcp port 80 or tcp port 443 or udp port 9999"
```

The Docker bridge capture focused on Magistrala/SuperMQ internal services:

```bash
"host 172.18.0.21 or host 172.18.0.16 or host 172.18.0.45 or host 172.18.0.3 or host 172.18.0.38 or host 172.18.0.20 or host 172.18.0.5 or tcp port 1883 or tcp port 8008 or tcp port 9012 or tcp port 5432 or tcp port 4222 or tcp port 5672 or udp port 9999"
```

## Packaging

After capture, light artifacts were produced to avoid including unnecessary large auxiliary traces.

Recommended package names:

```text
lorawan-e2e-r04-20260429_rpi_LIGHT.tar.gz
lorawan-e2e-r04-20260429_chirpstack_LIGHT.tar.gz
lorawan-e2e-r04-20260429_magistrala_LIGHT.tar.gz
```
