# Kubernetes Primitives

#kubernetes #devops #distributed-systems #containers

---

## What Kubernetes Actually Does

Kubernetes is a container orchestration system. It schedules containers onto nodes, restarts them on failure, manages networking between them, and provides abstractions for configuration, secrets, storage, and identity.

At its core, the control loop is declarative: you describe desired state, the control plane continuously reconciles actual state to match it. See [[Declarative vs Imperative Configuration]].

---

## Node vs Pod

> [!note] The most common point of confusion
> **Node** — a physical or virtual machine. The actual computer running in your cluster. Has CPU, RAM, disk. Runs the kubelet, container runtime, and kube-proxy. You have a small number of these (e.g. 10–100).
>
> **Pod** — a group of containers scheduled *onto* a node. Ephemeral — dies and gets replaced, possibly on a different node. You have many of these (e.g. hundreds to thousands).
>
> ```
> Cluster
> ├── Node A (VM, 16 CPU, 64GB RAM)
> │    ├── Pod 1 (order-service)
> │    ├── Pod 2 (payment-service)
> │    └── Pod 3 (nginx sidecar test)
> │
> └── Node B (VM, 16 CPU, 64GB RAM)
>      ├── Pod 4 (order-service replica)
>      └── Pod 5 (auth-service)
> ```
>
> The node is infrastructure. The pod is your workload. Kubernetes decides which pod runs on which node — you usually don't.

---

## The Pod

The smallest deployable unit — not a container, but a **group of containers** that always co-locate on the same node and share certain Linux namespaces.

```
Pod
├── pause container        ← holds network + IPC + UTS namespaces
├── init containers        ← run sequentially to completion before app starts
├── sidecar containers     ← (K8s 1.29+) start before app, stop after
└── app containers         ← your workload
```

Every pod gets one IP address (belonging to the pause container's network namespace). All containers in the pod share `localhost` and the same port space.

See [[Sidecar Pattern]] for how multi-container pods are used in practice.

---

## ConfigMap

Stores non-secret configuration data as key-value pairs. Keys become filenames when mounted as a volume; values become file contents.

### Creating a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  app.json: |
    {
      "log_level": "info",
      "feature_flags": { "new_checkout": true }
    }
  database.conf: |
    host=postgres.production.svc.cluster.local
    port=5432
    pool_size=10
```

### Mounting as a Volume (live-updating)

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config

containers:
  - name: my-app
    volumeMounts:
      - name: config-vol
        mountPath: /etc/config    # each key becomes a file here
```

Files under `/etc/config/` are symlinks into a versioned directory that the kubelet atomically swaps when the ConfigMap changes. Updates propagate within **30–90 seconds** (kubelet sync interval, configurable via `--sync-frequency`, default 60s).

### The `subPath` Trap

> [!important] `subPath` mounts are frozen at pod creation — they never update
>
> ```yaml
> # WRONG for hot-reload:
> volumeMounts:
>   - name: config-vol
>     mountPath: /etc/config/app.json
>     subPath: app.json           # bind-mounted at start, never refreshed
> ```
>
> With `subPath`, Kubernetes creates a direct bind mount at pod creation time. The kubelet's ConfigMap sync mechanism does not touch bind mounts. The file is frozen until the pod restarts.
>
> **Fix:** mount the whole directory. Let the application read the file from within the directory.

### Mounting as Environment Variables

```yaml
containers:
  - name: my-app
    envFrom:
      - configMapRef:
          name: app-config      # all keys injected as env vars

    # or selectively:
    env:
      - name: LOG_LEVEL
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: log_level
```

Environment variables are **never updated** after container start — even if the ConfigMap changes. Use volume mounts for anything that needs live updates.

### ConfigMap Size Limit

ConfigMaps are stored in etcd. The hard limit is **1MB per ConfigMap**. For larger config (binary assets, large ML model configs), use an init container to pull from an object store (S3, GCS) and write to an `emptyDir` volume.

---

## Secret

Stores sensitive data — passwords, tokens, certificates. Functionally similar to ConfigMap in how it mounts, but with additional access controls and optional encryption at rest.

> [!note] Secrets are base64 encoded, not encrypted by default
> base64 is encoding, not encryption. A cluster admin can read all Secrets. Enable **Encryption at Rest** (KMS provider) to actually encrypt Secret data in etcd. Alternatively, use an external secret store (Vault, AWS Secrets Manager) with an operator that syncs into Secrets or directly into pod volumes.

### Secret Types

| Type | Use case |
|---|---|
| `Opaque` | Arbitrary key-value (default) |
| `kubernetes.io/tls` | TLS cert + key (`tls.crt`, `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Image pull credentials |
| `kubernetes.io/service-account-token` | Legacy SA tokens (avoid — use projected tokens) |
| `kubernetes.io/ssh-auth` | SSH private keys |

### Mounting a TLS Secret

```yaml
volumes:
  - name: tls-certs
    secret:
      secretName: my-service-tls
      items:                        # optional: mount only specific keys
        - key: tls.crt
          path: tls.crt
        - key: tls.key
          path: tls.key
          mode: 0400                # file permissions (octal)

containers:
  - name: envoy
    volumeMounts:
      - name: tls-certs
        mountPath: /certs
        readOnly: true
```

### cert-manager Integration

cert-manager watches `Certificate` CRDs and issues certs via ACME (Let's Encrypt), Vault PKI, or self-signed CAs. It writes the result into a Secret and rotates before expiry.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-service-tls
spec:
  secretName: my-service-tls       # cert-manager writes here
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - my-service.example.com
  renewBefore: 720h                # renew 30 days before expiry
```

When cert-manager updates the Secret, the kubelet propagates the new cert to mounted volumes. Applications (e.g., Envoy) detect the file change and reload.

---

## ServiceAccount

Every pod runs as a ServiceAccount — its in-cluster identity. ServiceAccounts are used for:
- RBAC: what the pod is allowed to do with the Kubernetes API
- Workload identity: how the pod proves its identity to external systems (Vault, AWS IRSA, GCP Workload Identity)

### Default Behaviour

A pod that doesn't specify `serviceAccountName` uses the `default` SA in its namespace. The default SA is auto-mounted into every pod:

```
/var/run/secrets/kubernetes.io/serviceaccount/
  token      ← legacy, non-expiring SA token (avoid in production)
  ca.crt     ← cluster CA cert
  namespace  ← pod's namespace as a string
```

Disable automounting if the pod doesn't need API access:

```yaml
spec:
  automountServiceAccountToken: false
```

### Projected Service Account Tokens (OIDC)

The legacy token (`/var/run/secrets/kubernetes.io/serviceaccount/token`) never expires and has no audience binding — a leaked token is valid forever for any purpose.

**Projected tokens** are the modern replacement:

```yaml
volumes:
  - name: vault-token
    projected:
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600      # kubelet rotates before expiry
            audience: vault              # only Vault accepts this token
```

The kubelet automatically rotates the token before expiry and writes the new token to the volume. The application (or Vault Agent sidecar) re-reads the file on each use — it's always fresh.

This is the basis for **AWS IRSA** (IAM Roles for Service Accounts) and **GCP Workload Identity** — cloud providers validate the projected token against the cluster's OIDC endpoint to issue cloud credentials.

### RBAC — What the ServiceAccount Can Do

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production

---
# Role: what's allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding: bind SA to Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-configmap-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

`ClusterRole` + `ClusterRoleBinding` for cluster-wide permissions. `Role` + `RoleBinding` for namespace-scoped permissions. Prefer namespace-scoped — least privilege.

---

## Projected Volume

Combines multiple volume sources into a single `mountPath`. Avoids multiple volume mounts when a container needs config + secret + SA token + pod metadata together.

```yaml
volumes:
  - name: workload-identity
    projected:
      defaultMode: 0444
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600
            audience: my-service
        - configMap:
            name: app-config
            items:
              - key: config.json
                path: config.json
        - secret:
            name: app-tls
            items:
              - key: tls.crt
                path: tls.crt
        - downwardAPI:
            items:
              - path: pod-name
                fieldRef:
                  fieldPath: metadata.name
```

All four sources appear as files under the single mountPath. Vault Agent uses this pattern extensively.

---

## Downward API

Injects information about the pod itself into containers — without the app needing to call the Kubernetes API.

### As Environment Variables

```yaml
containers:
  - name: my-app
    env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: MY_CPU_LIMIT
        valueFrom:
          resourceFieldRef:
            containerName: my-app
            resource: limits.cpu
      - name: MY_MEM_REQUEST
        valueFrom:
          resourceFieldRef:
            containerName: my-app
            resource: requests.memory
```

Environment variables are set at container start and never updated. Good for stable values (name, namespace, node name).

### As a Volume (for labels and annotations)

Labels and annotations can be updated after pod creation. The only way to see live changes is via a Downward API volume:

```yaml
volumes:
  - name: pod-info
    downwardAPI:
      items:
        - path: labels
          fieldRef:
            fieldPath: metadata.labels      # updates when labels change
        - path: annotations
          fieldRef:
            fieldPath: metadata.annotations
```

Files in this volume are updated within ~30–60s when labels or annotations change. Useful for sidecars that need to react to label changes (e.g., a log shipper that changes log level based on a `log-level: debug` label added at runtime).

---

## Resource Requests and Limits

Every container should have both set. They serve different purposes.

```yaml
resources:
  requests:
    cpu: "100m"       # 0.1 core — what the scheduler reserves on the node
    memory: "128Mi"   # what the scheduler checks against available node memory
  limits:
    cpu: "500m"       # 0.5 core — CFS throttles above this
    memory: "256Mi"   # OOMKill above this
```

### CPU — Requests vs Limits Behaviour

**Request:** the scheduler uses this to find a node with enough available CPU. It's a reservation, not a guarantee — the container can use more if the node is idle.

**Limit:** enforced by the Linux CFS (Completely Fair Scheduler) as a quota. The container is throttled — not killed — when it exceeds this. Throttling manifests as latency, not crashes.

> [!important] CPU throttling is silent and causes latency spikes
> A container hitting its CPU limit gets its time slices cut short. From the outside it looks like slow upstream responses or increased tail latency. `kubectl top` doesn't show throttling — check `container_cpu_cfs_throttled_periods_total` in Prometheus.

**Common practice for latency-sensitive containers (Envoy sidecar):** set requests, omit limits for CPU. Allow bursting. Set memory limits (OOMKill is acceptable; invisible latency is not).

### Pod QoS Class

Derived from the containers' requests/limits. Affects eviction order under node memory pressure.

| Class | Condition | Evicted |
|---|---|---|
| **Guaranteed** | Every container: `requests == limits` for both CPU and memory | Last |
| **Burstable** | At least one container: requests set, limits > requests | Middle |
| **BestEffort** | No container has any requests or limits | First |

> [!important] One BestEffort container demotes the whole pod
> If your main app is Guaranteed but the sidecar has no resource config, the pod is BestEffort. Set requests + limits on every container, including sidecars.

---

## Init Containers

Run sequentially to completion before any app container starts. Each must exit 0 before the next begins. If one fails, Kubernetes restarts it (per the pod's `restartPolicy`).

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command:
      - sh
      - -c
      - until nc -z postgres.production.svc.cluster.local 5432; do sleep 2; done

  - name: run-migrations
    image: my-app:v2.1
    command: ["/app/migrate", "--up"]
    env:
      - name: DB_URL
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: url

containers:
  - name: my-app
    image: my-app:v2.1
```

**Use cases:**
- Wait for dependencies (DB, cache, upstream service) to be ready before app starts
- Run DB migrations exactly once per deploy
- Seed a shared volume with data the app needs
- Set up iptables rules (Istio's injection init container)

Init containers share the same volumes as app containers — a common pattern is seeding an `emptyDir` volume with data that the app reads on startup.

---

## Liveness, Readiness, and Startup Probes

Three distinct health signals Kubernetes uses for different purposes.

| Probe | Failure action | Purpose |
|---|---|---|
| **Startup** | Restart container | "Is the app done initialising?" — disables liveness until it passes |
| **Liveness** | Restart container | "Is the app alive?" — restart if stuck |
| **Readiness** | Remove from Service endpoints | "Is the app ready for traffic?" — take out of load balancer rotation |

```yaml
containers:
  - name: my-app
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30      # 30 × 10s = 5 minutes for slow startup
      periodSeconds: 10

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0    # startup probe guards this
      periodSeconds: 15
      failureThreshold: 3       # restart after 3 consecutive failures

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 1       # remove from load balancer after 1 failure
```

**Liveness vs Readiness:** a pod can be alive but not ready (e.g., loading a large model into memory). During that time, Kubernetes stops sending it traffic (readiness failure) but does not restart it (liveness is passing). This is the correct model for warm-up.

**Common mistake:** using the same endpoint for both probes. Liveness should only return failure if the process is truly stuck or corrupt (deadlock, infinite loop). Readiness should return failure whenever the app can't serve traffic correctly (dependency down, not warmed up). Different semantics, different endpoints.

---

## Deployments and Rolling Updates

A **Deployment** manages a ReplicaSet (which manages Pods). You describe the desired state; the Deployment controller ensures the right number of pods are running with the right spec.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # never go below 3 running pods
      maxSurge: 1           # allow up to 4 pods during rollout
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v2.1
```

`maxUnavailable: 0` + `maxSurge: 1` = zero-downtime rolling update. Kubernetes brings up one new pod, waits for it to pass readiness, then terminates one old pod. Repeats until all pods are updated.

**Rollback:**
```bash
kubectl rollout undo deployment/my-app            # rollback one version
kubectl rollout undo deployment/my-app --to-revision=3
kubectl rollout history deployment/my-app         # see revision history
```

---

## Services and DNS

A **Service** provides a stable DNS name and virtual IP for a set of pods selected by labels.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service          # selects pods with this label
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP               # only reachable within the cluster
```

Inside the cluster, this is reachable at:
- `order-service` (within same namespace)
- `order-service.production` (cross-namespace short form)
- `order-service.production.svc.cluster.local` (FQDN)

**Service types:**

| Type | Reachability | Use case |
|---|---|---|
| `ClusterIP` | Cluster-internal only | Default; internal service-to-service |
| `NodePort` | Node IP + port externally | Dev/testing; avoid in production |
| `LoadBalancer` | Cloud load balancer provisioned | External traffic entry point |
| `ExternalName` | DNS CNAME alias | Alias an external service inside the cluster |

---

## ConfigMap Live Update Flow — Full Picture

```
Developer updates ConfigMap:
  kubectl apply -f configmap.yaml
         │
         ▼
  API server stores new version in etcd
         │
  kubelet on each node polls API server (~60s interval)
         │
  kubelet detects ConfigMap version changed
         │
  kubelet writes new files to the volume's versioned directory
  e.g. /etc/config/..2026_07_11_10_00_00.123456789/
         │
  kubelet atomically swaps the symlink:
  /etc/config → ..2026_07_11_10_00_00.123456789/
         │
  Application's inotify watch fires on /etc/config/
         │
  Application reloads config in-process
```

The atomic symlink swap is why applications can safely read config files even during an update — they never see a partially-written file. The entire new config becomes visible in one `rename()` at the OS level.

---

## Key Takeaways

- **Pod** = atomic unit; pause container holds the namespaces; containers share localhost and can share volumes
- **ConfigMap** = non-secret config; directory mounts live-update, `subPath` mounts do not
- **Secret** = sensitive data; base64 encoded by default — enable KMS encryption or use Vault for real security
- **ServiceAccount** = pod identity; use projected tokens (time-limited, audience-bound), not legacy auto-mounted tokens
- **Downward API** = inject pod metadata into containers without API calls; labels/annotations only update via volume, not env vars
- **Projected volume** = combine ConfigMap + Secret + SA token + Downward API into one mountPath
- **CPU limits cause throttling, not crashes** — use requests, consider omitting limits for latency-sensitive containers
- **One BestEffort container makes the whole pod BestEffort** — always set resources on sidecars too
- **Init containers** guarantee ordered startup; native sidecars (1.29+) guarantee ordered shutdown

---

## Related Notes
- [[Container and Kubernetes Networking]] — how the pause container's network namespace actually gets its IP and connectivity (CNI, veth pairs, kube-proxy)
- [[Istio and Envoy]] — service mesh layer on top of K8s; Envoy xDS, VirtualService, DestinationRule, AuthorizationPolicy
- [[Sidecar Pattern]] — how these primitives are used together in multi-container pods
- [[Declarative vs Imperative Configuration]] — Kubernetes manifests as declarative desired-state config
- [[Distributed Systems/Delivery Semantics]] — Kubernetes controllers use reconciliation loops, not one-shot commands

