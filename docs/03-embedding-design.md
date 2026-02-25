# Section 3: Embedding Generation Design

## 🧠 Overview

Embeddings are the **semantic fingerprints** of tickets. The quality of embeddings directly determines the accuracy of similarity search, duplicate detection, and LLM reasoning. This section covers how to optimally combine ticket fields and choose the right embedding model.

---

## 📝 Ticket Field Combination Strategy

### Why Field Combination Matters
A raw ticket has multiple fields. Simply concatenating them without structure causes the embedding model to weight fields equally — but **description** and **analysis** are far more semantically rich than `status` or `ticket_id`.

### Optimal Embedding Text Format

```python
def format_ticket_for_embedding(ticket: dict) -> str:
    """
    Structured format that maximizes semantic richness.
    Field order matters — models weight earlier tokens more.
    """
    parts = []

    # Primary semantic content (highest weight)
    parts.append(f"Functional Group: {ticket.get('functional_group', 'Unknown')}")
    parts.append(f"Description: {ticket.get('description', '')}")

    # Secondary semantic content
    if ticket.get('analysis_notes'):
        parts.append(f"Analysis: {ticket['analysis_notes']}")

    # Resolution provides root cause context
    if ticket.get('resolution'):
        parts.append(f"Resolution: {ticket['resolution']}")

    # Status signals the ticket outcome
    parts.append(f"Status: {ticket.get('status', 'New')}")

    # Tags provide domain keywords
    if ticket.get('tags'):
        parts.append(f"Tags: {', '.join(ticket['tags'])}")

    return "\n".join(parts)
```

### Example Output

```
Functional Group: Bluetooth
Description: Device disconnects from Bluetooth when making a phone call. The audio
             routing switches back to phone speaker instead of car speakers. This
             happens consistently when Call Manager app is active.
Analysis: Issue traced to HFP profile negotiation failure during active call state.
          The A2DP stream is not properly suspended before HFP activation. Observed
          in firmware versions < 3.2.1. Platform: Android 13, Pixel 7.
Resolution: Updated HFP stack to properly sequence A2DP suspend before HFP connect.
            Fix deployed in firmware v3.2.2. Regression tested on 15 devices.
Status: Fixed
Tags: bluetooth, hfp, a2dp, audio-routing, android, call-manager
```

### Why This Format Works

| Field | Position | Reason |
|-------|----------|--------|
| Functional Group | First | Anchors domain context immediately |
| Description | Second | Core semantic content, most important |
| Analysis | Third | Technical context, root cause hints |
| Resolution | Fourth | Fix pattern, helps match similar resolved tickets |
| Status | Fifth | Outcome signal for steering |
| Tags | Last | Keyword reinforcement |

---

## 🤖 Embedding Model Recommendations

### Option 1: BGE-large-en-v1.5 ⭐ (Recommended for Local)

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-large-en-v1.5")

# BGE requires a query prefix for retrieval tasks
def embed_query(text: str) -> list[float]:
    return model.encode(
        f"Represent this sentence for searching relevant passages: {text}",
        normalize_embeddings=True
    ).tolist()

def embed_document(text: str) -> list[float]:
    return model.encode(
        text,
        normalize_embeddings=True
    ).tolist()
```

| Property | Value |
|----------|-------|
| Dimension | 1024 |
| Max tokens | 512 |
| MTEB Score | 64.23 |
| Cost | Free (local) |
| Latency | ~20ms per ticket (GPU) |
| License | MIT |

### Option 2: OpenAI text-embedding-3-large ⭐ (Recommended for Cloud)

```python
from openai import OpenAI

client = OpenAI(api_key=settings.OPENAI_API_KEY)

def embed_ticket(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-large",
        input=text,
        dimensions=1024  # Can reduce from 3072 to 1024 with negligible quality loss
    )
    return response.data[0].embedding
```

| Property | Value |
|----------|-------|
| Dimension | 3072 (reducible to 1024) |
| Max tokens | 8191 |
| MTEB Score | 64.6 |
| Cost | $0.13 / 1M tokens |
| Latency | ~100ms per batch API call |

### Option 3: Instructor-XL (Task-Aware)

```python
from InstructorEmbedding import INSTRUCTOR

model = INSTRUCTOR("hkunlp/instructor-xl")

def embed_ticket(text: str) -> list[float]:
    return model.encode([[
        "Represent the engineering ticket for similarity retrieval:", 
        text
    ]])[0].tolist()
```

| Property | Value |
|----------|-------|
| Dimension | 768 |
| Unique Feature | Task-specific instruction prefix |
| Best For | Domain adaptation without fine-tuning |
| Cost | Free (local), large model (1.5B params) |

### Model Comparison Matrix

| Model | Dim | Quality | Speed | Cost | Recommended For |
|-------|-----|---------|-------|------|-----------------|
| BGE-large-en-v1.5 | 1024 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Free | Production local |
| OpenAI text-3-large | 3072 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | $$ | Cloud, high quality |
| Instructor-XL | 768 | ⭐⭐⭐⭐ | ⭐⭐⭐ | Free | Task-specific needs |
| OpenAI text-3-small | 1536 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | $ | Cost-sensitive cloud |
| BGE-base-en-v1.5 | 768 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Free | High-throughput local |

---

## 🏗️ Embedding Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   EMBEDDING PIPELINE                         │
│                                                              │
│  Ticket Input                                                │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────┐                                     │
│  │  Text Preprocessor  │  ← Clean, normalize, truncate       │
│  │  - Remove HTML tags │                                     │
│  │  - Normalize unicode│                                     │
│  │  - Truncate to 512t │                                     │
│  └──────────┬──────────┘                                     │
│             │                                                │
│      ┌──────▼──────┐                                        │
│      │ Cache Check │  ← Redis: hash(text) → embedding        │
│      │  (Redis)    │                                        │
│      └──────┬──────┘                                        │
│       Hit ──┴── Miss                                        │
│        │          │                                         │
│        │    ┌─────▼────────────────┐                        │
│        │    │  Embedding Model     │                        │
│        │    │  BGE-large / OpenAI  │                        │
│        │    │  Batch: 32 tickets   │                        │
│        │    └─────┬────────────────┘                        │
│        │          │                                         │
│        │    ┌─────▼────────────────┐                        │
│        │    │  Normalize (L2)      │                        │
│        │    │  Validate dimension  │                        │
│        │    └─────┬────────────────┘                        │
│        │          │                                         │
│        │    ┌─────▼────────────────┐                        │
│        │    │  Cache Write         │                        │
│        │    │  TTL: 7 days         │                        │
│        │    └─────┬────────────────┘                        │
│        │          │                                         │
│        └────┬─────┘                                         │
│             │                                               │
│      ┌──────▼──────────────────────┐                        │
│      │  Qdrant Upsert              │                        │
│      │  + PostgreSQL Reference     │                        │
│      └─────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ Batch Processing

For bulk ingestion, batch embedding dramatically reduces latency:

```python
import asyncio
from typing import List

class EmbeddingPipeline:
    def __init__(self, model_name: str = "BAAI/bge-large-en-v1.5", batch_size: int = 32):
        self.model = SentenceTransformer(model_name)
        self.batch_size = batch_size

    async def embed_batch(self, tickets: List[dict]) -> List[List[float]]:
        """Embed a batch of tickets efficiently."""
        texts = [format_ticket_for_embedding(t) for t in tickets]

        # Process in mini-batches
        embeddings = []
        for i in range(0, len(texts), self.batch_size):
            batch = texts[i:i + self.batch_size]
            batch_embeddings = self.model.encode(
                batch,
                batch_size=self.batch_size,
                normalize_embeddings=True,
                show_progress_bar=False,
                convert_to_numpy=True
            )
            embeddings.extend(batch_embeddings.tolist())

        return embeddings

    def truncate_text(self, text: str, max_tokens: int = 512) -> str:
        """Truncate text to max token limit with smart boundary detection."""
        # Prioritize Description + Analysis (most semantically rich)
        # Truncate Resolution and Tags if needed
        words = text.split()
        if len(words) > max_tokens:
            return " ".join(words[:max_tokens]) + "..."
        return text
```

---

## 🔄 Re-Embedding Strategy

When embedding model is upgraded:

```
1. Create new Qdrant collection: tickets_v2
2. Batch re-embed all tickets with new model
3. Run A/B test: compare retrieval quality v1 vs v2
4. Atomic swap: rename tickets → tickets_v1, tickets_v2 → tickets
5. Keep v1 collection for 30 days as rollback
6. Update embedding_version in PostgreSQL
```

---

> **Next:** [Section 4 — Steering Vector Design](04-steering-vectors.md)
