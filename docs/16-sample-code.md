# Section 16: Sample Code Structure

## 💻 Overview

This section provides the complete, runnable code structure for the Ticket Intelligence System. All key files are implemented in the repository.

---

## 📂 Key Implementation Files

| File | Description |
|------|-------------|
| `services/api_gateway/main.py` | FastAPI application entry point |
| `services/api_gateway/routers/tickets.py` | Ticket CRUD endpoints |
| `services/api_gateway/routers/analysis.py` | Analysis endpoints |
| `services/embedding_service/embedding_service.py` | Embedding pipeline |
| `services/steering_service/steering_service.py` | Steering vector logic |
| `services/retrieval_service/retrieval_service.py` | Retrieval pipeline |
| `services/llm_service/llm_service.py` | LLM reasoning |
| `shared/database/models.py` | SQLAlchemy ORM models |
| `shared/qdrant/collections.py` | Qdrant collection management |
| `scripts/init_qdrant.py` | Initialize Qdrant collections |
| `scripts/seed_tickets.py` | Seed sample data |
| `docker-compose.yml` | Full stack deployment |

---

## 🗂️ File Index with Descriptions

### API Gateway

```
services/api_gateway/
├── main.py          ← FastAPI app factory, lifespan hooks, middleware registration
├── config.py        ← Pydantic Settings for all env vars
├── dependencies.py  ← get_db(), get_redis(), get_qdrant(), get_current_user()
├── routers/
│   ├── tickets.py   ← POST/GET/PATCH /api/v1/tickets
│   ├── analysis.py  ← POST /api/v1/analysis/similarity
│   ├── search.py    ← POST /api/v1/tickets/search
│   ├── steering.py  ← GET/POST /api/v1/steering
│   └── health.py    ← GET /health, /ready
└── schemas/
    ├── ticket.py    ← TicketInput, TicketResponse, TicketListResponse
    ├── analysis.py  ← AnalysisRequest, AnalysisResponse
    └── search.py    ← SearchRequest, SearchResponse, SearchResult
```

### Embedding Service

```
services/embedding_service/
├── main.py                    ← FastAPI app + POST /embed endpoint
├── embedding_service.py       ← EmbeddingService class (main logic)
├── models/
│   └── bge_large.py           ← BGELargeEmbedder (singleton model)
└── pipeline/
    ├── formatter.py           ← format_ticket_for_embedding()
    └── preprocessor.py        ← clean_text(), truncate_tokens()
```

### Steering Service

```
services/steering_service/
├── main.py            ← FastAPI app + GET/POST endpoints
├── steering_service.py← SteeringService (apply, select, compute)
└── registry.py        ← STEERING_VECTOR_CONFIG dict
```

### Shared Library

```
shared/
├── database/
│   ├── connection.py  ← AsyncSession factory, engine
│   └── models.py      ← Ticket, AnalysisResult, User, AuditLog ORM models
├── qdrant/
│   ├── client.py      ← QdrantClient factory
│   └── collections.py ← create_collections(), create_indexes()
└── models/
    └── ticket.py      ← Shared Pydantic models (TicketInput, EnrichedTicket, etc.)
```

---

## 🚀 Quick Start

```bash
# 1. Clone and configure
git clone https://github.com/your-org/ticket-intelligence-system
cd ticket-intelligence-system
cp .env.example .env
# Edit .env and add OPENAI_API_KEY, JWT_SECRET

# 2. Start all services
docker-compose up -d

# 3. Wait for services to be healthy
docker-compose ps

# 4. Initialize database schema
docker-compose exec api_gateway python -c "
from shared.database.connection import init_db
import asyncio
asyncio.run(init_db())
"

# 5. Initialize Qdrant collections
docker-compose exec api_gateway python scripts/init_qdrant.py

# 6. Seed sample tickets
docker-compose exec api_gateway python scripts/seed_tickets.py

# 7. Compute initial steering vectors
docker-compose exec api_gateway python scripts/compute_steering_vectors.py

# 8. Open API docs
echo "API docs: http://localhost:8000/docs"
```

---

## 🧪 Quick Test

```bash
# Test ticket ingestion
curl -X POST http://localhost:8000/api/v1/tickets \
  -H "X-API-Key: dev_test_key" \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": "TKT-2024-99001",
    "functional_group": "Bluetooth",
    "title": "Test BT ticket",
    "description": "Bluetooth audio drops when phone call received on Android 14",
    "status": "New",
    "severity": "High"
  }'

# Test similarity search
curl -X POST http://localhost:8000/api/v1/tickets/search \
  -H "X-API-Key: dev_test_key" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"description": "BT audio drops during call", "functional_group": "Bluetooth"},
    "options": {"top_k": 5}
  }'

# Test full analysis
curl -X POST http://localhost:8000/api/v1/analysis/similarity \
  -H "X-API-Key: dev_test_key" \
  -H "Content-Type: application/json" \
  -d '{"ticket_id": "TKT-2024-99001", "analysis_mode": "full"}'
```

---

> All actual implementation files are in the `services/` and `shared/` directories.
> See each subdirectory for the complete, runnable Python code.
