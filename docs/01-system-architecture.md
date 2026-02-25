# Section 1: Full System Architecture

## 🏗️ Overview

The Ticket Intelligence System is a **production-grade, microservices-based RAG pipeline** that combines:
- **Semantic vector search** via Qdrant
- **Steering vector biasing** for domain-aware retrieval
- **LLM reasoning** for intelligent analysis and recommendations

---

## 🧩 Microservices Architecture

```
                          ┌───────────────────────────────────────────────────────┐
                          │                    EXTERNAL CLIENTS                   │
                          │   Jira Webhook │ Custom Backend │ Frontend Dashboard  │
                          └────────────────────────┬──────────────────────────────┘
                                                   │ HTTPS
                          ┌────────────────────────▼──────────────────────────────┐
                          │                    NGINX (Reverse Proxy)               │
                          │          Rate Limiting │ TLS Termination │ LB          │
                          └────────────────────────┬──────────────────────────────┘
                                                   │
                          ┌────────────────────────▼──────────────────────────────┐
                          │              API GATEWAY (FastAPI :8000)               │
                          │   Auth │ Routing │ Request Validation │ API Docs       │
                          └──┬──────────────┬──────────────┬────────────────┬──────┘
                             │              │              │                │
               ┌─────────────▼──┐   ┌──────▼─────┐  ┌────▼──────┐  ┌─────▼──────┐
               │  INGESTION SVC │   │ RETRIEVAL  │  │ ANALYSIS  │  │  WEBHOOK   │
               │    :8001       │   │ SVC :8002  │  │ SVC :8003 │  │  SVC :8004 │
               └───────┬────────┘   └─────┬──────┘  └────┬──────┘  └─────┬──────┘
                       │                  │              │                │
               ┌───────▼────────┐         │              │                │
               │  EMBEDDING SVC │◄────────┘              │                │
               │    :8005       │                        │                │
               └───────┬────────┘                        │                │
                       │                                 │                │
               ┌───────▼────────┐                        │                │
               │ STEERING VEC   │                        │                │
               │ SERVICE :8006  │                        │                │
               └───────┬────────┘                        │                │
                       │                                 │                │
          ┌────────────▼─────────────┐         ┌─────────▼──────────────┐│
          │   QDRANT VECTOR DB :6333 │         │  LLM REASONING SERVICE ││
          │   Collections:           │         │  :8007                 ││
          │   - tickets              │         │  GPT-4o / Claude /     ││
          │   - steering_vectors     │         │  Mistral               ││
          └────────────┬─────────────┘         └─────────┬──────────────┘│
                       │                                 │                │
          ┌────────────▼─────────────┐    ┌─────────────▼──────────────┐ │
          │  POSTGRESQL DB :5432     │    │    REDIS CACHE :6379        │ │
          │  - tickets               │    │    - embedding cache        │ │
          │  - audit_logs            │    │    - query cache            │ │
          │  - embedding_refs        │    │    - session store          │ │
          └──────────────────────────┘    └────────────────────────────┘ │
                                                                          │
                                          ┌───────────────────────────────┘
                                          │
                               ┌──────────▼────────────┐
                               │   MESSAGE QUEUE        │
                               │   Redis Streams        │
                               │   - ticket.ingested    │
                               │   - embedding.done     │
                               │   - analysis.request   │
                               └───────────────────────┘
```

---

## 📦 Service Responsibilities

### 1. API Gateway (`:8000`)
| Responsibility | Details |
|----------------|---------|
| Request routing | Routes to ingestion, retrieval, analysis services |
| Authentication | JWT validation, API key verification |
| Rate limiting | Per-client request throttling |
| Documentation | Swagger / OpenAPI auto-generated docs |
| CORS | Cross-origin request handling |

### 2. Ingestion Service (`:8001`)
| Responsibility | Details |
|----------------|---------|
| Ticket intake | Accept new/updated tickets via REST or Webhook |
| Validation | Schema validation, field sanitization |
| Deduplication pre-check | Fast hash-based duplicate screening |
| Queue publishing | Publish to Redis Streams for async processing |
| Status tracking | Track ingestion state in PostgreSQL |

### 3. Embedding Service (`:8005`)
| Responsibility | Details |
|----------------|---------|
| Text formatting | Combine ticket fields into optimal embedding text |
| Model inference | BGE-large-en-v1.5 (local) or OpenAI API |
| Batch processing | Process 100+ tickets per batch |
| Cache management | Store/retrieve embeddings from Redis |
| Qdrant upsert | Write vectors + payload to Qdrant |

### 4. Steering Vector Service (`:8006`)
| Responsibility | Details |
|----------------|---------|
| Vector computation | Calculate steering vectors from contrastive pairs |
| Vector storage | Store computed steering vectors in Qdrant |
| Query biasing | Apply steering offset to query embeddings |
| Multi-domain | Separate vectors per domain (BT, WiFi, etc.) |
| Refresh scheduling | Recompute vectors as new tickets accumulate |

### 5. Retrieval Service (`:8002`)
| Responsibility | Details |
|----------------|---------|
| Query embedding | Embed incoming query ticket |
| Steering application | Apply relevant steering vectors |
| Qdrant search | Execute filtered vector similarity search |
| Metadata join | Join Qdrant results with PostgreSQL metadata |
| Result ranking | Re-rank results by score + business rules |

### 6. LLM Reasoning Service (`:8007`)
| Responsibility | Details |
|----------------|---------|
| Prompt construction | Build structured prompts with retrieved context |
| LLM inference | Call OpenAI/Anthropic/local model |
| Response parsing | Extract structured analysis from LLM output |
| Duplicate detection | Identify if new ticket duplicates existing ones |
| Recommendation | Suggest actions based on historical patterns |

### 7. Analysis Service (`:8003`)
| Responsibility | Details |
|----------------|---------|
| Orchestration | Coordinate retrieval + LLM pipeline |
| Result aggregation | Combine vector results + LLM analysis |
| Caching | Cache analysis results for repeated queries |
| Audit logging | Record all analysis events |

### 8. Webhook Service (`:8004`)
| Responsibility | Details |
|----------------|---------|
| Jira integration | Receive Jira webhook events |
| External polling | Poll external ticket systems |
| Event normalization | Transform external formats to internal schema |
| Retry logic | Handle failed webhook deliveries |

---

## 🔄 Communication Flow

```
SYNC (HTTP/REST):
  Client → Nginx → API Gateway → Service → Response

ASYNC (Event-Driven):
  Ingestion → Redis Stream → Embedding Worker → Qdrant
                          → Steering Worker → Qdrant
                          → Analysis Worker → LLM → PostgreSQL

INTERNAL:
  Service-to-Service via HTTP (service mesh in k8s: mTLS)
```

---

## 🚀 Deployment Architecture

```
┌──────────────────────────────────────────────────────┐
│                    DOCKER HOST / K8S CLUSTER          │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │   nginx     │  │  api-gw     │  │  ingestion   │  │
│  │  (1 pod)    │  │  (2 pods)   │  │  (2 pods)    │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │  embedding  │  │  retrieval  │  │  llm-svc     │  │
│  │  (3 pods)   │  │  (2 pods)   │  │  (2 pods)    │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │   qdrant    │  │ postgresql  │  │    redis     │  │
│  │  (cluster)  │  │  (primary + │  │  (sentinel)  │  │
│  │             │  │   replica)  │  │              │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

## 📈 Scalability Considerations

| Concern | Strategy |
|---------|----------|
| Embedding throughput | Horizontal scaling of embedding service (GPU nodes) |
| Search latency | Qdrant HNSW index + Redis query cache |
| LLM costs | Response caching, smaller models for pre-screening |
| Ingestion spikes | Redis Stream buffering, worker auto-scaling |
| Storage | Qdrant distributed mode for millions of vectors |
| Database | PostgreSQL read replicas for query distribution |

---

## ⚙️ Resource Requirements

| Service | CPU | RAM | Notes |
|---------|-----|-----|-------|
| API Gateway | 2 cores | 2 GB | Stateless, easy to scale |
| Embedding Service | 4 cores / GPU | 8 GB | GPU preferred for BGE-large |
| Steering Service | 2 cores | 4 GB | CPU-bound vector math |
| Retrieval Service | 2 cores | 2 GB | Mostly I/O bound |
| LLM Service | 2 cores | 2 GB | Calls external API |
| Qdrant | 4 cores | 16 GB | Depends on collection size |
| PostgreSQL | 4 cores | 8 GB | With connection pooling |
| Redis | 2 cores | 4 GB | Cache + queue |

---

> **Next:** [Section 2 — Vector Database Design (Qdrant)](02-qdrant-design.md)
