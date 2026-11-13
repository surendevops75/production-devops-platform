# 162 — User Service

## 1. Overview

The `user-service` manages the user's profile and account information after authentication.

The service is intentionally separated from `auth-service`.

`auth-service` answers:

> "Who are you and can you authenticate?"

`user-service` answers:

> "What information belongs to this user?"

The service is implemented using:

- Python 3.12
- FastAPI
- Pydantic
- SQLAlchemy
- PostgreSQL
- Redis
- Kafka
- Docker
- Kubernetes
- Helm
- AWS EKS
- Amazon ECR
- GitHub Actions
- ArgoCD
- Prometheus
- Grafana
- ELK
- OpenTelemetry
- Jaeger

---

# 2. Responsibility

The user-service owns:

- User profile
- First name
- Last name
- Email profile information
- Phone number
- User preferences
- Account metadata
- Profile update
- User lookup
- User lifecycle events

It does NOT own:

- Password authentication
- Password hashing
- Access-token generation
- Refresh-token management
- Orders
- Payments
- Customer-specific business workflows

Architecture:

```text
auth-service
     |
     | userId
     ↓
user-service
     |
     ├── Profile
     ├── Preferences
     └── Account metadata
```

---

# 3. Technology Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Framework | FastAPI |
| Validation | Pydantic |
| ORM | SQLAlchemy |
| Database | PostgreSQL |
| DB driver | psycopg |
| Cache | Redis |
| Messaging | Kafka |
| Dependency management | uv |
| Testing | pytest |
| Linting | Ruff |
| Type checking | mypy |
| Container | Docker |
| Orchestration | Kubernetes/EKS |
| Packaging | Helm |
| Registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| Metrics | Prometheus |
| Logs | ELK |
| Tracing | OpenTelemetry |
| Tracing backend | Jaeger |

---

# 4. Architecture

```text
                    Internet
                       |
                CloudFront / WAF
                       |
                      ALB
                       |
                 user-service
                       |
          ┌────────────┼────────────┐
          │            │            │
          ↓            ↓            ↓
     PostgreSQL      Redis        Kafka
       profile       cache       events
```

Request flow:

```text
Client
  |
  ↓
ALB
  |
  ↓
user-service
  |
  ├── Validate request
  |
  ├── Validate authenticated user
  |
  ├── Redis cache
  |
  └── PostgreSQL
```

---

# 5. API Design

## Get User

```http
GET /api/v1/users/{userId}
```

## Update User

```http
PUT /api/v1/users/{userId}
```

## Get Current User

```http
GET /api/v1/users/me
```

## Update Preferences

```http
PUT /api/v1/users/{userId}/preferences
```

## Health

```http
GET /actuator/health
```

For FastAPI we will expose:

```http
GET /health
GET /ready
```

and:

```http
GET /metrics
```

---

# 6. Repository Structure

```text
user-service/
│
├── app/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── users.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── user.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── user.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   └── user_service.py
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── user_repository.py
│   │
│   ├── database/
│   │   ├── __init__.py
│   │   └── database.py
│   │
│   ├── cache/
│   │   ├── __init__.py
│   │   └── redis.py
│   │
│   ├── messaging/
│   │   ├── __init__.py
│   │   └── kafka.py
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   └── settings.py
│   │
│   └── observability/
│       ├── __init__.py
│       └── metrics.py
│
├── tests/
│   ├── test_users.py
│   └── test_health.py
│
├── migrations/
│
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── .env.example
├── README.md
│
├── helm/
│   └── user-service/
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
│           ├── serviceaccount.yaml
│           ├── externalsecret.yaml
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

# 7. Why FastAPI?

FastAPI is useful for this service because it provides:

- High-performance HTTP APIs
- Python type hints
- Pydantic validation
- Automatic OpenAPI documentation
- Async request handling
- Simple dependency injection
- Good Kubernetes compatibility

Development flow:

```text
Python
   ↓
FastAPI
   ↓
API
   ↓
SQLAlchemy
   ↓
PostgreSQL
```

---

# 8. Python Dependency Management

We will use `uv`.

The mental model is:

```text
Python
  ↓
uv
  ↓
Dependencies
  ↓
Application
  ↓
Docker
```

Initialize:

```bash
uv init
```

Install dependencies:

```bash
uv add fastapi
uv add uvicorn
uv add sqlalchemy
uv add psycopg
uv add pydantic-settings
uv add redis
uv add aiokafka
uv add prometheus-fastapi-instrumentator
```

Development dependencies:

```bash
uv add --dev pytest
uv add --dev httpx
uv add --dev ruff
uv add --dev mypy
```

---

# 9. `pyproject.toml`

```toml
[project]
name = "user-service"
version = "1.0.0"
description = "User profile service for ecommerce platform"
requires-python = ">=3.12"

dependencies = [
    "fastapi>=0.115,<1.0",
    "uvicorn[standard]>=0.34,<1.0",
    "sqlalchemy>=2.0,<3.0",
    "psycopg[binary]>=3.2,<4.0",
    "pydantic-settings>=2.7,<3.0",
    "redis>=5.2,<6.0",
    "aiokafka>=0.12,<1.0",
    "prometheus-fastapi-instrumentator>=7.0,<8.0",
]

[dependency-groups]
dev = [
    "pytest>=8.3,<9.0",
    "httpx>=0.28,<1.0",
    "ruff>=0.9,<1.0",
    "mypy>=1.14,<2.0",
]

[tool.ruff]
line-length = 100

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.mypy]
python_version = "3.12"
strict = true
```

Exact dependency versions should be pinned in `uv.lock` and updated through the organization's dependency-management process.

---

# 10. Configuration

## `app/config/settings.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):

    app_name: str = "user-service"

    environment: str = "local"

    database_url: str = (
        "postgresql+psycopg://postgres:postgres@localhost:5432/users"
    )

    redis_host: str = "localhost"

    redis_port: int = 6379

    kafka_bootstrap_servers: str = "localhost:9092"

    db_pool_size: int = 10

    db_max_overflow: int = 5

    db_pool_timeout: int = 3

    model_config = SettingsConfigDict(
        env_file=".env",
        extra="ignore",
    )


settings = Settings()
```

---

# 11. Database Configuration

## `app/database/database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

from app.config.settings import settings


class Base(DeclarativeBase):
    pass


engine = create_engine(
    settings.database_url,
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
    pool_timeout=settings.db_pool_timeout,
    pool_pre_ping=True,
    pool_recycle=1500,
)


SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
)


def get_db():
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()
```

Important:

Python's SQLAlchemy configuration is different from Java HikariCP.

The capacity calculation becomes:

```text
Connections per process
=
pool_size + max_overflow
```

So:

```text
10 + 5 = 15
```

potential connections per process under the configured limits.

If Kubernetes runs:

```text
10 Pods
```

the theoretical maximum is approximately:

```text
10 × 15 = 150 connections
```

before considering other clients.

This must be included in our RDS connection budget.

---

# 12. User Model

## `app/models/user.py`

```python
from datetime import datetime
from uuid import UUID, uuid4

from sqlalchemy import DateTime, String, func
from sqlalchemy.dialects.postgresql import UUID as PGUUID
from sqlalchemy.orm import Mapped, mapped_column

from app.database.database import Base


class User(Base):

    __tablename__ = "users"

    id: Mapped[int] = mapped_column(
        primary_key=True,
        autoincrement=True,
    )

    user_id: Mapped[UUID] = mapped_column(
        PGUUID(as_uuid=True),
        unique=True,
        nullable=False,
        default=uuid4,
    )

    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        nullable=False,
    )

    first_name: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    last_name: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    phone: Mapped[str | None] = mapped_column(
        String(30),
        nullable=True,
    )

    status: Mapped[str] = mapped_column(
        String(30),
        nullable=False,
        default="ACTIVE",
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )

    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )
```

---

# 13. API Schemas

## `app/schemas/user.py`

```python
from uuid import UUID

from pydantic import BaseModel, EmailStr, ConfigDict


class UserResponse(BaseModel):

    model_config = ConfigDict(from_attributes=True)

    user_id: UUID

    email: EmailStr

    first_name: str

    last_name: str

    phone: str | None

    status: str


class UserUpdateRequest(BaseModel):

    first_name: str | None = None

    last_name: str | None = None

    phone: str | None = None
```

---

# 14. Repository

## `app/repositories/user_repository.py`

```python
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.user import User


class UserRepository:

    def __init__(self, db: Session):
        self.db = db

    def find_by_user_id(self, user_id: UUID) -> User | None:

        statement = select(User).where(
            User.user_id == user_id
        )

        return self.db.scalar(statement)

    def find_by_email(self, email: str) -> User | None:

        statement = select(User).where(
            User.email == email
        )

        return self.db.scalar(statement)

    def save(self, user: User) -> User:

        self.db.add(user)

        self.db.commit()

        self.db.refresh(user)

        return user
```

---

# 15. User Service

## `app/services/user_service.py`

```python
from uuid import UUID

from sqlalchemy.orm import Session

from app.models.user import User
from app.repositories.user_repository import UserRepository
from app.schemas.user import UserUpdateRequest


class UserNotFound(Exception):
    pass


class UserService:

    def __init__(self, db: Session):

        self.repository = UserRepository(db)

    def get_user(self, user_id: UUID) -> User:

        user = self.repository.find_by_user_id(user_id)

        if user is None:
            raise UserNotFound(
                f"User {user_id} not found"
            )

        return user

    def update_user(
        self,
        user_id: UUID,
        request: UserUpdateRequest,
    ) -> User:

        user = self.get_user(user_id)

        if request.first_name is not None:
            user.first_name = request.first_name

        if request.last_name is not None:
            user.last_name = request.last_name

        if request.phone is not None:
            user.phone = request.phone

        return self.repository.save(user)
```

---

# 16. API Controller

## `app/api/users.py`

```python
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.database.database import get_db
from app.schemas.user import UserResponse, UserUpdateRequest
from app.services.user_service import UserNotFound, UserService


router = APIRouter(
    prefix="/api/v1/users",
    tags=["users"],
)


@router.get(
    "/{user_id}",
    response_model=UserResponse,
)
def get_user(
    user_id: UUID,
    db: Session = Depends(get_db),
):

    service = UserService(db)

    try:

        return service.get_user(user_id)

    except UserNotFound as exc:

        raise HTTPException(
            status_code=404,
            detail=str(exc),
        ) from exc


@router.put(
    "/{user_id}",
    response_model=UserResponse,
)
def update_user(
    user_id: UUID,
    request: UserUpdateRequest,
    db: Session = Depends(get_db),
):

    service = UserService(db)

    try:

        return service.update_user(
            user_id,
            request,
        )

    except UserNotFound as exc:

        raise HTTPException(
            status_code=404,
            detail=str(exc),
        ) from exc
```

---

# 17. FastAPI Application

## `app/main.py`

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

from app.api.users import router as users_router


app = FastAPI(
    title="User Service",
    version="1.0.0",
)


app.include_router(users_router)


@app.get("/health")
def health():
    return {
        "status": "UP",
    }


@app.get("/ready")
def readiness():
    return {
        "status": "READY",
    }


Instrumentator().instrument(app).expose(
    app,
    endpoint="/metrics",
)
```

---

# 18. Database Migration

For production we use controlled migrations rather than automatically changing the schema when the application starts.

Example migration:

## `migrations/V1__create_users.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,

    user_id UUID NOT NULL UNIQUE DEFAULT uuid_generate_v4(),

    email VARCHAR(255) NOT NULL UNIQUE,

    first_name VARCHAR(100) NOT NULL,

    last_name VARCHAR(100) NOT NULL,

    phone VARCHAR(30),

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_user_id
ON users(user_id);

CREATE INDEX idx_users_email
ON users(email);
```

In the full production implementation, the migration mechanism will be standardized with the database migration strategy already established in the project.

---

# 19. Redis

Redis is useful for frequently accessed profiles.

Pattern:

```text
GET user
   |
   ↓
Redis
   |
   ├── HIT → return
   |
   └── MISS
        |
        ↓
     PostgreSQL
        |
        ↓
      Redis
        |
        ↓
      return
```

Example key:

```text
user:profile:<userId>
```

TTL:

```text
5–15 minutes
```

The exact TTL must be based on access patterns.

PostgreSQL remains the source of truth.

---

# 20. Redis Client

## `app/cache/redis.py`

```python
import redis

from app.config.settings import settings


redis_client = redis.Redis(
    host=settings.redis_host,
    port=settings.redis_port,
    decode_responses=True,
    socket_connect_timeout=1,
    socket_timeout=1,
)
```

Short timeouts are intentional.

We don't want an unavailable Redis instance to hold API requests for a long time.

---

# 21. Cache Failure Principle

Bad design:

```text
Redis failure
     ↓
Retry 100 times
     ↓
Request waits 30 seconds
     ↓
Database fallback
     ↓
RDS overload
```

Better:

```text
Redis failure
     ↓
Short timeout
     ↓
Controlled fallback
     ↓
PostgreSQL
     ↓
Bounded concurrency
```

This follows the same protection principles used in the Redis incident chapter.

---

# 22. Kafka Events

The user-service may publish:

```text
UserProfileCreated
UserProfileUpdated
UserStatusChanged
```

Example:

```json
{
  "eventId": "01J...",
  "eventType": "UserProfileUpdated",
  "version": 1,
  "occurredAt": "2026-10-05T10:00:00Z",
  "source": "user-service",
  "correlationId": "abc-123",
  "payload": {
    "userId": "..."
  }
}
```

Do not publish sensitive profile information unnecessarily.

---

# 23. Dockerfile

## `Dockerfile`

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /build

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

RUN pip install --no-cache-dir uv

COPY pyproject.toml uv.lock ./

RUN uv sync --frozen --no-dev

COPY app ./app


FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/app/.venv/bin:$PATH"

COPY --from=builder /build/.venv /app/.venv

COPY --from=builder /build/app /app/app

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

---

# 24. `.dockerignore`

```text
.git
.github
.gitignore

.venv
__pycache__
*.pyc

.pytest_cache
.mypy_cache
.ruff_cache

.env
.env.*

tests

README.md
```

---

# 25. Local Environment

## `.env.example`

```text
APP_NAME=user-service

ENVIRONMENT=local

DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/users

REDIS_HOST=localhost
REDIS_PORT=6379

KAFKA_BOOTSTRAP_SERVERS=localhost:9092

DB_POOL_SIZE=10
DB_MAX_OVERFLOW=5
DB_POOL_TIMEOUT=3
```

Never commit the actual `.env`.

---

# 26. Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: user-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: users

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
          "pg_isready -U postgres -d users"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: user-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  user-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: user-service

    restart: unless-stopped

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

    environment:

      ENVIRONMENT: local

      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/users

      REDIS_HOST: redis

      REDIS_PORT: 6379

      DB_POOL_SIZE: 10

      DB_MAX_OVERFLOW: 5

    ports:

      - "8080:8080"

volumes:

  postgres-data:
```

---

# 27. Local Test

Install dependencies:

```bash
uv sync
```

Run:

```bash
uv run uvicorn app.main:app --reload --port 8080
```

Health:

```bash
curl http://localhost:8080/health
```

Readiness:

```bash
curl http://localhost:8080/ready
```

Metrics:

```bash
curl http://localhost:8080/metrics
```

API documentation:

```text
http://localhost:8080/docs
```

---

# 28. Unit Tests

## `tests/test_health.py`

```python
from fastapi.testclient import TestClient

from app.main import app


client = TestClient(app)


def test_health():

    response = client.get("/health")

    assert response.status_code == 200

    assert response.json()["status"] == "UP"


def test_readiness():

    response = client.get("/ready")

    assert response.status_code == 200

    assert response.json()["status"] == "READY"
```

Run:

```bash
uv run pytest
```

---

# 29. Ruff

Run:

```bash
uv run ruff check .
```

Format:

```bash
uv run ruff format .
```

---

# 30. Mypy

Run:

```bash
uv run mypy app
```

---

# 31. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: user-service

description: User profile service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 32. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/user-service

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

  minReplicas: 2

  maxReplicas: 10

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: user-service


podDisruptionBudget:

  enabled: true

  minAvailable: 1


secretStore:

  name: ecommerce-secrets
```

---

# 33. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/user-service

  tag: ""

  digest: ""


resources:

  requests:

    cpu: 500m

    memory: 512Mi

  limits:

    cpu: "1"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

---

# 34. Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: user-service

  labels:

    app: user-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: user-service

  template:

    metadata:

      labels:

        app: user-service

    spec:

      serviceAccountName: {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: user-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: user-service-config

            - secretRef:

                name: user-service-secrets

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

# 35. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: user-service

  labels:

    app: user-service

spec:

  type: ClusterIP

  selector:

    app: user-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal service discovery:

```text
user-service.ecommerce.svc.cluster.local
```

---

# 36. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: user-service-config

data:

  ENVIRONMENT: prod

  DB_POOL_SIZE: "10"

  DB_MAX_OVERFLOW: "5"

  DB_POOL_TIMEOUT: "3"

  REDIS_PORT: "6379"
```

---

# 37. ExternalSecret

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: user-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: {{ .Values.secretStore.name }}

    kind: ClusterSecretStore

  target:

    name: user-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/user-service

        property: DATABASE_URL
```

Redis and Kafka credentials, if required by the selected managed/self-hosted architecture, should also come from the appropriate secret mechanism.

---

# 38. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: user-service
```

If the service needs AWS APIs:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
AWS API
```

Only required permissions should be granted.

---

# 39. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: user-service

spec:

  podSelector:

    matchLabels:

      app: user-service

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

              kubernetes.io/metadata.name: ecommerce

      ports:

        - protocol: TCP

          port: 5432

        - protocol: TCP

          port: 6379

        - protocol: TCP

          port: 9092

    - to:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: kube-system

      ports:

        - protocol: UDP

          port: 53

        - protocol: TCP

          port: 53
```

---

# 40. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: user-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: user-service

  minReplicas: {{ .Values.autoscaling.minReplicas }}

  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

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

---

# 41. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: user-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: user-service
```

---

# 42. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: user-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: user-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 43. GitHub CODEOWNERS

## `.github/CODEOWNERS`

```text
/app/       @company-ecommerce/backend-team
/helm/      @company-ecommerce/platform-team
/.github/   @company-ecommerce/platform-team
/security/  @company-ecommerce/security-team
```

---

# 44. GitHub Actions CI

## `.github/workflows/ci.yaml`

```yaml
name: User Service CI

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

  ECR_REPOSITORY: user-service


jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Install uv

        uses: astral-sh/setup-uv@v5

        with:

          version: latest


      - name: Set up Python

        run: uv python install 3.12


      - name: Install dependencies

        run: uv sync --frozen


      - name: Ruff

        run: uv run ruff check .


      - name: Ruff Format Check

        run: uv run ruff format --check .


      - name: Mypy

        run: uv run mypy app


      - name: Pytest

        run: uv run pytest


  filesystem-security:

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

      - filesystem-security

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


      - name: Build image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |
          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .


      - name: Scan image

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


      - name: Read image digest

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |
          aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="${GITHUB_SHA}" \
            --query 'imageDetails[0].imageDigest' \
            --output text
```

---

# 45. SonarQube

The Python pipeline should analyze:

```text
Python source
Tests
Coverage
Code quality
Security findings
Maintainability
```

The SonarQube scanner configuration should be maintained according to the organization's SonarQube version and approved CI integration.

Example project properties:

```properties
sonar.projectKey=user-service
sonar.projectName=user-service
sonar.sources=app
sonar.tests=tests
```

---

# 46. Trivy

We scan at two stages:

```text
Source
 ↓
Trivy filesystem scan

Docker image
 ↓
Trivy image scan
```

This provides defense in depth.

ECR scanning remains an additional registry-side security control.

---

# 47. ECR

Repository:

```text
company-ecommerce/user-service
```

Example image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/user-service:7d9a1c4
```

Production deployment should preferably reference the immutable digest:

```text
user-service@sha256:...
```

---

# 48. GitOps

The application repository is responsible for:

```text
Source code
Dockerfile
CI
Application implementation
```

The GitOps repository is responsible for:

```text
Helm deployment configuration
Environment configuration
Image version/digest
ArgoCD
```

Flow:

```text
user-service repository
        |
        ↓
GitHub Actions
        |
        ↓
ECR
        |
        ↓
GitOps PR
        |
        ↓
Approval
        |
        ↓
ArgoCD
        |
        ↓
EKS
```

---

# 49. Observability

## Metrics

Important metrics:

```text
HTTP request rate
HTTP error rate
HTTP latency
Python process CPU
Python process memory
Database pool usage
Redis latency
Kafka producer/consumer metrics
```

Application metrics:

```text
user_profile_get_total
user_profile_update_total
user_profile_not_found_total
user_profile_update_failure_total
```

Do not use:

```text
user_id
email
phone
```

as Prometheus labels.

---

# 50. Logging

Example:

```json
{
  "@timestamp": "2026-10-05T11:00:00Z",
  "level": "INFO",
  "service": "user-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "user-service-7d9c8f-x7abc",
  "correlationId": "abc-123",
  "message": "User profile retrieved"
}
```

Never log sensitive information unnecessarily.

---

# 51. Distributed Tracing

```text
Client
  |
  ↓
ALB
  |
  ↓
user-service
  |
  ├── PostgreSQL
  |
  ├── Redis
  |
  └── Kafka
```

OpenTelemetry:

```text
user-service
     ↓
OpenTelemetry instrumentation
     ↓
OTel Collector
     ↓
Jaeger
```

The trace should contain:

```text
HTTP request
Database query
Redis operation
Kafka operation
```

where instrumentation supports those operations.

---

# 52. Grafana Dashboard

User-service dashboard:

```text
HTTP
 ├── Requests/sec
 ├── P50
 ├── P95
 ├── P99
 ├── 4xx
 └── 5xx

Application
 ├── Python process CPU
 ├── Python process memory
 └── worker/process behavior

Database
 ├── Pool size
 ├── Active connections
 ├── Overflow
 ├── Wait time
 └── Query latency

Redis
 ├── Hit ratio
 ├── Latency
 ├── Errors
 └── Connections

Kubernetes
 ├── Pods
 ├── Restarts
 ├── CPU
 └── Memory
```

---

# 53. SLO

Example starting targets:

### Availability

```text
99.9%
```

### GET profile P95

```text
< 200ms
```

### Profile update P95

```text
< 300ms
```

These are initial simulation targets.

Production targets must be based on:

```text
Load testing
Business requirements
Dependency latency
Database capacity
Historical traffic
```

---

# 54. Failure Scenario — PostgreSQL

```text
User request
    ↓
user-service
    ↓
PostgreSQL unavailable
```

Investigate:

```bash
kubectl logs <pod> -n ecommerce
```

Then:

```text
RDS
 ↓
DNS
 ↓
Network
 ↓
Security Group
 ↓
Credentials
 ↓
Hikari/SQLAlchemy pool
 ↓
Database connections
```

Do not immediately increase the database connection pool.

---

# 55. Failure Scenario — Redis

```text
Redis unavailable
     ↓
Cache miss
     ↓
PostgreSQL
```

Protect PostgreSQL with:

```text
Short Redis timeout
+
Bounded retry
+
Request concurrency limit
+
Cache stampede protection
```

---

# 56. Failure Scenario — High Traffic

```text
Traffic ↑
   ↓
Pods ↑
   ↓
Database connections ↑
```

Because SQLAlchemy has:

```text
pool_size + max_overflow
```

we must calculate:

```text
Potential connections
=
Pods × (pool_size + max_overflow)
```

Example:

```text
15 Pods × (10 + 5)
=
225 potential connections
```

This must be compared against the safe RDS connection budget.

---

# 57. Example Production Incident

## Incident ID

```text
INC-2026-USER-001
```

## Title

```text
User Profile API Latency Increased During Traffic Spike
```

## Severity

```text
SEV-2
```

---

# 58. Impact

Customers experienced:

```text
Slow profile loading
Increased API latency
Some 5xx responses
```

Authentication itself was healthy.

---

# 59. Timeline

```text
14:00
Normal traffic.

14:05
Traffic increases significantly.

14:07
HPA scales user-service.

14:09
SQLAlchemy active connections increase.

14:10
RDS connection count approaches safe threshold.

14:11
Database query latency increases.

14:12
User-service P95 reaches 1.5 seconds.

14:13
5xx begins increasing.

14:14
SEV-2 declared.

14:16
Platform team identifies excessive application DB connections.

14:18
Noncritical traffic reduced.

14:20
HPA max replicas temporarily reduced.

14:22
RDS connections stabilize.

14:24
Latency begins recovering.

14:27
5xx returns to normal.

14:30
Incident resolved.
```

---

# 60. Root Cause

The immediate problem was:

```text
RDS connection pressure
```

The amplification mechanism was:

```text
Traffic increase
      ↓
HPA scales Pods
      ↓
Each Pod has SQLAlchemy pool
      ↓
pool_size + max_overflow
      ↓
Total DB connections increase
      ↓
RDS connection pressure
      ↓
Query latency
      ↓
API latency
```

---

# 61. Five Whys

### Why was the API slow?

Database queries were waiting longer.

### Why?

RDS was under connection pressure.

### Why?

More user-service Pods were created.

### Why?

HPA scaled during a traffic spike.

### Why?

The HPA maximum and database connection budget had not been jointly capacity-tested.

---

# 62. Corrective Actions

| Action | Owner |
|---|---|
| Recalculate SQLAlchemy pool | Backend |
| Define service DB connection budget | Platform |
| Align HPA max with DB capacity | Platform |
| Add DB connection alerts | SRE |
| Add load test | Performance |
| Add connection saturation dashboard | SRE |
| Review cache hit ratio | Backend |
| Optimize expensive profile queries | Backend |
| Add capacity test to release process | Platform |

---

# 63. Production Troubleshooting

## Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=user-service
```

## Logs

```bash
kubectl logs \
  -n ecommerce \
  <pod-name>
```

## Previous logs

```bash
kubectl logs \
  -n ecommerce \
  <pod-name> \
  --previous
```

## HPA

```bash
kubectl get hpa \
  -n ecommerce \
  user-service
```

## Deployment

```bash
kubectl get deployment \
  -n ecommerce \
  user-service
```

## Endpoints

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=user-service
```

## Network

```bash
kubectl exec \
  -n ecommerce \
  <pod-name> \
  -- getent hosts user-service
```

---

# 64. CI/CD Flow

```text
Developer
    ↓
Feature branch
    ↓
Pull Request
    ↓
GitHub Actions
    |
    ├── uv
    ├── Ruff
    ├── mypy
    ├── pytest
    ├── SonarQube
    ├── Dependency scan
    ├── Secret scan
    ├── Docker build
    └── Trivy
    ↓
ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
EKS
```

---

# 65. Security Checklist

- [ ] No secrets in Git
- [ ] No `.env` committed
- [ ] Least-privilege IAM
- [ ] EKS Pod Identity where AWS access is required
- [ ] NetworkPolicy
- [ ] Non-root container
- [ ] Read-only root filesystem where compatible
- [ ] Dependency scanning
- [ ] Trivy
- [ ] SonarQube
- [ ] No sensitive information in logs
- [ ] PostgreSQL private
- [ ] Redis private
- [ ] TLS for production dependencies where required
- [ ] Authentication handled by auth-service
- [ ] Authorization enforced at appropriate service boundaries

---

# 66. Production Checklist

## Application

- [ ] FastAPI
- [ ] Pydantic validation
- [ ] SQLAlchemy
- [ ] Error handling
- [ ] Unit tests
- [ ] Integration tests
- [ ] OpenAPI
- [ ] Health endpoint
- [ ] Readiness endpoint

## Database

- [ ] PostgreSQL
- [ ] Migration strategy
- [ ] Connection pool configured
- [ ] Connection budget calculated
- [ ] Backup/PITR
- [ ] Private networking

## Redis

- [ ] Cache-aside
- [ ] TTL
- [ ] Short timeout
- [ ] Bounded retry
- [ ] Stampede protection
- [ ] Failure fallback

## Kafka

- [ ] Event contract
- [ ] Correlation ID
- [ ] Event ID
- [ ] Versioning
- [ ] Idempotency

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

- [ ] uv dependency lock
- [ ] Ruff
- [ ] mypy
- [ ] pytest
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] Docker
- [ ] Trivy
- [ ] ECR
- [ ] GitHub OIDC

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] ELK
- [ ] Correlation ID
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Alerts
- [ ] SLO
- [ ] Runbook

---

# 67. Final Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                      user-service
                            |
            ┌───────────────┼────────────────┐
            │               │                │
            ↓               ↓                ↓
       PostgreSQL         Redis             Kafka
            │               │                │
            │               │                │
            └───────────────┼────────────────┘
                            |
                       User Profile
                            |
                 ┌──────────┴──────────┐
                 │                     │
             Prometheus          OpenTelemetry
                 │                     │
              Grafana              Collector
                                       |
                                     Jaeger

Application logs
       |
       ↓
  Fluent Bit
       |
       ↓
   Logstash
       |
       ↓
 Elasticsearch
       |
       ↓
     Kibana
```

---

# 68. DevOps Learning

This service demonstrates the difference between application technologies and platform technologies.

Application layer:

```text
Python
FastAPI
Pydantic
SQLAlchemy
```

Platform layer:

```text
Docker
Kubernetes
EKS
Helm
ECR
GitHub Actions
ArgoCD
Terraform
```

Operations layer:

```text
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
SLO
Incident Management
```

The language changed from Java in `161-Auth-Service` to Python here, but our production platform remains standardized.

That is the objective of the polyglot microservices architecture.
```

### Phase 19 progress

```text
161 Auth Service          ✅ Java / Spring Boot
162 User Service          ✅ Python / FastAPI

163 Customer Service      ⏳ Node.js / TypeScript / NestJS
164 Catalog Service       ⏳ Go / Gin
165 Product Service       ⏳ Java / Spring Boot
166 Inventory Service     ⏳ Go / Gin
167 Pricing Service       ⏳ Python / FastAPI
168 Promotion Service     ⏳ Node.js / TypeScript / NestJS
169 Cart Service          ⏳ Java / Spring Boot
170 Order Service         ⏳ Java / Spring Boot
171 Payment Service       ⏳ C# / ASP.NET Core
172 Fraud Service         ⏳ Python / FastAPI
173 Shipping Service      ⏳ Node.js / TypeScript / NestJS
174 Delivery Service      ⏳ Go / Gin
175 Notification Service  ⏳ Python / FastAPI
176 Search Service        ⏳ Java / Spring Boot
177 Recommendation Service⏳ Python / FastAPI
178 Review Service        ⏳ Ruby / Rails
179 Tax Service           ⏳ C# / ASP.NET Core
180 Invoice Service       ⏳ Node.js / TypeScript / NestJS
181 Reporting Service     ⏳ Python / FastAPI
```

**Next: `163-Customer-Service.md` — Node.js + TypeScript + NestJS.**