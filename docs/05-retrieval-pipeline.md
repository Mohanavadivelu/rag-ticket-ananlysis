# Section 5: Retrieval Pipeline Design

## 🔍 Overview

The retrieval pipeline is the **orchestration layer** that takes a new ticket, applies embeddings and steering, queries Qdrant, joins metadata from PostgreSQL, and prepares context for the LLM. It is the bridge between raw ticket input and intelligent reasoning.

---

## 📋 Step-by-Step Pipeline Flow

```
Step 1: New Ticket Arrives
         │
         ▼
Step 2: Embedding Generated
         │
         ▼
Step 3: Steering Applied
         │
         ▼
Step 4: Query Sent to Qdrant
         │
         ▼
Step 5: Similar Tickets Retrieved
         │
         ▼
Step 6: Metadata Retrieved from PostgreSQL
         │
         ▼
Step 7: Results Sent to LLM
```

---

## 🔄 Sequence Diagram

```
Client          API Gateway     Retrieval Svc   Embedding Svc   Steering Svc    Qdrant          PostgreSQL      LLM Svc
  │                  │                │                │               │             │                │              │
  │ POST /analyze    │                │                │               │             │                │              │
  │─────────────────►│                │                │               │             │                │              │
  │                  │ route request  │                │               │             │                │              │
  │                  │───────────────►│                │               │             │                │              │
  │                  │                │ embed(ticket)  │               │             │                │              │
  │                  │                │───────────────►│               │             │                │              │
  │                  │                │                │ check cache   │             │                │              │
  │                  │                │                │ (Redis)       │             │                │              │
  │                  │                │◄───────────────│               │             │                │              │
  │                  │                │  q (1024-dim)  │               │             │                │              │
  │                  │                │                │               │             │                │              │
  │                  │                │ get_steering_vectors()         │             │                │              │
  │                  │                │───────────────────────────────►│             │                │              │
  │                  │                │◄───────────────────────────────│             │                │              │
  │                  │                │  [s_rejected, s_bluetooth]     │             │                │              │
  │                  │                │                │               │             │                │              │
  │                  │                │ apply_steering(q, vectors)     │             │                │              │
  │                  │                │─── internal ──────────────────────────────► │                │              │
  │                  │                │  q_steered (1024-dim)          │             │                │              │
  │                  │                │                │               │             │                │              │
  │                  │                │ search(q_steered, filter, k=10)│             │                │              │
  │                  │                │────────────────────────────────────────────►│                │              │
  │                  │                │◄────────────────────────────────────────────│                │              │
  │                  │                │  [{id, score, payload}, ...]   │             │                │              │
  │                  │                │                │               │             │                │              │
  │                  │                │ fetch_metadata(ticket_ids)     │             │                │              │
  │                  │                │────────────────────────────────────────────────────────────►│              │
  │                  │                │◄────────────────────────────────────────────────────────────│              │
  │                  │                │  full ticket records           │             │                │              │
  │                  │                │                │               │             │                │              │
  │                  │                │ analyze(ticket, context)       │             │                │              │
  │                  │                │────────────────────────────────────────────────────────────────────────────►│
  │                  │                │◄────────────────────────────────────────────────────────────────────────────│
  │                  │                │  analysis result               │             │                │              │
  │                  │                │                │               │             │                │              │
  │◄─────────────────│◄───────────────│                │               │             │                │              │
  │  AnalysisResponse│                │                │               │             │                │              │
```

---

## 💻 Implementation

### RetrievalService

```python
# services/retrieval/retrieval_service.py
from typing import Optional
import numpy as np

class RetrievalService:
    def __init__(
        self,
        qdrant_client: QdrantClient,
        embedding_service: EmbeddingService,
        steering_service: SteeringService,
        db: AsyncSession,
        redis: Redis
    ):
        self.qdrant = qdrant_client
        self.embedder = embedding_service
        self.steering = steering_service
        self.db = db
        self.redis = redis

    async def retrieve_similar_tickets(
        self,
        query_ticket: TicketInput,
        search_config: SearchConfig
    ) -> RetrievalResult:
        """
        Full retrieval pipeline: embed → steer → search → join metadata.
        """

        # Step 1: Generate base embedding
        embedding = await self.embedder.embed_ticket(query_ticket)

        # Step 2: Select steering vectors based on ticket context
        steering_vectors = self.steering.select_vectors(
            functional_group=query_ticket.functional_group,
            analysis_hints=query_ticket.analysis_notes,
            search_mode=search_config.mode  # "rejected", "duplicate", "similar"
        )

        # Step 3: Apply steering
        if steering_vectors:
            steered_embedding = self.steering.apply(
                embedding=embedding,
                vectors=steering_vectors
            )
        else:
            steered_embedding = embedding

        # Step 4: Build Qdrant filter
        qdrant_filter = self._build_filter(search_config)

        # Step 5: Search Qdrant
        qdrant_results = await self.qdrant.search(
            collection_name="tickets",
            query_vector=steered_embedding.tolist(),
            query_filter=qdrant_filter,
            limit=search_config.top_k,
            score_threshold=search_config.score_threshold,
            with_payload=True,
            with_vectors=False
        )

        if not qdrant_results:
            return RetrievalResult(tickets=[], steering_applied=bool(steering_vectors))

        # Step 6: Fetch full metadata from PostgreSQL
        ticket_ids = [r.payload["ticket_id"] for r in qdrant_results]
        pg_tickets = await self._fetch_pg_metadata(ticket_ids)

        # Step 7: Merge Qdrant scores with PostgreSQL metadata
        enriched = self._merge_results(qdrant_results, pg_tickets)

        return RetrievalResult(
            tickets=enriched,
            steering_applied=bool(steering_vectors),
            steering_names=[sv.name for sv in steering_vectors],
            query_embedding_norm=float(np.linalg.norm(embedding))
        )

    def _build_filter(self, config: SearchConfig) -> Optional[Filter]:
        """Build Qdrant filter from search configuration."""
        must_conditions = []
        must_not_conditions = []

        if config.status_filter:
            must_conditions.append(
                FieldCondition(key="status", match=MatchValue(value=config.status_filter))
            )

        if config.functional_group_filter:
            must_conditions.append(
                FieldCondition(key="functional_group",
                               match=MatchValue(value=config.functional_group_filter))
            )

        if config.exclude_ticket_id:
            must_not_conditions.append(
                FieldCondition(key="ticket_id",
                               match=MatchValue(value=config.exclude_ticket_id))
            )

        if not must_conditions and not must_not_conditions:
            return None

        return Filter(must=must_conditions or None, must_not=must_not_conditions or None)

    async def _fetch_pg_metadata(self, ticket_ids: list[str]) -> dict:
        """Fetch full ticket records from PostgreSQL."""
        cache_keys = [f"ticket:meta:{tid}" for tid in ticket_ids]
        cached = await self.redis.mget(*cache_keys)

        result = {}
        missing_ids = []

        for i, (tid, val) in enumerate(zip(ticket_ids, cached)):
            if val:
                result[tid] = json.loads(val)
            else:
                missing_ids.append(tid)

        if missing_ids:
            db_records = await self.db.execute(
                select(Ticket).where(Ticket.ticket_id.in_(missing_ids))
            )
            for record in db_records.scalars():
                data = record.to_dict()
                result[record.ticket_id] = data
                # Cache for 1 hour
                await self.redis.setex(
                    f"ticket:meta:{record.ticket_id}", 3600, json.dumps(data)
                )

        return result

    def _merge_results(self, qdrant_results, pg_tickets) -> list[EnrichedTicket]:
        """Merge Qdrant similarity scores with full PostgreSQL records."""
        enriched = []
        for r in qdrant_results:
            tid = r.payload["ticket_id"]
            pg_data = pg_tickets.get(tid, {})
            enriched.append(EnrichedTicket(
                ticket_id=tid,
                similarity_score=r.score,
                status=r.payload.get("status"),
                functional_group=r.payload.get("functional_group"),
                description=pg_data.get("description", ""),
                analysis_notes=pg_data.get("analysis_notes", ""),
                resolution=pg_data.get("resolution", ""),
                created_at=pg_data.get("created_at"),
                resolved_at=pg_data.get("resolved_at"),
            ))
        return enriched
```

---

## ⚙️ Search Configuration Modes

| Mode | Steering Vectors | Score Threshold | Top-K | Use Case |
|------|-----------------|-----------------|-------|----------|
| `similar` | domain only | 0.70 | 10 | General similarity |
| `rejected` | rejected + domain | 0.75 | 10 | Rejection check |
| `duplicate` | duplicate | 0.92 | 5 | Dedup screening |
| `root_cause` | firmware/hardware | 0.72 | 15 | Root cause grouping |
| `analysis` | all applicable | 0.70 | 10 | Full LLM analysis |

---

## 🧮 Result Ranking Enhancement

After Qdrant retrieval, a **re-ranking step** adds business logic scoring:

```python
def rerank_results(results: list[EnrichedTicket], query_ticket: TicketInput) -> list[EnrichedTicket]:
    """
    Re-rank retrieval results using business rules on top of vector similarity.
    """
    for r in results:
        bonus = 0.0

        # Boost exact functional group match
        if r.functional_group == query_ticket.functional_group:
            bonus += 0.05

        # Boost resolved tickets (they have root cause context)
        if r.status in ("Fixed", "Resolved") and r.resolution:
            bonus += 0.03

        # Boost recently resolved (more relevant than old fixes)
        if r.resolved_at and (datetime.utcnow() - r.resolved_at).days < 180:
            bonus += 0.02

        r.final_score = min(r.similarity_score + bonus, 1.0)

    return sorted(results, key=lambda x: x.final_score, reverse=True)
```

---

> **Next:** [Section 6 — LLM Reasoning Design](06-llm-reasoning.md)
