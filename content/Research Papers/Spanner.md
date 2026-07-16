# Spanner: Google's Globally-Distributed Database

> **Papers:** "Spanner: Google's Globally-Distributed Database" — Corbett, Dean et al., OSDI 2012 · "Spanner: Becoming a SQL System" — Bacon, Bales et al., SIGMOD 2017
> **Related:** [[Consensus Algorithms]] · [[CAP Theorem]] · [[Consistency Models]]

---

## What Spanner Is

Spanner is Google's globally-distributed, synchronously-replicated, externally-consistent relational database. At the time of the 2012 paper it was the **first system to provide externally-consistent distributed transactions at planetary scale**.

Before Spanner, the landscape at Google looked like this:

| System | What it did | Why it fell short |
|---|---|---|
| **Bigtable** | Schemaless KV store, single-datacenter strong consistency | No cross-row transactions, only eventual consistency across datacenters |
| **Megastore** | Semi-relational, synchronous replication across DCs | Poor write throughput (multiple replicas could initiate writes — all conflict in Paxos) |
| **Sharded MySQL (F1)** | Traditional SQL per shard | Manual resharding took 2+ years, cross-shard transactions impossible |

Spanner was built to replace all three. It combines:
- The scalability and fault-tolerance of a distributed KV store
- The schema, transactions, and SQL of a relational database
- The global replication and strong consistency that neither could provide together

By 2017, it was backing hundreds of mission-critical Google services, serving tens of millions of QPS, managing hundreds of petabytes, replicated across datacenters worldwide.

---

## The Core Guarantees

Three guarantees define Spanner and distinguish it from everything that came before:

**1. External consistency (= linearizability across the globe)**
If transaction T1 commits before T2 starts, T1's commit timestamp is strictly smaller than T2's. The system behaves as a single global database regardless of how many machines or datacenters are involved.

**2. Globally-consistent reads at a timestamp**
You can issue a read at any past timestamp `t` and get a snapshot of the database that reflects exactly the transactions committed as of `t` — across all shards, all datacenters — without taking any locks.

**3. Lock-free read-only transactions**
Read-only transactions that span multiple shards can execute without acquiring any locks, without blocking writers, and without any two-phase coordination.

These are not incremental improvements. They required a fundamentally new approach to time in distributed systems.

---

## Architecture Overview

### The physical hierarchy

```
Universe (one global deployment)
  └── Zones (unit of administrative deployment, ~= one datacenter)
        ├── Zonemaster (assigns data to spanservers)
        ├── Location proxies (help clients find spanservers)
        └── Spanservers (100–1000 per zone, each holds 100–1000 tablets)

Global singletons:
  Universe master — status console for debugging
  Placement driver — moves data across zones for load balance / replication constraints
```

A zone maps roughly to a Bigtable deployment. Zones can be added or removed from a live system as datacenters come online or are decommissioned.

### Spanserver software stack

Each spanserver is responsible for 100–1000 **tablets**. A tablet is a key-value mapping:

```
(key: string, timestamp: int64) → string
```

The timestamp dimension is what makes Spanner a **multi-version database** rather than a KV store. Every value is versioned by its commit timestamp. Old versions are subject to configurable garbage collection but can be read by snapshot queries.

For each tablet, the spanserver runs a **single Paxos state machine**. This is the replication layer — writes go through Paxos at the leader replica, reads can go to any sufficiently-up-to-date replica.

On top of Paxos, every leader-replica spanserver runs two additional components:

- **Lock table** — implements two-phase locking for read-write transactions; maps key ranges to lock states
- **Transaction manager** — coordinates two-phase commit for transactions that span multiple Paxos groups

All data is stored on **Colossus** (Google's distributed filesystem, successor to GFS), in a write-ahead log + B-tree file format (later replaced by Ressi — see the SQL paper section).

```
Spanserver (leader replica)
  ├── Lock table
  ├── Transaction manager
  ├── Paxos state machine
  └── Tablet → Colossus
```

### Directories: the unit of placement

A **directory** is a set of contiguous keys sharing a common prefix. Directories are the unit of:
- Data placement (all data in a directory has the same replication config)
- Data movement between Paxos groups
- Geographic replication policy control

Applications control locality by choosing key prefixes. A company might store each user's data under `users/{user_id}/...` as its own directory, enabling per-user replication policies (user A's data in 3 European replicas, user B's data in 5 North American replicas).

Directories are moved between Paxos groups by a background task called **Movedir**, which copies data in the background and uses a final atomic transaction to commit the metadata update — avoiding blocking ongoing reads/writes during bulky data moves.

A Paxos group can hold multiple directories. If a directory grows too large, Spanner shards it into **fragments** that can be served from different Paxos groups.

### Data model

Spanner's data model is semi-relational. Tables have schemas, typed columns, primary keys, and SQL — but with a twist: tables must form a **hierarchy** via `INTERLEAVE IN PARENT` declarations.

```sql
CREATE TABLE Users {
  uid INT64 NOT NULL, email STRING
} PRIMARY KEY (uid), DIRECTORY;

CREATE TABLE Albums {
  uid INT64 NOT NULL, aid INT64 NOT NULL, name STRING
} PRIMARY KEY (uid, aid),
  INTERLEAVE IN PARENT Users ON DELETE CASCADE;
```

Physical layout: `Albums` rows are stored **interleaved** with their parent `Users` row. `Albums(2, *)` is physically adjacent to `Users(2)`. This co-location is critical — without it, a join between Users and Albums would require cross-server network calls for every row. With it, joins between interleaved tables are entirely local.

The primary key is also the row's **name** — a row has existence only if some value is defined for its key columns. This gives Spanner its KV-store lineage while surfacing a relational interface.

---

## TrueTime: The Key to Everything

Every interesting property of Spanner depends on a single internal API called **TrueTime**.

### The problem with distributed clocks

In a distributed system, you cannot trust wall-clock time. Clocks drift. Network Time Protocol (NTP) gives you millisecond-level accuracy at best, and that's under good conditions. The standard approach is to use logical clocks (Lamport timestamps, vector clocks) that have no relationship to real time.

Spanner takes a radically different approach: **expose the clock uncertainty explicitly**, and reason about it.

### The API

```
TT.now()     → TTinterval: [earliest, latest]
TT.after(t)  → true if t has definitely passed
TT.before(t) → true if t has definitely not arrived
```

`TT.now()` does not return a point in time. It returns an **interval** `[earliest, latest]` guaranteed to contain the true absolute time at the moment of the call. The width of the interval represents uncertainty. If the interval is narrow, the system knows the time precisely. If it's wide, it's less certain.

In Google's production environment, the uncertainty `ε` (half the interval width) is **typically 1–7 ms**, averaging around 4 ms.

### The hardware implementation

TrueTime uses two independent time sources because each has different failure modes:

- **GPS receivers** — accurate but can fail due to antenna issues, radio interference, leap-second handling bugs, or signal spoofing
- **Atomic clocks (Armageddon masters)** — drift independently of GPS failures, but accumulate frequency error over time

Each datacenter has a set of **time master machines**:
- Majority have GPS receivers with dedicated, physically separated antennas
- Some are Armageddon masters with atomic clocks

Each machine runs a **timeslave daemon** that polls multiple time masters (nearby GPS masters, distant GPS masters, Armageddon masters) and applies Marzullo's algorithm to detect and exclude liars. The daemon advertises a slowly increasing uncertainty between polls. The poll interval is 30 seconds, drift rate 200 µs/sec, giving the sawtooth pattern of 0–6 ms uncertainty. Communication delay adds ~1 ms, keeping the 99th percentile below 7 ms.

### Why this matters

Prior distributed systems either assumed synchronized clocks (and were wrong) or avoided wall-clock time entirely (and lost real-time ordering). Spanner does neither — it uses bounded uncertainty to **convert real time into a correctness guarantee**.

---

## Concurrency Control

### Four operation types

| Operation | Concurrency control | Replica required |
|---|---|---|
| **Read-write transaction** | Pessimistic (2PL + 2PC) | Leader only |
| **Read-only transaction** | Lock-free, snapshot isolation | Leader for timestamp; any up-to-date replica for reads |
| **Snapshot read (client timestamp)** | Lock-free | Any up-to-date replica |
| **Snapshot read (client bound)** | Lock-free | Any up-to-date replica |

### Read-write transactions

Read-write transactions use **two-phase locking** within each Paxos group and **two-phase commit** across groups.

**Within a single Paxos group (the common case):**
1. Client reads go to the leader, which acquires read locks
2. Writes are buffered at the client until commit
3. At commit, client sends writes to the leader
4. Leader runs Paxos to replicate the commit record
5. Locks released after commit

**Across multiple Paxos groups (distributed transaction):**
1. Client picks one group as the coordinator
2. All non-coordinator participant leaders acquire write locks and log a prepare record through Paxos, each choosing a prepare timestamp
3. Coordinator acquires write locks, collects prepare timestamps from all participants, and chooses a commit timestamp `s` that must be:
   - ≥ all prepare timestamps
   - ≥ `TT.now().latest` at the time of the commit message
   - > any previous timestamp assigned by this leader
4. **Commit wait**: coordinator waits until `TT.after(s)` is true before applying — this ensures `s` is genuinely in the past
5. Coordinator notifies all participants of the commit timestamp; all apply at the same timestamp and release locks

The commit wait is typically 2ε ≈ 8 ms. This is the price Spanner pays for external consistency.

### External consistency — the proof

The invariant: if T1 commits before T2 starts, then `s1 < s2` (T1's commit timestamp is strictly less than T2's).

```
s1 < tabs(e1_commit)        — commit wait ensures s1 < real commit time
tabs(e1_commit) < tabs(e2_start)  — assumption: T1 commits before T2 starts
tabs(e2_start) ≤ tabs(e2_server)  — causality
tabs(e2_server) ≤ s2        — coordinator sets s2 ≥ TT.now().latest ≥ real time
∴ s1 < s2                   — transitivity
```

The **Start** rule (choose timestamp ≥ `TT.now().latest`) and the **Commit Wait** rule (wait until `TT.after(s)` is true before revealing results) together guarantee this ordering holds globally, without any inter-transaction coordination.

### Read-only transactions — lock-free global consistency

A read-only transaction executes in two phases:
1. Assign timestamp `sread = TT.now().latest`
2. Execute all reads as snapshot reads at `sread` — on any replica that is sufficiently up-to-date

No locks. No blocking. Reads can go to any nearby replica.

The replica's **safe time** `tsafe` is the maximum timestamp at which it is guaranteed to be up-to-date. A replica can serve a read at timestamp `t` if `t ≤ tsafe`. If `tsafe` hasn't advanced enough, the read waits briefly (usually milliseconds) or is redirected to a more up-to-date replica.

**Why this works**: The timestamp `sread` is chosen from real time using TrueTime. Because Spanner assigns commit timestamps monotonically within each Paxos group and each group's writes are applied in order, a replica that has applied all Paxos writes up to `sread` has a correct and complete snapshot at that time.

**The killer feature**: read a consistent snapshot of the entire database — trillions of rows across thousands of shards — without holding a single lock and without blocking any writer.

### Snapshot reads in the past

A client can issue a read at any past timestamp `t`:
- The system finds any replica where `tsafe ≥ t`
- That replica serves the read from its version history
- No coordination required

This enables non-blocking consistent backups, consistent MapReduce executions, and incremental change feeds — all without impacting ongoing transactions.

### Atomic schema changes

Schema changes normally require locking the entire database. With millions of data shards, that's infeasible. TrueTime makes schema changes non-blocking:

1. The schema change transaction is assigned a **future timestamp** `t`
2. Operations with timestamps before `t` proceed normally
3. Operations with timestamps after `t` block until the schema change completes
4. Because TrueTime gives a real-time bound, all spanservers can independently determine which side of `t` they're on — no coordination needed

---

## Performance Numbers (OSDI 2012)

### Microbenchmarks

| Configuration | Write latency | Read-only txn latency | Snapshot read latency |
|---|---|---|---|
| 1 replica | 14.4 ± 1.0 ms | 1.4 ± 0.1 ms | 1.3 ± 0.1 ms |
| 3 replicas | 13.9 ± 0.6 ms | 1.3 ± 0.1 ms | 1.2 ± 0.1 ms |
| 5 replicas | 14.4 ± 0.4 ms | 1.4 ± 0.05 ms | 1.3 ± 0.04 ms |

Key observations:
- Write latency stays roughly constant as replicas increase — Paxos runs in parallel across replicas, so quorum is dominated by the fastest majority, not all replicas
- Read-only throughput scales nearly linearly with replicas (reads go to any up-to-date replica)
- Commit wait (~5 ms, ~2ε) is dominated by the Paxos round-trip (~9 ms) in practice — they overlap

### Two-phase commit scalability

| Participants | Mean latency | 99th percentile |
|---|---|---|
| 1 | 17.0 ms | 75.0 ms |
| 10 | 30.0 ms | 95.6 ms |
| 50 | 42.7 ms | 93.7 ms |
| 100 | 71.4 ms | 131.2 ms |
| 200 | 150.5 ms | 320.3 ms |

Up to 50 participants is practical for both mean and tail latency. Latency begins rising noticeably at 100 participants.

### F1 production numbers (Google's ad backend)

| Operation | Mean latency | Std dev |
|---|---|---|
| All reads | 8.7 ms | 376.4 ms |
| Single-site commit | 72.3 ms | 112.8 ms |
| Multi-site commit | 103.0 ms | 52.2 ms |

The high standard deviation on reads is due to Paxos leaders distributed across two east-coast datacenters, only one of which had SSDs. F1 had 5 replicas across the US (2 west, 3 east) and experienced automatic failover invisibly — the only manual intervention needed after cluster failures was updating schema hints to keep Paxos leaders close to frontend servers.

---

## Spanner as a SQL System (SIGMOD 2017)

The 2012 paper described a globally-distributed KV store with transaction and replication guarantees. By 2017, Spanner had evolved into a full relational database. The 2017 paper describes the database DNA added on top.

### Why the evolution was necessary

Early Spanner had a NoSQL API: point lookups and range scans. Developers building OLTP applications found this insufficient. The pain points:
- No robust query language → developers wrote complex application code to process and aggregate data
- Schema-less design → type safety enforced in application, not storage
- No native SQL → couldn't leverage existing SQL tooling, ORMs, or analyst skills

The SQL layer was initially "bolted on" as a high-level API. This design didn't leverage Spanner's unique storage architecture (interleaved tables, range sharding) and left significant performance on the table. The 2017 paper describes the deep integration that followed.

### Distributed query execution

#### The DistributedUnion operator

Spanner's query compiler transforms SQL into a relational algebra tree and then introduces explicit **distribution operators**.

The fundamental operator is `DistributedUnion`: it ships a subquery to each shard and concatenates results.

```
Scan(T) → DistributedUnion[shard ⊆ T](Scan(shard))
```

The compiler then **pulls DistributedUnion up the tree** using rewriting rules, pushing as much computation as possible down to the shards. This is the key to efficiency — filter, project, sort, aggregate on each shard, then merge a small result set at the coordinator.

An operation is **partitionable** if:
```
F(Scan(T)) = OrderedUnionAll[shard ⊆ T](F(Scan(shard)))
```

Projections and filters are trivially partitionable. GroupBy and Top are partitionable when the sharding columns are a prefix of the grouping/sorting columns. For the general case, Spanner uses multi-stage processing: partial local aggregation pushed to shards, final merge at the DistributedUnion.

**Interleaved table joins** are pushed below DistributedUnion entirely — since child rows are co-located with parent rows, joins between interleaved tables execute as local joins on each shard with zero network calls.

#### Distributed Apply (batched key-lookup joins)

Standard nested-loop join in a distributed environment means one cross-machine call per row from the left side — catastrophically expensive. Spanner implements `DistributedApply`: it **batches** rows from the left side, sends a batch to each relevant shard, and executes the join locally on each shard.

```
Apply(input, map) → DistributedApply(Batch(input), Unnest, CrossApply(map))
```

Steps:
1. Collect a batch of rows from the left input
2. Extract sharding key ranges for each row in the batch
3. Merge ranges and compute minimal set of target shards
4. Send each shard only the batch rows relevant to it, in parallel
5. Execute the join locally per shard

This converts full-table scans into minimal range seeks and reduces cross-machine calls from O(rows) to O(shards × batches).

#### Query distribution APIs

**Single-consumer API**: query is sent to a root server (chosen via location hint extraction — a cached pattern of which shard owns which key range), which distributes subqueries to relevant shards and merges results.

**Parallel-consumer API**: for MapReduce-style data pipelines. The client requests the query to be partitioned into N pieces; Spanner returns N opaque partition descriptors; N independent client processes each execute one partition directly against the relevant shards in parallel. The concatenated results are identical to the single-consumer results.

### Query range extraction

Range extraction is the process of determining which rows a query touches, expressed as primary key intervals. Spanner uses this for three purposes:

- **Distribution range extraction** — which shards to route the query to
- **Seek range extraction** — which row ranges within a shard to actually read
- **Lock range extraction** — which key ranges to lock (or check for pending modifications)

**Compile-time rewriting**: The filter predicate is rewritten into a tree of correlated self-joins. Each join peels off one key column and emits the range for that column, enabling minimal seeks into the storage layer.

```sql
-- Original
WHERE d.ProjectId = @param1
AND STARTS_WITH(d.DocumentPath, '/proposals')
AND d.Version = @param2

-- Rewritten as three correlated scans:
Scan1(ProjectId)  → emits @project_id = @param1
Scan2(DocumentPath, with @project_id) → seeks to '/proposals/*', emits @doc_path
Scan3(all columns, with @project_id, @doc_path) → seeks to exact Version
```

Normalization steps include: pushing NOT to leaf predicates, normalizing key references (e.g., `1 > k` → `k < 1`), discretizing small integer ranges.

**Runtime filter tree**: A data structure that simultaneously computes key ranges via bottom-up interval arithmetic (intersect for AND, union for OR nodes) and evaluates post-filter conditions. The filter tree memoizes predicates whose values haven't changed and prunes branches that become unsatisfiable as key column values are fixed by enclosing scans.

### Query restarts

Spanner **fully hides transient failures** during query execution. Most distributed query processors surface retryable errors to the client; Spanner does not. A snapshot transaction will never return a retriable error.

Failures hidden:
- Network disconnects, machine reboots, process crashes
- Data resharding mid-query (shard boundaries change during execution)
- Replica not caught up to desired timestamp (distributed wait)

The mechanism: every batch of query results is accompanied by an opaque **restart token**. If a failure occurs, the client attaches the restart token to the re-issued request, and Spanner resumes from exactly where it left off — no duplicate rows, no missing rows.

Implementation challenges:
- **Dynamic resharding**: if shard `[20, 30)` is merged into `[10, 40)` mid-query, the restart token must encode progress in key space, not in shard identity
- **Non-determinism**: parallel execution across shards produces rows in non-repeatable order; restart must compensate without buffering large intermediate results
- **Cross-version restarts**: a query started on server version N must be restartable on version N+1 — restart token format, query plans, and operator behavior must all be version-compatible

Benefits:
- No retry loops in application code
- Streaming pagination without paging hacks
- Rolling server upgrades without disrupting latency-sensitive workloads (Spanner deploys new versions on a ~biweekly schedule across thousands of machines with no client-visible disruption)

### Standard SQL and the GoogleSQL library

Spanner adopted **Standard SQL** — a shared dialect used by Spanner, F1, Dremel, and BigQuery. This required:

**Shared compiler front-end**: parsing, name resolution, type checking, semantic validation — shared across all systems. This prevents subtle divergence (e.g., different coercion rules, different NULL handling). Output is a Resolved AST consumed by each system's own algebra.

**Shared scalar function library**: function implementations shared to prevent corner-case divergence.

**Shared test framework**: compliance tests (developer-written queries with expected results) plus randomized query generation (millions of random queries per day) compared against a reference implementation.

**Additions to standard SQL**:
- Protocol Buffer message and enum types as first-class SQL types
- `ARRAY` and `STRUCT` (row types) for nested data
- UTF8-based `STRING` instead of `CHAR`/`VARCHAR`

Impact: internal SQL adoption doubled in the year before the common dialect, then grew 5x in the year after. Distributed query complexity (measured by cross-machine calls per query) grew 4x — queries became more expressive, touching more data.

### Ressi: blockwise-columnar storage

The original storage format was inherited from Bigtable: **SSTables** (sorted string tables). SSTable is optimized for schemaless NoSQL — large string values, self-describing, and therefore highly redundant. It performs poorly for schematized relational data with small values accessed column-by-column.

**Ressi** is the replacement storage format, designed for hybrid OLTP/OLAP workloads.

Layout:
- Stores data as an LSM tree (log-structured merge-tree), same as SSTables
- Within each layer, organizes rows in **row-major order across blocks** (preserving Spanner's range-key locality)
- Within each block, stores data in **column-major order** (PAX layout — columnar within a row-group)
- Child table rows stored in same or adjacent blocks as their parent rows (honors `INTERLEAVE IN PARENT`)

Versioning:
- **Active file**: most recent values only — fast for current-data queries
- **Inactive file**: older versions — loaded only for historical reads
- **Large-value files**: segregated to avoid I/O cost during scans of tables with large blobs

Ressi's fundamental unit is the **vector**: an ordinally indexed, homogeneously typed sequence. Columns within a block are stored as one or more vectors. Ressi can apply operations directly on compressed vectors — no decompression needed for filter evaluation.

Migration: done live, group by group, using Spanner's own data movement mechanism. Each Paxos group's storage format specification is updated, new format replicas are brought up alongside old-format replicas for verification, then old replicas are decommissioned. Fully reversible at any stage.

---

## What Makes Spanner Unique

### The CAP position

Spanner is a CP system — it sacrifices availability during partition to maintain consistency. But "availability" here means availability of writes (which require Paxos quorum). Read-only transactions and snapshot reads can continue from any sufficiently up-to-date replica even during a partition, because they need no cross-replica coordination. In practice, with 5-replica groups, Spanner tolerates 2 simultaneous datacenter failures while remaining fully operational.

Brewer himself wrote in 2017 that Spanner essentially "proves CAP doesn't mean you have to choose — if your latency tolerance includes the ~10ms commit wait, you get both C and A in all but the worst partitions."

### TrueTime vs. logical clocks

Every prior distributed database either:
- Used logical clocks (Lamport timestamps, vector clocks) — correct but no real-time ordering
- Assumed synchronized clocks — fast but incorrect under clock skew

Spanner's insight: expose the uncertainty as a first-class primitive. Build the protocol so that uncertainty causes latency (commit wait = wait for the interval to close) rather than incorrectness. As hardware improves (better GPS, better atomic clocks, smaller ε), the performance improves automatically.

### Two-phase commit over Paxos

Conventional wisdom (cited in the paper) was that 2PC is too expensive due to availability and performance concerns. Spanner's response: run 2PC over Paxos. Each participant is a Paxos group, not a single machine. A participant failure doesn't block the 2PC — the Paxos group elects a new leader and continues. The availability concern disappears.

### External consistency without a global lock server

Chubby (Google's distributed lock service) uses Paxos to elect leaders and maintain locks. For global timestamp ordering, you might expect to need a single global authority. Spanner avoids this: TrueTime gives each spanserver the ability to independently choose timestamps that are guaranteed to be ordered correctly without any coordination with other spanservers. The commit wait is what makes this safe.

---

## The Evolution: From KV Store to SQL System

| Year | Milestone |
|---|---|
| ~2007 | Spanner begins as a globally-replicated namespace (KV store) |
| 2012 | OSDI paper: external consistency, TrueTime, global transactions |
| 2012 | F1 migrates Google's advertising backend off sharded MySQL |
| ~2013–2016 | SQL query processor added, initially as external API layer |
| 2017 | SIGMD paper: deep SQL integration, DistributedUnion, Ressi, query restarts, Standard SQL |
| 2017 | Cloud Spanner public beta on GCP |
| 2017 | 5,000+ internal databases, tens of millions QPS, hundreds of petabytes |

The key architectural lesson from the evolution: the storage layer (sharding, interleaving, key ranges) and the query layer (compilation, distribution, optimization) must be **co-designed**. SQL bolted on top of a KV store leaves performance on the table. Tight coupling — where interleaved tables inform join pushdown, sharding informs aggregation splitting, and range keys drive query routing — is what makes SQL at Spanner's scale practical.

---

## Key Concepts Quick Reference

| Term | Definition |
|---|---|
| **TrueTime** | Google's bounded-uncertainty clock API; returns `[earliest, latest]` interval guaranteed to contain true absolute time |
| **ε (epsilon)** | Half the TrueTime interval width; represents clock uncertainty; typically 1–7 ms in production |
| **Commit wait** | After assigning commit timestamp `s`, wait until `TT.after(s)` before revealing results; ensures `s < real commit time` |
| **External consistency** | If T1 commits before T2 starts, `s1 < s2`; equivalent to linearizability at global scale |
| **Paxos group** | Set of replicas that agree on the contents of a log via Paxos; one per shard or set of shards |
| **Tablet** | A key-value mapping `(key, timestamp) → value`; each spanserver holds 100–1000 tablets |
| **Directory** | Set of contiguous keys with a common prefix; unit of placement, replication config, and data movement |
| **Movedir** | Background task that moves directories between Paxos groups without blocking live traffic |
| **Safe time (tsafe)** | Maximum timestamp at which a replica is guaranteed up-to-date; gates read-only and snapshot reads |
| **DistributedUnion** | Query operator that ships a subquery to each shard and concatenates results |
| **DistributedApply** | Batched version of Apply join; sends batches of rows to relevant shards, executes join locally |
| **Range extraction** | Analyzing a query's WHERE clause to determine the minimal key ranges to scan/lock/route |
| **Filter tree** | Runtime data structure for computing key ranges via interval arithmetic and efficient post-filtering |
| **Restart token** | Opaque token attached to query results; allows resuming a query from mid-execution after failure |
| **Ressi** | Spanner's blockwise-columnar storage format; PAX layout within blocks; replaces SSTables |
| **Standard SQL** | Google's shared SQL dialect across Spanner, F1, Dremel, BigQuery |
| **INTERLEAVE IN** | Schema declaration that physically co-locates child table rows with parent rows |

---

## Sources

- Corbett et al., "Spanner: Google's Globally-Distributed Database," OSDI 2012
- Bacon et al., "Spanner: Becoming a SQL System," SIGMOD 2017
- Brewer, "Spanner, TrueTime and the CAP Theorem," Google Technical Report 2017
- Lamport, "The Part-Time Parliament" (Paxos), ACM TOCS 1998
- Baker et al., "Megastore," CIDR 2011
- Chang et al., "Bigtable," OSDI 2006
- Dean & Ghemawat, "MapReduce," CACM 2010
- Herlihy & Wing, "Linearizability," ACM TOPLAS 1990
- O'Neil et al., "The Log-Structured Merge-Tree," Acta Informatica 1996
