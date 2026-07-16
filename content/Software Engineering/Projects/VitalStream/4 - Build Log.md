Order the project was actually built in, file by file. Useful for rebuilding the mental model of "what depends on what."

## 1. Foundations: config, logging, db session
- `app/config.py` — a single `Settings` class reading everything from env vars with sane local defaults (`DATABASE_URL`, `REDIS_URL`, `KAFKA_BOOTSTRAP_SERVERS`, topic names, alert thresholds). One object, imported everywhere as `from app.config import settings` — no scattered `os.getenv` calls.
- `app/logging_config.py` — `configure_logging()` sets up a single stdout stream handler with a consistent format, called once at process startup (both in `main.py` and `kafka_consumer.py`).
- `app/db.py` — SQLAlchemy 2.0 engine + `sessionmaker` + a `Base(DeclarativeBase)` class + a `get_db()` generator for FastAPI's dependency injection (`Depends(get_db)`).

## 2. Data model: `app/models.py`
Three tables using SQLAlchemy 2.0's `Mapped[]`/`mapped_column` typed style:
- `Device` — `external_id` (unique, e.g. `sleepiz-one-plus-001`), `label`, relationships to readings/alerts with cascade delete
- `Reading` — `heart_rate`, `respiration_rate`, `sleep_stage`, `recorded_at` (device-reported time) vs `ingested_at` (server time) kept as separate columns deliberately
- `Alert` — `metric`, `value`, `severity` (enum: warning/critical), `message`, linked to both device and the specific reading that triggered it

## 3. API contracts: `app/schemas.py`
Pydantic models separate from the SQLAlchemy models — `DeviceCreate`/`DeviceOut`, `ReadingIn`/`ReadingOut`, `AlertOut`. `ReadingIn` is specifically the shape of a raw Kafka message (what a device publishes), decoupled from `ReadingOut` (what the API returns) even though they overlap — kept separate on purpose so the Kafka wire format and the HTTP response format can evolve independently.

## 4. Redis cache: `app/cache.py`
`get_client()` lazily creates a single module-level Redis client. `set_latest_reading`/`get_latest_reading` wrap Redis calls in try/except `redis.RedisError`, logging and returning `None`/no-op on failure rather than raising — see [[3 - Design Decisions]] for why.

## 5. Alerting logic: `app/alerts.py`
Pure function `evaluate_reading(heart_rate, respiration_rate) -> list[AlertCandidate]`. Deliberately has zero I/O — no DB, no Kafka — so it's trivially unit-testable in isolation (see `tests/test_alerts.py`). `AlertCandidate` is a small `@dataclass`, not a full model, since it's an intermediate value, not something persisted directly.

## 6. Kafka producer: `app/kafka_producer.py`
Lazy singleton `KafkaProducer` (`acks="all"`, `retries=5`), JSON value serializer. Two thin wrapper functions: `publish_reading` (used by the simulator) and `publish_alert` (used by the consumer). Kept as plain functions, not a class, since there's no state beyond the singleton client.

## 7. Kafka consumer: `app/kafka_consumer.py`
The core of the pipeline — `process_message(db, raw)` does the actual work (validate → upsert device → insert reading → cache → evaluate alerts → persist+publish alerts), and `run()` is a thin infinite-loop wrapper (`for message in consumer: process_message(...)`) that owns a DB session per message and logs+continues on any exception rather than crashing the whole consumer on one bad message. Splitting `process_message` out from `run()` was specifically so tests could call it directly without needing a real Kafka connection (see `tests/test_devices_api.py::test_readings_and_alerts_flow`).

## 8. Device simulator: `scripts/simulate_device.py`
CLI script (`argparse`) generating fake readings for N devices and publishing them via `app.kafka_producer.publish_reading`. `--anomaly-rate` flag exists specifically to make the alert path exercisable on demand during manual testing, rather than waiting on random chance.

## 9. FastAPI routers: `app/routers/health.py`, `app/routers/devices.py`
`health.py` — one endpoint, runs `SELECT 1` to prove the DB connection is actually alive (not just that the process is up). `devices.py` — all the device/reading/alert endpoints, with a shared `_get_device_or_404` helper so every endpoint 404s consistently on an unknown device.

## 10. Wiring it together: `app/main.py`
FastAPI app with a `lifespan` context manager (not the deprecated `@app.on_event("startup")` — see [[5 - Debugging Journal]]) that runs `Base.metadata.create_all()`, includes both routers, and mounts Prometheus instrumentation at `/metrics`.

## 11. Containerizing: `Dockerfile`, `docker-compose.yml`
Single `Dockerfile` (python:3.11-slim, installs `requirements.txt`, copies `app/` and `scripts/`) used by both the `api` and `consumer` Compose services — same image, different `command:`. Compose also brings up `postgres` (16-alpine), `redis` (7-alpine), and `kafka` (`apache/kafka:3.7.0`, KRaft mode — no Zookeeper needed), all with healthchecks so `api`/`consumer` wait via `depends_on: condition: service_healthy` rather than racing the infra on startup.

## 12. Tests: `tests/`
`conftest.py` swaps Postgres for an in-memory SQLite session (`StaticPool` so the single connection is shared, since `:memory:` SQLite is otherwise per-connection) and overrides FastAPI's `get_db` dependency. Kafka/Redis side effects are monkeypatched out at the point of use (see [[5 - Debugging Journal]] for a real gotcha here). `test_alerts.py` tests the pure threshold logic directly with no infra at all; `test_devices_api.py` drives the full HTTP+DB flow including calling `process_message` directly to simulate a Kafka message arriving.

## 13. CI: `.github/workflows/ci.yml`, `pyproject.toml`
GitHub Actions installs `requirements-dev.txt`, runs `ruff check` then `pytest -v` on every push/PR. `pyproject.toml` holds ruff config (line length 110, `E`/`F`/`I` rule sets) and `pytest` config (`pythonpath = ["."]` so `app`/`tests` imports resolve without an installed package).

## 14. Kubernetes manifests: `k8s/`
Written last, once the Compose setup was already proven to work — `deployment.yaml`/`service.yaml` for the API (readiness/liveness on `/health`), `consumer-deployment.yaml` for the consumer, `configmap.yaml` for shared env vars, `hpa.yaml` scaling the API on CPU. Not applied to a live cluster — see [[2 - Architecture]].

## 15. Validation
Everything above was actually run, not just written — see [[5 - Debugging Journal]] for the three real bugs hit doing this: the Bitnami Kafka image being pulled from Docker Hub, the single-broker replication-factor stall, and a monkeypatch-target mistake in the tests.
