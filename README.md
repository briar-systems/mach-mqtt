# mach-mqtt

MQTT 3.1.1 implemented in Mach: a protocol library (packet codec, client, broker core) and `brokerd`, a standalone broker daemon.

## Scope

The 3.1.1 subset that covers real-world local deployments:

- QoS 0 and 1 (QoS 2 is rejected: the broker closes such a connection and the
  client refuses `publish(qos=2)` and caps a requested subscribe QoS at 1)
- retained messages
- Last Will and Testament, including on client-id takeover
- username/password authentication
- topic wildcards (`+`, `#`)

Out of scope: MQTT 5, bridging, clustering, TLS.

Whole packets are capped at 1 MiB (`frame.MAX_PACKET_SIZE`); a larger remaining
length is rejected before the body is allocated. Client ids must be non-empty.

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
client.next_message(?c, ?msg);   # blocks; msg valid until the next next_message

client.publish(?c, "sensors/room", data, len, 1, 0);
client.disconnect(?c);
```

The client is single-threaded: use one connection per thread. Application
messages that arrive while `subscribe`, `unsubscribe`, or a QoS 1 `publish` is
waiting for its ack are acknowledged and buffered (up to `client.QUEUE_CAP`),
then returned in order by `next_message`; if that bound is exceeded the control
call returns an error rather than dropping a message. The `Message` handed back
by `next_message` owns its topic and payload and stays valid until the following
`next_message` call, so intervening control calls do not invalidate it.

`client.options` defaults `keepalive` to 0 (disabled). A synchronous client
cannot send PINGREQ while blocked in `next_message`, so set a nonzero keepalive
only if the application drives `ping()` itself.

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
- **Keepalive is not enforced**: the broker decodes the CONNECT keepalive but
  does not disconnect a silent client at 1.5x keepalive, because the standard
  library exposes no socket read timeout (no `SO_RCVTIMEO`/`poll` wrapper on
  `tcp.Stream`). A will therefore fires on graceful disconnect and on client-id
  takeover, but on a silent network partition it is delayed until the transport
  itself reports the connection closed. Enforcing the deadline is blocked on a
  stdlib read-timeout primitive.
