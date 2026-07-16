# Fuzzy Checkpoint

#databases #postgresql #wal #durability #recovery

---

## WAL Write Order — The Foundation

> [!important] WAL is always written before the data file. Always.
> This is what "write-ahead" means.

```
1. Transaction modifies page in buffer pool (memory only)
2. WAL record written to WAL buffer (memory)
3. WAL buffer flushed to disk ← must happen before commit
4. COMMIT acknowledged to client
5. Dirty page written to data file on disk ← happens later, lazily
```

A transaction is not considered committed until its WAL is on disk. The data page being on disk is **not** required for commit.

**Why this order?**
If the database crashes after commit, WAL already has the record. On restart, PostgreSQL replays WAL and recovers the change. If you wrote data first and crashed before WAL was written — no recovery possible.

---

## What a Checkpoint Does

Flushes dirty pages from buffer pool → disk, so old WAL can be safely discarded. Once a page is on disk, you don't need the WAL that describes changes to it.

- **Sharp checkpoint** — pause everything, flush all at once. Causes a massive I/O spike.
- **Fuzzy checkpoint** — flush gradually in the background while transactions keep running. No stall.

PostgreSQL uses fuzzy checkpoints.

---

## Fuzzy Checkpoint: Step by Step

```
Time ──────────────────────────────────────────────────────────►
      │                                    │
Checkpoint starts                   Checkpoint ends
      │                                    │
  [CHECKPOINT_START written to WAL]  [CHECKPOINT_END written to WAL]
      │                                    │
      └── transactions keep running throughout ──────────────────┘
```

**Step 1 — Record the redo point**
Checkpointer writes `CHECKPOINT_START` to WAL. The current LSN becomes the **redo point**.

**Step 2 — Flush dirty pages gradually**
Checkpointer sweeps the buffer pool, writing dirty pages to disk slowly — paced by `checkpoint_completion_target`.

**Step 3 — Write CHECKPOINT_END**
All dirty pages from the redo point list are on disk. Old WAL before the redo point can now be recycled.

---

## checkpoint_completion_target

`checkpoint_completion_target = 0.9` (default) means: finish flushing all dirty pages within **90% of the checkpoint interval**.

With `checkpoint_timeout = 5min` (300s):

```
0s                    270s        300s
│─────────────────────│────────────│
│  flush dirty pages  │  buffer    │← next checkpoint
│←──── 270s ─────────→│
        (90% of 300s)
```

**Why not 100%?** The 10% slack absorbs new dirty pages created mid-checkpoint and slower-than-expected disk throughput. At 100%, any hiccup causes checkpoints to pile up.

**Effect on I/O:**

```
Without pacing:              With pacing (0.9):
│        ████               │  ██ ██ ██ ██ ██ ██ ██
│        ████               │  ██ ██ ██ ██ ██ ██ ██
│________████___            │__████████████████████
         spike                 spread evenly
```

Total I/O is the same — just whether you spike or spread it.

---

## The Redo Point and WAL Recycling

The redo point is the LSN at which `CHECKPOINT_START` was written.

```
WAL:
LSN: 0100  0110  0200  0210  [0300]  0400  0500  0600
                              ↑
                          redo point
                       (CHECKPOINT_START)

← before redo point →  ← after redo point →
   already on disk         still needed
   RECYCLABLE              for recovery
```

**On crash recovery:**
1. Find latest `CHECKPOINT_END` record
2. Get redo point from it
3. Replay all WAL **from redo point forward**
4. WAL before redo point is never needed — those changes are already in data files

PostgreSQL doesn't always delete old WAL — it often **renames** segment files for reuse (cheaper than allocating new files, avoids fragmentation).

> [!note] Replicas can delay recycling
> If a replica is lagging behind, PostgreSQL holds onto WAL segments the replica hasn't consumed yet — even if the checkpoint has moved past them.

---

## Concrete Example

**Table:** `accounts(id, balance)`  
**Initial disk state:** `id=1: 500, id=2: 800`

### Pre-checkpoint WAL
```
LSN 0100 │ XID 201 │ HEAP_UPDATE │ page 7 │ id=1: 500→750
LSN 0110 │ XID 201 │ COMMIT
LSN 0200 │ XID 202 │ HEAP_UPDATE │ page 7 │ id=2: 800→950
LSN 0210 │ XID 202 │ COMMIT
```
Buffer pool: `page 7 [DIRTY] — id=1:750, id=2:950`  
Disk: `page 7 — id=1:500, id=2:800` ← stale

### Checkpoint Begins
```
LSN 0300 │ CHECKPOINT_START │ redo_point=0300 │ dirty=[page 7]
```
Nothing flushed yet. Transactions keep running.

### New Transaction Mid-Checkpoint
```
LSN 0400 │ XID 203 │ HEAP_UPDATE │ page 7 │ id=1: 750→1000
```
XID 203 has not committed yet. Buffer pool: `page 7 — id=1:1000, id=2:950`

### Checkpointer Flushes Page 7
Writes current in-memory state to disk:  
Disk now: `page 7 — id=1:1000, id=2:950`

XID 203 is still uncommitted. The on-disk page has an uncommitted value. **This is the "fuzzy" part** — the data file is not in a clean committed state. WAL will fix this on recovery if needed.

### XID 203 Commits
```
LSN 0500 │ XID 203 │ COMMIT
```

### Checkpoint Ends
```
LSN 0600 │ CHECKPOINT_END │ redo_point=0300
```

**Final WAL:**
```
LSN 0100 │ XID 201 │ HEAP_UPDATE │ 500→750       ┐
LSN 0110 │ XID 201 │ COMMIT                       │ RECYCLABLE
LSN 0200 │ XID 202 │ HEAP_UPDATE │ 800→950        │ (before redo point)
LSN 0210 │ XID 202 │ COMMIT                       ┘
LSN 0300 │ CHECKPOINT_START ← redo point
LSN 0400 │ XID 203 │ HEAP_UPDATE │ 750→1000       ┐
LSN 0500 │ XID 203 │ COMMIT                       │ MUST KEEP
LSN 0600 │ CHECKPOINT_END                         ┘
```

---

## Crash Recovery Scenario

**Crash happens after LSN 0400 but before LSN 0500** (XID 203 UPDATE written, no COMMIT yet).

Disk has `id=1:1000` (uncommitted value written during checkpoint flush).

**Recovery:**
1. Find `CHECKPOINT_END` → redo point = LSN 0300
2. Replay from LSN 0300:
   - LSN 0400: apply XID 203's UPDATE tentatively
   - LSN 0500: not found (crash) → XID 203 never committed
3. Roll back XID 203 → `id=1` reverts to 750
4. Database is in correct committed state

The fuzzy inconsistency is fully resolved by WAL replay. WAL is always the final authority.

---

## Key Takeaways

- WAL is written **before** data file. Always. Commit = WAL on disk.
- Fuzzy checkpoint = background flush, world doesn't pause
- Redo point = the LSN recorded at checkpoint start; recovery replays from here
- Data files are transiently inconsistent during checkpoint — WAL replay fixes this
- `checkpoint_completion_target` throttles I/O to avoid disk spikes
- Old WAL before redo point is safe to recycle after checkpoint completes

---

## Related Notes
- [[PostgreSQL Internals]] — MVCC, VACUUM, WAL buffer, circular buffers
- [[Clock Sweep Algorithm]] — how dirty pages get evicted from buffer pool
- [[Steal and Force Policies]] — why fuzzy checkpoints exist (NO-FORCE) and why mid-checkpoint uncommitted data on disk is safe (STEAL + UNDO)
- [[Copy-on-Write]] — CoW semantics at the buffer pool level during checkpoint flush
