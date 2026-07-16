# Ambassador Pattern

#distributed-systems #architecture #kubernetes #sidecar #proxy

---

## What Is It

The Ambassador pattern deploys a **helper process co-located with your application** that acts as a proxy for all **outbound** traffic. The application talks to localhost as if there's a local version of the remote service. The ambassador handles everything else: routing, auth, retries, sharding, protocol translation.

```
Without Ambassador:
  App → knows about shards, auth tokens, retries, service discovery
        (application logic tangled with infrastructure)

With Ambassador:
  App → localhost:6379     Ambassador → Redis shard A (keys 0–5000)
                                      → Redis shard B (keys 5001–10000)
        (app sees one Redis)           (ambassador knows the routing)
```

The application stays simple. The ambassador is the infrastructure expert.

> [!note] Ambassador vs Sidecar
> These two patterns are often confused because both co-locate a helper in the same Pod.
> - **Sidecar**: augments or observes your application. Often handles **inbound** traffic (TLS termination, metrics collection, log shipping).
> - **Ambassador**: proxies **outbound** requests. Your app calls out through it to reach external services.
> In practice, Envoy does both — Istio uses it as a sidecar that intercepts both inbound and outbound. The distinction is conceptual: it tells you *why* the helper is there.

---

## The Co-location Contract

Like the sidecar, the ambassador lives in the same Pod. This gives it:

| Shared resource | What it enables |
|---|---|
| `localhost` network | App calls `localhost:PORT`, no DNS resolution, no auth needed to reach the ambassador |
| Process namespace (optional) | Ambassador can observe the app's process table |
| Volumes | Shared config files, Unix domain sockets (lower latency than TCP loopback) |

The ambassador listens on a **well-known localhost port**. The app is configured to point at that port instead of the real remote. Everything is transparent to the application.

```
Pod (shared network namespace)
┌────────────────────────────────────────────────┐
│                                                │
│  App container          Ambassador container   │
│  ┌──────────────┐       ┌───────────────────┐  │
│  │              │       │                   │  │
│  │  redis.Get() │──────▶│ :6379 (local)     │  │
│  │  host=localhost       │                   │  │
│  │  port=6379   │       │  shard routing    │  │
│  └──────────────┘       │  retry logic      │  │
│                         │  auth injection   │  │
│                         └────────┬──────────┘  │
└──────────────────────────────────│─────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
              Redis shard A               Redis shard B
              (keys 0–8191)               (keys 8192–16383)
```

---

## Use Case 1 — Sharding a Service

### The Problem

Your Redis (or Memcached, or Cassandra) cluster is sharded. The sharding logic is: `hash(key) % num_shards → shard_index`. Every client that talks to Redis must implement this logic. Now you have sharding logic in your Python service, your Go service, your Node.js service, and any new service you write.

When the shard count changes, you update every client. When you want to change the hashing algorithm, you update every client. This is untenable at scale.

### The Ambassador Solution

Deploy a **Redis proxy** as an ambassador. Every app talks to `localhost:6379`. The ambassador:

1. Receives the command (`GET user:1234`)
2. Hashes the key (`CRC16("user:1234") % 16384 = slot 5298`)
3. Looks up which shard owns slot 5298
4. Forwards to `redis-shard-a:6379`
5. Returns the response to the app

The app knows nothing about shards. The ambassador owns all sharding logic.

**Real tool: [Envoy Redis proxy filter](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_protocols/redis)**

```yaml
# Envoy filter chain for Redis protocol sharding
- filters:
  - name: envoy.filters.network.redis_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy
      stat_prefix: egress_redis
      settings:
        op_timeout: 5s
      prefix_routes:
        catch_all_route:
          cluster: redis_cluster
```

**Real tool: Twemproxy (Twitter's proxy), Codis (Chinese Redis proxy)**

Both were built exactly for this use case — shard-aware Redis/Memcached proxy that the application treats as a single Redis instance.

### Why Not Client-Side Sharding

Client-side sharding libraries exist (e.g., `redis-py`'s `RedisCluster`). They work. But:

- Every language needs its own client library implementation
- Library updates require redeploying every service
- A bug in the client library is a bug in every service simultaneously
- Adding auth, TLS, metrics, or changing topology requires updating every client

The ambassador centralizes all of that. Language-agnostic. Deployable independently.

---

## Use Case 2 — Service Brokering

### The Problem

Your application needs to connect to several external services: a managed database, a cloud storage bucket, a third-party payment API. Each has its own:

- Authentication mechanism (API key, OAuth token, mTLS certificate)
- Connection pool configuration
- Retry policy
- Circuit breaker settings

Without an ambassador, every service you write has to implement all of that. More importantly, **secrets live inside the application container** — which makes them harder to rotate, audit, and secure.

### The Ambassador Solution

The ambassador is a **local service broker** — it holds all credentials and connection config. The app calls `localhost:8080/storage/object` and gets back an object from S3. It never sees the AWS credentials, never manages a connection pool to S3, never implements retry-with-exponential-backoff for S3 throttling.

```
App                    Ambassador                    External
────                   ─────────                    ────────
PUT /storage/img.png → translate to S3 PutObject  → S3 (with AWS SigV4 auth)
                        (signs request with
                         IRSA credentials the
                         app never sees)

GET /db/query        → pool connection #3          → RDS Postgres
                        (connection pooling,         (pg credentials
                         prepared statements,         from Secrets Manager)
                         query logging)
```

**Real implementation: Dapr (Distributed Application Runtime)**

Dapr is exactly this pattern at scale. Your app calls `http://localhost:3500/v1.0/state/statestore/mykey`. Dapr's sidecar (ambassador role) translates that to Redis, Cosmos DB, DynamoDB, or any configured state store. The app is agnostic to the underlying store.

```
App → localhost:3500/v1.0/state/redis/user:1234
Dapr ambassador → actual Redis cluster
                  (handles auth, retries, serialization)
```

This also makes service brokering **auditable**: all external calls go through one process you can log and trace centrally, without instrumenting every application.

**Service Mesh + Egress Ambassador**

In Istio, an `EgressGateway` acts as an ambassador for calls leaving the mesh. All pods route external traffic through it. The gateway enforces policy (which services can call which external endpoints), does TLS origination, and provides a single audit point.

```
Pod (app) → Envoy sidecar → EgressGateway (ambassador) → External API
                              (policy enforcement,
                               TLS origination,
                               audit logs)
```

---

## Use Case 3 — Experimentation via Request Splitting

### The Problem

You have a new version of a downstream service (or a different backend entirely) and you want to test it against real traffic without impacting users. Options:

- **Canary deployment**: route X% of users to new version. If it fails, those users are affected.
- **Shadow traffic / dark launch**: send a copy of real traffic to the new version in parallel. New version's responses are discarded. Users see responses from old version only.
- **A/B split**: split traffic by user segment, compare outcomes.

Implementing any of this in the application requires the application to know about multiple backends, implement splitting logic, and handle divergent responses. That's infrastructure code inside business logic.

### The Ambassador Solution

The ambassador intercepts outbound calls and handles the splitting transparently:

```
App → localhost:8080/api/recommend    (one call, app doesn't know about split)
         │
         Ambassador
         │
         ├──── 100% → v1.recommendation-service:8080   (real response, returned to app)
         │
         └──── 100% → v2.recommendation-service:8080   (shadow call, response discarded)
                       (async, doesn't block the real response)
```

The v2 service sees identical real-world traffic. You can compare its latency, error rate, and response correctness against v1 without any user impact.

**Envoy's mirror filter does exactly this:**

```yaml
routes:
- match:
    prefix: "/api/recommend"
  route:
    cluster: recommendation_v1
    request_mirror_policies:
    - cluster: recommendation_v2
      runtime_fraction:
        default_value:
          numerator: 100          # mirror 100% of traffic
          denominator: HUNDRED
```

`recommendation_v1` gets the real request and its response goes back to the app. `recommendation_v2` gets an async copy — its response is thrown away. This is **zero user impact testing**.

### Traffic Splitting for Canary

Different from mirroring — here you actually route a percentage of requests to the new version and return those responses to users:

```yaml
# Istio VirtualService — 10% of traffic to v2
spec:
  http:
  - route:
    - destination:
        host: recommendation-service
        subset: v1
      weight: 90
    - destination:
        host: recommendation-service
        subset: v2
      weight: 10
```

This is the Istio service mesh doing the ambassador role at the cluster level. Per-pod Envoy sidecars implement the split locally for each pod's outbound traffic.

### Header-Based Splitting for A/B Testing

Route by request attribute — e.g., internal users, beta group, specific user IDs:

```yaml
http:
- match:
  - headers:
      x-user-group:
        exact: "beta"
  route:
  - destination:
      host: recommendation-service
      subset: v2
- route:
  - destination:
      host: recommendation-service
      subset: v1
```

Beta users hit v2. Everyone else hits v1. The ambassador (Envoy) reads the header and makes the routing decision. The application made one call to `recommendation-service` — it doesn't know or care which version answered.

---

## Practical Wiring in Kubernetes

The ambassador runs as a second container in the same Pod:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: REDIS_HOST
      value: "localhost"    # points at ambassador, not real Redis
    - name: REDIS_PORT
      value: "6379"

  - name: redis-ambassador
    image: envoyproxy/envoy:v1.29.0
    args: ["-c", "/etc/envoy/envoy.yaml"]
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy

  volumes:
  - name: envoy-config
    configMap:
      name: redis-ambassador-config
```

The app container's `REDIS_HOST=localhost` is the only change. All Redis logic moves to the Envoy config in the ConfigMap.

> [!important] Lifecycle Coupling
> Ambassador and app share the Pod lifecycle — they start and stop together. If the ambassador crashes, the app loses its connection to the external service. Run the ambassador with a `restartPolicy` and consider a **startup probe** on the app container that waits for the ambassador to be ready on its local port before the app starts.
> ```yaml
> startupProbe:
>   tcpSocket:
>     port: 6379      # ambassador's local port
>   initialDelaySeconds: 2
>   periodSeconds: 3
>   failureThreshold: 10
> ```

---

## Ambassador vs Other Patterns

| | Ambassador | Sidecar | API Gateway |
|---|---|---|---|
| Traffic direction | Outbound (egress) | Inbound or both | Inbound only |
| Scope | Per-pod | Per-pod | Cluster-wide |
| Who it helps | App calling external services | App receiving external calls | All clients of a service |
| Deployment | Same Pod | Same Pod | Separate service |
| Examples | Envoy egress filter, Dapr, Twemproxy | Envoy, Linkerd, Prometheus reloader | AWS API Gateway, Kong, NGINX Ingress |
| Latency | ~1ms (localhost) | ~1ms (localhost) | Network hop (5–50ms) |

---

## When to Use Ambassador

Use it when:
- Your application needs to talk to a **sharded or clustered** backend and you don't want sharding logic in every client
- You have **multiple services in different languages** that all need the same egress behavior (auth, retries, TLS)
- You want to do **dark launching or canary testing** without modifying application code
- You need **centralized egress auditing** — all external calls go through one process that can log them
- You're wrapping a legacy service that can't be modified but needs new egress behavior

Avoid it when:
- The overhead of a second container (memory, init latency) outweighs the benefit — simple services making one or two external calls with stable requirements don't need it
- The logic genuinely belongs in the client library and you have a single language across your stack

---

## Key Takeaways

- The ambassador is a **localhost proxy for outbound traffic** — the app calls localhost, the ambassador calls the world
- It decouples **infrastructure concerns** (sharding, auth, retries, routing) from **business logic** (what you're actually computing)
- The three canonical use cases: **sharding** (route to the right shard transparently), **service brokering** (hold credentials and connection pools so the app doesn't have to), **request splitting** (canary, shadow, A/B without application changes)
- Envoy, Dapr, Twemproxy, and Codis are real-world ambassador implementations
- In Kubernetes: same Pod, second container, app points `localhost:PORT` at the ambassador

---

## Related Notes
- [[Distributed Systems/Sidecar Pattern]] — co-location model, Pod namespaces, pause container; ambassador is a specialization of single-node patterns
- [[Kubernetes/Istio and Envoy]] — Envoy as the ambassador implementation; VirtualService for traffic splitting; EgressGateway for cluster-level egress
- [[Kubernetes/Kubernetes Primitives]] — Pod spec, ConfigMaps for ambassador config, startup probes
- [[Distributed Systems/L4 and L7 Load Balancers]] — the ambassador operates as an L7 proxy; traffic splitting is L7 routing
- [[Distributed Systems/Delivery Semantics]] — ambassador retry logic must account for at-least-once delivery and idempotency
