# 175 — Notification Service

**File:** `175-Notification-Service.md`

```markdown
# 175 — Notification Service

## 1. Overview

The `notification-service` is responsible for delivering customer and
operational notifications across multiple channels.

Supported channels in this project:

- Email
- SMS
- Push notification
- In-app notification

The service consumes business events from Kafka/SQS and converts them into
channel-specific notifications.

It is designed as an asynchronous service because notification delivery should
not block critical business workflows such as order creation or payment.

Technology:

- Python 3.13
- FastAPI
- PostgreSQL
- Redis
- Kafka
- Amazon SQS
- AWS SES
- SMS provider
- Push notification provider
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

# 2. Why Notification Is a Separate Service

Consider an order flow:

```text
Customer
   |
   ↓
Order Service
   |
   ↓
Payment
   |
   ↓
Order Confirmed
```

The order should not wait for:

```text
Email provider
SMS provider
Push provider
```

because an external notification provider may be slow or unavailable.

Bad architecture:

```text
Order Service
    |
    ├── PostgreSQL
    ├── Payment
    └── Email Provider
             |
             X
          timeout
```

This can make the order API slow.

Better:

```text
Order Service
     |
     ↓
OrderConfirmed
     |
     ↓
Kafka/SQS
     |
     ↓
Notification Service
     |
     ├── Email
     ├── SMS
     ├── Push
     └── In-App
```

The business transaction remains independent from notification delivery.

---

# 3. Notification Responsibilities

The service owns:

```text
Notification
Notification Template
Notification Preference
Notification Attempt
Provider Response
Delivery Status
Retry
DLQ handling
Idempotency
Channel routing
Provider failover
```

It does not own:

```text
Order
Payment
Inventory
Shipment
Delivery
Customer master data
```

It consumes information from those domains.

---

# 4. Notification Types

Examples:

```text
ORDER_CREATED
ORDER_CONFIRMED
ORDER_CANCELLED

PAYMENT_SUCCESS
PAYMENT_FAILED

SHIPMENT_CREATED
SHIPMENT_IN_TRANSIT

DELIVERY_OUT_FOR_DELIVERY
DELIVERY_FAILED
DELIVERY_DELIVERED

PASSWORD_RESET
ACCOUNT_CREATED

INVOICE_READY
REFUND_COMPLETED
```

---

# 5. Notification Channels

```text
                    Notification Service
                           |
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
        Email             SMS             Push
          |                |                |
       SES/provider      Provider        FCM/APNs
```

In-app:

```text
Notification Service
        |
        ↓
PostgreSQL
        |
        ↓
Customer Notification Center
```

---

# 6. Notification Architecture

```text
                         Kafka
                           |
                           ↓
                   Notification Service
                  /          |          \
                 ↓           ↓           ↓
            PostgreSQL     Redis       SQS
                 |                       |
                 |                       ↓
                 |                 Notification
                 |                    Workers
                 |                       |
                 |          ┌────────────┼───────────┐
                 |          ↓            ↓           ↓
                 |        Email         SMS         Push
                 |          |            |           |
                 |          ↓            ↓           ↓
                 |       Provider     Provider     Provider
                 |
                 ↓
             In-App Inbox
```

---

# 7. Why Kafka + SQS

Kafka is useful for:

```text
Business event distribution
Durable event stream
Multiple consumers
Event replay
Ordering within partitions
```

SQS is useful for:

```text
Notification work queue
Retries
Backpressure
DLQ
Independent worker scaling
```

Example:

```text
OrderConfirmed
      |
      ↓
Kafka
      |
      ↓
Notification Service
      |
      ├── Email → SQS
      ├── SMS   → SQS
      └── Push  → SQS
```

This prevents a slow SMS provider from blocking email processing.

---

# 8. Repository Structure

```text
notification-service/
│
├── app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   │
│   ├── models/
│   │   ├── notification.py
│   │   ├── template.py
│   │   └── preference.py
│   │
│   ├── schemas/
│   │   ├── notification.py
│   │   └── event.py
│   │
│   ├── repositories/
│   │   └── notification_repository.py
│   │
│   ├── services/
│   │   ├── notification_service.py
│   │   ├── template_service.py
│   │   ├── preference_service.py
│   │   └── provider_service.py
│   │
│   ├── providers/
│   │   ├── email/
│   │   │   ├── base.py
│   │   │   └── ses.py
│   │   ├── sms/
│   │   │   ├── base.py
│   │   │   └── provider.py
│   │   └── push/
│   │       ├── base.py
│   │       └── fcm.py
│   │
│   ├── messaging/
│   │   ├── kafka_consumer.py
│   │   └── sqs_worker.py
│   │
│   ├── api/
│   │   └── notification.py
│   │
│   ├── observability/
│   │   └── metrics.py
│   │
│   └── health/
│       └── health.py
│
├── migrations/
├── tests/
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
│   └── notification-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── worker-deployment.yaml
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

# 9. Python Dependencies

## `pyproject.toml`

```toml
[project]
name = "notification-service"
version = "1.0.0"
description = "Ecommerce notification service"
requires-python = ">=3.13"

dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "pydantic",
    "pydantic-settings",
    "sqlalchemy",
    "psycopg[binary]",
    "redis",
    "boto3",
    "aiokafka",
    "prometheus-client",
]

[project.optional-dependencies]

test = [
    "pytest",
    "pytest-asyncio",
    "httpx",
]

dev = [
    "ruff",
]
```

Production dependency versions should be pinned and tested.

---

# 10. Configuration

## `app/config.py`

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):

    app_name: str = "notification-service"

    port: int = 8080

    database_url: str

    redis_url: str = "redis://localhost:6379/0"

    kafka_brokers: str = "localhost:9092"

    aws_region: str = "ap-south-1"

    notification_queue_url: str

    email_sender: str

    provider_timeout_seconds: float = 5.0

    class Config:
        env_file = ".env"


settings = Settings()
```

---

# 11. Environment Configuration

## `.env.example`

```text
DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/notifications

REDIS_URL=redis://localhost:6379/0

KAFKA_BROKERS=localhost:9092

AWS_REGION=ap-south-1

NOTIFICATION_QUEUE_URL=http://localhost:4566/000000000000/notifications

EMAIL_SENDER=no-reply@example.com

PROVIDER_TIMEOUT_SECONDS=5
```

Production secrets must not be stored in `.env`.

---

# 12. Database Model

## `notification.py`

```python
from datetime import datetime

from sqlalchemy import (
    DateTime,
    Integer,
    String,
    Text,
)
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    mapped_column,
)


class Base(DeclarativeBase):
    pass


class Notification(Base):

    __tablename__ = "notifications"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True,
    )

    notification_id: Mapped[str] = mapped_column(
        String(100),
        unique=True,
        nullable=False,
    )

    event_id: Mapped[str] = mapped_column(
        String(200),
        nullable=False,
    )

    customer_id: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    channel: Mapped[str] = mapped_column(
        String(30),
        nullable=False,
    )

    notification_type: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    recipient: Mapped[str] = mapped_column(
        String(500),
        nullable=False,
    )

    status: Mapped[str] = mapped_column(
        String(50),
        nullable=False,
    )

    provider_message_id: Mapped[str | None] = (
        mapped_column(
            String(200),
            nullable=True,
        )
    )

    failure_reason: Mapped[str | None] = (
        mapped_column(
            Text,
            nullable=True,
        )
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        nullable=False,
    )

    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        nullable=False,
    )
```

---

# 13. Notification Status

```text
PENDING
PROCESSING
SENT
DELIVERED
FAILED
RETRYING
CANCELLED
```

Provider-specific delivery states may be mapped into these internal states.

---

# 14. Database Migration

## `V1__create_notifications.sql`

```sql
CREATE TABLE notifications (

    id BIGSERIAL PRIMARY KEY,

    notification_id VARCHAR(100)
        NOT NULL UNIQUE,

    event_id VARCHAR(200)
        NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    channel VARCHAR(30)
        NOT NULL,

    notification_type VARCHAR(100)
        NOT NULL,

    recipient VARCHAR(500)
        NOT NULL,

    status VARCHAR(50)
        NOT NULL,

    provider_message_id VARCHAR(200),

    failure_reason TEXT,

    created_at TIMESTAMP
        NOT NULL,

    updated_at TIMESTAMP
        NOT NULL
);

CREATE INDEX idx_notifications_customer
    ON notifications(customer_id);

CREATE INDEX idx_notifications_status
    ON notifications(status);

CREATE INDEX idx_notifications_event
    ON notifications(event_id);
```

---

# 15. Event Idempotency

Notification events can be delivered more than once.

Example:

```text
OrderConfirmed
      |
      ↓
Kafka
      |
      ├── delivery 1
      ├── delivery 2
      └── delivery 3
```

Without idempotency:

```text
Email
Email
Email
```

could be sent.

Store:

```text
event_id
+
notification_type
+
channel
```

with an appropriate uniqueness strategy.

---

# 16. Notification Event

Example:

```json
{
  "eventId": "evt-order-1001",
  "eventType": "OrderConfirmed",
  "version": 1,
  "occurredAt": "2026-10-05T16:00:00Z",
  "source": "order-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "ORD-1001",
  "payload": {
    "orderId": "ORD-1001",
    "customerId": "CUS-1001",
    "customerEmail": "customer@example.com",
    "customerPhone": "+91XXXXXXXXXX"
  }
}
```

Sensitive information should be minimized.

---

# 17. Channel Selection

The notification engine determines:

```text
Notification Type
        |
        ↓
Customer Preferences
        |
        ↓
Business Rules
        |
        ↓
Available Channel
```

Example:

```text
ORDER_CONFIRMED
      |
      ├── Email = enabled
      ├── SMS = enabled
      └── Push = disabled
```

Result:

```text
Email → send
SMS   → send
Push  → skip
```

Mandatory security notifications may follow different preference rules.

---

# 18. Notification Preferences

Example:

```text
Customer
   |
   ├── Order updates → Email + Push
   ├── Promotions    → Email
   ├── Security      → Email + SMS
   └── Delivery      → Push + SMS
```

Preferences should be stored separately from notification delivery history.

---

# 19. Preference Table

```sql
CREATE TABLE notification_preferences (

    id BIGSERIAL PRIMARY KEY,

    customer_id VARCHAR(100)
        NOT NULL,

    notification_type VARCHAR(100)
        NOT NULL,

    email_enabled BOOLEAN
        NOT NULL DEFAULT TRUE,

    sms_enabled BOOLEAN
        NOT NULL DEFAULT TRUE,

    push_enabled BOOLEAN
        NOT NULL DEFAULT TRUE,

    updated_at TIMESTAMP
        NOT NULL,

    CONSTRAINT uq_notification_preference
        UNIQUE (
            customer_id,
            notification_type
        )
);
```

---

# 20. Templates

Do not hard-code every notification message inside business logic.

Example:

```text
ORDER_CONFIRMED
```

Template:

```text
Subject:
Order {{order_id}} confirmed

Body:
Your order {{order_id}} has been confirmed.
```

Template variables:

```text
order_id
customer_name
amount
tracking_number
delivery_date
```

---

# 21. Template Versioning

Example:

```text
ORDER_CONFIRMED:v1
ORDER_CONFIRMED:v2
```

When a template changes, preserve the ability to identify which version was
used.

This is useful for:

```text
auditing
incident investigation
customer support
A/B testing
regulatory requirements
```

---

# 22. Provider Abstraction

Do not hard-code:

```text
notification_service.py
    ↓
AWS SES everywhere
```

Instead:

```text
Notification Service
       |
       ↓
EmailProvider
       |
       ├── SES
       └── Backup Provider
```

SMS:

```text
SmsProvider
    |
    ├── Provider A
    └── Provider B
```

Push:

```text
PushProvider
    |
    ├── FCM
    └── APNs
```

---

# 23. Email Provider Interface

```python
from abc import ABC, abstractmethod


class EmailProvider(ABC):

    @abstractmethod
    async def send(
        self,
        recipient: str,
        subject: str,
        body: str,
    ):
        raise NotImplementedError
```

---

# 24. SES Provider

Conceptual implementation:

```python
import boto3


class SESProvider:

    def __init__(
        self,
        region: str,
    ):
        self.client = boto3.client(
            "ses",
            region_name=region,
        )

    async def send(
        self,
        sender: str,
        recipient: str,
        subject: str,
        body: str,
    ):

        response = self.client.send_email(
            Source=sender,
            Destination={
                "ToAddresses": [
                    recipient
                ]
            },
            Message={
                "Subject": {
                    "Data": subject
                },
                "Body": {
                    "Text": {
                        "Data": body
                    }
                }
            }
        )

        return response["MessageId"]
```

In a high-throughput implementation, provider SDK usage should be wrapped
carefully so blocking calls do not unnecessarily consume async application
workers.

---

# 25. SMS Provider

Conceptual interface:

```python
class SmsProvider:

    async def send(
        self,
        phone_number: str,
        message: str,
    ) -> str:

        # Call approved SMS provider.

        return "provider-message-id"
```

Never put provider-specific code into the core notification business logic.

---

# 26. Push Provider

```python
class PushProvider:

    async def send(
        self,
        device_token: str,
        title: str,
        body: str,
    ) -> str:

        # Call approved push provider.

        return "provider-message-id"
```

---

# 27. Notification Worker

The worker receives:

```text
SQS
 ↓
Notification Worker
```

Then:

```text
Receive message
      ↓
Validate
      ↓
Check idempotency
      ↓
Check customer preference
      ↓
Render template
      ↓
Select provider
      ↓
Send
      ↓
Persist status
      ↓
Delete message
```

---

# 28. Why Notification Workers Are Separate

API deployment:

```text
notification-api
```

Worker deployment:

```text
notification-worker
```

This allows independent scaling.

Example:

```text
API:
3 Pods

Email Worker:
10 Pods

SMS Worker:
5 Pods

Push Worker:
3 Pods
```

If email volume increases:

```text
Email workers scale
```

without unnecessarily scaling the API.

---

# 29. Worker Failure

Suppose:

```text
Worker
  |
  ↓
SMS Provider
  |
  X
timeout
```

The worker should:

```text
retry
```

rather than:

```text
delete message
```

If retries are exhausted:

```text
DLQ
```

---

# 30. Retry Strategy

Transient errors:

```text
HTTP 429
HTTP 500
HTTP 502
HTTP 503
HTTP 504
timeout
temporary DNS failure
```

Use:

```text
exponential backoff
+
jitter
+
maximum attempts
```

Permanent errors:

```text
invalid recipient
invalid template
provider authentication failure
invalid request
```

should not be retried indefinitely.

---

# 31. Retry Example

```text
Attempt 1
   ↓
1 second
   ↓
Attempt 2
   ↓
2 seconds
   ↓
Attempt 3
   ↓
4 seconds
   ↓
Attempt 4
   ↓
DLQ
```

Actual values depend on provider limits and business requirements.

---

# 32. Provider Rate Limiting

Suppose:

```text
SMS provider limit = 100 requests/sec
```

but KEDA scales:

```text
5 workers → 50 workers
```

Without rate limiting:

```text
50 workers × concurrency
```

may exceed provider limits.

Therefore:

```text
SQS
 ↓
KEDA
 ↓
Workers
 ↓
Rate Limiter
 ↓
SMS Provider
```

Scaling consumers and respecting provider capacity must be designed together.

---

# 33. Notification Deduplication

Example:

```text
OrderConfirmed
```

should normally generate one logical notification per configured channel.

Store:

```text
event_id
notification_type
channel
```

and use a unique constraint or transactional idempotency mechanism.

---

# 34. Exactly-Once Reality

Messaging systems commonly provide:

```text
at-least-once delivery
```

Therefore:

```text
exactly-once business effect
```

must be implemented by the application.

For example:

```text
Message delivered twice
       |
       ↓
Same idempotency key
       |
       ↓
Second processing skipped
```

---

# 35. Notification Delivery Status

Provider response:

```text
ACCEPTED
```

does not necessarily mean:

```text
DELIVERED
```

Example:

```text
Notification
    |
    ↓
Provider
    |
    ↓
ACCEPTED
    |
    ↓
Provider processing
    |
    ↓
DELIVERED / BOUNCED / FAILED
```

Therefore provider callbacks/webhooks can update final status.

---

# 36. Email Bounce

Example:

```text
Email Sent
    |
    ↓
Provider
    |
    ↓
BOUNCE
```

Store:

```text
status = FAILED
failure_reason = EMAIL_BOUNCE
```

Repeated bounces may require the address to be suppressed to protect sender
reputation.

---

# 37. SMS Failure

Possible reasons:

```text
invalid number
provider rejected
carrier unavailable
rate limit
country restriction
temporary provider failure
```

Classify failures into:

```text
retryable
non-retryable
```

rather than retrying everything.

---

# 38. Push Token Handling

Push tokens can expire or become invalid.

Example:

```text
Push
 ↓
Provider
 ↓
INVALID_TOKEN
```

The service should mark the token inactive rather than repeatedly retrying it.

---

# 39. Notification API

Example endpoint:

```http
GET /api/v1/notifications/{customerId}
```

Response:

```json
{
  "notifications": [
    {
      "notificationId": "NOTIF-1001",
      "type": "ORDER_CONFIRMED",
      "channel": "IN_APP",
      "status": "DELIVERED",
      "createdAt": "2026-10-05T16:00:00Z"
    }
  ]
}
```

The API should authorize access based on the authenticated customer identity.

Never trust an arbitrary `customerId` supplied by an unauthenticated caller.

---

# 40. Mark as Read

```http
POST /api/v1/notifications/{notificationId}/read
```

This updates the in-app notification state.

It should verify:

```text
notification belongs to authenticated customer
```

---

# 41. Redis Caching

Useful cache:

```text
notification:preferences:{customerId}
```

and:

```text
notification:template:{type}:{version}
```

TTL should be short enough to prevent stale preference behavior.

When preferences change:

```text
DB update
 ↓
Redis invalidation
```

---

# 42. PostgreSQL Connection Pool

For Python SQLAlchemy:

```python
engine = create_async_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=5,
    pool_timeout=3,
    pool_recycle=1500,
    pool_pre_ping=True,
)
```

Important:

```text
pool_size + max_overflow
```

is per application process.

If:

```text
10 Pods
×
(10 + 5)
```

the potential pool capacity can reach:

```text
150 connections
```

before accounting for other clients.

Therefore Kubernetes scaling and DB connection capacity must be planned
together.

---

# 43. Database Connection Protection

Notification workers can generate significant DB load.

Do not allow:

```text
KEDA
 ↓
100 workers
 ↓
100 × DB pools
```

without calculating database capacity.

Use:

```text
maximum worker replicas
+
small DB pool
+
queue backpressure
```

---

# 44. FastAPI Application

## `app/main.py`

```python
from fastapi import FastAPI

from app.config import settings


app = FastAPI(
    title=settings.app_name,
)


@app.get("/health")
async def health():
    return {
        "status": "UP"
    }


@app.get("/ready")
async def ready():
    return {
        "status": "READY"
    }


@app.get("/metrics")
async def metrics():
    return {
        "status": "OK"
    }
```

Production implementation should expose Prometheus metrics through the chosen
metrics integration rather than returning a placeholder response.

---

# 45. Dockerfile

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /app

COPY pyproject.toml ./

RUN pip install --no-cache-dir \
    --prefix=/install \
    fastapi \
    uvicorn[standard] \
    pydantic \
    pydantic-settings \
    sqlalchemy \
    psycopg[binary] \
    redis \
    boto3 \
    aiokafka \
    prometheus-client

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

For production, use a lockfile/locked dependency installation so the image is
reproducible.

---

# 46. `.dockerignore`

```text
.git
.github

__pycache__
*.pyc

.pytest_cache
.coverage
htmlcov

.venv
venv

.env
.env.*

tests

README.md
```

---

# 47. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: notification-postgres

    environment:

      POSTGRES_DB: notifications

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: notification-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: notification-kafka

    ports:

      - "9092:9092"

    # Development-only configuration.
    # Pin the tested version for production.


  notification-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql+psycopg://postgres:postgres@postgres:5432/notifications

      REDIS_URL:
        redis://redis:6379/0

      KAFKA_BROKERS:
        kafka:9092

      AWS_REGION:
        ap-south-1

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 48. Kubernetes Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: notification-service

  labels:

    app: notification-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: notification-service

  template:

    metadata:

      labels:

        app: notification-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: notification-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: notification-service-config

            - secretRef:

                name: notification-service-secrets

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

# 49. Worker Deployment

Workers should be independently scalable.

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: notification-worker

  labels:

    app: notification-worker

spec:

  replicas: 3

  selector:

    matchLabels:

      app: notification-worker

  template:

    metadata:

      labels:

        app: notification-worker

    spec:

      serviceAccountName: notification-service

      containers:

        - name: notification-worker

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          command:

            - python

            - "-m"

            - app.messaging.sqs_worker

          envFrom:

            - configMapRef:

                name: notification-service-config

            - secretRef:

                name: notification-service-secrets

          resources:

            requests:

              cpu: 200m

              memory: 256Mi

            limits:

              cpu: "1"

              memory: 512Mi
```

---

# 50. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: notification-service

  labels:

    app: notification-service

spec:

  type: ClusterIP

  selector:

    app: notification-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 51. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: notification-service-config

data:

  REDIS_URL:
    redis://redis.ecommerce.svc.cluster.local:6379/0

  KAFKA_BROKERS:
    kafka.ecommerce.svc.cluster.local:9092

  AWS_REGION:
    ap-south-1

  PROVIDER_TIMEOUT_SECONDS:
    "5"
```

---

# 52. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: notification-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: notification-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/notification-service

        property: DATABASE_URL

    - secretKey: NOTIFICATION_PROVIDER_SECRET

      remoteRef:

        key: ecommerce/prod/notification-service

        property: NOTIFICATION_PROVIDER_SECRET

    - secretKey: WEBHOOK_SECRET

      remoteRef:

        key: ecommerce/prod/notification-service

        property: WEBHOOK_SECRET
```

---

# 53. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: notification-service

  labels:

    app: notification-service
```

If using AWS SES/SQS/S3:

```text
Notification Worker
       |
       ↓
EKS Pod Identity
       |
       ↓
notification-service IAM role
       |
       ├── SES
       ├── SQS
       └── S3
```

Only required actions should be granted.

---

# 54. Example IAM Permissions

Potential permissions:

```text
ses:SendEmail
ses:SendRawEmail

sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes

s3:PutObject
```

Scope them to the exact resources where possible.

Never give:

```text
AdministratorAccess
```

to the application role.

---

# 55. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: notification-service

spec:

  podSelector:

    matchLabels:

      app: notification-service

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

---

# 56. HPA

API workload:

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: notification-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: notification-service

  minReplicas: 3

  maxReplicas: 10

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:

          type: Utilization

          averageUtilization: 60
```

---

# 57. KEDA Worker Scaling

Conceptual architecture:

```text
SQS queue
    |
    ↓
KEDA
    |
    ↓
notification-worker
```

Example:

```yaml
apiVersion: keda.sh/v1alpha1

kind: ScaledObject

metadata:

  name: notification-worker

spec:

  scaleTargetRef:

    name: notification-worker

  minReplicaCount: 2

  maxReplicaCount: 20

  triggers:

    - type: aws-sqs-queue

      metadata:

        queueURLFromEnv: NOTIFICATION_QUEUE_URL

        queueLength: "50"

        awsRegion: ap-south-1
```

Exact KEDA configuration depends on the installed KEDA version and AWS
authentication model.

---

# 58. Why KEDA Is Useful Here

Suppose:

```text
Normal:
100 notifications/min

Flash sale:
50,000 notifications/min
```

CPU may not immediately represent queue pressure.

SQS backlog is a better signal:

```text
Queue depth
    ↓
KEDA
    ↓
Workers
```

But the maximum number of workers must remain bounded by provider limits.

---

# 59. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: notification-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: notification-service
```

Worker PDB requirements may differ from API requirements.

---

# 60. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: notification-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: notification-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 61. Notification Metrics

Application:

```text
notifications_created_total

notifications_sent_total

notifications_failed_total

notifications_retried_total

notification_duration_seconds
```

Channel:

```text
email_sent_total
email_failed_total

sms_sent_total
sms_failed_total

push_sent_total
push_failed_total
```

Provider:

```text
provider_request_total

provider_error_total

provider_timeout_total

provider_rate_limit_total
```

Queue:

```text
notification_queue_depth

notification_oldest_message_age
```

Business:

```text
order_notifications_sent_total

delivery_notifications_sent_total

security_notifications_sent_total
```

---

# 62. Grafana Dashboard

```text
Notification Service
│
├── Overall
│   ├── Notification Rate
│   ├── Success Rate
│   ├── Failure Rate
│   └── P95
│
├── Email
│   ├── Sent
│   ├── Failed
│   └── Provider Latency
│
├── SMS
│   ├── Sent
│   ├── Failed
│   └── Rate Limits
│
├── Push
│   ├── Sent
│   └── Invalid Tokens
│
├── Queue
│   ├── Depth
│   ├── Oldest Message
│   └── DLQ
│
└── Infrastructure
    ├── CPU
    ├── Memory
    ├── Pods
    ├── PostgreSQL
    └── Redis
```

---

# 63. Structured Logs

Example:

```json
{
  "@timestamp": "2026-10-05T17:00:00Z",
  "level": "INFO",
  "service": "notification-service",
  "environment": "prod",
  "correlationId": "corr-order-1001",
  "eventId": "evt-order-1001",
  "notificationId": "NOTIF-1001",
  "channel": "EMAIL",
  "notificationType": "ORDER_CONFIRMED",
  "provider": "SES",
  "status": "SENT",
  "message": "Notification sent"
}
```

Do not log:

```text
passwords
OTP
provider credentials
authorization headers
full sensitive customer information
```

---

# 64. OpenTelemetry

Architecture:

```text
Notification Service
       |
       ↓
OpenTelemetry
       |
       ↓
OTel Collector
       |
       ↓
Jaeger
```

Trace example:

```text
Order Service
     |
     ↓
Kafka
     |
     ↓
Notification Service
     |
     ↓
SQS
     |
     ↓
Notification Worker
     |
     ↓
Email Provider
```

The trace should allow engineers to determine where notification latency
occurred.

---

# 65. Notification SLOs

Example:

```text
API availability:
99.95%

Notification acceptance:
99.9%

Critical notification processing:
99.9% within target window

Email queue freshness:
99% under defined age

SMS queue freshness:
99% under defined age
```

For notification systems, latency SLOs should often be based on the business
requirement rather than simply HTTP request latency.

---

# 66. Critical Notification vs Marketing Notification

These should not necessarily share the same queue.

Example:

```text
Critical
├── Password Reset
├── Payment Failure
└── Delivery Update

Marketing
├── Offers
├── Promotions
└── Campaigns
```

Architecture:

```text
Critical Queue
     |
     ↓
Critical Workers

Marketing Queue
     |
     ↓
Marketing Workers
```

This prevents marketing traffic from delaying password-reset or payment
notifications.

---

# 67. Queue Isolation

Recommended:

```text
notification-critical
notification-standard
notification-marketing
```

Each can have:

```text
different
priority
retention
DLQ
scaling
provider limits
```

---

# 68. Backpressure

If SMS provider is unavailable:

```text
SMS Provider
     X
     |
     ↓
SMS Queue
     |
     ↓
Backlog
```

Do not continuously increase workers.

Instead:

```text
Provider outage
      ↓
Bound worker concurrency
      ↓
Retry with backoff
      ↓
DLQ after limit
      ↓
Protect provider
```

---

# 69. Provider Circuit Breaker

```text
SMS Provider
     |
     ↓
Circuit Breaker
     |
     ├── CLOSED
     ├── OPEN
     └── HALF_OPEN
```

When provider errors exceed a threshold:

```text
OPEN
```

Workers fail fast or pause provider calls.

---

# 70. Provider Failover

Example:

```text
SMS Provider A
      |
      X
      |
      ↓
Provider B
```

Failover should only happen when:

```text
provider contract permits it
recipient/data rules permit it
duplicate delivery risk is controlled
```

Do not blindly send the same SMS through two providers because the first
provider may have accepted the message even though the application timed out.

---

# 71. The Difficult Notification Failure

Scenario:

```text
Worker
  |
  ↓
SMS Provider
  |
  ↓
SMS accepted
  |
  X
Network timeout
```

Worker believes:

```text
FAILED
```

and retries.

Possible result:

```text
SMS #1 → delivered
SMS #2 → delivered
```

Customer receives duplicates.

Therefore provider-side idempotency should be used whenever the provider
supports it.

Otherwise use:

```text
application idempotency
+
provider request ID
+
reconciliation
```

---

# 72. Notification Reconciliation

For providers that support delivery-status queries:

```text
Unknown notifications
        |
        ↓
Provider status API
        |
        ↓
Compare
        |
        ↓
Update final state
```

This is especially important when:

```text
provider accepted request
+
application timed out
```

---

# 73. Security Notifications

Security notifications should have stronger guarantees.

Examples:

```text
Password reset
MFA change
New login
Email address changed
Payment security event
```

These may bypass marketing preferences.

Security events must not be suppressed merely because the customer disabled
promotional notifications.

---

# 74. Notification Data Retention

Notification records may contain personal information.

Define:

```text
retention period
archive policy
deletion/anonymization policy
audit requirements
```

Do not retain message bodies forever unless required.

---

# 75. Production Failure — Email Provider Down

```text
Email Provider
      X
      |
      ↓
Email Queue
      |
      ↓
Backlog increases
```

Investigation:

```text
Provider metrics
Queue age
Worker errors
HTTP status
DNS
TLS
IAM/credentials
Provider status
```

Mitigation:

```text
Open circuit
Bound retries
Protect queue
Use approved fallback provider if available
```

---

# 76. Production Failure — Notification Backlog

Check:

```bash
kubectl get pods \
  -n ecommerce \
  -l app=notification-worker
```

Then:

```text
SQS queue depth
oldest message age
worker count
worker CPU
worker errors
provider latency
provider rate limit
DLQ
```

Determine whether the bottleneck is:

```text
Queue
Worker
Database
Redis
Provider
```

---

# 77. Production Failure — Duplicate Notifications

Check:

```text
eventId
notificationId
channel
provider message ID
idempotency record
consumer logs
retry count
```

Determine whether duplication happened because of:

```text
duplicate Kafka event
worker retry
provider timeout
missing idempotency
consumer crash after provider acceptance
```

---

# 78. Production Failure — Wrong Template

Example:

```text
ORDER_CONFIRMED
```

renders:

```text
Order {{order_id}} confirmed
```

instead of:

```text
Order ORD-1001 confirmed
```

The infrastructure may be completely healthy.

Therefore add:

```text
template rendering tests
business smoke tests
sample notification tests
```

to CI.

---

# 79. Production Incident Example

## Incident ID

```text
INC-2026-NOTIFICATION-001
```

## Severity

```text
SEV-2
```

## Title

```text
SMS Provider Outage Caused Notification Backlog
```

## Date

```text
2026-10-05
```

---

# 80. Customer Impact

Customers experienced delayed:

```text
Delivery SMS
Payment failure SMS
Order update SMS
```

Email notifications continued normally.

No customer orders or payments were lost.

---

# 81. Timeline

```text
18:00
Normal SMS processing.

18:04
SMS provider latency increases.

18:06
Provider 5xx rate increases.

18:07
SMS queue starts growing.

18:09
SMS worker retry rate increases.

18:11
Queue age exceeds SLO.

18:12
SEV-2 declared.

18:14
Platform confirms EKS healthy.

18:16
Notification team confirms provider outage.

18:18
SMS circuit breaker opens.

18:19
SMS worker concurrency reduced.

18:20
Email and push continue normally.

18:25
Provider begins recovering.

18:27
Circuit breaker enters half-open.

18:29
Successful provider requests observed.

18:31
Normal processing restored.

18:38
SMS backlog cleared.

18:40
Incident resolved.
```

---

# 82. Root Cause

The SMS provider experienced an elevated 5xx error rate.

The notification service initially retried requests too aggressively.

This increased:

```text
provider requests
+
worker utilization
+
queue processing latency
```

---

# 83. Amplification Mechanism

```text
Provider outage
      ↓
SMS failures
      ↓
Retries
      ↓
More provider traffic
      ↓
Provider overload
      ↓
More failures
```

After mitigation:

```text
Provider failure
      ↓
Circuit breaker
      ↓
Bounded retries
      ↓
Queue backpressure
      ↓
Provider recovery
      ↓
Backlog drain
```

---

# 84. What Went Well

- Email remained operational
- Queue protected customer-facing APIs
- DLQ prevented permanent message loss
- Provider monitoring detected the outage
- Circuit breaker eventually limited traffic
- Notification events remained available for replay

---

# 85. What Went Wrong

- Retry policy was too aggressive
- Provider-specific circuit breaker threshold was too high
- SMS queue alert fired later than desired
- No automatic provider health score existed

---

# 86. Five Whys

### Why were SMS notifications delayed?

The SMS provider was returning errors.

### Why did the backlog grow?

Workers could not successfully deliver SMS.

### Why did processing become inefficient?

Workers retried failed requests too aggressively.

### Why?

Retry policy did not sufficiently account for provider-wide failure.

### Why was this not detected earlier?

Provider-specific SLO and circuit-breaker tuning were insufficient.

---

# 87. Corrective Actions

| Action | Owner |
|---|---|
| Tune SMS retry policy | Backend |
| Improve circuit breaker | Backend |
| Add provider-specific SLO | SRE |
| Add queue-age alert | SRE |
| Add provider health score | Platform |
| Test provider outage | SRE |
| Validate fallback provider | Architecture |
| Add retry storm test | Performance |
| Add provider rate-limit dashboard | SRE |

---

# 88. CI/CD Architecture

```text
Developer
   |
   ↓
GitHub PR
   |
   ↓
CI
   |
   ├── Ruff
   ├── Pytest
   ├── Coverage
   ├── SonarQube
   ├── Trivy FS
   ├── Docker Build
   └── Trivy Image
          |
          ↓
         ECR
          |
          ↓
       GitOps
          |
          ↓
        ArgoCD
          |
          ↓
         EKS
```

---

# 89. GitHub Actions

## `.github/workflows/ci.yaml`

```yaml
name: Notification Service CI

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

  ECR_REPOSITORY: notification-service

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

          pip install \
            fastapi \
            uvicorn \
            pydantic \
            pydantic-settings \
            sqlalchemy \
            psycopg[binary] \
            redis \
            boto3 \
            aiokafka \
            prometheus-client \
            pytest \
            pytest-asyncio \
            httpx \
            ruff

      - name: Ruff

        run: ruff check .

      - name: Tests

        run: pytest -q

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

Production repositories should pin action versions according to the
organization's supply-chain policy.

---

# 90. SonarQube

SonarQube analyzes:

```text
Python code
bugs
security issues
code smells
duplication
test coverage
```

It runs in CI.

SonarQube itself is a centralized platform component and should not be
installed inside the notification application Pod.

---

# 91. ECR

Repository:

```text
company-ecommerce/notification-service
```

Example image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/notification-service:<git-sha>
```

Use immutable tags.

Prefer deployment by digest where the GitOps process supports it.

---

# 92. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: notification-service

description: Ecommerce notification service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 93. Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/notification-service

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

  name: notification-service

autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 10

  targetCPUUtilizationPercentage: 60
```

---

# 94. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── notification-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Image promotion:

```text
CI
 ↓
ECR
 ↓
GitOps PR
 ↓
Approval
 ↓
ArgoCD
 ↓
EKS
```

CI does not directly deploy production using:

```text
kubectl apply
```

in this GitOps architecture.

---

# 95. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: notification-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/notification-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

Production approval policy should be enforced through the organization's
GitOps process.

---

# 96. Production Troubleshooting Flow

Start with:

```text
Customer impact
      ↓
Notification channel
      ↓
Queue
      ↓
Workers
      ↓
Provider
      ↓
Database
      ↓
Redis
      ↓
Kafka/SQS
      ↓
Recent deployment
```

Do not start by randomly restarting Pods.

---

# 97. Troubleshooting — Email Delayed

Check:

```text
Email queue
Queue age
Worker replicas
Worker errors
SES/provider latency
SES throttling
DNS
Network
IAM
```

Then determine:

```text
Provider issue?
Worker issue?
Queue issue?
Application issue?
```

---

# 98. Troubleshooting — SMS Duplicate

Check:

```text
eventId
notificationId
idempotency record
provider message ID
worker retry
provider response
timeout
```

The key question is:

```text
Did the provider accept the first request?
```

A client-side timeout does not prove the provider rejected it.

---

# 99. Troubleshooting — All Notifications Failed

Check:

```text
Kafka
SQS
PostgreSQL
Redis
Secrets
IAM
Provider credentials
NetworkPolicy
DNS
recent deployment
```

If all channels fail simultaneously, suspect a shared dependency.

If only one channel fails:

```text
Email works
SMS fails
Push works
```

focus first on SMS-specific dependencies.

---

# 100. Production Checklist

```text
[ ] Unit tests
[ ] Integration tests
[ ] Template tests
[ ] Preference tests
[ ] Duplicate event tests
[ ] Provider timeout tests
[ ] Provider 429 tests
[ ] Provider 5xx tests
[ ] Provider acceptance timeout tests
[ ] Webhook tests
[ ] Idempotency tests
[ ] Retry tests
[ ] DLQ tests
[ ] Queue backlog tests
[ ] Trivy
[ ] SonarQube
[ ] ECR
[ ] Immutable image
[ ] GitOps
[ ] ArgoCD
[ ] HPA
[ ] KEDA
[ ] PDB
[ ] NetworkPolicy
[ ] Secrets Manager
[ ] External Secrets
[ ] Pod Identity
[ ] Prometheus
[ ] Grafana
[ ] ELK
[ ] OpenTelemetry
[ ] Jaeger
[ ] SLO
[ ] Alerts
[ ] Runbook
[ ] Provider outage test
```

---

# 101. Final Architecture

```text
                       Business Services
                              |
                              ↓
                    Kafka Business Events
                              |
                              ↓
                   Notification Service
                              |
               ┌──────────────┼──────────────┐
               ↓              ↓              ↓
            Preferences    Templates       Redis
               |              |              |
               └──────────────┼──────────────┘
                              ↓
                         Routing Engine
                              |
             ┌────────────────┼────────────────┐
             ↓                ↓                ↓
        Critical Queue    Standard Queue   Marketing Queue
             |                |                |
             ↓                ↓                ↓
       Worker Pool       Worker Pool       Worker Pool
             |                |                |
             ↓                ↓                ↓
           Email             SMS              Push
             |                |                |
             ↓                ↓                ↓
          Provider         Provider         Provider
             |
             ↓
        Delivery Status
             |
             ↓
        PostgreSQL
             |
             ↓
       Kafka Events
             |
       ┌─────┼─────┐
       ↓     ↓     ↓
   Reporting Delivery Customer
```

---

# 102. Key Production Principles

1. Notification delivery must not block critical order workflows.
2. Kafka distributes business events.
3. SQS provides controlled asynchronous notification processing.
4. Email, SMS and push should have independent queues/workers where scale
   requires it.
5. Critical notifications should be isolated from marketing traffic.
6. Provider failures must not cause retry storms.
7. Provider calls require timeouts.
8. Retries require exponential backoff and jitter.
9. Provider rate limits must be respected.
10. Circuit breakers protect unhealthy providers.
11. Provider acceptance is not the same as final delivery.
12. Provider callbacks should update final delivery state.
13. Notification processing must be idempotent.
14. Duplicate events must not produce duplicate business effects.
15. PostgreSQL stores durable notification state.
16. Redis is an optimization layer.
17. SQS DLQs are operational signals, not garbage bins.
18. Customer preferences must be respected except where business/security
    rules require mandatory notifications.
19. Security notifications require stronger guarantees.
20. Sensitive information must not appear in logs or traces.
21. Provider credentials belong in Secrets Manager.
22. AWS access should use EKS Pod Identity rather than static keys.
23. KEDA is useful for queue-driven notification workers.
24. KEDA max replicas must respect provider and database capacity.
25. Business notification SLOs matter more than CPU utilization alone.
26. Provider outages should be tested deliberately.
27. Reconciliation is important when provider calls time out after acceptance.
28. GitOps remains the deployment source of truth.
29. Production incidents require an RCA.
30. Notification infrastructure must fail independently from core business
    transactions.

# End of 175 — Notification Service
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
173 Shipping Service        ✅
174 Delivery Service        ✅
175 Notification Service    ✅

176 Search Service          ⏳
177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `176-Search-Service.md`** — search architecture with OpenSearch/Elasticsearch, indexing pipeline, Kafka events, autocomplete, filters, relevance, aliases, zero-downtime reindexing, Kubernetes, EKS, CI/CD, GitOps, Prometheus/Grafana/ELK/OpenTelemetry/Jaeger, failure scenarios and production RCA.