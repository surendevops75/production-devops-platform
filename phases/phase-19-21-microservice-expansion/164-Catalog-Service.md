# 164 — Catalog Service

**File:** `164-Catalog-Service.md`

```markdown
# 164 — Catalog Service

## 1. Overview

The `catalog-service` manages the customer-facing product catalog.

It is one of the highest-read services in the e-commerce platform.

Its primary responsibility is to provide:

- Product categories
- Category hierarchy
- Catalog visibility
- Product listing
- Product discovery metadata
- Product availability for catalog browsing
- Catalog filtering metadata
- Catalog search preparation
- Read-optimized catalog responses

The service does NOT own:

- Product pricing
- Inventory quantity
- Orders
- Payments
- Customer profiles
- Authentication

The service is implemented using:

- Go 1.24
- Gin
- GORM
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

# 2. Why Catalog Service Exists

The e-commerce platform has multiple responsibilities that should not be
implemented inside one large application.

For example:

    catalog-service
        |
        +-- Categories
        +-- Catalog visibility
        +-- Product listing
        +-- Catalog metadata

    product-service
        |
        +-- Product details
        +-- SKU
        +-- Product attributes

    pricing-service
        |
        +-- Price
        +-- Discounts
        +-- Dynamic pricing

    inventory-service
        |
        +-- Stock
        +-- Reservation
        +-- Availability

This separation allows each service to scale independently.

Catalog traffic can be much higher than administrative catalog updates.

---

# 3. Business Responsibilities

Catalog service owns:

- Categories
- Subcategories
- Category hierarchy
- Catalog product relationships
- Catalog visibility
- Display ordering
- Catalog metadata
- Active/inactive catalog entries

Example:

    Electronics
        |
        +-- Mobiles
        |     |
        |     +-- Smartphones
        |
        +-- Laptops
        |
        +-- Accessories

The catalog service manages this hierarchy.

---

# 4. What Catalog Service Does NOT Own

It does not become a shared database for other services.

It does NOT own:

    Product price
    Inventory quantity
    Payment information
    Customer information
    Order state
    Authentication credentials

Instead:

    Catalog
       |
       ├── productId
       ↓
    Product Service

    Product
       |
       ↓
    Pricing Service

    Product
       |
       ↓
    Inventory Service

---

# 5. Technology Stack

| Layer | Technology |
|---|---|
| Language | Go |
| Runtime | Go runtime |
| HTTP framework | Gin |
| ORM | GORM |
| Database | PostgreSQL |
| Cache | Redis |
| Messaging | Kafka |
| Dependency management | Go Modules |
| Testing | Go testing |
| Mocking | GoMock/testify where appropriate |
| Linting | golangci-lint |
| Formatting | gofmt |
| Container | Docker |
| Orchestration | Kubernetes/EKS |
| Packaging | Helm |
| Registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| Metrics | Prometheus |
| Logs | ELK |
| Tracing | OpenTelemetry |
| Trace backend | Jaeger |

---

# 6. Why Go for Catalog Service?

Catalog is read-heavy.

Go is a good fit because it provides:

- Low runtime overhead
- Fast startup
- Efficient concurrency
- Strong typing
- Simple deployment model
- Small container images
- Good HTTP performance

The architecture is:

    Go
      ↓
    Gin
      ↓
    GORM
      ↓
    PostgreSQL

With:

    Redis
      ↓
    Read caching

And:

    Kafka
      ↓
    Catalog events

---

# 7. Repository Structure

    catalog-service/
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
    │   ├── category/
    │   │   ├── handler.go
    │   │   ├── service.go
    │   │   ├── repository.go
    │   │   └── model.go
    │   │
    │   ├── catalog/
    │   │   ├── handler.go
    │   │   ├── service.go
    │   │   ├── repository.go
    │   │   └── model.go
    │   │
    │   └── observability/
    │       └── metrics.go
    │
    ├── migrations/
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
    │   └── catalog-service/
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

---

# 8. Go Module

## `go.mod`

```go
module github.com/company-ecommerce/catalog-service

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

The exact versions are examples for this project.

In a real production repository:

- `go.mod` is committed.
- `go.sum` is committed.
- Dependency upgrades happen through PRs.
- CI validates the dependency graph.
- Security scanning is performed before release.

---

# 9. Environment Configuration

## `.env.example`

```text
APP_NAME=catalog-service
ENVIRONMENT=local
PORT=8080

DATABASE_URL=postgres://postgres:postgres@localhost:5432/catalog

REDIS_ADDR=localhost:6379

KAFKA_BROKERS=localhost:9092
KAFKA_TOPIC=catalog-events

DB_MAX_OPEN_CONNS=20
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME_SECONDS=1500
```

Production secrets are NOT stored in this file.

Production configuration comes from:

    AWS Secrets Manager
          ↓
    External Secrets Operator
          ↓
    Kubernetes Secret
          ↓
    Pod

---

# 10. Configuration Code

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
            "catalog-service",
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
            "postgres://postgres:postgres@localhost:5432/catalog",
        ),

        RedisAddr: getEnv(
            "REDIS_ADDR",
            "localhost:6379",
        ),

        KafkaBrokers: brokers,

        KafkaTopic: getEnv(
            "KAFKA_TOPIC",
            "catalog-events",
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

# 11. Database Connection

## `internal/database/postgres.go`

```go
package database

import (
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/company-ecommerce/catalog-service/internal/config"
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

# 12. Go Connection Pooling

Go uses the standard `database/sql` connection pool underneath GORM.

Important settings:

```text
SetMaxOpenConns
SetMaxIdleConns
SetConnMaxLifetime
SetConnMaxIdleTime
```

The capacity calculation is approximately:

    Pods × MaxOpenConns

Example:

    10 Pods
       ×
    20 max open connections
       =
    200 potential connections

This must be included in our RDS connection budget.

If Kubernetes performs a rolling deployment:

    Old Pods
        +
    New Pods

can temporarily increase the number of database connections.

Therefore database capacity planning must include:

- HPA
- RollingUpdate maxSurge
- Canary
- Blue/green deployments
- Background jobs
- Administrative connections
- Other microservices

---

# 13. Category Model

## `internal/category/model.go`

```go
package category

import (
    "time"

    "github.com/google/uuid"
)

type Category struct {

    ID uint `gorm:"primaryKey"`

    CategoryID uuid.UUID `gorm:"type:uuid;uniqueIndex;not null"`

    ParentID *uuid.UUID `gorm:"type:uuid;index"`

    Name string `gorm:"size:150;not null"`

    Slug string `gorm:"size:180;uniqueIndex;not null"`

    Description string `gorm:"type:text"`

    DisplayOrder int `gorm:"not null;default:0"`

    Active bool `gorm:"not null;default:true"`

    CreatedAt time.Time

    UpdatedAt time.Time
}
```

---

# 14. Catalog Product Model

## `internal/catalog/model.go`

```go
package catalog

import (
    "time"

    "github.com/google/uuid"
)

type CatalogProduct struct {

    ID uint `gorm:"primaryKey"`

    CatalogProductID uuid.UUID `gorm:"type:uuid;uniqueIndex;not null"`

    ProductID uuid.UUID `gorm:"type:uuid;uniqueIndex;not null"`

    CategoryID uuid.UUID `gorm:"type:uuid;index;not null"`

    DisplayOrder int `gorm:"not null;default:0"`

    Featured bool `gorm:"not null;default:false"`

    Visible bool `gorm:"not null;default:true"`

    CreatedAt time.Time

    UpdatedAt time.Time
}
```

The actual product details remain owned by `product-service`.

---

# 15. Category Repository

## `internal/category/repository.go`

```go
package category

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

func (r *Repository) FindByID(
    id uuid.UUID,
) (*Category, error) {

    var category Category

    result := r.db.
        Where(
            "category_id = ? AND active = ?",
            id,
            true,
        ).
        First(&category)

    if result.Error != nil {
        return nil, result.Error
    }

    return &category, nil
}

func (r *Repository) List(
) ([]Category, error) {

    var categories []Category

    result := r.db.
        Where("active = ?", true).
        Order("display_order ASC").
        Find(&categories)

    if result.Error != nil {
        return nil, result.Error
    }

    return categories, nil
}
```

---

# 16. Catalog Repository

## `internal/catalog/repository.go`

```go
package catalog

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

func (r *Repository) FindByCategory(
    categoryID uuid.UUID,
) ([]CatalogProduct, error) {

    var products []CatalogProduct

    result := r.db.
        Where(
            "category_id = ? AND visible = ?",
            categoryID,
            true,
        ).
        Order("display_order ASC").
        Find(&products)

    if result.Error != nil {
        return nil, result.Error
    }

    return products, nil
}
```

---

# 17. Redis Client

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

    client := redis.NewClient(
        &redis.Options{
            Addr: address,

            DialTimeout: 1 * time.Second,

            ReadTimeout: 1 * time.Second,

            WriteTimeout: 1 * time.Second,

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

Redis is a cache.

PostgreSQL remains the durable source of truth.

---

# 18. Catalog Service Layer

## `internal/catalog/service.go`

```go
package catalog

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/google/uuid"

    "github.com/company-ecommerce/catalog-service/internal/cache"
)

type Service struct {
    repository *Repository
    cache      *cache.Client
}

func NewService(
    repository *Repository,
    cacheClient *cache.Client,
) *Service {

    return &Service{
        repository: repository,
        cache:      cacheClient,
    }
}

func (s *Service) ProductsByCategory(
    ctx context.Context,
    categoryID uuid.UUID,
) ([]CatalogProduct, error) {

    key := fmt.Sprintf(
        "catalog:category:%s",
        categoryID.String(),
    )

    cached, err := s.cache.Get(
        ctx,
        key,
    )

    if err == nil {

        var products []CatalogProduct

        if json.Unmarshal(
            []byte(cached),
            &products,
        ) == nil {

            return products, nil
        }
    }

    products, err :=
        s.repository.FindByCategory(
            categoryID,
        )

    if err != nil {
        return nil, err
    }

    payload, err :=
        json.Marshal(products)

    if err == nil {

        _ = s.cache.Set(
            ctx,
            key,
            string(payload),
            5*time.Minute,
        )
    }

    return products, nil
}
```

Important:

Cache failures should normally degrade the cache path rather than making
the catalog unavailable.

However, cache errors must still be exposed through metrics/logs.

---

# 19. Category Service

## `internal/category/service.go`

```go
package category

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/google/uuid"

    "github.com/company-ecommerce/catalog-service/internal/cache"
)

type Service struct {
    repository *Repository
    cache      *cache.Client
}

func NewService(
    repository *Repository,
    cacheClient *cache.Client,
) *Service {

    return &Service{
        repository: repository,
        cache:      cacheClient,
    }
}

func (s *Service) Get(
    ctx context.Context,
    id uuid.UUID,
) (*Category, error) {

    key := fmt.Sprintf(
        "category:%s",
        id.String(),
    )

    cached, err := s.cache.Get(
        ctx,
        key,
    )

    if err == nil {

        var category Category

        if json.Unmarshal(
            []byte(cached),
            &category,
        ) == nil {

            return &category, nil
        }
    }

    category, err :=
        s.repository.FindByID(id)

    if err != nil {
        return nil, err
    }

    payload, err :=
        json.Marshal(category)

    if err == nil {

        _ = s.cache.Set(
            ctx,
            key,
            string(payload),
            10*time.Minute,
        )
    }

    return category, nil
}
```

---

# 20. HTTP Handler

## `internal/catalog/handler.go`

```go
package catalog

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

type Handler struct {
    service *Service
}

func NewHandler(
    service *Service,
) *Handler {

    return &Handler{
        service: service,
    }
}

func (h *Handler) ProductsByCategory(
    c *gin.Context,
) {

    categoryID, err :=
        uuid.Parse(
            c.Param("categoryId"),
        )

    if err != nil {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error": "invalid category id",
            },
        )

        return
    }

    products, err :=
        h.service.ProductsByCategory(
            c.Request.Context(),
            categoryID,
        )

    if err != nil {

        c.JSON(
            http.StatusInternalServerError,
            gin.H{
                "error": "unable to load catalog",
            },
        )

        return
    }

    c.JSON(
        http.StatusOK,
        products,
    )
}
```

---

# 21. Category Handler

## `internal/category/handler.go`

```go
package category

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

type Handler struct {
    service *Service
}

func NewHandler(
    service *Service,
) *Handler {

    return &Handler{
        service: service,
    }
}

func (h *Handler) Get(
    c *gin.Context,
) {

    categoryID, err :=
        uuid.Parse(
            c.Param("categoryId"),
        )

    if err != nil {

        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error": "invalid category id",
            },
        )

        return
    }

    category, err :=
        h.service.Get(
            c.Request.Context(),
            categoryID,
        )

    if err != nil {

        c.JSON(
            http.StatusNotFound,
            gin.H{
                "error": "category not found",
            },
        )

        return
    }

    c.JSON(
        http.StatusOK,
        category,
    )
}
```

---

# 22. Application Entry Point

## `cmd/server/main.go`

```go
package main

import (
    "log"

    "github.com/gin-gonic/gin"

    "github.com/company-ecommerce/catalog-service/internal/cache"
    "github.com/company-ecommerce/catalog-service/internal/category"
    "github.com/company-ecommerce/catalog-service/internal/catalog"
    "github.com/company-ecommerce/catalog-service/internal/config"
    "github.com/company-ecommerce/catalog-service/internal/database"
)

func main() {

    cfg := config.Load()

    db, err := database.Connect(cfg)

    if err != nil {
        log.Fatalf(
            "database connection failed: %v",
            err,
        )
    }

    redisClient :=
        cache.New(cfg.RedisAddr)

    categoryRepository :=
        category.NewRepository(db)

    categoryService :=
        category.NewService(
            categoryRepository,
            redisClient,
        )

    categoryHandler :=
        category.NewHandler(
            categoryService,
        )

    catalogRepository :=
        catalog.NewRepository(db)

    catalogService :=
        catalog.NewService(
            catalogRepository,
            redisClient,
        )

    catalogHandler :=
        catalog.NewHandler(
            catalogService,
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
                200,
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
                200,
                gin.H{
                    "status": "READY",
                },
            )
        },
    )

    api := router.Group(
        "/api/v1",
    )

    api.GET(
        "/categories/:categoryId",
        categoryHandler.Get,
    )

    api.GET(
        "/categories/:categoryId/products",
        catalogHandler.ProductsByCategory,
    )

    log.Printf(
        "starting catalog-service on port %s",
        cfg.Port,
    )

    if err := router.Run(
        ":" + cfg.Port,
    ); err != nil {

        log.Fatalf(
            "server failed: %v",
            err,
        )
    }
}
```

---

# 23. API Examples

## Get Category

```bash
curl \
  http://localhost:8080/api/v1/categories/<category-id>
```

## Get Products by Category

```bash
curl \
  http://localhost:8080/api/v1/categories/<category-id>/products
```

## Health

```bash
curl \
  http://localhost:8080/health
```

## Readiness

```bash
curl \
  http://localhost:8080/ready
```

---

# 24. Database Migration

Example:

## `migrations/V1__create_catalog_tables.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,

    category_id UUID NOT NULL UNIQUE
        DEFAULT uuid_generate_v4(),

    parent_id UUID,

    name VARCHAR(150) NOT NULL,

    slug VARCHAR(180) NOT NULL UNIQUE,

    description TEXT,

    display_order INTEGER NOT NULL DEFAULT 0,

    active BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_categories_parent_id
ON categories(parent_id);

CREATE INDEX idx_categories_active
ON categories(active);


CREATE TABLE catalog_products (
    id BIGSERIAL PRIMARY KEY,

    catalog_product_id UUID NOT NULL UNIQUE
        DEFAULT uuid_generate_v4(),

    product_id UUID NOT NULL UNIQUE,

    category_id UUID NOT NULL,

    display_order INTEGER NOT NULL DEFAULT 0,

    featured BOOLEAN NOT NULL DEFAULT FALSE,

    visible BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_catalog_products_category
ON catalog_products(category_id);

CREATE INDEX idx_catalog_products_visible
ON catalog_products(visible);
```

Production migrations follow the project's established controlled migration
process.

---

# 25. Makefile

## `Makefile`

```makefile
.PHONY: run build test lint format docker

run:
	go run ./cmd/server

build:
	go build -o bin/catalog-service ./cmd/server

test:
	go test ./...

lint:
	golangci-lint run

format:
	gofmt -w .

docker:
	docker build -t catalog-service:local .
```

---

# 26. Dockerfile

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
    -o /catalog-service \
    ./cmd/server


FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /

COPY --from=builder \
    /catalog-service \
    /catalog-service

USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/catalog-service"]
```

The runtime image is intentionally minimal.

There is no shell, package manager or compiler in the production image.

---

# 27. `.dockerignore`

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

# 28. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: catalog-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: catalog

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
          "pg_isready -U postgres -d catalog"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: catalog-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: catalog-kafka

    restart: unless-stopped

    ports:

      - "9092:9092"


  catalog-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: catalog-service

    restart: unless-stopped

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

      kafka:
        condition: service_started

    environment:

      APP_NAME: catalog-service

      ENVIRONMENT: local

      PORT: 8080

      DATABASE_URL: >-
        postgres://postgres:postgres@postgres:5432/catalog

      REDIS_ADDR: redis:6379

      KAFKA_BROKERS: kafka:9092

      KAFKA_TOPIC: catalog-events

      DB_MAX_OPEN_CONNS: 20

      DB_MAX_IDLE_CONNS: 5

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

Again, the floating Kafka tag is only for local experimentation.

Production uses approved immutable versions.

---

# 29. Local Development

Install dependencies:

```bash
go mod download
```

Format:

```bash
gofmt -w .
```

Run:

```bash
go run ./cmd/server
```

Build:

```bash
go build \
  -o bin/catalog-service \
  ./cmd/server
```

Run tests:

```bash
go test ./...
```

Run with race detector:

```bash
go test -race ./...
```

---

# 30. Unit Test

Example:

## `internal/catalog/service_test.go`

```go
package catalog

import (
    "testing"
)

func TestCatalogServiceExists(
    t *testing.T,
) {

    if NewService == nil {

        t.Fatal(
            "catalog service constructor missing",
        )
    }
}
```

In the actual project, repository dependencies should be mocked and service
behavior should be tested independently from PostgreSQL and Redis.

---

# 31. Integration Testing

The CI pipeline should eventually test:

```text
Catalog API
    |
    ├── PostgreSQL
    |
    └── Redis
```

For integration tests we should use isolated test infrastructure rather than
connecting to production databases.

Recommended flow:

```text
GitHub Actions
      |
      ├── PostgreSQL test container
      |
      ├── Redis test container
      |
      └── Catalog service
```

---

# 32. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: catalog-service

description: Ecommerce catalog service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 33. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/catalog-service

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

  maxReplicas: 20

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: catalog-service


podDisruptionBudget:

  enabled: true

  minAvailable: 1


secretStore:

  name: ecommerce-secrets
```

Catalog is a read-heavy service, so its maximum replica count may eventually
be higher than lower-traffic internal services.

The actual value must be established through performance testing.

---

# 34. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 4

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/catalog-service

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

  minReplicas: 4

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 3
```

These are starting simulation values, not claimed production capacity.

---

# 35. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: catalog-service

  labels:

    app: catalog-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: catalog-service

  template:

    metadata:

      labels:

        app: catalog-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: catalog-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: catalog-service-config

            - secretRef:

                name: catalog-service-secrets

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

# 36. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: catalog-service

  labels:

    app: catalog-service

spec:

  type: ClusterIP

  selector:

    app: catalog-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal DNS:

```text
catalog-service.ecommerce.svc.cluster.local
```

---

# 37. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: catalog-service-config

data:

  ENVIRONMENT: prod

  PORT: "8080"

  REDIS_ADDR: redis.ecommerce.svc.cluster.local:6379

  KAFKA_TOPIC: catalog-events

  DB_MAX_OPEN_CONNS: "20"

  DB_MAX_IDLE_CONNS: "5"

  DB_CONN_MAX_LIFETIME_SECONDS: "1500"
```

In the actual production environment, the Redis/Kafka endpoints should match
the selected managed or self-hosted architecture.

---

# 38. ExternalSecret

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: catalog-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: {{ .Values.secretStore.name }}

    kind: ClusterSecretStore

  target:

    name: catalog-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/catalog-service

        property: DATABASE_URL
```

Do not put:

```text
DATABASE_URL
DB_PASSWORD
AWS_SECRET_ACCESS_KEY
```

directly into Git.

---

# 39. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: catalog-service
```

If the service requires AWS APIs:

```text
Catalog Pod
     |
     ↓
EKS Pod Identity
     |
     ↓
Catalog IAM Role
     |
     ↓
Specific AWS API
```

Only the APIs actually required by the service should be permitted.

---

# 40. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: catalog-service

spec:

  podSelector:

    matchLabels:

      app: catalog-service

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

The policy should be tightened to the actual network architecture before
production.

---

# 41. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: catalog-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: catalog-service

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

For this service, CPU alone may eventually be insufficient.

We may later use:

- Request rate
- In-flight requests
- P95 latency
- Custom Prometheus metrics

as scaling signals after testing.

---

# 42. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: catalog-service

spec:

  minAvailable: 3

  selector:

    matchLabels:

      app: catalog-service
```

---

# 43. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: catalog-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: catalog-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

The `/metrics` implementation should expose Prometheus-compatible metrics in
the final production version.

---

# 44. GitHub CODEOWNERS

## `.github/CODEOWNERS`

```text
/internal/   @company-ecommerce/backend-team
/cmd/        @company-ecommerce/backend-team
/helm/       @company-ecommerce/platform-team
/.github/    @company-ecommerce/platform-team
/security/   @company-ecommerce/security-team
```

---

# 45. GitHub Actions CI

## `.github/workflows/ci.yaml`

```yaml
name: Catalog Service CI

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

  ECR_REPOSITORY: catalog-service


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
          -o catalog-service \
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

# 46. SonarQube

Catalog source should be analyzed for:

- Bugs
- Vulnerabilities
- Code smells
- Maintainability
- Duplicated code
- Test coverage

Example project configuration:

```text
sonar.projectKey=catalog-service
sonar.projectName=catalog-service
sonar.sources=.
```

The exact Sonar scanner configuration should match the organization's
approved SonarQube version and Go analysis configuration.

---

# 47. Trivy

We scan:

    Source repository
          ↓
    Trivy filesystem scan

and:

    Docker image
          ↓
    Trivy image scan

Additionally:

    ECR
      ↓
    Registry vulnerability scanning

This gives multiple security checkpoints.

---

# 48. ECR

Repository:

    company-ecommerce/catalog-service

Example:

    123456789012.dkr.ecr.ap-south-1.amazonaws.com/catalog-service:<git-sha>

Preferred production reference:

    catalog-service@sha256:<digest>

The image should be immutable.

---

# 49. GitOps

Application repository:

    company-ecommerce/catalog-service

GitOps repository:

    company-ecommerce/gitops

Example:

    gitops/
    └── apps/
        └── catalog-service/
            ├── dev/
            ├── qa/
            ├── staging/
            └── prod/

Promotion:

    Source
      ↓
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

---

# 50. ArgoCD

Example:

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: catalog-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/catalog-service/prod

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

Production approval behavior must follow our change-management workflow.

---

# 51. Observability

Catalog is a high-read service, so important metrics include:

    Request rate
    Request latency
    HTTP 4xx
    HTTP 5xx
    Cache hit ratio
    Cache miss ratio
    Redis latency
    PostgreSQL query latency
    PostgreSQL connections
    Go CPU
    Go memory
    Goroutines
    GC behavior
    Pod restarts
    HPA replicas

---

# 52. Business Metrics

Examples:

```text
catalog_category_view_total
catalog_product_list_total
catalog_cache_hit_total
catalog_cache_miss_total
catalog_product_not_found_total
catalog_request_error_total
```

Do not create Prometheus labels using:

```text
customerId
email
phone
productId
```

unless cardinality has been deliberately designed and justified.

For most request-level metrics, use low-cardinality labels such as:

```text
service
method
route
status
environment
```

---

# 53. Grafana Dashboard

Recommended dashboard:

    Catalog Service

    ├── Traffic
    │   ├── RPS
    │   └── Request distribution
    │
    ├── Latency
    │   ├── P50
    │   ├── P95
    │   └── P99
    │
    ├── Errors
    │   ├── 4xx
    │   └── 5xx
    │
    ├── Redis
    │   ├── Hit ratio
    │   ├── Miss ratio
    │   ├── Latency
    │   └── Errors
    │
    ├── PostgreSQL
    │   ├── Connections
    │   ├── Query latency
    │   └── DB load
    │
    ├── Kubernetes
    │   ├── Pods
    │   ├── CPU
    │   ├── Memory
    │   └── Restarts
    │
    └── HPA
        ├── Desired replicas
        ├── Current replicas
        └── Scaling events

---

# 54. Logging

Logs should be structured JSON.

Example:

```json
{
  "@timestamp": "2026-10-05T12:30:00Z",
  "level": "INFO",
  "service": "catalog-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "catalog-service-7f9f8d-x7abc",
  "version": "1.0.0",
  "correlationId": "abc-123",
  "route": "/api/v1/categories/{categoryId}/products",
  "message": "Catalog products retrieved"
}
```

Do not log:

    Authorization headers
    JWTs
    Passwords
    Database credentials
    Redis credentials
    Customer PII

---

# 55. OpenTelemetry

Architecture:

    Client
       |
       ↓
    ALB
       |
       ↓
    catalog-service
       |
       ├── Redis
       |
       ├── PostgreSQL
       |
       └── Kafka
              |
              ↓
             Events

Telemetry:

    catalog-service
          ↓
    OpenTelemetry
          ↓
    OTel Collector
          ↓
        Jaeger

A trace should allow us to identify whether latency came from:

    HTTP handler
        ↓
    Redis
        ↓
    PostgreSQL
        ↓
    Kafka

---

# 56. SLO

Initial simulation targets:

## Availability

    99.9%

## Catalog read P95

    < 150ms

## Catalog read P99

    < 300ms

## 5xx rate

    < 0.1%

These values are starting targets.

They must be validated using actual load tests.

---

# 57. Why Catalog Needs Redis

Catalog traffic can be much larger than catalog write traffic.

Example:

    1,000 catalog requests/sec

If every request reaches PostgreSQL:

    1,000 requests/sec
          ↓
    PostgreSQL

This can become unnecessary database load.

With caching:

    1,000 requests/sec
          ↓
        Redis
          |
          ├── 950 HIT
          |
          └── 50 MISS
                ↓
             PostgreSQL

Illustrative only.

The actual hit ratio must be measured.

---

# 58. Cache Stampede Risk

Suppose:

    Cache key expires

At the same time:

    5,000 requests
          ↓
    Cache MISS
          ↓
    5,000 DB requests

This can overload PostgreSQL.

Protection mechanisms include:

- Request coalescing
- Distributed locking
- Jittered TTL
- Cache warming
- Stale-while-revalidate
- Bounded DB concurrency
- Rate limiting

---

# 59. Catalog Cache Strategy

Example key:

    catalog:category:<categoryId>

TTL:

    5 minutes

Category key:

    category:<categoryId>

TTL:

    10 minutes

The exact TTL depends on:

    Update frequency
    Traffic
    Freshness requirement
    Database capacity
    Business requirements

---

# 60. Catalog Update Flow

Example:

    Admin
      |
      ↓
    Catalog API
      |
      ↓
    PostgreSQL
      |
      ↓
    Cache invalidation
      |
      ↓
    Kafka event
      |
      ↓
    Downstream consumers

Important principle:

Database is the source of truth.

Redis is not.

---

# 61. Failure Scenario — PostgreSQL

Symptoms:

    Catalog API latency increases
    5xx increases
    Redis misses fail
    Database connection errors

Investigation:

```bash
kubectl get pods -n ecommerce \
  -l app=catalog-service
```

Then:

```bash
kubectl logs -n ecommerce <pod>
```

Check:

    RDS status
    RDS connections
    Query latency
    Security groups
    NetworkPolicy
    DNS
    Credentials
    Go DB pool
    Recent deployment

Do not blindly increase DB connections.

---

# 62. Failure Scenario — Redis

Normal:

    Catalog request
         ↓
       Redis
         ↓
        HIT

Failure:

    Catalog request
         ↓
       Redis
         X
         ↓
      PostgreSQL

If thousands of requests simultaneously fall back to PostgreSQL:

    Redis failure
        ↓
    Cache miss storm
        ↓
    RDS overload

Therefore Redis failure must be treated as a dependency failure with
database-protection mechanisms.

---

# 63. Failure Scenario — High Traffic

Suppose:

    Normal = 500 RPS
    Peak = 3,000 RPS

The service may scale:

    5 Pods
       ↓
    10 Pods
       ↓
    20 Pods

But scaling Pods does not solve every bottleneck.

We must check:

    Pods
      ↓
    CPU
      ↓
    Redis
      ↓
    PostgreSQL
      ↓
    Network
      ↓
    ALB

Scaling the application beyond database capacity can make the incident worse.

---

# 64. HPA and Database Interaction

Example:

    Catalog Pods = 10

    DB MaxOpenConns = 20

Potential:

    10 × 20
      =
    200 connections

If HPA increases:

    10 → 30 Pods

Then:

    30 × 20
      =
    600 connections

If RDS cannot safely handle this:

    HPA scaling
        ↓
    RDS connection pressure
        ↓
    latency
        ↓
    errors

Therefore:

    HPA max replicas
        +
    DB connection budget

must be designed together.

---

# 65. Example Production Incident

## Incident ID

    INC-2026-CATALOG-001

## Severity

    SEV-2

## Title

    Catalog API Latency Increased During Redis Degradation

---

# 66. Customer Impact

Customers experienced:

    Slow product/category pages
    Increased catalog latency
    Some HTTP 5xx responses

Order processing remained available.

---

# 67. Timeline

    16:00
    Catalog normal.

    16:05
    Redis latency increases.

    16:06
    Cache hit ratio falls from 96% to 35%.

    16:07
    PostgreSQL read traffic increases.

    16:08
    RDS query latency increases.

    16:09
    Catalog P95 increases from 120ms to 900ms.

    16:10
    HPA increases catalog Pods.

    16:11
    Database connection count increases further.

    16:12
    HTTP 5xx begins increasing.

    16:13
    SEV-2 declared.

    16:15
    Platform team confirms Redis degradation.

    16:17
    Application team reduces DB fallback concurrency.

    16:19
    Catalog cache warming begins.

    16:21
    Redis recovers.

    16:23
    Cache hit ratio returns above 90%.

    16:25
    RDS load decreases.

    16:27
    Catalog latency returns to normal.

---

# 68. Root Cause

Primary failure:

    Redis degradation.

Amplification:

    Redis degradation
         ↓
    Cache misses
         ↓
    PostgreSQL traffic
         ↓
    RDS pressure
         ↓
    HPA scaling
         ↓
    Additional DB connections
         ↓
    Catalog latency

---

# 69. Five Whys

### Why was catalog latency high?

PostgreSQL queries became slower.

### Why?

Database read traffic increased.

### Why?

Redis cache hit ratio dropped.

### Why?

Redis experienced elevated latency.

### Why did Redis degradation become a customer-impacting incident?

The fallback path allowed too many simultaneous database requests.

---

# 70. Corrective Actions

| Action | Owner |
|---|---|
| Implement request coalescing | Backend |
| Add fallback concurrency limit | Backend |
| Improve Redis alerts | SRE |
| Add cache stampede test | Performance |
| Add RDS connection alert | SRE |
| Review catalog HPA max | Platform |
| Add cache hit SLO | SRE |
| Add Redis failure game day | Platform |
| Add catalog runbook | SRE |

---

# 71. Production Troubleshooting

## Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=catalog-service
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
  catalog-service
```

## Deployment

```bash
kubectl get deployment \
  -n ecommerce \
  catalog-service
```

## Endpoints

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=catalog-service
```

## Events

```bash
kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

## Resource usage

```bash
kubectl top pods \
  -n ecommerce
```

---

# 72. Security Checklist

- [ ] No secrets in Git
- [ ] No production `.env`
- [ ] Least-privilege IAM
- [ ] EKS Pod Identity when AWS access is needed
- [ ] NetworkPolicy
- [ ] Non-root container
- [ ] Minimal runtime image
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] PostgreSQL private
- [ ] Redis private
- [ ] TLS for production dependencies
- [ ] Structured logging
- [ ] No credentials in logs
- [ ] API validation
- [ ] Immutable ECR images

---

# 73. Production Checklist

## Application

- [ ] Go
- [ ] Gin
- [ ] GORM
- [ ] Error handling
- [ ] Input validation
- [ ] Unit tests
- [ ] Integration tests
- [ ] Health endpoint
- [ ] Readiness endpoint
- [ ] Graceful shutdown

## Database

- [ ] PostgreSQL
- [ ] Controlled migrations
- [ ] Connection pool
- [ ] Connection budget
- [ ] RDS backups
- [ ] PITR
- [ ] Private networking

## Redis

- [ ] Cache-aside
- [ ] TTL
- [ ] Cache invalidation
- [ ] Short timeouts
- [ ] Bounded fallback
- [ ] Stampede protection
- [ ] Cache warming

## Kafka

- [ ] Event schema
- [ ] Event ID
- [ ] Correlation ID
- [ ] Version
- [ ] Idempotency
- [ ] Outbox where required

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
- [ ] Startup probe
- [ ] Readiness probe
- [ ] Liveness probe

## CI/CD

- [ ] Go modules
- [ ] gofmt
- [ ] Unit tests
- [ ] Race detector
- [ ] golangci-lint
- [ ] SonarQube
- [ ] Dependency scan
- [ ] Secret scan
- [ ] Docker build
- [ ] Trivy
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] Environment values
- [ ] ArgoCD
- [ ] Immutable image
- [ ] Image digest
- [ ] Approval
- [ ] Rollback

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

# 74. Final Architecture

                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                     catalog-service
                            |
              ┌─────────────┼─────────────┐
              │             │             │
              ↓             ↓             ↓
         PostgreSQL       Redis         Kafka
              │             │             │
              │             │             │
              └─────────────┼─────────────┘
                            |
                      Catalog Domain
                            |
               ┌────────────┴────────────┐
               │                         │
          Prometheus              OpenTelemetry
               │                         │
            Grafana                  Collector
                                         |
                                       Jaeger


Application Logs
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

---

# 75. DevOps Learning

We now have three different application stacks:

    161 Auth Service

        Java
          ↓
        Spring Boot
          ↓
        Maven


    162 User Service

        Python
          ↓
        FastAPI
          ↓
        uv


    163 Customer Service

        TypeScript
          ↓
        Node.js
          ↓
        NestJS
          ↓
        pnpm


    164 Catalog Service

        Go
          ↓
        Gin
          ↓
        Go Modules

But the platform remains standardized:

    Source
      ↓
    GitHub
      ↓
    CI
      ↓
    SonarQube
      ↓
    Trivy
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
      ↓
    Prometheus
      ↓
    Grafana
      ↓
    ELK
      ↓
    OpenTelemetry
      ↓
    Jaeger

This is the key platform-engineering principle:

Different application languages do NOT require a completely different
production platform.

The application build layer changes.

The platform lifecycle remains standardized.

# End of 164 — Catalog Service
```

### Phase 19 progress

```text
161 Auth Service            ✅ Java / Spring Boot
162 User Service            ✅ Python / FastAPI
163 Customer Service        ✅ TypeScript / Node.js / NestJS
164 Catalog Service         ✅ Go / Gin

165 Product Service         ⏳ Java / Spring Boot
166 Inventory Service       ⏳ Go / Gin
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

**Next: `165-Product-Service.md` — Java + Spring Boot + Maven.**