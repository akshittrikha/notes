# Vector Databases

#databases #vector-db #embeddings #ann #hnsw #interview-prep

---

## Core Idea

An embedding is a dense float vector (typically 384–3072 dimensions). The query you run against embeddings is almost always:

> "Give me the **k vectors most similar** to this one" — by cosine, dot product, or L2 distance.

That is **k-Nearest-Neighbor (kNN) search**, and it is the whole reason vector databases exist.

---

## Why a Traditional Database Struggles

Traditional indexes are built for **exact match** and **range** queries:

- **B-trees** need a total ordering of keys. A 1536-dim vector has no meaningful sort order — "closeness" in high-dim space doesn't map onto a 1-D line.
- **Hash indexes** only answer equality.
- Spatial indexes (k-d trees, R-trees) degrade to near-linear scans above ~20 dimensions — the **curse of dimensionality**.

So a plain DB falls back to a **full scan**: compute distance to every row → `O(N · d)` per query. At 100M vectors × 1536 dims, that's seconds per query, not milliseconds.

> [!note] Contrast with [[Storage Engines]]
> B-tree and LSM engines optimize *how a key is found on disk*. A vector index answers a different question entirely — *which keys are near this point* — so it needs a different data structure.

---

## Why Vector DBs Win

### 1. ANN Indexes (the main reason)

They trade **exactness for speed** using Approximate Nearest Neighbor indexes — ~95–99% recall at millisecond latency.

| Index | How it works | Trade-off |
|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | Multi-layer proximity graph. Search greedily descends from sparse upper layers to the dense bottom layer. ~`O(log N)` | High recall, fast; **memory-hungry**, expensive deletes |
| **IVF** (Inverted File) | k-means clusters the space; query only scans the `nprobe` nearest clusters | Cheaper memory; recall depends on `nprobe` |
| **PQ** (Product Quantization) | Split each vector into sub-vectors, quantize each against a small codebook | 10–50× compression; some accuracy loss |
| **DiskANN** | Graph index designed to live on SSD | Billion-scale without all-in-RAM |

Tunable knobs (`ef_search`, `nprobe`) let you slide along the **recall ↔ latency** curve.

```
HNSW (simplified)

Layer 2:   A ─────────────── F            ← few nodes, long jumps
           │                 │
Layer 1:   A ──── C ──── E ── F ── H      ← more nodes
           │      │      │    │    │
Layer 0:   A─B─C─D─E─F─G─H─I─J─K─L        ← all nodes, short links
                     ↑
             greedy walk ends near query
```

### 2. Similarity Is First-Class

Distance metrics, top-k, and score thresholds are native query primitives — not something you compute in application code.

### 3. Filtered Search

"Similar docs **where** `tenant_id = X AND date > Y`" is hard:

- **Post-filter** (ANN first, then filter) → may return fewer than k results.
- **Pre-filter** (filter first, then ANN) → can break graph traversal / degrade to brute force.

Vector DBs implement **filter-aware traversal** (e.g. Qdrant, Weaviate) to do this efficiently.

### 4. Hybrid Search

Combine **dense** (semantic) vectors with **sparse** (BM25 / keyword) scoring, then merge with **Reciprocal Rank Fusion (RRF)**. Big win for RAG quality — dense search misses exact identifiers, keyword search misses paraphrases.

### 5. Scale & Operations

- Sharding and replication of vector indexes
- In-memory or on-disk index layouts
- SIMD / GPU-accelerated distance computation
- Real-time upserts without full index rebuilds

### 6. Memory Efficiency

- **Scalar quantization**: float32 → int8 = 4× smaller
- **Binary quantization**: float32 → 1 bit = 32× smaller
- Typically paired with **re-ranking** the top candidates using full-precision vectors.

---

## The Nuance — You Don't Always Need One

"Vector DB" = **vector index + database features**. Pick based on scale and operational cost:

| Option | When to use |
|---|---|
| **pgvector** ([[PostgreSQL Internals\|Postgres]]) | < ~10M vectors; want ACID, joins, one system. Supports HNSW + IVFFlat. **Often the right default.** |
| **FAISS / ScaNN** (libraries) | In-process index, no persistence/CRUD — offline or batch jobs |
| **Elasticsearch / OpenSearch** | Already running it; need hybrid keyword + vector search |
| **Dedicated** (Pinecone, Milvus, Qdrant, Weaviate) | 100M+ vectors, high QPS, heavy filtering, multi-tenancy |

> [!warning] Trade-offs to mention
> - **Approximate** — recall < 100%
> - **HNSW is memory-hungry** (graph links + raw vectors in RAM)
> - **Deletes are expensive** in graph indexes → tombstones + periodic compaction
> - **Another datastore** → dual-write / sync and consistency problems with your source of truth
> - Often **eventually consistent**

---

## 30-Second Interview Answer

> "Embedding queries are nearest-neighbor searches in high-dimensional space, where B-trees don't help, so a regular DB degrades to a linear scan. Vector databases use ANN indexes like HNSW or IVF-PQ to get sub-linear, millisecond search with tunable recall, plus metadata filtering, hybrid search, quantization, and horizontal scaling. That said, at moderate scale pgvector is often enough — I'd choose based on vector count, QPS, filtering needs, and operational overhead."

---

## Likely Follow-Ups

- [ ] How does HNSW work? (layers, `M`, `ef_construction`, `ef_search`)
- [ ] Recall vs. latency — how do you measure recall in prod? (compare against brute-force on a sample)
- [ ] Cosine vs. dot product — equivalent when vectors are **L2-normalized**
- [ ] Chunking strategy for RAG (size, overlap, semantic boundaries)
- [ ] Re-embedding when switching models — vectors from different models are **not comparable**; need a full re-index (blue/green index swap)
- [ ] Design a RAG system for 100M documents

---

## Related

- [[Storage Engines]]
- [[PostgreSQL Internals]]
