# Deployment, Performance & End-to-End Flow

## Overview

The system runs as a **single application container** plus separate data service containers. The observability stack (Prometheus, Grafana, OpenTelemetry) is included in docker-compose.

---

## Docker Compose

```yaml
version: "3.9"

services:
  # ─── Application ────────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - JWT_SECRET=${JWT_SECRET}
      - OTEL_ENDPOINT=http://otel-collector:4317
      - EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
      - LLM_MODEL=gpt-4o
      - BM25_CANDIDATE_COUNT=50
      - QDRANT_RERANK_TOP_K=10
      - FEEDBACK_REFRESH_THRESHOLD=50
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
      qdrant:   { condition: service_started }
    volumes:
      - embedding_models:/app/models   # BGE model cached between restarts
    deploy:
      resources:
        limits:    { memory: 10G }
        reservations: { memory: 6G }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ─── Vector Database ─────────────────────────────────────────────
  qdrant:
    image: qdrant/qdrant:v1.9.0
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_storage:/qdrant/storage
      - ./infrastructure/qdrant/config.yaml:/qdrant/config/production.yaml:ro
    restart: unless-stopped

  # ─── Relational Database ──────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: tickets
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infrastructure/postgres/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    command:
      - postgres
      - -c
      - max_connections=200
      - -c
      - shared_buffers=512MB
      - -c
      - effective_cache_size=2GB
    restart: unless-stopped

  # ─── Cache ────────────────────────────────────────────────────────
  redis:
    image: redis:7.2-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./infrastructure/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ─── Observability ────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.52.0
    ports:
      - "9090:9090"
    volumes:
      - ./infrastructure/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d

  grafana:
    image: grafana/grafana:10.4.2
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_PATHS_PROVISIONING: /etc/grafana/provisioning
    volumes:
      - grafana_data:/var/lib/grafana
      - ./infrastructure/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    depends_on: [prometheus]

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.102.0
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    volumes:
      - ./infrastructure/otel/collector.yaml:/etc/otel/config.yaml:ro
    command: ["--config=/etc/otel/config.yaml"]

volumes:
  postgres_data:
  qdrant_storage:
  redis_data:
  embedding_models:
  prometheus_data:
  grafana_data:
```

---

## Dockerfile

```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ─── Production stage ─────────────────────────────────────────────
FROM python:3.11-slim AS production

WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .

RUN adduser --disabled-password --gecos "" appuser
USER appuser

ENV PATH=/root/.local/bin:$PATH
ENV PYTHONPATH=/app

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000",
     "--workers", "4", "--loop", "uvloop", "--http", "httptools"]
```

---

## Nginx Configuration

```nginx
worker_processes auto;
events { worker_connections 4096; }

http {
    limit_req_zone $binary_remote_addr zone=api:10m      rate=100r/m;
    limit_req_zone $binary_remote_addr zone=search:10m   rate=30r/m;
    limit_req_zone $binary_remote_addr zone=analysis:10m rate=10r/m;

    upstream app { server app:8000; keepalive 32; }

    server {
        listen 80;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        ssl_certificate     /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        ssl_protocols       TLSv1.2 TLSv1.3;

        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        location /api/v1/tickets/search {
            limit_req zone=search burst=10 nodelay;
            proxy_pass http://app;
            proxy_read_timeout 30s;
        }

        location /api/v1/analysis {
            limit_req zone=analysis burst=5 nodelay;
            proxy_pass http://app;
            proxy_read_timeout 60s;
        }

        location / {
            limit_req zone=api burst=50 nodelay;
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

---

## Observability Stack

### OpenTelemetry Tracing

```python
# app/observability/tracing.py
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

def setup_tracing(app: FastAPI) -> None:
    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint=settings.OTEL_ENDPOINT, insecure=True)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    FastAPIInstrumentor.instrument_app(app)   # Auto-traces all HTTP requests
```

Manual spans for custom operations:

```python
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("retrieval.bm25_search") as span:
    span.set_attribute("bm25.candidate_count", len(results))
    bm25_ids = await self.bm25_search(...)

with tracer.start_as_current_span("llm.analyze") as span:
    span.set_attribute("llm.model", self.model)
    result = await self.client.chat.completions.create(...)
```

### Prometheus Metrics

```python
# app/observability/metrics.py
from prometheus_fastapi_instrumentator import Instrumentator
from prometheus_client import Counter, Histogram

# HTTP metrics auto-collected by Instrumentator (latency, status codes, etc.)

# Custom business metrics
TICKETS_INGESTED    = Counter("tickets_ingested_total",       "Tickets ingested",       ["functional_group"])
ANALYSES_COMPLETED  = Counter("analyses_completed_total",     "LLM analyses completed")
EMBEDDING_LATENCY   = Histogram("embedding_latency_seconds",  "BGE embedding latency")
BM25_CANDIDATES     = Histogram("retrieval_bm25_candidates",  "BM25 pre-filter count",  buckets=[5,10,20,30,50])
LLM_CALLS           = Counter("llm_calls_total",              "OpenAI API calls",       ["model"])
FEEDBACK_SUBMITTED  = Counter("feedback_submitted_total",     "Feedback records",       ["is_correct"])
STEERING_REFRESHES  = Counter("steering_refreshes_total",     "Steering vector updates",["vector_name"])

def setup_metrics(app: FastAPI) -> None:
    Instrumentator().instrument(app).expose(app, endpoint="/metrics")
```

### Prometheus Scrape Config

```yaml
# infrastructure/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: rag_app
    static_configs:
      - targets: ["app:8000"]
    metrics_path: /metrics
```

### Grafana Dashboard Panels

Auto-provisioned at `http://localhost:3000` (admin/admin):

| Panel | Metric |
|-------|--------|
| Tickets Ingested/min | `rate(tickets_ingested_total[1m])` |
| Analysis P95 Latency | `histogram_quantile(0.95, rate(http_request_duration_bucket{handler="/api/v1/analysis/similarity"}[5m]))` |
| BM25 Candidate Count | `histogram_quantile(0.50, rate(retrieval_bm25_candidates_bucket[5m]))` |
| LLM Calls/min | `rate(llm_calls_total[1m])` |
| Feedback Rate | `rate(feedback_submitted_total[1m])` by `is_correct` |
| Steering Refreshes | `increase(steering_refreshes_total[1d])` |
| Embedding P99 Latency | `histogram_quantile(0.99, rate(embedding_latency_seconds_bucket[5m]))` |

---

## Performance Targets

| Operation | Target | Typical |
|-----------|--------|---------|
| Ticket ingestion (sync) | < 50ms | ~10ms |
| BGE-large embedding (GPU) | < 50ms | ~40ms |
| PostgreSQL BM25 search | < 5ms | ~2ms |
| Qdrant ANN search (top-10) | < 15ms | ~8ms |
| Full retrieval pipeline | < 100ms | ~55ms |
| Full RAG analysis | < 3s | ~1.9s |
| Batch embed (100 tickets) | < 3s | ~2s |

### Latency Breakdown (Full Analysis, P95)

```
JWT validation:               2ms
Ticket DB lookup (cached):    2ms
Embedding (GPU, cached=1ms): 40ms
BM25 search:                  2ms
Steering vector apply:        1ms
Qdrant HNSW search:           8ms
PostgreSQL metadata:          3ms   (Redis cache hit)
LLM API (GPT-4o):          1800ms
Response serialization:       2ms
                           ──────
Total P95:               ~1860ms   (well under 3s target)
```

### Cost Optimization

| Strategy | Savings |
|----------|---------|
| BGE-large local vs OpenAI API | 100% embedding cost |
| GPT-4o-mini for pre-screening | 95% LLM cost (use for low-confidence cases) |
| 24h analysis cache | ~60% LLM calls avoided |
| Scalar quantization in Qdrant | 4x memory reduction |
| Batch embedding (32 per call) | 30x throughput increase |

---

## First-Time Setup

```bash
# 1. Copy environment config
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, JWT_SECRET

# 2. Start data services first
docker-compose up -d postgres redis qdrant

# 3. Initialize schema and collections
docker-compose run --rm app python scripts/init_db.py
docker-compose run --rm app python scripts/init_qdrant.py

# 4. Seed development data
docker-compose run --rm app python scripts/seed_tickets.py

# 5. Bootstrap steering vectors from seed data
docker-compose run --rm app python scripts/compute_steering_vectors.py

# 6. Start full stack
docker-compose up -d

# 7. Verify
curl http://localhost:8000/health
curl http://localhost:8000/ready
open http://localhost:3000   # Grafana (admin/admin)
open http://localhost:9090   # Prometheus
```

---

## End-to-End Flow Example

**Scenario:** Engineer submits ticket "BT audio drops when receiving call on Android 14"

### Phase 1: Ticket Submission (T+0ms → T+10ms)

```
POST /api/v1/tickets  →  Nginx  →  FastAPI
  │
  ├── Pydantic validation
  ├── SHA-256 hash dedup check (SELECT WHERE text_hash = ?)
  ├── INSERT into tickets table
  ├── Return 201 with embedding_status: "queued"
  └── [BackgroundTask] → embed_and_index(ticket_id)
```

### Phase 2: Async Embedding (T+100ms → T+200ms, after response)

```
BackgroundTask: embed_and_index()
  │
  ├── format_ticket_for_embedding() → structured text
  ├── Redis cache check → MISS (new ticket)
  ├── BGE-large-en-v1.5 inference → [0.023, -0.412, 0.891, ...] (1024-dim)
  ├── Qdrant upsert (vector + payload)
  └── UPDATE tickets SET embedding_status='indexed', embedding_id=...
```

### Phase 3: Analysis Request (T+200ms)

```
POST /api/v1/analysis/similarity {"ticket_id": "TKT-2024-01247"}
```

**Step 3a — Stage 1 BM25 (T+202ms):**
```sql
SELECT ticket_id FROM tickets, plainto_tsquery('english', 'bluetooth audio drops call') q
WHERE search_vector @@ q ORDER BY ts_rank_cd DESC LIMIT 50
-- Returns 50 candidate IDs
```

**Step 3b — Embed + Steer (T+244ms):**
```
embed_query("bluetooth audio drops call") → q (1024-dim)
auto_select_vectors("Bluetooth") → ["bluetooth", "rejected", "firmware_root_cause"]
q_steered = normalize(q + 0.3×s_bluetooth + 0.5×s_rejected + 0.4×s_firmware)
```

**Step 3c — Stage 2 Qdrant (T+252ms):**
```python
qdrant.search(
    query_vector=q_steered,
    filter=Filter(must=[FieldCondition("ticket_id", MatchAny(any=bm25_50_ids))]),
    limit=10, score_threshold=0.65
)
# Returns:
# Rank 1: TKT-2024-00847  score=0.923  status=Fixed    "BT HFP routing fail Android 12"
# Rank 2: TKT-2024-00612  score=0.889  status=Rejected "BT audio dropout Android 12 known issue"
# Rank 3: TKT-2024-01180  score=0.871  status=Fixed    "HFP A2DP conflict iOS 17"
# ...
```

**Step 3d — Metadata Join (T+255ms):**
```
Fetch full descriptions + resolutions from PostgreSQL (Redis cache hit for hot tickets)
```

### Phase 4: LLM Reasoning (T+300ms → T+2100ms)

```
Prompt sent to GPT-4o:
  - Role: Senior embedded systems engineer
  - New ticket: TKT-2024-01247 (full text)
  - Context: 10 similar historical tickets with resolutions
  - Instructions: analyze duplicate/rejection/root_cause/action/reference

GPT-4o response (~1800ms):
  → is_duplicate: false
  → rejection_probability: 15% (recommendation: Accept)
  → root_cause: firmware_bug (confidence: 87%)
  → assignee_team: firmware-team
  → resolution_reference: TKT-2024-00847 (port v3.2.2 HFP fix to Android 14)
  → confidence_score: 87
```

### Phase 5: Store + Return (T+2100ms)

```
INSERT into analysis_results (ticket_id, is_duplicate, rejection_probability, ...)
SET analysis:TKT-2024-01247:v1 in Redis (TTL: 24h)
Return AnalysisResult JSON to caller
```

### Timeline Summary

| Phase | Time | Key Action |
|-------|------|-----------|
| Ticket submission | 0–10ms | Validate + DB insert + return 201 |
| Async embedding | 100–200ms | BGE-large + Qdrant upsert (background) |
| BM25 pre-filter | 202–204ms | PostgreSQL tsvector → 50 candidates |
| Embed + steer | 204–245ms | Query embedding + steering vectors |
| Qdrant ANN | 245–253ms | Scoped to 50 candidates → top-10 |
| Metadata join | 253–256ms | Redis cache hit |
| LLM reasoning | 300–2100ms | GPT-4o with 10-ticket context |
| Store + return | 2100ms | PostgreSQL + Redis + response |
| **Total** | **~2.1s** | **Full analysis** |

---

## Scale-Out Path

When load demands it, extract modules in this order:

1. **Embedding** — most resource-intensive; move to GPU nodes first
2. **Retrieval** — I/O bound; horizontal scaling is trivial
3. **LLM** — stateless HTTP calls; scales with OpenAI API quota

For production Kubernetes deployment, refer to the resource estimates in [01-architecture.md](01-architecture.md).
