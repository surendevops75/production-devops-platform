# 119 — Grafana Production Dashboards

> **Goal:** Build production-grade Grafana dashboards for the 21-microservice E-Commerce Platform using Prometheus as the primary metrics source, with dashboards designed for operations, SRE, developers, and incident response.

---

## 1. Objective

In Chapter 118, Prometheus was implemented as the metrics collection and alerting data source.

Now we need the visualization and operational layer:

```text
21 Microservices
       │
       ├── Spring Boot /metrics
       ├── Kubernetes metrics
       ├── Node Exporter
       └── kube-state-metrics
              │
              ▼
        ┌────────────┐
        │ Prometheus │
        └─────┬──────┘
              │
              ▼
        ┌────────────┐
        │   Grafana  │
        └─────┬──────┘
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
   Dashboards Alerts   SRE
```

Grafana should answer questions such as:

```text
Is the platform healthy?

Which service is failing?

Are users experiencing errors?

Are pods restarting?

Are nodes running out of CPU/memory?

Is the database saturated?

Is Redis healthy?

Is Kafka lag increasing?

Is SQS backlog increasing?

Did the latest deployment cause the problem?
```

---

# 2. Production Dashboard Strategy

Do not create one giant dashboard containing everything.

Instead, create dashboards by operational purpose.

Recommended structure:

```text
Grafana
│
├── 01 - Executive / Business Overview
│
├── 02 - EKS Cluster Overview
│
├── 03 - Kubernetes Nodes
│
├── 04 - Kubernetes Pods
│
├── 05 - Microservices Overview
│
├── 06 - Order Service
│
├── 07 - Payment Service
│
├── 08 - Inventory Service
│
├── 09 - Kafka
│
├── 10 - RDS PostgreSQL
│
├── 11 - Redis
│
├── 12 - ALB
│
├── 13 - SQS
│
├── 14 - Application Golden Signals
│
├── 15 - SLO / SLA
│
└── 16 - Production Incident Dashboard
```

This gives different teams the views they actually need.

---

# 3. Grafana Namespace

Create a dedicated namespace:

```bash
kubectl create namespace monitoring
```

In production, this should be managed through GitOps rather than manually.

Example:

```text
gitops/
└── infrastructure/
    └── monitoring/
        ├── namespace.yaml
        ├── grafana/
        │   ├── values.yaml
        │   └── application.yaml
        └── prometheus/
```

ArgoCD manages the monitoring stack.

---

# 4. Grafana Installation

A commonly used production approach is the `kube-prometheus-stack` Helm chart, which includes Grafana alongside Prometheus-related components.

Example:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

Check the available chart:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

Before production installation:

```bash
helm show values prometheus-community/kube-prometheus-stack
```

Pin the chart version rather than relying on whatever happens to be latest.

Example:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version <PINNED_VERSION> \
  -f values.yaml
```

For this project, the long-term target is:

```text
Git
 ↓
ArgoCD
 ↓
Helm
 ↓
Grafana
```

not:

```text
Engineer laptop
 ↓
helm install
```

---

# 5. Grafana Production Configuration

Example:

```yaml
grafana:
  enabled: true

  replicas: 2

  resources:
    requests:
      cpu: 200m
      memory: 512Mi

    limits:
      cpu: "1"
      memory: 1Gi

  persistence:
    enabled: true
    storageClassName: gp3
    size: 20Gi
```

However, Grafana persistence is not the same thing as Prometheus metric persistence.

Prometheus needs its own persistent storage.

Grafana primarily stores:

```text
Dashboards
Users
Organizations
Preferences
Data source configuration
Alert configuration
```

depending on the chosen provisioning/storage model.

---

# 6. Grafana Database

For a small environment, Grafana's embedded database may be sufficient.

For a production enterprise environment, consider an external PostgreSQL database when Grafana scale and availability requirements justify it.

Architecture:

```text
                 ┌─────────────┐
                 │   Grafana   │
                 │   Pod 1     │
                 └──────┬──────┘
                        │
                 ┌──────▼──────┐
                 │ PostgreSQL  │
                 └─────────────┘
                        ▲
                 ┌──────┴──────┐
                 │   Grafana   │
                 │   Pod 2     │
                 └─────────────┘
```

For our platform, Grafana can initially use its supported deployment configuration and later move to an external PostgreSQL backend if required.

---

# 7. Grafana Access

Do not expose Grafana directly to the public Internet.

Bad:

```text
Internet
   │
   ▼
Grafana
```

Preferred:

```text
Corporate User
      │
      ▼
VPN / Zero Trust / Identity Proxy
      │
      ▼
Internal Load Balancer
      │
      ▼
Grafana
```

Alternatively:

```text
Engineer
   │
   ▼
VPN
   │
   ▼
Internal ALB
   │
   ▼
Grafana Service
```

Grafana should be treated as an operational platform component.

---

# 8. Authentication

Production Grafana should use centralized authentication where appropriate.

Examples:

```text
OIDC
SAML
LDAP
OAuth
```

Typical enterprise flow:

```text
Engineer
   │
   ▼
Grafana
   │
   ▼
Corporate Identity Provider
   │
   ▼
Authentication
   │
   ▼
Grafana Role
```

Possible roles:

```text
Admin
Editor
Viewer
```

Example organizational model:

```text
SRE Team
    │
    └── Admin / Editor

DevOps Team
    │
    └── Editor

Developers
    │
    └── Viewer

Management
    │
    └── Viewer
```

Avoid giving every engineer Grafana administrator access.

---

# 9. Prometheus Data Source

Grafana needs Prometheus as a data source.

Inside the Kubernetes cluster:

```text
Grafana
   │
   │ HTTP
   ▼
Prometheus Service
```

Example data source URL:

```text
http://prometheus-operated.monitoring.svc.cluster.local:9090
```

The exact service name depends on the Helm release and configuration.

Always verify:

```bash
kubectl get svc -n monitoring
```

Example:

```text
NAME                    TYPE        CLUSTER-IP
prometheus-operated     ClusterIP   10.x.x.x
grafana                 ClusterIP   10.x.x.x
alertmanager-operated   ClusterIP   10.x.x.x
```

---

# 10. Data Source Provisioning

Do not manually configure every production Grafana instance.

Use provisioning/GitOps.

Conceptually:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-operated:9090
    isDefault: true
```

The exact Helm values structure depends on the chart version.

The important production principle is:

```text
Configuration
      ↓
Git
      ↓
Review
      ↓
ArgoCD
      ↓
Grafana
```

---

# 11. Dashboard Variables

One of the most important Grafana features is dashboard templating.

Instead of creating:

```text
Order Service - Production
Order Service - QA
Order Service - Dev
```

create one dashboard:

```text
Order Service
```

with variables:

```text
environment
namespace
service
pod
instance
```

Example:

```text
Environment: [prod ▼]
Namespace:   [ecommerce ▼]
Service:     [order-service ▼]
Pod:         [All ▼]
```

Now the same dashboard works across environments.

---

# 12. Environment Variable

Example PromQL:

```promql
label_values(up, environment)
```

Depending on how labels are implemented, you may instead derive the environment from:

```text
namespace
cluster
job
```

The important requirement is to standardize labels.

For example:

```text
environment="prod"
cluster="ecommerce-prod"
namespace="ecommerce"
service="order-service"
```

---

# 13. Dashboard 01 — Executive / Business Overview

This dashboard should answer:

> Is the business platform healthy?

Do not fill it with Kubernetes internals.

Example panels:

```text
Orders Created
Orders Completed
Payment Success Rate
Payment Failure Rate
Order Error Rate
API Availability
Checkout Latency
Active Users
```

Example business metric:

```promql
sum(rate(orders_created_total[5m])) * 60
```

This gives approximate orders/minute if the metric represents a monotonically increasing counter.

---

# 14. Business Success Rate

Suppose we have:

```text
orders_completed_total
orders_failed_total
```

A conceptual success percentage:

```promql
100 *
sum(rate(orders_completed_total[5m]))
/
(
  sum(rate(orders_completed_total[5m]))
  +
  sum(rate(orders_failed_total[5m]))
)
```

This is much more useful to business stakeholders than:

```text
CPU = 43%
```

because CPU alone doesn't tell us whether customers can successfully place orders.

---

# 15. Dashboard 02 — EKS Cluster Overview

This dashboard focuses on the cluster.

Panels:

```text
Cluster CPU Usage
Cluster Memory Usage
Node Count
Ready Nodes
NotReady Nodes
Pod Count
Running Pods
Pending Pods
Failed Pods
Pod Restarts
CPU Requests
CPU Limits
Memory Requests
Memory Limits
```

Example node CPU:

```promql
100 *
(
  1 -
  avg by(instance)
  (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

---

# 16. Cluster Capacity

A useful capacity panel:

```text
CPU Allocatable
CPU Requested
CPU Utilization
Memory Allocatable
Memory Requested
Memory Utilization
```

Example:

```text
CPU Allocatable: 48 cores
CPU Requested:   31 cores
CPU Used:        19 cores
```

This immediately tells the platform team:

```text
We have capacity.

or

We are approaching scheduling limits.
```

---

# 17. Dashboard 03 — Kubernetes Nodes

Each node should be visible.

Example table:

| Node         | CPU | Memory | Pods | Disk | Status |
| ------------ | --: | -----: | ---: | ---: | ------ |
| ip-10-0-1-10 | 42% |    61% |   38 |  52% | Ready  |
| ip-10-0-2-11 | 67% |    73% |   51 |  69% | Ready  |
| ip-10-0-3-12 | 81% |    84% |   57 |  77% | Ready  |

Useful metrics:

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
node_network_receive_bytes_total
node_network_transmit_bytes_total
```

---

# 18. Node Disk Monitoring

Disk pressure is frequently overlooked.

Example:

```promql
100 *
(
  1 -
  node_filesystem_avail_bytes{mountpoint="/"}
  /
  node_filesystem_size_bytes{mountpoint="/"}
)
```

Monitor:

```text
/
 /var/lib/containerd
 /var/lib/kubelet
```

depending on the node image/runtime and filesystem layout.

---

# 19. Dashboard 04 — Kubernetes Pods

Important panels:

```text
Pods by Namespace
Pods Running
Pods Pending
Pods Failed
Pods Restarting
CrashLoopBackOff
OOMKilled
Container CPU
Container Memory
Pod Network
```

Restart query:

```promql
sum by(namespace, pod) (
  increase(kube_pod_container_status_restarts_total[15m])
)
```

This helps identify:

```text
Which pod started restarting recently?
```

---

# 20. Pending Pods

One of the most useful production panels:

```promql
sum by(namespace, pod) (
  kube_pod_status_phase{phase="Pending"}
)
```

If pending pods increase:

```text
Application
   ↓
HPA
   ↓
More replicas
   ↓
Scheduler
   ↓
No available node capacity
   ↓
Pods Pending
```

This could indicate:

```text
Node capacity problem
CPU requests too large
Memory requests too large
Affinity constraints
Taints/tolerations
Topology constraints
Insufficient IPs
```

---

# 21. Dashboard 05 — Microservices Overview

We have 21 services.

Create one dashboard containing:

```text
Service
Requests/sec
Error %
P50
P95
P99
Pods
Restarts
CPU
Memory
Availability
```

Example:

| Service           | RPS | Error % |   P95 | Pods | Restarts |
| ----------------- | --: | ------: | ----: | ---: | -------: |
| order-service     |  42 |    0.2% | 180ms |    4 |        0 |
| payment-service   |  18 |    0.1% | 240ms |    3 |        0 |
| inventory-service |  31 |    1.7% | 410ms |    4 |        2 |

This is the dashboard engineers open first during a service-level incident.

---

# 22. RED Method

For every HTTP service:

```text
R = Rate
E = Errors
D = Duration
```

### Rate

```promql
sum(rate(http_server_requests_seconds_count[5m]))
```

### Errors

```promql
sum(rate(http_server_requests_seconds_count{
  status=~"5.."
}[5m]))
```

### Error percentage

```promql
100 *
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
```

### Duration

Use histogram metrics.

Example p95:

```promql
histogram_quantile(
  0.95,
  sum by(le) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

---

# 23. Dashboard 06 — Order Service

Order service is our primary reference service.

Dashboard sections:

```text
HTTP
Database
JVM
Pods
Business
Dependencies
Deployment
```

---

## HTTP

```text
Request Rate
Error Rate
P50
P95
P99
4xx
5xx
```

---

## Database

```text
Connection Pool
Active Connections
Idle Connections
Connection Timeout
Query Latency
Database Errors
```

---

## JVM

```text
Heap Used
Heap Max
GC Pause
Threads
CPU
```

---

## Business

```text
Orders Created
Orders Updated
Orders Cancelled
Orders Completed
```

---

# 24. Dashboard 07 — Payment Service

Payment deserves its own operational dashboard.

Monitor:

```text
Payment Requests
Authorization Success
Authorization Failure
Capture Success
Capture Failure
Timeouts
Provider Errors
Latency
Retries
Idempotency Conflicts
```

Example:

```text
Payment request
      │
      ▼
payment-service
      │
      ├── provider latency
      ├── provider errors
      ├── retries
      └── successful payments
```

A payment system can have:

```text
CPU = 30%
Memory = 40%
```

while still experiencing serious business failure.

Therefore business metrics are critical.

---

# 25. Dashboard 08 — Inventory Service

Inventory metrics:

```text
Stock Checks
Stock Reservations
Reservation Failures
Out-of-Stock Events
Reservation Latency
Inventory DB Errors
Kafka Consumer Lag
```

A critical business metric:

```text
inventory_reservation_failures_total
```

A rising value can indicate:

```text
Database issue
Race condition
Insufficient inventory
Consumer lag
Downstream dependency failure
```

---

# 26. Dashboard 09 — Kafka

Kafka dashboard should contain:

```text
Broker Health
Topic Throughput
Producer Rate
Consumer Rate
Consumer Lag
Under-Replicated Partitions
Offline Partitions
ISR Changes
Request Latency
Disk Usage
```

---

# 27. Consumer Lag

Consumer lag is one of the most important Kafka metrics.

Conceptually:

```text
Producer
   │
   ▼
Kafka
   │
   ├── Message 100
   ├── Message 101
   ├── Message 102
   ├── Message 103
   └── Message 104
                ▲
                │
         Consumer processed 101
```

Lag:

```text
104 - 101 = 3
```

Monitor lag by:

```text
consumer group
topic
partition
```

For example:

```text
order-events
inventory-consumer
payment-consumer
notification-consumer
reporting-consumer
```

---

# 28. Kafka Business Interpretation

Suppose:

```text
order-service
    ↓
order-events
    ↓
inventory-service
```

If:

```text
Order creation = 500/sec
Inventory consumption = 300/sec
```

then backlog grows.

Grafana should make this obvious.

Example:

```text
Producer Rate       500/sec
Consumer Rate       300/sec
Lag                 ↑↑↑
```

This indicates the consumer side cannot keep up.

---

# 29. Dashboard 10 — RDS PostgreSQL

RDS dashboard should combine:

```text
Database
Connection Pool
Queries
Latency
Storage
CPU
Memory
IOPS
Locks
Replication
```

Depending on the exact AWS metrics/exporter architecture, combine Prometheus metrics with AWS/RDS telemetry where appropriate.

Important values:

```text
CPUUtilization
DatabaseConnections
FreeStorageSpace
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
FreeableMemory
```

---

# 30. Database Connections

One of the most important panels:

```text
Application Connections
RDS Connections
Connection Pool Usage
```

Remember:

```text
Pods × Maximum Pool Size
```

Example:

```text
10 pods
×
20 connections
=
200 potential connections
```

If multiple services share the database:

```text
order-service      100
payment-service     60
inventory-service   80
user-service        40
...
```

the total can become significant.

---

# 31. Dashboard 11 — Redis

Redis dashboard:

```text
Memory Used
Memory %
Commands/sec
Cache Hits
Cache Misses
Evictions
Connected Clients
CPU
Network
Replication
Failover
```

Cache hit ratio:

```promql
100 *
rate(redis_keyspace_hits_total[5m])
/
(
  rate(redis_keyspace_hits_total[5m])
  +
  rate(redis_keyspace_misses_total[5m])
)
```

A sudden drop in hit ratio can increase:

```text
Redis misses
    ↓
RDS queries
    ↓
Database load
    ↓
Latency
    ↓
Timeouts
```

This is why Redis dashboards should be connected conceptually to database dashboards.

---

# 32. Dashboard 12 — ALB

Monitor:

```text
Request Count
Target Response Time
HTTP 4xx
HTTP 5xx
Target 5xx
Healthy Targets
Unhealthy Targets
Rejected Connections
Connection Errors
```

Important distinction:

```text
ALB 5xx
```

versus:

```text
Target 5xx
```

The latter can indicate the application itself is returning errors.

---

# 33. ALB → Application Troubleshooting

Suppose users report:

> Checkout is failing.

Grafana investigation:

```text
ALB
 ↓
5xx increasing
 ↓
Target 5xx increasing
 ↓
payment-service error rate increasing
 ↓
payment-service latency increasing
 ↓
provider timeout increasing
```

Now the incident has a likely dependency chain.

---

# 34. Dashboard 13 — SQS

Monitor:

```text
Messages Available
Messages In Flight
Messages Delayed
Oldest Message Age
Messages Sent
Messages Deleted
Messages Received
DLQ Messages
```

Critical metric:

```text
ApproximateAgeOfOldestMessage
```

A queue may have:

```text
100 messages
```

and still be healthy.

But:

```text
10 messages
```

that have been waiting for 30 minutes may be a serious operational problem.

---

# 35. SQS Backlog Visualization

Dashboard:

```text
Messages Available
        │
        │       /
        │      /
        │     /
        │____/
        └────────────── time
```

Combine it with:

```text
Consumer replicas
Consumer processing rate
Oldest message age
DLQ count
```

Then you can distinguish:

```text
temporary traffic burst
```

from:

```text
consumer failure
```

---

# 36. Dashboard 14 — Application Golden Signals

Create a standard dashboard for all services.

Four major signals:

```text
Latency
Traffic
Errors
Saturation
```

---

## Traffic

```promql
sum(rate(http_server_requests_seconds_count[5m]))
```

## Errors

```promql
sum(rate(http_server_requests_seconds_count{
  status=~"5.."
}[5m]))
```

## Latency

```promql
histogram_quantile(
  0.95,
  sum by(le) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

## Saturation

Could include:

```text
CPU
Memory
Thread pool
Connection pool
Queue depth
Kafka lag
Database connections
```

---

# 37. Dashboard 15 — SLO / SLA

Example service SLO:

```text
Availability SLO = 99.9%
```

Monthly theoretical downtime for a 30-day month is approximately:

```text
43.2 minutes
```

But don't treat this as a license to consume the entire budget.

Track:

```text
SLO
Current availability
Error budget
Error budget consumed
Burn rate
```

Example:

```text
Availability:      99.96%
SLO:               99.90%
Budget remaining:  healthy
```

The actual SLO definitions should come from the service's agreed reliability objectives.

---

# 38. Error Budget

Conceptually:

```text
Allowed failure
       │
       ▼
Error Budget
       │
       ├── consumed slowly → normal
       │
       └── consumed rapidly → investigate
```

This helps SRE teams move from:

```text
"CPU looks okay"
```

to:

```text
"Are we consuming reliability budget too quickly?"
```

---

# 39. Dashboard 16 — Production Incident Dashboard

This is the dashboard used during an incident.

Put the most important information together:

```text
┌─────────────────────────────────────┐
│ Overall Availability                │
├─────────────────────────────────────┤
│ Request Rate     │ Error Rate       │
├─────────────────────────────────────┤
│ P95 Latency      │ P99 Latency      │
├─────────────────────────────────────┤
│ 5xx by Service                      │
├─────────────────────────────────────┤
│ Pod Restarts                         │
├─────────────────────────────────────┤
│ Node CPU / Memory                    │
├─────────────────────────────────────┤
│ RDS Connections / CPU                │
├─────────────────────────────────────┤
│ Redis Health                         │
├─────────────────────────────────────┤
│ Kafka Consumer Lag                   │
├─────────────────────────────────────┤
│ SQS Oldest Message Age               │
├─────────────────────────────────────┤
│ Recent Deployment Events              │
└─────────────────────────────────────┘
```

---

# 40. Deployment Correlation

One of the most useful operational features is identifying:

```text
Did the incident start after a deployment?
```

Example:

```text
12:00  Deployment
12:05  Error rate starts increasing
12:06  P95 increases
12:07  Pods restart
12:08  Customer complaints
```

Grafana annotations can mark deployment events on graphs.

Conceptually:

```text
Error Rate
   │
   │             /────
   │            /
   │___________/
               ↑
           Deployment
```

This makes incident investigation much faster.

---

# 41. Standard Dashboard Variables

Use variables such as:

```text
$environment
$cluster
$namespace
$service
$pod
$instance
```

Example PromQL:

```promql
sum(
  rate(http_server_requests_seconds_count{
    environment="$environment",
    namespace="$namespace",
    application="$service"
  }[5m])
)
```

For multi-select:

```text
$service
```

can represent:

```text
order-service
payment-service
inventory-service
```

---

# 42. Standard Labels

Every service should follow the same label conventions.

Recommended:

```text
environment
cluster
namespace
service
version
pod
instance
method
route
status
```

Example:

```text
environment="prod"
cluster="ecommerce-prod"
namespace="ecommerce"
service="order-service"
version="1.4.2"
```

This standardization is essential for reusable dashboards.

---

# 43. Avoid High Cardinality

Never blindly create Prometheus labels from values such as:

```text
orderId
customerId
paymentId
requestId
traceId
email
phone
```

Bad:

```text
http_requests_total{
    order_id="123456789"
}
```

Then:

```text
order_id="123456790"
order_id="123456791"
order_id="123456792"
...
```

This can create enormous numbers of time series.

Instead, keep those values in:

```text
logs
traces
events
```

not metric labels.

---

# 44. Grafana + Logs + Traces

Eventually Grafana should provide links between:

```text
Metrics
   ↓
Logs
   ↓
Traces
```

Example:

```text
Grafana
  │
  ├── Prometheus
  │
  ├── Elasticsearch
  │
  └── Tempo / Jaeger / another tracing backend
```

For a request:

```text
POST /api/v1/orders
```

you should eventually be able to go from:

```text
High latency
      ↓
Order service
      ↓
Specific trace
      ↓
Database span
      ↓
Specific application logs
```

That is the direction of the later observability chapters.

---

# 45. Dashboard Folder Structure

Recommended:

```text
Grafana
│
├── Executive
│   └── Business Overview
│
├── Kubernetes
│   ├── EKS Overview
│   ├── Nodes
│   └── Pods
│
├── Applications
│   ├── Microservices Overview
│   ├── Order Service
│   ├── Payment Service
│   └── Inventory Service
│
├── Data
│   ├── RDS
│   └── Redis
│
├── Messaging
│   ├── Kafka
│   └── SQS
│
├── Network
│   └── ALB
│
├── Reliability
│   ├── Golden Signals
│   └── SLO
│
└── Incidents
    └── Production Incident
```

---

# 46. GitOps Repository

Example:

```text
gitops/
└── infrastructure/
    └── monitoring/
        └── grafana/
            ├── application.yaml
            ├── values.yaml
            └── dashboards/
                ├── business-overview.json
                ├── eks-overview.json
                ├── nodes.json
                ├── pods.json
                ├── microservices.json
                ├── order-service.json
                ├── payment-service.json
                ├── inventory-service.json
                ├── kafka.json
                ├── rds.json
                ├── redis.json
                ├── alb.json
                ├── sqs.json
                ├── golden-signals.json
                ├── slo.json
                └── incident.json
```

---

# 47. Dashboard-as-Code

Do not make production dashboards dependent on one engineer's browser.

Bad:

```text
Engineer
   ↓
Creates dashboard manually
   ↓
Nobody knows configuration
```

Preferred:

```text
Dashboard JSON
      ↓
Git
      ↓
PR
      ↓
Review
      ↓
ArgoCD
      ↓
Grafana
```

Benefits:

```text
Version control
Review
Rollback
Auditability
Environment consistency
Disaster recovery
```

---

# 48. Dashboard Provisioning

A production model can use:

```text
ConfigMap
   ↓
Grafana dashboard provider
   ↓
Dashboard JSON
   ↓
Grafana
```

For example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-order-service
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  order-service.json: |
    {
      "title": "Order Service",
      ...
    }
```

The exact mechanism depends on the Grafana Helm configuration.

The key principle remains:

```text
Dashboard = code/configuration
```

---

# 49. Incident Example

Suppose users report:

> Checkout is slow.

Engineer opens:

```text
Production Incident Dashboard
```

Observes:

```text
Error rate: 0.2% → 4.8%
P95:        220ms → 2.7s
```

Then:

```text
Microservices Overview
```

shows:

```text
payment-service
Error: 8.4%
P95:   3.2s
```

Then:

```text
Payment Dashboard
```

shows:

```text
Provider timeout ↑
```

Then:

```text
ALB
 ↓
payment-service
 ↓
external payment provider
```

The incident is now narrowed to the payment dependency rather than the entire EKS cluster.

---

# 50. Another Incident Example — Database

Suppose:

```text
Order latency ↑
```

Grafana:

```text
Order Service
     │
     ├── CPU normal
     ├── Memory normal
     ├── P95 ↑
     └── DB latency ↑
              │
              ▼
          PostgreSQL
              │
              ├── connections ↑
              ├── CPU ↑
              └── locks ↑
```

This immediately directs investigation toward PostgreSQL rather than Kubernetes nodes.

---

# 51. Another Incident — Redis Failure

Suppose Redis becomes unavailable.

Grafana:

```text
Redis
  │
  ├── Availability ↓
  └── Cache hits ↓
          │
          ▼
Order Service
  │
  └── RDS requests ↑
          │
          ▼
PostgreSQL
  │
  ├── connections ↑
  └── latency ↑
```

This illustrates why dashboards must be correlated rather than isolated.

---

# 52. Grafana Alerting vs Alertmanager

We already have Prometheus/Alertmanager architecture.

Keep responsibilities clear.

```text
Prometheus
   │
   └── evaluates PrometheusRule
             │
             ▼
        Alertmanager
             │
             ├── Slack
             ├── PagerDuty
             ├── Email
             └── other channels
```

Grafana can also have its own alerting capabilities, but do not create an uncontrolled mixture of alert systems without a clear ownership model.

For this project:

```text
PrometheusRule
       ↓
Alertmanager
```

will remain the primary infrastructure/application alerting path.

Grafana is primarily the visualization and investigation layer.

---

# 53. Production Dashboard Rules

Every dashboard should have:

### Clear title

```text
Order Service — Production
```

### Environment

```text
prod
```

### Time range

```text
Last 1 hour
```

### Refresh

For incident dashboards:

```text
10s
30s
```

For normal dashboards:

```text
30s
1m
```

Do not refresh every dashboard every second.

That creates unnecessary query load.

---

# 54. Dashboard Time Ranges

Useful presets:

```text
Last 15 minutes
Last 1 hour
Last 6 hours
Last 24 hours
Last 7 days
```

During incidents:

```text
Last 30 minutes
```

is often the most useful starting point.

For post-incident analysis:

```text
Before incident
During incident
After recovery
```

---

# 55. Production Grafana Performance

Grafana can become expensive if dashboards issue huge queries.

Avoid:

```text
30 panels
×
1-second refresh
×
7-day range
×
high-cardinality metrics
```

Instead:

```text
Appropriate time range
+
recording rules
+
reasonable refresh interval
+
well-designed PromQL
```

Prometheus recording rules are particularly useful for expensive repeated calculations.

---

# 56. Recording Rules

For example:

```yaml
groups:
  - name: ecommerce-recording-rules

    interval: 30s

    rules:

      - record: service:http_requests_per_second:rate5m

        expr:
          sum by (service) (
            rate(
              http_server_requests_seconds_count[5m]
            )
          )
```

Grafana can then query:

```promql
service:http_requests_per_second:rate5m
```

instead of repeatedly calculating the expensive expression.

---

# 57. Standard Service Dashboard Template

Every one of the 21 services should eventually follow the same layout.

```text
Service Overview
│
├── Request Rate
├── Error Rate
├── P50
├── P95
├── P99
│
├── CPU
├── Memory
├── Restarts
│
├── JVM
│   ├── Heap
│   ├── GC
│   └── Threads
│
├── Database
│   ├── Connections
│   └── Latency
│
├── Dependencies
│   ├── Redis
│   ├── Kafka
│   ├── SQS
│   └── HTTP dependencies
│
└── Business Metrics
```

This becomes extremely valuable when the project expands from 3 services to all 21.

---

# 58. 21-Service Dashboard Matrix

| Service                | Main Business Metrics | Important Dependencies |
| ---------------------- | --------------------- | ---------------------- |
| auth-service           | Login success/failure | DB, Redis              |
| user-service           | User operations       | PostgreSQL             |
| customer-service       | Customer operations   | PostgreSQL             |
| catalog-service        | Catalog requests      | PostgreSQL, Redis      |
| product-service        | Product operations    | PostgreSQL, Redis      |
| inventory-service      | Reservations          | PostgreSQL, Kafka      |
| pricing-service        | Price calculations    | Redis, DB              |
| promotion-service      | Promotions            | Redis, DB              |
| cart-service           | Cart operations       | Redis                  |
| order-service          | Orders                | PostgreSQL, Kafka      |
| payment-service        | Payments              | DB, external provider  |
| fraud-service          | Fraud decisions       | Kafka                  |
| shipping-service       | Shipments             | Kafka, DB              |
| delivery-service       | Deliveries            | Kafka                  |
| notification-service   | Notifications         | SQS, SNS               |
| search-service         | Search queries        | Search backend         |
| recommendation-service | Recommendations       | ML/data services       |
| review-service         | Reviews               | PostgreSQL             |
| tax-service            | Tax calculations      | External/API/DB        |
| invoice-service        | Invoices              | S3, DB                 |
| reporting-service      | Reports               | Kafka/data stores      |

---

# 59. Grafana Production Checklist

## Installation

* [ ] Grafana deployed through Helm/GitOps
* [ ] Version pinned
* [ ] Resources configured
* [ ] Persistence strategy defined
* [ ] HA requirements defined
* [ ] Authentication configured
* [ ] RBAC configured

## Data Source

* [ ] Prometheus configured
* [ ] Connection tested
* [ ] TLS/authentication configured where required
* [ ] NetworkPolicy configured

## Dashboards

* [ ] EKS dashboard
* [ ] Node dashboard
* [ ] Pod dashboard
* [ ] Microservices dashboard
* [ ] Order dashboard
* [ ] Payment dashboard
* [ ] Inventory dashboard
* [ ] Kafka dashboard
* [ ] RDS dashboard
* [ ] Redis dashboard
* [ ] ALB dashboard
* [ ] SQS dashboard
* [ ] Golden Signals
* [ ] SLO
* [ ] Incident dashboard

## Operations

* [ ] Dashboard-as-code
* [ ] Git review
* [ ] Environment variables
* [ ] Deployment annotations
* [ ] Recording rules
* [ ] Cardinality controls
* [ ] Query performance tested

---

# 60. Final Production Architecture

```text
                       USERS
                         │
                         ▼
                Route 53 / CloudFront
                         │
                         ▼
                       ALB
                         │
                         ▼
                 ┌───────────────┐
                 │      EKS      │
                 │               │
                 │ 21 Services   │
                 │               │
                 └───────┬───────┘
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
  Prometheus           Redis              RDS
       │
       │
       ├──── Node Exporter
       │
       ├──── kube-state-metrics
       │
       └──── Application Metrics
                         │
                         ▼
                     Grafana
                         │
       ┌─────────────────┼───────────────────┐
       ▼                 ▼                   ▼
   Dashboards        Investigation        SLO
       │
       ├── Business
       ├── Kubernetes
       ├── Services
       ├── Kafka
       ├── Redis
       ├── RDS
       ├── SQS
       └── Incidents
```

The operational workflow becomes:

```text
Customer reports problem
        ↓
Grafana Incident Dashboard
        ↓
Golden Signals
        ↓
Affected Service
        ↓
Service Dashboard
        ↓
Dependency Dashboard
        ↓
Logs
        ↓
Trace
        ↓
Root Cause
        ↓
Remediation
```

That is the actual purpose of Grafana in this project: **not just pretty graphs, but a common operational interface for detecting, narrowing, and investigating production problems.**

---

# Next Chapter

## `120-Alertmanager-Production-Alerts.md`

We will build the actual alerting layer:

```text
Prometheus
    ↓
PrometheusRule
    ↓
Alertmanager
    ↓
┌───────────┬────────────┬────────────┐
│ Slack     │ PagerDuty  │ Email      │
└───────────┴────────────┴────────────┘
```

including:

* alert severity design
* warning vs critical
* routing
* grouping
* inhibition
* silences
* maintenance windows
* Slack/PagerDuty/email
* SLO burn-rate alerts
* Kubernetes alerts
* application alerts
* RDS alerts
* Redis alerts
* Kafka lag alerts
* SQS backlog alerts
* alert fatigue prevention
* escalation policies
* production incident flow
* real incident examples.
