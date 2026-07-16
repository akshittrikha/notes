Anticipated questions and the honest answer to each — practice saying these out loud, not just reading them.

## "Walk me through your architecture."
Use [[2 - Architecture]]'s data flow diagram. Lead with the shape (producer → Kafka → consumer → Postgres/Redis, API reads from storage), then immediately name *why* the consumer is a separate process from the API — different scaling axis, different failure mode. Don't recite every file; narrate the flow of one reading through the system.

## "Why did you use Kafka here?"
Answer from [[3 - Design Decisions]]: honest that at this scale you don't strictly need it, then pivot to what actually justifies it — multiple independent consumers, durability across outages, ordering per device. Name that a single-device MVP wouldn't need it. This shows judgment, not just tool familiarity.

## "Would a real device talk to Kafka directly?"
No — realistically MQTT or HTTPS to a gateway, and Kafka lives behind that gateway. Say this before they catch it themselves.

## "What would you add next?"
Two honest answers, both real gaps: a WebSocket endpoint for live dashboard push (Redis Pub/Sub → WS, see [[3 - Design Decisions]]), and Alembic migrations instead of `create_all`. Having a ready, correct "here's the gap and here's the fix" is stronger than pretending it's finished.

## "Tell me about a bug you debugged recently."
Use the single-broker Kafka replication-factor stall from [[5 - Debugging Journal]]. It has everything a good debugging story needs: a confusing silent failure (no errors, just nothing happening), a methodical narrowing process (checked topic existed → tried group describe → tried console consumer → noticed the common thread was anything needing group coordination), and a root cause that's specific and non-obvious (`__consumer_offsets` can't be created at replication factor 3 on one broker). Don't undersell it — this is a genuinely good story.

## "How did you test this without a live Kafka/Postgres in CI?"
`tests/conftest.py` swaps Postgres for in-memory SQLite (`StaticPool` to keep one shared connection) and monkeypatches the Redis/Kafka calls at their point of use inside `app.kafka_consumer` (not at their point of definition — mention the `from x import y` binding gotcha from [[5 - Debugging Journal]] if asked why that distinction matters). `evaluate_reading` in `app/alerts.py` has zero I/O by design, so it's tested with no mocking at all.

## "Why separate the consumer from the API instead of running Kafka consumption inside the FastAPI process?"
Scaling and blast radius. The API scales on HTTP request volume (K8s HPA on CPU, see `k8s/hpa.yaml`); the consumer would scale on partition count / ingestion throughput — different signal entirely. And a crash or slowdown in Kafka consumption shouldn't take the HTTP API down with it, or vice versa.

## "What's the cache-aside pattern doing here, and why not write-through?"
`GET /devices/{id}/latest` checks Redis first, falls back to Postgres. The consumer writes Postgres (source of truth) first, then best-effort writes Redis — a Redis failure is logged and swallowed, never blocks the pipeline. Losing the cache for one device for a few minutes is cheap; losing a reading because a cache hiccup blocked the write path is not. That asymmetry is the actual design decision, not just "I used Redis for caching."

## "This alerting logic looks too simple for real vitals monitoring."
Correct, and intentional — say so before they do. Fixed global thresholds, not per-patient baselines. Real vitals monitoring would calibrate thresholds per patient/condition; this project keeps it simple so the alerting logic is honest about what it does rather than dressing up a toy check as clinical-grade.

## If asked directly: "Did you build this yourself?"
Yes, with AI-assisted tooling for scaffolding — same as using any modern IDE/copilot. The distinction that matters: every design decision in [[3 - Design Decisions]] and every bug in [[5 - Debugging Journal]] was actually reasoned through and actually hit, not invented after the fact. Understanding *why* each piece exists, being able to extend it live, and being able to defend the tradeoffs is what makes it legitimately "yours" — not whether every keystroke was manually typed.
