# PostgreSQL Internals

#postgres #databases #backend

---

## Lockstep Replication

Lockstep is a replication pattern where the primary cannot commit a transaction until all (or a quorum of) replicas have acknowledged they received and written the data.

```
Client → Primary
           │
           ├──► Replica 1 ──► ACK ─┐
           ├──► Replica 2 ──► ACK ─┼──► Primary commits ──► Client gets success
           └──► Replica 3 ──► ACK ─┘
```

### Properties
- **Zero replication lag** — replicas are always current when the commit returns
- **No data loss on failover** — any replica can become primary immediately
- **Higher write latency** — every write pays the network round-trip to all replicas

### vs Async Replication
In async replication, the primary commits immediately and ships the log to replicas in the background. Faster writes, but replicas lag — a primary crash can lose committed data.

### Where you see it
- PostgreSQL: `synchronous_commit = on`
- etcd, CockroachDB, Spanner (Raft/Paxos-based)
- MySQL Group Replication (single-primary mode)

> [!tip] The tradeoff
> Lockstep trades **write throughput and latency** for **durability and consistency**. Right for financial systems; too expensive for high-throughput general workloads.

Most production systems use a hybrid — one synchronous replica for durability, additional async replicas for read scaling.

---

## Circular Buffers in PostgreSQL

PostgreSQL uses circular buffers in two distinct places.

### WAL Buffer

The Write-Ahead Log buffer (`wal_buffers`, default 4MB) is a **circular buffer in shared memory** where WAL records are staged before being flushed to disk.

```
[ record ][ record ][ record ][ record ][ record ]
     ^                                       ^
   oldest (flush point)               newest (insert point)
```

WAL writers append to the head; the WAL writer process flushes from the tail to disk. When the buffer wraps around, it overwrites old space — but only after confirming those segments have been flushed.

**Why circular?** Pre-allocated once at startup — avoids allocation overhead and fragmentation.

### Ring Buffers for Bulk Operations

PostgreSQL uses a small ring buffer strategy inside the shared buffer pool for:
- Sequential scans on large tables
- VACUUM
- COPY / bulk inserts

Instead of evicting useful cached pages to buffer a full table scan, it caps the scan to a small ring (256KB by default). Once the ring wraps, it reuses those same buffers.

> [!note] Practical implication
> A large `SELECT *` on a big table won't warm the cache for a second run — PostgreSQL intentionally didn't cache it all to protect the hot working set.

---

## VACUUM (Garbage Collection)

VACUUM is PostgreSQL's garbage collector. It cleans up **dead tuples** — old row versions left behind by [[#MVCC]] that no active transaction can see anymore.

```
Before VACUUM:
[ live tuple ][ dead tuple ][ live tuple ][ dead tuple ][ live tuple ]

After VACUUM:
[ live tuple ][    free    ][ live tuple ][    free    ][ live tuple ]
```

### Key difference from language GC
VACUUM reclaims space **within the data file on disk** and marks it reusable — it doesn't shrink the file or return memory to the OS. `VACUUM FULL` does compact the file, but locks the table exclusively and is rarely run in production.

### Autovacuum
PostgreSQL runs autovacuum in the background automatically. It kicks in based on how many dead tuples have accumulated (configurable thresholds). You generally don't call VACUUM manually.

### Two jobs VACUUM does
1. **Cleans dead tuples** — reclaims space from dead row versions
2. **Freezes old XIDs** — prevents [[#XID Wraparound]]

The second job is more critical. Table bloat is annoying; XID wraparound is catastrophic.

---

## MVCC (Multi-Version Concurrency Control)

### The problem it solves
Without MVCC, reads and writes would need to lock the same rows, serializing all access. MVCC eliminates that by keeping multiple versions of rows so **readers and writers never block each other**.

### The mechanism

Every row has two hidden system columns:
- `xmin` — the transaction ID (XID) that **created** this row version
- `xmax` — the transaction ID that **deleted/updated** this row version (0 if still live)

```
Row versions in the heap:
┌─────────────────────────────────────┐
│ xmin=100, xmax=0,   name="Alice"   │  ← live
├─────────────────────────────────────┤
│ xmin=50,  xmax=102, name="Bob"     │  ← dead (deleted by txn 102)
└─────────────────────────────────────┘
```

When your transaction starts, it gets a **snapshot** of which XIDs were committed at that moment. Visibility rules:
- Row is visible if `xmin` is committed in your past **and** `xmax` is 0 or in your future

### What past and future mean

Not clock time — **transaction ordering**. Your snapshot records which XIDs committed before it was taken.

```
Past  = committed before my snapshot → I can see their rows
Future = started after my snapshot   → I cannot see their rows
```

#### Example
```
XID 100: INSERT name="Alice"  → committed ✓
XID 101: INSERT name="Bob"    → committed ✓
XID 102: [YOU start here]     → snapshot taken
XID 103: INSERT name="Carol"  → commits after you started
```

Carol is in your future. She doesn't exist from your perspective, even if she commits while you're still running.

### How this gives repeatable reads

Your snapshot is taken **once** at transaction start and used for every query. No matter what commits happen while you're running, your view of the database is frozen.

```
T=0: You start → snapshot = {committed: 100, 101, 102}
T=1: XID 103 inserts a row and commits
T=2: XID 104 updates a row and commits
T=4: You run SELECT → still using snapshot from T=0
     → 103, 104 are "future" → their changes invisible
```

Without this, two queries in the same transaction could see different states — a **non-repeatable read**.

### Isolation levels and snapshots

| Isolation Level | When snapshot is taken |
|---|---|
| READ COMMITTED | Per query — you see commits mid-transaction |
| REPEATABLE READ | Once at transaction start — full MVCC guarantee |
| SERIALIZABLE | Once + conflict tracking — prevents all anomalies |

Most apps run READ COMMITTED by default. You don't get repeatable reads unless you explicitly set `REPEATABLE READ`.

### The cost

Dead tuples accumulate. Every old row version stays on disk until no active snapshot needs it. A long-running transaction holds back the snapshot horizon, preventing VACUUM from cleaning anything newer — causing **table bloat**.

---

## XID Wraparound

### The setup

PostgreSQL assigns every transaction a **32-bit integer ID (XID)**. 32 bits = ~4 billion transaction IDs total.

Visibility is determined by comparing XIDs using **modular arithmetic** — the XID space is a circle. Half the circle (~2 billion) is your past, half is your future.

```
         2 billion "past" XIDs    |    2 billion "future" XIDs
                      [YOU are here]
```

### What goes wrong

After ~4 billion transactions, the XID counter resets toward 0. Old rows with low XIDs suddenly appear in your **future** half of the circle.

```
XID space wraps:
... 4,294,967,294 → 4,294,967,295 → 0 → 1 → 2 ...

A row written at XID 5 now looks NEWER than your transaction at XID 3,000,000,000
→ that row becomes invisible → data appears to vanish
```

### How VACUUM prevents it: Freezing

VACUUM marks old rows with a special **frozen** flag (`xmin = FrozenTransactionId = 2`). Frozen rows are always visible to everyone — they're outside the XID comparison system entirely.

```
Before freeze:  row has xmin=500  (vulnerable to wraparound)
After freeze:   row has xmin=2    (permanently visible, immune)
```

### Key settings

| Setting | Default | Purpose |
|---|---|---|
| `vacuum_freeze_min_age` | 50M txns | How old a row must be before VACUUM freezes it |
| `autovacuum_freeze_max_age` | 200M txns | Forces autovacuum before wraparound risk |

### When it goes catastrophic

If autovacuum falls behind, PostgreSQL will warn you:

```
WARNING: database "mydb" must be vacuumed within 177009986 transactions
```

Ignore it long enough and **PostgreSQL shuts down and refuses all connections** to protect data integrity. Recovery requires running VACUUM in single-user mode.

---

## Summary Table

| Concept | What it solves | What it leaves behind |
|---|---|---|
| MVCC | Readers don't block writers | Dead row versions on disk |
| VACUUM | Cleans dead tuples + freezes old XIDs | Prevents bloat + XID wraparound |
| XID Wraparound | N/A — it's the problem VACUUM prevents | Data loss / DB shutdown if ignored |
| Lockstep | Zero data loss on failover | Higher write latency |
| Circular Buffer | Efficient WAL staging + scan isolation | N/A |

---

## Interview Cheat Sheet: MVCC

**Lead with the problem:**
> "Without MVCC, reads and writes would need to lock the same rows, serializing all access. MVCC eliminates that by keeping multiple versions of rows so readers and writers never block each other."

**Explain the mechanism:**
> "Every row has two hidden fields — xmin and xmax — recording which transaction created and deleted it. Your transaction gets a snapshot at start time and uses it to decide which versions are visible."

**Name the consequence:**
> "The tradeoff is dead row versions accumulating on disk. Old versions can't be removed until no active snapshot needs them — that's what VACUUM cleans up."

**Know the isolation angle:**
> "READ COMMITTED takes a fresh snapshot per query, so you can see commits mid-transaction. REPEATABLE READ locks the snapshot at transaction start — that's the full MVCC guarantee."
