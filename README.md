# Distributed Network Analytics Pipeline

Multi-source telemetry ingestion pipeline for interview portfolio work. Simulates 5G/RAN network
node metrics, streams them through Kafka, and indexes into OpenSearch — with FastAPI, alerting,
and Kubernetes deployment planned across an 8-week build.

**Resume alignment:** Python, Kafka, OpenSearch, FastAPI, Docker, Kubernetes, Helm, Prometheus.

## Architecture

```
Simulated Network Nodes (Producer)
        │
        ▼
   Kafka (telemetry.raw)
        │
        ├──► DLQ (telemetry.dlq)  ← Phase 2
        │
        ▼
   Consumer + Validator
        │
        ▼
   OpenSearch (telemetry-*)      ← Phase 3
        │
        ├──► Alert Engine          ← Phase 5
        │
        ▼
   FastAPI (search, metrics)      ← Phase 4
```

## Stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.11 |
| Messaging | Kafka 3.x (Bitnami) |
| Search | OpenSearch 2.x |
| API | FastAPI + Uvicorn (Phase 4) |
| Packaging | Docker Compose → K8s/Helm (Phase 7) |

## Quick Start

### 1. Prerequisites

- Python 3.11+
- Docker Desktop (with Compose v2)

### 2. Clone and install

```powershell
cd E:\Switch\network-analytics-pipeline
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
copy .env.example .env
```

### 3. Start infrastructure

```powershell
docker compose up -d
docker compose ps
```

Wait until `kafka`, `opensearch`, and `kafka-init` complete. Topics `telemetry.raw` (6 partitions)
and `telemetry.dlq` (3 partitions) are created automatically.

Verify:

```powershell
docker exec nap-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
curl http://localhost:9200/_cluster/health
```

### 4. Run producer and consumer

Terminal 1 — producer (100 events/sec):

```powershell
python -m producer.main
```

Terminal 2 — consumer:

```powershell
python -m consumer.main
```

Optional: limit consumer for smoke test:

```powershell
python -m consumer.main --max-messages 50
```

### 5. Run tests

```powershell
pytest -v
```

## Project Layout

Matches **section 4.3** of `E:\Switch\JobSwitchPlan\Distributed-Network-Analytics-Pipeline.docx`:

```
network-analytics-pipeline/
├── producer/              # Kafka telemetry producer
├── consumer/              # Validator, indexer, DLQ handler
├── api/                   # FastAPI (Phase 4)
├── alert_engine/          # Rule evaluation + notifications (Phase 5)
├── shared/                # Pydantic schemas, logging, config
├── deploy/
│   ├── docker/            # Dockerfiles (Phase 7)
│   ├── compose/           # docker-compose.yml
│   └── helm/              # Helm chart (Phase 7)
├── docs/                  # ARCHITECTURE.md, RUNBOOK.md, ADRs
├── tests/
├── requirements.txt
├── docker-compose.yml     # includes deploy/compose/docker-compose.yml
└── pyproject.toml
```

## Event Schema

| Field | Type | Required | Example |
|-------|------|----------|---------|
| event_id | UUID | Yes | Idempotency key |
| device_id | string | Yes | `gNB-blr-014` (Kafka partition key) |
| timestamp | ISO8601 UTC | Yes | `2026-06-24T10:00:00Z` |
| metric_name | string | Yes | `prb.utilization` |
| value | float | Yes | `87.5` |
| severity | enum | No | info / warn / critical |
| site_id | string | No | `blr-south-01` |
| technology | string | No | `5G-NR` |

## Build Phases (60-day plan)

| Phase | Days | Milestone | Status |
|-------|------|-----------|--------|
| 1 — Foundation | 1–6 | Producer → Kafka → Consumer | **Done (Week 1)** |
| 2 — Reliability | 7–11 | Validation, DLQ, structured logging | Planned |
| 3 — OpenSearch | 12–17 | Bulk index + search helpers | Planned |
| 4 — FastAPI | 18–22 | REST APIs + auth | Planned |
| 5 — Alerting | 23–26 | Rule engine + notifications | Planned |
| 6 — Observability | 27–29 | Prometheus, OTel, Grafana | Planned |
| 7 — K8s/Helm | 30–35 | Production-style deploy | Planned |
| 8 — Load test | 36–40 | 10K events/sec + demo | Planned |

## Configuration

Copy `.env.example` to `.env`. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| KAFKA_BOOTSTRAP_SERVERS | localhost:9092 | Broker address |
| KAFKA_TOPIC_RAW | telemetry.raw | Ingest topic |
| PRODUCER_EVENTS_PER_SEC | 100 | Producer rate |
| PRODUCER_DEVICE_COUNT | 10 | Simulated devices |

## Interview Talking Points

- **Partition by `device_id`:** ordering per device, parallel consumers across partitions
- **Manual offset commit:** at-least-once delivery; idempotent indexing with `event_id` as `_id` (Phase 3)
- **DLQ pattern:** poison messages isolated without blocking the pipeline (Phase 2)
- **Telecom metrics:** PRB utilization, handover count, latency — mirrors RAN/OSS background

## References

- **Canonical spec (Word):** `E:\Switch\JobSwitchPlan\Distributed-Network-Analytics-Pipeline.docx`
- Regenerate doc: `python E:\Switch\JobSwitchPlan\generate_project_docx.py`
- Markdown guide: `E:\Switch\JobSwitchPlan\distributed-network-analytics-pipeline-guide.md`
- Worksheet tracker: `E:\Switch\JobSwitchPlan\worksheets\10-distributed-analytics-pipeline.md`
