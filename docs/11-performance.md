# Section 11: Performance & Scalability

## ⚡ Overview

The system is designed to handle **millions of tickets** at **sub-100ms retrieval latency** through layered caching, HNSW indexing, async processing, and horizontal scaling.

---

## 📊 Performance Targets

| Operation | Target Latency | Notes |
|-----------|---------------|-------|
| Single ticket embed | < 50ms | GPU-accelerated BGE-large |
| Qdrant vector search | < 10ms | HNSW index, top-10 |
| PostgreSQL metadata fetch | < 5ms | Redis-cached, indexed |
| Full retrieval pipeline | < 100ms | embed + steer + search + join |
| Full RAG analysis | < 3s | retrieval + LLM (GPT-4o) |
| Batch embed (100 tickets) | < 2s | BGE-large batch=32 |

---

## 🗂️ Indexing Strategy

### HNSW Parameters for Scale

```python
# Optimal HNSW settings for 1M+ ticket collections
HnswConfigDiff(
    m=16,              # 16 edges per node → good recall/memory balance
    ef_construct=200,  # Higher = better graph quality at index time
    full_scan_threshold=10000  # Use full scan only for very small collections
)

# At search time:
SearchParams(
    hnsw_ef=128,       # Higher ef = better recall, more latency
    exact=False        # ANN not exact (ANN is 10x faster, <1% accuracy loss)
)
```

### Index Size vs Recall Trade-off

| ef_search | Recall@10 | Search Latency |
|-----------|-----------|----------------|
| 32 | ~91% | 3ms |
| 64 | ~95% | 6ms |
| 128 | ~98% | 10ms ← **recommended** |
| 256 | ~99% | 18ms |

---

## 🗄️ Caching Strategy

### 3-Layer Cache Architecture

```
Layer 1: In-Process Cache (Python dict + TTLCache)
  - Steering vectors: loaded once, refreshed weekly
  - Hot ticket metadata: LRU cache, max 1000 entries
  - TTL: 1 hour

Layer 2: Redis Cache (shared across service instances)
  - Embedding cache: hash(text) → vector, TTL: 7 days
  - Ticket metadata cache: ticket_id → JSON, TTL: 1 hour
  - Analysis cache: ticket_id:version → JSON, TTL: 24 hours
  - Query result cache: hash(query+filters) → results, TTL: 5 min

Layer 3: Qdrant Payload Cache (automatic)
  - Qdrant caches hot segments in memory
  - Most-accessed vectors stay in RAM
```

### Cache Key Design

```python
# Embedding cache
embed_key = f"embed:{model_version}:{sha256(text)[:16]}"

# Analysis cache
analysis_key = f"analysis:{ticket_id}:{analysis_version}"

# Search result cache (for repeated identical queries)
search_key = f"search:{sha256(json.dumps(query, sort_keys=True))[:16]}"
```

---

## 📈 Scaling to Millions of Tickets

### Qdrant Distributed Mode

```yaml
# For 10M+ tickets: Qdrant cluster with sharding
# Each shard: 2M vectors × 1024 dims × 4 bytes = ~8GB RAM

qdrant_node_1:
  collections:
    tickets:
      shards: 4        # Distribute across 4 nodes
      replication: 2   # 2 copies per shard (HA)
```

### PostgreSQL Partitioning

```sql
-- Partition tickets by created_at (monthly) for 100M+ rows
CREATE TABLE tickets (
    id UUID,
    created_at TIMESTAMPTZ NOT NULL,
    -- ... fields
) PARTITION BY RANGE (created_at);

CREATE TABLE tickets_2024_q1 PARTITION OF tickets
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE tickets_2024_q2 PARTITION OF tickets
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### Async Ingestion Pipeline

```
Ticket API (sync: 201 Created immediately)
      │
      ▼
Redis Stream: ticket.ingested
      │
      ├── Embedding Worker (async, 3 instances)
      │         └── Batch 32 tickets → embed → upsert Qdrant
      │
      └── Analysis Worker (async, 2 instances)
                └── Retrieve → LLM → store analysis
```

---

## 🔥 Latency Breakdown Analysis

```
Typical Full Analysis Request (P95):

  Client → Nginx:              1ms  (network)
  Nginx → API Gateway:         1ms  (proxy)
  API Gateway routing:         2ms  (JWT validation)
  Ticket lookup (cached):      2ms  (Redis hit)
  Embedding generation:       40ms  (BGE-large GPU, cached=1ms)
  Steering vector apply:       1ms  (in-process, loaded)
  Qdrant HNSW search:          8ms  (ef=128, 1M vectors)
  PostgreSQL metadata:         3ms  (Redis cache miss: 15ms)
  LLM API call (GPT-4o):    1800ms  (OpenAI network + inference)
  Response serialization:      2ms
                           ──────
  Total P95:               ~1860ms  (well under 3s target)
```

---

## 📉 Cost Optimization

| Strategy | Savings | Trade-off |
|----------|---------|-----------|
| Use GPT-4o-mini for pre-screening | 95% cost reduction | Slightly lower quality |
| Cache analysis results (24h) | ~60% LLM calls avoided | Stale results possible |
| Scalar quantization in Qdrant | 4x memory reduction | <1% accuracy loss |
| BGE-large local vs OpenAI API | 100% embedding cost saved | Requires GPU server |
| Batch embedding (32x) | 30x throughput increase | Small latency increase |

---

> **Next:** [Section 12 — Integration Design](12-integration.md)
