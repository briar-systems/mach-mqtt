# mach-mqtt

MQTT 3.1.1 implemented in Mach: a protocol library (packet codec, client, broker core) and `brokerd`, a standalone broker daemon.

## Scope

The 3.1.1 subset that covers real-world local deployments:

- QoS 0 and 1 (no QoS 2)
- retained messages
- Last Will and Testament
- username/password authentication
- topic wildcards (`+`, `#`)

Out of scope: MQTT 5, bridging, clustering, TLS.

## Layout

```
src/
  packet/      wire protocol: constants, remaining-length varint, field
               primitives, the Packet model + codec, and framed stream I/O
  broker/      topic-filter matching, broker core, and the accept loop
  client.mach  synchronous client library
  bin/
    brokerd.mach   the broker daemon
```

The library modules are reachable under the `mqtt` project id, e.g.
`use packet: mqtt.packet.packet;`, `use client: mqtt.client;`,
`use core: mqtt.broker.core;`.

## Build and test

```
mach build .          # builds the mqtt static lib and the brokerd binary
mach test .           # runs the codec round-trip, filter, and end-to-end tests
mach run . -- 1883    # build and run brokerd on a port (default 1883)
```

`brokerd [port]` binds `0.0.0.0:<port>` and serves one OS thread per connection.

## Client

```mach
use client: mqtt.client;
use ip:     std.net.ip;

var opts: client.Options = client.options("my-id");
var c: client.Client;
client.connect(?c, ip.endpoint(127, 0, 0, 1, 1883), ?opts);

client.subscribe(?c, "sensors/#", 1);
var msg: client.Message;
client.next_message(?c, ?msg);   # blocks; msg borrows the client's decode arena

client.publish(?c, "sensors/room", data, len, 1, 0);
client.disconnect(?c);
```

The client is single-threaded: use one connection per thread, and do not
interleave QoS 1 publishing with `next_message` on the same client.

## Authentication

The broker core accepts an optional username/password hook. When unset, all
connections are accepted.

```mach
use core: mqtt.broker.core;

fun check(user: str, pass: str) u8 { ... }   # return 1 to accept

core.set_auth(?broker, check);
```

## Known limitations (v0.1)

- **Persistent sessions**: `clean_session = 0` is accepted but sessions are not
  persisted across reconnects (no offline queueing); a reconnect starts fresh.
- **QoS 1 delivery**: inbound QoS 1 is acknowledged with PUBACK; delivery to
  subscribers is best-effort at the negotiated QoS without inflight-retry
  tracking (sufficient over a reliable local transport).
- **Slow consumers**: the broker holds a single lock across socket writes, so a
  stalled subscriber can delay delivery to others. This is a correctness-first
  v0.1 choice, not a scalability target.
- **Release builds of `brokerd`**: build and run `brokerd` with the default debug
  profile. The `--profile release` (`-O2`) build of the daemon currently
  miscompiles (a Mach codegen bug when `serve` is inlined into `main`); the
  library itself is correct at all optimization levels (the release test suite
  passes).
