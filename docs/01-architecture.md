# System Architecture

## Overview

The Ticket Intelligence System is a **modular monolith** — a single FastAPI application organised into independent internal modules, each owning one responsibility.

**Architecture principles:**

| Principle | Decision |
| --- | --- |
| **Single deployable unit** | One application container; no inter-service HTTP overhead |
| **Module isolation** | Each module has its own router, service, and models — no cross-module shared state |
| **In-process communication** | Modules call each other as Python functions; no network hops |
| **Background for slow work** | Embedding and steering refresh run as `BackgroundTasks` after the HTTP response is returned |
| **Scale-out path** | Any module can be extracted into a standalone service if load demands it — the boundaries are already clean |

---

## Architecture Diagram

```text
┌────────────────────────────────────────────────────────────────┐
│                      EXTERNAL CLIENTS                          │
│   Jira Webhook │ Custom Backend │ Frontend Dashboard           │
└───────────────────────────┬────────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼────────────────────────────────────┐
│                   NGINX (Reverse Proxy)                        │
│         Rate Limiting │ TLS Termination │ Security Headers     │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│              FASTAPI APP  (:8000, single process)              │
│                                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  ┌─────────┐ │
│  │  ingestion  │  │  retrieval  │  │ analysis  │  │ webhook │ │
│  │  module     │  │  module     │  │  module   │  │ module  │ │
│  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘  └────┬────┘ │
│         │                │               │              │      │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼─────┐  ┌────▼────┐ │
│  │  embedding  │  │  steering   │  │    llm    │  │feedback │ │
│  │  module     │  │  module     │  │  module   │  │ module  │ │
│  └─────────────┘  └─────────────┘  └───────────┘  └─────────┘ │
│                                                                │
│  app/observability/ ── OTel tracing + Prometheus metrics       │
│  shared/ ──────────── database, qdrant, cache, auth, models    │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│                       DATA SERVICES                            │
│  Qdrant :6333   │   PostgreSQL :5432   │   Redis :6379         │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                     OBSERVABILITY STACK                        │
│  Prometheus :9090  │  Grafana :3000  │  OTel Collector :4317   │
└────────────────────────────────────────────────────────────────┘
```

---

## Module Responsibilities

| Module | Path | Responsibility |
| --- | --- | --- |
| `ingestion` | `app/modules/ingestion/` | Accept tickets via REST, dedup by SHA-256 hash, queue background embed |
| `embedding` | `app/modules/embedding/` | BGE-large-en-v1.5 inference, format ticket text, upsert to Qdrant |
| `retrieval` | `app/modules/retrieval/` | Two-stage: PostgreSQL BM25 pre-filter (top-50) → Qdrant ANN rerank (top-10) |
| `steering` | `app/modules/steering/` | Load/apply/refresh steering vectors; auto-select by functional group |
| `llm` | `app/modules/llm/` | Format prompts, call GPT-4o with `response_format=json_object`, parse output |
| `analysis` | `app/modules/analysis/` | Orchestrate retrieval + steering + LLM; cache results 24h in Redis |
| `feedback` | `app/modules/feedback/` | Record engineer corrections, trigger steering refresh every 50 records |
| `webhook` | `app/modules/webhook/` | Normalize Jira webhook events to internal `TicketInput` format |

---

## Communication Patterns

```text
SYNC (HTTP):
  Client → Nginx → FastAPI → module router → module service → Response

ASYNC (BackgroundTasks — runs after response sent):
  POST /api/v1/tickets
    └─► save to PostgreSQL → return 201 → [BackgroundTask] embed + upsert Qdrant

  POST /api/v1/analysis/{id}/feedback
    └─► save feedback → count unprocessed → if ≥ 50 → [BackgroundTask] refresh steering vectors

INTERNAL (direct Python calls, no HTTP):
  analysis.service → retrieval.service → steering.service → embedding.service
```

---

## Folder Structure

```text
rag-ticket-ananlysis/
│
├── app/
│   ├── main.py                      # FastAPI app factory, lifespan, middleware wiring
│   ├── config.py                    # pydantic-settings Settings class
│   ├── dependencies.py              # Shared FastAPI deps (db, redis, qdrant, settings)
│   ├── modules/
│   │   ├── ingestion/
│   │   │   ├── router.py            # POST/GET/PATCH /api/v1/tickets
│   │   │   └── service.py           # Hash dedup, DB save, BackgroundTask trigger
│   │   ├── embedding/
│   │   │   ├── service.py           # BGE-large wrapper, text formatter, batch embed
│   │   │   └── tasks.py             # BackgroundTask: embed_and_index()
│   │   ├── retrieval/
│   │   │   ├── router.py            # POST /api/v1/tickets/search
│   │   │   └── service.py           # Two-stage BM25 (PostgreSQL) + Qdrant rerank
│   │   ├── steering/
│   │   │   ├── router.py            # GET /api/v1/steering, POST /api/v1/steering/compute
│   │   │   ├── service.py           # apply_steering(), auto_select(), refresh_from_feedback()
│   │   │   └── registry.py          # STEERING_VECTOR_CONFIG dict
│   │   ├── llm/
│   │   │   ├── service.py           # OpenAI async call + JSON response parse
│   │   │   └── prompts.py           # ANALYSIS_PROMPT_TEMPLATE
│   │   ├── analysis/
│   │   │   ├── router.py            # POST /api/v1/analysis/similarity, GET /api/v1/analysis/{id}
│   │   │   └── service.py           # Orchestrates retrieval + steering + LLM + caching
│   │   ├── feedback/
│   │   │   ├── router.py            # POST /api/v1/analysis/{id}/feedback
│   │   │   └── service.py           # Save record, threshold check, trigger steering refresh
│   │   └── webhook/
│   │       ├── router.py            # POST /api/v1/webhooks/jira
│   │       └── handler.py           # Jira payload → TicketInput normalizer + HMAC verify
│   └── observability/
│       ├── tracing.py               # OpenTelemetry setup (FastAPIInstrumentor, OTLP exporter)
│       └── metrics.py               # Prometheus custom counters/histograms
│
├── shared/
│   ├── models/
│   │   └── ticket.py                # All Pydantic models, enums, request/response types
│   ├── database/
│   │   ├── connection.py            # Async SQLAlchemy engine + session factory
│   │   ├── models.py                # SQLAlchemy ORM (all tables)
│   │   └── migrations/              # Alembic migrations
│   │       └── versions/
│   │           ├── 001_initial_schema.py
│   │           └── 002_add_feedback_and_tsvector.py
│   ├── qdrant/
│   │   ├── client.py                # QdrantClient factory
│   │   └── collections.py           # Collection init (tickets + steering_vectors)
│   ├── cache/
│   │   └── redis_client.py          # aioredis factory
│   ├── auth/
│   │   ├── jwt_handler.py           # JWT encode/decode + revocation check
│   │   └── api_key.py               # API key hashing + verification
│   └── utils/
│       ├── hashing.py               # SHA-256 text hash for dedup
│       └── logging.py               # Structured JSON logging (structlog)
│
├── infrastructure/
│   ├── nginx/nginx.conf             # Reverse proxy, rate limiting, TLS, security headers
│   ├── postgres/init.sql            # Applied at first boot by docker-compose
│   ├── qdrant/config.yaml
│   ├── redis/redis.conf
│   ├── prometheus/prometheus.yml    # Scrapes app:8000/metrics
│   ├── grafana/dashboards/
│   │   └── rag_system.json          # Auto-provisioned Grafana dashboard
│   └── otel/collector.yaml          # OTLP → logging exporter (add Jaeger/Tempo for prod)
│
├── scripts/
│   ├── init_db.py                   # Run Alembic migrations
│   ├── init_qdrant.py               # Create collections + payload indexes
│   ├── seed_tickets.py              # Insert sample data for dev/testing
│   └── compute_steering_vectors.py  # Bootstrap initial steering vectors from seed data
│
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_embedding.py
│   │   ├── test_steering.py
│   │   ├── test_retrieval.py        # BM25 + Qdrant two-stage logic
│   │   └── test_feedback.py         # Threshold trigger logic
│   └── integration/
│       ├── test_ingestion.py
│       ├── test_analysis_pipeline.py
│       └── test_feedback_triggers_refresh.py
│
├── docker-compose.yml               # app + postgres + qdrant + redis + prometheus + grafana + otel
├── Dockerfile                       # Single image (Python 3.11-slim, BGE model cached in volume)
├── requirements.txt
└── .env.example
```

---

## Key Design Principles

| Principle | Implementation |
| --- | --- |
| **Async by default** | FastAPI async/await, asyncpg, aioredis throughout |
| **Background for slow work** | Embedding (~100ms) and steering refresh use `BackgroundTasks` |
| **Config as code** | All settings via environment variables (12-factor) |
| **Migration-first** | All DB changes via Alembic migrations — never direct SQL edits |
| **Shared models once** | `shared/models/ticket.py` is the single source of truth for all Pydantic types |
| **Observable by default** | Every request traced via OTel; business metrics exported to Prometheus |

---

## Resource Requirements

### Development (docker-compose)

| Component | CPU | RAM | Notes |
| --- | --- | --- | --- |
| App (FastAPI + BGE model) | 2 cores | 10 GB | BGE-large ~4 GB; rest for app |
| Qdrant | 2 cores | 8 GB | Scales with collection size |
| PostgreSQL | 1 core | 2 GB | |
| Redis | 0.5 cores | 1 GB | |
| Prometheus + Grafana + OTel | 1 core | 1 GB | |
| **Total** | **~7 cores** | **~22 GB** | Single machine feasible |

### Scale-Out Path

When load demands it, extract these modules first (in order):

1. `embedding` — most resource-intensive; move to GPU nodes
2. `retrieval` — I/O bound; horizontal scaling is straightforward
3. `llm` — stateless HTTP calls; scales with OpenAI API quota

> **Next:** [02-data-layer.md](02-data-layer.md)
