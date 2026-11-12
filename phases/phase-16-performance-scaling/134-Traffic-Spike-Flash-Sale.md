# 134 — TRAFFIC SPIKE / FLASH SALE ARCHITECTURE

## 1. Objective

In the previous chapters we designed:

```text
131 — LOW TRAFFIC
10–30 RPS

132 — MEDIUM TRAFFIC
100–500 RPS

133 — HIGH TRAFFIC
500–5,000+ RPS
```

Now we introduce the most dangerous workload pattern:

> **A sudden traffic spike.**

A normal day might look like:

```text
500 RPS
```

Then a flash sale starts:

```text
500
 ↓
1,000
 ↓
2,500
 ↓
5,000
 ↓
10,000 RPS
```

The important question is not:

> "Can Kubernetes create more Pods?"

The real question is:

> **Can the entire platform absorb a sudden increase without causing a cascading failure?**

---

# 2. Flash Sale Scenario

Assume our e-commerce platform launches a major sale at:

```text
12:00 PM
```

Before the sale:

```text
Normal traffic:
500 RPS
```

At 12:00:

```text
Traffic:
5,000 RPS
```

At peak:

```text
Traffic:
10,000 RPS
```

At the same time:

```text
Product views increase
Search increases
Cart operations increase
Order attempts increase
Payment requests increase
Inventory checks increase
Notifications increase
Kafka events increase
SQS messages increase
```

This creates a synchronized load increase across the entire platform.

---

# 3. Normal Architecture

Before the sale:

```text
                       USERS
                          |
                          v
                   CloudFront/WAF
                          |
                          v
                         ALB
                          |
                          v
                        EKS
                          |
             +------------+------------+
             |            |            |
             v            v            v
           Redis         RDS         Kafka/SQS
```

Everything is operating normally.

---

# 4. Flash Sale Architecture

During the spike:

```text
                         USERS
                           |
                           v
                    CloudFront/WAF
                           |
                           v
                          ALB
                           |
                           v
                    +-------------+
                    |     EKS     |
                    |             |
                    | HPA / KEDA  |
                    |      |      |
                    |      v      |
                    | Karpenter   |
                    +------+------+
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
     Redis                RDS             Kafka/SQS
       |                   |                   |
       +-------------------+-------------------+
                           |
                    External APIs
```

The difference is the **speed and magnitude of the load change**.

---

# 5. Flash Sale Is Different From High Traffic

High traffic:

```text
5,000 RPS
```

may remain stable for:

```text
hours
```

Flash sale:

```text
500 RPS
   ↓
10,000 RPS
```

may happen within:

```text
seconds/minutes
```

This creates problems such as:

```text
HPA reaction delay
node provisioning delay
image pull delay
database connection spikes
cache stampede
queue growth
retry storms
external API saturation
```

---

# 6. The Flash Sale Failure Chain

A common failure chain:

```text
Traffic spike
     |
     v
More requests
     |
     v
CPU increases
     |
     v
HPA increases Pods
     |
     v
Pods create more DB connections
     |
     v
RDS connection pressure
     |
     v
Requests become slow
     |
     v
Clients retry
     |
     v
More requests
     |
     v
More Pods
     |
     v
More DB connections
     |
     v
RDS overload
```

This is a **positive feedback loop**.

Our architecture must break this loop.

---

# 7. Flash Sale Design Principles

We need:

```text
1. Absorb traffic at the edge
2. Cache aggressively where safe
3. Pre-scale predictable workloads
4. Autoscale safely
5. Protect the database
6. Protect external APIs
7. Queue asynchronous work
8. Limit retries
9. Apply backpressure
10. Preserve critical business operations
```

---

# 8. Predictable vs Unpredictable Traffic

Flash sales are often predictable.

Suppose marketing announces:

```text
Sale starts:
12:00 PM
```

We already know traffic will increase.

Therefore:

> **Do not rely entirely on reactive autoscaling.**

Instead:

```text
Before sale
    |
    v
Pre-scale
    |
    v
Flash sale
    |
    v
HPA/KEDA handles additional demand
```

---

# 9. Pre-Scaling

Suppose normal:

```text
catalog-service
5 replicas
```

Expected flash-sale demand:

```text
20 replicas
```

Instead of waiting for:

```text
5
 ↓
HPA detects CPU
 ↓
10
 ↓
Karpenter creates nodes
 ↓
20
```

we can pre-scale:

```text
Before sale:

5
 ↓
15
```

Then HPA handles additional unexpected demand.

This reduces reaction time.

---

# 10. Why Reactive Scaling Alone Is Dangerous

Suppose:

```text
Traffic:
500 -> 10,000 RPS
```

HPA needs:

```text
metric collection
+
evaluation
+
scale decision
+
Pod creation
+
image pull
+
application startup
+
readiness
```

Karpenter may additionally need:

```text
capacity selection
+
EC2 provisioning
+
node bootstrap
+
CNI/IP allocation
+
Pod scheduling
```

This can take much longer than the traffic spike itself.

Therefore:

```text
Reactive scaling
```

is necessary but not sufficient for predictable flash-sale events.

---

# 11. Pre-Warming the Cluster

Before the event:

```text
Normal
500 RPS
```

Prepare:

```text
EKS
 |
 +-- additional nodes
 |
 +-- application replicas
 |
 +-- Redis capacity
 |
 +-- RDS capacity
 |
 +-- Kafka/SQS consumers
```

The cluster enters:

```text
READY FOR SALE
```

state.

---

# 12. Flash Sale Capacity Plan

Create a runbook:

```text
FLASH SALE START
      |
      +-- Increase minimum replicas
      |
      +-- Increase node capacity
      |
      +-- Verify RDS connections
      |
      +-- Verify Redis
      |
      +-- Verify Kafka
      |
      +-- Verify SQS
      |
      +-- Verify ALB
      |
      +-- Verify dashboards
      |
      +-- Verify alerts
```

After the sale:

```text
Reduce temporary capacity
```

carefully.

---

# 13. Capacity Before the Event

Example simulation:

```text
Current:
500 RPS

Expected:
5,000 RPS

Safety target:
7,500 RPS
```

Suppose load testing established:

```text
catalog-service:
50 RPS/pod
```

Then:

```text
5,000 / 50
=
100 pods
```

With 50% headroom:

```text
100 × 1.5
=
150 pods
```

This is only an example.

The real value must come from measured testing.

---

# 14. Pre-Scaling Critical Services

High-priority services may include:

```text
catalog-service
product-service
search-service
pricing-service
cart-service
order-service
inventory-service
payment-service
```

For example:

```text
catalog:
10 -> 30

product:
10 -> 30

search:
8 -> 25

pricing:
8 -> 20

cart:
8 -> 20

order:
8 -> 20
```

These numbers are illustrative.

The actual values should come from the flash-sale load model.

---

# 15. Don't Pre-Scale Every Service Equally

This is important.

You don't necessarily need:

```text
reporting-service:
1 -> 30
```

because reporting may not be part of the immediate customer request path.

Instead:

```text
Customer-facing:
pre-scale aggressively

Background:
scale from queues
```

This protects resources for the most important workloads.

---

# 16. Traffic Classification

During the sale:

```text
              Traffic
                 |
       +---------+---------+
       |                   |
   Customer               Async
       |                   |
       v                   v
    ALB/API            Kafka/SQS
       |                   |
       v                   v
 Critical APIs          Workers
```

Customer-facing workloads should receive priority.

---

# 17. CDN Protection

Product pages may be heavily requested.

Instead of:

```text
10,000 users
     |
     v
ALB
     |
     v
EKS
```

use caching where appropriate:

```text
10,000 users
     |
     v
CloudFront
     |
     +---- cache HIT
     |
     +---- cache MISS
             |
             v
            ALB
```

This can significantly reduce origin traffic for cacheable content.

---

# 18. What Should Be Cached?

Potential candidates:

```text
product information
catalog information
images
static assets
public promotional content
selected read-heavy API responses
```

Be careful with:

```text
inventory availability
payment status
personalized cart
order status
```

because stale data can have business consequences.

Caching decisions must respect correctness requirements.

---

# 19. Product Page Architecture

Example:

```text
User
 |
 v
CloudFront
 |
 +---- cached product data
 |
 +---- cached images
 |
 +---- cache miss
          |
          v
     product-service
          |
          v
        Redis
          |
          v
         RDS
```

During a flash sale, this can prevent enormous numbers of repetitive database reads.

---

# 20. Redis Pre-Warming

Before the sale:

```text
Popular products
      |
      v
Redis
```

can be warmed.

Example:

```text
product:1001
product:1002
product:1003
product:1004
```

The objective is to prevent:

```text
Flash sale begins
      |
      v
millions of cache misses
      |
      v
RDS surge
```

---

# 21. Cache Stampede During Flash Sale

Suppose:

```text
product:1001
```

expires just as:

```text
10,000 users
```

request it.

Without protection:

```text
10,000 requests
      |
      v
Redis MISS
      |
      v
RDS
```

Potential database overload.

Use techniques such as:

```text
TTL jitter
request coalescing
early refresh
locking
stale-while-revalidate
```

depending on the application design.

---

# 22. Inventory Is Special

During a flash sale:

```text
product views
```

may be huge.

But:

```text
inventory updates
```

must remain correct.

Example:

```text
Stock:
100 units
```

Thousands of users attempt:

```text
BUY
```

We cannot simply cache:

```text
stock = 100
```

and let everyone purchase.

Inventory must use an authoritative transactional mechanism.

---

# 23. Inventory Reservation

A conceptual flow:

```text
Customer
   |
   v
order-service
   |
   v
inventory-service
   |
   v
Reserve stock
   |
   +---- success
   |
   +---- insufficient stock
```

For high concurrency, inventory operations may require:

```text
atomic update
locking strategy
reservation records
idempotency
event processing
```

The exact mechanism depends on the inventory data model.

---

# 24. Prevent Overselling

Suppose:

```text
Available:
1 unit
```

Requests:

```text
Customer A
Customer B
Customer C
```

All arrive simultaneously.

Incorrect:

```text
read stock = 1
read stock = 1
read stock = 1
```

All three may think the item is available.

Correct design requires atomicity around the inventory decision.

For example, conceptually:

```sql
UPDATE inventory
SET available_quantity = available_quantity - 1
WHERE product_id = ?
  AND available_quantity > 0;
```

Then verify the affected row count.

This is only an example; actual transaction and locking design must match the database model.

---

# 25. Order Service During Flash Sale

Order-service is a critical bottleneck.

Architecture:

```text
                order-service
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    Customer       Pricing         Tax
       |              |              |
       +--------------+--------------+
                      |
                      v
                    RDS
                      |
                      v
                  Outbox
                      |
                      v
                    Kafka
```

The synchronous path should remain as small as practical.

---

# 26. Don't Put Everything in Checkout

Avoid:

```text
Order
 |
 +-- notification
 +-- reporting
 +-- invoice generation
 +-- analytics
 +-- email
 +-- recommendation
```

inside the synchronous request.

Instead:

```text
Order committed
       |
       v
OrderCreated
       |
       +---- Notification
       +---- Invoice
       +---- Reporting
       +---- Analytics
```

This prevents non-critical services from blocking checkout.

---

# 27. Payment Protection

During a flash sale:

```text
payment-service
```

can become extremely sensitive.

External payment providers may have:

```text
rate limits
timeouts
latency
quotas
```

Therefore:

```text
order-service
      |
      v
payment-service
      |
      v
rate limiter
      |
      v
external provider
```

The payment service should use:

```text
timeouts
bounded retries
idempotency
circuit breaker
```

where appropriate.

---

# 28. Payment Idempotency

Suppose the customer clicks:

```text
PAY
```

and the network times out.

The client retries.

Without idempotency:

```text
Payment attempt 1
Payment attempt 2
```

could potentially charge twice.

Use a business idempotency key:

```text
payment:
order-123
```

or another suitable unique operation identifier.

The exact key design must match the payment provider's idempotency model.

---

# 29. Rate Limiting

Flash-sale systems need traffic protection.

Possible layers:

```text
CloudFront
WAF
ALB/API layer
application
payment provider
```

Conceptually:

```text
User
 |
 v
Rate Limit
 |
 +---- allowed
 |
 +---- rejected/throttled
```

Rate limiting protects the platform from:

```text
abusive clients
bots
accidental retry storms
unbounded clients
```

---

# 30. Queue Instead of Rejecting Everything

For some workloads, queues can absorb temporary bursts.

Example:

```text
Order event
    |
    v
Kafka
    |
    v
inventory/payment/fraud
```

or:

```text
Notification
    |
    v
SQS
    |
    v
notification workers
```

This changes the workload from:

```text
instant processing
```

to:

```text
controlled asynchronous processing
```

---

# 31. Backpressure

Suppose:

```text
Kafka producer:
20,000 events/sec
```

but downstream can process:

```text
10,000 events/sec
```

Don't blindly create unlimited consumers.

Instead:

```text
Producer
   |
   v
Kafka
   |
   v
controlled consumer concurrency
   |
   v
downstream
```

The backlog becomes a buffer.

Monitor:

```text
consumer lag
message age
processing latency
```

---

# 32. SQS During Flash Sale

Suppose:

```text
notification events
```

increase from:

```text
1,000/min
```

to:

```text
100,000/min
```

Architecture:

```text
Order Events
     |
     v
SQS
     |
     v
KEDA
     |
     v
notification workers
```

KEDA can scale consumers based on queue state.

But downstream email/SMS/push providers may have their own rate limits.

---

# 33. KEDA Guardrail

Don't configure:

```text
maxReplicaCount: 1000
```

just because the queue can become large.

Suppose provider capacity is:

```text
5,000 notifications/sec
```

and one worker processes:

```text
100 notifications/sec
```

Then:

```text
5,000 / 100
=
50 workers
```

may be enough from the provider-capacity perspective.

The correct number must be measured.

---

# 34. Retry Storm

Flash sales create a dangerous situation:

```text
Traffic spike
   |
   v
dependency slows
   |
   v
requests timeout
   |
   v
clients retry
   |
   v
traffic increases
   |
   v
dependency becomes slower
```

This is retry amplification.

Use:

```text
bounded retries
exponential backoff
jitter
timeouts
circuit breakers
```

and avoid retrying errors that are unlikely to succeed.

---

# 35. Client Retry Control

The client can also make the incident worse.

Example:

```text
Request
  |
  X timeout
  |
Retry immediately
  |
X timeout
  |
Retry immediately
```

Instead:

```text
Request
  |
  X
  |
backoff
  |
retry
```

with a bounded retry count.

The client and server retry policies should be designed together.

---

# 36. Circuit Breaker

Suppose:

```text
payment provider
```

is failing.

Without circuit breaker:

```text
1,000 requests
    |
    v
payment
    |
    v
provider
    |
    X
```

The application continues sending requests.

With a circuit breaker:

```text
payment
   |
   v
circuit
   |
   X provider
```

After the configured failure conditions are reached, calls are temporarily rejected/fail fast rather than continuously consuming resources.

The exact thresholds require testing.

---

# 37. Database Protection

During a flash sale:

```text
RDS
```

must be treated as a finite resource.

Protection mechanisms:

```text
connection pools
query timeouts
connection timeouts
rate limiting
caching
read replicas where appropriate
queueing
circuit breakers
HPA max replicas
```

---

# 38. Connection Pool Guardrail

Suppose:

```text
order-service:
20 pods
```

and:

```text
maximumPoolSize = 10
```

Potential:

```text
200 connections
```

Now suppose HPA goes:

```text
20 -> 50
```

Potential:

```text
500 connections
```

That could overwhelm the database depending on its capacity and other workloads.

Therefore:

```text
HPA max
```

must be compatible with:

```text
DB connection budget
```

---

# 39. Flash Sale Database Architecture

```text
                  Application
                       |
             +---------+---------+
             |                   |
          Cached Reads        Writes
             |                   |
             v                   v
           Redis              RDS Primary
             |                   |
             |              +----+----+
             |              |         |
             |          Read Replica  ...
             |
             +----------------------+
```

The exact read-replica strategy depends on consistency requirements.

---

# 40. Database Write Hotspot

Suppose:

```text
one product
```

is extremely popular.

All requests attempt to update:

```text
same inventory row
```

This can cause:

```text
lock contention
```

and:

```text
transaction latency
```

Scaling application Pods does not solve row-level contention.

Instead investigate:

```text
transaction design
atomic updates
partitioning
reservation architecture
queue-based serialization
hot-item strategy
```

where appropriate.

---

# 41. Hot Product Strategy

For extremely popular products, one possible architecture is:

```text
Users
 |
 v
Order/Reservation Queue
 |
 v
Inventory Processor
 |
 v
Atomic Inventory Update
```

This reduces uncontrolled concurrent writes to the same inventory record.

But queueing introduces:

```text
eventual consistency
```

and therefore must be designed around the business requirements.

---

# 42. Flash Sale Architecture With Queue

```text
                USERS
                  |
                  v
               ALB/WAF
                  |
                  v
              order-service
                  |
                  v
          reservation request
                  |
                  v
                Kafka
                  |
                  v
          inventory processor
                  |
                  v
                 RDS
```

This can protect the database from massive simultaneous inventory writes.

---

# 43. Trade-Off

Queueing can improve:

```text
throughput
stability
backpressure
```

but may introduce:

```text
processing delay
eventual consistency
more complex state management
```

Therefore don't queue everything.

Use it where the business workflow permits asynchronous processing.

---

# 44. Flash Sale Search Traffic

Search can experience a huge spike.

Example:

```text
10,000 users
```

searching:

```text
"discount shoes"
```

If every search reaches PostgreSQL:

```text
Search
  |
  v
RDS
```

the database can become the bottleneck.

Use:

```text
search-service
    |
    v
OpenSearch/Elasticsearch
```

for search-specific workloads.

---

# 45. Search Cache

Frequently repeated queries can potentially be cached:

```text
"iphone"
"shoes"
"laptop"
```

Architecture:

```text
Search request
     |
     v
Redis
     |
     +-- HIT
     |
     +-- MISS
            |
            v
       Search backend
```

Cache carefully because search results can change.

---

# 46. Recommendation Service

Recommendation traffic can be deprioritized if necessary.

For example:

```text
checkout
```

is critical.

Personalized recommendations may not be.

If recommendation-service becomes overloaded:

```text
recommendations unavailable
```

the product page can still operate with:

```text
default recommendations
cached recommendations
popular products
```

This is **graceful degradation**.

---

# 47. Graceful Degradation

A resilient flash-sale system doesn't require every component to be healthy for every user action.

Example:

```text
Recommendation service
       |
       X
```

Fallback:

```text
"Popular products"
```

instead of:

```text
500 error
```

Another example:

```text
Notification service
       |
       X
```

Order creation can still succeed if notification is asynchronous.

---

# 48. Criticality Classification

Classify services:

### Critical

```text
auth
cart
order
inventory
payment
```

### Important

```text
catalog
product
pricing
promotion
tax
shipping
```

### Degradable

```text
recommendation
reporting
notification
analytics-related workloads
```

The exact classification should reflect actual business requirements.

---

# 49. Priority During Flash Sale

When resources are constrained:

```text
Critical workloads
        |
        v
highest protection

Non-critical workloads
        |
        v
scale down / defer / degrade
```

This prevents:

```text
reporting
```

from consuming capacity needed by:

```text
checkout
```

---

# 50. Kubernetes Priority

Kubernetes `PriorityClass` can be used where appropriate.

Conceptually:

```text
Critical
    |
    v
high priority

Normal
    |
    v
standard

Batch
    |
    v
lower priority
```

This should be implemented carefully.

Priority does not create additional physical capacity.

---

# 51. Flash Sale Node Strategy

Before sale:

```text
Normal:
10 nodes
```

Pre-scale:

```text
20 nodes
```

Expected peak:

```text
30 nodes
```

Karpenter can handle unexpected additional capacity.

This provides:

```text
pre-warmed capacity
+
reactive capacity
```

instead of relying entirely on EC2 provisioning during the spike.

---

# 52. Avoid Huge Over-Provisioning

If normal traffic is:

```text
500 RPS
```

and predicted peak:

```text
5,000 RPS
```

don't necessarily keep:

```text
100 nodes
```

running all day.

Instead:

```text
Normal
10 nodes
      |
      v
Before sale
25 nodes
      |
      v
During sale
25–40 nodes
      |
      v
After sale
10 nodes
```

The actual values should come from the capacity model.

---

# 53. Scheduled Scaling

Because flash sales are predictable, scheduled scaling can be used where supported.

Conceptually:

```text
11:00
increase minimum replicas

11:30
increase node capacity

12:00
sale starts

14:00
traffic falls

14:30
gradually reduce capacity
```

Do not immediately scale down when traffic first starts falling.

---

# 54. Slow Scale-Down

Suppose:

```text
10,000 RPS
```

drops to:

```text
1,000 RPS
```

Immediately terminating hundreds of Pods can cause:

```text
connection churn
cache churn
new pod startup
database connection changes
```

Use controlled scale-down.

For example:

```text
high capacity
    |
    v
medium capacity
    |
    v
normal capacity
```

rather than:

```text
high
 |
 v
minimum
```

---

# 55. Database Scale-Down

Be particularly careful with database changes.

Application capacity can scale down quickly.

Database infrastructure often should not be resized aggressively immediately after a flash sale.

First confirm:

```text
traffic pattern
connection usage
CPU
IO
storage
replication
```

and then perform planned capacity changes.

---

# 56. Flash Sale Monitoring Dashboard

Create a dedicated dashboard:

```text
FLASH SALE COMMAND CENTER
```

Sections:

### Traffic

```text
RPS
active users
4xx
5xx
```

### API

```text
P50
P95
P99
```

### EKS

```text
Pods
Pending Pods
Nodes
CPU
Memory
HPA
KEDA
```

### Database

```text
connections
CPU
IO
query latency
locks
```

### Redis

```text
hit ratio
latency
memory
evictions
```

### Messaging

```text
Kafka lag
SQS depth
message age
```

### Business

```text
orders/min
checkout success
payment success
inventory rejection
```

---

# 57. Business Metrics Are Critical

Suppose infrastructure looks healthy:

```text
CPU = 50%
Memory = 50%
Pods = healthy
```

but:

```text
checkout success = 70%
```

The platform is not healthy from the customer's perspective.

During a flash sale monitor:

```text
orders/minute
checkout success
payment success
inventory reservation success
cart success
```

---

# 58. Flash Sale SLO

Define temporary operational targets before the event.

Example:

```text
Checkout availability:
>= defined SLO

Checkout P95:
<= defined target

Payment success:
>= defined business target

Inventory correctness:
no overselling
```

The actual values should be agreed with product/business teams.

---

# 59. Alert Thresholds

During flash sale, alerts must distinguish:

```text
expected high traffic
```

from:

```text
unexpected failure
```

For example:

```text
CPU 80%
```

may be normal during a planned sale.

But:

```text
5xx increasing
```

is potentially a serious problem.

Therefore alerts should focus on:

```text
errors
latency
saturation
business failures
dependency failures
```

not simply high CPU.

---

# 60. Flash Sale Alert Examples

```text
HighCheckoutErrorRate
HighPaymentFailureRate
HighOrderLatency
HighRDSConnections
HighRDSLatency
RedisHighLatency
KafkaConsumerLag
SQSMessageAgeHigh
PodsPending
KarpenterProvisioningDelay
ALB5xxHigh
InventoryReservationFailure
```

---

# 61. Flash Sale Runbook

## T-60 minutes

```text
[ ] Verify infrastructure health
[ ] Verify database
[ ] Verify Redis
[ ] Verify Kafka
[ ] Verify SQS
[ ] Verify external APIs
[ ] Verify dashboards
[ ] Verify alerts
[ ] Verify rollback plan
```

---

# 62. T-30 Minutes

```text
[ ] Increase minimum replicas
[ ] Pre-scale nodes
[ ] Warm critical caches
[ ] Verify DB connections
[ ] Verify Redis capacity
[ ] Verify Kafka consumers
[ ] Verify SQS workers
```

---

# 63. T-5 Minutes

```text
[ ] Freeze risky production deployments
[ ] Confirm on-call engineers
[ ] Open incident/war-room channel
[ ] Confirm dashboards
[ ] Confirm communication plan
[ ] Confirm rollback procedures
```

---

# 64. Sale Start

Monitor continuously:

```text
RPS
P95
P99
5xx
checkout success
payment success
RDS
Redis
Kafka
SQS
Pods
Nodes
```

---

# 65. During the Spike

If traffic increases:

```text
500
 ↓
2,000
 ↓
5,000
```

do not immediately panic.

Check:

```text
Are errors increasing?
Is latency increasing?
Are dependencies healthy?
Are Pods scaling?
Are nodes available?
```

High CPU alone may be expected.

---

# 66. If Pods Become Pending

Flow:

```text
HPA
 |
 v
more replicas
 |
 v
Pending Pods
 |
 v
Karpenter
 |
 v
new nodes
```

Investigate:

```text
Karpenter logs/events
NodePool constraints
instance availability
subnet/IP capacity
Pod resource requests
taints/tolerations
```

---

# 67. If RDS Becomes Saturated

Don't immediately:

```text
increase application replicas
```

Instead:

```text
1. protect DB
2. inspect connections
3. inspect queries
4. inspect locks
5. increase cache usage where safe
6. reduce unnecessary DB traffic
7. control retries
8. throttle non-critical workloads
```

---

# 68. If Redis Fails

Critical question:

> What happens if the cache disappears?

For cache-aside:

```text
Redis
  |
  X
  |
  v
RDS
```

must be protected.

Use:

```text
bounded DB fallback
request throttling
connection pool protection
cache recovery
```

Avoid sending the full flash-sale workload directly to RDS.

---

# 69. If Kafka Lag Explodes

Suppose:

```text
Kafka lag:
1M events
```

Do not automatically scale consumers to:

```text
1,000 Pods
```

Investigate:

```text
partition count
consumer processing
database capacity
external APIs
consumer errors
```

Then increase concurrency only when the downstream system can handle it.

---

# 70. If Payment Provider Fails

Do not continuously retry.

Instead:

```text
Payment Provider
      |
      X
      |
payment-service
      |
      v
Circuit Breaker
      |
      v
controlled failure
```

The customer should receive a clear status.

The order state machine must handle:

```text
payment pending
payment failed
payment retry
payment success
```

according to business requirements.

---

# 71. If Inventory Becomes the Bottleneck

Inventory correctness is more important than raw throughput.

Use:

```text
atomic reservation
idempotency
queueing where appropriate
bounded concurrency
```

Avoid overselling just to increase throughput.

---

# 72. If Notification Becomes Backlogged

Don't block checkout.

Instead:

```text
Order
 |
 v
Kafka/SQS
 |
 v
notification-service
```

The queue can absorb temporary backlog.

Monitor:

```text
message age
queue depth
processing rate
DLQ
```

---

# 73. If Reporting Falls Behind

Reporting is generally a candidate for graceful degradation.

Allow:

```text
reporting backlog
```

while protecting:

```text
checkout
payment
inventory
```

This is an example of **workload prioritization**.

---

# 74. Flash Sale Failure Isolation

A good architecture should look like:

```text
               CHECKOUT
                  |
       +----------+----------+
       |          |          |
    Order      Payment    Inventory
       |          |          |
       +----------+----------+
                  |
              critical
                  |
          +-------+-------+
          |               |
     Notification      Reporting
          |               |
       async           async
```

A failure in reporting should not take down checkout.

---

# 75. Circuit Breaker Dependency Graph

Example:

```text
order-service
     |
     +---- pricing
     |
     +---- promotion
     |
     +---- tax
     |
     +---- payment
```

Each external dependency should have:

```text
timeout
retry budget
circuit breaker where appropriate
fallback where possible
```

This prevents one dependency from consuming all request threads.

---

# 76. Thread Pool Protection

Suppose order-service has:

```text
200 request threads
```

and payment API becomes slow.

Without protection:

```text
200 threads
   |
   v
waiting for payment
```

Eventually:

```text
new requests
   |
   v
no available threads
```

The entire service appears down.

Timeouts and bulkhead/concurrency controls can prevent one dependency from consuming all application resources.

---

# 77. Bulkhead Concept

Conceptually:

```text
order-service
 |
 +-- payment pool
 |
 +-- pricing pool
 |
 +-- customer pool
```

If payment becomes slow:

```text
payment pool exhausted
```

but:

```text
pricing pool
customer pool
```

can continue operating, depending on the application/client implementation.

---

# 78. Flash Sale Graceful Degradation

Possible degradation hierarchy:

```text
Level 1
Full experience

Level 2
Disable recommendations

Level 3
Reduce non-critical personalization

Level 4
Delay notifications

Level 5
Delay reporting

Level 6
Protect checkout/payment/inventory
```

This must be agreed with product/business teams before the event.

---

# 79. Emergency Traffic Controls

If the platform approaches unsafe capacity:

```text
WAF
 |
 v
Rate limit
 |
 v
controlled traffic
```

Possible mechanisms:

```text
rate limiting
bot protection
queue/waiting room
feature flags
temporary endpoint restrictions
```

A virtual waiting room can be useful for extremely high-demand events when business requirements justify it.

---

# 80. Waiting Room Concept

Instead of:

```text
100,000 users
      |
      v
EKS
```

use:

```text
100,000 users
      |
      v
Waiting Room
      |
      v
Controlled admission
      |
      v
Application
```

This converts:

```text
unbounded concurrency
```

into:

```text
controlled concurrency
```

which can protect downstream systems.

---

# 81. Flash Sale and DNS

Don't change DNS during an active incident unless the architecture specifically requires it.

DNS changes have:

```text
TTL
caching
propagation
```

and can complicate an already stressed system.

For normal flash-sale scaling:

```text
Route 53
   |
   v
stable endpoint
```

while:

```text
CloudFront/WAF/ALB/EKS
```

handle traffic scaling.

---

# 82. Flash Sale Observability Architecture

```text
                        FLASH SALE
                            |
                            v
                         TRAFFIC
                            |
          +-----------------+-----------------+
          |                 |                 |
        Metrics           Logs             Traces
          |                 |                 |
          v                 v                 v
     Prometheus           ELK             OTel/Jaeger
          |                 |                 |
          +-----------------+-----------------+
                            |
                            v
                         Grafana
                            |
                            v
                       Alertmanager
```

---

# 83. Trace a Slow Checkout

Suppose:

```text
Checkout P95 = 2 seconds
```

Jaeger:

```text
checkout
 |
 +-- order-service 1.8s
       |
       +-- pricing 50ms
       +-- promotion 40ms
       +-- tax 60ms
       +-- inventory 100ms
       +-- DB 1.4s
```

Now we know:

```text
RDS/database path
```

needs investigation.

---

# 84. Logs During Flash Sale

Use:

```text
correlationId
traceId
orderId
service
environment
```

Example:

```json id="t8wy1k"
{
  "level": "ERROR",
  "service": "payment-service",
  "correlationId": "abc123",
  "traceId": "7f8...",
  "orderId": "ORD-123",
  "message": "Payment provider timeout"
}
```

Then:

```text
Kibana
   |
   v
correlationId
   |
   +-- order-service
   +-- payment-service
   +-- inventory-service
```

---

# 85. Flash Sale Metrics

Track:

```text
RPS
P50
P95
P99
5xx
4xx
checkout success
payment success
inventory success
pod count
pending pods
node count
RDS connections
RDS latency
Redis hit ratio
Kafka lag
SQS message age
```

This becomes the **flash-sale command center**.

---

# 86. Pre-Sale Load Test

Never make the actual sale the first test.

Run:

```text
Test environment
```

with:

```text
500 RPS
1,000 RPS
2,500 RPS
5,000 RPS
7,500 RPS
10,000 RPS
```

Measure:

```text
breaking point
safe operating point
recovery time
```

---

# 87. Determine the Safe Operating Point

Suppose testing shows:

```text
5,000 RPS
```

is healthy.

At:

```text
7,000 RPS
```

RDS becomes unstable.

Then the platform's tested safe capacity is not:

```text
10,000 RPS
```

just because Kubernetes can create more Pods.

Capacity is constrained by the weakest critical dependency.

---

# 88. Bottleneck Mapping

Create a table:

| Layer           | Tested Capacity | Bottleneck         |
| --------------- | --------------: | ------------------ |
| CloudFront      |        measured | cache/origin       |
| ALB             |        measured | target latency     |
| catalog-service |        measured | CPU                |
| order-service   |        measured | DB                 |
| payment-service |        measured | provider           |
| Redis           |        measured | memory/ops         |
| RDS             |        measured | CPU/connections    |
| Kafka           |        measured | partitions/brokers |
| SQS workers     |        measured | downstream         |
| ELK             |        measured | ingestion/storage  |

This gives us a real capacity model.

---

# 89. Flash Sale Capacity Formula

For every service:

```text
Required Capacity
=
Expected Peak Load
+
Safety Headroom
+
Failure Headroom
+
Deployment Headroom
```

Then verify:

```text
Dependency Capacity
>=
Application Capacity
```

For example:

```text
Application:
500 Pods

Database:
cannot support connections generated by 500 Pods
```

Therefore the application capacity is artificially high and unsafe.

---

# 90. Flash Sale Recovery

After the event:

```text
Traffic decreases
```

Don't immediately scale everything down.

First verify:

```text
RPS
queue depth
Kafka lag
DB connections
Redis
payment provider
```

Then:

```text
reduce workers
reduce API replicas
reduce nodes
```

gradually.

---

# 91. Post-Sale Review

After the sale, collect:

```text
peak RPS
peak pods
peak nodes
peak RDS connections
peak RDS CPU
Redis hit ratio
Kafka lag
SQS backlog
P95/P99
5xx
checkout success
payment success
inventory failures
```

Compare:

```text
predicted
vs
actual
```

This improves the next event's capacity plan.

---

# 92. Flash Sale Game Day

Before a major event, conduct a game day.

Participants:

```text
DevOps
Platform
SRE
Backend
Security
Database
Product
Support
```

Simulate:

```text
traffic spike
RDS saturation
Redis failure
Kafka lag
payment provider timeout
bad deployment
node failure
```

Measure response.

---

# 93. Flash Sale Roles

Example:

```text
Incident Commander
        |
        +---- Platform Lead
        |
        +---- Application Lead
        |
        +---- Database Lead
        |
        +---- Messaging Lead
        |
        +---- Communications
```

One person should coordinate the incident rather than everyone independently changing infrastructure.

---

# 94. Change Freeze

Before a major sale:

```text
production deployments
```

may be restricted during a defined window.

Example:

```text
T-24 hours
change freeze begins
```

except:

```text
emergency changes
```

according to the organization's change-management policy.

This reduces the probability of introducing an unrelated failure during the event.

---

# 95. Flash Sale Rollback Plan

Before the event, define:

```text
application rollback
database rollback/forward-fix
feature flag rollback
configuration rollback
infrastructure rollback
```

Important:

> Database migrations should already be backward compatible.

Do not rely on restoring the entire database as the first response to an application deployment issue.

---

# 96. Flash Sale Architecture — Complete

```text
                              USERS
                                |
                                v
                        CloudFront / WAF
                                |
                         Rate Limiting
                                |
                                v
                               ALB
                                |
             +------------------+------------------+
             |                  |                  |
            AZ-A               AZ-B               AZ-C
             |                  |                  |
         EKS Nodes          EKS Nodes          EKS Nodes
             |                  |                  |
             +------------------+------------------+
                                |
                       21 Microservices
                                |
          +---------------------+---------------------+
          |                     |                     |
          v                     v                     v
        Redis                  RDS                Kafka/SQS
          |                     |                     |
          |              +------+------+
          |              |             |
          |           Primary       Replicas
          |
          +---------------------+---------------------+
                                |
                         External Providers
                                |
                                v
                       Observability Platform
                                |
          +---------------------+---------------------+
          |                     |                     |
      Prometheus              ELK                 Jaeger
          |                     |                     |
       Grafana               Kibana               Traces
          |
     Alertmanager
```

---

# 97. Flash Sale Scaling Flow

```text
                     TRAFFIC SPIKE
                           |
                           v
                    CloudFront/WAF
                           |
                  +--------+--------+
                  |                 |
             Cache HIT         Origin traffic
                  |                 |
                  |                 v
                  |                ALB
                  |                 |
                  |                 v
                  |              EKS
                  |                 |
                  |                HPA
                  |                 |
                  |          Pending Pods?
                  |                 |
                  |                 v
                  |             Karpenter
                  |                 |
                  |                 v
                  |              Nodes
                  |
                  v
               RESPONSE
```

Async path:

```text
Order Event
    |
    v
Kafka/SQS
    |
    v
KEDA
    |
    v
Consumers
    |
    v
Controlled downstream processing
```

---

# 98. Flash Sale Protection Layers

```text
Layer 1
CloudFront caching

Layer 2
WAF / rate limiting

Layer 3
ALB

Layer 4
HPA / pre-scaling

Layer 5
Karpenter

Layer 6
Redis caching

Layer 7
Database connection protection

Layer 8
Kafka/SQS buffering

Layer 9
KEDA-controlled workers

Layer 10
Circuit breakers / timeouts

Layer 11
Graceful degradation

Layer 12
Observability / incident response
```

This is the real defense-in-depth architecture.

---

# 99. Flash Sale Checklist

## T-24 Hours

```text
[ ] Change freeze
[ ] Capacity plan approved
[ ] Load-test results reviewed
[ ] Database capacity verified
[ ] Redis capacity verified
[ ] Kafka/SQS capacity verified
[ ] External API quotas verified
[ ] Dashboards verified
[ ] Alerts verified
[ ] Runbooks verified
```

## T-60 Minutes

```text
[ ] Pre-scale EKS
[ ] Pre-scale critical services
[ ] Warm caches
[ ] Verify RDS
[ ] Verify Redis
[ ] Verify Kafka
[ ] Verify SQS
```

## T-5 Minutes

```text
[ ] Confirm on-call
[ ] Confirm war room
[ ] Confirm deployment freeze
[ ] Confirm rollback
[ ] Confirm business metrics
```

## During Sale

```text
[ ] RPS
[ ] P95/P99
[ ] 5xx
[ ] Checkout success
[ ] Payment success
[ ] Inventory success
[ ] RDS
[ ] Redis
[ ] Kafka
[ ] SQS
[ ] Pending Pods
[ ] Nodes
```

## After Sale

```text
[ ] Drain backlog
[ ] Verify orders
[ ] Verify payments
[ ] Verify inventory
[ ] Gradually scale down
[ ] Capture metrics
[ ] Conduct review
[ ] Update capacity model
```

---

# 100. Final Mental Model

The most important lesson from this chapter is:

```text
Flash Sale
    |
    v
Don't simply add Pods
    |
    v
Protect the entire system
```

The correct architecture is:

```text
Predict
   |
   v
Pre-scale
   |
   v
Cache
   |
   v
Rate-limit
   |
   v
Autoscale
   |
   v
Queue
   |
   v
Protect dependencies
   |
   v
Gracefully degrade
   |
   v
Observe
   |
   v
Recover
```

The ultimate goal is not:

> **"Serve unlimited traffic."**

It is:

> **"Keep critical customer journeys healthy while controlling the load imposed on every dependency."**

---

# Traffic Architecture Phase Complete

We have now completed:

```text
131 — LOW TRAFFIC
        ↓
132 — MEDIUM TRAFFIC
        ↓
133 — HIGH TRAFFIC
        ↓
134 — TRAFFIC SPIKE / FLASH SALE
```

The next phase moves from **performance and scaling** into **Disaster Recovery**.

# 135 — EKS RECOVERY

We will simulate an actual EKS failure and recover the platform, covering:

```text
EKS cluster failure
EKS control-plane considerations
node-group failure
Karpenter failure
AZ/node loss
application recovery
ArgoCD recovery
GitOps repository recovery
ECR image availability
Secrets/IAM recovery
persistent volumes
RDS dependency
DNS
observability
RTO/RPO
backup strategy
restore validation
```

and, importantly, distinguish between:

```text
"recreating EKS"
```

and:

```text
"recovering the application platform."
```

That distinction is critical in real production DR.
