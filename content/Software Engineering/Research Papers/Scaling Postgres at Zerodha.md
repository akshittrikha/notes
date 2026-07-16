# Scaling 7M+ Postgres Tables at Zerodha

> **Source:** Kailash Nadh (CTO, Zerodha) — IndiaFOSS 2024
> **Slides:** https://nadh.in/files/indiafoss-2024-dungbeetle.pdf
> **GitHub:** github.com/zerodha/dungbeetle (formerly: `sql-jobber`)
> **Related:** [[Spanner]] · [[Consensus Algorithms]]

> **Who this is for:** An SDE2 who understands basic databases and web services. No prior knowledge of financial systems required.

---

## The Problem in One Sentence

Zerodha (India's largest stock broker by active users) has hundreds of billions of financial records in Postgres, and millions of users need complex reports from this data daily — especially on tax day. Naive synchronous SQL queries would collapse the database.

---

## Context: What "Reports" Mean Here

When financial platforms say "report," they mean:

- "Show me all my trades for this financial year"
- "Generate my P&L statement for the last 12 months"
- "Download my contract notes for a date range"

Each report = an SQL query. Some simple (single table, indexed date range, instant). Some brutal:
- **100+ lines of SQL**
- **10+ table JOINs**
- **Cross-database JOINs** (data lives in different systems)
- **10–30 seconds to execute**

Zerodha's database has **hundreds of billions of rows** across financial records. Scanning this to produce one user's report is inherently slow. Doing it for millions of users simultaneously is impossible.

---

## The OLTP vs OLAP Split

Before diving into the solution, it helps to understand the data access pattern dichotomy that every large organization faces:

| | OLTP (Transactional) | OLAP (Analytical) |
|---|---|---|
| **What it does** | Individual writes/reads — record a trade, update balance | Aggregate queries — "total volume by user by month" |
| **Access pattern** | Many small, fast queries | Few large, expensive queries |
| **Data freshness** | Real-time | Often tolerates slight lag |
| **Example system** | Postgres, MySQL | ClickHouse, Redshift, BigQuery |

Zerodha has both. Trades are recorded in OLTP stores. Reports aggregate across that data. The problem is that users want real-time reports from OLTP-style Postgres databases — which weren't designed for 30-second aggregate queries under high concurrent load.

Most banks solve this by:
1. Running entirely separate UI/systems for recent vs. old reports
2. Moving data to warehouses (Redshift, BigQuery, etc.) for historical queries

Zerodha instead built something different, which the talk calls **Dung Beetle**.

---

## Why the Naive Approach Fails

### The Synchronous Model

```
User clicks "Download Report"
      ↓
Browser → App Server
      ↓
App Server opens DB connection
App Server issues SQL query
App Server waits...  (10–30 seconds)
      ↓
DB returns results
      ↓
App Server → Browser
```

**Why this doesn't scale:**

1. **Connection exhaustion**: Each waiting user holds an open database connection. Postgres has a hard limit on concurrent connections (typically 100–500 on most instances). At 10,000 concurrent users waiting for 20-second queries, you need 10,000 connections. You don't have them.

2. **Database overload**: Postgres doesn't process queries in isolation. 10,000 concurrent 20-second queries don't take 20 seconds. They compete for CPU, I/O, shared buffers, lock tables. They take 200 seconds each, or they time out, or the DB crashes.

3. **Snowball effect**: Slow queries → more waiting connections → more load on DB → even slower queries → users retry → more load. Classic thundering herd.

4. **Tax day spike**: On India's tax filing deadline, millions of users all want the same class of heavy report simultaneously. A synchronous system dies immediately.

### Visualized

```
User 1 → App → [long query.....waiting.....] → DB ←─┐
User 2 → App → [long query.....waiting.....] → DB   │
User 3 → App → [long query.....waiting.....] → DB   ├── DB is overwhelmed
...                                                   │
User N → App → [long query.....waiting.....] → DB ←─┘
```

---

## The Solution: Async Queuing + Results Cache

The core insight is not novel on its own — asynchronous queuing of heavy jobs is a well-known pattern. What's clever here is the **specific implementation** of how results are stored and delivered.

### The Async Queuing Pattern (Standard)

Instead of hitting the database directly, every report request goes into a queue. A pool of workers drains the queue at a controlled rate — a rate the database can actually handle. The user waits asynchronously and retrieves the result when it's ready.

```
User 1 → [Queue] ──┐
User 2 → [Queue]   │ Workers drain queue at controlled pace
User N → [Queue] ──┼──► DB (at manageable load)
                   │       ↓
                   └── Results stored somewhere
                            ↓
                       Users retrieve results
```

Banks do this too — "your statement is being generated, come back later."

### The Dung Beetle System

Zerodha's implementation of this pattern is called **Dung Beetle** (named after a small insect that can move dung 1,000x+ its body weight — a small system that moves massive amounts of data).

It's a **single Go binary**, ~1,700 lines of code, stateless, horizontally scalable.

**Architecture:**

```
                     ┌─────────────────┐
App (any language) → │   Dung Beetle   │ → Large Slow DB (Postgres / ClickHouse)
                     │   (HTTP API)    │         (hundreds of billions of rows)
                     │                 │                  ↓
                     │   Queue mgmt    │         Results ─────────────────────┐
                     │   Traffic ctrl  │                                       │
                     └─────────────────┘                                       ↓
                             ↑                                         Results Postgres DB
                             │                                         (one fresh table
                      App polls for status                              per report)
                             ↑                                                 │
                      App reads results ─────────────────────────────────────►┘
```

**Flow step by step:**

1. **App submits a job**: HTTP POST to Dung Beetle — "run the `get_profit_entries_by_date` report for user X with these parameters"
2. **Dung Beetle responds immediately**: "Job queued, here's your job ID" — the app is not blocked
3. **Dung Beetle executes at its own pace**: Queries the large slow DB when capacity allows, controlling concurrency to protect the DB
4. **Results written to Results DB**: When the query completes, results go into a fresh Postgres table in the results database
5. **App polls Dung Beetle**: "Is job X done?" — Dung Beetle returns "pending" or "complete, here's your table"
6. **App reads from Results DB**: Direct `SELECT *` on the small results table — instant

### Report Definitions (Tasks)

Every report in Dung Beetle is a **task** — a named SQL query in a `.sql` file. Multiple tasks live in the same file, separated by metadata comments:

```sql
-- reports.sql

-- name: get_profit_summary
-- db: ledger-db
SELECT SUM(amount) AS total, entry_date FROM entries GROUP BY entry_date WHERE user_id = $1;

-- name: get_profit_entries_by_date
-- db: ledger-db
SELECT * FROM entries WHERE user_id = $1 AND timestamp > $2 AND timestamp < $3;

-- name: get_transaction_history
-- db: tx-db
SELECT * FROM tx WHERE user_id = $2;
```

Key points:
- `-- name:` sets the task name used in API calls
- `-- db:` specifies which source database to run this query against
- Parameters are positional (`$1`, `$2`, `$3`) — passed in at job-submission time
- Results can have any column names and any types — Dung Beetle introspects them at runtime

### The HTTP API (Real Format from Slides)

**Submit a job** — POST to `/tasks/{task_name}/jobs`:

```bash
$ curl localhost:6060/tasks/get_profit_entries_by_date/jobs \
  -H "Content-Type: application/json" -X POST \
  --data '{"job_id": "get_profit_user1", "args": ["user1", "2015-01-01", "2015-06-30"]}'

{
  "status": "success",
  "data": {
    "job_id": "get_profit_user1",
    "task_name": "get_profit_entries_by_date",
    "queue": "queue1",
    "eta": null,
    "retries": 0
  }
}
```

The `job_id` is app-defined — Zerodha typically uses something like `{task_name}_{user_id}` so the results table name is predictable. Once the job completes, the app does:

```sql
SELECT * FROM get_profit_user1;  -- instant, hits only the results DB
```

No polling of the large DB. No waiting. The app just checks Dung Beetle's status endpoint for completion, then hits the results DB directly.

### Group Jobs (Feature Not in Talk)

The slides mention an additional feature: **group jobs** — submit multiple jobs as a group that all complete together before being considered "done." This is useful for reports that require data from multiple source databases:

```
Group: "annual_summary_user1"
  ├── Job: get_profit_summary_user1    (runs against ledger-db)
  ├── Job: get_transaction_count_user1 (runs against tx-db)
  └── Job: get_charges_user1           (runs against fees-db)

→ All three run concurrently (or sequentially, configurable)
→ Group is "complete" when all member jobs finish
→ App gets one completion signal for the whole group
```

---

## The Innovative Part: One Table Per Result

This is the hacky, novel piece of the system that the talk focuses on.

**For every completed job, a brand new Postgres table is created in the results database.**

```
User A pulls P&L report → results.job_a8f3k  (12 columns, 450 rows)
User B pulls trade history → results.job_b2m9p  (7 columns, 2,100 rows)
User C pulls contract notes → results.job_c7x1q  (15 columns, 89 rows)
...
7 million users → 7 million tables
```

Each table has a different schema — different columns, different types — because each report type returns different data. Dung Beetle handles the type mapping automatically:

```
Source column type (Postgres/ClickHouse/etc.)  →  Results DB column type
─────────────────────────────────────────────────────────────────────────
bigint         →  bigint
text           →  text
numeric(18,4)  →  numeric(18,4)
timestamp      →  timestamp
boolean        →  boolean
...
```

This type mapping works across heterogeneous sources — pull from ClickHouse, store in Postgres; pull from MariaDB, store in Postgres. The actual Go code from the slides:

```go
for i := 0; i < len(cols); i++ {
    typ = colTypes[i].DatabaseTypeName()
    switch colTypes[i].DatabaseTypeName() {
    case "INT2", "INT4", "INT8",                              // Postgres
        "TINYINT", "SMALLINT", "INT", "MEDIUMINT", "BIGINT": // MySQL/MariaDB
        typ = "BIGINT"
    case "FLOAT4", "FLOAT8",                                  // Postgres
        "DECIMAL", "FLOAT", "DOUBLE", "NUMERIC":              // MySQL/MariaDB
        typ = "DECIMAL"
    case "TIMESTAMP", "DATETIME":                             // Postgres + MySQL
        typ = "TIMESTAMP"
    case "DATE":
        typ = "DATE"
    case "BOOLEAN":
        typ = "BOOLEAN"
    case "JSON", "JSONB":                                     // Postgres
        if s.opt.DBType != dbTypePostgres {
            typ = "TEXT"                                      // ClickHouse JSON → TEXT
        }
    case "_INT4", "_INT8", "_TEXT":                          // Postgres array types
        typ = colTypes[i].DatabaseTypeName()
    default:
        typ = "TEXT"
    }

    if nullable, ok := colTypes[i].Nullable(); ok && !nullable {
        typ += " NOT NULL"
    }
    fields[i] = fmt.Sprintf(`"%s" %s`, cols[i], typ)
}
```

This runs at job-completion time. Dung Beetle introspects the result columns from the Go database driver, maps each to a Postgres-compatible type, issues a `CREATE TABLE` DDL with that exact schema, then bulk-inserts the rows. The whole operation — schema creation + insert — is transparent to the calling app.

### Why This Is Clever

**The alternative approaches and why they're worse:**

| Approach | Problem |
|---|---|
| One shared results table with a job_id column | Schema conflicts — every report has different columns. You'd need a blob/JSON column, losing query-ability |
| Serialize results to files (CSV/Parquet) | Loses SQL-ability; you can't do `ORDER BY`, `WHERE`, pagination on a file |
| One table per report type | Still forces a fixed schema; can't handle dynamic column sets |
| Serialize to Redis/Memcached | TTL management complexity, no SQL-ability, memory pressure |

**One table per result gives you:**
- **Arbitrary schema per report**: each table has exactly the right columns for that query's output
- **SQL-ability on results**: app can do `SELECT * FROM results.job_abc123 WHERE date > '2024-01-01' ORDER BY profit DESC` — filter, sort, paginate, all on the results cache, never touching the large DB again
- **Zero coordination**: no locking, no shared state — each job writes to its own isolated table
- **Instant retrieval**: result table has hundreds or thousands of rows at most — `SELECT *` is microseconds

The app decouples completely from the large database. All downstream operations (user sorts, filters, exports) hit the lightweight results cache, not the 500-billion-row monster.

---

## The Numbers: What 7 Million Tables Looks Like

**Hardware**: Single Postgres instance on a single EC2 node
- 64 vCPUs
- 128 GB RAM
- Network-attached storage

**Stats from a random trading day (from the slides):**

| Metric | Value | Notes |
|---|---|---|
| **Tables** | ~7 million | |
| **Size on disk** | ~1 TB | Varies wildly by report type |
| **pg_attribute** | 48 GB | One row per column per table |
| **pg_class** | 9 GB | One row per table/index/sequence |
| **pg_index** | 2.5 GB | Index metadata |
| **pg_statistic** | 566 KB | Query planner statistics |
| **pg_constraint** | 128 KB | Constraint definitions |
| **Total metadata** | **60 GB** | |

And yet — **Postgres boots in 3 seconds**. Postgres does not scan all tables on startup; it loads catalog data lazily. This is a testament to Postgres's architecture.

The disk is a **2 TB AWS EBS volume**. The ~1 TB actual data + ~60 GB metadata fits comfortably, with headroom for heavy report days.

### How Postgres Handles Millions of Tables

Internally, every Postgres table is a file (or a set of files) in the data directory. 7 million tables = 7 million sets of files. Postgres tracks them via its system catalog:

| System catalog table | What it stores | Size at 7M tables |
|---|---|---|
| `pg_class` | One row per table/index/sequence | **9 GB** |
| `pg_attribute` | One row per column per table | **48 GB** |
| `pg_index` | Index metadata | **2.5 GB** |
| `pg_statistic` | Planner statistics per column | 566 KB (tiny — Zerodha doesn't ANALYZE) |
| `pg_constraint` | Constraint definitions | 128 KB (tiny — no constraints on results tables) |

The fact that `pg_statistic` is only 566 KB at 7M tables is interesting: Zerodha never runs `ANALYZE` on the results tables (no point — they're written once and read once). The query planner falls back to defaults, which is fine for `SELECT *` on a small table.

Normal applications never stress-test these catalog tables at this scale. Zerodha's setup is probably one of the most unusual production Postgres deployments in the world.

---

## The Reset Problem: You Can't DROP 7 Million Tables

Here's a problem that sounds simple but isn't: **how do you clean up?**

Results tables are ephemeral — they're a cache. Once the user has downloaded their report, the data can be discarded. You don't want 7 million tables accumulating forever.

**Naive approach: `DROP TABLE results.job_abc123`**

This doesn't work at scale. `DROP TABLE` in Postgres acquires an `AccessExclusiveLock` on the table, writes to the WAL (Write-Ahead Log), updates the system catalog, and removes the file. Doing this 7 million times overnight:
- Locks during peak cleanup would block concurrent reads
- WAL would be enormous
- The operation would take hours

**What Zerodha does instead: disk swap**

```
1. Shut down Postgres (takes seconds)
2. Detach the storage volume
3. Attach a fresh, empty storage volume
4. Start Postgres on the clean volume
```

The entire reset takes **a few seconds**. You've gone from 7 million tables and 60 GB of catalog data to a clean empty instance. No DDL, no catalog scans, no WAL writes. Just a block device swap.

This is possible because the results database is **purely ephemeral** — there's no data here that needs to persist. The actual data lives in the large source databases. The results cache is just a temporary delivery mechanism.

This is similar to how you'd restart a container from a fresh image rather than trying to undo all state changes — the reset is a blunt instrument, but it's correct and extremely fast.

---

## Traffic Control: Protecting the Source Databases

The queue isn't just a buffer — it's a throttle. Different reports have different costs:

```
Heavy reports (30s queries):
  → Separate queue
  → 10 Dung Beetle workers draining it
  → Concurrency limited to 10 in-flight queries

Light reports (100ms queries):
  → Separate queue
  → 2 Dung Beetle workers
  → Much higher throughput

Urgent reports (real-time dashboards):
  → Priority queue
  → Fast lane
```

Because Dung Beetle is stateless and horizontally scalable, you can run more workers for heavier queues without changing any code. This is the key architectural decision that makes it work at Zerodha's scale — the database never sees more concurrent queries than you explicitly configure.

---

## The Polyglot Database Problem

Zerodha has multiple databases for legitimate reasons:

- **Postgres**: primary transactional data (trades, orders, user accounts)
- **ClickHouse**: high-volume time-series data (tick data, market data, analytics)
- **MariaDB**: some legacy systems (Dung Beetle supports Postgres, MariaDB, and ClickHouse as sources)

Dung Beetle abstracts over all of them. A report task just specifies a `db` parameter:

```json
{
  "task": "get_market_data_report",
  "db": "clickhouse_prod",
  "params": ["NIFTY", "2024-04-01", "2025-03-31"]
}
```

The source database can be anything. The result is always written to the same Postgres results instance. This is what the type mapping table enables — bridging column types from any source to Postgres.

Without Dung Beetle, each application team would have to implement their own async queuing, their own type mapping, their own results storage, in whatever language they're using (Go, Python, Ruby, JavaScript). Dung Beetle centralizes all of this into a single HTTP API that any language can call.

---

## What Makes This a "Hack"

The speaker calls it a hack throughout, and it is — in the best sense. It's a solution that:

1. **Misuses Postgres as a cache**: Postgres is a durable, ACID-compliant relational database. Using it as a throwaway results cache where tables are created by the millions and deleted by disk-swap is not what it was designed for. But it works.

2. **Exploits Postgres's architecture**: The fact that Postgres boots in 3 seconds with 7 million tables, and that 60 GB of catalog metadata is manageable, is a side effect of how Postgres lazily loads catalog data. Zerodha is relying on implementation details, not documented guarantees.

3. **Replaces complexity with simplicity**: The "correct" enterprise solution to this problem would involve a message broker (Kafka/RabbitMQ), a dedicated cache (Redis), a separate results store (S3 + Parquet), and a metadata service. Dung Beetle replaces all of that with 1,700 lines of Go and two Postgres instances.

4. **Blunt force cleanup**: Detaching and swapping a disk to reset a database is not how database cleanup is supposed to work. But it's faster and more reliable than any DDL-based approach.

---

## Lessons for SDE2s

### 1. Async queuing is the default answer for any "slow operation under high load" problem

If an operation takes more than a few hundred milliseconds and you expect concurrent users, it should be async. This is not specific to databases — it applies to file processing, email sending, ML inference, payment processing.

### 2. Understand your access patterns before choosing storage

Zerodha's results are:
- Written once (when the query completes)
- Read a few times (user views/downloads report)
- Deleted in bulk (nightly reset)

This access pattern is perfect for a throwaway Postgres instance. If results needed to persist for months, a different solution would be required. Match the storage choice to the access pattern.

### 3. Separation of concerns enables independent scaling

By separating the "reporting layer" (Dung Beetle + results DB) from the "application layer" (the trading/broker apps), Zerodha can:
- Scale the reporting infrastructure without touching the apps
- Add new report types without deploying new app code
- Protect source databases from application bugs causing runaway queries

Tight coupling (application directly querying the DB) would have made this impossible.

### 4. Sometimes the right abstraction is a dumb HTTP API

Every application at Zerodha — in any language, any framework — submits jobs to Dung Beetle via a simple HTTP POST and polls via HTTP GET. This is the right interface for a shared middleware system. gRPC would work too, but the simplicity of HTTP means there are zero client libraries to maintain.

### 5. Blunt instrument resets are often better than elegant cleanup

The disk-swap approach to nightly reset is inelegant. It's also correct, fast, and zero-risk of partial failure. Elegant cleanup (DROP TABLE at scale) is harder to get right, slower, and can fail halfway. When you can afford to treat storage as ephemeral, do so.

### 6. Metadata is data

60 GB of Postgres catalog data for 7 million tables is not a bug — it's a consequence of the design. Before scaling any system to extreme counts of a resource (tables, files, queues, connections), understand the metadata cost. Postgres's `pg_attribute` is 48 GB because each column in each table needs a row. This is manageable but must be understood and planned for.

---

## Architecture Diagram (Complete)

```
                            ┌──────────────────────────────────┐
                            │        Source Databases          │
                            │                                  │
                            │  Postgres (trading_db)          │
   Financial Reports        │  ClickHouse (market_data_db)    │
   ~500 billion rows        │  MySQL (legacy_db)              │
                            └──────────────┬───────────────────┘
                                           │ Dung Beetle queries here
                                           │ at controlled rate
                                           │
┌──────────────┐  HTTP POST  ┌────────────▼─────────────────────┐
│   Zerodha    │────────────►│           Dung Beetle            │
│     Apps     │             │                                  │
│  (any lang)  │◄────────────│  - Receives job submissions      │
└──────────────┘  "queued"   │  - Manages multiple queues       │
       │                     │  - Controls concurrency per DB   │
       │ poll status         │  - Distributed (run N copies)    │
       │                     │  - ~1,700 lines of Go            │
       ▼                     └────────────────┬─────────────────┘
  "complete,                                  │ writes results
  table: job_x"                               ▼
       │               ┌──────────────────────────────────────────┐
       │               │           Results Postgres               │
       │               │                                          │
       └──────────────►│  results.job_a8f3k  (450 rows, 12 cols) │
   SELECT *            │  results.job_b2m9p  (2100 rows, 7 cols) │
   instant ✓           │  results.job_c7x1q  (89 rows, 15 cols)  │
                       │  ... 7 million tables ...               │
                       │                                          │
                       │  Single EC2: 64 vCPU, 128 GB RAM        │
                       │  pg_attribute: 48 GB                    │
                       │  Total metadata: ~60 GB                 │
                       │  Boot time: 3 seconds                   │
                       │                                          │
                       │  ⟳ Nightly reset: detach disk,         │
                       │    attach fresh disk, restart Postgres   │
                       └──────────────────────────────────────────┘
```

---

## Quick Reference

| Concept | Detail |
|---|---|
| **System name** | Dung Beetle (formerly: `sql-jobber`) |
| **GitHub** | github.com/zerodha/dungbeetle |
| **Language** | Go |
| **Lines of code** | ~1,700 |
| **Interface** | HTTP API on port 6060 |
| **Source DBs** | Postgres, MariaDB, ClickHouse |
| **Results DB** | Postgres |
| **Tables per day (avg)** | ~7 million |
| **Data on disk (avg day)** | ~1 TB |
| **Results DB hardware** | Single EC2, 64 vCPU, 128 GB RAM, 2 TB EBS |
| **pg_attribute** | 48 GB |
| **pg_class** | 9 GB |
| **pg_index** | 2.5 GB |
| **Total metadata** | ~60 GB |
| **Postgres boot time** | ~3 seconds |
| **Nightly reset method** | Stop Postgres → detach EBS → attach fresh EBS → start Postgres |
| **Report definition** | Named SQL tasks in `.sql` files with `-- name:` / `-- db:` headers |
| **Traffic control** | Separate named queues per job class, N workers per queue |
| **Type mapping** | Go switch on `DatabaseTypeName()` → Postgres DDL type |
| **Group jobs** | Multiple jobs submitted as a group, single completion signal |

---

## Sources

- Kailash Nadh (CTO, Zerodha) — "Scaling 7M+ Postgres tables at Zerodha," HasGeek
- Zerodha Tech Blog — engineering.zerodha.com
- PostgreSQL Documentation — System Catalogs (pg_class, pg_attribute)
- "Designing Data-Intensive Applications" (Kleppmann) — Chapter 11: Stream Processing (async queue patterns)
- PostgreSQL MVCC and catalog architecture internals
