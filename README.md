# OpenSky Observatory

**A worldwide aircraft-data pipeline with live dashboards for 20 major airlines.**

Python · Apache Kafka · PostgreSQL · dbt · Grafana · Docker Compose

OpenSky → Kafka → Python → PostgreSQL → dbt → Grafana

The pipeline ingests real aircraft-position data, validates messages, quarantines invalid records, handles duplicate and out-of-order events, and exposes airspace activity and pipeline health.

Synthetic data is available for offline testing and CI. Grafana dashboards display **real OpenSky data only**.

## Features

- Worldwide aircraft-position ingestion through the OpenSky API.
- OAuth2 authentication, API-budget awareness, token refresh, and retry handling.
- Kafka buffering with 48-hour retention.
- Python consumer with message validation and quarantine.
- PostgreSQL raw observations, latest aircraft positions, rejected messages, and pipeline metrics.
- Atomic database transactions and safe Kafka offset commits.
- Duplicate protection and out-of-order event handling.
- Incremental dbt transformations and analytical models.
- Grafana dashboards for live traffic and pipeline health.
- Docker Compose deployment and automated tests.

## Architecture

```text
OpenSky API
    |
Python Producer
    |
Apache Kafka
    |
Python Consumer
    |
PostgreSQL
    |             \
    dbt             Grafana
    |
Analytics Models
```

## Data Models

| Model                        | Grain / purpose                               |
| ---------------------------- | --------------------------------------------- |
| `stg_flight_states`          | One aircraft-position timestamp               |
| `dim_aircraft`               | One observed aircraft identifier              |
| `dim_airlines`               | Curated airline callsign reference            |
| `mart_daily_flight_activity` | Daily activity by source and inferred airline |
| `mart_active_flights`        | Minute-level observed aircraft activity       |

The aircraft dimension is derived from observations, not an authoritative aircraft registry. Airline identification uses curated callsign-prefix matching. Counts represent observations, not completed flights or routes.

## Reliability and Data Quality

- Validates aircraft identifiers, coordinates, timestamps, and numeric fields.
- Quarantines invalid messages with their original payload and rejection reason.
- Uses database transactions to commit raw data, current state, rejects, and processing metadata together.
- Commits Kafka offsets only after successful database commits.
- Prevents older positions from overwriting newer aircraft state.
- Uses an incremental dbt anti-join strategy to avoid missing late-committing records.
- Includes integration tests for rollback and message replay.

## Dashboards

Two Grafana dashboards are provisioned:

1. **Air Traffic:** Recent aircraft positions and activity for 20 selected major airlines.
2. **Pipeline Health:** Ingestion freshness, latency, throughput, API credits, and operational metrics.

Traffic dashboards query recent live records directly from PostgreSQL. Analytical dbt models refresh every five minutes.

## Retention

| Data                                   | Default retention |
| -------------------------------------- | ----------------- |
| Kafka messages                         | 48 hours          |
| Raw observations and rejected messages | 14 days           |
| Latest aircraft state                  | 24 hours          |
| Pipeline metrics                       | 30 days           |
| Processed-message registry             | 3 days            |

## Run Locally

Requirements: Docker Desktop with Linux containers and approximately 4 GB of available memory.

1. Add OpenSky credentials to `.secrets/opensky.json`.
2. Configure environment variables in `.env`.
3. Start the stack:

```bash
docker compose up -d --build
```

4. Open Grafana at `http://localhost:3000`.

The default local dashboard credentials are documented in `.env`.

Useful commands:

```bash
docker compose logs -f --tail 50 producer consumer dbt-runner
docker compose --profile test run --rm tests
docker compose run --rm --no-deps dbt-runner dbt build
docker compose down
```

See `docs/runbook.md` for recovery, replay, and operational procedures.

## Validation

The documented local validation results are:

- 42 Python tests passed with Kafka and PostgreSQL.
- 22 dbt build nodes passed.
- 18 Grafana dashboard queries executed successfully.

See `docs/validation.md` for validation evidence.

## Limitations

- A single Kafka broker and PostgreSQL instance do not provide node-level high availability.
- Producer crashes can lose a source snapshot before Kafka acknowledges it.
- Historical OpenSky backfill is not implemented.
- API availability and receiver coverage affect data completeness.
- Production deployment would require stronger security, backups, replication, and infrastructure monitoring.

## Repository Structure

```text
app/                         Python ingestion and validation
postgres/                    Database schema and roles
dbt_project/flight_analytics/ dbt models, seeds, and tests
dbt-runner/                  Scheduled dbt execution
grafana/                     Dashboards and datasource
tests/                       Unit and integration tests
scripts/                     Setup and utility scripts
docs/                         Runbook and validation evidence
.github/workflows/            CI pipeline
```
