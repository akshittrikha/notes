# Sidecar Pattern

#distributed-systems #kubernetes #service-mesh #single-node-patterns #istio

---

## Context — Single-Node Patterns

Before distributed system patterns (multi-node coordination, consensus, etc.), there is a class of patterns operating within a single node — multiple processes or containers co-located on the same machine, sharing resources and cooperating.

Three canonical single-node patterns:

| Pattern | Role | Concern |
|---|---|---|
| **Sidecar** | Augment / extend the main container | Adds capability the app doesn't have |
| **Ambassador** | Proxy outbound traffic | Translates/routes calls the app makes |
| **Adapter** | Normalize outbound interface | Standardizes what the app exposes |

The sidecar is the most important. The ambassador and adapter are specialisations of the same co-location idea.

---

## What the Sidecar Pattern Is

A **sidecar** is a secondary process or container that runs alongside the main application on the same node, sharing its local resources — network, filesystem, sometimes process namespace.

```
┌─────────────────────────────────┐
│           Single Node           │
│                                 │
│  ┌──────────────┐               │
│  │  Main App    │               │
│  │  (business   │◄──────────────┼── external traffic
│  │   logic)     │               │
│  └──────┬───────┘               │
│         │  shared               │
│         │  localhost / volume   │
│  ┌──────▼───────┐               │
│  │   Sidecar    │               │
│  │  (cross-     │               │
│  │   cutting    │               │
│  │   concern)   │               │
│  └──────────────┘               │
└─────────────────────────────────┘
```

The sidecar handles concerns the application shouldn't need to know about: TLS, observability, config sync, secret rotation, traffic management.

---

## Why Co-Location Matters

The sidecar is not just "another service." The physical co-location on the same node gives it properties a remote service cannot have.

**Shared network namespace (localhost)**
Sidecar and main app communicate over `127.0.0.1`. No DNS lookup, no TLS overhead, no extra network hop. Sub-millisecond latency. The sidecar can bind to a localhost port and intercept the app's traffic transparently.

**Shared filesystem via volume mounts**
Both containers mount the same volume. The app writes logs to `/var/log/app`; the sidecar reads from the same path and ships them. No network protocol needed between them.

**Same lifecycle**
In Kubernetes, containers in a Pod start and stop together. If the node dies, both die. If the pod is rescheduled, both move. The sidecar is guaranteed to be present whenever the app is present.

**Process isolation**
Despite co-location, the sidecar is a separate process. A crash in the sidecar does not crash the main app (though it may affect its functionality). Resources (CPU/memory) can be limited independently.

---

## Kubernetes — The Pod as Atomic Container Group

The Pod is Kubernetes' concrete implementation of the single-node pattern. It is the **smallest deployable unit** — not a container, but a *group* of containers that must always co-locate and co-schedule.

The word "atomic" here means the scheduler treats the group as one unit. You never schedule "container A on node 1 and container B on node 2." The pod lands on one node, or it doesn't land at all.

### What the Pod Actually Is — The Pause Container

Every pod, before any of your containers start, creates a hidden **pause container** (also called the infra container):

```
Pod
├── pause container          ← created first, never stops
│    holds: network namespace
│           IPC namespace
│           UTS namespace (hostname)
│
├── init containers (ordered)
│    └── run to completion before app starts
│
├── sidecar containers (K8s 1.29+ native)
│    └── start before app, stop after app
│
└── main app containers
```

The pause container's only job is to hold the Linux namespaces for the pod's lifetime. All other containers are attached to those namespaces. This is why:

- If the pause container dies, the entire pod is destroyed (namespaces disappear)
- Containers can join the pod at different times and still share the same network/IPC
- The pod's IP address is stable — it belongs to the pause container's network namespace, not any individual app container

### Shared Namespaces — What "Co-Location" Actually Means at the Kernel Level

Linux containers are isolated via kernel namespaces. Containers in the same Pod **intentionally share** certain namespaces:

| Namespace | Shared in Pod? | Effect |
|---|---|---|
| **Network** (`net`) | Yes, always | Same IP, same port space, `localhost` is shared |
| **IPC** | Yes, always | Shared memory segments, semaphores, message queues |
| **UTS** | Yes, always | Same hostname |
| **PID** | Optional (`shareProcessNamespace: true`) | Containers can see each other's processes |
| **Mount** (`mnt`) | No — each container has its own | Filesystem is isolated unless volumes are shared |
| **User** | No | UIDs are independent |

The network namespace sharing is the most important. From the kernel's perspective, all containers in a pod run as processes attached to the same network interface. `127.0.0.1` is genuinely local — not a loopback to another host, not another network hop. It's the same interface.

```
Node kernel
└── Network namespace (owned by pause container)
     ├── eth0 (pod IP: 10.0.0.42)
     ├── lo   (loopback: 127.0.0.1)
     │
     ├── Main app process   → binds :8080
     └── Sidecar process    → binds :9090, intercepts :8080 via iptables
```

Both processes see the same `eth0`, the same `lo`. A sidecar listening on `:9090` can be reached at `localhost:9090` from the main app with zero network overhead.

### Volume Sharing — Filesystem Bridge Between Containers

Containers in a pod do not share a filesystem by default (separate mount namespaces). You bridge them explicitly with volumes:

```yaml
spec:
  containers:
    - name: main-app
      image: my-service:v2.1
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app

    - name: log-shipper
      image: fluentd:v1.16
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app    # same path in both containers' mount namespaces

  volumes:
    - name: log-vol
      emptyDir: {}                   # lives on the node, dies with the pod
```

`emptyDir` is the standard sidecar communication volume — it lives on the node's local disk (or tmpfs if `medium: Memory`), is created when the pod starts, and deleted when the pod dies. Both containers mount it at whatever path they choose — they're looking at the same underlying directory.

**`tmpfs` for secrets** — `emptyDir: { medium: Memory }` creates an in-memory filesystem. Vault Agent writes secrets here; they never touch the node's disk and disappear when the pod is evicted.

### Init Containers — Ordered Startup

Init containers run **sequentially** before any app container starts. Each must exit 0 before the next starts. Used for:

- Seeding a shared volume with config or assets
- Waiting for a dependency to be ready (`until curl svc; do sleep 2; done`)
- Running DB migrations exactly once before the app can accept traffic
- Setting up iptables rules (how Istio's injection works)

```yaml
initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']

  - name: run-migrations
    image: my-service:v2.1
    command: ['./migrate', '--up']

containers:
  - name: main-app
    image: my-service:v2.1
```

The app container does not start until `wait-for-db` exits 0 and `run-migrations` exits 0, in that order.

### Native Sidecar Containers (Kubernetes 1.29+)

Before 1.29, the only option was to put sidecars in `containers[]` alongside the main app — no guaranteed startup order, no guaranteed shutdown order. This caused a real problem for **Jobs**:

```
Job pod completes:
  main container exits 0  → job done
  log-shipper container   → still running (hasn't flushed final logs)
  Pod termination:         → SIGTERM to all containers simultaneously
  log-shipper gets killed  → last N log lines lost
```

Kubernetes 1.29 introduced native sidecars: `initContainers` with `restartPolicy: Always`.

```yaml
initContainers:
  - name: log-shipper
    image: fluentd:v1.16
    restartPolicy: Always          # this makes it a native sidecar

containers:
  - name: main-app
    image: my-service:v2.1
```

Native sidecar lifecycle:

```
Pod start:
  1. pause container created
  2. log-shipper starts (init container with restartPolicy: Always)
  3. log-shipper signals ready
  4. main-app starts

Pod running:
  log-shipper: restarted on crash (unlike regular init containers)
  main-app:    runs normally

Pod termination:
  1. main-app receives SIGTERM → exits
  2. log-shipper receives SIGTERM → flushes buffer → exits
  3. Pod terminates cleanly
```

The key guarantee: **sidecar stops after the main container**. Flush-on-exit is now reliable.

### Resource Limits and Pod QoS Class

Each container in a pod has its own resource requests and limits. The pod's **QoS class** is derived from them and affects eviction priority under memory pressure:

| QoS Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has `requests == limits` set | Last to be evicted |
| **Burstable** | At least one container has requests < limits | Middle |
| **BestEffort** | No container has requests or limits | First to be evicted |

A common mistake: the main app has Guaranteed QoS but the sidecar has no limits set → the entire pod is demoted to BestEffort because one container is BestEffort. Set resource limits on every container in the pod, including sidecars.

```yaml
containers:
  - name: main-app
    resources:
      requests: { cpu: "500m", memory: "256Mi" }
      limits:   { cpu: "500m", memory: "256Mi" }   # Guaranteed

  - name: log-shipper
    resources:
      requests: { cpu: "50m",  memory: "64Mi"  }
      limits:   { cpu: "50m",  memory: "64Mi"  }   # also Guaranteed → pod is Guaranteed
```

---

## Common Use Cases

### 1. Service Mesh — Traffic Interception (Istio + Envoy)

The most sophisticated sidecar deployment. Every pod gets an Envoy proxy injected automatically by Istio's mutating admission webhook. The app is completely unaware.

**How traffic interception works:**

An `init container` runs before the app starts and configures `iptables` rules on the pod's network namespace:

```
All inbound traffic  → redirect to Envoy port 15006
All outbound traffic → redirect to Envoy port 15001
```

From that point:

```
Caller ──→ [Envoy sidecar A] ──→ network ──→ [Envoy sidecar B] ──→ Main App B
             (outbound proxy)                  (inbound proxy)
```

The app thinks it's talking directly to another service. It isn't. Every call goes through two Envoy hops. Those hops handle:

- **mTLS** — automatic mutual TLS between all services; app speaks plaintext on localhost
- **Retries** — configurable retry policy with backoff
- **Circuit breaking** — eject unhealthy upstreams
- **Traffic shifting** — route 10% of traffic to v2, 90% to v1 (canary deploys)
- **Observability** — trace headers propagated, metrics emitted, access logs written

No changes to application code. The mesh is entirely in the data plane.

> [!important] iptables interception is transparent but not free
> Every request pays two extra process hops (through Envoy sidecar on each end). Latency overhead is typically 0.5–3ms per hop in Istio. At high RPS, this accumulates. eBPF-based meshes (Cilium, Istio Ambient) are the response to this overhead — see below.

### 2. Secret Injection — Vault Agent Sidecar

Vault Agent runs as a sidecar, authenticates to HashiCorp Vault using the pod's Kubernetes service account token, fetches secrets, and writes them to a shared in-memory volume (`tmpfs`). The main app reads secrets from the filesystem — never authenticates to Vault directly.

```
┌─────────────────────────────────────────────────────┐
│  Pod                                                │
│                                                     │
│  ┌────────────┐    /secrets/db-password             │
│  │  Main App  │◄────────── (tmpfs volume) ──────────┤
│  └────────────┘                                     │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │  Vault Agent (sidecar)                       │   │
│  │   1. auth with K8s SA token                  │   │
│  │   2. fetch secrets from Vault                │──→│── Vault (external)
│  │   3. write to tmpfs                          │   │
│  │   4. renew before expiry (auto-rotation)     │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Why this is better than mounting secrets as K8s Secrets:**
- Secrets are never stored in etcd (K8s Secrets are base64, not encrypted by default)
- Dynamic secrets — Vault generates short-lived DB credentials per pod
- Auto-rotation — Vault Agent renews before expiry; app never sees a stale credential
- Audit trail — every secret fetch is logged in Vault

### 3. Log Aggregation

App writes to stdout or a shared volume. Sidecar (Fluentd, Vector, Filebeat) reads, enriches with Kubernetes metadata (pod name, namespace, labels), and ships to a log backend.

Why not write directly from the app?

- Language-agnostic — same log shipper works for Go, Python, Java services
- Config changes (new log destination, new enrichment) don't require app redeployment
- Backpressure and buffering handled in the sidecar, not the app

### 4. Configuration Sync

Sidecar polls a remote config store (Consul, etcd, AWS AppConfig) and writes to a shared volume. App hot-reloads when files change (using `inotify` or a polling loop).

```
App reads: /config/feature-flags.json   ← shared volume
                                                │
Sidecar polls Consul every 30s ────────────────┘
On change: write new JSON to /config/feature-flags.json
App detects file change: reload config in memory
```

App never makes a network call to Consul. Config delivery is decoupled from the app's runtime.

### 5. Metrics Adapter (Adapter Pattern Variant)

App exposes metrics in a custom format. Sidecar translates to Prometheus format on a separate port. Prometheus scrapes the sidecar, not the app.

This is the **Adapter** single-node pattern — normalising the app's output for external consumers.

---

## Traffic Flow in a Service Mesh — Deep Dive

```
Pod A                                    Pod B
┌──────────────────────┐                ┌──────────────────────┐
│ App A                │                │ App B                │
│  └─ POST /orders ────┼──(iptables)──→ │                      │
│                      │                │ ┌─ Envoy (inbound)   │
│ Envoy (outbound)  ←──┼─(iptables)     │ │   verify mTLS      │
│  ├─ mTLS to Pod B    │                │ │   check authz      │
│  ├─ add trace header │                │ │   emit metrics     │
│  ├─ apply retry cfg  │                │ └─→ App B :8080      │
│  └─ emit metrics     │                │                      │
└──────────────────────┘                └──────────────────────┘
        │                                          │
        └──────────── mutual TLS ──────────────────┘
                  (cert from Istio CA)
```

The app on Pod A makes a plain HTTP call to `http://order-service`. iptables catches it, hands it to Envoy on port 15001. Envoy looks up the destination via xDS (control plane), establishes mTLS to Pod B's Envoy. Pod B's iptables hands it to Envoy on 15006, which verifies the client cert, checks RBAC policy, and forwards to the app on localhost:8080.

Neither app knows any of this happened.

---

## Mesh-Wide Deployment — Sidecar on Every Pod

For a service mesh, the sidecar is not opt-in per pod — it must run on **every pod in the mesh**. If some pods lack the sidecar, mTLS cannot be enforced universally, observability has gaps, and traffic policy has blind spots. It's all-or-nothing for the guarantees to hold.

### Automatic Injection via Mutating Admission Webhook

Developers do not add sidecars manually — that would be untenable at any scale. Instead, Istio registers a **Mutating Admission Webhook** with the Kubernetes API server. The webhook intercepts every pod creation request before the pod is scheduled:

```
Developer: kubectl apply -f my-deployment.yaml
                │
                ▼
        Kubernetes API server
                │
                ├── validation
                │
                ├── calls Istio Mutating Admission Webhook
                │     "here is the pod spec — modify it?"
                │
                │   Istio webhook responds:
                │     + add Envoy container
                │     + add iptables init container
                │     + set ISTIO_META_* env vars
                │
                ▼
        Pod created with sidecar injected
        (developer never touched the pod spec)
```

Enable injection at the namespace level:

```bash
kubectl label namespace production istio-injection=enabled
```

Every pod deployed into that namespace gets the sidecar automatically. Opt specific pods out:

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"   # batch jobs, debug pods, etc.
```

Platform/infra teams own the sidecar version and configuration. App teams deploy their containers and the mesh capability is simply there.

### The Overhead Math

The per-pod cost is real and scales linearly:

```
Envoy idle memory: ~50MB per pod

  100 pods  →    5 GB  (comfortable)
 1,000 pods →   50 GB  (significant — explicit capacity budget needed)
10,000 pods →  500 GB  (prohibitive without alternatives)
```

At 1000+ pods, the sidecar overhead must be factored into cluster sizing. This is not a reason to avoid the pattern — the capabilities (mTLS everywhere, circuit breaking, distributed tracing with zero app changes) are often worth it. It is a reason to set tight resource limits on Envoy and to evaluate eBPF alternatives at the high end.

### Linkerd — A Lighter Sidecar Alternative

Linkerd uses `linkerd2-proxy`, written in Rust, as its data plane sidecar instead of Envoy:

| | Istio + Envoy | Linkerd + linkerd2-proxy |
|---|---|---|
| Proxy language | C++ | Rust |
| Idle RAM per pod | ~50MB | ~10MB |
| Latency overhead | 0.5–3ms per hop | ~0.5ms per hop |
| Feature set | Extensive (L7 routing, WASM extensions) | Focused (mTLS, observability, retries) |
| Operational complexity | High | Lower |

Linkerd is the right choice when overhead is a primary concern and you don't need Istio's advanced L7 traffic management features.

### Version Management at Scale

When you need to update the sidecar (new Envoy version, security patch), you must restart all pods to pick up the new injected container. At 1000 pods, this is a rolling restart of the entire cluster — a non-trivial operation.

Strategies:
- **Rolling restart per namespace** — `kubectl rollout restart deployment -n production`
- **Canary the sidecar version** — label a subset of pods to receive the new version before rolling it to all
- **Use native sidecars (K8s 1.29+)** — the independent update story for sidecars is evolving; native sidecars may allow in-place sidecar updates without restarting the main container in future Kubernetes versions

---

## The eBPF Evolution — Beyond Sidecars

The sidecar model has a scaling ceiling. The industry response is moving the data plane into the Linux kernel via eBPF, eliminating per-pod proxy processes entirely.

```
Sidecar model:                    eBPF model:
Pod → Envoy (sidecar) → network   Pod → kernel eBPF hook → network
Pod → Envoy (sidecar) → network   Pod → kernel eBPF hook → network
Pod → Envoy (sidecar) → network   Pod → kernel eBPF hook → network
1000 pods = 1000 Envoy processes  1000 pods = 1 eBPF program per node
```

**Cilium** — CNI plugin that replaces kube-proxy with eBPF. Handles network policy, load balancing, and observability at kernel level. Cilium Service Mesh adds mTLS and L7 policy via eBPF, with no sidecar.

**Istio Ambient Mesh** (GA since Istio 1.22) — removes the per-pod sidecar entirely with a two-layer architecture:

```
Node
├── ztunnel (DaemonSet, one per node)
│    handles: mTLS, L4 policy, telemetry for all pods on this node
│    cost: one process per node, not one per pod
│
└── Waypoint proxy (one per namespace or service account, optional)
     handles: L7 policy (HTTP routing, retries, header manipulation)
     only deployed for services that need L7 features
```

The ztunnel handles the universal baseline (mTLS + observability) at near-zero per-pod overhead. The waypoint handles advanced L7 features only where needed. A cluster of 10,000 pods might have 100 ztunnel pods (one per node) and 20 waypoint proxies — instead of 10,000 Envoy sidecars.

**Trade-offs of eBPF approaches:**
- Requires Linux kernel 5.x+ — limits use on older infrastructure
- Debugging is harder — `kubectl logs <sidecar>` is gone; need eBPF-aware tooling
- Ambient Mesh is newer — less battle-tested than the sidecar model in production

The sidecar pattern remains dominant for non-mesh use cases (Vault Agent, config sync, log shipping) where eBPF doesn't apply. For service mesh specifically, eBPF is where the industry is heading.

---

## Sidecar vs DaemonSet — Choosing the Right Scope

A common architectural decision: should this concern be a sidecar (per-pod) or a DaemonSet (per-node)?

| Use case | Right pattern | Why |
|---|---|---|
| Service mesh (mTLS, traffic mgmt) | Sidecar per pod | Must be universal; needs per-pod network namespace |
| Secret injection (Vault Agent) | Sidecar per pod | Each pod needs its own credential lifecycle and tmpfs volume |
| Config sync | Sidecar per pod | Config is per-service; needs shared volume with specific pod |
| Log shipping | **DaemonSet preferred** | One collector per node reads `/var/log/containers/` for all pods |
| Node metrics (CPU, disk, network) | **DaemonSet only** | Node-level data; no per-pod context needed |
| App-level metrics (Prometheus) | In-process SDK or sidecar | Depends on whether you control the app code |

Log shipping is the interesting case. A single Fluentd or Vector DaemonSet pod per node can read from the node's `/var/log/containers/` directory, which contains stdout/stderr from every pod on that node — no per-pod sidecar needed, and the overhead scales with nodes, not pods.

---

## Practical Examples

### 1. Adding TLS to a Legacy Service

A legacy app speaks plain HTTP on port 8080. It was written before TLS was a requirement, has no maintainer, and modifying it is off the table. You need it to serve HTTPS.

**Sidecar approach — Envoy as TLS terminator:**

```
Client ──── HTTPS :443 ────→ Envoy sidecar ──── HTTP localhost:8080 ────→ Legacy App
                              (terminates TLS)        (plaintext, same pod)
```

Envoy config as a ConfigMap, mounted into the sidecar:

```yaml
static_resources:
  listeners:
    - address:
        socket_address: { address: 0.0.0.0, port_value: 443 }
      filter_chains:
        - transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              common_tls_context:
                tls_certificates:
                  - certificate_chain: { filename: /certs/tls.crt }
                    private_key:       { filename: /certs/tls.key }
          filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                route_config:
                  virtual_hosts:
                    - routes:
                        - match: { prefix: "/" }
                          route: { cluster: legacy_app }
  clusters:
    - name: legacy_app
      load_assignment:
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: 127.0.0.1, port_value: 8080 }
```

The pod spec mounts the TLS cert from a Kubernetes Secret:

```yaml
containers:
  - name: legacy-app
    image: legacy-service:v1.0
    ports:
      - containerPort: 8080

  - name: tls-proxy
    image: envoyproxy/envoy:v1.28
    ports:
      - containerPort: 443
    volumeMounts:
      - name: envoy-config
        mountPath: /etc/envoy
      - name: tls-certs
        mountPath: /certs
        readOnly: true

volumes:
  - name: tls-certs
    secret:
      secretName: legacy-service-tls   # cert-manager can auto-rotate this
```

**What you gain:**
- Legacy app is unchanged
- cert-manager can rotate the certificate in the Secret; Envoy hot-reloads it without pod restart
- mTLS downstream from the load balancer — zero app changes

### 2. Dynamic Configuration with Hot Reload

Many apps read config from disk at startup and never check for changes. Redeploying to push a config change is wasteful and slow. A config-sync sidecar solves this without touching the app.

**Architecture:**

```
Remote config store                  Pod
(AWS AppConfig / Consul / etcd)      ┌─────────────────────────────┐
           │                         │                             │
           │  poll every 30s         │  ┌────────────────────────┐ │
           └────────────────────────→│  │ config-sync (sidecar)  │ │
                                     │  │  1. fetch config       │ │
                                     │  │  2. checksum diff      │ │
                                     │  │  3. atomic write       │ │
                                     │  │     to shared volume   │ │
                                     │  └──────────┬─────────────┘ │
                                     │             │ emptyDir vol  │
                                     │  ┌──────────▼─────────────┐ │
                                     │  │ Main App               │ │
                                     │  │  inotify watch on      │ │
                                     │  │  /config/app.json      │ │
                                     │  │  → reload on change    │ │
                                     │  └────────────────────────┘ │
                                     └─────────────────────────────┘
```

The **atomic write** is critical. The sidecar must not write config mid-read by the app. Standard approach: write to a temp file in the same volume, then `rename()` — rename is atomic at the filesystem level on Linux.

```python
# config-sync sidecar pseudocode
def sync_config():
    new_config = fetch_from_remote()
    if checksum(new_config) == checksum(current_config):
        return  # no change, skip write

    # write to temp file, then atomic rename
    tmp = Path("/config/.app.json.tmp")
    tmp.write_text(new_config)
    tmp.rename("/config/app.json")   # atomic — app never sees partial write
```

The app uses `inotify` (Linux) or a polling loop to detect the file change and reload in-memory config without restarting.

**Why this is better than env vars or K8s ConfigMap volume mounts:**
- K8s ConfigMap mounts update automatically but have ~1–2 minute propagation delay via kubelet
- The sidecar can check for changes every few seconds against a source of truth the app doesn't know about (Consul, AWS AppConfig with rollout controls, etc.)
- The sidecar can validate config before writing — reject malformed JSON and alert before it reaches the app

### 3. Protocol Translation — gRPC to REST Bridge

An internal gRPC service needs to be called by a third-party webhook that only speaks REST/JSON. You cannot modify the gRPC service (it's owned by another team) and you don't want to run a standalone translation service.

**Sidecar approach — Envoy gRPC-JSON transcoder:**

```
Third-party webhook ──── POST /v1/orders (REST/JSON) ────→ Envoy sidecar
                                                                   │
                                                          transcode to gRPC
                                                                   │
                                                                   ▼
                                                         localhost:50051 (gRPC)
                                                                   │
                                                            Order service
```

Envoy's `grpc_json_transcoder` filter does this using the `.proto` descriptor file — it reads the Protobuf schema and maps REST paths to gRPC methods automatically. No custom code.

The same pattern applies in reverse: a gRPC client that needs to call a REST service can use an Envoy sidecar as a local gRPC endpoint that translates outbound calls to REST. The client speaks gRPC to localhost; the sidecar handles REST externally.

### 4. Rate Limiting and Throttling for a Legacy Service

Similar to the TLS example — the legacy app has no rate limiting, and adding it would require code changes. An Envoy sidecar can enforce rate limits on inbound traffic before a single request reaches the app.

```
Client ──→ Envoy sidecar ──→ ratelimit service (gRPC) ──→ Redis
                │                                           (token counts)
                │  allowed? yes
                ▼
           Legacy App
           (never sees throttled requests)
```

Envoy calls a remote `ratelimit` service (open source, by Envoy authors) on each request. The rate limit service checks against Redis counters (per IP, per API key, per endpoint) and returns ALLOW or DENY. Envoy enforces it — the app is completely unaware.

This is the same pattern used in front of most large-scale APIs. The rate limiting concern lives entirely in the sidecar layer.

---

## Kubernetes Wiring — Sidecar-Specific Gotchas

For full coverage of Kubernetes primitives (ConfigMap, Secret, ServiceAccount, Downward API, Projected Volume, resource limits, probes), see [[Kubernetes/Kubernetes Primitives]].

The two things that specifically burn sidecar users:

### The `subPath` ConfigMap Trap

Mounting a ConfigMap file via `subPath` freezes it at pod creation — the kubelet never updates it when the ConfigMap changes. This silently breaks hot-reload.

```yaml
# WRONG — file is frozen:
volumeMounts:
  - name: envoy-config
    mountPath: /etc/envoy/envoy.yaml
    subPath: envoy.yaml              # bind-mounted at start, never refreshed

# CORRECT — directory mount gets live updates (~60s via kubelet):
volumeMounts:
  - name: envoy-config
    mountPath: /etc/envoy            # all keys appear as files, all update live
```

### CPU Limits Cause Silent Latency, Not Crashes

Memory limits OOMKill the container — visible, clear. CPU limits throttle it via the Linux CFS — the container keeps running but slows down. An Envoy sidecar hitting its CPU limit adds latency to every proxied request. The symptom looks like a slow upstream service, not a resource issue. Check `container_cpu_cfs_throttled_periods_total` in Prometheus.

Industry practice for latency-sensitive sidecars (Envoy, Linkerd proxy): set CPU *requests*, omit CPU *limits*. Allow bursting. Set memory limits — OOMKill is acceptable; invisible latency is not.

```yaml
containers:
  - name: envoy-sidecar
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        # cpu intentionally omitted
        memory: "256Mi"
```

Also: one container with no resource config makes the entire pod **BestEffort** QoS — first evicted under memory pressure. Set resources on every container in the pod, including sidecars. See [[Kubernetes/Kubernetes Primitives#Resource Requests and Limits]].

---

## Industry-Standard Implementations

### Prometheus — config-reloader Sidecar

The **Prometheus Operator** is one of the cleanest real-world sidecar implementations. Every Prometheus pod runs with a `config-reloader` sidecar:

```
Pod
├── prometheus (main)
│    reads: /etc/prometheus/prometheus.yml
│    hot-reloads on SIGHUP or /-/reload HTTP endpoint
│
└── config-reloader (sidecar)
     watches: ConfigMap mounted at /etc/prometheus/
     on change: POST http://localhost:9090/-/reload
```

When a `ServiceMonitor` or `PrometheusRule` CRD is updated, the Prometheus Operator regenerates the Prometheus config and updates the ConfigMap. The kubelet propagates the change to the volume (~1 min). The config-reloader detects the change (via inotify on the volume directory) and sends a reload signal to Prometheus. Prometheus reloads its config in-process — no restart, no downtime.

The config-reloader is `prometheus-operator/prometheus-config-reloader`, ~5MB, does one job, does it well. This is the ideal sidecar: tiny, single-purpose, well-defined interface (HTTP reload endpoint on localhost).

---

### OpenTelemetry Collector — Observability Sidecar

The **OpenTelemetry Operator** can inject an OTel Collector as a sidecar. The app emits traces, metrics, and logs to `localhost:4317` (OTLP gRPC) or `localhost:4318` (OTLP HTTP). The collector sidecar handles everything downstream.

```
App (any language)
  └─ emit OTLP to localhost:4317

OTel Collector sidecar
  ├── receive: OTLP gRPC
  ├── process: batch, add k8s attributes, sample
  └── export: Jaeger (traces), Prometheus (metrics), Loki (logs)
```

**Why this is better than the app exporting directly:**
- The app is decoupled from the observability backend — swap Jaeger for Tempo without touching the app
- The collector handles batching, retry, and backpressure — the app just fires and forgets
- Sampling decisions can be made in the collector, not the app (tail-based sampling requires seeing the full trace)
- One collector config update (via ConfigMap) changes observability behaviour for all instances

The Operator enables this with an annotation:

```yaml
metadata:
  annotations:
    sidecar.opentelemetry.io/inject: "true"
```

Same webhook injection pattern as Istio.

---

### Datadog — DaemonSet by Default, Sidecar on Fargate

Datadog illustrates the DaemonSet vs sidecar decision perfectly.

**Standard Kubernetes (DaemonSet):**
One `datadog-agent` pod per node. All pods on the node send metrics/traces to the agent via the node's IP address or a Unix socket on the node's filesystem. The app uses the Downward API to get the node IP:

```yaml
env:
  - name: DD_AGENT_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP    # node's IP, not pod's IP
```

App sends to `DD_AGENT_HOST:8125` (DogStatsD) and `DD_AGENT_HOST:8126` (APM traces). The DaemonSet agent receives for all pods on that node.

**AWS Fargate (Sidecar mode):**
Fargate doesn't support DaemonSets — there are no accessible nodes. Datadog runs as a sidecar inside every pod:

```yaml
containers:
  - name: my-app
    env:
      - name: DD_AGENT_HOST
        value: localhost             # sidecar is on localhost
  - name: datadog-agent
    image: datadog/agent:latest
    env:
      - name: DD_API_KEY
        valueFrom:
          secretKeyRef:
            name: datadog-secret
            key: api-key
```

Same app code, different `DD_AGENT_HOST` — `status.hostIP` on EKS, `localhost` on Fargate. The sidecar pattern is the fallback when DaemonSet isn't available.

---

### Consul Connect — Sidecar for Service Mesh Without Kubernetes

Consul Connect predates Kubernetes service meshes and works in non-Kubernetes environments (bare metal, VMs, Nomad). Each service runs a local Envoy proxy as a sidecar process (not a container — a literal sidecar process managed by Consul).

```
VM or bare metal host
├── my-service     (listens on 127.0.0.1:8080)
└── consul-envoy   (sidecar process, managed by Consul agent)
     ├── inbound:  0.0.0.0:20000 → 127.0.0.1:8080 (with mTLS verification)
     └── outbound: 127.0.0.1:9191 → upstream-service (with mTLS)
```

The principle is identical to Istio — but the implementation is a process, not a container, and the control plane is Consul instead of Istiod. This shows the sidecar pattern is a general systems design idea, not a Kubernetes concept. Kubernetes just makes it easier to enforce and automate.

---

| | Benefit | Cost |
|---|---|---|
| **Separation of concerns** | App code stays focused on business logic | More containers to operate, monitor, debug |
| **Language agnostic** | One sidecar works for all languages | Version skew between app and sidecar versions |
| **Independent updates** | Update sidecar without redeploying app | Sidecar must maintain backward-compatible interface |
| **Resource isolation** | Sidecar crash doesn't crash app | Each sidecar consumes CPU + memory per pod |
| **Transparency** | App doesn't need an SDK | Harder to debug — more layers in the call path |

### When NOT to Use a Sidecar

- **High pod density with tight resource budgets** — if you run 50 pods per node and each sidecar costs 50MB, that's 2.5GB just for sidecars
- **The concern can be a DaemonSet** — if all pods on a node need the same capability (node-level log collection, node metrics), a DaemonSet is more efficient than a sidecar per pod
- **You control the app code** — if the app is in-house and the team can own the library, an in-process SDK (e.g., OpenTelemetry SDK) avoids the overhead entirely
- **Latency is extremely sensitive** — even localhost hops add time; an in-process solution is always faster

---

## Relationship to Other Patterns

**Ambassador (single-node)**
Where the sidecar augments the main container in general, the ambassador specifically proxies *outbound* connections. The app talks to `localhost:5432` (ambassador), which handles connection pooling, routing, and failover to the actual Postgres cluster. Twemproxy (Redis), PgBouncer (Postgres) are examples.

**Adapter (single-node)**
The adapter normalises the main container's *output* for external consumers. The app exposes metrics in a custom format; the adapter translates to Prometheus. The difference: sidecar adds new capability, adapter transforms existing output.

**DaemonSet**
Node-level version of the sidecar. Instead of one sidecar per pod, one DaemonSet pod runs per node and serves all pods. Efficient for node-level concerns (node metrics, kernel-level log collection). Cannot share a filesystem volume with individual pods — communicates via network or node-local socket.

---

## Key Takeaways

- Sidecar co-locates a helper process on the same node as the main app, sharing localhost and volumes
- The Pod in Kubernetes is the concrete implementation — the pause container owns the namespaces; all containers attach to it
- In a service mesh, sidecar must run on **every pod** — injection is automatic via Mutating Admission Webhook, not manual
- Envoy uses ~50MB idle RAM per pod — at 1000 pods that's 50GB; set resource limits, evaluate Linkerd (~10MB) or eBPF at scale
- Istio Ambient Mesh (GA since 1.22) eliminates per-pod sidecars: ztunnel (one per node) for mTLS, waypoint (one per namespace) for L7
- Kubernetes 1.29 native sidecars solve the Job lifecycle problem — sidecar starts before app, stops after
- Config writes from sidecar to shared volume must be atomic — write to temp file, then `rename()` to avoid partial reads
- Log shipping and node metrics belong in a DaemonSet, not a sidecar — one per node, not one per pod
- Sidecar enables retrofitting capabilities (TLS, rate limiting, protocol translation) onto legacy apps with zero code changes

---

## Related Notes
- [[Kubernetes/Kubernetes Primitives]] — ConfigMap, Secret, ServiceAccount, Downward API, Projected Volume, resource limits — the full reference
- [[Declarative vs Imperative Configuration]] — Kubernetes manifests as declarative config for sidecar injection
- [[Delivery Semantics]] — service mesh sidecars enforce delivery guarantees (retries, circuit breaking) transparently
- [[MQTT]] — contrast: MQTT broker is centralised routing; sidecar is co-located, decentralised
- [[Ambassador Pattern]] — the ambassador is the outbound-traffic specialisation of the same single-node co-location model

