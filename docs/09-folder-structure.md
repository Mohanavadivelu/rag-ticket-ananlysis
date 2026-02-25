# Section 9: Enterprise Folder Structure

## 📁 Complete Project Layout

```
ticket-intelligence-system/
│
├── README.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .gitignore
│
├── docs/                              # Architecture & design documentation
│   ├── 01-system-architecture.md
│   ├── 02-qdrant-design.md
│   ├── 03-embedding-design.md
│   ├── 04-steering-vectors.md
│   ├── 05-retrieval-pipeline.md
│   ├── 06-llm-reasoning.md
│   ├── 07-database-design.md
│   ├── 08-api-design.md
│   ├── 09-folder-structure.md
│   ├── 10-deployment.md
│   ├── 11-performance.md
│   ├── 12-integration.md
│   ├── 13-security.md
│   ├── 14-end-to-end-flow.md
│   ├── 15-advanced-features.md
│   └── 16-sample-code.md
│
├── services/                          # All microservices
│   │
│   ├── api_gateway/                   # Main FastAPI entry point (:8000)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py                    # FastAPI app factory
│   │   ├── config.py                  # Settings (pydantic-settings)
│   │   ├── dependencies.py            # FastAPI dependencies (auth, db, etc.)
│   │   ├── middleware/
│   │   │   ├── auth.py                # JWT + API key middleware
│   │   │   ├── rate_limiter.py        # Sliding window rate limiting
│   │   │   ├── request_id.py          # X-Request-ID injection
│   │   │   └── cors.py
│   │   ├── routers/
│   │   │   ├── tickets.py             # /api/v1/tickets endpoints
│   │   │   ├── analysis.py            # /api/v1/analysis endpoints
│   │   │   ├── search.py              # /api/v1/tickets/search
│   │   │   ├── steering.py            # /api/v1/steering endpoints
│   │   │   ├── webhooks.py            # /api/v1/webhooks (Jira, etc.)
│   │   │   └── health.py              # /health, /ready, /metrics
│   │   └── schemas/
│   │       ├── ticket.py              # Pydantic request/response models
│   │       ├── analysis.py
│   │       ├── search.py
│   │       └── common.py              # Shared models (pagination, error, etc.)
│   │
│   ├── embedding_service/             # Embedding generation (:8005)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── embedding_service.py       # Core embedding logic
│   │   ├── models/
│   │   │   ├── bge_large.py           # BGE-large-en-v1.5 wrapper
│   │   │   └── openai_embedder.py     # OpenAI embedding wrapper
│   │   ├── pipeline/
│   │   │   ├── preprocessor.py        # Text cleaning, truncation
│   │   │   ├── formatter.py           # Ticket → embedding text formatter
│   │   │   └── batch_processor.py     # Async batch embedding
│   │   └── workers/
│   │       └── embedding_worker.py    # Redis Stream consumer
│   │
│   ├── steering_service/              # Steering vector management (:8006)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py
│   │   ├── steering_service.py        # Core steering logic
│   │   ├── registry.py                # STEERING_VECTOR_CONFIG
│   │   ├── contrastive_pairs.py       # Pair generation from DB
│   │   └── scheduler.py               # Weekly refresh scheduler
│   │
│   ├── retrieval_service/             # Search & retrieval (:8002)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py
│   │   ├── retrieval_service.py       # Core retrieval pipeline
│   │   ├── reranker.py                # Business-logic re-ranking
│   │   └── search_config.py           # SearchConfig modes
│   │
│   ├── llm_service/                   # LLM reasoning (:8007)
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py
│   │   ├── llm_service.py             # LLM call + response parsing
│   │   ├── prompt_templates.py        # All prompt templates
│   │   ├── context_formatter.py       # Format retrieved tickets for LLM
│   │   └── response_parser.py         # Parse + validate LLM JSON output
│   │
│   └── webhook_service/               # External integrations (:8004)
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py
│       ├── jira_handler.py            # Jira webhook receiver
│       ├── normalizer.py              # External → internal ticket format
│       └── retry_handler.py           # Exponential backoff retry
│
├── shared/                            # Shared code across services
│   ├── __init__.py
│   ├── database/
│   │   ├── __init__.py
│   │   ├── connection.py              # Async PostgreSQL (asyncpg + SQLAlchemy)
│   │   ├── models.py                  # SQLAlchemy ORM models
│   │   └── migrations/                # Alembic migrations
│   │       ├── env.py
│   │       ├── alembic.ini
│   │       └── versions/
│   │           ├── 001_initial_schema.py
│   │           ├── 002_add_analysis_results.py
│   │           └── 003_add_steering_metadata.py
│   ├── qdrant/
│   │   ├── __init__.py
│   │   ├── client.py                  # Qdrant client factory
│   │   ├── collections.py             # Collection setup & management
│   │   └── operations.py              # upsert, search, delete helpers
│   ├── cache/
│   │   ├── __init__.py
│   │   └── redis_client.py            # Async Redis client factory
│   ├── models/
│   │   ├── __init__.py
│   │   ├── ticket.py                  # Shared Pydantic models
│   │   ├── analysis.py
│   │   └── steering.py
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_handler.py             # JWT encode/decode
│   │   └── api_key.py                 # API key hashing & validation
│   └── utils/
│       ├── __init__.py
│       ├── hashing.py                 # SHA-256 text hash for dedup
│       ├── logging.py                 # Structured JSON logging
│       └── metrics.py                 # Prometheus metrics helpers
│
├── infrastructure/                    # Infrastructure configuration
│   ├── nginx/
│   │   ├── nginx.conf
│   │   └── ssl/                       # TLS certificates (gitignored)
│   ├── postgres/
│   │   ├── init.sql                   # Initial schema
│   │   └── seed_data.sql              # Sample data for dev
│   ├── qdrant/
│   │   └── config.yaml                # Qdrant configuration
│   └── redis/
│       └── redis.conf
│
├── scripts/                           # Utility scripts
│   ├── init_db.py                     # Initialize PostgreSQL schema
│   ├── init_qdrant.py                 # Create Qdrant collections + indexes
│   ├── seed_tickets.py                # Insert sample tickets for dev/testing
│   ├── compute_steering_vectors.py    # Bootstrap steering vectors
│   ├── reindex_all.py                 # Re-embed all tickets (for model upgrades)
│   └── export_tickets.py              # Export tickets to JSON/CSV
│
├── tests/                             # Test suite
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_embedding.py
│   │   ├── test_steering.py
│   │   ├── test_retrieval.py
│   │   └── test_llm_service.py
│   ├── integration/
│   │   ├── test_api_tickets.py
│   │   ├── test_api_search.py
│   │   ├── test_api_analysis.py
│   │   └── test_qdrant_operations.py
│   └── e2e/
│       └── test_full_pipeline.py
│
└── config/                            # Environment configurations
    ├── .env.development
    ├── .env.staging
    └── .env.production
```

---

## 🔑 Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Separation of Concerns** | Each service owns exactly one responsibility |
| **Shared Nothing** | Services communicate only via HTTP or Redis Streams |
| **Shared Libraries** | Common code in `shared/` — imported, never duplicated |
| **Config as Code** | All settings via environment variables (12-factor app) |
| **Migration-First** | All DB changes via Alembic migrations — never direct SQL |
| **Test Pyramid** | Unit > Integration > E2E tests |

---

> **Next:** [Section 10 — Deployment Design](10-deployment.md)
