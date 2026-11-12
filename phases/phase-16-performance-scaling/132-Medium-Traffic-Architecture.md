# 132 — MEDIUM TRAFFIC ARCHITECTURE

## 1. Objective

In Chapter 131, we designed the platform for **low traffic**.

Now traffic is increasing.

The application is no longer receiving only:

```text
10–30 RPS
```

Assume the platform has grown to approximately:

```text
Normal traffic:
100–300 RPS

Peak traffic:
300–500 RPS

Background/event traffic:
hundreds to thousands of messages/minute
```

These are **simulation values**. Real production limits must come from load testing and observed telemetry.

The architecture should now handle:

* more concurrent users
* more API requests
* more database queries
* more Redis traffic
* more asynchronous events
* more pods
* more Kubernetes nodes
* more observability data
* more deployments
* more failure scenarios

The important principle remains:

> **We scale the bottleneck, not everything.**

---

# 2. Low vs Medium Traffic

The architecture evolves from:

```text
LOW

10–30 RPS
    |
    v
small pod footprint
    |
    v
small node capacity
```

to:

```text
MEDIUM

100–500 RPS
    |
    v
more replicas
    |
    v
more nodes
    |
    v
greater database pressure
    |
    v
greater cache/messaging pressure
```

The architecture itself remains recognizable.

---

# 3. Medium-Traffic Architecture

```text
                         INTERNET
                            |
                            v
                       Route 53
                            |
                            v
                    CloudFront / WAF
                            |
                            v
                           ALB
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
           AZ-A           AZ-B           AZ-C
             |              |              |
        EKS Nodes       EKS Nodes       EKS Nodes
             |              |              |
             +--------------+--------------+
                            |
                    21 Microservices
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
       RDS                Redis              Messaging
   PostgreSQL          ElastiCache          SQS/Kafka
        |                   |                   |
        +-------------------+-------------------+
                            |
                    Observability
                            |
       +--------------------+--------------------+
       |                    |                    |
   Prometheus              ELK                Jaeger
       |                    |                    |
    Grafana              Kibana             Trace UI
       |
  Alertmanager
```

---

# 4. What Changes From Low Traffic?

The major changes are:

```text
LOW
 |
 +-- fewer replicas
 +-- fewer nodes
 +-- lower database load
 +-- smaller queues
 +-- lower observability volume

MEDIUM
 |
 +-- more replicas
 +-- more nodes
 +-- HPA becomes active more frequently
 +-- Karpenter becomes important
 +-- RDS connections increase
 +-- Redis throughput increases
 +-- messaging throughput increases
 +-- more telemetry
```

---

# 5. Don't Scale All 21 Services Equally

This is one of the most important lessons.

Suppose traffic increases from:

```text
30 RPS
```

to:

```text
300 RPS
```

It does **not** mean:

```text
every service
10 replicas
```

Instead, traffic distribution may look like:

```text
catalog-service       100 RPS
product-service        80 RPS
search-service         60 RPS
pricing-service        50 RPS
cart-service           40 RPS
order-service          20 RPS
payment-service        15 RPS
other services         lower
```

Therefore each service scales according to its own workload.

---

# 6. Traffic Profile

A realistic medium-traffic system may have:

```text
API traffic
       |
       +---- catalog
       +---- product
       +---- search
       +---- pricing
       +---- promotion
       +---- cart
       +---- order
```

while asynchronous workloads may look like:

```text
Events
 |
 +---- inventory
 +---- fraud
 +---- delivery
 +---- notification
 +---- invoice
 +---- reporting
```

This naturally creates two scaling models.

---

# 7. Synchronous Scaling

For API services:

```text
User
 |
 v
ALB
 |
 v
Service
 |
 v
HPA
 |
 v
Pods
```

Example:

```text
catalog-service

2 replicas
   |
   | traffic increases
   v
4 replicas
   |
   | traffic increases
   v
8 replicas
```

---

# 8. Asynchronous Scaling

For workers:

```text
Producer
    |
    v
SQS/Kafka
    |
    v
KEDA
    |
    v
Consumer Pods
```

Example:

```text
Queue depth = 100
       |
       v
2 workers

Queue depth = 1,000
       |
       v
5 workers

Queue depth = 10,000
       |
       v
10 workers
```

The maximum should be bounded by downstream capacity.

---

# 9. Medium-Traffic Replica Strategy

A possible starting point:

| Service                | Min | Max | Primary Scaling Signal |
| ---------------------- | --: | --: | ---------------------- |
| auth-service           |   2 |   6 | CPU/requests           |
| user-service           |   2 |   5 | CPU/requests           |
| customer-service       |   2 |   5 | CPU/requests           |
| catalog-service        |   2 |  10 | CPU/requests           |
| product-service        |   2 |  10 | CPU/requests           |
| inventory-service      |   2 |   8 | CPU/events             |
| pricing-service        |   2 |   8 | CPU/requests           |
| promotion-service      |   2 |   6 | CPU/requests           |
| cart-service           |   2 |   8 | CPU/requests           |
| order-service          |   2 |   8 | CPU/requests           |
| payment-service        |   2 |   6 | CPU/requests           |
| fraud-service          |   2 |   6 | CPU/events             |
| shipping-service       |   2 |   5 | CPU/requests           |
| delivery-service       |   2 |   6 | events                 |
| notification-service   |   1 |  10 | SQS                    |
| search-service         |   2 |  10 | CPU/requests           |
| recommendation-service |   2 |   6 | CPU/custom             |
| review-service         |   2 |   5 | CPU/requests           |
| tax-service            |   2 |   5 | CPU/requests           |
| invoice-service        |   1 |   8 | SQS                    |
| reporting-service      |   1 |   5 | queue/schedule         |

These values are **illustrative starting points only**.

The correct values must be established from:

```text
load testing
+
production telemetry
+
dependency limits
```

---

# 10. Why Minimum Replicas Increase

At low traffic we might have:

```text
catalog-service
2 replicas
```

At medium traffic, keeping two replicas provides basic HA while HPA can add more.

For example:

```text
Normal:

Pod A
Pod B


Traffic increases:

Pod A
Pod B
Pod C
Pod D
Pod E
```

The service doesn't need a manual deployment every time traffic changes.

---

# 11. HPA Becomes an Active Scaling Mechanism

Example:

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

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Traffic:

```text
100 RPS
```

might require:

```text
2 pods
```

while:

```text
300 RPS
```

may require:

```text
5 pods
```

and:

```text
500 RPS
```

may require:

```text
8 pods
```

But the actual relationship must be measured.

---

# 12. CPU Is Not Always Enough

Suppose:

```text
search-service CPU = 45%
```

but:

```text
P95 latency = 900ms
```

CPU does not indicate that the service is healthy.

Possible bottleneck:

```text
search backend
database
network
external API
thread pool
connection pool
```

Therefore medium-traffic scaling should increasingly use:

```text
CPU
+
memory
+
request rate
+
latency
+
custom application metrics
```

where appropriate.

---

# 13. Example — Custom Scaling

Suppose:

```text
order-service
```

has a target:

```text
requests per pod = 20 RPS
```

If traffic reaches:

```text
100 RPS
```

we need approximately:

```text
100 / 20
=
5 pods
```

This can be incorporated into an autoscaling strategy using an appropriate custom/external metric.

The important point is:

> Scale based on the workload characteristic that actually predicts saturation.

---

# 14. Kubernetes Nodes

At low traffic:

```text
Node A
Node B
```

might be sufficient.

At medium traffic:

```text
Node A
Node B
Node C
Node D
Node E
```

may become necessary.

But we don't manually keep adding nodes.

Instead:

```text
HPA
 |
 v
more Pods
 |
 v
Pending Pods
 |
 v
Karpenter
 |
 v
new Nodes
```

---

# 15. Karpenter Flow

Example:

```text
Traffic increases
        |
        v
HPA
        |
        v
catalog-service
2 -> 5 pods
        |
        v
Existing nodes lack capacity
        |
        v
Pods become Pending
        |
        v
Karpenter
        |
        v
new EC2 capacity
        |
        v
pods scheduled
```

This is the interaction between:

```text
HPA = pod scaling
Karpenter = node scaling
```

They solve different problems.

---

# 16. HPA Does Not Create Nodes

This is a common misunderstanding.

HPA:

```text
Deployment
2 replicas
      |
      v
5 replicas
```

But if the cluster has no room:

```text
Pod 1 -> Running
Pod 2 -> Running
Pod 3 -> Pending
Pod 4 -> Pending
Pod 5 -> Pending
```

Karpenter or another cluster autoscaler must provide capacity.

---

# 17. Node Packing

Medium traffic gives us an opportunity to improve node utilization.

Example:

```text
Node A
 |
 +-- catalog
 +-- product
 +-- pricing
 +-- cart

Node B
 |
 +-- order
 +-- payment
 +-- customer
```

Kubernetes uses resource requests when scheduling.

We should avoid:

```text
Node A
 |
 +-- one tiny pod
```

unless isolation is intentional.

---

# 18. But Don't Over-Pack Critical Services

Suppose:

```text
Node A
 |
 +-- order-service
 +-- payment-service
 +-- inventory-service
 +-- auth-service
```

Then:

```text
Node A failure
```

could affect several critical services simultaneously.

Therefore medium-traffic architecture needs:

```text
topology spread
+
anti-affinity where justified
+
multiple nodes
+
multiple AZs
```

---

# 19. Three-AZ Distribution

For production:

```text
                EKS
                 |
       +---------+---------+
       |         |         |
      AZ-A      AZ-B      AZ-C
       |         |         |
     Nodes     Nodes     Nodes
```

Critical service:

```text
order-service
```

can be distributed:

```text
AZ-A -> Pod 1
AZ-B -> Pod 2
AZ-C -> Pod 3
```

if capacity and replica count justify that layout.

---

# 20. AZ Failure Headroom

Suppose we need:

```text
6 application nodes
```

normal operation.

We shouldn't necessarily run exactly:

```text
2 nodes per AZ
```

and assume that is enough for an AZ failure.

If AZ-A disappears:

```text
AZ-A -> lost
```

the remaining AZs must have sufficient capacity to absorb critical workloads.

This is why capacity planning includes:

```text
normal capacity
+
failure headroom
```

---

# 21. RDS Becomes a Major Bottleneck

At medium traffic, the database often becomes more important than Kubernetes CPU.

Example:

```text
300 RPS
   |
   v
microservices
   |
   v
PostgreSQL
```

Potential bottlenecks:

```text
connections
CPU
IOPS
query latency
locks
buffer/cache behavior
slow queries
indexes
transactions
```

---

# 22. Connection Pool Calculation

Suppose:

```text
order-service
5 pods
```

and:

```text
Hikari maximumPoolSize = 10
```

Potential maximum:

```text
5 × 10
=
50 connections
```

Now:

```text
payment-service
4 pods × 10
=
40
```

and:

```text
inventory-service
5 pods × 10
=
50
```

Combined:

```text
50 + 40 + 50
=
140 connections
```

Then add:

```text
other services
administrative connections
migration jobs
monitoring
failover/surge capacity
```

This is why connection budgets must be calculated across the entire platform.

---

# 23. HPA + RDS Danger

Consider:

```text
Traffic spike
     |
     v
HPA
     |
     v
pods increase
     |
     v
DB connections increase
     |
     v
RDS reaches connection limit
     |
     v
requests fail
     |
     v
retries
     |
     v
more DB connections
```

This can turn a traffic increase into a complete outage.

Therefore:

> HPA max replicas must be compatible with database capacity.

---

# 24. Connection Pool Strategy

Example production configuration:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000
```

These values are examples, not universal production defaults.

Measure:

```text
DB latency
connections
query concurrency
CPU
transaction time
```

before tuning.

---

# 25. RDS Read Replicas

At medium traffic, read-heavy services may begin generating substantial database reads.

For example:

```text
catalog-service
product-service
search-related workloads
reporting
```

could create high read volume.

Possible architecture:

```text
Write traffic
    |
    v
RDS Primary


Read traffic
    |
    v
Read Replica
```

But don't automatically add read replicas.

First determine whether the bottleneck is actually:

```text
read workload
```

and whether the application can safely route reads.

---

# 26. Database Caching

Before scaling the database aggressively, examine:

```text
Redis
```

For frequently accessed data:

```text
product
catalog
pricing
promotion
```

flow:

```text
Request
   |
   v
Redis
   |
   +---- HIT -> return
   |
   +---- MISS
          |
          v
         RDS
          |
          v
       Redis SET
```

This reduces repetitive database reads.

---

# 27. Cache Hit Ratio

Monitor:

```text
cache hits
cache misses
```

Example:

```text
1,000 requests

900 hits
100 misses
```

Hit ratio:

```text
900 / 1000
=
90%
```

A falling hit ratio can increase RDS pressure even if application traffic remains constant.

---

# 28. Redis Medium-Traffic Architecture

```text
                 Microservices
                      |
                      v
               ElastiCache
                /        \
          Primary       Replica
                \        /
                 Multi-AZ
```

Monitor:

```text
memory
CPU
connections
evictions
cache hit rate
latency
hot keys
network throughput
```

---

# 29. Messaging Growth

At low traffic:

```text
SQS
 |
 +-- small backlog
```

At medium traffic:

```text
SQS
 |
 +-- continuously changing backlog
```

We now need stronger controls around:

```text
consumer concurrency
visibility timeout
retry
DLQ
message age
KEDA
downstream capacity
```

---

# 30. SQS + KEDA

Example:

```text
notification queue
```

Normal:

```text
queue depth = 100
consumer replicas = 2
```

Traffic increases:

```text
queue depth = 2,000
```

KEDA:

```text
2 -> 5 -> 8 replicas
```

If the queue keeps increasing:

```text
8 -> maxReplicas
```

At that point, we should investigate:

```text
consumer processing speed
external provider
database
network
visibility timeout
message failures
```

instead of simply increasing replicas forever.

---

# 31. Kafka Consumer Scaling

For Kafka:

```text
topic
 |
 +-- partition 0
 +-- partition 1
 +-- partition 2
 +-- partition 3
```

Consumer group:

```text
consumer-1
consumer-2
consumer-3
consumer-4
```

A consumer group cannot gain unlimited parallelism from a topic with only a small number of partitions.

Therefore:

```text
Kafka partitions
        |
        v
maximum useful consumer parallelism
```

is an important capacity constraint.

---

# 32. Medium-Traffic Kafka Architecture

```text
                    Kafka
                      |
        +-------------+-------------+
        |             |             |
   partition-0   partition-1   partition-2
        |             |             |
        v             v             v
    consumer-1    consumer-2    consumer-3
```

Monitor:

```text
consumer lag
partition skew
broker CPU
broker storage
network
ISR
under-replicated partitions
```

---

# 33. ALB Capacity

At medium traffic, ALB remains the public entry point:

```text
Internet
   |
   v
ALB
   |
   v
Kubernetes Services
```

Monitor:

```text
request count
target response time
HTTP 4xx
HTTP 5xx
target connection errors
healthy targets
```

If ALB metrics are healthy but application latency is high:

```text
ALB
  |
  v
healthy
  |
  v
application bottleneck
```

Don't immediately blame the load balancer.

---

# 34. Application Bottleneck Example

Suppose:

```text
ALB target response time
=
900ms
```

and:

```text
order-service CPU
=
30%
```

Possible cause:

```text
RDS query
```

Tracing:

```text
order-service
   |
   +-- customer-service 20ms
   |
   +-- pricing-service 30ms
   |
   +-- RDS query 700ms
```

Now we know:

```text
database query
```

is the bottleneck.

This is why:

```text
metrics
+
logs
+
traces
```

must be used together.

---

# 35. Medium-Traffic Observability

Telemetry volume increases.

Therefore:

```text
applications
     |
     +---- metrics
     +---- logs
     +---- traces
```

produces significantly more data.

We need to manage:

```text
Prometheus retention
Elasticsearch retention
trace sampling
log volume
metric cardinality
storage
```

---

# 36. Prometheus Scaling

As services and traffic increase:

```text
metrics
   |
   v
Prometheus
```

may collect more samples.

Monitor:

```text
Prometheus CPU
Prometheus memory
TSDB storage
ingestion rate
query latency
cardinality
```

Do not blindly increase retention.

For larger environments consider:

```text
Amazon Managed Service for Prometheus
Thanos
Mimir
Cortex
```

depending on architecture and operational requirements.

---

# 37. Log Volume

At medium traffic:

```text
100 RPS
```

could generate substantial logs if every request produces multiple log lines.

Bad:

```text
INFO request received
INFO validating
INFO querying database
INFO query completed
INFO response generated
```

for every request.

This can create huge log volume.

Prefer structured logs with appropriate levels.

Use:

```text
INFO
WARN
ERROR
DEBUG
```

intentionally.

---

# 38. Trace Sampling

Tracing every request can become expensive at scale.

At medium traffic, sampling becomes more important.

For example:

```text
normal successful requests
        |
        v
sample percentage

errors
        |
        v
retain high priority

slow requests
        |
        v
retain high priority
```

Tail sampling can be used where the architecture supports it.

---

# 39. Medium-Traffic SLO

Suppose:

```text
Order API
SLO = 99.9% availability
```

and:

```text
P95 latency target = 300ms
```

We monitor:

```text
availability
latency
error budget
burn rate
```

If latency increases:

```text
P95
300ms
  |
  v
450ms
  |
  v
SLO violation risk
```

This should trigger investigation before customers experience a major outage.

---

# 40. Medium-Traffic HPA Example

A service might start:

```text
replicas = 2
```

Traffic increases:

```text
CPU = 70%
```

HPA:

```text
2 -> 3
```

After scaling:

```text
CPU = 55%
```

Traffic increases further:

```text
3 -> 5
```

Eventually:

```text
5 pods
```

might hit the service's tested capacity.

Then:

```text
maxReplicas
```

becomes an important safety boundary.

---

# 41. Why MaxReplicas Is a Safety Control

Suppose:

```text
maxReplicas = 10
```

and:

```text
RDS capacity
```

supports the application safely up to approximately:

```text
10 pods
```

Then HPA cannot blindly create:

```text
50
100
200
```

pods and overwhelm dependencies.

Autoscaling should have **guardrails**.

---

# 42. Pod Requests at Medium Traffic

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

Again:

> These values must come from performance testing.

A common mistake is copying:

```text
500m CPU
1Gi memory
```

to every microservice without measuring.

---

# 43. JVM Considerations

All 21 services use:

```text
Java + Spring Boot
```

Therefore medium traffic requires JVM monitoring.

Important metrics:

```text
heap usage
GC pause
GC frequency
thread count
CPU
allocation rate
container memory
```

Potential problem:

```text
Kubernetes memory limit
       |
       v
JVM heap
       |
       v
native memory
       |
       v
metaspace
       |
       v
OOMKilled
```

Container memory must be sized with the complete JVM memory footprint in mind.

---

# 44. Spring Boot Actuator

Expose internal management endpoints:

```text
/actuator/health
/actuator/prometheus
```

Readiness:

```text
GET /actuator/health/readiness
```

Liveness:

```text
GET /actuator/health/liveness
```

Prometheus:

```text
GET /actuator/prometheus
```

These endpoints should not unnecessarily be exposed to the public Internet.

---

# 45. Deployment Surge

Suppose:

```text
order-service
5 replicas
```

and we deploy a new version.

Rolling update may temporarily require extra pods.

Example:

```text
old pods = 5
new pods = 2
```

During rollout:

```text
total pods = 7
```

This creates additional:

```text
CPU
memory
database connections
```

capacity requirements.

Therefore deployment capacity must be included in medium-traffic planning.

---

# 46. Database Connection Surge During Deployment

Suppose:

```text
5 old pods
+
5 new pods
```

and each pod can open:

```text
10 DB connections
```

Potential:

```text
10 × 10
=
100 connections
```

instead of:

```text
5 × 10
=
50
```

This is why:

```text
rolling update
+
HPA
+
connection pool
```

must be considered together.

---

# 47. Graceful Shutdown

When Kubernetes terminates a pod:

```text
SIGTERM
```

should allow the application to:

```text
stop accepting new work
finish active requests where possible
close connections
flush telemetry
exit
```

Example:

```yaml
terminationGracePeriodSeconds: 30
```

The exact value depends on request duration and application behavior.

---

# 48. Readiness During Shutdown

The pod should stop receiving traffic before it terminates.

Conceptually:

```text
Pod
 |
 | SIGTERM
 v
Not Ready
 |
 v
Traffic stops
 |
 v
Existing requests finish
 |
 v
Application exits
```

This prevents avoidable connection resets during deployments.

---

# 49. Medium-Traffic Security

As the system grows, security boundaries become more important.

Architecture:

```text
Internet
   |
   v
WAF
   |
   v
ALB
   |
   v
EKS
   |
   +---- NetworkPolicy
   |
   +---- IAM Pod Identity
   |
   +---- Secrets Manager
   |
   +---- Security Groups
```

CI:

```text
GitHub
   |
   v
OIDC
   |
   v
AWS IAM Role
   |
   v
ECR
```

No long-lived AWS credentials.

---

# 50. Medium-Traffic GitOps

The deployment flow remains:

```text
Developer
    |
    v
GitHub PR
    |
    v
CI
    |
    +-- Maven
    +-- Unit Tests
    +-- SonarQube
    +-- Dependency Scan
    +-- Trivy
    +-- Docker
    |
    v
ECR
    |
    v
GitOps Repository
    |
    v
ArgoCD
    |
    v
EKS
```

The difference is that deployments now need stronger:

```text
progressive rollout
health checks
rollback
observability
capacity validation
```

---

# 51. Canary Deployment

Medium traffic is a good stage to introduce controlled canary deployments.

Example:

```text
Existing version:
v1
```

Deploy:

```text
v2
```

with a small percentage of traffic.

Conceptually:

```text
                 ALB
                  |
          +-------+-------+
          |               |
          v               v
        v1 90%          v2 10%
```

Monitor:

```text
5xx
latency
CPU
business errors
RDS
logs
traces
```

Then increase:

```text
10%
25%
50%
100%
```

if the rollout is healthy.

The exact mechanism can use your chosen progressive delivery tooling and should remain GitOps-controlled.

---

# 52. Rollback

Suppose v2 causes:

```text
5xx = 8%
```

instead of:

```text
0.5%
```

Rollback:

```text
v2
 |
 X
 |
 v
v1
```

With GitOps:

```text
Git
 |
 v
revert deployment change
 |
 v
ArgoCD
 |
 v
EKS
```

Rollback should be a controlled, rehearsed operation.

---

# 53. Medium-Traffic Incident Example

Scenario:

```text
Traffic = 350 RPS
```

Users report:

```text
slow checkout
```

Initial metrics:

```text
ALB:
healthy

order-service CPU:
45%

order-service P95:
900ms

RDS CPU:
85%

RDS query latency:
high
```

Tracing:

```text
order-service
      |
      +-- customer-service 30ms
      |
      +-- pricing-service 40ms
      |
      +-- PostgreSQL 750ms
```

Root investigation moves toward:

```text
RDS query/index/lock/connection issue
```

rather than blindly scaling order-service.

---

# 54. Another Incident — HPA Makes Things Worse

Scenario:

```text
Traffic spike
```

HPA:

```text
order-service
2 -> 8 pods
```

Database connections:

```text
40 -> 160
```

RDS reaches connection pressure.

Requests fail.

Retries increase.

HPA sees high CPU and creates more pods.

This creates:

```text
positive feedback loop
```

The solution may involve:

```text
reduce HPA maximum
reduce retry amplification
optimize DB queries
tune connection pools
protect RDS
```

not simply:

```text
increase maxReplicas
```

---

# 55. Medium-Traffic Capacity Model

We now calculate capacity across multiple layers.

```text
             Traffic
                |
                v
              ALB
                |
                v
             Pods
                |
       +--------+--------+
       |        |        |
      RDS     Redis   Messaging
       |        |        |
       +--------+--------+
                |
         External APIs
```

For each layer:

```text
required capacity
+
normal utilization
+
peak utilization
+
failure headroom
+
deployment surge
```

---

# 56. Application Capacity Formula

For a service:

```text
required replicas =
ceil(
    peak RPS /
    tested RPS per pod
)
```

Example:

```text
peak = 240 RPS

tested pod capacity = 40 RPS
```

Therefore:

```text
240 / 40
=
6
```

Add engineering headroom according to your policy.

Do not simply assume:

```text
1 pod = 100 RPS
```

without testing.

---

# 57. Database Capacity Model

For PostgreSQL consider:

```text
connections
CPU
memory
IOPS
storage throughput
query latency
locks
transactions/sec
replication lag
```

A database can become saturated even when:

```text
CPU = 50%
```

because the bottleneck may be:

```text
locks
IO
connections
slow queries
```

---

# 58. Redis Capacity Model

Consider:

```text
requests/sec
memory
connections
network
latency
evictions
hot keys
```

Don't measure Redis only by CPU.

---

# 59. Kafka Capacity Model

Consider:

```text
partitions
broker CPU
network
disk
replication
producer rate
consumer rate
consumer lag
```

Example:

```text
producer rate
=
10,000 events/sec

consumer rate
=
8,000 events/sec
```

Then:

```text
lag increases
```

even if Kubernetes CPU is low.

---

# 60. SQS Capacity Model

Consider:

```text
queue depth
message age
producer rate
consumer rate
visibility timeout
DLQ count
consumer concurrency
```

If:

```text
producer = 1,000 msg/s
consumer = 700 msg/s
```

then backlog grows at roughly:

```text
300 msg/s
```

until capacity changes.

---

# 61. Medium-Traffic Node Pool Strategy

A practical model:

```text
                 EKS
                  |
        +---------+---------+
        |                   |
   System/Platform      Application
     Node Pool           Node Pool
        |                   |
  monitoring/etc.       21 services
```

As workloads grow, additional specialized pools can be introduced if there is a real reason:

```text
general workloads
memory optimized
compute optimized
spot/batch workloads
```

Don't create many node pools without a workload/isolation requirement.

---

# 62. Spot Instances

Spot capacity can be useful for:

```text
non-critical
interruptible
stateless
batch
async
```

workloads.

Examples:

```text
reporting-service
background processing
some CI workloads
```

But critical synchronous services should have sufficient reliable/on-demand capacity.

A mixed strategy can be:

```text
On-Demand
   +
Spot
   +
Karpenter
```

with appropriate workload constraints.

---

# 63. Medium-Traffic Cost Controls

Useful controls:

```text
right-sized requests
HPA
KEDA
Karpenter
Spot for suitable workloads
log retention
metric retention
trace sampling
RDS right-sizing
Redis right-sizing
S3 lifecycle policies
ECR lifecycle policies
```

Avoid cost optimization that removes required reliability.

---

# 64. Medium-Traffic Monitoring Dashboard

Create a dashboard containing:

### Traffic

```text
RPS
requests
4xx
5xx
```

### Application

```text
P50
P95
P99
CPU
memory
restarts
replicas
```

### Kubernetes

```text
nodes
pending pods
CPU utilization
memory utilization
HPA status
KEDA status
```

### Database

```text
connections
CPU
IO
latency
locks
```

### Redis

```text
memory
hits
misses
evictions
latency
```

### Messaging

```text
queue depth
message age
Kafka lag
```

---

# 65. Alerting

Examples:

```yaml
groups:
  - name: medium-traffic-alerts

    rules:

      - alert: HighApplicationErrorRate
        expr: ...
        for: 5m

      - alert: HighApplicationLatency
        expr: ...
        for: 10m

      - alert: RDSConnectionPressure
        expr: ...
        for: 5m

      - alert: SQSBacklogGrowing
        expr: ...
        for: 10m

      - alert: KafkaConsumerLag
        expr: ...
        for: 10m
```

The exact PromQL must be based on the metrics exposed by your actual environment.

---

# 66. Medium-Traffic Testing

Use k6 from Chapter 127.

Example progression:

```text
10 RPS
   |
   v
50 RPS
   |
   v
100 RPS
   |
   v
200 RPS
   |
   v
300 RPS
   |
   v
500 RPS
```

At each stage measure:

```text
P50
P95
P99
5xx
CPU
memory
HPA
nodes
RDS
Redis
SQS
Kafka
```

---

# 67. Test HPA

Example:

```text
Initial:
2 pods
```

Load:

```text
50 RPS
```

Then:

```text
100 RPS
```

Observe:

```text
2 -> 3
```

Then:

```text
200 RPS
```

Observe:

```text
3 -> 5
```

Measure:

```text
scale-up time
```

Then reduce traffic.

Measure:

```text
scale-down time
```

This is much more useful than simply confirming:

> HPA exists.

---

# 68. Test Karpenter

Start:

```text
3 nodes
```

Generate workload.

Observe:

```text
Pending pods
```

Then:

```text
Karpenter
   |
   v
new node
```

Measure:

```text
time from Pending -> Scheduled
```

Also test:

```text
node consolidation
```

where appropriate.

---

# 69. Test RDS Connection Pressure

Controlled test:

```text
increase application replicas
```

Monitor:

```text
DatabaseConnections
```

Compare:

```text
2 pods
4 pods
6 pods
8 pods
```

and calculate:

```text
connections per pod
```

This lets you establish a real connection budget.

---

# 70. Test Redis

Measure:

```text
cache hit ratio
```

before and during load.

Then simulate:

```text
cache misses
```

and observe:

```text
RDS load
```

This demonstrates the relationship:

```text
Redis
 |
 v
RDS
```

---

# 71. Test SQS

Generate messages.

Example:

```text
1,000 messages
```

Observe:

```text
queue depth
consumer replicas
processing rate
oldest message age
```

Then:

```text
2,000
5,000
10,000
```

and verify KEDA behavior.

---

# 72. Test Kafka

Generate events.

Observe:

```text
producer rate
consumer rate
lag
partition distribution
```

Then increase consumers.

Determine whether:

```text
lag decreases
```

If not, identify the actual bottleneck.

---

# 73. Medium-Traffic Failure Testing

Test:

```text
pod failure
node failure
application crash
RDS failover
Redis failure
SQS backlog
Kafka lag
bad deployment
network policy issue
DNS issue
```

For each test record:

```text
start time
detection time
mitigation time
recovery time
customer impact
root cause
```

This creates measurable operational maturity.

---

# 74. Medium-Traffic Deployment Strategy

Recommended flow:

```text
Developer
   |
   v
PR
   |
   v
CI
   |
   +-- Build
   +-- Test
   +-- SonarQube
   +-- Trivy
   +-- Docker
   |
   v
ECR
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
Canary/Rolling
   |
   v
Metrics
Logs
Traces
   |
   v
Promotion
```

---

# 75. Low vs Medium Comparison

| Area              | Low Traffic               | Medium Traffic             |
| ----------------- | ------------------------- | -------------------------- |
| API load          | 10–30 RPS                 | 100–500 RPS                |
| replicas          | low                       | moderate                   |
| HPA               | mostly standby            | frequently active          |
| nodes             | few                       | more                       |
| Karpenter         | occasional                | important                  |
| RDS pressure      | low                       | moderate/high              |
| Redis             | low                       | higher throughput          |
| SQS               | small backlog             | continuous workload        |
| Kafka             | lower throughput          | higher throughput          |
| telemetry         | lower                     | significantly higher       |
| tracing           | broader sampling possible | sampling becomes important |
| deployment surge  | manageable                | capacity planning required |
| DB connections    | low                       | important constraint       |
| failure headroom  | important                 | more critical              |
| capacity planning | basic                     | detailed                   |

---

# 76. The Most Important Difference

Low traffic asks:

> **Can the platform run safely with small capacity?**

Medium traffic asks:

> **Can the platform scale without one dependency becoming the bottleneck?**

That is a major shift.

At medium traffic:

```text
Kubernetes scaling
```

alone is not enough.

We must understand:

```text
Application
      |
      +---- Kubernetes
      |
      +---- Database
      |
      +---- Cache
      |
      +---- Messaging
      |
      +---- External APIs
```

as one system.

---

# 77. Final Medium-Traffic Architecture

```text
                              USERS
                                |
                                v
                         Route 53 / DNS
                                |
                                v
                        CloudFront / WAF
                                |
                                v
                               ALB
                                |
              +-----------------+-----------------+
              |                 |                 |
             AZ-A              AZ-B              AZ-C
              |                 |                 |
          EKS Nodes         EKS Nodes         EKS Nodes
              |                 |                 |
              +-----------------+-----------------+
                                |
                         21 Microservices
                                |
             +------------------+------------------+
             |                  |                  |
             v                  v                  v
           RDS                Redis             Messaging
       PostgreSQL          ElastiCache        SQS / Kafka
             |                  |                  |
             +------------------+------------------+
                                |
                       External Dependencies
                                |
                                v
                         Observability
                                |
          +---------------------+---------------------+
          |                     |                     |
      Prometheus               ELK                  OTel
          |                     |                     |
       Grafana               Kibana                Jaeger
          |
     Alertmanager
```

Scaling:

```text
                     TRAFFIC
                        |
                        v
                      ALB
                        |
                        v
                 Application Pods
                        |
                       HPA
                        |
                  Pending Pods
                        |
                    Karpenter
                        |
                    EKS Nodes
```

Async:

```text
                SQS / Kafka
                     |
                     v
                    KEDA
                     |
                     v
                Worker Pods
                     |
                     v
                 Karpenter
```

---

# 78. Final Production Checklist

### Kubernetes

```text
[ ] Multi-AZ
[ ] HPA
[ ] KEDA
[ ] Karpenter
[ ] Resource requests
[ ] Resource limits
[ ] PDB
[ ] Topology spread
[ ] NetworkPolicy
[ ] Readiness probes
[ ] Liveness probes
[ ] Startup probes
[ ] Graceful shutdown
```

### Database

```text
[ ] RDS capacity tested
[ ] Connection budget calculated
[ ] Hikari pool configured
[ ] Slow queries monitored
[ ] Indexes reviewed
[ ] Backup/PITR tested
[ ] Failover tested
```

### Cache

```text
[ ] Redis capacity tested
[ ] Hit ratio monitored
[ ] Evictions monitored
[ ] Failure fallback tested
```

### Messaging

```text
[ ] SQS backlog monitored
[ ] KEDA tested
[ ] DLQ configured
[ ] Kafka lag monitored
[ ] Partition capacity reviewed
[ ] Retry behavior tested
```

### Observability

```text
[ ] Prometheus
[ ] Grafana
[ ] Alertmanager
[ ] ELK
[ ] OpenTelemetry
[ ] Jaeger
[ ] Correlation IDs
[ ] SLOs
[ ] Runbooks
```

### Deployment

```text
[ ] GitOps
[ ] ArgoCD
[ ] Immutable images
[ ] Rolling/canary strategy
[ ] Rollback tested
[ ] Deployment surge capacity
```

---

# Chapter 132 Complete

The important architecture progression is now:

```text
CHAPTER 131
LOW TRAFFIC
10–30 RPS
      |
      v
small but production-safe platform
      |
      v
CHAPTER 132
MEDIUM TRAFFIC
100–500 RPS
      |
      v
HPA + KEDA + Karpenter
      |
      v
database/cache/messaging capacity becomes critical
      |
      v
CHAPTER 133
HIGH TRAFFIC
```

**Next: `133 — HIGH TRAFFIC ARCHITECTURE`**

That chapter will take the same 21 services into a substantially heavier workload and cover **large-scale EKS capacity, aggressive HPA/KEDA behavior, Karpenter provisioning, RDS scaling/read replicas, Redis scaling, Kafka partition/broker capacity, SQS consumer fleets, ALB/WAF, observability scaling, failure headroom, and the bottlenecks that appear when Kubernetes itself is no longer the only scaling problem.**
