# Database Architecture

#databases #architecture #query-processor #execution-engine #storage-engine

---

## Core Idea

A database is a pipeline of layers, each with a narrow job, stacked between "client sends bytes" and "bytes land on disk." Every layer only talks to its immediate neighbor — the transport layer doesn't know what a B-tree is, and the storage engine doesn't know what SQL looks like.

```
Client
  │  bytes over the wire
  ▼
┌─────────────────┐
│    Transport     │  ← speaks the wire protocol, manages connections
└─────────────────┘
  │  raw query string
  ▼
┌─────────────────┐
│  Query Processor │  ← parse → validate → optimize → physical plan
└─────────────────┘
  │  physical plan (tree of operators)
  ▼
┌─────────────────┐
│ Execution Engine │  ← walks the plan, pulls rows operator by operator
└─────────────────┘
  │  get/put/scan calls on specific pages & keys
  ▼
┌─────────────────┐
│  Storage Engine  │  ← transactions, locking, buffer pool, WAL, disk access
└─────────────────┘
  │
  ▼
 Disk
```

---

## Transport

Owns the client connection: accepting sockets, speaking the wire protocol (e.g. PostgreSQL's message protocol, MySQL's protocol), authentication, and framing requests/responses. It hands the query processor a plain string (or a pre-parsed statement, for prepared statements) and has zero opinion about what the query means.

**Communicates:** client ↔ transport (protocol messages) → transport hands query text to query processor, and later ships result rows back to the client in wire format.

---

## Query Processor

Turns a query string into an executable plan. Three sub-stages:

1. **Parser** — query string → AST (syntax only, no semantic knowledge yet)
2. **Binder / validator** — resolves table and column names against the catalog, checks types, rejects invalid queries (this is where "table does not exist" errors come from)
3. **Optimizer** — AST → **logical plan** (what to do: join these two tables, filter this, aggregate that) → **physical plan** (how to do it: hash join vs. nested loop, index scan vs. sequential scan). Cost-based optimizers estimate I/O and CPU cost of candidate plans using table statistics and pick the cheapest.

**Communicates:** receives raw text from transport, queries the **catalog** (metadata: schemas, indexes, statistics) to validate and cost plans, hands the finished physical plan to the execution engine. Never touches actual data pages.

---

## Execution Engine

Takes the physical plan — a tree of operators (scan, filter, join, aggregate, sort) — and actually runs it, usually via the **iterator model** (a.k.a. Volcano model): every operator exposes a `next()` call that pulls one row at a time from its children, so a whole complex query streams row-by-row without materializing intermediate results in memory.

```
Aggregate.next()
  → calls Join.next()
      → calls Scan(orders).next()   → asks storage engine for next row
      → calls Scan(users).next()    → asks storage engine for next row
```

**Communicates:** pulls plan-shaped requests down to the storage engine (`get key`, `scan range`, `insert row`) and pushes result rows up to the transport layer for encoding back to the client. It doesn't know about disk, pages, or WAL — it just calls the storage engine's access API.

---

## Storage Engine

Everything below "give me row X" lives here. This is the layer covered by every note already in this vault:

- **Transaction manager** — tracks transaction state, assigns XIDs, coordinates commit/abort
- **Lock manager / MVCC** — controls concurrent access; see [[PostgreSQL Internals|MVCC]] for the snapshot-based version that avoids locking entirely
- **Access methods** — the actual on-disk data structure: [[Storage Engines|B-Tree or LSM-Tree]]
- **Buffer manager** — in-memory page cache sitting in front of disk; eviction policy is [[Clock Sweep Algorithm|Clock Sweep]] (PostgreSQL) or [[TinyLFU|W-TinyLFU]] (Caffeine, RocksDB)
- **Recovery manager** — WAL, [[Fuzzy Checkpoint|checkpointing]], and the [[Steal and Force Policies|STEAL/FORCE policy]] that determines whether UNDO and/or REDO logging is needed

**Communicates:** receives get/put/scan calls from the execution engine, translates them into page-level operations against the buffer manager, which in turn talks to disk. Every write path also routes through the transaction/recovery manager to satisfy durability guarantees before acknowledging success.

---

## End-to-End Example

```
Client: SELECT name FROM users WHERE id = 42;

1. Transport:        decodes wire protocol → "SELECT name FROM users WHERE id = 42"
2. Query Processor:   parse → AST
                       bind → validate `users` exists, `id`/`name` are real columns
                       optimize → physical plan: IndexScan(users_pkey, id=42) → Project(name)
3. Execution Engine:   IndexScan.next() called
                       → asks Storage Engine: "get row where id=42 via users_pkey index"
4. Storage Engine:     Buffer manager checks if the relevant B-tree page is cached
                       → cache hit: return page from memory (no disk I/O)
                       → cache miss: clock sweep evicts a victim, page read from disk
                       MVCC snapshot check filters row visibility (xmin/xmax)
                       row returned up to Execution Engine
5. Execution Engine:   Project(name) strips row down to just `name`
6. Transport:          encodes result row(s) into wire protocol → sent to client
```

---

## Why the Separation Matters

> [!important] Each layer can be optimized or swapped independently
> This is why PostgreSQL can change its optimizer's cost model without touching the buffer pool, or why RocksDB (just a storage engine) can be embedded under completely different query layers (MyRocks under MySQL's SQL layer, or used directly with no SQL at all). The interfaces between layers are the actual API contracts of a database system.

---

## Key Takeaways

- Four layers, strict one-directional data flow: Transport → Query Processor → Execution Engine → Storage Engine
- Transport: protocol + connections only, no query semantics
- Query Processor: parse → bind/validate → optimize (logical plan → physical plan), consults the catalog
- Execution Engine: runs the physical plan via the iterator/Volcano model, pulling rows operator by operator
- Storage Engine: transaction manager, lock/MVCC, access methods, buffer manager, recovery manager — where every concept already in this vault lives
- Clean separation between layers is what lets each be independently optimized, tested, or replaced

---

## Related Notes
- [[Storage Engines]] — the access-methods layer in detail (B-Tree vs LSM-Tree)
- [[PostgreSQL Internals]] — MVCC, VACUUM, WAL buffer (storage engine internals)
- [[Clock Sweep Algorithm]] / [[TinyLFU]] — buffer manager eviction policies
- [[Fuzzy Checkpoint]] / [[Steal and Force Policies]] — recovery manager mechanics
