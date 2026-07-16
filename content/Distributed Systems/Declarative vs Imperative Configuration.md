# Declarative vs Imperative Configuration

#distributed-systems #devops #kubernetes #terraform #configuration

---

## The Core Difference

> **Imperative** — describe *how* to get there (the steps).  
> **Declarative** — describe *what* you want (the end state).

| | Imperative | Declarative |
|---|---|---|
| You specify | Exact operations in order | Desired end state |
| System role | Dumb executor | Smart reconciler |
| Run twice | May break (creates duplicates, errors) | No-op (already at desired state) |
| Self-healing | No | Yes |
| Example | Bash scripts, AWS CLI | Kubernetes, Terraform, SQL |

---

## Imperative — "Here's What to Do"

You are the planner. The system does exactly what you say, in order.

```bash
# Imperative: spin up 3 web servers
create_server --name web-1 --type t3.medium
create_server --name web-2 --type t3.medium
create_server --name web-3 --type t3.medium
attach_load_balancer web-1
attach_load_balancer web-2
attach_load_balancer web-3
```

**Problem:** Run this twice → 6 servers. The script has no concept of existing state. It just follows orders.

---

## Declarative — "Here's What I Want"

You are the specifier. The system figures out how to get there — and stay there.

```yaml
# Declarative: I want 3 web servers behind a load balancer
servers:
  count: 3
  type: t3.medium
  load_balancer: true
```

Run this twice → nothing changes. The system sees 3 servers already exist. Already at desired state.

---

## The Reconciliation Loop

The engine behind every declarative system. Runs continuously:

```
┌─────────────────────────────────────────────┐
│                                             │
│   Observe actual state                      │
│          │                                  │
│          ▼                                  │
│   Compare to desired state                  │
│          │                                  │
│     gap? │ no gap                           │
│          │──────────→ do nothing            │
│          │                                  │
│     gap detected                            │
│          │                                  │
│          ▼                                  │
│   Take action to close gap                  │
│          │                                  │
│          └──────────────────────────────────┤
│                   (loop forever)            │
└─────────────────────────────────────────────┘
```

**Example — Kubernetes:**
```
Desired state:  3 replicas of web-app
Actual state:   2 replicas (one crashed at 3am)
                      │
               gap detected
                      │
               controller creates 1 new pod
                      │
               actual = desired ✓
```

This is **self-healing**. No human intervention needed. The config is the source of truth; reality must match it.

---

## Idempotency

Declarative configs are idempotent by design — applying the same config 10 times = applying it once.

Imperative scripts are not idempotent by default — you must add guards manually:

```bash
# Imperative: not idempotent
mkdir /var/app/logs          # ERROR if already exists

# Imperative: made idempotent (you're reinventing declarative)
if [ ! -d /var/app/logs ]; then
    mkdir /var/app/logs
fi
```

```yaml
# Declarative: idempotent by design
directory "/var/app/logs":
  state: present             # create if missing, skip if exists
```

---

## Real-World Examples

| Tool | Style | You write |
|---|---|---|
| Kubernetes (`deployment.yaml`) | Declarative | "I want 3 replicas of this container" |
| Terraform (`.tf` files) | Declarative | "I want this VPC, subnet, RDS instance" |
| SQL (`SELECT`) | Declarative | "I want rows where age > 30" |
| Bash / shell scripts | Imperative | Exact commands in order |
| AWS CLI commands | Imperative | `aws ec2 run-instances ...` |
| Docker Compose | Declarative | "I want these 4 services wired together" |
| Ansible playbooks | Mostly declarative | "ensure nginx is installed and running" |

---

## SQL — The Oldest Example

Most developers use declarative config without realising it:

```sql
-- Declarative: what you want
SELECT name FROM users WHERE age > 30 ORDER BY name;

-- Imperative equivalent: how to get it
for each row in users:
    if row.age > 30:
        collect row.name
sort collected names alphabetically
return collected names
```

The query planner decides whether to use an index, what join algorithm to pick, how to order operations. You specify what — never how.

---

## When Imperative Is Still the Right Choice

Declarative isn't always better. Use imperative when:

- **Order genuinely matters** — database migrations (step 1 must complete before step 2 runs)
- **One-off operations** — a cleanup script that should run exactly once
- **Non-idempotent by nature** — sending an email, charging a card, publishing an event
- **Complex branching logic** — "if region is us-east-1, do X, else Y based on Z"

> [!note] Both tools coexist
> Terraform (declarative) manages infrastructure. You still write imperative migration scripts for data changes. Neither replaces the other — they solve different problems.

---

## The Mental Model

> **Imperative:** you write a recipe. The system follows it blindly.
>
> **Declarative:** you write a photograph of what the kitchen should look like. The system makes reality match the photograph — and keeps checking that it still does.

The photograph (config file) is the **source of truth**. This is why Kubernetes manifests, Terraform files, and Helm charts live in git. A `git push` is how you change the world — the reconciliation loop handles the rest.

---

## Key Takeaways

- Imperative = you plan + system executes; declarative = you specify + system plans and executes
- Declarative configs are idempotent — safe to apply repeatedly
- The reconciliation loop is what makes self-healing possible
- Declarative systems store desired state in git — the repo *is* the system state
- Use imperative for ordered, one-shot, or inherently non-idempotent operations
- SQL, Kubernetes, and Terraform are all declarative — the paradigm is everywhere

---

## Related Notes
- [[RPC and APIs]] — APIs as contracts, similar philosophy to config-as-contract
- [[Distributed Systems/Consistency Models]] — how distributed systems reason about state

