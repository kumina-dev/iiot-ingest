# iiot-ingest

Demo project: an IIoT telemetry ingestion pipeline. Simulated industrial devices publish
measurements over MQTT; a Java service ingests them in batches into TimescaleDB and exposes
a REST API for querying. Built to demonstrate backend design for high-volume device data.

## Stack (fixed — do not swap without asking)
- Java 17+, Maven (multi-module)
- Quarkus (Jakarta EE APIs: JAX-RS, CDI, Bean Validation)
- MQTT: Eclipse Mosquitto broker, SmallRye Reactive Messaging MQTT connector
- TimescaleDB (PostgreSQL), Flyway migrations
- Testcontainers + JUnit 5 for integration tests
- Docker Compose for the whole stack
- Optional later: small TypeScript dashboard (Vite, no heavy framework)

## Modules
- `simulator/` — standalone Java app; spawns N virtual devices (default 5000), each publishing
  readings (temperature, vibration, pressure) at its own rate. Injects realistic faults:
  out-of-range values, duplicate messages, devices dropping offline and reconnecting.
- `ingest/` — Quarkus service; subscribes to MQTT, validates, buffers, writes to DB in batches
  (flush by size OR time, whichever first). Bad messages go to a dead-letter table, never crash.
- `db/` — Flyway migrations: hypertable `measurements(time, device_id, metric, value)`,
  continuous aggregates for 1-minute and 1-hour averages, retention policy.

## MQTT topic and payload
- Topic: `factory/{site}/{device_id}/telemetry`
- Payload (JSON): `{"ts": "<ISO-8601>", "metrics": {"temperature": 71.3, "vibration": 0.42}}`

## REST API (ingest service)
- `GET /devices/{id}/latest`
- `GET /devices/{id}/series?metric=&from=&to=&bucket=1m|1h`
- `GET /alerts?metric=&above=` — devices whose latest value exceeds a threshold
- `GET /health`, `GET /metrics` (ingest rate, batch sizes, dead-letter count)

## Working rules
- Work in small steps; one phase at a time (see Plan). Stop after each phase and summarize
  what was built and WHY each design choice was made — the owner must be able to explain
  every part in a job interview.
- Prefer simple, readable code over clever code. No unused abstractions.
- Every phase ends with passing tests and `docker compose up` working.
- Record notable design decisions in `docs/decisions.md` (short: decision, alternatives, reason).

## Plan
1. Compose skeleton (Mosquitto + TimescaleDB), Maven multi-module setup, Flyway schema
2. Simulator
3. Ingest service: MQTT consume, validation, batching, dead-letter
4. Load test: measure sustained messages/sec, document results in README
5. REST API + continuous aggregates + integration tests
6. README: architecture diagram (Mermaid), how to run, benchmark numbers,
   "what I'd change in production" (AWS IoT Core / Azure IoT Hub, Kubernetes, partitioning,
   backpressure), and how AI agents were used and reviewed during development
7. Optional: TypeScript dashboard
