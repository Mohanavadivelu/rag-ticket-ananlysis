# Section 2: Vector Database Design (Qdrant)

## 🗄️ Overview

Qdrant is the **core vector search engine** of this system. It stores ticket embeddings as high-dimensional vectors and enables sub-millisecond approximate nearest-neighbor (ANN) search across millions of tickets.

---

## 📦 How Qdrant Stores Data

### 1. Embeddings (Vectors)
Each ticket is represented as a **dense float32 vector** of size 1024 (BGE-large-en-v1.5) or 3072 (OpenAI text-embedding-3-large). These vectors capture the **semantic meaning** of the combined ticket text.

```
Ticket Text → Embedding Model → [0.023, -0.412, 0.891, ..., 0.334]  (1024 dims)
                                 └─────────────────────────────────┘
                                           Stored in Qdrant
```

### 2. Metadata Payload
Each vector point in Qdrant carries a **JSON payload** — structured metadata that enables **hybrid search** (vector similarity + metadata filtering).

### 3. Ticket References
Each Qdrant point uses the **PostgreSQL ticket UUID** as its ID, creating a direct reference link between the vector store and the relational database.

---

## 🏛️ Collection Structure

### Collection: `tickets`

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, HnswConfigDiff

client = QdrantClient(host="qdrant", port=6333)

client.create_collection(
    collection_name="tickets",
    vectors_config=VectorParams(
        size=1024,              # BGE-large-en-v1.5 embedding dimension
        distance=Distance.COSINE  # Cosine similarity for semantic search
    ),
    hnsw_config=HnswConfigDiff(
        m=16,                   # HNSW graph connections per node
        ef_construct=200,       # Build-time search depth (higher = better quality)
        full_scan_threshold=10000  # Switch to full scan below this count
    ),
    optimizers_config={
        "default_segment_number": 4,  # Parallel search segments
        "max_segment_size": 200000,   # Points per segment
    }
)
```

### Collection: `steering_vectors`

```python
client.create_collection(
    collection_name="steering_vectors",
    vectors_config=VectorParams(
        size=1024,
        distance=Distance.COSINE
    )
)
```

---

## 📐 Vector Configuration

| Parameter | Value | Reason |
|-----------|-------|--------|
| **Vector Size** | 1024 | BGE-large-en-v1.5 output dimension |
| **Distance Metric** | Cosine | Normalized similarity, model-independent magnitude |
| **HNSW m** | 16 | Balance between graph quality and memory |
| **HNSW ef_construct** | 200 | High-quality graph construction |
| **Quantization** | Scalar (int8) | 4x memory reduction, <5% accuracy loss |

---

## 📋 Payload Schema

Each Qdrant point payload follows this schema:

```json
{
  "ticket_id": "TKT-2024-00847",
  "functional_group": "Bluetooth",
  "status": "Rejected",
  "sub_status": "Known Issue",
  "severity": "High",
  "created_at": "2024-01-15T10:30:00Z",
  "resolved_at": "2024-01-20T14:22:00Z",
  "tags": ["bluetooth", "pairing", "android", "connection-drop"],
  "embedding_model": "BAAI/bge-large-en-v1.5",
  "embedding_version": "v2",
  "has_resolution": true,
  "root_cause_category": "firmware_bug",
  "platform": "Android",
  "vehicle_model": "Model-X-2024",
  "reporter": "team_bt_infotainment",
  "text_hash": "sha256:abc123..."
}
```

---

## 🔍 Indexing Strategy

### Payload Indexes
Create keyword indexes on frequently filtered fields:

```python
# Create indexes for fast metadata filtering
client.create_payload_index(
    collection_name="tickets",
    field_name="status",
    field_schema="keyword"
)

client.create_payload_index(
    collection_name="tickets",
    field_name="functional_group",
    field_schema="keyword"
)

client.create_payload_index(
    collection_name="tickets",
    field_name="severity",
    field_schema="keyword"
)

client.create_payload_index(
    collection_name="tickets",
    field_name="created_at",
    field_schema="datetime"
)

client.create_payload_index(
    collection_name="tickets",
    field_name="tags",
    field_schema="keyword"
)
```

### HNSW Index (Automatic)
Qdrant automatically builds the HNSW graph as points are inserted. The graph allows **O(log n)** approximate nearest-neighbor search.

---

## 🔎 How Similarity Search Works Internally

### Step 1: HNSW Graph Traversal
```
Query Vector Q
      │
      ▼
Entry Point (top layer of HNSW graph)
      │
      ▼
Greedy graph traversal through layers:
  Layer 3: Rough neighborhood  (few nodes, fast)
  Layer 2: Closer neighborhood (more nodes)
  Layer 1: Fine neighborhood   (many nodes)
  Layer 0: Exact candidates    (all connections)
      │
      ▼
ef_search candidates evaluated
      │
      ▼
Top-K by cosine similarity returned
```

### Step 2: Payload Filtering
Qdrant supports two filter modes:
- **Pre-filtering**: Filter payload first, then search (good for high selectivity)
- **Post-filtering**: Search first, then filter (good for low selectivity)

Qdrant automatically selects the optimal strategy based on filter cardinality.

---

## 🔎 Example Queries

### Query 1: Find Similar Rejected Tickets
```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, SearchRequest

results = client.search(
    collection_name="tickets",
    query_vector=query_embedding,      # steered embedding of new ticket
    query_filter=Filter(
        must=[
            FieldCondition(
                key="status",
                match=MatchValue(value="Rejected")
            ),
            FieldCondition(
                key="functional_group",
                match=MatchValue(value="Bluetooth")
            )
        ]
    ),
    limit=10,
    score_threshold=0.75,             # Only return high-confidence matches
    with_payload=True,                # Include metadata in results
    with_vectors=False                # Don't return raw vectors (saves bandwidth)
)
```

### Query 2: Find Duplicates (Any Status)
```python
results = client.search(
    collection_name="tickets",
    query_vector=query_embedding,
    query_filter=Filter(
        must_not=[
            FieldCondition(
                key="ticket_id",
                match=MatchValue(value=current_ticket_id)  # Exclude self
            )
        ]
    ),
    limit=5,
    score_threshold=0.92              # Very high threshold for duplicates
)
```

### Query 3: Find Similar Tickets by Date Range
```python
from qdrant_client.models import DatetimeRange

results = client.search(
    collection_name="tickets",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(
                key="created_at",
                range=DatetimeRange(
                    gte="2024-01-01T00:00:00Z",
                    lte="2024-12-31T23:59:59Z"
                )
            )
        ]
    ),
    limit=10
)
```

### Query 4: Batch Search (Multiple Tickets at Once)
```python
# Search for similar tickets for multiple query embeddings simultaneously
batch_results = client.search_batch(
    collection_name="tickets",
    requests=[
        SearchRequest(
            vector=embedding_1,
            filter=Filter(must=[FieldCondition(key="status", match=MatchValue(value="Fixed"))]),
            limit=5
        ),
        SearchRequest(
            vector=embedding_2,
            filter=Filter(must=[FieldCondition(key="functional_group", match=MatchValue(value="WiFi"))]),
            limit=5
        )
    ]
)
```

---

## 📊 Storage Estimation

| Metric | Formula | Example (1M tickets) |
|--------|---------|---------------------|
| Raw vector storage | `n × dim × 4 bytes` | 1M × 1024 × 4 = **4 GB** |
| Scalar quantized (int8) | `n × dim × 1 byte` | 1M × 1024 × 1 = **1 GB** |
| Payload storage | `n × avg_payload_size` | 1M × 500B = **500 MB** |
| HNSW graph | `n × m × 8 bytes` | 1M × 16 × 8 = **128 MB** |
| **Total (quantized)** | | **~1.6 GB** |

---

## 🛡️ Data Integrity

```python
# Upsert (insert or update) a ticket vector
client.upsert(
    collection_name="tickets",
    points=[
        PointStruct(
            id=ticket_uuid,              # PostgreSQL UUID as Qdrant point ID
            vector=embedding.tolist(),   # float32 list
            payload={
                "ticket_id": "TKT-2024-00847",
                "status": "Rejected",
                "functional_group": "Bluetooth",
                # ... full payload
            }
        )
    ]
)

# Delete a ticket (e.g., GDPR request)
client.delete(
    collection_name="tickets",
    points_selector=PointIdsList(points=[ticket_uuid])
)
```

---

> **Next:** [Section 3 — Embedding Generation Design](03-embedding-design.md)
