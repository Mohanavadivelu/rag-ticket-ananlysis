# Section 12: Integration Design

## 🔗 Overview

The Ticket Intelligence System is designed to integrate with existing enterprise ticketing systems through webhooks, REST API polling, and direct database sync.

---

## 🔵 Jira Integration

### Webhook-Based (Real-Time)

```python
# services/webhook_service/jira_handler.py
from fastapi import APIRouter, Request, HTTPException
import hmac, hashlib

router = APIRouter(prefix="/webhooks/jira")

@router.post("/events")
async def handle_jira_webhook(request: Request, ingestion_svc: IngestionService = Depends()):
    """Receive Jira issue events via webhook."""

    # Verify Jira webhook signature
    signature = request.headers.get("X-Hub-Signature")
    body = await request.body()
    if not verify_jira_signature(body, signature, settings.JIRA_WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid webhook signature")

    event = await request.json()
    event_type = event.get("webhookEvent")

    if event_type in ("jira:issue_created", "jira:issue_updated"):
        # Normalize Jira issue to internal ticket format
        ticket = normalize_jira_to_ticket(event["issue"])

        # Ingest ticket asynchronously
        await ingestion_svc.ingest(ticket, source="jira")
        return {"status": "accepted"}

    return {"status": "ignored", "event_type": event_type}


def normalize_jira_to_ticket(jira_issue: dict) -> TicketInput:
    """Transform Jira issue fields to internal ticket schema."""
    fields = jira_issue["fields"]
    return TicketInput(
        ticket_id=jira_issue["key"],           # e.g. "BT-1247"
        external_id=jira_issue["id"],
        source_system="jira",
        title=fields.get("summary", ""),
        description=fields.get("description", ""),
        functional_group=map_jira_component(fields.get("components", [])),
        status=map_jira_status(fields["status"]["name"]),
        severity=map_jira_priority(fields.get("priority", {}).get("name", "Medium")),
        analysis_notes=fields.get("comment", {}).get("comments", [{}])[-1].get("body", ""),
        tags=extract_jira_labels(fields.get("labels", [])),
        reporter_email=fields.get("reporter", {}).get("emailAddress"),
    )
```

### Jira Status Mapping

```python
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

---

## 🔌 REST API Integration (Custom Backend)

Any external system can integrate via the standard REST API:

```python
import httpx

class TicketIntelligenceClient:
    """SDK for integrating with the Ticket Intelligence System."""

    def __init__(self, base_url: str, api_key: str):
        self.client = httpx.AsyncClient(
            base_url=base_url,
            headers={"X-API-Key": api_key},
            timeout=30.0
        )

    async def ingest_ticket(self, ticket: dict) -> dict:
        """Push a ticket to the intelligence system."""
        response = await self.client.post("/api/v1/tickets", json=ticket)
        response.raise_for_status()
        return response.json()

    async def analyze_ticket(self, ticket_id: str) -> dict:
        """Get AI analysis for a ticket."""
        response = await self.client.post(
            "/api/v1/analysis/similarity",
            json={"ticket_id": ticket_id, "analysis_mode": "full"}
        )
        response.raise_for_status()
        return response.json()

    async def search_similar(self, description: str, functional_group: str) -> list:
        """Find similar historical tickets."""
        response = await self.client.post(
            "/api/v1/tickets/search",
            json={
                "query": {"description": description, "functional_group": functional_group},
                "options": {"top_k": 10}
            }
        )
        response.raise_for_status()
        return response.json()["results"]
```

---

## 📤 Outbound Integrations

### Write Analysis Back to Jira

```python
async def update_jira_with_analysis(jira_key: str, analysis: AnalysisResult):
    """Post AI analysis as a Jira comment."""
    comment_body = f"""
*🤖 AI Ticket Analysis*

*Duplicate Detection:* {'⚠️ Potential Duplicate of ' + analysis.duplicate_of if analysis.is_duplicate else '✅ No duplicates found'}
*Rejection Probability:* {analysis.rejection_probability}%
*Root Cause:* {analysis.root_cause_category} (confidence: {analysis.root_cause_confidence}%)
*Recommendation:* {analysis.recommendation}
*Suggested Team:* {analysis.assignee_team}

*Summary:* {analysis.summary}
"""
    async with httpx.AsyncClient() as client:
        await client.post(
            f"{JIRA_BASE_URL}/rest/api/3/issue/{jira_key}/comment",
            auth=(JIRA_USER, JIRA_TOKEN),
            json={"body": comment_body}
        )
```

---

## 🔄 Sync Strategies

| Integration Type | Method | Latency | Reliability |
|-----------------|--------|---------|-------------|
| Jira Webhook | Push (real-time) | < 1s | High (with retry) |
| Custom Backend REST | Pull/Push | On-demand | High |
| Batch CSV Import | Scheduled job | Daily | High |
| Database Direct Sync | Change Data Capture | Near real-time | Very High |

---

> **Next:** [Section 13 — Security Design](13-security.md)
