# 176 — Search Service

**File:** `176-Search-Service.md`

```markdown
# 176 — Search Service

## 1. Overview

The `search-service` provides fast product discovery for the e-commerce
platform.

It is responsible for:

- Product search
- Full-text search
- Autocomplete
- Suggestions
- Category filtering
- Brand filtering
- Price filtering
- Attribute filtering
- Sorting
- Pagination
- Search relevance
- Search indexing
- Index synchronization
- Reindexing
- Search aliases
- Index versioning
- Search analytics

The service does **not** own product business data.

The source of truth remains the Product Service and its database.

Technology:

- Go 1.24+
- Gin
- OpenSearch
- Redis
- Kafka
- PostgreSQL only where needed for search metadata/analytics
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

# 2. Why We Need a Search Service

A product database is optimized for transactional operations.

For example:

```text
Product Service
      |
      ↓
PostgreSQL
```

is excellent for:

```text
GET product by ID
UPDATE product
INSERT product
DELETE product
```

But customer search can require:

```text
"wireless bluetooth headphones"
```

with:

```text
fuzzy matching
relevance
autocomplete
brand filters
category filters
price ranges
sorting
facets
```

A relational database can perform some of these operations, but a dedicated
search engine is better suited for large-scale product discovery.

Architecture:

```text
                         Product Service
                               |
                               ↓
                         Product Database
                               |
                               ↓
                         Product Events
                               |
                               ↓
                             Kafka
                               |
                               ↓
                        Search Service
                               |
                               ↓
                           OpenSearch
                               |
                               ↓
                            Customer
```

---

# 3. Source of Truth

Important principle:

```text
PostgreSQL = business source of truth
OpenSearch  = searchable projection
```

Never treat OpenSearch as the authoritative product database.

Example:

```text
Product Service
      |
      ↓
PostgreSQL
      |
      ↓
ProductUpdated
      |
      ↓
Kafka
      |
      ↓
Search Indexer
      |
      ↓
OpenSearch
```

If OpenSearch is lost:

```text
Product data is NOT lost.
```

We can rebuild the index from Product Service data/events.

---

# 4. Search Architecture

```text
                         ┌──────────────────┐
                         │   Product DB     │
                         └────────┬─────────┘
                                  │
                                  ↓
                         Product Service
                                  │
                                  ↓
                         Product Events
                                  │
                                  ↓
                                Kafka
                                  │
                    ┌─────────────┴─────────────┐
                    ↓                           ↓
             Search Indexer              Other Consumers
                    |
                    ↓
              OpenSearch
                    |
             ┌──────┴──────┐
             ↓             ↓
        Search API     Autocomplete
             |
             ↓
          Customer
```

---

# 5. Search Request Flow

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
Search Service
   |
   ↓
OpenSearch
   |
   ↓
Search Results
```

The customer should not connect directly to OpenSearch.

OpenSearch remains a private backend dependency.

---

# 6. Responsibilities

## Search API

Handles:

```text
Search request
Filters
Sorting
Pagination
Autocomplete
Suggestions
```

## Search Indexer

Handles:

```text
ProductCreated
ProductUpdated
ProductDeleted
CategoryUpdated
PriceUpdated
InventoryUpdated
```

and updates OpenSearch.

## Reindex Worker

Handles:

```text
Full reindex
Partial reindex
Index migration
Recovery
```

---

# 7. Repository Structure

```text
search-service/
│
├── cmd/
│   ├── api/
│   │   └── main.go
│   │
│   ├── indexer/
│   │   └── main.go
│   │
│   └── reindex/
│       └── main.go
│
├── internal/
│   ├── config/
│   │   └── config.go
│   │
│   ├── search/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── opensearch.go
│   │   ├── query.go
│   │   └── response.go
│   │
│   ├── indexing/
│   │   ├── indexer.go
│   │   ├── mapper.go
│   │   └── bulk.go
│   │
│   ├── kafka/
│   │   └── consumer.go
│   │
│   ├── redis/
│   │   └── redis.go
│   │
│   ├── observability/
│   │   └── metrics.go
│   │
│   └── health/
│       └── health.go
│
├── mappings/
│   ├── products-v1.json
│   └── products-v2.json
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
│   └── search-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── api-deployment.yaml
│           ├── indexer-deployment.yaml
│           ├── reindex-job.yaml
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

# 8. Why Go

Search APIs are generally:

```text
high request volume
low latency
I/O intensive
concurrent
```

Go provides:

- Lightweight processes
- Goroutines
- Low memory overhead
- Fast startup
- Good HTTP performance
- Good Kafka client support
- Simple container images

The search engine performs the heavy search computation; the Go service
primarily handles API, validation, query construction, authorization and
response transformation.

---

# 9. Go Module

## `go.mod`

```go
module github.com/company-ecommerce/search-service

go 1.24

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/redis/go-redis/v9 v9.7.0
    github.com/google/uuid v1.6.0
)
```

The OpenSearch and Kafka client versions should be pinned to versions tested
by the project.

---

# 10. Configuration

## `internal/config/config.go`

```go
package config

import "os"

type Config struct {
    Port             string
    OpenSearchURL    string
    RedisURL         string
    KafkaBrokers     string
    ProductIndex     string
}

func Load() Config {
    return Config{
        Port: getEnv(
            "PORT",
            "8080",
        ),

        OpenSearchURL: getEnv(
            "OPENSEARCH_URL",
            "https://localhost:9200",
        ),

        RedisURL: getEnv(
            "REDIS_URL",
            "redis://localhost:6379/0",
        ),

        KafkaBrokers: getEnv(
            "KAFKA_BROKERS",
            "localhost:9092",
        ),

        ProductIndex: getEnv(
            "PRODUCT_INDEX",
            "products-read",
        ),
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

# 11. Environment Variables

```text
PORT=8080

OPENSEARCH_URL=https://opensearch.example.internal

REDIS_URL=redis://redis.ecommerce.svc.cluster.local:6379/0

KAFKA_BROKERS=kafka.ecommerce.svc.cluster.local:9092

PRODUCT_INDEX=products-read
```

OpenSearch credentials must come from Secrets Manager/External Secrets.

Do not store:

```text
OPENSEARCH_USERNAME
OPENSEARCH_PASSWORD
```

in Git.

---

# 12. Product Search Document

The OpenSearch document should contain fields optimized for search.

Example:

```json
{
  "productId": "PROD-1001",
  "sku": "SKU-1001",
  "name": "Wireless Bluetooth Headphones",
  "description": "Noise cancelling wireless headphones",
  "brand": "ExampleBrand",
  "categoryId": "electronics-audio",
  "categoryName": "Headphones",
  "price": 5999.00,
  "currency": "INR",
  "rating": 4.5,
  "reviewCount": 1234,
  "available": true,
  "attributes": {
    "color": "black",
    "connectivity": "bluetooth"
  },
  "searchable": true,
  "updatedAt": "2026-10-05T10:00:00Z"
}
```

---

# 13. Search Index

Example:

```text
products-v1
```

But customers should not query the physical index directly.

Use an alias:

```text
products-read
```

Architecture:

```text
products-read
      |
      ↓
products-v2
```

Later:

```text
products-read
      |
      ↓
products-v3
```

The alias allows zero-downtime index replacement.

---

# 14. Why Index Aliases Matter

Suppose we need to change:

```text
analyzer
mapping
field type
search behavior
```

We create:

```text
products-v2
```

Then:

```text
Build v2
   ↓
Bulk index
   ↓
Validate
   ↓
Switch alias
```

No need to stop customer search.

---

# 15. Index Mapping

Example:

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {

      "productId": {
        "type": "keyword"
      },

      "name": {
        "type": "text"
      },

      "description": {
        "type": "text"
      },

      "brand": {
        "type": "keyword"
      },

      "categoryId": {
        "type": "keyword"
      },

      "price": {
        "type": "double"
      },

      "rating": {
        "type": "float"
      },

      "reviewCount": {
        "type": "integer"
      },

      "available": {
        "type": "boolean"
      },

      "updatedAt": {
        "type": "date"
      }
    }
  }
}
```

Shard counts are illustrative.

They must be selected based on:

```text
document count
document size
query volume
indexing rate
node capacity
replication
failure requirements
```

---

# 16. `text` vs `keyword`

This is important.

## `text`

Used for:

```text
product name
description
searchable content
```

Example:

```text
"wireless bluetooth headphones"
```

## `keyword`

Used for:

```text
brand
categoryId
SKU
productId
status
```

Example:

```text
brand = Sony
```

---

# 17. Product Search API

```text
GET /api/v1/search
```

Example:

```text
/api/v1/search?q=wireless+headphones
```

Filters:

```text
category
brand
minPrice
maxPrice
rating
availability
```

Sorting:

```text
relevance
price_asc
price_desc
rating
newest
```

---

# 18. Example Request

```http
GET /api/v1/search?q=wireless+headphones&brand=ExampleBrand&minPrice=3000&maxPrice=10000&page=0&size=20
```

Response:

```json
{
  "query": "wireless headphones",
  "page": 0,
  "size": 20,
  "total": 125,
  "products": [
    {
      "productId": "PROD-1001",
      "name": "Wireless Bluetooth Headphones",
      "brand": "ExampleBrand",
      "price": 5999,
      "rating": 4.5
    }
  ]
}
```

---

# 19. Pagination

Avoid unrestricted pagination.

Bad:

```text
page=100000
```

Large offsets become expensive.

For deep pagination, consider:

```text
search_after
```

or another cursor-based strategy.

Normal customer search can use bounded:

```text
page
size
```

with a reasonable maximum size.

Example:

```text
maximum size = 100
```

---

# 20. Search Query

Conceptual query:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "wireless headphones",
            "fields": [
              "name^5",
              "brand^3",
              "description"
            ]
          }
        }
      ],
      "filter": [
        {
          "term": {
            "available": true
          }
        },
        {
          "term": {
            "brand": "ExampleBrand"
          }
        },
        {
          "range": {
            "price": {
              "gte": 3000,
              "lte": 10000
            }
          }
        }
      ]
    }
  }
}
```

The `filter` portion is used for exact filtering and does not need to influence
relevance in the same way as full-text matching.

---

# 21. Relevance

Search relevance can prioritize:

```text
name
brand
category
description
popularity
rating
availability
business rules
```

Example:

```text
name^5
brand^3
description^1
```

means the product name receives a higher relevance weight.

Actual weights must be validated using real search analytics.

---

# 22. Autocomplete

Endpoint:

```text
GET /api/v1/search/autocomplete?q=wire
```

Possible response:

```json
{
  "suggestions": [
    "wireless headphones",
    "wireless earbuds",
    "wireless keyboard"
  ]
}
```

Autocomplete should be optimized separately from full search.

Do not execute a huge full-text query for every keystroke.

---

# 23. Redis Autocomplete Cache

Popular suggestions can be cached:

```text
search:suggestion:wire
search:suggestion:wirel
search:suggestion:wireless
```

Short TTL:

```text
30–300 seconds
```

is illustrative.

Search cache should never become the source of truth.

---

# 24. Product Events

Product Service publishes:

```text
ProductCreated
ProductUpdated
ProductDeleted
```

Inventory may publish:

```text
InventoryChanged
```

Pricing:

```text
PriceChanged
```

Review:

```text
ReviewUpdated
```

Search consumes only the events necessary to maintain its projection.

---

# 25. ProductCreated Event

```json
{
  "eventId": "evt-product-1001",
  "eventType": "ProductCreated",
  "version": 1,
  "occurredAt": "2026-10-05T10:00:00Z",
  "source": "product-service",
  "correlationId": "corr-product-1001",
  "aggregateId": "PROD-1001",
  "payload": {
    "productId": "PROD-1001",
    "name": "Wireless Bluetooth Headphones",
    "brand": "ExampleBrand",
    "categoryId": "electronics-audio",
    "price": 5999
  }
}
```

---

# 26. Indexing Flow

```text
Product Service
      |
      ↓
PostgreSQL
      |
      ↓
Transactional Outbox
      |
      ↓
Kafka
      |
      ↓
Search Indexer
      |
      ↓
OpenSearch
```

The transactional outbox protects against:

```text
DB update succeeds
Kafka publish fails
```

without silently losing the event.

---

# 27. Indexer Responsibilities

The indexer:

1. Consumes product events.
2. Validates the event.
3. Checks idempotency/version.
4. Builds the search document.
5. Updates OpenSearch.
6. Records success/failure.
7. Commits Kafka offset only after successful processing.

---

# 28. At-Least-Once Processing

Kafka may deliver:

```text
ProductUpdated
ProductUpdated
```

multiple times.

Therefore the indexer must be idempotent.

Example:

```text
eventId = evt-product-1001
```

If already processed:

```text
skip
```

---

# 29. Event Versioning

Suppose:

```text
ProductUpdated v1
```

later becomes:

```text
ProductUpdated v2
```

The indexer must support compatible schema evolution.

Never silently assume:

```text
every event always has the same payload
```

---

# 30. Out-of-Order Product Events

Suppose:

```text
10:00 ProductUpdated price=500
10:05 ProductUpdated price=600
```

but the events arrive:

```text
price=600
price=500
```

The final search document must remain:

```text
price=600
```

Use:

```text
event version
updatedAt
sequence number
```

where available.

---

# 31. Index Version Field

A search document can include:

```json
{
  "productId": "PROD-1001",
  "sourceVersion": 27,
  "updatedAt": "2026-10-05T10:05:00Z"
}
```

Before applying an event:

```text
incoming version <= indexed version
```

means:

```text
ignore stale event
```

This is an application-level protection.

---

# 32. Delete Event

```json
{
  "eventId": "evt-product-delete-1001",
  "eventType": "ProductDeleted",
  "version": 1,
  "aggregateId": "PROD-1001"
}
```

The indexer removes:

```text
PROD-1001
```

from the searchable index.

---

# 33. Soft Delete

An alternative is:

```json
{
  "searchable": false
}
```

This can be useful for audit/reconciliation scenarios.

The search query should filter:

```text
searchable = true
```

---

# 34. Search and Inventory

Do not assume search index inventory is perfectly real-time.

Example:

```text
Inventory = 0
Search index = available=true
```

A customer might briefly see a product that has just sold out.

For critical checkout correctness:

```text
Search
  ↓
Product
  ↓
Inventory validation
```

Search should help discovery, not authorize inventory.

---

# 35. Search and Pricing

Similarly:

```text
Search price = ₹5,999
```

does not necessarily guarantee the final order price.

The Order/Pricing flow must perform authoritative price validation.

Search is a read projection.

---

# 36. OpenSearch Deployment Options

Possible production choices:

```text
Amazon OpenSearch Service
```

or:

```text
OpenSearch running on EKS
```

For this project, managed OpenSearch is the preferred enterprise production
architecture unless there is a strong reason to operate the search cluster
inside EKS.

---

# 37. Why Managed OpenSearch

Operating search infrastructure requires managing:

```text
nodes
storage
shards
replicas
JVM memory
upgrades
snapshots
security
certificates
scaling
failover
```

With Amazon OpenSearch Service, AWS manages much of the underlying service
infrastructure.

DevOps still owns:

```text
domain configuration
networking
security
access policies
capacity
index design
monitoring
retention
snapshots
```

---

# 38. OpenSearch Inside EKS

If using OpenSearch inside Kubernetes:

```text
EKS
 |
 ├── Master/control nodes
 ├── Data nodes
 ├── Ingest nodes
 └── Coordinating nodes
```

It requires:

```text
StatefulSets
PVCs
EBS CSI
topology
anti-affinity
PDB
backup
restore
JVM sizing
cluster formation
```

A StatefulSet alone does not make OpenSearch highly available.

For the enterprise ecommerce architecture, managed OpenSearch is preferred.

---

# 39. Search Service Does Not Store OpenSearch Credentials

Architecture:

```text
AWS Secrets Manager
        |
        ↓
External Secrets Operator
        |
        ↓
Kubernetes Secret
        |
        ↓
Search Service
```

Authentication may also use IAM/SigV4 depending on the OpenSearch deployment.

---

# 40. Redis

Redis can cache:

```text
popular searches
autocomplete
category suggestions
```

but should not be used to permanently store the product search index.

Architecture:

```text
Customer
   |
   ↓
Search Service
   |
   ├── Redis → cached response
   |
   └── OpenSearch → authoritative search projection
```

---

# 41. Cache Key

Example:

```text
search:v1:
q=wireless headphones
brand=ExampleBrand
minPrice=3000
maxPrice=10000
page=0
size=20
```

Normalize the query before creating the key.

For example:

```text
" Wireless   Headphones "
```

should not create many separate cache entries.

---

# 42. Cache Stampede

Suppose a popular search cache expires:

```text
search:wireless-headphones
```

10,000 requests arrive simultaneously.

Without protection:

```text
10,000 requests
      ↓
10,000 OpenSearch queries
```

Use:

```text
request coalescing
short TTL jitter
cache warming
bounded concurrency
```

---

# 43. Search API Handler

Conceptual:

```go
func (h *Handler) Search(c *gin.Context) {

    query := c.Query("q")

    if len(query) > 200 {
        c.JSON(
            http.StatusBadRequest,
            gin.H{
                "error": "query too long",
            },
        )

        return
    }

    result, err := h.service.Search(
        c.Request.Context(),
        query,
    )

    if err != nil {

        c.JSON(
            http.StatusInternalServerError,
            gin.H{
                "error": "search failed",
            },
        )

        return
    }

    c.JSON(
        http.StatusOK,
        result,
    )
}
```

---

# 44. Query Validation

Protect OpenSearch from abusive queries.

Validate:

```text
query length
page size
sort values
filter values
maximum filters
wildcards
regex
```

Never directly pass arbitrary user-controlled query DSL into OpenSearch.

The service constructs the query.

---

# 45. Search API Limits

Example:

```text
maximum query length = 200
maximum page size = 100
maximum filters = 20
```

These are illustrative.

The purpose is to protect the search backend.

---

# 46. Search Timeout

Search requests should have bounded timeouts.

Example:

```text
Customer request timeout
        ↓
Search Service
        ↓
OpenSearch timeout
```

Never allow an unbounded OpenSearch query.

If OpenSearch is slow:

```text
fail fast
```

rather than exhausting all application goroutines.

---

# 47. Search Degradation

If OpenSearch becomes unavailable:

```text
Search Service
      |
      X
OpenSearch
```

Possible behavior:

```text
autocomplete unavailable
```

while:

```text
GET product by ID
```

continues through Product Service.

Do not attempt to replace OpenSearch with an uncontrolled PostgreSQL full-table
scan during an incident.

---

# 48. Health Endpoints

```text
GET /health
GET /ready
GET /metrics
```

Liveness should not fail merely because OpenSearch has a temporary outage.

Readiness may check essential dependencies depending on the service's startup
requirements, but avoid dependency-heavy probes that cause cascading restarts.

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
    -o /out/search-api \
    ./cmd/api


FROM gcr.io/distroless/static-debian12

WORKDIR /app

COPY --from=builder \
    /out/search-api \
    /app/search-api

USER 10001

EXPOSE 8080

ENTRYPOINT ["/app/search-api"]
```

The indexer can use the same base image with a different entrypoint.

---

# 50. `.dockerignore`

```text
.git
.github

bin
tmp

coverage

*.log

.env
.env.*

tests
README.md
```

---

# 51. Local Docker Compose

```yaml
services:

  opensearch:

    image: opensearchproject/opensearch:3

    container_name: search-opensearch

    environment:

      discovery.type: single-node

      DISABLE_SECURITY_PLUGIN: "true"

      OPENSEARCH_JAVA_OPTS: "-Xms512m -Xmx512m"

    ports:

      - "9200:9200"


  redis:

    image: redis:7

    container_name: search-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: search-kafka

    ports:

      - "9092:9092"


  search-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - opensearch

      - redis

      - kafka

    environment:

      PORT: 8080

      OPENSEARCH_URL: http://opensearch:9200

      REDIS_URL: redis://redis:6379/0

      KAFKA_BROKERS: kafka:9092

      PRODUCT_INDEX: products-read

    ports:

      - "8080:8080"
```

The OpenSearch security-disabled configuration above is for local development
only.

Do not use it in production.

---

# 52. Local OpenSearch Test

Check:

```bash
curl http://localhost:9200
```

Create an index:

```bash
curl -X PUT \
  http://localhost:9200/products-v1
```

Insert a product:

```bash
curl -X POST \
  http://localhost:9200/products-v1/_doc/PROD-1001 \
  -H 'Content-Type: application/json' \
  -d '{
    "productId": "PROD-1001",
    "name": "Wireless Bluetooth Headphones",
    "brand": "ExampleBrand",
    "price": 5999,
    "available": true
  }'
```

Search:

```bash
curl \
  'http://localhost:9200/products-v1/_search?q=wireless'
```

---

# 53. Kubernetes Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: search-service

  labels:

    app: search-service

spec:

  replicas: {{ .Values.api.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: search-service

  template:

    metadata:

      labels:

        app: search-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: search-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: search-service-config

            - secretRef:

                name: search-service-secrets

          resources:

            {{- toYaml .Values.api.resources | nindent 12 }}

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

# 54. Indexer Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: search-indexer

  labels:

    app: search-indexer

spec:

  replicas: 3

  selector:

    matchLabels:

      app: search-indexer

  template:

    metadata:

      labels:

        app: search-indexer

    spec:

      serviceAccountName: search-service

      containers:

        - name: search-indexer

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          command:

            - /app/search-indexer

          envFrom:

            - configMapRef:

                name: search-service-config

            - secretRef:

                name: search-service-secrets

          resources:

            requests:

              cpu: 250m

              memory: 256Mi

            limits:

              cpu: "1"

              memory: 512Mi
```

---

# 55. Kubernetes Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: search-service

  labels:

    app: search-service

spec:

  type: ClusterIP

  selector:

    app: search-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 56. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: search-service-config

data:

  OPENSEARCH_URL: https://opensearch.example.internal

  REDIS_URL: redis://redis.ecommerce.svc.cluster.local:6379/0

  KAFKA_BROKERS: kafka.ecommerce.svc.cluster.local:9092

  PRODUCT_INDEX: products-read

  SEARCH_TIMEOUT_MS: "300"
```

---

# 57. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: search-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: search-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: OPENSEARCH_USERNAME

      remoteRef:

        key: ecommerce/prod/search-service

        property: OPENSEARCH_USERNAME

    - secretKey: OPENSEARCH_PASSWORD

      remoteRef:

        key: ecommerce/prod/search-service

        property: OPENSEARCH_PASSWORD
```

If IAM-based authentication is used, static OpenSearch credentials may not be
required.

---

# 58. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: search-service

  labels:

    app: search-service
```

Potential AWS permissions depend on architecture.

For example, if the service needs S3 access for index snapshots:

```text
EKS Pod Identity
        ↓
IAM role
        ↓
specific S3 bucket/prefix
```

Do not give the search API broad AWS permissions merely because the indexer
might need one.

---

# 59. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: search-service

spec:

  podSelector:

    matchLabels:

      app: search-service

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

# 60. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: search-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: search-service

  minReplicas: 3

  maxReplicas: 20

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

Search API scaling does not automatically scale OpenSearch.

OpenSearch capacity must be managed independently.

---

# 61. Search Backend Capacity

This is critical.

Suppose:

```text
Search API:
3 → 30 Pods
```

but:

```text
OpenSearch:
unchanged
```

The bottleneck can simply move from:

```text
Search Service
```

to:

```text
OpenSearch
```

Therefore:

```text
Application HPA
+
OpenSearch capacity
```

must be planned together.

---

# 62. Search PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: search-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: search-service
```

---

# 63. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: search-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: search-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 64. Search Metrics

Application:

```text
search_requests_total

search_request_duration_seconds

search_errors_total

search_timeout_total

search_cache_hits_total

search_cache_misses_total
```

Indexing:

```text
search_index_events_total

search_index_success_total

search_index_failure_total

search_index_lag_seconds

search_index_bulk_documents_total
```

Business:

```text
search_zero_result_total

search_queries_total

search_autocomplete_total
```

---

# 65. Important Business Metric — Zero Results

Suppose:

```text
Search requests = 1,000,000
Zero-result searches = 300,000
```

Infrastructure might be perfectly healthy.

But:

```text
30% zero-result rate
```

may indicate:

```text
bad indexing
incorrect analyzer
missing products
wrong filters
stale index
mapping problem
```

Therefore search quality is an operational concern.

---

# 66. Search SLO

Example:

```text
Search API availability:
99.95%

Search P95 latency:
< 200 ms

Search P99 latency:
< 500 ms

Autocomplete P95:
< 100 ms

Index freshness:
99% of product changes searchable
within defined target
```

Exact thresholds must come from performance testing.

---

# 67. Search Quality Metrics

Monitor:

```text
zero-result rate
click-through rate
search-to-product-view rate
search-to-cart rate
search-to-order rate
popular queries
failed queries
```

These are not just marketing metrics.

They can identify technical search problems.

---

# 68. Search Logging

Example:

```json
{
  "@timestamp": "2026-10-05T18:00:00Z",
  "level": "INFO",
  "service": "search-service",
  "correlationId": "corr-1001",
  "query": "wireless headphones",
  "filters": {
    "brand": "ExampleBrand"
  },
  "resultCount": 25,
  "durationMs": 42
}
```

Avoid logging unnecessary personal information.

---

# 69. OpenTelemetry

Trace:

```text
Customer
   |
   ↓
ALB
   |
   ↓
Search Service
   |
   ↓
Redis
   |
   ↓
OpenSearch
```

Example spans:

```text
HTTP GET /search
   |
   ├── Redis GET
   |
   └── OpenSearch _search
```

This makes slow search requests easier to diagnose.

---

# 70. Jaeger Investigation

Example:

```text
Search request = 1.8 seconds
```

Jaeger shows:

```text
HTTP                  1800ms
 └── Redis              2ms
 └── OpenSearch       1750ms
      ├── query        900ms
      └── fetch        850ms
```

Now the bottleneck is clearly the search backend rather than the Go service.

---

# 71. Search Failure Scenarios

Test:

```text
OpenSearch unavailable
OpenSearch slow
OpenSearch cluster yellow/red
Redis unavailable
Kafka unavailable
Kafka lag
Product event duplicated
Product event out of order
Index corruption
Bad mapping
Bad analyzer
Index too large
Search query timeout
Zero-result spike
Stale index
```

---

# 72. OpenSearch Unavailable

```text
Search API
     |
     X
OpenSearch
```

Response should be controlled.

For example:

```http
503 Service Unavailable
```

or a carefully designed degraded response.

Do not:

```text
Search API
   ↓
PostgreSQL full table scan
```

because this can create a second outage.

---

# 73. OpenSearch Slow

Suppose:

```text
P95 = 150ms
```

becomes:

```text
P95 = 2.5s
```

Check:

```text
OpenSearch CPU
JVM memory
GC
heap pressure
search thread pool
index size
shard count
slow queries
segment count
disk
IOPS
query complexity
```

Also inspect application traces.

---

# 74. Bad Mapping Deployment

Suppose:

```text
price
```

was accidentally mapped as:

```text
text
```

instead of:

```text
double
```

Then price filtering may fail.

Never mutate production mappings casually.

Instead:

```text
Create products-v2
        ↓
Validate
        ↓
Bulk reindex
        ↓
Test
        ↓
Switch alias
```

---

# 75. Zero-Downtime Reindexing

Architecture:

```text
products-v1
      |
      ↓
Create products-v2
      |
      ↓
Bulk index
      |
      ↓
Validate document count
      |
      ↓
Validate sample searches
      |
      ↓
Alias switch
      |
      ↓
products-read → products-v2
```

Old index can remain temporarily for rollback.

---

# 76. Alias Switch

Conceptual:

```json
{
  "actions": [
    {
      "remove": {
        "index": "products-v1",
        "alias": "products-read"
      }
    },
    {
      "add": {
        "index": "products-v2",
        "alias": "products-read"
      }
    }
  ]
}
```

The exact OpenSearch API should be tested against the deployed version.

---

# 77. Reindex Validation

Before switching:

```text
document count
sample products
search relevance
filters
sorting
autocomplete
price ranges
availability
brand filters
category filters
zero-result rate
latency
```

Compare:

```text
v1 vs v2
```

before production promotion.

---

# 78. Full Reindex

A full reindex can be necessary after:

```text
mapping change
analyzer change
data corruption
search schema change
index loss
```

Flow:

```text
Product Database
       |
       ↓
Read products in batches
       |
       ↓
Transform
       |
       ↓
Bulk OpenSearch
       |
       ↓
Validate
       |
       ↓
Alias switch
```

Do not load millions of products into application memory at once.

Use batching.

---

# 79. Bulk Indexing

Example conceptual batch:

```text
10,000 documents
    ↓
Bulk request
    ↓
OpenSearch
```

Tune batch size based on:

```text
document size
network
OpenSearch capacity
CPU
memory
bulk queue
```

Do not assume larger batches are always faster.

---

# 80. Kafka Backlog During Reindex

A reindex can happen while normal product updates continue.

Example:

```text
Full Reindex
      |
      ↓
products-v2
      |
      +
      |
Kafka ProductUpdated events
```

Possible strategy:

```text
1. Start v2 rebuild.
2. Continue consuming product events.
3. Ensure latest updates are applied to v2.
4. Validate.
5. Switch alias.
```

The exact synchronization strategy must prevent updates from being lost between
the initial snapshot and final alias switch.

---

# 81. Search Data Consistency

The architecture intentionally allows:

```text
Product DB
      |
      ↓
Kafka
      |
      ↓
Search Index
```

to have slight eventual consistency.

Example:

```text
Product price changed
10:00:00

Search index updated
10:00:02
```

Two seconds of delay may be acceptable.

The business requirement should define the maximum allowed freshness.

---

# 82. Search Does Not Authorize Business Transactions

Customer sees:

```text
Product
Price
Availability
```

from search.

At checkout:

```text
Order Service
   |
   ↓
Pricing Service
   |
   ↓
Inventory Service
```

performs authoritative validation.

Therefore:

```text
Search result ≠ final business truth
```

---

# 83. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Search Service CI

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

  ECR_REPOSITORY: search-service

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

---

# 84. SonarQube

SonarQube checks:

```text
Go bugs
security issues
code smells
duplication
test coverage
```

Search-specific tests should also cover:

```text
query construction
filter validation
pagination
mapping transformation
event processing
```

---

# 85. ECR

Repository:

```text
company-ecommerce/search-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/search-service:<git-sha>
```

Use:

```text
immutable tag
+
image digest
```

for controlled deployments.

---

# 86. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: search-service

description: Ecommerce search service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 87. Production Values

```yaml
api:

  replicaCount: 3

indexer:

  replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/search-service

  tag: ""

  pullPolicy: IfNotPresent

apiResources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi

indexerResources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi

serviceAccount:

  create: true

  name: search-service

autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 20
```

---

# 88. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── search-service/
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

# 89. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: search-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/search-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

OpenSearch infrastructure itself should be managed separately from the
application deployment.

---

# 90. Production Incident Example

## Incident ID

```text
INC-2026-SEARCH-001
```

## Severity

```text
SEV-2
```

## Title

```text
Search Latency Increased After High-Cardinality Query
```

## Date

```text
2026-10-05
```

---

# 91. Customer Impact

Customers experienced:

```text
Search P95:
180ms → 2.8s

Search P99:
450ms → 7s

Search 5xx:
0.3% → 6.2%
```

Product checkout itself remained available.

---

# 92. Timeline

```text
19:00
Search healthy.

19:04
New search configuration deployed.

19:07
Search traffic increases.

19:09
OpenSearch CPU increases.

19:10
Search P95 crosses SLO.

19:12
Search 5xx increases.

19:14
SEV-2 declared.

19:16
Jaeger identifies slow OpenSearch queries.

19:18
OpenSearch investigation identifies expensive query pattern.

19:20
Search configuration rolled back through GitOps.

19:23
OpenSearch CPU decreases.

19:25
P95 returns below SLO.

19:28
5xx returns to normal.

19:35
Incident resolved.
```

---

# 93. Root Cause

A new query configuration allowed an expensive search pattern across a
high-cardinality field.

This increased OpenSearch CPU and query latency.

The search API itself was healthy.

The bottleneck was:

```text
Search Service
      ↓
OpenSearch
      ↓
Expensive query
```

---

# 94. Amplification

```text
Expensive query
      ↓
OpenSearch CPU
      ↓
Search latency
      ↓
Requests remain in flight longer
      ↓
More concurrent requests
      ↓
OpenSearch pressure
      ↓
More timeouts
```

This is a feedback loop.

---

# 95. What Went Well

- Prometheus detected latency
- Jaeger identified OpenSearch as the slow dependency
- Search API remained horizontally scalable
- GitOps allowed quick rollback
- Product/order systems were unaffected

---

# 96. What Went Wrong

- Query configuration was not tested under production-like load
- Search backend capacity was not included in the release test
- No query-cost guardrail existed
- High-cardinality query alerting was insufficient

---

# 97. Five Whys

### Why did search latency increase?

OpenSearch queries became expensive.

### Why?

A new query configuration performed costly operations on a high-cardinality
field.

### Why was it deployed?

The configuration passed functional tests.

### Why didn't functional tests detect the issue?

They did not represent production query volume and cardinality.

### Why?

Performance testing focused on API RPS rather than backend query behavior.

---

# 98. Corrective Actions

| Action | Owner |
|---|---|
| Add production-like search load tests | Performance |
| Add query complexity validation | Backend |
| Add OpenSearch CPU alerts | SRE |
| Add slow-query dashboard | Platform |
| Review mappings | Search |
| Add high-cardinality test data | QA |
| Add canary search configuration | DevOps |
| Add query regression tests | Backend |

---

# 99. Search Disaster Recovery

Search index recovery should not require restoring business data from search.

Architecture:

```text
Product Database
      |
      ↓
Product Events
      |
      ↓
Search Index
```

If search is lost:

```text
Recreate OpenSearch
      |
      ↓
Create products-vN
      |
      ↓
Bulk rebuild
      |
      ↓
Validate
      |
      ↓
Alias switch
```

Product data remains safe in the primary data store.

---

# 100. Search Recovery Runbook

```text
1. Confirm OpenSearch failure.
2. Determine cluster/index scope.
3. Stop dangerous indexing if necessary.
4. Protect product source of truth.
5. Verify Kafka event retention.
6. Restore/recreate search backend.
7. Create new index.
8. Bulk rebuild products.
9. Replay required product events.
10. Validate document counts.
11. Validate search queries.
12. Validate filters.
13. Validate autocomplete.
14. Validate latency.
15. Switch alias.
16. Monitor.
17. Resume normal indexing.
18. Document RCA.
```

---

# 101. Search Monitoring

Monitor both:

## Application

```text
request rate
P50
P95
P99
5xx
timeouts
cache hit ratio
zero-result rate
```

## OpenSearch

```text
CPU
memory
JVM pressure
disk
search latency
indexing latency
shard health
cluster health
rejected requests
slow queries
```

## Kafka

```text
consumer lag
processing rate
consumer errors
```

---

# 102. Critical Alerts

```text
Search API P95 high

Search API 5xx high

Search timeout high

OpenSearch CPU high

OpenSearch JVM pressure high

OpenSearch disk high

OpenSearch cluster unhealthy

OpenSearch rejected requests high

Search indexing lag high

Product event consumer lag high

Search zero-result rate high

Search cache hit ratio low
```

---

# 103. Production Security Checklist

```text
[ ] OpenSearch private
[ ] No public OpenSearch endpoint
[ ] TLS enabled
[ ] Authentication enabled
[ ] Least privilege
[ ] NetworkPolicy
[ ] Security groups
[ ] Secrets Manager
[ ] External Secrets
[ ] EKS Pod Identity where applicable
[ ] No credentials in Git
[ ] No credentials in images
[ ] Query validation
[ ] Query size limits
[ ] Pagination limits
[ ] Audit logging
[ ] Trivy
[ ] SonarQube
```

---

# 104. Performance Checklist

```text
[ ] Search load test
[ ] Autocomplete load test
[ ] Large result set test
[ ] Filter test
[ ] Sort test
[ ] High-cardinality test
[ ] OpenSearch capacity test
[ ] Indexing throughput test
[ ] Kafka lag test
[ ] Reindex test
[ ] Cache stampede test
[ ] OpenSearch failure test
[ ] Search timeout test
[ ] Zero-result monitoring
```

---

# 105. Final Architecture

```text
                         Product Service
                               |
                               ↓
                           PostgreSQL
                               |
                               ↓
                       Transactional Outbox
                               |
                               ↓
                             Kafka
                               |
                               ↓
                       Search Indexer
                               |
                               ↓
                        OpenSearch Index
                               |
                         products-read
                               |
                    ┌──────────┴──────────┐
                    ↓                     ↓
               Search API            Reindex Worker
                    |
          ┌─────────┴─────────┐
          ↓                   ↓
        Redis             OpenSearch
          |                   |
          └─────────┬─────────┘
                    ↓
                 Customer
```

---

# 106. Final Production Principles

1. PostgreSQL/Product Service remains the source of truth.
2. OpenSearch is a searchable projection.
3. Search is eventually consistent.
4. Search should never be the authority for checkout.
5. Inventory must be validated by Inventory Service.
6. Price must be validated by Pricing Service.
7. Product changes should flow through durable events.
8. Index processing must be idempotent.
9. Duplicate events must be safe.
10. Out-of-order events must be handled.
11. Search indexes should use aliases.
12. Mapping changes should use versioned indexes.
13. Reindexing should not require customer downtime.
14. OpenSearch should remain private.
15. Search API should not expose raw OpenSearch DSL.
16. Query size and complexity must be controlled.
17. Search timeouts must be bounded.
18. Redis is an optimization layer, not the source of truth.
19. HPA scaling must consider OpenSearch capacity.
20. More API Pods do not automatically provide more search capacity.
21. Search quality requires business metrics.
22. Zero-result rate is an important operational signal.
23. Product event lag directly affects search freshness.
24. Reconciliation and rebuild procedures must exist.
25. Managed OpenSearch is preferred for enterprise production unless there is
    a strong reason to operate the cluster ourselves.
26. Search failure should not cause order or payment failure.
27. GitOps controls application deployment.
28. Search backend infrastructure is managed separately.
29. Performance testing must include realistic query patterns and cardinality.
30. Every major search incident requires an RCA.

# End of 176 — Search Service
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
176 Search Service          ✅

177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `177-Recommendation-Service.md`** — recommendation architecture, product/user signals, recommendation engine, Redis, Kafka, model-serving considerations, fallback strategies, cold-start handling, Kubernetes/EKS, CI/CD, GitOps, Prometheus/Grafana/ELK/OpenTelemetry/Jaeger, failure scenarios and production RCA.