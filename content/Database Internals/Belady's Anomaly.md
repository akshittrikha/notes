# Bélády's Anomaly

#databases #os #cache #page-replacement

---

## The Core Idea

> Adding more cache frames should reduce page faults. Bélády's anomaly is when the **opposite happens** — more frames, more faults.

This only occurs in algorithms that **violate the stack property**. FIFO is the classic example.

---

## Why FIFO is Vulnerable

FIFO evicts the page that arrived first — regardless of how recently or frequently it's been used. When you add a frame, the eviction order reshuffles, and a page that would've survived in a smaller cache gets evicted in a larger one.

The sets of pages kept at different frame sizes are **not nested** — they overlap but neither contains the other.

---

## Quick Example

**Reference string:** `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`

| Frames | Page Faults |
|--------|-------------|
| 3 | 9 |
| 4 | **10** |

More frames → more faults. That's the anomaly.

---

## The Stack Property

The key concept behind immunity to Bélády's anomaly.

**Definition:** The set of pages in memory with *n* frames is always a subset of the pages with *n+1* frames — for any reference string.

```
Pages with 3 frames: {A, B, C}
Pages with 4 frames: {A, B, C, D}  ← strictly adds, never swaps
```

Algorithms satisfying this are immune. Algorithms that don't are vulnerable.

**Why it's called "stack":** For LRU, you can model the cache as a stack of all referenced pages, most recent on top. *n* frames = top *n* items. Adding a frame just keeps one more item — the top *n* never change.

---

## Which Algorithms Are Affected?

| Algorithm | Stack Property | Vulnerable? |
|---|---|---|
| FIFO | No | Yes |
| Second Chance | No (FIFO-based) | Yes |
| Random Replacement | No | Yes |
| LRU | Yes | No |
| LFU | Yes | No |
| Optimal (OPT) | Yes | No |
| Clock Sweep | LRU approximation | No (in practice) |

> [!note] Random is also vulnerable
> Random replacement has no consistent ranking — more frames doesn't guarantee fewer faults since you can get unlucky with a larger cache.

---

## Key Takeaways

- Bélády's anomaly is a **property of algorithm families**, not unique to FIFO
- The dividing line is the **stack property** — nested page sets = immune
- FIFO gets all the attention because it's simple and the anomaly was first demonstrated on it
- **LRU and LFU are both immune** — adding frames strictly expands the kept set
- This is why real systems (PostgreSQL's buffer pool, OS kernels) use clock sweep or LRU approximations, never FIFO

---

## Related Notes
- [[PostgreSQL Internals]] — circular buffers and PostgreSQL's clock sweep strategy
- [[Database Architecture]] — the buffer manager's place within the storage engine layer
