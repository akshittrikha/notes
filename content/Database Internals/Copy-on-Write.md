# Copy-on-Write (CoW)

#databases #os #memory #concurrency #snapshots

---

## Core Idea

> Multiple readers share one physical copy of data. A private copy is made only when someone *writes* — and only for the writer.

Copying upfront is wasteful if most of the data never changes. CoW defers the cost: **share until a write forces separation**.

---

## How It Works

```
Before any write:
  Process A ──┐
              ├──→ [Page X: value=42]   ← one physical copy, shared
  Process B ──┘

Process B writes value=99 to Page X:
  Process A ──────→ [Page X: value=42]  ← original, untouched
  Process B ──────→ [Page X: value=99]  ← new private copy, just for B
```

**Mechanism (OS level):**
1. Shared pages are marked **read-only** in the page table
2. Either party reads → no fault, no copy, zero cost
3. A party writes → CPU raises a **page fault**
4. OS intercepts, duplicates the page, remaps the writer to the new copy
5. Write proceeds on the private copy

---

## Where It Appears

### OS `fork()`
Classic example. Child process gets a snapshot of parent's address space instantly — shared, not copied. Pages are only duplicated when either parent or child modifies them.

```
fork() → child shares all pages (O(1))
child modifies page → that page is copied (O(1) per page)
exec() after fork → child replaces memory anyway, copy never needed
```

### PostgreSQL MVCC
Old row versions are preserved rather than overwritten. Writers append a new tuple version; readers keep pointing at the old one. CoW at the tuple level.

```
UPDATE id=1: 500 → 750

Before: [tuple: id=1, val=500, xmin=100, xmax=0]
After:  [tuple: id=1, val=500, xmin=100, xmax=201]  ← old, visible to old txns
        [tuple: id=1, val=750, xmin=201, xmax=0]    ← new, visible to new txns
```

### btrfs / ZFS Snapshots
A filesystem snapshot shares all blocks with the live filesystem at creation time — O(1). As the live filesystem writes, only the modified blocks are copied to new locations. The snapshot retains the original blocks.

### Redis BGSAVE
Redis forks to serialize memory to disk. The fork shares all memory. The parent keeps serving writes; each write CoW's the modified page into a new copy. The child sees a frozen snapshot of memory at fork time.

### String types (historical C++)
`std::string` copies were O(1) via shared backing buffer; CoW triggered on mutation. Dropped in C++11 due to multithreading complexity.

---

## Trade-offs

| Scenario | Benefit | Cost |
|---|---|---|
| Read-heavy | Zero overhead — pages shared, no copies | — |
| Write-heavy | Pay only for pages actually modified | Page fault + memcpy per first write |
| Memory | Only diverged pages duplicated | Each writer accumulates its own diff |
| Multithreaded | Complex — page fault synchronization needed | Lock contention on fault handling |

> [!important] CoW is lazy copying
> You pay only for writes that actually happen, not all writes that *might* happen. For read-dominated workloads, the savings are enormous.

---

## CoW vs Eager Copy

```
Eager copy (fork without CoW):
  fork() → copy all 8 GB of memory → child runs
  Cost: always 8 GB copy, even if child exec()s immediately

CoW (fork with CoW):
  fork() → share all 8 GB → child modifies 50 MB → 50 MB copied
  Cost: 50 MB copy. 8 GB never touched.
```

---

## Connection to Database Internals

**MVCC + CoW:** PostgreSQL's [[PostgreSQL Internals|MVCC]] is CoW at the tuple level. The old version is preserved for concurrent readers — no lock needed, readers never block writers.

**Fuzzy checkpoint + STEAL policy:** During a [[Fuzzy Checkpoint|fuzzy checkpoint]], the checkpointer flushes a page while a new transaction may be modifying it in memory. These aren't the same copy — the transaction works on the in-memory page, the checkpointer flushes what was there. CoW semantics at the buffer pool level.

**Redis BGSAVE + WAL:** Redis's fork-based snapshotting is CoW in action. PostgreSQL takes the opposite approach — WAL + [[Steal and Force Policies|NO-FORCE]] — because CoW doesn't work well when the dataset exceeds memory.

---

## Key Takeaways

- Share first, copy only on write — defers and often eliminates copy cost
- OS implements it via read-only page table entries + page fault handler
- Foundation of `fork()`, filesystem snapshots, MVCC, and in-memory DB backups
- Cost is per-write, not per-fork — great for read-heavy or snapshot workloads
- Multithreading complicates CoW — page fault handling needs synchronization

---

## Related Notes
- [[PostgreSQL Internals]] — MVCC uses CoW at the tuple level
- [[Fuzzy Checkpoint]] — buffer pool page lifecycle during checkpoint
- [[Steal and Force Policies]] — how uncommitted pages are managed relative to disk

