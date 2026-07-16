# MQTT

#distributed-systems #messaging #iot #pub-sub #protocol

---

## What and Why

**MQTT (Message Queuing Telemetry Transport)** is a lightweight pub/sub messaging protocol designed for constrained devices over unreliable, high-latency networks.

Originally designed in 1999 by Andy Stanford-Clark (IBM) and Arlen Nipper to monitor oil pipelines over satellite links — where bandwidth was expensive, connections dropped constantly, and devices ran on battery. The protocol reflects those constraints at every design decision.

Today it is the dominant protocol for IoT — sensors, industrial equipment, smart home devices, vehicle telemetry — and increasingly used in any system requiring lightweight, persistent, bidirectional messaging.

> [!note] "Message Queuing" in the name is a misnomer
> MQTT does not use queues in the traditional sense. Messages are not stored and forwarded through a queue — they are routed through a broker based on topic subscriptions. The name is historical.

---

## Architecture

MQTT is **broker-mediated pub/sub**. There is no direct connection between publishers and subscribers.

```
Publisher A ──┐
              │   PUBLISH msg
              ▼
           [Broker]  ──→ subscriber list for topic
              │
              ├──→ Subscriber X
              ├──→ Subscriber Y
              └──→ Subscriber Z

Publisher B ──→ [Broker] ──→ different subscribers
```

- **Publisher** — sends a message to a topic. Has no knowledge of subscribers.
- **Subscriber** — registers interest in a topic. Has no knowledge of publishers.
- **Broker** — the central routing authority. All messages flow through it.

This decoupling is fundamental. Publishers and subscribers don't know about each other, don't need to be online simultaneously (with persistent sessions), and can scale independently.

---

## Topics

Topics are UTF-8 strings, hierarchical, slash-delimited.

```
home/living-room/temperature
sensors/factory-1/machine-42/pressure
vehicles/truck-007/gps/lat
```

No topic pre-registration needed. Publishers publish, subscribers subscribe — the broker handles it.

### Wildcards

| Wildcard | Level | Example | Matches |
|---|---|---|---|
| `+` | Single level | `home/+/temperature` | `home/living-room/temperature`, `home/bedroom/temperature` |
| `#` | Multi-level (must be last) | `home/living-room/#` | `home/living-room/temperature`, `home/living-room/humidity/sensor-1` |

```
home/+/temperature     matches:  home/kitchen/temperature
                       no match: home/kitchen/oven/temperature  (two levels)

home/#                 matches:  home/kitchen/temperature
                       matches:  home/kitchen/oven/temperature
                       matches:  home/                          (zero levels)
```

### Reserved Topics

Topics beginning with `$` are reserved for broker use. `$SYS/` is a de-facto standard for broker statistics:

```
$SYS/broker/clients/connected
$SYS/broker/messages/received
$SYS/broker/uptime
```

Wildcard subscriptions to `#` do **not** match `$SYS/` topics — intentional. You can't accidentally subscribe to broker internals with a catch-all.

### Topic Design Anti-Patterns

```
# Anti-pattern: data embedded in topic names
sensors/temperature/23.5/celsius/device-42
→ makes wildcard filtering nearly impossible; topic names are routing keys, not payloads

# Anti-pattern: subscribing to # at root in production
→ every message flows to your subscriber; overwhelms slow consumers

# Good pattern: structured hierarchy, data in payload
sensors/{device_id}/temperature
payload: { "value": 23.5, "unit": "celsius", "ts": 1720000000 }
```

---

## QoS — The Core of the Protocol

Quality of Service levels define the delivery guarantee between a sender and the broker, and separately between the broker and each subscriber.

### QoS 0 — At-Most-Once (Fire and Forget)

```
Publisher          Broker
    │                │
    │── PUBLISH ────→│
    │                │ (may or may not deliver to subscribers)
```

No ACK. No retry. If the packet drops, the message is lost. Suitable for sensor readings where a missed sample is acceptable and bandwidth is precious.

### QoS 1 — At-Least-Once

```
Publisher          Broker
    │                │
    │── PUBLISH ────→│  (packet ID: 1001)
    │                │
    │←── PUBACK ─────│
    │                │
(if no PUBACK received within timeout, publisher resends with DUP=1)
```

Publisher stores the message until PUBACK is received. If the connection drops before PUBACK arrives, publisher retransmits — meaning the broker may receive the same message twice. Subscribers **must tolerate duplicates**.

The `DUP` flag in the retransmitted PUBLISH signals the broker that this may be a duplicate. The broker does not deduplicate — it delivers again.

### QoS 2 — Exactly-Once (Four-Way Handshake)

The most expensive. Two round trips to ensure exactly-once delivery.

```
Publisher          Broker             Subscriber
    │                │                    │
    │── PUBLISH ────→│                    │
    │   (ID: 1001)   │── PUBLISH ────────→│
    │                │   (ID: 1001)       │
    │←── PUBREC ─────│                    │
    │   (ID: 1001)   │                    │
    │                │                    │
    │── PUBREL ──────│                    │
    │   (ID: 1001)   │                    │
    │                │── deliver to app   │
    │←── PUBCOMP ────│                    │
    │   (ID: 1001)   │                    │
```

**Why four messages?**

- PUBLISH → PUBREC: broker has received the message, stored it
- PUBREL → PUBCOMP: broker has delivered to subscribers, safe to discard

If the publisher crashes after PUBLISH but before PUBREC: it retransmits on reconnect. Broker deduplicates using the packet ID.
If the publisher crashes after PUBREC but before PUBREL: broker holds the message. On reconnect, publisher sends PUBREL (it knows PUBREC was received). Broker completes delivery exactly once.

> [!important] QoS is negotiated per message, per subscription
> If a publisher sends at QoS 2 but a subscriber subscribed at QoS 1, the broker delivers at QoS 1 to that subscriber.
> **Effective QoS = min(publish QoS, subscribe QoS)**
> You can't get stronger guarantees than the weakest link in the chain.

### QoS Comparison

| | QoS 0 | QoS 1 | QoS 2 |
|---|---|---|---|
| Delivery guarantee | At-most-once | At-least-once | Exactly-once |
| Network round trips | 0 | 1 | 2 |
| Duplicate possible | N/A | Yes | No |
| State stored | None | Until PUBACK | Until PUBCOMP |
| Use case | Sensor streams | Commands, events | Financial, billing |

---

## Retained Messages

A message published with the `RETAIN` flag set is stored by the broker — one per topic. Any new subscriber to that topic immediately receives the last retained message, even if it was published long ago.

```
Publisher publishes: RETAIN=1, topic=sensor/temp, payload=23.5

[Broker stores: sensor/temp → 23.5]

New subscriber joins sensor/temp → immediately receives 23.5
(no need to wait for the next publish)
```

**Only one retained message per topic.** Each new retained publish replaces the previous one. Delete by publishing an empty payload with RETAIN=1.

This solves a classic pub/sub problem: a subscriber that joins after the last event has no idea what the current state is. Retained messages give you "last known value" semantics — critical for device state topics.

---

## Last Will and Testament (LWT)

A message the broker publishes on the client's behalf if the client disconnects ungracefully.

Declared at CONNECT time:

```
CONNECT {
  clientId: "device-42",
  will: {
    topic: "devices/device-42/status",
    payload: "offline",
    qos: 1,
    retain: true
  }
}
```

The broker publishes the LWT when:
- The TCP connection drops without a DISCONNECT packet
- The keepalive timer expires (client stopped sending PINGREQs)
- The broker closes the connection due to a protocol error

**Common pattern — device availability tracking:**

```
On connect:  publish devices/{id}/status = "online"  (retain=true)
LWT:         devices/{id}/status = "offline"         (retain=true, qos=1)
```

Any subscriber to `devices/+/status` gets real-time device presence without polling.

---

## Persistent Sessions

By default (`clean_session=true`), when a client disconnects all its subscriptions and queued messages are discarded. `clean_session=false` tells the broker to persist session state across reconnects.

**What the broker stores for a persistent session:**
- All subscriptions
- Undelivered QoS 1 and QoS 2 messages
- Partially completed QoS 2 handshakes

On reconnect, the client receives all messages published while it was offline (up to broker limits). Session is keyed on `clientId` — same client ID on reconnect resumes the session.

> [!note] Persistent sessions on the client side
> The client must also store: partially sent QoS 1/2 messages and in-flight handshake state — so it can retransmit after reconnect.

---

## Keepalive and Half-Open Connections

MQTT runs over long-lived TCP connections. TCP does not detect dead connections quickly — a broken link (cable unplugged, NAT table expiry) leaves the broker holding a "half-open" connection with no traffic.

**Keepalive:** client sends PINGREQ if no message has been sent within the keepalive interval (set at CONNECT). Broker responds with PINGRESP.

```
keepalive = 60s

t=0s    client sends PUBLISH
t=55s   client sends PINGREQ   (no message in 55s)
t=55s   broker sends PINGRESP
t=115s  no PINGREQ or message → broker treats client as disconnected
```

Broker threshold: **1.5× the keepalive value** before declaring the client dead and publishing the LWT.

---

## Packet Format

MQTT packets are compact by design. Every packet has:

```
┌─────────────────────────────────┐
│ Fixed Header (2+ bytes)         │
│  [4-bit type][4-bit flags]      │
│  [variable-length remaining]    │
├─────────────────────────────────┤
│ Variable Header (type-specific) │
│  topic name, packet ID, etc.    │
├─────────────────────────────────┤
│ Payload                         │
│  message content (opaque bytes) │
└─────────────────────────────────┘
```

**Remaining length encoding** — similar to protobuf varint. Each byte uses 7 bits for value, 1 bit as continuation flag:

```
< 128 bytes    → 1 byte
< 16,384 bytes → 2 bytes
< 2 MB         → 3 bytes
< 256 MB       → 4 bytes (maximum packet size)
```

A minimal PUBLISH packet for a small sensor reading can be **under 10 bytes total**. Compare to HTTP/1.1 which has hundreds of bytes of headers alone.

### Packet Types

| Packet | Direction | Purpose |
|---|---|---|
| CONNECT / CONNACK | Client→Broker / Broker→Client | Session establishment |
| PUBLISH | Both | Send a message |
| PUBACK | Both | QoS 1 acknowledgement |
| PUBREC / PUBREL / PUBCOMP | Both | QoS 2 four-way handshake |
| SUBSCRIBE / SUBACK | Client→Broker / Broker→Client | Register topic interest |
| UNSUBSCRIBE / UNSUBACK | Client→Broker / Broker→Client | Deregister topic interest |
| PINGREQ / PINGRESP | Client→Broker / Broker→Client | Keepalive |
| DISCONNECT | Client→Broker | Clean disconnect |

---

## MQTT 5 vs MQTT 3.1.1

MQTT 5 (2019) added significant capabilities missing from 3.1.1 (2014).

### Reason Codes on All ACKs
3.1.1 CONNACK had two return codes: accepted or refused. MQTT 5 ACK packets carry detailed reason codes. A SUBACK can now tell you "topic filter invalid" vs "quota exceeded" vs "not authorised" — instead of a binary fail.

### User Properties
Key-value metadata attachable to any packet. Enables routing metadata, trace IDs, content-type hints — without embedding them in the payload.

### Message Expiry Interval
Set a TTL on a PUBLISH. If the broker can't deliver it within the interval, it drops the message. Critical for time-sensitive telemetry — you don't want a device receiving stale sensor commands 10 minutes after the fact.

### Topic Aliases
Replace a long topic string with a short integer for the duration of a connection. Reduces overhead on high-frequency, same-topic publishers.

```
First message:  topic="sensors/factory-1/machine-42/pressure", alias=1
Next messages:  alias=1  (no topic string, saves bandwidth)
```

### Shared Subscriptions
`$share/{GroupName}/{filter}` — broker distributes messages across subscribers in a group. Each message goes to exactly one subscriber in the group (round-robin or broker-defined).

```
$share/workers/jobs/#

Worker A ──┐
Worker B ──┤── all subscribed to same group
Worker C ──┘

Message → broker → delivered to Worker B (round-robin)
Next message → broker → delivered to Worker C
```

This is **load balancing** over MQTT — impossible cleanly in 3.1.1.

### Flow Control (Receive Maximum)
Client and broker advertise the maximum number of in-flight QoS 1/2 messages they can handle simultaneously. Prevents fast publishers from overwhelming slow subscribers.

### Request/Response Pattern
`ResponseTopic` and `CorrelationData` properties enable RPC-style request/response over MQTT without a separate correlation mechanism:

```
Requester publishes:
  topic: devices/device-42/command
  ResponseTopic: responses/client-xyz/1234
  CorrelationData: <opaque bytes>

Device responds:
  topic: responses/client-xyz/1234
  CorrelationData: <same opaque bytes>  ← requester matches response to request
```

---

## Security

### Transport Security
- Port **1883** — plaintext TCP (development only)
- Port **8883** — MQTT over TLS
- Port **443** — MQTT over WebSockets over TLS (firewall-friendly)

### Authentication
```
CONNECT {
  username: "device-42",
  password: "<token>"   ← plaintext in packet, requires TLS
}
```

**Mutual TLS (mTLS):** both broker and client present certificates. Stronger than username/password — the device's private key proves identity. Standard in industrial IoT.

**MQTT 5 Enhanced Authentication:** AUTH packet enables multi-step SASL-style flows (e.g., SCRAM-SHA-256). The broker and client exchange multiple AUTH packets before CONNACK — supports challenge-response without username/password in the clear.

### Authorisation
Topic-level ACLs on the broker:
- `device-42` can publish to `sensors/device-42/#`
- `device-42` cannot publish to `sensors/device-99/#`
- `dashboard` can subscribe to `sensors/#` but not publish

---

## Scaling

The broker is the central routing authority — and the bottleneck.

### Broker Clustering
**EMQX** (Erlang) — nodes form a cluster via a distributed Erlang mesh. Topic subscriptions are replicated across nodes via a distributed routing table. A message published to any node is routed to the correct subscriber regardless of which node they're connected to.

**VerneMQ** — similar Erlang-based clustering model.

Clustering is hard in MQTT because sessions are stateful — QoS 1/2 in-flight state and subscription lists must be consistent across nodes. This is why naive horizontal scaling (just add brokers) doesn't work; the state must be shared or partitioned correctly.

### Bridge Mode
Brokers connect to each other. A message published on Broker A can be forwarded to Broker B if configured. Used for:
- Geographic distribution (local broker per region, central broker for aggregation)
- Edge-to-cloud architectures (local MQTT broker on-premise bridges to cloud broker)

### The Stateful Protocol Problem
QoS 2 requires both the broker and client to maintain in-flight state. This means you can't load-balance MQTT connections across stateless broker replicas the way you can with HTTP. Either:
- Use session affinity (sticky routing) — same client always hits the same broker node
- Use shared state in a distributed store (what EMQX does)
- Accept QoS 0/1 only and use idempotent consumers

---

## MQTT vs Other Protocols

| | MQTT | Kafka | AMQP (RabbitMQ) | HTTP/SSE |
|---|---|---|---|---|
| Pattern | Pub/sub | Log streaming | Queue + pub/sub | Request/response |
| Broker | Required | Required | Required | Optional |
| Message retention | LWT + retained only | Configurable (days/forever) | Per-queue config | None |
| QoS | 0, 1, 2 | At-least-once | At-most/at-least once | At-most-once |
| Protocol overhead | Tiny (binary) | Moderate | Heavy | Heavy (HTTP headers) |
| Connection | Long-lived TCP | Long-lived TCP | Long-lived TCP | Short-lived or SSE |
| Fan-out | Native | Consumer groups | Exchange/bindings | N/A |
| Primary use | IoT, telemetry | Event streaming, analytics | Enterprise messaging | Web APIs |

---

## Key Takeaways

- MQTT is a **broker-mediated pub/sub protocol** — no direct publisher-subscriber connection
- **QoS levels** are per-message, per-subscription delivery contracts; effective QoS = min(publish, subscribe)
- **QoS 2 exactly-once** requires 4-way handshake and stateful tracking on both sides — expensive but reliable
- **Retained messages** give new subscribers the last known value immediately — solves late-joiner problem
- **LWT** is the standard pattern for device presence detection
- **Persistent sessions** let offline clients receive missed QoS 1/2 messages on reconnect
- **MQTT 5 shared subscriptions** enable proper load balancing across consumer groups
- **Scaling MQTT horizontally is hard** because sessions are stateful; use clustering (EMQX/VerneMQ) or session affinity
- Always use **TLS (port 8883)** — username/password are plaintext in CONNECT
- MQTT packets are tiny by design — a sensor reading can be under 10 bytes total

---

## Related Notes
- [[RPC and APIs]] — RPC vs pub/sub as messaging paradigms
- [[Delivery Semantics]] — at-most-once / at-least-once / exactly-once explained in depth; QoS levels are concrete implementations of these
- [[Distributed Systems/Consistency Models]] — delivery semantics and exactly-once guarantees in distributed systems

