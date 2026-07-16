# Clock Sweep Algorithm

#databases #cache #page-replacement #buffer-pool

---

## Core Idea

Clock sweep is an **LRU approximation** used in PostgreSQL's buffer pool. It's called "clock" because a pointer sweeps around a circular list of pages — like a clock hand.

It avoids the cost of true LRU (which needs to update a sorted structure on every access) while retaining the same eviction quality in practice.

---

## How It Works

Every page in the buffer pool has a **use bit** (1 = recently accessed, 0 = not).

```
         [Page A]  use bit: 1
        /
[Page F]              [Page B]  use bit: 0
   1                   
[Page E]              [Page C]  use bit: 1
        \            /
         [Page D]  use bit: 0
              ↑
           Clock hand
```

**On every page access:** set use bit = 1 (cheap, no ordering needed).

**When eviction is needed:**
1. Look at page the hand points to
2. If use bit = **1** → set to 0, advance hand ("second chance")
3. If use bit = **0** → evict this page, insert new page here, advance hand

The hand stops as soon as it finds a victim. It does not scan the whole pool.

---

## Why It Approximates LRU

- Recently accessed pages almost certainly have bit = 1 → survive
- Pages not touched since the last pass have bit = 0 → get evicted
- Frequently hot pages keep getting their bit reset and keep surviving

It doesn't track exact recency order — but it correctly separates hot from cold in practice.

---

## When Does It Run?

> [!important] On demand only — not on a schedule
> The clock sweep runs **every time a cache miss requires eviction**. If your working set fits in memory and you're hitting cache constantly, it barely runs. Under heavy misses, it runs continuously.

**Cost scales with how hot the pool is:**
- Cold pool (many 0 bits) → hand finds victim immediately
- Hot pool (all 1 bits) → hand does a full loop clearing bits before finding anything (worst case, still O(n) once)

---

## True LRU vs Clock Sweep

| | True LRU | Clock Sweep |
|---|---|---|
| Data structure | Sorted linked list | Circular array + pointer |
| On access | Move page to front (expensive) | Set bit = 1 (cheap) |
| On eviction | Pop from tail | Sweep until bit = 0 |
| Accuracy | Exact recency order | Approximate |
| Used by | Conceptual baseline | PostgreSQL buffer pool |

---

## Not the Same as CPU Clock Speed

> [!note] The naming is coincidental
> - **CPU clock speed** = hardware oscillator frequency (GHz) — how many cycles per second
> - **Clock sweep** = a logical pointer sweeping a circular data structure
>
> One is physics, the other is an algorithm metaphor. No relationship.

---

## Key Takeaways

- Clock sweep is immune to Bélády's anomaly (it's an LRU approximation, satisfies the stack property)
- Bit-setting on access is O(1) — almost zero overhead per read
- Eviction cost is amortized — worst case clears bits on one full loop, then finds victim
- PostgreSQL uses this for its shared buffer pool (`shared_buffers`)

---

## Related Notes
- [[Belady's Anomaly]] — stack property and why LRU approximations are immune
- [[TinyLFU]] — more sophisticated eviction using frequency + recency
- [[Fuzzy Checkpoint]] — what happens to dirty pages the clock sweep evicts
