# Section 8: API Design

## 🌐 Overview

The API is built with **FastAPI** and follows RESTful conventions with OpenAPI 3.0 documentation. All endpoints are versioned under `/api/v1/`, protected by JWT or API key authentication, and return structured JSON responses.

---

## 📋 Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/tickets` | Ingest a new ticket |
| `GET` | `/api/v1/tickets/{id}` | Retrieve ticket by ID |
| `PATCH` | `/api/v1/tickets/{id}` | Update ticket fields |
| `GET` | `/api/v1/tickets` | List/filter tickets |
| `POST` | `/api/v1/tickets/search` | Semantic similarity search |
| `POST` | `/api/v1/analysis/similarity` | Full RAG analysis pipeline |
| `GET` | `/api/v1/analysis/{ticket_id}` | Get stored analysis result |
| `POST` | `/api/v1/steering/compute` | Compute/refresh steering vector |
| `GET` | `/api/v1/steering` | List all steering vectors |
| `GET` | `/api/v1/health` | Health check |

---

## 📨 POST `/api/v1/tickets` — Ingest New Ticket

### Request

```http
POST /api/v1/tickets
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

```json
{
  "ticket_id": "TKT-2024-01247",
  "functional_group": "Bluetooth",
  "title": "BT audio drops when receiving call on Android 14",
  "description": "Device disconnects Bluetooth audio when incoming call is received. Audio routes to phone speaker instead of car. Happens consistently with Android 14 devices on firmware v3.3.0.",
  "analysis_notes": "Observed HFP profile negotiation failure in BT logs. A2DP not suspended before HFP activation.",
  "resolution": null,
  "status": "New",
  "severity": "High",
  "platform": "Android 14",
  "vehicle_model": "Model-X-2024",
  "firmware_version": "3.3.0",
  "tags": ["bluetooth", "hfp", "a2dp", "android", "audio-routing"],
  "statistics": {
    "affected_devices": 47,
    "reproduction_rate": "100%",
    "first_reported": "2024-03-15"
  }
}
```

### Response `201 Created`

```json
{
  "success": true,
  "ticket_id": "TKT-2024-01247",
  "internal_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "New",
  "embedding_status": "queued",
  "message": "Ticket ingested successfully. Embedding queued for async processing.",
  "created_at": "2024-03-15T10:32:44Z"
}
```

### Response `409 Conflict` (Duplicate Hash)

```json
{
  "success": false,
  "error": "DUPLICATE_TICKET",
  "message": "A ticket with identical content already exists.",
  "existing_ticket_id": "TKT-2024-01100",
  "similarity": 0.98
}
```

---

## 🔍 POST `/api/v1/tickets/search` — Semantic Similarity Search

### Request

```http
POST /api/v1/tickets/search
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

```json
{
  "query": {
    "description": "Bluetooth audio drops during phone call",
    "functional_group": "Bluetooth",
    "analysis_notes": "HFP profile issue suspected"
  },
  "filters": {
    "status": ["Fixed", "Rejected"],
    "functional_group": "Bluetooth",
    "severity": ["High", "Critical"],
    "date_from": "2023-01-01T00:00:00Z",
    "date_to": "2024-12-31T23:59:59Z"
  },
  "steering": {
    "vectors": ["rejected", "bluetooth"],
    "alpha": 0.5
  },
  "options": {
    "top_k": 10,
    "score_threshold": 0.75,
    "include_resolution": true
  }
}
```

### Response `200 OK`

```json
{
  "success": true,
  "query_id": "q-8f3d2a1b",
  "total_results": 8,
  "search_latency_ms": 47,
  "steering_applied": ["rejected", "bluetooth"],
  "results": [
    {
      "rank": 1,
      "ticket_id": "TKT-2024-00847",
      "similarity_score": 0.9234,
      "final_score": 0.9434,
      "functional_group": "Bluetooth",
      "status": "Fixed",
      "severity": "High",
      "title": "BT audio routing fails during HFP call handoff",
      "description": "Bluetooth audio drops to phone speaker when call starts...",
      "resolution": "Updated HFP stack in firmware v3.2.2",
      "created_at": "2024-01-15T10:30:00Z",
      "resolved_at": "2024-01-20T14:22:00Z"
    },
    {
      "rank": 2,
      "ticket_id": "TKT-2024-00612",
      "similarity_score": 0.8891,
      "final_score": 0.9091,
      "functional_group": "Bluetooth",
      "status": "Rejected",
      "severity": "Medium",
      "title": "BT audio dropout - Android 12 known issue",
      "description": "Intermittent BT audio dropout on Android 12 devices...",
      "resolution": "Rejected - Known Android 12 limitation, fixed in Android 13",
      "created_at": "2023-11-05T08:15:00Z",
      "resolved_at": "2023-11-10T16:00:00Z"
    }
  ]
}
```

---

## 🧠 POST `/api/v1/analysis/similarity` — Full RAG Analysis

### Request

```http
POST /api/v1/analysis/similarity
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

```json
{
  "ticket_id": "TKT-2024-01247",
  "analysis_mode": "full",
  "steering_config": {
    "auto_select": true,
    "override_vectors": null
  },
  "llm_config": {
    "model": "gpt-4o",
    "temperature": 0.1
  },
  "context_config": {
    "top_k": 10,
    "score_threshold": 0.72,
    "include_rejected": true,
    "include_duplicates": false
  }
}
```

### Response `200 OK`

```json
{
  "success": true,
  "ticket_id": "TKT-2024-01247",
  "analysis_id": "ana-9c4e7f2a",
  "processing_time_ms": 1843,
  "context_used": {
    "ticket_count": 10,
    "steering_applied": ["rejected", "bluetooth", "firmware_root_cause"],
    "top_similarity_score": 0.9234
  },
  "analysis": {
    "duplicate_detection": {
      "is_duplicate": false,
      "duplicate_of": null,
      "duplicate_score": 0.84,
      "reason": "TKT-2024-00847 is similar but on Android 12. Different OS version."
    },
    "rejection_prediction": {
      "rejection_probability": 15,
      "rejection_reason": "Similar issue fixed in v3.2.2. Android 14 regression suspected.",
      "similar_rejected_tickets": [],
      "recommendation": "Accept"
    },
    "root_cause": {
      "category": "firmware_bug",
      "confidence": 87,
      "explanation": "HFP A2DP sequencing issue. A2DP not suspended before HFP connect.",
      "evidence_tickets": ["TKT-2024-00847", "TKT-2024-00612"]
    },
    "recommended_action": {
      "action": "Investigate HFP A2DP sequencing on Android 14",
      "priority": "High",
      "assignee_team": "firmware-team",
      "estimated_effort": "3-5 days",
      "steps": [
        "Reproduce on Android 14 test device (Pixel 8)",
        "Capture HCI Bluetooth logs",
        "Compare with Android 12 fix in v3.2.2",
        "Port fix and regression test"
      ]
    },
    "resolution_reference": {
      "found": true,
      "reference_ticket": "TKT-2024-00847",
      "resolution_summary": "Updated HFP stack to sequence A2DP suspend before HFP connect. Fix in v3.2.2.",
      "applicable": true
    },
    "confidence_score": 87,
    "summary": "Likely regression of TKT-2024-00847 on Android 14. Apply firmware v3.2.2 HFP fix."
  }
}
```

---

## 📄 GET `/api/v1/tickets/{id}` — Get Ticket by ID

### Request

```http
GET /api/v1/tickets/TKT-2024-01247
Authorization: Bearer <jwt_token>
```

### Response `200 OK`

```json
{
  "ticket_id": "TKT-2024-01247",
  "internal_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "functional_group": "Bluetooth",
  "title": "BT audio drops when receiving call on Android 14",
  "description": "...",
  "analysis_notes": "...",
  "resolution": null,
  "status": "New",
  "severity": "High",
  "platform": "Android 14",
  "vehicle_model": "Model-X-2024",
  "tags": ["bluetooth", "hfp", "a2dp"],
  "statistics": { "affected_devices": 47 },
  "embedding_status": "indexed",
  "created_at": "2024-03-15T10:32:44Z",
  "updated_at": "2024-03-15T10:32:44Z",
  "latest_analysis": {
    "analysis_id": "ana-9c4e7f2a",
    "recommendation": "Accept",
    "root_cause_category": "firmware_bug",
    "confidence_score": 87,
    "analyzed_at": "2024-03-15T10:32:47Z"
  }
}
```

---

## ⚠️ Error Response Format

All errors follow a consistent format:

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human readable description",
  "details": {},
  "request_id": "req-abc123",
  "timestamp": "2024-03-15T10:32:44Z"
}
```

| HTTP Code | Error Code | Meaning |
|-----------|-----------|---------|
| 400 | `VALIDATION_ERROR` | Invalid request body |
| 401 | `UNAUTHORIZED` | Missing/invalid auth token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `TICKET_NOT_FOUND` | Ticket ID doesn't exist |
| 409 | `DUPLICATE_TICKET` | Exact duplicate detected |
| 422 | `EMBEDDING_FAILED` | Embedding generation error |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |
| 503 | `SERVICE_UNAVAILABLE` | Qdrant/LLM unreachable |

---

## 🔒 Authentication Headers

```http
# Option 1: JWT Bearer Token
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Option 2: API Key (for service-to-service)
X-API-Key: tis_live_sk_xxxxxxxxxxxxxxxxxxxxxxxx
```

---

> **Next:** [Section 9 — Folder Structure](09-folder-structure.md)
