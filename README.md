# 🎫 Ticket Intelligence System
### Production-Grade RAG + Steering Vectors + Qdrant

> **AI-powered engineering ticket analysis using Retrieval-Augmented Generation, Steering Vectors, and Qdrant vector database.**

---

## 🏗️ System Architecture Flow

```
New Ticket Input
      │
      ▼
┌─────────────────┐
│  Ingestion API  │  ← REST API Gateway (FastAPI + Nginx)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Embedding Svc   │  ← BGE-large / OpenAI text-embedding-3-large
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Steering Vector │  ← Apply domain-specific directional vectors
│    Service      │    (Rejected / FunctionalGroup / RootCause)
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────────┐
│  Qdrant Vector  │ ←──→ │  PostgreSQL Metadata  │
│    Database     │      │      Database         │
└────────┬────────┘      └──────────────────────┘
         │
         ▼
┌─────────────────┐
│  Retrieval Svc  │  ← Top-K similar tickets + metadata
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM Reasoning  │  ← GPT-4o / Claude / Mistral
│    Service      │    Context + Steering → Final Analysis
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Final Response │  ← Duplicate detection, Root cause, Recommendations
└─────────────────┘
```

---

## 📋 Table of Contents

| Section | Topic | Status |
|---------|-------|--------|
| [01](docs/01-system-architecture.md) | Full System Architecture | ✅ |
| [02](docs/02-qdrant-design.md) | Vector Database Design (Qdrant) | ✅ |
| [03](docs/03-embedding-design.md) | Embedding Generation Design | ✅ |
| [04](docs/04-steering-vectors.md) | Steering Vector Design | ✅ |
| [05](docs/05-retrieval-pipeline.md) | Retrieval Pipeline Design | ✅ |
| [06](docs/06-llm-reasoning.md) | LLM Reasoning Design | ✅ |
| [07](docs/07-database-design.md) | Database Design (PostgreSQL) | ✅ |
| [08](docs/08-api-design.md) | API Design | ✅ |
| [09](docs/09-folder-structure.md) | Folder Structure | ✅ |
| [10](docs/10-deployment.md) | Deployment Design | ✅ |
| [11](docs/11-performance.md) | Performance & Scalability | ✅ |
| [12](docs/12-integration.md) | Integration Design | ✅ |
| [13](docs/13-security.md) | Security Design | ✅ |
| [14](docs/14-end-to-end-flow.md) | Complete End-to-End Flow | ✅ |
| [15](docs/15-advanced-features.md) | Advanced Features | ✅ |
| [16](docs/16-sample-code.md) | Sample Code Structure | ✅ |

---

## 🚀 Quick Start

```bash
# Clone and setup
git clone https://github.com/your-org/ticket-intelligence-system.git
cd ticket-intelligence-system

# Start all services
docker-compose up -d

# Initialize database
docker-compose exec api python scripts/init_db.py

# Ingest sample tickets
docker-compose exec api python scripts/seed_tickets.py

# Access API docs
open http://localhost:8000/docs
```

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| API Gateway | FastAPI + Nginx |
| Vector DB | Qdrant |
| Metadata DB | PostgreSQL |
| Embedding | BGE-large-en-v1.5 / OpenAI |
| LLM | GPT-4o / Claude 3.5 |
| Cache | Redis |
| Containerization | Docker + Docker Compose |
| Message Queue | Redis Streams |

---

## 📊 Ticket Domain Coverage

- 🔵 **Bluetooth** — pairing, audio, connectivity issues
- 📶 **Wi-Fi** — connectivity, authentication, performance
- 📽️ **Projection** — screen mirroring, resolution, latency
- 🤖 **Android Auto** — compatibility, audio routing, navigation
- 🍎 **CarPlay** — USB/wireless, app compatibility, audio
- ⚡ **General Embedded** — firmware, power, hardware

---

*Built with ❤️ for enterprise-scale engineering ticket intelligence.*
