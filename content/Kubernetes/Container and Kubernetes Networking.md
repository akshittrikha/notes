# Container and Kubernetes Networking

#kubernetes #docker #networking #distributed-systems #linux

---

## The Mental Model

Everything in this note is built on one Linux primitive: **network namespaces**. A container is not a lightweight VM — it's a process with its own private slice of the kernel's network stack (its own interfaces, routing table, iptables rules, `/proc/net`). Docker and Kubernetes networking is, almost entirely, the story of how you wire many of these isolated namespaces together so processes in different namespaces can talk to each other as if they were on a normal LAN.

```
Host network namespace
├── eth0 (real NIC)
├── docker0 / cni0 (bridge)
│
├── netns "container A"  ← Docker/K8s creates this
│    └── eth0@peer  ← other end of a veth pair
│
├── netns "container B"
│    └── eth0@peer
```

If you understand namespaces + veth pairs + bridges + iptables/routing, the rest (CNI, kube-proxy, Services, overlays) is just "which component configures these primitives, and when."

---

## Linux Network Namespaces — The Foundation

A network namespace is a full copy of the network stack: interfaces, routes, iptables/nftables rules, `/proc/net/*`. Processes in different namespaces cannot see each other's sockets or interfaces at all, even on `localhost`.

```bash
# create a namespace by hand — this is literally what containerd does
ip netns add ns1
ip netns exec ns1 ip addr        # empty — only loopback, and it's down
ip netns exec ns1 ip link set lo up
```

A brand-new namespace has no connectivity to anything — not even the host. Every container needs some plumbing added to reach the outside world. That plumbing is a **veth pair**.

---

## veth Pairs and Bridges — How a Container Gets Network Access

A **veth pair** is two virtual interfaces that act like the two ends of an ethernet cable — anything sent into one comes out the other. One end stays in the host namespace, the other end is moved into the container's namespace.

```bash
ip link add veth-host type veth peer name veth-ctr
ip link set veth-ctr netns ns1        # move one end into the container

# host side
ip addr add 172.17.0.1/16 dev veth-host
ip link set veth-host up

# container side
ip netns exec ns1 ip addr add 172.17.0.2/16 dev veth-ctr
ip netns exec ns1 ip link set veth-ctr up
```

At this point the host and the one container can ping each other directly. To let **many** containers talk to each other, the host-side veth ends are all plugged into a **Linux bridge** — a kernel-level software switch (L2, works on MAC addresses, just like a physical switch).

```
docker0 (bridge, 172.17.0.1/16)
   ├── veth-host-A ↔ veth-ctr-A (in container A's netns, 172.17.0.2)
   ├── veth-host-B ↔ veth-ctr-B (in container B's netns, 172.17.0.3)
   └── veth-host-C ↔ veth-ctr-C (in container C's netns, 172.17.0.4)
```

This is exactly what `docker0` is. Every container on the default Docker bridge network gets a veth pair plugged into `docker0`, and the bridge switches frames between them at L2 — same mechanism as plugging four laptops into a physical switch.

> [!note] Kubernetes' `pause` container owns this, not your app container
> In K8s, the veth pair and IP address belong to the **pause container**'s network namespace (see [[Kubernetes Primitives#The Pod]]). Your app container and any sidecars join that same namespace with `--net=container:pause` — they share the veth, the IP, the routing table. This is *why* all containers in a pod share `localhost`.

---

## Docker Networking Modes

Docker gives you four fundamentally different ways to attach a container's namespace to the world. Knowing which one you're in explains 90% of "why can't my container reach X" questions.

| Mode | What happens | `docker run` flag |
|---|---|---|
| **bridge** (default) | veth pair + `docker0` bridge, private subnet, NAT to reach outside | (default) |
| **host** | No new namespace at all — container shares the host's network stack directly | `--network host` |
| **none** | Namespace created, but no interfaces added except loopback | `--network none` |
| **container:\<id\>** | Join an *existing* container's namespace instead of creating a new one | `--network container:X` |

### bridge mode — NAT is doing the work

A container on the default bridge (`172.17.0.0/16`) is **not directly reachable** from outside the host. Two NAT rules make it usable:

```bash
# outbound: container → internet, masquerade as host's IP
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# inbound: docker run -p 8080:80 → DNAT host:8080 to container:80
iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

`-p 8080:80` isn't magic — it's a DNAT rule in the `DOCKER` chain. `docker port <container>` just reads these rules back.

### host mode — no isolation, no NAT, no port mapping needed

The container's process binds directly to the host's network stack. `-p` flags are ignored (there's no separate namespace to map into). Fastest option — zero NAT/bridge overhead — but you lose port isolation: two containers in host mode can't both bind `:8080`.

### container:\<id\> mode — this is how pods work

`--network container:X` makes a new container join an *existing* container's namespace instead of getting its own. This is the exact mechanism Kubernetes uses to put every container in a pod on the same network namespace as the pause container.

---

## The Kubernetes Networking Model — Four Rules

Kubernetes doesn't implement pod networking itself — it *delegates* it (via CNI, below) but it does mandate a contract every network implementation must satisfy:

1. **Every pod gets its own IP** — no port-mapping tricks like Docker's `-p`, ever, for pod-to-pod traffic.
2. **Pods can reach all other pods' IPs directly, on any node, without NAT.** A pod on node A talks to a pod on node B as if they were on the same L2 segment, even though they're not.
3. **Agents on a node (kubelet) can reach all pods on that node.**
4. **Containers within a pod share an IP and port space** (this one Kubernetes *does* implement itself — the pause-container trick above).

Rule 2 is the hard one. Docker's default bridge is per-host and NAT'd; it flatly does not satisfy rule 2. Kubernetes needs a **flat, routable, cluster-wide address space** spanning every node. That's the problem CNI plugins exist to solve.

---

## CNI — How Kubernetes Delegates Network Setup

CNI (Container Network Interface) is a spec, not an implementation: a JSON-in/JSON-out executable contract between the container runtime and a network plugin.

```
kubelet creates pod
      │
      ▼
kubelet calls container runtime (containerd/CRI-O) to create the pause container
      │
      ▼
runtime creates the network namespace, then invokes the CNI plugin binary:
      $ /opt/cni/bin/calico ADD <<< '{"cniVersion":"1.0.0", "name":"k8s-pod-network", ...}'
      │
      ▼
CNI plugin:
  1. creates the veth pair
  2. moves one end into the pod's netns
  3. calls IPAM plugin to allocate an IP from the pod CIDR
  4. sets up routes so the IP is reachable cluster-wide
  5. returns the assigned IP as JSON to the runtime
      │
      ▼
kubelet now knows the pod's IP, reports it to the API server (pod.status.podIP)
```

Only one CNI plugin runs per cluster (Calico, Cilium, Flannel, AWS VPC CNI, Azure CNI, etc.) — configured via a file in `/etc/cni/net.d/` that kubelet reads. Everything about *how* pod IPs get allocated and how cross-node routing works is that plugin's problem, not kubelet's.

```bash
# see what CNI plugin a node is configured to use
cat /etc/cni/net.d/*.conflist

# see the veth end for a running pod, from the node
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
ip addr    # find the veth whose peer index matches the one inside the pod's netns
```

---

## Pod-to-Pod Networking Across Nodes

This is where CNI plugins diverge the most, and picking one is a real architectural decision, not a checkbox.

### Overlay networks (VXLAN) — Flannel, Calico in overlay mode

Wraps every pod packet inside a UDP packet addressed to the destination *node's* real IP. The underlying physical network only ever sees node-to-node traffic — it doesn't need to know anything about pod IPs.

```
Pod A (10.244.1.5, node1) → Pod B (10.244.2.7, node2)

Original packet:  [src:10.244.1.5 dst:10.244.2.7] [payload]
                          │
                  VXLAN-encapsulated on node1:
[src: node1-IP dst: node2-IP UDP:4789] [VXLAN header] [src:10.244.1.5 dst:10.244.2.7] [payload]
                          │
                  travels over the real network as node1→node2 traffic
                          │
                  node2 decapsulates, delivers original packet to Pod B's veth
```

**Pro:** works on any underlying network — no coordination with routers or cloud VPC needed. **Con:** encapsulation overhead (VXLAN header ≈ 50 bytes) — see the MTU gotcha below — and every packet costs an extra encap/decap on both ends.

### Native routing / BGP — Calico in BGP mode

No encapsulation. Each node runs a BGP speaker (`bird`, part of Calico) that advertises "I own pod CIDR `10.244.1.0/24`" to the other nodes (or to the datacenter's top-of-rack routers). Every node's kernel routing table gets a route: `10.244.1.0/24 via node1-IP`.

```bash
# on a Calico node, in BGP mode:
ip route
# 10.244.1.0/24 via 10.0.0.11 dev eth0   ← learned via BGP from node1
# 10.244.2.0/24 via 10.0.0.12 dev eth0   ← learned via BGP from node2
```

**Pro:** no encapsulation overhead, packets are native IP the whole way, easier to debug (`tcpdump` shows real pod IPs everywhere). **Con:** needs L3 reachability between nodes and, for full performance, BGP peering with physical routers (or "node-to-node mesh" BGP as a simpler fallback that doesn't need router support).

### Cloud-native ENI-based — AWS VPC CNI, GKE alias IP ranges

The cloud provider's VPC *is* the flat network. AWS VPC CNI attaches real secondary ENIs (or ENI-assigned secondary IPs) to each node and hands one out per pod — a pod's IP is a first-class routable IP inside your VPC, not an overlay address.

**Pro:** zero encapsulation, pod IPs are visible to (and reachable from) other VPC resources like RDS security groups. **Con:** you're bounded by ENI/IP limits per EC2 instance type — this is the actual reason large clusters on AWS sometimes hit "not enough IPs" pod scheduling failures that have nothing to do with CPU/memory.

### Comparison

| | Overlay (VXLAN) | BGP (native routing) | Cloud-native (ENI) |
|---|---|---|---|
| Encapsulation | Yes (~50B overhead) | No | No |
| Underlying network awareness needed | None | Router BGP peering (or node mesh) | Cloud VPC routing |
| Pod IP visible outside cluster | No | Sometimes | Yes (real VPC IP) |
| Performance | Lower (encap/decap cost) | Highest | Highest |
| Portability | Runs anywhere | Needs L3 adjacency | Cloud-specific |

---

## kube-proxy — Service Virtual IPs

A pod IP is unstable (pods die, get rescheduled, get new IPs). A `Service`'s `ClusterIP` is stable. Something has to rewrite "traffic to the stable VIP" into "traffic to whichever pod IPs are currently healthy" — that's kube-proxy's entire job. It runs as a DaemonSet, watches the API server for Service/Endpoints changes, and reprograms the node's packet-forwarding rules. It never sits in the data path itself — see [[L4 and L7 Load Balancers#Where They Appear in Kubernetes]] for where this sits relative to Ingress and Envoy.

### iptables mode (the default, historically)

For a Service `order-service` at ClusterIP `10.96.0.5:80` with 3 pod endpoints, kube-proxy writes a chain of DNAT rules:

```
KUBE-SERVICES (dst 10.96.0.5:80?)
      │
      ▼
KUBE-SVC-ORDERSVC   ← per-Service chain, does weighted random jump
      │
      ├── 33% → KUBE-SEP-POD1  → DNAT to 10.244.1.5:8080
      ├── 33% → KUBE-SEP-POD2  → DNAT to 10.244.2.7:8080
      └── 34% → KUBE-SEP-POD3  → DNAT to 10.244.3.9:8080
```

```bash
# see it for real on any node
iptables -t nat -L KUBE-SERVICES -n | head
iptables -t nat -L KUBE-SVC-<hash> -n     # per-service chain with the random jump
```

The "random" load balancing uses `iptables -m statistic --mode random --probability`. It's **not connection-aware** — each *new connection* gets an independent random roll, not each packet (existing conntrack entries keep hitting the same backend).

> [!important] iptables is O(n) — this is a real scaling wall
> Every rule is evaluated linearly, top to bottom, per packet, for the first packet of every new connection (subsequent packets hit conntrack and skip the chain). With thousands of Services, the chain gets enormous, and iptables' rule-reload-on-every-Service-change behavior becomes a measurable source of latency and control-plane CPU at scale. This is precisely the motivation for IPVS and eBPF modes — see [[L4 and L7 Load Balancers#Linux IPVS — Kernel-Level L4]].

### IPVS mode

Same job, but backed by IPVS (kernel L4 LB, hash-table based — see the linked note) instead of a linear iptables scan. O(1) lookup regardless of Service count. Supports real algorithms (round-robin, least-connection, weighted) instead of the probability-chain hack. Requires `ip_vs` kernel modules loaded on every node.

### eBPF mode — Cilium replacing kube-proxy entirely

Cilium can run with `kubeProxyReplacement: true` and skip kube-proxy altogether — an eBPF program attached at the network interface (or even the socket layer, for pod-to-Service traffic) does the VIP → pod IP rewrite before the packet ever traverses the normal netfilter/iptables path. This is now the direction most performance-sensitive clusters go: no iptables/IPVS chain at all, rewrite decisions made in-kernel at the earliest possible hook.

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode   # iptables vs ipvs
cilium status | grep KubeProxyReplacement                              # eBPF mode, if using Cilium
```

---

## Services, Endpoints, EndpointSlices

`Service.spec.selector` tells the endpoint controller which pods to track. It writes the result to an `Endpoints` object (legacy) or `EndpointSlice` (current — introduced because a single `Endpoints` object listing thousands of pods for one Service became a control-plane bottleneck: one giant object, rewritten in full on every single pod change).

```bash
kubectl get endpointslices -l kubernetes.io/service-name=order-service
```

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: order-service-x7f2k
  labels:
    kubernetes.io/service-name: order-service   # which Service this belongs to
addressType: IPv4
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true          # kube-proxy only programs rules for ready endpoints
    targetRef:
      kind: Pod
      name: order-service-7d4f-abc12
ports:
  - port: 8080
```

EndpointSlices cap at 100 endpoints each and shard automatically — a Service with 500 pods gets 5 slices, and a single pod's readiness flip only rewrites the one slice it belongs to, not a 500-entry object. This directly determines how fast a pod becoming ready/unready propagates to kube-proxy across the whole cluster.

### Headless Services — bypassing the VIP entirely

`clusterIP: None` gives you no virtual IP at all — DNS for the Service name returns the **individual pod IPs directly** (an A record per pod), instead of one VIP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None
  selector:
    app: cassandra
  ports:
    - port: 9042
```

```bash
dig cassandra.production.svc.cluster.local
# returns: 10.244.1.5, 10.244.2.7, 10.244.3.9  ← all pod IPs, no VIP
```

This is what StatefulSets use, and it's the fix mentioned in [[L4 and L7 Load Balancers#kube-proxy and gRPC]] for gRPC clients that want to do their own client-side load balancing across the real backend list instead of getting collapsed onto one pod through a single VIP connection.

---

## DNS in Kubernetes — CoreDNS

Every pod's `/etc/resolv.conf` points at the `kube-dns`/CoreDNS Service ClusterIP. CoreDNS watches Services and Endpoints via the API server and answers queries from in-memory state — no etcd round trip per query.

```
Service DNS:  <svc>.<namespace>.svc.cluster.local        → ClusterIP
Pod DNS:      <pod-ip-dashed>.<namespace>.pod.cluster.local → pod IP  (rarely used directly)
```

### The `ndots:5` gotcha — a real, common latency bug

```bash
cat /etc/resolv.conf
# search production.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

`ndots:5` means: any name with **fewer than 5 dots** gets every search-domain suffix tried *before* being queried as-is. So `api.stripe.com` (2 dots) triggers, in order:

```
api.stripe.com.production.svc.cluster.local   → NXDOMAIN
api.stripe.com.svc.cluster.local              → NXDOMAIN
api.stripe.com.cluster.local                  → NXDOMAIN
api.stripe.com                                → finally resolves
```

> [!important] External DNS calls from pods are 3-4x slower than they should be
> Every fully-qualified external hostname eats 3 failed lookups before the real one, each a network round trip to CoreDNS (and from CoreDNS out to the real resolver). At any meaningful QPS this is measurable added latency and CoreDNS load. Fixes: append a trailing dot to fully-qualified external names in code (`api.stripe.com.` — skips search-domain expansion entirely), or set `dnsConfig.options` with a lower `ndots` on pods that talk to a lot of external services.

---

## NetworkPolicy — Segmentation, Enforced by the CNI Plugin

`NetworkPolicy` is a Kubernetes API object — it does nothing by itself. It's a spec that the **CNI plugin** reads and enforces (typically via eBPF or iptables rules it manages). Flannel, for instance, implements zero NetworkPolicy enforcement — the object gets accepted by the API server and silently does nothing.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # applies to every pod in the namespace
  policyTypes: [Ingress]
  # no `ingress:` rules = deny all
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-checkout-to-orders
  namespace: production
spec:
  podSelector:
    matchLabels: { app: order-service }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: checkout-service }
      ports:
        - port: 8080
```

Compare to [[Istio and Envoy#AuthorizationPolicy]]: `NetworkPolicy` is L3/L4 (IP + port), enforced in-kernel by the CNI, and identity is "which pods match this label selector" — not cryptographically verified. `AuthorizationPolicy` is L7, checks a SPIFFE identity from an mTLS cert, and can't be spoofed by a compromised pod's IP. Defense in depth means running both: NetworkPolicy as the coarse kernel-level fence, AuthorizationPolicy as the cryptographic, app-aware gate.

---

## MTU — The Overlay Gotcha That Causes "Works for Small Requests, Hangs for Large Ones"

Standard ethernet MTU is 1500 bytes. VXLAN adds ~50 bytes of encapsulation header. If the pod's veth interface is still set to MTU 1500 but the packet gets wrapped in VXLAN before hitting the real 1500-byte-MTU physical NIC, the resulting frame is too big.

```
Pod thinks MTU = 1500
      │
sends a 1500-byte packet
      │
CNI VXLAN-encapsulates it → 1550 bytes
      │
physical NIC MTU is 1500 → either fragments (slow) or drops (if DF bit set)
```

**Symptom that makes this maddening to debug:** small requests (health checks, short API calls) work fine — they never approach 1500 bytes. Large payloads (file uploads, big JSON responses, TLS handshakes with large certs) mysteriously hang or time out. `ping` (small ICMP packets) works, ruling out "no connectivity" — but `curl` on a large POST hangs. This exact symptom signature is the fingerprint of an MTU mismatch.

**Fix:** the CNI plugin must set the pod veth MTU to `physical_MTU - encapsulation_overhead` (e.g. 1450 for VXLAN over standard ethernet), not just inherit 1500. Flannel/Calico compute this correctly *if configured with the right underlying MTU* — this breaks most often after a node's physical MTU changes (e.g., moving to a cloud provider with jumbo frames, or adding a VPN/tunnel underneath the cluster network) without updating the CNI config to match.

```bash
# check pod's interface MTU
kubectl exec my-pod -- ip link show eth0 | grep mtu

# check the physical MTU CNI thinks it should subtract from
cat /etc/cni/net.d/*.conflist | grep -i mtu
```

---

## Debugging Toolkit — Where to Actually Look

```bash
# 1. Does the pod have an IP and is the veth up?
kubectl get pod my-pod -o wide
kubectl exec my-pod -- ip addr

# 2. Is DNS resolving?
kubectl exec my-pod -- nslookup order-service.production.svc.cluster.local
kubectl exec my-pod -- cat /etc/resolv.conf     # check ndots, search domains

# 3. Is the Service pointing at healthy endpoints?
kubectl get endpointslices -l kubernetes.io/service-name=order-service
kubectl describe svc order-service              # shows resolved Endpoints inline

# 4. Are kube-proxy's rules actually programmed on the node?
iptables -t nat -L KUBE-SERVICES -n | grep order-service
ipvsadm -Ln                                      # if in IPVS mode

# 5. Is a NetworkPolicy silently dropping traffic?
kubectl get networkpolicy -n production
# then check whether the CNI actually enforces NetworkPolicy at all (Flannel: no)

# 6. Raw packet trace from inside the pod's netns (needs a debug container / nsenter)
kubectl debug my-pod -it --image=nicolaka/netshoot
tcpdump -i eth0 -n host order-service

# 7. Cross-node connectivity at the CNI layer, bypassing K8s entirely
ip route                          # BGP-learned routes, if using Calico BGP mode
bridge fdb show                   # L2 forwarding table on the node's bridge
```

The order matters: pod IP exists → DNS resolves → Service has healthy endpoints → kube-proxy rules exist → NetworkPolicy isn't blocking → raw packet trace. Each step rules out an entire layer; jumping straight to `tcpdump` without checking Endpoints first wastes time chasing a packet that was never going to be sent because there were zero ready backends.

---

## Key Takeaways

- Everything is namespaces + veth pairs + bridges + iptables/routing — Docker and Kubernetes just automate wiring them together differently
- Docker's default bridge network is per-host and NAT'd; Kubernetes mandates a flat, cluster-wide, NAT-free pod address space — that gap is exactly what CNI plugins exist to close
- CNI is a plugin contract (JSON in/out) invoked by the container runtime when a pod is created — it allocates the IP and wires the veth, nothing more, nothing less
- Cross-node pod traffic is either encapsulated (VXLAN overlay — portable, has overhead) or natively routed (BGP/cloud ENI — faster, needs network cooperation)
- kube-proxy doesn't proxy traffic — it programs iptables/IPVS/eBPF rules that rewrite Service VIP → pod IP; iptables mode is O(n) per Service and is the reason IPVS and eBPF (Cilium) modes exist
- EndpointSlices replaced the monolithic Endpoints object specifically to stop rewriting a giant object on every single pod readiness change
- `ndots:5` silently triples/quadruples DNS latency for external hostnames — a trailing dot or a `dnsConfig` override fixes it
- NetworkPolicy is enforced by the CNI plugin, not the API server — some CNIs (Flannel) accept the object and enforce nothing
- MTU mismatches in overlay networks produce the specific symptom of "small requests fine, large ones hang" — check `ip link show` MTU before anything else when you see that pattern
- Debug in layer order: pod IP → DNS → Endpoints → kube-proxy rules → NetworkPolicy → raw packet trace — each step eliminates a whole class of cause

---

## Related Notes
- [[Kubernetes Primitives]] — the Pod's pause container owns the network namespace everything here attaches to
- [[Istio and Envoy]] — Envoy sidecars and iptables traffic redirection sit on top of this networking layer; AuthorizationPolicy vs NetworkPolicy comparison
- [[L4 and L7 Load Balancers]] — kube-proxy is an L4 load balancer implementation; IPVS deep dive lives there
- [[Distributed Systems/Sidecar Pattern]] — how the iptables redirection used by service meshes interacts with pod networking
