# TinyLFU (W-TinyLFU)

#databases #cache #page-replacement #eviction

---

## Why LFU Alone Isn't Enough

Pure LFU tracks frequency — but new items start at frequency 0. A brand-new hot item will always lose to an old cold item with accumulated counts. TinyLFU fixes this with an **admission gate**.

---

## Page-in vs Page Reference Events

> [!note] Key distinction
> - **Page-in event** — fires when a page is loaded from disk (cache miss only). Counts how many times it was *fetched*.
> - **Page reference event** — fires on *every* access, hit or miss. Counts how many times it was *used*.

LFU must track **reference events**, not page-in events. A page loaded once but accessed 1000 times has 1 page-in but 1000 references — if you count page-ins, it looks cold. Counting references gives the correct picture.

---

## W-TinyLFU Architecture

Full name: **Window TinyLFU**. Three segments + an admission filter.

```
New items
   ↓
┌──────────────┐    evicted     ┌─────────────────────────────────┐
│    Window    │ ─────────────→ │           Main Cache            │
│   Cache (1%) │  TinyLFU       │                                 │
│    (LRU)     │  admission     │  ┌────────────┐ ┌───────────┐   │
└──────────────┘  filter        │  │  Protected │ │ Probation │   │
                                │  │   (~80%)   │ │  (~20%)   │   │
                                │  └────────────┘ └───────────┘   │
                                └─────────────────────────────────┘
```

---

## Each Segment Explained

### Window (Admission Queue) — 1%
- Every new item enters here, no questions asked
- Managed as LRU
- Tiny by design — gives new items a chance to accumulate recency before facing the frequency filter
- Without this, a new hot item (frequency = 0) would always be rejected

### Probation — ~20% of main cache
- Items that passed the admission filter land here
- On trial — haven't proven sustained usage yet
- **Eviction candidates come from here first**
- Access while on probation → promoted to Protected

### Protected — ~80% of main cache
- Safe from eviction (until segment fills)
- Earned by being accessed while on probation
- If Protected overflows → least recently used item demoted back to Probation (not evicted)

---

## The Admission Filter

When the window is full and evicts an item, that item is a **candidate** for the main cache.

```
Window candidate  ──┐
                    ├──→ Compare frequencies → winner enters Probation
Probation victim  ──┘    loser is dropped
```

Frequency comparison uses a **Count-Min Sketch** — a compact probabilistic data structure that approximates frequency with very little memory. It doesn't store exact per-item counts; it gives a close enough estimate.

Counts also **decay over time** so old stale frequency doesn't keep old items alive forever.

---

## Why This Design Works

| Problem | Solution |
|---|---|
| New hot items have frequency 0 | Window gives them recency-based admission first |
| Old items inflate frequency counts | Count-Min Sketch decays counts over time |
| One-time scans pollute cache | Window is tiny; scan items never accumulate enough frequency to enter main cache |
| Hot items evicted by slightly-hotter competitors | Protected segment shields them |

---

## Key Takeaways

- **Window** = recency guard for new items
- **Probation** = items that passed admission but not yet proven
- **Protected** = hot, safe items
- **Admission filter** = frequency check at the window→main boundary
- Count-Min Sketch makes frequency tracking memory-efficient
- Used in **Caffeine** (Java), **RocksDB** — not PostgreSQL (which uses [[Clock Sweep Algorithm]])

---

## Related Notes
- [[Clock Sweep Algorithm]] — LRU approximation used by PostgreSQL
- [[Belady's Anomaly]] — why algorithm choice matters for cache correctness
- [[PostgreSQL Internals]] — broader PostgreSQL context
- [[Database Architecture]] — the buffer manager's place within the storage engine layer
