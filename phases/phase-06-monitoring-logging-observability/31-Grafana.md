# 31 — Grafana

> Production-grade Grafana implementation for the Enterprise E-Commerce Platform on AWS EKS

---

## 1. Purpose

Grafana is the **visualization and observability UI** for our platform.

Prometheus collects and stores metrics.

Grafana queries those metrics and turns them into:

* dashboards
* graphs
* tables
* gauges
* service health views
* Kubernetes capacity views
* AWS infrastructure views
* SLO views
* operational troubleshooting views

Our monitoring architecture becomes:

```text
                    ┌─────────────────────┐
                    │       Grafana       │
                    │ Dashboards / Queries│
                    └──────────┬──────────┘
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
       ┌───────────┐     ┌────────────┐    ┌───────────┐
       │ Prometheus│     │ CloudWatch │    │Elasticsearch
       └───────────┘     └────────────┘    └───────────┘
             │                 │                 │
             ▼                 ▼                 ▼
        Kubernetes          AWS Services       Logs
        Application          RDS              ELK
        Metrics              SQS
        Node Metrics         Redis
```

Grafana does **not** replace Prometheus.

```text
Prometheus = metrics collection + storage + PromQL
Grafana    = visualization + dashboards + exploration
```

---

# 2. Production Architecture

Our EKS observability architecture:

```text
                         USERS
                           │
                           ▼
                    Internal Access
                           │
                    ALB / Ingress
                           │
                           ▼
                     ┌───────────┐
                     │  Grafana  │
                     │  EKS      │
                     └─────┬─────┘
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
       ▼                   ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌──────────────┐
│ Prometheus  │      │ CloudWatch  │      │ Elasticsearch│
│             │      │             │      │              │
│ Kubernetes  │      │ AWS Metrics │      │ Application  │
│ App Metrics │      │ RDS         │      │ Logs         │
│ Node Metrics│      │ SQS         │      │              │
└─────────────┘      │ Redis       │      └──────────────┘
                     └─────────────┘
```

Grafana itself should normally be treated as a **platform component**, not as part of an application namespace.

Example:

```text
argocd
monitoring
logging
tracing
production
```

Grafana:

```text
namespace: monitoring
```

---

# 3. Where Grafana Runs

For our EKS platform:

```text
AWS
└── VPC
    └── EKS
        ├── system nodes
        ├── application nodes
        │
        └── monitoring
            ├── Prometheus
            ├── Grafana
            ├── Alertmanager
            ├── kube-state-metrics
            └── Node Exporter
```

Grafana runs as Kubernetes Pods.

Example:

```bash
kubectl get pods -n monitoring
```

Possible result:

```text
grafana-6f7c8d9f7b-xk82m
grafana-6f7c8d9f7b-pq91d
prometheus-kube-prometheus-prometheus-0
alertmanager-kube-prometheus-alertmanager-0
```

---

# 4. How Grafana Is Installed

Do not manually install Grafana on an EKS node.

Production approach:

```text
Terraform
   │
   └── EKS infrastructure

Helm
   │
   └── Grafana deployment

GitOps
   │
   └── ArgoCD manages Helm configuration
```

Recommended operational model:

```text
Git
 ↓
GitHub PR
 ↓
Review
 ↓
ArgoCD
 ↓
Helm
 ↓
EKS
 ↓
Grafana
```

This gives us:

* version control
* audit history
* repeatability
* rollback
* environment consistency

---

# 5. Helm Installation

For a learning environment:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Create namespace:

```bash
kubectl create namespace monitoring
```

Install:

```bash
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --version <approved-chart-version>
```

In production, do not blindly use the latest chart.

Pin:

```text
Grafana version
+
Helm chart version
```

and promote changes through GitOps.

---

# 6. GitOps Structure

Example:

```text
gitops/
├── applications/
│
├── environments/
│   ├── dev/
│   │   └── monitoring/
│   │
│   ├── qa/
│   │   └── monitoring/
│   │
│   └── prod/
│       └── monitoring/
│           └── grafana/
│               ├── values.yaml
│               ├── datasources.yaml
│               ├── dashboards/
│               └── dashboards.yaml
│
└── argocd/
    └── applications/
        └── grafana.yaml
```

---

# 7. Grafana Resources

Grafana needs resources just like applications.

Example:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

The correct values depend on:

* number of users
* dashboard count
* dashboard complexity
* query frequency
* datasource latency
* alert rules
* dashboard refresh interval

Do not blindly copy resource values into production.

Measure first.

---

# 8. High Availability

For production:

```yaml
replicas: 2
```

Architecture:

```text
                ALB
                 │
          ┌──────┴──────┐
          ▼             ▼
      Grafana-1     Grafana-2
```

Spread replicas across AZs/nodes using:

* topology spread constraints
* pod anti-affinity
* PDB
* appropriate node pools

Example:

```text
AZ-a → Grafana-1
AZ-b → Grafana-2
```

If one node fails:

```text
Grafana-1 → unavailable
Grafana-2 → continues serving
```

---

# 9. Grafana Persistent Storage

Grafana can store:

* users
* dashboards
* folders
* preferences
* configuration metadata
* alert configuration depending on architecture/version

For production HA, do not casually depend on a single local SQLite file.

Better:

```text
Grafana
   │
   ▼
PostgreSQL / supported external database
```

Example:

```text
Grafana-1 ──┐
            ├── PostgreSQL
Grafana-2 ──┘
```

The database becomes shared state.

Alternatively, dashboards and configuration can be managed through Git provisioning, reducing dependence on manually edited UI state.

---

# 10. Grafana Database

For an enterprise deployment:

```text
Grafana
   │
   ▼
RDS PostgreSQL
```

The Grafana database is separate from application databases.

Do not use:

```text
order-service DB
      ↓
Grafana metadata
```

Instead:

```text
order-service
   ↓
order database

Grafana
   ↓
observability database
```

---

# 11. Authentication

Never expose an administrative Grafana instance publicly without proper authentication.

Possible enterprise authentication:

```text
User
 ↓
SSO
 ↓
Identity Provider
 ↓
Grafana
```

Examples:

* OIDC
* OAuth
* SAML depending on edition/configuration
* enterprise identity provider

Typical organization:

```text
Engineering
├── Developers
├── DevOps
├── SRE
├── Platform
└── Security
```

Map groups to Grafana roles.

---

# 12. Grafana RBAC

Typical permissions:

```text
Admin
  ↓
Platform/SRE

Editor
  ↓
DevOps

Viewer
  ↓
Developers / Support
```

Example:

```text
production dashboards
      │
      ├── SRE → edit
      ├── DevOps → edit
      ├── Developers → view
      └── Support → view
```

Production dashboard modification should be controlled.

---

# 13. Datasources

Grafana does not generate the metrics.

It connects to datasources.

Our major datasources:

```text
Grafana
├── Prometheus
├── CloudWatch
├── Elasticsearch
└── Jaeger
```

Later:

```text
Grafana
 ├── Prometheus → metrics
 ├── CloudWatch → AWS metrics
 ├── Elasticsearch → logs
 └── Jaeger → traces
```

---

# 14. Prometheus Datasource

Prometheus is our primary metrics datasource.

Conceptually:

```text
Grafana
   │
   │ PromQL
   ▼
Prometheus
   │
   ▼
Metrics
```

Example query:

```promql
up
```

Another:

```promql
rate(http_server_requests_seconds_count[5m])
```

Grafana sends the PromQL query to Prometheus.

---

# 15. Prometheus URL

Inside Kubernetes:

```text
http://prometheus-operated.monitoring.svc.cluster.local:9090
```

The exact Service name depends on the Prometheus deployment/chart.

Always discover it:

```bash
kubectl get svc -n monitoring
```

Example:

```text
prometheus-operated   ClusterIP   10.0.20.10   9090
grafana               ClusterIP   10.0.20.20   80
```

Grafana should normally access Prometheus over the internal cluster network.

---

# 16. CloudWatch Datasource

AWS services expose metrics through CloudWatch.

For example:

```text
RDS
SQS
SNS
ElastiCache
ALB
NAT Gateway
EKS
```

Grafana can query CloudWatch.

Architecture:

```text
Grafana
   │
   ▼
CloudWatch API
   │
   ├── RDS
   ├── SQS
   ├── ALB
   └── ElastiCache
```

---

# 17. Grafana AWS Authentication

Avoid:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

stored permanently inside Grafana.

Prefer workload identity.

For example:

```text
Grafana Pod
    │
    ▼
EKS Pod Identity
    │
    ▼
IAM Role
    │
    ▼
CloudWatch permissions
```

Grant only required permissions.

---

# 18. Example CloudWatch Permissions

Depending on the datasource/configuration:

```text
cloudwatch:GetMetricData
cloudwatch:GetMetricStatistics
cloudwatch:ListMetrics
```

Do not automatically attach:

```text
AdministratorAccess
```

to Grafana.

---

# 19. Elasticsearch Datasource

Later in our ELK architecture:

```text
Application
 ↓
stdout
 ↓
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
```

Grafana can query Elasticsearch directly.

```text
Grafana
   │
   ▼
Elasticsearch
   │
   ▼
Application logs
```

This is useful for dashboards combining metrics and logs.

Example:

```text
CPU spike
   ↓
Grafana metric
   ↓
Identify affected service
   ↓
Open Elasticsearch logs
   ↓
Find application exception
```

Kibana remains our primary log exploration UI.

---

# 20. Jaeger Datasource

Later:

```text
Application
   ↓
OpenTelemetry
   ↓
Jaeger
```

Grafana can query trace data depending on the chosen tracing backend/integration.

Then we can correlate:

```text
Metrics
   ↓
Logs
   ↓
Traces
```

Example:

```text
Order API latency increased
        │
        ▼
Grafana
        │
        ▼
Trace ID
        │
        ▼
Jaeger
        │
        ▼
payment-service
        │
        ▼
RDS query
```

---

# 21. Dashboard Design

Do not create one giant dashboard containing everything.

Instead create focused dashboards.

Example:

```text
Grafana
│
├── Kubernetes
│   ├── Cluster Overview
│   ├── Node Health
│   ├── Namespace Health
│   └── Pod Health
│
├── Applications
│   ├── Order Service
│   ├── Payment Service
│   ├── Catalog Service
│   └── ...
│
├── AWS
│   ├── RDS
│   ├── Redis
│   ├── SQS
│   └── ALB
│
├── GitOps
│   └── ArgoCD
│
└── SRE
    ├── SLO
    └── Error Budget
```

---

# 22. Dashboard Hierarchy

For our 21 services:

```text
Applications
│
├── Business Overview
│
├── Service Overview
│
├── order-service
├── payment-service
├── inventory-service
├── catalog-service
├── cart-service
│
└── ...
```

Do not manually duplicate dashboards unnecessarily.

Use variables.

---

# 23. Grafana Variables

Example variable:

```text
namespace
```

Query:

```promql
label_values(kube_pod_info, namespace)
```

Another:

```text
service
```

Then dashboard panels can use:

```promql
{namespace="$namespace", app="$service"}
```

This allows one dashboard to work for many services.

---

# 24. Environment Variable

Create:

```text
environment
```

Values:

```text
dev
qa
prod
```

Dashboard:

```text
Environment: [prod]
Service:     [order-service]
Namespace:   [production]
```

Now the same dashboard can investigate different services/environments.

---

# 25. Service RED Dashboard

For microservices, one of the most useful dashboard patterns is RED:

```text
R = Rate
E = Errors
D = Duration
```

---

## Rate

Requests per second:

```promql
sum(
  rate(http_server_requests_seconds_count{
    application="$service"
  }[5m])
)
```

---

## Errors

Example:

```promql
sum(
  rate(http_server_requests_seconds_count{
    application="$service",
    status=~"5.."
  }[5m])
)
```

---

## Duration

p95 latency:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_server_requests_seconds_bucket{
      application="$service"
    }[5m])
  )
)
```

This tells us:

```text
95% of requests completed below X seconds
```

---

# 26. Example Order-Service Dashboard

Dashboard:

```text
Order Service - Production
```

Panels:

```text
┌─────────────────────────────────────────┐
│ Request Rate                            │
└─────────────────────────────────────────┘

┌──────────────────┬──────────────────────┐
│ Error Rate       │ P95 Latency          │
└──────────────────┴──────────────────────┘

┌──────────────────┬──────────────────────┐
│ Pod Count        │ Restart Count        │
└──────────────────┴──────────────────────┘

┌──────────────────┬──────────────────────┐
│ CPU              │ Memory               │
└──────────────────┴──────────────────────┘

┌─────────────────────────────────────────┐
│ Database Connection Pool                │
└─────────────────────────────────────────┘
```

---

# 27. Kubernetes Dashboard

Cluster overview should include:

```text
Nodes
Pods
CPU
Memory
Restarts
Pending Pods
Failed Pods
Namespaces
```

Useful metrics:

```promql
sum(kube_pod_container_resource_requests{resource="cpu"})
```

and:

```promql
sum(kube_pod_container_resource_requests{resource="memory"})
```

---

# 28. Node Dashboard

Use the USE methodology:

```text
U = Utilization
S = Saturation
E = Errors
```

Monitor:

```text
CPU
Memory
Disk
Filesystem
Network
Load
Pod density
```

Node Exporter provides many host-level metrics.

We will cover Node Exporter in chapter 33.

---

# 29. EKS Cluster Dashboard

Useful panels:

```text
Cluster CPU utilization
Cluster memory utilization
Node count
Ready nodes
NotReady nodes
Pending pods
Running pods
Failed pods
Pod restarts
API server latency
Scheduler health
Controller health
```

Also monitor AWS-specific EKS components:

```text
VPC CNI
CoreDNS
kube-proxy
AWS Load Balancer Controller
EBS CSI
```

---

# 30. RDS Dashboard

Our RDS dashboard should include:

```text
CPUUtilization
DatabaseConnections
FreeableMemory
FreeStorageSpace
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
NetworkReceiveThroughput
NetworkTransmitThroughput
```

Example:

```text
RDS
 │
 ├── CPU
 ├── Memory
 ├── Storage
 ├── Connections
 ├── IOPS
 └── Latency
```

---

# 31. Redis Dashboard

For ElastiCache:

```text
CPU
Memory
Evictions
Cache hits
Cache misses
Connections
Network bytes
Commands
Replication health
```

A sudden eviction increase could indicate:

```text
Application traffic ↑
      ↓
Cache memory pressure
      ↓
Evictions ↑
      ↓
Cache hit ratio ↓
      ↓
Database traffic ↑
```

Grafana should make this relationship visible.

---

# 32. SQS Dashboard

For each important queue:

```text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
ApproximateAgeOfOldestMessage
NumberOfMessagesSent
NumberOfMessagesReceived
NumberOfMessagesDeleted
```

Example:

```text
Order Events Queue
        │
        ├── Messages Visible
        ├── Messages In Flight
        ├── Oldest Message Age
        └── Consumer Throughput
```

The most operationally important metric is often:

```text
Age of oldest message
```

because it tells us whether consumers are falling behind.

---

# 33. ALB Dashboard

Useful metrics:

```text
RequestCount
TargetResponseTime
HTTPCode_Target_2XX_Count
HTTPCode_Target_4XX_Count
HTTPCode_Target_5XX_Count
HealthyHostCount
UnHealthyHostCount
```

A useful investigation:

```text
ALB 5xx ↑
   ↓
Target 5xx?
   ↓
Service error?
   ↓
Pod health?
```

---

# 34. Grafana Dashboard for ArgoCD

Create an operational GitOps dashboard:

```text
Applications
Synced
OutOfSync
Healthy
Degraded
Missing
Unknown
Sync failures
```

Example:

```text
Production Applications

order-service       Healthy
payment-service     Healthy
catalog-service     Degraded
inventory-service   Healthy
```

Then SRE immediately knows:

```text
Is this an application problem
or
a deployment problem?
```

---

# 35. Dashboard-as-Code

Do not make production dashboards only through the UI.

A better model:

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

Example:

```text
gitops/
└── monitoring/
    └── grafana/
        └── dashboards/
            ├── kubernetes-overview.json
            ├── order-service.json
            ├── rds.json
            └── sqs.json
```

This provides version control.

---

# 36. ConfigMap-Based Dashboard Provisioning

A simplified example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  order-service.json: |
    {
      "title": "Order Service",
      "panels": []
    }
```

The exact label/configuration depends on the Grafana Helm chart/provisioning setup being used.

The important production principle is:

```text
Dashboard definition → Git
```

not:

```text
Dashboard exists only in someone's browser
```

---

# 37. Dashboard Folder Structure

Example:

```text
Grafana
│
├── Kubernetes
│
├── Applications
│   ├── Order
│   ├── Payment
│   ├── Catalog
│   └── Inventory
│
├── AWS
│   ├── RDS
│   ├── ElastiCache
│   ├── SQS
│   └── ALB
│
├── GitOps
│
└── SRE
    ├── SLO
    └── Capacity
```

---

# 38. Production Dashboard Principles

A dashboard should answer a question.

Bad:

```text
CPU
Memory
CPU
Memory
Random metric
Random graph
```

Good:

```text
Is the service healthy?

How much traffic are we receiving?

Are requests failing?

Is latency increasing?

Are pods healthy?

Is the database saturated?

Are queues backing up?
```

---

# 39. Avoid High-Cardinality Dashboard Queries

Bad labels:

```text
user_id
request_id
transaction_id
session_id
```

These can produce enormous numbers of time series.

Better:

```text
service
namespace
method
route
status
environment
```

Be especially careful with:

```text
route
```

Use normalized routes:

```text
/api/v1/orders/{id}
```

rather than:

```text
/api/v1/orders/12345
/api/v1/orders/12346
/api/v1/orders/12347
```

---

# 40. Dashboard Query Optimization

A dashboard with 30 panels can generate many PromQL queries.

Suppose:

```text
20 panels
×
10-second refresh
×
100 users
```

The monitoring system itself can become expensive.

Avoid unnecessarily aggressive refresh intervals.

Typical operational choices:

```text
10s → only where genuinely needed
30s → common operational dashboard
1m  → general monitoring
5m  → historical/capacity views
```

---

# 41. Recording Rules

If an expensive query is repeatedly used:

```promql
histogram_quantile(
  0.95,
  ...
)
```

consider a Prometheus recording rule.

For example:

```yaml
groups:
  - name: ecommerce.rules
    rules:
      - record: service:http_request_rate:5m
        expr: |
          sum by (service) (
            rate(http_server_requests_seconds_count[5m])
          )
```

Grafana then queries:

```promql
service:http_request_rate:5m
```

instead of repeatedly calculating the expensive expression.

---

# 42. Grafana and Alerting

Grafana can display alert state, but our architecture separates responsibilities.

```text
Prometheus
    │
    ▼
Alert Rules
    │
    ▼
Alertmanager
    │
    ├── Slack
    ├── Email
    ├── PagerDuty
    └── Incident tooling
```

Grafana:

```text
visualization
+
exploration
+
alert visibility
```

Chapter 32 will cover Alertmanager deeply.

---

# 43. SLO Dashboard

For critical services:

```text
Order Service
```

Dashboard:

```text
Availability SLO
99.9%

Current availability
99.95%

Error budget remaining
72%

Burn rate
0.8x
```

This changes monitoring from:

```text
"CPU is 80%"
```

to:

```text
"Are customers receiving a reliable service?"
```

---

# 44. Business + Technical Observability

Grafana should eventually provide both.

Technical:

```text
CPU
Memory
Latency
Errors
Database
Queues
```

Business:

```text
Orders/minute
Payment success rate
Checkout conversion
Failed payments
Order processing delay
```

For example:

```text
Orders created
        ↓
Payment success
        ↓
Inventory reservation
        ↓
Shipping creation
```

This allows engineers to connect infrastructure health to business impact.

---

# 45. Example High-Traffic Incident

Suppose traffic suddenly increases.

Grafana:

```text
Request Rate
       ↑↑↑
```

Then:

```text
CPU
 ↑
```

HPA:

```text
3 pods → 10 pods
```

But:

```text
RDS connections
       ↑↑
```

Then:

```text
RDS connection pool saturation
```

and:

```text
Order latency
       ↑
```

Grafana allows us to see the chain:

```text
Traffic
  ↓
Pods
  ↓
Connections
  ↓
RDS
  ↓
Latency
```

---

# 46. Example Queue Incident

Suppose order consumers cannot process messages.

Grafana:

```text
SQS Visible Messages
       ↑↑↑↑
```

Then:

```text
Oldest Message Age
       ↑↑↑
```

Consumer pods:

```text
CPU = 20%
```

This suggests CPU isn't necessarily the bottleneck.

Next investigate:

```text
Application errors
Database connections
External dependency latency
Consumer concurrency
IAM failures
Network failures
```

---

# 47. Grafana Security

Grafana should generally be internal.

Architecture:

```text
Internet
   X
   │
   │
Grafana
   │
   └── Internal access only
```

Possible access:

```text
VPN
+
SSO
+
Internal ALB
```

Do not expose:

```text
grafana.company.com
```

without authentication and authorization.

---

# 48. Kubernetes NetworkPolicy

Example concept:

```text
Ingress
 ↓
Grafana
 ↓
Prometheus
```

NetworkPolicy should allow only required communication.

For example:

```text
Grafana
 ├── allowed → Prometheus:9090
 ├── allowed → CloudWatch API
 ├── allowed → Elasticsearch
 └── allowed → Jaeger
```

Do not allow unrestricted namespace-to-namespace traffic.

---

# 49. Secrets

Datasource credentials must not be committed.

Bad:

```yaml
password: SuperSecretPassword
```

inside Git.

Better:

```text
Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Grafana
```

Or use the supported authentication mechanism for the datasource.

Our GitOps repository should contain:

```text
secret references
```

rather than:

```text
secret values
```

---

# 50. TLS

Production Grafana should use HTTPS.

Example:

```text
User
 │
 ▼
HTTPS
 │
 ▼
ALB
 │
 ▼
Grafana Service
```

TLS termination can happen at the ALB depending on the architecture.

Internal datasource communication should also use TLS where supported and appropriate.

---

# 51. Grafana Logs

Grafana itself needs monitoring.

Collect:

```text
stdout/stderr
```

through our logging pipeline:

```text
Grafana
 ↓
stdout
 ↓
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
 ↓
Kibana
```

This lets us investigate:

```text
Datasource errors
Authentication failures
Plugin errors
Query failures
OOM issues
```

---

# 52. Monitor Grafana

Important Grafana health signals:

```text
CPU
Memory
Pod restarts
HTTP request latency
HTTP errors
Datasource errors
Database latency
Active users
Query duration
```

Grafana's own metrics can be exposed depending on configuration/version.

---

# 53. Grafana Availability

We should define an operational target.

Example:

```text
Grafana availability target:
99.9%
```

But remember:

Grafana is an observability interface.

If Grafana goes down while Prometheus continues collecting:

```text
Metrics are still being collected.
```

Therefore:

```text
Grafana outage != monitoring data outage
```

This distinction is important.

---

# 54. Prometheus vs Grafana

| Component          | Responsibility                       |
| ------------------ | ------------------------------------ |
| Prometheus         | Collect/store metrics                |
| Grafana            | Visualize/query metrics              |
| Alertmanager       | Route alerts                         |
| Node Exporter      | Host metrics                         |
| kube-state-metrics | Kubernetes object metrics            |
| Elasticsearch      | Log storage/search                   |
| Kibana             | Log visualization                    |
| OpenTelemetry      | Telemetry collection/instrumentation |
| Jaeger             | Distributed tracing                  |

Mental model:

```text
Exporters
   ↓
Prometheus
   ↓
Grafana

Applications
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
   ↓
Kibana

Applications
   ↓
OpenTelemetry
   ↓
Jaeger
```

---

# 55. Grafana Production Troubleshooting

## Problem 1 — Grafana Pod CrashLoopBackOff

Check:

```bash
kubectl get pods -n monitoring
```

Then:

```bash
kubectl logs -n monitoring <grafana-pod>
```

Also:

```bash
kubectl describe pod -n monitoring <grafana-pod>
```

Check:

```text
Configuration
Environment variables
Secrets
Database connection
Memory
Volume
Permissions
```

---

# 56. Problem 2 — Grafana Cannot Reach Prometheus

Test:

```bash
kubectl get svc -n monitoring
```

Then from Grafana:

```bash
kubectl exec -n monitoring -it <grafana-pod> -- \
  curl http://<prometheus-service>:9090/-/ready
```

Possible causes:

```text
Wrong Service name
Wrong port
NetworkPolicy
DNS
Prometheus unavailable
TLS configuration
```

---

# 57. Problem 3 — Dashboard Shows No Data

Start with Grafana:

```text
Datasource
   ↓
Query
   ↓
Variables
   ↓
Prometheus
```

Test the query directly in Prometheus.

For example:

```promql
up
```

If:

```text
Prometheus has data
Grafana doesn't
```

investigate:

```text
Datasource configuration
Time range
Variables
Query
Permissions
```

---

# 58. Problem 4 — Dashboard Is Slow

Investigate:

```text
Query complexity
Time range
Panel count
Refresh interval
High cardinality
Prometheus load
Recording rules
```

Example bad query:

```promql
huge aggregation
+
30-day range
+
many labels
+
10-second refresh
```

Better:

```text
short operational range
+
recording rules
+
controlled labels
+
reasonable refresh
```

---

# 59. Problem 5 — Grafana Memory Increases

Check:

```bash
kubectl top pod -n monitoring
```

Then inspect:

```text
Dashboard count
Concurrent users
Query volume
Plugins
Datasource behavior
Dashboard refresh intervals
```

Do not simply increase memory.

Find the workload causing the increase.

---

# 60. Production Grafana Deployment Model

Our final model:

```text
                   GitHub
                     │
                     ▼
              GitOps Repository
                     │
                     ▼
                   ArgoCD
                     │
                     ▼
                    Helm
                     │
                     ▼
                  EKS
                     │
              ┌──────┴──────┐
              ▼             ▼
          Grafana-1     Grafana-2
              │             │
              └──────┬──────┘
                     │
          ┌──────────┼───────────┐
          ▼          ▼           ▼
     Prometheus  CloudWatch  Elasticsearch
          │          │           │
          ▼          ▼           ▼
      Metrics       AWS          Logs
```

---

# 61. Full E-Commerce Observability View

Our platform now looks like:

```text
                         USERS
                           │
                           ▼
                  Route53 / CloudFront
                           │
                           ▼
                       ALB / WAF
                           │
                           ▼
                         EKS
                           │
       ┌───────────────────┼──────────────────┐
       │                   │                  │
       ▼                   ▼                  ▼
  Microservices         Redis              SQS/SNS
       │                   │                  │
       ▼                   ▼                  ▼
      RDS               Cache             Messaging
       │
       │
       └───────────────────────┐
                               │
                         OBSERVABILITY
                               │
        ┌──────────────────────┼─────────────────────┐
        │                      │                     │
        ▼                      ▼                     ▼
   Prometheus              Fluent Bit          OpenTelemetry
        │                      │                     │
        ▼                      ▼                     ▼
     Grafana               Logstash               Jaeger
        │                      │
        │                      ▼
        │                Elasticsearch
        │                      │
        │                      ▼
        │                    Kibana
        │
        ▼
    Alertmanager
```

---

# 62. What Grafana Gives the DevOps/SRE Team

Instead of manually checking:

```bash
kubectl top pods
kubectl get pods
kubectl get nodes
aws cloudwatch ...
aws rds ...
aws sqs ...
```

we can start with:

```text
Grafana
   ↓
Production Overview
```

Then drill down:

```text
Cluster
  ↓
Namespace
  ↓
Service
  ↓
Pod
  ↓
Dependency
  ↓
Logs
  ↓
Trace
```

This is the operational workflow we want.

---

# 63. Production Checklist

Before calling Grafana production-ready:

### Deployment

* [ ] Helm version pinned
* [ ] Grafana version pinned
* [ ] GitOps managed
* [ ] ArgoCD managed
* [ ] At least 2 replicas where HA is required
* [ ] PDB configured
* [ ] AZ/node spreading configured
* [ ] Resource requests configured

### Authentication

* [ ] SSO configured
* [ ] Admin access restricted
* [ ] RBAC configured
* [ ] Production access audited

### Datasources

* [ ] Prometheus configured
* [ ] CloudWatch configured
* [ ] Elasticsearch configured if required
* [ ] Jaeger configured if required
* [ ] TLS configured
* [ ] Least-privilege IAM

### Dashboards

* [ ] Kubernetes dashboard
* [ ] Node dashboard
* [ ] Service RED dashboard
* [ ] RDS dashboard
* [ ] Redis dashboard
* [ ] SQS dashboard
* [ ] ALB dashboard
* [ ] ArgoCD dashboard
* [ ] SLO dashboard

### Operations

* [ ] Dashboards stored as code
* [ ] Query performance reviewed
* [ ] Cardinality controlled
* [ ] Refresh intervals controlled
* [ ] Grafana monitored
* [ ] Grafana logs centralized
* [ ] Backup/recovery strategy defined

---

# 64. Most Important Production Lessons

### Lesson 1

Grafana is **not the monitoring system by itself**.

```text
Prometheus = metrics
Grafana = visualization
```

### Lesson 2

Do not manually configure production dashboards and forget about them.

Use:

```text
GitOps + dashboard-as-code
```

### Lesson 3

Do not expose Grafana publicly without strong authentication.

### Lesson 4

Do not give Grafana excessive AWS permissions.

### Lesson 5

Do not create dashboards full of high-cardinality queries.

### Lesson 6

A dashboard should help answer an operational question.

### Lesson 7

Observability is not:

```text
CPU + Memory
```

It is:

```text
Metrics
+
Logs
+
Traces
+
Context
```

---

# 65. Final Mental Model

Remember this:

```text
                    ┌──────────────┐
                    │   Grafana    │
                    │   "SEE IT"   │
                    └──────┬───────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
  Prometheus          Elasticsearch          Jaeger
  "MEASURE"             "SEARCH"            "TRACE"
       │                   │                   │
       ▼                   ▼                   ▼
    Metrics               Logs               Traces
```

And operationally:

```text
Something looks wrong
        │
        ▼
     Grafana
        │
        ├── Metrics → Prometheus
        │
        ├── Logs → Elasticsearch/Kibana
        │
        └── Traces → Jaeger
```

That is the core observability workflow for our EKS production platform.

---

## Next

**`32-Alertmanager.md`**

That chapter will connect the metrics we built with Prometheus to **real production alerting**:

```text
Prometheus
    ↓
Alert Rules
    ↓
Alertmanager
    ↓
Grouping
    ↓
Deduplication
    ↓
Inhibition
    ↓
Routing
    ↓
Slack / Email / PagerDuty
    ↓
On-call Engineer
```

It will also cover **severity design, alert fatigue, SLO/burn-rate alerts, maintenance silences, escalation, HA Alertmanager, production routing, and real EKS incident scenarios**.
