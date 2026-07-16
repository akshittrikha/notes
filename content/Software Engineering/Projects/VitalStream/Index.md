## Overview
A backend service that ingests streamed vitals readings (heart rate, respiration rate, sleep stage) from simulated contactless bedside monitors, persists them, caches the latest reading per device, and raises threshold-based alerts. Built specifically to close the gap between my resume (Node/TS/Go background) and a Python/FastAPI/Kafka-heavy backend JD (Sleepiz — contactless vitals monitoring).

Repo: `~/repos/vitalstream`

This is a real, working project — built, dockerized, and run end-to-end (not just written). Read [[5 - Debugging Journal]] for the actual bugs hit along the way; that's the part worth remembering cold for an interview, since it's proof of hands-on work rather than copy-pasted boilerplate.

## Pages
- [[1 - Requirements]] — what the JD asked for, mapped to what the project demonstrates
- [[2 - Architecture]] — components, data flow, why it's shaped this way
- [[3 - Design Decisions]] — the specific tradeoffs I'd defend in an interview (Kafka vs MQTT vs WebSockets, cache-aside, separate consumer process, threshold alerts)
- [[4 - Build Log]] — step by step, what was built and in what order, file by file
- [[5 - Debugging Journal]] — the three real bugs hit while standing this up, and how they were diagnosed
- [[6 - Interview Prep]] — anticipated questions and honest answers
- [[7 - Packages]] — every package in use, filled in incrementally as we discuss them
- [[8 - Python Internals]] — core language mechanics (not packages), SDE3-depth, filled in as we discuss them

## Stack
Python 3.11, FastAPI, SQLAlchemy 2.0, PostgreSQL, Redis, Kafka (kafka-python), Docker + Docker Compose, Kubernetes manifests, Prometheus (via `prometheus-fastapi-instrumentator`), pytest, GitHub Actions, ruff.

## Status
- [x] Core API (device registration, readings, alerts, latest-reading cache)
- [x] Kafka producer (device simulator) + consumer (persists, caches, alerts)
- [x] Dockerfile + docker-compose (Postgres, Redis, Kafka, API, consumer)
- [x] Kubernetes manifests (Deployment/Service/ConfigMap/HPA) — written, not applied to a live cluster
- [x] pytest suite (9 tests, in-memory SQLite, no infra needed) + GitHub Actions CI
- [x] Validated end-to-end via `docker compose up` — real Kafka messages flowing through to Postgres/Redis/API
- [ ] WebSocket push endpoint for live dashboard updates (discussed, not yet built — see [[3 - Design Decisions]])
- [ ] Alembic migrations (currently `Base.metadata.create_all` — a noted shortcut, not "the real answer")
