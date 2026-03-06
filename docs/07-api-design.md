# API Design, Integration & Security

## Overview

The API is built with **FastAPI** and follows RESTful conventions with OpenAPI 3.0 auto-documentation. All endpoints are versioned under `/api/v1/`, protected by JWT or API key auth, and return structured JSON.

---

## Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/tickets` | Ingest a new ticket |
| `GET` | `/api/v1/tickets/{id}` | Retrieve ticket by ID |
| `PATCH` | `/api/v1/tickets/{id}` | Update ticket fields |
| `GET` | `/api/v1/tickets` | List/filter tickets (paginated) |
| `POST` | `/api/v1/tickets/search` | Two-stage semantic search |
| `POST` | `/api/v1/analysis/similarity` | Full RAG analysis pipeline |
| `GET` | `/api/v1/analysis/{ticket_id}` | Get stored analysis result |
| `POST` | `/api/v1/analysis/{id}/feedback` | Submit engineer feedback on analysis |
| `GET` | `/api/v1/steering` | List all steering vectors |
| `POST` | `/api/v1/steering/compute` | Manually trigger steering vector refresh |
| `POST` | `/api/v1/webhooks/jira` | Receive Jira issue events |
| `GET` | `/health` | Liveness check |
| `GET` | `/ready` | Readiness check (BGE model loaded) |
| `GET` | `/metrics` | Prometheus metrics scrape endpoint |

---

## Ticket Ingestion

### `POST /api/v1/tickets`

**Request:**

```json
{
  "ticket_id": "TKT-2024-01247",
  "functional_group": "Bluetooth",
  "title": "BT audio drops when receiving call on Android 14",
  "description": "Device disconnects Bluetooth audio when call received. Audio routes to phone speaker. 100% reproducible on Android 14, firmware v3.3.0.",
  "analysis_notes": "HFP negotiation failure in BT logs. A2DP not suspended before HFP activation.",
  "status": "New",
  "severity": "High",
  "platform": "Android 14",
  "vehicle_model": "Model-X-2024",
  "firmware_version": "3.3.0",
  "tags": ["bluetooth", "hfp", "a2dp", "android", "audio-routing"],
  "statistics": { "affected_devices": 47, "reproduction_rate": "100%" }
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "ticket_id": "TKT-2024-01247",
  "internal_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "New",
  "embedding_status": "queued",
  "message": "Ticket ingested. Embedding queued for async processing.",
  "created_at": "2024-03-15T10:32:44Z"
}
```

**Response `409 Conflict` (duplicate hash):**

```json
{
  "success": false,
  "error": "DUPLICATE_TICKET",
  "message": "A ticket with identical content already exists.",
  "existing_ticket_id": "TKT-2024-01100"
}
```

---

## Semantic Search

### `POST /api/v1/tickets/search`

**Request:**

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
    "date_from": "2023-01-01T00:00:00Z"
  },
  "steering": {
    "vectors": ["rejected", "bluetooth"],
    "alpha": 0.5,
    "auto_select": false
  },
  "options": {
    "top_k": 10,
    "score_threshold": 0.70,
    "include_resolution": true
  }
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "query_id": "q-8f3d2a1b",
  "total_results": 8,
  "search_latency_ms": 55,
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
    }
  ]
}
```

---

## Full RAG Analysis

### `POST /api/v1/analysis/similarity`

**Request:**

```json
{
  "ticket_id": "TKT-2024-01247",
  "analysis_mode": "full",
  "steering_config": { "auto_select": true },
  "llm_config": { "model": "gpt-4o", "temperature": 0.1 },
  "context_config": { "top_k": 10, "score_threshold": 0.65 }
}
```

**Response `200 OK`:** Returns full `AnalysisResult` JSON — see [05-llm-analysis.md](05-llm-analysis.md) for the complete schema example.

---

## Engineer Feedback

### `POST /api/v1/analysis/{analysis_id}/feedback`

Allows engineers to correct LLM analyses. Accumulated feedback drives automatic steering vector improvement.

**Request:**

```json
{
  "is_correct": false,
  "feedback_type": "root_cause",
  "corrected_value": { "category": "hardware_limitation" },
  "notes": "This is a known antenna placement issue, not a firmware bug"
}
```

`feedback_type` options: `"duplicate"` | `"root_cause"` | `"rejection"` | `"overall"`

**Response `201 Created`:**

```json
{
  "feedback_id": "fb-3a9c12ef",
  "accepted": true,
  "steering_refresh_queued": false,
  "message": "Feedback recorded."
}
```

When the unprocessed feedback count reaches the threshold (default: 50), `steering_refresh_queued` will be `true` and a background task automatically refreshes the relevant steering vectors.

---

## Jira Integration

### `POST /api/v1/webhooks/jira`

Receives Jira issue events, verifies HMAC signature, normalizes to `TicketInput`, and calls the ingestion pipeline.

**Headers required:**
```
X-Hub-Signature: sha256=<hmac-signature>
Content-Type: application/json
```

**Supported event types:** `jira:issue_created`, `jira:issue_updated`

```python
# Field mapping: Jira → TicketInput
JIRA_STATUS_MAP = {
    "Open":        "New",
    "In Progress": "Investigating",
    "Resolved":    "Fixed",
    "Closed":      "Closed",
    "Won't Fix":   "Rejected",
    "Duplicate":   "Duplicate",
    "Invalid":     "Rejected",
}

JIRA_PRIORITY_MAP = {
    "Blocker":  "Critical",
    "Critical": "Critical",
    "Major":    "High",
    "Minor":    "Medium",
    "Trivial":  "Low",
}
```

**Response `202 Accepted`:**
```json
{ "status": "accepted", "ticket_id": "BT-1247" }
```

### Writing Analysis Back to Jira

The system can optionally post analysis results as a Jira comment:

```python
async def post_analysis_to_jira(jira_key: str, analysis: AnalysisResult):
    comment = f"""*AI Ticket Analysis*

*Duplicate:* {analysis.duplicate_detection.is_duplicate}
*Rejection Probability:* {analysis.rejection_prediction.rejection_probability}%
*Root Cause:* {analysis.root_cause.category} ({analysis.root_cause.confidence}% confidence)
*Recommended Team:* {analysis.recommended_action.assignee_team}

*Summary:* {analysis.summary}"""

    await httpx.post(
        f"{JIRA_BASE_URL}/rest/api/3/issue/{jira_key}/comment",
        auth=(JIRA_USER, JIRA_TOKEN),
        json={"body": comment}
    )
```

---

## Authentication

### JWT Bearer Token (for human users)

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

```python
# shared/auth/jwt_handler.py
def create_access_token(user_id: str, role: str, team: str) -> str:
    payload = {
        "sub": user_id,
        "role": role,
        "team": team,
        "exp": datetime.utcnow() + timedelta(hours=8),
        "jti": str(uuid.uuid4())    # Unique token ID for revocation
    }
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")

def verify_token(token: str) -> dict:
    payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
    if await redis.exists(f"revoked:{payload['jti']}"):
        raise HTTPException(401, "Token has been revoked")
    return payload
```

### API Key (for service-to-service)

```http
X-API-Key: tis_live_sk_xxxxxxxxxxxxxxxxxxxxxxxx
```

```python
# shared/auth/api_key.py
def generate_api_key() -> tuple[str, str]:
    raw_key = f"tis_live_sk_{secrets.token_urlsafe(32)}"
    key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
    return raw_key, key_hash    # raw shown once to user; hash stored in DB
```

---

## Authorization (RBAC)

```python
ROLES = {
    "admin":    ["read", "write", "delete", "admin", "steering:manage"],
    "engineer": ["read", "write", "analyze", "feedback"],
    "viewer":   ["read"],
    "service":  ["read", "write", "analyze", "ingest"],
}

def require_permission(permission: str):
    async def checker(current_user: User = Depends(get_current_user)):
        if permission not in ROLES.get(current_user.role, {}).get("permissions", []):
            raise HTTPException(403, f"Permission denied: {permission}")
        return current_user
    return checker
```

---

## Security Headers (Nginx)

```nginx
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
add_header Content-Security-Policy "default-src 'self'";
```

## Rate Limiting (Nginx)

```nginx
limit_req_zone $binary_remote_addr zone=api:10m      rate=100r/m;
limit_req_zone $binary_remote_addr zone=search:10m   rate=30r/m;
limit_req_zone $binary_remote_addr zone=analysis:10m rate=10r/m;
```

---

## Input Validation

All input is validated by Pydantic models in `shared/models/ticket.py`. Key constraints:

```python
class TicketInput(BaseModel):
    ticket_id:         str  = Field(..., max_length=50)
    description:       str  = Field(..., min_length=5, max_length=10000)
    functional_group:  FunctionalGroup               # Literal type — no free-form values
    status:            TicketStatus = "New"
    severity:          SeverityLevel = "Medium"
    tags:              Optional[List[str]] = []      # Normalized to lowercase on input

    @validator("ticket_id")
    def normalize_ticket_id(cls, v):
        return v.strip().upper()
```

---

## Data Protection

| Data | Protection |
|------|-----------|
| Ticket text | PostgreSQL encrypted at rest (AES-256) |
| API keys | SHA-256 hashed — never stored in plaintext |
| JWT secrets | Environment variables; rotate every 90 days |
| Embeddings | Qdrant disk encryption |
| OpenAI API key | Environment variable — never logged |
| PII in tickets | Reporter emails masked in structured logs |

---

## Error Response Format

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
| 401 | `UNAUTHORIZED` | Missing/invalid auth |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `TICKET_NOT_FOUND` | Ticket ID doesn't exist |
| 409 | `DUPLICATE_TICKET` | Exact duplicate detected |
| 422 | `EMBEDDING_FAILED` | Embedding generation error |
| 429 | `RATE_LIMITED` | Too many requests |
| 503 | `SERVICE_UNAVAILABLE` | Qdrant/LLM unreachable |

---

## Security Checklist

- [x] JWT with 8h expiry + Redis-based revocation
- [x] API key SHA-256 hashed, never stored plaintext
- [x] RBAC per endpoint via `require_permission()` dependency
- [x] Input validation at all API boundaries (Pydantic)
- [x] SQL injection prevention (SQLAlchemy ORM — no raw SQL in services)
- [x] TLS 1.2+ enforced at Nginx
- [x] Security headers on all responses
- [x] Rate limiting per IP at Nginx
- [x] Secrets via environment variables
- [x] Full audit trail in `audit_logs` table
- [x] Non-root container user
- [x] Jira webhook HMAC signature verification

> **Next:** [08-deployment.md](08-deployment.md)
