# Section 6: LLM Reasoning Design

## 🤖 Overview

The LLM Reasoning Service transforms retrieved ticket context into **structured, actionable intelligence**. It uses retrieved similar tickets as grounding context and generates analysis covering duplicate detection, rejection prediction, root cause identification, and resolution recommendations.

---

## 📋 How the LLM Uses Retrieved Context

The LLM never has to hallucinate — it reasons **purely from retrieved evidence**:

```
Query Ticket (new, unresolved)
        +
Retrieved Context (top-10 most similar historical tickets)
        +
Structured Prompt (role + instructions + output schema)
        │
        ▼
   LLM (GPT-4o / Claude 3.5 Sonnet)
        │
        ▼
Structured JSON Analysis Output
```

---

## 📝 Prompt Template

```python
ANALYSIS_PROMPT_TEMPLATE = """
You are a senior embedded systems engineer and ticket triage specialist with deep expertise in:
- Bluetooth (HFP, A2DP, BLE, AVRCP profiles)
- Wi-Fi (802.11 standards, WPA2/WPA3, DHCP, DNS)
- Apple CarPlay (USB and Wireless)
- Android Auto (USB and Wireless)
- Projection / HDMI / Display protocols
- Automotive infotainment firmware

You are analyzing a NEW engineering ticket by comparing it to HISTORICAL similar tickets retrieved from our knowledge base.

---

## NEW TICKET TO ANALYZE

**Ticket ID:** {ticket_id}
**Functional Group:** {functional_group}
**Status:** {status}
**Created:** {created_at}

**Description:**
{description}

**Analysis Notes:**
{analysis_notes}

---

## RETRIEVED SIMILAR HISTORICAL TICKETS

The following {context_count} tickets were retrieved from the knowledge base based on semantic similarity.
Similarity scores range from 0.0 (unrelated) to 1.0 (identical).

{context_tickets}

---

## YOUR ANALYSIS TASK

Based on the new ticket and the retrieved historical evidence, provide a structured analysis:

1. **DUPLICATE DETECTION**: Is this ticket a duplicate of any retrieved ticket? If similarity score > 0.92 and description matches, flag as duplicate.

2. **REJECTION LIKELIHOOD**: Based on patterns in rejected historical tickets, what is the probability (0-100%) that this ticket will be rejected? Explain why.

3. **ROOT CAUSE ANALYSIS**: What is the most likely root cause? Categorize as:
   - firmware_bug
   - hardware_limitation
   - configuration_error
   - known_platform_issue
   - user_error
   - external_dependency
   - unknown

4. **RECOMMENDED ACTION**: What should the engineering team do next?

5. **RESOLUTION REFERENCE**: If any retrieved ticket was successfully resolved with a similar issue, reference it and suggest applying the same fix.

6. **CONFIDENCE SCORE**: Rate your confidence in this analysis (0-100%) based on the quality of retrieved context.

---

## OUTPUT FORMAT

Respond ONLY with valid JSON matching this exact schema:

```json
{
  "ticket_id": "string",
  "analysis_timestamp": "ISO8601 datetime",
  "duplicate_detection": {
    "is_duplicate": boolean,
    "duplicate_of": "ticket_id or null",
    "duplicate_score": float,
    "reason": "explanation string"
  },
  "rejection_prediction": {
    "rejection_probability": integer (0-100),
    "rejection_reason": "string",
    "similar_rejected_tickets": ["ticket_id1", "ticket_id2"],
    "recommendation": "Accept | Reject | Needs Investigation"
  },
  "root_cause": {
    "category": "firmware_bug | hardware_limitation | configuration_error | known_platform_issue | user_error | external_dependency | unknown",
    "confidence": integer (0-100),
    "explanation": "string",
    "evidence_tickets": ["ticket_id1"]
  },
  "recommended_action": {
    "action": "string",
    "priority": "Critical | High | Medium | Low",
    "assignee_team": "string",
    "estimated_effort": "string",
    "steps": ["step1", "step2", "step3"]
  },
  "resolution_reference": {
    "found": boolean,
    "reference_ticket": "ticket_id or null",
    "resolution_summary": "string",
    "applicable": boolean
  },
  "confidence_score": integer (0-100),
  "summary": "One paragraph executive summary"
}
```
"""
```

---

## 📦 Context Ticket Formatting

Each retrieved ticket is formatted into a structured block for the prompt:

```python
def format_context_tickets(retrieved_tickets: list[EnrichedTicket]) -> str:
    """Format retrieved tickets into LLM-readable context blocks."""
    blocks = []
    for i, ticket in enumerate(retrieved_tickets, 1):
        block = f"""
### Context Ticket {i} (Similarity: {ticket.similarity_score:.3f})
- **ID:** {ticket.ticket_id}
- **Functional Group:** {ticket.functional_group}
- **Status:** {ticket.status}
- **Created:** {ticket.created_at}
- **Description:** {ticket.description[:400]}
- **Analysis:** {ticket.analysis_notes[:300] if ticket.analysis_notes else 'None'}
- **Resolution:** {ticket.resolution[:300] if ticket.resolution else 'Not resolved'}
"""
        blocks.append(block.strip())

    return "\n\n---\n\n".join(blocks)
```

---

## 🧠 LLM Service Implementation

```python
# services/llm/llm_service.py
from openai import AsyncOpenAI
import json

class LLMReasoningService:
    def __init__(self, settings: Settings, redis: Redis):
        self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.LLM_MODEL  # "gpt-4o" or "gpt-4o-mini"
        self.redis = redis

    async def analyze_ticket(
        self,
        query_ticket: TicketInput,
        context_tickets: list[EnrichedTicket]
    ) -> AnalysisResult:
        """Generate LLM analysis with retrieved context."""

        # Check analysis cache
        cache_key = f"analysis:{query_ticket.ticket_id}:v1"
        cached = await self.redis.get(cache_key)
        if cached:
            return AnalysisResult(**json.loads(cached))

        # Format prompt
        context_str = format_context_tickets(context_tickets)
        prompt = ANALYSIS_PROMPT_TEMPLATE.format(
            ticket_id=query_ticket.ticket_id,
            functional_group=query_ticket.functional_group,
            status=query_ticket.status,
            created_at=query_ticket.created_at,
            description=query_ticket.description,
            analysis_notes=query_ticket.analysis_notes or "Not provided",
            context_count=len(context_tickets),
            context_tickets=context_str
        )

        # Call LLM with structured output
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a precise engineering ticket analysis system. Always respond with valid JSON only."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.1,        # Low temperature for consistent, factual analysis
            max_tokens=2000,
            response_format={"type": "json_object"}  # Force JSON output (GPT-4o)
        )

        # Parse response
        raw_json = response.choices[0].message.content
        analysis_data = json.loads(raw_json)

        result = AnalysisResult(**analysis_data)

        # Cache result for 24 hours
        await self.redis.setex(cache_key, 86400, json.dumps(analysis_data))

        return result
```

---

## 🎯 Detection Logic

### Duplicate Detection

```python
# Triggers when:
# - similarity_score >= 0.92 AND
# - same functional_group AND
# - LLM confirms semantic overlap

if analysis.duplicate_detection.is_duplicate:
    # Auto-link tickets in PostgreSQL
    await db.link_duplicate(
        original_id=analysis.duplicate_detection.duplicate_of,
        duplicate_id=query_ticket.ticket_id
    )
    # Notify reporter
    await notify_duplicate(query_ticket, analysis.duplicate_detection)
```

### Rejection Pattern Detection

```python
# LLM identifies rejection patterns from:
# - Retrieved tickets with status="Rejected"
# - Rejection reasons in resolution fields
# - Known patterns: "hardware limitation", "out of scope", "by design"

if analysis.rejection_prediction.rejection_probability >= 75:
    logger.warning(
        "Ticket %s has high rejection probability: %d%%",
        query_ticket.ticket_id,
        analysis.rejection_prediction.rejection_probability
    )
    # Add to review queue for senior engineer
    await review_queue.add(query_ticket.ticket_id, priority="high")
```

### Root Cause Identification

```python
# Root cause is identified by:
# 1. Cross-referencing description with historical patterns
# 2. Matching error signatures in analysis_notes
# 3. Consulting resolved tickets with same root cause

ROOT_CAUSE_TEAMS = {
    "firmware_bug":        "firmware-team",
    "hardware_limitation": "hardware-team",
    "configuration_error": "integration-team",
    "known_platform_issue":"platform-team",
    "user_error":          "documentation-team",
    "external_dependency": "partnerships-team",
    "unknown":             "triage-team"
}
```

---

## 📊 Example LLM Output

**New Ticket:** "BT audio drops when receiving a phone call on Android 14"

```json
{
  "ticket_id": "TKT-2024-01247",
  "analysis_timestamp": "2024-03-15T10:32:44Z",
  "duplicate_detection": {
    "is_duplicate": false,
    "duplicate_of": null,
    "duplicate_score": 0.84,
    "reason": "TKT-2024-00847 is similar (HFP audio routing) but on Android 12. Different OS version, likely same root cause but not identical."
  },
  "rejection_prediction": {
    "rejection_probability": 15,
    "rejection_reason": "Similar issues on Android 12 were fixed in firmware v3.2.2. Android 14 may have regressed.",
    "similar_rejected_tickets": [],
    "recommendation": "Accept"
  },
  "root_cause": {
    "category": "firmware_bug",
    "confidence": 87,
    "explanation": "Based on TKT-2024-00847 (similarity: 0.84), this is likely an HFP profile sequencing issue where A2DP stream is not suspended before HFP handshake. Android 14 changed the Bluetooth stack timing requirements.",
    "evidence_tickets": ["TKT-2024-00847", "TKT-2024-00612"]
  },
  "recommended_action": {
    "action": "Investigate HFP A2DP sequencing on Android 14 Bluetooth stack",
    "priority": "High",
    "assignee_team": "firmware-team",
    "estimated_effort": "3-5 days investigation + 2 days fix",
    "steps": [
      "Reproduce on Android 14 test device (Pixel 8)",
      "Capture Bluetooth HCI logs during audio drop",
      "Compare A2DP/HFP timing with Android 12 reference (TKT-2024-00847 fix)",
      "Apply fix from firmware v3.2.2 to Android 14 variant",
      "Regression test on 5+ Android 14 devices"
    ]
  },
  "resolution_reference": {
    "found": true,
    "reference_ticket": "TKT-2024-00847",
    "resolution_summary": "Updated HFP stack to properly sequence A2DP suspend before HFP connect. Fix in firmware v3.2.2.",
    "applicable": true
  },
  "confidence_score": 87,
  "summary": "This ticket is likely a regression of TKT-2024-00847 (Android 12 HFP audio routing bug). The fix applied in firmware v3.2.2 may not fully cover Android 14's updated Bluetooth stack timing. Recommend assigning to firmware team with high priority to port the existing fix to Android 14."
}
```

---

> **Next:** [Section 7 — Database Design](07-database-design.md)
