# Banking Core System (Community Bank) — Phased Development Plan

> Project: 216-banking-core-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ | LLM/AI integration is central to the product (BSA/AML scoring, fraud detection, regulatory change monitoring). Python has the strongest ecosystem for ML pipelines (scikit-learn, transformers), LLM SDKs (anthropic, openai), and financial libraries (pandas, numpy). FastAPI provides high-performance async APIs. Community banks hiring Python developers is easier than Go/Rust. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 schema generation (required by FDX and standards.md), async support for long-running AI inference calls, Pydantic v2 for request/response validation, dependency injection for RBAC. |
| Database | PostgreSQL 16+ | Required by the data model (JSONB, GIN indexes, CHECK constraints, UUID generation). All four data model suggestions target PostgreSQL. Mature financial-grade database with ACID guarantees. |
| ORM / query layer | SQLAlchemy 2.0 + Alembic | SQLAlchemy 2.0 provides async support, type-safe query construction, and Alembic for versioned migrations. Preferred over raw SQL for maintainability across 28+ tables. |
| Data model | Hybrid Relational + JSONB (Model 3) with Graph Layer from Model 4 | Model 3 offers the best balance: relational rigor for GL, payments, and compliance; JSONB flexibility for product-specific fields; fewer tables (28 vs 35). The graph_node/graph_edge tables from Model 4 are added for AI-native AML/fraud detection. Combined total: ~30 tables. |
| Task queue | Celery 5.x + Redis | ACH file generation, AML batch scoring, wire OFAC screening, interest accrual, and regulatory change monitoring are all async workloads. Celery is the Python standard; Redis serves as both broker and cache. |
| Event streaming | PostgreSQL LISTEN/NOTIFY + Redis Streams | For MVP, PostgreSQL notifications trigger event consumers (graph projections, AML scoring). Redis Streams provide durable event log for AI pipeline consumers. Avoids Kafka operational complexity at MVP scale. |
| Frontend | None (API-only MVP) | The MVP is API-first with OpenAPI 3.1 documentation. A teller UI is deferred to Phase 8 (v1.1). Digital banking UIs are out of scope — institutions pair with existing digital banking vendors (Q2, Banno). |
| Authentication | OAuth 2.0 + JWT (python-jose) + FAPI 1.0 profile | Standards.md mandates FAPI 1.0 for FDX certification. OAuth 2.0 Authorization Code with PKCE for user flows, client_credentials for service-to-service. mTLS support for high-value API calls (RFC 8705). |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a key differentiator over SaaS-only competitors. Docker Compose bundles the API, PostgreSQL, Redis, and Celery worker for single-command startup. |
| Testing framework | pytest + pytest-asyncio + pytest-cov | Standard Python testing. pytest-asyncio for async endpoint tests. Factory Boy for test data generation. |
| Code quality | Ruff (linter + formatter) + mypy (strict) | Ruff replaces black/isort/flake8 in a single tool. mypy strict mode enforces type safety across the financial domain layer. |
| Package manager | uv | Fast, modern Python package manager. Generates lock files for reproducible builds. |
| Key libraries | `anthropic` (LLM calls), `python-jose` (JWT), `passlib[bcrypt]` (password hashing), `httpx` (async HTTP client), `pydantic` v2 (validation), `jinja2` (ACH file templates), `lxml` (ISO 20022 XML), `cryptography` (PII encryption) |
| MCP server | `mcp` Python SDK | For AI agent integration layer (v1.1). Aligns with Nymbus's MCP Server pattern documented in standards.md. |

### Project Structure

```
banking-core-system/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── src/
│   └── banking_core/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory
│       ├── config.py                  # Pydantic Settings
│       ├── database.py                # SQLAlchemy async engine + session
│       ├── models/                    # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── customer.py
│       │   ├── account.py
│       │   ├── transaction.py
│       │   ├── ledger.py
│       │   ├── ach.py
│       │   ├── wire.py
│       │   ├── compliance.py
│       │   ├── auth.py
│       │   ├── audit.py
│       │   └── graph.py
│       ├── schemas/                   # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── customer.py
│       │   ├── account.py
│       │   ├── transaction.py
│       │   ├── ledger.py
│       │   ├── ach.py
│       │   ├── wire.py
│       │   ├── compliance.py
│       │   └── auth.py
│       ├── api/                       # FastAPI routers
│       │   ├── __init__.py
│       │   ├── deps.py                # Shared dependencies (auth, db session)
│       │   ├── customers.py
│       │   ├── accounts.py
│       │   ├── transactions.py
│       │   ├── ledger.py
│       │   ├── ach.py
│       │   ├── wire.py
│       │   ├── compliance.py
│       │   ├── auth.py
│       │   └── health.py
│       ├── services/                  # Business logic layer
│       │   ├── __init__.py
│       │   ├── customer_service.py
│       │   ├── account_service.py
│       │   ├── transaction_service.py
│       │   ├── ledger_service.py
│       │   ├── ach_service.py
│       │   ├── wire_service.py
│       │   ├── interest_service.py
│       │   ├── compliance_service.py
│       │   └── graph_service.py
│       ├── ai/                        # AI/ML pipeline
│       │   ├── __init__.py
│       │   ├── aml_scorer.py
│       │   ├── fraud_detector.py
│       │   ├── overdraft_predictor.py
│       │   └── regulatory_monitor.py
│       ├── tasks/                     # Celery async tasks
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── ach_tasks.py
│       │   ├── interest_tasks.py
│       │   ├── compliance_tasks.py
│       │   └── graph_tasks.py
│       ├── events/                    # Event publishing and handling
│       │   ├── __init__.py
│       │   ├── publisher.py
│       │   └── handlers.py
│       └── utils/
│           ├── __init__.py
│           ├── encryption.py          # PII encryption/decryption
│           ├── ach_file.py            # NACHA file generation/parsing
│           ├── iso20022.py            # ISO 20022 XML generation
│           └── audit.py               # Audit log helper
├── tests/
│   ├── conftest.py                    # Fixtures, test DB, factories
│   ├── factories/                     # Factory Boy factories
│   │   ├── __init__.py
│   │   ├── customer_factory.py
│   │   ├── account_factory.py
│   │   └── transaction_factory.py
│   ├── unit/
│   │   ├── test_ledger_service.py
│   │   ├── test_ach_file.py
│   │   ├── test_encryption.py
│   │   ├── test_interest_calc.py
│   │   └── ...
│   ├── integration/
│   │   ├── test_customer_api.py
│   │   ├── test_account_api.py
│   │   ├── test_transaction_api.py
│   │   ├── test_ach_api.py
│   │   └── ...
│   └── fixtures/
│       ├── sample_ach_file.ach
│       ├── sample_pacs008.xml
│       └── chart_of_accounts.json
└── scripts/
    ├── seed_reference_data.py         # Seed currencies, products, GL chart
    └── generate_openapi.py            # Export OpenAPI 3.1 spec
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the project skeleton, build system, database connection, configuration management, and CI toolchain. After this phase, a developer can clone the repo, run `docker-compose up`, connect to PostgreSQL, and execute the test suite against an empty database. This phase produces no banking features but ensures every subsequent phase builds on a stable, tested foundation.

### Tasks

#### 1.1 — Project Initialization and Build System

**What**: Create the Python project with uv, configure pyproject.toml, Dockerfile, docker-compose.yml, and development tooling.

**Design**:

```toml
# pyproject.toml
[project]
name = "banking-core-system"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "pydantic>=2.10.0",
    "pydantic-settings>=2.6.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "httpx>=0.28.0",
    "celery[redis]>=5.4.0",
    "redis>=5.2.0",
    "cryptography>=44.0.0",
    "lxml>=5.3.0",
    "jinja2>=3.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "factory-boy>=3.3.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "sqlalchemy[mypy]>=2.0.36",
]

[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "S", "B", "A", "C4", "SIM", "TCH"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
EXPOSE 8000
CMD ["uvicorn", "banking_core.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://banking:banking@db:5432/banking_core
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: dev-secret-key-change-in-production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: banking_core
      POSTGRES_USER: banking
      POSTGRES_PASSWORD: banking
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U banking -d banking_core"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  worker:
    build: .
    command: celery -A banking_core.tasks.celery_app worker --loglevel=info
    environment:
      DATABASE_URL: postgresql+asyncpg://banking:banking@db:5432/banking_core
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

volumes:
  pgdata:
```

**Testing**:
- Unit: `pyproject.toml` is valid TOML and installable with `uv sync`
- Unit: `Dockerfile` builds without errors
- Integration: `docker-compose up` starts all four services (api, db, redis, worker)
- Integration: API responds to `GET /health` with `{"status": "ok"}`

#### 1.2 — Configuration Management

**What**: Create a Pydantic Settings class that loads configuration from environment variables with sensible defaults.

**Design**:

```python
# src/banking_core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Database
    database_url: str = "postgresql+asyncpg://banking:banking@localhost:5432/banking_core"
    database_echo: bool = False
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Security
    secret_key: str  # Required — no default
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    password_min_length: int = 12
    mfa_issuer: str = "BankingCore"

    # Encryption (for PII fields)
    encryption_key: str  # Required — Fernet key for PII encryption

    # Institution
    institution_name: str = "Community Bank"
    institution_routing_number: str = "000000000"
    institution_ein: str = "00-0000000"

    # AI/ML
    anthropic_api_key: str = ""
    ai_aml_model: str = "claude-sonnet-4-20250514"
    ai_aml_confidence_threshold: float = 0.75

    # ACH
    ach_immediate_destination: str = ""
    ach_immediate_origin: str = ""
    ach_originator_status_code: str = "1"

    # Compliance
    ctr_threshold: float = 10000.00
    aml_batch_interval_minutes: int = 15

    model_config = {"env_prefix": "BANKING_", "env_file": ".env"}

settings = Settings()
```

**Testing**:
- Unit: Settings loads with all required env vars set -> correct attribute values
- Unit: Missing `secret_key` -> `ValidationError` with field name
- Unit: Missing `encryption_key` -> `ValidationError`
- Unit: Non-numeric `database_pool_size` -> `ValidationError`
- Unit: Default values are applied when optional vars are absent

#### 1.3 — Database Connection and Session Management

**What**: Set up SQLAlchemy async engine, session factory, and FastAPI dependency for database sessions.

**Design**:

```python
# src/banking_core/database.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from banking_core.config import settings

engine = create_async_engine(
    settings.database_url,
    echo=settings.database_echo,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
)

async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

```python
# src/banking_core/models/__init__.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import MetaData
import uuid
from datetime import datetime

# Naming convention for constraints (Alembic auto-generation)
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=datetime.utcnow, onupdate=datetime.utcnow)
```

**Testing**:
- Integration: `get_db()` yields a session that can execute `SELECT 1`
- Integration: Session commits on success, rolls back on exception
- Unit: `Base.metadata.naming_convention` matches expected patterns

#### 1.4 — FastAPI Application Factory and Health Check

**What**: Create the FastAPI application with middleware, exception handlers, and a health check endpoint.

**Design**:

```python
# src/banking_core/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from banking_core.config import settings
from banking_core.api import health, auth, customers, accounts, transactions, ledger, ach, wire, compliance

def create_app() -> FastAPI:
    app = FastAPI(
        title="Banking Core System",
        description="AI-native core banking platform for community banks and credit unions",
        version="0.1.0",
        docs_url="/docs",
        openapi_url="/openapi.json",
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],  # Restrict in production
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    app.include_router(health.router, tags=["health"])
    return app

app = create_app()
```

```python
# src/banking_core/api/health.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
from banking_core.database import get_db

router = APIRouter()

@router.get("/health")
async def health_check(db: AsyncSession = Depends(get_db)) -> dict:
    await db.execute(text("SELECT 1"))
    return {"status": "ok", "database": "connected"}
```

**Testing**:
- Integration: `GET /health` returns 200 with `{"status": "ok", "database": "connected"}`
- Integration: `GET /docs` returns Swagger UI HTML
- Integration: `GET /openapi.json` returns valid OpenAPI 3.1 spec
- Unit: `create_app()` returns a FastAPI instance with expected title and version

#### 1.5 — Alembic Migration Infrastructure

**What**: Configure Alembic for async SQLAlchemy migrations with the naming convention from the Base model.

**Design**:

```python
# alembic/env.py
from banking_core.models import Base
target_metadata = Base.metadata
# Configure async migration runner using asyncpg
```

```ini
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = postgresql+asyncpg://banking:banking@localhost:5432/banking_core
```

Alembic must be configured to auto-detect model changes and generate migration scripts.

**Testing**:
- Integration: `alembic revision --autogenerate -m "initial"` produces a migration file
- Integration: `alembic upgrade head` applies migrations without error
- Integration: `alembic downgrade -1` reverts the latest migration

#### 1.6 — PII Encryption Utility

**What**: Create a Fernet-based encryption utility for sensitive fields (tax IDs, identification numbers) per PCI DSS v4.0 and FFIEC requirements.

**Design**:

```python
# src/banking_core/utils/encryption.py
from cryptography.fernet import Fernet
from banking_core.config import settings

class PIIEncryptor:
    def __init__(self, key: str):
        self._fernet = Fernet(key.encode())

    def encrypt(self, plaintext: str) -> bytes:
        return self._fernet.encrypt(plaintext.encode())

    def decrypt(self, ciphertext: bytes) -> str:
        return self._fernet.decrypt(ciphertext).decode()

    @staticmethod
    def generate_key() -> str:
        return Fernet.generate_key().decode()

    @staticmethod
    def last_four(value: str) -> str:
        return value[-4:] if len(value) >= 4 else value

encryptor = PIIEncryptor(settings.encryption_key)
```

**Testing**:
- Unit: `encrypt("123-45-6789")` -> bytes that are not the plaintext
- Unit: `decrypt(encrypt("123-45-6789"))` -> `"123-45-6789"`
- Unit: `last_four("123-45-6789")` -> `"6789"`
- Unit: `last_four("12")` -> `"12"`
- Unit: Invalid key -> raises `ValueError`
- Unit: Corrupted ciphertext -> raises `InvalidToken`

---

## Phase 2: Core Data Models and Customer Management

### Purpose
Implement the foundational database models (customer, account, product, branch, RBAC) and the customer information management (CIF) API. After this phase, the system can onboard customers with KYC data, manage addresses and contact information, and enforce role-based access control. This is the prerequisite for all account and transaction operations.

### Tasks

#### 2.1 — Reference Data Models and Seed Script

**What**: Create SQLAlchemy models for reference data (product, branch, currency, jurisdiction) and a seed script that populates them.

**Design**:

```python
# src/banking_core/models/account.py (partial — reference tables)
class Product(Base, TimestampMixin):
    __tablename__ = "product"

    product_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    product_code: Mapped[str] = mapped_column(String(30), unique=True)
    product_type: Mapped[str] = mapped_column(String(20))  # 'dda', 'savings', 'money_market', 'cd', 'consumer_loan', 'loc'
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)
    currency_code: Mapped[str] = mapped_column(String(3), default="USD")
    interest_rate_type: Mapped[str | None] = mapped_column(String(20))
    default_interest_rate: Mapped[Decimal | None] = mapped_column(Numeric(8, 5))
    min_balance: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    max_balance: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    is_active: Mapped[bool] = mapped_column(default=True)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    property_schema: Mapped[dict | None] = mapped_column(JSONB)

class Branch(Base, TimestampMixin):
    __tablename__ = "branch"

    branch_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    branch_code: Mapped[str] = mapped_column(String(10), unique=True)
    name: Mapped[str] = mapped_column(String(200))
    routing_number: Mapped[str] = mapped_column(String(9))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    is_active: Mapped[bool] = mapped_column(default=True)
```

Seed script populates:
- Currency: USD (minor_units=2)
- Products: CONSUMER_CHECKING, CONSUMER_SAVINGS, MONEY_MARKET, CD_12M, CD_24M, CONSUMER_INSTALLMENT, CONSUMER_LOC
- Default branch: MAIN with the institution's routing number
- Jurisdictions: US states (50 + DC + territories)

**Testing**:
- Integration: Seed script runs idempotently (safe to run twice)
- Integration: All seeded products are queryable by product_code
- Unit: Product model validates product_type enum values

#### 2.2 — Customer ORM Models

**What**: Create SQLAlchemy models for customer, customer_address, customer_identification, and customer_relationship, matching the Hybrid Relational + JSONB schema (Data Model 3).

**Design**:

```python
# src/banking_core/models/customer.py
class Customer(Base, TimestampMixin):
    __tablename__ = "customer"

    customer_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    customer_number: Mapped[str] = mapped_column(String(20), unique=True)
    customer_type: Mapped[str] = mapped_column(String(20))  # 'individual', 'business', 'trust', 'estate'
    legal_name: Mapped[str] = mapped_column(String(300))
    display_name: Mapped[str | None] = mapped_column(String(200))
    tax_id_type: Mapped[str | None] = mapped_column(String(10))  # 'SSN', 'EIN', 'ITIN'
    tax_id_encrypted: Mapped[bytes | None] = mapped_column(LargeBinary)
    tax_id_last_four: Mapped[str | None] = mapped_column(String(4))
    date_of_birth: Mapped[date | None]
    kyc_status: Mapped[str] = mapped_column(String(20), default="pending")
    kyc_verified_at: Mapped[datetime | None]
    cip_verified: Mapped[bool] = mapped_column(default=False)
    risk_rating: Mapped[str] = mapped_column(String(10), default="standard")
    # Denormalized primary address (ISO 20022 structured)
    primary_street: Mapped[str | None] = mapped_column(String(200))
    primary_building_number: Mapped[str | None] = mapped_column(String(20))
    primary_city: Mapped[str | None] = mapped_column(String(100))
    primary_state: Mapped[str | None] = mapped_column(String(100))
    primary_postal_code: Mapped[str | None] = mapped_column(String(20))
    primary_country: Mapped[str | None] = mapped_column(String(2))
    primary_email: Mapped[str | None] = mapped_column(String(200))
    primary_phone: Mapped[str | None] = mapped_column(String(30))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    is_active: Mapped[bool] = mapped_column(default=True)

    # Relationships
    addresses: Mapped[list["CustomerAddress"]] = relationship(back_populates="customer")
    identifications: Mapped[list["CustomerIdentification"]] = relationship(back_populates="customer")
    account_ownerships: Mapped[list["AccountOwner"]] = relationship(back_populates="customer")

class CustomerAddress(Base):
    __tablename__ = "customer_address"

    address_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    customer_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("customer.customer_id"))
    address_type: Mapped[str] = mapped_column(String(20))
    street_name: Mapped[str | None] = mapped_column(String(200))
    building_number: Mapped[str | None] = mapped_column(String(20))
    city: Mapped[str] = mapped_column(String(100))
    state_province: Mapped[str | None] = mapped_column(String(100))
    postal_code: Mapped[str | None] = mapped_column(String(20))
    country_code: Mapped[str] = mapped_column(String(2))
    effective_from: Mapped[date] = mapped_column(default=date.today)
    effective_to: Mapped[date | None]
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    customer: Mapped["Customer"] = relationship(back_populates="addresses")

class CustomerIdentification(Base):
    __tablename__ = "customer_identification"

    identification_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    customer_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("customer.customer_id"))
    id_type: Mapped[str] = mapped_column(String(30))
    id_number_encrypted: Mapped[bytes | None] = mapped_column(LargeBinary)
    issuing_authority: Mapped[str | None] = mapped_column(String(100))
    expiry_date: Mapped[date | None]
    verified_at: Mapped[datetime | None]
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    customer: Mapped["Customer"] = relationship(back_populates="identifications")

class CustomerRelationship(Base):
    __tablename__ = "customer_relationship"

    relationship_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    customer_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("customer.customer_id"))
    related_customer_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("customer.customer_id"))
    relationship_type: Mapped[str] = mapped_column(String(30))
    effective_from: Mapped[date] = mapped_column(default=date.today)
    effective_to: Mapped[date | None]
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

**Testing**:
- Integration: Create a customer -> persisted with generated UUID and customer_number
- Integration: Add addresses and identifications -> retrievable via relationship
- Unit: Customer model rejects invalid kyc_status values at the schema layer
- Unit: Tax ID encryption -> tax_id_encrypted is bytes, tax_id_last_four is last 4 digits

#### 2.3 — RBAC Models and Auth Service

**What**: Create the app_user, role, permission, role_permission, and user_role models. Implement password hashing, JWT token generation/validation, and RBAC dependency injection.

**Design**:

```python
# src/banking_core/models/auth.py
class AppUser(Base, TimestampMixin):
    __tablename__ = "app_user"

    user_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(200), unique=True)
    password_hash: Mapped[str | None] = mapped_column(String(200))
    full_name: Mapped[str] = mapped_column(String(200))
    employee_id: Mapped[str | None] = mapped_column(String(20))
    branch_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("branch.branch_id"))
    is_active: Mapped[bool] = mapped_column(default=True)
    last_login_at: Mapped[datetime | None]
    mfa_enabled: Mapped[bool] = mapped_column(default=False)

    roles: Mapped[list["UserRole"]] = relationship()

class Role(Base):
    __tablename__ = "role"
    role_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    role_name: Mapped[str] = mapped_column(String(50), unique=True)
    description: Mapped[str | None] = mapped_column(String(300))

class Permission(Base):
    __tablename__ = "permission"
    permission_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    permission_code: Mapped[str] = mapped_column(String(50), unique=True)
    description: Mapped[str | None] = mapped_column(String(300))

class RolePermission(Base):
    __tablename__ = "role_permission"
    role_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("role.role_id"), primary_key=True)
    permission_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("permission.permission_id"), primary_key=True)

class UserRole(Base):
    __tablename__ = "user_role"
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("app_user.user_id"), primary_key=True)
    role_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("role.role_id"), primary_key=True)
    branch_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("branch.branch_id"))
```

Auth dependency:
```python
# src/banking_core/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> AppUser:
    """Decode JWT, load user, verify active status."""
    ...

def require_permission(permission_code: str):
    """Returns a dependency that checks the current user has the given permission."""
    async def checker(user: AppUser = Depends(get_current_user)):
        user_permissions = await _load_user_permissions(user)
        if permission_code not in user_permissions:
            raise HTTPException(status_code=403, detail=f"Missing permission: {permission_code}")
        return user
    return checker
```

Seeded roles and permissions:
- Roles: `admin`, `branch_manager`, `teller`, `loan_officer`, `compliance_officer`
- Permissions: `customer.read`, `customer.create`, `customer.update`, `account.read`, `account.create`, `account.close`, `transaction.read`, `transaction.post`, `ach.originate`, `wire.initiate`, `aml.review`, `sar.file`, `gl.read`, `gl.post`, `admin.manage_users`

**Testing**:
- Unit: Password hashing -> hash differs from plaintext, verify returns True
- Unit: JWT creation -> valid token with user_id, exp, and permissions claims
- Unit: JWT with expired token -> raises 401
- Integration: `POST /auth/token` with valid credentials -> returns access_token
- Integration: `POST /auth/token` with wrong password -> 401
- Integration: Request with valid token but missing permission -> 403
- Integration: Request with valid token and correct permission -> 200

#### 2.4 — Customer API Endpoints

**What**: Implement CRUD endpoints for customer management with KYC workflow support.

**Design**:

```python
# src/banking_core/api/customers.py
router = APIRouter(prefix="/customers", tags=["customers"])

# POST /customers
# Request: CustomerCreate schema
# Response: 201, CustomerResponse
# Permission: customer.create

# GET /customers
# Query params: customer_type, kyc_status, risk_rating, search (name), page, size
# Response: 200, PaginatedResponse[CustomerSummary]
# Permission: customer.read

# GET /customers/{customer_id}
# Response: 200, CustomerDetailResponse (includes addresses, identifications)
# Permission: customer.read

# PATCH /customers/{customer_id}
# Request: CustomerUpdate schema
# Response: 200, CustomerResponse
# Permission: customer.update

# POST /customers/{customer_id}/addresses
# Request: AddressCreate schema
# Response: 201, AddressResponse
# Permission: customer.update

# POST /customers/{customer_id}/identifications
# Request: IdentificationCreate schema (id_number is encrypted before storage)
# Response: 201, IdentificationResponse
# Permission: customer.update

# POST /customers/{customer_id}/kyc/verify
# Request: KYCVerification schema (verification_method, risk_rating)
# Response: 200, CustomerResponse (kyc_status updated to 'verified')
# Permission: customer.update
```

Pydantic schemas:
```python
# src/banking_core/schemas/customer.py
class CustomerCreate(BaseModel):
    customer_type: Literal["individual", "business", "trust", "estate"]
    legal_name: str = Field(max_length=300)
    display_name: str | None = Field(None, max_length=200)
    tax_id: str | None = None  # Will be encrypted before storage
    tax_id_type: Literal["SSN", "EIN", "ITIN"] | None = None
    date_of_birth: date | None = None
    primary_address: AddressCreate | None = None
    primary_email: EmailStr | None = None
    primary_phone: str | None = None
    properties: dict = Field(default_factory=dict)

class CustomerResponse(BaseModel):
    customer_id: uuid.UUID
    customer_number: str
    customer_type: str
    legal_name: str
    display_name: str | None
    tax_id_last_four: str | None  # Never return full tax ID
    kyc_status: str
    risk_rating: str
    is_active: bool
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

Customer number generation: auto-increment sequence formatted as `C` + zero-padded 9 digits (e.g., `C000000001`).

**Testing**:
- Integration: `POST /customers` with valid individual data -> 201, customer_number assigned
- Integration: `POST /customers` with business data including EIN -> 201, tax_id_last_four populated
- Integration: `GET /customers?kyc_status=pending` -> returns only pending customers
- Integration: `GET /customers/{id}` -> includes addresses and identifications
- Integration: `PATCH /customers/{id}` with name change -> 200, audit log entry created
- Integration: `POST /customers/{id}/kyc/verify` -> kyc_status changes to `verified`, kyc_verified_at set
- Integration: `POST /customers` without auth token -> 401
- Integration: `POST /customers` with teller role (has customer.create) -> 201
- Unit: CustomerCreate rejects missing legal_name -> ValidationError
- Unit: CustomerResponse never includes tax_id_encrypted field
- Unit: Tax ID "123-45-6789" -> encrypted in DB, last_four = "6789"

#### 2.5 — Audit Log Infrastructure

**What**: Create the audit_log model and a service that automatically records changes to auditable entities (FFIEC-aligned).

**Design**:

```python
# src/banking_core/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"

    audit_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    event_type: Mapped[str] = mapped_column(String(50))
    entity_type: Mapped[str] = mapped_column(String(30))
    entity_id: Mapped[uuid.UUID]
    action: Mapped[str] = mapped_column(String(10))  # 'create', 'read', 'update', 'delete'
    old_values: Mapped[dict | None] = mapped_column(JSONB)
    new_values: Mapped[dict | None] = mapped_column(JSONB)
    user_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("app_user.user_id"))
    ip_address: Mapped[str | None] = mapped_column(String(45))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

class AccessLog(Base):
    __tablename__ = "access_log"

    access_log_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("app_user.user_id"))
    event_type: Mapped[str] = mapped_column(String(20))  # 'login', 'logout', 'login_failed'
    ip_address: Mapped[str | None] = mapped_column(String(45))
    user_agent: Mapped[str | None] = mapped_column(String(500))
    success: Mapped[bool]
    failure_reason: Mapped[str | None] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

```python
# src/banking_core/utils/audit.py
async def record_audit(
    db: AsyncSession,
    event_type: str,
    entity_type: str,
    entity_id: uuid.UUID,
    action: str,
    user_id: uuid.UUID | None = None,
    old_values: dict | None = None,
    new_values: dict | None = None,
    ip_address: str | None = None,
) -> None:
    """Insert an audit log entry. Called by service layer after mutations."""
    ...
```

**Testing**:
- Integration: Customer create -> audit_log entry with action='create', new_values containing customer data
- Integration: Customer update -> audit_log entry with old_values and new_values showing diff
- Integration: Login success -> access_log entry with success=True
- Integration: Login failure -> access_log entry with success=False and failure_reason
- Unit: Audit log does not include encrypted PII fields (tax_id_encrypted excluded)

---

## Phase 3: Account Management and General Ledger

### Purpose
Implement the account lifecycle (open, transact, close) for deposit products (DDA, savings, money market, CD) and the double-entry general ledger. After this phase, the system can open accounts linked to customers, record transactions that update balances, and maintain a balanced GL. This is the accounting backbone of the core.

### Tasks

#### 3.1 — Account ORM Models

**What**: Create SQLAlchemy models for account, account_owner, and loan_schedule, following the Hybrid Relational + JSONB pattern.

**Design**:

Models follow the schema from Data Model Suggestion 3 (account with JSONB `properties`, account_owner junction table, loan_schedule for amortization). The account model includes typed columns for universally-needed fields (`current_balance`, `available_balance`, `hold_amount`, `interest_rate`, `interest_accrued`, `account_status`) and a JSONB `properties` column for product-specific fields.

Account statuses: `pending` -> `active` -> `dormant` | `frozen` | `closed`. Transitions are enforced at the service layer.

**Testing**:
- Integration: Create an account linked to a product and customer -> persisted with status 'pending'
- Unit: Account model validates account_type against allowed values
- Unit: JSONB properties validated against product.property_schema at service layer

#### 3.2 — General Ledger Models and Chart of Accounts

**What**: Create GL models (gl_account, gl_journal, gl_journal_entry, gl_period) and seed a standard community bank chart of accounts.

**Design**:

```python
# src/banking_core/models/ledger.py
class GLAccount(Base, TimestampMixin):
    __tablename__ = "gl_account"

    gl_account_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    account_number: Mapped[str] = mapped_column(String(20), unique=True)
    name: Mapped[str] = mapped_column(String(200))
    account_type: Mapped[str] = mapped_column(String(20))  # 'asset', 'liability', 'equity', 'revenue', 'expense'
    account_subtype: Mapped[str | None] = mapped_column(String(30))
    parent_account_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("gl_account.gl_account_id"))
    normal_balance: Mapped[str] = mapped_column(String(6))  # 'debit', 'credit'
    is_active: Mapped[bool] = mapped_column(default=True)

class GLJournal(Base, TimestampMixin):
    __tablename__ = "gl_journal"

    journal_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    journal_number: Mapped[str] = mapped_column(String(30), unique=True)
    journal_date: Mapped[date]
    description: Mapped[str | None] = mapped_column(String(500))
    source: Mapped[str | None] = mapped_column(String(30))
    source_reference_id: Mapped[uuid.UUID | None]
    status: Mapped[str] = mapped_column(String(20), default="posted")
    posted_by: Mapped[uuid.UUID | None]
    posted_at: Mapped[datetime | None]

    entries: Mapped[list["GLJournalEntry"]] = relationship(back_populates="journal")

class GLJournalEntry(Base):
    __tablename__ = "gl_journal_entry"

    entry_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    journal_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("gl_journal.journal_id"))
    gl_account_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("gl_account.gl_account_id"))
    debit_amount: Mapped[Decimal] = mapped_column(Numeric(18, 2), default=0)
    credit_amount: Mapped[Decimal] = mapped_column(Numeric(18, 2), default=0)
    description: Mapped[str | None] = mapped_column(String(300))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    journal: Mapped["GLJournal"] = relationship(back_populates="entries")

    # CHECK constraint: exactly one of debit or credit must be positive
    __table_args__ = (
        CheckConstraint(
            "(debit_amount > 0 AND credit_amount = 0) OR (credit_amount > 0 AND debit_amount = 0)",
            name="chk_debit_or_credit",
        ),
    )
```

Chart of accounts seed (community bank standard):
- 1000-1999: Assets (Cash, Loans Receivable, Fixed Assets)
- 2000-2999: Liabilities (Deposits - DDA, Savings, CD, Money Market, Borrowings)
- 3000-3999: Equity (Retained Earnings, Capital)
- 4000-4999: Revenue (Interest Income - Loans, Fee Income)
- 5000-5999: Expenses (Interest Expense - Deposits, Operating Expenses, Provision for Loan Losses)

**Testing**:
- Integration: Seed chart of accounts -> all GL accounts queryable by account_number
- Unit: GLJournalEntry rejects entry where both debit and credit are > 0
- Unit: GLJournalEntry rejects entry where both debit and credit are 0

#### 3.3 — Ledger Service (Double-Entry Posting)

**What**: Implement a service that creates balanced journal entries for every financial event, enforcing the fundamental accounting equation.

**Design**:

```python
# src/banking_core/services/ledger_service.py
class LedgerService:
    async def post_journal(
        self,
        db: AsyncSession,
        description: str,
        entries: list[JournalEntryInput],
        source: str | None = None,
        source_reference_id: uuid.UUID | None = None,
        posted_by: uuid.UUID | None = None,
    ) -> GLJournal:
        """
        Create a balanced journal entry.
        Raises UnbalancedJournalError if sum(debits) != sum(credits).
        """
        total_debit = sum(e.debit_amount for e in entries)
        total_credit = sum(e.credit_amount for e in entries)
        if total_debit != total_credit:
            raise UnbalancedJournalError(total_debit, total_credit)
        # Create journal + entries in a single transaction
        ...

    async def get_account_balance(
        self,
        db: AsyncSession,
        gl_account_number: str,
        as_of_date: date | None = None,
    ) -> Decimal:
        """Sum debits and credits for a GL account up to as_of_date."""
        ...

    async def trial_balance(
        self,
        db: AsyncSession,
        as_of_date: date | None = None,
    ) -> list[TrialBalanceRow]:
        """Generate trial balance: for each GL account, show debit/credit totals."""
        ...

    async def close_period(
        self,
        db: AsyncSession,
        period_id: uuid.UUID,
        posted_by: uuid.UUID,
    ) -> GLPeriod:
        """Close a GL period: verify trial balance, prevent future posting to closed periods."""
        ...
```

```python
class JournalEntryInput(BaseModel):
    gl_account_number: str
    debit_amount: Decimal = Decimal("0")
    credit_amount: Decimal = Decimal("0")
    description: str | None = None
```

**Testing**:
- Unit: Balanced journal (debit $1000, credit $1000) -> journal created with 2 entries
- Unit: Unbalanced journal (debit $1000, credit $999) -> UnbalancedJournalError
- Unit: Trial balance after 3 journals -> all debits equal all credits
- Integration: Close period -> period status changes to 'closed', posting to closed period raises error
- Integration: get_account_balance with as_of_date -> excludes journals after that date

#### 3.4 — Account Service and API

**What**: Implement account lifecycle operations (open, deposit, withdraw, transfer, close) with GL integration.

**Design**:

```python
# src/banking_core/services/account_service.py
class AccountService:
    async def open_account(
        self, db: AsyncSession, customer_id: uuid.UUID, product_code: str,
        initial_deposit: Decimal = Decimal("0"), properties: dict | None = None,
    ) -> Account:
        """
        1. Validate product exists and is active
        2. Validate customer exists and KYC is verified
        3. Generate account number
        4. Create account with status='active' (or 'pending' if no initial deposit)
        5. If initial_deposit > 0: post GL journal (debit Cash, credit Deposits)
        6. Create account_owner record
        7. Record audit log
        """
        ...

    async def close_account(
        self, db: AsyncSession, account_id: uuid.UUID, reason: str,
    ) -> Account:
        """
        1. Verify balance is zero
        2. Verify no pending holds
        3. Set status='closed', closed_date=today
        4. Record audit log
        """
        ...
```

API endpoints:
```
POST   /accounts                          # Open account (permission: account.create)
GET    /accounts                          # List accounts (filter by customer, type, status)
GET    /accounts/{account_id}             # Account details
GET    /accounts/{account_id}/owners      # List account owners
POST   /accounts/{account_id}/owners      # Add account owner
PATCH  /accounts/{account_id}/status      # Change status (freeze, close, reactivate)
GET    /accounts/{account_id}/balance     # Current and available balance
```

Account number generation: product-type prefix + branch code + sequence (e.g., `10-001-0000001` for DDA at branch 001).

**Testing**:
- Integration: Open DDA with $500 deposit -> account active, balance=$500, GL entry posted
- Integration: Open CD with $10,000 -> account active, properties contain term_months
- Integration: Close account with zero balance -> status='closed'
- Integration: Close account with non-zero balance -> 400 error
- Integration: Open account for customer with kyc_status='pending' -> 400 error
- Unit: Account number generation follows expected format
- Unit: Product property_schema validation rejects invalid JSONB properties

#### 3.5 — Transaction Service and API

**What**: Implement transaction posting (deposit, withdrawal, internal transfer) with balance updates and GL integration.

**Design**:

```python
# src/banking_core/services/transaction_service.py
class TransactionService:
    async def post_deposit(
        self, db: AsyncSession, account_id: uuid.UUID, amount: Decimal,
        channel: str, reference_number: str | None = None,
        description: str | None = None, details: dict | None = None,
    ) -> Transaction:
        """
        1. Validate account is active
        2. Create transaction with direction='credit', compute running_balance
        3. Update account.current_balance and available_balance
        4. Post GL journal: debit Cash/Due-From, credit Customer Deposits
        5. Emit event for downstream consumers
        """
        ...

    async def post_withdrawal(
        self, db: AsyncSession, account_id: uuid.UUID, amount: Decimal,
        channel: str, **kwargs,
    ) -> Transaction:
        """
        1. Validate account is active
        2. Check available_balance >= amount (or overdraft_limit if enabled)
        3. Create transaction with direction='debit', compute running_balance
        4. Update account balances
        5. Post GL journal: debit Customer Deposits, credit Cash
        """
        ...

    async def post_transfer(
        self, db: AsyncSession, from_account_id: uuid.UUID,
        to_account_id: uuid.UUID, amount: Decimal, description: str | None = None,
    ) -> tuple[Transaction, Transaction]:
        """
        1. Post withdrawal from source account
        2. Post deposit to target account
        3. Link transactions via reference_number
        4. Single GL journal: debit source deposit liability, credit target deposit liability
        """
        ...
```

API endpoints:
```
POST   /accounts/{account_id}/transactions          # Post a transaction
GET    /accounts/{account_id}/transactions           # Transaction history (paginated, date range)
GET    /accounts/{account_id}/transactions/{txn_id}  # Transaction detail
POST   /transactions/transfer                        # Internal transfer between accounts
```

Transaction model follows Data Model 3 with JSONB `details` for channel-specific data.

**Testing**:
- Integration: Deposit $500 -> balance increases by $500, GL balanced, running_balance correct
- Integration: Withdraw $200 -> balance decreases, GL balanced
- Integration: Withdraw $10,000 from account with $500 balance and no overdraft -> 400 insufficient funds
- Integration: Withdraw $600 from account with $500 balance and $200 overdraft limit -> success, balance = -$100
- Integration: Transfer $100 between two accounts -> source decreases, target increases, two transactions linked
- Integration: Transaction history with date range filter -> returns correct subset
- Unit: Running balance calculation: after deposit $500 and withdrawal $200, running balance is $300

---

## Phase 4: ACH Payment Processing

### Purpose
Implement NACHA-compliant ACH payment processing for origination and receipt. After this phase, the system can generate ACH files for outbound payments (e.g., payroll credits), receive and process inbound ACH entries, handle returns, and enforce the 2026 NACHA fraud monitoring requirements. ACH is the most critical payment rail for community banks.

### Tasks

#### 4.1 — ACH ORM Models

**What**: Create SQLAlchemy models for ach_file, ach_batch, and ach_entry, faithfully mapping the NACHA record types.

**Design**:

Models follow the Data Model 3 ACH schema. Field widths and types match NACHA record layouts exactly (e.g., company_name VARCHAR(16), standard_entry_class CHAR(3), transaction_code CHAR(2)).

Standard Entry Class codes supported: PPD (prearranged payment/deposit), CCD (cash concentration/disbursement), WEB (internet-initiated), TEL (telephone-initiated).

NACHA transaction codes: 22 (checking credit), 27 (checking debit), 32 (savings credit), 37 (savings debit).

**Testing**:
- Unit: ACH entry model enforces transaction_code is 2 characters
- Unit: ACH batch model enforces standard_entry_class is 3 characters
- Integration: Create ach_file -> ach_batch -> ach_entries -> all linked by foreign keys

#### 4.2 — ACH File Generation (NACHA Format)

**What**: Implement NACHA file generation from ach_file/batch/entry records, producing a compliant fixed-width text file.

**Design**:

```python
# src/banking_core/utils/ach_file.py
class NACHAFileGenerator:
    """Generates NACHA-compliant ACH files from database records."""

    def generate(self, ach_file: ACHFile, batches: list[ACHBatch], entries: dict[uuid.UUID, list[ACHEntry]]) -> str:
        """
        Produces a fixed-width text file with:
        - File Header Record (record type 1, 94 chars)
        - For each batch:
          - Batch Header Record (record type 5, 94 chars)
          - Entry Detail Records (record type 6, 94 chars)
          - Batch Control Record (record type 8, 94 chars)
        - File Control Record (record type 9, 94 chars)
        - Padding lines of 9s to fill to a multiple of 10 lines
        """
        ...

    def _format_file_header(self, ach_file: ACHFile) -> str:
        """Record type 1: priority_code(2) + immediate_dest(10) + immediate_origin(10) + ..."""
        ...

    def _format_entry_detail(self, entry: ACHEntry) -> str:
        """Record type 6: transaction_code(2) + receiving_dfi(8) + check_digit(1) + dfi_account(17) + amount(10) + ..."""
        ...
```

NACHA 2026 compliance:
- `company_entry_description` field: support standardised descriptors `PAYROLL` and `PURCHASE` (new 2026 requirement)
- Entry hash calculation: sum of first 8 digits of all receiving DFI routing numbers, mod 10^10

**Testing**:
- Unit: Generate file with 1 batch, 2 entries -> valid 94-character fixed-width lines
- Unit: File control record totals match sum of batch totals
- Unit: Entry hash is correctly calculated
- Unit: File is padded to multiple of 10 lines with 9-filled records
- Fixture: Compare generated file against `tests/fixtures/sample_ach_file.ach` reference file

#### 4.3 — ACH File Parsing (Inbound)

**What**: Parse inbound NACHA files received from the Federal Reserve or correspondent bank into ach_file/batch/entry records.

**Design**:

```python
class NACHAFileParser:
    def parse(self, file_content: str) -> ParsedACHFile:
        """
        Parse a NACHA fixed-width file and return structured data.
        Validates:
        - Record type indicators
        - Field lengths and positions
        - Batch totals match entry sums
        - File control totals match batch sums
        - Entry hash verification
        Raises NACHAParseError with line number and field for any violation.
        """
        ...
```

**Testing**:
- Unit: Parse valid reference file -> correct number of batches and entries
- Unit: Parse file with mismatched batch totals -> NACHAParseError with specific field
- Unit: Parse file with invalid record type -> NACHAParseError
- Unit: Round-trip: generate -> parse -> compare -> identical data

#### 4.4 — ACH Service and API

**What**: Implement ACH origination (create batches, submit files) and receipt (process inbound entries, post to accounts) workflows.

**Design**:

```python
class ACHService:
    async def create_batch(
        self, db: AsyncSession, standard_entry_class: str,
        company_name: str, company_entry_description: str,
        effective_entry_date: date, entries: list[ACHEntryInput],
    ) -> ACHBatch:
        """Create an outbound ACH batch with entry detail records."""
        ...

    async def generate_file(
        self, db: AsyncSession, batch_ids: list[uuid.UUID],
    ) -> tuple[ACHFile, str]:
        """
        Combine batches into a NACHA file.
        Returns the ACHFile record and the generated file content string.
        """
        ...

    async def process_inbound_file(
        self, db: AsyncSession, file_content: str,
    ) -> ACHFile:
        """
        Parse inbound ACH file and:
        1. Create ach_file, ach_batch, ach_entry records
        2. Match entries to local accounts by DFI account number
        3. Post transactions to matched accounts
        4. Post GL journals for each entry
        5. Flag unmatched entries for manual review
        """
        ...

    async def process_return(
        self, db: AsyncSession, ach_entry_id: uuid.UUID,
        return_reason_code: str,
    ) -> ACHEntry:
        """
        Process an ACH return:
        1. Update entry status to 'returned', set return_reason_code
        2. Reverse the original transaction
        3. Post reversal GL journal
        4. Track return rate per NACHA 2026 0.5% threshold
        """
        ...
```

API endpoints:
```
POST   /ach/batches                    # Create outbound batch
GET    /ach/batches                    # List batches
GET    /ach/batches/{batch_id}         # Batch details with entries
POST   /ach/files/generate             # Generate NACHA file from batches
POST   /ach/files/inbound              # Upload and process inbound ACH file
GET    /ach/files                      # List ACH files
POST   /ach/entries/{entry_id}/return  # Process a return
GET    /ach/returns/rate               # Current return rate vs 0.5% threshold
```

**Testing**:
- Integration: Create batch with 3 PPD credit entries -> batch created, entries linked
- Integration: Generate file from batch -> valid NACHA file content returned
- Integration: Process inbound file with matching account -> transaction posted, balance updated
- Integration: Process inbound file with no matching account -> entry flagged for review
- Integration: Process return with R01 (insufficient funds) -> original transaction reversed, GL balanced
- Integration: Return rate calculation -> correct percentage when threshold exceeded, warning returned
- Unit: NACHA 2026 company_entry_description 'PAYROLL' accepted
- Unit: Invalid standard_entry_class 'XXX' rejected

---

## Phase 5: BSA/AML Compliance and AI-Driven Monitoring

### Purpose
Implement BSA/AML transaction monitoring with AI-driven alert scoring, CTR auto-generation, and SAR workflow. This is the project's primary AI-native differentiator: AML monitoring operates on the same data layer as the core, eliminating the latency gap that exists when community banks use third-party AML bolt-ons. After this phase, the system generates intelligent AML alerts with explainable AI confidence scores, auto-files CTRs for cash transactions over $10,000, and supports the SAR drafting and filing workflow.

### Tasks

#### 5.1 — Compliance ORM Models

**What**: Create SQLAlchemy models for aml_alert, suspicious_activity_report, and currency_transaction_report.

**Design**:

Models follow Data Model 3 compliance schema. Key fields:
- `aml_alert`: alert_source includes 'ai_model' and 'rule_engine'; ai_confidence_score (0.0000-1.0000); ai_explanation text; triggering_transactions UUID array.
- `suspicious_activity_report`: FinCEN SAR field layout (Parts I-V); filing_status lifecycle (draft -> review -> filed -> amended).
- `currency_transaction_report`: auto-generated when daily cash exceeds $10,000 threshold.

**Testing**:
- Integration: Create AML alert with ai_confidence_score -> persisted, queryable by severity
- Unit: SAR filing_status transitions enforced (cannot go from 'filed' back to 'draft')

#### 5.2 — Rule-Based AML Detection Engine

**What**: Implement configurable rule-based detection for common BSA/AML scenarios as the baseline layer.

**Design**:

```python
# src/banking_core/ai/aml_scorer.py
class AMLRuleEngine:
    """Rule-based AML detection — runs on every transaction batch."""

    async def evaluate_transaction(
        self, db: AsyncSession, transaction: Transaction,
    ) -> list[AMLAlertInput] | None:
        """
        Check transaction against rules:
        1. Structuring: multiple cash deposits/withdrawals below $10,000 within 48 hours
        2. Rapid movement: large deposit followed by wire/ACH out within 24 hours
        3. High-risk geography: transactions involving OFAC-sanctioned countries
        4. Unusual activity: transaction amount > 3x customer's 90-day average
        5. Round-dollar: high frequency of round-dollar amounts
        """
        ...

    rules: list[AMLRule] = [
        StructuringRule(threshold=10000, time_window_hours=48, min_transactions=3),
        RapidMovementRule(time_window_hours=24, amount_threshold=5000),
        UnusualActivityRule(multiplier=3.0, lookback_days=90),
        RoundDollarRule(frequency_threshold=0.8, min_transactions=5),
    ]
```

**Testing**:
- Unit: 3 cash deposits of $9,500 within 48 hours -> structuring alert generated
- Unit: 2 cash deposits of $9,500 within 48 hours -> no alert (below min_transactions)
- Unit: $50,000 deposit followed by $49,000 wire within 24 hours -> rapid movement alert
- Unit: Transaction of $500 when 90-day average is $100 -> unusual activity alert (5x)
- Unit: Transaction of $250 when 90-day average is $100 -> no alert (2.5x < 3x threshold)

#### 5.3 — AI-Powered AML Scoring

**What**: Implement LLM-based AML alert scoring that provides explainable confidence scores and natural-language explanations.

**Design**:

```python
class AIAMLScorer:
    """LLM-enhanced AML scoring — enriches rule-based alerts with AI analysis."""

    async def score_alert(
        self, db: AsyncSession, alert: AMLAlert,
    ) -> AMLScoringResult:
        """
        1. Gather context: customer profile, transaction history (90 days),
           account relationships, prior alerts
        2. Send to Claude with structured prompt:
           - System: "You are a BSA/AML analyst at a US community bank..."
           - User: structured data about the alert, customer, and transactions
           - Response: JSON with confidence_score (0-1), explanation, recommended_action
        3. Update alert with ai_confidence_score and ai_explanation
        4. If confidence >= threshold: auto-escalate to compliance officer
        """
        ...

    def _build_prompt(self, context: AlertContext) -> list[dict]:
        system_prompt = """You are a BSA/AML compliance analyst at a US community bank.
        Analyze the following suspicious activity alert and provide:
        1. A confidence score (0.0 to 1.0) indicating the likelihood of genuine suspicious activity
        2. A plain-English explanation suitable for a compliance officer
        3. A recommended action: 'escalate', 'review', or 'clear'

        Consider: transaction patterns, customer risk profile, account history,
        NACHA 2026 fraud indicators, and FinCEN typologies for structuring,
        rapid movement, and unusual activity.

        Respond in JSON format:
        {"confidence_score": 0.XX, "explanation": "...", "recommended_action": "..."}"""
        ...
```

```python
class AMLScoringResult(BaseModel):
    confidence_score: float = Field(ge=0.0, le=1.0)
    explanation: str
    recommended_action: Literal["escalate", "review", "clear"]
```

**Testing**:
- Integration (mocked LLM): Alert with high-risk pattern -> AI returns confidence > 0.75, alert escalated
- Integration (mocked LLM): Alert with low-risk pattern -> AI returns confidence < 0.5, alert stays for review
- Unit: Prompt builder includes customer risk_rating, transaction amounts, and alert type
- Unit: AI response parsing handles malformed JSON gracefully (falls back to rule score)
- Unit: AMLScoringResult validates confidence_score range [0, 1]

#### 5.4 — CTR Auto-Generation

**What**: Automatically generate Currency Transaction Reports when a customer's cash transactions exceed $10,000 in a calendar day.

**Design**:

```python
class CTRService:
    async def evaluate_daily_cash(
        self, db: AsyncSession, customer_id: uuid.UUID, transaction_date: date,
    ) -> CurrencyTransactionReport | None:
        """
        Sum all cash-in and cash-out transactions for the customer on the given date.
        If total_cash_in >= $10,000 OR total_cash_out >= $10,000:
        1. Create CTR record with transaction_ids
        2. Set filing_status = 'draft'
        3. Notify compliance officer for review before filing
        """
        ...
```

Celery task runs at end-of-day to sweep all customers with cash activity and generate CTRs.

**Testing**:
- Integration: Customer deposits $11,000 cash in one transaction -> CTR generated
- Integration: Customer deposits $6,000 and $5,000 cash -> CTR generated (aggregate $11,000)
- Integration: Customer deposits $6,000 and $3,000 cash -> no CTR ($9,000 < threshold)
- Unit: CTR number generation follows format `CTR-YYYY-NNNNNN`

#### 5.5 — SAR Workflow API

**What**: Implement SAR drafting, review, and filing workflow endpoints.

**Design**:

```
POST   /compliance/alerts                    # List AML alerts (filter by status, severity, date range)
GET    /compliance/alerts/{alert_id}         # Alert detail with triggering transactions
PATCH  /compliance/alerts/{alert_id}         # Update disposition (clear, escalate)
POST   /compliance/sars                      # Create SAR draft from alert
GET    /compliance/sars/{sar_id}             # SAR detail
PATCH  /compliance/sars/{sar_id}             # Update SAR (edit narrative, change status)
POST   /compliance/sars/{sar_id}/file        # Submit SAR to FinCEN (mock in MVP)
GET    /compliance/ctrs                      # List CTRs
POST   /compliance/ctrs/{ctr_id}/file        # Submit CTR to FinCEN (mock in MVP)
GET    /compliance/dashboard                 # Compliance metrics (open alerts, pending SARs, return rates)
```

SAR narrative auto-generation: use LLM to draft the SAR Part V narrative from alert context, customer data, and transaction details. Compliance officer reviews and edits before filing.

**Testing**:
- Integration: Create SAR from alert -> SAR draft with subject info, activity dates, and auto-generated narrative
- Integration: File SAR -> filing_status changes to 'filed', filed_at set
- Integration: File SAR without narrative -> 400 error
- Integration: Dashboard returns correct counts of open alerts and pending SARs

#### 5.6 — Compliance Celery Tasks

**What**: Create async tasks for batch AML monitoring, CTR generation, and alert scoring.

**Design**:

```python
# src/banking_core/tasks/compliance_tasks.py
@celery_app.task
def run_aml_batch_scan():
    """
    Every 15 minutes (configurable):
    1. Fetch all transactions posted since last scan
    2. Run rule engine on each transaction
    3. For any new alerts, run AI scorer
    4. Persist alerts with scores
    """
    ...

@celery_app.task
def run_daily_ctr_sweep():
    """
    End-of-day:
    1. For each customer with cash activity today
    2. Sum cash-in and cash-out
    3. Generate CTRs where thresholds exceeded
    """
    ...
```

**Testing**:
- Integration: Batch scan after posting structuring-pattern transactions -> alert created with AI score
- Integration: Daily CTR sweep -> CTRs generated for qualifying customers

---

## Phase 6: Loan Origination and Servicing

### Purpose
Implement consumer loan origination (application, approval, disbursement) and servicing (payment processing, amortization schedule, delinquency tracking). After this phase, the system supports the full lifecycle for consumer instalment loans and lines of credit, with GL integration for all financial events.

### Tasks

#### 6.1 — Loan Product Configuration

**What**: Extend the product model with loan-specific configuration in the JSONB `config` and `property_schema` fields.

**Design**:

Product configurations for loan types:
```json
{
  "product_code": "CONSUMER_INSTALLMENT",
  "product_type": "consumer_loan",
  "config": {
    "max_term_months": 60,
    "min_credit_score": 620,
    "origination_fee_pct": 1.0,
    "late_payment_fee": 35.00,
    "late_payment_grace_days": 15,
    "payment_frequency_options": ["monthly", "biweekly"]
  },
  "property_schema": {
    "original_principal": {"type": "number", "required": true},
    "remaining_principal": {"type": "number", "required": true},
    "rate_type": {"type": "string", "enum": ["fixed", "variable"], "required": true},
    "payment_frequency": {"type": "string", "enum": ["monthly", "biweekly"], "required": true},
    "payment_amount": {"type": "number", "required": true},
    "next_payment_date": {"type": "string", "format": "date", "required": true},
    "origination_date": {"type": "string", "format": "date", "required": true},
    "maturity_date": {"type": "string", "format": "date", "required": true},
    "collateral": {"type": "object"},
    "delinquency_days": {"type": "integer", "default": 0}
  }
}
```

**Testing**:
- Unit: Loan product config validates max_term_months is positive integer
- Integration: Seed loan products -> queryable with correct config

#### 6.2 — Loan Origination Service

**What**: Implement loan application, approval decision, and disbursement workflow.

**Design**:

```python
class LoanService:
    async def originate_loan(
        self, db: AsyncSession, customer_id: uuid.UUID,
        product_code: str, principal: Decimal, interest_rate: Decimal,
        term_months: int, payment_frequency: str,
        collateral: dict | None = None,
    ) -> Account:
        """
        1. Validate customer KYC status
        2. Validate product configuration (term within max, rate within bounds)
        3. Create account with account_type='consumer_loan', status='active'
        4. Calculate amortization schedule -> insert loan_schedule rows
        5. Set account.properties with loan-specific fields
        6. Post GL journal: debit Loans Receivable, credit Cash/DDA
        7. Calculate origination fee -> post fee transaction
        """
        ...

    async def calculate_amortization(
        self, principal: Decimal, annual_rate: Decimal,
        term_months: int, payment_frequency: str,
        start_date: date,
    ) -> list[AmortizationRow]:
        """
        Standard amortization calculation:
        monthly_rate = annual_rate / 12
        payment = principal * (monthly_rate * (1 + monthly_rate)^n) / ((1 + monthly_rate)^n - 1)
        """
        ...
```

**Testing**:
- Unit: Amortization for $25,000 at 6.5% for 60 months -> payment ~$489.15, total interest correct
- Integration: Originate loan -> account created, schedule populated, GL balanced
- Integration: Originate loan for customer with kyc_status='pending' -> rejected

#### 6.3 — Loan Payment Processing

**What**: Process loan payments, update amortization schedule, handle early payoff and late fees.

**Design**:

```python
class LoanService:
    async def process_payment(
        self, db: AsyncSession, loan_account_id: uuid.UUID,
        amount: Decimal, payment_date: date,
    ) -> Transaction:
        """
        1. Find next due installment(s)
        2. Apply payment: interest first, then principal (standard application order)
        3. Update loan_schedule installment status to 'paid'
        4. Update account.properties.remaining_principal
        5. Post GL journal: debit Cash, credit Loans Receivable (principal) + Interest Income (interest)
        """
        ...

    async def assess_late_fees(
        self, db: AsyncSession, as_of_date: date,
    ) -> list[Transaction]:
        """
        For all loans with overdue installments past grace period:
        1. Create late fee transaction
        2. Update delinquency_days
        3. Post GL journal: debit Loan account, credit Fee Income
        """
        ...
```

**Testing**:
- Integration: Pay exact monthly amount -> installment marked 'paid', principal reduced correctly
- Integration: Pay more than monthly amount -> excess applied to principal, schedule adjusted
- Integration: Skip payment past grace period -> late fee assessed, delinquency_days updated
- Unit: Payment application splits correctly between interest and principal

#### 6.4 — Loan API Endpoints

**What**: REST API for loan operations.

**Design**:

```
POST   /loans/originate                         # Originate a new loan
GET    /accounts/{account_id}/schedule           # Amortization schedule
POST   /accounts/{account_id}/payments           # Process loan payment
GET    /loans/delinquent                          # List delinquent loans
POST   /loans/late-fees                           # Batch assess late fees (admin)
GET    /accounts/{account_id}/payoff-quote        # Calculate payoff amount as of date
```

**Testing**:
- Integration: Full lifecycle: originate -> 3 monthly payments -> payoff quote -> early payoff -> account closed
- Integration: Delinquent loans endpoint returns loans with delinquency_days > 0

---

## Phase 7: Interest Accrual and Period-Close

### Purpose
Implement daily interest accrual for deposit and loan accounts, and the GL period-close process. After this phase, the system performs the core back-office operations that community banks run nightly and monthly: computing interest earned/owed, crediting/debiting accounts, and closing the books.

### Tasks

#### 7.1 — Interest Calculation Engine

**What**: Implement interest calculation for different product types (simple, compound, tiered).

**Design**:

```python
# src/banking_core/services/interest_service.py
class InterestService:
    async def calculate_daily_interest(
        self, account: Account, as_of_date: date,
    ) -> Decimal:
        """
        Daily interest = balance * (annual_rate / 365)
        For tiered rates: apply rate tiers from product.config to balance bands
        For CDs: verify not past maturity, apply penalty if early withdrawal
        """
        ...

    async def accrue_interest_batch(
        self, db: AsyncSession, as_of_date: date,
    ) -> int:
        """
        For all active deposit and loan accounts:
        1. Calculate daily interest
        2. Update account.interest_accrued
        3. Post GL journal: debit Interest Expense (deposits) or debit Loans Receivable (loans),
           credit Interest Payable (deposits) or credit Interest Income (loans)
        Returns count of accounts processed.
        """
        ...

    async def pay_interest(
        self, db: AsyncSession, account_id: uuid.UUID,
    ) -> Transaction:
        """
        Credit accrued interest to account (typically monthly for savings/DDA, at maturity for CD).
        Post GL: debit Interest Payable, credit Customer Deposits.
        Reset interest_accrued to 0.
        """
        ...
```

**Testing**:
- Unit: $10,000 at 4.5% APY -> daily accrual of ~$1.23
- Unit: Tiered rate: first $5,000 at 2%, next $5,000 at 3% -> weighted daily accrual
- Integration: Batch accrual for 100 accounts -> all interest_accrued updated, GL balanced
- Integration: Pay interest on savings account -> balance increases, interest_accrued resets to 0

#### 7.2 — Interest Accrual Celery Task

**What**: Nightly Celery task for batch interest accrual.

**Design**:

```python
@celery_app.task
def run_nightly_interest_accrual():
    """
    Scheduled at 00:30 institution local time:
    1. Run accrue_interest_batch for today
    2. On last business day of month: run pay_interest for all deposit accounts
    3. Log summary: accounts processed, total interest accrued, GL verification
    """
    ...
```

**Testing**:
- Integration: Run nightly task -> all active accounts have interest_accrued updated
- Integration: Month-end run -> interest credited to deposit accounts, GL balanced

#### 7.3 — GL Period-Close Process

**What**: Implement monthly and quarterly GL period-close with trial balance verification.

**Design**:

```python
class LedgerService:
    async def close_period(
        self, db: AsyncSession, period_id: uuid.UUID, posted_by: uuid.UUID,
    ) -> GLPeriod:
        """
        1. Run trial balance -> verify debits == credits
        2. For year-end close: post closing entries (revenue/expense -> retained earnings)
        3. Update period status to 'closed'
        4. Prevent any future posting to closed periods
        """
        ...
```

API endpoint:
```
POST   /ledger/periods/{period_id}/close    # Close a GL period (admin permission)
GET    /ledger/trial-balance                # Trial balance as of date
GET    /ledger/periods                      # List GL periods with status
```

**Testing**:
- Integration: Close period with balanced trial balance -> period status = 'closed'
- Integration: Post journal to closed period -> 400 error
- Integration: Trial balance report -> all accounts listed with debit/credit totals, totals match

---

## Phase 8: Wire Transfer Processing (ISO 20022)

### Purpose
Implement wire transfer processing for Fedwire and SWIFT, with ISO 20022 message generation and structured address support per the November 2026 SWIFT CBPR+ mandate. This is a v1.1 feature that extends the payment rails beyond ACH.

### Tasks

#### 8.1 — Wire Transfer Model and Service

**What**: Create the wire_transfer model and implement initiation, OFAC screening, and settlement workflows.

**Design**:

Wire transfer model follows Data Model 3 with ISO 20022 pacs.008 fields (instruction_id, end_to_end_id, structured creditor address). OFAC screening is stubbed with a configurable screening provider interface.

```python
class WireService:
    async def initiate_wire(
        self, db: AsyncSession, account_id: uuid.UUID,
        amount: Decimal, creditor_name: str, creditor_account: str,
        creditor_agent_bic: str | None, creditor_address: StructuredAddress,
        purpose_code: str | None, remittance_info: str | None,
    ) -> WireTransfer:
        """
        1. Validate account has sufficient funds
        2. Place hold for wire amount
        3. Run OFAC screening
        4. Generate ISO 20022 pacs.008 message
        5. Create wire_transfer record with status='initiated'
        6. Post GL journal: debit Customer Deposits, credit Due-To (correspondent bank)
        """
        ...
```

```python
class StructuredAddress(BaseModel):
    """ISO 20022 structured postal address — mandatory from Nov 2026."""
    street_name: str = Field(max_length=200)
    building_number: str | None = Field(None, max_length=20)
    city: str = Field(max_length=100)
    postal_code: str | None = Field(None, max_length=20)
    country: str = Field(min_length=2, max_length=2)  # ISO 3166-1 alpha-2
```

**Testing**:
- Integration: Initiate outbound wire -> wire record created, hold placed, GL balanced
- Integration: Wire with unstructured address (missing street_name) -> 400 validation error
- Unit: ISO 20022 pacs.008 XML generated with correct namespace and field mapping

#### 8.2 — ISO 20022 XML Generation

**What**: Generate ISO 20022 pacs.008 XML messages for outbound wire transfers.

**Design**:

```python
# src/banking_core/utils/iso20022.py
class ISO20022Generator:
    def generate_pacs008(self, wire: WireTransfer) -> str:
        """
        Generate ISO 20022 pacs.008.001.10 FIToFICustomerCreditTransfer message.
        Uses lxml for XML generation with proper namespaces.
        Structured address fields are mandatory (SWIFT CBPR+ Nov 2026).
        """
        ...
```

**Testing**:
- Unit: Generated XML validates against pacs.008.001.10 XSD schema
- Unit: Structured address fields present in creditor postal address element
- Fixture: Compare against `tests/fixtures/sample_pacs008.xml`

#### 8.3 — Wire Transfer API

**What**: REST endpoints for wire operations.

**Design**:

```
POST   /wires/outbound                      # Initiate outbound wire
GET    /wires                                # List wire transfers
GET    /wires/{wire_id}                      # Wire detail
PATCH  /wires/{wire_id}/status               # Update wire status (settlement, rejection)
GET    /wires/{wire_id}/message              # Download ISO 20022 XML message
```

**Testing**:
- Integration: Full wire lifecycle: initiate -> OFAC screen -> settle -> GL balanced
- Integration: Wire to OFAC-flagged entity -> wire rejected, hold released

---

## Phase 9: Graph Layer and AI Fraud Detection

### Purpose
Add the graph analytical overlay (graph_node, graph_edge tables from Data Model 4) and implement AI-native fraud detection that operates on the transaction graph. After this phase, the system can detect fraud patterns (account takeover, synthetic identity, structuring networks) by analyzing relationships between customers, accounts, addresses, and transaction flows.

### Tasks

#### 9.1 — Graph Models and Projection Service

**What**: Create graph_node and graph_edge models. Implement event-driven projection that maintains the graph from relational data changes.

**Design**:

```python
# src/banking_core/models/graph.py
class GraphNode(Base):
    __tablename__ = "graph_node"

    node_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    node_type: Mapped[str] = mapped_column(String(30))
    entity_id: Mapped[uuid.UUID | None]
    label: Mapped[str] = mapped_column(String(200))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    is_active: Mapped[bool] = mapped_column(default=True)
    first_seen_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    last_seen_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

class GraphEdge(Base):
    __tablename__ = "graph_edge"

    edge_id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    source_node_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("graph_node.node_id"))
    target_node_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("graph_node.node_id"))
    edge_type: Mapped[str] = mapped_column(String(40))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    weight: Mapped[Decimal] = mapped_column(Numeric(10, 4), default=Decimal("1.0"))
    first_seen_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    last_seen_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

```python
# src/banking_core/services/graph_service.py
class GraphService:
    async def project_customer(self, db: AsyncSession, customer: Customer) -> GraphNode:
        """Create/update customer node and RESIDES_AT, USES_PHONE, USES_EMAIL edges."""
        ...

    async def project_account_ownership(self, db: AsyncSession, owner: AccountOwner) -> GraphEdge:
        """Create OWNS_ACCOUNT edge between customer and account nodes."""
        ...

    async def project_transaction(self, db: AsyncSession, txn: Transaction) -> GraphEdge | None:
        """
        For internal transfers: create/update TRANSFERRED_TO edge with cumulative amount and count.
        For ACH: create RECEIVED_ACH or SENT_ACH edge to external entity node.
        For wires: create WIRE_TO or WIRE_FROM edge.
        """
        ...

    async def detect_shared_attributes(self, db: AsyncSession, customer_id: uuid.UUID) -> list[GraphEdge]:
        """
        Find other customers sharing address, phone, or email with this customer.
        Create SHARES_ADDRESS, SHARES_PHONE edges.
        """
        ...
```

**Testing**:
- Integration: Create customer with address -> graph node created, RESIDES_AT edge created
- Integration: Transfer between two accounts -> TRANSFERRED_TO edge created/updated with cumulative amount
- Integration: Two customers share same address -> SHARES_ADDRESS edge detected and created
- Unit: Graph projection is idempotent (running twice produces same result)

#### 9.2 — Graph-Based Fraud Detection

**What**: Implement fraud detection algorithms that traverse the graph for patterns invisible to single-transaction rules.

**Design**:

```python
# src/banking_core/ai/fraud_detector.py
class GraphFraudDetector:
    async def detect_structuring_network(
        self, db: AsyncSession, customer_node_id: uuid.UUID,
    ) -> FraudAnalysisResult | None:
        """
        Detect coordinated structuring across related accounts:
        1. Traverse SHARES_ADDRESS/SHARES_PHONE edges (2 hops)
        2. Collect all connected accounts via OWNS_ACCOUNT edges
        3. Sum cash deposits below $10,000 across all connected accounts in 48-hour windows
        4. If aggregate exceeds threshold -> generate alert with graph_context snapshot
        """
        ...

    async def detect_account_takeover(
        self, db: AsyncSession, customer_node_id: uuid.UUID,
    ) -> FraudAnalysisResult | None:
        """
        Detect potential account takeover:
        1. Check for new ACCESSED_FROM (IP) or USES_DEVICE edges in last 24 hours
        2. If new device/IP AND high-value transaction -> flag for review
        """
        ...

    async def calculate_network_risk_score(
        self, db: AsyncSession, customer_node_id: uuid.UUID,
    ) -> float:
        """
        Score 0.0-1.0 based on:
        - Number of high-risk connections (SHARES_* with high-risk customers)
        - Number of flagged connections (FLAGGED_WITH edges)
        - Transfer concentration (single recipient receiving >50% of outflows)
        """
        ...
```

**Testing**:
- Integration: 3 customers sharing address each deposit $9,500 -> structuring network detected
- Integration: Customer with 0 high-risk connections -> network risk score near 0.0
- Integration: Customer connected to 3 flagged customers -> elevated risk score
- Unit: Network risk score calculation weights high-risk connections correctly

#### 9.3 — Graph Maintenance Celery Tasks

**What**: Async tasks for graph projection and batch fraud analysis.

**Design**:

```python
@celery_app.task
def run_graph_projection():
    """Every 5 minutes: project new/updated relational entities into graph."""
    ...

@celery_app.task
def run_graph_fraud_scan():
    """Every 30 minutes: run graph-based fraud detection on recently active customers."""
    ...
```

**Testing**:
- Integration: Projection task after 10 new customers and 50 transactions -> graph updated correctly
- Integration: Fraud scan detects previously undetected structuring network

---

## Phase 10: Digital Banking API (FDX) and MCP Server

### Purpose
Expose a consumer-permissioned data sharing API aligned with FDX v6.x for fintech integrations and CFPB Section 1033 readiness. Add an MCP (Model Context Protocol) server for AI agent integration, following Nymbus's production MCP Server pattern. These are v1.1 features that position the core for open banking and AI-driven member self-service.

### Tasks

#### 10.1 — FDX-Aligned Data Sharing API

**What**: Implement read-only API endpoints for consumer-permissioned data sharing, aligned with FDX v6.5 entity names and field structures.

**Design**:

FDX entity mapping:
- FDX `Account` -> local `account` (with FDX-specific field names in response)
- FDX `Transaction` -> local `transaction`
- FDX `Customer` -> local `customer` (PII fields masked by default)

FAPI 1.0 authorization: endpoints require OAuth 2.0 tokens issued with consumer consent scope. Tokens carry `fdx:accounts:read`, `fdx:transactions:read` scopes.

```
GET  /fdx/v6/accounts                          # List accounts the consumer has permissioned
GET  /fdx/v6/accounts/{account_id}             # Account detail in FDX format
GET  /fdx/v6/accounts/{account_id}/transactions # Transactions in FDX format (paginated)
GET  /fdx/v6/customers/current                 # Consumer profile in FDX format
```

**Testing**:
- Integration: FDX account response matches FDX v6.5 entity field names
- Integration: Request without fdx scope -> 403
- Integration: Transaction pagination follows FDX cursor-based pattern
- Unit: FDX response schema validates against FDX entity definitions

#### 10.2 — MCP Server for AI Agent Integration

**What**: Implement an MCP server exposing core banking actions as tools for AI agents.

**Design**:

MCP tools (aligned with Nymbus's 19-tool pattern):
1. `customer_lookup` — search by name, account number, or customer number
2. `get_account_details` — retrieve account balances and status
3. `get_transaction_history` — retrieve recent transactions
4. `get_account_owners` — list account owners/signers
5. `post_deposit` — post a deposit transaction
6. `post_withdrawal` — post a withdrawal
7. `initiate_transfer` — internal transfer between accounts
8. `get_loan_schedule` — retrieve amortization schedule
9. `get_loan_payoff_quote` — calculate payoff amount
10. `process_loan_payment` — apply a loan payment
11. `check_compliance_alerts` — list open AML alerts for a customer
12. `get_account_holds` — list active holds
13. `place_hold` — place an administrative hold
14. `release_hold` — release a hold

Security:
- Token-based authentication (same OAuth 2.0 infrastructure)
- RBAC: each tool requires specific permissions
- PII masking: tax IDs, encrypted fields never returned to AI agents
- Audit logging: all MCP tool invocations logged

```python
# src/banking_core/mcp_server.py
from mcp import Server, Tool

server = Server("banking-core")

@server.tool("customer_lookup")
async def customer_lookup(query: str, search_type: str = "name") -> dict:
    """Search for customers by name, account number, or customer number.
    Returns masked PII — tax IDs show last 4 only."""
    ...

@server.tool("post_deposit")
async def post_deposit(account_id: str, amount: float, description: str) -> dict:
    """Post a deposit transaction. Requires transaction.post permission."""
    ...
```

**Testing**:
- Integration: MCP `customer_lookup` with valid name -> returns matching customers without full tax IDs
- Integration: MCP `post_deposit` -> transaction created, balance updated
- Integration: MCP tool call without auth token -> rejected
- Integration: MCP tool call without required permission -> rejected
- Unit: All MCP responses exclude encrypted PII fields

#### 10.3 — Automated Regulatory Change Monitoring

**What**: Implement an AI-powered service that monitors FFIEC, FinCEN, and CFPB regulatory updates and flags which core configurations need review.

**Design**:

```python
# src/banking_core/ai/regulatory_monitor.py
class RegulatoryMonitor:
    async def check_updates(self) -> list[RegulatoryUpdate]:
        """
        1. Fetch recent updates from FinCEN, FFIEC, CFPB RSS/API feeds
        2. Use LLM to analyze each update's impact on core banking configuration
        3. Return structured results: update title, source, impact assessment, affected modules
        """
        ...
```

Celery task runs weekly, stores results in a `regulatory_update` table, and notifies compliance officers of high-impact changes.

**Testing**:
- Integration (mocked feeds): New FinCEN rule -> parsed, impact assessed, notification generated
- Unit: Regulatory update schema validates source, impact_level, affected_modules

---

## Phase 11: Teller and Branch Operations UI

### Purpose
Build a web-based staff interface for teller operations: customer lookup, account inquiry, transaction posting, and cash drawer management. This is the v1.1 feature that replaces the legacy green-screen teller workflow at community banks.

### Tasks

#### 11.1 — Teller UI Application

**What**: Create a React/Next.js frontend for teller operations, consuming the existing REST API.

**Design**:

Technology: Next.js 15 + Tailwind CSS + shadcn/ui components. The frontend is a separate project within the monorepo, served independently or via the FastAPI static file mount.

Key screens:
- **Customer Search**: Search by name, account number, tax ID last 4. Display customer profile with accounts.
- **Account Detail**: Show balance, recent transactions, holds. Quick-action buttons: deposit, withdrawal, transfer.
- **Transaction Entry**: Amount, type, memo. Confirmation dialog before posting.
- **Cash Drawer**: Open/close drawer, count, reconciliation.
- **Compliance Dashboard**: Open AML alerts, pending SARs/CTRs.

Authentication: OAuth 2.0 login flow with the core API as the OAuth provider.

**Testing**:
- E2E (Playwright): Login -> search customer -> select account -> post deposit -> verify balance updated
- E2E: Login with teller credentials -> compliance dashboard not accessible (missing permission)
- Unit: React component tests for transaction entry form validation

#### 11.2 — Cash Drawer Management

**What**: Implement cash drawer tracking for teller sessions.

**Design**:

New tables: `teller_session` (open/close, opening_balance, closing_balance, teller_id, branch_id) and `cash_count` (denomination breakdown).

API endpoints:
```
POST   /teller/sessions/open              # Open cash drawer
POST   /teller/sessions/close             # Close and reconcile
GET    /teller/sessions/current            # Current session details
```

**Testing**:
- Integration: Open drawer with $5,000 -> session created
- Integration: Post 3 cash deposits, 1 cash withdrawal -> drawer balance adjusted correctly
- Integration: Close drawer with count -> reconciliation difference calculated

---

## Phase 12: Overdraft Prediction and Credit Recommendations

### Purpose
Implement AI-driven predictive overdraft management and credit limit recommendations, replacing static rule-based thresholds with real-time cash flow analysis. This is the AI-native feature that differentiates the core from legacy systems that use fixed overdraft limits.

### Tasks

#### 12.1 — Cash Flow Prediction Model

**What**: Build a predictive model that forecasts account cash flows based on transaction history patterns.

**Design**:

```python
# src/banking_core/ai/overdraft_predictor.py
class OverdraftPredictor:
    async def predict_overdraft_risk(
        self, db: AsyncSession, account_id: uuid.UUID,
        horizon_days: int = 7,
    ) -> OverdraftPrediction:
        """
        1. Gather 90 days of transaction history
        2. Identify recurring patterns (payroll deposits, bill payments)
        3. Use LLM to analyze cash flow trajectory
        4. Return: probability of overdraft within horizon, recommended actions
        """
        ...

    async def recommend_overdraft_limit(
        self, db: AsyncSession, account_id: uuid.UUID,
    ) -> Decimal:
        """
        Based on income patterns, spending volatility, and account age,
        recommend an appropriate overdraft limit.
        """
        ...
```

**Testing**:
- Unit (mocked LLM): Account with stable payroll and predictable bills -> low overdraft risk
- Unit (mocked LLM): Account with irregular income and high-variance spending -> elevated risk
- Integration: Prediction endpoint returns structured result with probability and recommendations

#### 12.2 — Proactive Overdraft Notifications

**What**: Celery task that runs daily, identifies at-risk accounts, and generates notifications.

**Design**:

```python
@celery_app.task
def run_overdraft_risk_scan():
    """
    Daily at 06:00:
    1. Identify accounts with balance below their 7-day forecast spending
    2. Run overdraft prediction for each
    3. Generate notifications for high-risk accounts
    4. Optionally adjust overdraft limits per AI recommendation
    """
    ...
```

**Testing**:
- Integration: Account predicted to overdraft in 3 days -> notification generated
- Integration: Account with healthy balance -> no notification

---

## Phase Summary & Dependencies

```
Phase 1:  Foundation & Scaffolding          ─── required by everything
    │
Phase 2:  Customer Management & RBAC        ─── requires Phase 1
    │
Phase 3:  Accounts & General Ledger         ─── requires Phase 2
    │
    ├── Phase 4: ACH Payment Processing     ─── requires Phase 3
    │       │
    │       └── Phase 5: BSA/AML & AI       ─── requires Phase 3, 4 (for ACH-based alerts)
    │
    ├── Phase 6: Loan Origination           ─── requires Phase 3 (can parallel with 4, 5)
    │
    └── Phase 7: Interest & Period-Close    ─── requires Phase 3 (can parallel with 4, 5, 6)

Phase 8:  Wire Transfer (ISO 20022)         ─── requires Phase 3 (can parallel with 4-7)

Phase 9:  Graph Layer & Fraud Detection     ─── requires Phase 2, 3, 4, 5

Phase 10: FDX API & MCP Server              ─── requires Phase 2, 3, 4, 8

Phase 11: Teller UI                         ─── requires Phase 2, 3, 4

Phase 12: Overdraft Prediction              ─── requires Phase 3, 7
```

**Parallelism opportunities:**
- Phases 4, 6, 7, 8 can all be developed concurrently after Phase 3 is complete
- Phase 5 can start after Phase 3 but benefits from Phase 4 for ACH-based AML rules
- Phase 11 can start after Phase 4 is complete (needs core + ACH for the teller workflow)
- Phase 12 can start after Phase 7 (needs interest accrual for cash flow analysis)

---

## Definition of Done (per phase)

1. All tasks implemented per the design specification.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Ruff linting passes with zero errors (`ruff check src/ tests/`).
5. Ruff formatting passes (`ruff format --check src/ tests/`).
6. mypy strict mode passes (`mypy src/`).
7. Docker build succeeds (`docker build -t banking-core .`).
8. `docker-compose up` starts all services and `GET /health` returns 200.
9. All new API endpoints appear in the auto-generated OpenAPI 3.1 spec at `/openapi.json`.
10. Database migrations created and apply cleanly (`alembic upgrade head`).
11. New configuration options documented in `config.py` with environment variable names.
12. GL trial balance remains balanced after all test scenarios.
13. Audit log entries created for all state-changing operations.
14. Test coverage >= 80% for new code (`pytest --cov`).
