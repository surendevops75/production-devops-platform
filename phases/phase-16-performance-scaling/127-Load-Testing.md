# 127 — LOAD TESTING — PRODUCTION IMPLEMENTATION

> **Project:** Enterprise E-Commerce Platform
> **Environment:** AWS EKS
> **Primary Tool:** k6
> **Observability:** Prometheus + Grafana + ELK + OpenTelemetry + Jaeger
> **Database:** Amazon RDS PostgreSQL
> **Cache:** Amazon ElastiCache Redis/Valkey-compatible
> **Messaging:** SQS + SNS + RabbitMQ + Kafka
> **Autoscaling:** HPA + KEDA + Karpenter
> **Deployment:** ArgoCD + Helm + GitOps

---

# 1. Objective

Load testing answers:

> **“Can our platform handle the expected traffic without violating our performance and reliability targets?”**

For this 21-microservice e-commerce platform, load testing must validate more than application response time.

We need to understand:

```text
Traffic
   ↓
CloudFront / WAF
   ↓
ALB
   ↓
EKS
   ↓
21 Microservices
   ↓
Redis / RDS
   ↓
SQS / SNS / RabbitMQ / Kafka
   ↓
Downstream processing
```

And simultaneously observe:

```text
Prometheus → Metrics
Grafana    → Dashboards
ELK        → Logs
OpenTelemetry → Traces
Jaeger     → Trace investigation
```

The goal is to determine:

* maximum sustainable throughput
* latency under load
* error rate
* CPU saturation
* memory pressure
* pod scaling behavior
* node scaling behavior
* database connection pressure
* Redis performance
* queue backlog
* Kafka consumer lag
* ALB behavior
* bottlenecks
* failure points
* scaling limits

---

# 2. Load Testing Is Not the Same as Stress Testing

These terms should be separated.

### Load testing

Expected production traffic.

```text
100 RPS
200 RPS
500 RPS
```

Question:

> Can the system handle expected traffic?

### Stress testing

Push beyond expected capacity.

```text
500 RPS
750 RPS
1000 RPS
1500 RPS
```

Question:

> Where does the system start degrading?

### Spike testing

Sudden traffic increase.

```text
100 RPS
   ↓
1000 RPS
```

Question:

> How does the platform react to sudden traffic?

### Soak testing

Sustained traffic for a long period.

```text
500 RPS
500 RPS
500 RPS
500 RPS
...
```

Question:

> Does the system remain stable over time?

### Capacity testing

Determine the sustainable maximum.

```text
100 RPS
200 RPS
300 RPS
400 RPS
500 RPS
600 RPS
...
```

Question:

> What is our sustainable capacity?

These tests will later be used in:

```text
127 Load Testing
128 Performance Testing
129 Kubernetes Capacity Planning
130 21-Microservice Scaling
131 Low Traffic Architecture
132 Medium Traffic Architecture
133 High Traffic Architecture
134 Traffic Spike / Flash Sale
```

---

# 3. Our Primary Tool — k6

For this project, use **k6** as the primary load-testing tool.

Other tools can be used:

* JMeter
* Gatling
* Locust
* k6

But standardize the platform implementation around:

```text
k6
```

because our tests can be stored as code.

Example:

```text
performance/
├── k6/
│   ├── smoke/
│   ├── load/
│   ├── stress/
│   ├── spike/
│   ├── soak/
│   └── scenarios/
├── test-data/
├── thresholds/
├── reports/
└── README.md
```

This allows load tests to become part of the engineering process instead of a manual activity.

---

# 4. Load Testing Architecture

The basic architecture:

```text
                   ┌───────────────┐
                   │      k6       │
                   │ Load Generator│
                   └───────┬───────┘
                           │
                           ▼
                    ┌────────────┐
                    │    ALB     │
                    └─────┬──────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │       EKS        │
                 │                  │
                 │  Microservices   │
                 └───────┬──────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
          Redis         RDS        Messaging
                                      │
                              ┌───────┼────────┐
                              ▼       ▼        ▼
                             SQS    Kafka   RabbitMQ
```

Observability:

```text
                 ┌──────────────┐
                 │   Prometheus │
                 └──────┬───────┘
                        ▼
                    Grafana

Applications
    │
    ├── Logs ────────► Fluent Bit ─► Logstash ─► Elasticsearch ─► Kibana
    │
    └── Traces ──────► OpenTelemetry ─► Jaeger
```

---

# 5. Never Start With Production

Do not immediately run:

```text
1000 users
5000 users
10000 users
```

against production.

Start with:

```text
Developer environment
        ↓
Test environment
        ↓
Performance environment
        ↓
Staging
        ↓
Controlled production test
```

Production load testing requires:

* approved test window
* known traffic limits
* monitoring
* rollback plan
* application owner approval
* infrastructure owner approval
* database monitoring
* security approval where required
* customer-impact safeguards

---

# 6. Dedicated Performance Environment

Ideally create:

```text
performance
```

or:

```text
perf
```

environment.

Example:

```text
AWS Account
   │
   └── Performance Environment
          │
          ├── VPC
          ├── EKS
          ├── RDS
          ├── Redis
          ├── SQS
          ├── Kafka
          └── Observability
```

The performance environment should resemble production sufficiently to make results meaningful.

For example:

```text
Production:
3 AZ
10 nodes
RDS Multi-AZ
Redis replicas
Kafka cluster

Performance:
3 AZ
similar instance families
similar RDS class
similar Redis architecture
similar application configuration
```

You don't necessarily need production-sized capacity initially.

The important point is to document differences.

---

# 7. Test Environment Differences

Create a comparison document.

Example:

| Component  | Production          | Performance       |
| ---------- | ------------------- | ----------------- |
| EKS AZs    | 3                   | 3                 |
| Node type  | m7i.large           | m7i.large         |
| RDS        | production class    | smaller initially |
| Redis      | production topology | similar topology  |
| Kafka      | production topology | reduced capacity  |
| Replicas   | production          | controlled        |
| HPA        | enabled             | enabled           |
| KEDA       | enabled             | enabled           |
| Karpenter  | enabled             | enabled           |
| Monitoring | enabled             | enabled           |

Never claim:

> "Performance environment is identical to production"

unless it actually is.

Instead document:

```text
Performance environment is representative of production
with the following capacity differences...
```

---

# 8. Define the Business Workload

Don't generate random requests.

Real users don't do:

```text
GET /health
GET /health
GET /health
```

for 30 minutes.

Our e-commerce workload should resemble actual customer behavior.

Example:

```text
User opens website
        ↓
Browse catalog
        ↓
Search product
        ↓
View product
        ↓
Add to cart
        ↓
View cart
        ↓
Create order
        ↓
Payment
        ↓
Order confirmation
```

---

# 9. Workload Distribution

Example workload:

```text
1000 virtual users
```

could generate:

```text
40% Browse catalog
20% Search
15% Product details
10% Cart
5% Login
5% Order
5% Other
```

This is only an example.

Actual percentages should come from:

* production access logs
* ALB metrics
* API gateway metrics
* business analytics
* application metrics
* historical traffic

Don't invent traffic distribution for capacity planning if production data is available.

---

# 10. Example User Journey

A realistic test:

```text
Login
 ↓
Get customer profile
 ↓
Search product
 ↓
Get product
 ↓
Get pricing
 ↓
Add product to cart
 ↓
Get cart
 ↓
Create order
 ↓
Check order
```

This creates cross-service traffic.

For example:

```text
order-service
   │
   ├── customer-service
   ├── product-service
   ├── pricing-service
   ├── promotion-service
   └── tax-service
```

Then:

```text
order-service
     ↓
PostgreSQL
     ↓
Outbox
     ↓
Kafka
     ↓
inventory-service
payment-service
fraud-service
notification-service
reporting-service
```

Now we're testing the actual architecture.

---

# 11. First Test — Smoke Test

Before load testing:

```text
1–5 virtual users
```

Example:

```text
10 requests/sec
```

Duration:

```text
1–2 minutes
```

Purpose:

* validate endpoint
* validate authentication
* validate test data
* validate network
* validate metrics
* validate logs
* validate traces
* verify no obvious errors

Do not start with thousands of users.

---

# 12. Basic k6 Test

Create:

```text
performance/k6/smoke/order-smoke.js
```

Example:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 5,
  duration: '1m',

  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};

export default function () {

  const response = http.get(
    'https://api.example.com/api/v1/products'
  );

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

Run:

```bash
k6 run order-smoke.js
```

---

# 13. Why Thresholds Matter

A test should not simply say:

```text
Test completed successfully
```

We need defined acceptance criteria.

Example:

```javascript
thresholds: {
  http_req_failed: ['rate<0.01'],
  http_req_duration: ['p(95)<500'],
}
```

Meaning:

```text
Error rate < 1%

AND

95th percentile latency < 500ms
```

If thresholds fail:

```text
k6 exit status != 0
```

This becomes useful for CI/CD.

---

# 14. P50, P95 and P99

Suppose we receive:

```text
1000 requests
```

Response times:

```text
P50 = 120 ms
P95 = 350 ms
P99 = 700 ms
```

Interpretation:

### P50

50% of requests completed faster than:

```text
120 ms
```

### P95

95% completed faster than:

```text
350 ms
```

### P99

99% completed faster than:

```text
700 ms
```

P99 is particularly useful for identifying tail latency.

A service can have:

```text
P50 = 100 ms
P95 = 200 ms
P99 = 3 seconds
```

and still have serious user-impacting tail latency.

---

# 15. Load Test

After smoke testing, run a controlled load test.

Example:

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 50 },
    { duration: '10m', target: 100 },
    { duration: '10m', target: 200 },
    { duration: '10m', target: 300 },
    { duration: '5m', target: 0 },
  ],

  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};
```

Traffic pattern:

```text
0
│
│       ┌──────────────
│      /
│     /
│    /
│___/
│
└──────────────────────── time
```

This allows us to observe scaling.

---

# 16. What We Monitor During the Test

Never look only at k6.

Monitor:

### Application

```text
RPS
P50
P95
P99
5xx
4xx
timeouts
```

### Kubernetes

```text
Pod CPU
Pod memory
Pod count
Pod restarts
Pending pods
HPA
```

### Nodes

```text
CPU
Memory
Disk
Network
Node count
```

### ALB

```text
RequestCount
TargetResponseTime
HTTPCode_Target_5XX_Count
HealthyHostCount
```

### RDS

```text
CPU
Connections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
FreeStorage
Database locks
```

### Redis

```text
CPU
Memory
Connections
Cache hits
Cache misses
Evictions
Network
```

### SQS

```text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
ApproximateAgeOfOldestMessage
```

### Kafka

```text
Consumer lag
Partition throughput
Under-replicated partitions
ISR
Broker CPU
Broker disk
```

---

# 17. Prometheus During Load Testing

Prometheus should answer:

> What happened inside Kubernetes?

Example:

```promql
sum(rate(http_server_requests_seconds_count[5m]))
```

Application throughput.

CPU:

```promql
sum(rate(container_cpu_usage_seconds_total[5m]))
```

Pod restarts:

```promql
increase(kube_pod_container_status_restarts_total[15m])
```

Memory:

```promql
sum(container_memory_working_set_bytes)
```

HTTP errors:

```promql
sum(rate(http_server_requests_seconds_count{
  status=~"5.."
}[5m]))
```

---

# 18. Grafana Load-Test Dashboard

Create:

```text
Load Testing Dashboard
```

Panels:

```text
┌────────────────────────────────────┐
│ Total RPS                          │
├──────────────────┬─────────────────┤
│ P50 Latency      │ P95 Latency     │
├──────────────────┼─────────────────┤
│ P99 Latency      │ Error Rate      │
├──────────────────┴─────────────────┤
│ Pod Count                           │
├────────────────────────────────────┤
│ CPU Utilization                     │
├────────────────────────────────────┤
│ Memory Utilization                  │
├────────────────────────────────────┤
│ HPA Replicas                        │
├────────────────────────────────────┤
│ Node Count                          │
├────────────────────────────────────┤
│ RDS Connections                     │
├────────────────────────────────────┤
│ Redis Hit/Miss                      │
├────────────────────────────────────┤
│ SQS Backlog                         │
├────────────────────────────────────┤
│ Kafka Consumer Lag                  │
└────────────────────────────────────┘
```

This dashboard should be open before the test starts.

---

# 19. Kubernetes Scaling During Load

Suppose:

```text
Initial replicas = 3
```

Traffic increases.

HPA observes:

```text
CPU > target
```

Then:

```text
3 pods
 ↓
5 pods
 ↓
8 pods
 ↓
12 pods
```

But if the cluster has insufficient capacity:

```text
HPA
 ↓
creates pods
 ↓
pods Pending
 ↓
Karpenter
 ↓
creates node
 ↓
pods scheduled
```

Our test must therefore observe:

```text
Load
 ↓
HPA
 ↓
Pending Pods
 ↓
Karpenter
 ↓
New Nodes
 ↓
Scheduling
```

---

# 20. HPA Scaling Delay

Autoscaling is not instantaneous.

Example:

```text
10:00:00
Traffic increases

10:00:30
CPU becomes high

10:01:00
HPA detects high utilization

10:01:30
New pods created

10:02:00
Pods become Ready
```

During this period:

```text
latency may increase
```

This is why spike testing is important.

---

# 21. Karpenter During Load Testing

If Karpenter is configured:

```text
Pods Pending
      ↓
Karpenter observes unschedulable pods
      ↓
Provisioner/NodePool constraints
      ↓
New EC2 node
      ↓
Node joins EKS
      ↓
Pods scheduled
```

Monitor:

```bash
kubectl get nodes -w
```

and:

```bash
kubectl get pods -A -w
```

Also inspect:

```bash
kubectl describe pod <pod>
```

if pods remain Pending.

---

# 22. Database Is Usually a Critical Bottleneck

Imagine:

```text
500 RPS
```

and:

```text
10 application pods
```

Each pod has:

```text
maxPoolSize = 20
```

Potential maximum application connections:

```text
10 × 20 = 200
```

If HPA scales to:

```text
30 pods
```

then:

```text
30 × 20 = 600
```

Now the application may overwhelm RDS.

Therefore:

> More pods does not automatically mean more database capacity.

Load testing must test:

```text
Application scaling
+
Database capacity
```

together.

---

# 23. Connection Pool Monitoring

For Spring Boot/Hikari:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 2000
      idle-timeout: 30000
      max-lifetime: 1800000
```

The exact values must be capacity-tested.

Monitor:

```text
Active connections
Idle connections
Pending connection requests
Connection acquisition time
Database CPU
Query latency
```

---

# 24. Redis Under Load

Suppose:

```text
1000 RPS
```

Product requests may be:

```text
GET product:123
GET product:456
GET product:789
```

With cache hits:

```text
Application
   ↓
Redis
```

instead of:

```text
Application
   ↓
RDS
```

Measure:

```text
Cache hit ratio
Cache miss ratio
Redis latency
Redis connections
Redis CPU
Redis memory
Evictions
```

---

# 25. Cache Failure During Load

An important test:

```text
Load
 ↓
Redis
 ↓
Redis becomes unavailable
```

Application should:

```text
Redis timeout
 ↓
Fallback to RDS
```

But this creates:

```text
1000 RPS
 ↓
RDS
```

instead of:

```text
1000 RPS
 ↓
Redis
```

This can overload RDS.

Therefore test:

```text
Redis failure + normal traffic
```

and:

```text
Redis failure + high traffic
```

This connects directly to Chapter 111.

---

# 26. SQS Under Load

Suppose:

```text
Order API = 500 orders/sec
```

Orders generate events:

```text
OrderCreated
```

Consumers process:

```text
Inventory
Payment
Notification
Reporting
```

If producers are faster than consumers:

```text
Producer
  ↓
SQS
  ↓
████████████████
Queue backlog
```

Monitor:

```text
Visible messages
Oldest message age
Consumer throughput
DLQ messages
```

The API itself may remain fast while asynchronous processing falls behind.

That means:

> HTTP latency alone cannot determine whether the platform is healthy.

---

# 27. Kafka Under Load

For Kafka:

```text
Producer throughput
       ↓
Kafka
       ↓
Consumer groups
```

If consumers cannot keep up:

```text
Producer
   ↓
Kafka
   ↓
████████████
Consumer lag
```

Monitor:

```text
Consumer lag
Messages/sec
Partition utilization
Broker CPU
Broker disk
ISR
Under-replicated partitions
```

A load test should include asynchronous completion checks.

---

# 28. End-to-End Order Load Test

For our platform:

```text
Customer
   ↓
ALB
   ↓
order-service
   ↓
customer-service
   ↓
product-service
   ↓
pricing-service
   ↓
promotion-service
   ↓
tax-service
   ↓
PostgreSQL
   ↓
Outbox
   ↓
Kafka
   ↓
inventory-service
   ↓
payment-service
   ↓
fraud-service
   ↓
notification-service
```

The test should verify both:

### Synchronous

```text
POST /orders
```

and:

### Asynchronous

```text
OrderCreated
 ↓
Inventory processed
 ↓
Payment processed
 ↓
Notification generated
```

---

# 29. Correlation ID

Every load-test transaction should have correlation information.

Example:

```text
X-Correlation-ID:
loadtest-123456
```

Then search:

```text
Kibana
```

and:

```text
Jaeger
```

for the same transaction.

This allows:

```text
k6 request
 ↓
ALB
 ↓
order-service
 ↓
payment-service
 ↓
Kafka
 ↓
inventory-service
```

to be investigated.

---

# 30. Trace Analysis

Suppose k6 reports:

```text
P95 = 1.8 seconds
```

Don't immediately blame Kubernetes.

Open Jaeger.

Example trace:

```text
order-service
   100 ms
      │
      ├── customer-service 50 ms
      │
      ├── product-service 80 ms
      │
      ├── pricing-service 100 ms
      │
      ├── tax-service 900 ms   ← bottleneck
      │
      └── database 300 ms
```

Now we know where the latency originated.

---

# 31. Logs During Load

Search Kibana for:

```text
service.name:"order-service"
```

Then:

```text
level:ERROR
```

or:

```text
status:500
```

Also search:

```text
correlationId:"loadtest-123456"
```

Look for:

```text
TimeoutException
ConnectionTimeout
PoolExhausted
SQLException
RedisConnectionFailure
KafkaTimeout
SQS errors
```

---

# 32. Test Data Management

Load testing with the same user is unrealistic.

Create controlled test data.

Example:

```text
100,000 test customers
500,000 products
1,000,000 cart combinations
```

depending on the test objective.

Use:

```text
test-*
```

identifiers.

Example:

```text
loadtest-user-001
loadtest-user-002
```

Do not accidentally send production-like payment transactions to real payment systems.

For external integrations:

```text
Sandbox
Mock
Stub
Synthetic endpoint
```

should be used where appropriate.

---

# 33. Avoid Shared Test Data Collisions

Bad:

```text
1000 users
 ↓
same cart
 ↓
same product
```

This can produce artificial contention.

Better:

```text
VU 1 → User 1
VU 2 → User 2
VU 3 → User 3
...
```

Use deterministic test data.

Example:

```javascript
const userId = `loadtest-user-${__VU}`;
```

---

# 34. Load Test Scenario Types

Create separate tests.

```text
01-smoke
02-baseline
03-load
04-stress
05-spike
06-soak
07-capacity
08-failure-under-load
```

Example:

```text
performance/k6/
├── smoke/
│   └── smoke.js
├── baseline/
│   └── baseline.js
├── load/
│   └── normal-load.js
├── stress/
│   └── stress.js
├── spike/
│   └── spike.js
├── soak/
│   └── soak.js
├── capacity/
│   └── capacity.js
└── failure/
    ├── redis-failure.js
    ├── database-pressure.js
    └── queue-backlog.js
```

---

# 35. Baseline Test

Before testing scaling:

```text
10 RPS
```

for:

```text
10 minutes
```

Record:

```text
P50
P95
P99
Error rate
CPU
Memory
RDS connections
Redis
Queue lag
```

This becomes your baseline.

Example:

| Metric          | Baseline |
| --------------- | -------: |
| RPS             |       10 |
| P50             |   100 ms |
| P95             |   220 ms |
| P99             |   400 ms |
| Error rate      |     0.1% |
| Pod CPU         |      20% |
| RDS connections |       30 |
| Redis hit ratio |      95% |

These values are examples, not target production numbers.

---

# 36. Gradual Load Test

Example:

```text
10 RPS
 ↓
25 RPS
 ↓
50 RPS
 ↓
100 RPS
 ↓
200 RPS
 ↓
300 RPS
 ↓
500 RPS
```

At every stage record:

```text
Latency
Errors
CPU
Memory
Pods
Nodes
RDS
Redis
Messaging
```

---

# 37. Finding the Saturation Point

Suppose:

| RPS |   P95 | Error |
| --: | ----: | ----: |
| 100 | 180ms |  0.1% |
| 200 | 210ms |  0.2% |
| 300 | 250ms |  0.3% |
| 400 | 330ms |  0.5% |
| 500 | 550ms |  1.2% |
| 600 |  1.4s |  4.8% |

You would investigate the transition around:

```text
400 → 500 RPS
```

rather than simply saying:

> "500 RPS failed."

The important question is:

> What resource became constrained?

---

# 38. Bottleneck Investigation

Possible bottlenecks:

```text
ALB
 ↓
EKS
 ↓
Application
 ↓
CPU
 ↓
Memory
 ↓
RDS
 ↓
Redis
 ↓
Kafka
 ↓
SQS
 ↓
External API
```

Investigate from top to bottom.

---

# 39. CPU Bottleneck

Symptoms:

```text
CPU > 80–90%
P95 increasing
HPA scaling
```

Check:

```bash
kubectl top pods -A
```

Then:

```bash
kubectl top nodes
```

Check HPA:

```bash
kubectl get hpa -A
```

Then inspect:

```bash
kubectl describe hpa <name>
```

Possible solutions:

```text
Optimize application
Increase resources
Scale replicas
Improve HPA
Optimize expensive code
Use caching
```

---

# 40. Memory Bottleneck

Symptoms:

```text
Memory increasing
OOMKilled
Pod restarts
GC pressure
```

Check:

```bash
kubectl get pods -A
```

Then:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Reason: OOMKilled
```

For Java services also investigate:

```text
JVM heap
GC pauses
heap allocation
off-heap usage
container limits
```

---

# 41. Database Bottleneck

Symptoms:

```text
Application latency increases
RDS CPU high
connections high
queries slow
```

Check:

```text
RDS Performance Insights
CloudWatch metrics
pg_stat_activity
slow queries
locks
```

Potential causes:

```text
Missing index
Connection pool too large
Too many pods
Slow SQL
Lock contention
Insufficient RDS capacity
```

Do not automatically solve:

```text
database slow
```

by increasing RDS size.

First identify the cause.

---

# 42. Redis Bottleneck

Check:

```text
CPU
memory
connections
evictions
cache hit ratio
latency
```

Potential problems:

```text
hot key
large values
poor TTL
cache stampede
high miss rate
connection storm
```

---

# 43. Messaging Bottleneck

SQS:

```text
queue age ↑
backlog ↑
consumer throughput ↓
```

Kafka:

```text
consumer lag ↑
```

RabbitMQ:

```text
queue depth ↑
consumer throughput ↓
```

This means asynchronous processing is becoming slower than ingestion.

---

# 44. Spike Test

A spike test simulates:

```text
Flash sale
Breaking news
Marketing campaign
Product launch
```

Example:

```text
100 RPS
   ↓
   ↓
1000 RPS
   ↓
   ↓
100 RPS
```

k6:

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '30s', target: 500 },
    { duration: '30s', target: 1000 },
    { duration: '5m', target: 1000 },
    { duration: '2m', target: 50 },
    { duration: '2m', target: 0 },
  ],
};
```

The exact rate must be derived from the test objective and environment capacity.

---

# 45. What Spike Testing Reveals

Spike testing exposes:

```text
HPA delay
Karpenter delay
cold starts
connection storms
cache misses
database pressure
queue backlog
load balancer behavior
```

Example:

```text
Traffic spike
    ↓
New pods required
    ↓
Pods Pending
    ↓
Karpenter provisioning
    ↓
Node joins
    ↓
Pods start
    ↓
Application warms
```

During this period latency may temporarily increase.

---

# 46. Soak Test

Example:

```text
500 RPS
```

for:

```text
2–4 hours
```

or longer depending on the test objective.

Look for:

```text
memory leak
connection leak
file descriptor leak
log growth
disk growth
queue growth
Kafka lag
RDS connection accumulation
Redis memory growth
```

A system can survive:

```text
500 RPS for 10 minutes
```

but fail after:

```text
500 RPS for 4 hours
```

---

# 47. Capacity Test

Capacity testing determines sustainable throughput.

Example:

```text
100 RPS
200 RPS
300 RPS
400 RPS
500 RPS
600 RPS
700 RPS
```

At each step:

```text
10–15 minutes
```

Then determine:

```text
Sustainable capacity
```

based on:

```text
SLO
error rate
latency
resource saturation
database limits
queue health
```

Not merely:

```text
CPU < 100%
```

---

# 48. Capacity Is Not One Number

The platform may have different limits.

Example:

```text
API capacity          = 500 RPS
RDS capacity          = 450 RPS
Redis capacity        = 2000 RPS
Kafka capacity        = 5000 msg/sec
Payment provider      = 100 TPS
```

The actual business workflow capacity may therefore be constrained by:

```text
payment provider
```

rather than Kubernetes.

This is why end-to-end testing is important.

---

# 49. External Dependencies

Never blindly load-test external services.

Examples:

```text
Payment gateway
Tax API
SMS provider
Email provider
Shipping provider
Fraud API
```

Use:

```text
sandbox
mock
stub
rate-limited integration
```

where appropriate.

Otherwise your load test may:

```text
trigger provider limits
create costs
cause account suspension
generate fake transactions
```

---

# 50. k6 Checks vs Thresholds

A check:

```javascript
check(response, {
  'status is 200': (r) => r.status === 200,
});
```

records whether the condition passed.

A threshold:

```javascript
thresholds: {
  http_req_failed: ['rate<0.01'],
}
```

determines whether the overall test should pass/fail.

Use both.

---

# 51. Example Production-Style k6 Test

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

const BASE_URL = __ENV.BASE_URL;

export const options = {
  stages: [
    { duration: '5m', target: 50 },
    { duration: '10m', target: 100 },
    { duration: '10m', target: 200 },
    { duration: '10m', target: 300 },
    { duration: '5m', target: 0 },
  ],

  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};

export default function () {

  const response = http.get(
    `${BASE_URL}/api/v1/products`
  );

  check(response, {
    'status 200': (r) => r.status === 200,
    'latency < 500ms': (r) =>
      r.timings.duration < 500,
  });

  sleep(1);
}
```

Run:

```bash
k6 run \
  -e BASE_URL=https://performance-api.example.com \
  normal-load.js
```

---

# 52. Authentication

Real APIs may require authentication.

Example:

```javascript
const loginResponse = http.post(
  `${BASE_URL}/api/v1/auth/login`,
  JSON.stringify({
    username: `loadtest-${__VU}`,
    password: 'test-password'
  }),
  {
    headers: {
      'Content-Type': 'application/json'
    }
  }
);

const token = loginResponse.json('token');
```

Then:

```javascript
const params = {
  headers: {
    Authorization: `Bearer ${token}`,
  },
};

http.get(
  `${BASE_URL}/api/v1/orders`,
  params
);
```

Do not hardcode real credentials.

Use:

```text
environment variables
secret store
dedicated test credentials
```

---

# 53. Don't Login on Every Request

A common bad test:

```text
Every API request
    ↓
Login
    ↓
Get token
    ↓
API request
```

This creates artificial authentication load.

Instead:

```text
VU
 ↓
authenticate
 ↓
reuse token
 ↓
perform user journey
```

unless the specific test is intentionally testing the authentication service.

---

# 54. Load Generator Capacity

The load generator itself can become a bottleneck.

Suppose you want:

```text
10,000 RPS
```

but your k6 machine can generate only:

```text
4,000 RPS
```

Then you are measuring:

```text
k6 generator capacity
```

rather than:

```text
EKS capacity
```

Therefore monitor the load generator:

```text
CPU
Memory
Network
connections
```

For large tests, use distributed load generation.

---

# 55. Distributed Load Generation

Architecture:

```text
                 ┌──────────────┐
                 │ Test Control │
                 └───────┬──────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       k6 Gen 1       k6 Gen 2       k6 Gen 3
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                        ALB
```

Possible deployment:

```text
Dedicated performance VPC
```

or:

```text
Kubernetes-based load generators
```

depending on scale.

Keep load generators separate from production application capacity.

---

# 56. Load Testing From Inside EKS

For internal services, we can run a controlled k6 Job.

Example concept:

```text
k6 Job
   ↓
Internal ALB / Service
   ↓
Microservices
```

This is useful for:

```text
internal API
service-to-service
private endpoints
```

But for realistic customer traffic:

```text
external load generator
        ↓
CloudFront/WAF/ALB
```

may be more representative.

---

# 57. Kubernetes Job Model

Conceptual:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-load-test
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: k6
          image: grafana/k6:<PINNED_VERSION>
          command:
            - k6
            - run
            - /scripts/load.js
          volumeMounts:
            - name: scripts
              mountPath: /scripts
      volumes:
        - name: scripts
          configMap:
            name: k6-load-test
```

**Important:** Pin the k6 image version rather than using:

```text
latest
```

in controlled environments.

---

# 58. Load-Test Results

Every test should generate a report containing:

```text
Test name
Environment
Git commit
Application version
Infrastructure version
Start time
End time
Duration
Virtual users
RPS
P50
P95
P99
Error rate
CPU
Memory
Pods
Nodes
RDS connections
Redis
SQS
Kafka
Observations
Bottlenecks
Actions
```

Example:

```text
Test:
Order API Load Test

Environment:
performance

Application:
order-service 1.4.2

Git SHA:
abc1234

Peak RPS:
300

P95:
420 ms

P99:
780 ms

Error:
0.4%

Peak pods:
12

Peak nodes:
5

RDS connections:
180

Kafka lag:
1200
```

---

# 59. Correlate Test With Deployment

Record:

```text
Application Git SHA
Docker image
ECR digest
Helm version
ArgoCD revision
Terraform revision
```

Example:

```text
Application:
order-service

Git:
abc1234

Image:
sha256:...

Helm:
1.8.2

ArgoCD:
abc1234
```

Otherwise six months later you won't know which version produced the test result.

---

# 60. Load Testing + GitOps

Recommended structure:

```text
gitops/
├── applications/
├── environments/
├── platform/
└── performance/
    ├── k6/
    │   ├── smoke/
    │   ├── load/
    │   ├── stress/
    │   ├── spike/
    │   └── soak/
    └── README.md
```

The test code itself can be version-controlled.

---

# 61. CI Integration

Load testing can be integrated into CI/CD.

Example:

```text
Developer
   ↓
PR
   ↓
Unit Tests
   ↓
Security
   ↓
Build
   ↓
Deploy Performance Environment
   ↓
Smoke Load Test
   ↓
Threshold
   ↓
Continue / Fail
```

But don't run a huge:

```text
4-hour soak test
```

on every PR.

Use:

```text
PR → small smoke/performance test
Nightly → load test
Scheduled → stress/soak
Release → capacity validation
```

---

# 62. Example GitHub Actions

```yaml
name: Performance Test

on:
  workflow_dispatch:

jobs:
  k6:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run k6
        uses: grafana/k6-action@v0.3.1
        with:
          filename: performance/k6/load/normal-load.js
        env:
          BASE_URL: ${{ secrets.PERFORMANCE_BASE_URL }}
```

For an actual production implementation:

* pin action versions
* use approved runners
* use environment protection
* avoid exposing credentials
* restrict test environments
* record results

---

# 63. Test Gates

Example gate:

```text
P95 < 500 ms
Error rate < 1%
No CrashLoopBackOff
No OOMKilled
RDS connections < approved limit
Kafka lag returns toward baseline
SQS oldest-message age within limit
```

If the test violates a hard threshold:

```text
FAIL
```

But thresholds must be based on documented SLOs/capacity requirements, not arbitrary numbers.

---

# 64. Load Testing and SLO

Chapter 125 defined SLOs.

Now connect them.

Example:

```text
Availability SLO:
99.9%

Latency:
95% < 500 ms
```

Load test:

```text
500 RPS
```

Result:

```text
P95 = 700 ms
```

Then:

```text
Performance target violated
```

This becomes a capacity issue.

---

# 65. Error Budget During Testing

Don't confuse:

```text
controlled performance test
```

with:

```text
normal production traffic
```

If running a production test, explicitly account for the test's impact and approval.

Ideally:

```text
Performance testing
        ↓
isolated environment
```

so production error budget isn't unnecessarily consumed.

---

# 66. Failure Under Load

A powerful test is:

```text
Load
+
controlled failure
```

Example:

```text
300 RPS
+
Redis failure
```

or:

```text
300 RPS
+
one EKS node failure
```

or:

```text
300 RPS
+
Kafka consumer slowdown
```

Then measure:

```text
Customer latency
Error rate
Recovery time
Queue backlog
Database pressure
Autoscaling
```

This connects directly to:

```text
Incident Simulation
DR
Chaos Engineering
```

---

# 67. Example: Redis Failure During Load

Normal:

```text
300 RPS
 ↓
Redis
 ↓
RDS
```

Failure:

```text
300 RPS
 ↓
Redis unavailable
 ↓
RDS fallback
```

Observe:

```text
RDS CPU
RDS connections
API latency
5xx
Redis timeout
```

Then restore Redis.

Observe:

```text
cache recovery
cache warming
RDS recovery
latency normalization
```

---

# 68. Example: Node Failure During Load

Normal:

```text
10 nodes
30 pods
```

Controlled failure:

```text
1 node unavailable
```

Observe:

```text
Pods evicted
Pods Pending
HPA
Karpenter
New node
Pod scheduling
Application latency
```

The goal isn't merely:

> Node recovered.

The real question is:

> Did customer-facing SLOs remain within acceptable limits?

---

# 69. Load Test Safety Controls

Before every test:

```text
☑ Correct environment
☑ Correct endpoint
☑ Correct test account
☑ Test data verified
☑ External integrations sandboxed
☑ Monitoring open
☑ Alerts configured
☑ Database capacity checked
☑ Rollback available
☑ Test owner assigned
☑ Stop conditions defined
```

---

# 70. Stop Conditions

Define stop conditions before starting.

Example:

```text
Error rate > 5%
P95 > 5 seconds
RDS connections > approved ceiling
RDS CPU sustained > 90%
Kafka lag exceeds critical threshold
SQS backlog exceeds test boundary
Unexpected customer traffic impact
```

If a stop condition is reached:

```text
STOP LOAD
```

Don't continue simply to see how badly the system fails.

---

# 71. Immediate Stop

For k6, terminate the test process:

```bash
Ctrl+C
```

For Kubernetes Job:

```bash
kubectl delete job k6-load-test
```

Verify:

```bash
kubectl get pods
```

Then confirm:

```text
traffic returned to baseline
queues recovering
database recovering
CPU recovering
```

---

# 72. Common Mistake — Only Looking at CPU

Suppose:

```text
CPU = 40%
```

but:

```text
P95 = 2 seconds
```

Potential bottleneck:

```text
RDS
Redis
network
external API
lock contention
connection pool
thread pool
```

Therefore:

> CPU is only one signal.

---

# 73. Common Mistake — Only Looking at Average Latency

Average:

```text
200 ms
```

could hide:

```text
P99 = 4 seconds
```

Always monitor:

```text
P50
P95
P99
```

---

# 74. Common Mistake — Increasing HPA Too Quickly

Suppose:

```text
CPU high
```

and we increase:

```text
maxReplicas:
20 → 100
```

But RDS can support only:

```text
300 connections
```

The result may be:

```text
100 pods
 ↓
100 × connection pool
 ↓
RDS overload
 ↓
application failure
```

Scaling limits must therefore be coordinated across:

```text
HPA
Karpenter
RDS
Redis
Kafka
SQS
external dependencies
```

---

# 75. Common Mistake — Using Production Data

Never blindly copy:

```text
production customer data
production passwords
production tokens
payment information
PII
```

into performance environments.

Use:

```text
synthetic data
anonymized data
approved test fixtures
```

---

# 76. Common Mistake — Testing Only One Endpoint

Testing:

```text
GET /products
```

doesn't validate:

```text
order-service
payment-service
inventory-service
Kafka
RDS
Redis
```

Create both:

### Endpoint tests

```text
individual service capacity
```

and:

### Business journey tests

```text
complete user workflow
```

---

# 77. Test Matrix

Create a matrix:

| Test     |    Traffic | Duration | Purpose           |
| -------- | ---------: | -------: | ----------------- |
| Smoke    |       tiny |     1–2m | Validate          |
| Baseline |        low |      10m | Reference         |
| Load     |   expected |   30–60m | Normal capacity   |
| Stress   | increasing | variable | Find limits       |
| Spike    |     sudden |   10–20m | Burst behavior    |
| Soak     |  sustained |    hours | Stability         |
| Capacity | increasing | variable | Determine ceiling |
| Failure  | controlled | variable | Resilience        |

---

# 78. Performance Test Execution Flow

Use:

```text
1. Define workload
       ↓
2. Define SLOs
       ↓
3. Prepare environment
       ↓
4. Prepare test data
       ↓
5. Validate monitoring
       ↓
6. Run smoke
       ↓
7. Run baseline
       ↓
8. Run load
       ↓
9. Analyze bottleneck
       ↓
10. Tune
       ↓
11. Repeat
       ↓
12. Run stress
       ↓
13. Run spike
       ↓
14. Run soak
       ↓
15. Document capacity
```

---

# 79. Performance Tuning Loop

The correct process is:

```text
Measure
   ↓
Identify bottleneck
   ↓
Change ONE major variable
   ↓
Test
   ↓
Compare
   ↓
Validate
```

Avoid:

```text
Increase CPU
Increase replicas
Increase RDS
Increase Redis
Increase Kafka
Change HPA
Change JVM
```

all at once.

Otherwise you won't know which change helped.

---

# 80. Example Real Investigation

Suppose:

```text
Traffic = 400 RPS

P95 = 450 ms
P99 = 1.2 sec
Error = 0.5%
```

At:

```text
500 RPS
```

we see:

```text
P95 = 1.4 sec
P99 = 4 sec
Error = 3%
```

Metrics:

```text
Pod CPU = 60%
RDS CPU = 45%
Redis CPU = 35%
Kafka lag = normal
```

But:

```text
RDS connections = 95% of limit
```

And:

```text
Hikari pending threads ↑
```

Likely bottleneck:

```text
DB connection capacity
```

not:

```text
CPU
```

This is where distributed observability becomes valuable.

---

# 81. Trace Example

Jaeger might show:

```text
POST /orders
   |
   +-- customer-service 40ms
   |
   +-- product-service 60ms
   |
   +-- pricing-service 50ms
   |
   +-- tax-service 80ms
   |
   +-- PostgreSQL 900ms
```

Then PostgreSQL investigation:

```text
slow query
 ↓
missing index
 ↓
sequential scan
```

Fix:

```text
index
```

Repeat test.

Result:

```text
PostgreSQL = 150ms
P95 = 500ms
```

This is the correct performance engineering cycle.

---

# 82. Load Test Documentation

Store every test result.

Example:

```text
performance/
├── results/
│   ├── 2026-09-30-baseline/
│   ├── 2026-09-30-load/
│   └── 2026-09-30-stress/
├── reports/
│   ├── baseline.md
│   ├── load.md
│   └── stress.md
└── README.md
```

Each report should contain:

```text
Objective
Environment
Version
Test configuration
Traffic model
Results
Graphs
Bottleneck
Actions
Retest
Final conclusion
```

---

# 83. Production Capacity Record

After testing, create:

```text
capacity.md
```

Example structure:

```markdown
# Platform Capacity

## Environment

Production-like EKS

## Application Version

abc1234

## Sustainable API Throughput

TBD

## P95 Latency

TBD

## P99 Latency

TBD

## Maximum Tested RPS

TBD

## RDS Capacity

TBD

## Redis Capacity

TBD

## Kafka Throughput

TBD

## SQS Consumer Capacity

TBD

## Known Bottlenecks

- TBD

## Scaling Limits

- TBD
```

Never fill this with assumed numbers.

Populate it from actual test results.

---

# 84. Final Load Testing Architecture

```text
                         ┌─────────────────┐
                         │      k6         │
                         │ Load Generators │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ CloudFront/WAF  │
                         └────────┬────────┘
                                  │
                                  ▼
                              ┌───────┐
                              │  ALB  │
                              └───┬───┘
                                  │
                                  ▼
                       ┌────────────────────┐
                       │        EKS         │
                       │                    │
                       │  21 Microservices  │
                       │                    │
                       │ HPA / KEDA         │
                       │ Karpenter          │
                       └──────┬─────┬───────┘
                              │     │
                 ┌────────────┘     └──────────────┐
                 ▼                                 ▼
              Redis                               RDS
                 │                                 │
                 └──────────────┬──────────────────┘
                                │
                                ▼
                         Messaging Layer
                      ┌─────────┼─────────┐
                      ▼         ▼         ▼
                     SQS      Kafka    RabbitMQ
                      
Observability:

EKS ─────► Prometheus ─────► Grafana
 │
 ├───────► Fluent Bit ─► Logstash ─► Elasticsearch ─► Kibana
 │
 └───────► OpenTelemetry ─► Jaeger
```

---

# 85. Production Load Testing Checklist

## Planning

```text
☐ Workload defined
☐ Traffic model defined
☐ Test objective defined
☐ SLO defined
☐ Success criteria defined
☐ Stop criteria defined
```

## Environment

```text
☐ Performance environment available
☐ EKS capacity verified
☐ RDS configured
☐ Redis configured
☐ Messaging configured
☐ Test data prepared
```

## Observability

```text
☐ Prometheus healthy
☐ Grafana dashboard ready
☐ ELK healthy
☐ Jaeger healthy
☐ Alertmanager configured
```

## Execution

```text
☐ Smoke test
☐ Baseline
☐ Load
☐ Stress
☐ Spike
☐ Soak
☐ Capacity
```

## Analysis

```text
☐ P50
☐ P95
☐ P99
☐ Error rate
☐ CPU
☐ Memory
☐ Pods
☐ Nodes
☐ HPA
☐ Karpenter
☐ RDS
☐ Redis
☐ SQS
☐ Kafka
☐ Logs
☐ Traces
```

## Finalization

```text
☐ Results stored
☐ Bottleneck documented
☐ Capacity documented
☐ Changes documented
☐ Retest completed
☐ Runbook updated
```

---

# 86. What You Should Be Able to Explain After This Chapter

You should be able to explain:

### Why load testing?

```text
To validate capacity, latency, reliability and scaling
under representative traffic.
```

### What tool?

```text
k6
```

### What do you measure?

```text
RPS
P50/P95/P99
Error rate
CPU
Memory
Pods
Nodes
RDS
Redis
SQS
Kafka
```

### How do you find bottlenecks?

```text
Metrics
 ↓
Logs
 ↓
Traces
 ↓
Dependencies
 ↓
Infrastructure
```

### How does scaling work?

```text
Traffic
 ↓
HPA
 ↓
Pods
 ↓
Pending Pods
 ↓
Karpenter
 ↓
Nodes
```

### What happens when RDS becomes the bottleneck?

```text
More pods
      ↓
More DB connections
      ↓
RDS saturation
      ↓
Latency/errors
```

Therefore:

> **Application scaling must be coordinated with dependency capacity.**

---

# 87. Final Practical Exercise

Run this sequence in the performance environment:

```text
STEP 1
5 VUs
2 minutes
```

Then:

```text
STEP 2
10 RPS
10 minutes
```

Then:

```text
STEP 3
50 RPS
15 minutes
```

Then:

```text
STEP 4
100 RPS
15 minutes
```

Then:

```text
STEP 5
200 RPS
15 minutes
```

Then:

```text
STEP 6
Increase gradually
until the documented test boundary is reached
```

At every stage capture:

```text
RPS
P50
P95
P99
5xx
CPU
Memory
Pod count
Node count
RDS connections
RDS CPU
Redis hit ratio
SQS backlog
Kafka lag
```

Then investigate the first meaningful degradation using:

```text
Grafana
   ↓
Kibana
   ↓
Jaeger
```

Finally document:

```text
Observed capacity
Bottleneck
Evidence
Change made
Retest result
```

That becomes the foundation for **Chapter 128 — Performance Testing**, where we will go deeper into **application/JVM performance, database/query performance, Redis performance, Kubernetes performance, network performance, profiling, garbage collection, thread pools, connection pools, and systematic bottleneck optimization**.
