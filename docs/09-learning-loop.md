# Learning Loop

## Overview

The agent improves over time through **3-layer memory** and **3 learning mechanisms**. Every engineer correction tightens the embedding space, boosts the most reliable historical tickets in retrieval, and enriches the LLM prompt with validated examples — without any model fine-tuning.

---

## Memory Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        FEEDBACK EVENT                               │
│  POST /api/v1/analysis/{id}/feedback                                │
│  { is_correct: false, feedback_type: "root_cause",                  │
│    corrected_value: { "category": "hardware_limitation" } }         │
└──────────────────────┬──────────────────────────────────────────────┘
                       │ immediately
         ┌─────────────┼──────────────────────┐
         ▼             ▼                      ▼
┌──────────────┐  ┌──────────────┐   ┌────────────────────┐
│   LAYER 1    │  │   LAYER 2    │   │      LAYER 3       │
│ SHORT-TERM   │  │ MEDIUM-TERM  │   │    LONG-TERM       │
│   MEMORY     │  │   MEMORY     │   │     MEMORY         │
│              │  │              │   │                    │
│ Redis Cache  │  │ Steering     │   │ gold_examples      │
│              │  │ Vectors      │   │ (PostgreSQL)       │
│ - Invalidate │  │ (Qdrant +    │   │                    │
│   stale      │  │  Redis +     │   │ - Confirmed-       │
│   analysis   │  │  PostgreSQL) │   │   correct analyses │
│   cache on   │  │              │   │ - Retrieved at     │
│   wrong      │  │ - Rebuilt    │   │   query time       │
│   feedback   │  │   from       │   │ - Injected as      │
│              │  │   corrections│   │   few-shot into    │
│ TTLs:        │  │   (batch 50) │   │   GPT-4o prompt    │
│ analysis 24h │  │              │   │                    │
│ steering 1h  │  │ Refresh:     │   │ Growth: only from  │
│ embed 7d     │  │ threshold or │   │ is_correct=true    │
│              │  │ drift alert  │   │ feedback           │
└──────────────┘  └──────────────┘   └────────────────────┘
```

| Layer | Storage | Scope | TTL / Lifecycle |
| --- | --- | --- | --- |
| Short-term | Redis | Per-ticket, per-query | 5 min – 7 days |
| Medium-term | Qdrant + Redis | Per steering vector | Refreshed on threshold or drift |
| Long-term | PostgreSQL `gold_examples` | Per functional group | Permanent (soft-deleted only) |

---

## Learning Mechanism 1 — Corrective Learning

Feedback corrections become contrastive training pairs that directly update steering vectors.

### Data Flow

```text
Engineer correction (is_correct=false)
  ↓
Save to analysis_feedback (feedback_type, corrected_value JSONB)
  ↓
Immediate: DEL Redis analysis:{ticket_id}:v1
  ↓
Threshold check: COUNT WHERE used_for_training=FALSE
  ├─ count < 50 → done
  └─ count ≥ 50 → BackgroundTask: refresh_from_feedback()
      ↓
      For each vector V in STEERING_VECTOR_CONFIG:
        1. Query matching feedback via Attribution Table
        2. Classify into positive pole / negative pole
        3. Re-embed ticket texts with BGE-large
        4. sv = normalize(mean(pos_embs) - mean(neg_embs))
        5. Upsert to Qdrant steering_vectors collection
        6. DEL Redis steering:v:{V}
        7. Mark feedback rows used_for_training=TRUE
        8. INSERT steering_refresh_log (pair_count, feedback_count)
```

### Attribution Table

This is the exact mapping used by `refresh_from_feedback()` to route each feedback record to the correct vector(s). One feedback row can update at most **two** vectors: one intent vector (from `feedback_type`) + one domain vector (from `functional_group`).

| `feedback_type` | `corrected_value` field | Positive pole | Vector updated |
| --- | --- | --- | --- |
| `root_cause` | `category=firmware_bug` | Confirmed firmware tickets | `firmware_root_cause` |
| `root_cause` | `category=hardware_limitation` | Confirmed hardware tickets | `hardware_root_cause` |
| `root_cause` | `category=configuration_error` | Confirmed config tickets | `configuration_root_cause` |
| `rejection` | `recommendation=Reject` + `is_correct=true` | Confirmed rejected tickets | `rejected` |
| `rejection` | `recommendation=Accept` + `is_correct=false` | Pulls vector away from false positives | `rejected` |
| `duplicate` | `is_duplicate=true` + `is_correct=true` | Confirmed duplicate pairs | `duplicate` |
| `duplicate` | `is_duplicate=false` + `is_correct=false` | Pulls apart false-positive pairs | `duplicate` |
| `overall` | `is_correct=true` | Gold example store only — no vector update | — |
| `overall` | `is_correct=false` | Drift tracking only — no vector update | — |
| any `root_cause` | `functional_group=Bluetooth` | Also updates domain vector | `bluetooth` |
| any `root_cause` | `functional_group=WiFi` | Also updates domain vector | `wifi` |
| any `root_cause` | `functional_group=CarPlay` | Also updates domain vector | `carplay` |
| any `root_cause` | `functional_group=Android Auto` | Also updates domain vector | `android_auto` |

**Fallback**: if fewer than `min_pairs` feedback rows exist for a vector, pad with confirmed historical tickets (status `Fixed`/`Rejected`) to maintain pair quality. This preserves the original bootstrap behaviour as a floor.

**Exclusion rule**: `used_for_training=TRUE` rows are never reprocessed. Threshold counters are per vector family — root_cause corrections trigger `firmware_root_cause`/`hardware_root_cause` refresh independently of domain vector refreshes.

---

## Learning Mechanism 2 — Outcome-Weighted Retrieval

Confirmed-correct analyses get a persistent retrieval boost so the most reliable historical tickets surface first.

### Storage

When `is_correct=true` feedback arrives, the Qdrant payload for the ticket's point is updated:

```text
new_weight = min(1.0 + (confirmation_count + 1) × 0.1, 1.5)   # floor 1.0, cap 1.5
```

Fields written to Qdrant payload: `outcome_weight`, `confirmation_count`, `last_confirmed_at`.

A nightly job decays `outcome_weight` by 1 % per 30 days of no new confirmations, preventing stale tickets from permanently dominating retrieval.

### Application in `rerank_results()`

```python
# retrieval/service.py — rerank_results()
outcome_weight = r.payload.get("outcome_weight", 1.0)   # from Qdrant payload
bonus = 0.0
if r.functional_group == query.functional_group:    bonus += 0.05
if r.status in ("Fixed", "Closed") and r.resolution: bonus += 0.03
if recent(r.resolved_at):                            bonus += 0.02
r.final_score = min((r.similarity_score * outcome_weight) + bonus, 1.0)
```

The `outcome_weight` flows from Qdrant payload through the metadata join step into `EnrichedTicket` — the existing `with_payload=True` Qdrant search already returns it.

---

## Learning Mechanism 3 — Dynamic Few-Shot Prompt Injection

Confirmed-correct analyses are stored as **gold examples** and injected into the GPT-4o prompt at query time, giving the LLM domain-calibrated examples without any fine-tuning.

### Gold Example Creation

When `is_correct=true` feedback arrives:

1. Fetch the full `analysis_results` row and the linked `tickets` row
2. Render a compact `example_summary` (~200 tokens):

```text
Ticket: TKT-2024-00847 | Group: Bluetooth | Status: Fixed
Root Cause: firmware_bug (confidence: 87%) — HFP A2DP sequencing
Rejection: 15% — accepted
Action: firmware-team, port v3.2.2 HFP fix
Resolution: Updated HFP stack. v3.2.2.
[Engineer confirmed: correct]
```

3. `UPSERT` to `gold_examples` (`UNIQUE(ticket_id)`) — repeated confirmations increment `confirmation_count`

### Gold Example Retrieval

At analysis time in `analysis/service.py`, before calling `llm_svc.analyze_ticket()`:

```sql
SELECT example_summary
FROM gold_examples
WHERE functional_group = :fg
  AND is_active = TRUE
ORDER BY confirmation_count DESC, confirmed_at DESC
LIMIT 3
```

### Prompt Injection

A conditional section is inserted **between** the role block and the `RETRIEVED SIMILAR HISTORICAL TICKETS` section in `ANALYSIS_PROMPT_TEMPLATE`. It is omitted entirely when no gold examples exist for the functional group.

```text
## CONFIRMED ANALYSIS EXAMPLES (engineer-validated for {functional_group})

The following {gold_count} examples were confirmed correct by engineers.
Use them as calibration references — your analysis should be consistent
with these validated patterns.

{gold_examples_block}

---

## RETRIEVED SIMILAR HISTORICAL TICKETS
...
```

Token cost: 3 examples × ~200 tokens = ~600 tokens. Negligible against GPT-4o's 128 K context.

---

## Cache Invalidation Rules

| Event | Keys deleted | Timing |
| --- | --- | --- |
| `is_correct=false` feedback for ticket T | `analysis:{T}:v1` | Synchronously, before returning 201 |
| `refresh_from_feedback()` completes for vector V | `steering:v:{V}` | After Qdrant upsert |
| `PATCH /api/v1/tickets/{id}` for ticket T | `analysis:{T}:v1`, `ticket:meta:{T}` | On PATCH completion |
| Gold example created for ticket T | `analysis:{T}:v1` | After INSERT to gold_examples |
| `is_correct=true` feedback for ticket T | No invalidation | Confirmed analysis stays cached |
| Search result cache | Not invalidated on feedback | 5-min TTL is short enough |

All invalidation calls are fire-and-forget (`await redis.delete(key)`). Redis unavailability degrades gracefully — stale data is served until TTL expiry.

---

## Drift Detection

A nightly scheduled job measures per-vector accuracy and flags degradation.

### Algorithm

```text
drift_detection_job() — runs nightly (APScheduler or cron):

For each vector_name V in STEERING_VECTOR_CONFIG:

  1. Query 7-day rolling window
     SELECT
       COUNT(*) FILTER (WHERE is_correct = TRUE)  AS correct_count,
       COUNT(*)                                    AS total_feedback
     FROM analysis_feedback af
     JOIN analysis_results ar ON ar.id = af.analysis_result_id
     WHERE ar.steering_applied @> ARRAY[V]
       AND af.created_at >= NOW() - INTERVAL '7 days'

  2. accuracy_rate = correct_count / total_feedback
     (skip if total_feedback < 10 — insufficient sample)

  3. baseline_rate = most recent accuracy_rate from vector_accuracy_log

  4. delta = accuracy_rate - baseline_rate
     (negative delta = accuracy dropped)

  5. drift_flagged = (baseline_rate - accuracy_rate > 0.20)
                  AND total_feedback >= 10

  6. INSERT to vector_accuracy_log

  7. If drift_flagged:
       - LOG WARN { event: "steering_vector_drift", vector_name, delta, ... }
       - steering_drift_alerts_total{vector_name}.inc()

  8. If delta < -0.35 (severe regression):
       - LOG ERROR { event: "steering_vector_drift", auto_refresh: true }
       - BackgroundTask: refresh_from_feedback(triggered_by='drift_auto')
```

### Drift Thresholds

| Delta | Action |
| --- | --- |
| > -0.20 | Normal — log DEBUG |
| ≤ -0.20, n ≥ 10 | WARN log + Prometheus counter |
| ≤ -0.35, n ≥ 10 | ERROR log + automatic `refresh_from_feedback()` |

### New Prometheus Metrics (added to `observability/metrics.py`)

```python
STEERING_DRIFT_ALERTS = Counter(
    "steering_drift_alerts_total",
    "Steering vector drift detection alerts",
    ["vector_name"]
)
VECTOR_ACCURACY_GAUGE = Gauge(
    "steering_vector_accuracy",
    "7-day rolling feedback accuracy per steering vector",
    ["vector_name"]
)
```

**Grafana alert**: `increase(steering_drift_alerts_total[24h]) > 0`

---

## Database Schema (Migration 003)

### New table: `gold_examples`

```sql
CREATE TABLE gold_examples (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           UUID NOT NULL REFERENCES tickets(id),
    analysis_result_id  UUID NOT NULL REFERENCES analysis_results(id),
    functional_group    VARCHAR(50) NOT NULL,
    example_summary     TEXT NOT NULL,
    confirmation_count  INTEGER NOT NULL DEFAULT 1,
    confirmed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_confirmed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_by        UUID REFERENCES users(id),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(ticket_id)
);

CREATE INDEX idx_gold_fg        ON gold_examples(functional_group) WHERE is_active = TRUE;
CREATE INDEX idx_gold_confirmed ON gold_examples(confirmation_count DESC, confirmed_at DESC);
```

### New table: `vector_accuracy_log`

```sql
CREATE TABLE vector_accuracy_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vector_name     VARCHAR(100) NOT NULL,
    measurement_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    period_days     INTEGER NOT NULL DEFAULT 7,
    total_feedback  INTEGER NOT NULL,
    correct_count   INTEGER NOT NULL,
    incorrect_count INTEGER NOT NULL,
    accuracy_rate   DECIMAL(5,4) NOT NULL,
    baseline_rate   DECIMAL(5,4),
    delta           DECIMAL(5,4),
    drift_flagged   BOOLEAN NOT NULL DEFAULT FALSE,
    notes           TEXT
);

CREATE INDEX idx_vaclog_vector ON vector_accuracy_log(vector_name, measurement_at DESC);
CREATE INDEX idx_vaclog_drift  ON vector_accuracy_log(drift_flagged) WHERE drift_flagged = TRUE;
```

### New columns on existing tables

```sql
-- Track which vectors each feedback record updated (audit trail)
ALTER TABLE analysis_feedback
    ADD COLUMN vectors_updated TEXT[] DEFAULT '{}';

-- Track accuracy delta across refreshes
ALTER TABLE steering_refresh_log
    ADD COLUMN pre_refresh_accuracy  DECIMAL(5,4),
    ADD COLUMN post_refresh_accuracy DECIMAL(5,4),
    ADD COLUMN triggered_by          VARCHAR(30) DEFAULT 'feedback_threshold';
```

### Qdrant payload additions (tickets collection)

Defined in `scripts/init_qdrant.py` — not PostgreSQL:

| Field | Type | Default | Purpose |
| --- | --- | --- | --- |
| `outcome_weight` | FLOAT | 1.0 | Retrieval score multiplier |
| `confirmation_count` | INTEGER | 0 | Number of `is_correct=true` feedback records |
| `last_confirmed_at` | DATETIME | null | Timestamp of most recent confirmation |

---

## End-to-End Learning Cycle

```text
T+0   Engineer marks analysis wrong (root_cause=hardware, Bluetooth ticket)
      → analysis_feedback INSERT
      → Redis DEL analysis:{ticket_id}:v1  (stale wrong answer gone immediately)

T+0   Engineer marks different analysis correct
      → gold_examples UPSERT (example_summary stored)
      → Qdrant payload: outcome_weight += 0.1 for that ticket
      → Redis DEL analysis:{ticket_id}:v1  (force re-run with new gold examples)

T+?   50th unprocessed feedback record arrives
      → BackgroundTask: refresh_from_feedback()
        → Attribution Table routes rows to hardware_root_cause + bluetooth vectors
        → BGE re-embeds ticket texts → new contrastive mean-diff vectors computed
        → Qdrant steering_vectors upsert → Redis DEL steering:v:hardware_root_cause
                                         → Redis DEL steering:v:bluetooth

T+24h Nightly drift_detection_job runs
      → Computes 7-day accuracy for each vector
      → If accuracy dropped > 20%: WARN log + Prometheus counter
      → If accuracy dropped > 35%: auto-triggers refresh_from_feedback()

Next query for a Bluetooth ticket:
      → Steered query vector uses updated hardware_root_cause + bluetooth vectors
      → rerank_results() applies outcome_weight boost to confirmed tickets
      → analysis/service.py injects top-3 gold examples into GPT-4o prompt
      → GPT-4o sees validated examples → produces more consistent analysis
```

> **Back to:** [01-architecture.md](01-architecture.md)
