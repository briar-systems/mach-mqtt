# mach-mqtt

MQTT 3.1.1 implemented in Mach: a protocol library (packet codec, client, broker core) and `brokerd`, a standalone broker daemon.

## Scope

The 3.1.1 subset that covers real-world local deployments:

- QoS 0 and 1 (no QoS 2)
- retained messages
- Last Will and Testament
- username/password authentication
- topic wildcards (`+`, `#`)

Out of scope: MQTT 5, bridging, clustering.

## Status

Pre-alpha. API unstable. Nothing works yet.
