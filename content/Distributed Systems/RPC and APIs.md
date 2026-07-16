# RPC and APIs in Distributed Systems

#distributed-systems #rpc #api #networking #grpc

---

## What is an RPC?

**Remote Procedure Call** — call a function on a *different machine* as if it were a local function.

The goal is to make a network call look like this:

```go
// Feels like a local call
result := userService.GetUser(userId)

// But actually crosses a network boundary, serializes arguments,
// sends bytes, deserializes on the other side, executes, and returns
```

The network is intentionally abstracted away from the caller. This is both RPC's superpower and its biggest lie.

---

## How RPC Works — Under the Hood

```
Client Machine                          Server Machine
─────────────────                       ─────────────────
                                        
  Your code                               Actual function
  getUserById(42)                         getUserById(id int)
       │                                        ▲
       ▼                                        │
  Client Stub                            Server Skeleton
  (generated code)                       (generated code)
       │                                        │
  Marshal args                          Unmarshal args
  [42] → bytes                          bytes → [42]
       │                                        │
       └────── network (TCP/HTTP) ─────────────┘
               bytes travel here
```

**Steps:**
1. Caller invokes the local **stub** — looks like a normal function
2. Stub **marshals** (serializes) arguments into bytes
3. Bytes sent over the wire (TCP, HTTP/2, etc.)
4. Server **skeleton** receives bytes, **unmarshals** back into arguments
5. Real function executes on the server
6. Return value marshalled, sent back
7. Client stub unmarshals, returns to caller as if local

**IDL (Interface Definition Language):** The stub and skeleton are usually auto-generated from a schema file (`.proto` for gRPC, `.thrift` for Thrift). You define the contract once; the toolchain generates both sides.

---

## What is an API?

**API (Application Programming Interface)** — a defined interface through which software components communicate.

> [!note] API is the broader concept
> An API is the *contract* — what functions exist, what arguments they take, what they return. RPC is one *mechanism* for fulfilling that contract remotely. REST is another. GraphQL is another.

APIs can be:
- **In-process** — a library's public functions (`List.sort()`)
- **Out-of-process, same machine** — IPC, Unix sockets
- **Remote** — across a network (this is where RPC and REST live)

---

## RPC vs REST

The most common confusion. Both are ways to expose remote functionality — but with fundamentally different philosophies.

| | RPC | REST |
|---|---|---|
| **Orientation** | Action / verb | Resource / noun |
| **Interface** | `createUser()`, `deleteOrder()` | `POST /users`, `DELETE /orders/42` |
| **Protocol** | Usually TCP or HTTP/2 | HTTP |
| **Format** | Binary (Protobuf, Thrift) or JSON | JSON (usually) |
| **Contract** | Strongly typed IDL (`.proto`) | OpenAPI/Swagger (optional) |
| **Network** | Hidden — abstracted away | Embraced — HTTP semantics matter |
| **Caching** | Hard — procedural calls aren't cacheable by default | Easy — GET is idempotent and cacheable |
| **Streaming** | Native (gRPC bidirectional streaming) | Awkward (SSE, WebSockets are hacks) |
| **Examples** | gRPC, Thrift, tRPC, JSON-RPC | GitHub API, Stripe API, Twitter API |

**The key philosophical difference:**

```
RPC thinks in verbs:        REST thinks in nouns:
  sendMessage()               POST /messages
  getUnreadCount()            GET /messages?status=unread
  markAllRead()               PATCH /messages  { read: true }
```

REST says: model your system as resources, use HTTP verbs (GET/POST/PUT/DELETE) to act on them, and use HTTP status codes to communicate outcomes. The network is not hidden — it's the interface.

RPC says: define functions, call them. The caller shouldn't have to think about HTTP verbs or resource URLs.

---

## Failure Modes — Why RPC is Hard

A local function call has two outcomes: returns or throws. An RPC has five:

```
Local call:
  success → got result
  failure → got exception

RPC call:
  success          → got result
  client failure   → never sent (easy to detect)
  network failure  → sent, never arrived (timeout)
  server failure   → arrived, executed, response lost (did it run??)
  partial failure  → ran, partially wrote state, then crashed
```

> [!important] The hardest case: did it execute?
> If you get a timeout on an RPC, you don't know whether the server received and executed the call. Retrying might execute it twice.

This is why **idempotency** matters in distributed systems. An idempotent operation produces the same result whether called once or many times — safe to retry.

### Delivery Semantics

| Semantic | Meaning | Tradeoff |
|---|---|---|
| **At-most-once** | Send once, don't retry | May lose calls on timeout |
| **At-least-once** | Retry until ACK | May execute twice — caller must handle |
| **Exactly-once** | Execute precisely once | Expensive — requires distributed coordination |

Most systems settle for **at-least-once + idempotent operations**.

---

## The 8 Fallacies of Distributed Computing

RPC tries to hide the network. This is dangerous because developers forget:

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every one of these is false. RPC abstractions that hide the network completely set developers up to violate these assumptions.

---

## Modern RPC Frameworks

### gRPC (Google)
- Uses **Protocol Buffers** (binary, strongly typed, compact)
- Runs over **HTTP/2** — multiplexed, header compression, full-duplex streaming
- Auto-generates client + server code from `.proto` files
- Supports unary calls, server streaming, client streaming, bidirectional streaming
- Used heavily in microservices (Kubernetes internal comms, etc.)

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}

message GetUserRequest { int64 user_id = 1; }
message User { int64 id = 1; string name = 2; }
```

### Apache Thrift (Meta)
- Similar to gRPC — IDL + code generation
- Supports more languages, more transport/protocol combinations
- Older; gRPC has largely superseded it for new projects

### tRPC (TypeScript)
- Type-safe RPC for TypeScript full-stack apps — no code generation step
- Types flow from server to client automatically via TypeScript inference
- Not a general-purpose distributed systems tool — lives in the TS/Node ecosystem

### JSON-RPC
- Simple standard: send `{ method, params, id }` JSON over HTTP or WebSocket
- No code generation, no binary encoding — purely human-readable
- Used by Ethereum nodes, some LSP (Language Server Protocol) tooling

---

## When to Use RPC vs REST

**Prefer RPC (gRPC) when:**
- Internal service-to-service communication (microservices)
- Performance matters — binary encoding is 3–10x smaller than JSON
- You need streaming (live data, large result sets)
- Strong typing and generated clients are valuable
- Both ends are under your control

**Prefer REST when:**
- Public-facing API consumed by third parties
- Clients are browsers (gRPC needs a proxy like grpc-web)
- Cacheability of resources matters
- You want HTTP semantics (status codes, content negotiation)
- Simplicity over performance

---

## Key Takeaways

- RPC makes remote calls look local — powerful abstraction, but hides real failure modes
- APIs are the broader concept (the contract); RPC and REST are mechanisms
- REST is resource-oriented and embraces HTTP; RPC is action-oriented and hides the network
- Network failures are partial — timeouts don't tell you if the call executed
- Idempotency is the practical solution to at-least-once delivery
- gRPC (Protobuf + HTTP/2) is the dominant modern RPC framework for microservices

---

## Related Notes
- [[Delivery Semantics]] — at-most-once / at-least-once / exactly-once; RPC failure modes map directly onto these
- [[Distributed Systems/Consistency Models]] — what guarantees RPC calls can make
- [[Database Internals/Fuzzy Checkpoint]] — WAL and durability as parallel to RPC delivery semantics

