# 181 — Reporting Service

**File:** `181-Reporting-Service.md`

```markdown
# 181 — Reporting Service

## 1. Overview

The `reporting-service` is the final microservice in Phase 19.

Its responsibility is to provide operational, business, and analytical reports
without putting unnecessary reporting load on the transactional microservices.

The Reporting Service is intentionally designed differently from services such
as Order, Payment, Cart, and Inventory.

Transactional services optimize for:

- Low latency
- High availability
- Short transactions
- Customer-facing requests

Reporting workloads optimize for:

- Aggregation
- Historical analysis
- Large scans
- Scheduled reports
- Business metrics
- Data exports
- Long-running queries

Therefore reporting must be isolated from the customer-facing transaction path.

Technology:

- Python 3.13
- FastAPI
- PostgreSQL
- Kafka
- Amazon S3
- Redis
- Kubernetes
- EKS
- KEDA
- Kubernetes CronJobs
- Docker
- ECR
- GitHub Actions
- ArgoCD
- Prometheus
- Grafana
- ELK
- OpenTelemetry
- Jaeger
```

---

# 2. Why Reporting Needs Its Own Service

A common mistake is allowing the Order Service to execute large reporting queries.

For example:

```text
GET /orders/report?from=2026-01-01&to=2026-10-01
```

If the Order Service performs:

```sql
SELECT ...
FROM orders
JOIN ...
GROUP BY ...
ORDER BY ...
```

against the same database used by checkout, reporting can affect:

```text
Order latency
Payment latency
Checkout availability
Database connections
Database CPU
Database I/O
```

Instead:

```text
Customer-facing traffic
        |
        ↓
Transactional Services
        |
        ↓
Events
        |
        ↓
Kafka
        |
        ↓
Reporting Service
        |
        ├── PostgreSQL reporting store
        ├── S3 data lake/files
        └── Report workers
```

---

# 3. Core Principle

Reporting must be **isolated from the critical transaction path**.

The architecture should prevent:

```text
Reporting query
      ↓
RDS overload
      ↓
Order latency
      ↓
Checkout failure
```

The preferred architecture is:

```text
Transactional DB
      |
      ↓
Events
      |
      ↓
Reporting pipeline
      |
      ↓
Reporting storage
      |
      ↓
Reports
```

---

# 4. Reporting Responsibilities

The service provides:

```text
Sales reports
Order reports
Revenue reports
Product reports
Customer reports
Tax reports
Payment reports
Inventory reports
Daily summaries
Monthly summaries
CSV exports
Scheduled reports
Operational dashboards
```

It should not become an unrestricted replacement for a data warehouse.

For large enterprise analytics, the architecture can later evolve toward:

```text
Kafka
  ↓
S3
  ↓
Data Lake
  ↓
Data Warehouse/Lakehouse
  ↓
BI Platform
```

---

# 5. Reporting Architecture

```text
                    ┌─────────────────────┐
                    │ Transactional Apps  │
                    └──────────┬──────────┘
                               |
                               ↓
                            Kafka
                               |
              ┌────────────────┼────────────────┐
              ↓                ↓                ↓
       Reporting Consumer   S3 Archive     Other Consumers
              |
              ↓
      Reporting PostgreSQL
              |
       ┌──────┴──────┐
       ↓             ↓
   Fast Reports   Aggregations
       |
       ↓
  Reporting API
       |
       ↓
     Users
```

---

# 6. Event-Driven Reporting

The Reporting Service should consume events such as:

```text
ORDER_CREATED
ORDER_COMPLETED
ORDER_CANCELLED
PAYMENT_COMPLETED
PAYMENT_FAILED
INVENTORY_RESERVED
INVENTORY_RELEASED
INVOICE_GENERATED
REFUND_COMPLETED
CUSTOMER_CREATED
PRODUCT_UPDATED
TAX_CALCULATED
```

The exact event contracts are governed centrally.

---

# 7. Why Kafka Is Useful

Suppose 1,000 orders are completed.

Instead of repeatedly querying Order Service:

```text
Reporting Service
       |
       ├── Order Service
       ├── Payment Service
       ├── Product Service
       ├── Customer Service
       └── Invoice Service
```

the reporting system receives events:

```text
OrderCompleted
PaymentCompleted
InvoiceGenerated
```

and builds its own reporting view.

This creates:

```text
eventual consistency
```

which is acceptable for most reporting workloads.

---

# 8. Reporting Does Not Need Real-Time Transactional Consistency

For example:

```text
Transaction completed:
10:00:00

Reporting dashboard:
10:00:03
```

A few seconds of reporting lag may be acceptable.

But:

```text
Payment authorization
Inventory reservation
Order completion
```

cannot depend on the reporting database.

---

# 9. Repository Structure

```text
reporting-service/
│
├── app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   │
│   ├── models/
│   │   ├── order_fact.py
│   │   ├── payment_fact.py
│   │   ├── product_dimension.py
│   │   └── daily_sales.py
│   │
│   ├── schemas/
│   │   ├── report.py
│   │   └── export.py
│   │
│   ├── repositories/
│   │   └── report_repository.py
│   │
│   ├── services/
│   │   ├── report_service.py
│   │   ├── aggregation_service.py
│   │   └── export_service.py
│   │
│   ├── consumers/
│   │   ├── order_consumer.py
│   │   ├── payment_consumer.py
│   │   └── invoice_consumer.py
│   │
│   ├── producers/
│   │   └── report_event_producer.py
│   │
│   ├── storage/
│   │   └── s3_service.py
│   │
│   ├── api/
│   │   └── reports.py
│   │
│   └── observability/
│       └── metrics.py
│
├── migrations/
│   ├── V1__create_order_facts.sql
│   ├── V2__create_payment_facts.sql
│   └── V3__create_daily_sales.sql
│
├── jobs/
│   ├── daily_sales.py
│   └── monthly_sales.py
│
├── tests/
│   ├── test_reports.py
│   ├── test_aggregation.py
│   └── test_events.py
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
│   └── reporting-service/
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
│           ├── servicemonitor.yaml
│           └── cronjob.yaml
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
        ├── ci.yaml
        └── security.yaml
```

---

# 10. Python Stack

```text
Python
  ↓
FastAPI
  ↓
SQLAlchemy
  ↓
PostgreSQL
```

Event processing:

```text
Kafka
  ↓
Consumer
  ↓
Reporting Store
```

Object storage:

```text
S3
  ↓
CSV
Parquet
Generated Reports
Exports
Archives
```

---

# 11. `pyproject.toml`

```toml
[project]

name = "reporting-service"

version = "1.0.0"

requires-python = ">=3.13"

dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "sqlalchemy",
    "psycopg[binary]",
    "pydantic",
    "pydantic-settings",
    "aiokafka",
    "boto3",
    "redis",
    "prometheus-client",
    "pandas"
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

For production, pin tested versions rather than relying on floating dependency
versions.

---

# 12. Configuration

## `app/config.py`

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):

    app_name: str = "reporting-service"

    database_url: str

    redis_url: str

    kafka_brokers: str

    kafka_group_id: str = "reporting-service"

    aws_region: str = "ap-south-1"

    s3_bucket: str

    s3_prefix: str = "reports/"


settings = Settings()
```

---

# 13. Environment Variables

```text
DATABASE_URL=postgresql+psycopg://reporting_user:password@localhost:5432/reporting

REDIS_URL=redis://localhost:6379

KAFKA_BROKERS=localhost:9092

KAFKA_GROUP_ID=reporting-service

AWS_REGION=ap-south-1

S3_BUCKET=company-ecommerce-reporting

S3_PREFIX=reports/
```

Production credentials must come from the project's Secrets Manager /
External Secrets architecture.

---

# 14. Reporting Data Model

A reporting database should be designed for reporting queries rather than
simply copying every transactional table.

One useful model is:

```text
Fact tables
    |
    ├── order_fact
    ├── payment_fact
    └── invoice_fact

Dimension tables
    |
    ├── customer_dimension
    ├── product_dimension
    └── date_dimension
```

This resembles a simplified star-schema approach.

---

# 15. Order Fact

```sql
CREATE TABLE order_fact (

    event_id UUID PRIMARY KEY,

    order_id VARCHAR(100) NOT NULL,

    customer_id VARCHAR(100) NOT NULL,

    order_date DATE NOT NULL,

    currency VARCHAR(10) NOT NULL,

    subtotal NUMERIC(19,4) NOT NULL,

    discount_amount NUMERIC(19,4) NOT NULL,

    tax_amount NUMERIC(19,4) NOT NULL,

    shipping_amount NUMERIC(19,4) NOT NULL,

    total_amount NUMERIC(19,4) NOT NULL,

    status VARCHAR(50) NOT NULL,

    occurred_at TIMESTAMP NOT NULL,

    ingested_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_order_fact_date
ON order_fact(order_date);

CREATE INDEX idx_order_fact_customer
ON order_fact(customer_id);

CREATE INDEX idx_order_fact_status
ON order_fact(status);
```

---

# 16. Payment Fact

```sql
CREATE TABLE payment_fact (

    event_id UUID PRIMARY KEY,

    payment_id VARCHAR(100) NOT NULL,

    order_id VARCHAR(100) NOT NULL,

    payment_method VARCHAR(100),

    amount NUMERIC(19,4) NOT NULL,

    currency VARCHAR(10) NOT NULL,

    status VARCHAR(50) NOT NULL,

    occurred_at TIMESTAMP NOT NULL,

    ingested_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payment_fact_date
ON payment_fact(occurred_at);

CREATE INDEX idx_payment_fact_order
ON payment_fact(order_id);
```

---

# 17. Daily Sales Aggregate

```sql
CREATE TABLE daily_sales (

    report_date DATE PRIMARY KEY,

    order_count BIGINT NOT NULL,

    successful_order_count BIGINT NOT NULL,

    cancelled_order_count BIGINT NOT NULL,

    gross_sales NUMERIC(19,4) NOT NULL,

    discounts NUMERIC(19,4) NOT NULL,

    taxes NUMERIC(19,4) NOT NULL,

    shipping NUMERIC(19,4) NOT NULL,

    net_sales NUMERIC(19,4) NOT NULL,

    generated_at TIMESTAMP NOT NULL
);
```

---

# 18. Why Aggregation Tables Exist

Instead of calculating:

```sql
SELECT
    DATE(order_date),
    COUNT(*),
    SUM(total_amount)
FROM order_fact
GROUP BY DATE(order_date);
```

for every dashboard request, a scheduled aggregation job can generate:

```text
daily_sales
```

once.

Then the dashboard performs a very small query.

---

# 19. Reporting APIs

```text
GET /api/v1/reports/sales/daily

GET /api/v1/reports/sales/monthly

GET /api/v1/reports/orders

GET /api/v1/reports/payments

GET /api/v1/reports/products/top

POST /api/v1/reports/exports

GET /api/v1/reports/exports/{exportId}
```

Large report generation should be asynchronous.

---

# 20. Synchronous vs Asynchronous Reports

Small query:

```text
GET /reports/sales/daily
        |
        ↓
API
        |
        ↓
response
```

Large report:

```text
POST /reports/exports
        |
        ↓
Create export job
        |
        ↓
Kafka/SQS
        |
        ↓
Worker
        |
        ↓
S3
        |
        ↓
Download URL
```

Do not keep an HTTP request open for 10 minutes while generating a large CSV.

---

# 21. Report Export State

```text
REQUESTED
    |
    ↓
PROCESSING
    |
    ├── COMPLETED
    |
    └── FAILED
```

Example:

```json
{
  "exportId": "EXP-1001",
  "status": "PROCESSING"
}
```

Later:

```json
{
  "exportId": "EXP-1001",
  "status": "COMPLETED",
  "downloadUrl": "..."
}
```

---

# 22. Kafka Event Consumer

Conceptual:

```python
from aiokafka import AIOKafkaConsumer
import json


async def consume_orders():

    consumer = AIOKafkaConsumer(
        "ecommerce.order.completed",
        bootstrap_servers=settings.kafka_brokers,
        group_id=settings.kafka_group_id,
        enable_auto_commit=False,
    )

    await consumer.start()

    try:

        async for message in consumer:

            event = json.loads(
                message.value.decode("utf-8")
            )

            await process_order_event(
                event
            )

            await consumer.commit()

    finally:

        await consumer.stop()
```

The actual implementation should include:

- retries
- poison-message handling
- DLQ strategy
- graceful shutdown
- observability
- idempotency
- partition-aware processing

---

# 23. Idempotent Event Processing

Kafka may redeliver:

```text
ORDER_COMPLETED
```

Therefore:

```text
event_id
```

should be unique.

The database constraint:

```sql
PRIMARY KEY(event_id)
```

prevents duplicate insertion.

---

# 24. Processing Flow

```text
Kafka Event
     |
     ↓
Validate schema
     |
     ↓
Check event ID
     |
     ├── Already processed → ignore safely
     |
     └── New
          |
          ↓
      Insert fact
          |
          ↓
       Commit
          |
          ↓
     Kafka offset commit
```

Never acknowledge the event before durable processing.

---

# 25. Event Ordering

Reporting may receive:

```text
ORDER_CREATED
ORDER_COMPLETED
ORDER_CANCELLED
```

Events may arrive in unexpected ways depending on topic/partition design.

Partitioning should use an appropriate key such as:

```text
orderId
```

when order-level ordering is required.

However, consumers should still be designed defensively.

---

# 26. Eventual Consistency

Example:

```text
Order completed
10:00:00

Reporting database
10:00:02
```

The reporting API may temporarily show:

```text
999 orders
```

while the transactional system has:

```text
1000 orders
```

This is acceptable if the reporting SLO explicitly allows that lag.

---

# 27. Reporting Freshness SLO

Example:

```text
99% of reporting events processed
within 30 seconds.
```

This is different from an API availability SLO.

Measure:

```text
event occurred time
        vs
reporting ingestion time
```

---

# 28. Kafka Consumer Lag

Critical metric:

```text
consumer lag
```

If:

```text
producer rate = 1,000 events/sec

consumer rate = 500 events/sec
```

lag increases.

The solution is not always:

```text
increase replicas
```

First check:

```text
Kafka partitions
consumer processing time
PostgreSQL throughput
S3 throughput
CPU
memory
downstream bottlenecks
```

---

# 29. KEDA

Reporting consumers are good candidates for event-driven scaling.

Architecture:

```text
Kafka
  |
  ↓
Consumer Lag
  |
  ↓
KEDA
  |
  ↓
Reporting Worker Pods
```

Example concept:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: reporting-consumer

spec:

  scaleTargetRef:
    name: reporting-service

  minReplicaCount: 2

  maxReplicaCount: 20

  triggers:

    - type: kafka

      metadata:

        bootstrapServers: kafka:9092

        consumerGroup: reporting-service

        topic: ecommerce.order.completed

        lagThreshold: "1000"
```

Exact KEDA syntax and version should be verified against the deployed KEDA
version.

---

# 30. KEDA Does Not Remove Bottlenecks

Suppose:

```text
Kafka lag ↑
```

and KEDA scales:

```text
5 Pods → 30 Pods
```

but PostgreSQL can only handle the existing workload.

Then:

```text
30 Pods
   ↓
PostgreSQL
   ↓
connection exhaustion
```

Therefore scaling must consider:

```text
Kafka
+
Pods
+
PostgreSQL
+
S3
+
CPU
+
memory
```

---

# 31. Reporting Database

Preferred production architecture:

```text
Transactional RDS
        |
        X
  reporting queries
```

Instead:

```text
Events
  |
  ↓
Reporting DB
  |
  ↓
Reports
```

This keeps analytical queries away from the primary transaction database.

---

# 32. Read Replica Alternative

A read replica can sometimes help:

```text
RDS Primary
   |
   ↓
Read Replica
   |
   ↓
Reporting queries
```

But a read replica is not a complete reporting architecture.

It can still have:

- replication lag
- resource contention
- expensive queries
- operational coupling to the transactional database

For heavier analytics, use a separate reporting store/data platform.

---

# 33. S3 Data Lake Pattern

For larger reporting workloads:

```text
Kafka
  |
  ↓
Reporting Pipeline
  |
  ↓
S3
  |
  ├── raw/
  ├── curated/
  └── aggregates/
```

Example:

```text
s3://company-ecommerce-reporting/
    raw/orders/
    raw/payments/
    curated/orders/
    aggregates/daily-sales/
```

S3 becomes a durable analytical data layer.

---

# 34. CSV Export

Example:

```python
import csv
from io import StringIO


def create_csv(rows):

    output = StringIO()

    writer = csv.writer(output)

    writer.writerow([
        "order_id",
        "customer_id",
        "total",
        "currency",
    ])

    for row in rows:

        writer.writerow([
            row.order_id,
            row.customer_id,
            row.total_amount,
            row.currency,
        ])

    return output.getvalue()
```

Large exports should stream/process in batches rather than loading millions of
records into memory.

---

# 35. Large Export Problem

Bad:

```python
rows = database.query_all()

csv = create_csv(rows)
```

If:

```text
10 million rows
```

memory consumption can become dangerous.

Better:

```text
Database
   |
   ↓
paged/batched query
   |
   ↓
streamed writer
   |
   ↓
S3 multipart upload
```

---

# 36. S3 Storage

Reports should be stored privately.

Example:

```text
reports/
    daily/
        2026/
            10/
                05/
                    daily-sales.csv

    monthly/
        2026/
            10/
                monthly-sales.csv
```

Use:

```text
SSE-KMS
versioning
Block Public Access
lifecycle policies
least-privilege IAM
```

---

# 37. Reporting Service IAM

The workload may require:

```text
s3:GetObject
s3:PutObject
```

only for:

```text
arn:aws:s3:::company-ecommerce-reporting/reports/*
```

It should not have unrestricted:

```text
s3:*
```

permissions.

---

# 38. Redis

Redis can cache frequently requested reports:

```text
report:daily-sales:2026-10-05
```

TTL:

```text
5 minutes
```

for example.

But Redis is not the source of truth.

If Redis fails:

```text
reporting should continue
```

with higher latency.

---

# 39. Daily Report Job

Example:

```text
23:59
   |
   ↓
CronJob
   |
   ↓
Calculate daily aggregates
   |
   ↓
PostgreSQL
   |
   ↓
S3
```

The actual schedule should account for:

- timezone
- late-arriving events
- business cut-off time
- reconciliation

---

# 40. Kubernetes CronJob

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:

  name: reporting-daily-aggregation

spec:

  schedule: "10 0 * * *"

  concurrencyPolicy: Forbid

  successfulJobsHistoryLimit: 3

  failedJobsHistoryLimit: 5

  jobTemplate:

    spec:

      backoffLimit: 3

      template:

        spec:

          restartPolicy: OnFailure

          serviceAccountName: reporting-service

          containers:

            - name: aggregation

              image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

              command:

                - python

                - "-m"

                - "jobs.daily_sales"

              envFrom:

                - configMapRef:

                    name: reporting-service-config

                - secretRef:

                    name: reporting-service-secrets

              resources:

                requests:

                  cpu: 250m

                  memory: 512Mi

                limits:

                  cpu: "2"

                  memory: 2Gi
```

---

# 41. Why `concurrencyPolicy: Forbid`

Suppose yesterday's report is still running.

A second copy should not automatically start and execute the same expensive
aggregation.

Otherwise:

```text
Job A
   +
Job B
   +
Job C
```

could overload the reporting database.

---

# 42. Late-Arriving Events

Suppose:

```text
Order occurred:
23:59

Reporting received:
00:05
```

The daily report generated at:

```text
00:10
```

may need to include it.

Therefore reporting systems need a strategy for late events.

Possible approach:

```text
Daily report generated
        |
        ↓
Reconciliation window
        |
        ↓
Recalculate previous day
```

---

# 43. Report Reconciliation

A production reporting system should periodically compare:

```text
Transactional totals
        vs
Reporting totals
```

Example:

```text
Order Service:
10,000 completed orders

Reporting:
9,997 completed orders
```

Difference:

```text
3
```

This should trigger investigation.

---

# 44. Why Reporting Reconciliation Matters

Events can be:

```text
delayed
duplicated
malformed
dropped due to configuration errors
processed into DLQ
```

A reconciliation process detects data drift.

---

# 45. Dead Letter Queue

If an event cannot be processed:

```text
Kafka
  |
  ↓
Reporting Consumer
  |
  X
invalid event
  |
  ↓
DLQ
```

Do not continuously retry a permanently invalid event.

DLQ records should include:

```text
eventId
eventType
original topic
partition
offset
failure reason
first failure time
last retry time
```

---

# 46. Poison Message

Example:

```json
{
  "eventType": "ORDER_COMPLETED",
  "payload": {
    "total": "INVALID"
  }
}
```

If the consumer retries forever:

```text
same message
↓
failure
↓
retry
↓
failure
↓
retry
```

the partition may become blocked.

Use:

```text
bounded retries
+
DLQ
+
alert
```

---

# 47. Reporting Security

Reports can contain sensitive business data.

Security controls:

```text
Authentication
Authorization
RBAC
Customer-level access control
Admin-only reports
Audit logging
Encryption
Private S3
Short-lived signed URLs
PII minimization
```

Do not allow:

```text
GET /reports/all-customers
```

to every authenticated user.

---

# 48. Customer Report Access

If customers are allowed to retrieve their own reports:

```text
JWT
 |
 ↓
customerId
 |
 ↓
Authorization
 |
 ↓
Only customer's data
```

Never trust:

```text
/customer/{customerId}
```

alone.

The authenticated identity must be checked.

---

# 49. Admin Report Access

Internal reporting may use:

```text
role:
REPORTING_ADMIN
```

or:

```text
REPORTING_ANALYST
```

with controlled access.

Sensitive fields should be masked when unnecessary.

---

# 50. Dockerfile

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /build

COPY pyproject.toml .

RUN pip install \
    --no-cache-dir \
    --prefix=/install \
    .

COPY app ./app
COPY jobs ./jobs

FROM python:3.13-slim

WORKDIR /app

COPY --from=builder \
    /install \
    /usr/local

COPY app ./app
COPY jobs ./jobs

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

---

# 51. `.dockerignore`

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

# 52. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: reporting-postgres

    environment:

      POSTGRES_DB: reporting

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: reporting-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: reporting-kafka

    ports:

      - "9092:9092"


  reporting-service:

    build:
      context: .

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql+psycopg://postgres:postgres@postgres:5432/reporting

      REDIS_URL:
        redis://redis:6379

      KAFKA_BROKERS:
        kafka:9092

      KAFKA_GROUP_ID:
        reporting-service

      AWS_REGION:
        ap-south-1

      S3_BUCKET:
        local-reporting-bucket

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For production-like S3 tests, use an isolated AWS test environment or an
AWS-compatible local emulator.

---

# 53. Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: reporting-service

  labels:

    app: reporting-service

spec:

  replicas: 2

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: reporting-service

  template:

    metadata:

      labels:

        app: reporting-service

    spec:

      serviceAccountName: reporting-service

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: reporting-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: reporting-service-config

            - secretRef:

                name: reporting-service-secrets

          resources:

            requests:

              cpu: 250m

              memory: 512Mi

            limits:

              cpu: "1"

              memory: 1Gi

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

# 54. Kubernetes Service

```yaml
apiVersion: v1
kind: Service

metadata:

  name: reporting-service

  labels:

    app: reporting-service

spec:

  type: ClusterIP

  selector:

    app: reporting-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 55. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:

  name: reporting-service-config

data:

  KAFKA_GROUP_ID:
    reporting-service

  S3_PREFIX:
    reports/

  REPORT_TIMEZONE:
    Asia/Kolkata

  REPORT_CACHE_TTL:
    "300"
```

Timezone is configuration, not hard-coded business logic.

---

# 56. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:

  name: reporting-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: reporting-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/reporting-service

        property: DATABASE_URL

    - secretKey: REDIS_URL

      remoteRef:

        key: ecommerce/prod/reporting-service

        property: REDIS_URL

    - secretKey: KAFKA_BROKERS

      remoteRef:

        key: ecommerce/prod/reporting-service

        property: KAFKA_BROKERS
```

AWS credentials are not stored here.

S3 access uses EKS Pod Identity.

---

# 57. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:

  name: reporting-service

  labels:

    app: reporting-service
```

The associated workload IAM role should only access the required reporting S3
prefix and KMS key.

---

# 58. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:

  name: reporting-service

spec:

  podSelector:

    matchLabels:

      app: reporting-service

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

# 59. HPA

The API layer can use HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:

  name: reporting-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: reporting-service

  minReplicas: 2

  maxReplicas: 10

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

Consumer workers should use KEDA when Kafka lag is the more meaningful scaling
signal.

---

# 60. PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:

  name: reporting-service

spec:

  minAvailable: 1

  selector:

    matchLabels:

      app: reporting-service
```

Reporting can usually tolerate more disruption than Order or Payment, but this
depends on reporting freshness requirements.

---

# 61. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:

  name: reporting-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: reporting-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 62. Metrics

Application:

```text
report_requests_total

report_request_duration_seconds

report_failures_total

report_rows_processed_total
```

Kafka:

```text
reporting_consumer_lag

reporting_events_processed_total

reporting_events_failed_total

reporting_dlq_events_total
```

Database:

```text
reporting_db_query_duration_seconds

reporting_db_connections

reporting_db_query_errors_total
```

S3:

```text
report_export_upload_total

report_export_upload_failures_total

report_export_upload_duration_seconds
```

Business:

```text
orders_reported_total

sales_reported_total

reporting_reconciliation_difference
```

---

# 63. Grafana Dashboard

```text
Reporting Service
│
├── API
│   ├── RPS
│   ├── P95
│   └── Errors
│
├── Kafka
│   ├── Consumer Lag
│   ├── Event Rate
│   ├── Processing Rate
│   └── DLQ
│
├── PostgreSQL
│   ├── Query Latency
│   ├── Connections
│   └── CPU
│
├── S3
│   ├── Export Count
│   ├── Upload Latency
│   └── Upload Errors
│
└── Business
    ├── Daily Sales
    ├── Orders
    ├── Payments
    └── Reconciliation
```

---

# 64. ELK Logging

Example:

```json
{
  "@timestamp": "2026-10-05T23:59:00Z",
  "level": "INFO",
  "service": "reporting-service",
  "event": "DAILY_REPORT_COMPLETED",
  "reportDate": "2026-10-05",
  "orders": 10542,
  "sales": "5421000.00",
  "durationMs": 1850,
  "correlationId": "report-2026-10-05"
}
```

Do not log:

```text
customer passwords
payment credentials
AWS credentials
large report contents
unnecessary PII
```

---

# 65. OpenTelemetry

Example:

```text
Scheduled Report
      |
      ├── PostgreSQL query
      |
      ├── Aggregation
      |
      └── S3 upload
```

For event processing:

```text
Kafka event
    |
    ↓
Consumer
    |
    ├── PostgreSQL
    |
    └── Aggregation
```

This makes slow report generation easier to investigate.

---

# 66. Jaeger

Example slow report:

```text
Total:
12 seconds

PostgreSQL:
9 seconds

Aggregation:
2 seconds

S3:
1 second
```

The trace immediately points toward the database query.

Then investigate:

```text
execution plan
indexes
rows scanned
locks
query structure
partitioning
```

---

# 67. Reporting SLOs

Example:

```text
API availability:
99.9%

Report API P95:
< 1 second

Event processing freshness:
99% < 30 seconds

Daily report completion:
before agreed business deadline

Export success:
99.9%

Reconciliation accuracy:
100% within defined tolerance
```

Reporting SLOs are different from Order Service SLOs.

---

# 68. Alerts

Critical:

```text
Reporting API unavailable
Consumer completely stopped
Daily report missed business deadline
Large reconciliation discrepancy
```

Warning:

```text
Kafka lag high
DLQ increasing
Database query latency high
S3 export failures
Report generation duration high
Reporting database connections high
```

---

# 69. CI Pipeline

```text
Developer
    |
    ↓
Pull Request
    |
    ├── Ruff
    ├── Unit Tests
    ├── Integration Tests
    ├── SonarQube
    └── Trivy
          |
          ↓
      Docker Build
          |
          ↓
      Image Scan
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

# 70. GitHub Actions

```yaml
name: Reporting Service CI

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

  ECR_REPOSITORY: reporting-service

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

      - name: ECR Login

        id: ecr

        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .

      - name: Trivy Image Scan

        uses: aquasecurity/trivy-action@master

        with:

          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          severity: HIGH,CRITICAL

          exit-code: "1"

          ignore-unfixed: true

      - name: Push Image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

Actions should be pinned according to the organization's supply-chain policy.

---

# 71. SonarQube

SonarQube checks:

```text
Python quality
security issues
duplication
complexity
test coverage
maintainability
```

Reporting-specific tests should include:

```text
event idempotency
aggregation correctness
late events
duplicate events
large datasets
empty datasets
export generation
authorization
```

---

# 72. ECR

Repository:

```text
company-ecommerce/reporting-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/reporting-service:<git-sha>
```

Do not use:

```text
latest
```

for controlled production promotion.

---

# 73. Helm

## `Chart.yaml`

```yaml
apiVersion: v2

name: reporting-service

description: Ecommerce reporting service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 74. Production Values

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/reporting-service

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

  minReplicas: 2

  maxReplicas: 10

serviceAccount:

  create: true

  name: reporting-service
```

---

# 75. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── reporting-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Deployment:

```text
ECR
 ↓
GitOps PR
 ↓
Review
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

---

# 76. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:

  name: reporting-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/reporting-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 77. Failure Scenario — Kafka Lag

Symptoms:

```text
reporting_consumer_lag ↑
report freshness ↓
```

Investigate:

```text
Kafka partitions
consumer replicas
consumer processing time
PostgreSQL
CPU
memory
network
DLQ
recent deployment
```

Do not immediately increase replicas.

---

# 78. Failure Scenario — Reporting Database Slow

Symptoms:

```text
report P95 ↑
PostgreSQL CPU ↑
queries slow
```

Possible causes:

```text
missing index
large table scan
bad query
data growth
lock contention
too many concurrent reports
```

Mitigation:

```text
cancel runaway report if safe
rate-limit reports
optimize query
add appropriate index
precompute aggregation
move large analytics to S3/data platform
```

Do not solve every reporting problem by increasing database size.

---

# 79. Failure Scenario — S3 Export Failure

```text
Report generation
      |
      ↓
S3
      X
```

The export should remain:

```text
PROCESSING
```

or:

```text
FAILED
```

depending on the retry state.

Retry should use the same:

```text
exportId
```

rather than generating duplicate export records.

---

# 80. Failure Scenario — Duplicate Event

```text
ORDER_COMPLETED
       |
       ↓
Consumer
       |
       ↓
database
       |
       X
consumer crash
       |
       ↓
same event
```

Expected:

```text
one fact record
```

Database primary key on:

```text
event_id
```

provides a final idempotency barrier.

---

# 81. Failure Scenario — Reporting Pod OOMKilled

Large report:

```text
10 million rows
```

loaded into memory.

Result:

```text
OOMKilled
```

Correct approach:

```text
pagination
streaming
batch processing
S3 multipart upload
```

Do not simply increase:

```text
memory:
8Gi
```

without fixing the unbounded memory pattern.

---

# 82. Failure Scenario — Transactional Database Protected

One of the most important production tests:

```text
Large Reporting Query
        |
        X
Transactional RDS
```

The architecture should prevent reporting from reaching the primary
transactional database for normal reporting workloads.

If reporting accidentally starts querying the production transaction database,
the incident should be detected quickly.

---

# 83. Example Production Incident

## Incident ID

```text
INC-2026-REPORTING-001
```

## Severity

```text
SEV-2
```

## Title

```text
Reporting Consumer Lag Caused Stale Business Dashboard
```

## Date

```text
2026-10-05
```

---

# 84. Customer/Business Impact

Customer checkout remained healthy.

However:

```text
business dashboard
sales report
order analytics
```

were approximately 18 minutes behind real-time events.

No transactional orders were lost.

---

# 85. Timeline

```text
14:00
Normal reporting lag: < 5 seconds.

14:05
Order volume increases.

14:08
Kafka reporting lag starts increasing.

14:10
Consumer lag exceeds alert threshold.

14:12
Reporting dashboard becomes several minutes stale.

14:14
SEV-2 declared.

14:16
Prometheus shows reporting consumer CPU saturation.

14:18
Investigation finds expensive aggregation introduced
by a recent reporting code change.

14:20
Consumer processing time increases significantly.

14:22
Bad reporting deployment identified.

14:24
GitOps rollback initiated.

14:27
Consumer processing rate improves.

14:30
Kafka lag starts decreasing.

14:38
Reporting catches up.

14:42
Dashboard freshness returns to normal.

14:50
Incident closed.
```

---

# 86. Root Cause

A new reporting transformation introduced an inefficient aggregation operation
that significantly increased CPU consumption per event.

Before:

```text
average processing:
5ms/event
```

After:

```text
average processing:
60ms/event
```

Kafka production rate exceeded consumer processing capacity.

Therefore:

```text
event rate
    >
consumer rate
```

and lag increased.

---

# 87. Amplification

The primary failure:

```text
inefficient aggregation
```

Amplification:

```text
slow consumer
     ↓
Kafka lag
     ↓
KEDA increases workers
     ↓
PostgreSQL CPU increases
     ↓
workers become slower
     ↓
lag remains high
```

This is an important example of why autoscaling can sometimes amplify a
downstream bottleneck.

---

# 88. What Went Well

- Transactional services were unaffected.
- Kafka retained the events.
- Reporting had its own database.
- Prometheus detected consumer lag.
- GitOps made rollback controlled and auditable.
- Jaeger helped identify slow processing.
- No data loss occurred.

---

# 89. What Went Wrong

- Performance regression reached production.
- Consumer processing-time SLO was insufficient.
- Load testing did not include peak event rates.
- KEDA scaling limits were not tuned against database capacity.
- The reporting query path lacked an efficient aggregation strategy.

---

# 90. Five Whys

### Why was the dashboard stale?

Reporting consumers could not process events fast enough.

### Why?

Per-event processing time increased.

### Why?

A new aggregation implementation was computationally expensive.

### Why was it deployed?

Performance tests did not use production-scale event volume.

### Why did scaling not solve it?

Additional workers increased pressure on the reporting database.

---

# 91. Corrective Actions

| Action | Owner |
|---|---|
| Optimize aggregation | Backend |
| Add production-sized load test | Performance |
| Add consumer processing SLO | SRE |
| Add database saturation guardrail | Platform |
| Tune KEDA max replicas | Platform |
| Add event-rate dashboard | Observability |
| Add reporting performance regression tests | QA |
| Add canary deployment | DevOps |
| Add reconciliation checks | Data |
| Document reporting incident runbook | SRE |

---

# 92. Reporting Runbook

```text
1. Confirm report freshness impact.
2. Check Kafka consumer lag.
3. Check event production rate.
4. Check consumer processing rate.
5. Check Pod CPU/memory.
6. Check KEDA replica count.
7. Check reporting DB CPU/connections.
8. Check slow queries.
9. Check recent deployment.
10. Check DLQ.
11. Determine application vs infrastructure bottleneck.
12. Protect reporting database.
13. Roll back if deployment caused regression.
14. Tune scaling only after dependency capacity is known.
15. Wait for backlog recovery.
16. Validate report reconciliation.
17. Confirm freshness SLO.
18. Complete RCA.
```

---

# 93. Security Checklist

```text
[ ] Authentication
[ ] Authorization
[ ] RBAC
[ ] Customer data isolation
[ ] Private S3
[ ] Block Public Access
[ ] SSE-KMS
[ ] Pod Identity
[ ] Least privilege
[ ] NetworkPolicy
[ ] Signed URLs
[ ] Short URL expiry
[ ] PII minimization
[ ] Audit logging
[ ] Trivy
[ ] SonarQube
```

---

# 94. Reliability Checklist

```text
[ ] Kafka consumer idempotency
[ ] Event schema validation
[ ] DLQ
[ ] Retry policy
[ ] Consumer lag alerts
[ ] Reporting freshness SLO
[ ] Database isolation
[ ] Query timeouts
[ ] Large export isolation
[ ] KEDA limits
[ ] HPA limits
[ ] CronJob concurrency control
[ ] Reconciliation
[ ] S3 durability
[ ] DR testing
[ ] Backup testing
```

---

# 95. Performance Checklist

```text
[ ] Query indexes
[ ] Query execution plans
[ ] Pagination
[ ] Streaming exports
[ ] Batch processing
[ ] Precomputed aggregates
[ ] PostgreSQL connection limits
[ ] Kafka partition planning
[ ] Consumer processing rate
[ ] CPU profiling
[ ] Memory profiling
[ ] Load testing
[ ] Soak testing
[ ] Peak event testing
```

---

# 96. Reporting vs Transactional Services

| Area | Transactional Service | Reporting Service |
|---|---|---|
| Main goal | Transactions | Analytics |
| Latency | Very low | Moderate |
| Consistency | Strong | Eventual |
| Database | Transactional | Reporting |
| Queries | Small | Aggregation |
| Workload | Customer requests | Analytical |
| Scaling | HPA | HPA/KEDA |
| Events | Produces | Consumes |
| Large exports | No | Yes |
| S3 | Optional | Important |
| Failure impact | Customer-facing | Business reporting |

---

# 97. Reporting Data Flow

```text
                    ┌───────────────────┐
                    │ 21 Microservices  │
                    └─────────┬─────────┘
                              |
                              ↓
                           Kafka
                              |
              ┌───────────────┼────────────────┐
              ↓               ↓                ↓
        Reporting DB         S3          Other Systems
              |
              ↓
       Aggregation Jobs
              |
       ┌──────┴───────┐
       ↓              ↓
   Dashboards      Exports
       |
       ↓
   Reporting API
```

---

# 98. Disaster Recovery

Reporting recovery should consider:

```text
Kafka
PostgreSQL
S3
KMS
ECR
EKS
Secrets
IAM
GitOps
```

The reporting system is generally reconstructible from durable event history,
depending on retention.

This is a major architectural advantage.

Example:

```text
Reporting DB lost
       |
       ↓
Recreate DB
       |
       ↓
Replay Kafka / S3 source data
       |
       ↓
Rebuild reporting tables
```

The exact replay strategy depends on event retention and data lake retention.

---

# 99. Reporting Database Recovery

Do not assume:

```text
reporting database backup
```

is the only recovery mechanism.

Possible recovery:

```text
Backup restore
+
Kafka replay
+
S3 historical data
+
Reconciliation
```

After recovery:

```text
Transactional totals
       vs
Reporting totals
```

must be reconciled.

---

# 100. Final Production Principles

1. Reporting Service owns reporting workloads.
2. Transactional services own transactional state.
3. Reporting must not overload the Order database.
4. Kafka provides event-driven data ingestion.
5. Reporting is intentionally eventually consistent.
6. Reporting freshness must have an explicit SLO.
7. Event IDs provide an idempotency boundary.
8. Kafka offsets must not be committed before durable processing.
9. Poison messages must not retry forever.
10. DLQs are operational tools, not silent garbage bins.
11. Reporting databases should be optimized for analytical access.
12. Large exports should be asynchronous.
13. Large exports should use streaming/batching.
14. PDFs/CSVs should be stored in S3 rather than database blobs where practical.
15. S3 should remain private.
16. S3 access should use EKS Pod Identity.
17. KMS should be least privilege.
18. Customer reports require authorization.
19. Redis is a cache, not the source of truth.
20. KEDA is useful for event-driven reporting workers.
21. KEDA scaling must consider PostgreSQL and other downstream limits.
22. CronJobs should prevent overlapping expensive aggregations.
23. Late-arriving events require reconciliation.
24. Reporting should periodically reconcile against authoritative business
    totals.
25. PostgreSQL query optimization is part of reporting capacity planning.
26. Prometheus monitors lag, processing rate, database pressure and API health.
27. Grafana provides operational and business dashboards.
28. ELK provides centralized logs.
29. OpenTelemetry and Jaeger provide distributed tracing.
30. Trivy scans dependencies/filesystem/images.
31. SonarQube provides code-quality and security analysis.
32. ECR stores immutable application images.
33. GitOps controls deployment state.
34. ArgoCD reconciles deployment state.
35. Reporting failures should not take down checkout.
36. Reporting workloads should be independently scalable.
37. Reporting recovery should support replay and reconciliation.
38. Production reporting incidents require RCA.
39. The final deployment path remains:

    `Developer → PR → CI → ECR → GitOps → ArgoCD → EKS`

# End of 181 — Reporting Service
```

## Phase 19 is now complete

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
173 Shipping Service         ✅
174 Delivery Service         ✅
175 Notification Service     ✅
176 Search Service           ✅
177 Recommendation Service   ✅
178 Review Service           ✅
179 Tax Service              ✅
180 Invoice Service          ✅
181 Reporting Service        ✅
```

# Phase 20 — Platform Engineering

Now we move to the **exact structure you defined**:

```text
182-Reusable-Helm-Templates.md
183-Reusable-GitHub-Actions.md
184-Terraform-Modules.md
185-GitHub-Actions-Reusable-Workflows.md
186-Security-Policy-Templates.md
187-Monitoring-Templates.md
188-Alert-Rule-Templates.md
189-NetworkPolicy-Templates.md
190-ServiceAccount-Templates.md
191-GitOps-Generators.md
192-Microservice-Golden-Path.md
```

**Next: `182-Reusable-Helm-Templates.md`** — where we take the repeated Kubernetes/Helm patterns from all 21 services and turn them into a production-grade reusable platform template.