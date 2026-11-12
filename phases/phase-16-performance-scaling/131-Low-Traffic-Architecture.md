# 131 — LOW TRAFFIC ARCHITECTURE

## 1. Objective

In this chapter we will design and implement the **low-traffic production architecture** for the complete 21-microservice e-commerce platform.

The goal is **not** to build an oversized infrastructure just because the application has 21 services.

Instead, we want to answer:

> If the platform initially receives relatively low traffic, how do we run all 21 microservices safely, maintain production-grade reliability, and avoid unnecessary infrastructure consumption?

The architecture must still support:

* Kubernetes/EKS
* 21 microservices
* HPA
* KEDA
* Karpenter
* PostgreSQL/RDS
* Redis
* SQS/SNS
* Kafka where required
* RabbitMQ where required
* Prometheus
* Grafana
* Alertmanager
* ELK
* OpenTelemetry
* Jaeger
* ArgoCD
* GitOps
* IAM/Pod Identity
* Secrets Manager
* Multi-AZ workload placement
* Production monitoring
* Disaster recovery

The important principle is:

> **Low traffic does not mean low reliability.**

We reduce capacity where possible, not the architectural safety controls.

---

# 2. Low-Traffic Scenario

For this simulation, assume the platform has recently launched.

Example workload:

```text
Normal traffic:
        10–30 requests/second

Peak traffic:
        50–100 requests/second

Occasional background jobs:
        10–100 messages/second

Users:
        10,000–50,000 registered users

Concurrent users:
        relatively low

Traffic pattern:
        mostly steady
        occasional short spikes
```

These numbers are **simulation assumptions**, not production sizing recommendations.

Actual capacity must come from load testing.

---

# 3. The 21 Microservices

Our application contains:

```text
01 auth-service
02 user-service
03 customer-service
04 catalog-service
05 product-service
06 inventory-service
07 pricing-service
08 promotion-service
09 cart-service
10 order-service
11 payment-service
12 fraud-service
13 shipping-service
14 delivery-service
15 notification-service
16 search-service
17 recommendation-service
18 review-service
19 tax-service
20 invoice-service
21 reporting-service
```

All services use:

```text
Java
Spring Boot
Maven
Docker
Kubernetes
AWS EKS
```

---

# 4. High-Level Low-Traffic Architecture

```text
                         Internet
                            |
                            v
                       Route 53
                            |
                            v
                      CloudFront/WAF
                            |
                            v
                          ALB
                            |
                +-----------+-----------+
                |                       |
                v                       v
          EKS Private Nodes       EKS Private Nodes
                |                       |
                +-----------+-----------+
                            |
                     21 Microservices
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
        RDS              Redis             Messaging
      PostgreSQL       ElastiCache       SQS/Kafka/etc.
          |                 |                  |
          +-----------------+------------------+
                            |
                    Observability
                            |
       +--------------------+---------------------+
       |                    |                     |
   Prometheus             ELK                  Jaeger
       |                    |                     |
    Grafana             Kibana              Trace UI
       |
  Alertmanager
```

---

# 5. Low-Traffic Design Philosophy

There are four important principles.

## Principle 1 — Don't run unnecessary replicas

We don't need:

```text
10 replicas
20 replicas
50 replicas
```

for every service when traffic is low.

Instead:

```text
minimum replicas = 1 or 2
maximum replicas = based on tested capacity
```

---

# 6. Principle 2 — Keep Critical Services Highly Available

Not every service has the same importance.

For example:

```text
order-service
payment-service
inventory-service
auth-service
```

are critical to the core business flow.

A reporting service doesn't have the same immediate customer impact.

Therefore:

```text
Critical API
    -> minimum 2 replicas

Non-critical API
    -> minimum 1 replica

Async worker
    -> KEDA-controlled
```

---

# 7. Principle 3 — Scale Based on the Correct Signal

Don't scale everything based only on CPU.

Example:

```text
notification-service
```

CPU may be only:

```text
30%
```

while SQS contains:

```text
100,000 messages
```

CPU-based HPA might not scale quickly enough.

Therefore:

```text
SQS backlog
      |
      v
     KEDA
      |
      v
notification-service replicas
```

---

# 8. Principle 4 — Database Capacity Is Part of Application Capacity

Suppose:

```text
order-service
10 pods
```

and each pod has:

```text
maximumPoolSize = 20
```

Potential connections:

```text
10 × 20 = 200
```

Now imagine 5 database-heavy services doing the same thing.

You can exhaust PostgreSQL long before Kubernetes reaches its CPU limit.

Therefore:

```text
Application scaling
        +
Database connection capacity
        +
Query capacity
        +
Redis capacity
        +
Messaging capacity
```

must be planned together.

---

# 9. EKS Low-Traffic Cluster

For the learning/production simulation, use:

```text
AWS Region:
ap-south-1

Availability Zones:
3

EKS:
1 cluster

Worker nodes:
small initial capacity

Node placement:
private subnets
```

Architecture:

```text
                EKS Cluster
                     |
       +-------------+-------------+
       |             |             |
      AZ-A          AZ-B          AZ-C
       |             |             |
     Node A        Node B        Node C
```

The exact EC2 instance family and size should be selected using:

* workload CPU
* workload memory
* daemonset overhead
* IP availability
* pod density
* failure headroom
* tested performance

rather than simply choosing an instance because it is inexpensive.

---

# 10. Node Strategy

A reasonable starting model is:

```text
System node capacity
        +
Application node capacity
```

For example:

```text
system/platform workloads
        |
        v
system node pool

application workloads
        |
        v
application node pool
```

Possible workloads:

### System

```text
CoreDNS
kube-proxy
VPC CNI
EBS CSI
AWS Load Balancer Controller
```

### Platform

```text
Prometheus
Grafana
Alertmanager
Fluent Bit
OpenTelemetry Collector
ArgoCD
```

### Application

```text
21 microservices
```

---

# 11. Should Every Service Have Its Own Node?

No.

That would be wasteful.

Incorrect low-traffic architecture:

```text
node-1 -> auth
node-2 -> user
node-3 -> customer
node-4 -> catalog
...
```

This creates unnecessary infrastructure fragmentation.

Instead:

```text
Application nodes
        |
        +-- auth
        +-- user
        +-- customer
        +-- catalog
        +-- product
        +-- inventory
        +-- ...
```

Kubernetes schedules workloads according to:

* CPU requests
* memory requests
* affinity
* anti-affinity
* topology spread
* taints/tolerations
* available capacity

---

# 12. Resource Requests

Every service must define resource requests.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"

  limits:
    cpu: "500m"
    memory: "512Mi"
```

This does **not** mean:

```text
service always consumes 500m CPU
```

It means:

```text
request = scheduling reservation

limit = maximum allowed resource consumption
```

Actual values must come from testing.

---

# 13. Low-Traffic Replica Strategy

A possible starting classification:

| Service                | Initial Replicas | Scaling   |
| ---------------------- | ---------------: | --------- |
| auth-service           |                2 | HPA       |
| user-service           |                1 | HPA       |
| customer-service       |                1 | HPA       |
| catalog-service        |                2 | HPA       |
| product-service        |                2 | HPA       |
| inventory-service      |                2 | HPA/KEDA  |
| pricing-service        |                1 | HPA       |
| promotion-service      |                1 | HPA       |
| cart-service           |                2 | HPA       |
| order-service          |                2 | HPA       |
| payment-service        |                2 | HPA       |
| fraud-service          |                1 | HPA/KEDA  |
| shipping-service       |                1 | HPA       |
| delivery-service       |                1 | HPA/KEDA  |
| notification-service   |                1 | KEDA      |
| search-service         |                1 | HPA       |
| recommendation-service |                1 | HPA       |
| review-service         |                1 | HPA       |
| tax-service            |                1 | HPA       |
| invoice-service        |                1 | KEDA      |
| reporting-service      |                1 | KEDA/Cron |

Again, these are **initial simulation values**.

They are not universal production values.

---

# 14. Why Some Services Have Two Replicas

Suppose:

```text
order-service
```

has:

```text
replicas: 1
```

and the only pod crashes.

Then:

```text
Order API
   |
   X
```

There is a service interruption until Kubernetes recreates it.

With:

```text
replicas: 2
```

we have:

```text
order-service
   |
   +---- Pod A
   |
   +---- Pod B
```

If Pod A fails:

```text
Pod A -> FAILED

Pod B -> still serving traffic
```

This is particularly important for critical customer-facing services.

---

# 15. Topology Spread

Two replicas should ideally not land on the same failure domain.

Example:

```text
AZ-A
  |
  +-- order-service-1

AZ-B
  |
  +-- order-service-2
```

instead of:

```text
AZ-A
  |
  +-- order-service-1
  +-- order-service-2
```

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: order-service
```

For truly critical workloads, the policy can be made stricter after verifying that sufficient cluster capacity exists.

---

# 16. Pod Disruption Budget

For a two-replica critical service:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: order-service

spec:
  minAvailable: 1

  selector:
    matchLabels:
      app: order-service
```

This helps protect availability during voluntary disruptions such as:

* node maintenance
* controlled node replacement
* cluster operations

A PDB does **not** protect against every failure.

It does not magically prevent:

```text
node crash
AZ outage
application crash
```

---

# 17. HPA Strategy

Example:

```text
order-service

minReplicas: 2
maxReplicas: 5
```

CPU-based example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: order-service

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  minReplicas: 2
  maxReplicas: 5

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

The threshold should be established from load testing.

---

# 18. Why We Don't Immediately Set maxReplicas to 100

Suppose:

```text
order-service
maxReplicas = 100
```

Traffic spike occurs.

HPA launches:

```text
100 pods
```

But PostgreSQL supports only:

```text
300 connections
```

and every pod opens:

```text
20 connections
```

Potential:

```text
100 × 20 = 2,000 connections
```

The result could be:

```text
HPA scaling
      |
      v
DB connection exhaustion
      |
      v
requests fail
      |
      v
retries increase
      |
      v
more load
      |
      v
incident
```

This is called a **scaling cascade**.

---

# 19. KEDA for Async Services

For:

```text
notification-service
invoice-service
reporting-service
```

CPU is not necessarily the correct scaling signal.

Example:

```text
SQS
 |
 | 10 messages
 v
notification-service
 |
 +-- Pod
```

Low traffic:

```text
Queue depth = 10
Replicas = 1
```

Spike:

```text
Queue depth = 10,000
```

KEDA can increase consumers.

```text
10,000 messages
       |
       v
     KEDA
       |
       v
  1 -> 5 -> 10 pods
```

The maximum must still respect downstream capacity.

---

# 20. Low-Traffic Messaging Architecture

Use messaging only where it provides a real architectural benefit.

Example:

```text
order-service
      |
      v
   Kafka/SQS
      |
      +---- inventory-service
      |
      +---- payment-service
      |
      +---- notification-service
      |
      +---- reporting-service
```

Don't introduce Kafka merely because:

> "Production systems use Kafka."

The technology must have a business or technical reason.

---

# 21. SQS for Background Work

Good low-traffic candidates:

```text
notification-service
invoice-service
reporting-service
```

Architecture:

```text
Producer
   |
   v
SQS
   |
   v
KEDA
   |
   v
Consumer Pods
```

When the queue is empty:

```text
consumer replicas
=
minimum
```

When the queue grows:

```text
queue depth
     |
     v
   KEDA
     |
     v
more consumers
```

---

# 22. Redis

Low traffic does not eliminate the need for caching.

Possible Redis use cases:

```text
session/cache
product cache
catalog cache
pricing cache
cart data
rate limiting
short-lived data
```

Architecture:

```text
Microservice
     |
     v
 Redis
     |
     v
RDS
```

Redis should generally be treated as:

```text
performance layer
```

rather than the authoritative source for durable business data unless a specific design requires otherwise.

---

# 23. RDS Low-Traffic Strategy

PostgreSQL remains the authoritative transactional database.

Architecture:

```text
EKS
 |
 | private network
 v
RDS PostgreSQL
```

RDS should be:

```text
private
encrypted
backed up
monitored
Multi-AZ where required
```

Don't expose PostgreSQL publicly just because traffic is low.

---

# 24. Database Ownership

Each microservice should logically own its data.

Example:

```text
order-service
    |
    v
orders database/schema

payment-service
    |
    v
payment database/schema

inventory-service
    |
    v
inventory database/schema
```

Avoid:

```text
21 services
      |
      v
one giant shared schema
```

because it creates tight coupling.

The exact physical RDS layout can be:

```text
one RDS cluster
+
multiple databases/schemas
```

or:

```text
multiple RDS instances
```

depending on isolation, scale, security, and operational requirements.

Low traffic does not automatically mean every service needs a separate RDS instance.

---

# 25. ALB

Internet traffic:

```text
Internet
   |
   v
Route 53
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
Ingress
   |
   v
Kubernetes Services
```

ALB routes traffic using host/path rules.

Example:

```text
api.example.com/auth
        -> auth-service

api.example.com/products
        -> product-service

api.example.com/orders
        -> order-service

api.example.com/cart
        -> cart-service
```

---

# 26. Internal Service Communication

Not every request should go through ALB.

For internal calls:

```text
order-service
     |
     v
customer-service
```

use Kubernetes service DNS:

```text
http://customer-service:8080
```

or the appropriate namespace-qualified DNS name.

Therefore:

```text
Internet
   |
   v
ALB
   |
   v
external API

Internal
   |
   v
Kubernetes Service
   |
   v
Pod
```

---

# 27. NetworkPolicy

Even in low traffic, we don't want every pod communicating with every other pod.

Example:

```text
order-service
     |
     +---- customer-service
     |
     +---- inventory-service
     |
     +---- payment-service
```

But:

```text
order-service
      X
      |
notification-service
```

if there is no legitimate dependency.

NetworkPolicy should implement the intended service communication graph.

---

# 28. Secrets

Production credentials should not be placed directly into:

```text
Git
Dockerfile
Helm values
GitHub repository
```

Instead:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Spring Boot
```

For example:

```text
DB_USERNAME
DB_PASSWORD
REDIS_PASSWORD
PAYMENT_API_KEY
```

---

# 29. IAM

Application pods should use workload identity.

Architecture:

```text
order-service Pod
       |
       v
EKS Pod Identity
       |
       v
IAM Role
       |
       +---- SQS
       +---- Secrets Manager
       +---- S3
```

Do not put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside application containers.

---

# 30. Observability in Low Traffic

We still deploy:

```text
Prometheus
Grafana
Alertmanager
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry Collector
Jaeger
```

However, we size them according to actual telemetry volume.

Architecture:

```text
                 EKS
                  |
     +------------+-------------+
     |            |             |
 Prometheus     Fluent Bit     OTel
     |            |             |
     v            v             v
 Grafana       Logstash      Jaeger
     |
 Alertmanager
```

---

# 31. Prometheus

Monitor:

```text
EKS
nodes
pods
deployments
services
HPA
KEDA
RDS
Redis
ALB
SQS
Kafka
applications
```

Important application metrics:

```text
request rate
error rate
latency
JVM memory
JVM GC
thread count
DB connection pool
business transactions
```

---

# 32. Grafana Low-Traffic Dashboard

At minimum:

```text
Dashboard 1
EKS Cluster

Dashboard 2
Microservices

Dashboard 3
RDS

Dashboard 4
Redis

Dashboard 5
Messaging

Dashboard 6
ALB

Dashboard 7
SLO
```

For each service:

```text
RPS
5xx
P95
P99
CPU
Memory
Restarts
Replicas
HPA status
DB connections
```

---

# 33. Logging

Application:

```text
Spring Boot
     |
     v
stdout/stderr
     |
     v
Fluent Bit
     |
     v
Logstash
     |
     v
Elasticsearch
     |
     v
Kibana
```

Example:

```json
{
  "level": "INFO",
  "service": "order-service",
  "environment": "prod",
  "message": "Order created",
  "orderId": "12345",
  "correlationId": "abc123"
}
```

Sensitive information must not be logged.

Never log:

```text
password
access token
secret
payment card data
private credentials
```

---

# 34. Distributed Tracing

Architecture:

```text
Client
  |
  v
ALB
  |
  v
order-service
  |
  +---- customer-service
  |
  +---- product-service
  |
  +---- pricing-service
  |
  +---- payment-service
```

OpenTelemetry records:

```text
trace
  |
  +-- span: order-service
  |
  +-- span: customer-service
  |
  +-- span: product-service
  |
  +-- span: payment-service
```

Then:

```text
OpenTelemetry Collector
          |
          v
       Jaeger
```

---

# 35. ArgoCD

Deployment flow:

```text
Developer
   |
   v
GitHub
   |
   v
CI
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

CI should not directly perform:

```text
kubectl apply
```

against production in this GitOps architecture.

---

# 36. Low-Traffic GitOps Structure

Example:

```text
gitops/
├── apps/
│   ├── order-service/
│   ├── payment-service/
│   └── ...
│
├── environments/
│   ├── dev/
│   ├── qa/
│   └── prod/
│
└── platform/
    ├── prometheus/
    ├── grafana/
    ├── alertmanager/
    ├── elk/
    ├── opentelemetry/
    └── jaeger/
```

---

# 37. Low-Traffic Deployment Example

Suppose:

```text
order-service
version = 1.2.0
```

CI:

```text
GitHub
  |
  v
Maven build
  |
  v
Unit tests
  |
  v
SonarQube
  |
  v
Trivy
  |
  v
Docker build
  |
  v
ECR
```

Image:

```text
order-service:1.2.0
```

or preferably an immutable image reference/digest.

Then:

```text
GitOps update
      |
      v
ArgoCD
      |
      v
EKS
```

---

# 38. Low-Traffic Cost Optimization

The goal is not:

> "Make everything as cheap as possible."

The goal is:

> "Remove unnecessary capacity without destroying reliability."

Good optimization:

```text
right-size pods
right-size nodes
scale workers from queue depth
use autoscaling
remove idle resources
control log retention
control metrics retention
use appropriate storage
```

Bad optimization:

```text
remove backups
disable monitoring
run database publicly
use one replica for critical services
disable security scanning
disable encryption
remove HA without understanding impact
```

---

# 39. Karpenter

Karpenter can provision nodes based on pending workloads.

Example:

```text
Traffic increases
       |
       v
HPA increases replicas
       |
       v
Pods become Pending
       |
       v
Karpenter detects capacity requirement
       |
       v
New EC2 node
       |
       v
Pods scheduled
```

For low traffic:

```text
few pods
   |
   v
few nodes
```

For higher traffic:

```text
more pods
   |
   v
more nodes
```

---

# 40. Scale-to-Zero Consideration

For some non-critical asynchronous services, scale-to-zero can be useful.

Example:

```text
reporting-service
```

If:

```text
queue = 0
```

there may be little reason to keep many workers running.

However, scale-to-zero should not automatically be applied to:

```text
auth-service
order-service
payment-service
```

because these are synchronous customer-facing workloads.

---

# 41. Low-Traffic Architecture by Service Type

### Customer-facing synchronous

```text
auth
user
customer
catalog
product
pricing
promotion
cart
order
payment
shipping
search
review
tax
recommendation
```

Typical:

```text
Deployment
+
Service
+
HPA
```

### Event-driven

```text
inventory
fraud
delivery
notification
invoice
reporting
```

Typical:

```text
Deployment
+
Queue/Topic
+
KEDA
```

Some services may use both HPA and event-driven scaling depending on their workload.

---

# 42. Low-Traffic Request Flow

Consider:

```text
Customer places order
```

Flow:

```text
Internet
   |
   v
Route 53
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
order-service
   |
   +---- customer-service
   |
   +---- product-service
   |
   +---- pricing-service
   |
   +---- promotion-service
   |
   +---- tax-service
   |
   v
PostgreSQL
   |
   v
Outbox/Event
   |
   v
Kafka
   |
   +---- inventory-service
   |
   +---- payment-service
   |
   +---- fraud-service
   |
   +---- notification-service
   |
   +---- reporting-service
```

This gives us both:

```text
synchronous request processing
```

and:

```text
asynchronous event processing
```

---

# 43. Failure Scenario — One Pod Crashes

Suppose:

```text
order-service

Pod A -> Running
Pod B -> Running
```

Pod A crashes.

Kubernetes:

```text
Pod A -> Failed
       |
       v
ReplicaSet
       |
       v
New Pod
```

Meanwhile:

```text
Pod B
```

continues serving traffic.

Monitoring:

```text
Prometheus
    |
    v
Alertmanager
    |
    v
Notification
```

Logs:

```text
Fluent Bit
   |
   v
ELK
```

Traces:

```text
OpenTelemetry
   |
   v
Jaeger
```

---

# 44. Failure Scenario — Node Failure

Suppose:

```text
AZ-A

Node A
   |
   +-- order-service-1
   +-- product-service-1
   +-- cart-service-1
```

Node A fails.

If replicas are distributed:

```text
AZ-B

Node B
   |
   +-- order-service-2
```

Kubernetes reschedules workloads when sufficient capacity exists.

Karpenter can provision replacement capacity where configured.

This is why:

```text
replica count
+
topology spread
+
PDB
+
multi-AZ nodes
+
autoscaling
```

work together.

---

# 45. Failure Scenario — RDS Unavailable

Application:

```text
order-service
      |
      v
RDS
      X
```

The application should not respond by endlessly retrying.

Instead use:

```text
connection timeout
query timeout
bounded retries
exponential backoff
circuit breaker where appropriate
```

Monitor:

```text
RDS connections
CPU
memory
IO
latency
locks
errors
failover events
```

---

# 46. Failure Scenario — Redis Unavailable

Application:

```text
order-service
      |
      v
Redis
      X
```

For cache use cases:

```text
Redis failure
      |
      v
fallback to RDS
```

But fallback must be controlled.

Otherwise:

```text
Redis failure
     |
     v
all traffic -> RDS
     |
     v
RDS overload
     |
     v
application failure
```

This is why cache recovery was covered in Chapter 111.

---

# 47. Failure Scenario — SQS Backlog

Suppose:

```text
notification queue

Messages:
10
20
50
100
1,000
10,000
```

Monitor:

```text
ApproximateNumberOfMessagesVisible
ApproximateAgeOfOldestMessage
```

Then:

```text
SQS
 |
 v
KEDA
 |
 v
consumer replicas
```

But:

```text
max replicas
```

must respect downstream provider limits.

---

# 48. Low-Traffic SLO

Suppose we define:

```text
Order API availability SLO:
99.9%
```

For a 30-day month:

```text
30 days × 24 × 60
=
43,200 minutes
```

Allowed downtime:

```text
43,200 × 0.1%
=
43.2 minutes
```

This is the error budget.

The SLO should be measured from actual customer-facing behavior, not simply:

```text
EC2 CPU < 80%
```

---

# 49. Low-Traffic Alert Examples

### High API error rate

```text
5xx > defined threshold
for defined duration
```

### High latency

```text
P95 > SLO threshold
```

### Pod crash

```text
restart count increasing
```

### RDS connection exhaustion

```text
connections approaching configured capacity
```

### SQS backlog

```text
oldest message age increasing
```

### Kafka consumer lag

```text
consumer lag continuously increasing
```

---

# 50. Capacity Calculation

Suppose load testing shows:

```text
one order-service pod
=
20 requests/sec
```

Peak traffic:

```text
60 requests/sec
```

Required pods:

```text
ceil(60 / 20)
=
3
```

Now add headroom:

```text
3 × safety factor
```

For example, if engineering policy chooses 30% headroom:

```text
3 × 1.3
=
3.9
```

Therefore:

```text
4 pods
```

would be a starting capacity target.

But this must be validated through actual load tests.

---

# 51. Node Capacity Calculation

Suppose:

```text
Pod CPU request = 100m
Pod memory request = 256Mi
```

and:

```text
20 pods
```

Then requested CPU:

```text
20 × 100m
=
2 CPU
```

Memory:

```text
20 × 256Mi
=
5120Mi
≈ 5Gi
```

But the cluster also needs:

```text
system pods
daemonsets
kube-system
monitoring
logging
networking
failure headroom
```

Therefore we don't size nodes using application requests alone.

---

# 52. IP Capacity

AWS VPC CNI assigns VPC IP addresses to pods.

Therefore:

```text
more pods
    |
    v
more IP addresses
    |
    v
subnet capacity
```

must be included in capacity planning.

Low traffic is easier, but we should still choose subnet sizes with future growth in mind.

---

# 53. Low-Traffic Architecture Summary

```text
                         INTERNET
                            |
                       Route 53
                            |
                      CloudFront/WAF
                            |
                           ALB
                            |
                     +------+------+
                     |             |
                    AZ-A          AZ-B
                     |             |
                  EKS Nodes      EKS Nodes
                     |             |
                     +------+------+
                            |
                     21 Microservices
                            |
        +-------------------+-------------------+
        |                   |                   |
       RDS                Redis             Messaging
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

# 54. Production Safety Checklist

## EKS

```text
[ ] Private worker nodes
[ ] Multi-AZ
[ ] Resource requests defined
[ ] Resource limits reviewed
[ ] HPA configured
[ ] KEDA configured where appropriate
[ ] Karpenter configured/tested
[ ] PDB configured for critical workloads
[ ] Topology spread configured
[ ] NetworkPolicy configured
```

## Application

```text
[ ] Readiness probe
[ ] Liveness probe
[ ] Startup probe where needed
[ ] Graceful shutdown
[ ] Timeouts
[ ] Retry policies
[ ] Idempotency
[ ] Structured logging
[ ] Correlation IDs
[ ] OpenTelemetry tracing
```

## Database

```text
[ ] Private RDS
[ ] Encryption
[ ] Backups
[ ] PITR
[ ] Multi-AZ where required
[ ] Connection pool limits
[ ] Migration strategy
[ ] Monitoring
```

## Messaging

```text
[ ] DLQ
[ ] Retry strategy
[ ] Idempotency
[ ] Visibility timeout
[ ] Consumer scaling
[ ] Backlog monitoring
```

## Security

```text
[ ] IAM least privilege
[ ] Pod Identity
[ ] Secrets Manager
[ ] NetworkPolicy
[ ] Security groups
[ ] Trivy
[ ] SonarQube
[ ] Image scanning
```

---

# 55. Low-Traffic Validation Plan

We should not consider the architecture complete until we test it.

### Test 1 — Normal traffic

```text
10 RPS
```

Measure:

```text
CPU
memory
latency
errors
RDS
Redis
```

### Test 2 — Moderate traffic

```text
30 RPS
```

Observe:

```text
HPA
pod count
node utilization
database connections
```

### Test 3 — Peak traffic

```text
60–100 RPS
```

Observe:

```text
HPA
Karpenter
RDS
Redis
ALB
```

### Test 4 — Pod failure

```text
delete one application pod
```

Verify:

```text
replacement
traffic continuity
alerts
```

### Test 5 — Node failure

Controlled test:

```text
terminate one test node
```

Verify:

```text
pod rescheduling
capacity recovery
```

### Test 6 — Redis failure

Verify:

```text
application fallback
RDS protection
alerts
```

### Test 7 — SQS backlog

Inject:

```text
large number of messages
```

Verify:

```text
KEDA
consumer scaling
backlog reduction
```

---

# 56. What We Should Observe During the Test

The most important mistake is looking only at Kubernetes.

We need the complete dependency chain.

```text
Traffic
   |
   v
ALB
   |
   v
Pods
   |
   +---- RDS
   |
   +---- Redis
   |
   +---- SQS/Kafka
   |
   v
External APIs
```

For each layer measure:

```text
Requests
Errors
Latency
Saturation
Capacity
```

---

# 57. Low-Traffic → Medium-Traffic Transition

The architecture should grow naturally.

Current:

```text
LOW TRAFFIC

10–30 RPS
```

Then:

```text
MEDIUM TRAFFIC

100–500 RPS
```

Then:

```text
HIGH TRAFFIC

500–5000+ RPS
```

The architecture doesn't need to be redesigned from scratch.

Instead:

```text
HPA
  |
  v
more pods

Karpenter
  |
  v
more nodes

RDS
  |
  v
larger capacity/read replicas/etc.

Redis
  |
  v
larger/scaled topology

Kafka/SQS
  |
  v
more consumers/partitions where appropriate
```

The exact thresholds must come from measured performance.

---

# 58. Most Important Production Lesson

A low-traffic system should **not** be designed as:

```text
Traffic is low
      |
      v
remove HA
remove monitoring
remove backups
remove security
```

Instead:

```text
Traffic is low
      |
      v
reduce unnecessary capacity
      |
      +--> right-size workloads
      +--> autoscale
      +--> control replicas
      +--> optimize telemetry
      +--> control storage
      +--> use managed services appropriately
      |
      v
KEEP production safety controls
```

---

# 59. Final Low-Traffic Architecture

```text
                           USERS
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
                 +-----------+-----------+
                 |                       |
                 v                       v
              EKS AZ-A                EKS AZ-B
                 |                       |
                 +-----------+-----------+
                             |
                      21 Microservices
                             |
             +---------------+---------------+
             |               |               |
             v               v               v
           RDS             Redis          Messaging
        PostgreSQL       ElastiCache       SQS/Kafka
             |               |               |
             +---------------+---------------+
                             |
                  +----------+----------+
                  |          |          |
                  v          v          v
             Prometheus     ELK       OTel
                  |          |          |
                  v          v          v
               Grafana    Kibana      Jaeger
                  |
                  v
             Alertmanager
```

The key scaling relationship is:

```text
LOW TRAFFIC
     |
     v
Small but HA-capable workload footprint
     |
     v
HPA/KEDA
     |
     v
Karpenter
     |
     v
MEDIUM TRAFFIC
     |
     v
HIGH TRAFFIC
```

This gives us the foundation for the next chapter, where we take the **same 21-microservice platform** and determine what changes when traffic moves from low to **medium traffic**.

---

## Chapter 131 Complete

**Next:**

# 132 — MEDIUM TRAFFIC ARCHITECTURE

We will take this exact architecture and work through:

```text
LOW → MEDIUM TRAFFIC

10–30 RPS
      ↓
100–500 RPS
```

including **HPA behavior, Karpenter node scaling, RDS connection pressure, Redis scaling, Kafka/SQS consumer scaling, ALB capacity, pod distribution, database bottlenecks, and the exact differences between low- and medium-traffic architecture.**
