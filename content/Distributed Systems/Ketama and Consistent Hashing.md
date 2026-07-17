# Ketama and Consistent Hashing

#distributed-systems #redis #caching #hashing #sharding

---

## The Problem — Why Simple Hashing Breaks

You have 3 Redis servers and 1 million keys. The naive approach to distributing keys:

```
server = hash(key) % 3
```

This works fine until you add a 4th server. Now the formula is `hash(key) % 4`. Hash values don't change, but almost every result of `% 4` is different from `% 3`. Let's see why:

```
key = "user:1234"
hash("user:1234") = 9823746

With 3 servers:  9823746 % 3 = 0  → Server A
With 4 servers:  9823746 % 4 = 2  → Server C   ← DIFFERENT
```

When you go from N to N+1 servers, roughly `(N-1)/N` of all keys remap to a different server. With 3 servers, **67% of your cache becomes invalid overnight**. Every remapped key is a cache miss, which hits your database. This is a **cache stampede** — the database gets hammered by millions of simultaneous misses.

The same thing happens when a server crashes: you drop from N to N-1, remapping ~(N-1)/N keys again.

> [!important] The Core Requirement
> When the number of servers changes, only the **minimum number of keys necessary** should move. If you add 1 server to a pool of 10, only ~10% of keys should remap — the ones that now belong to the new server. The other 90% should stay put.

---

## Consistent Hashing — The Ring Idea

Consistent hashing solves this by thinking about hash values as positions on a **ring** (a circle from 0 to 2³²−1, about 4 billion positions).

**Step 1:** Place servers on the ring by hashing their identifiers:

```
hash("redis-a:6379") = 400M   → position 400,000,000
hash("redis-b:6379") = 1.2B   → position 1,200,000,000
hash("redis-c:6379") = 3.1B   → position 3,100,000,000
```

```
                    0 / 4B
                      │
              ┌───────┴───────┐
         3.1B │               │ 400M
          [C] │               │ [A]
              │               │
         1.2B └───────────────┘
              [B]

(simplified — positions distributed around a circle)
```

**Step 2:** For each key, hash it to get its ring position, then walk **clockwise** until you hit a server:

```
hash("user:1234") = 600M  → walk clockwise → hits B at 1.2B
hash("session:X") = 50M   → walk clockwise → hits A at 400M
hash("order:999") = 2.5B  → walk clockwise → hits C at 3.1B
hash("cart:42")   = 3.5B  → walk clockwise → wraps around → hits A at 400M
```

**Step 3 — What happens when you add a server D at position 2.0B:**

```
Before:  keys in range (1.2B → 3.1B) → Server C
After:   keys in range (1.2B → 2.0B) → Server D  ← moved
         keys in range (2.0B → 3.1B) → Server C   ← stayed
```

Only the keys that fall between B (1.2B) and D's new position (2.0B) move. Everything else is untouched. With 4 servers, you'd expect ~25% of keys to move — and that's exactly what happens.

> [!note] Why This Is Minimal Movement
> Any key that maps to a server X on the ring stays on X after you add a new server, unless the new server lands between that key's position and X. Only the new server's "predecessor slice" is affected.

---

## The Problem With Basic Consistent Hashing

Basic consistent hashing has a flaw: **uneven distribution**.

If you hash 3 server names to random positions on a 4-billion-point ring, you'll split the ring into 3 arcs. Those arcs won't be equal — by pure chance, one server might own 60% of the ring and another only 15%.

```
         0
         │
    ──A──┤──────────────────B──
                              │
              (C is squeezed  │
               into a tiny    C
               arc here)      │
    ─────────────────────────┘
```

This means one server handles 60% of all requests. Not a sharding strategy — a hotspot strategy.

---

## Ketama — The Fix

**Ketama** (developed at Last.fm, widely adopted for Memcached and Redis client-side sharding) solves uneven distribution with one idea: **virtual nodes**.

Instead of placing each server once on the ring, place each server **160 times**. Each real server gets 160 fake "copies" scattered across the ring. The ring now has 160×N points instead of N, and they distribute much more evenly by the law of large numbers.

```
With 3 servers × 160 virtual nodes = 480 points on the ring

     A-12  B-7   A-89   C-3   B-44  A-131  C-78 ...  (480 points total)
──┬──┬─────┬──────┬──────┬─────┬──────┬─────┬──────→ ring
  0                                                    4B
```

Each arc between adjacent points is tiny. On average, each server owns 1/3 of the ring, and the variance is low because 160 samples per server is enough to smooth out the randomness.

---

## How Ketama Actually Computes the 160 Points

This is the concrete algorithm (the part that makes Ketama *Ketama*, not just "generic consistent hashing"):

**For each server** (`"redis-a:6379"`):

1. Generate 40 strings: `"redis-a:6379-0"`, `"redis-a:6379-1"`, ..., `"redis-a:6379-39"`
2. MD5-hash each string → produces a 16-byte (128-bit) digest
3. Split each 16-byte digest into **4 chunks of 4 bytes** each
4. Interpret each 4-byte chunk as a 32-bit unsigned integer → a ring position
5. 40 MD5s × 4 positions each = **160 ring positions per server**

```python
import hashlib, struct

def ketama_points(server: str) -> list[int]:
    points = []
    for i in range(40):
        digest = hashlib.md5(f"{server}-{i}".encode()).digest()
        for j in range(4):
            # little-endian 32-bit int from 4 bytes
            point = struct.unpack_from("<I", digest, j * 4)[0]
            points.append(point)
    return points  # 160 positions
```

**For each key** (e.g., `"user:1234"`):

1. MD5-hash the key → 16-byte digest
2. Take the first 4 bytes as a little-endian 32-bit int → ring position
3. Binary search the sorted list of 160×N server points to find the first point ≥ key's position
4. The server that owns that point is the answer

```python
import bisect

def get_server(key: str, ring: list[int], ring_map: dict[int, str]) -> str:
    key_hash = struct.unpack_from("<I", hashlib.md5(key.encode()).digest())[0]
    idx = bisect.bisect(ring, key_hash) % len(ring)  # wrap around
    return ring_map[ring[idx]]
```

> [!note] Why MD5?
> MD5 produces a uniformly distributed output — similar keys hash to very different positions. It's not used for security here (it's cryptographically broken, but we don't care about that). It's used because it distributes keys evenly. Murmur3 is sometimes used instead for speed, but MD5 is the Ketama standard.

---

## The Numbers — How Even Is the Distribution?

With 3 servers and 160 virtual nodes each (480 total ring points):

| Configuration | Std deviation from ideal | Worst server load |
|---|---|---|
| 1 vnode/server | ~50% | Can own 3× its share |
| 10 vnodes/server | ~15% | Can own 1.5× its share |
| 100 vnodes/server | ~5% | Close to ideal |
| 160 vnodes/server (Ketama) | ~4% | Very close to 1/N |
| 500 vnodes/server | ~2% | Near-perfect |

160 vnodes is the empirical sweet spot: good enough distribution without the memory overhead of thousands of ring points.

---

## Ketama vs Redis Cluster Hash Slots

This is a common point of confusion.

**Redis Cluster** (the built-in clustering) does **not** use Ketama. It uses a completely different scheme called **hash slots**:

```
hash_slot = CRC16(key) % 16384
```

There are 16,384 slots. Each node owns a range of slots (e.g., node A owns slots 0–5460, node B owns 5461–10922, node C owns 10923–16383). The cluster tracks which node owns which slot range in a cluster state table gossiped between nodes.

**Ketama** is used for **client-side sharding** — when you have multiple independent Redis instances (not a cluster) and want the client to distribute keys among them. No Redis-side coordination needed — every client independently computes the same ring and routes to the same server.

| | Ketama (client-side) | Redis Cluster (hash slots) |
|---|---|---|
| Where logic lives | Client library | Redis nodes |
| Server awareness | Servers don't know about each other | Full cluster coordination |
| Rebalancing | Manual (move keys) | Cluster handles resharding |
| Adding a node | Update client config | `CLUSTER MEET` + reshard |
| Failover | Manual or via proxy | Automatic (replica promotion) |
| Use case | Small setups, no Redis Cluster | Production HA clusters |

---

## Where Ketama Is Used in Practice

**Twemproxy (nutcracker)** — Twitter's Redis/Memcached proxy. Sits between your app and Redis servers, implements Ketama internally. Your app talks to one Twemproxy endpoint; Twemproxy routes to the right Redis shard.

```
App → Twemproxy :6379 → Redis-A :6380
                      → Redis-B :6381
                      → Redis-C :6382
      (Ketama decides which one)
```

**Redis client libraries:**

```python
# redis-py with consistent hashing
from redis import RedisCluster
# or for independent instances:
from rediscluster import RedisCluster

# ioredis (Node.js) — built-in Ketama support
const Redis = require('ioredis')
const cluster = new Redis.Cluster(nodes, {
  scaleReads: 'slave',
  natMap: { ... }
})

# Jedis (Java) — ShardedJedis uses Ketama
ShardedJedis jedis = new ShardedJedis(shards, Hashing.MURMUR_HASH)
```

**AWS ElastiCache** — when you configure a Memcached cluster with auto-discovery, the ElastiCache client uses Ketama to distribute keys.

---

## What Happens When a Node Goes Down

Say Redis-B fails. Its 160 virtual nodes are removed from the ring. Every key that was mapped to one of B's virtual nodes now maps to the **next clockwise point** — which belongs to either A or C.

```
Before: key at position 1.1B → B-virtual-node at 1.2B → Redis-B
After:  key at position 1.1B → next point is C-virtual-node at 1.35B → Redis-C
```

The 50% of keys that belonged to B are now split between A and C (approximately evenly, thanks to the virtual nodes). A and C each absorb ~25% more load. The other 50% of keys (on A and C originally) are completely unaffected.

Compare to modulo hashing: if B fails and you go from 3 to 2 servers with `% 2`, all keys remap.

---

## Implementation Gotcha — Key Replication Tags

Redis Cluster has `{hashtag}` to force related keys to the same slot:

```
MSET {user:1234}:profile "..." {user:1234}:settings "..."
```

Ketama has the same concept — some client libraries extract a substring between `{` and `}` and hash only that part. This lets you guarantee `{user:1234}:profile` and `{user:1234}:settings` land on the same server, enabling `MGET`/`MSET`/transactions across those keys.

Without this, a `MGET user:1234:profile user:1234:settings` would fail if the keys land on different servers (Ketama doesn't coordinate cross-server operations).

---

## Key Takeaways

- `hash(key) % N` remaps ~(N-1)/N of all keys when N changes → cache stampede
- Consistent hashing places servers and keys on a ring; only the minimum necessary keys move when servers change
- Basic consistent hashing has uneven distribution because 3 points don't divide a circle evenly
- **Ketama** fixes this with **160 virtual nodes per server** using MD5 hashing → even distribution
- Ketama lives in the **client** — no server-side coordination, every client independently computes the same routing
- **Redis Cluster uses CRC16 + 16384 hash slots, not Ketama** — Ketama is for client-side sharding of independent Redis instances
- Real tools using Ketama: Twemproxy, Jedis `ShardedJedis`, older `redis-py` ring mode, ElastiCache clients

---

## Related Notes
- [[Distributed Systems/Ambassador Pattern]] — Twemproxy is an ambassador pattern implementation with Ketama inside
- [[Distributed Systems/L4 and L7 Load Balancers]] — ring hash and Maglev hashing in load balancers are the same consistent hashing idea applied to backend pools
- [[Distributed Systems/Delivery Semantics]] — cache misses during resharding cause at-least-once load on the database; idempotent DB reads make this safe
