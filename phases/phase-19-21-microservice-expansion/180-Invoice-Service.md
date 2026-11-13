# 180 — Invoice Service

**File:** `180-Invoice-Service.md`

```markdown
# 180 — Invoice Service

## 1. Overview

The `invoice-service` generates, stores, retrieves, and manages customer invoices.

The service receives completed-order information and creates an immutable financial
document representing the transaction.

Primary responsibilities:

- Generate invoice numbers
- Consume order-completed events
- Capture final order amounts
- Capture tax breakdown
- Capture discounts
- Generate invoice documents
- Store invoice documents in Amazon S3
- Store invoice metadata in PostgreSQL
- Provide invoice retrieval APIs
- Prevent duplicate invoice generation
- Publish invoice-generated events
- Support invoice regeneration when technically necessary
- Maintain audit information
- Integrate with notification-service
- Integrate with reporting-service

Technology:

- Python 3.13
- FastAPI
- PostgreSQL
- Redis
- Kafka
- Amazon S3
- AWS KMS
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
```

---

# 2. Why Invoice Service Exists

Invoice generation is different from order processing.

The Order Service owns:

```text
Order
Order status
Order lifecycle
Order items
```

The Tax Service owns:

```text
Tax calculation
Tax rules
Tax breakdown
```

The Invoice Service owns:

```text
Invoice
Invoice number
Invoice document
Invoice metadata
Invoice storage
```

Architecture:

```text
Order Service
      |
      ↓
Order Completed Event
      |
      ↓
Kafka
      |
      ↓
Invoice Service
      |
      ├── PostgreSQL
      |
      ├── PDF Generator
      |
      └── S3
             |
             ↓
       Invoice Document
```

---

# 3. Important Financial Principle

The invoice should represent the **final committed transaction**, not recalculate
the order from current prices.

Do not do:

```text
Current Product Price
        +
Current Tax Rule
        +
Current Promotion
        ↓
Invoice
```

Instead:

```text
Completed Order
      |
      ↓
Final Order Snapshot
      |
      ↓
Invoice
```

This is critical because prices and tax rules can change after an order is
completed.

---

# 4. Invoice Snapshot

The invoice should preserve:

```text
product name at purchase time
SKU
quantity
unit price
discount
tax
shipping charge
final total
currency
customer billing address
shipping address
tax information
payment reference
order ID
```

This creates an immutable historical record.

---

# 5. High-Level Architecture

```text
                        Order Service
                              |
                              ↓
                     OrderCompleted Event
                              |
                              ↓
                           Kafka
                              |
                              ↓
                       Invoice Service
                      /       |       \
                     ↓        ↓        ↓
              PostgreSQL    S3       Redis
                     |
                     ↓
                Invoice Metadata
                             
                     S3
                      |
                      ↓
                 Invoice PDF

Invoice Service
      |
      ↓
InvoiceGenerated Event
      |
      ├── Notification Service
      └── Reporting Service
```

---

# 6. Invoice Generation Flow

```text
Order Completed
      |
      ↓
Consume Kafka event
      |
      ↓
Check idempotency
      |
      ↓
Create invoice number
      |
      ↓
Persist invoice metadata
      |
      ↓
Generate PDF
      |
      ↓
Upload PDF to S3
      |
      ↓
Update invoice status
      |
      ↓
Publish InvoiceGenerated
```

---

# 7. Invoice Status

Recommended states:

```text
PENDING
GENERATING
GENERATED
FAILED
CANCELLED
```

Example:

```text
PENDING
   |
   ↓
GENERATING
   |
   ├── GENERATED
   |
   └── FAILED
```

---

# 8. Invoice Number

Do not use:

```text
invoice ID = database auto-increment ID
```

as the public invoice number.

Instead:

```text
Invoice ID:
internal UUID

Invoice Number:
INV-2026-000001
```

The public invoice number should follow a controlled business numbering
strategy.

---

# 9. Invoice Number Requirements

The numbering mechanism should provide:

- uniqueness
- controlled format
- auditability
- no accidental reuse
- concurrency safety
- predictable business rules where required

For high-volume systems, invoice numbering should not rely on:

```text
timestamp + random number
```

if legal/business requirements require strict sequential numbering.

---

# 10. Repository Structure

```text
invoice-service/
│
├── app/
│   ├── main.py
│   ├── config.py
│   │
│   ├── database.py
│   │
│   ├── models/
│   │   ├── invoice.py
│   │   ├── invoice_item.py
│   │   └── outbox_event.py
│   │
│   ├── schemas/
│   │   ├── invoice.py
│   │   └── events.py
│   │
│   ├── repositories/
│   │   └── invoice_repository.py
│   │
│   ├── services/
│   │   ├── invoice_service.py
│   │   ├── invoice_number_service.py
│   │   ├── pdf_service.py
│   │   ├── s3_service.py
│   │   └── idempotency_service.py
│   │
│   ├── consumers/
│   │   └── order_completed_consumer.py
│   │
│   ├── producers/
│   │   └── invoice_event_producer.py
│   │
│   ├── api/
│   │   └── invoice.py
│   │
│   ├── observability/
│   │   └── metrics.py
│   │
│   └── dependencies.py
│
├── migrations/
│   ├── V1__create_invoices.sql
│   ├── V2__create_invoice_items.sql
│   └── V3__create_outbox.sql
│
├── tests/
│   ├── test_invoice.py
│   ├── test_invoice_number.py
│   └── test_idempotency.py
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
│   └── invoice-service/
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

# 11. Technology Stack

```text
Python
  ↓
FastAPI
  ↓
SQLAlchemy
  ↓
PostgreSQL
```

Additional components:

```text
Kafka → event consumption
S3    → invoice PDF/object storage
Redis → idempotency/cache where useful
```

---

# 12. `pyproject.toml`

```toml
[project]

name = "invoice-service"

version = "1.0.0"

requires-python = ">=3.13"

dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "sqlalchemy",
    "psycopg[binary]",
    "pydantic",
    "pydantic-settings",
    "boto3",
    "redis",
    "aiokafka",
    "reportlab",
    "alembic",
    "prometheus-client"
]


[project.optional-dependencies]

dev = [
    "pytest",
    "pytest-asyncio",
    "httpx",
    "ruff",
    "mypy"
]


[tool.pytest.ini_options]

asyncio_mode = "auto"


[tool.ruff]

line-length = 100
```

Pin exact tested dependency versions in the actual repository.

---

# 13. Configuration

## `app/config.py`

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):

    app_name: str = "invoice-service"

    port: int = 8080

    database_url: str

    redis_url: str

    kafka_brokers: str

    kafka_group_id: str = "invoice-service"

    aws_region: str = "ap-south-1"

    s3_bucket: str

    invoice_prefix: str = "invoices/"

    pdf_expiry_seconds: int = 900


settings = Settings()
```

---

# 14. Environment Variables

```text
DATABASE_URL=postgresql+psycopg://invoice_user:password@localhost:5432/invoices

REDIS_URL=redis://localhost:6379

KAFKA_BROKERS=localhost:9092

KAFKA_GROUP_ID=invoice-service

AWS_REGION=ap-south-1

S3_BUCKET=company-ecommerce-invoices

INVOICE_PREFIX=invoices/

PDF_EXPIRY_SECONDS=900
```

Production secrets must not be stored in `.env` files committed to Git.

---

# 15. Database Model

The invoice metadata table:

```sql
CREATE TABLE invoices (

    id BIGSERIAL PRIMARY KEY,

    invoice_id UUID NOT NULL UNIQUE,

    invoice_number VARCHAR(100) NOT NULL UNIQUE,

    order_id VARCHAR(100) NOT NULL UNIQUE,

    customer_id VARCHAR(100) NOT NULL,

    currency VARCHAR(10) NOT NULL,

    subtotal NUMERIC(19,4) NOT NULL,

    discount_amount NUMERIC(19,4) NOT NULL,

    tax_amount NUMERIC(19,4) NOT NULL,

    shipping_amount NUMERIC(19,4) NOT NULL,

    total_amount NUMERIC(19,4) NOT NULL,

    status VARCHAR(50) NOT NULL,

    s3_bucket VARCHAR(255),

    s3_key VARCHAR(1000),

    document_sha256 VARCHAR(64),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    generated_at TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_invoices_customer
ON invoices(customer_id);

CREATE INDEX idx_invoices_order
ON invoices(order_id);
```

---

# 16. Invoice Items

```sql
CREATE TABLE invoice_items (

    id BIGSERIAL PRIMARY KEY,

    invoice_id UUID NOT NULL,

    product_id VARCHAR(100) NOT NULL,

    sku VARCHAR(100),

    product_name VARCHAR(500) NOT NULL,

    quantity INTEGER NOT NULL,

    unit_price NUMERIC(19,4) NOT NULL,

    discount_amount NUMERIC(19,4) NOT NULL,

    taxable_amount NUMERIC(19,4) NOT NULL,

    tax_amount NUMERIC(19,4) NOT NULL,

    total_amount NUMERIC(19,4) NOT NULL,

    CONSTRAINT fk_invoice_items_invoice
        FOREIGN KEY (invoice_id)
        REFERENCES invoices(invoice_id)
);
```

---

# 17. Invoice Snapshot

The item record should preserve:

```text
productId
SKU
productName
quantity
unitPrice
discount
tax
total
```

Do not retrieve the current Product Service price while generating an old
invoice.

---

# 18. Why Historical Snapshot Matters

Suppose an order was placed:

```text
2026-01-01
```

Product price:

```text
₹10,000
```

Six months later:

```text
Current price = ₹15,000
```

The old invoice must still show:

```text
₹10,000
```

Therefore:

```text
Invoice
  ↓
Historical transaction snapshot
```

rather than:

```text
Invoice
  ↓
Current microservice state
```

---

# 19. Outbox Table

```sql
CREATE TABLE outbox_events (

    id BIGSERIAL PRIMARY KEY,

    event_id UUID NOT NULL UNIQUE,

    event_type VARCHAR(200) NOT NULL,

    aggregate_id VARCHAR(200) NOT NULL,

    payload JSONB NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    published_at TIMESTAMP
);

CREATE INDEX idx_outbox_unpublished
ON outbox_events(published_at)
WHERE published_at IS NULL;
```

---

# 20. Why Outbox Is Important

Without an outbox:

```text
Save invoice
     |
     ↓
Publish Kafka event
     |
     X
Kafka unavailable
```

Invoice exists but downstream services do not know about it.

With outbox:

```text
Transaction
   |
   ├── Invoice
   |
   └── Outbox Event
   |
   ↓
Commit
   |
   ↓
Outbox Publisher
   |
   ↓
Kafka
```

---

# 21. Order Completed Event

Example:

```json
{
  "eventId": "evt-order-1001",
  "eventType": "ORDER_COMPLETED",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "order-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "ORD-1001",
  "payload": {
    "orderId": "ORD-1001",
    "customerId": "CUS-1001",
    "currency": "INR",
    "subtotal": "5000.00",
    "discount": "500.00",
    "tax": "810.00",
    "shipping": "100.00",
    "total": "5410.00",
    "items": [
      {
        "productId": "PROD-1001",
        "sku": "SKU-1001",
        "name": "Example Product",
        "quantity": 2,
        "unitPrice": "2500.00",
        "discount": "500.00",
        "tax": "810.00",
        "total": "5310.00"
      }
    ]
  }
}
```

The exact event contract should be versioned and governed.

---

# 22. Idempotency

Kafka normally provides at-least-once delivery semantics.

Therefore:

```text
ORDER_COMPLETED
      |
      ↓
Invoice Service
      |
      ↓
processing
      |
      X
consumer crashes
      |
      ↓
event delivered again
```

Without idempotency:

```text
Invoice #1
Invoice #2
```

could be generated for the same order.

---

# 23. Database Constraint

This provides a strong safety boundary:

```sql
UNIQUE(order_id)
```

Only one invoice can be associated with a completed order under the chosen
business model.

The application should also perform an idempotency check, but the database
constraint is the final protection.

---

# 24. Invoice Number Generation

One approach is a database sequence:

```sql
CREATE SEQUENCE invoice_number_seq
START WITH 1
INCREMENT BY 1;
```

Then format the generated number:

```text
INV-2026-000001
INV-2026-000002
INV-2026-000003
```

The actual numbering strategy must comply with the organization's accounting
and regulatory requirements.

---

# 25. Concurrency

Suppose two identical `ORDER_COMPLETED` events arrive:

```text
Consumer A
    |
    ↓
check order
    |
Consumer B
    |
    ↓
check order
```

Both may initially see no invoice.

Therefore:

```text
Application idempotency
+
Database UNIQUE constraint
```

should be used together.

---

# 26. Invoice Model

```python
from sqlalchemy import (
    Column,
    String,
    Numeric,
    DateTime,
    UUID,
)
from sqlalchemy.orm import declarative_base

Base = declarative_base()


class Invoice(Base):

    __tablename__ = "invoices"

    id = Column(
        String,
        primary_key=True,
    )

    invoice_id = Column(
        UUID,
        unique=True,
        nullable=False,
    )

    invoice_number = Column(
        String(100),
        unique=True,
        nullable=False,
    )

    order_id = Column(
        String(100),
        unique=True,
        nullable=False,
    )

    customer_id = Column(
        String(100),
        nullable=False,
    )

    currency = Column(
        String(10),
        nullable=False,
    )

    subtotal = Column(
        Numeric(19, 4),
        nullable=False,
    )

    discount_amount = Column(
        Numeric(19, 4),
        nullable=False,
    )

    tax_amount = Column(
        Numeric(19, 4),
        nullable=False,
    )

    shipping_amount = Column(
        Numeric(19, 4),
        nullable=False,
    )

    total_amount = Column(
        Numeric(19, 4),
        nullable=False,
    )

    status = Column(
        String(50),
        nullable=False,
    )

    s3_key = Column(
        String(1000),
    )

    document_sha256 = Column(
        String(64),
    )

    created_at = Column(
        DateTime,
        nullable=False,
    )

    generated_at = Column(
        DateTime,
    )
```

---

# 27. Invoice API

```text
GET /api/v1/invoices/{invoiceId}

GET /api/v1/orders/{orderId}/invoice

GET /api/v1/customers/{customerId}/invoices
```

For downloading:

```text
GET /api/v1/invoices/{invoiceId}/download
```

The API should preferably generate a short-lived signed S3 URL rather than
streaming large documents through the application unnecessarily.

---

# 28. Invoice Download Flow

```text
Customer
   |
   ↓
Invoice Service
   |
   ↓
Authorize invoice ownership
   |
   ↓
Check invoice metadata
   |
   ↓
Generate short-lived S3 URL
   |
   ↓
Customer
   |
   ↓
S3
```

The S3 bucket remains private.

Do not make invoices publicly accessible.

---

# 29. S3 Object Key

Example:

```text
invoices/2026/10/05/ORD-1001/INV-2026-000001.pdf
```

Advantages:

- predictable organization
- easier lifecycle management
- easier troubleshooting
- easier audit
- avoids dumping everything into one flat logical namespace

---

# 30. S3 Security

Use:

```text
private bucket
SSE-KMS
Block Public Access
least-privilege IAM
versioning
CloudTrail where required
lifecycle policy
```

Example architecture:

```text
Invoice Pod
    |
    ↓
EKS Pod Identity
    |
    ↓
IAM Role
    |
    ↓
S3
```

The application does not receive long-lived AWS access keys.

---

# 31. IAM Permissions

The invoice workload should have only the permissions it needs.

For example:

```text
s3:PutObject
s3:GetObject
```

restricted to:

```text
arn:aws:s3:::company-ecommerce-invoices/invoices/*
```

Do not grant:

```text
s3:*
```

to the service.

---

# 32. KMS

If using SSE-KMS:

```text
Invoice Service
      |
      ↓
S3
      |
      ↓
KMS
```

The workload role may require the appropriate KMS permissions depending on
the chosen S3/KMS configuration.

Keep KMS permissions narrowly scoped.

---

# 33. PDF Generation

Invoice PDFs can be generated using:

```text
ReportLab
```

or another approved document-generation library.

Example:

```python
from reportlab.pdfgen import canvas
from io import BytesIO


def generate_invoice_pdf(invoice):

    buffer = BytesIO()

    pdf = canvas.Canvas(buffer)

    pdf.drawString(
        50,
        800,
        f"Invoice: {invoice.invoice_number}",
    )

    pdf.drawString(
        50,
        780,
        f"Order: {invoice.order_id}",
    )

    pdf.drawString(
        50,
        760,
        f"Total: {invoice.total_amount}",
    )

    pdf.save()

    buffer.seek(0)

    return buffer.getvalue()
```

This is a minimal example.

A production invoice template would contain:

- company information
- invoice number
- invoice date
- billing address
- shipping address
- item lines
- tax breakdown
- discounts
- shipping
- final total
- currency
- payment information
- required legal/tax information

---

# 34. PDF Hash

After generation:

```text
PDF bytes
   |
   ↓
SHA-256
   |
   ↓
document_sha256
```

Example:

```python
import hashlib

checksum = hashlib.sha256(
    pdf_bytes
).hexdigest()
```

Store the checksum in PostgreSQL.

This helps detect unexpected document changes.

---

# 35. Immutable Invoice

Once an invoice is issued:

```text
invoice number
invoice totals
tax
customer transaction snapshot
```

should generally not be silently modified.

If a correction is required, the accounting process may require:

```text
credit note
debit note
corrected invoice
cancellation
replacement invoice
```

rather than silently editing the historical document.

---

# 36. Invoice vs Order Cancellation

If an order is cancelled after invoice generation:

```text
Order
  |
  ↓
Cancelled
```

the Invoice Service should follow the organization's accounting process.

It should not simply:

```text
DELETE invoice
```

if the invoice has already become an official financial record.

---

# 37. Redis

Redis can be used for:

```text
invoice metadata cache
idempotency
short-lived download tokens
rate limiting
```

But the authoritative invoice record remains:

```text
PostgreSQL + S3
```

---

# 38. Kafka Topics

Invoice Service consumes:

```text
ecommerce.order.completed
```

It publishes:

```text
ecommerce.invoice.generated
ecommerce.invoice.failed
```

Potential downstream consumers:

```text
Notification Service
Reporting Service
Customer Service
```

---

# 39. Invoice Generated Event

```json
{
  "eventId": "evt-invoice-1001",
  "eventType": "INVOICE_GENERATED",
  "version": 1,
  "occurredAt": "2026-10-05T12:05:00Z",
  "source": "invoice-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "INV-2026-000001",
  "payload": {
    "invoiceId": "invoice-uuid",
    "invoiceNumber": "INV-2026-000001",
    "orderId": "ORD-1001",
    "customerId": "CUS-1001",
    "totalAmount": "5410.00",
    "currency": "INR",
    "s3Key": "invoices/2026/10/05/ORD-1001/INV-2026-000001.pdf"
  }
}
```

Do not expose private S3 credentials or internal security information in the
event.

---

# 40. Notification Integration

Notification Service can consume:

```text
INVOICE_GENERATED
```

and send:

```text
email
SMS
push notification
```

The Notification Service should retrieve the invoice through an authorized
mechanism rather than receiving the PDF as a Kafka payload.

---

# 41. Why Not Put PDF in Kafka?

Avoid:

```text
Kafka
  ↓
10 MB PDF
```

for every invoice.

Kafka should carry metadata/events.

Store large documents in:

```text
S3
```

and publish the object reference.

---

# 42. S3 Lifecycle

Invoice retention depends on:

- accounting requirements
- legal requirements
- company policy
- region
- business type

Do not automatically delete invoices after an arbitrary short period.

Lifecycle policies should be reviewed with the appropriate finance/legal
owners.

---

# 43. FastAPI Application

```python
from fastapi import FastAPI

app = FastAPI(
    title="Invoice Service",
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

---

# 44. Invoice Endpoint

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/api/v1/invoices"
)


@router.get("/{invoice_id}")
def get_invoice(invoice_id: str):

    invoice = invoice_service.get_invoice(
        invoice_id
    )

    return invoice
```

Authorization should be added before returning customer invoice data.

---

# 45. Download Endpoint

Conceptually:

```python
@router.get("/{invoice_id}/download")
def download_invoice(invoice_id: str):

    invoice = invoice_service.get_invoice(
        invoice_id
    )

    authorize_invoice_access(invoice)

    url = s3_service.generate_presigned_url(
        invoice.s3_key
    )

    return {
        "downloadUrl": url
    }
```

The signed URL should have a short expiration.

---

# 46. Dockerfile

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /build

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

Use a fully locked dependency installation strategy in the production
repository.

---

# 47. `.dockerignore`

```text
.git
.github

__pycache__
*.pyc

.pytest_cache
.mypy_cache
.ruff_cache

.venv
venv

.env
.env.*

coverage
htmlcov

README.md
```

---

# 48. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: invoice-postgres

    environment:

      POSTGRES_DB: invoices

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: invoice-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: invoice-kafka

    ports:

      - "9092:9092"


  invoice-service:

    build:
      context: .

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql+psycopg://postgres:postgres@postgres:5432/invoices

      REDIS_URL:
        redis://redis:6379

      KAFKA_BROKERS:
        kafka:9092

      AWS_REGION:
        ap-south-1

      S3_BUCKET:
        local-invoice-bucket

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For local testing, use LocalStack or another isolated AWS-compatible test
environment if S3 integration is required.

---

# 49. Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: invoice-service

  labels:

    app: invoice-service

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: invoice-service

  template:

    metadata:

      labels:

        app: invoice-service

    spec:

      serviceAccountName: invoice-service

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: invoice-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: invoice-service-config

            - secretRef:

                name: invoice-service-secrets

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

# 50. Service

```yaml
apiVersion: v1
kind: Service

metadata:

  name: invoice-service

  labels:

    app: invoice-service

spec:

  type: ClusterIP

  selector:

    app: invoice-service

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

  name: invoice-service-config

data:

  KAFKA_GROUP_ID:
    invoice-service

  INVOICE_PREFIX:
    invoices/

  PDF_EXPIRY_SECONDS:
    "900"
```

---

# 52. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:

  name: invoice-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: invoice-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/invoice-service

        property: DATABASE_URL

    - secretKey: REDIS_URL

      remoteRef:

        key: ecommerce/prod/invoice-service

        property: REDIS_URL

    - secretKey: KAFKA_BROKERS

      remoteRef:

        key: ecommerce/prod/invoice-service

        property: KAFKA_BROKERS
```

AWS S3 access should use workload identity rather than putting AWS access keys
into this Secret.

---

# 53. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:

  name: invoice-service

  labels:

    app: invoice-service
```

The corresponding EKS Pod Identity association should grant only the required
S3/KMS permissions.

---

# 54. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:

  name: invoice-service

spec:

  podSelector:

    matchLabels:

      app: invoice-service

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

# 55. HPA

Invoice generation is primarily event-driven.

For the HTTP API:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:

  name: invoice-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: invoice-service

  minReplicas: 3

  maxReplicas: 15

  behavior:

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

For invoice generation workers, KEDA can be a better fit because the workload
is event-driven.

---

# 56. KEDA Architecture

```text
Kafka
  |
  ↓
Invoice Events
  |
  ↓
KEDA
  |
  ↓
Invoice Consumer Pods
  |
  ↓
PDF generation
  |
  ↓
S3
```

Scaling should be bounded by:

```text
CPU
memory
PostgreSQL connections
S3 throughput
PDF generation capacity
Kafka partitions
```

Do not scale consumers without considering these dependencies.

---

# 57. PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:

  name: invoice-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: invoice-service
```

---

# 58. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:

  name: invoice-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: invoice-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 59. Metrics

Application:

```text
invoice_requests_total

invoice_request_duration_seconds

invoice_generation_total

invoice_generation_failures_total

invoice_generation_duration_seconds
```

Kafka:

```text
invoice_events_consumed_total

invoice_events_failed_total

invoice_consumer_lag
```

S3:

```text
invoice_s3_upload_total

invoice_s3_upload_failures_total

invoice_s3_upload_duration_seconds
```

Database:

```text
hikaricp_connections_active

hikaricp_connections_pending

invoice_db_query_duration_seconds
```

Business:

```text
invoices_generated_total

invoices_failed_total

duplicate_invoice_events_total
```

---

# 60. Grafana Dashboard

```text
Invoice Service
│
├── API
│   ├── RPS
│   ├── P95
│   ├── P99
│   └── 5xx
│
├── Kafka
│   ├── Consumer Lag
│   ├── Processing Rate
│   └── Failed Events
│
├── PDF
│   ├── Generation Time
│   ├── Success
│   └── Failure
│
├── S3
│   ├── Upload Success
│   ├── Upload Failure
│   └── Latency
│
├── PostgreSQL
│   ├── Connections
│   └── Query Latency
│
└── Business
    ├── Generated
    ├── Failed
    └── Duplicate Events
```

---

# 61. ELK Logging

Example:

```json
{
  "@timestamp": "2026-10-05T12:10:00Z",
  "level": "INFO",
  "service": "invoice-service",
  "event": "INVOICE_GENERATED",
  "invoiceId": "INV-ID-1001",
  "invoiceNumber": "INV-2026-000001",
  "orderId": "ORD-1001",
  "correlationId": "corr-order-1001",
  "s3Key": "invoices/2026/10/05/ORD-1001/INV-2026-000001.pdf"
}
```

Do not log invoice PDFs or sensitive customer information.

---

# 62. OpenTelemetry

Example trace:

```text
OrderCompleted Kafka Event
        |
        ↓
Invoice Consumer
        |
        ├── PostgreSQL
        |
        ├── PDF Generation
        |
        ├── S3 PutObject
        |
        └── Outbox Event
```

Trace propagation should use the project's standard correlation and W3C Trace
Context approach.

---

# 63. Jaeger Investigation

Suppose invoice generation takes:

```text
2.5 seconds
```

Jaeger shows:

```text
Kafka consume       20ms
PostgreSQL           50ms
PDF generation     1800ms
S3 upload           600ms
PostgreSQL update    30ms
```

The bottleneck is immediately visible.

Without tracing, engineers might incorrectly investigate PostgreSQL first.

---

# 64. SLO

Example:

```text
Invoice API availability:
99.95%

Invoice generation success:
99.9%

Invoice generation P95:
< 2 seconds

Event processing freshness:
99% within agreed target

Invoice retrieval P95:
< 150ms

Invoice data integrity:
100%
```

Exact targets must be established from business requirements and load testing.

---

# 65. Alerts

Critical:

```text
Invoice Service unavailable
Invoice generation failure rate high
Kafka consumer completely stalled
S3 upload failures sustained
```

Warning:

```text
Kafka lag increasing
PDF generation latency high
S3 upload latency high
RDS connections high
Hikari pending connections
Invoice generation backlog increasing
```

Business:

```text
Duplicate invoice events
Invoice/order mismatch
Invoice number generation failure
Invoice generation failure after order completion
```

---

# 66. CI Pipeline

```text
Developer
   |
   ↓
Pull Request
   |
   ↓
Lint
   |
   ↓
Unit Tests
   |
   ↓
Integration Tests
   |
   ↓
SonarQube
   |
   ↓
Trivy Filesystem
   |
   ↓
Docker Build
   |
   ↓
Trivy Image Scan
   |
   ↓
AWS OIDC
   |
   ↓
ECR
   |
   ↓
GitOps
```

---

# 67. GitHub Actions

```yaml
name: Invoice Service CI

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

  ECR_REPOSITORY: invoice-service

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

      - name: Install

        run: |

          pip install .

          pip install ".[dev]"

      - name: Lint

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

      - name: Configure AWS

        uses: aws-actions/configure-aws-credentials@v4

        with:

          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}

          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR

        id: ecr

        uses: aws-actions/amazon-ecr-login@v2

      - name: Build

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .

      - name: Image scan

        uses: aquasecurity/trivy-action@master

        with:

          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          severity: HIGH,CRITICAL

          exit-code: "1"

          ignore-unfixed: true

      - name: Push

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

---

# 68. SonarQube

SonarQube should analyze:

```text
Python code
test coverage
security issues
duplication
complexity
maintainability
```

Invoice-specific tests should include:

```text
duplicate events
invoice number concurrency
PDF generation
S3 failures
order snapshot
financial calculations
```

---

# 69. ECR

Repository:

```text
company-ecommerce/invoice-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/invoice-service:<git-sha>
```

Use immutable tags.

---

# 70. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: invoice-service

description: Ecommerce invoice generation service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 71. Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/invoice-service

  tag: ""

  pullPolicy: IfNotPresent

resources:

  requests:

    cpu: 250m

    memory: 512Mi

  limits:

    cpu: "1"

    memory: 1Gi

autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

serviceAccount:

  create: true

  name: invoice-service
```

---

# 72. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── invoice-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Promotion:

```text
ECR
 ↓
GitOps PR
 ↓
Approval
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

---

# 73. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:

  name: invoice-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/invoice-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 74. Failure Scenario — Kafka Down

```text
Order Completed
      |
      ↓
Kafka
      X
```

Invoice generation should not silently lose the order event.

Recovery mechanisms:

```text
Kafka durability
consumer retry
consumer group recovery
outbox/event replay
monitoring
```

If the order event was durably committed to Kafka, the Invoice Service can
resume consumption after recovery.

---

# 75. Failure Scenario — S3 Down

```text
Order Completed
      |
      ↓
Invoice Service
      |
      ↓
PDF generated
      |
      X
S3
```

The service should:

```text
persist invoice state
mark document generation/upload failure
retry safely
```

Do not generate multiple invoice numbers simply because S3 upload failed.

---

# 76. Failure Scenario — PDF Generation Failure

```text
Order Event
   |
   ↓
Invoice Metadata
   |
   ↓
PDF generation
   X
```

The invoice remains:

```text
GENERATING / FAILED
```

depending on where the failure occurred.

A retry worker can regenerate the document.

---

# 77. Important Idempotency Rule

If PDF generation retries:

```text
Same invoice
+
same invoice number
+
same financial snapshot
```

should be used.

Do not create:

```text
INV-2026-000001
INV-2026-000002
```

for the same order simply because the first PDF upload failed.

---

# 78. Failure Scenario — Duplicate Kafka Event

Input:

```text
ORDER_COMPLETED ORD-1001
```

twice.

Expected:

```text
one invoice
```

Not:

```text
two invoices
```

Protection:

```text
application idempotency
+
UNIQUE(order_id)
```

---

# 79. Failure Scenario — Database Down

If PostgreSQL is unavailable:

```text
invoice metadata cannot safely be persisted
```

The service should not report:

```text
Invoice generated successfully
```

unless the required durable state exists.

Kafka provides replay/recovery if the consumer has not acknowledged the event
prematurely.

---

# 80. Failure Scenario — KMS Permission Failure

S3 upload may fail with access errors if:

```text
IAM policy
KMS key policy
bucket policy
```

are incorrect.

Investigate:

```text
Pod Identity
IAM role
S3 policy
KMS policy
bucket policy
region
```

Do not blindly grant:

```text
kms:*
```

or:

```text
s3:*
```

as an emergency fix.

---

# 81. Example Production Incident

## Incident ID

```text
INC-2026-INVOICE-001
```

## Severity

```text
SEV-2
```

## Title

```text
Invoice S3 Upload Failures Caused Growing Invoice Backlog
```

## Date

```text
2026-10-05
```

---

# 82. Customer Impact

Orders completed successfully.

However:

```text
invoice PDFs
```

were delayed.

Customers could see:

```text
Invoice generation pending
```

instead of receiving the invoice immediately.

No order or payment data was lost.

---

# 83. Timeline

```text
15:00
Normal invoice generation.

15:08
S3 upload failures begin.

15:10
Invoice generation success rate drops.

15:12
Kafka consumer lag increases.

15:15
Alert fires.

15:17
SEV-2 declared.

15:20
DevOps checks S3 access errors.

15:23
IAM role is confirmed.

15:25
KMS AccessDenied identified.

15:28
Recent KMS policy change identified.

15:32
Approved KMS policy correction applied.

15:35
S3 uploads recover.

15:38
Kafka consumer lag begins falling.

15:45
Backlog cleared.

15:50
All invoice generation metrics normal.

16:00
Incident resolved.
```

---

# 84. Root Cause

A KMS key policy change removed the required permission for the Invoice
Service's workload identity to use the KMS key associated with the invoice
S3 bucket.

Therefore:

```text
Invoice Service
      |
      ↓
S3 PutObject
      |
      ↓
SSE-KMS
      |
      X
KMS AccessDenied
```

---

# 85. Amplification

The direct failure was KMS authorization.

The amplification mechanism was:

```text
S3 failures
    ↓
invoice retries
    ↓
Kafka backlog
    ↓
more concurrent invoice workers
    ↓
additional failed attempts
```

---

# 86. What Went Well

- Invoice records remained durable in PostgreSQL.
- Kafka retained unprocessed events.
- No duplicate invoices were created.
- Prometheus detected consumer lag.
- Jaeger/ELK provided evidence.
- GitOps allowed controlled infrastructure correction.

---

# 87. What Went Wrong

- KMS policy change was not caught before production.
- S3/KMS integration was not covered by a pre-production game day.
- Retry behavior generated unnecessary repeated failures.
- Alerting detected backlog after customer-visible delay had already begun.

---

# 88. Five Whys

### Why were invoices not uploaded?

S3 PutObject failed.

### Why?

KMS denied the encryption operation.

### Why?

The KMS key policy no longer permitted the Invoice Service workload role.

### Why?

A policy change was deployed without validating the dependent workload.

### Why?

The infrastructure pipeline lacked a complete application-to-KMS integration
test.

---

# 89. Corrective Actions

| Action | Owner |
|---|---|
| Restore least-privilege KMS permission | Cloud |
| Add KMS integration test | DevOps |
| Add S3/KMS pre-production test | QA |
| Add KMS AccessDenied alert | Security |
| Tune retry/backoff | Backend |
| Add invoice backlog SLO | SRE |
| Add dependency dashboard | Observability |
| Protect KMS changes with review | Platform |
| Add DR invoice restoration test | SRE |

---

# 90. Invoice Runbook

```text
1. Confirm invoice customer impact.
2. Check invoice generation rate.
3. Check Kafka consumer lag.
4. Check invoice failure rate.
5. Check PostgreSQL.
6. Check S3 upload metrics.
7. Check S3 AccessDenied.
8. Check KMS errors.
9. Check Pod Identity.
10. Check recent IAM/KMS changes.
11. Check PDF generation latency.
12. Stop uncontrolled retry amplification.
13. Fix dependency.
14. Verify new invoice generation.
15. Process backlog.
16. Check duplicate protection.
17. Validate customer invoice access.
18. Close incident.
19. Complete RCA.
```

---

# 91. Security Checklist

```text
[ ] Private S3 bucket
[ ] Block Public Access
[ ] SSE-KMS
[ ] Least-privilege IAM
[ ] EKS Pod Identity
[ ] No AWS keys in Kubernetes
[ ] No AWS keys in Git
[ ] Short-lived download URLs
[ ] Authorization on invoice access
[ ] Customer ownership validation
[ ] TLS
[ ] NetworkPolicy
[ ] Trivy
[ ] SonarQube
[ ] Audit logging
[ ] PII protection
```

---

# 92. Reliability Checklist

```text
[ ] Kafka consumer retry
[ ] Idempotent invoice generation
[ ] UNIQUE(order_id)
[ ] Transactional outbox
[ ] S3 retry
[ ] KMS monitoring
[ ] PDF retry
[ ] Kafka lag alert
[ ] S3 failure alert
[ ] Database monitoring
[ ] Redis monitoring
[ ] HPA limits
[ ] KEDA for event workers
[ ] PDB
[ ] Multi-AZ EKS
[ ] RDS Multi-AZ
[ ] S3 versioning
[ ] DR testing
```

---

# 93. Performance Considerations

PDF generation can consume:

```text
CPU
memory
temporary filesystem
```

Therefore invoice workers should have explicit:

```text
CPU requests
memory requests
CPU limits
memory limits
ephemeral-storage limits
```

Do not allow unlimited PDF generation to consume node resources.

---

# 94. PDF Worker Isolation

At higher scale:

```text
Invoice API
     |
     ↓
Kafka
     |
     ↓
Invoice Worker
     |
     ↓
PDF
     |
     ↓
S3
```

The PDF worker can use a separate Kubernetes workload/node pool if CPU-heavy
document generation begins competing with customer-facing APIs.

---

# 95. Capacity Planning

Example:

```text
1 invoice worker
=
20 invoices/sec
```

If measured capacity is actually 20 invoices/sec and peak workload is:

```text
200 invoices/sec
```

then:

```text
ceil(200 / 20)
=
10 workers
```

Then add headroom for:

```text
AZ failure
rolling deployment
CPU spikes
S3 latency
PDF complexity
```

The 20 invoices/sec value is only an example. Real capacity must come from
load testing.

---

# 96. S3 vs Database

Do not store PDF binary data directly inside PostgreSQL unless there is a
specific architectural reason.

Recommended:

```text
PostgreSQL
    ↓
invoice metadata

S3
    ↓
invoice document
```

PostgreSQL stores:

```text
invoice ID
invoice number
order ID
amount
status
S3 key
checksum
timestamps
```

S3 stores:

```text
invoice.pdf
```

---

# 97. Disaster Recovery

Invoice DR must cover:

```text
PostgreSQL
S3
KMS
ECR
EKS
Kafka
Secrets
IAM
DNS
```

S3 cross-region replication may be used where business requirements justify it.

KMS must also be deliberately designed for the DR region.

A database restore alone is insufficient if the referenced S3 documents are
missing.

---

# 98. Invoice Recovery Scenario

Suppose:

```text
PostgreSQL restored
```

but:

```text
S3 invoices missing
```

Database records may contain:

```text
s3://company-ecommerce-invoices/invoices/...
```

but the object cannot be downloaded.

Therefore invoice DR must test:

```text
metadata
+
documents
+
encryption
+
access
```

together.

---

# 99. Production Deployment Flow

```text
Developer
   |
   ↓
Feature Branch
   |
   ↓
Pull Request
   |
   ↓
CI
   |
   ├── Tests
   ├── SonarQube
   ├── Trivy
   └── Docker
          |
          ↓
         ECR
          |
          ↓
       GitOps PR
          |
          ↓
        ArgoCD
          |
          ↓
         EKS
```

---

# 100. Final Production Principles

1. Invoice Service owns invoice metadata and documents.
2. Order Service remains authoritative for order state.
3. Tax Service remains authoritative for tax calculation.
4. Invoice data should represent the completed transaction snapshot.
5. Never recalculate historical invoices using current prices.
6. Never recalculate historical invoices using current tax rules.
7. Use decimal-safe monetary calculations.
8. Store monetary values using appropriate database precision.
9. Invoice numbers require concurrency-safe generation.
10. Invoice generation must be idempotent.
11. `UNIQUE(order_id)` is an important database protection.
12. Kafka events should use versioned contracts.
13. Transactional outbox protects event publication.
14. Store PDFs in private S3.
15. Store invoice metadata in PostgreSQL.
16. Use SSE-KMS where required.
17. Use EKS Pod Identity instead of long-lived AWS credentials.
18. S3 objects should not be public.
19. Use short-lived signed URLs for customer downloads.
20. Protect invoice access with authorization.
21. Do not place large PDFs directly into Kafka events.
22. PDF generation should be independently scalable.
23. KEDA can scale event-driven invoice workers.
24. PDF generation must have resource limits.
25. S3 failures must not create duplicate invoices.
26. KMS failures must be observable.
27. Kafka backlog must have alerts and SLOs.
28. Invoice documents should be treated as financial records.
29. Corrections should follow the appropriate accounting process.
30. PostgreSQL and S3 must both be included in DR testing.
31. Prometheus monitors technical and business metrics.
32. Grafana provides operational dashboards.
33. ELK provides centralized logs.
34. OpenTelemetry and Jaeger provide distributed tracing.
35. Trivy scans filesystem and container images.
36. SonarQube checks code quality and security.
37. ECR stores immutable application images.
38. GitOps controls desired deployment state.
39. ArgoCD reconciles deployment state.
40. The final deployment path remains:

    `Developer → PR → CI → ECR → GitOps → ArgoCD → EKS`

# End of 180 — Invoice Service
```

### Phase 19 progress

```text
161 Auth Service             ✅
162 User Service             ✅
163 Customer Service         ✅
164 Catalog Service          ✅
165 Product Service          ✅
166 Inventory Service        ✅
167 Pricing Service          ✅
168 Promotion Service        ✅
169 Cart Service             ✅
170 Order Service            ✅
171 Payment Service          ✅
172 Fraud Service            ✅
173 Shipping Service          ✅
174 Delivery Service          ✅
175 Notification Service     ✅
176 Search Service           ✅
177 Recommendation Service   ✅
178 Review Service           ✅
179 Tax Service              ✅
180 Invoice Service          ✅

181 Reporting Service        ⏳
```

**Next: `181-Reporting-Service.md`** — the final Phase 19 service, covering event-driven reporting, Kafka, S3, batch processing, analytical workloads, PostgreSQL/read replicas, reporting isolation, Kubernetes/KEDA/CronJobs, CI/CD, GitOps, observability, failure scenarios and production RCA.