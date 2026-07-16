Each of these is something I'd actually be asked to defend. Written so I can explain the *reasoning*, not just recite the choice.

## Why Kafka at all, for one or two simulated devices?
Honestly: at this scale, you don't need it — a device could `POST` straight to the API. Kafka earns its place once there's more than one consumer of the same stream (alerting, analytics/ML, audit log, live dashboard all reading the same events independently), you need durability across consumer outages (a crashed alerting service shouldn't lose readings), you need to absorb bursty load without hammering Postgres directly, and you need per-device ordering (partition key = device ID). This project uses Kafka to demonstrate the pattern the JD asked for, not because one simulated device needs it. Saying this openly in an interview is a stronger answer than pretending it was load-bearing from day one.

## Kafka vs MQTT vs WebSockets — where each actually belongs
- **Device → backend ingestion**: real IoT/medical devices typically speak **MQTT**, not Kafka directly — it's built for constrained devices (tiny overhead, keep-alives, QoS levels for at-least-once delivery, session resumption over flaky networks). The realistic architecture is `device --MQTT--> gateway --Kafka--> backend consumers`. My `simulate_device.py` publishes straight to Kafka, skipping the gateway — a simplification I should name if asked "would the real device talk to Kafka directly?" (Answer: no.)
- **Backend → client (dashboard)**: this is where **WebSockets** (or SSE) are actually the right tool, and it's a real gap in the current build. `GET /devices/{id}/latest` requires polling. A live clinician/patient dashboard should get pushed updates the moment the consumer processes a reading — e.g. consumer publishes to a Redis Pub/Sub channel, a WebSocket endpoint in FastAPI subscribes and pushes to connected clients. Kafka (durable ingestion) and WebSockets (last-mile live push) are complementary, not competing — REST is still right for one-shot queries (historical readings, device registration).
- **Not yet built**: the WebSocket layer. If asked, this is an honest "here's what I'd add next," not a claimed feature.

## Separate consumer process, not embedded in the API
Covered in [[2 - Architecture]] — different scaling axis, different failure mode. A crash in Kafka consumption shouldn't take down the HTTP API serving dashboard queries.

## Cache-aside on `/latest`, not write-through
The consumer writes to Postgres first (source of truth), then best-effort writes to Redis (`app/cache.py` swallows `RedisError` and just logs a warning). Losing the cache for one device for the TTL window is cheap; losing a reading because a Redis hiccup blocked the write path is not. This is a chosen asymmetry — Postgres write failures do propagate (500 to the caller / message retried), Redis write failures do not.

## Threshold-based alerts, not per-patient baselines
`app/alerts.py` uses fixed global thresholds (heart rate 40-120, respiration 8-30) via env-configurable settings. Real vitals monitoring would calibrate per patient/condition — a fixed threshold is medically naive by design. Kept deliberately simple so the alerting logic is honest about what it actually does, rather than dressing up a toy threshold check as something more sophisticated.

## The single-broker Kafka replication factor bug (and why it's worth remembering)
Kafka defaults `offsets.topic.replication.factor` to 3. On a single-broker dev cluster, the internal `__consumer_offsets` topic can't be created at that replication factor, which silently stalls consumer-group coordination — messages get produced fine, topic metadata looks fine, but the consumer never actually joins the group and processes anything. Fixed by setting `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1` (and the transaction-log equivalents) in `docker-compose.yml`. Full diagnosis story in [[5 - Debugging Journal]] — this is a good "tell me about a bug you debugged" answer because it's real, specific, and not googleable in 30 seconds.

## Schema via `Base.metadata.create_all`, not Alembic
Named directly in `app/main.py` as a demo shortcut. A real deployment would use Alembic migrations so schema changes are versioned and reversible; `create_all` can't alter existing tables or express a migration path. Left out of scope here to keep the 3-5 day build focused on the ingestion pipeline itself — if asked, the answer is "I know this is the gap, here's what I'd do instead," not a pretense that it's production-ready.

## Why kafka-python instead of confluent-kafka
`confluent-kafka` wraps `librdkafka` (a C library) and can be a pain to build on some ARM/M1 setups without extra native dependencies. `kafka-python` is pure Python — simpler local dev, no native build step, fine for this scale. Tradeoff: `confluent-kafka` is generally faster and more actively aligned with newer Kafka protocol features in high-throughput production use. Worth naming as a deliberate "optimize for dev velocity at this scale" choice, not ignorance of the alternative.
