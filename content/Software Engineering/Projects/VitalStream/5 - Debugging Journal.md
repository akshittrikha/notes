Three real bugs hit while standing this up via `docker compose up` and `pytest`. This is the material most worth remembering cold — anyone can describe an architecture from a diagram; being able to narrate an actual debugging session is what proves you built it.

## Bug 1 — `bitnami/kafka:3.7` no longer resolves
**Symptom**: `docker compose up` failed immediately — `failed to resolve reference "docker.io/bitnami/kafka:3.7": not found`.
**Cause**: Bitnami restructured their image tagging/distribution; the old floating tag scheme for community images changed and that specific tag was no longer available on Docker Hub.
**Fix**: Switched to the official `apache/kafka:3.7.0` image. Had to re-derive the KRaft env var names, since Bitnami used a `KAFKA_CFG_*` prefix and the official image uses plain `KAFKA_*` (e.g. `KAFKA_CFG_NODE_ID` → `KAFKA_NODE_ID`, `KAFKA_CFG_PROCESS_ROLES` → `KAFKA_PROCESS_ROLES`). Confirmed the `kafka-topics.sh` script path by shelling into the image directly: `docker run --rm --entrypoint bash apache/kafka:3.7.0 -c "ls /opt/kafka/bin"`.
**Lesson**: don't trust a docker-compose snippet from memory/training data for a fast-moving image ecosystem — verify the image actually pulls and its env var conventions before wiring healthchecks around it.

## Bug 2 — consumer silently never processes anything (the big one)
**Symptom**: `docker compose up` reported all services healthy. The producer script ran and printed "sent {...}" for every reading. But `GET /devices/{id}/readings` came back empty, and the consumer logs showed it had connected to Kafka and subscribed to `vitals.raw` — then nothing. No errors, no processed-message logs, just silence.
**Investigation**:
1. Checked the topic existed and had the message: `docker exec vitalstream-kafka-1 /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic vitals.raw` — topic existed, 1 partition, looked fine.
2. Tried to inspect consumer group lag: `kafka-consumer-groups.sh --describe --group vitalstream-consumer` — this **hung and timed out** with `TimeoutException: Call(callName=describeGroups(api=FIND_COORDINATOR)...)`.
3. Tried reading the topic directly with `kafka-console-consumer.sh --from-beginning` — also hung, `Processed a total of 0 messages`.
4. The pattern (metadata/list operations work, anything requiring **group coordination** hangs) pointed at the internal `__consumer_offsets` topic never successfully being created.
**Root cause**: Kafka defaults `offsets.topic.replication.factor` to **3**. On a single-broker dev cluster there's no way to satisfy replication factor 3, so the internal offsets topic can't be created, and any consumer-group coordination (which depends on writing to that topic) stalls indefinitely — silently, no error surfaced to the consumer client.
**Fix**: added to the `kafka` service in `docker-compose.yml`:
```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: "1"
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: "1"
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: "1"
```
Then `docker compose down -v` (to clear the half-broken state) and rebuilt from scratch. Consumer logs immediately showed `Discovered coordinator`, `Successfully joined group vitalstream-consumer with generation 1`, and readings started flowing into Postgres/Redis correctly.
**Lesson**: this is a well-known but easy-to-miss single-broker Kafka gotcha. The failure mode is unusually quiet — no crash, no obvious error in the consumer's own logs — which makes "check whether `__consumer_offsets` was actually created" a genuinely useful diagnostic instinct to have, not just trivia.

## Bug 3 — monkeypatched Kafka/Redis calls didn't actually get patched in tests
**Symptom**: after adding `monkeypatch.setattr("app.cache.set_latest_reading", ...)` and `monkeypatch.setattr("app.kafka_producer.publish_alert", ...)` in `tests/conftest.py`, one test still made a real (failing) network call to Redis/Kafka during `pytest`.
**Cause**: `app/kafka_consumer.py` imports these with `from app.cache import set_latest_reading` and `from app.kafka_producer import publish_alert` — that binds a *new name* into `app.kafka_consumer`'s own namespace at import time. Patching `app.cache.set_latest_reading` afterward changes the attribute on the `app.cache` module, but `app.kafka_consumer.set_latest_reading` is already a separate reference pointing at the original function — patching the source module doesn't touch it.
**Fix**: patch at the point of use instead — `monkeypatch.setattr("app.kafka_consumer.set_latest_reading", ...)` and `monkeypatch.setattr("app.kafka_consumer.publish_alert", ...)`.
**Lesson**: classic Python monkeypatching trap — `from x import y` creates an independent binding, so always patch where a name is *looked up at call time*, not where it's *defined*. Worth having a crisp explanation of this ready, since "how does Python import binding affect mocking" is a fair follow-up question after mentioning tests.

## Smaller fixes along the way
- `@app.on_event("startup")` is deprecated in current FastAPI in favor of a `lifespan` async context manager — switched `app/main.py` to `@asynccontextmanager def lifespan(app): ...; yield`, which also fixed a real problem: the FastAPI test fixture triggers startup, and the old handler tried to hit a real Postgres instance during unit tests. The lifespan swap plus monkeypatching `Base.metadata.create_all` in `conftest.py` was needed together to make tests infra-free.
- `ruff` flagged three lines over the 110-char limit (log statements with several `%s` args) — wrapped them onto multiple lines rather than raising the limit, since the log statements were genuinely more readable split up.
