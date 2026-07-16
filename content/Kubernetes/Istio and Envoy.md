# Istio and Envoy

#kubernetes #service-mesh #istio #envoy #distributed-systems

---

## What This Stack Is

**Envoy** — a high-performance L4/L7 proxy written in C++, originally built at Lyft. It is the data plane: it sits in the request path and actually moves traffic.

**Istio** — a service mesh control plane that manages a fleet of Envoy proxies. It does not move traffic itself. It tells Envoy how to move traffic, rotate certificates, enforce policy, and emit telemetry.

```
Istio (control plane)     ← configures
       │
       ▼
Envoy sidecars (data plane) ← actually handle all traffic
```

Every concept in Istio is either about configuring what Envoy does (traffic management, security policy) or consuming what Envoy produces (metrics, traces, access logs).

---

## Envoy — Core Architecture

### The Four Primitive Concepts

Envoy models traffic through four objects, always processed in this order:

```
Inbound request
      │
      ▼
┌─────────────┐
│  Listener   │  ← "what port/address to accept connections on"
│  (LDS)      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Route     │  ← "given this request, which cluster handles it?"
│   (RDS)     │     matches on: path, headers, method, weight
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Cluster   │  ← "a named upstream service"
│   (CDS)     │     has: load balancing policy, circuit breaker config
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Endpoint   │  ← "the actual IP:port of a pod"
│  (EDS)      │
└─────────────┘
```

**Listener** — a socket Envoy binds. Defined by address + port. A single Envoy instance can have many listeners (one for inbound traffic, one for outbound, one for the admin interface).

**Route** — the routing table for a listener. Matches request attributes (path prefix, exact path, headers, query params) and forwards to a cluster. Can also: redirect, rewrite paths, inject headers, set timeouts, configure retries.

**Cluster** — a named group of endpoints that Envoy load balances across. Has: load balancing algorithm (round-robin, least-request, ring-hash), health check config, circuit breaker thresholds, TLS config for upstream connections.

**Endpoint** — the resolved IP:port of a backend instance. In Kubernetes, each pod IP + container port is an endpoint. Updated dynamically as pods come and go.

### Filter Chains

Envoy processes traffic through a chain of filters. Each filter can inspect, modify, or terminate the request.

```
Request → [filter 1] → [filter 2] → [filter 3] → upstream
                          │
                    can reject here
                    (rate limit, authz)
```

Key filters:
- `envoy.filters.network.http_connection_manager` — HTTP/1.1 and HTTP/2 parsing
- `envoy.filters.http.router` — the actual forwarding to a cluster (always last in HTTP filter chain)
- `envoy.filters.http.jwt_authn` — JWT verification
- `envoy.filters.http.ext_authz` — delegate authz decision to an external gRPC service
- `envoy.filters.http.ratelimit` — call the ratelimit service
- `envoy.filters.http.lua` — inline Lua scripts for custom logic
- `envoy.filters.http.grpc_json_transcoder` — REST↔gRPC translation

### Admin Interface

Envoy exposes an admin HTTP interface at `localhost:9901` (by default, never exposed externally):

```bash
curl localhost:9901/clusters          # all upstream clusters + health status
curl localhost:9901/config_dump       # full xDS config Envoy has received
curl localhost:9901/stats             # all metrics as plaintext
curl localhost:9901/listeners         # all listeners
curl localhost:9901/ready             # returns 200 when Envoy is ready
```

`/config_dump` is the most useful debugging endpoint — it shows exactly what config Istiod pushed to this Envoy instance, including routes, clusters, certificates.

---

## xDS — How Envoy Gets Its Config

xDS (x Discovery Service) is the protocol Envoy uses to receive config from a control plane. It is a set of gRPC streaming APIs. Envoy opens a long-lived gRPC connection to the control plane (Istiod) and receives updates as a stream — no polling, no restarts.

| API | Full name | What it delivers |
|---|---|---|
| **LDS** | Listener Discovery Service | Listener definitions (what ports to bind) |
| **RDS** | Route Discovery Service | Route tables (HTTP routing rules) |
| **CDS** | Cluster Discovery Service | Cluster definitions (upstream services) |
| **EDS** | Endpoint Discovery Service | Pod IPs for each cluster |
| **SDS** | Secret Discovery Service | TLS certificates and private keys |

When a new pod starts, Istiod pushes an EDS update to all Envoy instances that route to that service. The new pod starts receiving traffic within seconds — no config file change, no restart.

When a `VirtualService` (Istio CRD) is applied, Istiod translates it into Envoy route config and pushes an RDS update to affected sidecars.

> [!note] Envoy in Istio never reads a file for its main config
> In standalone Envoy you write `envoy.yaml`. In Istio, Envoy starts with a bootstrap config that just points it at Istiod's xDS endpoint. Everything else — listeners, routes, clusters, certs — arrives via xDS streaming. The bootstrap config is injected via a ConfigMap, but it's tiny (just the Istiod address).

---

## Istio Architecture

### Control Plane — Istiod

Before Istio 1.5, the control plane was three separate processes: Pilot (config), Citadel (certs), Galley (validation). Istio 1.5 merged them into a single binary: **Istiod**.

```
Istiod
├── Pilot        ← watches K8s API, translates Service/Endpoint/CRDs into xDS
├── Citadel      ← certificate authority, issues SPIFFE SVIDs to each workload
└── Galley       ← validates Istio CRD config before it's accepted
```

**Pilot** watches the Kubernetes API server for changes to Services, Endpoints, Pods, and Istio CRDs. When anything changes, it recomputes the xDS config and streams updates to affected Envoy sidecars.

**Citadel** is a full X.509 certificate authority. It issues a certificate to every pod in the mesh via SDS. The certificate encodes the pod's SPIFFE identity (`spiffe://cluster.local/ns/production/sa/my-app-sa`). These certs are used for mTLS between sidecars.

### Data Plane — Envoy Sidecars

Every pod in the mesh has an Envoy sidecar injected via the Mutating Admission Webhook. The injection adds:

1. An **init container** that sets iptables rules to redirect all pod traffic through Envoy
2. The **Envoy sidecar container** itself

```
Pod network namespace after init container:
  All inbound traffic  → port 15006 (Envoy inbound listener)
  All outbound traffic → port 15001 (Envoy outbound listener)
  Envoy's own traffic  → exempt from redirect (via UID 1337 rule)
```

The UID exemption is how Envoy avoids intercepting its own traffic — it runs as UID 1337, and iptables rules exempt that UID from redirection.

### Full Request Flow

```
Pod A (App + Envoy)                      Pod B (App + Envoy)
┌────────────────────────┐               ┌────────────────────────┐
│ App A                  │               │ App B :8080            │
│  └─ HTTP GET /orders   │               │   ▲                    │
│      (to order-svc)    │               │   │ localhost          │
│          │             │               │   │                    │
│       iptables         │               │ Envoy (inbound)        │
│          │             │               │   verify mTLS cert     │
│          ▼             │               │   check AuthzPolicy    │
│ Envoy (outbound)       │               │   emit access log      │
│   lookup order-svc     │               │   forward to :8080     │
│   via EDS → Pod B IP   │               └────────────────────────┘
│   establish mTLS       │                          ▲
│   add trace headers    │                          │
│   apply retry policy   │──── mTLS over network ──┘
│   emit metrics         │
└────────────────────────┘
```

App A makes a plain HTTP call to `http://order-service`. It never establishes a TLS connection — Envoy handles that. App B receives a plain HTTP request on localhost — it never sees the TLS handshake. mTLS is entirely in the sidecar layer.

---

## Istio CRDs — Traffic Management

### VirtualService

Defines routing rules for traffic destined for a service. The most powerful traffic management primitive.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service          # applies to traffic going to this host
  http:
    - match:
        - headers:
            x-user-group:
              exact: "beta"  # beta users get v2
      route:
        - destination:
            host: order-service
            subset: v2

    - route:                 # everyone else gets v1
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10         # 10% canary to v2
```

**Other things VirtualService can do:**
- **Fault injection** — inject artificial delays or HTTP errors for chaos testing
- **Retries** — retry on 5xx, with attempt count and per-try timeout
- **Timeouts** — global timeout for the route
- **Mirror** — copy traffic to a second destination (dark launch testing)
- **Path rewriting** — rewrite URI before forwarding
- **Header manipulation** — add/remove/modify request and response headers

### DestinationRule

Defines policy for traffic *after* routing — how Envoy connects to the upstream. Works alongside VirtualService: VirtualService says *where* to send traffic, DestinationRule says *how*.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN          # override default round-robin
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        pendingRequests: 100      # circuit breaker: queue limit
    outlierDetection:             # circuit breaker: eject failing pods
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50      # never eject more than 50% of pods
  subsets:
    - name: v1
      labels:
        version: v1               # selects pods with this label
    - name: v2
      labels:
        version: v2
```

**`outlierDetection` is Istio's circuit breaker.** If a pod returns 5 consecutive 5xx errors within a 30s window, Envoy ejects it from the load balancing pool for at least 30s. The pod is still running — it just stops receiving traffic. After the ejection period, Envoy probes it again.

### Gateway

Controls ingress and egress at the mesh boundary. An Istio Gateway configures the standalone Envoy proxy running as the ingress controller (not a sidecar — a dedicated gateway pod).

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  selector:
    istio: ingressgateway        # targets the gateway pod, not sidecars
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: main-tls  # K8s Secret with cert
      hosts:
        - "api.example.com"
```

Pair with a VirtualService that binds to this Gateway to route inbound traffic to services inside the mesh.

### ServiceEntry

Registers an external service (outside the cluster) in the mesh's service registry. Without this, Envoy treats external calls as pass-through with no policy applied.

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: stripe-api
spec:
  hosts:
    - api.stripe.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

After this, you can attach a DestinationRule to `api.stripe.com` — set timeouts, retries, circuit breaking on the external call — as if it were an internal service.

---

## Istio CRDs — Security

### PeerAuthentication

Controls mTLS policy for a namespace or specific workload.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT        # reject any plaintext connection into this namespace
```

| Mode | Behaviour |
|---|---|
| `PERMISSIVE` | Accept both mTLS and plaintext — migration mode |
| `STRICT` | Reject plaintext — all callers must present a valid cert |
| `DISABLE` | No mTLS — plaintext only |

Start with `PERMISSIVE` when onboarding a namespace to the mesh. Switch to `STRICT` once all callers have sidecars.

### AuthorizationPolicy

L7 access control — which services can call which, on which paths, with which methods.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/checkout-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/orders", "/orders/*"]
```

This policy says: only the `checkout-service` ServiceAccount may call `order-service`, only on `GET`/`POST` to `/orders*`. Everything else is denied. The principal is the SPIFFE identity from the mTLS certificate — not an IP address (which can be spoofed), not a header (which can be forged).

---

## SPIFFE — Service Identity

Istio uses the **SPIFFE** (Secure Production Identity Framework for Everyone) standard for service identity.

Every pod gets a SPIFFE identity encoded in its mTLS certificate:

```
spiffe://cluster.local/ns/<namespace>/sa/<service-account>
```

This identity:
- Is cryptographically verifiable — it's in the X.509 cert, signed by Istiod's CA
- Cannot be spoofed by a compromised pod — it would need the CA's private key
- Is stable across pod restarts — tied to the ServiceAccount, not the pod IP
- Is what AuthorizationPolicy rules check against

Istio's Citadel (inside Istiod) issues these certs via SDS. Envoy requests a cert at startup, and Citadel rotates it before expiry — no manual cert management.

---

## Observability — What Envoy Emits

Every Envoy sidecar emits three signals automatically, without app changes.

### Metrics (Prometheus)

Envoy exposes thousands of metrics at `localhost:15090/stats/prometheus`. Key ones:

```
# Request rate
istio_requests_total{
  source_workload="checkout",
  destination_service="order-service",
  response_code="200"
}

# Latency histogram
istio_request_duration_milliseconds_bucket{
  destination_service="order-service",
  le="100"
}

# Circuit breaker ejections
envoy_cluster_outlier_detection_ejections_active{
  cluster_name="outbound|80||order-service.production.svc.cluster.local"
}
```

Prometheus scrapes these via pod annotations. No instrumentation in app code required.

### Distributed Tracing

Envoy propagates trace context headers (B3 or W3C `traceparent`) on every request. It creates spans for each hop automatically.

> [!important] The one thing the app must do
> Envoy generates the span for the inbound request and the span for the outbound call. But it cannot correlate them without the trace headers being passed through the app. The app must forward `x-b3-traceid`, `x-b3-spanid`, `x-request-id` (and W3C equivalents) from incoming request to outgoing request. Without this, traces appear as disconnected single-hop spans.

### Access Logs

Envoy writes a structured access log entry for every request:

```json
{
  "start_time": "2026-07-12T10:00:00.000Z",
  "method": "POST",
  "path": "/orders",
  "protocol": "HTTP/2",
  "response_code": 201,
  "duration": 42,
  "upstream_cluster": "outbound|8080||order-service.production.svc.cluster.local",
  "upstream_host": "10.0.1.42:8080",
  "bytes_sent": 312,
  "bytes_received": 128
}
```

Available at `kubectl logs <pod> -c istio-proxy`. Contains upstream host IP — useful for debugging which specific pod a request landed on.

---

## Debugging Istio in Practice

```bash
# Check what config Istiod has pushed to a sidecar
istioctl proxy-config all <pod-name> -n production

# Check routes Envoy knows about
istioctl proxy-config routes <pod-name> -n production

# Check clusters (upstream services)
istioctl proxy-config clusters <pod-name> -n production

# Check which endpoints are healthy for a service
istioctl proxy-config endpoints <pod-name> -n production \
  --cluster "outbound|8080||order-service.production.svc.cluster.local"

# Analyse config for misconfigurations
istioctl analyze -n production

# Check if mTLS is working between two services
istioctl authn tls-check <pod-name> order-service.production.svc.cluster.local
```

`istioctl proxy-config` reaches into the Envoy admin interface (`/config_dump`) of the target pod and formats the output. It is the primary debugging tool — if traffic is being routed wrong, the answer is always in `proxy-config routes` or `proxy-config clusters`.

---

## Key Takeaways

- Envoy is the proxy (data plane); Istio is the config system (control plane) — they are separate concerns
- Envoy's four primitives: Listener → Route → Cluster → Endpoint — traffic flows through all four
- xDS (LDS/RDS/CDS/EDS/SDS) is the streaming gRPC protocol Istiod uses to configure Envoy — no file reads, no restarts
- Istiod = Pilot (config) + Citadel (certs) + Galley (validation), merged in 1.5
- **VirtualService** = where to send traffic (routing, canary, fault injection)
- **DestinationRule** = how to send traffic (circuit breaking, load balancing, mTLS mode)
- **AuthorizationPolicy** checks SPIFFE identity from the mTLS cert — cryptographically verifiable, not spoofable
- The only thing apps must do for tracing: forward trace headers from inbound to outbound requests
- `istioctl proxy-config` and `/config_dump` are the ground truth for debugging

---

## Related Notes
- [[Container and Kubernetes Networking]] — iptables pod-traffic redirection, CNI, and kube-proxy — the networking layer Envoy's sidecar interception sits on top of
- [[Kubernetes Primitives]] — Pod, ServiceAccount, ConfigMap — the K8s layer Istio runs on top of
- [[Distributed Systems/Sidecar Pattern]] — Envoy as the canonical sidecar; iptables injection; mesh-wide deployment
- [[Distributed Systems/Delivery Semantics]] — Istio retries and circuit breaking implement at-least-once and failure isolation at the proxy layer
