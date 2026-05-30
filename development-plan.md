# Open Banking Platform -- Phased Development Plan

> Project: 221-open-banking-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ | Heavy on regulatory business logic, ML/LLM integration for enrichment and fraud detection, and async I/O for bank API aggregation. Python's ecosystem (Pydantic, SQLAlchemy, httpx) is the strongest fit for all three. |
| API framework | FastAPI (0.115+) | Native async, automatic OpenAPI 3.1 spec generation (required by open banking standards), Pydantic v2 integration for request/response validation, dependency injection for auth/tenant isolation. |
| Database | PostgreSQL 16+ | Multi-tenant row-level security, JSONB for standard-specific field variation (Hybrid model from data-model-suggestion-3), declarative partitioning for audit logs, GIN indexes for JSONB queries. Required by the regulatory audit trail mandate. |
| Migrations | Alembic | Standard SQLAlchemy migration tool; supports auto-generation from model changes, branching for parallel development. |
| ORM | SQLAlchemy 2.0+ (async) | Mature PostgreSQL support including JSONB, arrays, and RLS. Async engine for FastAPI integration. |
| Task queue | Celery + Redis | Async workloads: bank API polling, webhook delivery with retries, transaction enrichment batch jobs, consent expiry monitoring. Redis doubles as rate-limit counter store. |
| Cache / Rate limiting | Redis 7+ | Token-bucket rate limiting per tenant/client, OAuth token caching, institution health status caching. |
| HTTP client | httpx (async) | Async HTTP for bank API calls, connection pooling, timeout management, mTLS support for FAPI. |
| Authentication | Custom OAuth 2.0 / FAPI 2.0 server | Open banking requires FAPI 2.0 security profile (private_key_jwt, PAR, sender-constrained tokens). No off-the-shelf library fully implements FAPI 2.0; build on authlib + python-jose. |
| JWT | python-jose + jwcrypto | JWT creation/validation with PS256 algorithm support (FAPI 2.0 requirement). |
| Frontend | Next.js 15 (React) | Admin dashboard for consent audit trails, transaction monitoring, institution health. Server components for initial load performance; client components for real-time updates. |
| Frontend UI | shadcn/ui + Tailwind CSS | Consistent, accessible components; Tailwind for rapid styling without custom CSS overhead. |
| Containerisation | Docker + docker-compose | Self-hosted deployment target; compose for local development (API, PostgreSQL, Redis, worker). |
| Testing | pytest + pytest-asyncio | Async test support, fixtures for database setup/teardown, parametrize for multi-standard test cases. |
| API testing | httpx (TestClient) | FastAPI's recommended test client; async-aware. |
| Frontend testing | Vitest + Playwright | Unit tests (Vitest) and E2E browser tests (Playwright) for the admin dashboard. |
| Code quality | ruff (linter + formatter) + mypy | ruff replaces black + isort + flake8 in a single tool. mypy for static type checking -- critical for a compliance platform. |
| Package manager | uv | Fast dependency resolution, lockfile support, replaces pip + pip-tools. |
| API documentation | Auto-generated OpenAPI 3.1 | FastAPI generates the spec; Redoc/Swagger UI served at /docs. Berlin Group NextGenPSD2 uses OpenAPI as its contract format. |
| Validation | Pydantic v2 | Request/response validation, config parsing, data model serialisation. JSON Schema export for JSONB column validation. |
| Logging | structlog | Structured JSON logging for compliance audit trails; correlation IDs across request chains. |
| Key libraries | authlib (OAuth), cryptography (mTLS/JWE), schwifty (IBAN validation), iso4217 (currency), pycountry (ISO 3166) | Domain-specific validation aligned with ISO standards referenced in standards.md. |

### Project Structure

```
open-banking-platform/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   └── versions/
├── src/
│   └── openbanking/
│       ├── __init__.py
│       ├── main.py                     # FastAPI app factory
│       ├── config.py                   # Pydantic Settings
│       ├── database.py                 # SQLAlchemy async engine + session
│       ├── models/                     # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── user.py
│       │   ├── oauth.py
│       │   ├── consent.py
│       │   ├── institution.py
│       │   ├── account.py
│       │   ├── transaction.py
│       │   ├── payment.py
│       │   ├── webhook.py
│       │   └── audit.py
│       ├── schemas/                    # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── consent.py
│       │   ├── account.py
│       │   ├── transaction.py
│       │   ├── payment.py
│       │   └── webhook.py
│       ├── api/                        # FastAPI routers
│       │   ├── __init__.py
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── consents.py
│       │   │   ├── accounts.py
│       │   │   ├── transactions.py
│       │   │   ├── payments.py
│       │   │   ├── webhooks.py
│       │   │   ├── admin.py
│       │   │   └── health.py
│       │   └── deps.py                 # Dependency injection (auth, tenant, db)
│       ├── auth/                       # OAuth 2.0 / FAPI 2.0 implementation
│       │   ├── __init__.py
│       │   ├── fapi.py
│       │   ├── sca.py
│       │   ├── jwt.py
│       │   └── middleware.py
│       ├── services/                   # Business logic
│       │   ├── __init__.py
│       │   ├── consent_service.py
│       │   ├── account_service.py
│       │   ├── transaction_service.py
│       │   ├── payment_service.py
│       │   ├── webhook_service.py
│       │   └── audit_service.py
│       ├── connectors/                 # Bank API connectors
│       │   ├── __init__.py
│       │   ├── base.py                 # Abstract connector interface
│       │   ├── nextgenpsd2.py          # Berlin Group connector
│       │   ├── obie.py                 # UK Open Banking connector
│       │   ├── fdx.py                  # FDX connector
│       │   └── registry.py            # Connector registry/factory
│       ├── enrichment/                 # Transaction enrichment (ML)
│       │   ├── __init__.py
│       │   ├── categoriser.py
│       │   └── merchant.py
│       ├── workers/                    # Celery tasks
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── sync_accounts.py
│       │   ├── deliver_webhooks.py
│       │   ├── consent_monitor.py
│       │   └── health_checker.py
│       └── utils/
│           ├── __init__.py
│           ├── iban.py
│           ├── iso20022.py
│           └── rate_limiter.py
├── tests/
│   ├── conftest.py                     # Shared fixtures
│   ├── factories.py                    # Test data factories
│   ├── unit/
│   │   ├── test_consent_service.py
│   │   ├── test_payment_service.py
│   │   ├── test_iban.py
│   │   └── ...
│   ├── integration/
│   │   ├── test_consent_api.py
│   │   ├── test_account_api.py
│   │   └── ...
│   ├── e2e/
│   │   └── test_consent_flow.py
│   └── fixtures/
│       ├── nextgenpsd2/                # Sample bank API responses
│       ├── obie/
│       └── fdx/
├── dashboard/                          # Next.js admin dashboard
│   ├── package.json
│   ├── src/
│   │   └── app/
│   │       ├── layout.tsx
│   │       ├── page.tsx
│   │       ├── consents/
│   │       ├── transactions/
│   │       ├── payments/
│   │       ├── institutions/
│   │       └── components/
│   └── tests/
└── docs/
    └── openapi/                        # Generated OpenAPI specs
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the project skeleton, database connectivity, configuration management, multi-tenant data model, and CI tooling. After this phase, the application boots, connects to PostgreSQL, runs migrations, and serves a health endpoint. Every subsequent phase builds on this foundation without restructuring.

### Tasks

#### 1.1 -- Project Initialisation & Tooling

**What**: Create the Python project with uv, configure ruff, mypy, pytest, and Docker.

**Design**:

`pyproject.toml` (key sections):
```toml
[project]
name = "open-banking-platform"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "pydantic>=2.7",
    "pydantic-settings>=2.3",
    "redis>=5.0",
    "httpx>=0.27",
    "structlog>=24.1",
    "python-jose[cryptography]>=3.3",
    "authlib>=1.3",
    "celery[redis]>=5.4",
    "schwifty>=2024.1",
]

[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

`Dockerfile`:
```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
EXPOSE 8000
CMD ["uvicorn", "openbanking.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`docker-compose.yml`:
```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://ob:ob@postgres:5432/openbanking
      REDIS_URL: redis://redis:6379/0
    depends_on: [postgres, redis]
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ob
      POSTGRES_PASSWORD: ob
      POSTGRES_DB: openbanking
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  worker:
    build: .
    command: celery -A openbanking.workers.celery_app worker -l info
    environment:
      DATABASE_URL: postgresql+asyncpg://ob:ob@postgres:5432/openbanking
      REDIS_URL: redis://redis:6379/0
    depends_on: [postgres, redis]
volumes:
  pgdata:
```

**Testing**:
- Unit: `ruff check src/` passes with zero violations
- Unit: `mypy src/` passes with zero errors
- Integration: `docker compose up --build` starts all services; `curl localhost:8000/health` returns 200

#### 1.2 -- Configuration Management

**What**: Centralised configuration via Pydantic Settings with environment variable and `.env` file support.

**Design**:

```python
# src/openbanking/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    # Database
    database_url: str = "postgresql+asyncpg://ob:ob@localhost:5432/openbanking"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # API
    api_prefix: str = "/api/v1"
    api_title: str = "Open Banking Platform"
    api_version: str = "0.1.0"
    cors_origins: list[str] = ["http://localhost:3000"]

    # Auth / FAPI
    jwt_algorithm: str = "PS256"
    jwt_issuer: str = "https://openbanking.local"
    access_token_ttl_seconds: int = 300   # 5 minutes (FAPI recommended)
    refresh_token_ttl_seconds: int = 86400

    # Consent defaults
    default_consent_ttl_days: int = 90
    max_frequency_per_day: int = 4

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    # Logging
    log_level: str = "INFO"
    log_format: str = "json"

settings = Settings()
```

**Testing**:
- Unit: `Settings()` with no env vars -> all defaults populated correctly
- Unit: `Settings()` with `DATABASE_URL` env var override -> database_url matches env value
- Unit: invalid `database_pool_size` (negative int) -> `ValidationError`

#### 1.3 -- Database Engine & Session Management

**What**: Async SQLAlchemy engine, session factory, and base model class with tenant isolation.

**Design**:

```python
# src/openbanking/database.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import MetaData
import uuid
from datetime import datetime

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=datetime.utcnow, onupdate=datetime.utcnow)

class TenantMixin:
    tenant_id: Mapped[uuid.UUID] = mapped_column(index=True)

engine = create_async_engine(settings.database_url, pool_size=settings.database_pool_size)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
```

**Testing**:
- Integration: `get_session()` yields a connected session; `SELECT 1` returns 1
- Integration: session commits and rolls back correctly within a transaction
- Unit: `Base` metadata uses correct naming convention

#### 1.4 -- Core Data Models (Tenant, User, RBAC)

**What**: SQLAlchemy models for tenants, users, roles, and permissions following the Hybrid Relational + JSONB design (data-model-suggestion-3).

**Design**:

```python
# src/openbanking/models/tenant.py
from sqlalchemy import String, Boolean, Enum, CheckConstraint
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
import uuid
from openbanking.database import Base, TimestampMixin

class Tenant(Base, TimestampMixin):
    __tablename__ = "tenants"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255))
    slug: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    tenant_type: Mapped[str] = mapped_column(String(50))  # fintech, bank, aspsp, tpp, enterprise
    jurisdiction: Mapped[str] = mapped_column(String(2))   # ISO 3166-1 alpha-2
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    # config holds: supported_standards, regulatory_status, rate_limits, branding, psd3_ready

    users: Mapped[list["User"]] = relationship(back_populates="tenant")

# src/openbanking/models/user.py
class User(Base, TimestampMixin):
    __tablename__ = "users"
    __table_args__ = (UniqueConstraint("tenant_id", "email"),)

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    email: Mapped[str] = mapped_column(String(255))
    password_hash: Mapped[str | None] = mapped_column(String(255))
    full_name: Mapped[str | None] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    mfa_enabled: Mapped[bool] = mapped_column(Boolean, default=False)
    profile: Mapped[dict] = mapped_column(JSONB, default=dict)
    # profile holds: phone, preferred_language, gdpr metadata, kyc_status
    last_login_at: Mapped[datetime | None]

    tenant: Mapped["Tenant"] = relationship(back_populates="users")

class Role(Base, TimestampMixin):
    __tablename__ = "roles"
    __table_args__ = (UniqueConstraint("tenant_id", "name"),)

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"))
    name: Mapped[str] = mapped_column(String(100))
    permissions: Mapped[dict] = mapped_column(JSONB, default=list)
    # permissions: [{"resource": "accounts", "actions": ["read"]}, ...]
    is_system_role: Mapped[bool] = mapped_column(Boolean, default=False)

class UserRole(Base):
    __tablename__ = "user_roles"
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"), primary_key=True)
    role_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("roles.id", ondelete="CASCADE"), primary_key=True)
    granted_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

**Testing**:
- Integration: create a tenant, create a user under that tenant, verify FK constraint
- Integration: unique constraint on (tenant_id, email) prevents duplicate users
- Integration: tenant config JSONB stores and retrieves nested structures correctly
- Unit: Tenant model default config is empty dict

#### 1.5 -- FastAPI Application & Health Endpoint

**What**: FastAPI application factory with structured logging, CORS, and health check.

**Design**:

```python
# src/openbanking/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from openbanking.config import settings
from openbanking.database import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: verify DB connection
    async with engine.begin() as conn:
        await conn.execute(text("SELECT 1"))
    yield
    # Shutdown
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.api_title,
        version=settings.api_version,
        lifespan=lifespan,
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    app.include_router(health_router, prefix=settings.api_prefix)
    return app

app = create_app()

# src/openbanking/api/v1/health.py
from fastapi import APIRouter, Depends
from sqlalchemy import text

router = APIRouter(tags=["health"])

@router.get("/health")
async def health_check(session: AsyncSession = Depends(get_session)):
    await session.execute(text("SELECT 1"))
    return {"status": "healthy", "version": settings.api_version}
```

**Testing**:
- Integration: `GET /api/v1/health` returns `{"status": "healthy", "version": "0.1.0"}` with status 200
- Integration: health check fails with 503 when database is unreachable
- Unit: `create_app()` returns a FastAPI instance with correct title and version

#### 1.6 -- Alembic Migrations Setup

**What**: Alembic configured for async SQLAlchemy; initial migration creating tenant, user, role, and user_role tables.

**Design**:

`alembic/env.py` configured to:
1. Import all models from `openbanking.models`
2. Use `async` engine with `run_async`
3. Target metadata from `Base.metadata`

Initial migration file creates tables: `tenants`, `users`, `roles`, `user_roles` with all indexes and constraints.

**Testing**:
- Integration: `alembic upgrade head` on a fresh database creates all 4 tables
- Integration: `alembic downgrade base` drops all tables cleanly
- Integration: `alembic check` shows no pending model changes after migration

#### 1.7 -- Structured Logging & Request Correlation

**What**: structlog configured for JSON output with request-scoped correlation IDs.

**Design**:

```python
# Middleware adds X-Request-ID to every request
# structlog processors: add_log_level, TimeStamper, JSONRenderer
# Every log entry includes: correlation_id, tenant_id (when available), timestamp, level, event
```

Correlation ID flow:
1. Middleware reads `X-Request-ID` header or generates UUID
2. Stored in `contextvars.ContextVar`
3. structlog processor injects it into every log entry
4. Response includes `X-Request-ID` header

**Testing**:
- Integration: request without `X-Request-ID` -> response has auto-generated UUID in header
- Integration: request with `X-Request-ID: abc-123` -> response echoes same value
- Unit: log output includes `correlation_id` field

---

## Phase 2: OAuth 2.0 / FAPI 2.0 Security Layer

### Purpose
Implement the OAuth 2.0 authorization server with FAPI 2.0 security profile support. This is the gateway for all API access and must be in place before any business endpoints. After this phase, TPP clients can register, authenticate via `private_key_jwt`, obtain access tokens, and the API enforces token-based authorization on all endpoints.

### Tasks

#### 2.1 -- OAuth Client Registration & Management

**What**: CRUD for OAuth client records with FAPI 2.0 metadata storage.

**Design**:

```python
# src/openbanking/models/oauth.py
class OAuthClient(Base, TimestampMixin):
    __tablename__ = "oauth_clients"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    client_id: Mapped[str] = mapped_column(String(255), unique=True)  # public identifier
    client_secret_hash: Mapped[str | None] = mapped_column(String(255))
    client_name: Mapped[str] = mapped_column(String(255))
    redirect_uris: Mapped[list[str]] = mapped_column(ARRAY(Text))
    allowed_scopes: Mapped[list[str]] = mapped_column(ARRAY(Text))  # accounts, payments, funds-confirmation
    tpp_role: Mapped[str | None] = mapped_column(String(50))  # aisp, pisp, cbpii
    regulatory_id: Mapped[str | None] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    fapi_metadata: Mapped[dict] = mapped_column(JSONB, default=dict)
    # fapi_metadata: fapi_version, token_endpoint_auth_method, jwks_uri,
    #   software_statement, request_object_signing_alg, tls_client_certificate_bound_access_tokens

class OAuthToken(Base):
    __tablename__ = "oauth_tokens"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    client_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("oauth_clients.id"))
    user_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    token_type: Mapped[str] = mapped_column(String(20))  # access, refresh, id
    token_hash: Mapped[str] = mapped_column(String(255), unique=True)
    scopes: Mapped[list[str]] = mapped_column(ARRAY(Text))
    expires_at: Mapped[datetime]
    revoked_at: Mapped[datetime | None]
    token_metadata: Mapped[dict] = mapped_column(JSONB, default=dict)
    # token_metadata: cnf (sender-constrained), dpop_thumbprint
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

API endpoints:
- `POST /api/v1/admin/clients` -- register a new OAuth client (admin only)
- `GET /api/v1/admin/clients` -- list clients for tenant
- `GET /api/v1/admin/clients/{client_id}` -- get client details
- `PATCH /api/v1/admin/clients/{client_id}` -- update client (e.g., rotate secret, update redirect URIs)
- `DELETE /api/v1/admin/clients/{client_id}` -- deactivate client

**Testing**:
- Integration: register client with valid data -> 201 with client_id and client_secret
- Integration: register client with duplicate redirect URI scheme (http://) -> 400
- Integration: list clients returns only clients for the authenticated tenant
- Unit: client_secret is hashed with bcrypt before storage; raw secret never persisted

#### 2.2 -- JWT Token Issuance & Validation

**What**: Token endpoint implementing OAuth 2.0 `client_credentials` grant with FAPI 2.0 `private_key_jwt` client authentication.

**Design**:

```python
# src/openbanking/auth/jwt.py
from jose import jwt, JWK
from datetime import datetime, timedelta
from openbanking.config import settings

class TokenService:
    def create_access_token(
        self,
        client_id: str,
        scopes: list[str],
        tenant_id: uuid.UUID,
        user_id: uuid.UUID | None = None,
    ) -> str:
        """Create a signed JWT access token (PS256 per FAPI 2.0)."""
        now = datetime.utcnow()
        payload = {
            "iss": settings.jwt_issuer,
            "sub": client_id,
            "aud": settings.jwt_issuer,
            "exp": now + timedelta(seconds=settings.access_token_ttl_seconds),
            "iat": now,
            "jti": str(uuid.uuid4()),
            "scope": " ".join(scopes),
            "tenant_id": str(tenant_id),
            "client_id": client_id,
        }
        if user_id:
            payload["psu_id"] = str(user_id)
        return jwt.encode(payload, self.private_key, algorithm="PS256")

    def validate_token(self, token: str) -> dict:
        """Validate and decode a JWT. Raises JWTError on failure."""
        return jwt.decode(
            token,
            self.public_key,
            algorithms=["PS256"],
            issuer=settings.jwt_issuer,
            audience=settings.jwt_issuer,
        )

    def validate_client_assertion(self, assertion: str, client: OAuthClient) -> bool:
        """Validate private_key_jwt client authentication per FAPI 2.0."""
        # Fetch client's JWKS from jwks_uri
        # Verify assertion signature with client's public key
        # Check iss == client_id, sub == client_id, aud == token_endpoint
        # Check exp, iat, jti for replay protection
        ...
```

Token endpoint:
- `POST /oauth/token` with `grant_type=client_credentials`, `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`, `client_assertion=<signed JWT>`, `scope=accounts payments`

**Testing**:
- Integration: valid client_credentials grant -> 200 with access_token, token_type=Bearer, expires_in
- Integration: expired client_assertion -> 401 with `invalid_client` error
- Integration: scope requested exceeds client's allowed_scopes -> 400 with `invalid_scope`
- Unit: access token contains correct claims (iss, sub, scope, tenant_id, exp)
- Unit: token with tampered signature -> validation raises JWTError

#### 2.3 -- Authentication Middleware & Dependency Injection

**What**: FastAPI dependencies that extract and validate the Bearer token, resolve tenant context, and enforce scope-based authorization.

**Design**:

```python
# src/openbanking/api/deps.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

bearer_scheme = HTTPBearer()

async def get_current_client(
    credentials: HTTPAuthorizationCredentials = Security(bearer_scheme),
    session: AsyncSession = Depends(get_session),
) -> AuthContext:
    """Validate Bearer token and return auth context."""
    token_data = token_service.validate_token(credentials.credentials)
    client = await session.get(OAuthClient, token_data["client_id"])
    if not client or not client.is_active:
        raise HTTPException(401, "Client not found or inactive")
    return AuthContext(
        client_id=client.client_id,
        tenant_id=uuid.UUID(token_data["tenant_id"]),
        scopes=token_data["scope"].split(),
        psu_id=token_data.get("psu_id"),
    )

def require_scope(scope: str):
    """Dependency that checks if the current token has a required scope."""
    def checker(auth: AuthContext = Depends(get_current_client)):
        if scope not in auth.scopes:
            raise HTTPException(403, f"Scope '{scope}' required")
        return auth
    return checker

@dataclass
class AuthContext:
    client_id: str
    tenant_id: uuid.UUID
    scopes: list[str]
    psu_id: uuid.UUID | None = None
```

**Testing**:
- Integration: request with valid token -> endpoint returns 200
- Integration: request without Authorization header -> 401
- Integration: request with expired token -> 401
- Integration: request with token missing required scope -> 403
- Unit: AuthContext correctly parses all token claims

#### 2.4 -- SCA Session Model

**What**: Strong Customer Authentication session tracking with factor completion state.

**Design**:

```python
# src/openbanking/models/oauth.py (addition)
class SCASession(Base):
    __tablename__ = "sca_sessions"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    consent_id: Mapped[uuid.UUID | None]  # linked after consent creation
    auth_method: Mapped[str] = mapped_column(String(50))  # redirect, decoupled, embedded
    factor_knowledge: Mapped[bool] = mapped_column(Boolean, default=False)
    factor_possession: Mapped[bool] = mapped_column(Boolean, default=False)
    factor_inherence: Mapped[bool] = mapped_column(Boolean, default=False)
    sca_status: Mapped[str] = mapped_column(String(30), default="started")
    # States: started -> factor1_complete -> authenticated -> (or failed/expired)
    started_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    completed_at: Mapped[datetime | None]
    expires_at: Mapped[datetime]
    ip_address: Mapped[str | None] = mapped_column(String(45))
    user_agent: Mapped[str | None]
```

SCA is considered complete when at least two of the three factor booleans are `True` (per PSD2 RTS on SCA).

**Testing**:
- Unit: SCA session with `factor_knowledge=True, factor_possession=True` -> `is_authenticated` returns True
- Unit: SCA session with only one factor -> `is_authenticated` returns False
- Unit: expired SCA session -> status transitions to `expired`
- Integration: creating SCA session persists all fields correctly

#### 2.5 -- Rate Limiting Middleware

**What**: Token-bucket rate limiting per tenant and per client, using Redis.

**Design**:

```python
# src/openbanking/utils/rate_limiter.py
class RateLimiter:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def check_rate_limit(
        self,
        key: str,          # e.g., "rl:tenant:{tenant_id}:accounts"
        max_requests: int,
        window_seconds: int,
    ) -> tuple[bool, int]:  # (allowed, remaining)
        """Token-bucket rate limiting via Redis INCR + EXPIRE."""
        ...

# Default limits (configurable per tenant via config JSONB):
# - accounts: 100 requests/minute
# - transactions: 200 requests/minute
# - payments: 50 requests/minute
```

Middleware applies rate limits based on:
1. Global defaults
2. Tenant-level overrides (from `tenants.config.rate_limits`)
3. Client-level overrides (if configured)

Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

**Testing**:
- Integration: 100 requests within window -> all succeed; 101st returns 429
- Integration: after window expires -> requests succeed again
- Integration: response includes correct rate limit headers
- Unit: key generation produces correct Redis key format

---

## Phase 3: Consent Management (Core Regulatory Requirement)

### Purpose
Implement PSD2/NextGenPSD2-compliant consent management -- the regulatory foundation for all data access and payment initiation. After this phase, TPPs can create, authorise, query, and revoke consents. The consent lifecycle (received -> valid -> expired/revoked) is fully tracked with immutable audit records.

### Tasks

#### 3.1 -- Consent Data Model & Migrations

**What**: Consent and consent_audit tables following the Hybrid model with `api_standard` discriminator and `standard_data` JSONB.

**Design**:

```python
# src/openbanking/models/consent.py
class Consent(Base, TimestampMixin, TenantMixin):
    __tablename__ = "consents"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    tpp_client_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("oauth_clients.id"), index=True)
    psu_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"), index=True)
    api_standard: Mapped[str] = mapped_column(String(30))  # nextgenpsd2, obie, fdx
    consent_type: Mapped[str] = mapped_column(String(30))
    # consent_type: ais, pis, pis_recurring, funds_confirmation, combined
    status: Mapped[str] = mapped_column(String(30), default="received", index=True)
    # Status lifecycle: received -> valid | rejected
    #   valid -> revokedByPsu | expired | terminatedByTpp
    recurring_indicator: Mapped[bool] = mapped_column(Boolean, default=False)
    valid_until: Mapped[date | None]
    frequency_per_day: Mapped[int] = mapped_column(default=4)
    standard_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    sca_completed: Mapped[bool] = mapped_column(Boolean, default=False)

class ConsentAudit(Base):
    __tablename__ = "consent_audit"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    consent_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("consents.id"), index=True)
    old_status: Mapped[str | None] = mapped_column(String(30))
    new_status: Mapped[str] = mapped_column(String(30))
    changed_by: Mapped[str] = mapped_column(String(50))  # psu, tpp, aspsp, system
    reason: Mapped[str | None]
    event_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    changed_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

Consent status state machine:
```
received --> valid          (after SCA authorisation)
received --> rejected       (PSU or ASPSP rejects)
valid    --> revokedByPsu   (PSU revokes)
valid    --> expired        (valid_until date passed)
valid    --> terminatedByTpp (TPP terminates)
```

**Testing**:
- Integration: migration creates consents and consent_audit tables with all indexes
- Integration: consent status CHECK constraint rejects invalid status values
- Unit: consent with `consent_type="invalid"` -> validation error at Pydantic schema level

#### 3.2 -- Consent Service (Business Logic)

**What**: Service layer implementing consent lifecycle operations with audit trail recording.

**Design**:

```python
# src/openbanking/services/consent_service.py
class ConsentService:
    def __init__(self, session: AsyncSession, audit_service: AuditService):
        self.session = session
        self.audit = audit_service

    async def create_consent(
        self,
        tenant_id: uuid.UUID,
        tpp_client_id: uuid.UUID,
        request: ConsentCreateRequest,
    ) -> Consent:
        """Create a new consent in 'received' status."""
        consent = Consent(
            tenant_id=tenant_id,
            tpp_client_id=tpp_client_id,
            api_standard=request.api_standard,
            consent_type=request.consent_type,
            status="received",
            recurring_indicator=request.recurring_indicator,
            valid_until=request.valid_until,
            frequency_per_day=request.frequency_per_day,
            standard_data=request.standard_data,
        )
        self.session.add(consent)
        await self.session.flush()
        await self._record_audit(consent, None, "received", "tpp")
        return consent

    async def authorise_consent(
        self,
        consent_id: uuid.UUID,
        psu_id: uuid.UUID,
        sca_session_id: uuid.UUID,
    ) -> Consent:
        """Authorise a consent after SCA (transition: received -> valid)."""
        consent = await self._get_consent(consent_id)
        if consent.status != "received":
            raise ConsentStateError(f"Cannot authorise consent in '{consent.status}' status")
        consent.status = "valid"
        consent.psu_id = psu_id
        consent.sca_completed = True
        await self._record_audit(consent, "received", "valid", "psu")
        return consent

    async def revoke_consent(
        self,
        consent_id: uuid.UUID,
        revoked_by: str,  # "psu" or "tpp"
        reason: str | None = None,
    ) -> Consent:
        """Revoke an active consent."""
        consent = await self._get_consent(consent_id)
        if consent.status != "valid":
            raise ConsentStateError(f"Cannot revoke consent in '{consent.status}' status")
        new_status = "revokedByPsu" if revoked_by == "psu" else "terminatedByTpp"
        consent.status = new_status
        await self._record_audit(consent, "valid", new_status, revoked_by, reason)
        return consent

    async def get_consent(self, consent_id: uuid.UUID, tenant_id: uuid.UUID) -> Consent:
        """Get consent by ID, scoped to tenant."""
        ...

    async def list_consents(
        self,
        tenant_id: uuid.UUID,
        status: str | None = None,
        psu_id: uuid.UUID | None = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Consent]:
        """List consents with optional filtering."""
        ...

    async def _record_audit(
        self, consent: Consent, old_status: str | None, new_status: str, changed_by: str, reason: str | None = None
    ) -> None:
        audit = ConsentAudit(
            consent_id=consent.id,
            old_status=old_status,
            new_status=new_status,
            changed_by=changed_by,
            reason=reason,
        )
        self.session.add(audit)
```

**Testing**:
- Unit: `create_consent` -> consent in "received" status, audit record created
- Unit: `authorise_consent` on "received" consent -> status becomes "valid"
- Unit: `authorise_consent` on "valid" consent -> raises `ConsentStateError`
- Unit: `revoke_consent` by PSU -> status becomes "revokedByPsu"
- Unit: `revoke_consent` by TPP -> status becomes "terminatedByTpp"
- Unit: `revoke_consent` on "expired" consent -> raises `ConsentStateError`
- Integration: full lifecycle: create -> authorise -> revoke; 3 audit records exist
- Integration: list consents filters by status correctly

#### 3.3 -- Consent API Endpoints (NextGenPSD2-Aligned)

**What**: REST endpoints for consent CRUD, aligned with Berlin Group NextGenPSD2 consent API structure.

**Design**:

Endpoints:
- `POST /api/v1/consents` -- create consent (scope: `accounts` or `payments`)
- `GET /api/v1/consents/{consentId}` -- get consent status and details
- `GET /api/v1/consents/{consentId}/status` -- get consent status only
- `DELETE /api/v1/consents/{consentId}` -- revoke/terminate consent
- `GET /api/v1/consents` -- list consents (admin, with filters)
- `PUT /api/v1/consents/{consentId}/authorisations` -- start SCA authorisation
- `GET /api/v1/consents/{consentId}/authorisations/{authId}` -- get SCA status

Request/response schemas (Pydantic):

```python
# src/openbanking/schemas/consent.py
class ConsentCreateRequest(BaseModel):
    consent_type: Literal["ais", "pis", "funds_confirmation", "combined"]
    api_standard: Literal["nextgenpsd2", "obie", "fdx"] = "nextgenpsd2"
    recurring_indicator: bool = False
    valid_until: date | None = None
    frequency_per_day: int = Field(default=4, ge=1, le=100)
    standard_data: dict = Field(default_factory=dict)
    # NextGenPSD2 standard_data example:
    # {"access": {"accounts": [...], "balances": [...], "transactions": [...]}}

class ConsentResponse(BaseModel):
    consent_id: uuid.UUID
    consent_type: str
    status: str
    api_standard: str
    recurring_indicator: bool
    valid_until: date | None
    frequency_per_day: int
    standard_data: dict
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)

class ConsentStatusResponse(BaseModel):
    consent_status: str

class ConsentListResponse(BaseModel):
    consents: list[ConsentResponse]
    total: int
    limit: int
    offset: int
```

NextGenPSD2 requires these response headers on consent endpoints:
- `X-Request-ID` (correlation ID -- already handled by Phase 1.7)
- `ASPSP-SCA-Approach` (on consent creation: `REDIRECT`, `DECOUPLED`, or `EMBEDDED`)
- `Location` (on 201 responses: URL of the created resource)

**Testing**:
- Integration: `POST /api/v1/consents` with valid AIS consent -> 201 with ConsentResponse
- Integration: `GET /api/v1/consents/{id}` -> 200 with full consent details
- Integration: `GET /api/v1/consents/{id}/status` -> 200 with `{"consent_status": "received"}`
- Integration: `DELETE /api/v1/consents/{id}` on valid consent -> 204, status becomes "terminatedByTpp"
- Integration: `DELETE /api/v1/consents/{id}` on already revoked consent -> 409
- Integration: consent for different tenant -> 404 (tenant isolation)
- Integration: request without `accounts` scope -> 403

#### 3.4 -- Consent Expiry Monitoring (Celery Worker)

**What**: Background worker that monitors consent expiry dates and transitions expired consents.

**Design**:

```python
# src/openbanking/workers/consent_monitor.py
@celery_app.task(name="consent.check_expiry")
async def check_consent_expiry():
    """Run every hour. Find valid consents past valid_until and expire them."""
    async with async_session() as session:
        expired = await session.execute(
            select(Consent)
            .where(Consent.status == "valid")
            .where(Consent.valid_until < date.today())
        )
        for consent in expired.scalars():
            consent.status = "expired"
            audit = ConsentAudit(
                consent_id=consent.id,
                old_status="valid",
                new_status="expired",
                changed_by="system",
                reason="Consent validity period expired",
            )
            session.add(audit)
        await session.commit()
```

Celery beat schedule:
```python
celery_app.conf.beat_schedule = {
    "check-consent-expiry": {
        "task": "consent.check_expiry",
        "schedule": crontab(minute=0),  # every hour
    },
}
```

**Testing**:
- Integration: consent with `valid_until` = yesterday -> status transitions to "expired" after task runs
- Integration: consent with `valid_until` = tomorrow -> status remains "valid"
- Integration: expired consent has audit record with changed_by="system"
- Unit: task only selects consents in "valid" status

---

## Phase 4: Account Information Services (AIS)

### Purpose
Implement the account aggregation core -- connecting to bank APIs to retrieve account details, balances, and transactions. After this phase, authorised TPPs can access PSU account data through the platform's API, and data is normalised across banking standards into a unified schema.

### Tasks

#### 4.1 -- Institution Data Model & Registry

**What**: Institution table and connector registry for multi-bank connectivity.

**Design**:

```python
# src/openbanking/models/institution.py
class Institution(Base, TimestampMixin):
    __tablename__ = "institutions"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255))
    bic: Mapped[str | None] = mapped_column(String(11))   # ISO 9362
    country_code: Mapped[str] = mapped_column(String(2))   # ISO 3166-1
    api_standard: Mapped[str] = mapped_column(String(30))  # nextgenpsd2, obie, fdx
    supports_ais: Mapped[bool] = mapped_column(Boolean, default=True)
    supports_pis: Mapped[bool] = mapped_column(Boolean, default=False)
    supports_vrp: Mapped[bool] = mapped_column(Boolean, default=False)
    health_status: Mapped[str] = mapped_column(String(20), default="healthy")
    # healthy, degraded, down, unknown
    connection_config: Mapped[dict] = mapped_column(JSONB, default=dict)
    # connection_config: api_base_url, sandbox_url, sca_methods, auth_url, token_url,
    #   rate_limit, custom_headers, quirks
    last_health_check: Mapped[datetime | None]
```

Connector registry:

```python
# src/openbanking/connectors/registry.py
class ConnectorRegistry:
    _connectors: dict[str, type[BankConnector]] = {
        "nextgenpsd2": NextGenPSD2Connector,
        "obie": OBIEConnector,
        "fdx": FDXConnector,
    }

    def get_connector(self, institution: Institution) -> BankConnector:
        connector_cls = self._connectors[institution.api_standard]
        return connector_cls(institution)
```

**Testing**:
- Integration: create institution with NextGenPSD2 config -> persists correctly
- Integration: GIN index on connection_config enables `@>` containment queries
- Unit: ConnectorRegistry returns correct connector class for each api_standard
- Unit: unknown api_standard -> raises `UnsupportedStandardError`

#### 4.2 -- Abstract Bank Connector Interface

**What**: Abstract base class defining the contract for all bank API connectors.

**Design**:

```python
# src/openbanking/connectors/base.py
from abc import ABC, abstractmethod

class BankConnector(ABC):
    def __init__(self, institution: Institution):
        self.institution = institution
        self.config = institution.connection_config
        self.http = httpx.AsyncClient(
            base_url=self.config["api_base_url"],
            timeout=30.0,
        )

    @abstractmethod
    async def get_accounts(self, consent_id: str) -> list[NormalizedAccount]:
        """Fetch accounts accessible under the given consent."""
        ...

    @abstractmethod
    async def get_balances(self, account_resource_id: str) -> list[NormalizedBalance]:
        """Fetch balances for a specific account."""
        ...

    @abstractmethod
    async def get_transactions(
        self,
        account_resource_id: str,
        date_from: date | None = None,
        date_to: date | None = None,
    ) -> list[NormalizedTransaction]:
        """Fetch transactions for a specific account."""
        ...

    @abstractmethod
    async def verify_account_ownership(self, account_resource_id: str) -> AccountVerification:
        """Verify account ownership."""
        ...

# Normalized data classes (cross-standard):
@dataclass
class NormalizedAccount:
    resource_id: str          # bank-assigned ID
    iban: str | None
    account_type: str         # current, savings, credit_card, loan, mortgage, investment
    currency: str             # ISO 4217
    display_name: str | None
    owner_name: str | None
    raw_data: dict            # original bank response for debugging/compliance

@dataclass
class NormalizedBalance:
    balance_type: str         # closingBooked, interimAvailable, etc.
    amount: Decimal
    currency: str
    reference_date: date | None
    raw_data: dict

@dataclass
class NormalizedTransaction:
    amount: Decimal
    currency: str
    booking_date: date | None
    value_date: date | None
    creditor_name: str | None
    debtor_name: str | None
    remittance_info: str | None
    transaction_status: str   # booked, pending, information
    raw_data: dict
```

**Testing**:
- Unit: NormalizedAccount with missing currency -> validation error
- Unit: NormalizedTransaction amount stored as Decimal (not float) for financial precision
- Unit: BankConnector subclass missing abstract method -> TypeError at instantiation

#### 4.3 -- NextGenPSD2 Connector Implementation

**What**: Berlin Group NextGenPSD2 XS2A connector implementing the BankConnector interface.

**Design**:

NextGenPSD2 API paths (per Berlin Group specification):
- `GET /v1/accounts` -- list accounts
- `GET /v1/accounts/{account-id}/balances` -- get balances
- `GET /v1/accounts/{account-id}/transactions` -- get transactions

Required headers per NextGenPSD2:
- `X-Request-ID`: UUID (correlation)
- `Consent-ID`: consent reference
- `PSU-IP-Address`: PSU's IP (for consent tracking)
- `Authorization`: Bearer token

```python
# src/openbanking/connectors/nextgenpsd2.py
class NextGenPSD2Connector(BankConnector):
    async def get_accounts(self, consent_id: str) -> list[NormalizedAccount]:
        response = await self.http.get(
            "/v1/accounts",
            headers={
                "X-Request-ID": str(uuid.uuid4()),
                "Consent-ID": consent_id,
            },
        )
        response.raise_for_status()
        data = response.json()
        return [self._normalize_account(acc) for acc in data.get("accounts", [])]

    def _normalize_account(self, raw: dict) -> NormalizedAccount:
        return NormalizedAccount(
            resource_id=raw["resourceId"],
            iban=raw.get("iban"),
            account_type=self._map_account_type(raw.get("cashAccountType", "CACC")),
            currency=raw.get("currency", "EUR"),
            display_name=raw.get("product"),
            owner_name=raw.get("ownerName"),
            raw_data=raw,
        )

    def _map_account_type(self, cash_account_type: str) -> str:
        mapping = {
            "CACC": "current", "SVGS": "savings", "CARD": "credit_card",
            "LOAN": "loan", "MORT": "mortgage",
        }
        return mapping.get(cash_account_type, "current")
```

**Testing**:
- Unit (fixture-based): NextGenPSD2 account response fixture -> normalized to NormalizedAccount with correct fields
- Unit: account type mapping: CACC -> current, SVGS -> savings, unknown -> current
- Integration (mocked API): get_accounts with valid consent -> returns list of NormalizedAccount
- Integration (mocked API): bank returns 401 -> raises `BankAuthenticationError`
- Integration (mocked API): bank returns 429 -> raises `BankRateLimitError` with retry-after

#### 4.4 -- Account & Transaction Data Models

**What**: Account, balance, and transaction tables following the Hybrid model.

**Design**:

```python
# src/openbanking/models/account.py
class Account(Base, TimestampMixin, TenantMixin):
    __tablename__ = "accounts"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    institution_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("institutions.id"))
    psu_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    consent_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("consents.id"))
    iban: Mapped[str | None] = mapped_column(String(34), index=True)
    account_type: Mapped[str] = mapped_column(String(30))
    currency: Mapped[str] = mapped_column(String(3))  # ISO 4217
    display_name: Mapped[str | None] = mapped_column(String(255))
    owner_name: Mapped[str | None] = mapped_column(String(255))
    status: Mapped[str] = mapped_column(String(20), default="enabled")
    raw_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    last_synced_at: Mapped[datetime | None]

class Balance(Base):
    __tablename__ = "balances"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    account_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("accounts.id"), index=True)
    balance_type: Mapped[str] = mapped_column(String(30))
    amount: Mapped[Decimal] = mapped_column(Numeric(18, 4))
    currency: Mapped[str] = mapped_column(String(3))
    reference_date: Mapped[date | None]
    raw_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    fetched_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

# src/openbanking/models/transaction.py
class Transaction(Base, TenantMixin):
    __tablename__ = "transactions"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    account_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("accounts.id"))
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    amount: Mapped[Decimal] = mapped_column(Numeric(18, 4))
    currency: Mapped[str] = mapped_column(String(3))
    booking_date: Mapped[date | None]
    value_date: Mapped[date | None]
    creditor_name: Mapped[str | None] = mapped_column(String(255))
    debtor_name: Mapped[str | None] = mapped_column(String(255))
    remittance_info: Mapped[str | None]
    transaction_status: Mapped[str] = mapped_column(String(20), default="booked")
    enrichment: Mapped[dict] = mapped_column(JSONB, default=dict)
    raw_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    fetched_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

Composite index on `(account_id, booking_date)` for efficient transaction range queries.

**Testing**:
- Integration: create account linked to institution and user -> FK constraints pass
- Integration: balance with Decimal(18,4) precision -> stores and retrieves correctly
- Integration: transaction JSONB enrichment and raw_data -> round-trip without loss
- Integration: composite index on (account_id, booking_date) exists

#### 4.5 -- Account & Transaction API Endpoints

**What**: REST API for account listing, balance retrieval, and transaction history.

**Design**:

Endpoints:
- `GET /api/v1/accounts` -- list accounts for PSU under active consent (scope: `accounts`)
- `GET /api/v1/accounts/{accountId}` -- get account details
- `GET /api/v1/accounts/{accountId}/balances` -- get current balances
- `GET /api/v1/accounts/{accountId}/transactions` -- get transaction history
  - Query params: `dateFrom`, `dateTo`, `bookingStatus` (booked/pending/both)
  - Pagination: `limit` (default 50, max 500), `offset`
- `GET /api/v1/accounts/{accountId}/transactions/{transactionId}` -- get single transaction

Response schemas:

```python
class AccountResponse(BaseModel):
    account_id: uuid.UUID
    iban: str | None
    account_type: str
    currency: str
    display_name: str | None
    owner_name: str | None
    status: str
    institution_name: str
    last_synced_at: datetime | None

class BalanceResponse(BaseModel):
    balance_type: str
    amount: Decimal
    currency: str
    reference_date: date | None

class TransactionResponse(BaseModel):
    transaction_id: uuid.UUID
    amount: Decimal
    currency: str
    booking_date: date | None
    value_date: date | None
    creditor_name: str | None
    debtor_name: str | None
    remittance_info: str | None
    transaction_status: str
    enrichment: dict

class TransactionListResponse(BaseModel):
    transactions: list[TransactionResponse]
    total: int
    limit: int
    offset: int
    account_id: uuid.UUID
```

Consent validation middleware: every account/transaction request checks:
1. Bearer token has `accounts` scope
2. Consent linked to the account is in `valid` status
3. Consent's `frequency_per_day` has not been exceeded (tracked in Redis)

**Testing**:
- Integration: `GET /api/v1/accounts` with valid consent -> returns accounts list
- Integration: `GET /api/v1/accounts/{id}/transactions?dateFrom=2026-01-01&dateTo=2026-03-31` -> filtered results
- Integration: request exceeding consent frequency_per_day -> 429
- Integration: request for account not covered by consent -> 403
- Integration: request with expired consent -> 403 with `CONSENT_EXPIRED` error code
- E2E: create consent -> authorise -> list accounts -> get balances -> get transactions

#### 4.6 -- Account Sync Worker

**What**: Celery task that periodically syncs account data and balances from bank APIs.

**Design**:

```python
# src/openbanking/workers/sync_accounts.py
@celery_app.task(name="accounts.sync")
async def sync_accounts(consent_id: str):
    """Fetch latest accounts, balances, and recent transactions for a consent."""
    async with async_session() as session:
        consent = await session.get(Consent, consent_id)
        if consent.status != "valid":
            return
        institution = await session.get(Institution, ...)
        connector = connector_registry.get_connector(institution)

        # Sync accounts
        bank_accounts = await connector.get_accounts(consent.id)
        for bank_acc in bank_accounts:
            account = await upsert_account(session, bank_acc, consent)
            # Sync balances
            balances = await connector.get_balances(bank_acc.resource_id)
            await upsert_balances(session, account.id, balances)

        await session.commit()
```

Scheduling: triggered on consent authorisation and then runs on a configurable interval (default: every 6 hours for active consents).

**Testing**:
- Integration (mocked bank API): sync task creates new accounts from bank response
- Integration (mocked bank API): sync task updates existing account balances
- Integration: sync skips consents not in "valid" status
- Unit: upsert logic correctly handles new vs existing accounts (matching on resource_id + institution_id)

---

## Phase 5: Webhook Event System & Audit Trail

### Purpose
Implement the event notification system (webhooks) and the immutable audit log. After this phase, tenants receive real-time notifications for consent lifecycle changes, transaction updates, and payment status changes. The audit log captures every significant action for regulatory compliance.

### Tasks

#### 5.1 -- Webhook Subscription Management

**What**: CRUD API for webhook subscriptions with secret-based signature verification.

**Design**:

```python
# src/openbanking/models/webhook.py
class WebhookSubscription(Base, TimestampMixin, TenantMixin):
    __tablename__ = "webhook_subscriptions"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    url: Mapped[str]
    secret_hash: Mapped[str] = mapped_column(String(255))
    event_types: Mapped[list[str]] = mapped_column(ARRAY(Text))
    # Event types: consent.created, consent.authorised, consent.revoked, consent.expired,
    #   transaction.new, payment.status_changed, institution.health_changed
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    # config: max_retries (default 5), retry_backoff_seconds (default [60, 300, 900, 3600, 7200])

class WebhookDelivery(Base):
    __tablename__ = "webhook_deliveries"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    subscription_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("webhook_subscriptions.id"))
    event_type: Mapped[str] = mapped_column(String(100))
    payload: Mapped[dict] = mapped_column(JSONB)
    http_status: Mapped[int | None]
    attempt_number: Mapped[int] = mapped_column(default=1)
    delivered_at: Mapped[datetime | None]
    next_retry_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

Endpoints:
- `POST /api/v1/webhooks` -- create subscription
- `GET /api/v1/webhooks` -- list subscriptions
- `DELETE /api/v1/webhooks/{id}` -- deactivate subscription
- `POST /api/v1/webhooks/{id}/test` -- send test event

**Testing**:
- Integration: create subscription -> 201 with subscription ID
- Integration: list subscriptions -> only returns tenant's subscriptions
- Integration: test webhook -> sends test payload to URL and records delivery
- Unit: secret is hashed before storage

#### 5.2 -- Webhook Delivery Worker

**What**: Celery task that delivers webhook events with exponential backoff retries.

**Design**:

```python
# src/openbanking/workers/deliver_webhooks.py
@celery_app.task(name="webhooks.deliver", bind=True, max_retries=5)
async def deliver_webhook(self, delivery_id: str):
    """Deliver a webhook event. Retries with exponential backoff on failure."""
    async with async_session() as session:
        delivery = await session.get(WebhookDelivery, delivery_id)
        subscription = await session.get(WebhookSubscription, delivery.subscription_id)

        # Compute HMAC-SHA256 signature
        signature = hmac.new(
            subscription.secret.encode(),
            json.dumps(delivery.payload).encode(),
            hashlib.sha256
        ).hexdigest()

        try:
            response = await httpx.AsyncClient().post(
                subscription.url,
                json=delivery.payload,
                headers={
                    "X-Webhook-Signature": f"sha256={signature}",
                    "X-Webhook-Event": delivery.event_type,
                    "X-Webhook-Delivery-ID": str(delivery.id),
                },
                timeout=10.0,
            )
            delivery.http_status = response.status_code
            delivery.delivered_at = datetime.utcnow()
            if response.status_code >= 400:
                raise WebhookDeliveryError(f"HTTP {response.status_code}")
        except Exception as exc:
            delivery.attempt_number += 1
            backoff = [60, 300, 900, 3600, 7200]
            delay = backoff[min(delivery.attempt_number - 1, len(backoff) - 1)]
            delivery.next_retry_at = datetime.utcnow() + timedelta(seconds=delay)
            await session.commit()
            raise self.retry(exc=exc, countdown=delay)

        await session.commit()
```

Webhook payload structure:
```json
{
    "event_id": "uuid",
    "event_type": "consent.authorised",
    "timestamp": "2026-05-29T10:30:00Z",
    "data": {
        "consent_id": "uuid",
        "status": "valid",
        "psu_id": "uuid"
    }
}
```

**Testing**:
- Integration (mocked endpoint): successful delivery -> http_status=200, delivered_at set
- Integration (mocked endpoint): endpoint returns 500 -> retries with backoff
- Integration: after max retries -> delivery marked as failed, no more retries
- Unit: HMAC signature computation is correct
- Unit: backoff intervals follow configured pattern

#### 5.3 -- Audit Log Service

**What**: Append-only audit log capturing all significant platform actions.

**Design**:

```python
# src/openbanking/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(index=True)
    actor_type: Mapped[str] = mapped_column(String(30))  # user, tpp, system, ai_agent
    actor_id: Mapped[uuid.UUID | None]
    action: Mapped[str] = mapped_column(String(100))
    # Actions: consent.created, consent.authorised, consent.revoked, account.synced,
    #   payment.initiated, payment.status_changed, webhook.delivered, client.registered
    resource_type: Mapped[str] = mapped_column(String(50))
    resource_id: Mapped[uuid.UUID | None]
    details: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

# src/openbanking/services/audit_service.py
class AuditService:
    async def log(
        self,
        session: AsyncSession,
        tenant_id: uuid.UUID,
        actor_type: str,
        actor_id: uuid.UUID | None,
        action: str,
        resource_type: str,
        resource_id: uuid.UUID | None,
        details: dict | None = None,
    ) -> None:
        entry = AuditLog(
            tenant_id=tenant_id,
            actor_type=actor_type,
            actor_id=actor_id,
            action=action,
            resource_type=resource_type,
            resource_id=resource_id,
            details=details or {},
        )
        session.add(entry)
```

Admin query endpoint:
- `GET /api/v1/admin/audit` -- query audit log with filters (action, resource_type, date range)
- Pagination with cursor-based approach (using created_at + id for deterministic ordering)

**Testing**:
- Integration: AuditService.log creates an audit entry with all fields
- Integration: audit log query by action -> returns matching entries
- Integration: audit log query by date range -> correct filtering
- Unit: audit entries are append-only (no UPDATE or DELETE operations exposed)

---

## Phase 6: Payment Initiation Service (PIS)

### Purpose
Implement account-to-account payment initiation, including single payments and the payment status lifecycle. After this phase, authorised TPPs can initiate payments through the platform, and payment status transitions are tracked through ISO 20022-aligned status codes.

### Tasks

#### 6.1 -- Payment Data Model

**What**: Payment and payment_status_log tables with ISO 20022 status codes and JSONB for standard-specific fields.

**Design**:

```python
# src/openbanking/models/payment.py
class Payment(Base, TimestampMixin, TenantMixin):
    __tablename__ = "payments"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), index=True)
    consent_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("consents.id"))
    api_standard: Mapped[str] = mapped_column(String(30))
    payment_type: Mapped[str] = mapped_column(String(30))  # single, bulk, periodic, vrp
    creditor_name: Mapped[str] = mapped_column(String(255))
    creditor_iban: Mapped[str | None] = mapped_column(String(34))
    amount: Mapped[Decimal] = mapped_column(Numeric(18, 4))
    currency: Mapped[str] = mapped_column(String(3))
    status: Mapped[str] = mapped_column(String(30), default="received", index=True)
    # Status lifecycle (ISO 20022 aligned):
    # received -> accp (accepted) -> acsp (settlement in progress) -> acsc (settled)
    # received -> rjct (rejected)
    # acsp -> acwc (accepted with change) | rjct
    # any -> canc (cancelled by TPP before execution)
    standard_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    iso20022_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    aspsp_payment_id: Mapped[str | None] = mapped_column(String(100))

class PaymentStatusLog(Base):
    __tablename__ = "payment_status_log"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    payment_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("payments.id"), index=True)
    old_status: Mapped[str | None] = mapped_column(String(30))
    new_status: Mapped[str] = mapped_column(String(30))
    status_detail: Mapped[dict] = mapped_column(JSONB, default=dict)
    # status_detail: iso20022_code, reason_code, reason_text
    changed_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

**Testing**:
- Integration: payment with ISO 20022 status codes persists correctly
- Integration: payment_status_log creates entry on each status transition
- Integration: iso20022_data JSONB stores nested pacs.008 structures

#### 6.2 -- Payment Service & API Endpoints

**What**: Payment initiation business logic and REST endpoints.

**Design**:

Service methods:
- `initiate_payment(tenant_id, consent_id, request)` -- create payment, forward to bank connector
- `get_payment_status(payment_id, tenant_id)` -- check current status (with bank API poll if needed)
- `cancel_payment(payment_id, tenant_id)` -- cancel if still in cancellable state

Endpoints:
- `POST /api/v1/payments/sepa-credit-transfers` -- initiate SEPA payment (scope: `payments`)
- `GET /api/v1/payments/{paymentId}` -- get payment details
- `GET /api/v1/payments/{paymentId}/status` -- get payment status
- `DELETE /api/v1/payments/{paymentId}` -- cancel payment

Request schema:
```python
class PaymentInitiationRequest(BaseModel):
    creditor_name: str = Field(max_length=255)
    creditor_iban: str = Field(pattern=r"^[A-Z]{2}\d{2}[A-Z0-9]{1,30}$")  # ISO 13616
    amount: Decimal = Field(gt=0, decimal_places=2)
    currency: str = Field(pattern=r"^[A-Z]{3}$")  # ISO 4217
    end_to_end_id: str | None = Field(default=None, max_length=35)
    remittance_info: str | None = Field(default=None, max_length=140)
    requested_execution_date: date | None = None
```

**Testing**:
- Integration: `POST /api/v1/payments/sepa-credit-transfers` with valid data -> 201
- Integration: payment with invalid IBAN format -> 422
- Integration: payment without `payments` scope -> 403
- Integration: payment with consent not covering PIS -> 403
- Integration: `GET /api/v1/payments/{id}/status` -> returns current ISO 20022 status
- Integration: `DELETE /api/v1/payments/{id}` on settled payment -> 409
- Unit: IBAN validation via schwifty library

#### 6.3 -- Bank Payment Connector (NextGenPSD2)

**What**: Extend the NextGenPSD2 connector with payment initiation methods.

**Design**:

NextGenPSD2 PIS API paths:
- `POST /v1/payments/{payment-product}` -- initiate payment
- `GET /v1/payments/{payment-product}/{paymentId}/status` -- get payment status

```python
# Addition to NextGenPSD2Connector:
async def initiate_payment(
    self,
    consent_id: str,
    payment_product: str,  # sepa-credit-transfers, instant-sepa-credit-transfers
    payment_data: dict,
) -> PaymentInitiationResponse:
    response = await self.http.post(
        f"/v1/payments/{payment_product}",
        json=payment_data,
        headers={"Consent-ID": consent_id, "X-Request-ID": str(uuid.uuid4())},
    )
    response.raise_for_status()
    return PaymentInitiationResponse(
        aspsp_payment_id=response.json()["paymentId"],
        status=response.json()["transactionStatus"],
    )
```

**Testing**:
- Integration (mocked bank API): initiate payment -> returns paymentId and status
- Integration (mocked bank API): bank rejects payment (insufficient funds) -> RJCT status with reason
- Unit: payment request body conforms to NextGenPSD2 specification

---

## Phase 7: Admin Dashboard (Next.js)

### Purpose
Build the operator-facing admin dashboard for consent audit trails, transaction monitoring, institution health, and webhook management. After this phase, platform operators have visual oversight of all platform activity.

### Tasks

#### 7.1 -- Next.js Project Setup

**What**: Initialise the Next.js 15 dashboard with shadcn/ui, Tailwind CSS, and API client.

**Design**:

```
dashboard/
├── package.json
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── src/
│   ├── app/
│   │   ├── layout.tsx          # Root layout with sidebar navigation
│   │   ├── page.tsx            # Dashboard overview
│   │   ├── consents/
│   │   │   ├── page.tsx        # Consent list with filters
│   │   │   └── [id]/page.tsx   # Consent detail with audit timeline
│   │   ├── transactions/
│   │   │   └── page.tsx        # Transaction browser
│   │   ├── payments/
│   │   │   ├── page.tsx        # Payment list
│   │   │   └── [id]/page.tsx   # Payment detail with status timeline
│   │   ├── institutions/
│   │   │   └── page.tsx        # Institution health dashboard
│   │   ├── webhooks/
│   │   │   └── page.tsx        # Webhook subscription management
│   │   └── audit/
│   │       └── page.tsx        # Audit log viewer
│   ├── components/
│   │   ├── sidebar.tsx
│   │   ├── data-table.tsx      # Reusable data table with sorting/filtering
│   │   ├── status-badge.tsx    # Color-coded status indicator
│   │   ├── timeline.tsx        # Consent/payment status timeline
│   │   └── charts/
│   │       ├── api-calls.tsx   # API usage chart
│   │       └── health.tsx      # Institution health overview
│   └── lib/
│       ├── api-client.ts       # Typed API client (fetch-based)
│       └── types.ts            # TypeScript types matching API schemas
```

**Testing**:
- E2E (Playwright): dashboard loads, sidebar navigation works
- E2E (Playwright): consent list displays data from API

#### 7.2 -- Consent Management View

**What**: Consent list page with filtering, detail page with audit timeline.

**Design**:
- List view: table with columns (consent ID, type, status, TPP name, PSU, created, expires)
- Filters: status, consent_type, date range
- Detail view: consent details card + vertical timeline showing all audit events
- Actions: revoke consent button (with confirmation dialog)

**Testing**:
- E2E: filter consents by status -> table shows filtered results
- E2E: click consent row -> navigates to detail page with timeline
- E2E: revoke consent -> confirmation dialog -> consent status updates

#### 7.3 -- Transaction Browser & Institution Health

**What**: Transaction list with search/filter and institution health status dashboard.

**Design**:
- Transaction browser: searchable table with date range picker, account filter, category filter
- Institution health: card grid showing each institution's status (healthy/degraded/down), response time, last check time
- Health history: sparkline chart per institution showing uptime over 30 days

**Testing**:
- E2E: filter transactions by date range -> correct results
- E2E: institution health cards show correct status colors
- E2E: institution with "down" status shows red indicator

---

## Phase 8: Transaction Enrichment (AI/ML)

### Purpose
Add ML-powered transaction categorisation and merchant identification. After this phase, transactions are automatically enriched with spending categories, merchant names, and confidence scores.

### Tasks

#### 8.1 -- Enrichment Service Interface

**What**: Abstract enrichment interface and rule-based baseline implementation.

**Design**:

```python
# src/openbanking/enrichment/categoriser.py
from abc import ABC, abstractmethod

class TransactionEnricher(ABC):
    @abstractmethod
    async def enrich(self, transaction: Transaction) -> EnrichmentResult:
        ...

@dataclass
class EnrichmentResult:
    category: str | None
    subcategory: str | None
    merchant_name: str | None
    merchant_mcc: str | None  # ISO 18245
    confidence: float         # 0.0 to 1.0
    model_version: str
    tags: list[str]           # recurring, essential, discretionary

class RuleBasedEnricher(TransactionEnricher):
    """Baseline enrichment using MCC codes and keyword matching."""
    MCC_CATEGORY_MAP = {
        "5411": ("groceries", "supermarket"),
        "5812": ("dining", "restaurant"),
        "5541": ("transport", "fuel"),
        # ... 100+ mappings
    }

    async def enrich(self, transaction: Transaction) -> EnrichmentResult:
        mcc = transaction.raw_data.get("merchantCategoryCode")
        if mcc and mcc in self.MCC_CATEGORY_MAP:
            cat, subcat = self.MCC_CATEGORY_MAP[mcc]
            return EnrichmentResult(
                category=cat, subcategory=subcat,
                merchant_name=transaction.creditor_name,
                merchant_mcc=mcc, confidence=0.85,
                model_version="rule-based-v1", tags=[],
            )
        # Keyword-based fallback
        ...
```

**Testing**:
- Unit: transaction with MCC 5411 -> category "groceries", confidence 0.85
- Unit: transaction without MCC -> keyword fallback attempts enrichment
- Unit: transaction with no matching rules -> EnrichmentResult with None category and 0.0 confidence

#### 8.2 -- Enrichment Worker & Batch Processing

**What**: Celery task that enriches new transactions in batches.

**Design**:

```python
@celery_app.task(name="enrichment.batch")
async def enrich_transactions_batch():
    """Find unenriched transactions and enrich them in batches of 100."""
    async with async_session() as session:
        unenriched = await session.execute(
            select(Transaction)
            .where(Transaction.enrichment == {})
            .limit(100)
        )
        enricher = get_enricher()  # RuleBasedEnricher or MLEnricher
        for tx in unenriched.scalars():
            result = await enricher.enrich(tx)
            tx.enrichment = {
                "category": result.category,
                "subcategory": result.subcategory,
                "merchant": {
                    "name": result.merchant_name,
                    "mcc": result.merchant_mcc,
                },
                "confidence": result.confidence,
                "model_version": result.model_version,
                "enriched_at": datetime.utcnow().isoformat(),
                "tags": result.tags,
            }
        await session.commit()
```

**Testing**:
- Integration: 50 unenriched transactions -> all enriched after task runs
- Integration: already-enriched transactions not re-processed
- Unit: batch size limit of 100 is respected

#### 8.3 -- ML Model Registry

**What**: Table and service for tracking ML model versions and deployment.

**Design**:

```python
class MLModel(Base, TimestampMixin):
    __tablename__ = "ml_models"
    __table_args__ = (UniqueConstraint("model_name", "model_version"),)

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    model_name: Mapped[str] = mapped_column(String(100))
    model_version: Mapped[str] = mapped_column(String(50))
    model_type: Mapped[str] = mapped_column(String(50))
    # model_type: transaction_enrichment, fraud_detection, anomaly_detection
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    # config: algorithm, training_data_range, accuracy, f1_score, artifact_url
    is_active: Mapped[bool] = mapped_column(Boolean, default=False)
    deployed_at: Mapped[datetime | None]
```

**Testing**:
- Integration: register model version -> persists with config
- Integration: activate new model -> previous version deactivated
- Unit: unique constraint on (model_name, model_version)

---

## Phase 9: Institution Health Monitoring

### Purpose
Implement automated monitoring of bank API availability and performance, with auto-routing to fallback endpoints when institutions are degraded or down. After this phase, the platform proactively detects bank API issues and maintains operational resilience.

### Tasks

#### 9.1 -- Health Check Worker

**What**: Periodic Celery task that checks each institution's API health.

**Design**:

```python
# src/openbanking/workers/health_checker.py
@celery_app.task(name="health.check_institutions")
async def check_institution_health():
    """Check API health for all active institutions."""
    async with async_session() as session:
        institutions = await session.execute(select(Institution))
        for inst in institutions.scalars():
            connector = connector_registry.get_connector(inst)
            start = time.monotonic()
            try:
                await connector.health_check()  # lightweight /status or /accounts ping
                response_time_ms = int((time.monotonic() - start) * 1000)
                new_status = "healthy" if response_time_ms < 5000 else "degraded"
            except Exception as e:
                response_time_ms = None
                new_status = "down"

            # Record health check
            health_log = InstitutionHealthLog(
                institution_id=inst.id,
                status=new_status,
                response_time_ms=response_time_ms,
                error_detail={"error": str(e)} if new_status == "down" else {},
            )
            session.add(health_log)

            # Update institution status
            if inst.health_status != new_status:
                inst.health_status = new_status
                # Trigger webhook: institution.health_changed
                await enqueue_webhook_event(
                    tenant_id=None,  # broadcast to all tenants using this institution
                    event_type="institution.health_changed",
                    data={"institution_id": str(inst.id), "old_status": inst.health_status, "new_status": new_status},
                )
            inst.last_health_check = datetime.utcnow()

        await session.commit()
```

Health check model:
```python
class InstitutionHealthLog(Base):
    __tablename__ = "institution_health_log"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    institution_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("institutions.id"), index=True)
    status: Mapped[str] = mapped_column(String(20))
    response_time_ms: Mapped[int | None]
    error_detail: Mapped[dict] = mapped_column(JSONB, default=dict)
    checked_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

Schedule: every 5 minutes.

**Testing**:
- Integration (mocked): healthy bank API -> status "healthy", response time recorded
- Integration (mocked): bank API timeout -> status "down", error details recorded
- Integration: status change triggers webhook event
- Unit: response time > 5000ms -> "degraded"

#### 9.2 -- Health Dashboard API

**What**: API endpoints for institution health data consumed by the admin dashboard.

**Design**:

Endpoints:
- `GET /api/v1/admin/institutions` -- list institutions with current health status
- `GET /api/v1/admin/institutions/{id}/health-history` -- health check history with response times
  - Query params: `period` (1h, 24h, 7d, 30d)

**Testing**:
- Integration: health history returns time series data suitable for charting
- Integration: institution list includes health_status and last_health_check

---

## Phase 10: Variable Recurring Payments (VRP) & Multi-Currency

### Purpose
Extend payment capabilities with VRP support (recurring payments of varying amounts without re-authorisation) and multi-currency transaction handling. These are key v1.1 differentiators identified in the features analysis.

### Tasks

#### 10.1 -- VRP Consent & Payment Extension

**What**: Extend consent and payment models to support VRP-specific fields.

**Design**:

VRP consent `standard_data` fields:
```json
{
    "vrp": {
        "max_individual_amount": 100.00,
        "max_cumulative_amount": 1000.00,
        "max_cumulative_period": "monthly",
        "valid_from": "2026-06-01",
        "valid_to": "2027-06-01",
        "allowed_frequency": "daily"
    }
}
```

VRP payment validation: before initiating each VRP payment, check:
1. Individual amount <= max_individual_amount
2. Cumulative amount within current period <= max_cumulative_amount
3. Payment count within period <= allowed_frequency
4. Current date within valid_from/valid_to range

**Testing**:
- Integration: VRP payment within limits -> accepted
- Integration: VRP payment exceeding individual max -> rejected with reason
- Integration: VRP cumulative amount check across multiple payments in period
- Unit: period boundary calculation (monthly, weekly) is correct

#### 10.2 -- Multi-Currency Support

**What**: Handle accounts and transactions in non-EUR currencies; add exchange rate context.

**Design**:

- All monetary amounts stored as `Decimal(18, 4)` with explicit `currency` (ISO 4217)
- Transaction response includes `instructed_amount` (original) and `transaction_amount` (after conversion) when currencies differ
- No automatic conversion -- amounts stored as received from bank APIs
- Currency validation via iso4217 library

**Testing**:
- Integration: GBP account with GBP transactions -> correct currency throughout
- Integration: cross-currency payment (EUR to GBP) -> both currencies preserved
- Unit: invalid currency code "XXX" -> validation error (unless it's a valid ISO 4217 code)

---

## Phase 11: White-Label & Tenant Customisation

### Purpose
Enable fintech partners to customise the consent and account-linking user flows with their own branding. After this phase, each tenant can configure logos, colours, and consent page text.

### Tasks

#### 11.1 -- Tenant Branding Configuration

**What**: Extend tenant config JSONB with branding fields; serve branding via API.

**Design**:

Tenant `config.branding` structure:
```json
{
    "branding": {
        "logo_url": "https://...",
        "favicon_url": "https://...",
        "primary_color": "#1a2b3c",
        "accent_color": "#4d5e6f",
        "company_name": "Acme Fintech",
        "consent_page_title": "Connect Your Bank Account",
        "consent_page_description": "We use open banking to securely access your account data.",
        "custom_css_url": null
    }
}
```

Endpoint: `GET /api/v1/branding/{tenant_slug}` -- public, no auth required (consumed by consent UX)

**Testing**:
- Integration: branding endpoint returns tenant's branding config
- Integration: unknown tenant slug -> 404
- Unit: branding with missing optional fields -> defaults applied

#### 11.2 -- Hosted Consent Page

**What**: Server-rendered consent page that applies tenant branding.

**Design**:
- Simple Next.js page at `/consent/{tenant_slug}/{consent_id}` in the dashboard app
- Fetches branding from API, displays consent details (accounts requested, TPP name, permissions)
- PSU approves or rejects; page calls consent authorisation API
- SCA redirect flow: consent page redirects to bank's SCA URL, handles callback

**Testing**:
- E2E (Playwright): consent page renders with tenant's logo and colours
- E2E (Playwright): approve consent -> redirects to TPP callback URL
- E2E (Playwright): reject consent -> consent status becomes "rejected"

---

## Phase 12: Anomaly Detection & Fraud Signals

### Purpose
Add AI-powered anomaly detection on consent patterns and payment flows. After this phase, the platform generates fraud signals when unusual patterns are detected -- such as abnormal consent request velocity, payment amounts deviating from historical patterns, or suspicious counterparty relationships.

### Tasks

#### 12.1 -- Anomaly Detection Service

**What**: Service that analyses consent and payment patterns and flags anomalies.

**Design**:

```python
# src/openbanking/services/anomaly_service.py
class AnomalyDetector:
    async def check_consent_velocity(
        self, tpp_client_id: uuid.UUID, window_hours: int = 24
    ) -> AnomalyResult | None:
        """Flag if a TPP is requesting consents at an abnormal rate."""
        recent_count = await self._count_recent_consents(tpp_client_id, window_hours)
        baseline = await self._get_baseline_rate(tpp_client_id)
        if recent_count > baseline * 3:  # 3x above baseline
            return AnomalyResult(
                anomaly_type="consent_velocity",
                severity="high",
                details={"recent_count": recent_count, "baseline": baseline},
            )
        return None

    async def check_payment_amount_deviation(
        self, account_id: uuid.UUID, amount: Decimal
    ) -> AnomalyResult | None:
        """Flag if a payment amount significantly deviates from account history."""
        stats = await self._get_payment_stats(account_id)
        if stats and amount > stats.mean + (3 * stats.stddev):
            return AnomalyResult(
                anomaly_type="payment_amount_deviation",
                severity="medium",
                details={"amount": float(amount), "mean": float(stats.mean), "stddev": float(stats.stddev)},
            )
        return None

@dataclass
class AnomalyResult:
    anomaly_type: str
    severity: str  # low, medium, high, critical
    details: dict
```

Anomaly checks run:
1. On consent creation (consent_velocity check)
2. On payment initiation (payment_amount_deviation check)
3. Periodically via Celery task (batch analysis of transaction patterns)

Results stored in audit log with `action="anomaly.detected"` and details in JSONB.

**Testing**:
- Unit: TPP with 100 consents in 24h when baseline is 10 -> anomaly flagged
- Unit: payment of 10,000 EUR when mean is 50 EUR -> anomaly flagged
- Unit: normal payment amount -> no anomaly
- Integration: anomaly triggers audit log entry with correct details
- Integration: anomaly triggers webhook event to tenant

#### 12.2 -- Fraud Signal API

**What**: API endpoint for querying detected anomalies and fraud signals.

**Design**:

Endpoints:
- `GET /api/v1/admin/anomalies` -- list recent anomalies with filters (severity, type, date range)
- `GET /api/v1/admin/anomalies/summary` -- aggregated anomaly counts by type and severity

**Testing**:
- Integration: anomalies endpoint returns filtered results
- Integration: summary endpoint returns correct counts

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    --- required by everything
    |
Phase 2: OAuth / FAPI Security        --- requires Phase 1
    |
Phase 3: Consent Management           --- requires Phase 2
    |
    +--- Phase 4: Account Info (AIS)   --- requires Phase 3
    |        |
    |        +--- Phase 8: Enrichment  --- requires Phase 4
    |
    +--- Phase 5: Webhooks & Audit     --- requires Phase 3 (can parallel with Phase 4)
    |
    +--- Phase 6: Payments (PIS)       --- requires Phase 3
    |        |
    |        +--- Phase 10: VRP & Multi-Currency --- requires Phase 6
    |
    +--- Phase 9: Health Monitoring    --- requires Phase 4 (connector infra)
    |
Phase 7: Admin Dashboard              --- requires Phases 3, 4, 5 (reads their data)
    |
Phase 11: White-Label                  --- requires Phases 3, 7
    |
Phase 12: Anomaly Detection           --- requires Phases 3, 4, 6 (analyses their data)
```

**Parallelism opportunities:**
- Phases 4, 5, and 6 can all begin concurrently after Phase 3 is complete
- Phase 8 (enrichment) and Phase 9 (health monitoring) can develop concurrently after Phase 4
- Phase 10 (VRP) and Phase 12 (anomaly detection) can develop concurrently
- Phase 7 (dashboard) can begin after Phase 3 with incremental additions as Phases 4-6 complete

---

## Definition of Done (per phase)

1. All tasks implemented with complete code.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. `ruff check src/` reports zero violations.
5. `mypy src/` reports zero errors.
6. `docker compose up --build` starts all services successfully.
7. Feature works end-to-end (manual or automated verification).
8. New API endpoints appear in auto-generated OpenAPI spec at `/docs`.
9. Database migrations created via Alembic and tested (upgrade + downgrade).
10. New configuration options documented in `config.py` with sensible defaults.
11. Audit log entries created for all significant actions.
12. Structured log output includes correlation IDs for all request flows.
13. Webhook events emitted for relevant state changes.
