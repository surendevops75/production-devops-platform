# 177 — Recommendation Service

**File:** `177-Recommendation-Service.md`

```markdown
# 177 — Recommendation Service

## 1. Overview

The `recommendation-service` provides personalized and contextual product
recommendations for the e-commerce platform.

Examples:

- Recommended for you
- Frequently bought together
- Similar products
- Customers also viewed
- Customers also bought
- Trending products
- Recently viewed products
- Personalized home-page recommendations
- Category-based recommendations

The service should improve product discovery without becoming a dependency
for critical business transactions.

Technology:

- Python 3.13
- FastAPI
- PostgreSQL
- Redis
- Kafka
- S3
- Optional model-serving layer
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

# 2. Important Architecture Principle

Recommendations are **not authoritative business data**.

For example:

```text
Recommendation Service
        |
        ↓
"Recommended product = PROD-1001"
```

does not mean:

```text
Product exists
Product is available
Price is current
Inventory exists
Product can be purchased
```

The authoritative services remain:

```text
Product Service
Pricing Service
Inventory Service
Order Service
```

Therefore:

```text
Recommendation
      ↓
Product
      ↓
Pricing
      ↓
Inventory
      ↓
Order
```

---

# 3. Why Recommendation Is a Separate Service

Without a separate service:

```text
Product Service
      |
      ├── product database
      ├── search
      ├── recommendation logic
      ├── ML processing
      └── customer behavior processing
```

This creates a large and difficult-to-scale application.

Better:

```text
                         Ecommerce Platform
                                |
              ┌─────────────────┼─────────────────┐
              ↓                 ↓                 ↓
        Product Service    Search Service    Recommendation
                                                    |
                                                    ↓
                                               ML / Rules
```

Recommendation workloads can then scale independently.

---

# 4. Recommendation Types

The service supports several recommendation strategies.

## Personalized

```text
Customer → historical behavior → personalized products
```

## Similar Products

```text
Product A → product attributes → similar products
```

## Frequently Bought Together

```text
Product A
   +
Product B
   +
Product C
```

## Trending

```text
Recent product views/orders
          ↓
       popularity
```

## Recently Viewed

```text
Customer
   ↓
recent products
```

## Business Rules

Examples:

```text
promoted products
category recommendations
seasonal products
inventory-aware recommendations
```

---

# 5. High-Level Architecture

```text
                     Customer Activity
                            |
             ┌──────────────┼──────────────┐
             ↓              ↓              ↓
          Product View    Add Cart       Order
             |              |              |
             └──────────────┼──────────────┘
                            ↓
                          Kafka
                            |
                            ↓
                  Recommendation Service
                     /              \
                    ↓                ↓
             Feature Processing    Model
                    |                |
                    └───────┬────────┘
                            ↓
                         Redis
                            |
                            ↓
                     Recommendation API
                            |
                            ↓
                         Customer
```

---

# 6. Event-Driven Design

The recommendation service should not receive every customer interaction through
synchronous REST calls.

Instead:

```text
Product Viewed
Cart Updated
Order Completed
Product Rated
Search Performed
```

are emitted as events.

Example:

```text
Customer
   |
   ↓
Product Service / Cart / Order / Search
   |
   ↓
Kafka
   |
   ↓
Recommendation Event Consumer
```

---

# 7. Event Types

```text
PRODUCT_VIEWED
PRODUCT_SEARCHED
PRODUCT_ADDED_TO_CART
PRODUCT_REMOVED_FROM_CART
ORDER_COMPLETED
PRODUCT_RATED
PRODUCT_FAVORITED
```

The service should process only the events relevant to recommendation logic.

---

# 8. Event Envelope

```json
{
  "eventId": "evt-100001",
  "eventType": "PRODUCT_VIEWED",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "product-service",
  "correlationId": "corr-100001",
  "aggregateId": "PROD-1001",
  "payload": {
    "customerId": "CUS-1001",
    "productId": "PROD-1001",
    "categoryId": "electronics",
    "sessionId": "SESSION-1001"
  }
}
```

Sensitive information should be minimized.

---

# 9. Why Kafka

Recommendation processing is naturally asynchronous.

For example:

```text
100,000 product views
```

should not result in:

```text
100,000 synchronous calls
```

from the product API to recommendation logic.

Instead:

```text
Product View
     ↓
Kafka
     ↓
Recommendation Consumer
```

Benefits:

- Decoupling
- Buffering
- Replay
- Consumer scaling
- Independent failure handling
- Event-driven feature generation

---

# 10. Repository Structure

```text
recommendation-service/
│
├── app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   │
│   ├── api/
│   │   └── recommendation.py
│   │
│   ├── models/
│   │   ├── recommendation.py
│   │   ├── user_preference.py
│   │   └── interaction.py
│   │
│   ├── schemas/
│   │   ├── recommendation.py
│   │   └── event.py
│   │
│   ├── repositories/
│   │   └── recommendation_repository.py
│   │
│   ├── services/
│   │   ├── recommendation_service.py
│   │   ├── ranking_service.py
│   │   ├── feature_service.py
│   │   └── fallback_service.py
│   │
│   ├── consumers/
│   │   └── kafka_consumer.py
│   │
│   ├── model/
│   │   ├── inference.py
│   │   └── model_loader.py
│   │
│   ├── cache/
│   │   └── redis.py
│   │
│   ├── observability/
│   │   └── metrics.py
│   │
│   └── health/
│       └── health.py
│
├── models/
│   └── recommendation-model/
│
├── training/
│   ├── prepare_data.py
│   ├── train.py
│   └── evaluate.py
│
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
│   └── recommendation-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── api-deployment.yaml
│           ├── consumer-deployment.yaml
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

# 11. Technology Choice

The service uses:

```text
Python
   ↓
FastAPI
   ↓
Recommendation APIs
```

Python is particularly useful for:

- Data processing
- Machine learning
- Feature engineering
- Model inference
- Experimentation
- Recommendation algorithms

---

# 12. Dependencies

## `pyproject.toml`

```toml
[project]

name = "recommendation-service"

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
    "numpy",
    "scikit-learn",
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

For production ML workloads, dependency versions must be locked and tested.

---

# 13. Configuration

## `app/config.py`

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):

    app_name: str = "recommendation-service"

    port: int = 8080

    database_url: str

    redis_url: str

    kafka_brokers: str

    recommendation_cache_ttl: int = 300

    model_uri: str = "s3://company-ecommerce-models/recommendation/latest"

    class Config:
        env_file = ".env"


settings = Settings()
```

---

# 14. Environment Variables

```text
DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/recommendations

REDIS_URL=redis://localhost:6379/0

KAFKA_BROKERS=localhost:9092

RECOMMENDATION_CACHE_TTL=300

MODEL_URI=s3://company-ecommerce-models/recommendation/v1/model.pkl
```

Production credentials are injected through the platform.

---

# 15. Recommendation Database

PostgreSQL can store:

```text
Recommendation metadata
Model versions
Feature metadata
Customer preference summaries
Recommendation audit information
Experiment configuration
```

It should not necessarily store every raw click/view forever.

Large-scale behavioral data is better handled through appropriate event/data
platforms.

---

# 16. Interaction Table

Example:

```sql
CREATE TABLE customer_interactions (

    id BIGSERIAL PRIMARY KEY,

    event_id VARCHAR(200)
        NOT NULL UNIQUE,

    customer_id VARCHAR(100),

    session_id VARCHAR(100),

    event_type VARCHAR(100)
        NOT NULL,

    product_id VARCHAR(100),

    category_id VARCHAR(100),

    occurred_at TIMESTAMP
        NOT NULL
);

CREATE INDEX idx_interactions_customer
    ON customer_interactions(customer_id);

CREATE INDEX idx_interactions_product
    ON customer_interactions(product_id);

CREATE INDEX idx_interactions_occurred_at
    ON customer_interactions(occurred_at);
```

For very large production volumes, do not assume this single PostgreSQL table
should hold the entire event history.

Use the organization's analytics/data platform as volume grows.

---

# 17. Recommendation Table

```sql
CREATE TABLE recommendations (

    id BIGSERIAL PRIMARY KEY,

    customer_id VARCHAR(100)
        NOT NULL,

    recommendation_type VARCHAR(100)
        NOT NULL,

    product_id VARCHAR(100)
        NOT NULL,

    score DOUBLE PRECISION
        NOT NULL,

    model_version VARCHAR(100),

    generated_at TIMESTAMP
        NOT NULL,

    expires_at TIMESTAMP
);
```

---

# 18. Recommendation API

Primary endpoint:

```http
GET /api/v1/recommendations/{customerId}
```

Example:

```http
GET /api/v1/recommendations/CUS-1001?type=PERSONALIZED&limit=10
```

The authenticated customer identity should be checked before returning
personalized recommendations.

Do not allow:

```text
customer A
```

to request:

```text
customer B recommendations
```

simply by changing the URL.

---

# 19. API Response

```json
{
  "customerId": "CUS-1001",
  "type": "PERSONALIZED",
  "modelVersion": "model-2026-10-05-01",
  "recommendations": [
    {
      "productId": "PROD-1001",
      "score": 0.98
    },
    {
      "productId": "PROD-1002",
      "score": 0.91
    },
    {
      "productId": "PROD-1003",
      "score": 0.87
    }
  ]
}
```

---

# 20. Recommendation Pipeline

```text
Customer Events
      |
      ↓
Kafka
      |
      ↓
Feature Processing
      |
      ↓
Feature Store / Database
      |
      ↓
Recommendation Model
      |
      ↓
Ranking
      |
      ↓
Redis
      |
      ↓
Recommendation API
```

---

# 21. Recommendation Strategies

The service should not depend entirely on machine learning.

Use a layered approach:

```text
Layer 1
Personalized ML recommendations

Layer 2
Collaborative filtering

Layer 3
Content-based recommendations

Layer 4
Popular products

Layer 5
Category fallback

Layer 6
Business-safe default products
```

This is important for resilience.

---

# 22. Cold Start Problem

New customer:

```text
Customer
   |
   ↓
No history
```

There is no personalized model signal.

Fallback:

```text
New customer
   ↓
Trending products
   ↓
Category popularity
   ↓
General recommendations
```

---

# 23. New Product Cold Start

New product:

```text
Product
   |
   ↓
No clicks
No orders
No ratings
```

Possible recommendation signals:

```text
category
brand
attributes
price range
text embeddings
business rules
```

This is called an item cold-start problem.

---

# 24. Content-Based Recommendation

Example:

```text
Customer viewed:

"Wireless Bluetooth Headphones"
```

Product attributes:

```text
category = audio
brand = ExampleBrand
features = wireless, bluetooth, ANC
```

Find products with similar attributes.

```text
Viewed Product
      |
      ↓
Attributes
      |
      ↓
Similarity
      |
      ↓
Recommended Products
```

---

# 25. Collaborative Filtering

Collaborative filtering uses behavior patterns.

Example:

```text
Customer A:
Headphones
Keyboard
Mouse

Customer B:
Headphones
Keyboard
?
```

The system can infer:

```text
Customer B may be interested in Mouse
```

This uses behavior patterns rather than only product attributes.

---

# 26. Frequently Bought Together

Example:

```text
Product:
Laptop
```

Historical orders:

```text
Laptop + Mouse
Laptop + Backpack
Laptop + Keyboard
```

Recommendation:

```text
Frequently bought together:
Mouse
Backpack
Keyboard
```

This can be generated from order-event data.

---

# 27. Ranking

Candidates:

```text
Product A → score 0.90
Product B → score 0.85
Product C → score 0.60
```

After ranking:

```text
A
B
C
```

Ranking can consider:

```text
model score
product popularity
customer affinity
category relevance
availability
business constraints
```

Availability filtering should not be used as a substitute for authoritative
inventory validation.

---

# 28. Recommendation Filtering

Before returning recommendations:

```text
Candidate Products
       |
       ↓
Remove deleted products
       |
       ↓
Remove invalid products
       |
       ↓
Remove blocked products
       |
       ↓
Apply business rules
       |
       ↓
Return top N
```

---

# 29. Product Validation

A recommendation may reference:

```text
PROD-1001
```

but the product may have been:

```text
deleted
unpublished
restricted
```

Therefore the recommendation system needs a strategy for invalid candidates.

Possible approaches:

```text
event-driven product status updates
periodic reconciliation
short-lived cache
Product Service validation for critical cases
```

Do not synchronously call Product Service for every candidate at high scale.

---

# 30. Redis Recommendation Cache

Example:

```text
recommendations:v1:CUS-1001:PERSONALIZED
```

Value:

```json
{
  "modelVersion": "model-2026-10-05-01",
  "products": [
    "PROD-1001",
    "PROD-1002",
    "PROD-1003"
  ]
}
```

TTL:

```text
300 seconds
```

is illustrative.

---

# 31. Why Cache Recommendations

Recommendation generation may involve:

```text
database
model inference
feature retrieval
ranking
```

Doing this for every request can be expensive.

Instead:

```text
Customer
   |
   ↓
Redis
   |
   ├── HIT → return
   |
   └── MISS → generate → cache → return
```

---

# 32. Cache Stampede

Suppose a popular customer segment has expired recommendations.

Thousands of requests arrive:

```text
Cache expired
     |
     ↓
10,000 requests
     |
     ↓
10,000 recommendation calculations
```

Protection:

```text
request coalescing
distributed lock
TTL jitter
background refresh
pre-computation
```

---

# 33. Precomputed Recommendations

For many users:

```text
Kafka events
    ↓
Recommendation worker
    ↓
Generate recommendations
    ↓
Redis
```

Then API requests are cheap:

```text
GET recommendations
       ↓
Redis
       ↓
response
```

This is often better than expensive synchronous model inference on every
customer request.

---

# 34. Online vs Offline Recommendations

## Offline

```text
Events
  ↓
Batch processing
  ↓
Model training
  ↓
Recommendations
```

## Online

```text
Request
  ↓
Real-time features
  ↓
Model inference
  ↓
Recommendation
```

## Hybrid

This project uses:

```text
Offline model
+
Near-real-time events
+
Redis cached recommendations
```

---

# 35. Model Training

Example:

```text
Historical Events
      |
      ↓
Data Preparation
      |
      ↓
Feature Engineering
      |
      ↓
Training
      |
      ↓
Evaluation
      |
      ↓
Model Artifact
      |
      ↓
S3
```

Example artifact:

```text
s3://company-ecommerce-models/recommendation/
    v1/
        model.pkl
        metadata.json
```

---

# 36. Model Versioning

Never use:

```text
latest.pkl
```

as the only reference.

Prefer:

```text
model-2026-10-05-01
model-2026-10-12-01
model-2026-10-20-01
```

Record:

```text
model version
training data period
features
algorithm
evaluation metrics
deployment date
```

---

# 37. Model Deployment

Architecture:

```text
S3
 |
 ↓
Model Artifact
 |
 ↓
Recommendation Worker
 |
 ↓
Load model into memory
```

Do not download the model from S3 on every customer request.

Load it during:

```text
startup
or
controlled model refresh
```

---

# 38. Model Memory

Some models can be large.

For example:

```text
Model = 2 GB
```

If:

```text
20 Pods
```

then:

```text
20 × 2 GB
```

of model memory may be required.

Therefore model size affects:

```text
Pod memory
node capacity
Karpenter
startup time
deployment time
```

---

# 39. Model Loading

Startup flow:

```text
Pod starts
   |
   ↓
Download model
   |
   ↓
Load model
   |
   ↓
Validate model
   |
   ↓
Ready
```

Readiness should not become true before the model is actually usable if the
model is mandatory for the selected serving mode.

---

# 40. Model Download Failure

```text
Pod
 |
 ↓
S3
 |
 X
network/IAM
```

The Pod should not serve misleading recommendations using a partially loaded
model.

Possible strategies:

```text
fail readiness
use previous cached model
use rule-based fallback
```

The choice depends on business requirements.

---

# 41. S3 Model Storage

Recommended:

```text
S3 bucket
company-ecommerce-models
```

Structure:

```text
recommendation/
├── model-001/
├── model-002/
└── model-003/
```

Enable:

```text
versioning
encryption
access control
lifecycle
audit logging
```

---

# 42. IAM

Recommendation workload may require:

```text
s3:GetObject
```

for:

```text
arn:aws:s3:::company-ecommerce-models/recommendation/*
```

It should not receive:

```text
s3:*
```

across all buckets.

Use EKS Pod Identity where appropriate.

---

# 43. Kafka Consumer

Conceptual:

```python
async def process_event(event):

    if event.event_type == "PRODUCT_VIEWED":
        await process_product_view(event)

    elif event.event_type == "ORDER_COMPLETED":
        await process_order(event)

    elif event.event_type == "PRODUCT_RATED":
        await process_rating(event)
```

After successful processing:

```text
commit Kafka offset
```

not before.

---

# 44. Idempotency

Store:

```text
eventId
```

and check:

```text
already processed?
```

Example:

```text
evt-1001
```

received twice:

```text
First → process
Second → skip
```

This prevents duplicate feature updates.

---

# 45. Event Ordering

Suppose:

```text
PRODUCT_VIEWED
PRODUCT_ADDED_TO_CART
ORDER_COMPLETED
```

The recommendation engine may use event timestamps and sequence information
where required.

Do not assume globally ordered events across all customers.

Kafka ordering is generally guaranteed only within the relevant partitioning
boundary.

---

# 46. Kafka Partitioning

For customer behavior events:

```text
partition key = customerId
```

can help maintain ordering for events belonging to the same customer.

For product-centric aggregation:

```text
partition key = productId
```

may be appropriate.

The correct key depends on the processing model.

---

# 47. Consumer Scaling

```text
Kafka
  |
  ├── Partition 1
  ├── Partition 2
  ├── Partition 3
  └── Partition 4
```

Consumer group:

```text
recommendation-consumer
```

Maximum useful consumer parallelism is constrained by partition count.

If:

```text
4 partitions
```

and:

```text
20 consumers
```

only a subset can actively own partitions at once.

---

# 48. KEDA

For Kafka consumers:

```text
Kafka lag
   ↓
KEDA
   ↓
Consumer replicas
```

Example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: recommendation-consumer
spec:
  scaleTargetRef:
    name: recommendation-consumer
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.ecommerce.svc.cluster.local:9092
        consumerGroup: recommendation-consumer
        topic: ecommerce-events
        lagThreshold: "1000"
```

Exact KEDA syntax depends on the deployed version and authentication model.

---

# 49. KEDA + Kafka Constraint

If:

```text
Kafka topic
=
10 partitions
```

then scaling consumers beyond the useful partition parallelism will not
increase processing throughput for that consumer group.

Therefore:

```text
Kafka partitions
+
consumer replicas
+
event processing time
```

must be planned together.

---

# 50. HPA for Recommendation API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: recommendation-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: recommendation-service
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

CPU is only one signal.

For recommendation APIs, useful custom metrics can include:

```text
request rate
latency
queueing
model inference time
```

---

# 51. Worker and API Separation

Use:

```text
recommendation-api
```

for customer requests.

Use:

```text
recommendation-consumer
```

for event processing.

Use:

```text
recommendation-training
```

for training workloads.

These workloads should not necessarily share the same node pool.

---

# 52. Kubernetes Architecture

```text
EKS
│
├── recommendation-api
│
├── recommendation-consumer
│
└── recommendation-training
        |
        ↓
       S3
```

Training jobs can be scheduled separately from customer-facing workloads.

---

# 53. Training Jobs

Training should not run continuously inside an API Pod.

Bad:

```text
Recommendation API
      |
      └── train model
```

Better:

```text
Scheduled / triggered Job
       |
       ↓
Training
       |
       ↓
Evaluation
       |
       ↓
Model artifact
       |
       ↓
S3
```

---

# 54. Training Approval

A model should not automatically reach production simply because training
finished.

Pipeline:

```text
Training
   ↓
Evaluation
   ↓
Compare with current model
   ↓
Approval
   ↓
Register model
   ↓
Deploy
   ↓
Monitor
```

---

# 55. Model Evaluation

Possible metrics:

```text
Precision@K
Recall@K
NDCG@K
CTR
Conversion rate
Coverage
Diversity
Novelty
```

The exact metric depends on the recommendation objective.

---

# 56. Business Metrics

Monitor:

```text
recommendation impressions
recommendation clicks
add-to-cart rate
conversion rate
revenue influenced
zero recommendation rate
fallback rate
```

Infrastructure health alone cannot tell whether the recommendation system is
useful.

---

# 57. Recommendation Diversity

Bad result:

```text
Product A
Product A
Product A
Product A
```

or:

```text
same brand
same category
same product family
```

for every customer.

Ranking should consider diversity where appropriate.

---

# 58. Business Rules

Example:

```text
Do not recommend:
- unpublished products
- prohibited products
- deleted products
- unavailable products where business rules disallow them
```

Business rules should be explicit.

Do not bury them inside a machine-learning model where they cannot be audited.

---

# 59. Fallback Strategy

Recommendation model unavailable:

```text
ML model
   X
   |
   ↓
Personalized cache
   ↓
Popular products
   ↓
Category products
   ↓
Safe default
```

The customer should still be able to browse and purchase products.

---

# 60. Recommendation Failure Must Not Break Checkout

This is critical.

Bad architecture:

```text
Order Service
     |
     ↓
Recommendation Service
     |
     X
     |
     ↓
Order fails
```

Correct:

```text
Order Service
     |
     ├── Product
     ├── Pricing
     ├── Inventory
     └── Payment
```

Recommendation is optional.

---

# 61. API Timeout

Recommendation API should have a bounded timeout.

For example:

```text
homepage request
      |
      ├── recommendation timeout
      |
      ↓
homepage continues
```

Possible fallback:

```text
popular products
```

This is graceful degradation.

---

# 62. Redis Failure

```text
Recommendation API
       |
       X
     Redis
```

Do not allow Redis failure to take down the entire application.

Possible fallback:

```text
Redis failure
    ↓
Database/precomputed store
    ↓
Popular products
```

Avoid uncontrolled regeneration for every request.

---

# 63. Model Failure

If model inference fails:

```text
Model
  X
  |
  ↓
Rule-based recommendation
```

Alert:

```text
recommendation_model_failure_total
```

but customer browsing remains available.

---

# 64. Kafka Failure

If Kafka is unavailable:

```text
Event ingestion
      X
```

The recommendation API can continue serving previously generated
recommendations.

Once Kafka recovers:

```text
consumer catches up
```

and recommendations become current again.

---

# 65. Recommendation Staleness

Example:

```text
Recommendation generated:
10:00

Current time:
10:30
```

If the freshness requirement is:

```text
< 10 minutes
```

the recommendation should be refreshed or fallback behavior applied.

Metric:

```text
recommendation_age_seconds
```

---

# 66. Dockerfile

```dockerfile
FROM python:3.13-slim AS builder

WORKDIR /app

COPY pyproject.toml ./

RUN pip install \
    --no-cache-dir \
    --prefix=/install \
    fastapi \
    uvicorn[standard] \
    pydantic \
    pydantic-settings \
    sqlalchemy \
    psycopg[binary] \
    redis \
    aiokafka \
    numpy \
    scikit-learn \
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

For production, use locked dependencies rather than resolving floating
dependencies during image creation.

---

# 67. `.dockerignore`

```text
.git
.github

__pycache__
*.pyc

.pytest_cache
.coverage

.venv
venv

.env
.env.*

tests
training

README.md
```

Training code may be built into a separate training image.

---

# 68. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: recommendation-postgres

    environment:

      POSTGRES_DB: recommendations

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: recommendation-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: recommendation-kafka

    ports:

      - "9092:9092"


  recommendation-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql+psycopg://postgres:postgres@postgres:5432/recommendations

      REDIS_URL:
        redis://redis:6379/0

      KAFKA_BROKERS:
        kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 69. FastAPI Main

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

The production implementation should expose actual Prometheus metrics through
the chosen metrics library.

---

# 70. Recommendation Service Logic

Conceptual:

```python
class RecommendationService:

    async def get_recommendations(
        self,
        customer_id: str,
        recommendation_type: str,
        limit: int,
    ):

        cached = await self.cache.get(
            customer_id,
            recommendation_type,
        )

        if cached:
            return cached

        recommendations = (
            await self.generate_recommendations(
                customer_id,
                recommendation_type,
            )
        )

        if not recommendations:

            recommendations = (
                await self.fallback()
            )

        await self.cache.set(
            customer_id,
            recommendation_type,
            recommendations,
        )

        return recommendations
```

Production implementation should also protect against cache stampedes and
bounded generation concurrency.

---

# 71. Rule-Based Fallback

```python
async def fallback(self):

    return await self.product_repository.get_popular_products(
        limit=10
    )
```

The fallback must not perform an expensive unrestricted database scan.

---

# 72. Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: recommendation-service

  labels:

    app: recommendation-service

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: recommendation-service

  template:

    metadata:

      labels:

        app: recommendation-service

    spec:

      serviceAccountName: recommendation-service

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: recommendation-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: recommendation-service-config

            - secretRef:

                name: recommendation-service-secrets

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

# 73. Consumer Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: recommendation-consumer

  labels:

    app: recommendation-consumer

spec:

  replicas: 2

  selector:

    matchLabels:

      app: recommendation-consumer

  template:

    metadata:

      labels:

        app: recommendation-consumer

    spec:

      serviceAccountName: recommendation-service

      containers:

        - name: recommendation-consumer

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          command:

            - python

            - "-m"

            - app.consumers.kafka_consumer

          envFrom:

            - configMapRef:

                name: recommendation-service-config

            - secretRef:

                name: recommendation-service-secrets

          resources:

            requests:

              cpu: 250m

              memory: 512Mi

            limits:

              cpu: "2"

              memory: 2Gi
```

---

# 74. Why Consumer Memory Is Higher

The consumer may perform:

```text
feature processing
batch processing
ranking
model operations
```

which can consume more memory than a lightweight HTTP API.

Actual limits must be measured.

---

# 75. Service

```yaml
apiVersion: v1
kind: Service

metadata:

  name: recommendation-service

spec:

  type: ClusterIP

  selector:

    app: recommendation-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 76. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:

  name: recommendation-service-config

data:

  REDIS_URL:
    redis://redis.ecommerce.svc.cluster.local:6379/0

  KAFKA_BROKERS:
    kafka.ecommerce.svc.cluster.local:9092

  RECOMMENDATION_CACHE_TTL:
    "300"

  MODEL_URI:
    s3://company-ecommerce-models/recommendation/model-001/model.pkl
```

---

# 77. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:

  name: recommendation-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: recommendation-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/recommendation-service

        property: DATABASE_URL
```

---

# 78. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:

  name: recommendation-service

  labels:

    app: recommendation-service
```

Potential AWS access:

```text
S3 model bucket
```

through:

```text
EKS Pod Identity
```

with least-privilege IAM.

---

# 79. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:

  name: recommendation-service

spec:

  podSelector:

    matchLabels:

      app: recommendation-service

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

          port: 443

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

# 80. PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:

  name: recommendation-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: recommendation-service
```

---

# 81. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:

  name: recommendation-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: recommendation-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 82. Recommendation Metrics

API:

```text
recommendation_requests_total

recommendation_request_duration_seconds

recommendation_errors_total

recommendation_timeout_total
```

Cache:

```text
recommendation_cache_hits_total

recommendation_cache_misses_total

recommendation_cache_generation_total
```

Model:

```text
model_inference_total

model_inference_duration_seconds

model_load_failure_total

model_version
```

Events:

```text
recommendation_events_processed_total

recommendation_events_failed_total

recommendation_consumer_lag
```

Business:

```text
recommendation_impressions_total

recommendation_clicks_total

recommendation_conversion_total

recommendation_fallback_total
```

---

# 83. Grafana Dashboard

```text
Recommendation Service
│
├── API
│   ├── Request Rate
│   ├── P95
│   ├── P99
│   └── Errors
│
├── Cache
│   ├── Hit Ratio
│   └── Misses
│
├── Model
│   ├── Version
│   ├── Inference Time
│   └── Failures
│
├── Kafka
│   ├── Consumer Lag
│   ├── Processing Rate
│   └── Errors
│
├── Business
│   ├── Impressions
│   ├── CTR
│   └── Conversion
│
└── Infrastructure
    ├── CPU
    ├── Memory
    └── Pods
```

---

# 84. Logging

Example:

```json
{
  "@timestamp": "2026-10-05T20:00:00Z",
  "level": "INFO",
  "service": "recommendation-service",
  "customerId": "CUS-1001",
  "recommendationType": "PERSONALIZED",
  "modelVersion": "model-001",
  "cacheHit": true,
  "resultCount": 10,
  "durationMs": 8
}
```

Do not log:

```text
raw behavioral datasets
sensitive customer attributes
authentication tokens
private model data
```

unless explicitly required and protected.

---

# 85. OpenTelemetry

Request:

```text
Customer
   |
   ↓
Recommendation API
   |
   ├── Redis
   |
   └── Feature Store
```

Event processing:

```text
Kafka
  |
  ↓
Recommendation Consumer
  |
  ├── PostgreSQL
  ├── Redis
  └── Model
```

OpenTelemetry allows engineers to correlate processing latency across these
components.

---

# 86. Jaeger Investigation

Example:

```text
Recommendation API
       |
       ├── Redis GET       3ms
       |
       └── Feature lookup 80ms
             |
             └── PostgreSQL 75ms
```

If the API becomes slow:

```text
P95 = 500ms
```

Jaeger can determine whether the problem is:

```text
Redis
Database
Model
Network
Application
```

---

# 87. SLO

Example:

```text
Recommendation API availability:
99.9%

Recommendation API P95:
< 200ms

Recommendation cache hit ratio:
> 90%

Recommendation generation success:
> 99%

Event processing freshness:
99% within defined target
```

These are example targets and must be validated through performance testing.

---

# 88. Alerts

Critical:

```text
Recommendation API 5xx high
Recommendation API unavailable
Kafka consumer stopped
Model unavailable
```

Warning:

```text
Consumer lag high
Cache hit ratio low
Recommendation latency high
Fallback rate high
Model inference slow
Recommendation freshness degraded
```

Business:

```text
CTR suddenly drops
Zero recommendation rate increases
Recommendation conversion drops
```

Business alerts should be designed carefully to avoid false positives caused by
normal traffic changes.

---

# 89. Failure Scenario — Redis Down

```text
Recommendation API
       |
       X
     Redis
```

Expected:

```text
Fallback to controlled recommendation generation
```

But avoid:

```text
10,000 concurrent requests
      ↓
10,000 model computations
```

Use:

```text
bounded concurrency
request coalescing
fallback recommendations
```

---

# 90. Failure Scenario — Kafka Down

Existing recommendations:

```text
Redis
```

can continue serving.

New customer behavior events:

```text
not processed
```

until Kafka recovers.

Monitor:

```text
consumer lag
event freshness
```

After recovery:

```text
consumer catches up
```

---

# 91. Failure Scenario — Model Corrupt

Suppose:

```text
model-002
```

was deployed.

Inference fails.

Do not immediately restart all Pods repeatedly.

Use:

```text
model-001
```

or:

```text
rule-based fallback
```

if available.

Rollback the model deployment through the controlled release mechanism.

---

# 92. Failure Scenario — Bad Recommendations

Example:

```text
Customer searches:
Laptop

Recommendations:
Baby products
Cleaning products
```

Infrastructure may show:

```text
CPU normal
Memory normal
Latency normal
```

This is a model/data problem.

Investigate:

```text
model version
feature pipeline
training data
ranking
filters
business rules
```

---

# 93. Failure Scenario — Model Latency

Suppose:

```text
Normal inference:
20ms

Current:
800ms
```

Check:

```text
model size
CPU
memory
threading
batch size
feature retrieval
serialization
Python runtime
node pressure
```

If model serving becomes CPU-heavy, consider:

```text
dedicated compute nodes
optimized model
batch inference
separate model-serving service
```

---

# 94. Recommendation Node Pool

For heavy inference:

```text
EKS
│
├── General Node Pool
│
├── API Node Pool
│
└── ML/Compute Node Pool
```

This prevents:

```text
ML workload
```

from consuming capacity needed by:

```text
order-service
payment-service
```

---

# 95. Karpenter

Karpenter can provision appropriate nodes when recommendation consumers or
model-serving workloads scale.

Example:

```text
Recommendation workload
       |
       ↓
Pod pending
       |
       ↓
Karpenter
       |
       ↓
Compute node
       |
       ↓
Pod scheduled
```

Node requirements should match the workload.

---

# 96. CI/CD

Pipeline:

```text
Developer
   ↓
PR
   ↓
Tests
   ↓
SonarQube
   ↓
Trivy
   ↓
Docker Build
   ↓
Image Scan
   ↓
ECR
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

Model deployment should have a separate controlled lifecycle from application
code when appropriate.

---

# 97. GitHub Actions

```yaml
name: Recommendation Service CI

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

  ECR_REPOSITORY: recommendation-service

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

          pip install \
            fastapi \
            uvicorn \
            pydantic \
            pydantic-settings \
            sqlalchemy \
            psycopg[binary] \
            redis \
            aiokafka \
            numpy \
            scikit-learn \
            pytest \
            pytest-asyncio \
            httpx \
            ruff

      - name: Lint

        run: ruff check .

      - name: Test

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

      - name: Build

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

      - name: Push

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

Actions should be pinned according to the organization's supply-chain policy.

---

# 98. SonarQube

SonarQube should analyze:

```text
Python source
test coverage
security issues
code smells
duplication
complexity
```

ML-specific testing should additionally validate:

```text
feature generation
ranking
fallback
model loading
model version
```

---

# 99. ECR

Repository:

```text
company-ecommerce/recommendation-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/recommendation-service:<git-sha>
```

Use immutable image tags.

---

# 100. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: recommendation-service

description: Ecommerce recommendation service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 101. Production Values

```yaml
api:

  replicaCount: 3

consumer:

  replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/recommendation-service

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

  name: recommendation-service
```

---

# 102. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── recommendation-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

The GitOps repository controls:

```text
image
replicas
resources
environment configuration
HPA
KEDA
network policies
```

---

# 103. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: recommendation-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/recommendation-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 104. Production Incident

## Incident ID

```text
INC-2026-RECOMMENDATION-001
```

## Severity

```text
SEV-2
```

## Title

```text
Recommendation Model Deployment Increased API Latency
```

## Date

```text
2026-10-05
```

---

# 105. Customer Impact

The recommendation API experienced:

```text
P95:
180ms → 1.9s

P99:
400ms → 5.2s
```

The home page became slower.

Core purchasing remained operational because recommendation requests were
optional.

---

# 106. Timeline

```text
20:00
Recommendation service healthy.

20:05
Model version 2026-10-05-02 deployed.

20:07
Model inference latency increases.

20:09
Recommendation P95 crosses SLO.

20:11
Homepage latency increases.

20:12
SEV-2 declared.

20:14
Jaeger identifies model inference span.

20:16
CPU utilization increases on recommendation Pods.

20:18
Model team confirms new model has significantly higher inference cost.

20:20
Traffic routing moved back to previous model.

20:23
Inference latency decreases.

20:25
Recommendation API returns to SLO.

20:30
Homepage latency normal.

20:35
Incident resolved.
```

---

# 107. Root Cause

The new model produced better offline recommendation metrics but had much
higher runtime inference cost.

The model was not tested under production traffic before deployment.

---

# 108. Amplification

```text
New model
   ↓
Higher inference CPU
   ↓
Longer requests
   ↓
More concurrent requests
   ↓
Higher CPU
   ↓
More latency
```

HPA increased replicas, but the model's per-request cost remained high.

Scaling helped only partially.

---

# 109. What Went Well

- Recommendation was not a hard dependency for checkout.
- Prometheus detected latency.
- Jaeger identified model inference.
- Previous model remained available.
- GitOps/model rollback was quick.
- Core product ordering remained operational.

---

# 110. What Went Wrong

- No production-like inference load test.
- Offline model quality was evaluated, but runtime cost was not.
- No model CPU budget.
- No automated model rollback based on latency.

---

# 111. Five Whys

### Why did recommendation latency increase?

Model inference became slower.

### Why?

The new model required more computation.

### Why was it deployed?

Offline evaluation showed improved recommendation quality.

### Why was runtime cost missed?

Production-scale inference testing was not performed.

### Why?

Model validation and platform performance validation were treated as separate
processes.

---

# 112. Corrective Actions

| Action | Owner |
|---|---|
| Add inference load testing | ML/Performance |
| Add model CPU benchmark | ML |
| Add model latency gate | SRE |
| Add automated rollback threshold | Platform |
| Record model resource requirements | ML |
| Add model canary deployment | DevOps |
| Add model-version dashboard | SRE |
| Add recommendation fallback test | QA |

---

# 113. Recommendation SLO Dashboard

```text
Recommendation
│
├── API Availability
├── P50
├── P95
├── P99
├── Error Rate
├── Timeout Rate
│
├── Cache
│   ├── Hit Ratio
│   └── Generation Rate
│
├── Model
│   ├── Version
│   ├── Inference Latency
│   ├── Failure Rate
│   └── CPU
│
├── Kafka
│   ├── Lag
│   └── Processing Rate
│
└── Business
    ├── CTR
    ├── Conversion
    └── Fallback Rate
```

---

# 114. Production Checklist

```text
[ ] API implemented
[ ] Authentication/authorization
[ ] Customer isolation
[ ] Kafka events
[ ] Event idempotency
[ ] Event versioning
[ ] Consumer scaling
[ ] KEDA
[ ] Redis cache
[ ] Cache stampede protection
[ ] PostgreSQL
[ ] Model storage
[ ] S3
[ ] Pod Identity
[ ] Least-privilege IAM
[ ] NetworkPolicy
[ ] Resource requests
[ ] Resource limits
[ ] HPA
[ ] PDB
[ ] Docker
[ ] ECR
[ ] Trivy
[ ] SonarQube
[ ] GitHub Actions
[ ] GitOps
[ ] ArgoCD
[ ] Prometheus
[ ] Grafana
[ ] ELK
[ ] OpenTelemetry
[ ] Jaeger
[ ] SLO
[ ] Alerts
[ ] Runbook
[ ] Cold-start fallback
[ ] Model rollback
[ ] Model versioning
[ ] Model performance testing
[ ] Kafka failure test
[ ] Redis failure test
[ ] Model failure test
[ ] Recommendation quality validation
[ ] RCA process
```

---

# 115. Final Architecture

```text
                         Customer
                            |
                            ↓
                     Recommendation API
                            |
                    ┌───────┴────────┐
                    ↓                ↓
                  Redis          Feature Store
                    |                |
                    └───────┬────────┘
                            ↓
                     Ranking / Model
                            |
                            ↓
                     Recommendations
                            |
                            ↓
                         Customer


Customer/Product/Cart/Order/Search Events
                  |
                  ↓
                Kafka
                  |
                  ↓
       Recommendation Consumer
          /               \
         ↓                 ↓
     Features          Aggregations
         |                 |
         └────────┬────────┘
                  ↓
               Redis
                  |
                  ↓
          Recommendation API


Historical Data
      |
      ↓
Training Pipeline
      |
      ↓
Model Evaluation
      |
      ↓
Approved Model
      |
      ↓
S3
      |
      ↓
Recommendation Workload
```

---

# 116. Final Production Principles

1. Recommendation is an optional capability, not a critical transaction
   dependency.
2. Product, pricing and inventory remain authoritative elsewhere.
3. Kafka carries behavioral events asynchronously.
4. Recommendation processing must be idempotent.
5. Customer events should use an appropriate Kafka partitioning strategy.
6. Consumer scaling is limited by Kafka partition parallelism.
7. KEDA can scale consumers based on Kafka lag.
8. Redis provides fast recommendation retrieval.
9. Redis is not the source of truth.
10. Cache stampede protection is required.
11. Personalized recommendations need a cold-start strategy.
12. New products require an item cold-start strategy.
13. ML should not be the only recommendation strategy.
14. Rule-based and popularity fallbacks protect availability.
15. Model artifacts should be versioned.
16. Models should be stored outside the application container image when
    appropriate.
17. S3 should use encryption and least-privilege access.
18. EKS Pod Identity should be preferred over static AWS credentials.
19. Model loading should happen before a model-dependent Pod becomes ready.
20. Model inference latency must be monitored.
21. Offline model quality does not guarantee production performance.
22. Recommendation quality requires business metrics such as CTR and
    conversion.
23. Search/product/order systems must not fail because recommendations are
    unavailable.
24. HPA scaling must consider model CPU and memory requirements.
25. Heavy ML workloads should use appropriate node pools.
26. Karpenter can provide elastic compute capacity where appropriate.
27. Model deployment should have rollback capability.
28. Production model releases should be canaried or otherwise controlled.
29. Recommendation freshness should have an explicit SLO.
30. Every major recommendation failure requires an RCA.

# End of 177 — Recommendation Service
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
173 Shipping Service         ✅
174 Delivery Service         ✅
175 Notification Service     ✅
176 Search Service           ✅
177 Recommendation Service   ✅

178 Review Service           ⏳
179 Tax Service              ⏳
180 Invoice Service          ⏳
181 Reporting Service        ⏳
```

**Next: `178-Review-Service.md`** — review creation/moderation, ratings aggregation, anti-spam/abuse controls, PostgreSQL + Redis, asynchronous moderation events, Kafka, Kubernetes/EKS, CI/CD, GitOps, observability, SLOs, production failures and RCA.