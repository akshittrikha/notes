# Storage Engines

#databases #storage-engines #b-tree #lsm-tree #architecture

---

## Core Idea

A storage engine is the piece of a database that owns the physical layer — how rows/values are actually laid out, written, and found on disk (or in memory). The query planner decides *what* to fetch; the storage engine decides *how* that fetch touches disk.

Almost every storage engine design decision traces back to one trade-off triangle, often called the **RUM conjecture**:

- **R**ead amplification — how many disk reads does one logical lookup cost?
- **U**pdate (write) amplification — how much I/O does one logical write actually trigger?
- **M**emory (space) amplification — how much extra disk space is used beyond the bare minimum for the same logical data?

You can optimize for at most two of the three. The two dominant engine families each pick a different two.

---

## B-Tree Family (in-place update)

A balanced tree of fixed-size pages (typically 4–16 KB), stored on disk, with a huge fan-out per node (hundreds of keys per page) so tree height stays tiny — 3–4 levels covers billions of rows.

**Writes:** modify the page in place. This means **random I/O** — the page you need to update could be anywhere on disk. To make this safe, the engine needs a write-ahead log (already covered — this is exactly why [[Fuzzy Checkpoint|WAL]] and [[Steal and Force Policies|STEAL/NO-FORCE]] exist).

**Reads:** optimized — one tree traversal, one version of the truth, no merging of multiple sources.

**Examples:** PostgreSQL (heap + B-tree indexes), MySQL InnoDB, SQLite.

---

## LSM-Tree Family (log-structured, append-only)

Writes never touch existing on-disk structures directly.

```
Write path:
  1. Append to WAL (durability)
  2. Insert into in-memory memtable (sorted structure, e.g. skip list)
  3. Memtable fills → flush as an immutable sorted file (SSTable) on disk
  4. Background compaction merges SSTables, drops overwritten/deleted keys
```

**Writes:** always sequential — append to WAL, append a new SSTable. Cheap.

**Reads:** a key might live in the memtable, or any of several SSTables (newest first). Worst case you check them all. **Bloom filters** per SSTable let you skip files that definitely don't contain the key, mitigating this.

**Examples:** RocksDB, LevelDB, Cassandra, HBase.

---

## Comparison

| | B-Tree | LSM-Tree |
|---|---|---|
| Write pattern | Random, in-place | Sequential, append-only |
| Write amplification | Low–medium | High (compaction rewrites data repeatedly) |
| Read amplification | Low (single traversal) | Higher (multiple SSTables; bloom filters help) |
| Space amplification | Low (mostly one copy, some page fragmentation) | Higher until compaction runs (stale versions coexist) |
| Best for | Read-heavy / balanced workloads | Write-heavy workloads |
| Examples | PostgreSQL, InnoDB, SQLite | RocksDB, Cassandra, LevelDB |

---

## The Same Problem Wearing Two Costumes

> [!important] Compaction ≈ VACUUM
> LSM compaction and PostgreSQL's [[PostgreSQL Internals|VACUUM]] solve the identical problem: **reclaim space from stale/overwritten versions of a key**. B-trees overwrite in place so there's no "dead tuple" *inside* the tree — but MVCC still leaves dead row versions behind, which VACUUM cleans up. LSM-trees never overwrite at all — every update is a brand-new entry — so *compaction* is where old versions actually get discarded. Same job, different layer.

Note the inversion: B-trees pay their amplification cost on the **write** path (random I/O per update), LSM-trees defer it to a **background** process (compaction) and pay on writes only in aggregate, over time.

---

## Key Takeaways

- Storage engine = the physical layer: how data actually sits on disk, independent of SQL/query planning
- RUM conjecture: read, write, and space amplification trade off — no engine wins all three
- B-Trees: in-place random writes, cheap single-traversal reads — needs WAL + STEAL/FORCE policy for crash safety
- LSM-Trees: sequential append-only writes, reads may span multiple SSTables — needs compaction to bound read/space amplification
- Compaction (LSM) and VACUUM (MVCC/B-tree) are the same underlying idea: reclaiming space from stale versions
- This is the organizing split for the book's storage-engine chapters — most concrete mechanisms (B-tree page layout, LSM compaction strategies) will get their own dedicated notes as we cover them

---

## Related Notes
- [[PostgreSQL Internals]] — VACUUM as a B-tree engine's answer to space reclamation
- [[Fuzzy Checkpoint]] — WAL/checkpoint mechanics that B-tree engines rely on for crash safety
- [[Steal and Force Policies]] — buffer pool policies specific to in-place B-tree engines
