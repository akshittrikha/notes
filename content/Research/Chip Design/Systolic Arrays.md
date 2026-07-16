 # Systolic Arrays

*Section 3 of [[Chip Design Summary]] — BTech first-year level explanation*

---

## The Problem: Matrix Multiplication is Everything in AI

Almost all AI workloads reduce to multiplying two matrices:

```
A (m×k)  ×  B (k×n)  =  C (m×n)
```

Each element of C is a **dot product** — multiply pairs of numbers, sum them up. That's the MAC (Multiply-Accumulate) operation. For large matrices (4096×4096), you're doing billions of MACs.

The naive approach — fetch numbers from memory, compute, write back, repeat — runs into a brutal bottleneck:

| Operation | Energy Cost |
|---|---|
| 32-bit FP addition | ~0.9 pJ |
| Read from 32KB SRAM | ~5 pJ |
| Read from DRAM | ~640 pJ |

**Moving data costs 700× more energy than the math itself.** If your chip constantly fetches from memory, most energy goes to fetching, not computing.

---

## The Core Idea: Reuse Data, Don't Re-fetch It

A systolic array preloads the **weight matrix** (matrix B) locally into the array, then streams the **input matrix** (matrix A) through it.

- Each element of B sits in one Processing Element (PE) and **never moves**
- Each element of A **flows through** multiple PEs, accumulating partial results as it goes

This is called **weight-stationary dataflow**.

---

## The Physical Structure

A 2D grid of **Processing Elements (PEs)**. Each PE can:
1. Multiply two numbers
2. Add the result to a running sum
3. Pass data to its neighbor

```
Input →  [PE] → [PE] → [PE]
          ↓       ↓       ↓
         [PE] → [PE] → [PE]
          ↓       ↓       ↓
         [PE] → [PE] → [PE]
                              → Output (partial sums flow down)
```

- **Weights** are preloaded into each PE before computation
- **Inputs (activations)** stream in from the left, one row per cycle
- **Partial sums** accumulate as they flow downward
- After k cycles, the bottom row holds the final output matrix C

---

## Why This Is Efficient: The Quadratic Argument

For an **x×y** array:
- Total MACs performed = **x × y** (scales quadratically with size)
- Data fed in from outside = **x** values per cycle (scales linearly)

Doubling the array size **quadruples** the compute but only **doubles** the data you need to feed in. You get more math per byte of memory access — this ratio is called **arithmetic intensity**.

Google TPU v1 uses a **256×256 array = 65,536 PEs**, delivering ~92 TOPS at INT8. All of that from feeding data along just one edge.

---

## The "Systolic" Part: Clock-Locked Timing

"Systolic" means rhythmic — like a heartbeat. Every clock cycle:

1. Each PE reads its left-neighbor's output and top-neighbor's partial sum
2. Computes: `sum += input × weight`
3. Passes input rightward, passes partial sum downward

Everything moves in lockstep. No PE ever waits for another. This makes the array:
- **Branch-free** — no conditional logic, no mispredictions
- **Fully deterministic** — compiler can predict timing exactly
- **Pipeline-friendly** — you can clock it aggressively (see [[Clock Cycles and Pipeline Registers]])

---

## Dataflow Variants

| Dataflow | What stays fixed | What moves |
|---|---|---|
| Weight-stationary | Weights (B) | Inputs + partial sums |
| Output-stationary | Partial sums (C) | Inputs + weights |
| Row-stationary (MIT Eyeriss) | One row at a time | Minimizes all movement |

The optimal choice depends on model layer shapes and on-chip SRAM budget.

---

## Scratchpad vs Cache — Why TPUs Don't Use Caches

CPU caches are automatic — hardware decides what stays in fast memory. But that decision burns die area and can cause unpredictable stalls (cache miss = stall).

TPUs use **scratchpad memory** — the compiler explicitly says "load this tile of matrix B into on-chip SRAM *now*, then start the systolic array." Timing is 100% deterministic because access patterns for matrix multiply are fully known ahead of time.

**Double-buffering**: load the next tile while computing on the current one — overlaps compute and memory transfer, hiding latency entirely.

---

## Key Trade-offs

| Decision | Gain | Cost |
|---|---|---|
| Large array | Fewer mux overheads per MAC | Less flexibility for small ops |
| Weight-stationary | Weight reuse is free | Input/output still need bandwidth |
| Scratchpad over cache | Deterministic, area-efficient | Compiler must handle orchestration |
| Lower precision (INT8 vs FP32) | 4× more MACs in same area (quadratic) | Risk of accuracy loss |

---

## One-Line Mental Model

A systolic array is a grid of simple multipliers that pass data like an assembly line — weights are loaded once and reused thousands of times, eliminating the memory bottleneck that would otherwise dominate compute cost.
