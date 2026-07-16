## Data flow
```
scripts/simulate_device.py --> Kafka (vitals.raw) --> app/kafka_consumer.py --> PostgreSQL (readings, alerts)
                                                                             \-> Redis (latest reading per device, TTL)
                                                                             \-> Kafka (vitals.alerts) on threshold breach

app/main.py (FastAPI) --> reads PostgreSQL + Redis --> REST API (/devices, /health, /metrics)
```

## Components

### Producer — `scripts/simulate_device.py`
Simulates N devices (`sleepiz-one-plus-001`, `...-002`, ...), each emitting a reading every `--interval` seconds: heart rate (`random.gauss(65, 6)`), respiration rate (`random.gauss(16, 2)`), sleep stage (random choice), timestamp. `--anomaly-rate` controls how often it deliberately emits an out-of-range heart rate (20-39 or 121-160) to exercise the alerting path. Publishes JSON to the `vitals.raw` Kafka topic, keyed by device ID (so all of one device's readings land on the same partition, preserving per-device order).

### Consumer — `app/kafka_consumer.py`
Standalone process (own container in Compose, own Deployment in K8s — **not** part of the API process). For each message:
1. `_get_or_create_device` — upsert the device row (handles the race of two messages for a brand-new device arriving concurrently, via catching `IntegrityError` and re-querying)
2. Insert a `Reading` row into Postgres — source of truth
3. `set_latest_reading` — best-effort write to Redis with a TTL (failures are logged and swallowed, never block the pipeline — see [[3 - Design Decisions]])
4. `evaluate_reading` (`app/alerts.py`) — simple threshold check (heart rate outside [40,120] → critical, respiration outside [8,30] → warning). Any hits become `Alert` rows in Postgres and get published to `vitals.alerts`.

### API — `app/main.py`, `app/routers/`
FastAPI service, separate from the consumer:
- `POST /devices` — register a device (external_id + optional label), 409 on duplicate
- `GET /devices` — list devices
- `GET /devices/{id}/latest` — Redis first, Postgres fallback, 404 if truly nothing recorded
- `GET /devices/{id}/readings?limit=` — historical readings from Postgres, newest first
- `GET /devices/{id}/alerts?limit=` — alerts for a device
- `GET /health` — liveness/readiness check (`SELECT 1` against Postgres)
- `GET /metrics` — Prometheus format, via `prometheus-fastapi-instrumentator`

### Storage
- **Postgres** (`app/models.py`): `Device`, `Reading`, `Alert` tables, string UUID primary keys, SQLAlchemy 2.0 `Mapped[]` style models. Schema created via `Base.metadata.create_all()` in the FastAPI lifespan handler — a deliberate demo shortcut, not how you'd do it in production (see [[3 - Design Decisions]]).
- **Redis** (`app/cache.py`): one key per device (`latest_reading:{external_id}`), JSON-encoded, TTL from `settings.latest_reading_cache_ttl_seconds` (default 300s).

## Why two separate processes (API vs consumer)
They scale on completely different axes — the API scales on HTTP request volume, the consumer scales on Kafka partition count / ingestion throughput. Coupling them into one process means you can't scale one without the other, and a slow/crashing consumer would take the API down with it. This is reflected in Docker Compose (`api` and `consumer` are separate services, same image, different `command`) and in K8s (`k8s/deployment.yaml` for the API, `k8s/consumer-deployment.yaml` for the consumer, both built from the same image).

## Deployment shape (K8s)
- `k8s/deployment.yaml` — API Deployment, 2 replicas, readiness/liveness probes on `/health`
- `k8s/consumer-deployment.yaml` — consumer Deployment, 1 replica (would scale with partition count, not CPU)
- `k8s/service.yaml` — ClusterIP Service in front of the API pods
- `k8s/configmap.yaml` — shared env vars (DB/Redis/Kafka connection strings, topic names)
- `k8s/hpa.yaml` — HPA scaling the API 2-6 replicas on 70% CPU utilization

These are written to be structurally correct (`kubectl apply -f k8s/` against a real cluster with the image available) but were **not applied to a live cluster** in this project — validation was via Docker Compose. Worth saying this plainly rather than implying it was cluster-tested.
