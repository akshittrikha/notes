## Source
Sleepiz AG — Backend Engineer JD. Sleepiz makes a contactless bedside vitals monitor (Sleepiz One+) and needs backend engineers to build the services that ingest and serve that vitals data.

## What the JD actually asked for
| JD requirement | Why it matters to them |
|---|---|
| Python, FastAPI/Flask | Their whole backend stack |
| Relational (MySQL/Postgres) + non-relational (Cassandra) DBs | Vitals readings + patient/device metadata need durable structured storage |
| SQL optimization, complex data models | Time-series-ish vitals data at scale |
| Microservices, REST APIs | Multiple services (ingestion, alerting, dashboards) talking to each other |
| Docker + Kubernetes | Containerized deployment, orchestration at scale |
| CI/CD pipelines | Automated deploys |
| Kafka / data streaming | Continuous vitals data from many devices, not request/response |
| Git, code review workflows | Standard team hygiene |
| Cloud (AWS/GCP/Azure) | Where this actually runs |
| Testing, debugging, logging | Baseline engineering discipline |
| (Good to have) Redis/Cassandra, MLOps, Prometheus/Grafana/ELK | Bonus signal |

## Gap this project was built to close
My actual experience (Rupeek, LG Soft) is Node.js/TypeScript/Go/Java/C++ — solid backend fundamentals (event reliability patterns, security middleware, distributed flows) but **zero real Python/FastAPI/Kafka/Kubernetes track record**. Listing Python as a skill with no project behind it doesn't survive scrutiny.

Rather than reword existing bullets to sound Python-flavored (dishonest, falls apart in the interview), the decision was: build one real, small, working system that touches almost every item in the JD table above, in the same problem domain Sleepiz operates in (streamed vitals from home devices) — see [[Index]] for why domain match matters, not just tech-list match.

## Scoping decisions
Given a 3-5 day budget (see full project chat), the project was scoped to hit breadth over depth:
- **In scope**: FastAPI, Postgres, Redis, Kafka producer+consumer, Docker/Compose, K8s manifests, pytest, GitHub Actions CI, Prometheus metrics.
- **Deliberately out of scope** (and why, so I can say so honestly in an interview rather than get caught pretending): Alembic migrations, API auth, multi-partition Kafka scaling, MQTT device gateway layer, WebSocket push. See [[3 - Design Decisions]] for the reasoning on each.

## How this maps to resume framing
Not "I have 2 years of Kafka experience" — it's "I built a working ingestion pipeline exercising Kafka, Postgres, Redis, Docker, and K8s in the same domain as your product, and I can walk through every design decision in it." That distinction is the entire point — see [[6 - Interview Prep]].
