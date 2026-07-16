# L4 and L7 Load Balancers

#distributed-systems #networking #load-balancing #envoy #kubernetes

---

## OSI Context

Load balancers are named after the OSI layer they operate at:

- **L4 — Transport layer**: sees TCP/UDP packets. Knows: source IP, destination IP, source port, destination port, protocol. Knows nothing above that.
- **L7 — Application layer**: sees the actual content. Knows: HTTP method, URL path, headers, cookies, gRPC method, WebSocket frames.

The layer determines what information is available to make routing decisions, and what the LB must do to the connection.

---

## L4 Load Balancer

### What It Sees

```
IP packet:
┌──────────────────────────────────────────┐
│ src IP: 203.0.113.5                      │ ← visible
│ dst IP: 10.0.0.1 (VIP)                  │ ← visible
│ src port: 54321                          │ ← visible
│ dst port: 443                            │ ← visible
│ protocol: TCP                            │ ← visible
│ ─────────────────────────────────────── │
│ payload: [encrypted or opaque bytes]     │ ← NOT visible
└──────────────────────────────────────────┘
```

The L4 LB sees the envelope, never the letter inside.

### How It Works — DNAT

The standard mechanism is **DNAT (Destination NAT)**. The LB rewrites the destination IP of the packet in the kernel and forwards it:

```
Client → LB (VIP: 10.0.0.1:443)
                │
        pick backend (e.g. 10.0.1.5:443)
                │
        rewrite packet: dst = 10.0.1.5:443
                │
        forward to backend
```

The TCP connection is **end-to-end between client and backend**. The LB is not in the connection — it just redirects packets at the IP layer. The backend sees the client's IP directly (unless the LB also does SNAT, which hides it).

### Connection Tracking

Once a TCP connection is established, all packets in that flow must go to the same backend. The LB maintains a **flow table** (also called a connection table or conntrack table):

```
5-tuple                                  → backend
(203.0.113.5, 54321, 10.0.0.1, 443, TCP) → 10.0.1.5:443
(203.0.113.6, 61234, 10.0.0.1, 443, TCP) → 10.0.1.8:443
```

Keyed on the 5-tuple. Every packet matching an entry gets forwarded to the same backend. This is why L4 LBs are inherently connection-sticky — not by choice, but by the nature of TCP.

### Direct Server Return (DSR)

Normal DNAT: client → LB → backend → LB → client. LB is in both directions.

**DSR**: client → LB → backend → **client directly**. LB is only in the request path.

```
Normal (full NAT):                DSR:
Client ──→ LB ──→ Backend         Client ──→ LB ──→ Backend
Client ←── LB ←── Backend         Client ←────────── Backend
```

How it works: the LB rewrites the **destination MAC address** (L2), not the IP. The backend has the VIP configured as a loopback alias (`lo:0`). From the client's perspective the response comes from the VIP — the backend uses the VIP as the source IP in the response.

**Why it matters:** response traffic (downloads, API responses) is typically 10–100x larger than request traffic. DSR removes the LB from the high-volume return path entirely. Used in high-throughput systems (CDNs, large-scale APIs).

**Constraint:** requires L2 adjacency — backend must be on the same broadcast domain as the LB. Doesn't work across subnets without tunneling.

### Linux IPVS — Kernel-Level L4

IPVS (IP Virtual Server) is built into the Linux kernel. It implements L4 load balancing as a kernel module — far faster than iptables because it uses hash tables instead of a linear rule scan.

Modes:
- **NAT** — standard DNAT, LB in both paths
- **DR (Direct Routing)** — DSR via MAC rewrite
- **TUN (Tunneling)** — encapsulate packets in IP-in-IP, backend decapsulates

Kubernetes `kube-proxy` can run in IPVS mode (vs the default iptables mode). At large scale (thousands of Services and Pods), IPVS is significantly faster — iptables scales O(n) with rule count, IPVS scales O(1) with hash lookups.

```bash
# check kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

### PROXY Protocol — Preserving Client IP

An L4 LB doing full NAT hides the client IP — the backend only sees the LB's IP. But the application usually needs the real client IP for logging, rate limiting, geo-blocking.

L4 LBs can't add HTTP headers (they don't understand HTTP). The solution is **PROXY Protocol** — a small header prepended to the raw TCP stream before the application data:

```
PROXY TCP4 203.0.113.5 10.0.0.1 54321 443\r\n
<original TCP payload starts here>
```

The backend must support PROXY Protocol to parse this header before processing the connection. HAProxy and NGINX support it natively. AWS NLB can send it. Without PROXY Protocol support on the backend, the connection appears as garbage data.

---

## L7 Load Balancer

### What It Sees

The L7 LB **terminates the TCP connection** (and TLS). It reads the full application-layer content before making a routing decision:

```
HTTP/2 request:
  method:    POST
  path:      /api/v1/orders
  host:      api.example.com
  headers:   { Authorization: Bearer ..., X-User-Group: beta }
  body:      { "item_id": 42, "qty": 1 }
```

All of this is available for routing. The L7 LB can route on any of it.

### Two TCP Connections

The fundamental architectural difference from L4:

```
L4 LB:
  Client ←─────────────── TCP connection ───────────────→ Backend
                   (LB rewrites IP, stays in path)

L7 LB:
  Client ←── TCP conn 1 ──→ LB ←── TCP conn 2 ──→ Backend
             (terminated)          (new connection)
```

The L7 LB is a full HTTP client on the backend side and a full HTTP server on the client side. It parses the request, makes a routing decision, opens (or reuses) a connection to the chosen backend, and forwards the request.

This has a major consequence: **the L7 LB can maintain a persistent connection pool to backends** and reuse connections across many client requests. A thousand short-lived client connections can share ten long-lived backend connections.

### HTTP/2 and gRPC — Why L7 Matters Here

HTTP/2 multiplexes many requests over a single TCP connection. An L4 LB sees one TCP connection and routes it to one backend. All requests on that connection go to one pod — even if you have 10 replicas.

```
L4 LB + gRPC:
  Client ──── 1 TCP connection ────→ LB ──────────────→ Backend A
              (100 RPCs on it)              (all 100 RPCs land here)
              Backend B, C, D get nothing

L7 LB + gRPC:
  Client ──── 1 TCP connection ────→ LB ──── RPC 1 ──→ Backend A
                                         ├── RPC 2 ──→ Backend B
                                         ├── RPC 3 ──→ Backend C
                                         └── RPC 4 ──→ Backend D
```

The L7 LB understands HTTP/2 frames and can distribute **individual RPCs** across backends. This is why gRPC requires an L7 load balancer (Envoy, NGINX) for proper load distribution — an L4 LB gives you connection-level balancing, which collapses to "all traffic on one connection goes to one backend."

### Connection Pooling

Because the L7 LB maintains its own connections to backends, it can pool them:

```
100 client connections (HTTP/1.1, each making 10 req/s)
         │
         ▼
      L7 LB
         │
    connection pool (10 persistent connections to backend)
         │
      Backend (sees 10 connections, not 100)
```

Reduces connection overhead on backends significantly. Envoy does this by default — it maintains a configurable pool of HTTP/2 connections to each upstream cluster.

---

## Side-by-Side Comparison

| | L4 | L7 |
|---|---|---|
| OSI layer | Transport | Application |
| Sees | IP, port, protocol | HTTP headers, URL, body, gRPC method |
| TCP connection | Passes through end-to-end | Terminates and creates a new one |
| TLS | Passthrough or terminate (then it's L7) | Always terminates |
| Routing basis | IP, port, 5-tuple hash | Path, host, headers, cookies, JWT claims |
| Session stickiness | Connection-level (5-tuple) | Session-level (cookie, header) |
| Health checks | TCP connect success | HTTP 200 from `/health` endpoint |
| Client IP visibility | Native (DSR) or PROXY Protocol | X-Forwarded-For header |
| Performance | Faster — kernel-level, no parsing | Slower — parse content, two connections |
| gRPC load balancing | Connection-level only (bad) | Per-RPC (correct) |
| Examples | AWS NLB, IPVS, HAProxy (TCP mode) | AWS ALB, NGINX, Envoy, Traefik |

---

## Health Checks

### L4 Health Check — TCP Connect

```
LB → backend: SYN
Backend → LB: SYN-ACK   ← healthy
LB → backend: RST        (close immediately, just testing)

No SYN-ACK within timeout → unhealthy, remove from pool
```

Tells you: the backend's TCP stack is alive. Does not tell you: whether the application is actually working.

### L7 Health Check — HTTP

```
LB → backend: GET /health HTTP/1.1
Backend → LB: HTTP/200 OK   ← healthy

HTTP 5xx, timeout, connection refused → unhealthy
```

Tells you: the application responded correctly. A backend with a crashed DB connection pool might still accept TCP connections — the L7 health check catches this, the L4 health check doesn't.

### Passive Health Checking (Outlier Detection)

Active health checks poll on a schedule. Passive health checking watches real traffic:

Envoy's outlier detection: if backend A returns 5 consecutive 5xx responses, eject it from the pool for 30s. No separate health check needed — real request failures drive the decision. See [[Istio and Envoy#DestinationRule]].

---

## Load Balancing Algorithms

### L4 Algorithms

**5-tuple hash** — hash(src IP, src port, dst IP, dst port, protocol). Deterministic — same client always hits the same backend. No state needed in the LB. Problem: one client can't be moved to a different backend mid-connection.

**Round-robin** — cycle through backends in order. Simple. Ignores connection count — a backend with long-lived connections gets as many new connections as one with short-lived ones.

**Least connections** — route to the backend with the fewest active connections. Better for workloads with variable connection duration.

### L7 Algorithms

**Round-robin** — works at request level (not connection level). Works well for uniform request cost.

**Least requests** — route to backend with fewest in-flight requests. Better than least connections for HTTP/2 where connection count is decoupled from request count.

**Ring hash / consistent hash** — hash a request attribute (URL, user ID, session cookie) to a point on a ring. Each backend owns a range on the ring. Same attribute always hits the same backend. Used for: cache locality (CDN, Redis), stateful session routing.

**Maglev hashing** (Google) — consistent hashing with a property that when a backend is added or removed, the minimum number of existing mappings are disrupted. Used in Envoy and Google's internal infrastructure. More complex than simple ring hash but more evenly distributed.

---

## Where They Appear in Kubernetes

```
External traffic
      │
      ▼
┌─────────────────────────────────────────────────┐
│  Cloud L4 LB (AWS NLB / GCP TCP LB)             │
│  — routes TCP to NodePort on cluster nodes       │
└─────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────┐
│  Ingress Controller (NGINX / Traefik / Envoy)   │
│  — L7 LB inside the cluster                     │
│  — TLS termination, host/path routing           │
└─────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────┐
│  kube-proxy (ClusterIP Services)                │
│  — L4 LB via iptables or IPVS                   │
│  — routes to pod IPs                            │
└─────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────┐
│  Envoy sidecar (if Istio mesh)                  │
│  — L7 LB at pod level                           │
│  — per-RPC load balancing, circuit breaking     │
└─────────────────────────────────────────────────┘
      │
      ▼
    Pod
```

Three layers of load balancing stack on each other. A single gRPC call from outside the cluster passes through all four.

**kube-proxy and gRPC:** kube-proxy implements L4 load balancing for ClusterIP Services. For gRPC, this means connection-level balancing — all RPCs on one gRPC connection go to one pod. Without Istio or a client-side load balancing library, gRPC workloads in Kubernetes have uneven distribution. The fix is either Istio's Envoy sidecars (L7, per-RPC) or headless Services with client-side load balancing.

---

## TLS and L4/L7

**TLS passthrough (L4):** the LB forwards encrypted bytes without decrypting. Backend terminates TLS. The LB cannot read content, cannot add headers, cannot inspect the request. The backend holds the private key.

**TLS termination (L7):** the LB decrypts traffic, reads the HTTP content, makes routing decisions, then optionally re-encrypts to the backend (TLS re-origination) or forwards plaintext (TLS offload).

```
TLS offload:
Client ──── TLS ────→ LB ──── plaintext ────→ Backend
             (terminates here)     (no TLS internally)

TLS re-origination:
Client ──── TLS ────→ LB ──── TLS ────→ Backend
         (terminates)    (new TLS session)
```

mTLS in a service mesh (Istio) is TLS re-origination at every hop — client Envoy terminates the client's connection, establishes mTLS to the server Envoy, server Envoy terminates that and forwards plaintext to the app.

---

## Key Takeaways

- **L4** sees IP/port only, passes TCP end-to-end, routes by connection — fast, limited
- **L7** terminates TCP, reads application content, routes by request — powerful, more overhead
- L4 LB is a packet forwarder; L7 LB is a full HTTP proxy — they are architecturally different, not just different in speed
- gRPC requires L7 LB for correct per-RPC distribution; L4 LB collapses all RPCs to one backend per connection
- DSR removes the LB from the response path — critical for high-throughput systems
- PROXY Protocol is how L4 LBs pass client IP to backends without understanding HTTP
- In Kubernetes: cloud LB (L4) → Ingress (L7) → kube-proxy (L4) → Envoy sidecar (L7)
- Health checks at L7 check application health, not just TCP reachability

---

## Related Notes
- [[Kubernetes/Container and Kubernetes Networking]] — kube-proxy internals (iptables/IPVS/eBPF chain mechanics), CNI, and the pod networking layer these load balancers sit on top of
- [[Kubernetes/Istio and Envoy]] — Envoy as an L7 LB; DestinationRule circuit breaking; per-RPC gRPC balancing
- [[Distributed Systems/Sidecar Pattern]] — Envoy sidecar as the L7 LB layer inside a pod
- [[Distributed Systems/RPC and APIs]] — why gRPC needs L7 load balancing

