# LLM Analysis

## Overview

The LLM analysis module transforms retrieved ticket context into **structured, actionable intelligence**. It uses the top-10 retrieved tickets as grounding context and generates a JSON analysis covering duplicate detection, rejection prediction, root cause, and recommended action.

The LLM never hallucinate — it reasons purely from retrieved evidence. Over time, the prompt is enriched with **gold examples** — engineer-confirmed correct analyses — that calibrate GPT-4o for each functional group without any fine-tuning (see [09-learning-loop.md](09-learning-loop.md#learning-mechanism-3--dynamic-few-shot-prompt-injection)).

```text
New Ticket (unresolved)
    +
Top-3 Gold Examples (engineer-confirmed, same functional group — optional)
    +
Top-10 Retrieved Tickets (two-stage BM25 + Qdrant)
    +
Structured Prompt (role + instructions + output schema)
    │
    ▼
GPT-4o (response_format: json_object)
    │
    ▼
Structured JSON → AnalysisResult (Pydantic)
    │
    ▼
PostgreSQL (analysis_results table) + Redis cache (24h TTL)
```

---

## Prompt Template

The template includes an optional `{gold_examples_block}` section injected between the role block and the retrieved tickets. It is omitted when `gold_count == 0`.

```python
# app/modules/llm/prompts.py
ANALYSIS_PROMPT_TEMPLATE = """
You are a senior embedded systems engineer and ticket triage specialist with deep expertise in:
- Bluetooth (HFP, A2DP, BLE, AVRCP profiles)
- Wi-Fi (802.11 standards, WPA2/WPA3, DHCP, DNS)
- Apple CarPlay (USB and Wireless)
- Android Auto (USB and Wireless)
- Projection / HDMI / Display protocols
- Automotive infotainment firmware

You are analyzing a NEW engineering ticket by comparing it to HISTORICAL similar tickets
retrieved from the knowledge base.

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

{gold_examples_block}
---

## RETRIEVED SIMILAR HISTORICAL TICKETS

The following {context_count} tickets were retrieved based on semantic similarity.
Similarity scores range from 0.0 (unrelated) to 1.0 (identical).

{context_tickets}

---

## YOUR ANALYSIS TASK

Based on the new ticket and the retrieved historical evidence, provide:

1. **DUPLICATE DETECTION**: Is this a duplicate of any retrieved ticket?
   Flag as duplicate if similarity_score >= 0.92 AND description matches semantically.

2. **REJECTION LIKELIHOOD**: Based on patterns in rejected historical tickets,
   what is the probability (0-100%) this will be rejected? Explain why.

3. **ROOT CAUSE ANALYSIS**: Most likely root cause category:
   firmware_bug | hardware_limitation | configuration_error |
   known_platform_issue | user_error | external_dependency | unknown

4. **RECOMMENDED ACTION**: What should the engineering team do next?

5. **RESOLUTION REFERENCE**: If any retrieved ticket was resolved with a similar issue,
   reference it and suggest applying the same fix.

6. **CONFIDENCE SCORE**: Rate your confidence (0-100%) based on retrieved context quality.

---

## OUTPUT FORMAT

Respond ONLY with valid JSON matching this exact schema:

{
  "ticket_id": "string",
  "analysis_timestamp": "ISO8601 datetime",
  "duplicate_detection": {
    "is_duplicate": boolean,
    "duplicate_of": "ticket_id or null",
    "duplicate_score": float,
    "reason": "string"
  },
  "rejection_prediction": {
    "rejection_probability": integer (0-100),
    "rejection_reason": "string",
    "similar_rejected_tickets": ["ticket_id"],
    "recommendation": "Accept | Reject | Needs Investigation"
  },
  "root_cause": {
    "category": "firmware_bug | hardware_limitation | ...",
    "confidence": integer (0-100),
    "explanation": "string",
    "evidence_tickets": ["ticket_id"]
  },
  "recommended_action": {
    "action": "string",
    "priority": "Critical | High | Medium | Low",
    "assignee_team": "string",
    "estimated_effort": "string",
    "steps": ["step1", "step2"]
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
"""
```

---

## Context Ticket Formatting

```python
def format_context_tickets(retrieved_tickets: list[EnrichedTicket]) -> str:
    """Format retrieved tickets into LLM-readable context blocks."""
    blocks = []
    for i, ticket in enumerate(retrieved_tickets, 1):
        block = f"""### Context Ticket {i} (Similarity: {ticket.similarity_score:.3f})
- **ID:** {ticket.ticket_id}
- **Functional Group:** {ticket.functional_group}
- **Status:** {ticket.status}
- **Created:** {ticket.created_at}
- **Description:** {ticket.description[:400]}
- **Analysis:** {ticket.analysis_notes[:300] if ticket.analysis_notes else 'None'}
- **Resolution:** {ticket.resolution[:300] if ticket.resolution else 'Not resolved'}"""
        blocks.append(block.strip())

    return "\n\n---\n\n".join(blocks)


def format_gold_examples(gold_shots: list) -> str:
    """
    Render confirmed-correct analyses as a few-shot block.
    Returns empty string if no gold examples — the template handles omission.
    """
    if not gold_shots:
        return ""

    header = (
        "## CONFIRMED ANALYSIS EXAMPLES (engineer-validated for this functional group)\n\n"
        "The following examples were confirmed correct by engineers. "
        "Use them as calibration references — your analysis should be consistent "
        "with these validated patterns.\n"
    )
    blocks = [f"### Validated Example {i+1}\n{ex.example_summary}"
              for i, ex in enumerate(gold_shots)]
    return header + "\n\n".join(blocks) + "\n\n---\n\n"
```

---

## LLM Service Implementation

```python
# app/modules/llm/service.py
from openai import AsyncOpenAI

class LLMService:
    def __init__(self, settings: Settings, redis: Redis):
        self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.LLM_MODEL        # "gpt-4o"
        self.temperature = settings.LLM_TEMPERATURE  # 0.1
        self.redis = redis

    async def analyze_ticket(
        self,
        query_ticket: TicketInput,
        context_tickets: list[EnrichedTicket],
        gold_shots: list = []        # confirmed-correct examples for this functional group
    ) -> AnalysisResult:

        # Cache check (24h TTL — analyses don't change for same ticket)
        cache_key = f"analysis:{query_ticket.ticket_id}:v1"
        if cached := await self.redis.get(cache_key):
            return AnalysisResult.model_validate_json(cached)

        # Format prompt — gold_examples_block is empty string when no gold shots exist
        context_str = format_context_tickets(context_tickets)
        gold_block   = format_gold_examples(gold_shots)
        prompt = ANALYSIS_PROMPT_TEMPLATE.format(
            ticket_id=query_ticket.ticket_id,
            functional_group=query_ticket.functional_group,
            status=query_ticket.status,
            created_at=query_ticket.created_at,
            description=query_ticket.description,
            analysis_notes=query_ticket.analysis_notes or "Not provided",
            context_count=len(context_tickets),
            context_tickets=context_str,
            gold_examples_block=gold_block,   # empty string omits the section
        )

        # Call GPT-4o with JSON mode
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a precise engineering ticket analysis system. Always respond with valid JSON only."},
                {"role": "user", "content": prompt}
            ],
            temperature=self.temperature,
            max_tokens=2000,
            response_format={"type": "json_object"}  # Enforces valid JSON output
        )

        raw_json = response.choices[0].message.content
        result = AnalysisResult.model_validate_json(raw_json)

        # Cache for 24 hours
        await self.redis.setex(cache_key, 86400, result.model_dump_json())

        return result
```

---

## Analysis Orchestration

The analysis service coordinates the full pipeline:

```python
# app/modules/analysis/service.py

class AnalysisService:
    async def analyze(self, ticket_id: str, db, retrieval_svc, llm_svc, redis) -> AnalysisResult:

        # Load ticket from DB
        ticket_orm = await self._get_ticket(ticket_id, db)
        ticket_input = TicketInput.model_validate(ticket_orm.__dict__)

        # Two-stage retrieval with auto-steering
        search_req = SearchRequest(
            query=SearchQuery(
                description=ticket_input.description,
                functional_group=ticket_input.functional_group,
                analysis_notes=ticket_input.analysis_notes,
            ),
            steering=SteeringConfig(auto_select=True),
            options=SearchOptions(top_k=10, score_threshold=0.65),
        )
        search_resp = await retrieval_svc.search(search_req)

        # Fetch gold examples for this functional group (learning loop — Layer 3 memory)
        gold_shots = await gold_examples_svc.fetch_for_group(
            functional_group=ticket_input.functional_group,
            limit=3
        )

        # LLM analysis with retrieved context + gold few-shots
        context_tickets = self._search_results_to_enriched(search_resp.results)
        result = await llm_svc.analyze_ticket(ticket_input, context_tickets, gold_shots)

        # Persist to analysis_results table
        await self._save_analysis(result, ticket_orm.id, search_resp.steering_applied, db)

        return result
```

---

## Detection Logic

### Duplicate Detection

```python
# Triggered when:
# - similarity_score >= 0.92 AND same functional_group AND LLM confirms semantic overlap

if analysis.duplicate_detection.is_duplicate:
    await db.execute(
        update(TicketORM)
        .where(TicketORM.ticket_id == query_ticket.ticket_id)
        .values(
            status="Duplicate",
            duplicate_of=analysis.duplicate_detection.duplicate_of
        )
    )
```

### Root Cause → Team Routing

```python
ROOT_CAUSE_TEAMS = {
    "firmware_bug":        "firmware-team",
    "hardware_limitation": "hardware-team",
    "configuration_error": "integration-team",
    "known_platform_issue":"platform-team",
    "user_error":          "documentation-team",
    "external_dependency": "partnerships-team",
    "unknown":             "triage-team",
}
```

---

## Example Output

**New Ticket:** "BT audio drops when receiving call on Android 14"

```json
{
  "ticket_id": "TKT-2024-01247",
  "analysis_timestamp": "2024-03-15T10:32:44Z",
  "duplicate_detection": {
    "is_duplicate": false,
    "duplicate_of": null,
    "duplicate_score": 0.84,
    "reason": "TKT-2024-00847 is similar (HFP audio routing) but on Android 12. Different OS, likely same root cause."
  },
  "rejection_prediction": {
    "rejection_probability": 15,
    "rejection_reason": "Similar issues on Android 12 were fixed in v3.2.2. Android 14 regression suspected.",
    "similar_rejected_tickets": [],
    "recommendation": "Accept"
  },
  "root_cause": {
    "category": "firmware_bug",
    "confidence": 87,
    "explanation": "HFP A2DP sequencing regression on Android 14. A2DP not suspended before HFP connect.",
    "evidence_tickets": ["TKT-2024-00847", "TKT-2024-00612"]
  },
  "recommended_action": {
    "action": "Port HFP fix from v3.2.2 to Android 14 variant",
    "priority": "High",
    "assignee_team": "firmware-team",
    "estimated_effort": "3-5 days",
    "steps": [
      "Reproduce on Pixel 8 (Android 14)",
      "Capture Bluetooth HCI logs during audio drop",
      "Compare A2DP/HFP timing with Android 12 fix (TKT-2024-00847)",
      "Apply v3.2.2 fix to Android 14 code path",
      "Regression test on 5+ Android 14 devices"
    ]
  },
  "resolution_reference": {
    "found": true,
    "reference_ticket": "TKT-2024-00847",
    "resolution_summary": "Updated HFP stack to sequence A2DP suspend before HFP connect. Deployed in v3.2.2.",
    "applicable": true
  },
  "confidence_score": 87,
  "summary": "Android 14 regression of TKT-2024-00847. Port the v3.2.2 HFP fix to Android 14 — high confidence."
}
```

> **Next:** [06-database-design.md](06-database-design.md)
