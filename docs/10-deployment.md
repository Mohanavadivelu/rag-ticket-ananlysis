# Section 10: Deployment Design

## 🚀 Overview

The system is containerized with **Docker Compose** for development and **Kubernetes** for production. All services are stateless (except the databases) enabling horizontal scaling.

---

## 🐳 Docker Compose (Development)

```yaml
# docker-compose.yml
version: "3.9"

services:
  # ─── Reverse Proxy ────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./infrastructure/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./infrastructure/nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api_gateway
    restart: unless-stopped

  # ─── API Gateway ──────────────────────────────────────
  api_gateway:
    build:
      context: ./services/api_gateway
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - EMBEDDING_SERVICE_URL=http://embedding_service:8005
      - RETRIEVAL_SERVICE_URL=http://retrieval_service:8002
      - LLM_SERVICE_URL=http://llm_service:8007
      - JWT_SECRET=${JWT_SECRET}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      qdrant:
        condition: service_started
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ─── Embedding Service ────────────────────────────────
  embedding_service:
    build:
      context: ./services/embedding_service
      dockerfile: Dockerfile
    ports:
      - "8005:8005"
    environment:
      - EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
    volumes:
      - embedding_models:/app/models   # Cache downloaded models
    depends_on:
      - redis
      - qdrant
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 8G
        reservations:
          memory: 4G

  # ─── Steering Service ─────────────────────────────────
  steering_service:
    build:
      context: ./services/steering_service
      dockerfile: Dockerfile
    ports:
      - "8006:8006"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
      - EMBEDDING_SERVICE_URL=http://embedding_service:8005
    depends_on:
      - redis
      - qdrant
      - embedding_service
    restart: unless-stopped

  # ─── Retrieval Service ────────────────────────────────
  retrieval_service:
    build:
      context: ./services/retrieval_service
      dockerfile: Dockerfile
    ports:
      - "8002:8002"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
      - EMBEDDING_SERVICE_URL=http://embedding_service:8005
      - STEERING_SERVICE_URL=http://steering_service:8006
    depends_on:
      - redis
      - qdrant
      - embedding_service
      - steering_service
    restart: unless-stopped

  # ─── LLM Service ──────────────────────────────────────
  llm_service:
    build:
      context: ./services/llm_service
      dockerfile: Dockerfile
    ports:
      - "8007:8007"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - LLM_MODEL=gpt-4o
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/tickets
    depends_on:
      - redis
    restart: unless-stopped

  # ─── Vector Database ──────────────────────────────────
  qdrant:
    image: qdrant/qdrant:v1.7.4
    ports:
      - "6333:6333"   # REST API
      - "6334:6334"   # gRPC
    volumes:
      - qdrant_storage:/qdrant/storage
      - ./infrastructure/qdrant/config.yaml:/qdrant/config/production.yaml:ro
    environment:
      - QDRANT__SERVICE__HTTP_PORT=6333
      - QDRANT__SERVICE__GRPC_PORT=6334
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ─── Relational Database ──────────────────────────────
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=tickets
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infrastructure/postgres/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=512MB"
      - "-c"
      - "effective_cache_size=2GB"

  # ─── Cache & Message Queue ────────────────────────────
  redis:
    image: redis:7.2-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./infrastructure/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  qdrant_storage:
  redis_data:
  embedding_models:
```

---

## 📦 Dockerfile Pattern (API Gateway Example)

```dockerfile
# services/api_gateway/Dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ─── Production Stage ────────────────────────────────────
FROM python:3.11-slim AS production

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY . .

# Copy shared library
COPY ../../shared /app/shared

# Non-root user for security
RUN adduser --disabled-password --gecos "" appuser
USER appuser

ENV PATH=/root/.local/bin:$PATH
ENV PYTHONPATH=/app

EXPOSE 8000

# Async production server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "4", "--loop", "uvloop", "--http", "httptools"]
```

---

## ⚙️ Nginx Configuration

```nginx
# infrastructure/nginx/nginx.conf
worker_processes auto;
events { worker_connections 4096; }

http {
    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=search:10m rate=30r/m;

    upstream api_gateway {
        server api_gateway:8000;
        keepalive 32;
    }

    server {
        listen 80;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.ticket-intelligence.internal;

        ssl_certificate     /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        ssl_protocols       TLSv1.2 TLSv1.3;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        location /api/v1/tickets/search {
            limit_req zone=search burst=10 nodelay;
            proxy_pass http://api_gateway;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 30s;
        }

        location /api/v1/analysis {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://api_gateway;
            proxy_read_timeout 60s;  # LLM can take up to 60s
        }

        location / {
            limit_req zone=api burst=50 nodelay;
            proxy_pass http://api_gateway;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

---

## 📊 Resource Requirements Summary

| Service | CPU | RAM | Notes |
|---------|-----|-----|-------|
| Nginx | 0.5 cores | 256 MB | |
| API Gateway (×2) | 1 core | 1 GB | Stateless |
| Embedding (×3) | 4 cores | 8 GB | GPU optional |
| Steering | 1 core | 2 GB | |
| Retrieval (×2) | 1 core | 1 GB | |
| LLM (×2) | 0.5 cores | 1 GB | External API calls |
| Qdrant | 4 cores | 16 GB | Main bottleneck |
| PostgreSQL | 4 cores | 8 GB | With PgBouncer |
| Redis | 1 core | 4 GB | |
| **Total** | **~20 cores** | **~41 GB** | |

---

> **Next:** [Section 11 — Performance & Scalability](11-performance.md)
