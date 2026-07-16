# API Design — Client Libraries, IDLs, and SLOs

#distributed-systems #api #slo #openapi #grpc #developer-experience

---

## APIs Need Clients

An API is just a contract — a description of what you can call. Without a client library, every developer consuming your API must:

- Manually construct the HTTP request
- Attach auth headers
- Serialize the request body to JSON
- Parse the response
- Map error codes to exceptions
- Handle retries and timeouts

That's 50+ lines of boilerplate before writing a single line of business logic. And every team does it differently — different timeout values, different error handling, different field name assumptions.

```python
# Hand-rolled HTTP call (every consumer reinvents this)
response = requests.post(
    "https://api.example.com/v1/users",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json={"name": "Alice", "email": "alice@example.com"},
    timeout=5
)
if response.status_code == 201:
    user = response.json()["data"]["user"]
elif response.status_code == 409:
    raise DuplicateUserError(...)

# With a generated client library
user = client.users.create(name="Alice", email="alice@example.com")
```

The second form is idiomatic, type-safe, and consistent. The first is a bug waiting to happen.

---

## IDLs — Write Once, Generate Everywhere

**Hand-writing** client libraries works if you control the language. The moment you have external developers or multiple languages, maintaining 6 manually-written SDKs that must stay in sync is untenable.

**IDL (Interface Description Language)** — a machine-readable schema that describes your API structure. Tools use it to auto-generate clients for any language.

```
OpenAPI spec (YAML)
        │
        ▼
  openapi-generator
        │
        ├──→ Python client
        ├──→ TypeScript client
        ├──→ Java client
        ├──→ Go client
        └──→ Ruby client
```

The schema is the single source of truth. When the API changes, update the schema and regenerate — all clients stay in sync automatically.

### Common IDLs

| IDL | Used with | Format |
|---|---|---|
| **OpenAPI** (Swagger) | REST APIs | YAML / JSON |
| **Protocol Buffers** | gRPC | `.proto` file |
| **GraphQL Schema** | GraphQL | SDL (Schema Definition Language) |
| **Thrift IDL** | Apache Thrift | `.thrift` file |

### OpenAPI Example

```yaml
paths:
  /users:
    post:
      summary: Create a user
      requestBody:
        content:
          application/json:
            schema:
              properties:
                name:  { type: string }
                email: { type: string }
      responses:
        "201":
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        "409":
          description: User already exists
```

From this spec, `openapi-generator` produces a fully functional client in 40+ languages — with typed request/response models, error classes, and auth handling.

---

## SLOs — The Performance Contract

> [!important] Without an SLO, callers build on assumptions. Assumptions become production incidents.

### Three Terms, Often Confused

```
SLI  (Service Level Indicator)  — what you measure
                                  "our p99 latency is 180ms this week"

SLO  (Service Level Objective)  — the target you commit to internally
                                  "p99 latency must stay below 200ms"

SLA  (Service Level Agreement)  — SLO with legal/financial consequences
                                  "if we breach the SLO, you get a refund"
```

SLIs are measurements. SLOs are targets. SLAs are contracts with teeth.

### Why SLOs Matter to the *Caller*

If you call my API and I have no published SLO, you have no idea:
- What timeout to set
- Whether to implement a circuit breaker
- Whether to cache responses to reduce dependency
- Whether your own reliability target is achievable given my availability

With a published SLO:

```
My SLO:
  p99 latency  < 200ms
  availability   99.9%   ← ~8.7 hours downtime/year
  throughput     1000 rps per client

Your system can now:
  timeout      = 500ms    (SLO + comfortable buffer)
  circuit break if error rate spikes above threshold
  cache responses if 99.9% availability isn't enough for your needs
  rate limit your own calls to stay under 1000 rps
```

### Common SLO Metrics

| Metric | What it measures | Example target |
|---|---|---|
| **Latency (p50/p99)** | How fast for median / tail requests | p99 < 200ms |
| **Availability** | % of time the service is up | 99.9% |
| **Error rate** | % of requests that fail | < 0.1% |
| **Throughput** | Requests per second the service handles | 5000 rps |

> [!note] p99 latency matters more than p50
> The median (p50) looks fine even when 1% of users are experiencing 10-second timeouts. Always publish tail latency.

---

## Practices to Follow

### As an API Producer

**1. Design the OpenAPI spec first, before writing code.**
Define the contract, then implement it. Tools like Stoplight or Swagger Editor let you prototype the schema before a single line of server code is written.

**2. Generate client SDKs from the spec automatically.**
Set up CI to regenerate and publish clients whenever the spec changes. Never hand-write clients for a schema-described API.

**3. Version your API in the URL.**
`/v1/`, `/v2/` — so existing clients aren't broken when you evolve the API.

**4. Publish explicit SLOs in your documentation.**
p50 latency, p99 latency, availability target, throughput limit. If you don't know your SLOs yet, instrument first and derive them from real traffic.

**5. Measure SLIs continuously.**
An SLO no one tracks is a fiction. Dashboard it, alert on it.

### As an API Consumer

**1. Set explicit timeouts based on the upstream SLO.**
Never use the default (often infinite). Rule of thumb: `timeout = p99_latency_SLO × 2.5`.

**2. Implement circuit breakers.**
If a downstream service is breaching its SLO, stop hammering it. Fail fast and return a degraded response.

**3. Design for the availability SLO.**
If your dependency has 99.9% availability (~8.7h downtime/year), your own availability can't exceed that unless you cache or degrade gracefully.

**4. Use idempotency keys on mutations.**
Makes retries safe when you don't know whether the first call executed. See: [[RPC and APIs#Failure Modes]].

**5. Use the generated client, don't hand-roll.**
If they publish an OpenAPI or Protobuf spec, generate or use the published client SDK.

---

## Key Takeaways

- A bare API is not a product — the client library is the developer experience
- IDLs (OpenAPI, Protobuf) let you generate clients for any language from a single schema
- The schema is the source of truth — clients should be generated, not handwritten
- SLO = the performance contract callers depend on to build their own systems
- Without SLOs, callers guess at timeouts and availability — and guess wrong
- Publish p99 latency, availability, and throughput as explicit, measured targets

---

## Related Notes
- [[RPC and APIs]] — how RPC and REST are mechanisms for implementing these API contracts
- [[Declarative vs Imperative Configuration]] — OpenAPI specs are declarative API contracts

