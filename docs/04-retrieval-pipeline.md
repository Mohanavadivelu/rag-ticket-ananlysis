# Retrieval Pipeline: Two-Stage BM25 + Qdrant

## Overview

The retrieval pipeline uses a **two-stage approach** to combine keyword recall with semantic precision:

1. **Stage 1 — PostgreSQL BM25 (fast, keyword-based):** Full-text search using `tsvector` returns the top-50 candidate ticket IDs from the entire database in ~2ms.
2. **Stage 2 — Qdrant ANN (semantic, steered):** Vector similarity search scoped to only those 50 candidates returns the top-10 most semantically similar results in ~8ms.

### Why Two Stages?

| Concern | Single Qdrant Search | Two-Stage |
|---------|---------------------|-----------|
| Exact keyword matches (e.g., "HFP", "WPA3", "AVRCP") | May miss — ANN is fuzzy | PostgreSQL BM25 catches exact terms |
| Semantic paraphrase matching | Strong | Strong (Qdrant stage) |
| Operational dependencies | None | Requires `tsvector` column on tickets |
| Search latency | ~10ms | ~12ms (+2ms BM25 — negligible) |

---

## Stage 1: PostgreSQL BM25 (tsvector)

The `tickets` table has a generated `tsvector` column that is kept automatically in sync:

```sql
-- Added in migration 002
ALTER TABLE tickets ADD COLUMN search_vector TSVECTOR
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')),          'A') ||
    setweight(to_tsvector('english', coalesce(description, '')),    'B') ||
    setweight(to_tsvector('english', coalesce(analysis_notes, '')), 'C')
  ) STORED;

CREATE INDEX idx_tickets_search_vector ON tickets USING GIN(search_vector);
```

**Why `GENERATED ALWAYS AS`?** The column is automatically updated on every INSERT/UPDATE — no application-layer triggers or background sync needed.

**Weight mapping:**
- `A` (title): highest weight
- `B` (description): standard weight
- `C` (analysis_notes): lower weight

### BM25 Query

```python
async def bm25_search(
    query_text: str,
    filters: Optional[SearchFilters],
    limit: int = 50
) -> list[str]:
    """
    PostgreSQL full-text search returning ticket_id strings.
    plainto_tsquery is safe for user input (no special syntax needed).
    ts_rank_cd uses cover density — approximate BM25 ranking.
    """
    filter_clause = ""
    params: dict = {"query": query_text, "limit": limit}

    if filters:
        if filters.functional_group:
            filter_clause += " AND functional_group = :fg"
            params["fg"] = filters.functional_group
        if filters.status:
            filter_clause += " AND status = ANY(:statuses)"
            params["statuses"] = filters.status
        if filters.date_from:
            filter_clause += " AND created_at >= :date_from"
            params["date_from"] = filters.date_from

    sql = text(f"""
        SELECT ticket_id, ts_rank_cd(search_vector, query) AS rank
        FROM tickets,
             plainto_tsquery('english', :query) query
        WHERE search_vector @@ query
          {filter_clause}
        ORDER BY rank DESC
        LIMIT :limit
    """)

    result = await db.execute(sql, params)
    return [row.ticket_id for row in result.fetchall()]
```

---

## Stage 2: Qdrant ANN (Semantic + Steered)

The Qdrant search is scoped to only the BM25 candidate IDs using `MatchAny`:

```python
from qdrant_client.models import Filter, FieldCondition, MatchAny

def build_qdrant_filter(
    bm25_ids: list[str],
    filters: Optional[SearchFilters]
) -> Filter:
    """
    Scopes ANN search to BM25 candidates only.
    MatchAny on ticket_id is the key two-stage mechanism.
    """
    must = [
        FieldCondition(
            key="ticket_id",
            match=MatchAny(any=bm25_ids)   # Only search within BM25 pre-filtered set
        )
    ]
    if filters and filters.functional_group:
        must.append(FieldCondition(
            key="functional_group",
            match=MatchValue(value=filters.functional_group)
        ))
    return Filter(must=must)
```

---

## Full Pipeline Implementation

```python
class TwoStageRetrievalService:

    async def search(self, request: SearchRequest) -> SearchResponse:
        start = time.monotonic()

        # Stage 1: PostgreSQL BM25 pre-filter (50 candidates)
        bm25_ids = await self.bm25_search(
            query_text=request.query.description,
            filters=request.filters,
            limit=self.settings.BM25_CANDIDATE_COUNT  # default: 50
        )

        if not bm25_ids:
            return SearchResponse(success=True, total_results=0, ...)

        # Embed query
        query_text = self._build_query_text(request.query)
        query_vec = self.embedder.embed_query(query_text)

        # Apply steering if requested
        steering_applied = []
        if request.steering:
            query_vec, steering_applied = await self.steering.apply_from_config(
                query_vec, request.steering
            )

        # Stage 2: Qdrant ANN scoped to BM25 candidates (top-10)
        qdrant_filter = self.build_qdrant_filter(bm25_ids, request.filters)
        qdrant_results = await self.qdrant.search(
            collection_name="tickets",
            query_vector=query_vec.tolist(),
            query_filter=qdrant_filter,
            limit=self.settings.QDRANT_RERANK_TOP_K,   # default: 10
            score_threshold=request.options.score_threshold,
            with_payload=True,
            with_vectors=False
        )

        # Fetch full metadata from PostgreSQL (Redis-cached)
        enriched = await self._enrich_and_rerank(qdrant_results, request.query)

        latency_ms = int((time.monotonic() - start) * 1000)
        return SearchResponse(
            success=True,
            total_results=len(enriched),
            search_latency_ms=latency_ms,
            steering_applied=steering_applied,
            results=[self._to_search_result(i + 1, t) for i, t in enumerate(enriched)],
        )
```

---

## Metadata Join + Re-Ranking

After Qdrant returns similarity scores, full ticket metadata is fetched from PostgreSQL (Redis cache layer reduces DB hits):

```python
async def _fetch_pg_metadata(self, ticket_ids: list[str]) -> dict:
    """Fetch full records — Redis cache → PostgreSQL fallback."""
    cache_keys = [f"ticket:meta:{tid}" for tid in ticket_ids]
    cached = await self.redis.mget(*cache_keys)

    result = {}
    missing_ids = []
    for tid, val in zip(ticket_ids, cached):
        if val:
            result[tid] = json.loads(val)
        else:
            missing_ids.append(tid)

    if missing_ids:
        db_records = await self.db.execute(
            select(TicketORM).where(TicketORM.ticket_id.in_(missing_ids))
        )
        for record in db_records.scalars():
            data = record.__dict__
            result[record.ticket_id] = data
            await self.redis.setex(f"ticket:meta:{record.ticket_id}", 3600, json.dumps(data))

    return result
```

After metadata join, a re-ranking step adds business rule adjustments and an **outcome weight** from the learning loop:

```python
def rerank_results(results: list[EnrichedTicket], query: SearchQuery) -> list[EnrichedTicket]:
    for r in results:
        bonus = 0.0

        # Boost exact functional group match
        if r.functional_group == query.functional_group:
            bonus += 0.05

        # Boost resolved tickets (they have root cause context)
        if r.status in ("Fixed", "Closed") and r.resolution:
            bonus += 0.03

        # Boost recently resolved (more relevant than old fixes)
        if r.resolved_at and (datetime.utcnow() - r.resolved_at).days < 180:
            bonus += 0.02

        # Outcome weight: confirmed-correct feedback increments this (1.0 default, 1.5 cap)
        # See: docs/09-learning-loop.md — Learning Mechanism 2
        outcome_weight = r.payload.get("outcome_weight", 1.0)
        r.final_score = min((r.similarity_score * outcome_weight) + bonus, 1.0)

    return sorted(results, key=lambda x: x.final_score, reverse=True)
```

`outcome_weight` comes from the Qdrant payload and is populated by the feedback service whenever an engineer confirms an analysis as correct. Tickets that have been repeatedly confirmed surface higher in re-ranking. See [09-learning-loop.md](09-learning-loop.md#learning-mechanism-2--outcome-weighted-retrieval) for the full algorithm.

---

## Pipeline Flow Diagram

```
POST /api/v1/tickets/search
        │
        ▼
┌───────────────────────────────────────────────┐
│  Stage 1: PostgreSQL BM25                      │
│                                               │
│  plainto_tsquery('english', query_text)        │
│  WHERE search_vector @@ query                 │
│  ORDER BY ts_rank_cd DESC                     │
│  LIMIT 50                                     │
│                                               │
│  ~2ms latency                                 │
│  Returns: [ticket_id, ticket_id, ...]  (50)   │
└───────────────────────────┬───────────────────┘
                            │ 50 candidate IDs
                            ▼
┌───────────────────────────────────────────────┐
│  Embed Query                                   │
│  BGE-large-en-v1.5 / Redis cache               │
│  → 1024-dim vector                            │
│  ~40ms (cached: 1ms)                          │
└───────────────────────────┬───────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────┐
│  Apply Steering Vectors                        │
│  auto_select() based on functional_group       │
│  q_steered = normalize(q + α×s₁ + α×s₂)       │
│  ~1ms (in-process)                            │
└───────────────────────────┬───────────────────┘
                            │ steered 1024-dim vector
                            ▼
┌───────────────────────────────────────────────┐
│  Stage 2: Qdrant ANN                           │
│                                               │
│  filter: ticket_id IN [50 BM25 candidates]    │
│  limit: 10                                    │
│  score_threshold: 0.65                        │
│                                               │
│  ~8ms latency (ef_search=128)                 │
│  Returns: top-10 by cosine similarity         │
└───────────────────────────┬───────────────────┘
                            │ 10 results with scores
                            ▼
┌───────────────────────────────────────────────┐
│  Metadata Join                                 │
│  Redis cache → PostgreSQL fallback             │
│  ~3ms (cache hit) / ~15ms (cache miss)        │
└───────────────────────────┬───────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────┐
│  Business Rule Re-Ranking                      │
│  outcome_weight × similarity_score             │
│  +0.05 exact group, +0.03 resolved, +0.02 recent │
│  Sort by final_score DESC                     │
└───────────────────────────┬───────────────────┘
                            │
                            ▼
                    SearchResponse
           (total: ~55ms typical latency)
```

---

## Search Configuration Modes

| Mode | BM25 Limit | Qdrant Top-K | Steering | Score Threshold | Use Case |
|------|-----------|-------------|---------|----------------|----------|
| `similar` | 50 | 10 | domain only | 0.70 | General similarity |
| `rejected` | 50 | 10 | rejected + domain | 0.75 | Rejection check |
| `duplicate` | 20 | 5 | duplicate | 0.92 | Dedup screening |
| `root_cause` | 50 | 15 | firmware/hardware | 0.72 | Root cause grouping |
| `analysis` | 50 | 10 | all applicable | 0.65 | Full LLM analysis |

> **Next:** [05-llm-analysis.md](05-llm-analysis.md)
