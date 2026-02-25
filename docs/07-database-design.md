# Section 7: Database Design (PostgreSQL)

## 🗃️ Overview

PostgreSQL serves as the **source of truth** for all ticket metadata, status tracking, audit logs, and embedding references. Qdrant stores vectors; PostgreSQL stores everything else.

---

## 📐 Schema Design

### Table 1: `tickets` — Core Ticket Data

```sql
CREATE TABLE tickets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id       VARCHAR(50) UNIQUE NOT NULL,        -- e.g. "TKT-2024-01247"
    functional_group VARCHAR(50) NOT NULL,               -- "Bluetooth", "WiFi", etc.
    title           VARCHAR(500),
    description     TEXT NOT NULL,
    analysis_notes  TEXT,
    resolution      TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'New',  -- New, Fixed, Rejected, Duplicate, Investigating
    sub_status      VARCHAR(50),                         -- Known Issue, Out of Scope, Won't Fix
    severity        VARCHAR(20) DEFAULT 'Medium',        -- Critical, High, Medium, Low
    priority        VARCHAR(20) DEFAULT 'Medium',
    platform        VARCHAR(100),                        -- "Android 14", "iOS 17", etc.
    vehicle_model   VARCHAR(100),
    firmware_version VARCHAR(50),
    reporter_id     UUID REFERENCES users(id),
    assignee_id     UUID REFERENCES users(id),
    assignee_team   VARCHAR(100),
    tags            TEXT[],                              -- Array of keywords
    statistics      JSONB DEFAULT '{}',                  -- Flexible stats blob
    duplicate_of    UUID REFERENCES tickets(id),         -- FK to parent if duplicate
    root_cause_category VARCHAR(50),                     -- firmware_bug, hardware_limitation, etc.
    embedding_id    UUID,                                -- Reference to Qdrant point
    embedding_version VARCHAR(10) DEFAULT 'v1',
    text_hash       VARCHAR(64),                         -- SHA-256 of embedding text (dedup)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    source_system   VARCHAR(50) DEFAULT 'internal',      -- jira, custom, api
    external_id     VARCHAR(200),                        -- ID in external system
    CONSTRAINT valid_status CHECK (
        status IN ('New', 'Investigating', 'Fixed', 'Rejected', 'Duplicate', 'Closed', 'Pending')
    ),
    CONSTRAINT valid_severity CHECK (
        severity IN ('Critical', 'High', 'Medium', 'Low')
    )
);

-- Indexes for common queries
CREATE INDEX idx_tickets_status          ON tickets(status);
CREATE INDEX idx_tickets_functional_group ON tickets(functional_group);
CREATE INDEX idx_tickets_created_at      ON tickets(created_at DESC);
CREATE INDEX idx_tickets_severity        ON tickets(severity);
CREATE INDEX idx_tickets_assignee_team   ON tickets(assignee_team);
CREATE INDEX idx_tickets_text_hash       ON tickets(text_hash);  -- Fast dedup check
CREATE INDEX idx_tickets_tags            ON tickets USING GIN(tags);
CREATE INDEX idx_tickets_statistics      ON tickets USING GIN(statistics);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tickets_updated_at
BEFORE UPDATE ON tickets
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

### Table 2: `embedding_references` — Vector Store Tracking

```sql
CREATE TABLE embedding_references (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id       UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    qdrant_point_id UUID NOT NULL UNIQUE,
    collection_name VARCHAR(100) NOT NULL DEFAULT 'tickets',
    embedding_model VARCHAR(100) NOT NULL,
    embedding_version VARCHAR(10) NOT NULL,
    vector_dimension INTEGER NOT NULL,
    embedding_text  TEXT NOT NULL,        -- The exact text that was embedded
    text_hash       VARCHAR(64) NOT NULL, -- SHA-256 for dedup
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,  -- Only one current per ticket
    UNIQUE(ticket_id, embedding_version)
);

CREATE INDEX idx_emb_ticket_id   ON embedding_references(ticket_id);
CREATE INDEX idx_emb_qdrant_id   ON embedding_references(qdrant_point_id);
CREATE INDEX idx_emb_text_hash   ON embedding_references(text_hash);
CREATE INDEX idx_emb_is_current  ON embedding_references(is_current) WHERE is_current = TRUE;
```

---

### Table 3: `analysis_results` — LLM Analysis Storage

```sql
CREATE TABLE analysis_results (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           UUID NOT NULL REFERENCES tickets(id),
    analysis_version    VARCHAR(10) NOT NULL DEFAULT 'v1',
    is_duplicate        BOOLEAN,
    duplicate_of        VARCHAR(50),
    duplicate_score     DECIMAL(4,3),
    rejection_probability INTEGER,        -- 0-100
    rejection_reason    TEXT,
    recommendation      VARCHAR(30),      -- Accept, Reject, Needs Investigation
    root_cause_category VARCHAR(50),
    root_cause_confidence INTEGER,
    root_cause_explanation TEXT,
    recommended_action  TEXT,
    priority            VARCHAR(20),
    assignee_team       VARCHAR(100),
    resolution_reference VARCHAR(50),
    resolution_summary  TEXT,
    confidence_score    INTEGER,          -- 0-100
    summary             TEXT,
    llm_model           VARCHAR(100),
    context_ticket_ids  TEXT[],           -- IDs of tickets used as context
    raw_response        JSONB,            -- Full LLM JSON response
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(ticket_id, analysis_version)
);

CREATE INDEX idx_analysis_ticket_id ON analysis_results(ticket_id);
CREATE INDEX idx_analysis_is_dup    ON analysis_results(is_duplicate) WHERE is_duplicate = TRUE;
CREATE INDEX idx_analysis_rej_prob  ON analysis_results(rejection_probability);
```

---

### Table 4: `audit_logs` — Full Activity Trail

```sql
CREATE TABLE audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,     -- ticket, analysis, steering_vector
    entity_id   UUID NOT NULL,
    action      VARCHAR(50) NOT NULL,     -- created, updated, deleted, analyzed, embedded
    actor_id    UUID REFERENCES users(id),
    actor_type  VARCHAR(20) DEFAULT 'user',  -- user, system, api
    old_value   JSONB,
    new_value   JSONB,
    metadata    JSONB DEFAULT '{}',
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity    ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_actor     ON audit_logs(actor_id);
CREATE INDEX idx_audit_created   ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_action    ON audit_logs(action);

-- Partition by month for scalability
-- ALTER TABLE audit_logs PARTITION BY RANGE (created_at);
```

---

### Table 5: `users` — Authentication

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    full_name   VARCHAR(255),
    team        VARCHAR(100),
    role        VARCHAR(50) DEFAULT 'engineer',  -- admin, engineer, viewer
    api_key     VARCHAR(64) UNIQUE,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login  TIMESTAMPTZ
);
```

---

### Table 6: `steering_vector_metadata` — Steering Vector Registry

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

## 🔗 Entity Relationship Diagram

```
┌──────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  users   │────►│      tickets        │────►│ embedding_refs   │
│          │     │                     │     │                  │
│ id       │     │ id (UUID)           │     │ id               │
│ email    │     │ ticket_id (human)   │     │ ticket_id (FK)   │
│ role     │     │ functional_group    │     │ qdrant_point_id  │
│ team     │     │ description         │     │ embedding_model  │
└──────────┘     │ analysis_notes      │     │ text_hash        │
                 │ resolution          │     └──────────────────┘
                 │ status              │
                 │ severity            │     ┌──────────────────┐
                 │ duplicate_of (FK)◄──┼─────│  analysis_results│
                 │ tags[]              │     │                  │
                 │ statistics JSONB    │     │ ticket_id (FK)   │
                 │ created_at          │     │ is_duplicate     │
                 └──────────┬──────────┘     │ rejection_prob   │
                            │                │ root_cause       │
                            ▼                │ summary          │
                 ┌──────────────────┐        └──────────────────┘
                 │   audit_logs     │
                 │                  │
                 │ entity_type      │
                 │ entity_id        │
                 │ action           │
                 │ old_value JSONB  │
                 └──────────────────┘
```

---

## ⚡ Performance Optimizations

```sql
-- Partial index for active unresolved tickets (most queried)
CREATE INDEX idx_tickets_active ON tickets(created_at DESC)
WHERE status IN ('New', 'Investigating');

-- Covering index for list view queries
CREATE INDEX idx_tickets_list_view ON tickets(
    functional_group, status, severity, created_at DESC
) INCLUDE (ticket_id, title, assignee_team);

-- Connection pooling via PgBouncer
-- Max connections: 20 per service pod
-- Pool mode: transaction (most efficient for async services)
```

---

> **Next:** [Section 8 — API Design](08-api-design.md)
