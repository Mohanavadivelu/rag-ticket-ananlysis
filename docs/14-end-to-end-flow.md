# Section 14: Complete End-to-End Flow

## 🎬 Real-World Example: New Bluetooth Ticket Arrives

**Scenario:** An engineer reports a new ticket for Bluetooth audio dropout during phone calls on Android 14.

---

## 📋 Full Flow Walkthrough

### Phase 1: Ticket Submission (T+0ms)

```
Engineer submits via REST API:

POST /api/v1/tickets
{
  "ticket_id": "TKT-2024-01247",
  "functional_group": "Bluetooth",
  "title": "BT audio drops when receiving call on Android 14",
  "description": "When a phone call is received, Bluetooth audio routes back to
                  phone speaker instead of car speakers. Reproducible 100% on
                  Android 14 devices. Firmware v3.3.0.",
  "analysis_notes": "HFP negotiation logs show A2DP stream still active when
                     HFP connection is initiated.",
  "status": "New",
  "severity": "High",
  "tags": ["bluetooth", "hfp", "a2dp", "android14"]
}
```

**System Response (T+5ms):**
```json
{
  "success": true,
  "ticket_id": "TKT-2024-01247",
  "embedding_status": "queued",
  "message": "Ticket created. Analysis will be available in ~2 minutes."
}
```

---

### Phase 2: Deduplication Check (T+5ms)

```python
# Fast hash-based pre-check before DB insert
text_hash = sha256(format_ticket_for_embedding(ticket))

existing = await db.query(
    "SELECT ticket_id FROM tickets WHERE text_hash = $1", text_hash
)
# Result: None — not a hash-level duplicate, proceed
```

---

### Phase 3: Async Embedding (T+100ms via Redis Stream)

```
Redis Stream Consumer picks up: ticket.ingested

Embedding Service:
  1. Format ticket text:
     "Functional Group: Bluetooth
      Description: When a phone call is received, Bluetooth audio routes...
      Analysis: HFP negotiation logs show A2DP stream still active...
      Status: New
      Tags: bluetooth, hfp, a2dp, android14"

  2. Check Redis embedding cache: MISS (new ticket)

  3. BGE-large-en-v1.5 inference:
     → [0.023, -0.412, 0.891, ..., 0.334]  (1024-dim vector)

  4. Upsert to Qdrant:
     collection: tickets
     id: 3fa85f64-5717-4562-b3fc-2c963f66afa6
     vector: [1024 floats]
     payload: {ticket_id, status, functional_group, ...}

  5. Update PostgreSQL:
     tickets SET embedding_id='...', embedding_version='v1'
```

---

### Phase 4: Analysis Request (T+200ms, triggered by engineer or auto)

```
POST /api/v1/analysis/similarity
{"ticket_id": "TKT-2024-01247", "analysis_mode": "full"}
```

**Step 4a: Retrieve Embedding (T+201ms)**
```python
embedding = await qdrant.retrieve_vector("TKT-2024-01247")
# Returns: [0.023, -0.412, 0.891, ...]
```

**Step 4b: Auto-Select Steering Vectors (T+202ms)**
```python
steering_vectors = SteeringService.select_vectors(
    functional_group="Bluetooth",       → s_bluetooth (α=0.3)
    description_keywords=["hfp","call"] → s_firmware (α=0.35)
    analysis_mode="full"                → s_rejected (α=0.4)
)
```

**Step 4c: Apply Steering (T+203ms)**
```python
q = [0.023, -0.412, 0.891, ...]   # original
q_steered = normalize(
    q + 0.3 × s_bluetooth + 0.35 × s_firmware + 0.4 × s_rejected
)
# q_steered now biased toward BT firmware rejected tickets
```

**Step 4d: Qdrant Search (T+211ms)**
```python
results = qdrant.search(
    collection="tickets",
    query_vector=q_steered,
    filter=None,   # no status filter for full analysis
    limit=10,
    score_threshold=0.70
)
# Returns 10 most similar tickets with scores
```

**Step 4e: Top Results**
```
Rank 1: TKT-2024-00847  score=0.923  status=Fixed    "BT HFP audio routing fail Android 12"
Rank 2: TKT-2024-00612  score=0.889  status=Rejected "BT audio dropout Android 12 known issue"
Rank 3: TKT-2024-01180  score=0.871  status=Fixed    "HFP A2DP conflict during call iOS 17"
Rank 4: TKT-2023-00934  score=0.845  status=Fixed    "BT call audio drops firmware race condition"
...
```

**Step 4f: Metadata Join from PostgreSQL (T+214ms)**
```python
# Fetch full descriptions + resolutions from PostgreSQL
# (Redis cache hit for recently accessed tickets)
pg_tickets = await fetch_metadata(
    ["TKT-2024-00847", "TKT-2024-00612", ...]
)
```

---

### Phase 5: LLM Reasoning (T+300ms → T+2100ms)

**Prompt sent to GPT-4o:**
```
You are a senior embedded systems engineer...

## NEW TICKET TO ANALYZE
Ticket ID: TKT-2024-01247
Functional Group: Bluetooth
Description: When a phone call is received, Bluetooth audio routes back to
             phone speaker...

## RETRIEVED SIMILAR HISTORICAL TICKETS

### Context Ticket 1 (Similarity: 0.923)
- ID: TKT-2024-00847
- Status: Fixed
- Description: Bluetooth audio drops to phone speaker when call starts...
- Resolution: Updated HFP stack to sequence A2DP suspend before HFP connect.
              Fix deployed in firmware v3.2.2.

### Context Ticket 2 (Similarity: 0.889)
- ID: TKT-2024-00612
- Status: Rejected
- Resolution: Rejected - Known Android 12 limitation. Fixed natively in Android 13.

[... 8 more context tickets ...]

## YOUR ANALYSIS TASK
...
```

---

### Phase 6: Final Analysis Response (T+2100ms)

```json
{
  "ticket_id": "TKT-2024-01247",
  "processing_time_ms": 1843,
  "analysis": {
    "duplicate_detection": {
      "is_duplicate": false,
      "duplicate_score": 0.84,
      "reason": "Similar to TKT-2024-00847 but different OS version (Android 14 vs 12)"
    },
    "rejection_prediction": {
      "rejection_probability": 15,
      "recommendation": "Accept"
    },
    "root_cause": {
      "category": "firmware_bug",
      "confidence": 87,
      "explanation": "HFP A2DP sequencing regression on Android 14 Bluetooth stack"
    },
    "recommended_action": {
      "action": "Port HFP fix from v3.2.2 to Android 14 variant",
      "priority": "High",
      "assignee_team": "firmware-team",
      "steps": [
        "Reproduce on Pixel 8 (Android 14)",
        "Capture HCI BT logs",
        "Apply v3.2.2 A2DP suspend fix to Android 14 path",
        "Regression test on 5+ Android 14 devices"
      ]
    },
    "resolution_reference": {
      "reference_ticket": "TKT-2024-00847",
      "resolution_summary": "Port firmware v3.2.2 HFP fix to Android 14",
      "applicable": true
    },
    "confidence_score": 87,
    "summary": "Android 14 regression of fixed HFP issue. High-confidence firmware fix available."
  }
}
```

---

### Phase 7: Downstream Actions (T+2101ms)

```
┌─ Store analysis in PostgreSQL (analysis_results table)
├─ Cache analysis in Redis (TTL: 24h)
├─ Update ticket status if auto-triage enabled
├─ Notify engineer via webhook/email
└─ If is_duplicate → link tickets in DB
```

---

## ⏱️ Total Timeline Summary

| Phase | Duration | Key Action |
|-------|---------|-----------|
| Ticket submission | 0–5ms | API validation + DB insert |
| Deduplication | 5–10ms | Hash check |
| Embedding (async) | 100–200ms | BGE-large inference + Qdrant upsert |
| Analysis request | 200–300ms | Retrieve + steer + search |
| LLM reasoning | 300–2100ms | GPT-4o analysis |
| **Total** | **~2.1 seconds** | **End-to-end** |

---

> **Next:** [Section 15 — Advanced Features](15-advanced-features.md)
