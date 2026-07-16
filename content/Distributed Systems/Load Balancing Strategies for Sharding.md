# Load Balancing Strategies for Sharding

#distributed-systems #networking #load-balancing #sharding #databases #envoy

---

## The Question Behind the Question

"Which load balancer is used for sharding?" doesn't have a single answer — because **sharding-aware routing is a different problem than generic request load balancing**, and the two are frequently confused.

Generic LB goal: spread N requests evenly across M interchangeable backends. Any backend can serve any request.

Sharding LB goal: route each request to the **one specific backend that owns that piece of data**. Backends are *not* interchangeable — sending a request to the wrong shard doesn't just cost you an even distribution, it returns wrong or missing data. "Load balancing" here is really **consistent routing**, and even distribution is a secondary concern (avoiding hot shards) achieved through the choice of hash function, not through the LB picking whichever backend is least busy.

This distinction is why the answer is "it depends on where the routing decision is made and what layer it lives at," not a specific product name.

---

## Where the Routing Decision Lives — Three Architectures

### 1. Client-Side Sharding (no LB in the traditional sense)

The client library computes the shard directly and opens a connection to the owning node. There is no intermediary in the data path at all.

```
Client
  │
  ├─ hash(key) → node ring position → backend node
  │
  └─ connect directly to that node
```

**Examples:** Redis Cluster (client uses `CRC16(key) mod 16384` to find the slot, then a client-side slot→node map, updated via `MOVED`/`ASK` redirects), Cassandra drivers (token-aware routing using the cluster's partition token ranges), DynamoDB SDK (partition key hash resolved server-side but connection routing is transparent via a single endpoint + internal request router).

**Why teams choose this:** zero extra network hop, no proxy to scale or become a bottleneck. Cost: routing logic (and the topology-change handling) is duplicated in every client/language, and clients must handle mid-flight topology changes (`MOVED` redirects, retries).

### 2. Protocol-Aware L7 Proxy (the "sharding load balancer" people usually mean)

A proxy sits in the data path, **speaks the backend's wire protocol**, parses the shard key out of the request, and forwards to the owning shard.

```
Client → Proxy (understands MySQL/Redis/Mongo wire protocol)
              │
        parse shard key from query/command
              │
        consult routing table (ranges or hash ring)
              │
        forward to owning shard
```

**Examples:**
- **Vitess VTGate** — parses SQL, extracts the sharding column from the `WHERE` clause, routes to the correct MySQL shard. Can also scatter-gather across shards for cross-shard queries.
- **MongoDB `mongos`** — routes based on shard key ranges tracked by config servers; scatter-gathers when the query doesn't include the shard key.
- **Twemproxy / `nutcracker`** — sits in front of Memcached/Redis, hashes the key (ketama consistent hashing by default), forwards to the owning cache node. No cross-shard query support — deliberately dumb and fast.
- **Citus coordinator** (Postgres extension) — routes by distribution column, can push down or scatter-gather queries across worker shards.

This is architecturally an **L7 load balancer**, but not a generic HTTP one — it's protocol-specific middleware that happens to do load-balancing-shaped work. This is the closest match to "a load balancer for sharding" as a distinct product category.

### 3. Generic L7 HTTP/gRPC LB with Hash-Based Routing

When you're sharding *services* (not a database) — e.g., a stateful service where each shard owns a partition of users/tenants/sessions — a general-purpose L7 proxy like **Envoy** or **NGINX** can do the routing if the shard key is exposed in something HTTP-visible (header, cookie, path segment, JWT claim).

```yaml
# Envoy example — ring_hash LB policy keyed on a header
lb_policy: RING_HASH
ring_hash_lb_config:
  hash_function: XX_HASH
route:
  hash_policy:
    - header:
        header_name: "x-user-id"
```

Envoy supports two LB policies purpose-built for this: `RING_HASH` (classic consistent-hashing ring with virtual nodes) and `MAGLEV` (Google's algorithm — faster O(1) lookup table, more even distribution, used internally at Google and inside GCP's network LB). Both guarantee the same key routes to the same backend and minimize remapping when backends are added/removed.

---

## Why L4 Is (Almost) Never the Answer

This is the detail that trips people up: **an L4 load balancer structurally cannot do shard-aware routing** in the general case, because it only sees the 5-tuple (src/dst IP, src/dst port, protocol) — it has no visibility into the request body, headers, or query where the shard key actually lives.

```
L4 LB sees:              Shard key usually lives in:
  src IP, src port         → HTTP header / path / body
  dst IP, dst port         → SQL WHERE clause
  protocol                 → Redis command argument
                            → gRPC message field

L4 LB CANNOT read any of the right column.
```

The one exception: if the shard key *happens to correlate* with client source IP (rare, and fragile — NATs and shared egress IPs break this immediately), an L4 5-tuple hash could look consistent. This is not a real design pattern; if you find yourself relying on it, you have a bug waiting to happen the moment a client's IP or port changes mid-session.

**Conclusion:** sharding requires either L7 (to read the key out of the application-layer request) or client-side logic that never goes through an LB at all. This is the single most important fact to know at a systems-design-interview / SDE3 level: **"can an L4 LB shard my traffic?" → no, not correctly, ever, except by accident.**

---

## The Hashing Algorithms Underneath

The proxy/client architecture above is the "where." The hash function is the "how requests map to shards evenly and stably." See [[L4 and L7 Load Balancers]] for the general LB algorithm catalogue — this section goes deeper on the sharding-specific tradeoffs.

### Consistent Hashing (Ring)

```
Ring (hash space 0 to 2^32-1):

        Node A (hash: 500)
       ╱                    ╲
Node D (hash: 3000)      Node B (hash: 1200)
       ╲                    ╱
        Node C (hash: 2100)

key "user:42" → hash → 1800 → walk clockwise → lands on Node C
```

Adding/removing a node only remaps the keys between it and its counter-clockwise neighbor — not the whole keyspace. This bounds reshuffle cost to roughly `1/N` of keys on a topology change, vs. `mod N` hashing where nearly *all* keys remap.

**Virtual nodes:** with few physical nodes, plain ring hashing produces uneven arcs (one node might own 60% of the ring by chance). Fix: give each physical node many points on the ring (e.g., 100–256 virtual nodes each). Cassandra, Envoy's ring_hash, and ketama (Twemproxy) all do this.

**Hot shard problem:** even with virtual nodes, a skewed key distribution (a celebrity user, a viral object) can overload the shard that owns that specific key, regardless of how even the *hash* distribution is. Hashing solves *node* balance, not *access-pattern* balance. Mitigations: split hot keys further (salt the key, e.g. `user:42:shard:{0-9}` and fan out reads), cache in front of the shard, or move to a dedicated partition for known-hot keys.

### Rendezvous Hashing (HRW)

No ring — for each request, compute `score = hash(key, node_id)` for every node, pick the max. Same minimal-disruption property as consistent hashing, no ring data structure to maintain, easier to reason about with a small/dynamic node set. Used in some CDN request-routing layers and as an alternative to ring_hash inside Envoy configs.

### Maglev Hashing

Builds a large (typically 65537-slot, prime-sized) lookup table once per topology change; runtime lookup is O(1) array index, not O(log N) ring walk. Optimizes for the specific goal of *minimum disruption* on backend changes while keeping per-packet lookup extremely fast — designed for Google's software network load balancer operating at line rate. This is why it's the L4/L7 LB policy of choice when you need both consistent hashing *and* very high throughput.

### Range-Based Sharding (no hashing at all)

Instead of hashing, explicitly assign contiguous key ranges to shards, tracked by a metadata/config service:

```
Shard A: user_id [0, 1_000_000)
Shard B: user_id [1_000_000, 2_500_000)
Shard C: user_id [2_500_000, ∞)
```

**Examples:** MongoDB (chunks tracked by config servers, auto-splits and migrates on growth), HBase/Bigtable (region splits), CockroachDB (range splits at ~512MB by default).

Tradeoff vs. hashing: range sharding supports efficient range scans (`WHERE user_id BETWEEN x AND y`) and lets you rebalance by moving contiguous chunks — but is prone to hotspotting on monotonically-increasing keys (e.g., timestamp or auto-increment primary keys pile all new writes onto the last shard). Hash-based sharding avoids that hotspot but kills range scans (adjacent keys land on unrelated shards).

---

## Comparison Table

| | Consistent Hash Ring | Rendezvous (HRW) | Maglev | Range-Based |
|---|---|---|---|---|
| Data structure | Ring + virtual nodes | None (compute per request) | Precomputed lookup table | Ordered range map |
| Lookup cost | O(log N) | O(N) per request | O(1) | O(log N), binary search |
| Rebalance disruption | ~1/N of keys | ~1/N of keys | Minimal, table-driven | Only the split/moved range |
| Range scans | No | No | No | Yes |
| Hotspot risk | Low (with vnodes) | Low | Low | High on monotonic keys |
| Used by | Cassandra, ketama, Envoy `ring_hash` | Some CDNs, Envoy alt config | Google NLB, Envoy `maglev` | MongoDB, CockroachDB, HBase |

---

## Rebalancing / Resharding in Practice

Whatever the hash strategy, adding capacity means physically moving data, not just remapping a hash function on paper:

```
1. New node/shard joins → gets assigned a range (hash range or key range)
2. Data belonging to that range must be copied from old owner(s) to new owner
3. During migration: routing layer must know both old and new location
   (dual-write, or a "migrating" state with fallback lookups)
4. Once copy completes and is verified → cut over routing, remove old copy
```

This is why real systems separate the **hashing/routing scheme** (which is nearly static) from the **membership/topology state** (which changes live): Cassandra gossip protocol, MongoDB config servers, Vitess topology service (etcd/ZooKeeper-backed), Kafka controller + ISR. The load balancer or client library is only as correct as the topology state it's reading from — stale topology is the most common source of "requests routed to the wrong shard" bugs in production.

---

## Key Takeaways

- "Load balancer for sharding" is really **consistent, key-aware routing**, not load spreading — the two goals only align when the hash function also happens to be uniform.
- **L4 cannot do sharding-aware routing correctly** — it has no visibility into the shard key, which lives in application-layer content. This is the sharpest interview-level distinction.
- Three real architectures: **client-side hashing** (Redis Cluster, Cassandra drivers — no proxy hop), **protocol-aware L7 proxy** (Vitess VTGate, mongos, Twemproxy — understands the wire protocol and shard key), **generic L7 HTTP/gRPC LB with hash routing** (Envoy `ring_hash`/`maglev` keyed on a header/cookie — for sharding services, not databases).
- **Consistent hashing + virtual nodes** is the default algorithm; **Maglev** trades ring simplicity for O(1) lookup and better disruption minimization at very high throughput (Google NLB, Envoy).
- **Hashing balances nodes, not access patterns** — hot shards from skewed key popularity are a separate problem requiring key salting, caching, or dedicated partitions.
- **Range-based sharding** trades hotspot risk (on monotonic keys) for range-scan support and simpler rebalancing (move a contiguous chunk vs. remap a hash).
- Rebalancing is a two-part problem: the hash/range scheme decides *where a key should live*; a separate topology/membership layer (gossip, config servers, etcd) tracks *where it actually lives right now* during migration.

---

## Related Notes
- [[L4 and L7 Load Balancers]] — general OSI-layer LB mechanics (DNAT, DSR, health checks, connection pooling) that this note builds on for the sharding-specific case
- [[Sidecar Pattern]] — Envoy sidecar as the place `ring_hash`/`maglev` policies are typically configured in a service mesh
- [[Database Internals/PostgreSQL Internals]] — single-node internals context for when sharding (Citus) becomes necessary
