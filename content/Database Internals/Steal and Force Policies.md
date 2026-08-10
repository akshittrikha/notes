# Steal and Force Policies

#databases #postgresql #wal #buffer-pool #recovery

---

## The Two Questions

Every buffer pool must answer two questions about dirty pages:

> **STEAL / NO-STEAL:** Can the buffer pool evict a dirty page belonging to an *uncommitted* transaction to disk?

> **FORCE / NO-FORCE:** Must all of a transaction's dirty pages be flushed to disk *before* commit is acknowledged?

These two policies determine what crash recovery mechanism the database needs.

---

## Steal Policy

```
Buffer pool is full. Page X is dirty, owned by uncommitted XID 201.
A new page needs to load. Something must be evicted.

STEAL:    Evict page X to disk. XID 201 still running.
          Disk now has uncommitted data. ← dangerous?

NO-STEAL: Cannot evict page X. Must wait for XID 201 to finish.
          Uncommitted data never leaves memory. ← safe but costly
```

**NO-STEAL** is simple but impractical — a long-running transaction could fill the buffer pool and block all evictions.

**STEAL** is what real databases use. The clock sweep can freely evict dirty pages from uncommitted transactions.

> [!important] STEAL introduces uncommitted data on disk
> If XID 201 aborts after its page was stolen to disk, that page must be rolled back. This requires **UNDO logging** — WAL must store the *before-image* of every change.

---

## Force Policy

```
XID 201 commits.

FORCE:    Flush all of XID 201's dirty pages to disk before ACK.
          Commit = all pages on disk. Safe but causes I/O spike.

NO-FORCE: ACK commit immediately. Pages stay in memory,
          written to disk lazily by the checkpointer.
```

**FORCE** guarantees durability directly but is expensive — a transaction touching 10,000 pages forces 10,000 random writes before the client gets "OK".

**NO-FORCE** is what real databases use. Commit only requires WAL to be on disk (fast sequential write), not the data pages.

> [!important] NO-FORCE means committed data may not be on disk
> If the database crashes after commit but before pages are flushed, data must be reconstructed. This requires **REDO logging** — WAL must store the *after-image* of every change.

---

## The Four Combinations

| | FORCE | NO-FORCE |
|---|---|---|
| **NO-STEAL** | No UNDO, no REDO. Simple but memory-limited — must hold all active tx pages. | No UNDO, needs REDO. Pages can't be held forever anyway. Impractical. |
| **STEAL** | Needs UNDO, no REDO. All pages flushed at commit = I/O spike on every commit. | Needs UNDO + REDO. Maximum flexibility. **PostgreSQL uses this.** |

---

## Why STEAL + NO-FORCE

```
STEAL    → buffer pool can evict freely         → need UNDO
NO-FORCE → commit is fast (WAL write only)      → need REDO
WAL      → stores before AND after images       → satisfies both
```

WAL solves both problems with one sequential write stream:

- **REDO path:** on crash, replay WAL forward from redo point — recovers committed changes not yet in data files
- **UNDO path:** on crash, roll back uncommitted transactions whose pages were stolen to disk

> [!note] This is why WAL exists
> STEAL + NO-FORCE gives maximum throughput. WAL is what makes it safe. "Write-ahead" means the log precedes the data file — so no matter what's on disk, WAL can fix it.

---

## Recovery Implications

```
Crash scenario:

  XID 201 [committed]  — pages may not be on disk   → REDO needed
  XID 202 [aborted]    — pages may already be on disk → UNDO needed
  XID 203 [in-flight]  — unknown state               → UNDO needed

Recovery:
  1. Find CHECKPOINT_END → get redo point
  2. Replay WAL forward from redo point (REDO committed txns)
  3. Identify uncommitted txns at crash time
  4. Roll them back using before-images (UNDO)
```

---

## Key Takeaways

- **STEAL** = buffer pool freedom to evict uncommitted pages → requires UNDO
- **NO-STEAL** = memory pressure risk, impractical for long transactions
- **FORCE** = durability guaranteed at commit cost → I/O spike per commit
- **NO-FORCE** = fast commits, lazy flushes → requires REDO
- PostgreSQL uses **STEAL + NO-FORCE** — the most performant combination
- WAL provides both UNDO and REDO, making STEAL + NO-FORCE safe

---

## Related Notes
- [[Fuzzy Checkpoint]] — consequence of NO-FORCE; pages not on disk at commit, checkpoints bound recovery time
- [[Clock Sweep Algorithm]] — the eviction mechanism that does the stealing
- [[PostgreSQL Internals]] — WAL buffer, MVCC, durability guarantees
- [[Database Architecture]] — where these policies sit within the storage engine's recovery manager
- [[Storage Engines]] — STEAL/FORCE is a B-Tree-family concern; LSM-Trees sidestep it via immutable SSTables

