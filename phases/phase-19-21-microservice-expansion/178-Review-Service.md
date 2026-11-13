# 178 — Review Service

**File:** `178-Review-Service.md`

```markdown
# 178 — Review Service

## 1. Overview

The `review-service` manages customer reviews and ratings for products.

Responsibilities:

- Create product reviews
- Update reviews
- Delete reviews
- Retrieve product reviews
- Calculate rating summaries
- Calculate rating distributions
- Verify purchase eligibility
- Prevent duplicate reviews
- Moderate reviews
- Detect abusive/spam content
- Publish review events
- Maintain review statistics
- Cache frequently requested review summaries

Technology:

- TypeScript
- Node.js
- NestJS
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
```

---

# 2. Why Review Service Exists

Reviews are customer-generated data.

A product may have:

```text
Product
├── name
├── price
├── inventory
└── reviews
```

But storing all review functionality inside Product Service creates unnecessary
coupling.

Instead:

```text
Product Service
       |
       ↓
Product data

Review Service
       |
       ↓
Review data
```

The Product Service remains responsible for the product.

The Review Service owns customer reviews and ratings.

---

# 3. High-Level Architecture

```text
                         Customer
                            |
                            ↓
                    CloudFront / WAF
                            |
                            ↓
                           ALB
                            |
                            ↓
                    Review Service
                       /         \
                      ↓           ↓
                PostgreSQL       Redis
                      |
                      ↓
                  Kafka Events
                      |
          ┌───────────┼────────────┐
          ↓           ↓            ↓
   Notification   Moderation   Recommendation
```

---

# 4. Review Ownership

The Review Service owns:

```text
Review
Rating
Review status
Moderation status
Verified purchase state
Review metadata
Review statistics
```

It does not own:

```text
Product
Order
Payment
Customer account
Inventory
```

Those belong to other services.

---

# 5. Verified Purchase

A major production requirement is determining whether a customer actually
purchased the product.

The Review Service should not blindly trust:

```text
customerId
productId
```

provided by a client.

Architecture:

```text
Customer
   |
   ↓
Review Service
   |
   ↓
Order Service
   |
   ↓
Purchase verification
```

For scale, an event-driven approach is preferable.

Example:

```text
OrderCompleted
      |
      ↓
Kafka
      |
      ↓
Review Service
      |
      ↓
Verified Purchase Record
```

Then review creation can validate locally.

---

# 6. Review Lifecycle

```text
DRAFT
  |
  ↓
SUBMITTED
  |
  ↓
PENDING_MODERATION
  |
  ├── APPROVED
  |
  ├── REJECTED
  |
  └── FLAGGED
```

Example:

```text
Customer submits review
        |
        ↓
PENDING_MODERATION
        |
        ↓
Moderation
        |
        ├── APPROVED
        |
        └── REJECTED
```

For trusted/low-risk reviews, the organization may choose automated approval,
but moderation rules should remain configurable.

---

# 7. Review Data Model

A review can contain:

```text
reviewId
productId
customerId
orderId
rating
title
body
status
verifiedPurchase
helpfulCount
reportCount
createdAt
updatedAt
moderatedAt
```

---

# 8. Repository Structure

```text
review-service/
│
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   │
│   ├── config/
│   │   └── configuration.ts
│   │
│   ├── database/
│   │   └── database.module.ts
│   │
│   ├── redis/
│   │   ├── redis.module.ts
│   │   └── redis.service.ts
│   │
│   ├── kafka/
│   │   ├── kafka.module.ts
│   │   └── kafka.service.ts
│   │
│   ├── review/
│   │   ├── review.controller.ts
│   │   ├── review.service.ts
│   │   ├── review.repository.ts
│   │   ├── review.entity.ts
│   │   └── dto/
│   │       ├── create-review.dto.ts
│   │       └── update-review.dto.ts
│   │
│   ├── moderation/
│   │   ├── moderation.service.ts
│   │   └── moderation.controller.ts
│   │
│   ├── rating/
│   │   └── rating.service.ts
│   │
│   ├── health/
│   │   └── health.controller.ts
│   │
│   └── observability/
│       └── metrics.service.ts
│
├── migrations/
│
├── test/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
├── .env.example
├── README.md
│
├── helm/
│   └── review-service/
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

# 9. Technology Stack

```text
TypeScript
    ↓
Node.js
    ↓
NestJS
    ↓
TypeORM
    ↓
PostgreSQL
```

Additional infrastructure:

```text
Redis  → caching
Kafka  → events
EKS    → runtime
ArgoCD → deployment
```

---

# 10. `package.json`

```json
{
  "name": "review-service",
  "version": "1.0.0",
  "scripts": {
    "build": "nest build",
    "start": "node dist/main.js",
    "start:dev": "nest start --watch",
    "test": "jest",
    "test:cov": "jest --coverage",
    "lint": "eslint \"src/**/*.ts\"",
    "format": "prettier --write \"src/**/*.ts\""
  },
  "dependencies": {
    "@nestjs/common": "^11",
    "@nestjs/core": "^11",
    "@nestjs/platform-express": "^11",
    "@nestjs/config": "^4",
    "@nestjs/typeorm": "^11",
    "typeorm": "^0.3",
    "pg": "^8",
    "ioredis": "^5",
    "kafkajs": "^2",
    "class-validator": "^0.14",
    "class-transformer": "^0.5"
  },
  "devDependencies": {
    "@nestjs/cli": "^11",
    "@nestjs/testing": "^11",
    "@types/node": "^22",
    "typescript": "^5",
    "jest": "^30",
    "ts-jest": "^29",
    "eslint": "^9",
    "prettier": "^3"
  }
}
```

Pin and test exact versions before production adoption.

---

# 11. Configuration

## `src/config/configuration.ts`

```typescript
export default () => ({
  port: parseInt(
    process.env.PORT ?? '8080',
    10,
  ),

  database: {
    url: process.env.DATABASE_URL,
  },

  redis: {
    url: process.env.REDIS_URL,
  },

  kafka: {
    brokers: (
      process.env.KAFKA_BROKERS ?? ''
    ).split(','),

    clientId:
      process.env.KAFKA_CLIENT_ID ??
      'review-service',

    groupId:
      process.env.KAFKA_GROUP_ID ??
      'review-service',
  },

  review: {
    cacheTtl: parseInt(
      process.env.REVIEW_CACHE_TTL ?? '300',
      10,
    ),
  },
});
```

---

# 12. Environment Variables

```text
PORT=8080

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/reviews

REDIS_URL=redis://localhost:6379

KAFKA_BROKERS=localhost:9092

KAFKA_CLIENT_ID=review-service

KAFKA_GROUP_ID=review-service

REVIEW_CACHE_TTL=300
```

Production database credentials must come from:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
Review Service
```

---

# 13. PostgreSQL Schema

```sql
CREATE TABLE reviews (

    id BIGSERIAL PRIMARY KEY,

    review_id UUID NOT NULL UNIQUE,

    product_id VARCHAR(100) NOT NULL,

    customer_id VARCHAR(100) NOT NULL,

    order_id VARCHAR(100),

    rating SMALLINT NOT NULL,

    title VARCHAR(200),

    body TEXT,

    status VARCHAR(50) NOT NULL,

    verified_purchase BOOLEAN NOT NULL DEFAULT FALSE,

    helpful_count INTEGER NOT NULL DEFAULT 0,

    report_count INTEGER NOT NULL DEFAULT 0,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    moderated_at TIMESTAMP
);

CREATE INDEX idx_reviews_product
    ON reviews(product_id);

CREATE INDEX idx_reviews_customer
    ON reviews(customer_id);

CREATE INDEX idx_reviews_product_status
    ON reviews(product_id, status);
```

---

# 14. Preventing Duplicate Reviews

A business rule might be:

```text
One customer can review a product once per completed order.
```

The database should enforce the rule where possible.

For example:

```sql
CREATE UNIQUE INDEX ux_review_customer_order_product
ON reviews(customer_id, order_id, product_id);
```

This is much safer than relying only on application code.

Why?

Two requests can arrive simultaneously:

```text
Request A → check → no review
Request B → check → no review

A → insert
B → insert
```

Application-level checking alone can race.

A database constraint provides the final protection.

---

# 15. Rating Validation

Allowed ratings:

```text
1
2
3
4
5
```

Never trust the client.

DTO validation:

```typescript
import {
  IsInt,
  IsString,
  Max,
  Min,
  MaxLength,
} from 'class-validator';

export class CreateReviewDto {

  @IsString()
  productId: string;

  @IsInt()
  @Min(1)
  @Max(5)
  rating: number;

  @IsString()
  @MaxLength(200)
  title: string;

  @IsString()
  @MaxLength(5000)
  body: string;
}
```

The authenticated customer ID should come from the security context rather than
from a freely supplied request field.

---

# 16. Review Entity

```typescript
import {
  Column,
  Entity,
  PrimaryGeneratedColumn,
} from 'typeorm';

@Entity('reviews')
export class Review {

  @PrimaryGeneratedColumn()
  id: number;

  @Column({
    name: 'review_id',
    unique: true,
  })
  reviewId: string;

  @Column({
    name: 'product_id',
  })
  productId: string;

  @Column({
    name: 'customer_id',
  })
  customerId: string;

  @Column({
    name: 'order_id',
    nullable: true,
  })
  orderId?: string;

  @Column()
  rating: number;

  @Column({
    nullable: true,
  })
  title?: string;

  @Column({
    type: 'text',
    nullable: true,
  })
  body?: string;

  @Column()
  status: string;

  @Column({
    name: 'verified_purchase',
    default: false,
  })
  verifiedPurchase: boolean;

  @Column({
    name: 'helpful_count',
    default: 0,
  })
  helpfulCount: number;

  @Column({
    name: 'report_count',
    default: 0,
  })
  reportCount: number;
}
```

---

# 17. Review Status

Recommended values:

```text
PENDING_MODERATION
APPROVED
REJECTED
FLAGGED
DELETED
```

Do not expose internal moderation states unnecessarily to customers.

---

# 18. Create Review API

```http
POST /api/v1/reviews
```

Request:

```json
{
  "productId": "PROD-1001",
  "rating": 5,
  "title": "Excellent product",
  "body": "Works very well and battery life is good."
}
```

The authenticated identity determines:

```text
customerId
```

The service then checks whether the customer is eligible to review.

---

# 19. Review Creation Flow

```text
Customer
   |
   ↓
POST /reviews
   |
   ↓
Authentication
   |
   ↓
Validate request
   |
   ↓
Check purchase eligibility
   |
   ↓
Check duplicate
   |
   ↓
Create review
   |
   ↓
PENDING_MODERATION
   |
   ↓
Publish ReviewSubmitted
```

---

# 20. Purchase Verification

Recommended event:

```json
{
  "eventType": "ORDER_COMPLETED",
  "version": 1,
  "aggregateId": "ORD-1001",
  "payload": {
    "customerId": "CUS-1001",
    "items": [
      {
        "productId": "PROD-1001",
        "quantity": 1
      }
    ]
  }
}
```

Review Service consumes this event.

It can maintain a compact eligibility record.

---

# 21. Review Eligibility

Example table:

```sql
CREATE TABLE review_eligibility (

    customer_id VARCHAR(100) NOT NULL,

    product_id VARCHAR(100) NOT NULL,

    order_id VARCHAR(100) NOT NULL,

    eligible BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (
        customer_id,
        product_id,
        order_id
    )
);
```

This avoids calling Order Service synchronously for every review request.

---

# 22. Kafka Topics

Examples:

```text
ecommerce.order.completed
ecommerce.review.submitted
ecommerce.review.approved
ecommerce.review.rejected
ecommerce.review.updated
```

Topic ownership and naming should follow the organization's event governance
rules.

---

# 23. ReviewSubmitted Event

```json
{
  "eventId": "evt-review-1001",
  "eventType": "REVIEW_SUBMITTED",
  "version": 1,
  "occurredAt": "2026-10-05T12:30:00Z",
  "source": "review-service",
  "correlationId": "corr-review-1001",
  "aggregateId": "REV-1001",
  "payload": {
    "reviewId": "REV-1001",
    "productId": "PROD-1001",
    "customerId": "CUS-1001",
    "rating": 5,
    "verifiedPurchase": true
  }
}
```

Do not publish sensitive review body data unless downstream consumers actually
need it.

---

# 24. Moderation Architecture

```text
Review Submitted
       |
       ↓
Moderation
       |
       ├── Automated checks
       |
       ├── Abuse detection
       |
       ├── Spam detection
       |
       └── Human review
```

Automated moderation may detect:

```text
spam
duplicate content
abusive language
malicious links
promotional content
```

---

# 25. Important Security Principle

Never assume:

```text
"Review contains harmless text."
```

Review content is user-controlled input.

It can contain:

```text
HTML
JavaScript
URLs
malicious payloads
personal information
spam
```

Therefore:

```text
validate
sanitize
encode
moderate
```

before displaying review content.

---

# 26. XSS Protection

The backend should return structured data.

The frontend should safely render:

```text
title
body
```

Do not inject review body directly into HTML.

For example, never do:

```javascript
element.innerHTML = review.body;
```

without appropriate sanitization.

---

# 27. Review API

```text
POST   /api/v1/reviews
GET    /api/v1/products/{productId}/reviews
GET    /api/v1/reviews/{reviewId}
PUT    /api/v1/reviews/{reviewId}
DELETE /api/v1/reviews/{reviewId}
POST   /api/v1/reviews/{reviewId}/helpful
POST   /api/v1/reviews/{reviewId}/report
```

Administrative moderation APIs should be protected separately.

---

# 28. Get Product Reviews

```http
GET /api/v1/products/PROD-1001/reviews?page=0&size=20
```

Response:

```json
{
  "productId": "PROD-1001",
  "page": 0,
  "size": 20,
  "total": 1532,
  "reviews": [
    {
      "reviewId": "REV-1001",
      "rating": 5,
      "title": "Excellent product",
      "body": "Works very well.",
      "verifiedPurchase": true,
      "helpfulCount": 25
    }
  ]
}
```

---

# 29. Rating Summary

Endpoint:

```http
GET /api/v1/products/{productId}/rating-summary
```

Response:

```json
{
  "productId": "PROD-1001",
  "averageRating": 4.5,
  "totalReviews": 1532,
  "distribution": {
    "5": 900,
    "4": 450,
    "3": 100,
    "2": 50,
    "1": 32
  }
}
```

---

# 30. Rating Calculation

Do not calculate the average by reading every review on every request.

Bad:

```text
Request
 ↓
SELECT all reviews
 ↓
calculate average
```

Better:

```text
Review mutation
     ↓
update aggregate
     ↓
Redis cache
     ↓
fast read
```

---

# 31. Rating Aggregate Table

```sql
CREATE TABLE product_rating_summary (

    product_id VARCHAR(100) PRIMARY KEY,

    total_reviews BIGINT NOT NULL DEFAULT 0,

    rating_sum BIGINT NOT NULL DEFAULT 0,

    rating_1 BIGINT NOT NULL DEFAULT 0,

    rating_2 BIGINT NOT NULL DEFAULT 0,

    rating_3 BIGINT NOT NULL DEFAULT 0,

    rating_4 BIGINT NOT NULL DEFAULT 0,

    rating_5 BIGINT NOT NULL DEFAULT 0,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

Average:

```text
average = rating_sum / total_reviews
```

Use appropriate numeric precision in the actual implementation.

---

# 32. Rating Update

When a new approved review arrives:

```text
rating_sum += rating
total_reviews += 1
rating_N += 1
```

When a review changes:

```text
remove old rating
add new rating
```

This must be handled transactionally.

---

# 33. Why Only Approved Reviews Count

If a review is:

```text
PENDING_MODERATION
```

it should generally not affect the public rating.

Only:

```text
APPROVED
```

reviews contribute to the customer-visible rating.

---

# 34. Redis Cache

Examples:

```text
review:product:PROD-1001:summary
review:product:PROD-1001:page:0:size:20
review:review:REV-1001
```

TTL:

```text
300 seconds
```

is illustrative.

---

# 35. Cache Invalidation

When an approved review is created:

```text
Review created
    |
    ↓
Rating summary changed
    |
    ↓
Delete:
review:product:PROD-1001:summary
```

The next request regenerates it.

---

# 36. Cache Stampede

For extremely popular products:

```text
Cache expires
      |
      ↓
50,000 requests
      |
      ↓
50,000 DB queries
```

Protection:

```text
request coalescing
short TTL jitter
background refresh
bounded DB concurrency
```

---

# 37. Helpful Votes

Endpoint:

```http
POST /api/v1/reviews/{reviewId}/helpful
```

Do not allow the same customer to repeatedly increase the count.

Use:

```text
customerId + reviewId
```

with a unique constraint.

Example:

```sql
CREATE TABLE review_helpful_votes (

    review_id UUID NOT NULL,

    customer_id VARCHAR(100) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (
        review_id,
        customer_id
    )
);
```

---

# 38. Review Reporting

Customers can report reviews:

```http
POST /api/v1/reviews/{reviewId}/report
```

Reasons:

```text
SPAM
ABUSE
FAKE_REVIEW
OFFENSIVE
PERSONAL_INFORMATION
OTHER
```

Do not immediately delete based solely on one report.

Create a moderation signal.

---

# 39. Anti-Abuse Controls

Potential controls:

```text
authentication
verified purchase
one review per eligible order
rate limiting
duplicate detection
report thresholds
spam detection
moderation
```

Also monitor:

```text
many reviews from one account
many accounts from one IP
rapid review creation
identical review bodies
suspicious rating patterns
```

Avoid treating IP address alone as proof of abuse.

---

# 40. Rate Limiting

Example:

```text
Create Review:
10 requests / minute / customer
```

These numbers are illustrative.

Production limits should be based on:

```text
business rules
abuse analysis
traffic
customer behavior
```

Rate limiting can be implemented at:

```text
WAF
API Gateway/ALB layer
application
Redis
```

depending on architecture.

---

# 41. Review Database Connection Pool

NestJS/TypeORM uses PostgreSQL connections.

The same capacity principle applies:

```text
Pods × connections per Pod
```

Example:

```text
10 Pods × 10 connections
=
100 potential connections
```

This must be included in the global RDS connection budget.

Do not give every microservice a large pool.

---

# 42. Redis

Node.js uses:

```text
ioredis
```

Example:

```typescript
import Redis from 'ioredis';

const redis = new Redis(
  process.env.REDIS_URL!,
  {
    connectTimeout: 2000,
    maxRetriesPerRequest: 2,
  },
);
```

Exact retry settings should be tuned to prevent retry storms.

---

# 43. Kafka

Example:

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'review-service',
  brokers: (
    process.env.KAFKA_BROKERS ?? ''
  ).split(','),
});

const consumer = kafka.consumer({
  groupId: 'review-service',
});
```

---

# 44. Consumer Idempotency

Kafka delivery can be at-least-once.

Therefore:

```text
event
 ↓
process
 ↓
commit
```

may result in duplicate processing after a crash.

Use:

```text
eventId
```

and an idempotency table or equivalent mechanism.

---

# 45. Dockerfile

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

RUN addgroup \
      -S appgroup \
    && adduser \
      -S appuser \
      -G appgroup \
      -u 10001

USER 10001

EXPOSE 8080

CMD ["node", "dist/main.js"]
```

---

# 46. `.dockerignore`

```text
.git
.github

node_modules
dist

coverage

.env
.env.*

*.log

README.md
```

---

# 47. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: review-postgres

    environment:

      POSTGRES_DB: reviews

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: review-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: review-kafka

    ports:

      - "9092:9092"


  review-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql://postgres:postgres@postgres:5432/reviews

      REDIS_URL:
        redis://redis:6379

      KAFKA_BROKERS:
        kafka:9092

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

  name: review-service

  labels:

    app: review-service

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: review-service

  template:

    metadata:

      labels:

        app: review-service

    spec:

      serviceAccountName: review-service

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: review-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: review-service-config

            - secretRef:

                name: review-service-secrets

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

# 49. Kubernetes Service

```yaml
apiVersion: v1
kind: Service

metadata:

  name: review-service

  labels:

    app: review-service

spec:

  type: ClusterIP

  selector:

    app: review-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 50. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:

  name: review-service-config

data:

  REDIS_URL:
    redis://redis.ecommerce.svc.cluster.local:6379

  KAFKA_BROKERS:
    kafka.ecommerce.svc.cluster.local:9092

  KAFKA_CLIENT_ID:
    review-service

  KAFKA_GROUP_ID:
    review-service

  REVIEW_CACHE_TTL:
    "300"
```

---

# 51. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:

  name: review-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: review-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/review-service

        property: DATABASE_URL
```

---

# 52. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:

  name: review-service

  labels:

    app: review-service
```

If the service needs AWS APIs:

```text
Review Pod
    ↓
EKS Pod Identity
    ↓
IAM Role
    ↓
Specific AWS resource
```

Do not attach unnecessary permissions.

---

# 53. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:

  name: review-service

spec:

  podSelector:

    matchLabels:

      app: review-service

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

# 54. HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:

  name: review-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: review-service

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

Review APIs are usually more read-heavy than write-heavy, so API scaling and
database capacity should be planned independently.

---

# 55. PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:

  name: review-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: review-service
```

---

# 56. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:

  name: review-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: review-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 57. Review Metrics

Application:

```text
review_requests_total

review_request_duration_seconds

review_errors_total

review_creation_total

review_update_total

review_delete_total
```

Moderation:

```text
review_moderation_pending_total

review_moderation_approved_total

review_moderation_rejected_total

review_moderation_latency_seconds
```

Events:

```text
review_events_processed_total

review_events_failed_total

review_consumer_lag
```

Cache:

```text
review_cache_hits_total

review_cache_misses_total
```

Business:

```text
review_submission_success_total

review_submission_rejected_total

review_report_total

review_helpful_total
```

---

# 58. Grafana Dashboard

```text
Review Service
│
├── API
│   ├── RPS
│   ├── P95
│   ├── P99
│   └── 5xx
│
├── Reviews
│   ├── Submitted
│   ├── Approved
│   ├── Rejected
│   └── Reported
│
├── Moderation
│   ├── Pending
│   └── Processing Time
│
├── Kafka
│   ├── Lag
│   └── Processing Rate
│
├── Redis
│   └── Hit Ratio
│
└── PostgreSQL
    ├── Connections
    ├── Query Latency
    └── DB Load
```

---

# 59. ELK Logging

Example:

```json
{
  "@timestamp": "2026-10-05T14:00:00Z",
  "level": "INFO",
  "service": "review-service",
  "correlationId": "corr-1001",
  "reviewId": "REV-1001",
  "productId": "PROD-1001",
  "event": "REVIEW_SUBMITTED",
  "verifiedPurchase": true,
  "status": "PENDING_MODERATION"
}
```

Avoid logging full review bodies by default.

A review can contain personal or sensitive information.

---

# 60. OpenTelemetry

Example trace:

```text
POST /reviews
       |
       ├── authentication
       |
       ├── eligibility lookup
       |
       ├── PostgreSQL INSERT
       |
       └── Kafka publish
```

For reading:

```text
GET /products/PROD-1001/reviews
       |
       ├── Redis GET
       |
       └── PostgreSQL SELECT
```

Jaeger can reveal which dependency contributes to latency.

---

# 61. SLO

Example:

```text
Review read availability:
99.95%

Review write availability:
99.9%

GET review P95:
< 150ms

POST review P95:
< 300ms

Rating summary P95:
< 100ms

Moderation processing freshness:
99% within agreed target
```

Actual values must come from load testing and business requirements.

---

# 62. Important SLI

For review reads:

```text
successful review requests
--------------------------
eligible review requests
```

For review writes:

```text
successfully accepted valid review requests
-------------------------------------------
valid review requests
```

Do not count expected business rejections such as:

```text
duplicate review
unauthorized customer
invalid rating
```

as infrastructure availability failures unless the SLO explicitly defines
them that way.

---

# 63. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Review Service CI

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

  ECR_REPOSITORY: review-service

jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4

      - name: Setup Node

        uses: actions/setup-node@v4

        with:

          node-version: "22"

          cache: npm

      - name: Install

        run: npm ci

      - name: Lint

        run: npm run lint

      - name: Test

        run: npm test -- --runInBand

      - name: Build

        run: npm run build

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

      - name: ECR login

        id: ecr

        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image

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

      - name: Push image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

---

# 64. SonarQube

SonarQube should analyze:

```text
TypeScript
NestJS
tests
complexity
security issues
duplication
coverage
```

Review-specific tests should additionally cover:

```text
duplicate review
authorization
rating validation
purchase verification
moderation state transitions
rating aggregation
```

---

# 65. ECR

Repository:

```text
company-ecommerce/review-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/review-service:<git-sha>
```

Use immutable image tags.

---

# 66. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: review-service

description: Ecommerce review service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 67. Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/review-service

  tag: ""

  pullPolicy: IfNotPresent

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

serviceAccount:

  create: true

  name: review-service
```

---

# 68. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── review-service/
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
Review
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

---

# 69. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: review-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/review-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 70. Failure Scenario — PostgreSQL Unavailable

```text
Review API
    |
    X
PostgreSQL
```

Reads may temporarily use cached data where appropriate.

Writes should fail safely rather than pretending a review was successfully
stored.

Do not acknowledge a review submission before durable persistence succeeds.

---

# 71. Failure Scenario — Redis Unavailable

Redis should be treated as an optimization layer.

```text
Redis unavailable
      |
      ↓
PostgreSQL
```

However, protect PostgreSQL from a cache-miss storm.

Use:

```text
bounded concurrency
request coalescing
short Redis timeouts
```

---

# 72. Failure Scenario — Kafka Unavailable

The review itself can still be persisted.

But event publication becomes a consistency problem.

Bad:

```text
DB insert
 ↓
Kafka publish
 ↓
Kafka fails
```

If the transaction commits before the event is safely published, downstream
systems may never learn about the review.

Use the transactional outbox pattern.

---

# 73. Transactional Outbox

```text
PostgreSQL Transaction
        |
        ├── Review INSERT
        |
        └── Outbox INSERT
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

This provides reliable event publication.

---

# 74. Outbox Table

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

# 75. Failure Scenario — Duplicate Review

Customer clicks:

```text
Submit
```

twice.

Requests:

```text
A
B
```

arrive simultaneously.

Database unique constraint prevents duplicate persistence.

Application returns a controlled response.

---

# 76. Failure Scenario — Review Abuse

Suppose an attacker creates:

```text
10,000 fake reviews
```

Monitor:

```text
review creation rate
customer-level rate
IP/network patterns
verified purchase ratio
duplicate content
report rate
```

Mitigation can include:

```text
rate limiting
WAF rules
account controls
moderation
temporary review restrictions
```

---

# 77. Failure Scenario — Rating Corruption

Suppose:

```text
Average rating:
4.5
```

suddenly becomes:

```text
2.1
```

but review volume has not changed.

Investigate:

```text
rating aggregate
review status changes
migration
application deployment
database transaction
cache
```

The source reviews should remain available for reconciliation.

---

# 78. Rating Reconciliation

Periodically compare:

```text
Calculated from approved reviews
```

against:

```text
Stored product_rating_summary
```

If mismatch:

```text
detect
→ investigate
→ rebuild aggregate
→ invalidate cache
```

This provides an important data-integrity control.

---

# 79. Example Production Incident

## Incident ID

```text
INC-2026-REVIEW-001
```

## Severity

```text
SEV-2
```

## Title

```text
Duplicate Review Race Condition Created Incorrect Review Counts
```

## Date

```text
2026-10-05
```

---

# 80. Customer Impact

A small number of customers submitted reviews using rapid repeated requests.

Due to an application race condition:

```text
duplicate review records
```

were created for some product/customer combinations.

Customer-facing rating totals became temporarily incorrect.

No payment or order data was affected.

---

# 81. Timeline

```text
14:00
Normal review traffic.

14:08
Traffic spike for a popular product.

14:10
Duplicate review requests increase.

14:12
Review count anomaly detected.

14:14
SRE investigates PostgreSQL records.

14:17
Duplicate rows identified.

14:20
SEV-2 declared.

14:22
Review creation temporarily rate-limited.

14:25
Application team identifies missing database uniqueness constraint.

14:30
Duplicate creation stopped.

14:35
Invalid duplicate records identified.

14:40
Rating summaries reconciled.

14:45
Corrected deployment released.

14:50
Normal traffic restored.

15:00
Incident resolved.
```

---

# 82. Root Cause

The application implemented:

```text
SELECT existing review
        ↓
if none
        ↓
INSERT review
```

without a database uniqueness constraint.

Two concurrent requests could both pass the existence check.

```text
Request A → SELECT → none
Request B → SELECT → none

Request A → INSERT
Request B → INSERT
```

---

# 83. Amplification

The problem was amplified by:

```text
high request concurrency
+
popular product
+
client retries
+
lack of database constraint
```

The application had no atomic protection against the race.

---

# 84. What Went Well

- Review service was isolated from order/payment systems.
- PostgreSQL data allowed reconciliation.
- Prometheus exposed unusual review creation rates.
- GitOps enabled controlled deployment.
- Rating aggregates could be rebuilt.

---

# 85. What Went Wrong

- Duplicate prevention existed only in application logic.
- No database unique constraint.
- Concurrency tests were insufficient.
- Alerting detected the symptom rather than preventing the condition.

---

# 86. Five Whys

### Why were duplicate reviews created?

Concurrent requests inserted the same logical review.

### Why?

Both requests passed the existence check.

### Why?

The check and insert were not protected by a unique constraint.

### Why was there no constraint?

The design relied on application-level duplicate detection.

### Why?

Concurrency behavior was not included sufficiently in the database design and
test plan.

---

# 87. Corrective Actions

| Action | Owner |
|---|---|
| Add unique database constraint | Backend |
| Add concurrent request tests | QA |
| Add idempotency support | Backend |
| Add customer-level rate limiting | Platform |
| Add duplicate review alert | SRE |
| Reconcile rating aggregates | Data |
| Review other uniqueness assumptions | Architecture |
| Add database constraints to design review | Architecture |

---

# 88. Review Service Runbook

```text
1. Confirm customer impact.
2. Check API error rate.
3. Check review creation rate.
4. Check PostgreSQL connections.
5. Check PostgreSQL locks/queries.
6. Check Redis availability.
7. Check Kafka lag.
8. Check recent deployment.
9. Check moderation queue.
10. Check rating aggregate anomalies.
11. Apply safe mitigation.
12. Protect database.
13. Validate review integrity.
14. Reconcile aggregates if required.
15. Roll back bad deployment if applicable.
16. Verify business behavior.
17. Document RCA.
```

---

# 89. Review Security Checklist

```text
[ ] Authentication required
[ ] Authorization enforced
[ ] Customer identity from trusted auth context
[ ] Product ownership/validity checked
[ ] Purchase eligibility checked
[ ] Input validation
[ ] Review length limits
[ ] Safe rendering
[ ] XSS protection
[ ] Rate limiting
[ ] Abuse detection
[ ] Database constraints
[ ] Secrets Manager
[ ] External Secrets
[ ] Pod Identity
[ ] Least privilege
[ ] NetworkPolicy
[ ] TLS
[ ] Trivy
[ ] SonarQube
[ ] Audit logging
```

---

# 90. Performance Checklist

```text
[ ] Product review read load test
[ ] Review creation load test
[ ] Rating summary load test
[ ] Redis cache test
[ ] Redis failure test
[ ] PostgreSQL failure test
[ ] Kafka failure test
[ ] Kafka lag test
[ ] Concurrent review submission test
[ ] Duplicate request test
[ ] Moderation backlog test
[ ] Rating reconciliation test
[ ] HPA test
[ ] RDS connection budget test
```

---

# 91. Final Architecture

```text
                              Customer
                                 |
                                 ↓
                           CloudFront/WAF
                                 |
                                 ↓
                                ALB
                                 |
                                 ↓
                         Review Service
                          /     |      \
                         /      |       \
                        ↓       ↓        ↓
                  PostgreSQL  Redis     Kafka
                     |                    |
                     |                    ↓
                     |              Other Services
                     |
             ┌───────┴────────┐
             ↓                ↓
       Review Data       Outbox Events
             |
             ↓
      Rating Aggregates
```

---

# 92. Final Production Principles

1. Review Service owns review data.
2. Product Service owns product data.
3. Order Service remains authoritative for orders.
4. Payment Service remains authoritative for payments.
5. Customer identity must come from trusted authentication.
6. Never trust customer IDs supplied only by the request body.
7. Verified-purchase status should be based on authoritative order data.
8. Kafka can distribute order-completion events to Review Service.
9. Review creation should be idempotent.
10. Database constraints are required for concurrency-sensitive business rules.
11. Application-level duplicate checks are not sufficient by themselves.
12. PostgreSQL is the source of truth for reviews.
13. Redis is a performance optimization.
14. Redis failure must not cause permanent review loss.
15. Kafka failure must not cause review persistence to fail unnecessarily.
16. Transactional outbox protects database-to-event consistency.
17. Rating summaries should be maintained efficiently.
18. Rating aggregates should periodically be reconciled against source reviews.
19. Only approved reviews should normally affect public ratings.
20. Review content is untrusted user input.
21. Review content must be safely rendered.
22. Rate limiting and abuse detection are mandatory for public review APIs.
23. Moderation should be independently scalable.
24. HPA must be planned with PostgreSQL connection capacity.
25. Cache stampede protection is required for popular products.
26. Review availability should not be dependent on recommendation/search.
27. Review events should use versioned contracts.
28. OpenTelemetry and Jaeger should trace slow review operations.
29. Prometheus should monitor technical and business metrics.
30. Every major review integrity incident should have an RCA.
31. Data integrity is more important than blindly maximizing write throughput.
32. Production deployment remains:
    `CI → ECR → GitOps → ArgoCD → EKS`.
33. Phase 20 will extract repeated review-service patterns into reusable
    platform templates.
34. The service should remain independently deployable and independently
    scalable.

# End of 178 — Review Service
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
178 Review Service           ✅

179 Tax Service              ⏳
180 Invoice Service          ⏳
181 Reporting Service        ⏳
```

**Next: `179-Tax-Service.md`** — tax calculation, jurisdiction handling, tax rules/versioning, external tax-provider integration, caching, PostgreSQL, Redis, Kafka, failure/fallback strategy, Kubernetes/EKS, CI/CD, GitOps, observability, SLOs and a production RCA.