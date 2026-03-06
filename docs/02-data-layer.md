# Data Layer: Embeddings + Qdrant

## Overview

The data layer has two tightly-coupled components:

1. **Embedding Service** — converts ticket text to 1024-dimensional vectors using BGE-large-en-v1.5
2. **Qdrant** — stores those vectors and enables sub-10ms approximate nearest-neighbor (ANN) search

---

## Embedding Strategy

### Text Format

Tickets are formatted as structured text before embedding. **Field order matters** — transformer models weight earlier tokens more heavily, so the most semantically important fields come first.

```python
def format_ticket_for_embedding(ticket: dict) -> str:
    parts = []
    parts.append(f"Functional Group: {ticket.get('functional_group', 'Unknown')}")
    parts.append(f"Description: {ticket.get('description', '')}")
    if ticket.get('analysis_notes'):
        parts.append(f"Analysis: {ticket['analysis_notes']}")
    if ticket.get('resolution'):
        parts.append(f"Resolution: {ticket['resolution']}")
    parts.append(f"Status: {ticket.get('status', 'New')}")
    if ticket.get('tags'):
        parts.append(f"Tags: {', '.join(ticket['tags'])}")
    return "\n".join(parts)
```

| Field | Position | Reason |
|-------|----------|--------|
| Functional Group | 1st | Anchors domain context immediately |
| Description | 2nd | Core semantic content — most important |
| Analysis Notes | 3rd | Technical root cause hints |
| Resolution | 4th | Fix pattern; helps match similar resolved tickets |
| Status | 5th | Outcome signal used by steering vectors |
| Tags | Last | Keyword reinforcement |

**Example output:**

```
Functional Group: Bluetooth
Description: Device disconnects Bluetooth audio when incoming call received. Audio
             routes to phone speaker instead of car. Android 14, firmware v3.3.0.
Analysis: HFP profile negotiation failure in BT logs. A2DP not suspended before
          HFP activation.
Resolution: Updated HFP stack to sequence A2DP suspend before HFP connect. v3.2.2.
Status: Fixed
Tags: bluetooth, hfp, a2dp, audio-routing, android
```

---

### Embedding Model

**Default: BGE-large-en-v1.5** (recommended — local, free, 1024-dim)

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-large-en-v1.5")

def embed_document(text: str) -> list[float]:
    """For storing tickets in Qdrant."""
    return model.encode(text, normalize_embeddings=True).tolist()

def embed_query(text: str) -> list[float]:
    """For query-time search (BGE requires a prefix for retrieval tasks)."""
    prefixed = f"Represent this sentence for searching relevant passages: {text}"
    return model.encode(prefixed, normalize_embeddings=True).tolist()
```

**Alternative: OpenAI text-embedding-3-large** (cloud, API-based)

```python
response = client.embeddings.create(
    model="text-embedding-3-large",
    input=text,
    dimensions=1024  # Reduce from 3072 with negligible quality loss
)
```

| Model | Dim | Quality | Cost | Best For |
|-------|-----|---------|------|----------|
| BGE-large-en-v1.5 | 1024 | ⭐⭐⭐⭐⭐ | Free | Local/on-prem deployment |
| OpenAI text-3-large | 1024* | ⭐⭐⭐⭐⭐ | $0.13/1M tokens | Cloud deployment |
| BGE-base-en-v1.5 | 768 | ⭐⭐⭐ | Free | High-throughput, lower RAM |

*Reduced from 3072 via the `dimensions` parameter.

---

### Embedding Pipeline (per ticket)

```
Ticket Input
    │
    ▼
format_ticket_for_embedding() → structured text string
    │
    ▼
Redis cache check: embed:{model_version}:{sha256(text)[:16]}
    │                    │
    │ HIT                MISS
    │                    │
    │             BGE-large inference (normalize_embeddings=True)
    │             → 1024-dim float32 vector
    │             + cache write (TTL: 7 days)
    │                    │
    └────────────────────┤
                         ▼
                  Qdrant upsert (vector + payload)
                  PostgreSQL update (embedding_id, embedding_status="indexed")
```

### Batch Processing (bulk ingestion)

```python
embeddings = model.encode(
    texts,
    batch_size=32,
    normalize_embeddings=True,
    show_progress_bar=False,
    convert_to_numpy=True
)
# 32 tickets in ~100ms vs ~1600ms sequential → 30x throughput improvement
```

### Re-Embedding Strategy (model upgrades)

```
1. Create new Qdrant collection: tickets_v2
2. Batch re-embed all tickets with new model (scripts/reindex_all.py)
3. Run A/B comparison: retrieval quality v1 vs v2
4. Atomic rename: tickets → tickets_v1, tickets_v2 → tickets
5. Keep v1 collection for 30 days as rollback
6. Update embedding_version in PostgreSQL
```

---

## Qdrant Vector Database

### Collections

**Collection: `tickets`** — stores all ticket embeddings

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, HnswConfigDiff

client = QdrantClient(host="qdrant", port=6333)

client.create_collection(
    collection_name="tickets",
    vectors_config=VectorParams(
        size=1024,
        distance=Distance.COSINE
    ),
    hnsw_config=HnswConfigDiff(
        m=16,                      # Graph connections per node
        ef_construct=200,          # High-quality graph construction
        full_scan_threshold=10000  # Use full scan only for tiny collections
    ),
    optimizers_config={
        "default_segment_number": 4,
        "max_segment_size": 200000,
    }
)
```

**Collection: `steering_vectors`** — stores 8 pre-computed directional vectors

```python
client.create_collection(
    collection_name="steering_vectors",
    vectors_config=VectorParams(size=1024, distance=Distance.COSINE)
)
```

### Vector Configuration

| Parameter | Value | Reason |
|-----------|-------|--------|
| Vector Size | 1024 | BGE-large-en-v1.5 output dimension |
| Distance Metric | Cosine | Magnitude-independent; works with L2-normalized vectors |
| HNSW m | 16 | Balance between graph quality and memory |
| HNSW ef_construct | 200 | High-quality graph at index time |
| Quantization | Scalar (int8) | 4x memory reduction, <1% accuracy loss |

### Payload Schema

Each Qdrant point carries a JSON payload for hybrid search (vector similarity + metadata filtering):

```json
{
  "ticket_id": "TKT-2024-00847",
  "functional_group": "Bluetooth",
  "status": "Rejected",
  "severity": "High",
  "created_at": "2024-01-15T10:30:00Z",
  "resolved_at": "2024-01-20T14:22:00Z",
  "tags": ["bluetooth", "hfp", "a2dp"],
  "has_resolution": true,
  "root_cause_category": "firmware_bug",
  "platform": "Android 14",
  "embedding_model": "BAAI/bge-large-en-v1.5",
  "embedding_version": "v1"
}
```

### Payload Indexes

```python
for field, schema in [
    ("status",           "keyword"),
    ("functional_group", "keyword"),
    ("severity",         "keyword"),
    ("created_at",       "datetime"),
    ("tags",             "keyword"),
]:
    client.create_payload_index("tickets", field, schema)
```

---

## HNSW Search Mechanics

```
Query Vector Q
      │
      ▼
Entry Point (top HNSW layer)
      │
      ▼  Greedy traversal through layers:
  Layer 3: Rough neighborhood  (few nodes, fast)
  Layer 2: Closer neighborhood
  Layer 1: Fine neighborhood
  Layer 0: Exact candidates
      │
      ▼
ef_search=128 candidates evaluated
      │
      ▼
Top-K by cosine similarity returned
```

### Search Recall vs Latency

| ef_search | Recall@10 | Latency |
|-----------|-----------|---------|
| 32 | ~91% | ~3ms |
| 64 | ~95% | ~6ms |
| 128 | ~98% | ~10ms ← **recommended** |
| 256 | ~99% | ~18ms |

Qdrant automatically selects **pre-filtering** (filter then search) or **post-filtering** (search then filter) based on filter cardinality.

---

## Example Queries

**Find similar rejected Bluetooth tickets:**

```python
results = client.search(
    collection_name="tickets",
    query_vector=steered_embedding,
    query_filter=Filter(must=[
        FieldCondition(key="status", match=MatchValue(value="Rejected")),
        FieldCondition(key="functional_group", match=MatchValue(value="Bluetooth"))
    ]),
    limit=10,
    score_threshold=0.75,
    with_payload=True,
    with_vectors=False
)
```

**Two-stage retrieval — scope ANN to BM25 candidates only (Stage 2):**

```python
results = client.search(
    collection_name="tickets",
    query_vector=steered_embedding,
    query_filter=Filter(must=[
        FieldCondition(
            key="ticket_id",
            match=MatchAny(any=bm25_candidate_ids)  # Scope to pre-filtered 50 IDs
        )
    ]),
    limit=10,
    score_threshold=0.65,
    with_payload=True
)
```

**High-confidence duplicate check:**

```python
results = client.search(
    collection_name="tickets",
    query_vector=duplicate_steered_embedding,
    query_filter=Filter(
        must_not=[FieldCondition(key="ticket_id", match=MatchValue(value=current_ticket_id))]
    ),
    limit=5,
    score_threshold=0.92   # Very high threshold — only exact duplicates
)
```

---

## Storage Estimation

| Scale | Quantized Vectors | Payload | HNSW Graph | Total |
|-------|------------------|---------|------------|-------|
| 100K tickets | 100 MB | 50 MB | 13 MB | ~163 MB |
| 1M tickets | 1 GB | 500 MB | 128 MB | ~1.6 GB |
| 10M tickets | 10 GB | 5 GB | 1.3 GB | ~16 GB |

For 10M+ tickets, use Qdrant distributed mode:

```yaml
# 4 shards × 2 replicas, ~8GB RAM per shard
qdrant:
  collections:
    tickets:
      shard_number: 4
      replication_factor: 2
```

---

## Data Integrity

```python
# Upsert (insert or update) — idempotent
client.upsert(
    collection_name="tickets",
    points=[PointStruct(
        id=ticket_uuid,            # PostgreSQL UUID (creates direct reference link)
        vector=embedding.tolist(),
        payload={"ticket_id": "TKT-2024-00847", "status": "Fixed", ...}
    )]
)

# Update payload only (e.g., when ticket status changes)
client.set_payload(
    collection_name="tickets",
    payload={"status": "Fixed", "resolved_at": "2024-03-20T10:00:00Z"},
    points=[ticket_uuid]
)

# Delete (GDPR or data cleanup)
client.delete(
    collection_name="tickets",
    points_selector=PointIdsList(points=[ticket_uuid])
)
```

> **Next:** [03-steering-vectors.md](03-steering-vectors.md)
