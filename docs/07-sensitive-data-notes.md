# 07 — Sensitive data notes

This capture was collected from a production environment.

The packet captures may contain:

```text
real public IP addresses
private/local IP addresses
MQTT topics
sensor payloads
HTTP paths
domain IDs
channel IDs
client keys or authorization tokens
database protocol traffic
service/container topology
```

The light artifacts are easier to share, but they are not fully anonymized.

## Sensitive fields observed conceptually

Examples of sensitive data types that may appear in the traces:

```text
Authorization: Client <token>
POST /m/<domain-id>/c/<channel-id>/
MQTT topics carrying device/application identifiers
SenML payload values
container IP topology
```

## DISCLAIMER

```text
This repository contains packet captures from a production IoT deployment. The captures are intended for private research collaboration and may contain real payloads, topics, identifiers, and credentials. Do not redistribute publicly without sanitization.
```
