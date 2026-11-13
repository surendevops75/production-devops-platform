# 174 — Delivery Service

**File:** `174-Delivery-Service.md`

```markdown
# 174 — Delivery Service

## 1. Overview

The `delivery-service` manages the delivery lifecycle after a shipment has been
created by the Shipping Service.

Its responsibility is to track the physical delivery process and maintain the
customer-facing delivery state.

The service handles:

- Delivery creation
- Delivery assignment
- Delivery status
- Delivery attempts
- Out-for-delivery state
- Failed delivery
- Rescheduling
- Proof of delivery metadata
- Delivery exceptions
- Carrier/delivery-partner events
- Delivery notifications/events
- Tracking updates
- Idempotent event processing
- Delivery reconciliation

It does not own:

- Order creation
- Payment
- Inventory
- Product catalog
- Shipping-rate calculation
- Carrier shipment creation

Technology:

- Go 1.24+
- Gin
- PostgreSQL
- Redis
- Kafka
- SQS
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

Go is used here because delivery tracking is a good fit for a lightweight,
high-concurrency service that consumes a large number of asynchronous events.

---

# 2. Shipping vs Delivery

These two services have different responsibilities.

## Shipping Service

```text
Carrier selection
Shipping rates
Shipment creation
Tracking number
Carrier API integration
Shipping label
```

## Delivery Service

```text
Delivery lifecycle
Delivery attempts
Out-for-delivery
Delivered
Failed delivery
Reschedule
Proof of delivery
Delivery exceptions
```

Architecture:

```text
Order
  |
  ↓
Shipping Service
  |
  ↓
Shipment
  |
  ↓
Delivery Service
  |
  ├── Delivery Partner
  ├── Kafka
  ├── SQS
  ├── PostgreSQL
  └── Redis
```

---

# 3. Delivery Lifecycle

Typical lifecycle:

```text
CREATED
   |
   ↓
ASSIGNED
   |
   ↓
OUT_FOR_DELIVERY
   |
   ↓
DELIVERY_ATTEMPTED
   |
   ├── FAILED
   |     |
   |     ↓
   |   RESCHEDULED
   |     |
   |     └──────────────→ OUT_FOR_DELIVERY
   |
   └── SUCCESS
         |
         ↓
      DELIVERED
```

Other terminal states:

```text
CANCELLED
RETURN_TO_ORIGIN
```

---

# 4. Delivery State Machine

Allowed transitions should be explicit.

```text
CREATED
  ↓
ASSIGNED
  ↓
OUT_FOR_DELIVERY
  ↓
DELIVERY_ATTEMPTED
  ├── DELIVERED
  ├── FAILED
  └── RESCHEDULED
          ↓
     OUT_FOR_DELIVERY
```

Invalid transition example:

```text
DELIVERED
   ↓
OUT_FOR_DELIVERY
```

The application should reject it.

---

# 5. High-Level Architecture

```text
                       Shipping Service
                              |
                              ↓
                       ShipmentCreated
                              |
                              ↓
                       Delivery Service
                     /       |       \
                    ↓        ↓        ↓
              PostgreSQL   Redis    Kafka
                    |                 |
                    |                 ↓
                    |          Notification
                    |
                    ↓
             Delivery State
                    |
                    ↓
            Delivery Partner
                    |
                    ↓
              Status Events
                    |
                    ↓
             Delivery Service
```

---

# 6. Why Delivery Is Event-Driven

Delivery updates are often asynchronous.

Example:

```text
Delivery Partner
       |
       ↓
OUT_FOR_DELIVERY
       |
       ↓
Webhook/Event
       |
       ↓
Delivery Service
       |
       ↓
Kafka
       |
       ├── Notification Service
       ├── Reporting Service
       └── Customer-facing systems
```

The customer should not need to synchronously call the delivery partner every
time the tracking page is opened.

---

# 7. Repository Structure

```text
delivery-service/
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   ├── config/
│   │   └── config.go
│   │
│   ├── database/
│   │   └── postgres.go
│   │
│   ├── redis/
│   │   └── redis.go
│   │
│   ├── delivery/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── model.go
│   │   └── state.go
│   │
│   ├── events/
│   │   ├── kafka.go
│   │   └── consumer.go
│   │
│   ├── queue/
│   │   └── sqs.go
│   │
│   ├── webhook/
│   │   └── handler.go
│   │
│   └── observability/
│       └── metrics.go
│
├── migrations/
│
├── tests/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── go.mod
├── go.sum
├── Makefile
├── README.md
│
├── helm/
│   └── delivery-service/
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

# 8. Go Module

## `go.mod`

```go
module github.com/company-ecommerce/delivery-service

go 1.24

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/google/uuid v1.6.0
    github.com/jackc/pgx/v5 v5.7.0
    github.com/redis/go-redis/v9 v9.7.0
    github.com/prometheus/client_golang v1.20.5
)
```

Exact versions should be pinned to the versions tested by the project.

---

# 9. Configuration

## `internal/config/config.go`

```go
package config

import (
	"os"
	"strconv"
)

type Config struct {
	Port              string
	DatabaseURL       string
	RedisAddr         string
	KafkaBrokers      string
	DeliveryCacheTTL  int
}

func Load() Config {

	cacheTTL, _ := strconv.Atoi(
		getEnv("DELIVERY_CACHE_TTL_SECONDS", "60"),
	)

	return Config{
		Port: getEnv(
			"PORT",
			"8080",
		),

		DatabaseURL: getEnv(
			"DATABASE_URL",
			"",
		),

		RedisAddr: getEnv(
			"REDIS_ADDR",
			"localhost:6379",
		),

		KafkaBrokers: getEnv(
			"KAFKA_BROKERS",
			"localhost:9092",
		),

		DeliveryCacheTTL: cacheTTL,
	}
}

func getEnv(
	key string,
	defaultValue string,
) string {

	value := os.Getenv(key)

	if value == "" {
		return defaultValue
	}

	return value
}
```

---

# 10. Environment Variables

```text
PORT=8080

DATABASE_URL=postgres://postgres:postgres@localhost:5432/delivery

REDIS_ADDR=localhost:6379

KAFKA_BROKERS=localhost:9092

DELIVERY_CACHE_TTL_SECONDS=60
```

Production credentials must come from:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
Delivery Service
```

---

# 11. Delivery Model

## `internal/delivery/model.go`

```go
package delivery

import "time"

type Delivery struct {
	ID                  int64
	DeliveryReference   string
	ShipmentReference   string
	OrderID             string
	CustomerID          string
	Status              string
	AttemptNumber       int
	DeliveryPartner     string
	TrackingNumber      string
	ScheduledDate       *time.Time
	DeliveredAt         *time.Time
	FailureReason       string
	ProofOfDeliveryURL  string
	CreatedAt           time.Time
	UpdatedAt           time.Time
}
```

---

# 12. Delivery Status Constants

## `internal/delivery/state.go`

```go
package delivery

const (
	StatusCreated          = "CREATED"
	StatusAssigned         = "ASSIGNED"
	StatusOutForDelivery   = "OUT_FOR_DELIVERY"
	StatusAttempted        = "DELIVERY_ATTEMPTED"
	StatusDelivered        = "DELIVERED"
	StatusFailed           = "FAILED"
	StatusRescheduled      = "RESCHEDULED"
	StatusCancelled        = "CANCELLED"
	StatusReturnToOrigin   = "RETURN_TO_ORIGIN"
)
```

---

# 13. State Transition Validation

```go
package delivery

func IsValidTransition(
	current string,
	next string,
) bool {

	switch current {

	case StatusCreated:

		return next == StatusAssigned ||
			next == StatusCancelled

	case StatusAssigned:

		return next == StatusOutForDelivery ||
			next == StatusCancelled

	case StatusOutForDelivery:

		return next == StatusAttempted ||
			next == StatusCancelled

	case StatusAttempted:

		return next == StatusDelivered ||
			next == StatusFailed ||
			next == StatusRescheduled

	case StatusFailed:

		return next == StatusRescheduled ||
			next == StatusReturnToOrigin

	case StatusRescheduled:

		return next == StatusOutForDelivery

	case StatusDelivered:

		return false

	case StatusCancelled:

		return false

	case StatusReturnToOrigin:

		return false

	default:

		return false
	}
}
```

This protects the delivery state machine from invalid updates.

---

# 14. PostgreSQL Schema

## `V1__create_deliveries.sql`

```sql
CREATE TABLE deliveries (

    id BIGSERIAL PRIMARY KEY,

    delivery_reference VARCHAR(100)
        NOT NULL UNIQUE,

    shipment_reference VARCHAR(100)
        NOT NULL,

    order_id VARCHAR(100)
        NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    status VARCHAR(50)
        NOT NULL,

    attempt_number INTEGER
        NOT NULL DEFAULT 0,

    delivery_partner VARCHAR(100),

    tracking_number VARCHAR(200),

    scheduled_date DATE,

    delivered_at TIMESTAMP,

    failure_reason TEXT,

    proof_of_delivery_url TEXT,

    created_at TIMESTAMP
        NOT NULL,

    updated_at TIMESTAMP
        NOT NULL
);

CREATE INDEX idx_deliveries_order_id
    ON deliveries(order_id);

CREATE INDEX idx_deliveries_customer_id
    ON deliveries(customer_id);

CREATE INDEX idx_deliveries_tracking
    ON deliveries(tracking_number);

CREATE INDEX idx_deliveries_status
    ON deliveries(status);
```

---

# 15. Delivery Events Table

```sql
CREATE TABLE delivery_events (

    id BIGSERIAL PRIMARY KEY,

    provider_event_id VARCHAR(200)
        NOT NULL,

    delivery_reference VARCHAR(100)
        NOT NULL,

    event_type VARCHAR(100)
        NOT NULL,

    event_payload JSONB
        NOT NULL,

    received_at TIMESTAMP
        NOT NULL,

    processed_at TIMESTAMP,

    status VARCHAR(50)
        NOT NULL,

    CONSTRAINT uq_delivery_provider_event
        UNIQUE(provider_event_id)
);
```

This provides webhook/event idempotency.

---

# 16. Delivery Attempt

Each delivery attempt should be represented clearly.

Example:

```text
Delivery
  |
  ├── Attempt 1
  |      └── FAILED
  |
  ├── Attempt 2
  |      └── FAILED
  |
  └── Attempt 3
         └── DELIVERED
```

Do not simply overwrite history.

A separate attempt table can be introduced when detailed audit requirements
exist.

---

# 17. Delivery Attempt Table

```sql
CREATE TABLE delivery_attempts (

    id BIGSERIAL PRIMARY KEY,

    delivery_reference VARCHAR(100)
        NOT NULL,

    attempt_number INTEGER
        NOT NULL,

    status VARCHAR(50)
        NOT NULL,

    failure_reason VARCHAR(500),

    attempted_at TIMESTAMP
        NOT NULL,

    delivered_at TIMESTAMP
);

CREATE UNIQUE INDEX uq_delivery_attempt
    ON delivery_attempts(
        delivery_reference,
        attempt_number
    );
```

---

# 18. Delivery API

Primary APIs:

```text
POST /api/v1/deliveries
GET  /api/v1/deliveries/{deliveryReference}
GET  /api/v1/deliveries/{deliveryReference}/tracking
POST /api/v1/deliveries/{deliveryReference}/reschedule
POST /api/v1/deliveries/{deliveryReference}/cancel
POST /api/v1/deliveries/webhooks/{provider}
```

---

# 19. Create Delivery

```http
POST /api/v1/deliveries
Idempotency-Key: SHIP-1001-DELIVERY-1
```

Request:

```json
{
  "shipmentReference": "SHIP-1001",
  "orderId": "ORD-1001",
  "customerId": "CUS-1001",
  "trackingNumber": "TRK-1001",
  "deliveryPartner": "partner-a"
}
```

Response:

```json
{
  "deliveryReference": "DEL-1001",
  "status": "CREATED",
  "trackingNumber": "TRK-1001"
}
```

---

# 20. Tracking API

```http
GET /api/v1/deliveries/DEL-1001/tracking
```

Response:

```json
{
  "deliveryReference": "DEL-1001",
  "status": "OUT_FOR_DELIVERY",
  "trackingNumber": "TRK-1001",
  "scheduledDate": "2026-10-06",
  "attemptNumber": 1
}
```

---

# 21. Delivery Creation Flow

```text
ShipmentCreated
      |
      ↓
Delivery Service
      |
      ↓
Create Delivery
      |
      ↓
PostgreSQL
      |
      ↓
DeliveryCreated
      |
      ↓
Kafka
```

---

# 22. Kafka Events

Topics:

```text
ecommerce.delivery.events
```

Events:

```text
DeliveryCreated
DeliveryAssigned
DeliveryOutForDelivery
DeliveryAttempted
DeliveryFailed
DeliveryRescheduled
DeliveryDelivered
DeliveryCancelled
DeliveryReturned
```

---

# 23. Event Envelope

```json
{
  "eventId": "delivery-event-1001",
  "eventType": "DeliveryDelivered",
  "version": 1,
  "occurredAt": "2026-10-05T14:00:00Z",
  "source": "delivery-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "DEL-1001",
  "payload": {
    "deliveryReference": "DEL-1001",
    "orderId": "ORD-1001",
    "status": "DELIVERED"
  }
}
```

---

# 24. Kafka Consumer

Delivery Service consumes:

```text
ShipmentCreated
ShipmentCancelled
ShipmentReturned
```

Example:

```text
Shipping Service
      |
      ↓
ShipmentCreated
      |
      ↓
Kafka
      |
      ↓
Delivery Service
      |
      ↓
Create Delivery
```

The consumer must be idempotent.

---

# 25. Kafka Consumer Idempotency

Suppose Kafka delivers:

```text
ShipmentCreated
ShipmentCreated
ShipmentCreated
```

because Kafka consumers normally operate with at-least-once processing.

The service checks:

```text
eventId
```

before processing.

If already processed:

```text
return success
```

and commit the message.

---

# 26. Redis

Redis is used for:

```text
Current delivery tracking cache
Short-lived delivery state
Rate limiting
Frequently accessed tracking information
```

Example:

```text
delivery:tracking:DEL-1001
```

TTL:

```text
60 seconds
```

is illustrative.

PostgreSQL remains the source of truth.

---

# 27. Cache-Aside

```text
Client
  |
  ↓
Delivery Service
  |
  ↓
Redis
  |
  ├── HIT → return
  |
  └── MISS
        ↓
     PostgreSQL
        ↓
      Redis
        ↓
      return
```

Never treat Redis as the authoritative delivery database.

---

# 28. Redis Failure

If Redis fails:

```text
Delivery Service
      |
      X
    Redis
```

Fallback:

```text
PostgreSQL
```

The system may become slower but should remain functionally correct.

Use short Redis timeouts.

Do not retry Redis indefinitely.

---

# 29. Delivery Partner Integration

The delivery partner may provide:

```text
Assignment
Tracking
Out-for-delivery
Attempt
Delivered
Failed
```

Communication may happen through:

```text
REST API
Webhooks
Kafka
SFTP/file integration
```

depending on the actual partner.

---

# 30. Webhook Flow

```text
Delivery Partner
       |
       ↓
HTTPS
       |
       ↓
ALB
       |
       ↓
Delivery Service
       |
       ↓
Verify Signature
       |
       ↓
Validate Event
       |
       ↓
Persist Event
       |
       ↓
Update Delivery
       |
       ↓
Publish Kafka Event
```

---

# 31. Webhook Security

Validate:

```text
TLS
Provider signature
Timestamp
Provider event ID
Replay protection
Payload schema
```

Do not trust:

```json
{
  "status": "DELIVERED"
}
```

just because the endpoint received it.

---

# 32. Webhook Handler

Conceptual Go implementation:

```go
func (h *Handler) HandleWebhook(
	c *gin.Context,
) {

	signature :=
		c.GetHeader("X-Provider-Signature")

	payload, err :=
		io.ReadAll(c.Request.Body)

	if err != nil {
		c.JSON(
			http.StatusBadRequest,
			gin.H{"error": "invalid payload"},
		)

		return
	}

	if !h.verifySignature(
		payload,
		signature,
	) {

		c.JSON(
			http.StatusUnauthorized,
			gin.H{"error": "invalid signature"},
		)

		return
	}

	err = h.service.ProcessWebhook(
		payload,
	)

	if err != nil {

		c.JSON(
			http.StatusInternalServerError,
			gin.H{"error": "processing failed"},
		)

		return
	}

	c.Status(http.StatusOK)
}
```

Actual signature verification must use the delivery provider's documented
algorithm.

---

# 33. Delivery Attempt Failure

Example:

```text
OUT_FOR_DELIVERY
       |
       ↓
Delivery Attempt
       |
       ↓
CUSTOMER_NOT_AVAILABLE
```

The service records:

```text
attemptNumber = 1
status = FAILED
failureReason = CUSTOMER_NOT_AVAILABLE
```

Then:

```text
FAILED
  |
  ↓
RESCHEDULED
```

---

# 34. Rescheduling

API:

```text
POST /api/v1/deliveries/DEL-1001/reschedule
```

Request:

```json
{
  "date": "2026-10-07"
}
```

Validate:

```text
Delivery not already delivered
Date is valid
Partner supports rescheduling
Maximum attempts not exceeded
```

---

# 35. Maximum Delivery Attempts

Example:

```text
Maximum attempts = 3
```

Illustrative flow:

```text
Attempt 1 → FAILED
Attempt 2 → FAILED
Attempt 3 → FAILED
             ↓
       RETURN_TO_ORIGIN
```

The actual policy is business-specific.

---

# 36. Proof of Delivery

Possible proof:

```text
Signature
OTP verification
Photo
Recipient name
Timestamp
GPS metadata
```

The Delivery Service should store references/metadata rather than unnecessarily
storing large binary files.

Example:

```text
S3
 |
 ↓
proof-of-delivery/DEL-1001/photo.jpg
```

Database:

```text
proof_of_delivery_url
```

The actual bucket should be private.

---

# 37. S3 Architecture

```text
Delivery Service
      |
      ↓
S3
      |
      ↓
Proof-of-delivery object
```

Access:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
S3 prefix
```

IAM should be restricted to the required bucket/prefix.

---

# 38. S3 Security

```text
[ ] Private bucket
[ ] Block public access
[ ] Encryption
[ ] Least privilege IAM
[ ] Versioning where appropriate
[ ] Lifecycle policy
[ ] Access logging/auditing
[ ] No public object URLs
```

Do not expose:

```text
s3://bucket/internal/path
```

directly to customers.

Use controlled application access or appropriately secured temporary access
mechanisms where required.

---

# 39. Delivery Service

Conceptual service layer:

```go
func (s *Service) UpdateStatus(
	deliveryReference string,
	nextStatus string,
	reason string,
) error {

	delivery, err :=
		s.repository.FindByReference(
			deliveryReference,
		)

	if err != nil {
		return err
	}

	if !IsValidTransition(
		delivery.Status,
		nextStatus,
	) {
		return fmt.Errorf(
			"invalid delivery transition: %s -> %s",
			delivery.Status,
			nextStatus,
		)
	}

	delivery.Status = nextStatus

	if nextStatus == StatusDelivered {
		now := time.Now().UTC()
		delivery.DeliveredAt = &now
	}

	delivery.FailureReason = reason

	return s.repository.Update(
		delivery,
	)
}
```

---

# 40. Concurrency Protection

Two events may arrive almost simultaneously:

```text
Event A:
OUT_FOR_DELIVERY → DELIVERED

Event B:
OUT_FOR_DELIVERY → FAILED
```

Without concurrency control, the final state may depend on race timing.

Use:

```text
database transaction
+
row locking/versioning
+
state transition validation
```

Example PostgreSQL:

```sql
SELECT *
FROM deliveries
WHERE delivery_reference = $1
FOR UPDATE;
```

Then validate and update within the same transaction.

---

# 41. Optimistic Locking Alternative

Add:

```sql
version BIGINT NOT NULL DEFAULT 0
```

Update:

```sql
UPDATE deliveries
SET
    status = $1,
    version = version + 1
WHERE
    delivery_reference = $2
    AND version = $3;
```

If affected rows = `0`:

```text
Concurrent update detected
```

The service can reload the latest state and re-evaluate the event.

---

# 42. Why This Matters

Delivery events are asynchronous.

Therefore:

```text
same shipment
+
multiple event sources
+
network retries
+
out-of-order events
```

are normal distributed-system conditions.

The application must not assume:

```text
events always arrive once
events always arrive in order
events always arrive immediately
```

---

# 43. Out-of-Order Events

Example:

```text
Event 1:
OUT_FOR_DELIVERY

Event 2:
DELIVERED

Event 3:
OUT_FOR_DELIVERY
```

Event 3 is stale.

Use:

```text
event timestamp
provider sequence number
state transition rules
```

where the provider supports them.

Never blindly overwrite:

```text
DELIVERED
```

with:

```text
OUT_FOR_DELIVERY
```

---

# 44. Delivery Reconciliation

Periodically query partner systems for shipments that are:

```text
OUT_FOR_DELIVERY
DELIVERY_ATTEMPTED
UNKNOWN
```

for longer than the expected processing window.

Flow:

```text
Scheduled Reconciliation
        |
        ↓
Find suspicious deliveries
        |
        ↓
Partner Tracking API
        |
        ↓
Compare state
        |
        ↓
Correct internal state
```

---

# 45. SQS Usage

For large tracking workloads:

```text
Partner Event
      |
      ↓
SQS
      |
      ↓
Delivery Worker
      |
      ↓
PostgreSQL
      |
      ↓
Kafka
```

Benefits:

- Backpressure
- Retry
- DLQ
- Independent scaling
- Failure isolation

---

# 46. SQS DLQ

```text
Delivery Event
      |
      ↓
Consumer
      |
      ├── Success
      |
      └── Failure
            |
            ↓
          Retry
            |
            ↓
           DLQ
```

Investigate DLQ messages before replay.

---

# 47. KEDA

Because delivery processing can be queue-driven, KEDA can scale workers based
on SQS backlog.

Conceptually:

```text
SQS backlog
    ↓
KEDA
    ↓
Delivery workers
    ↓
Carrier/partner
```

But maximum consumer replicas must respect:

```text
Database capacity
Partner API rate limit
Redis capacity
Kafka capacity
```

---

# 48. HPA vs KEDA

Synchronous API:

```text
Delivery API
    ↓
HPA
```

Asynchronous processing:

```text
SQS
 ↓
KEDA
 ↓
Workers
```

Node capacity:

```text
Pods
 ↓
Karpenter
 ↓
EC2 nodes
```

Architecture:

```text
Traffic
  ↓
HPA
  ↓
Pods
  ↓
Karpenter

Queue
  ↓
KEDA
  ↓
Workers
  ↓
Karpenter
```

---

# 49. Dockerfile

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -o /out/delivery-service \
    ./cmd/server


FROM gcr.io/distroless/static-debian12

WORKDIR /app

COPY --from=builder \
    /out/delivery-service \
    /app/delivery-service

USER 10001

EXPOSE 8080

ENTRYPOINT ["/app/delivery-service"]
```

Use the architecture appropriate to the cluster nodes or build a multi-platform
image where required.

---

# 50. `.dockerignore`

```text
.git
.github

bin
tmp

coverage

.env
.env.*

*.log

README.md

tests
```

---

# 51. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: delivery-postgres

    environment:

      POSTGRES_DB: delivery

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: delivery-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: delivery-kafka

    ports:

      - "9092:9092"

    # Development-only configuration.
    # Pin a tested Kafka version for production.


  delivery-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      PORT: 8080

      DATABASE_URL:
        postgres://postgres:postgres@postgres:5432/delivery

      REDIS_ADDR:
        redis:6379

      KAFKA_BROKERS:
        kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 52. Kubernetes Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: delivery-service

  labels:

    app: delivery-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: delivery-service

  template:

    metadata:

      labels:

        app: delivery-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: delivery-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: delivery-service-config

            - secretRef:

                name: delivery-service-secrets

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

# 53. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: delivery-service

  labels:

    app: delivery-service

spec:

  type: ClusterIP

  selector:

    app: delivery-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 54. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: delivery-service-config

data:

  REDIS_ADDR: redis.ecommerce.svc.cluster.local:6379

  KAFKA_BROKERS: kafka.ecommerce.svc.cluster.local:9092

  DELIVERY_CACHE_TTL_SECONDS: "60"
```

---

# 55. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: delivery-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: delivery-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/delivery-service

        property: DATABASE_URL

    - secretKey: PARTNER_WEBHOOK_SECRET

      remoteRef:

        key: ecommerce/prod/delivery-service

        property: PARTNER_WEBHOOK_SECRET
```

---

# 56. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: delivery-service

  labels:

    app: delivery-service
```

If the service uploads proof-of-delivery metadata to S3:

```text
Pod
 ↓
EKS Pod Identity
 ↓
delivery-service IAM role
 ↓
S3
```

The role should only access the required bucket/prefix.

---

# 57. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: delivery-service

spec:

  podSelector:

    matchLabels:

      app: delivery-service

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

# 58. Resource Configuration

```yaml
resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi
```

Tune from load testing rather than copying these values blindly.

---

# 59. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: delivery-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: delivery-service

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

---

# 60. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: delivery-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: delivery-service
```

---

# 61. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: delivery-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: delivery-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 62. Delivery Metrics

Technical:

```text
delivery_requests_total

delivery_request_duration_seconds

delivery_errors_total

delivery_db_errors_total

delivery_redis_errors_total

delivery_kafka_errors_total
```

Business:

```text
deliveries_created_total

deliveries_assigned_total

deliveries_out_for_delivery_total

deliveries_delivered_total

deliveries_failed_total

deliveries_rescheduled_total

deliveries_returned_total
```

Operational:

```text
delivery_attempts_total

delivery_webhook_total

delivery_webhook_invalid_total

delivery_duplicate_event_total

delivery_reconciliation_mismatch_total
```

---

# 63. Grafana Dashboard

```text
Delivery Service
│
├── Request Rate
├── P95
├── P99
├── HTTP 5xx
│
├── Delivery State
│   ├── Created
│   ├── Assigned
│   ├── Out for Delivery
│   ├── Failed
│   └── Delivered
│
├── Delivery Attempts
│
├── Partner
│   ├── Latency
│   ├── Errors
│   └── Rate Limits
│
├── Webhooks
│   ├── Valid
│   ├── Invalid
│   └── Duplicate
│
├── Kafka
│   └── Consumer Lag
│
├── SQS
│   └── Queue Age
│
└── PostgreSQL / Redis
```

---

# 64. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T15:00:00Z",
  "level": "INFO",
  "service": "delivery-service",
  "environment": "prod",
  "correlationId": "corr-order-1001",
  "deliveryReference": "DEL-1001",
  "orderId": "ORD-1001",
  "trackingNumber": "TRK-1001",
  "previousStatus": "OUT_FOR_DELIVERY",
  "newStatus": "DELIVERED",
  "message": "Delivery status updated"
}
```

Avoid logging:

```text
full customer address
OTP values
authentication tokens
provider secrets
unnecessary personal data
```

---

# 65. Distributed Tracing

Example:

```text
Order Service
     |
     ↓
Shipping Service
     |
     ↓
Delivery Service
     |
     ↓
Delivery Partner
```

Use:

```text
traceId
spanId
correlationId
eventId
deliveryReference
```

The identifiers should be consistent across synchronous and asynchronous
boundaries.

---

# 66. SLO

Example initial SLOs:

Delivery API availability:

```text
99.95%
```

Tracking API P95:

```text
< 200ms
```

Webhook processing:

```text
99.9% within defined processing window
```

Critical delivery-event processing:

```text
99.9% successful
```

Tracking freshness:

```text
99% of partner updates reflected within agreed target
```

Exact values must be agreed with the business and delivery partners.

---

# 67. Alerts

```text
High Delivery API 5xx

High Delivery API Latency

High Kafka Consumer Lag

High SQS Backlog

High Webhook Failure Rate

Webhook Signature Failures

Delivery Failure Spike

Delivery Reschedule Spike

Delivery Reconciliation Mismatch

Redis Failure

PostgreSQL Connection Exhaustion

Unexpected Delivered-State Regression

Delivery Partner Failure
```

---

# 68. Critical Business Alert

Suppose:

```text
Delivered:
10,000/hour → 500/hour
```

while:

```text
CPU = 30%
Memory = 40%
Pods = healthy
```

This is still a major incident.

Why?

Because:

```text
Infrastructure healthy
≠
Business healthy
```

The alert should detect the business metric.

---

# 69. Delivery Partner Failure

Suppose partner A stops sending events.

```text
Partner A
    |
    X
 Webhooks
```

Delivery status stops changing.

Detection:

```text
Webhook freshness
       ↓
Alert
       ↓
Check partner
       ↓
Check network
       ↓
Check webhook endpoint
       ↓
Check partner status
```

Reconciliation can provide a secondary protection mechanism.

---

# 70. Delivery Event Ordering Problem

Suppose:

```text
15:00 OUT_FOR_DELIVERY
15:05 DELIVERED
15:06 OUT_FOR_DELIVERY
```

The final state must remain:

```text
DELIVERED
```

unless the provider explicitly supplies a legitimate correction.

Use:

```text
provider timestamp
+
sequence number where available
+
state transition validation
+
database concurrency control
```

---

# 71. Delivery Failure Scenario

```text
Customer unavailable
       |
       ↓
Delivery Attempt Failed
       |
       ↓
Reason recorded
       |
       ↓
Customer notified
       |
       ↓
Reschedule
       |
       ↓
New attempt
```

Notification Service consumes:

```text
DeliveryFailed
DeliveryRescheduled
```

events.

---

# 72. Return-to-Origin

After maximum failed attempts:

```text
Attempt 1 → FAILED
Attempt 2 → FAILED
Attempt 3 → FAILED
               |
               ↓
        RETURN_TO_ORIGIN
```

Event:

```text
DeliveryReturned
```

Consumers:

```text
Order Service
Inventory Service
Notification Service
Reporting Service
```

The Order/Saga workflow determines the business consequences.

---

# 73. Delivery Cancellation

A cancellation request should verify:

```text
Current status
Partner status
Cancellation policy
Whether delivery has started
```

For example:

```text
CREATED
→ CANCELLED
```

may be valid.

But:

```text
DELIVERED
→ CANCELLED
```

is invalid.

---

# 74. Production Incident Example

## Incident ID

```text
INC-2026-DELIVERY-001
```

## Severity

```text
SEV-2
```

## Title

```text
Out-of-Order Delivery Events Reverted Delivered Shipments
```

## Date

```text
2026-10-05
```

---

# 75. Customer Impact

Some customers saw:

```text
Delivered
```

followed later by:

```text
Out for Delivery
```

in the tracking system.

Physical deliveries were not reversed.

The issue was incorrect application state caused by stale partner events.

---

# 76. Timeline

```text
16:00
Delivery service healthy.

16:05
Partner sends DELIVERED event.

16:05
Delivery becomes DELIVERED.

16:07
Delayed OUT_FOR_DELIVERY event arrives.

16:07
Service overwrites status.

16:10
Customer support reports tracking inconsistency.

16:12
Dashboard shows delivered-state regression.

16:14
SEV-2 declared.

16:16
Logs reveal out-of-order partner events.

16:20
Engineering disables stale status overwrite.

16:23
Reconciliation identifies affected deliveries.

16:27
Delivered states corrected.

16:32
Customer tracking normal.

16:40
Incident resolved.
```

---

# 77. Root Cause

The service assumed that delivery events would always arrive in chronological
order.

The partner sent an older event after a newer event.

The application blindly overwrote the current state.

---

# 78. Amplification

```text
Out-of-order event
      ↓
State overwritten
      ↓
Incorrect customer tracking
      ↓
Notification events
      ↓
Additional customer confusion
```

---

# 79. What Went Well

- Event IDs were available
- PostgreSQL retained delivery history
- Reconciliation could identify affected records
- Business support detected the issue quickly
- No physical delivery was affected

---

# 80. What Went Wrong

- Event ordering was assumed
- Provider timestamp was not evaluated
- State transition protection was incomplete
- No alert existed for `DELIVERED → OUT_FOR_DELIVERY`

---

# 81. Five Whys

### Why did customers see incorrect tracking?

A stale event changed a delivered shipment back to out-for-delivery.

### Why was the stale event accepted?

The service did not compare event ordering information.

### Why?

The integration assumed provider events were ordered.

### Why wasn't the assumption tested?

The integration tests covered duplicate events but not out-of-order events.

### Why did production expose it?

No business alert existed for terminal-state regression.

---

# 82. Corrective Actions

| Action | Owner |
|---|---|
| Validate provider timestamps | Delivery |
| Add sequence validation | Delivery |
| Protect terminal states | Backend |
| Add out-of-order tests | QA |
| Add terminal-state regression alert | SRE |
| Improve reconciliation | Delivery |
| Add partner contract documentation | Architecture |
| Run partner failure simulations | SRE |

---

# 83. Production Troubleshooting — Tracking Wrong

Check:

```bash
kubectl logs \
  -n ecommerce \
  deploy/delivery-service \
  --tail=200
```

Then search by:

```text
deliveryReference
trackingNumber
orderId
correlationId
eventId
```

Check:

```text
PostgreSQL delivery state
delivery_events
provider timestamps
Kafka events
Redis cache
partner API
```

---

# 84. Production Troubleshooting — Delivery API 5xx

Check:

```text
Pods
Service
EndpointSlices
Ingress/ALB
Application logs
PostgreSQL
Redis
Kafka
Secrets
NetworkPolicy
Recent deployment
```

Architecture:

```text
ALB
 ↓
Service
 ↓
Pod
 ↓
PostgreSQL / Redis / Kafka
```

---

# 85. Production Troubleshooting — Kafka Lag

Check:

```text
Consumer group
Partition count
Consumer replicas
Processing latency
Database latency
Redis latency
CPU
Memory
Downstream partner calls
```

Do not blindly increase consumers.

More consumers may increase:

```text
PostgreSQL connections
partner API requests
Redis load
```

---

# 86. Production Troubleshooting — SQS Backlog

Check:

```text
ApproximateAgeOfOldestMessage
ApproximateNumberOfMessagesVisible
Consumer replicas
Consumer errors
DLQ
Partner latency
```

If partner API is rate-limited:

```text
Do NOT
```

simply scale:

```text
3 workers → 100 workers
```

because this may make the partner outage worse.

---

# 87. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Delivery Service CI

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

  ECR_REPOSITORY: delivery-service

jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4

      - name: Setup Go

        uses: actions/setup-go@v5

        with:

          go-version: "1.24"

      - name: Download dependencies

        run: go mod download

      - name: Test

        run: go test ./...

      - name: Build

        run: go build ./...

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

Production action versions should be pinned according to the repository's
supply-chain policy.

---

# 88. SonarQube

Go code should be analyzed for:

```text
bugs
security vulnerabilities
code smells
duplication
coverage
```

Pipeline:

```text
Checkout
 ↓
go test
 ↓
coverage
 ↓
SonarQube
 ↓
Trivy
 ↓
Docker
 ↓
Trivy Image
 ↓
ECR
```

SonarQube is a centralized platform service.

---

# 89. ECR

Repository:

```text
company-ecommerce/delivery-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/delivery-service:<git-sha>
```

Use immutable image tags.

Prefer digest-based deployment where supported by the GitOps workflow.

---

# 90. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: delivery-service

description: Ecommerce delivery service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 91. Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/delivery-service

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

  name: delivery-service

autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60
```

---

# 92. GitOps

Repository:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
└── apps/
    └── delivery-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Flow:

```text
Developer
   ↓
PR
   ↓
CI
   ↓
Security
   ↓
Docker
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

# 93. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: delivery-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/delivery-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

Production promotion remains controlled through the GitOps approval workflow.

---

# 94. Delivery Security Checklist

```text
[ ] Private database
[ ] Redis protected
[ ] NetworkPolicy
[ ] TLS
[ ] Webhook signature verification
[ ] Replay protection
[ ] Secrets Manager
[ ] External Secrets
[ ] EKS Pod Identity
[ ] Least privilege IAM
[ ] S3 private
[ ] No credentials in Git
[ ] No credentials in Docker image
[ ] Non-root container
[ ] Read-only filesystem
[ ] Trivy
[ ] SonarQube
[ ] Dependency scanning
[ ] Audit logs
```

---

# 95. Performance Checklist

```text
[ ] Tracking API latency
[ ] PostgreSQL connection pool
[ ] Redis latency
[ ] Kafka lag
[ ] SQS backlog
[ ] Partner API latency
[ ] Partner rate limits
[ ] Webhook throughput
[ ] HPA behavior
[ ] KEDA behavior
[ ] Karpenter capacity
[ ] Load testing
[ ] Spike testing
[ ] Failure testing
```

---

# 96. Production Deployment Checklist

```text
[ ] Unit tests
[ ] Integration tests
[ ] State transition tests
[ ] Duplicate event tests
[ ] Out-of-order event tests
[ ] Webhook signature tests
[ ] Retry tests
[ ] DLQ tests
[ ] Reconciliation tests
[ ] Partner timeout tests
[ ] Partner 429 tests
[ ] Trivy
[ ] SonarQube
[ ] ECR image
[ ] Image digest
[ ] GitOps PR
[ ] ArgoCD sync
[ ] Pods ready
[ ] HPA
[ ] PDB
[ ] NetworkPolicy
[ ] Secrets
[ ] Prometheus
[ ] Grafana
[ ] ELK
[ ] OpenTelemetry
[ ] Jaeger
[ ] Alerts
[ ] Runbook
```

---

# 97. Final Architecture

```text
                         Order Service
                              |
                              ↓
                        Shipping Service
                              |
                              ↓
                       Shipment Created
                              |
                              ↓
                       Delivery Service
                    /         |          \
                   /          |           \
                  ↓           ↓            ↓
            PostgreSQL      Redis        Kafka
                  |                        |
                  |                        ├── Notification
                  |                        ├── Reporting
                  |                        └── Analytics
                  |
                  ↓
             Delivery State
                  |
                  ↓
          Delivery Partner
                  |
                  ↓
              Webhook
                  |
                  ↓
          Delivery Service
                  |
                  ↓
          State Validation
                  |
             ┌────┴────┐
             ↓         ↓
          Success     Failure
             |         |
             ↓         ↓
         DELIVERED   RESCHEDULE
                       |
                       ↓
                  New Attempt
```

---

# 98. Final Production Principles

1. Delivery is separate from Shipping.
2. Delivery state must be modeled as a state machine.
3. Invalid state transitions must be rejected.
4. Delivery events are at-least-once.
5. Duplicate events must be safe.
6. Out-of-order events must be handled.
7. Terminal states require special protection.
8. PostgreSQL is the durable source of truth.
9. Redis is a performance layer.
10. Kafka distributes delivery events.
11. SQS provides asynchronous backpressure where appropriate.
12. Delivery webhooks must be authenticated.
13. Webhooks must support replay protection.
14. Delivery attempts must be auditable.
15. Failed deliveries must have explicit reasons.
16. Rescheduling must be controlled.
17. Maximum delivery attempts must be business-configurable.
18. Proof-of-delivery objects should use private S3 storage.
19. Carrier/partner rate limits must be respected.
20. More workers do not automatically mean more throughput.
21. HPA and KEDA must consider downstream limits.
22. Reconciliation is essential for distributed delivery systems.
23. Business metrics are as important as CPU and memory.
24. `DELIVERED → OUT_FOR_DELIVERY` should be treated as a serious anomaly.
25. GitOps remains the production deployment source of truth.
26. A healthy EKS cluster does not guarantee a healthy delivery system.
27. Every major delivery incident requires an RCA.
28. Correctness is more important than merely increasing throughput.

# End of 174 — Delivery Service
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

175 Notification Service    ⏳
176 Search Service          ⏳
177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `175-Notification-Service.md`** — multi-channel notifications, email/SMS/push, Kafka/SQS, provider adapters, templates, retry/DLQ, idempotency, rate limits, delivery status, user preferences, PostgreSQL/Redis, Kubernetes, CI/CD, GitOps, Prometheus/Grafana/ELK/OpenTelemetry/Jaeger, failure scenarios and production RCA.