# 128 — PERFORMANCE TESTING — PRODUCTION IMPLEMENTATION

> **Project:** Enterprise E-Commerce Platform
> **Primary load tool:** k6
> **Application:** 21 Java + Spring Boot microservices
> **Platform:** AWS EKS
> **Database:** Amazon RDS PostgreSQL
> **Cache:** Amazon ElastiCache Redis/Valkey-compatible
> **Messaging:** SQS + SNS + RabbitMQ + Kafka
> **Observability:** Prometheus + Grafana + ELK + OpenTelemetry + Jaeger
> **Deployment:** Helm + ArgoCD + GitOps

---

# 1. Objective

Chapter 127 answered:

> **How much traffic can the platform handle?**

This chapter answers a different question:

> **Why does performance degrade, and what exactly should we optimize?**

Performance testing goes deeper than load generation.

We will investigate:

```text
Application
   ↓
JVM
   ↓
Threads
   ↓
Connection pools
   ↓
Database queries
   ↓
Redis
   ↓
Network
   ↓
Kubernetes
   ↓
Messaging
   ↓
External dependencies
```

The objective is to identify bottlenecks and optimize them systematically.

---

# 2. Load Testing vs Performance Testing

These terms overlap, but for this project we will separate them.

## Load Testing

Example:

```text
100 RPS
200 RPS
500 RPS
```

Question:

> Can the platform handle this traffic?

## Performance Testing

We investigate:

```text
Why is P95 800 ms?

Why is CPU 40% but latency high?

Why are DB queries slow?

Why is JVM GC increasing?

Why are threads waiting?

Why is Kafka lag increasing?
```

So:

```text
Load Testing
    ↓
Find degradation
    ↓
Performance Testing
    ↓
Find root cause
    ↓
Optimization
    ↓
Retest
```

---

# 3. Performance Engineering Loop

The correct approach is:

```text
Measure
   ↓
Establish baseline
   ↓
Identify bottleneck
   ↓
Change one major variable
   ↓
Test again
   ↓
Compare
   ↓
Validate
```

Never blindly change ten things at once.

Bad:

```text
Increase CPU
Increase memory
Increase replicas
Increase RDS
Increase Redis
Change JVM
Change HPA
Change database
```

all at the same time.

Then you won't know which change actually helped.

---

# 4. Performance Testing Layers

For our platform:

```text
Layer 1
Application/API
        ↓
Layer 2
JVM
        ↓
Layer 3
Threads
        ↓
Layer 4
Connection pools
        ↓
Layer 5
Database
        ↓
Layer 6
Redis
        ↓
Layer 7
Messaging
        ↓
Layer 8
Kubernetes
        ↓
Layer 9
Network
        ↓
Layer 10
Infrastructure
```

Each layer needs different metrics.

---

# 5. Performance Test Types

We will use:

```text
1. Baseline Testing
2. Load Testing
3. Stress Testing
4. Spike Testing
5. Soak Testing
6. Capacity Testing
7. Scalability Testing
8. API Performance Testing
9. Database Performance Testing
10. JVM Performance Testing
11. Cache Performance Testing
12. Messaging Performance Testing
13. Kubernetes Performance Testing
```

---

# 6. Establish a Baseline

Before optimizing anything, record the current state.

Example:

```text
Test:
Order API

Traffic:
100 RPS

Duration:
15 minutes
```

Record:

```text
P50
P95
P99
Error rate
CPU
Memory
GC
Thread count
DB connections
DB latency
Redis latency
Kafka lag
SQS backlog
```

Example:

| Metric          | Baseline |
| --------------- | -------: |
| RPS             |      100 |
| P50             |   120 ms |
| P95             |   280 ms |
| P99             |   650 ms |
| Errors          |     0.2% |
| CPU             |      45% |
| Memory          |      55% |
| DB connections  |       80 |
| Redis hit ratio |      94% |

These are **illustrative values**, not project targets.

---

# 7. Golden Signals

For application performance, always watch:

```text
Latency
Traffic
Errors
Saturation
```

### Traffic

```text
requests/sec
```

### Latency

```text
P50
P95
P99
```

### Errors

```text
4xx
5xx
timeouts
exceptions
```

### Saturation

```text
CPU
memory
threads
connections
queue depth
```

---

# 8. Application Performance

Our Spring Boot architecture:

```text
Client
  ↓
ALB
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
PostgreSQL
```

A slow request could originate anywhere.

Example:

```text
POST /orders
      │
      ├── Controller      5 ms
      ├── Business logic 20 ms
      ├── Redis           10 ms
      ├── PostgreSQL      700 ms  ← problem
      └── Response        10 ms
```

Total:

```text
~745 ms
```

The application itself may not be CPU-bound.

---

# 9. Start With Endpoint-Level Testing

Test each important API independently.

Example:

```text
GET /products
GET /products/{id}
POST /cart
GET /cart
POST /orders
GET /orders/{id}
POST /payments
```

Create:

```text
performance/k6/api/
```

Example:

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 50,
  duration: '5m',

  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};

export default function () {

  const response = http.get(
    `${__ENV.BASE_URL}/api/v1/products`
  );

  check(response, {
    'status 200': r => r.status === 200,
  });
}
```

---

# 10. Response-Time Breakdown

k6 exposes request timing information.

Conceptually:

```text
Request
 │
 ├── DNS
 ├── TCP connection
 ├── TLS
 ├── waiting
 ├── receiving
 └── total duration
```

If total latency is high, determine which phase is responsible.

For internal application calls, OpenTelemetry and Jaeger become especially useful.

---

# 11. Distributed Tracing

Suppose:

```text
POST /orders
```

takes:

```text
2 seconds
```

Jaeger shows:

```text
order-service
│
├── customer-service    50ms
├── product-service     70ms
├── pricing-service     80ms
├── promotion-service   60ms
├── tax-service         90ms
└── PostgreSQL        1500ms  ← bottleneck
```

The next investigation should focus on PostgreSQL.

This prevents random Kubernetes tuning.

---

# 12. Spring Boot Performance Metrics

Spring Boot Actuator + Micrometer gives useful metrics.

Examples:

```text
HTTP requests
HTTP latency
JVM memory
JVM GC
threads
classes
process CPU
system CPU
```

Prometheus collects these.

Example endpoint:

```text
/actuator/prometheus
```

Keep actuator access appropriately restricted; it should not be publicly exposed just because the endpoint exists.

---

# 13. JVM Performance

Our services use:

```text
Java
Spring Boot
```

Therefore JVM behavior matters.

The JVM manages:

```text
Heap
Non-heap
Threads
Garbage Collection
Class loading
JIT compilation
```

Basic architecture:

```text
Spring Boot
     ↓
JVM
 ┌───┼─────────────┐
 │   │             │
Heap Threads       GC
```

---

# 14. Java Heap

The heap stores Java objects.

Example:

```text
JVM
 │
 └── Heap
      ├── objects
      ├── strings
      ├── collections
      └── application data
```

If heap usage grows excessively:

```text
Memory ↑
   ↓
GC ↑
   ↓
CPU ↑
   ↓
Latency ↑
```

Eventually:

```text
OOMKilled
```

can occur if the container reaches its memory limit.

---

# 15. JVM Memory vs Kubernetes Memory

This is extremely important.

Suppose Kubernetes says:

```yaml
resources:
  limits:
    memory: 1Gi
```

That does **not** mean:

```text
JVM heap = 1 GiB
```

The container also needs memory for:

```text
JVM heap
JVM native memory
thread stacks
metaspace
direct buffers
libraries
```

Therefore Java container memory must be sized deliberately.

---

# 16. Example Java Container

Conceptually:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "768Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

The JVM configuration must fit within the container budget.

Avoid blindly setting:

```text
-Xmx1g
```

inside a:

```text
1Gi
```

container.

That leaves almost no room for non-heap memory.

---

# 17. Garbage Collection

Garbage Collection removes objects that are no longer needed.

Conceptually:

```text
Application
   ↓
Creates objects
   ↓
Heap fills
   ↓
GC
   ↓
Unused objects removed
```

If allocation is excessive:

```text
Application
   ↓
Many objects
   ↓
Frequent GC
   ↓
CPU increases
   ↓
Latency increases
```

---

# 18. Symptoms of GC Problems

Look for:

```text
GC frequency ↑
GC pause time ↑
Heap utilization ↑
CPU ↑
P95/P99 latency ↑
```

Possible causes:

```text
Large object creation
Large JSON payloads
Excessive logging
Poor caching
Large collections
Memory leaks
```

---

# 19. Don't Immediately Change the Garbage Collector

A common mistake is:

> "Latency is high, let's change JVM GC settings."

First establish:

```text
Is GC actually the bottleneck?
```

Use:

```text
JVM metrics
GC logs
JFR
profiling
heap analysis
```

Only then tune JVM settings.

---

# 20. Java Flight Recorder

For deeper Java investigation, **Java Flight Recorder (JFR)** is extremely useful.

It can help analyze:

```text
CPU
threads
locks
GC
allocations
I/O
JVM events
```

Conceptually:

```text
Spring Boot
     ↓
JVM
     ↓
JFR recording
     ↓
Performance analysis
```

For production use, profiling should be controlled because configuration and recording overhead depend on the workload and Java version.

---

# 21. Thread Performance

Spring Boot applications use threads to process requests.

Simplified:

```text
Incoming requests
       ↓
Thread pool
       ↓
Controllers
       ↓
Services
```

Suppose:

```text
100 requests
```

but:

```text
20 threads
```

are available.

Requests may wait.

This creates:

```text
Thread pool saturation
       ↓
Queueing
       ↓
Latency
```

---

# 22. Thread Pool Bottleneck

Symptoms:

```text
CPU not necessarily high
P95 latency high
threads busy
requests waiting
```

This is a classic example of:

> Low CPU does not necessarily mean good performance.

The service may be waiting on:

```text
database
network
external API
locks
connection pool
```

---

# 23. Database Connection Pool

Our Spring Boot application uses HikariCP.

Conceptually:

```text
Application
     │
     ▼
HikariCP
 ┌───┼───┐
 │   │   │
 ▼   ▼   ▼
 DB connections
```

Example:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
```

The pool size must be capacity-tested.

---

# 24. Connection Pool Saturation

Suppose:

```text
maxPoolSize = 20
```

and all 20 connections are busy.

Request 21 waits.

```text
Request
   ↓
HikariCP
   ↓
20 busy connections
   ↓
WAIT
```

If the wait becomes large:

```text
P95 ↑
P99 ↑
timeouts ↑
```

This can happen while:

```text
CPU = 30%
```

---

# 25. Database Performance Testing

Database testing needs its own layer.

Measure:

```text
Query latency
Transactions/sec
Connections
CPU
IOPS
Storage latency
Locks
Buffer/cache behavior
```

For PostgreSQL, investigate:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

This helps identify active/waiting sessions.

---

# 26. Slow Queries

A typical problem:

```text
Application
 ↓
SELECT ...
 ↓
PostgreSQL
 ↓
Sequential scan
 ↓
Large table
 ↓
Slow response
```

Use:

```sql
EXPLAIN ANALYZE
```

to investigate query execution.

Example:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 123;
```

Do this carefully in production; `EXPLAIN ANALYZE` actually executes the query.

---

# 27. Missing Index

Suppose:

```sql
SELECT *
FROM orders
WHERE customer_id = 123;
```

and there is no appropriate index.

PostgreSQL may scan a large portion of the table.

An appropriate index could reduce work:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

But indexes also have:

```text
storage cost
write cost
maintenance cost
```

Therefore indexes should be based on query patterns and measured evidence.

---

# 28. N+1 Query Problem

A common application performance issue:

```text
Get 100 orders
     ↓
Query orders
     ↓
For each order:
    query customer
```

Instead of:

```text
1 query
```

you might produce:

```text
101 queries
```

This is the classic N+1 problem.

Under low traffic:

```text
looks fine
```

Under high traffic:

```text
database pressure
     ↓
latency
     ↓
connection exhaustion
```

---

# 29. Detect N+1

Use:

```text
Hibernate SQL logs
APM/traces
database query statistics
Jaeger
```

Jaeger might show:

```text
order-service
│
├── SQL query 1
├── SQL query 2
├── SQL query 3
├── ...
└── SQL query 101
```

That is a strong signal.

---

# 30. Database Connection Math

Suppose:

```text
order-service:
10 pods

maxPoolSize:
20
```

Potential connections:

```text
10 × 20
= 200
```

Now HPA scales:

```text
10 → 30 pods
```

Potential connections:

```text
30 × 20
= 600
```

If RDS cannot safely support that workload:

```text
Application scaling
      ↓
DB overload
```

Therefore:

> HPA configuration and database connection-pool configuration must be designed together.

---

# 31. Redis Performance

Redis should be tested independently.

Measure:

```text
GET latency
SET latency
hit ratio
miss ratio
memory
connections
CPU
evictions
network
```

Example:

```text
GET product:123
```

Expected flow:

```text
Application
   ↓
Redis
   ↓
Response
```

If Redis latency rises:

```text
API latency rises
```

---

# 32. Cache Hit Ratio

Suppose:

```text
1000 requests
```

and:

```text
950 cache hits
50 cache misses
```

Hit ratio:

```text
95%
```

Higher hit ratio can reduce database load, but the correct target depends on workload.

Don't optimize for a high hit ratio blindly if:

```text
cached data is inappropriate
TTL is too long
memory usage becomes excessive
```

---

# 33. Cache Stampede

Suppose:

```text
Product cache expires
```

At the same time:

```text
1000 requests
```

arrive.

All requests miss:

```text
1000 requests
     ↓
Redis MISS
     ↓
1000 database requests
```

This can overload RDS.

Solutions may include:

```text
Request coalescing
Distributed locking
Staggered TTLs
Cache warming
Stale-while-revalidate
```

depending on application requirements.

---

# 34. Messaging Performance

For asynchronous systems, latency is not just HTTP latency.

For SQS:

```text
Producer
   ↓
Queue
   ↓
Consumer
```

Measure:

```text
Producer rate
Consumer rate
Queue depth
Oldest message age
Processing time
DLQ
```

For Kafka:

```text
Producer
   ↓
Kafka
   ↓
Consumer
```

Measure:

```text
Producer throughput
Consumer throughput
Consumer lag
Partition distribution
Broker resources
```

---

# 35. Consumer Lag

Suppose:

```text
Producer = 10,000 msg/sec
Consumer = 8,000 msg/sec
```

Then:

```text
Lag increases
```

Even if:

```text
HTTP API = healthy
```

the overall business workflow is becoming slower.

Performance testing therefore needs:

```text
Synchronous metrics
+
Asynchronous metrics
```

---

# 36. Kubernetes Performance

Now investigate EKS.

Monitor:

```text
Pod CPU
Pod memory
Node CPU
Node memory
Pod startup time
Scheduling latency
Pending pods
HPA scaling
Karpenter provisioning
Network
```

A request can be slow because:

```text
Pod doesn't exist yet
```

during a scaling event.

---

# 37. Pod Startup Performance

When HPA scales:

```text
3 pods
 ↓
10 pods
```

new pods need:

```text
Scheduling
Image pull
Container startup
JVM startup
Spring Boot startup
Readiness
```

For Java services this can take meaningful time.

Therefore measure:

```text
HPA scale decision
       ↓
Pod created
       ↓
Pod Ready
```

---

# 38. Image Pull Performance

Large images slow pod startup.

Bad:

```text
1.5 GB image
```

Better:

```text
small runtime image
```

Our multi-stage Dockerfile helps:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

...

FROM eclipse-temurin:21-jre
```

The final image contains the runtime rather than the full build environment.

---

# 39. JVM Startup

Spring Boot startup itself can become part of scaling latency.

Observe:

```text
Pod created
 ↓
JVM starts
 ↓
Spring Boot initializes
 ↓
DB connection
 ↓
Redis connection
 ↓
Application ready
```

Readiness should not become `Ready` before the application is actually capable of serving traffic correctly.

---

# 40. Kubernetes Requests and Limits

Incorrect resources can affect performance.

Example:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

Requests influence scheduling.

Limits can constrain workloads.

CPU throttling can become a performance issue if limits are configured poorly.

Therefore monitor actual usage and throttling rather than blindly copying values.

---

# 41. CPU Throttling

A pod might report:

```text
CPU usage = 500m
```

but experience throttling because of its CPU limit.

Investigate:

```text
CPU usage
CPU throttling
request/limit
latency
```

If CPU throttling correlates with latency spikes, resource configuration needs investigation.

---

# 42. Network Performance

Network bottlenecks can occur between:

```text
ALB → EKS
EKS → Redis
EKS → RDS
EKS → SQS
EKS → Kafka
EKS → external API
```

Monitor:

```text
network throughput
connection count
timeouts
retransmissions where observable
DNS latency
TLS overhead
```

Don't assume every latency problem is CPU or database related.

---

# 43. DNS Performance

Microservices make many network calls.

Example:

```text
order-service
   ↓
product-service
```

DNS resolution occurs through the Kubernetes DNS architecture.

If CoreDNS becomes overloaded:

```text
service discovery
       ↓
slow
```

This can appear as application latency.

Monitor:

```text
CoreDNS CPU
CoreDNS memory
DNS request rate
DNS errors
```

---

# 44. External API Performance

Suppose:

```text
order-service
   ↓
tax-service
   ↓
external tax provider
```

External API:

```text
200ms
```

becomes:

```text
2 seconds
```

Then your service may become slow.

Use:

```text
timeout
retry
circuit breaker
bulkhead
fallback
```

appropriately.

Never configure unlimited retries.

---

# 45. Retry Storm

Bad configuration:

```text
Request
 ↓
External API
 ↓
Timeout
 ↓
Retry
 ↓
Timeout
 ↓
Retry
 ↓
Retry
```

Under high traffic:

```text
1000 requests
 ×
3 retries
 =
3000 downstream calls
```

This can amplify the incident.

Performance testing should deliberately test retry behavior.

---

# 46. Thread Pool + Connection Pool Interaction

A request may use:

```text
HTTP thread
   ↓
DB connection
   ↓
query
```

If DB connections are exhausted:

```text
HTTP threads
      ↓
waiting
      ↓
latency increases
```

Eventually:

```text
thread pool saturation
```

This creates cascading failure.

This is why performance analysis must examine the entire chain.

---

# 47. Performance Bottleneck Chain

A common real incident:

```text
Traffic ↑
   ↓
HPA ↑
   ↓
Pods ↑
   ↓
DB connections ↑
   ↓
RDS connections exhausted
   ↓
Requests wait for connection
   ↓
HTTP threads blocked
   ↓
P95 ↑
   ↓
Timeouts ↑
   ↓
Retries ↑
   ↓
More DB pressure
```

This is a **feedback loop**.

Simply increasing replicas can make it worse.

---

# 48. Performance Testing With Jaeger

During a test:

```text
k6
 ↓
POST /orders
```

Capture the trace.

In Jaeger:

```text
Trace
 ├── order-service
 ├── customer-service
 ├── product-service
 ├── pricing-service
 ├── tax-service
 ├── Redis
 └── PostgreSQL
```

Compare slow vs fast traces.

Example:

```text
Fast:
PostgreSQL = 40ms

Slow:
PostgreSQL = 900ms
```

Then investigate PostgreSQL rather than Kubernetes.

---

# 49. Performance Testing With ELK

Search logs by:

```text
correlationId
traceId
service
version
```

Example:

```text
correlationId:"loadtest-12345"
```

Look for:

```text
timeout
exception
connection pool
database
Redis
Kafka
```

The three pillars work together:

```text
Metrics
   ↓
Where is the problem?

Logs
   ↓
What happened?

Traces
   ↓
Where in the request path?
```

---

# 50. Performance Test Environment

Recommended:

```text
AWS
 │
 └── Performance Account/Environment
       │
       ├── VPC
       ├── EKS
       ├── RDS
       ├── Redis
       ├── SQS
       ├── SNS
       ├── Kafka/RabbitMQ
       ├── Prometheus
       ├── Grafana
       ├── ELK
       ├── OTel
       └── Jaeger
```

Load generation:

```text
Dedicated k6 environment
```

This gives us repeatable testing.

---

# 51. Performance Test Repository

Create:

```text
company-ecommerce-performance/
```

Structure:

```text
company-ecommerce-performance/
│
├── k6/
│   ├── smoke/
│   ├── baseline/
│   ├── load/
│   ├── stress/
│   ├── spike/
│   ├── soak/
│   └── capacity/
│
├── application/
│   ├── api/
│   ├── order/
│   └── checkout/
│
├── database/
│   ├── queries/
│   └── benchmarks/
│
├── dashboards/
│
├── reports/
│
├── test-data/
│
└── README.md
```

---

# 52. Performance Test Versioning

Every test must record:

```text
Application Git SHA
Docker image digest
Helm version
ArgoCD revision
Terraform version
EKS version
RDS configuration
Redis configuration
k6 version
Test script Git SHA
```

Example:

```text
Application:
order-service

Git:
abc1234

Image:
sha256:xxxx

k6:
<version used>

Test:
load-test-v3
```

This makes results reproducible.

---

# 53. Before and After Comparison

Suppose baseline:

```text
P95 = 900 ms
```

After adding an index:

```text
P95 = 300 ms
```

Create:

| Metric     | Before | After |
| ---------- | -----: | ----: |
| RPS        |    300 |   300 |
| P50        |  300ms | 100ms |
| P95        |  900ms | 300ms |
| P99        |   1.8s | 600ms |
| DB CPU     |    80% |   45% |
| Error rate |   1.5% |  0.2% |

Now the improvement is measurable.

---

# 54. Don't Optimize Only One Metric

Suppose:

```text
P95:
500ms → 300ms
```

but:

```text
RDS CPU:
50% → 95%
```

The optimization may have shifted the bottleneck.

Always compare:

```text
Latency
Errors
CPU
Memory
DB
Redis
Messaging
Cost
```

---

# 55. Performance vs Cost

A configuration may improve performance but increase cost significantly.

Example:

```text
3 pods → 30 pods
```

Latency improves.

But:

```text
EC2 cost ↑
RDS connections ↑
Network ↑
```

Therefore capacity planning must balance:

```text
Performance
Reliability
Scalability
Cost
```

The correct architecture depends on the required workload and SLOs.

---

# 56. Performance Test Acceptance Criteria

For each test define:

```text
Traffic target
Latency target
Error target
Resource ceiling
Database ceiling
Queue ceiling
Recovery expectation
```

Example structure:

```yaml
test:
  name: order-load

traffic:
  target_rps: 300

thresholds:
  p95_ms: <documented target>
  p99_ms: <documented target>
  error_rate: <documented target>

resources:
  max_cpu: <approved limit>
  max_memory: <approved limit>

database:
  max_connections: <approved limit>
```

Don't hardcode arbitrary numbers just because they look reasonable.

---

# 57. Performance Testing Workflow

The complete workflow is:

```text
                    ┌──────────────┐
                    │ Define SLO   │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Baseline     │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Load Test    │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Find Problem │
                    └──────┬───────┘
                           ▼
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
        Metrics           Logs           Traces
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                    Identify Bottleneck
                           │
                           ▼
                       Optimize
                           │
                           ▼
                        Retest
                           │
                           ▼
                      Compare
```

---

# 58. Real Example — Order Service

Suppose:

```text
Traffic = 300 RPS
```

Result:

```text
P95 = 1.2 sec
```

Prometheus:

```text
CPU = 45%
Memory = 50%
```

So CPU doesn't appear to be the bottleneck.

Jaeger:

```text
order-service
   ↓
PostgreSQL = 850ms
```

PostgreSQL:

```text
CPU = 70%
```

Query analysis:

```text
Sequential Scan
```

Investigate query.

Add appropriate index.

Retest:

```text
P95 = 350ms
```

Then:

```text
Database CPU = 40%
```

This is a successful performance optimization because we:

```text
Measured
 ↓
Located
 ↓
Changed
 ↓
Retested
```

---

# 59. Real Example — Java GC

Another scenario:

```text
P95 = 400ms
P99 = 3 seconds
```

Prometheus:

```text
CPU = 80%
```

Jaeger:

```text
No obvious downstream bottleneck
```

JVM:

```text
GC activity very high
```

Investigate:

```text
object allocation
heap
GC pauses
JFR
```

Suppose excessive object creation is found.

Optimize application code.

Retest.

If:

```text
P99:
3s → 800ms
```

then the optimization is supported by measured evidence.

---

# 60. Real Example — Connection Pool

Suppose:

```text
P95 = 2 seconds
CPU = 30%
```

Hikari:

```text
Active = 20
Pending = 50
```

Configuration:

```text
maxPoolSize = 20
```

Database query itself:

```text
50ms
```

The issue is not necessarily query execution.

The request is waiting for a connection.

Possible solution:

```text
query optimization
transaction duration reduction
pool tuning
DB capacity review
```

Do not simply increase:

```text
20 → 100
```

without checking database capacity.

---

# 61. Real Example — Kafka

Suppose API latency is:

```text
200ms
```

but order processing completion time is:

```text
20 seconds
```

Kafka:

```text
consumer lag ↑
```

Consumers:

```text
1000 msg/sec
```

Producers:

```text
3000 msg/sec
```

Therefore:

```text
producer > consumer
```

Increase consumer capacity if downstream dependencies can support it.

Potentially:

```text
KEDA
+
more consumers
+
partition capacity
```

But verify partitioning and downstream limits first.

---

# 62. Real Example — HPA + RDS

Suppose:

```text
300 RPS
```

HPA:

```text
5 pods
```

RDS:

```text
100 connections
```

Traffic increases:

```text
500 RPS
```

HPA:

```text
5 → 15 pods
```

Connection pool:

```text
20/pod
```

Potential DB connections:

```text
15 × 20
= 300
```

If RDS capacity is lower:

```text
DB saturation
```

Then scaling the application made the situation worse.

This is one of the most important lessons in production performance engineering.

---

# 63. Performance Optimization Priority

When a bottleneck is identified, investigate in this order:

```text
1. Correctness
2. Architecture
3. Expensive dependencies
4. Database queries
5. Network calls
6. Caching
7. Connection pools
8. Thread pools
9. JVM
10. Kubernetes resources
```

This isn't a universal ranking; it is a practical investigation sequence.

Don't start with JVM flags when the database query is taking 2 seconds.

---

# 64. Performance Testing Checklist

### Application

```text
☐ Endpoint latency
☐ Error rate
☐ Throughput
☐ Payload size
☐ Serialization
☐ External calls
```

### JVM

```text
☐ Heap
☐ GC
☐ Threads
☐ CPU
☐ JFR/profile where appropriate
```

### Database

```text
☐ Queries
☐ Indexes
☐ Connections
☐ Locks
☐ CPU
☐ I/O
☐ Slow queries
```

### Redis

```text
☐ Hit ratio
☐ Latency
☐ Memory
☐ Evictions
☐ Connections
```

### Messaging

```text
☐ Producer throughput
☐ Consumer throughput
☐ Queue depth
☐ Kafka lag
☐ DLQ
```

### Kubernetes

```text
☐ Pod CPU
☐ Pod memory
☐ Node capacity
☐ HPA
☐ Karpenter
☐ Scheduling
☐ Startup time
```

### Observability

```text
☐ Prometheus
☐ Grafana
☐ ELK
☐ Jaeger
☐ Correlation ID
```

---

# 65. Final Architecture

```text
                         k6
                          │
                          ▼
                         ALB
                          │
                          ▼
                  ┌───────────────┐
                  │      EKS      │
                  │               │
                  │ Spring Boot   │
                  │ 21 Services   │
                  └───────┬───────┘
                          │
             ┌────────────┼─────────────┐
             ▼            ▼             ▼
           Redis         RDS        Messaging
                                        │
                                  ┌─────┼─────┐
                                  ▼     ▼     ▼
                                 SQS  Kafka RabbitMQ

                    Observability
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
      Prometheus          ELK          OpenTelemetry
          │               │                │
          ▼               ▼                ▼
       Grafana          Kibana           Jaeger
```

The performance investigation flow is:

```text
k6
 ↓
Latency/error observed
 ↓
Prometheus
 ↓
Find affected service
 ↓
Jaeger
 ↓
Find slow dependency/span
 ↓
ELK
 ↓
Find application error/context
 ↓
Database/Redis/Messaging investigation
 ↓
Optimize
 ↓
k6 retest
 ↓
Compare results
```

---

# 66. What You Should Know Before Chapter 129

At this point, the distinction should be clear:

```text
127 — Load Testing
        ↓
Can the platform handle traffic?

128 — Performance Testing
        ↓
Why is the platform slow?

129 — Kubernetes Capacity Planning
        ↓
How much Kubernetes capacity do we actually need?
```

Chapter **129 — Kubernetes Capacity Planning** will connect the results from Chapters 127–128 to actual infrastructure sizing:

```text
RPS
 ↓
Requests per pod
 ↓
Required replicas
 ↓
CPU/memory requests
 ↓
Node capacity
 ↓
Node groups / Karpenter
 ↓
AZ distribution
 ↓
Pod density
 ↓
IP capacity
 ↓
EKS cluster capacity
 ↓
RDS/Redis dependency capacity
 ↓
Cost and headroom
```

That is where we'll turn performance-test results into a **real production EKS capacity model**.
