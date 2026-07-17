# Adapter Pattern

#distributed-systems #architecture #kubernetes #observability #single-node-patterns

---

## The Core Idea

You have an application that exposes data in its own format. Some external system needs that data in a completely different format. Neither side can change — the app is a legacy service you can't touch, and the external system is a company-wide standard you can't modify.

The adapter sits between them and **translates**.

```
Without Adapter:
  App (custom format) → ??? → Prometheus (needs /metrics in its own text format)
  App (custom log)    → ??? → Splunk (needs JSON with specific fields)

With Adapter:
  App (custom format) → Adapter → Prometheus ✓
  App (custom log)    → Adapter → Splunk ✓
```

The adapter's only job: speak your app's language inward, speak the external system's language outward.

> [!note] The Three Single-Node Patterns
> All three co-locate a helper container in the same Pod. The difference is direction:
> - **Sidecar**: adds capability to the app (inbound augmentation — TLS, tracing)
> - **Ambassador**: proxies outbound calls *from* the app (your app calls the ambassador, it calls the world)
> - **Adapter**: normalizes outbound *exposure* from the app (your app exposes something, the adapter re-exposes it in a standard format)
>
> Sidecar adds. Ambassador routes out. Adapter translates out.

---

## The Problem Without an Adapter

Imagine you have five services:
- A Java service logging with Log4j (plain text, Java stack traces)
- A Python service logging with structlog (JSON, Python format)
- A Go service logging to stdout (unstructured text)
- A Node.js service logging with Winston (semi-structured)
- A legacy C++ service writing to `/var/log/app.log` (proprietary format)

Your company uses a centralized log aggregator that expects one format: JSON with specific fields (`timestamp`, `level`, `service`, `message`, `trace_id`).

**Without adapters**: you have to modify every service to produce that format. That means code changes, testing, and deployments across five different tech stacks. The C++ service might take months.

**With adapters**: each service gets a small co-located process that reads the app's native log format and re-emits it in the company standard. The services themselves don't change at all.

---

## How It Deploys

Like the sidecar and ambassador, the adapter lives in the same **Pod** — sharing localhost, process namespace, and volumes with the main container.

```
Pod (shared filesystem + network namespace)
┌───────────────────────────────────────────────────────┐
│                                                       │
│  App container              Adapter container         │
│  ┌────────────────┐        ┌──────────────────────┐   │
│  │                │        │                      │   │
│  │  writes to     │        │  reads /var/log/     │   │
│  │  /var/log/     │──────→ │  app.log             │   │
│  │  app.log       │  (shared volume)               │   │
│  │  (proprietary  │        │  translates to JSON  │   │
│  │   format)      │        │  ships to Splunk     │   │
│  └────────────────┘        └──────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

The shared volume is the key: the app writes to a file, the adapter reads from that same file. No network call between them — just a file on the shared filesystem. Low overhead, zero latency between app and adapter.

---

## Use Case 1 — Prometheus Exporters (The Canonical Example)

Prometheus scrapes metrics by calling `GET /metrics` on every service. It expects a specific text format:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200"} 1234
http_requests_total{method="POST", status="500"} 7
```

Most databases, message brokers, and infrastructure tools don't speak Prometheus. MySQL doesn't expose `/metrics`. Neither does Redis, PostgreSQL, Kafka, or Nginx in this format.

**The adapter: an exporter.** It's a small process that:
1. Talks to the target service in its native protocol (MySQL client protocol, Redis RESP, etc.)
2. Fetches whatever metrics it exposes natively (`SHOW STATUS`, `INFO`, etc.)
3. Translates them into Prometheus text format
4. Exposes `/metrics` on a local port that Prometheus can scrape

```
Prometheus
    │
    │ GET :9104/metrics
    ▼
┌───────────────────────────────────────┐  Pod
│  mysqld_exporter (:9104)              │  ← adapter
│  (connects to MySQL on localhost,     │
│   runs SHOW STATUS, translates)       │
│                                       │
│  MySQL (:3306)                        │  ← main container
└───────────────────────────────────────┘
```

```yaml
# Pod spec for MySQL + exporter adapter
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    ports:
    - containerPort: 3306

  - name: mysqld-exporter           # adapter
    image: prom/mysqld-exporter:latest
    ports:
    - containerPort: 9104
    env:
    - name: DATA_SOURCE_NAME
      value: "exporter:password@tcp(localhost:3306)/"  # localhost — same Pod
```

The exporter connects to `localhost:3306` — no DNS, no network hop. Prometheus scrapes `:9104`. MySQL never knew any of this happened.

**Exporters exist for everything**: `redis_exporter`, `postgres_exporter`, `kafka_exporter`, `nginx-prometheus-exporter`, `node_exporter` (for the OS itself). This is the most common adapter pattern in the wild.

---

## Use Case 2 — Log Format Normalization

Your app writes logs in its own format. Your company's log pipeline (Splunk, Loki, Elasticsearch) expects structured JSON with specific fields.

**Fluent Bit** is the adapter. It runs as a co-located container, reads the app's log file or stdout stream, parses it using configurable rules, and ships structured JSON to the centralized collector.

```
App stdout:
  "2026-07-17 14:32:01 ERROR Failed to connect to DB: timeout after 30s"

Fluent Bit parses and ships:
  {
    "timestamp": "2026-07-17T14:32:01Z",
    "level": "ERROR",
    "message": "Failed to connect to DB",
    "detail": "timeout after 30s",
    "service": "payment-service",
    "pod": "payment-7d9c4b-xkp2z",
    "namespace": "production"
  }
```

```yaml
spec:
  containers:
  - name: payment-service
    image: payment:v2.3.1

  - name: fluent-bit               # adapter
    image: fluent/fluent-bit:3.1
    volumeMounts:
    - name: varlog
      mountPath: /var/log

  volumes:
  - name: varlog
    emptyDir: {}
```

```ini
# fluent-bit.conf
[INPUT]
    Name  tail
    Path  /var/log/app.log
    Parser java_log           # knows how to parse Log4j format

[FILTER]
    Name  record_modifier
    Add   service payment-service

[OUTPUT]
    Name  loki
    Host  loki.monitoring.svc
    Labels service=${service}
```

The app writes whatever it wants. Fluent Bit does the translation. The centralized logging system sees perfectly normalized JSON.

---

## Use Case 3 — Health Check Normalization for Legacy Services

Kubernetes probes expect HTTP: `GET /health → 200 OK`. A legacy TCP service might have its own health check mechanism — a custom binary protocol, a specific socket response, or a health file it writes to disk.

An adapter container exposes the HTTP endpoint Kubernetes expects, and internally checks the legacy service's actual health mechanism:

```
Kubernetes kubelet:
    │ GET localhost:8080/health
    ▼
┌──────────────────────────────────────────────┐  Pod
│  health-adapter (:8080)                      │  ← adapter
│  - receives HTTP GET /health                 │
│  - opens TCP connection to legacy app :9999  │
│  - sends proprietary health ping             │
│  - if response OK → returns HTTP 200         │
│  - if response bad → returns HTTP 503        │
│                                              │
│  Legacy C++ service (:9999)                  │  ← main container
└──────────────────────────────────────────────┘
```

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080          # adapter's port, not the legacy service's port
  initialDelaySeconds: 10
  periodSeconds: 5
```

The adapter is often just 30 lines of Go or Python. It's a translation shim — nothing more.

---

## Use Case 4 — Protocol Translation

Your service speaks gRPC internally. An external partner or a legacy integration expects REST/JSON. You can't modify the service (it's shared with 10 other internal teams that all need gRPC).

The adapter translates REST → gRPC for inbound calls:

```
External client (REST)
    │ POST /v1/orders {"item": 42, "qty": 1}
    ▼
┌──────────────────────────────────────────┐  Pod
│  grpc-gateway adapter (:8080)            │  ← adapter
│  (transcodes REST → gRPC)               │
│                                          │
│  Order service (:50051, gRPC)            │  ← main container
└──────────────────────────────────────────┘
```

**grpc-gateway** is a Go library that auto-generates this adapter from Protobuf annotations:

```protobuf
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }
}
```

The annotation tells grpc-gateway: "expose this RPC as `POST /v1/orders`." It generates the translation layer. Your gRPC service doesn't need to change a line.

---

## Use Case 5 — StatsD to Prometheus (Metric Protocol Translation)

Older services (especially those written before Prometheus existed) emit metrics via **StatsD** — a UDP-based protocol where you just fire-and-forget counters:

```
# StatsD format (UDP datagrams)
http.requests:1|c           ← counter, increment by 1
response.time:24.5|ms       ← timing, 24.5ms
cache.hit:1|g               ← gauge
```

Prometheus can't scrape StatsD UDP. A `statsd_exporter` adapter bridges them:

1. Listens on UDP for StatsD datagrams from the app (same Pod, localhost)
2. Accumulates them into Prometheus metrics (counters, histograms)
3. Exposes `/metrics` for Prometheus to scrape

```
App → UDP :9125 → statsd_exporter → /metrics → Prometheus
       (StatsD)     (adapter)      (Prometheus
                                    format)
```

No changes to the app. The metrics pipeline modernizes without a rewrite.

---

## Adapter vs Sidecar — The Clearest Distinction

Both co-locate in the same Pod. Both are helpers. But their purpose is opposite in direction:

| | Sidecar | Adapter |
|---|---|---|
| Direction | Augments what comes **in** to the app, or extends the app | Normalizes what comes **out** of the app |
| Who initiates contact | External traffic comes in, sidecar intercepts it | Adapter reaches into the app (reads its files, calls its local port) |
| App perspective | Sidecar gives the app new capabilities | App is unaware the adapter exists |
| Changes app behavior | Yes (adds TLS, tracing, retries) | No (app unchanged, adapter just reads its output) |
| Examples | Envoy (TLS termination), OTel agent (tracing injection) | mysqld_exporter, Fluent Bit, health-check shim |

In practice, tools like Fluent Bit and the OTel Collector can act as both — they can enrich logs (sidecar-like behavior) AND normalize format (adapter behavior). The pattern is a mental model, not a rigid taxonomy.

---

## Practical Kubernetes Wiring

The adapter needs access to the app's output. The three ways to wire this up:

**1. Shared volume (files):** App writes to `/var/log/`, adapter reads from the same mount.

```yaml
volumes:
- name: logs
  emptyDir: {}
containers:
- name: app
  volumeMounts:
  - name: logs
    mountPath: /var/log
- name: adapter
  volumeMounts:
  - name: logs
    mountPath: /var/log   # same mount
```

**2. localhost port:** App exposes something on `localhost:PORT`, adapter calls it.

```yaml
# App exposes :8081/stats (custom format)
# Adapter reads :8081/stats, exposes :9090/metrics (Prometheus format)
```

**3. stdout via shared process namespace:** App writes to stdout, adapter reads `/proc/<pid>/fd/1`.

```yaml
spec:
  shareProcessNamespace: true   # adapter can see app's /proc
```

> [!important] Startup Order
> If the adapter needs the app to be ready before it can function, use an **init container** to wait, or configure the adapter to retry on startup. Kubernetes doesn't guarantee container start order within a Pod beyond init containers.

---

## Key Takeaways

- The adapter **normalizes your app's output interface** to match what an external consumer expects — without changing the app itself
- It's co-located in the same Pod: shared volumes and localhost make the data transfer zero-latency
- The most common real-world form: **Prometheus exporters** (mysql_exporter, redis_exporter, etc.) — every major piece of infrastructure has one
- **Fluent Bit / Fluentd** for log normalization; **grpc-gateway** for REST↔gRPC; **statsd_exporter** for metric protocol bridging
- Adapter is distinct from ambassador: ambassador proxies *outbound calls the app makes*; adapter normalizes *outbound data the app exposes*
- The app doesn't need to know the adapter exists — that's the power of the pattern

---

## Related Notes
- [[Distributed Systems/Sidecar Pattern]] — co-location mechanics (pause container, shared namespaces, volumes); adapter is a specialisation of single-node patterns
- [[Distributed Systems/Ambassador Pattern]] — the outbound counterpart; ambassador proxies calls the app makes, adapter normalizes data the app exposes
- [[Kubernetes/Istio and Envoy]] — Envoy acts as sidecar but can also do protocol translation (gRPC↔REST transcoding), blurring sidecar/adapter lines
- [[Kubernetes/Kubernetes Primitives]] — Pod spec, shared volumes, shareProcessNamespace, init containers
