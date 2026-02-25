# Section 13: Security Design

## 🔐 Overview

Security is implemented at every layer: network, authentication, authorization, data protection, and audit logging.

---

## 🔑 Authentication

### JWT Bearer Token

```python
# shared/auth/jwt_handler.py
from jose import jwt, JWTError
from datetime import datetime, timedelta

SECRET_KEY = settings.JWT_SECRET
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = timedelta(hours=8)

def create_access_token(user_id: str, role: str, team: str) -> str:
    payload = {
        "sub": user_id,
        "role": role,
        "team": team,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + ACCESS_TOKEN_EXPIRE,
        "jti": str(uuid.uuid4())    # Unique token ID (for revocation)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        # Check token not revoked (check Redis blacklist)
        if is_token_revoked(payload["jti"]):
            raise HTTPException(status_code=401, detail="Token has been revoked")
        return payload
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### API Key Authentication (Service-to-Service)

```python
# shared/auth/api_key.py
import secrets, hashlib

def generate_api_key() -> tuple[str, str]:
    """Generate API key and its hash for storage."""
    raw_key = f"tis_live_sk_{secrets.token_urlsafe(32)}"
    key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
    return raw_key, key_hash   # raw_key shown once to user; hash stored in DB

async def verify_api_key(key: str, db: AsyncSession) -> User:
    """Verify API key against stored hash."""
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    user = await db.execute(
        select(User).where(User.api_key_hash == key_hash, User.is_active == True)
    )
    if not user:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return user.scalar_one()
```

---

## 🛡️ Authorization (RBAC)

```python
# Role definitions
ROLES = {
    "admin": {
        "permissions": ["read", "write", "delete", "admin", "steering:manage"]
    },
    "engineer": {
        "permissions": ["read", "write", "analyze"]
    },
    "viewer": {
        "permissions": ["read"]
    },
    "service": {
        "permissions": ["read", "write", "analyze", "ingest"]  # For service accounts
    }
}

# FastAPI dependency for route-level permission checking
def require_permission(permission: str):
    async def checker(current_user: User = Depends(get_current_user)):
        user_permissions = ROLES.get(current_user.role, {}).get("permissions", [])
        if permission not in user_permissions:
            raise HTTPException(status_code=403, detail=f"Permission denied: {permission}")
        return current_user
    return checker

# Usage in router
@router.delete("/api/v1/tickets/{id}")
async def delete_ticket(
    id: str,
    user: User = Depends(require_permission("delete"))
):
    ...
```

---

## 🌐 API Security

### Rate Limiting (Nginx)

```nginx
# Per-IP sliding window rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=analysis:10m rate=10r/m;
```

### Input Validation (Pydantic)

```python
class TicketInput(BaseModel):
    ticket_id: str = Field(..., pattern=r"^[A-Z]+-\d{4}-\d+$", max_length=50)
    description: str = Field(..., min_length=10, max_length=10000)
    functional_group: Literal["Bluetooth", "WiFi", "CarPlay", "AndroidAuto", "Projection", "General"]
    status: Literal["New", "Investigating", "Fixed", "Rejected", "Duplicate", "Closed"]
    severity: Literal["Critical", "High", "Medium", "Low"] = "Medium"

    @validator("description")
    def sanitize_description(cls, v):
        # Strip HTML tags, normalize whitespace
        import bleach
        return bleach.clean(v, tags=[], strip=True).strip()
```

---

## 🔒 Data Protection

| Data Type | Protection | Notes |
|-----------|-----------|-------|
| Ticket text | Encrypted at rest | PostgreSQL AES-256 encryption |
| API keys | SHA-256 hashed | Never stored in plaintext |
| JWT secrets | AWS Secrets Manager / Vault | Rotated every 90 days |
| Embeddings | Encrypted at rest | Qdrant disk encryption |
| OpenAI API key | Environment variable | Never logged |
| PII in tickets | Data masking | Reporter emails masked in logs |

### Secrets Management

```bash
# Use environment variables, never hardcode secrets
# In production: AWS Secrets Manager or HashiCorp Vault
JWT_SECRET=$(aws secretsmanager get-secret-value --secret-id tis/jwt-secret --query SecretString --output text)
OPENAI_API_KEY=$(aws secretsmanager get-secret-value --secret-id tis/openai-key --query SecretString --output text)
```

---

## 📋 Audit Logging

Every data-modifying action is logged:

```python
# Automatic audit logging via SQLAlchemy event listener
@event.listens_for(Ticket, "after_insert")
async def log_ticket_created(mapper, connection, target):
    await audit_log(
        entity_type="ticket",
        entity_id=target.id,
        action="created",
        actor_id=current_user_id(),
        new_value=target.to_dict()
    )

@event.listens_for(Ticket, "after_update")
async def log_ticket_updated(mapper, connection, target):
    await audit_log(
        entity_type="ticket",
        entity_id=target.id,
        action="updated",
        old_value=target._previous_state,
        new_value=target.to_dict()
    )
```

---

## 🔐 Security Checklist

- [x] JWT with short expiry (8h) + refresh tokens
- [x] API key hashing (SHA-256, never stored plaintext)
- [x] RBAC per endpoint
- [x] Input validation with Pydantic (regex, length limits)
- [x] SQL injection prevention (SQLAlchemy ORM, no raw SQL)
- [x] TLS 1.2+ enforced at Nginx
- [x] Security headers (HSTS, X-Frame-Options, CSP)
- [x] Rate limiting at Nginx
- [x] Secrets via environment variables / Vault
- [x] Full audit trail in PostgreSQL
- [x] Non-root containers
- [x] Docker image scanning (Trivy/Snyk)

---

> **Next:** [Section 14 — End-to-End Flow](14-end-to-end-flow.md)
