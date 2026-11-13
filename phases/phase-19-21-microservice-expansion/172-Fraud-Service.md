# 172 — Fraud Service

**File:** `172-Fraud-Service.md`

```markdown
# 172 — Fraud Service

## 1. Overview

The `fraud-service` evaluates transactions and customer activity for potential
fraud before an order is confirmed or a payment is captured.

The service does not decide whether an order exists and it does not own
payment processing.

Its responsibility is:

```text
Risk signals
     ↓
Fraud evaluation
     ↓
Risk score
     ↓
ALLOW / REVIEW / BLOCK
```

Technology:

- Python 3.13
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

Python is deliberately used here because fraud/risk services commonly benefit
from Python's ecosystem for rules, data processing, experimentation, and
future ML integration.

---

# 2. Business Responsibility

Fraud Service owns:

- Fraud risk evaluation
- Risk rules
- Risk score
- Fraud decision
- Review state
- Fraud evaluation history
- Risk signals
- Fraud events
- Idempotency of evaluations

It does not own:

- Customer identity
- Product catalog
- Inventory
- Payment transactions
- Order lifecycle
- Shipping

Those remain owned by their respective services.

---

# 3. Why Fraud Is a Separate Service

Fraud logic changes frequently.

Examples:

```text
New card + new customer
+
unusually large order
+
many payment attempts
+
suspicious IP
```

may increase risk.

Keeping this logic inside `order-service` or `payment-service` would create
tight coupling.

Instead:

```text
Order
   |
   ↓
Fraud Service
   |
   ↓
Risk Decision
```

This allows fraud rules to evolve independently.

---

# 4. Fraud Decision

The service produces three primary decisions:

```text
ALLOW
REVIEW
BLOCK
```

Example:

```text
Risk Score

0 - 39
   ↓
ALLOW

40 - 69
   ↓
REVIEW

70 - 100
   ↓
BLOCK
```

These thresholds are illustrative.

Real thresholds must be determined from business requirements, historical
fraud data, false-positive analysis, and controlled testing.

---

# 5. High-Level Architecture

```text
                         Order Service
                              |
                              ↓
                        Fraud Service
                       /      |       \
                      /       |        \
                     ↓        ↓         ↓
                PostgreSQL  Redis      Kafka
                     |
                     ↓
              Fraud Decisions
                     |
              ┌──────┼────────┐
              ↓      ↓        ↓
            ALLOW  REVIEW    BLOCK
```

Additional signals:

```text
Payment Service
Customer Service
Order Service
Device/Risk Provider
IP intelligence
Velocity counters
Historical behavior
```

---

# 6. Fraud Evaluation Flow

```text
Order Created
     |
     ↓
Fraud Evaluation Request
     |
     ↓
Load customer risk profile
     |
     ↓
Load velocity counters
     |
     ↓
Evaluate rules
     |
     ↓
Calculate risk score
     |
     ├── ALLOW
     |
     ├── REVIEW
     |
     └── BLOCK
```

---

# 7. Example Fraud Signals

Possible signals:

```text
order amount
customer account age
number of orders
number of failed payments
number of payment attempts
velocity by customer
velocity by device
velocity by IP
shipping/billing mismatch
unusual order frequency
known risky country/region
previous chargebacks
suspicious behavior
```

Never collect or retain more personal data than the business and legal
requirements allow.

---

# 8. Fraud Rules

Example rules:

```text
RULE-001
New account + high-value order

RULE-002
Many payment attempts within short period

RULE-003
Unusual order velocity

RULE-004
Multiple customers using same suspicious device

RULE-005
Known high-risk payment signal

RULE-006
Repeated failed transactions
```

Rules should be versioned.

Example:

```text
fraud-rules-v12
```

The decision record should retain the rule version used for the evaluation.

---

# 9. Example Risk Calculation

Illustrative model:

```text
New account                 +20
High order amount           +20
Many payment attempts       +25
Suspicious velocity         +20
Known risky signal          +20
```

Maximum:

```text
105
```

Normalize:

```text
0 - 100
```

Decision:

```text
0-39   → ALLOW
40-69  → REVIEW
70-100 → BLOCK
```

This is a demonstration rule engine, not a production fraud model.

---

# 10. Important Fraud Design Principle

Fraud detection must not create unnecessary checkout latency.

Therefore distinguish:

```text
Synchronous critical fraud check
```

from:

```text
Asynchronous fraud analytics
```

Critical order path:

```text
Order
 ↓
Fraud
 ↓
Decision
```

Analytics:

```text
Order Events
 ↓
Kafka
 ↓
Fraud Analytics
 ↓
Historical features
```

---

# 11. Repository Structure

```text
fraud-service/
│
├── app/
│   ├── main.py
│   ├── config.py
│   │
│   ├── api/
│   │   ├── fraud.py
│   │   └── health.py
│   │
│   ├── models/
│   │   ├── fraud_evaluation.py
│   │   ├── fraud_rule.py
│   │   └── risk_signal.py
│   │
│   ├── schemas/
│   │   ├── fraud.py
│   │   └── events.py
│   │
│   ├── repositories/
│   │   ├── fraud_repository.py
│   │   └── rule_repository.py
│   │
│   ├── services/
│   │   ├── fraud_service.py
│   │   ├── rules_engine.py
│   │   ├── risk_service.py
│   │   └── velocity_service.py
│   │
│   ├── cache/
│   │   └── redis.py
│   │
│   ├── messaging/
│   │   └── kafka.py
│   │
│   ├── observability/
│   │   └── metrics.py
│   │
│   └── database/
│       └── session.py
│
├── migrations/
│
├── tests/
│   ├── test_rules.py
│   ├── test_fraud_service.py
│   └── test_api.py
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── pyproject.toml
├── .env.example
├── Makefile
├── README.md
│
├── helm/
│   └── fraud-service/
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

# 12. Python Dependencies

## `pyproject.toml`

```toml
[project]

name = "fraud-service"

version = "1.0.0"

requires-python = ">=3.13"

dependencies = [

    "fastapi",

    "uvicorn[standard]",

    "pydantic",

    "pydantic-settings",

    "sqlalchemy",

    "psycopg[binary]",

    "redis",

    "aiokafka",

    "prometheus-client",

]


[project.optional-dependencies]

test = [

    "pytest",

    "pytest-asyncio",

    "httpx",

]


[tool.pytest.ini_options]

asyncio_mode = "auto"
```

Production repositories should pin tested dependency versions through the
chosen dependency-locking mechanism.

---

# 13. Configuration

## `app/config.py`

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):

    app_name: str = "fraud-service"

    database_url: str

    redis_host: str = "localhost"

    redis_port: int = 6379

    kafka_bootstrap_servers: str = "localhost:9092"

    risk_allow_threshold: int = 39

    risk_review_threshold: int = 69

    velocity_window_seconds: int = 900

    max_order_amount_for_auto_allow: float = 50000


settings = Settings()
```

In production:

```text
database_url
provider credentials
Redis credentials
Kafka credentials
```

must come from the approved secrets mechanism.

---

# 14. Database Connection

## `app/database/session.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.config import settings


engine = create_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=5,
    pool_timeout=3,
    pool_recycle=1500,
    pool_pre_ping=True,
)


SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
)
```

Important:

For SQLAlchemy:

```text
potential connections
=
pool_size + max_overflow
```

per application process.

If:

```text
10 Pods
×
(10 pool + 5 overflow)
```

then theoretical connection capacity can approach:

```text
150
```

before considering other database clients.

This must be included in database capacity planning.

---

# 15. Fraud Evaluation Model

## `app/models/fraud_evaluation.py`

```python
from datetime import datetime

from sqlalchemy import (
    Column,
    DateTime,
    Integer,
    String,
    Text,
    Numeric,
)

from sqlalchemy.orm import declarative_base


Base = declarative_base()


class FraudEvaluation(Base):

    __tablename__ = "fraud_evaluations"

    id = Column(
        Integer,
        primary_key=True,
    )

    evaluation_id = Column(
        String(100),
        unique=True,
        nullable=False,
    )

    order_id = Column(
        String(100),
        nullable=False,
        index=True,
    )

    customer_id = Column(
        String(100),
        nullable=False,
        index=True,
    )

    risk_score = Column(
        Integer,
        nullable=False,
    )

    decision = Column(
        String(30),
        nullable=False,
    )

    rules_version = Column(
        String(50),
        nullable=False,
    )

    reason = Column(
        Text,
        nullable=True,
    )

    amount = Column(
        Numeric(19, 4),
        nullable=False,
    )

    currency = Column(
        String(3),
        nullable=False,
    )

    created_at = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False,
    )
```

---

# 16. Fraud Rule Model

```python
from sqlalchemy import (
    Boolean,
    Column,
    Integer,
    String,
)

from app.models.fraud_evaluation import Base


class FraudRule(Base):

    __tablename__ = "fraud_rules"

    id = Column(
        Integer,
        primary_key=True,
    )

    rule_code = Column(
        String(100),
        unique=True,
        nullable=False,
    )

    description = Column(
        String(500),
        nullable=False,
    )

    score = Column(
        Integer,
        nullable=False,
    )

    enabled = Column(
        Boolean,
        nullable=False,
        default=True,
    )

    rules_version = Column(
        String(50),
        nullable=False,
    )
```

---

# 17. Fraud Request Schema

## `app/schemas/fraud.py`

```python
from decimal import Decimal

from pydantic import BaseModel, Field


class FraudEvaluationRequest(BaseModel):

    order_id: str

    customer_id: str

    amount: Decimal = Field(
        gt=Decimal("0")
    )

    currency: str = Field(
        min_length=3,
        max_length=3,
    )

    payment_attempts: int = Field(
        ge=0
    )

    account_age_days: int = Field(
        ge=0
    )

    failed_orders: int = Field(
        ge=0
    )

    ip_risk_score: int = Field(
        ge=0,
        le=100,
    )
```

---

# 18. Fraud Response

```python
from pydantic import BaseModel


class FraudEvaluationResponse(BaseModel):

    evaluation_id: str

    order_id: str

    risk_score: int

    decision: str

    rules_version: str

    reasons: list[str]
```

---

# 19. Rules Engine

## `app/services/rules_engine.py`

```python
from app.config import settings


RULES_VERSION = "fraud-rules-v1"


def calculate_risk(
    *,
    amount,
    account_age_days: int,
    payment_attempts: int,
    failed_orders: int,
    ip_risk_score: int,
):

    score = 0

    reasons = []

    if account_age_days < 7:

        score += 20

        reasons.append(
            "NEW_ACCOUNT"
        )

    if amount >= settings.max_order_amount_for_auto_allow:

        score += 20

        reasons.append(
            "HIGH_ORDER_VALUE"
        )

    if payment_attempts >= 5:

        score += 25

        reasons.append(
            "HIGH_PAYMENT_ATTEMPTS"
        )

    if failed_orders >= 3:

        score += 15

        reasons.append(
            "FAILED_ORDER_HISTORY"
        )

    if ip_risk_score >= 70:

        score += 20

        reasons.append(
            "HIGH_IP_RISK"
        )

    score = min(
        score,
        100
    )

    if score <= settings.risk_allow_threshold:

        decision = "ALLOW"

    elif score <= settings.risk_review_threshold:

        decision = "REVIEW"

    else:

        decision = "BLOCK"

    return score, decision, reasons
```

This is intentionally simple.

A production system may later replace or augment this with:

- ML model
- Feature store
- External fraud provider
- Rules engine
- Device intelligence
- Graph analysis

without changing the external API contract.

---

# 20. Fraud Service

## `app/services/fraud_service.py`

```python
from uuid import uuid4

from app.services.rules_engine import (
    calculate_risk,
    RULES_VERSION,
)


class FraudService:

    def evaluate(
        self,
        request,
    ):

        score, decision, reasons = calculate_risk(
            amount=request.amount,
            account_age_days=request.account_age_days,
            payment_attempts=request.payment_attempts,
            failed_orders=request.failed_orders,
            ip_risk_score=request.ip_risk_score,
        )

        evaluation_id = str(uuid4())

        return {
            "evaluation_id": evaluation_id,
            "order_id": request.order_id,
            "risk_score": score,
            "decision": decision,
            "rules_version": RULES_VERSION,
            "reasons": reasons,
        }
```

The production version should persist the evaluation and apply idempotency
before returning the decision.

---

# 21. Fraud API

## `app/api/fraud.py`

```python
from fastapi import APIRouter, Header

from app.schemas.fraud import (
    FraudEvaluationRequest,
    FraudEvaluationResponse,
)

from app.services.fraud_service import FraudService


router = APIRouter(
    prefix="/api/v1/fraud",
    tags=["fraud"],
)


fraud_service = FraudService()


@router.post(
    "/evaluate",
    response_model=FraudEvaluationResponse,
)
def evaluate_fraud(
    request: FraudEvaluationRequest,
    idempotency_key: str = Header(
        alias="Idempotency-Key"
    ),
):

    return fraud_service.evaluate(
        request
    )
```

---

# 22. Application Entry Point

## `app/main.py`

```python
from fastapi import FastAPI

from app.api.fraud import router as fraud_router


app = FastAPI(
    title="Fraud Service",
    version="1.0.0",
)


app.include_router(
    fraud_router
)


@app.get("/health")
def health():

    return {
        "status": "UP"
    }


@app.get("/ready")
def ready():

    return {
        "status": "READY"
    }
```

Production readiness should include appropriate dependency checks without
making liveness dependent on every external service.

---

# 23. Database Migration

## `V1__create_fraud_evaluations.sql`

```sql
CREATE TABLE fraud_evaluations (

    id BIGSERIAL PRIMARY KEY,

    evaluation_id VARCHAR(100)
        NOT NULL UNIQUE,

    order_id VARCHAR(100)
        NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    risk_score INTEGER
        NOT NULL,

    decision VARCHAR(30)
        NOT NULL,

    rules_version VARCHAR(50)
        NOT NULL,

    reason TEXT,

    amount NUMERIC(19,4)
        NOT NULL,

    currency VARCHAR(3)
        NOT NULL,

    created_at TIMESTAMP
        NOT NULL
);

CREATE INDEX idx_fraud_order_id
    ON fraud_evaluations(order_id);

CREATE INDEX idx_fraud_customer_id
    ON fraud_evaluations(customer_id);

CREATE INDEX idx_fraud_decision
    ON fraud_evaluations(decision);
```

---

# 24. Idempotency

Fraud evaluation should support:

```text
Idempotency-Key
```

Example:

```text
ORDER-1001-FRAUD-CHECK-1
```

If Order Service retries:

```text
same order
same idempotency key
```

Fraud Service should return the original evaluation.

It must not create multiple independent decisions for the same operation unless
the business explicitly requests a new evaluation.

---

# 25. Redis Velocity Counters

Redis is particularly useful for fraud velocity.

Example:

```text
fraud:velocity:customer:C1001
fraud:velocity:ip:203.0.113.10
fraud:velocity:device:D1001
```

Example:

```text
customer C1001
5 payment attempts
within 15 minutes
```

could increase risk.

Redis provides:

- Fast counters
- TTL
- Atomic increments
- Short-lived risk signals

PostgreSQL stores the durable evaluation history.

---

# 26. Redis Example

Conceptually:

```python
key = f"fraud:velocity:customer:{customer_id}"

count = redis.incr(key)

if count == 1:
    redis.expire(
        key,
        900
    )
```

This creates a 15-minute window.

For production, atomic Lua scripts or carefully designed Redis commands should
be used where multiple related counters must be updated consistently.

---

# 27. Why Redis Is Not the Fraud Source of Truth

Redis:

```text
velocity
temporary features
```

PostgreSQL:

```text
fraud evaluation
decision
rules version
audit history
```

If Redis is unavailable:

```text
fraud-service
      |
      X
    Redis
```

the service must have an explicit degraded-mode policy.

It should not silently treat:

```text
Redis unavailable
=
customer safe
```

---

# 28. Fraud Fail-Open vs Fail-Closed

This is a business decision.

### Fail-open

```text
Fraud service unavailable
        ↓
Allow transaction
```

Higher availability.

Higher fraud risk.

### Fail-closed

```text
Fraud service unavailable
        ↓
Block transaction
```

Lower fraud risk.

Higher customer impact.

### Better approach

Use risk-based degradation.

For example:

```text
Low-risk transaction
    ↓
Fallback policy

High-value/high-risk transaction
    ↓
Require fraud decision
```

The exact policy belongs to the business/security team.

DevOps should not invent the production fraud policy during an outage.

---

# 29. Fraud and Order Service

Normal:

```text
Order
  |
  ↓
Fraud
  |
  ├── ALLOW
  |     ↓
  |   continue
  |
  ├── REVIEW
  |     ↓
  |   manual/async review
  |
  └── BLOCK
        ↓
      reject/cancel
```

---

# 30. Fraud and Payment Service

Recommended critical flow:

```text
Order
  |
  ↓
Fraud
  |
  ↓
ALLOW
  |
  ↓
Payment Authorization
```

For high-risk transactions:

```text
Order
  ↓
Fraud
  ↓
REVIEW
  ↓
Manual/secondary decision
  ↓
Payment
```

The exact sequence depends on payment authorization semantics.

---

# 31. Fraud Event

Example:

```json
{
  "eventId": "fraud-event-1001",
  "eventType": "FraudDecisionMade",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "fraud-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "100001",
  "payload": {
    "orderId": "100001",
    "decision": "ALLOW",
    "riskScore": 25,
    "rulesVersion": "fraud-rules-v1"
  }
}
```

---

# 32. Kafka Topics

Example:

```text
ecommerce.fraud.events
```

Events:

```text
FraudEvaluationRequested
FraudDecisionMade
FraudReviewRequired
FraudBlocked
```

Kafka allows other services to consume fraud decisions without tightly coupling
the fraud service to every consumer.

---

# 33. Fraud Consumer

Fraud Service can consume:

```text
OrderCreated
PaymentFailed
PaymentCaptured
OrderCancelled
```

These events can update risk features.

Example:

```text
PaymentFailed
    |
    ↓
fraud-service
    |
    ↓
increment customer failure velocity
```

This improves future fraud decisions.

---

# 34. Fraud Analytics

Architecture:

```text
Kafka
  |
  ↓
Fraud Events
  |
  ├── Real-time risk
  |
  ├── Feature processing
  |
  └── Historical analytics
```

Potential future ML architecture:

```text
Events
 ↓
Feature Pipeline
 ↓
Feature Store
 ↓
ML Model
 ↓
Fraud Service
 ↓
Decision
```

Do not introduce an ML platform before the rule-based service has reliable
observability, data quality, and evaluation history.

---

# 35. Dockerfile

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /app

COPY pyproject.toml .

RUN pip install \
    --no-cache-dir \
    --prefix=/install \
    .

COPY app ./app


FROM python:3.13-slim

WORKDIR /app

COPY --from=builder \
    /install \
    /usr/local

COPY app ./app

RUN useradd \
    --uid 10001 \
    --create-home \
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

For production, dependency installation should use a tested lockfile and
reproducible build process.

---

# 36. `.dockerignore`

```text
.git
.github

.venv
venv
__pycache__

.pytest_cache

.mypy_cache

.ruff_cache

.env
.env.*

*.log

tests

README.md
```

---

# 37. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: fraud-postgres

    environment:

      POSTGRES_DB: fraud

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: fraud-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: fraud-kafka

    ports:

      - "9092:9092"

    # Development-only Kafka configuration.
    # Pin a tested version for the actual project.


  fraud-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/fraud

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 38. Kubernetes Deployment

## `deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: fraud-service

  labels:

    app: fraud-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: fraud-service

  template:

    metadata:

      labels:

        app: fraud-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: fraud-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: fraud-service-config

            - secretRef:

                name: fraud-service-secrets

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

            initialDelaySeconds: 10

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: /health

              port: http

            initialDelaySeconds: 30

            periodSeconds: 20

          securityContext:

            allowPrivilegeEscalation: false

            readOnlyRootFilesystem: true

            capabilities:

              drop:

                - ALL
```

---

# 39. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: fraud-service

  labels:

    app: fraud-service

spec:

  type: ClusterIP

  selector:

    app: fraud-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 40. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: fraud-service-config

data:

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  KAFKA_BOOTSTRAP_SERVERS: kafka.ecommerce.svc.cluster.local:9092

  RISK_ALLOW_THRESHOLD: "39"

  RISK_REVIEW_THRESHOLD: "69"

  VELOCITY_WINDOW_SECONDS: "900"
```

Fraud thresholds should ultimately be managed according to the approved
fraud-rule governance process rather than changed casually during incidents.

---

# 41. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: fraud-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: fraud-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/fraud-service

        property: DATABASE_URL
```

If an external fraud provider is used:

```text
FRAUD_PROVIDER_API_KEY
FRAUD_PROVIDER_SECRET
```

should also be retrieved through the same controlled secrets architecture.

---

# 42. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: fraud-service

  labels:

    app: fraud-service
```

If Fraud Service requires AWS access:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
specific AWS service
```

No static AWS access keys belong in the Pod.

---

# 43. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: fraud-service

spec:

  podSelector:

    matchLabels:

      app: fraud-service

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

    - ports:

        - protocol: TCP

          port: 5432

        - protocol: TCP

          port: 6379

        - protocol: TCP

          port: 9092

        - protocol: TCP

          port: 443

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

The actual external provider egress path must match the organization's VPC,
proxy, firewall, and NetworkPolicy architecture.

---

# 44. Resource Configuration

## `values.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/fraud-service

  tag: ""

  pullPolicy: IfNotPresent


resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi


serviceAccount:

  create: true

  name: fraud-service


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60
```

These values are starting examples.

---

# 45. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: fraud-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: fraud-service

  minReplicas: 3

  maxReplicas: 15

  behavior:

    scaleUp:

      stabilizationWindowSeconds: 0

    scaleDown:

      stabilizationWindowSeconds: 300

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:

          type: Utilization

          averageUtilization: 60
```

Fraud capacity must consider:

```text
CPU
+
PostgreSQL
+
Redis
+
Kafka
+
external provider rate limits
```

---

# 46. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: fraud-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: fraud-service
```

---

# 47. Prometheus

## `servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: fraud-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: fraud-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

FastAPI should expose Prometheus-compatible metrics through the application's
observability layer.

---

# 48. Fraud Metrics

Technical:

```text
fraud_requests_total

fraud_request_duration_seconds

fraud_errors_total

fraud_redis_errors_total

fraud_database_errors_total

fraud_kafka_errors_total
```

Business:

```text
fraud_allow_total

fraud_review_total

fraud_block_total

fraud_evaluation_duration_seconds

fraud_rule_trigger_total
```

Risk-quality metrics:

```text
fraud_false_positive_rate

fraud_false_negative_rate

fraud_manual_review_rate
```

The last metrics generally require offline/analytics data and should not be
treated as simple infrastructure counters.

---

# 49. Grafana Dashboard

```text
Fraud Service
│
├── Request Rate
├── P95
├── P99
├── Error Rate
│
├── Decisions
│   ├── ALLOW
│   ├── REVIEW
│   └── BLOCK
│
├── Risk
│   ├── Average Score
│   ├── High Risk
│   └── Rule Trigger Rate
│
├── Redis
│   ├── Latency
│   ├── Errors
│   └── Velocity Counters
│
├── PostgreSQL
│   ├── Connections
│   └── Query Latency
│
└── Kafka
    ├── Consumer Lag
    └── Processing Errors
```

---

# 50. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T13:00:00Z",
  "level": "INFO",
  "service": "fraud-service",
  "environment": "prod",
  "correlationId": "corr-order-1001",
  "evaluationId": "eval-1001",
  "orderId": "100001",
  "decision": "REVIEW",
  "riskScore": 62,
  "rulesVersion": "fraud-rules-v1",
  "message": "Fraud evaluation completed"
}
```

Do not log:

```text
full payment credentials
card data
authentication tokens
provider secrets
raw sensitive customer data
```

---

# 51. Distributed Tracing

Example:

```text
Order Service
      |
      ↓
Fraud Service
      |
      ├── Redis
      |
      ├── PostgreSQL
      |
      └── External Fraud Provider
```

Trace:

```text
traceId
 |
 ├── order-service
 |
 ├── fraud-service
 |
 ├── Redis
 |
 └── provider request
```

Trace attributes must avoid sensitive information.

---

# 52. SLO

Initial targets:

Availability:

```text
99.95%
```

Synchronous fraud decision P95:

```text
< 200ms
```

Decision error rate:

```text
< 0.5%
```

Critical transaction evaluation success:

```text
99.9%
```

These are starting targets.

Fraud SLOs should balance:

```text
Customer experience
+
Fraud prevention
+
Business risk
```

---

# 53. Important Alerts

```text
High Fraud API 5xx

High Fraud Decision Latency

High Redis Error Rate

High PostgreSQL Error Rate

Kafka Consumer Lag

High External Provider Failure Rate

Fraud Service Unavailable

Unexpected Spike in BLOCK decisions

Unexpected Spike in ALLOW decisions

High Manual Review Rate
```

A sudden change in:

```text
BLOCK = 5% → 70%
```

may indicate:

- Rule deployment error
- Feature-data problem
- External provider issue
- Model problem
- Configuration mistake

It is a business incident even when Kubernetes is healthy.

---

# 54. Fraud Rules Deployment

Fraud rules are production logic.

Example:

```text
fraud-rules-v12
```

Before production:

```text
Developer
 ↓
Rule change
 ↓
Pull Request
 ↓
Tests
 ↓
Backtesting
 ↓
Review
 ↓
Staging
 ↓
Canary
 ↓
Production
```

Never change critical fraud thresholds directly in a production Pod.

---

# 55. Rule Versioning

Every decision should record:

```text
evaluationId
rulesVersion
riskScore
decision
triggeredRules
createdAt
```

Example:

```text
evaluation = E1001
rules = fraud-rules-v12
score = 72
decision = BLOCK
```

Months later, investigators can determine why the transaction was blocked.

---

# 56. False Positives

Example:

```text
Legitimate customer
+
large order
+
new address
```

Fraud engine:

```text
BLOCK
```

Customer is legitimate.

This is a false positive.

Therefore track:

```text
false_positive_rate
```

and manual-review outcomes.

Fraud prevention is not simply:

```text
maximize BLOCK
```

The goal is:

```text
minimize fraud
while
preserving legitimate customer transactions
```

---

# 57. Manual Review

For:

```text
REVIEW
```

workflow:

```text
Fraud Service
      |
      ↓
REVIEW
      |
      ↓
Fraud Review Queue
      |
      ↓
Analyst
      |
      ├── APPROVE
      |
      └── REJECT
```

This could use:

```text
SQS
```

or:

```text
Kafka
```

depending on the organization's workflow architecture.

---

# 58. Fraud Review Event

Example:

```json
{
  "eventId": "review-1001",
  "eventType": "FraudReviewRequired",
  "version": 1,
  "correlationId": "corr-order-1001",
  "aggregateId": "100001",
  "payload": {
    "orderId": "100001",
    "riskScore": 62,
    "reason": "HIGH_PAYMENT_ATTEMPTS"
  }
}
```

---

# 59. Fraud Provider Failure

Suppose external fraud provider is unavailable.

```text
Fraud Service
      |
      ↓
Provider
      X
```

Do not automatically:

```text
provider unavailable
→ ALLOW everything
```

The fallback decision must follow the approved business risk policy.

Possible policy:

```text
Low-risk
→ continue with internal rules

High-risk
→ REVIEW

Very high-risk
→ BLOCK or hold
```

The exact policy belongs to fraud/security/business owners.

---

# 60. Fraud Database Failure

If PostgreSQL is unavailable:

```text
fraud evaluation
      |
      X
database
```

The service should not falsely report:

```text
evaluation stored successfully
```

The API behavior should follow the defined availability/risk policy.

Critical payment/order decisions should not silently become unaudited.

---

# 61. Fraud Redis Failure

Redis failure affects velocity features:

```text
Redis
 X
```

Potentially:

```text
customer velocity unavailable
```

The service can use:

```text
PostgreSQL historical data
+
static rules
+
safe fallback policy
```

but must not silently assume:

```text
velocity = zero
```

if that could create a security hole.

---

# 62. Fraud Kafka Failure

If Kafka is unavailable:

```text
Fraud Service
      |
      ↓
PostgreSQL evaluation
      |
      ↓
Outbox
      |
      X
Kafka
```

The durable evaluation should not be lost.

Use the same transactional outbox principle:

```text
Fraud decision
+
Outbox event
=
same database transaction
```

Then retry publishing later.

---

# 63. Fraud Outbox

Table:

```text
fraud_outbox_events
```

Fields:

```text
event_id
event_type
aggregate_id
payload
status
retry_count
created_at
published_at
```

Flow:

```text
Fraud Evaluation
      |
      ├── fraud_evaluations
      |
      └── fraud_outbox_events
              |
             COMMIT
              |
              ↓
        Outbox Publisher
              |
              ↓
             Kafka
```

---

# 64. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Fraud Service CI

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

  ECR_REPOSITORY: fraud-service


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

          pip install .

          pip install \
            pytest \
            pytest-asyncio \
            httpx


      - name: Test

        run: |

          pytest


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


      - name: Build image

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

In production, pin tested versions of actions and dependencies according to the
project's supply-chain policy.

---

# 65. SonarQube

Fraud Service quality gates should include:

```text
Python bugs
Security vulnerabilities
Code smells
Test coverage
Duplicated code
```

Pipeline:

```text
Test
 ↓
Coverage
 ↓
SonarQube
 ↓
Trivy
 ↓
Docker
 ↓
Trivy image
 ↓
ECR
```

SonarQube remains centralized.

---

# 66. ECR

Repository:

```text
company-ecommerce/fraud-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/fraud-service:<git-sha>
```

Use immutable tags.

Prefer deployment by image digest when the GitOps strategy supports it.

---

# 67. Helm

## `Chart.yaml`

```yaml
apiVersion: v2

name: fraud-service

description: Ecommerce fraud detection service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 68. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

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

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: fraud-service
```

---

# 69. GitOps

Application:

```text
company-ecommerce/fraud-service
```

GitOps:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
└── apps/
    └── fraud-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Deployment:

```text
Code
 ↓
CI
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

# 70. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: fraud-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/fraud-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

For fraud-rule changes, use stronger approval and canary controls than for
ordinary application changes.

---

# 71. Fraud Production Incident

## Incident ID

```text
INC-2026-FRAUD-001
```

## Severity

```text
SEV-2
```

## Title

```text
Fraud Rule Regression Caused Large Increase in Transaction Blocks
```

## Date

```text
2026-10-05
```

---

# 72. Customer Impact

Customers experienced:

- Legitimate orders being blocked
- Increased checkout failures
- Increased manual reviews
- Reduced successful order conversion

No evidence of unauthorized payment capture was found.

---

# 73. Timeline

```text
11:00
Fraud rules v12 deployed.

11:05
Fraud BLOCK rate increases.

11:07
Order success rate begins declining.

11:09
Customer support reports checkout failures.

11:10
Fraud dashboard shows BLOCK rate:
5% → 48%.

11:12
SEV-2 declared.

11:14
Platform confirms Kubernetes healthy.

11:16
Fraud team identifies rule threshold change.

11:18
Rule v12 disabled.

11:20
Rule v11 restored.

11:23
BLOCK rate returns toward baseline.

11:25
Order success rate recovers.

11:30
Incident resolved.
```

---

# 74. Root Cause

A fraud rule change unintentionally increased the score for a broad customer
segment.

The change was technically valid and deployed successfully.

However, the business behavior was incorrect.

This demonstrates why:

```text
Deployment success
≠
Business correctness
```

---

# 75. Amplification

```text
Rule change
    ↓
Higher risk scores
    ↓
More BLOCK decisions
    ↓
More checkout failures
    ↓
Lower conversion
```

---

# 76. What Went Well

- Fraud decision metrics existed
- BLOCK rate was observable
- Order success SLO detected business impact
- GitOps allowed controlled rollback
- Previous rule version was available

---

# 77. What Went Wrong

- Rule change lacked sufficient backtesting
- No canary traffic for the rule
- No automated anomaly threshold on BLOCK rate
- Business conversion was not part of the deployment gate

---

# 78. Five Whys

### Why did orders fail?

Fraud Service blocked legitimate transactions.

### Why?

Risk scores increased unexpectedly.

### Why?

A rule threshold was changed.

### Why wasn't the issue detected before production?

The rule was not sufficiently backtested against representative historical data.

### Why did the deployment pass?

Infrastructure and application health checks passed, but business-risk metrics
were not included in the deployment gate.

---

# 79. Corrective Actions

| Action | Owner |
|---|---|
| Add fraud-rule backtesting | Fraud |
| Add BLOCK-rate alert | SRE |
| Add conversion-rate alert | SRE |
| Add rule canary | Platform |
| Version fraud rules | Fraud |
| Add historical replay tests | Data/Fraud |
| Add false-positive monitoring | Fraud |
| Add business metrics to deployment gates | SRE |
| Add fraud rollback runbook | SRE |

---

# 80. Failure Scenario — Fraud Service Down

```text
Order
 ↓
Fraud Service
 X
```

The response depends on the approved risk policy.

Possible:

```text
Low risk
→ controlled fallback

High risk
→ REVIEW

Critical/high-risk
→ BLOCK or hold
```

This decision must be explicitly documented.

Never let an accidental application exception determine whether transactions
are allowed.

---

# 81. Failure Scenario — Redis Down

```text
Redis
 X
```

Velocity data unavailable.

Actions:

```text
Detect
 ↓
Determine missing signals
 ↓
Apply approved degraded-mode policy
 ↓
Protect high-risk transactions
 ↓
Restore Redis
 ↓
Verify counters
```

Do not blindly set:

```text
risk = 0
```

---

# 82. Failure Scenario — Kafka Down

```text
Fraud decision
 ↓
PostgreSQL
 ↓
Outbox
 ↓
Kafka unavailable
```

The decision remains durable.

Publisher retries.

This avoids losing fraud events.

---

# 83. Failure Scenario — External Fraud Provider Down

```text
Fraud Service
 ↓
External Provider
 X
```

Circuit breaker opens.

Internal rules continue if approved.

Example:

```text
Internal risk score
+
cached/static signals
```

then:

```text
ALLOW / REVIEW / BLOCK
```

according to the defined business policy.

---

# 84. Security Checklist

```text
[ ] No payment credentials stored
[ ] No card data stored
[ ] Secrets Manager
[ ] External Secrets
[ ] EKS Pod Identity
[ ] Least privilege IAM
[ ] NetworkPolicy
[ ] TLS
[ ] Non-root container
[ ] Read-only filesystem
[ ] Trivy
[ ] SonarQube
[ ] Dependency scanning
[ ] Secret scanning
[ ] Audit logging
[ ] Rule versioning
[ ] Access control
```

---

# 85. Performance Checklist

```text
[ ] Redis latency
[ ] Redis velocity operations
[ ] PostgreSQL pool sizing
[ ] HPA testing
[ ] Rule evaluation latency
[ ] Kafka lag
[ ] External provider latency
[ ] Provider rate limits
[ ] Cache failure testing
[ ] Database failure testing
[ ] Load testing
[ ] Spike testing
```

---

# 86. Operational Runbook

When fraud alerts fire:

```text
1. Confirm customer impact

2. Check ALLOW/REVIEW/BLOCK rates

3. Check recent rule deployment

4. Check application error rate

5. Check P95/P99

6. Check Redis

7. Check PostgreSQL

8. Check Kafka

9. Check external provider

10. Check rule version

11. Compare against baseline

12. Determine fail-open/fail-closed policy

13. Protect legitimate customers

14. Protect against fraud

15. Roll back bad rule if required

16. Verify business conversion

17. Validate risk metrics

18. Document RCA
```

---

# 87. Fraud Capacity Model

Suppose:

```text
Fraud Pods = 10
```

Each Pod:

```text
10 database connections
```

SQLAlchemy overflow:

```text
5
```

Potential:

```text
10 × (10 + 5)
=
150 database connections
```

This must be included in the shared RDS budget.

Fraud scaling must also consider:

```text
Redis throughput
Kafka throughput
External provider rate limits
CPU
Memory
```

---

# 88. Final Architecture

```text
                              Order Service
                                    |
                                    ↓
                              Fraud Service
                         /       |        \
                        /        |         \
                       ↓         ↓          ↓
                 PostgreSQL    Redis       Kafka
                       |         |
                       |         ↓
                       |    Velocity Signals
                       |
                       ↓
                Fraud Evaluation
                       |
              ┌────────┼────────┐
              ↓        ↓        ↓
            ALLOW    REVIEW    BLOCK
              |        |        |
              ↓        ↓        ↓
           Payment   Review    Cancel/Hold
```

Future ML extension:

```text
Kafka Events
     |
     ↓
Feature Pipeline
     |
     ↓
Feature Store
     |
     ↓
ML Model
     |
     ↓
Fraud Service
     |
     ↓
Risk Decision
```

---

# 89. Final Production Principles

1. Fraud is a risk decision, not a simple CRUD operation.
2. Keep fraud logic separate from Order and Payment services.
3. Every evaluation must be idempotent.
4. Every decision records the rule/model version.
5. Redis is useful for velocity counters.
6. PostgreSQL stores durable evaluation history.
7. Kafka distributes fraud events.
8. Transactional Outbox protects database/event consistency.
9. Fraud provider calls need timeouts.
10. Provider calls need bounded retries.
11. Circuit breakers prevent retry storms.
12. Fail-open/fail-closed must be an explicit business decision.
13. Never silently allow transactions because a fraud dependency failed.
14. Never blindly block every transaction because the fraud service failed.
15. False positives are production incidents too.
16. Business conversion is a fraud-service reliability signal.
17. Rule changes require testing and controlled deployment.
18. Fraud rules should be versioned.
19. Fraud decisions must be auditable.
20. Sensitive customer/payment information must be minimized.
21. Kubernetes health does not prove fraud-system correctness.
22. More Pods do not automatically mean more fraud-processing capacity.
23. External provider rate limits must be part of capacity planning.
24. Observability must include business-risk metrics.
25. Every fraud incident should produce an RCA and preventive actions.

# End of 172 — Fraud Service
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
168 Promotion Service       ✅
169 Cart Service            ✅
170 Order Service           ✅
171 Payment Service         ✅
172 Fraud Service           ✅

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

**Next: `173-Shipping-Service.md` — shipping-rate calculation, carrier integration, shipment creation, tracking IDs, carrier webhooks, retry/idempotency, SQS/Kafka, PostgreSQL, Redis, external carrier APIs, Kubernetes, CI/CD, GitOps, observability, failure scenarios and production RCA.**