# Delivery Semantics

#distributed-systems #messaging #reliability #kafka #mqtt

---

## The Problem

In a distributed system, a sender puts a message on the wire. The receiver is on the other side. The network is unreliable.

What guarantees can you make about whether the message arrives, and how many times?

That answer is the **delivery semantic**.

```
Sender                    Network                  Receiver
  │                         │                         │
  │── message ─────────────→│                         │
  │                         │                         │
  │           (packet drop? duplication? delay?)       │
  │                         │                         │
  │                         │──── message? ──────────→│
```

Three possible contracts:

---

## At-Most-Once

> The message is delivered **zero or one times**. It may be lost. It will never be delivered twice.

```
Sender sends message
Sender does NOT retry on failure
         │
         ├── network delivers → receiver gets it once ✓
         └── network drops    → message is lost ✗ (acceptable)
```

**How it works:** fire and forget. Send and move on. No ACK waited for, no retry on timeout.

**When to use:**
- Data where loss is acceptable — sensor telemetry, metrics, log events
- Real-time streams where a stale retransmitted value is worse than no value
- High-throughput pipelines where the overhead of ACKs outweighs the cost of occasional loss

**Examples in the wild:**
- MQTT QoS 0
- UDP (DNS, video streaming, gaming)
- Statsd metrics (intentionally lossy)
- `syslog` over UDP

**Cost:** cheapest. Zero round trips. Zero state stored on sender.

---

## At-Least-Once

> The message is delivered **one or more times**. It will never be permanently lost. It may be delivered as a duplicate.

```
Sender sends message
Sender waits for ACK
         │
         ├── ACK received → done ✓
         └── timeout / no ACK → RETRY
                  │
                  ├── message was lost → retry delivers it → ✓
                  └── message arrived, ACK was lost → retry is a DUPLICATE ⚠️
```

> [!important] The duplicate problem
> You cannot distinguish between "message was lost, retry is the first delivery" and "message arrived but ACK was lost, retry is a duplicate." From the sender's perspective they look identical. The receiver must handle both cases.

**How it works:** sender persists the message until it receives an ACK. On timeout or connection failure, sender retransmits. May transmit many times before getting an ACK.

**When to use:**
- Commands, events, notifications where loss is unacceptable
- Most messaging systems default to this — it's the practical sweet spot
- When receivers can be made idempotent (see below)

**Examples in the wild:**
- MQTT QoS 1
- Kafka (default producer config)
- HTTP with client-side retry
- SQS standard queues
- Most webhook systems

**Cost:** one round trip per message minimum. Sender must store message until ACK. Receiver must handle duplicates.

---

## Exactly-Once

> The message is delivered **exactly one time**. Never lost, never duplicated.

```
Sender sends message
         │
         ├── deliver to receiver
         │         │
         │         └── deduplicate if seen before
         │
         └── guarantee: receiver processes it once and only once
```

This sounds simple. It is extremely hard.

> [!important] Exactly-once is the hardest guarantee in distributed systems
> It requires coordination between sender, network, and receiver — all of which can fail independently. You cannot achieve true exactly-once delivery without either (a) idempotent receivers, or (b) distributed transactions.

**How it's actually implemented:**

**Option 1 — Idempotent producer + deduplication at receiver**
Sender attaches a unique message ID. Receiver tracks seen IDs and discards duplicates. This is "at-least-once delivery + at-most-once processing" — which *looks* like exactly-once from the application's perspective.

**Option 2 — Distributed transaction (2PC)**
Two-phase commit across sender and receiver. Both vote to commit, coordinator commits. Expensive, blocking, coordinator is a single point of failure. Rarely used in practice.

**Option 3 — Idempotent operations**
Make the operation itself safe to replay. Charging a card once vs twice matters. Setting `account_balance = 750` twice does not. Design operations to be naturally idempotent where possible.

**Examples in the wild:**
- MQTT QoS 2 (four-way handshake within a single broker)
- Kafka exactly-once semantics (EOS) — idempotent producer + transactional API
- AWS SQS FIFO queues (deduplication ID)

**Cost:** most expensive. Multiple round trips, state stored on both sides, deduplication infrastructure.

---

## Why Exactly-Once is Hard — The Core Insight

Imagine a sender sends a message and then crashes before receiving the ACK.

```
Sender ──── PUBLISH ────→ Receiver (processes it)
Sender crashes
         │
Receiver ── ACK ──────→ [sender is dead]

Sender restarts
         │
         └── "did my message arrive?" → no way to know
                    │
                    ├── don't retry → at-most-once (may have been lost)
                    └── retry       → at-least-once (may be a duplicate)
```

There is no third option without the receiver being part of the solution. This is why exactly-once always requires the receiver to participate — through deduplication, idempotency, or a distributed transaction.

The **Two Generals Problem** (and its generalisation, the **FLP impossibility theorem**) formally proves that guaranteed message delivery with consensus is impossible in an asynchronous network with failures. Exactly-once is a practical approximation, not a theoretical guarantee.

---

## Idempotency — The Practical Solution

Rather than solving exactly-once at the transport layer (expensive), most production systems use **at-least-once delivery + idempotent receivers**.

An operation is **idempotent** if applying it multiple times produces the same result as applying it once.

```
Idempotent:
  SET balance = 750             → same result if applied 10 times
  DELETE order WHERE id = 42    → same result if applied 10 times
  PUT /users/42 { name: "Alice" }

Not idempotent:
  balance += 250                → adds 250 each time
  INSERT INTO orders ...        → inserts a new row each time
  POST /payments { amount: 100 }
```

### Idempotency Keys

For non-idempotent operations (payments, emails, order creation), use an **idempotency key** — a unique ID the sender generates and sends with the request. The receiver tracks seen keys and returns the cached result for duplicates without reprocessing.

```
First request:
  POST /payments
  Idempotency-Key: a1b2c3d4-...
  { amount: 100, card: "..." }
  → charge card → return { payment_id: "pay_xyz", status: "success" }
  → store: a1b2c3d4 → { payment_id: "pay_xyz", status: "success" }

Duplicate request (retry):
  POST /payments
  Idempotency-Key: a1b2c3d4-...   ← same key
  { amount: 100, card: "..." }
  → lookup key → return cached { payment_id: "pay_xyz", status: "success" }
  → card NOT charged again ✓
```

The key must be:
- Generated by the **sender** (not the receiver) — so it's stable across retries
- Unique per logical operation — not reused across different payments
- Stored with a TTL on the receiver — keys don't need to live forever

---

## Delivery vs Processing Semantics

These are often conflated but are distinct.

```
Delivery semantic   — did the message reach the receiver's inbox?
Processing semantic — did the receiver's application logic execute once?
```

A message can be delivered exactly once to a queue but processed multiple times if the consumer crashes after processing but before acknowledging. A message can be delivered multiple times but processed exactly once if the consumer deduplicates.

```
Kafka example:
  Message delivered to partition → at-least-once (Kafka guarantees this)
  Consumer reads message
  Consumer processes (writes to DB)
  Consumer crashes before committing offset
  Consumer restarts → reads same message again
  → message delivered once, processed twice

Fix: make the DB write idempotent, or use Kafka transactions to atomically
     write to DB and commit offset.
```

> [!note] What "exactly-once" in Kafka actually means
> Kafka's exactly-once semantics (EOS) guarantees that a message produced by a transactional producer is written to the topic **exactly once**, and that consumer offset commits and downstream writes happen **atomically**. It does not protect you from processing the message twice in your application code if you don't use the transactional API correctly.

---

## Comparison Across Systems

| System | Default | Max guarantee | Mechanism |
|---|---|---|---|
| MQTT QoS 0 | At-most-once | At-most-once | Fire and forget |
| MQTT QoS 1 | At-least-once | At-least-once | PUBACK + retry |
| MQTT QoS 2 | Exactly-once | Exactly-once | 4-way handshake |
| Kafka (default) | At-least-once | Exactly-once (EOS) | Idempotent producer + transactions |
| SQS Standard | At-least-once | At-least-once | Consumer deduplication |
| SQS FIFO | Exactly-once | Exactly-once | Deduplication ID (5-min window) |
| HTTP (no retry) | At-most-once | At-most-once | — |
| HTTP (with retry + idempotency key) | At-least-once | Exactly-once (processing) | Idempotency key |
| gRPC (unary) | At-most-once | At-most-once (transport) | Client handles retry |

---

## Choosing the Right Semantic

```
Is message loss acceptable?
  Yes → at-most-once
  No  →
        Is duplicate processing acceptable?
          Yes → at-least-once (cheapest reliable option)
          No  →
                Can you make the operation idempotent?
                  Yes → at-least-once + idempotent receiver (recommended)
                  No  → exactly-once (expensive — use sparingly)
```

**Make the operation idempotent** is almost always the right answer. Exactly-once infrastructure (2PC, distributed transactions) is complex, slow, and fragile. Idempotent design is simpler and more resilient.

---

## Key Takeaways

- **At-most-once** — fast, lossy. For high-volume, loss-tolerant data (metrics, telemetry)
- **At-least-once** — reliable, may duplicate. The practical default for most systems
- **Exactly-once** — no loss, no duplicate. Expensive; usually approximated via idempotency
- Exactly-once is impossible to guarantee purely at the transport layer — the receiver must participate
- **Delivery semantic ≠ processing semantic** — a message delivered once can still be processed twice
- **Idempotency keys** are the standard pattern for non-idempotent operations over at-least-once transports
- Design operations to be naturally idempotent wherever possible — it eliminates an entire class of bugs

---

## Related Notes
- [[MQTT]] — QoS 0/1/2 as concrete implementations of these semantics
- [[RPC and APIs]] — RPC failure modes and why at-least-once + idempotency is the default recommendation
- [[API Design — Client Libraries, IDLs, and SLOs]] — idempotency keys in HTTP API design

