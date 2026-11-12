# 130 — 21-MICROSERVICE SCALING — PRODUCTION IMPLEMENTATION

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Application:** 21 Java + Spring Boot microservices
> **Deployment:** Helm + ArgoCD + GitOps
> **Autoscaling:** HPA + KEDA + Karpenter
> **Database:** Amazon RDS PostgreSQL
> **Cache:** Amazon ElastiCache Redis/Valkey-compatible
> **Messaging:** SQS + SNS + Kafka + RabbitMQ
> **Observability:** Prometheus + Grafana + ELK + OpenTelemetry + Jaeger

---

# 1. Objective

We now move from generic capacity planning to the actual **21 microservices**.

The important production principle is:

> **Do not scale all 21 microservices the same way.**

Different services have different:

* traffic patterns
* CPU requirements
* memory requirements
* database usage
* Redis usage
* synchronous dependencies
* asynchronous dependencies
* scaling metrics
* failure characteristics
* business criticality

Our architecture therefore looks like:

```text
                         Internet
                            │
                            ▼
                         ALB
                            │
                            ▼
                     ┌─────────────┐
                     │ EKS Cluster │
                     └──────┬──────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
   Customer APIs        Transaction APIs     Async Workers
       │                    │                    │
       ▼                    ▼                    ▼
    HPA                 HPA/KEDA              KEDA
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                        Karpenter
                            │
                            ▼
                         EC2 Nodes
```

---

# 2. The 21 Services

Our project contains:

```text
1.  auth-service
2.  user-service
3.  customer-service
4.  catalog-service
5.  product-service
6.  inventory-service
7.  pricing-service
8.  promotion-service
9.  cart-service
10. order-service
11. payment-service
12. fraud-service
13. shipping-service
14. delivery-service
15. notification-service
16. search-service
17. recommendation-service
18. review-service
19. tax-service
20. invoice-service
21. reporting-service
```

---

# 3. Why Every Service Needs Its Own Scaling Model

Consider:

```text
catalog-service
```

It may receive:

```text
1000 requests/sec
```

while:

```text
payment-service
```

may receive:

```text
50 requests/sec
```

And:

```text
notification-service
```

might process:

```text
500 messages/sec
```

rather than HTTP requests.

Therefore:

```text
catalog-service
      ↓
HPA based on API workload

notification-service
      ↓
KEDA based on queue depth
```

Using the same scaling policy for both would be poor capacity engineering.

---

# 4. Service Classification

We'll classify the 21 services into groups.

## Group A — Customer-facing APIs

```text
auth-service
user-service
customer-service
catalog-service
product-service
cart-service
search-service
review-service
recommendation-service
```

These primarily receive synchronous HTTP traffic.

---

## Group B — Transaction services

```text
order-service
payment-service
fraud-service
inventory-service
pricing-service
promotion-service
tax-service
```

These participate in the transaction workflow.

They may use both:

```text
HTTP
+
events
```

---

## Group C — Fulfillment

```text
shipping-service
delivery-service
```

These can contain both synchronous and asynchronous operations.

---

## Group D — Async/background services

```text
notification-service
invoice-service
reporting-service
```

These are strong candidates for event-driven scaling.

---

# 5. Important Scaling Principle

There are three different scaling dimensions:

```text
Horizontal scaling
Vertical scaling
Event-consumer scaling
```

### Horizontal

More Pods:

```text
3 pods
 ↓
10 pods
```

### Vertical

Larger resources:

```text
500m CPU
 ↓
1 CPU
```

### Consumer scaling

More workers:

```text
2 consumers
 ↓
10 consumers
```

For our architecture:

```text
HPA
+
KEDA
+
Karpenter
```

work together.

---

# 6. HPA Architecture

For HTTP services:

```text
Traffic
  ↓
ALB
  ↓
Service
  ↓
Pods
  ↓
HPA
  ↓
Replica count
```

HPA can use resource metrics and, where configured, custom/external metrics.

---

# 7. KEDA Architecture

For queue/event-driven services:

```text
SQS/Kafka
     ↓
KEDA
     ↓
Deployment replicas
     ↓
Consumer pods
```

This is particularly useful when CPU does not accurately represent workload.

---

# 8. Karpenter Architecture

HPA/KEDA scale pods.

Karpenter supplies node capacity when pods cannot be scheduled.

```text
Traffic
   ↓
HPA/KEDA
   ↓
More Pods
   ↓
Pending Pods
   ↓
Karpenter
   ↓
New EC2 Nodes
   ↓
Pods Scheduled
```

So the complete scaling chain is:

```text
Traffic
 ↓
HPA/KEDA
 ↓
Pods
 ↓
Pending pods
 ↓
Karpenter
 ↓
Nodes
```

---

# 9. Scaling Model for Each Service

For every service we will define:

```text
Traffic
CPU profile
Memory profile
Dependencies
Scaling mechanism
Min replicas
Max replicas
Primary scaling signal
Database impact
Cache impact
Messaging impact
Failure behavior
```

The replica numbers below are **illustrative starting points**, not final production values. Chapters 127–129 provide the methodology to derive real values from testing.

---

# 10. auth-service

### Responsibility

Handles:

```text
login
token generation
authentication
token validation
```

Traffic pattern:

```text
HTTP
```

Typical characteristics:

```text
CPU:
medium/high during authentication

Memory:
moderate

Database:
user/credential lookup

Redis:
possible token/session/cache usage
```

Scaling:

```text
HPA
```

Example:

```yaml
minReplicas: 3
maxReplicas: 15
```

Potential metrics:

```text
CPU
request rate
P95 latency
```

Important consideration:

```text
login spikes
```

For example:

```text
09:00
many users login
```

can create an authentication burst.

---

# 11. user-service

Responsibilities:

```text
user profile
user preferences
account information
```

Traffic:

```text
HTTP
```

Dependencies:

```text
RDS
Redis
```

Scaling:

```text
HPA
```

Example:

```text
min = 3
max = 15
```

Potential optimization:

```text
Redis cache
```

for frequently accessed profiles where appropriate.

---

# 12. customer-service

Responsibilities:

```text
customer information
addresses
customer preferences
```

Traffic:

```text
HTTP
```

Dependencies:

```text
RDS
Redis
```

Scaling:

```text
HPA
```

Important bottleneck:

```text
database queries
```

because customer operations often involve relational data.

---

# 13. catalog-service

This can be a high-traffic service.

Responsibilities:

```text
categories
catalog browsing
category pages
```

Traffic:

```text
HIGH
```

Scaling:

```text
HPA
```

Potential architecture:

```text
Client
 ↓
ALB
 ↓
catalog-service
 ↓
Redis
 ↓
RDS
```

Caching is particularly useful for read-heavy catalog operations.

Example starting configuration:

```text
minReplicas = 4
maxReplicas = 30
```

Actual values must come from performance testing.

---

# 14. product-service

Responsibilities:

```text
product details
product attributes
product availability information
```

Traffic:

```text
HIGH
```

Scaling:

```text
HPA
```

Potential dependency:

```text
Redis
RDS
```

A product detail endpoint might be:

```text
GET /api/v1/products/{id}
```

This can be heavily cached.

---

# 15. inventory-service

This service is different.

Responsibilities:

```text
stock
reservation
inventory updates
```

Inventory operations are often sensitive to:

```text
concurrency
database locking
transactions
```

Scaling:

```text
HPA + event-driven consumers
```

Possible flow:

```text
OrderCreated
     ↓
Kafka
     ↓
inventory-service
     ↓
Reserve stock
     ↓
PostgreSQL
```

The bottleneck may not be CPU.

It may be:

```text
database lock contention
```

Therefore simply adding replicas may not improve throughput.

---

# 16. Inventory Scaling Trap

Suppose:

```text
100 requests/sec
```

all attempt to update:

```text
product_id = 1001
```

More replicas:

```text
3 → 20
```

could actually create:

```text
20 concurrent transactions
        ↓
same database row
        ↓
locking
```

Therefore inventory capacity must be tested with realistic contention.

This is a major difference between:

```text
stateless read service
```

and:

```text
transactional write service
```

---

# 17. pricing-service

Responsibilities:

```text
product price
dynamic pricing
price rules
```

Traffic:

```text
HIGH
```

Potential architecture:

```text
pricing-service
     ↓
Redis
     ↓
RDS
```

Scaling:

```text
HPA
```

If pricing data is heavily cached:

```text
Redis
```

can absorb large read volumes.

But price correctness and cache invalidation must remain controlled.

---

# 18. promotion-service

Responsibilities:

```text
coupons
discounts
campaigns
promotion rules
```

Traffic:

```text
MEDIUM/HIGH
```

But flash-sale campaigns can cause:

```text
sudden spike
```

Scaling:

```text
HPA
```

Potential bottlenecks:

```text
database
Redis
rule evaluation CPU
```

Promotion calculations should be benchmarked independently.

---

# 19. cart-service

Cart is generally high-frequency.

Typical requests:

```text
GET cart
POST cart item
PUT cart item
DELETE cart item
```

Traffic:

```text
HIGH
```

Potential architecture:

```text
cart-service
     │
     ├── Redis
     │
     └── RDS
```

Redis can be useful for:

```text
active cart
temporary state
fast reads
```

But the source-of-truth model must be explicitly defined.

---

# 20. Cart Scaling

Example:

```text
300 RPS
```

Performance test:

```text
50 RPS/pod
```

Required:

```text
300 / 50
= 6 pods
```

If we need three pods minimum for HA:

```text
minReplicas = 3
```

and performance testing shows six are needed at peak:

```text
maxReplicas >= 6
```

Add operational headroom based on the validated capacity model.

---

# 21. order-service

This is one of the most important services.

Flow:

```text
Client
  ↓
order-service
  ↓
customer
  ↓
product
  ↓
pricing
  ↓
promotion
  ↓
tax
  ↓
RDS
  ↓
Outbox
  ↓
Kafka
```

Scaling:

```text
HPA
```

Potential event consumers:

```text
Kafka
```

Order creation is transaction-heavy.

Therefore capacity depends on:

```text
HTTP throughput
DB transactions
connection pool
downstream latency
Kafka publishing
```

---

# 22. Order Service Scaling Trap

Suppose:

```text
order-service:
10 pods
```

Each:

```text
20 DB connections
```

Potential connections:

```text
200
```

Scale to:

```text
30 pods
```

Potential:

```text
600
```

This can exceed the database's safe operating capacity.

Therefore:

```text
HPA maxReplicas
```

must be coordinated with:

```text
Hikari maxPoolSize
RDS capacity
```

---

# 23. payment-service

Responsibilities:

```text
payment authorization
capture
refund
payment status
```

Scaling:

```text
HPA
```

But external payment systems may impose:

```text
TPS limits
rate limits
timeouts
```

Therefore:

```text
payment-service maxReplicas
```

cannot be determined from EKS alone.

Example:

```text
Payment provider:
100 TPS
```

Increasing payment-service from:

```text
5 → 50 pods
```

doesn't necessarily increase payment capacity.

---

# 24. Payment Idempotency

Under load:

```text
timeout
 ↓
retry
 ↓
payment request again
```

could create duplicate payment operations.

Use:

```text
idempotency key
```

for payment operations where supported by the integration.

Performance testing must verify:

```text
high load
+
timeouts
+
retries
```

do not create duplicate business transactions.

---

# 25. fraud-service

Responsibilities:

```text
fraud checks
risk evaluation
```

Potentially CPU-heavy depending on implementation.

Scaling:

```text
HPA
```

If fraud processing is asynchronous:

```text
KEDA
```

can also be appropriate.

Dependencies:

```text
RDS
external fraud provider
ML/model service
```

Potential bottleneck:

```text
external dependency
```

rather than Kubernetes.

---

# 26. shipping-service

Responsibilities:

```text
shipping calculation
carrier integration
shipping method
```

Traffic:

```text
MEDIUM
```

Scaling:

```text
HPA
```

Potential external integrations:

```text
carrier APIs
```

Use:

```text
timeouts
retry
circuit breaker
```

to prevent downstream failures from consuming all application threads.

---

# 27. delivery-service

Responsibilities:

```text
delivery status
tracking
delivery updates
```

Can use:

```text
HTTP
+
events
```

For example:

```text
ShipmentCreated
     ↓
Kafka
     ↓
delivery-service
```

Scaling:

```text
HPA
+
KEDA where event consumption is significant
```

---

# 28. notification-service

This is strongly asynchronous.

Responsibilities:

```text
email
SMS
push notifications
```

Architecture:

```text
Kafka/SQS
    ↓
notification-service
    ↓
Email/SMS provider
```

Scaling should often be driven by:

```text
queue depth
```

rather than HTTP CPU alone.

Use:

```text
KEDA
```

where supported and appropriate.

Example:

```text
Queue:
10,000 messages
       ↓
KEDA
       ↓
consumer replicas ↑
```

---

# 29. Notification Scaling Trap

Suppose:

```text
100 notification pods
```

but the SMS provider supports:

```text
100 TPS
```

The application could generate:

```text
1000 TPS
```

and receive:

```text
rate limit errors
```

Therefore:

```text
KEDA maxReplicaCount
```

must respect:

```text
external provider capacity
```

---

# 30. search-service

Search can be very high traffic.

Architecture might use:

```text
search-service
      ↓
OpenSearch / Elasticsearch
```

rather than PostgreSQL for search-heavy workloads.

Scaling:

```text
HPA
```

But capacity also depends on:

```text
search backend
```

The API tier and search cluster must be capacity-tested together.

---

# 31. Search Scaling

Example:

```text
1000 search RPS
```

If one search-service pod handles:

```text
100 RPS
```

then:

```text
1000 / 100
= 10 pods
```

But if OpenSearch can only handle:

```text
700 RPS
```

then increasing API pods doesn't solve the bottleneck.

Architecture:

```text
k6
 ↓
search-service
 ↓
OpenSearch
```

Both layers must be tested.

---

# 32. recommendation-service

Recommendation may be:

```text
CPU-heavy
memory-heavy
model-heavy
```

depending on implementation.

Scaling:

```text
HPA
```

Potentially:

```text
custom metrics
```

instead of CPU alone.

For model workloads, investigate:

```text
inference latency
model memory
CPU utilization
request batching
cache hit ratio
```

---

# 33. review-service

Responsibilities:

```text
product reviews
ratings
review retrieval
```

Traffic:

```text
read-heavy
```

Scaling:

```text
HPA
```

Potential caching:

```text
Redis
```

Potential database issue:

```text
large review queries
pagination
sorting
```

Use appropriate database indexes and pagination.

---

# 34. tax-service

Responsibilities:

```text
tax calculation
```

Traffic:

```text
LOW/MEDIUM
```

But it can be on the critical order path.

Example:

```text
order-service
   ↓
tax-service
   ↓
response
```

Even if tax-service only receives:

```text
50 RPS
```

if latency increases from:

```text
50ms → 2s
```

order latency may also increase dramatically.

Therefore:

```text
traffic volume
```

isn't the only measure of importance.

---

# 35. invoice-service

Invoice generation may be asynchronous.

Architecture:

```text
OrderCompleted
       ↓
Kafka
       ↓
invoice-service
       ↓
Generate invoice
       ↓
S3
```

Scaling:

```text
KEDA
```

based on:

```text
Kafka lag
```

or another appropriate workload metric.

Potential bottlenecks:

```text
CPU
PDF generation
S3
database
```

---

# 36. reporting-service

Reporting is usually:

```text
background
batch
analytics
```

It should not consume unlimited resources from customer-facing services.

Use:

```text
separate node pool
```

where appropriate.

Potential scaling:

```text
KEDA
CronJob
scheduled workloads
```

depending on implementation.

---

# 37. Complete Service Scaling Table

| Service        | Traffic     | Primary Scaling | Main Dependency    |
| -------------- | ----------- | --------------- | ------------------ |
| auth           | High        | HPA             | RDS/Redis          |
| user           | Medium      | HPA             | RDS                |
| customer       | Medium      | HPA             | RDS/Redis          |
| catalog        | High        | HPA             | Redis/RDS          |
| product        | High        | HPA             | Redis/RDS          |
| inventory      | Medium      | HPA/KEDA        | RDS/Kafka          |
| pricing        | High        | HPA             | Redis/RDS          |
| promotion      | Medium/High | HPA             | Redis/RDS          |
| cart           | High        | HPA             | Redis/RDS          |
| order          | High        | HPA             | RDS/Kafka          |
| payment        | Medium      | HPA             | External payment   |
| fraud          | Medium      | HPA/KEDA        | External/DB        |
| shipping       | Medium      | HPA             | External carriers  |
| delivery       | Medium      | HPA/KEDA        | Kafka              |
| notification   | Async High  | KEDA            | SQS/Kafka/provider |
| search         | High        | HPA             | Search backend     |
| recommendation | Variable    | HPA/custom      | Model/data         |
| review         | Medium      | HPA             | RDS/Redis          |
| tax            | Medium      | HPA             | External/DB        |
| invoice        | Async       | KEDA            | Kafka/S3           |
| reporting      | Batch/Async | KEDA/Cron       | DB/object storage  |

This table is the **initial architecture classification**. Actual replica counts and thresholds come from measured workload.

---

# 38. Different Services Need Different HPA Targets

Don't use:

```text
70% CPU
```

for every service blindly.

Example:

```text
catalog:
CPU-heavy

order:
DB-bound

notification:
queue-bound

search:
search-backend-bound
```

Their scaling signals should reflect the actual bottleneck.

---

# 39. Example — Catalog HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalog-service
spec:
  minReplicas: 4
  maxReplicas: 30

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalog-service

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

The exact values must be validated with Chapter 127/128 testing.

---

# 40. Example — Notification KEDA

Conceptually:

```text
Kafka/SQS
   ↓
KEDA
   ↓
notification-service
```

KEDA might scale based on:

```text
queue depth
```

rather than:

```text
CPU
```

Example conceptual policy:

```text
minReplicaCount: 2
maxReplicaCount: 20
```

Again, the maximum must be checked against the downstream notification provider's capacity.

---

# 41. Service Scaling Dependencies

Think of scaling as a graph.

Example:

```text
catalog-service
      │
      ▼
    Redis
      │
      ▼
     RDS
```

Order:

```text
order-service
      │
 ┌────┼─────┐
 ▼    ▼     ▼
RDS Redis Kafka
```

Notification:

```text
Kafka/SQS
    ↓
notification
    ↓
external provider
```

Therefore:

> **Scaling one service can increase load on multiple downstream systems.**

---

# 42. Cascading Scaling

Suppose:

```text
order-service
3 → 20 pods
```

This increases:

```text
customer-service traffic
product-service traffic
pricing-service traffic
tax-service traffic
RDS connections
Kafka messages
```

So one HPA event can trigger load increases elsewhere.

This is why all 21 services must be modeled together.

---

# 43. Example Cascade

```text
Traffic ↑
   ↓
order-service HPA
   ↓
more order pods
   ↓
more calls
   ├── customer-service
   ├── product-service
   ├── pricing-service
   └── tax-service
             │
             ▼
            RDS
```

The system can experience:

```text
order scaling
      ↓
dependency saturation
      ↓
order latency increases
      ↓
more retries
      ↓
more dependency traffic
```

This can become a feedback loop.

---

# 44. Retry Amplification

Suppose:

```text
100 orders/sec
```

and each order calls:

```text
tax-service
```

At normal conditions:

```text
100 calls/sec
```

Tax service slows.

Clients retry twice:

```text
100 × 3
= 300 calls/sec
```

Now tax-service gets:

```text
300 RPS
```

instead of:

```text
100 RPS
```

This can make the incident worse.

Therefore scaling strategy must include:

```text
timeouts
retries
backoff
circuit breakers
```

---

# 45. Service-Level Capacity

Each service should have a capacity record.

Example:

```markdown
# order-service capacity

Peak RPS:
TBD

Validated RPS/pod:
TBD

Minimum replicas:
TBD

Maximum replicas:
TBD

CPU request:
TBD

Memory request:
TBD

RDS connection budget:
TBD

Kafka throughput:
TBD

P95 target:
TBD

P99 target:
TBD
```

This should be derived from testing.

---

# 46. 21-Service Scaling Repository

Create:

```text
performance/
└── scaling/
    ├── auth-service.md
    ├── user-service.md
    ├── customer-service.md
    ├── catalog-service.md
    ├── product-service.md
    ├── inventory-service.md
    ├── pricing-service.md
    ├── promotion-service.md
    ├── cart-service.md
    ├── order-service.md
    ├── payment-service.md
    ├── fraud-service.md
    ├── shipping-service.md
    ├── delivery-service.md
    ├── notification-service.md
    ├── search-service.md
    ├── recommendation-service.md
    ├── review-service.md
    ├── tax-service.md
    ├── invoice-service.md
    └── reporting-service.md
```

This becomes the source of truth for scaling decisions.

---

# 47. Standard Helm Values

Each service can expose standard scaling values.

Example:

```yaml
replicaCount: 3

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

Then service-specific values override defaults.

---

# 48. Reusable Helm HPA Template

For example:

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "service.fullname" . }}
spec:
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "service.fullname" . }}

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.cpuTarget }}
{{- end }}
```

This will later become part of **Phase 20 Platform Engineering**.

---

# 49. HPA + KEDA Should Not Fight Each Other

Do not configure:

```text
HPA
+
KEDA
```

on the same workload without understanding how their controllers interact.

For example:

```text
order-service
```

might use HPA.

While:

```text
notification-service
```

uses KEDA.

If a service needs multiple metrics, design the autoscaling mechanism deliberately rather than creating competing replica controllers.

---

# 50. Scaling Based on Business Metrics

CPU is not always the best signal.

Possible metrics:

```text
requests/sec
active requests
queue depth
Kafka lag
request latency
orders/sec
```

Example:

```text
notification-service
```

should care about:

```text
messages waiting
```

more than:

```text
CPU
```

if CPU is not the limiting factor.

---

# 51. Custom Metrics

Architecture:

```text
Application
   ↓
Micrometer
   ↓
Prometheus
   ↓
metrics adapter / autoscaling integration
   ↓
HPA
```

For example:

```text
http_requests_per_second
```

or another carefully defined metric.

Custom autoscaling metrics require careful validation:

* metric stability
* cardinality
* aggregation
* scaling behavior
* failure behavior

---

# 52. Scaling Based on Latency

Latency can be useful as a signal, but it requires care.

For example:

```text
P95 latency ↑
```

could mean:

```text
CPU bottleneck
DB bottleneck
network problem
external dependency
```

If HPA blindly reacts to latency, it could create more pods while the real bottleneck is RDS.

So:

> **Latency is a symptom; it is not automatically the correct scaling signal.**

---

# 53. Scaling Based on Queue Depth

Queue depth is often more directly related to async work.

Example:

```text
SQS:
100 messages
```

then:

```text
10,000 messages
```

KEDA can increase consumer replicas.

But also check:

```text
consumer throughput
downstream limits
visibility timeout
DLQ
```

---

# 54. Scaling and Database Connections

For every DB-connected service record:

```text
max replicas
×
max DB connections/pod
```

Example:

```text
order-service
max replicas = 20

pool = 15
```

Potential:

```text
20 × 15
= 300 connections
```

Then compare with:

```text
RDS safe connection budget
```

This is mandatory for production scaling.

---

# 55. Scaling and Redis Connections

Similarly:

```text
max replicas
×
Redis connections/pod
```

can create connection storms.

Example:

```text
30 pods
×
10 connections
=
300 Redis connections
```

Check the Redis service's capacity and client configuration.

---

# 56. Scaling and Kafka Partitions

Kafka consumer scaling has another constraint.

Suppose:

```text
topic partitions = 6
```

and:

```text
consumer pods = 20
```

Only up to approximately the number of partitions can have active consumers in a single consumer group for that topic-partition assignment.

The extra consumers may remain idle for that topic.

Therefore:

```text
Kafka partition count
```

affects consumer scalability.

---

# 57. Kafka Scaling Chain

```text
Producer traffic
      ↓
Kafka partitions
      ↓
Consumer group
      ↓
Consumer pods
```

Capacity planning therefore includes:

```text
messages/sec/partition
consumer processing rate
partition count
consumer replicas
broker capacity
```

---

# 58. RabbitMQ Scaling

For RabbitMQ:

```text
Queue
 ↓
Consumers
```

Scale consumers based on:

```text
queue depth
message processing time
consumer throughput
```

But also consider:

```text
prefetch
ACK behavior
connection/channel count
broker resource limits
```

---

# 59. SQS Scaling

For SQS:

```text
Queue
 ↓
KEDA
 ↓
Consumer pods
```

Capacity depends on:

```text
messages/sec/pod
processing time
visibility timeout
downstream DB
downstream API
```

Again:

> Scaling consumers does not automatically increase end-to-end capacity.

---

# 60. Service Criticality

Not all services deserve the same baseline.

### Critical path

```text
auth
catalog
product
cart
order
payment
inventory
```

These may require stronger baseline capacity.

### Non-critical/background

```text
reporting
invoice
notification
```

may tolerate backlog and asynchronous processing.

This affects:

```text
minReplicas
priority
node placement
PodDisruptionBudget
autoscaling
```

---

# 61. Priority Classes

For important workloads, Kubernetes supports:

```text
PriorityClass
```

Conceptually:

```text
critical application
     ↓
higher scheduling priority

background reporting
     ↓
lower priority
```

Use this carefully.

Priority does not create capacity; it determines scheduling preference when resources are constrained.

---

# 62. Separate Node Pools

We can logically separate workloads.

Example:

```text
NodePool
├── system
├── critical-app
├── general-app
├── async-worker
└── performance
```

For example:

```text
reporting-service
```

could run on:

```text
async-worker
```

so large reporting workloads don't unnecessarily interfere with latency-sensitive APIs.

Use taints/tolerations and node selectors/affinity deliberately.

---

# 63. Pod Topology

For critical services:

```text
order-service
```

we want replicas spread across AZs/nodes.

Example:

```text
6 replicas

AZ-A → 2
AZ-B → 2
AZ-C → 2
```

This reduces the effect of:

```text
node failure
AZ failure
```

---

# 64. Scaling During AZ Failure

Suppose:

```text
order-service:
6 replicas
```

distributed:

```text
2 / 2 / 2
```

AZ-A fails.

Remaining:

```text
4
```

Kubernetes recreates the desired replicas if capacity exists.

Karpenter can provision additional nodes.

But RDS/Redis/Kafka and networking must also remain available.

This is an end-to-end scaling scenario.

---

# 65. Scaling During Flash Sale

Example:

```text
Normal:
100 RPS

Flash sale:
2000 RPS
```

Architecture:

```text
2000 RPS
    ↓
ALB
    ↓
catalog HPA
product HPA
search HPA
cart HPA
    ↓
Karpenter
    ↓
EC2 capacity
```

But:

```text
order-service
payment-service
inventory-service
```

may have lower safe capacities.

Therefore:

```text
read traffic
```

and:

```text
transaction traffic
```

must be treated differently.

---

# 66. Read vs Write Scaling

Read-heavy services:

```text
catalog
product
search
review
```

can often scale horizontally very effectively.

Write-heavy transactional services:

```text
order
inventory
payment
```

can encounter:

```text
database locks
connection limits
external API limits
```

Therefore horizontal scaling has different effectiveness.

---

# 67. Stateless vs Stateful

Our microservices should generally be stateless.

Example:

```text
order-service Pod 1
order-service Pod 2
order-service Pod 3
```

Any pod should be able to process a request.

State should live in:

```text
RDS
Redis
S3
Kafka
SQS
```

This is what makes horizontal scaling practical.

---

# 68. Avoid Local Pod State

Bad:

```text
Pod
 └── local cart state
```

When pod disappears:

```text
state lost
```

Better:

```text
Pod
 ↓
Redis / RDS
```

Then:

```text
Pod 1
Pod 2
Pod 3
```

can all access the same state.

---

# 69. Complete 21-Service Scaling Model

```text
                    ALB
                     │
                     ▼
          ┌────────────────────┐
          │ Customer-facing    │
          │ APIs               │
          └─────────┬──────────┘
                    │
        ┌───────────┼───────────────┐
        ▼           ▼               ▼
      HPA          HPA             HPA
        │           │               │
      Pods         Pods            Pods
        │           │               │
        └───────────┼───────────────┘
                    │
                 Karpenter
                    │
                    ▼
                   Nodes

Async:
SQS/Kafka
    ↓
KEDA
    ↓
Consumer Pods
    ↓
Karpenter
    ↓
Nodes
```

---

# 70. Scaling Dependency Map

```text
auth
 ├── RDS
 └── Redis

user
 ├── RDS
 └── Redis

customer
 ├── RDS
 └── Redis

catalog
 ├── Redis
 └── RDS

product
 ├── Redis
 └── RDS

inventory
 ├── RDS
 └── Kafka

pricing
 ├── Redis
 └── RDS

promotion
 ├── Redis
 └── RDS

cart
 ├── Redis
 └── RDS

order
 ├── RDS
 ├── Redis
 └── Kafka

payment
 ├── RDS
 └── external provider

fraud
 ├── RDS
 └── external provider

shipping
 └── external carrier

delivery
 └── Kafka

notification
 ├── SQS/Kafka
 └── external provider

search
 └── search backend

recommendation
 ├── model/data
 └── Redis

review
 ├── RDS
 └── Redis

tax
 ├── RDS
 └── external provider

invoice
 ├── Kafka
 └── S3

reporting
 ├── RDS
 └── S3
```

---

# 71. 21-Service Scaling Checklist

For every service:

```text
☐ Traffic measured
☐ Peak traffic known
☐ RPS/pod measured
☐ CPU measured
☐ Memory measured
☐ Min replicas defined
☐ Max replicas defined
☐ HPA/KEDA selected
☐ Dependency capacity checked
☐ DB connection budget calculated
☐ Redis connection budget calculated
☐ Messaging capacity checked
☐ AZ distribution defined
☐ PodDisruptionBudget considered
☐ Resource requests defined
☐ Resource limits reviewed
☐ Observability configured
☐ Load tested
☐ Failure tested
```

---

# 72. Final Production Scaling Chain

For synchronous services:

```text
User
 ↓
ALB
 ↓
Service
 ↓
HPA
 ↓
Replicas
 ↓
Karpenter
 ↓
Nodes
```

For asynchronous services:

```text
Event
 ↓
SQS/Kafka/RabbitMQ
 ↓
KEDA
 ↓
Consumer replicas
 ↓
Karpenter
 ↓
Nodes
```

For dependencies:

```text
Pods
 ↓
RDS / Redis / Kafka / External API
 ↓
Dependency capacity
```

The complete architecture is:

```text
                    ┌─────────────┐
                    │   Traffic   │
                    └──────┬──────┘
                           ▼
                          ALB
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
        Customer APIs   Transactions   Search
            │              │              │
           HPA            HPA            HPA
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                       Kubernetes
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 KEDA          Karpenter
                    │             │
              Async Pods          ▼
                              EC2 Nodes
                    │
                    ▼
             SQS / Kafka / RMQ

Dependencies:
 ├── RDS
 ├── Redis
 ├── Search
 ├── S3
 └── External APIs

Observability:
 ├── Prometheus
 ├── Grafana
 ├── ELK
 ├── OpenTelemetry
 └── Jaeger
```

---

# 73. Key Production Lessons

### Lesson 1

**Do not scale every microservice identically.**

### Lesson 2

**HPA scales pods; Karpenter supplies nodes.**

### Lesson 3

**KEDA is particularly useful for event-driven workloads.**

### Lesson 4

**More application pods can overload RDS or Redis.**

### Lesson 5

**Kafka partition count can limit consumer scalability.**

### Lesson 6

**External API rate limits can become the real capacity limit.**

### Lesson 7

**Read-heavy and write-heavy services scale differently.**

### Lesson 8

**CPU is not always the correct scaling metric.**

### Lesson 9

**Capacity must include AZ/node failure requirements.**

### Lesson 10

**All capacity numbers should ultimately come from measurement and controlled testing.**

---

# 74. What We Have Now

We have moved from:

```text
Generic EKS
```

to:

```text
21 independent services
```

with:

```text
Individual workload
        ↓
Individual scaling strategy
        ↓
Individual dependencies
        ↓
Individual capacity requirements
```

The next three chapters will use this model to simulate different real-world traffic levels:

```text
131 — LOW TRAFFIC ARCHITECTURE
        ↓
132 — MEDIUM TRAFFIC ARCHITECTURE
        ↓
133 — HIGH TRAFFIC ARCHITECTURE
```

We will take the **same 21 services** and show exactly what changes between low, medium, and high traffic: **replicas, HPA/KEDA behavior, EKS nodes, Karpenter, RDS, Redis, Kafka/SQS, ALB, observability, cost considerations, and failure headroom**.
