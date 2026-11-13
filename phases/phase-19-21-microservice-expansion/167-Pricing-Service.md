# 167 — Pricing Service

**File:** `167-Pricing-Service.md`

```markdown
# 167 — Pricing Service

## 1. Overview

The `pricing-service` is responsible for calculating the price of products
during the customer journey.

Pricing is different from the product catalog.

The catalog answers:

> What is this product?

The pricing service answers:

> What price should this customer/order pay right now?

The service supports:

- Base price
- Customer-specific pricing
- Quantity-based pricing
- Effective dates
- Currency
- Price overrides
- Sale prices
- Pricing rules
- Price lookup
- Price calculation
- Price versioning
- Cached price reads

Technology for this service:

- Python 3.13
- FastAPI
- Pydantic
- SQLAlchemy
- PostgreSQL
- Redis
- Docker
- Kubernetes
- Helm
- EKS
- ECR
- GitHub Actions
- ArgoCD
- Prometheus
- Grafana
- ELK
- OpenTelemetry
- Jaeger

---

# 2. Why Pricing Is Its Own Service

We do not put all pricing logic inside:

```text
product-service
```

because pricing can become complex.

For example:

```text
Product price
+
Customer tier
+
Quantity
+
Promotion
+
Currency
+
Effective date
+
Region
+
Business rules
```

The architecture becomes:

```text
catalog-service
      |
      | Product information
      ↓
pricing-service
      |
      ├── PostgreSQL
      └── Redis
```

---

# 3. Business Responsibility

Pricing service owns:

- Base price
- Sale price
- Customer tier pricing
- Quantity pricing
- Price validity
- Currency
- Price version
- Price calculation
- Price lookup

It does NOT own:

- Product description
- Product images
- Inventory
- Customer profile
- Promotions themselves
- Orders
- Payments
- Tax

Those belong to other services.

---

# 4. Service Architecture

```text
                         Client
                           |
                           ↓
                         ALB
                           |
                           ↓
                    pricing-service
                       /        \
                      /          \
                     ↓            ↓
                Redis Cache    PostgreSQL
                                   |
                                   ↓
                              Price Rules
```

High-read path:

```text
Client
  |
  ↓
pricing-service
  |
  ↓
Redis
  |
  ├── HIT → return price
  |
  └── MISS
       |
       ↓
   PostgreSQL
       |
       ↓
   Cache result
```

---

# 5. Why Redis Is Important

Pricing is normally a read-heavy workload.

Example:

```text
Product page
    ↓
Price lookup
    ↓
Product page
    ↓
Price lookup
    ↓
Cart
    ↓
Price lookup
```

Thousands of requests can ask for the same product price.

Without caching:

```text
10,000 requests
       ↓
10,000 PostgreSQL reads
```

With Redis:

```text
10,000 requests
       ↓
Redis
       ↓
Most requests served from cache
       ↓
Small number of PostgreSQL reads
```

However:

> PostgreSQL remains the source of truth.

Redis is not the authoritative pricing database.

---

# 6. Price Data Model

Example:

```text
product_id
currency
base_price
sale_price
customer_tier
effective_from
effective_to
version
status
```

Example:

```text
Product: P1001

Base price:
₹10,000

Sale price:
₹9,499

Currency:
INR

Valid:
2026-10-01 → 2026-10-31
```

---

# 7. Repository Structure

```text
pricing-service/
│
├── app/
│   ├── main.py
│   │
│   ├── config.py
│   │
│   ├── database.py
│   │
│   ├── models/
│   │   └── pricing.py
│   │
│   ├── schemas/
│   │   └── pricing.py
│   │
│   ├── repositories/
│   │   └── pricing_repository.py
│   │
│   ├── services/
│   │   ├── pricing_service.py
│   │   └── cache_service.py
│   │
│   ├── api/
│   │   └── pricing.py
│   │
│   ├── observability/
│   │   └── metrics.py
│   │
│   └── dependencies.py
│
├── migrations/
│   ├── 001_create_prices.sql
│   └── 002_create_price_rules.sql
│
├── tests/
│   ├── test_pricing.py
│   └── test_api.py
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── .env.example
├── requirements.txt
├── pyproject.toml
├── Makefile
├── README.md
│
├── helm/
│   └── pricing-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── configmap.yaml
│           ├── externalsecret.yaml
│           ├── serviceaccount.yaml
│           ├── networkpolicy.yaml
│           ├── hpa.yaml
│           ├── pdb.yaml
│           └── servicemonitor.yaml
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
        ├── ci.yaml
        └── security.yaml
```

---

# 8. Python Dependency Management

This service uses:

```text
Python
   ↓
FastAPI
   ↓
SQLAlchemy
   ↓
PostgreSQL
```

Dependency management:

```text
requirements.txt
```

or:

```text
pyproject.toml
```

For this project we will use:

```text
pyproject.toml
```

because it provides a cleaner modern Python project structure.

---

# 9. `pyproject.toml`

```toml
[project]

name = "pricing-service"

version = "1.0.0"

description = "Ecommerce pricing service"

requires-python = ">=3.13"

dependencies = [
    "fastapi>=0.115,<1.0",
    "uvicorn[standard]>=0.34,<1.0",
    "sqlalchemy>=2.0,<3.0",
    "asyncpg>=0.30,<1.0",
    "redis>=6.0,<7.0",
    "pydantic>=2.10,<3.0",
    "pydantic-settings>=2.7,<3.0",
    "prometheus-client>=0.21,<1.0",
]

[project.optional-dependencies]

test = [
    "pytest>=8.3,<9.0",
    "pytest-asyncio>=0.25,<1.0",
    "httpx>=0.28,<1.0",
]

lint = [
    "ruff>=0.9,<1.0",
]

[tool.pytest.ini_options]

asyncio_mode = "auto"

[tool.ruff]

line-length = 100
```

Production versions should be pinned/locked through the project's chosen
dependency-management process.

---

# 10. Environment Configuration

## `.env.example`

```text
APP_NAME=pricing-service
ENVIRONMENT=local
PORT=8080

DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/pricing

REDIS_URL=redis://localhost:6379/0

PRICE_CACHE_TTL_SECONDS=60

LOG_LEVEL=INFO
```

Never commit:

```text
.env
```

with real credentials.

---

# 11. Configuration Code

## `app/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):

    app_name: str = "pricing-service"

    environment: str = "local"

    port: int = 8080

    database_url: str

    redis_url: str

    price_cache_ttl_seconds: int = 60

    log_level: str = "INFO"

    model_config = SettingsConfigDict(
        env_file=".env",
        case_sensitive=False,
        extra="ignore",
    )


settings = Settings()
```

---

# 12. Database Connection

## `app/database.py`

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.config import settings


engine = create_async_engine(
    settings.database_url,

    pool_size=10,

    max_overflow=5,

    pool_timeout=3,

    pool_recycle=1500,

    pool_pre_ping=True,
)


SessionLocal = async_sessionmaker(
    bind=engine,

    class_=AsyncSession,

    expire_on_commit=False,
)


async def get_db() -> AsyncGenerator[
    AsyncSession,
    None,
]:

    async with SessionLocal() as session:

        yield session
```

---

# 13. SQLAlchemy Pool

Python SQLAlchemy uses:

```text
pool_size
max_overflow
pool_timeout
pool_recycle
pool_pre_ping
```

Example:

```text
pool_size = 10
max_overflow = 5
```

Potential connections per Pod:

```text
10 + 5 = 15
```

If Kubernetes has:

```text
10 Pods
```

potential connections can be approximately:

```text
10 × 15 = 150
```

This must be included in RDS capacity planning.

Do not assume:

```text
HPA scaling
```

and:

```text
database capacity
```

are independent.

---

# 14. Price Model

## `app/models/pricing.py`

```python
from datetime import datetime
from decimal import Decimal

from sqlalchemy import (
    DateTime,
    Integer,
    Numeric,
    String,
)
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    mapped_column,
)


class Base(DeclarativeBase):
    pass


class Price(Base):

    __tablename__ = "prices"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True,
    )

    product_id: Mapped[str] = mapped_column(
        String(100),
        index=True,
        nullable=False,
    )

    currency: Mapped[str] = mapped_column(
        String(3),
        nullable=False,
    )

    base_price: Mapped[Decimal] = mapped_column(
        Numeric(19, 4),
        nullable=False,
    )

    sale_price: Mapped[Decimal | None] = mapped_column(
        Numeric(19, 4),
        nullable=True,
    )

    customer_tier: Mapped[str] = mapped_column(
        String(50),
        default="STANDARD",
        nullable=False,
    )

    effective_from: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
    )

    effective_to: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
    )

    status: Mapped[str] = mapped_column(
        String(30),
        default="ACTIVE",
        nullable=False,
    )
```

---

# 15. Why Decimal Is Used

Do NOT calculate monetary values using floating-point numbers such as:

```python
0.1 + 0.2
```

Money should use:

```python
Decimal
```

and database:

```text
NUMERIC
```

Example:

```text
NUMERIC(19,4)
```

This avoids common floating-point precision problems.

---

# 16. Database Migration

## `migrations/001_create_prices.sql`

```sql
CREATE TABLE prices (

    id BIGSERIAL PRIMARY KEY,

    product_id VARCHAR(100) NOT NULL,

    currency VARCHAR(3) NOT NULL,

    base_price NUMERIC(19,4) NOT NULL,

    sale_price NUMERIC(19,4),

    customer_tier VARCHAR(50) NOT NULL
        DEFAULT 'STANDARD',

    effective_from TIMESTAMPTZ NOT NULL,

    effective_to TIMESTAMPTZ,

    status VARCHAR(30) NOT NULL
        DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_prices_product
ON prices(product_id);

CREATE INDEX idx_prices_product_currency
ON prices(product_id, currency);

CREATE INDEX idx_prices_effective
ON prices(effective_from, effective_to);
```

---

# 17. Price Rules

## `migrations/002_create_price_rules.sql`

```sql
CREATE TABLE price_rules (

    id BIGSERIAL PRIMARY KEY,

    product_id VARCHAR(100) NOT NULL,

    rule_type VARCHAR(50) NOT NULL,

    rule_value NUMERIC(19,4) NOT NULL,

    minimum_quantity INTEGER,

    customer_tier VARCHAR(50),

    priority INTEGER NOT NULL DEFAULT 100,

    effective_from TIMESTAMPTZ NOT NULL,

    effective_to TIMESTAMPTZ,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_price_rules_product
ON price_rules(product_id);

CREATE INDEX idx_price_rules_priority
ON price_rules(priority);
```

---

# 18. Request Schema

## `app/schemas/pricing.py`

```python
from decimal import Decimal

from pydantic import BaseModel, Field


class PriceResponse(BaseModel):

    product_id: str

    currency: str

    base_price: Decimal

    final_price: Decimal

    customer_tier: str


class PriceCalculationRequest(BaseModel):

    product_id: str = Field(
        min_length=1,
        max_length=100,
    )

    quantity: int = Field(
        gt=0,
        le=10000,
    )

    currency: str = Field(
        min_length=3,
        max_length=3,
    )

    customer_tier: str = Field(
        default="STANDARD",
        max_length=50,
    )
```

---

# 19. Pricing Repository

## `app/repositories/pricing_repository.py`

```python
from datetime import datetime, timezone

from sqlalchemy import select

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.pricing import Price


class PricingRepository:

    async def find_active_price(
        self,
        session: AsyncSession,
        product_id: str,
        currency: str,
        customer_tier: str,
    ) -> Price | None:

        now = datetime.now(timezone.utc)

        query = (
            select(Price)
            .where(
                Price.product_id == product_id,
                Price.currency == currency,
                Price.customer_tier == customer_tier,
                Price.status == "ACTIVE",
                Price.effective_from <= now,
            )
            .where(
                (Price.effective_to.is_(None))
                | (Price.effective_to >= now)
            )
            .order_by(
                Price.effective_from.desc()
            )
            .limit(1)
        )

        result = await session.execute(query)

        return result.scalar_one_or_none()
```

---

# 20. Redis Cache Service

## `app/services/cache_service.py`

```python
import json

from redis.asyncio import Redis

from app.config import settings


class CacheService:

    def __init__(self, redis: Redis):

        self.redis = redis

    def build_key(
        self,
        product_id: str,
        currency: str,
        customer_tier: str,
    ) -> str:

        return (
            f"price:"
            f"{product_id}:"
            f"{currency}:"
            f"{customer_tier}"
        )

    async def get(
        self,
        key: str,
    ):

        value = await self.redis.get(key)

        if value is None:

            return None

        return json.loads(value)

    async def set(
        self,
        key: str,
        value: dict,
    ):

        await self.redis.setex(
            key,
            settings.price_cache_ttl_seconds,
            json.dumps(value),
        )

    async def delete(
        self,
        key: str,
    ):

        await self.redis.delete(key)
```

---

# 21. Pricing Service Layer

## `app/services/pricing_service.py`

```python
from decimal import Decimal

from sqlalchemy.ext.asyncio import AsyncSession

from app.repositories.pricing_repository import (
    PricingRepository,
)

from app.services.cache_service import (
    CacheService,
)


class PricingService:

    def __init__(
        self,
        repository: PricingRepository,
        cache: CacheService,
    ):

        self.repository = repository

        self.cache = cache

    async def calculate_price(
        self,
        session: AsyncSession,
        product_id: str,
        quantity: int,
        currency: str,
        customer_tier: str,
    ):

        cache_key = self.cache.build_key(
            product_id,
            currency,
            customer_tier,
        )

        cached = await self.cache.get(
            cache_key
        )

        if cached:

            final_price = Decimal(
                cached["final_price"]
            )

            return {
                **cached,
                "total_price":
                    str(
                        final_price * quantity
                    ),
            }

        price = await self.repository.find_active_price(
            session=session,
            product_id=product_id,
            currency=currency,
            customer_tier=customer_tier,
        )

        if price is None:

            raise ValueError(
                "price not found"
            )

        final_price = (
            price.sale_price
            if price.sale_price is not None
            else price.base_price
        )

        response = {
            "product_id":
                price.product_id,

            "currency":
                price.currency,

            "base_price":
                str(price.base_price),

            "final_price":
                str(final_price),

            "customer_tier":
                price.customer_tier,
        }

        await self.cache.set(
            cache_key,
            response,
        )

        return {
            **response,
            "total_price":
                str(
                    final_price * quantity
                ),
        }
```

---

# 22. API Routes

## `app/api/pricing.py`

```python
from fastapi import (
    APIRouter,
    Depends,
    HTTPException,
    status,
)

from sqlalchemy.ext.asyncio import AsyncSession

from redis.asyncio import Redis

from app.database import get_db
from app.schemas.pricing import (
    PriceCalculationRequest,
)
from app.repositories.pricing_repository import (
    PricingRepository,
)
from app.services.cache_service import (
    CacheService,
)
from app.services.pricing_service import (
    PricingService,
)


router = APIRouter(
    prefix="/api/v1/pricing",
    tags=["pricing"],
)


async def get_redis():

    redis = Redis.from_url(
        "redis://localhost:6379/0",
        decode_responses=True,
    )

    try:

        yield redis

    finally:

        await redis.close()


@router.post(
    "/calculate",
)
async def calculate_price(
    request: PriceCalculationRequest,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
):

    repository = PricingRepository()

    cache = CacheService(redis)

    service = PricingService(
        repository,
        cache,
    )

    try:

        return await service.calculate_price(
            session=session,
            product_id=request.product_id,
            quantity=request.quantity,
            currency=request.currency,
            customer_tier=request.customer_tier,
        )

    except ValueError as exc:

        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(exc),
        )
```

For production, Redis configuration should come from the centralized
configuration object rather than creating a hard-coded client inside the route.

---

# 23. Application Entry Point

## `app/main.py`

```python
from fastapi import FastAPI

from app.api.pricing import router as pricing_router


app = FastAPI(
    title="Pricing Service",
    version="1.0.0",
)


@app.get("/health")
async def health():

    return {
        "status": "UP",
    }


@app.get("/ready")
async def ready():

    return {
        "status": "READY",
    }


app.include_router(
    pricing_router
)
```

---

# 24. Production Health Checks

Health endpoints should be separated conceptually.

### Liveness

Answers:

> Is the process alive?

### Readiness

Answers:

> Should Kubernetes send traffic to this Pod?

Do not make liveness depend on:

```text
PostgreSQL
Redis
Kafka
```

Otherwise a temporary dependency failure can cause Kubernetes to restart healthy
application processes unnecessarily.

---

# 25. Dockerfile

## `Dockerfile`

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /build

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY pyproject.toml .

RUN pip install --no-cache-dir \
    --prefix=/install \
    fastapi \
    uvicorn[standard] \
    sqlalchemy \
    asyncpg \
    redis \
    pydantic \
    pydantic-settings \
    prometheus-client

COPY app ./app

FROM python:3.13-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY --from=builder /install /usr/local

COPY app ./app

RUN useradd \
    --create-home \
    --uid 10001 \
    appuser

USER 10001

EXPOSE 8080

CMD [
    "uvicorn",
    "app.main:app",
    "--host",
    "0.0.0.0",
    "--port",
    "8080"
]
```

For the final production implementation, dependency installation should be
based on the project's locked dependency file rather than floating package
resolution during every Docker build.

---

# 26. `.dockerignore`

```text
.git
.github

__pycache__
*.pyc

.pytest_cache

.venv
venv

.env
.env.*

coverage.xml
htmlcov

tests

README.md

Dockerfile
docker-compose.yaml
```

---

# 27. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: pricing-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: pricing

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data

    healthcheck:

      test:
        [
          "CMD-SHELL",
          "pg_isready -U postgres -d pricing"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: pricing-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  pricing-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: pricing-service

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

    environment:

      APP_NAME: pricing-service

      ENVIRONMENT: local

      PORT: 8080

      DATABASE_URL: >-
        postgresql+asyncpg://postgres:postgres@postgres:5432/pricing

      REDIS_URL: redis://redis:6379/0

      PRICE_CACHE_TTL_SECONDS: 60

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 28. Local Startup

```bash
docker compose up -d
```

Check:

```bash
curl http://localhost:8080/health
```

Expected:

```json
{
  "status": "UP"
}
```

---

# 29. Python Local Development

Create virtual environment:

```bash
python3.13 -m venv .venv
```

Activate:

```bash
source .venv/bin/activate
```

Windows:

```powershell
.venv\Scripts\activate
```

Install:

```bash
pip install -e ".[test,lint]"
```

Run:

```bash
uvicorn app.main:app --reload --port 8080
```

---

# 30. Tests

Run:

```bash
pytest
```

Lint:

```bash
ruff check .
```

Format:

```bash
ruff format .
```

---

# 31. Pricing API Test

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/pricing/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": "P1001",
    "quantity": 2,
    "currency": "INR",
    "customer_tier": "STANDARD"
  }'
```

Example:

```json
{
  "product_id": "P1001",
  "currency": "INR",
  "base_price": "10000.0000",
  "final_price": "9499.0000",
  "customer_tier": "STANDARD",
  "total_price": "18998.0000"
}
```

---

# 32. Cache Test

First request:

```text
Request
  ↓
Redis MISS
  ↓
PostgreSQL
  ↓
Redis SET
  ↓
Response
```

Second request:

```text
Request
  ↓
Redis HIT
  ↓
Response
```

This should reduce PostgreSQL read load.

---

# 33. Cache Key

Example:

```text
price:P1001:INR:STANDARD
```

For enterprise pricing:

```text
price:<product>:<currency>:<customer-tier>
```

Do not put uncontrolled high-cardinality data into Redis keys.

Keys should have:

- Clear naming
- TTL
- Ownership
- Version strategy

---

# 34. Cache Invalidation

Suppose:

```text
PostgreSQL price = ₹10,000
Redis price = ₹10,000
```

Admin changes:

```text
₹10,000 → ₹9,500
```

If Redis is not invalidated:

```text
PostgreSQL = ₹9,500
Redis = ₹10,000
```

Therefore price updates should perform:

```text
DB update
   ↓
Invalidate cache
   ↓
Publish PriceChanged event
```

For high-scale systems, event-driven cache invalidation can be used.

---

# 35. Cache Stampede

Suppose:

```text
Cache TTL = 60 seconds
```

At exactly the same moment:

```text
1000 requests
```

arrive after the key expires.

All may attempt:

```text
Redis MISS
   ↓
PostgreSQL
```

This can create a database spike.

Protection mechanisms include:

- Request coalescing
- Distributed lock
- Jittered TTL
- Background refresh
- Stale-while-revalidate
- Cache warming

---

# 36. Pricing Rules

Pricing can eventually support:

```text
STANDARD
SILVER
GOLD
PLATINUM
```

Example:

```text
Standard:
₹1000

Gold:
₹950

Platinum:
₹900
```

Quantity pricing:

```text
1-9:
₹1000

10-49:
₹950

50+:
₹900
```

The pricing engine should select the highest-priority applicable rule.

---

# 37. Do Not Put Promotion Logic Everywhere

Avoid:

```text
order-service
    ├── promotion logic
    ├── pricing logic
    ├── discount logic
    └── tax logic
```

Instead:

```text
order-service
      |
      ├── pricing-service
      |
      ├── promotion-service
      |
      └── tax-service
```

The final order should store the calculated price snapshot.

---

# 38. Price Snapshot

Suppose customer purchases:

```text
Product = P1001
Price at purchase = ₹9,499
```

Later:

```text
Price = ₹11,000
```

The old order must still show:

```text
₹9,499
```

Therefore:

> Orders should store the price snapshot used during the transaction.

Do not calculate historical order totals by querying the current pricing service.

---

# 39. Currency

Never assume:

```text
all prices = INR
```

The service should support:

```text
INR
USD
EUR
GBP
```

depending on business requirements.

Currency should be explicitly represented:

```text
currency = INR
```

Do not infer currency from formatting alone.

---

# 40. Rounding

Monetary rounding must be explicitly defined.

For example:

```text
base price
discount
tax
final amount
```

should have a documented rounding policy.

Do not allow different services to independently round values.

Otherwise:

```text
pricing-service = ₹99.99

order-service = ₹100.00

invoice-service = ₹100.01
```

can occur.

---

# 41. Kubernetes Deployment

## `helm/pricing-service/templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: pricing-service

  labels:

    app: pricing-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: pricing-service

  template:

    metadata:

      labels:

        app: pricing-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: pricing-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: pricing-service-config

            - secretRef:

                name: pricing-service-secrets

          resources:

            {{- toYaml .Values.resources | nindent 12 }}

          startupProbe:

            httpGet:

              path: /health

              port: http

            failureThreshold: 30

            periodSeconds: 10

          readinessProbe:

            httpGet:

              path: /ready

              port: http

            initialDelaySeconds: 5

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: /health

              port: http

            initialDelaySeconds: 20

            periodSeconds: 20

          securityContext:

            allowPrivilegeEscalation: false

            readOnlyRootFilesystem: true

            capabilities:

              drop:

                - ALL
```

---

# 42. Kubernetes Service

## `service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: pricing-service

  labels:

    app: pricing-service

spec:

  type: ClusterIP

  selector:

    app: pricing-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 43. ConfigMap

## `configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: pricing-service-config

data:

  ENVIRONMENT: prod

  PORT: "8080"

  REDIS_URL: redis://redis.ecommerce.svc.cluster.local:6379/0

  PRICE_CACHE_TTL_SECONDS: "60"

  LOG_LEVEL: INFO
```

The production Redis endpoint should match the actual architecture.

For Amazon ElastiCache, use the managed Redis/Valkey endpoint rather than
inventing a Kubernetes Service.

---

# 44. ExternalSecret

## `externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: pricing-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: pricing-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/pricing-service

        property: DATABASE_URL
```

---

# 45. ServiceAccount

## `serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: pricing-service

  labels:

    app: pricing-service
```

If the service needs AWS APIs:

```text
Pricing Pod
    ↓
EKS Pod Identity
    ↓
IAM Role
    ↓
Specific AWS API
```

Do not use:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside the Pod.

---

# 46. NetworkPolicy

## `networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: pricing-service

spec:

  podSelector:

    matchLabels:

      app: pricing-service

  policyTypes:

    - Ingress
    - Egress

  ingress:

    - from:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: ecommerce

      ports:

        - protocol: TCP

          port: 8080

  egress:

    - to:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: kube-system

      ports:

        - protocol: UDP

          port: 53

        - protocol: TCP

          port: 53

    - ports:

        - protocol: TCP

          port: 5432

        - protocol: TCP

          port: 6379
```

Actual production policy should use the real database/Redis connectivity model
and namespace/network controls.

---

# 47. HPA

Pricing is generally read-heavy and suitable for horizontal scaling.

## `hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: pricing-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: pricing-service

  minReplicas: 3

  maxReplicas: 30

  behavior:

    scaleUp:

      stabilizationWindowSeconds: 0

      policies:

        - type: Percent

          value: 100

          periodSeconds: 60

    scaleDown:

      stabilizationWindowSeconds: 300

      policies:

        - type: Percent

          value: 25

          periodSeconds: 60

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:

          type: Utilization

          averageUtilization: 60
```

Actual `maxReplicas` must be established using load testing and Redis/RDS
capacity.

---

# 48. Why Pricing Can Scale More Easily Than Inventory

Inventory:

```text
Request
  ↓
Database transaction
  ↓
Row contention
```

Pricing:

```text
Request
  ↓
Redis
  ↓
Return
```

Therefore pricing can often handle significantly more requests per Pod when
cache hit ratio is high.

But this does not mean:

```text
maxReplicas = unlimited
```

because:

- Redis has capacity limits
- PostgreSQL cache misses still exist
- network bandwidth matters
- downstream services matter
- Kubernetes node capacity matters

---

# 49. PodDisruptionBudget

## `pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: pricing-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: pricing-service
```

---

# 50. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: pricing-service

description: Ecommerce pricing service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 51. Helm Values

## `values.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/pricing-service

  tag: ""

  digest: ""

  pullPolicy: IfNotPresent


service:

  type: ClusterIP

  port: 8080


resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: pricing-service


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

---

# 52. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

resources:

  requests:

    cpu: 500m

    memory: 512Mi

  limits:

    cpu: "2"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60
```

Again:

> These numbers are starting values for the project simulation, not universal
> production values.

They must be validated through performance testing.

---

# 53. Prometheus ServiceMonitor

## `servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: pricing-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: pricing-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

The FastAPI application should expose Prometheus-compatible metrics through the
application's metrics implementation.

---

# 54. Important Metrics

Pricing metrics:

```text
pricing_requests_total

pricing_request_duration_seconds

pricing_cache_hits_total

pricing_cache_misses_total

pricing_cache_errors_total

pricing_database_queries_total

pricing_database_query_duration_seconds

pricing_calculation_errors_total
```

Business metrics:

```text
price_lookup_success

price_lookup_failure

price_rule_evaluation_count

price_override_count
```

---

# 55. Grafana Dashboard

```text
Pricing Service
│
├── Request Rate
│
├── Error Rate
│
├── P50
├── P95
└── P99
│
├── Redis
│   ├── Hit Ratio
│   ├── Miss Rate
│   ├── Latency
│   └── Connections
│
├── PostgreSQL
│   ├── Connections
│   ├── Query Latency
│   └── Cache Miss Load
│
├── Kubernetes
│   ├── Replicas
│   ├── CPU
│   ├── Memory
│   └── Restarts
│
└── Business
    ├── Price Lookup Success
    └── Pricing Errors
```

---

# 56. Structured Logs

Example:

```json
{
  "@timestamp": "2026-10-05T15:00:00Z",
  "level": "INFO",
  "service": "pricing-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "pricing-service-abc123",
  "version": "1.0.0",
  "correlationId": "corr-123",
  "productId": "P1001",
  "currency": "INR",
  "customerTier": "GOLD",
  "cacheHit": true,
  "message": "Price lookup completed"
}
```

Be careful with:

```text
Customer identifiers
Payment information
Authentication information
Sensitive pricing contracts
```

---

# 57. Distributed Tracing

Example:

```text
order-service
     |
     ↓
pricing-service
     |
     ├── Redis
     |
     └── PostgreSQL
```

Trace:

```text
Trace ID
  |
  ├── order request
  |
  ├── pricing calculation
  |
  ├── Redis GET
  |
  └── PostgreSQL SELECT
```

Jaeger allows the team to determine whether pricing latency comes from:

```text
application code
Redis
PostgreSQL
network
```

---

# 58. SLO

Example initial SLO:

## Availability

```text
99.95%
```

## Price Lookup P95

```text
< 150ms
```

## Price Lookup P99

```text
< 300ms
```

## Cache Availability

The application should continue functioning if Redis is temporarily
unavailable, although performance may degrade.

---

# 59. Redis Failure Strategy

If Redis fails:

```text
pricing-service
      |
      ↓
Redis failure
      |
      ↓
PostgreSQL
```

The application should not immediately return:

```text
HTTP 500
```

for every request.

However, uncontrolled fallback can overload PostgreSQL.

Therefore:

```text
Redis failure
     ↓
Cache miss
     ↓
Controlled DB fallback
     ↓
Timeout
     ↓
Circuit breaker/backpressure
```

---

# 60. Database Failure

If PostgreSQL fails:

```text
Redis HIT
```

may still allow cached prices to be returned for a short period depending on
business requirements.

But stale pricing can be dangerous.

Therefore pricing should define explicitly:

```text
acceptable staleness
```

For checkout:

> The final price should be validated against an authoritative pricing
> decision before order confirmation.

---

# 61. Pricing and Order Service

Example:

```text
Customer
   |
   ↓
Cart
   |
   ↓
Order
   |
   ├── Product
   ├── Pricing
   ├── Promotion
   └── Tax
```

Order should receive:

```text
unitPrice
quantity
discount
currency
pricingVersion
```

and store a snapshot.

---

# 62. Pricing Version

Example:

```text
Pricing version:
2026-10-05-v15
```

Order stores:

```text
productId
unitPrice
currency
pricingVersion
```

This helps auditing.

If a customer asks:

> Why was I charged ₹9,499?

the system can identify which pricing version produced that result.

---

# 63. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Pricing Service CI

on:

  pull_request:

    branches:
      - main

  push:

    branches:
      - main


permissions:

  contents: read

  id-token: write


env:

  AWS_REGION: ap-south-1

  ECR_REPOSITORY: pricing-service


jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Setup Python

        uses: actions/setup-python@v5

        with:

          python-version: "3.13"


      - name: Install dependencies

        run: |

          python -m pip install --upgrade pip

          pip install -e ".[test,lint]"


      - name: Lint

        run: ruff check .


      - name: Format check

        run: ruff format --check .


      - name: Unit tests

        run: pytest


  security:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Trivy filesystem scan

        uses: aquasecurity/trivy-action@master

        with:

          scan-type: fs

          scan-ref: .

          severity: HIGH,CRITICAL

          ignore-unfixed: true


  docker:

    needs:

      - test

      - security

    if: github.event_name == 'push'

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Configure AWS credentials

        uses: aws-actions/configure-aws-credentials@v4

        with:

          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}

          aws-region: ${{ env.AWS_REGION }}


      - name: Login to ECR

        id: ecr

        uses: aws-actions/amazon-ecr-login@v2


      - name: Build Docker image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .


      - name: Trivy image scan

        uses: aquasecurity/trivy-action@master

        with:

          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          severity: HIGH,CRITICAL

          exit-code: "1"

          ignore-unfixed: true


      - name: Push image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

---

# 64. SonarQube

CI flow:

```text
Checkout
   ↓
Install dependencies
   ↓
Tests
   ↓
Coverage
   ↓
SonarQube
   ↓
Security scans
   ↓
Docker build
```

SonarQube should be centrally managed.

The CI runner executes the scanner.

Do not install SonarQube inside every microservice Pod.

---

# 65. Trivy

Two important scans:

```text
Source
  ↓
Trivy filesystem
```

and:

```text
Docker image
  ↓
Trivy image
```

Then:

```text
ECR
```

ECR scanning provides another layer of protection.

---

# 66. ECR

Repository:

```text
company-ecommerce/pricing-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/pricing-service:<git-sha>
```

Prefer:

```text
Git SHA
```

or:

```text
immutable image digest
```

instead of:

```text
latest
```

---

# 67. GitOps Promotion

Application repository:

```text
pricing-service
```

GitOps repository:

```text
gitops
```

Flow:

```text
Developer
   ↓
GitHub PR
   ↓
CI
   ↓
Docker
   ↓
Trivy
   ↓
ECR
   ↓
GitOps update
   ↓
ArgoCD
   ↓
EKS
```

CI should not directly deploy to production with:

```bash
kubectl apply
```

in this GitOps architecture.

---

# 68. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: pricing-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/pricing-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 69. Failure Scenario — Redis Outage

Symptoms:

```text
Redis errors increase
Cache hit ratio drops
PostgreSQL reads increase
Pricing latency increases
```

Flow:

```text
Redis
  ↓
FAILURE
  ↓
Cache miss
  ↓
PostgreSQL
  ↓
Database pressure
```

Mitigation:

```text
Reduce cache fallback concurrency
+
Protect PostgreSQL
+
Fix Redis
+
Warm cache
```

Do not simply increase pricing-service replicas without checking PostgreSQL
capacity.

---

# 70. Failure Scenario — Stale Price

Redis:

```text
₹9,499
```

PostgreSQL:

```text
₹8,999
```

Possible causes:

- Cache invalidation failed
- Event processing failed
- TTL too long
- Incorrect cache key
- Deployment bug

Investigation:

```text
Pricing request
      ↓
Redis key
      ↓
Redis value
      ↓
PostgreSQL value
      ↓
Price update event
      ↓
Cache invalidation
```

---

# 71. Failure Scenario — PostgreSQL Connection Exhaustion

Potential cause:

```text
HPA
 ↓
3 Pods → 30 Pods
 ↓
Database pools multiply
 ↓
RDS connections exhausted
```

Example:

```text
pool_size = 10
max_overflow = 5

30 Pods

30 × 15 = 450 potential connections
```

Therefore:

```text
HPA maxReplicas
```

must be designed with:

```text
SQLAlchemy pool
+
RDS capacity
```

---

# 72. Failure Scenario — Incorrect Pricing Rule

Example:

Expected:

```text
Gold customer
₹9,000
```

Actual:

```text
₹7,000
```

Possible causes:

- Wrong rule priority
- Incorrect effective date
- Incorrect customer tier
- Cache contains stale rule
- Incorrect currency
- Bad deployment

Business correctness is a production reliability concern.

---

# 73. Example Production RCA

## Incident

```text
INC-2026-PRICING-001
```

## Severity

```text
SEV-2
```

## Title

```text
Stale Redis Pricing Cache Caused Incorrect Product Prices
```

## Date

```text
2026-10-05
```

---

# 74. Customer Impact

Some customers temporarily received an outdated product price during checkout.

No payment was captured using the incorrect final price because the order service
performed final pricing validation before order confirmation.

---

# 75. Timeline

```text
10:00
Pricing service healthy.

10:04
Pricing administrator updates product price.

10:05
PostgreSQL contains new price.

10:05
Redis cache invalidation event fails.

10:10
Customers continue receiving old cached price.

10:15
Customer support reports pricing discrepancy.

10:17
SRE identifies PostgreSQL and Redis values differ.

10:19
Cache invalidation failure identified.

10:21
Affected cache keys manually invalidated.

10:23
New prices appear correctly.

10:25
Event publisher issue identified.

10:30
Incident resolved.
```

---

# 76. Root Cause

The price update transaction successfully changed PostgreSQL.

However, the cache invalidation event was not published.

Therefore:

```text
PostgreSQL
₹8,999

Redis
₹9,499
```

remained inconsistent until TTL expiration.

---

# 77. Amplification

The cache TTL was:

```text
60 minutes
```

for that environment.

Therefore stale data remained available much longer than intended.

The absence of an invalidation alert delayed detection.

---

# 78. Five Whys

### Why did customers receive the old price?

Redis contained an outdated price.

### Why?

Cache invalidation did not occur.

### Why?

The price-change event was not successfully published.

### Why?

The event publisher failed and did not retry correctly.

### Why was the problem not detected?

There was no alert comparing cache invalidation failures against successful
price updates.

---

# 79. Corrective Actions

| Action | Owner |
|---|---|
| Implement transactional outbox | Backend |
| Add cache invalidation retry | Backend |
| Add invalidation failure metric | SRE |
| Add stale-price detection | SRE |
| Reduce cache TTL where appropriate | Architecture |
| Add pricing integration tests | Backend |
| Add pricing canary | Platform |
| Add cache consistency runbook | SRE |
| Validate final checkout price | Order team |
| Add pricing business SLO | SRE |

---

# 80. Security Checklist

- [ ] Secrets not in Git
- [ ] Secrets Manager
- [ ] External Secrets
- [ ] EKS Pod Identity
- [ ] Least-privilege IAM
- [ ] NetworkPolicy
- [ ] Private RDS
- [ ] Private Redis
- [ ] TLS where required
- [ ] Non-root container
- [ ] Read-only filesystem
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] No secrets in logs

---

# 81. Performance Checklist

- [ ] Redis caching
- [ ] Cache hit ratio monitored
- [ ] PostgreSQL indexes
- [ ] SQLAlchemy pool configured
- [ ] Connection budget calculated
- [ ] HPA tested
- [ ] Cache stampede tested
- [ ] Redis failure tested
- [ ] PostgreSQL failure tested
- [ ] Load testing
- [ ] Spike testing
- [ ] Soak testing
- [ ] P95/P99 monitored

---

# 82. Production Checklist

## Application

- [ ] FastAPI
- [ ] Pydantic
- [ ] SQLAlchemy
- [ ] Decimal for money
- [ ] Validation
- [ ] Error handling
- [ ] Unit tests
- [ ] Integration tests

## Pricing

- [ ] Price ownership
- [ ] Effective dates
- [ ] Currency
- [ ] Customer tiers
- [ ] Pricing rules
- [ ] Price version
- [ ] Rounding policy
- [ ] Cache invalidation
- [ ] Price snapshot

## Database

- [ ] PostgreSQL
- [ ] RDS
- [ ] Indexes
- [ ] Connection pool
- [ ] Backups
- [ ] PITR
- [ ] Private networking

## Redis

- [ ] Cache
- [ ] TTL
- [ ] Cache invalidation
- [ ] Stampede protection
- [ ] Hit ratio
- [ ] Failure fallback

## Kubernetes

- [ ] Deployment
- [ ] Service
- [ ] ConfigMap
- [ ] ExternalSecret
- [ ] ServiceAccount
- [ ] NetworkPolicy
- [ ] HPA
- [ ] PDB
- [ ] Resource requests
- [ ] Resource limits
- [ ] Probes

## CI/CD

- [ ] Tests
- [ ] Ruff
- [ ] SonarQube
- [ ] Dependency scan
- [ ] Secret scan
- [ ] Trivy
- [ ] Docker build
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] GitOps repository
- [ ] ArgoCD
- [ ] Immutable image
- [ ] Environment promotion
- [ ] Rollback

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] ELK
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Correlation ID
- [ ] Alerts
- [ ] SLO
- [ ] Runbook
- [ ] RCA

---

# 83. Final Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                     pricing-service
                       /          \
                      /            \
                     ↓              ↓
                  Redis          PostgreSQL
                     |               |
                     |               |
                     └───────┬───────┘
                             |
                        Price Rules


Observability:

pricing-service
      |
      ├── Prometheus → Grafana
      |
      ├── stdout → Fluent Bit → Logstash → Elasticsearch → Kibana
      |
      └── OpenTelemetry → Collector → Jaeger
```

---

# 84. Final DevOps Perspective

Pricing demonstrates a different scaling model from inventory.

Inventory:

```text
High concurrency
+
Database writes
+
Row contention
+
Transactions
```

Pricing:

```text
High read traffic
+
Redis
+
PostgreSQL fallback
+
Horizontal scaling
```

Therefore the infrastructure strategy differs.

For pricing:

```text
Traffic
   ↓
ALB
   ↓
HPA
   ↓
Pricing Pods
   ↓
Redis
   ↓
PostgreSQL
```

The key production lesson is:

> Scaling the application is useful only when the dependencies can handle the
> additional traffic.

# End of 167 — Pricing Service
```

### Phase 19 progress

```text
161 Auth Service            ✅
162 User Service            ✅
163 Customer Service        ✅
164 Catalog Service         ✅
165 Product Service         ✅
166 Inventory Service       ✅
167 Pricing Service         ✅

168 Promotion Service       ⏳
169 Cart Service            ⏳
170 Order Service           ⏳
171 Payment Service         ⏳
172 Fraud Service           ⏳
173 Shipping Service        ⏳
174 Delivery Service        ⏳
175 Notification Service    ⏳
176 Search Service          ⏳
177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `168-Promotion-Service.md` — Node.js + TypeScript + NestJS, with promotion rules, coupon validation, Redis caching, PostgreSQL, concurrency protection, CI/CD, Kubernetes, Helm, GitOps, observability, and a production RCA.**