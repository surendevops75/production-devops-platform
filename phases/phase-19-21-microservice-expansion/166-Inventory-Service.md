# 166 — Inventory Service

**File:** `166-Inventory-Service.md`

```markdown
# 166 — Inventory Service

## 1. Overview

The `inventory-service` manages product stock and inventory reservations for
the Enterprise E-Commerce Platform.

Inventory is one of the most critical services because incorrect concurrency
handling can result in:

- Overselling
- Negative inventory
- Duplicate reservations
- Lost reservations
- Incorrect stock counts
- Order failures
- Payment inconsistencies

This service is intentionally implemented using:

- Go 1.24
- Gin
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

The most important part of this service is NOT simply creating CRUD APIs.

The critical engineering problem is:

> How do we safely reserve inventory when thousands of customers attempt to
> purchase the same product simultaneously?

---

# 2. Business Responsibility

Inventory service owns:

- Available quantity
- Reserved quantity
- Inventory reservations
- Stock adjustments
- Inventory release
- Inventory confirmation
- Inventory availability checks
- Warehouse/location stock where applicable

It does NOT own:

- Product name
- Product description
- Product price
- Customer profile
- Orders
- Payments
- Product categories

Ownership:

```text
product-service
      |
      | productId
      ↓
inventory-service
      |
      ├── available quantity
      ├── reserved quantity
      └── reservations
```

---

# 3. Inventory Model

We maintain:

```text
total stock
available stock
reserved stock
```

Conceptually:

```text
Available Stock
      +
Reserved Stock
      =
Stock controlled by Inventory
```

Example:

```text
Total = 100

Available = 80

Reserved = 20
```

When a customer successfully reserves 5:

```text
Available = 75
Reserved = 25
```

When the order is cancelled:

```text
Available = 80
Reserved = 20
```

When the order is confirmed:

```text
Available = 75
Reserved = 20
```

The exact accounting model depends on the order workflow.

---

# 4. Why Inventory Is Different

Most CRUD services can perform:

```text
SELECT
UPDATE
```

without complicated concurrency handling.

Inventory cannot safely do:

```text
SELECT available_quantity

if available_quantity >= requested:
    UPDATE available_quantity
```

without protection.

Suppose:

```text
Available = 10
```

Two customers simultaneously request:

```text
Customer A = 8
Customer B = 8
```

Both could read:

```text
Available = 10
```

Both believe they can reserve.

Result:

```text
10 - 8 - 8 = -6
```

This is overselling.

Therefore inventory reservation must be atomic.

---

# 5. Technology Stack

| Layer | Technology |
|---|---|
| Language | Go |
| HTTP | Gin |
| Database | PostgreSQL |
| DB driver | pgx |
| ORM | GORM |
| Cache | Redis |
| Messaging | Kafka |
| Dependency management | Go Modules |
| Testing | Go testing |
| Container | Docker |
| Orchestration | Kubernetes/EKS |
| Packaging | Helm |
| Registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| Metrics | Prometheus |
| Dashboards | Grafana |
| Logs | ELK |
| Tracing | OpenTelemetry |
| Trace backend | Jaeger |

---

# 6. Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                    inventory-service
                            |
               ┌────────────┼────────────┐
               │            │            │
               ↓            ↓            ↓
          PostgreSQL      Redis         Kafka
               │                         │
               │                         ├── Order Service
               │                         ├── Product Service
               │                         ├── Reporting
               │                         └── Other Consumers
               │
               ↓
          Reservations
```

PostgreSQL is the source of truth.

Redis is used for read optimization where appropriate.

Kafka is used for asynchronous inventory events.

---

# 7. Request Flow

## Availability Check

```text
Client
  |
  ↓
inventory-service
  |
  ↓
Redis
  |
  ├── HIT
  |
  └── MISS
       |
       ↓
   PostgreSQL
```

## Reservation

```text
Order Service
     |
     ↓
inventory-service
     |
     ↓
PostgreSQL transaction
     |
     ├── Lock/check stock
     |
     ├── Reduce available
     |
     ├── Increase reserved
     |
     └── Create reservation
             |
             ↓
        Kafka event
```

---

# 8. APIs

## Check Availability

```text
GET /api/v1/inventory/{productId}
```

## Reserve Inventory

```text
POST /api/v1/inventory/{productId}/reservations
```

## Release Reservation

```text
POST /api/v1/inventory/reservations/{reservationId}/release
```

## Confirm Reservation

```text
POST /api/v1/inventory/reservations/{reservationId}/confirm
```

## Adjust Stock

```text
POST /api/v1/inventory/{productId}/adjust
```

## Health

```text
GET /health
```

## Readiness

```text
GET /ready
```

## Metrics

```text
GET /metrics
```

---

# 9. Repository Structure

```text
inventory-service/
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
│   ├── cache/
│   │   └── redis.go
│   │
│   ├── messaging/
│   │   └── kafka.go
│   │
│   ├── inventory/
│   │   ├── model.go
│   │   ├── repository.go
│   │   ├── service.go
│   │   └── handler.go
│   │
│   ├── reservation/
│   │   ├── model.go
│   │   ├── repository.go
│   │   ├── service.go
│   │   └── handler.go
│   │
│   └── observability/
│       └── metrics.go
│
├── migrations/
│   ├── V1__create_inventory.sql
│   └── V2__create_reservations.sql
│
├── tests/
│
├── go.mod
├── go.sum
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── .env.example
├── Makefile
├── README.md
│
├── helm/
│   └── inventory-service/
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

# 10. Go Module

## `go.mod`

```go
module github.com/company-ecommerce/inventory-service

go 1.24

require (
    github.com/gin-gonic/gin v1.10.1
    github.com/go-redis/redis/v9 v9.7.0
    github.com/google/uuid v1.6.0
    github.com/prometheus/client_golang v1.21.0
    github.com/segmentio/kafka-go v0.4.49
    gorm.io/driver/postgres v1.5.11
    gorm.io/gorm v1.25.12
)
```

Versions are examples for the project.

In production:

- Commit `go.mod`
- Commit `go.sum`
- Review dependency upgrades through PRs
- Run vulnerability scanning
- Avoid uncontrolled dependency upgrades

---

# 11. Configuration

## `.env.example`

```text
APP_NAME=inventory-service
ENVIRONMENT=local
PORT=8080

DATABASE_URL=postgres://postgres:postgres@localhost:5432/inventory

REDIS_ADDR=localhost:6379

KAFKA_BROKERS=localhost:9092
KAFKA_TOPIC=inventory-events

DB_MAX_OPEN_CONNS=20
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME_SECONDS=1500
```

Never commit real production credentials.

---

# 12. Configuration Code

## `internal/config/config.go`

```go
package config

import (
    "os"
    "strconv"
    "strings"
)

type Config struct {
    AppName     string
    Environment string
    Port        string

    DatabaseURL string

    RedisAddr string

    KafkaBrokers []string
    KafkaTopic   string

    DBMaxOpenConns int
    DBMaxIdleConns int
    DBConnLifetime int
}

func getEnv(
    key string,
    fallback string,
) string {

    value := os.Getenv(key)

    if value == "" {
        return fallback
    }

    return value
}

func getInt(
    key string,
    fallback int,
) int {

    value := os.Getenv(key)

    if value == "" {
        return fallback
    }

    parsed, err := strconv.Atoi(value)

    if err != nil {
        return fallback
    }

    return parsed
}

func Load() Config {

    brokers := strings.Split(
        getEnv(
            "KAFKA_BROKERS",
            "localhost:9092",
        ),
        ",",
    )

    return Config{
        AppName: getEnv(
            "APP_NAME",
            "inventory-service",
        ),

        Environment: getEnv(
            "ENVIRONMENT",
            "local",
        ),

        Port: getEnv(
            "PORT",
            "8080",
        ),

        DatabaseURL: getEnv(
            "DATABASE_URL",
            "postgres://postgres:postgres@localhost:5432/inventory",
        ),

        RedisAddr: getEnv(
            "REDIS_ADDR",
            "localhost:6379",
        ),

        KafkaBrokers: brokers,

        KafkaTopic: getEnv(
            "KAFKA_TOPIC",
            "inventory-events",
        ),

        DBMaxOpenConns: getInt(
            "DB_MAX_OPEN_CONNS",
            20,
        ),

        DBMaxIdleConns: getInt(
            "DB_MAX_IDLE_CONNS",
            5,
        ),

        DBConnLifetime: getInt(
            "DB_CONN_MAX_LIFETIME_SECONDS",
            1500,
        ),
    }
}
```

---

# 13. PostgreSQL Connection

## `internal/database/postgres.go`

```go
package database

import (
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/company-ecommerce/inventory-service/internal/config"
)

func Connect(
    cfg config.Config,
) (*gorm.DB, error) {

    db, err := gorm.Open(
        postgres.Open(cfg.DatabaseURL),
        &gorm.Config{},
    )

    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()

    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxOpenConns(
        cfg.DBMaxOpenConns,
    )

    sqlDB.SetMaxIdleConns(
        cfg.DBMaxIdleConns,
    )

    sqlDB.SetConnMaxLifetime(
        time.Duration(
            cfg.DBConnLifetime,
        ) * time.Second,
    )

    return db, nil
}
```

---

# 14. Connection Pool Planning

The connection pool is per application process.

If Kubernetes has:

```text
10 Pods
```

and:

```text
DB_MAX_OPEN_CONNS=20
```

the theoretical maximum is approximately:

```text
10 × 20 = 200 connections
```

If HPA scales to:

```text
30 Pods
```

then:

```text
30 × 20 = 600 connections
```

Therefore inventory HPA cannot be designed independently from PostgreSQL
capacity.

Inventory is particularly sensitive because reservation transactions may hold
connections while waiting for database locks.

---

# 15. Inventory Model

## `internal/inventory/model.go`

```go
package inventory

import (
    "time"

    "github.com/google/uuid"
)

type Inventory struct {

    ID uint `gorm:"primaryKey"`

    ProductID uuid.UUID `gorm:"type:uuid;uniqueIndex;not null"`

    AvailableQuantity int64 `gorm:"not null"`

    ReservedQuantity int64 `gorm:"not null"`

    Version int64 `gorm:"not null;default:0"`

    CreatedAt time.Time

    UpdatedAt time.Time
}
```

The `Version` column can support optimistic concurrency approaches.

For the core reservation operation we will demonstrate an atomic SQL update.

---

# 16. Reservation Model

## `internal/reservation/model.go`

```go
package reservation

import (
    "time"

    "github.com/google/uuid"
)

type Reservation struct {

    ID uint `gorm:"primaryKey"`

    ReservationID uuid.UUID
        `gorm:"type:uuid;uniqueIndex;not null"`

    ProductID uuid.UUID
        `gorm:"type:uuid;index;not null"`

    OrderID uuid.UUID
        `gorm:"type:uuid;index;not null"`

    Quantity int64
        `gorm:"not null"`

    Status string
        `gorm:"size:30;not null"`

    IdempotencyKey string
        `gorm:"size:200;uniqueIndex;not null"`

    ExpiresAt time.Time

    CreatedAt time.Time

    UpdatedAt time.Time
}
```

---

# 17. Reservation States

```text
PENDING
   |
   +----> CONFIRMED
   |
   +----> RELEASED
   |
   +----> EXPIRED
```

Example:

```text
Order created
     |
     ↓
Inventory reserved
     |
     ↓
Payment succeeds
     |
     ↓
Reservation confirmed
```

If payment fails:

```text
Payment failure
      |
      ↓
Release inventory
```

---

# 18. Database Migration

## `V1__create_inventory.sql`

```sql
CREATE TABLE inventory (

    id BIGSERIAL PRIMARY KEY,

    product_id UUID NOT NULL UNIQUE,

    available_quantity BIGINT NOT NULL,

    reserved_quantity BIGINT NOT NULL,

    version BIGINT NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT inventory_non_negative_available
        CHECK (available_quantity >= 0),

    CONSTRAINT inventory_non_negative_reserved
        CHECK (reserved_quantity >= 0)
);
```

---

# 19. Reservation Migration

## `V2__create_reservations.sql`

```sql
CREATE TABLE reservations (

    id BIGSERIAL PRIMARY KEY,

    reservation_id UUID NOT NULL UNIQUE,

    product_id UUID NOT NULL,

    order_id UUID NOT NULL,

    quantity BIGINT NOT NULL,

    status VARCHAR(30) NOT NULL,

    idempotency_key VARCHAR(200) NOT NULL UNIQUE,

    expires_at TIMESTAMPTZ NOT NULL,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT reservation_quantity_positive
        CHECK (quantity > 0)
);

CREATE INDEX idx_reservations_product
ON reservations(product_id);

CREATE INDEX idx_reservations_order
ON reservations(order_id);

CREATE INDEX idx_reservations_status
ON reservations(status);

CREATE INDEX idx_reservations_expiry
ON reservations(expires_at);
```

---

# 20. The Critical Concurrency Problem

Bad implementation:

```sql
SELECT available_quantity
FROM inventory
WHERE product_id = ?;
```

Then application code:

```text
if available >= requested:
    UPDATE inventory
```

This creates a race condition.

Two transactions can both read the same quantity.

---

# 21. Safer Atomic Reservation

One approach is to perform the stock deduction atomically:

```sql
UPDATE inventory
SET
    available_quantity =
        available_quantity - ?,
    reserved_quantity =
        reserved_quantity + ?,
    version =
        version + 1,
    updated_at =
        CURRENT_TIMESTAMP
WHERE
    product_id = ?
    AND available_quantity >= ?;
```

Then check:

```text
Rows affected = 1
```

Reservation succeeded.

If:

```text
Rows affected = 0
```

then either:

- Product does not exist
- Insufficient stock

This approach avoids a separate unsafe read-before-update sequence.

---

# 22. Repository

## `internal/inventory/repository.go`

```go
package inventory

import (
    "errors"

    "github.com/google/uuid"

    "gorm.io/gorm"
)

var ErrInsufficientStock = errors.New(
    "insufficient stock",
)

type Repository struct {
    db *gorm.DB
}

func NewRepository(
    db *gorm.DB,
) *Repository {

    return &Repository{
        db: db,
    }
}

func (r *Repository) Find(
    productID uuid.UUID,
) (*Inventory, error) {

    var inventory Inventory

    result := r.db.
        Where(
            "product_id = ?",
            productID,
        ).
        First(&inventory)

    if result.Error != nil {
        return nil, result.Error
    }

    return &inventory, nil
}

func (r *Repository) Reserve(
    tx *gorm.DB,
    productID uuid.UUID,
    quantity int64,
) error {

    result := tx.Model(
        &Inventory{},
    ).
        Where(
            "product_id = ? AND available_quantity >= ?",
            productID,
            quantity,
        ).
        Updates(
            map[string]interface{}{
                "available_quantity":
                    gorm.Expr(
                        "available_quantity - ?",
                        quantity,
                    ),

                "reserved_quantity":
                    gorm.Expr(
                        "reserved_quantity + ?",
                        quantity,
                    ),

                "version":
                    gorm.Expr(
                        "version + 1",
                    ),
            },
        )

    if result.Error != nil {
        return result.Error
    }

    if result.RowsAffected == 0 {
        return ErrInsufficientStock
    }

    return nil
}

func (r *Repository) Release(
    tx *gorm.DB,
    productID uuid.UUID,
    quantity int64,
) error {

    result := tx.Model(
        &Inventory{},
    ).
        Where(
            "product_id = ? AND reserved_quantity >= ?",
            productID,
            quantity,
        ).
        Updates(
            map[string]interface{}{
                "available_quantity":
                    gorm.Expr(
                        "available_quantity + ?",
                        quantity,
                    ),

                "reserved_quantity":
                    gorm.Expr(
                        "reserved_quantity - ?",
                        quantity,
                    ),

                "version":
                    gorm.Expr(
                        "version + 1",
                    ),
            },
        )

    if result.Error != nil {
        return result.Error
    }

    if result.RowsAffected == 0 {
        return ErrInsufficientStock
    }

    return nil
}
```

---

# 23. Why Atomic UPDATE Matters

Suppose:

```text
Available = 10
```

Request A:

```text
quantity = 8
```

Request B:

```text
quantity = 8
```

The database evaluates:

```sql
available_quantity >= 8
```

for each transaction while updating the row.

The first successful update changes:

```text
10 → 2
```

The second transaction then sees:

```text
2 >= 8
```

which is false.

Therefore:

```text
Request A → SUCCESS
Request B → INSUFFICIENT STOCK
```

This is much safer than:

```text
SELECT
application calculation
UPDATE
```

without concurrency protection.

---

# 24. Reservation Repository

## `internal/reservation/repository.go`

```go
package reservation

import (
    "github.com/google/uuid"

    "gorm.io/gorm"
)

type Repository struct {
    db *gorm.DB
}

func NewRepository(
    db *gorm.DB,
) *Repository {

    return &Repository{
        db: db,
    }
}

func (r *Repository) Create(
    tx *gorm.DB,
    reservation *Reservation,
) error {

    return tx.Create(
        reservation,
    ).Error
}

func (r *Repository) FindByID(
    reservationID uuid.UUID,
) (*Reservation, error) {

    var reservation Reservation

    result := r.db.
        Where(
            "reservation_id = ?",
            reservationID,
        ).
        First(&reservation)

    if result.Error != nil {
        return nil, result.Error
    }

    return &reservation, nil
}

func (r *Repository) FindByIdempotencyKey(
    key string,
) (*Reservation, error) {

    var reservation Reservation

    result := r.db.
        Where(
            "idempotency_key = ?",
            key,
        ).
        First(&reservation)

    if result.Error != nil {
        return nil, result.Error
    }

    return &reservation, nil
}

func (r *Repository) UpdateStatus(
    tx *gorm.DB,
    reservationID uuid.UUID,
    status string,
) error {

    return tx.Model(
        &Reservation{},
    ).
        Where(
            "reservation_id = ?",
            reservationID,
        ).
        Update(
            "status",
            status,
        ).Error
}
```

---

# 25. Inventory Service Layer

## `internal/inventory/service.go`

```go
package inventory

import (
    "context"

    "github.com/google/uuid"

    "gorm.io/gorm"

    "github.com/company-ecommerce/inventory-service/internal/reservation"
)

type Service struct {

    db *gorm.DB

    inventoryRepository *Repository

    reservationRepository
        *reservation.Repository
}

func NewService(
    db *gorm.DB,
    inventoryRepository *Repository,
    reservationRepository *reservation.Repository,
) *Service {

    return &Service{
        db: db,

        inventoryRepository:
            inventoryRepository,

        reservationRepository:
            reservationRepository,
    }
}

func (s *Service) Reserve(
    ctx context.Context,
    productID uuid.UUID,
    orderID uuid.UUID,
    quantity int64,
    idempotencyKey string,
) (*reservation.Reservation, error) {

    var result *reservation.Reservation

    err := s.db.WithContext(ctx).
        Transaction(
            func(tx *gorm.DB) error {

                existing, err :=
                    s.reservationRepository.
                        FindByIdempotencyKey(
                            idempotencyKey,
                        )

                if err == nil {

                    result = existing

                    return nil
                }

                if err != gorm.ErrRecordNotFound {
                    return err
                }

                err =
                    s.inventoryRepository.Reserve(
                        tx,
                        productID,
                        quantity,
                    )

                if err != nil {
                    return err
                }

                newReservation :=
                    &reservation.Reservation{
                        ReservationID:
                            uuid.New(),

                        ProductID:
                            productID,

                        OrderID:
                            orderID,

                        Quantity:
                            quantity,

                        Status:
                            "PENDING",

                        IdempotencyKey:
                            idempotencyKey,
                    }

                err =
                    s.reservationRepository.Create(
                        tx,
                        newReservation,
                    )

                if err != nil {
                    return err
                }

                result =
                    newReservation

                return nil
            },
        )

    if err != nil {
        return nil, err
    }

    return result, nil
}
```

The complete implementation should additionally set:

```text
ExpiresAt
```

and publish the reservation event using the transactional outbox pattern.

---

# 26. Why Idempotency Is Required

Suppose Order Service sends:

```text
Reserve 2 units
```

The request succeeds.

But the network response is lost.

Order Service retries:

```text
Reserve 2 units
```

Without idempotency:

```text
First request:
Stock 10 → 8

Retry:
Stock 8 → 6
```

Inventory has been reserved twice.

With:

```text
Idempotency-Key:
order-123-reservation-1
```

the second request returns the existing reservation.

Therefore:

```text
same idempotency key
        ↓
same logical operation
        ↓
do not reserve twice
```

---

# 27. Reservation Handler

## `internal/reservation/handler.go`

```go
package reservation

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"

    inventoryService
        "github.com/company-ecommerce/inventory-service/internal/inventory"
)

type Handler struct {
    service *inventoryService.Service
}

type ReserveRequest struct {

    OrderID uuid.UUID `json:"orderId" binding:"required"`

    Quantity int64 `json:"quantity" binding:"required,gt=0"`
}

func NewHandler(
    service *inventoryService.Service,
) *Handler {

    return &Handler{
        service: service,
    }
}

func (h *Handler) Reserve(
    c *gin.Context,
) {

    productID, err :=
        uuid.Parse(
            c.Param("productId"),
        )

    if err != nil {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error": "invalid product id",
            },
        )

        return
    }

    var request ReserveRequest

    if err :=
        c.ShouldBindJSON(&request);
        err != nil {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error": "invalid request",
            },
        )

        return
    }

    idempotencyKey :=
        c.GetHeader(
            "Idempotency-Key",
        )

    if idempotencyKey == "" {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error":
                    "Idempotency-Key required",
            },
        )

        return
    }

    result, err :=
        h.service.Reserve(
            c.Request.Context(),
            productID,
            request.OrderID,
            request.Quantity,
            idempotencyKey,
        )

    if err != nil {

        c.JSON(
            http.StatusConflict,
            gin.H{
                "error":
                    err.Error(),
            },
        )

        return
    }

    c.JSON(
        http.StatusCreated,
        result,
    )
}
```

---

# 28. Inventory Handler

## `internal/inventory/handler.go`

```go
package inventory

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

type Handler struct {
    repository *Repository
}

func NewHandler(
    repository *Repository,
) *Handler {

    return &Handler{
        repository: repository,
    }
}

func (h *Handler) Get(
    c *gin.Context,
) {

    productID, err :=
        uuid.Parse(
            c.Param("productId"),
        )

    if err != nil {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error":
                    "invalid product id",
            },
        )

        return
    }

    inventory, err :=
        h.repository.Find(
            productID,
        )

    if err != nil {

        c.JSON(
            http.StatusNotFound,
            gin.H{
                "error":
                    "inventory not found",
            },
        )

        return
    }

    c.JSON(
        http.StatusOK,
        inventory,
    )
}
```

---

# 29. Main Application

## `cmd/server/main.go`

```go
package main

import (
    "log"

    "github.com/gin-gonic/gin"

    "github.com/company-ecommerce/inventory-service/internal/config"
    "github.com/company-ecommerce/inventory-service/internal/database"
    "github.com/company-ecommerce/inventory-service/internal/inventory"
    "github.com/company-ecommerce/inventory-service/internal/reservation"
)

func main() {

    cfg := config.Load()

    db, err :=
        database.Connect(cfg)

    if err != nil {

        log.Fatalf(
            "database connection failed: %v",
            err,
        )
    }

    inventoryRepository :=
        inventory.NewRepository(db)

    reservationRepository :=
        reservation.NewRepository(db)

    inventoryService :=
        inventory.NewService(
            db,
            inventoryRepository,
            reservationRepository,
        )

    inventoryHandler :=
        inventory.NewHandler(
            inventoryRepository,
        )

    reservationHandler :=
        reservation.NewHandler(
            inventoryService,
        )

    router := gin.New()

    router.Use(
        gin.Logger(),
        gin.Recovery(),
    )

    router.GET(
        "/health",
        func(c *gin.Context) {

            c.JSON(
                http.StatusOK,
                gin.H{
                    "status": "UP",
                },
            )
        },
    )

    router.GET(
        "/ready",
        func(c *gin.Context) {

            c.JSON(
                http.StatusOK,
                gin.H{
                    "status": "READY",
                },
            )
        },
    )

    api :=
        router.Group(
            "/api/v1",
        )

    api.GET(
        "/inventory/:productId",
        inventoryHandler.Get,
    )

    api.POST(
        "/inventory/:productId/reservations",
        reservationHandler.Reserve,
    )

    log.Printf(
        "starting inventory-service on port %s",
        cfg.Port,
    )

    if err :=
        router.Run(
            ":" + cfg.Port,
        );
        err != nil {

        log.Fatalf(
            "server failed: %v",
            err,
        )
    }
}
```

The import section should include:

```go
"net/http"
```

in the final source file.

---

# 30. Redis

Redis can be used for:

- Availability read caching
- Frequently requested stock views
- Short-lived inventory metadata

Redis must NOT be the authoritative inventory database.

The critical reservation operation always uses PostgreSQL.

---

# 31. Redis Client

## `internal/cache/redis.go`

```go
package cache

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

type Client struct {
    Redis *redis.Client
}

func New(
    address string,
) *Client {

    client :=
        redis.NewClient(
            &redis.Options{

                Addr: address,

                DialTimeout:
                    1 * time.Second,

                ReadTimeout:
                    1 * time.Second,

                WriteTimeout:
                    1 * time.Second,

                PoolSize: 20,
            },
        )

    return &Client{
        Redis: client,
    }
}

func (c *Client) Get(
    ctx context.Context,
    key string,
) (string, error) {

    return c.Redis.Get(
        ctx,
        key,
    ).Result()
}

func (c *Client) Set(
    ctx context.Context,
    key string,
    value string,
    ttl time.Duration,
) error {

    return c.Redis.Set(
        ctx,
        key,
        value,
        ttl,
    ).Err()
}

func (c *Client) Delete(
    ctx context.Context,
    key string,
) error {

    return c.Redis.Del(
        ctx,
        key,
    ).Err()
}
```

---

# 32. Cache Consistency

Inventory caching is more dangerous than product caching because stock changes
frequently.

Example:

```text
Actual DB:
Available = 5

Redis:
Available = 10
```

If customers rely on stale Redis data for reservation decisions, overselling
can occur.

Therefore:

> Redis availability data can be used for informational reads, but the
> authoritative reservation decision must happen in PostgreSQL.

---

# 33. Kafka Events

Important events:

```text
InventoryReserved
InventoryReleased
InventoryConfirmed
InventoryAdjusted
InventoryLowStock
```

Example:

```json
{
  "eventId": "uuid",
  "eventType": "InventoryReserved",
  "version": 1,
  "occurredAt": "2026-10-05T10:00:00Z",
  "source": "inventory-service",
  "correlationId": "order-correlation-id",
  "payload": {
    "reservationId": "uuid",
    "productId": "uuid",
    "orderId": "uuid",
    "quantity": 2
  }
}
```

---

# 34. Inventory Event Consumers

Potential consumers:

```text
InventoryReserved
       |
       ├── order-service
       ├── reporting-service
       ├── notification-service
       └── analytics

InventoryLowStock
       |
       ├── notification-service
       └── reporting-service
```

Consumers must be idempotent.

Kafka delivery is not assumed to be exactly once at the business level.

---

# 35. Transactional Outbox

The production design should use:

```text
PostgreSQL Transaction
       |
       ├── inventory update
       |
       ├── reservation record
       |
       └── outbox event
                  |
                  ↓
           Outbox Publisher
                  |
                  ↓
                Kafka
```

This avoids:

```text
Inventory update SUCCESS
        |
        X
Kafka publish FAILURE
```

The outbox publisher can retry the event.

---

# 36. Dockerfile

## `Dockerfile`

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /build

RUN apk add --no-cache ca-certificates

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -ldflags="-s -w" \
    -o /inventory-service \
    ./cmd/server


FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /

COPY --from=builder \
    /inventory-service \
    /inventory-service

USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT [
    "/inventory-service"
]
```

---

# 37. `.dockerignore`

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

Dockerfile

docker-compose.yaml
```

---

# 38. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: inventory-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: inventory

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
          "pg_isready -U postgres -d inventory"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: inventory-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: inventory-kafka

    restart: unless-stopped

    ports:

      - "9092:9092"


  inventory-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: inventory-service

    restart: unless-stopped

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

      kafka:
        condition: service_started

    environment:

      APP_NAME: inventory-service

      ENVIRONMENT: local

      PORT: 8080

      DATABASE_URL: >-
        postgres://postgres:postgres@postgres:5432/inventory

      REDIS_ADDR: redis:6379

      KAFKA_BROKERS: kafka:9092

      KAFKA_TOPIC: inventory-events

      DB_MAX_OPEN_CONNS: 20

      DB_MAX_IDLE_CONNS: 5

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For local development only, the Kafka image may be replaced with the exact
approved version used by the project.

---

# 39. Local Development

Start dependencies:

```bash
docker compose up -d postgres redis kafka
```

Download dependencies:

```bash
go mod download
```

Run:

```bash
go run ./cmd/server
```

Build:

```bash
go build \
  -o bin/inventory-service \
  ./cmd/server
```

Test:

```bash
go test ./...
```

Race detection:

```bash
go test -race ./...
```

---

# 40. Check Inventory

```bash
curl \
  http://localhost:8080/api/v1/inventory/<product-id>
```

Example response:

```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "availableQuantity": 100,
  "reservedQuantity": 0
}
```

---

# 41. Reserve Inventory

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/inventory/<product-id>/reservations \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-1001-reservation-1" \
  -d '{
    "orderId": "650e8400-e29b-41d4-a716-446655440000",
    "quantity": 2
  }'
```

Expected result:

```text
HTTP 201
```

---

# 42. Duplicate Reservation Test

Send the exact request again:

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/inventory/<product-id>/reservations \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-1001-reservation-1" \
  -d '{
    "orderId": "650e8400-e29b-41d4-a716-446655440000",
    "quantity": 2
  }'
```

The inventory must NOT be reduced again.

---

# 43. Concurrency Test

Initial:

```text
Available = 10
```

Run:

```text
100 concurrent requests
```

Each requests:

```text
quantity = 1
```

Expected:

```text
Successful reservations = 10

Rejected reservations = 90

Available = 0

Reserved = 10
```

The exact test implementation will be added to the performance-testing phase.

This is one of the most important tests in the entire 21-service project.

---

# 44. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: inventory-service

  labels:

    app: inventory-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: inventory-service

  template:

    metadata:

      labels:

        app: inventory-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: inventory-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: inventory-service-config

            - secretRef:

                name: inventory-service-secrets

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

# 45. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: inventory-service

  labels:

    app: inventory-service

spec:

  type: ClusterIP

  selector:

    app: inventory-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal DNS:

```text
inventory-service.ecommerce.svc.cluster.local
```

---

# 46. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: inventory-service-config

data:

  ENVIRONMENT: prod

  PORT: "8080"

  REDIS_ADDR: redis.ecommerce.svc.cluster.local:6379

  KAFKA_TOPIC: inventory-events

  DB_MAX_OPEN_CONNS: "20"

  DB_MAX_IDLE_CONNS: "5"

  DB_CONN_MAX_LIFETIME_SECONDS: "1500"
```

---

# 47. ExternalSecret

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: inventory-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: inventory-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/inventory-service

        property: DATABASE_URL
```

Production secret architecture:

```text
AWS Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
inventory-service
```

---

# 48. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: inventory-service
```

If inventory needs AWS APIs:

```text
Pod
 ↓
EKS Pod Identity
 ↓
Inventory IAM Role
 ↓
Specific AWS APIs
```

No static AWS access keys should be stored in the Pod.

---

# 49. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: inventory-service

spec:

  podSelector:

    matchLabels:

      app: inventory-service

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

# 50. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: inventory-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: inventory-service

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

Important:

Inventory scaling cannot be based only on CPU.

We should eventually evaluate:

```text
CPU
+
Request rate
+
Reservation throughput
+
Database lock wait
+
Kafka lag
```

---

# 51. KEDA

Inventory can also have asynchronous consumers.

For example:

```text
Kafka
  ↓
Inventory Consumer
  ↓
Inventory updates
```

KEDA can scale consumers based on Kafka lag.

Architecture:

```text
Kafka lag
    ↓
KEDA
    ↓
Consumer replicas
    ↓
Karpenter
    ↓
EKS nodes
```

The exact KEDA configuration will be introduced in the platform/event-driven
implementation where reusable patterns are defined.

---

# 52. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: inventory-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: inventory-service
```

---

# 53. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: inventory-service

description: Ecommerce inventory and reservation service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 54. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/inventory-service

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

  name: inventory-service


podDisruptionBudget:

  enabled: true

  minAvailable: 1
```

Inventory has a lower example maximum replica count than read-heavy catalog or
product services because scaling must be evaluated against:

- PostgreSQL
- Lock contention
- Transaction throughput
- Kafka
- Business concurrency

This is an example, not a final production limit.

---

# 55. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/inventory-service

  tag: ""

  digest: ""


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

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

The real values will come from concurrency/load testing.

---

# 56. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: inventory-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: inventory-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 57. Metrics

Important inventory metrics:

```text
inventory_reservation_total
inventory_reservation_success_total
inventory_reservation_failure_total

inventory_release_total
inventory_confirmation_total

inventory_available_quantity

inventory_reservation_latency_seconds

inventory_db_lock_wait_seconds

inventory_idempotency_duplicate_total

inventory_low_stock_total
```

Avoid using:

```text
productId
orderId
customerId
```

as uncontrolled Prometheus labels.

Those can create extremely high cardinality.

---

# 58. Business Metrics

The most important metrics are not simply:

```text
CPU
Memory
```

We also need:

```text
Reservation success rate
Reservation failure rate
Oversell incidents
Inventory adjustment rate
Low-stock events
Reservation latency
Reservation expiry rate
```

---

# 59. Grafana Dashboard

Recommended dashboard:

```text
Inventory Service
│
├── Traffic
│
├── Reservation Rate
│
├── Reservation Success
│
├── Reservation Failure
│
├── Reservation P95/P99
│
├── Database
│   ├── Connections
│   ├── DB Load
│   ├── Lock waits
│   └── Query latency
│
├── Redis
│   ├── Hit ratio
│   └── Errors
│
├── Kafka
│   ├── Producer errors
│   └── Consumer lag
│
├── Kubernetes
│   ├── Pods
│   ├── CPU
│   ├── Memory
│   └── Restarts
│
└── Business
    ├── Low stock
    ├── Reservation failures
    └── Expired reservations
```

---

# 60. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T14:00:00Z",
  "level": "INFO",
  "service": "inventory-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "inventory-service-7c8d9f",
  "version": "1.0.0",
  "correlationId": "abc-123",
  "eventId": "event-123",
  "reservationId": "reservation-123",
  "message": "Inventory reservation completed"
}
```

Never log:

```text
Authorization tokens
Database passwords
Redis passwords
Secrets
Customer PII
Payment credentials
```

---

# 61. OpenTelemetry

Reservation tracing:

```text
order-service
     |
     ↓
POST /inventory/.../reservations
     |
     ↓
inventory-service
     |
     ├── PostgreSQL transaction
     |
     ├── Reservation insert
     |
     └── Kafka event
```

Tracing architecture:

```text
inventory-service
       ↓
OpenTelemetry
       ↓
OTel Collector
       ↓
Jaeger
```

A trace allows us to answer:

> Did the order fail because inventory was slow, PostgreSQL was slow, a
> database lock occurred, or Kafka publication was delayed?

---

# 62. Inventory Concurrency

This is the most important technical section.

Suppose:

```text
Available = 100
```

and:

```text
1,000 customers
```

simultaneously request:

```text
1 unit
```

Expected:

```text
100 successful reservations
900 rejected
```

Not:

```text
1,000 successful
```

and not:

```text
negative inventory
```

---

# 63. PostgreSQL Atomic Update

Core operation:

```sql
UPDATE inventory
SET
    available_quantity =
        available_quantity - 1,

    reserved_quantity =
        reserved_quantity + 1

WHERE
    product_id = '...'
    AND available_quantity >= 1;
```

If:

```text
RowsAffected = 1
```

reservation succeeded.

If:

```text
RowsAffected = 0
```

reservation failed.

This is much safer than performing the check entirely inside application
memory.

---

# 64. Alternative — Row Lock

Another valid approach:

```sql
SELECT *
FROM inventory
WHERE product_id = ?
FOR UPDATE;
```

Then:

```text
Check quantity
      ↓
Update quantity
      ↓
Commit
```

`FOR UPDATE` locks the selected row until the transaction completes.

This can be useful when reservation logic requires multiple dependent database
operations.

However, long transactions can increase lock contention.

Therefore:

> Keep inventory transactions short.

---

# 65. What NOT to Do

Do not implement:

```text
Redis GET stock
    ↓
if stock > requested
    ↓
Redis DECREMENT
    ↓
PostgreSQL later
```

as the authoritative reservation mechanism unless the system has a deliberately
designed atomic inventory architecture.

Redis cache state can become inconsistent with the durable database.

---

# 66. Reservation Expiration

Reservations should have an expiration time.

Example:

```text
Reservation created
       |
       ↓
Expires in 10 minutes
```

If payment is never completed:

```text
Reservation expires
       |
       ↓
Release stock
```

A background worker can process expired reservations:

```text
PostgreSQL
    ↓
Expired reservations
    ↓
Release inventory
    ↓
Publish InventoryReleased
```

For high-scale asynchronous processing, this can be implemented using Kafka,
SQS, scheduled workers, or another controlled mechanism.

---

# 67. Payment Interaction

Example order workflow:

```text
Customer
   |
   ↓
Order Service
   |
   ↓
Reserve Inventory
   |
   ├── FAILURE → Stop order
   |
   ↓
Payment
   |
   ├── FAILURE
   |      ↓
   |   Release Inventory
   |
   ↓
Payment Success
   |
   ↓
Confirm Inventory
   |
   ↓
Order Confirmed
```

This is part of the order Saga.

---

# 68. Idempotency Across Services

Example:

```text
orderId = ORD-1001
reservationId = RES-1001
```

Reservation request:

```text
Idempotency-Key:
ORD-1001-RESERVATION
```

If the request is retried:

```text
same key
    ↓
same reservation
```

No double deduction.

---

# 69. Failure Scenario — Insufficient Stock

Customer requests:

```text
quantity = 5
```

Available:

```text
3
```

Response:

```text
HTTP 409 Conflict
```

Example:

```json
{
  "error": "INSUFFICIENT_STOCK"
}
```

This should not be treated as an infrastructure failure.

It is a legitimate business outcome.

Therefore:

```text
4xx business response
```

should not automatically count as a service outage in the availability SLO.

---

# 70. Failure Scenario — PostgreSQL Lock Contention

Symptoms:

```text
Reservation latency increases
Database CPU increases
Lock wait increases
Hikari pending increases
API latency increases
```

Investigate:

```text
PostgreSQL
    ↓
Locks
    ↓
Long-running transactions
    ↓
Application transaction duration
```

Do not immediately increase Hikari.

Increasing application concurrency against a lock bottleneck can make the
problem worse.

---

# 71. Failure Scenario — Redis Failure

Redis is not the source of truth.

Therefore:

```text
Redis failure
    ↓
Cache unavailable
    ↓
PostgreSQL availability reads
```

Reservation itself still uses PostgreSQL.

However, a Redis failure can increase PostgreSQL read traffic.

Therefore:

```text
Redis failure
    ↓
Cache misses
    ↓
RDS read pressure
```

must be monitored.

---

# 72. Failure Scenario — Kafka Failure

The inventory transaction succeeds:

```text
PostgreSQL
    ↓
SUCCESS
```

Kafka:

```text
FAILURE
```

This is why the outbox pattern is required.

Without an outbox:

```text
Stock reserved
    +
Event missing
```

can create downstream inconsistency.

With an outbox:

```text
Stock reserved
    +
Outbox event stored
    ↓
Publisher retries
    ↓
Kafka
```

---

# 73. Failure Scenario — Duplicate Request

Request:

```text
Order 1001
Quantity 2
```

Network timeout.

Caller retries.

Without idempotency:

```text
100 → 98
98 → 96
```

With idempotency:

```text
100 → 98
retry → existing reservation returned
```

---

# 74. Failure Scenario — Overselling Attempt

Initial:

```text
Available = 1
```

Concurrent requests:

```text
Customer A → 1
Customer B → 1
```

Expected:

```text
A → SUCCESS
B → INSUFFICIENT_STOCK
```

Never:

```text
Available = -1
```

The database constraint also protects against negative quantities.

---

# 75. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Inventory Service CI

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

  ECR_REPOSITORY: inventory-service


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


      - name: Verify dependencies

        run: go mod verify


      - name: Format check

        run: |

          test -z "$(gofmt -l .)"


      - name: Unit tests

        run: go test ./...


      - name: Race detector

        run: go test -race ./...


      - name: Build

        run: |

          CGO_ENABLED=0 \
          GOOS=linux \
          GOARCH=amd64 \
          go build \
          -o inventory-service \
          ./cmd/server


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


      - name: Get image digest

        run: |

          aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="${GITHUB_SHA}" \
            --query 'imageDetails[0].imageDigest' \
            --output text
```

---

# 76. SonarQube

For Go, SonarQube can analyze:

- Bugs
- Vulnerabilities
- Code smells
- Duplication
- Maintainability
- Test coverage

The scanner runs in CI.

SonarQube itself remains a centralized platform service.

---

# 77. Trivy

Pipeline:

```text
Source
  ↓
Trivy filesystem scan
  ↓
Docker build
  ↓
Trivy image scan
  ↓
ECR
```

The application Pod does not need Trivy installed.

---

# 78. ECR

Repository:

```text
company-ecommerce/inventory-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/inventory-service:<git-sha>
```

Deployment should ultimately use an immutable image digest.

---

# 79. GitOps

Application repository:

```text
company-ecommerce/inventory-service
```

GitOps:

```text
company-ecommerce/gitops
```

Example:

```text
gitops/
└── apps/
    └── inventory-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

---

# 80. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: inventory-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/inventory-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true

    syncOptions:

      - CreateNamespace=true
```

---

# 81. Production Incident

## Incident ID

```text
INC-2026-INVENTORY-001
```

## Severity

```text
SEV-1
```

## Title

```text
Inventory Reservation Latency Caused Checkout Failures
```

---

# 82. Customer Impact

Customers experienced:

- Checkout failures
- Inventory reservation timeouts
- Increased order latency

No permanent overselling occurred.

---

# 83. Timeline

```text
15:00
Normal inventory reservation latency.

15:05
Traffic increases because of a promotional event.

15:06
Inventory request rate increases sharply.

15:07
Reservation latency increases.

15:08
PostgreSQL lock wait increases.

15:09
Hikari pending connections increase.

15:10
Order-service begins timing out on inventory calls.

15:11
Checkout failures increase.

15:12
SEV-1 declared.

15:14
Platform team identifies inventory DB lock contention.

15:16
Application team identifies a long-running transaction introduced
by a recent inventory deployment.

15:18
Deployment rollback initiated.

15:21
Old inventory version restored.

15:23
Lock wait decreases.

15:25
Reservation latency normalizes.

15:27
Checkout error rate returns to normal.
```

---

# 84. Root Cause

The new inventory-service version introduced a transaction that remained open
while performing unnecessary processing.

Architecture became:

```text
BEGIN TRANSACTION
      |
      ↓
Database update
      |
      ↓
External processing
      |
      ↓
More processing
      |
      ↓
COMMIT
```

This held database resources longer than necessary.

The correct design is:

```text
Validate
   ↓
Short DB transaction
   ↓
Commit
   ↓
External processing
```

where business semantics allow it.

---

# 85. Amplification

```text
Long transaction
      ↓
Row/lock contention
      ↓
Reservation latency
      ↓
Hikari pending
      ↓
Order-service timeout
      ↓
Retries
      ↓
More inventory requests
      ↓
More contention
```

The retry behavior amplified the incident.

---

# 86. Five Whys

### Why did checkout fail?

Inventory reservation requests timed out.

### Why?

Inventory database transactions were waiting on locks.

### Why?

A deployment introduced long-running transactions.

### Why did that become severe?

Traffic increased and multiple requests contended for the same inventory rows.

### Why did the incident amplify?

Retry behavior generated additional reservation attempts.

---

# 87. Corrective Actions

| Action | Owner |
|---|---|
| Reduce transaction scope | Backend |
| Add lock-wait alerts | DBA/SRE |
| Add reservation latency SLO | SRE |
| Add concurrency load test | Performance |
| Add retry backoff/jitter | Backend |
| Add inventory deployment canary | Platform |
| Add transaction-duration metric | Backend |
| Add lock contention dashboard | DBA/SRE |
| Add flash-sale simulation | Platform |
| Review HPA max | Platform |

---

# 88. Inventory SLO

Initial targets:

## Reservation Availability

```text
99.95%
```

## Reservation P95

```text
< 200ms
```

## Reservation P99

```text
< 500ms
```

## Overselling

```text
0
```

The final targets must be based on measured business requirements.

---

# 89. Important Alerts

Examples:

```text
InventoryReservationFailureRateHigh

InventoryReservationLatencyHigh

InventoryDatabaseLockWaitHigh

InventoryHikariPendingHigh

InventoryKafkaLagHigh

InventoryLowStockHigh

InventoryPodRestartHigh

Inventory5xxHigh

InventorySLOBurnRateHigh
```

---

# 90. Security Checklist

- [ ] No secrets in Git
- [ ] Secrets Manager
- [ ] External Secrets
- [ ] EKS Pod Identity
- [ ] Least-privilege IAM
- [ ] NetworkPolicy
- [ ] Private PostgreSQL
- [ ] Private Redis
- [ ] Secured Kafka
- [ ] Non-root container
- [ ] Read-only filesystem
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] No credentials in logs
- [ ] Immutable ECR images

---

# 91. Production Checklist

## Application

- [ ] Go
- [ ] Gin
- [ ] GORM
- [ ] Validation
- [ ] Error handling
- [ ] Graceful shutdown
- [ ] Unit tests
- [ ] Integration tests
- [ ] Race tests

## Inventory

- [ ] Atomic reservation
- [ ] Idempotency
- [ ] Reservation expiration
- [ ] Release workflow
- [ ] Confirmation workflow
- [ ] Concurrency testing
- [ ] Negative stock prevention
- [ ] Lock monitoring

## Database

- [ ] PostgreSQL
- [ ] RDS
- [ ] Connection pool
- [ ] Connection budget
- [ ] Short transactions
- [ ] Backups
- [ ] PITR
- [ ] Private networking

## Messaging

- [ ] Kafka
- [ ] Event versioning
- [ ] Correlation ID
- [ ] Event ID
- [ ] Idempotent consumers
- [ ] Transactional outbox
- [ ] Consumer lag monitoring

## Kubernetes

- [ ] Deployment
- [ ] Service
- [ ] ConfigMap
- [ ] ExternalSecret
- [ ] ServiceAccount
- [ ] NetworkPolicy
- [ ] HPA
- [ ] PDB
- [ ] Requests
- [ ] Limits
- [ ] Probes

## CI/CD

- [ ] Go modules
- [ ] Unit tests
- [ ] Race detector
- [ ] Lint
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] Docker build
- [ ] Trivy
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] Dev
- [ ] QA
- [ ] Staging
- [ ] Production
- [ ] ArgoCD
- [ ] Image digest
- [ ] Approval
- [ ] Rollback

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] ELK
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Correlation IDs
- [ ] Alerts
- [ ] SLO
- [ ] Runbook
- [ ] RCA

---

# 92. Final Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                   inventory-service
                            |
               ┌────────────┼────────────┐
               │            │            │
               ↓            ↓            ↓
          PostgreSQL      Redis         Kafka
               │                         │
               │                         ├── Order
               │                         ├── Reporting
               │                         └── Notification
               │
               ↓
          Reservations
               |
               ↓
          Inventory State


Metrics
   ↓
Prometheus
   ↓
Grafana

Logs
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
   ↓
Kibana

Traces
   ↓
OpenTelemetry
   ↓
OTel Collector
   ↓
Jaeger
```

---

# 93. DevOps Perspective

Inventory is a good example of why DevOps cannot only focus on:

```text
Docker
Kubernetes
CI/CD
```

The platform engineer must understand the application's operational behavior.

For inventory:

```text
Application
    ↓
Concurrency
    ↓
PostgreSQL locks
    ↓
Connection pool
    ↓
HPA
    ↓
Kubernetes
    ↓
Node capacity
```

A seemingly harmless decision such as:

```text
Increase HPA maxReplicas
```

can actually make the database problem worse.

Therefore production DevOps engineering requires understanding the complete
dependency chain.

---

# 94. Final Service Boundary

```text
Product Service
       |
       | Product identity
       ↓
Inventory Service
       |
       ├── Available Stock
       ├── Reserved Stock
       ├── Reservation
       └── Stock Adjustment
       |
       ↓
Order Service
       |
       ↓
Payment Service
```

Inventory remains the authoritative owner of stock state.

# End of 166 — Inventory Service
```

### Phase 19 progress

```text
161 Auth Service            ✅ Java / Spring Boot / Maven
162 User Service            ✅ Python / FastAPI
163 Customer Service        ✅ TypeScript / Node.js / NestJS
164 Catalog Service         ✅ Go / Gin / Go Modules
165 Product Service         ✅ Java / Spring Boot / Maven
166 Inventory Service       ✅ Go / Gin / PostgreSQL / Kafka

167 Pricing Service         ⏳ Python / FastAPI
168 Promotion Service       ⏳ Node.js / TypeScript / NestJS
169 Cart Service            ⏳ Java / Spring Boot
170 Order Service           ⏳ Java / Spring Boot
171 Payment Service         ⏳ C# / ASP.NET Core
172 Fraud Service           ⏳ Python / FastAPI
173 Shipping Service        ⏳ Node.js / TypeScript / NestJS
174 Delivery Service        ⏳ Go / Gin
175 Notification Service    ⏳ Python / FastAPI
176 Search Service          ⏳ Java / Spring Boot
177 Recommendation Service  ⏳ Python / FastAPI
178 Review Service          ⏳ Ruby / Rails
179 Tax Service             ⏳ C# / ASP.NET Core
180 Invoice Service         ⏳ Node.js / TypeScript / NestJS
181 Reporting Service       ⏳ Python / FastAPI
```

**Next: `167-Pricing-Service.md` — Python + FastAPI, PostgreSQL + Redis, with pricing rules, caching, and high-read scaling.**