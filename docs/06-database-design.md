# Database Design (PostgreSQL)

## Overview

PostgreSQL is the **source of truth** for all ticket metadata, analysis results, engineer feedback, and audit logs. Qdrant stores vectors; PostgreSQL stores everything else.

Two Alembic migrations build the full schema:
- **001_initial_schema** — 6 core tables from the original design
- **002_add_feedback_and_tsvector** — adds BM25 search support + feedback loop tables

---

## Migration 001: Core Schema

### Table: `users`

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    full_name   VARCHAR(255),
    team        VARCHAR(100),
    role        VARCHAR(50) DEFAULT 'engineer',  -- admin, engineer, viewer, service
    api_key     VARCHAR(64) UNIQUE,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login  TIMESTAMPTZ
);
```

### Table: `tickets` (core ticket data)

```sql
CREATE TABLE tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           VARCHAR(50) UNIQUE NOT NULL,      -- e.g. "TKT-2024-01247"
    functional_group    VARCHAR(50) NOT NULL,              -- "Bluetooth", "WiFi", etc.
    title               VARCHAR(500),
    description         TEXT NOT NULL,
    analysis_notes      TEXT,
    resolution          TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'New',
    sub_status          VARCHAR(50),
    severity            VARCHAR(20) DEFAULT 'Medium',
    priority            VARCHAR(20) DEFAULT 'Medium',
    platform            VARCHAR(100),
    vehicle_model       VARCHAR(100),
    firmware_version    VARCHAR(50),
    reporter_id         UUID REFERENCES users(id),
    assignee_id         UUID REFERENCES users(id),
    assignee_team       VARCHAR(100),
    tags                TEXT[],
    statistics          JSONB DEFAULT '{}',
    duplicate_of        UUID REFERENCES tickets(id),
    root_cause_category VARCHAR(50),
    embedding_id        UUID,
    embedding_status    VARCHAR(20) DEFAULT 'pending',    -- pending, queued, indexed, failed
    embedding_version   VARCHAR(10) DEFAULT 'v1',
    text_hash           VARCHAR(64),                       -- SHA-256 for dedup
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at         TIMESTAMPTZ,
    source_system       VARCHAR(50) DEFAULT 'internal',
    external_id         VARCHAR(200),
    CONSTRAINT valid_status   CHECK (status IN ('New','Investigating','Fixed','Rejected','Duplicate','Closed','Pending')),
    CONSTRAINT valid_severity CHECK (severity IN ('Critical','High','Medium','Low'))
);

CREATE INDEX idx_tickets_status           ON tickets(status);
CREATE INDEX idx_tickets_functional_group ON tickets(functional_group);
CREATE INDEX idx_tickets_created_at       ON tickets(created_at DESC);
CREATE INDEX idx_tickets_severity         ON tickets(severity);
CREATE INDEX idx_tickets_text_hash        ON tickets(text_hash);
CREATE INDEX idx_tickets_tags             ON tickets USING GIN(tags);
CREATE INDEX idx_tickets_statistics       ON tickets USING GIN(statistics);

-- Partial index for active unresolved tickets (most queried)
CREATE INDEX idx_tickets_active ON tickets(created_at DESC)
    WHERE status IN ('New', 'Investigating');

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tickets_updated_at
    BEFORE UPDATE ON tickets
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### Table: `embedding_references`

```sql
CREATE TABLE embedding_references (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    qdrant_point_id     UUID NOT NULL UNIQUE,
    collection_name     VARCHAR(100) NOT NULL DEFAULT 'tickets',
    embedding_model     VARCHAR(100) NOT NULL,
    embedding_version   VARCHAR(10) NOT NULL,
    vector_dimension    INTEGER NOT NULL,
    embedding_text      TEXT NOT NULL,        -- Exact text that was embedded
    text_hash           VARCHAR(64) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(ticket_id, embedding_version)
);

CREATE INDEX idx_emb_ticket_id  ON embedding_references(ticket_id);
CREATE INDEX idx_emb_qdrant_id  ON embedding_references(qdrant_point_id);
CREATE INDEX idx_emb_is_current ON embedding_references(is_current) WHERE is_current = TRUE;
```

### Table: `analysis_results`

```sql
CREATE TABLE analysis_results (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id               UUID NOT NULL REFERENCES tickets(id),
    analysis_version        VARCHAR(10) NOT NULL DEFAULT 'v1',
    is_duplicate            BOOLEAN,
    duplicate_of            VARCHAR(50),
    duplicate_score         DECIMAL(4,3),
    rejection_probability   INTEGER,          -- 0-100
    rejection_reason        TEXT,
    recommendation          VARCHAR(30),      -- Accept, Reject, Needs Investigation
    root_cause_category     VARCHAR(50),
    root_cause_confidence   INTEGER,
    root_cause_explanation  TEXT,
    recommended_action      TEXT,
    priority                VARCHAR(20),
    assignee_team           VARCHAR(100),
    resolution_reference    VARCHAR(50),
    resolution_summary      TEXT,
    confidence_score        INTEGER,          -- 0-100
    summary                 TEXT,
    llm_model               VARCHAR(100),
    steering_applied        TEXT[],           -- Names of steering vectors used
    context_ticket_ids      TEXT[],           -- Ticket IDs used as LLM context
    raw_response            JSONB,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(ticket_id, analysis_version)
);

CREATE INDEX idx_analysis_ticket_id ON analysis_results(ticket_id);
CREATE INDEX idx_analysis_is_dup    ON analysis_results(is_duplicate) WHERE is_duplicate = TRUE;
CREATE INDEX idx_analysis_rej_prob  ON analysis_results(rejection_probability);
```

### Table: `audit_logs`

```sql
CREATE TABLE audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,      -- ticket, analysis, steering_vector
    entity_id   UUID NOT NULL,
    action      VARCHAR(50) NOT NULL,      -- created, updated, deleted, analyzed, embedded
    actor_id    UUID REFERENCES users(id),
    actor_type  VARCHAR(20) DEFAULT 'user',
    old_value   JSONB,
    new_value   JSONB,
    metadata    JSONB DEFAULT '{}',
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity  ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_actor   ON audit_logs(actor_id);
CREATE INDEX idx_audit_created ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_action  ON audit_logs(action);
```

### Table: `steering_vector_metadata`

```sql
CREATE TABLE steering_vector_metadata (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) UNIQUE NOT NULL,
    description     TEXT,
    qdrant_point_id UUID NOT NULL,
    vector_dimension INTEGER NOT NULL DEFAULT 1024,
    pair_count      INTEGER NOT NULL,
    default_alpha   DECIMAL(3,2) NOT NULL DEFAULT 0.5,
    version         VARCHAR(10) NOT NULL DEFAULT 'v1',
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Migration 002: Feedback Loop + BM25 Search

### tsvector Column (BM25 full-text search)

```sql
-- Auto-synced on every INSERT/UPDATE — no application triggers needed
ALTER TABLE tickets ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')),          'A') ||
        setweight(to_tsvector('english', coalesce(description, '')),    'B') ||
        setweight(to_tsvector('english', coalesce(analysis_notes, '')), 'C')
    ) STORED;

-- GIN index enables ~2ms BM25 queries on millions of rows
CREATE INDEX idx_tickets_search_vector ON tickets USING GIN(search_vector);
```

**Weight mapping:**
| Weight | Field | Rationale |
|--------|-------|-----------|
| `A` (highest) | title | Short, precise; exact keyword match is very relevant |
| `B` (standard) | description | Primary semantic content |
| `C` (lower) | analysis_notes | Technical context, supplementary |

### Table: `analysis_feedback`

Stores engineer corrections on LLM-generated analyses. Used to retrain steering vectors.

```sql
CREATE TABLE analysis_feedback (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_result_id  UUID NOT NULL REFERENCES analysis_results(id) ON DELETE CASCADE,
    ticket_id           UUID NOT NULL REFERENCES tickets(id),
    engineer_id         UUID REFERENCES users(id),
    is_correct          BOOLEAN NOT NULL,
    feedback_type       VARCHAR(30) NOT NULL,  -- 'duplicate', 'root_cause', 'rejection', 'overall'
    corrected_value     JSONB,                 -- What engineer says the correct answer is
    notes               TEXT,
    used_for_training   BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_feedback_analysis_id   ON analysis_feedback(analysis_result_id);
CREATE INDEX idx_feedback_used_training ON analysis_feedback(used_for_training)
    WHERE used_for_training = FALSE;           -- Partial index for fast threshold check
CREATE INDEX idx_feedback_created_at    ON analysis_feedback(created_at DESC);
```

### Table: `steering_refresh_log`

Audit trail for steering vector refresh events.

```sql
CREATE TABLE steering_refresh_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vector_name     VARCHAR(100) NOT NULL,
    pair_count      INTEGER NOT NULL,           -- Number of contrastive pairs used
    feedback_count  INTEGER NOT NULL,           -- Number of feedback records consumed
    triggered_by    VARCHAR(30) DEFAULT 'feedback_threshold',  -- or 'manual'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_log_created ON steering_refresh_log(created_at DESC);
```

---

## Entity Relationship Diagram

```
┌──────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│  users   │────►│         tickets          │────►│ embedding_references  │
│          │     │                          │     │                      │
│ id       │     │ id (UUID)                │     │ ticket_id (FK)       │
│ email    │     │ ticket_id (human)        │     │ qdrant_point_id      │
│ role     │     │ functional_group         │     │ embedding_model      │
│ api_key  │     │ description              │     │ text_hash            │
└──────────┘     │ analysis_notes           │     └──────────────────────┘
                 │ resolution               │
                 │ status / severity        │     ┌──────────────────────┐
                 │ embedding_status         │────►│   analysis_results   │
                 │ text_hash (dedup)        │     │                      │
                 │ search_vector (BM25)     │     │ ticket_id (FK)       │
                 │ tags[] / statistics {}   │     │ is_duplicate         │
                 │ duplicate_of (FK self)◄──┘     │ rejection_probability │
                 └──────────┬──────────────┘      │ root_cause_category  │
                            │                     │ steering_applied[]   │
                            ▼                     └──────────┬───────────┘
                 ┌──────────────────────┐                    │
                 │     audit_logs        │         ┌──────────▼───────────┐
                 │                      │         │  analysis_feedback    │
                 │ entity_type          │         │                      │
                 │ entity_id            │         │ analysis_result_id FK│
                 │ action               │         │ is_correct           │
                 │ old/new JSONB        │         │ feedback_type        │
                 └──────────────────────┘         │ used_for_training    │
                                                  └──────────────────────┘

                 ┌──────────────────────┐
                 │ steering_refresh_log  │
                 │                      │
                 │ vector_name          │
                 │ pair_count           │
                 │ feedback_count       │
                 │ triggered_by         │
                 └──────────────────────┘
```

---

## Migration 003: Learning Loop Tables

Added by the learning loop (see [09-learning-loop.md](09-learning-loop.md)).

### Table: `gold_examples`

Stores engineer-confirmed correct analyses used as dynamic few-shot examples in the LLM prompt.

```sql
CREATE TABLE gold_examples (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           UUID NOT NULL REFERENCES tickets(id),
    analysis_result_id  UUID NOT NULL REFERENCES analysis_results(id),
    functional_group    VARCHAR(50) NOT NULL,
    example_summary     TEXT NOT NULL,        -- ~200-token compact analysis text
    confirmation_count  INTEGER NOT NULL DEFAULT 1,
    confirmed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_confirmed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_by        UUID REFERENCES users(id),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(ticket_id)   -- one gold example per ticket; repeated confirmations update it
);

CREATE INDEX idx_gold_fg        ON gold_examples(functional_group) WHERE is_active = TRUE;
CREATE INDEX idx_gold_confirmed ON gold_examples(confirmation_count DESC, confirmed_at DESC);
```

### Table: `vector_accuracy_log`

Nightly accuracy measurements per steering vector for drift detection.

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
-- Track which vectors each feedback record updated (audit)
ALTER TABLE analysis_feedback
    ADD COLUMN vectors_updated TEXT[] DEFAULT '{}';

-- Track accuracy delta across refreshes
ALTER TABLE steering_refresh_log
    ADD COLUMN pre_refresh_accuracy  DECIMAL(5,4),
    ADD COLUMN post_refresh_accuracy DECIMAL(5,4),
    ADD COLUMN triggered_by          VARCHAR(30) DEFAULT 'feedback_threshold';
    -- triggered_by values: 'feedback_threshold' | 'manual' | 'drift_auto'
```

### Qdrant payload additions (tickets collection)

These are added to the `tickets` Qdrant collection payload schema in `scripts/init_qdrant.py` — not PostgreSQL:

| Field | Type | Default | Purpose |
| --- | --- | --- | --- |
| `outcome_weight` | FLOAT | 1.0 | Retrieval score multiplier; incremented on `is_correct=true` feedback |
| `confirmation_count` | INTEGER | 0 | Number of confirmed-correct feedback records for this ticket |
| `last_confirmed_at` | DATETIME | null | Timestamp of most recent confirmation |

---

## Redis Cache Keys

| Key Pattern | Content | TTL |
| --- | --- | --- |
| `embed:{model_ver}:{text_hash[:16]}` | 1024-dim float32 vector (bytes) | 7 days |
| `analysis:{ticket_id}:v1` | Full `AnalysisResult` JSON | 24 hours |
| `ticket:meta:{ticket_id}` | Full ticket dict JSON | 1 hour |
| `steering:v:{vector_name}` | 1024-dim float32 vector (bytes) | 1 hour |
| `search:{query_hash[:16]}` | `SearchResponse` JSON | 5 minutes |

> **Next:** [07-api-design.md](07-api-design.md)
