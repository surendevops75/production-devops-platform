# 38 — Kibana

> Production-focused DevOps notes for operating Kibana with Elasticsearch in an AWS EKS-based microservices platform.

---

## 1. What Is Kibana?

Kibana is the visualization and exploration interface for Elasticsearch.

In our architecture:

```text
21 Microservices
       │
       ▼
stdout/stderr
       │
       ▼
Fluent Bit
       │
       ▼
Logstash
       │
       ▼
Elasticsearch
       │
       ▼
Kibana
```

Think of the responsibilities as:

```text
Fluent Bit
    ↓
Collect

Logstash
    ↓
Process

Elasticsearch
    ↓
Store + Search

Kibana
    ↓
Explore + Visualize
```

Kibana does **not** replace Elasticsearch.

Kibana depends on Elasticsearch for the underlying data.

---

# 2. Why Do We Need Kibana?

Suppose production has:

```text
21 microservices
300+ Pods
multiple EKS nodes
millions of log events
```

Searching raw node logs manually would be extremely difficult.

Without Kibana:

```text
kubectl logs
grep
awk
tail
```

might help with immediate debugging, but not with centralized historical analysis.

With Kibana:

```text
service.name: "payment-service"
AND
log.level: "ERROR"
```

can search across centralized production logs.

---

# 3. Kibana's Role During an Incident

Suppose users report:

> Checkout is failing.

A DevOps/SRE engineer can use Kibana to investigate:

```text
payment-service
order-service
inventory-service
```

within the same time window.

For example:

```text
14:01:03 order-service → order created
14:01:04 inventory-service → reservation successful
14:01:05 payment-service → gateway timeout
14:01:10 order-service → payment failed
```

Kibana helps reconstruct what happened.

---

# 4. Kibana Is Not a Logging Collector

A common misunderstanding:

> "Kibana collects logs."

No.

The collection path is:

```text
Application
    ↓
Fluent Bit
```

Kibana is at the end:

```text
Elasticsearch
    ↓
Kibana
```

Therefore:

```text
Kibana broken
```

doesn't necessarily mean:

```text
logs lost
```

The data may still exist in Elasticsearch.

---

# 5. Kibana in EKS

Our platform namespace:

```text
logging
│
├── Fluent Bit
├── Logstash
├── Elasticsearch
└── Kibana
```

For production:

```text
Kibana
  ↓
private EKS service
  ↓
internal access
```

Kibana should normally not be directly exposed to the public Internet.

---

# 6. Kibana Deployment Options

There are several ways to operate Kibana:

```text
Option 1
Kibana on EKS using ECK

Option 2
Elastic Cloud

Option 3
another managed Elastic deployment
```

For this project we use:

```text
ECK
+
EKS
+
GitOps
```

because it teaches the Kubernetes operating model.

---

# 7. ECK Architecture

We already introduced ECK for Elasticsearch.

Now the relationship is:

```text
ECK Operator
      │
      ├───────────────┐
      ▼               ▼
Elasticsearch       Kibana
      │               │
      └───────┬───────┘
              ▼
        Logging Platform
```

The Kubernetes desired state is stored in Git.

```text
Git
 ↓
ArgoCD
 ↓
EKS
 ↓
ECK
 ↓
Kibana
```

---

# 8. Conceptual Kibana Resource

A simplified ECK resource looks like:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: logging
  namespace: logging

spec:
  version: "<approved-version>"

  count: 2

  elasticsearchRef:
    name: logging

  podTemplate:
    spec:
      containers:
        - name: kibana
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1"
              memory: "2Gi"
```

The exact fields and supported configuration depend on the ECK and Elastic versions deployed.

---

# 9. Why Two Kibana Replicas?

Suppose:

```text
Kibana-1
Kibana-2
```

If:

```text
Kibana-1
    ↓
node failure
```

then:

```text
Kibana-2
    ↓
continues serving requests
```

This provides availability.

However, Kibana replicas are useful only when traffic is actually routed through a suitable Service/load-balancing layer.

---

# 10. Kibana Service

Kibana normally has a Kubernetes Service.

Conceptually:

```text
Internal ALB
     │
     ▼
Kibana Service
     │
 ┌───┴───┐
 ▼       ▼
Pod 1   Pod 2
```

The Service provides stable discovery.

For example:

```text
kibana.logging.svc.cluster.local
```

can resolve to the Service.

---

# 11. Internal ALB

A common production access pattern is:

```text
Engineer
    │
    ▼
Corporate VPN / Zero Trust
    │
    ▼
Internal ALB
    │
    ▼
Kibana Service
    │
    ▼
Kibana Pods
```

This avoids exposing Kibana directly to the Internet.

---

# 12. Public Kibana Is Usually a Bad Default

Avoid:

```text
Internet
   │
   ▼
Kibana
   │
   ▼
Production logs
```

Production logs may contain:

* customer identifiers
* internal IP addresses
* request information
* infrastructure details
* stack traces
* potentially sensitive information

Therefore access should be restricted.

---

# 13. Authentication

Kibana must authenticate users.

Possible enterprise mechanisms include:

```text
SSO
OIDC
SAML
LDAP
native Elastic authentication
```

The exact mechanism depends on the organization's identity provider and Elastic deployment.

A common enterprise model is:

```text
Engineer
   │
   ▼
Corporate SSO
   │
   ▼
Kibana
   │
   ▼
Elasticsearch
```

---

# 14. Authentication vs Authorization

Authentication:

```text
Who are you?
```

Authorization:

```text
What are you allowed to do?
```

Example:

```text
Developer
  ↓
read application logs

SRE
  ↓
read + investigate operational logs

Security team
  ↓
security/audit logs

Platform admin
  ↓
administrative capabilities
```

Never give every engineer unrestricted access.

---

# 15. Role-Based Access Control

We can define roles such as:

```text
developer-readonly
sre
security-audit
platform-admin
```

Conceptually:

```text
User
 ↓
Role
 ↓
Index/data permissions
```

For example:

```text
developer
   ↓
logs-development-*
```

while:

```text
production-sre
   ↓
logs-production-*
```

---

# 16. Environment Isolation

Our environments are:

```text
DEV
QA
STAGING
PROD
```

Kibana data should be clearly separated.

For example:

```text
logs-dev-*
logs-qa-*
logs-staging-*
logs-production-*
```

Access can then be controlled.

---

# 17. Kibana Data Views

A Data View tells Kibana which Elasticsearch data it should search.

For example:

```text
logs-production-*
```

could represent all production application logs.

Then Kibana can search:

```text
logs-production-order-service-*
logs-production-payment-service-*
logs-production-catalog-service-*
```

depending on the index/data-stream strategy.

---

# 18. Data View Is Not the Data

Important distinction:

```text
Elasticsearch
    │
    └── actual documents

Kibana Data View
    │
    └── tells Kibana where to look
```

Deleting a Data View does not mean deleting the underlying log data.

---

# 19. Time Field

Kibana should know which field represents event time.

For our logs:

```text
@timestamp
```

Example:

```json
{
  "@timestamp": "2026-09-10T12:30:00Z",
  "service.name": "order-service",
  "log.level": "ERROR"
}
```

Kibana uses this field for time-based filtering.

---

# 20. Discover

The **Discover** interface is one of the most important Kibana tools for DevOps.

Conceptually:

```text
Kibana
  ↓
Discover
  ↓
Search logs
```

Example query:

```text
service.name : "order-service"
```

Then:

```text
log.level : "ERROR"
```

Then:

```text
deployment.environment : "production"
```

---

# 21. KQL

Kibana Query Language (KQL) is commonly used to filter Elasticsearch data in Kibana.

Example:

```text
service.name : "payment-service"
```

Multiple conditions:

```text
service.name : "payment-service"
and log.level : "ERROR"
```

Time-based investigation:

```text
service.name : "payment-service"
and log.level : "ERROR"
```

combined with the Kibana time picker.

---

# 22. Search by Order ID

Suppose:

```text
order.id = "ORD-1001"
```

Search:

```text
order.id : "ORD-1001"
```

This can reveal logs across:

```text
order-service
payment-service
inventory-service
shipping-service
notification-service
```

if those services consistently propagate the identifier.

---

# 23. Search by Trace ID

Suppose:

```text
trace.id = "abc123"
```

Search:

```text
trace.id : "abc123"
```

Now logs from multiple services participating in that request can be correlated.

This becomes especially powerful when combined with OpenTelemetry.

---

# 24. Search by Pod

Example:

```text
kubernetes.pod.name : "order-service-7d8c9f"
```

Useful when debugging:

```text
one specific Pod
```

For example:

```text
Pod A → errors
Pod B → normal
Pod C → normal
```

This can indicate:

* bad node
* stale configuration
* bad Pod startup
* connection issue
* localized resource problem

---

# 25. Search by Node

Example:

```text
kubernetes.node.name : "ip-10-0-12-45"
```

This helps investigate node-specific problems.

Suppose:

```text
20 Pods
```

on Node A suddenly show:

```text
network errors
```

while Pods elsewhere are healthy.

Kibana can help identify the pattern.

---

# 26. Search by Namespace

Example:

```text
kubernetes.namespace : "production"
```

or:

```text
kubernetes.namespace : "logging"
```

This helps isolate workloads.

---

# 27. Error Investigation

A useful query pattern:

```text
log.level : "ERROR"
```

Then narrow:

```text
service.name : "order-service"
```

Then:

```text
kubernetes.namespace : "production"
```

Then choose:

```text
last 15 minutes
```

This progressively reduces the search space.

---

# 28. Don't Search "Everything"

A common operational mistake:

```text
all logs
+
last 30 days
```

This can create expensive searches.

Start narrow:

```text
service
+
environment
+
time range
```

Then expand if required.

---

# 29. Time Range Matters

Suppose incident started:

```text
14:05
```

Search:

```text
14:00 → 14:15
```

instead of:

```text
last 30 days
```

This improves:

* query performance
* investigation speed
* signal-to-noise ratio

---

# 30. Saved Searches

A frequently used investigation can be saved.

For example:

```text
Production Errors
```

with:

```text
environment = production
AND
log.level = ERROR
```

This allows engineers to quickly reuse operational searches.

---

# 31. Dashboards

Kibana dashboards combine visualizations and saved searches.

Example:

```text
Production Application Dashboard
```

could contain:

```text
Total logs
ERROR count
WARN count
Top failing services
HTTP status distribution
Top exceptions
Logs by namespace
Logs by Pod
```

---

# 32. Example Dashboard

Conceptually:

```text
┌─────────────────────────────────────────┐
│ Production Logging Overview             │
├────────────┬────────────┬───────────────┤
│ Logs/sec   │ Errors     │ Services      │
├────────────┴────────────┴───────────────┤
│ Error rate over time                    │
├─────────────────────────────────────────┤
│ Top failing services                    │
├──────────────────────┬──────────────────┤
│ Top exceptions       │ Logs by namespace│
└──────────────────────┴──────────────────┘
```

---

# 33. What Should a DevOps Dashboard Show?

A useful production logging dashboard should answer:

```text
What is happening?
Where is it happening?
Which service?
Which environment?
Which node?
Which Pod?
When did it start?
How frequently?
```

---

# 34. Service-Level Dashboard

For `order-service`:

```text
order-service
│
├── total logs
├── ERROR logs
├── WARN logs
├── top exceptions
├── logs by Pod
├── logs by node
└── logs over time
```

This gives quick service visibility.

---

# 35. Production Incident Dashboard

During an incident:

```text
Incident Dashboard
│
├── Error volume
├── affected services
├── top exceptions
├── HTTP 5xx logs
├── database errors
├── timeout errors
├── deployment changes
└── trace IDs
```

This becomes much more useful when combined with Grafana metrics.

---

# 36. Kibana + Grafana

Both can visualize operational information, but they have different strengths.

```text
Grafana
   ↓
metrics-first

Kibana
   ↓
logs/search-first
```

For example:

Grafana:

```text
payment-service error rate = 25%
```

Kibana:

```text
why are the payment requests failing?
```

Together:

```text
Grafana
   ↓
detect anomaly
   ↓
Kibana
   ↓
investigate logs
```

---

# 37. Kibana + Prometheus

Our monitoring architecture:

```text
Application
   │
   ├── metrics ──► Prometheus ──► Grafana
   │
   └── logs ─────► Fluent Bit
                       │
                       ▼
                    Logstash
                       │
                       ▼
                 Elasticsearch
                       │
                       ▼
                     Kibana
```

This separation keeps each tool focused.

---

# 38. Kibana + OpenTelemetry

Later we will introduce:

```text
OpenTelemetry
```

which provides distributed tracing.

Then:

```text
Metrics → Prometheus/Grafana
Logs    → Elasticsearch/Kibana
Traces  → Jaeger
```

If all three carry:

```text
trace.id
```

we can move between signals.

Example:

```text
Kibana
  ↓
trace.id = abc123
  ↓
Jaeger
```

---

# 39. Log Correlation

A production log should ideally contain:

```text
trace.id
span.id
service.name
```

Example:

```json
{
  "@timestamp": "...",
  "service.name": "payment-service",
  "trace.id": "abc123",
  "span.id": "span456",
  "log.level": "ERROR",
  "message": "Payment gateway timeout"
}
```

Now the same request can be investigated across:

```text
logs
+
traces
+
metrics
```

---

# 40. Visualization Types

Kibana supports many visualization approaches.

Common examples:

```text
Line chart
Bar chart
Area chart
Metric
Table
Pie/donut
Maps
Lens visualizations
```

Use the simplest visualization that answers the operational question.

---

# 41. Error Trend

A line chart can show:

```text
ERROR logs
   │
   │       /
   │      /
   │_____/________ time
```

If errors suddenly increase:

```text
normal
   ↓
deployment
   ↓
error spike
```

this gives immediate visual correlation.

---

# 42. Top Failing Services

A bar chart:

```text
payment-service      ███████████
order-service        ███████
inventory-service    ████
shipping-service     ██
catalog-service      █
```

Immediately tells the engineer where to investigate.

---

# 43. Exception Analysis

Suppose:

```text
NullPointerException
ConnectionTimeout
SQLException
OutOfMemoryError
```

A visualization can show their frequency.

This helps identify:

```text
dominant failure mode
```

rather than reading millions of individual events.

---

# 44. Logs per Second

A useful metric:

```text
logs/sec
```

Why?

Because sudden increases can indicate:

```text
traffic spike
error storm
retry storm
logging bug
misconfigured log level
```

For example:

```text
100 logs/sec
      ↓
10,000 logs/sec
```

could overload the entire logging pipeline.

---

# 45. Logging Storm

A logging storm occurs when applications suddenly produce huge numbers of logs.

Example:

```text
Database unavailable
      ↓
Application retries
      ↓
Each retry logs ERROR
      ↓
More retries
      ↓
More logs
```

This creates:

```text
application
   ↓
log storm
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
   ↓
storage/CPU pressure
```

A small application failure can therefore become a logging-platform incident.

---

# 46. Kibana Helps Detect the Storm

Look for:

```text
logs/sec ↑
```

and:

```text
ERROR count ↑
```

and:

```text
specific exception ↑
```

Example:

```text
ConnectionTimeout
10/min
    ↓
500/min
    ↓
20,000/min
```

This is an important production signal.

---

# 47. Alerting

Depending on the Elastic/Kibana version and licensing/deployment model, Kibana can support alerting and rules.

However, our primary infrastructure alerting architecture remains:

```text
Prometheus
    ↓
Alertmanager
```

For infrastructure and application metrics.

Kibana/Elastic alerting can be useful for log-oriented conditions such as:

```text
specific security event
error threshold
pattern detection
```

Use each alerting system deliberately to avoid duplicate notifications.

---

# 48. Avoid Duplicate Alerts

Bad:

```text
Prometheus
   ↓
Payment error alert
   ↓
PagerDuty

Kibana
   ↓
same payment error alert
   ↓
PagerDuty
```

One incident produces:

```text
2 pages
```

This creates alert fatigue.

Define clear ownership.

---

# 49. Kibana Security Logs

Security teams may need queries such as:

```text
authentication failures
authorization failures
suspicious activity
admin actions
```

These logs may require stricter access control than ordinary application logs.

Therefore:

```text
application logs
    ≠
security audit logs
```

even if both ultimately use Elasticsearch.

---

# 50. Audit Logs

Audit logs often require:

```text
longer retention
restricted access
stronger integrity controls
```

For example:

```text
Who changed production configuration?
Who accessed sensitive data?
Who performed an administrative action?
```

Do not treat audit logs as disposable debug logs.

---

# 51. Production Log Retention

Kibana does not determine retention by itself.

Retention is primarily handled by Elasticsearch lifecycle/index/data-stream policies and the storage architecture.

Example:

```text
Application logs
   ↓
7 days

Security logs
   ↓
90 days

Audit logs
   ↓
365 days
```

The exact periods must come from organizational and compliance requirements.

---

# 52. Kibana and Index Lifecycle

Kibana queries whatever data remains available.

Conceptually:

```text
Day 0
 ↓
Elasticsearch
 ↓
Kibana

Day 8
 ↓
application log deleted according to policy
 ↓
Kibana cannot search it
```

Kibana cannot recover data that Elasticsearch has legitimately deleted.

---

# 53. Kibana Resource Sizing

Kibana is generally lighter than Elasticsearch, but dashboards and searches can still consume resources.

Example starting point:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "2Gi"
```

Actual sizing depends on:

```text
number of users
dashboard complexity
query frequency
concurrent searches
visualizations
```

---

# 54. Kibana Is Usually Stateless

Unlike Elasticsearch:

```text
Elasticsearch
   ↓
stores large volumes of data
```

Kibana mainly provides:

```text
UI
saved objects
configuration
```

Its operational model is therefore much closer to a stateless web application.

That is why:

```text
Deployment
```

is generally appropriate.

---

# 55. Saved Objects

Kibana stores things such as:

```text
dashboards
visualizations
searches
data views
alert definitions
```

These are called saved objects.

For production, they should be managed carefully.

---

# 56. Dashboard-as-Code

Don't manually create every production dashboard and then hope nobody deletes it.

A better approach is:

```text
Dashboard definition
       │
       ▼
Git
       │
       ▼
review
       │
       ▼
deployment
```

This gives:

* version control
* review
* repeatability
* rollback

---

# 57. GitOps for Kibana

Our desired flow:

```text
Git
 │
 ▼
Kibana configuration
 │
 ▼
ArgoCD / deployment mechanism
 │
 ▼
EKS
 │
 ▼
Kibana
```

Depending on the specific Elastic/Kibana version and deployment approach, saved-object provisioning can use supported APIs/export-import mechanisms or configuration-as-code patterns.

Do not assume every UI object is automatically represented by a Kubernetes ConfigMap.

---

# 58. Production Changes

Bad:

```text
Engineer
   ↓
changes production dashboard manually
```

Better:

```text
Engineer
   ↓
Git change
   ↓
PR
   ↓
Review
   ↓
Merge
   ↓
deployment
```

Manual UI changes may still be appropriate for temporary investigation, but permanent operational dashboards should be controlled.

---

# 59. Kibana Upgrade

Kibana and Elasticsearch versions must be compatible.

Do not independently upgrade:

```text
Kibana
```

without checking:

```text
Elasticsearch
ECK
Logstash
```

compatibility.

A typical stack upgrade looks like:

```text
Backup/snapshot
      ↓
Compatibility check
      ↓
ECK/operator planning
      ↓
Elasticsearch upgrade
      ↓
Kibana upgrade
      ↓
Logstash compatibility validation
      ↓
Functional testing
```

Follow the supported upgrade path for the exact Elastic versions.

---

# 60. Kibana Startup Failure

Check:

```bash
kubectl get pods -n logging
```

Then:

```bash
kubectl logs -n logging <kibana-pod>
```

Possible causes:

```text
Elasticsearch unavailable
TLS failure
authentication failure
invalid configuration
resource pressure
version incompatibility
```

---

# 61. Kibana Cannot Connect to Elasticsearch

Flow:

```text
Kibana
   │
   ▼
Elasticsearch
```

Check:

```text
DNS
Service
Endpoints
TLS
certificate
credentials
NetworkPolicy
Elasticsearch health
```

Useful commands:

```bash
kubectl get svc -n logging
kubectl get endpoints -n logging
kubectl get pods -n logging
```

---

# 62. Kibana Shows No Data

Don't immediately assume Elasticsearch has no logs.

Check:

```text
1. Correct Data View?
2. Correct time range?
3. Correct @timestamp field?
4. Correct environment?
5. Correct index/data stream?
6. Elasticsearch contains documents?
7. User has permission?
```

A wrong time range is one of the simplest causes.

---

# 63. Example: "No Logs"

Developer says:

> Production logs disappeared.

You check Kibana.

No data.

Instead of restarting everything:

```text
Check Elasticsearch
       ↓
documents exist?
       ↓
YES
       ↓
check Kibana Data View
       ↓
wrong pattern
```

Root cause:

```text
Kibana configuration
```

not:

```text
logging pipeline
```

---

# 64. Elasticsearch Has Data but Kibana Is Slow

Possible causes:

```text
expensive query
large time range
too many visualizations
high-cardinality fields
large aggregations
Elasticsearch resource pressure
too many concurrent users
```

Start by narrowing:

```text
time range
+
service
+
environment
```

---

# 65. Dashboard Performance

A dashboard containing:

```text
20 visualizations
```

with each querying:

```text
30 days
```

can generate a large workload.

Better:

```text
default time range = 15 minutes
```

and allow users to expand when necessary.

---

# 66. High-Cardinality Fields

Fields such as:

```text
request_id
trace_id
session_id
user_id
```

can have extremely high cardinality.

They are useful for exact searches but may be poor choices for certain aggregations.

For example:

```text
Top 100,000 trace IDs
```

is usually not a useful dashboard.

---

# 67. Kibana Query Optimization

Prefer:

```text
service.name : "payment-service"
AND
log.level : "ERROR"
```

over:

```text
search everything
```

Also avoid unnecessary:

```text
huge time ranges
large aggregations
wildcard-heavy searches
```

---

# 68. Production Access Model

A secure architecture:

```text
Engineer
   │
   ▼
Corporate Identity
   │
   ▼
VPN / Zero Trust
   │
   ▼
Internal ALB
   │
   ▼
Kibana Service
   │
   ▼
Kibana Pods
   │
   ▼
Elasticsearch
```

This creates multiple security layers.

---

# 69. NetworkPolicy Model

Conceptually:

```text
Internal ALB
      │
      ▼
Kibana
      │
      ▼
Elasticsearch
```

While:

```text
Random application Pod
       X
       │
       ▼
Elasticsearch
```

and:

```text
Internet
   X
   │
   ▼
Elasticsearch
```

Network policies and AWS network controls should reinforce this architecture.

---

# 70. Incident Example — OOMKilled

Suppose:

```text
order-service
   ↓
OOMKilled
```

Kubernetes:

```text
kubectl describe pod
```

shows:

```text
Reason: OOMKilled
```

Now use Kibana:

```text
service.name : "order-service"
AND
log.level : "ERROR"
```

Search around the incident.

You may find:

```text
large response
memory allocation
GC warnings
```

Then correlate with Prometheus:

```text
container memory ↑
```

This demonstrates the observability workflow:

```text
Kubernetes
+
Kibana
+
Prometheus
```

---

# 71. Incident Example — Database Failure

Suppose:

```text
RDS connection failures
```

Kibana query:

```text
service.name : "order-service"
AND
message : "*connection*"
```

Find:

```text
connection timeout
```

Then Grafana:

```text
RDS DatabaseConnections ↑
```

Then:

```text
RDS CPU
RDS memory
RDS latency
```

Now you can correlate application symptoms with infrastructure metrics.

---

# 72. Incident Example — Deployment Problem

Suppose:

```text
payment-service
```

was deployed at:

```text
14:00
```

At:

```text
14:03
```

errors increase.

Kibana:

```text
service.name : "payment-service"
AND
log.level : "ERROR"
```

shows:

```text
configuration missing
```

ArgoCD:

```text
deployment revision changed
```

Git:

```text
configuration commit
```

Now the chain becomes:

```text
Git commit
   ↓
GitOps
   ↓
ArgoCD
   ↓
Deployment
   ↓
Application error
   ↓
Kibana
```

This is exactly why we built the entire platform with GitOps and observability.

---

# 73. Kibana and ArgoCD

ArgoCD tells us:

```text
What was deployed?
```

Kibana tells us:

```text
What did the application log after deployment?
```

Together:

```text
ArgoCD
   +
Kibana
```

can dramatically reduce incident investigation time.

---

# 74. Kibana and Kubernetes

Kibana can show:

```text
namespace
pod
container
node
deployment
```

This allows you to move from:

```text
application failure
```

to:

```text
specific Pod
```

and then use:

```bash
kubectl describe pod
```

or:

```bash
kubectl logs
```

for deeper investigation.

---

# 75. Kibana vs kubectl logs

`kubectl logs`:

```text
best for
recent/live Pod-specific investigation
```

Kibana:

```text
best for
centralized historical multi-service investigation
```

Use both.

Example:

```text
Kibana
 ↓
identify problematic Pod
 ↓
kubectl logs
 ↓
inspect live/recent container output
```

---

# 76. Kibana vs CloudWatch Logs

Because we're on AWS, another question is:

> Why not simply use CloudWatch Logs?

You can.

CloudWatch is excellent for AWS-native infrastructure and managed-service logging.

Our project uses Elastic Stack to teach:

```text
Fluent Bit
Logstash
Elasticsearch
Kibana
```

This gives deeper hands-on experience with:

* log pipelines
* parsing
* indexing
* search
* mappings
* dashboards
* lifecycle
* self-managed observability

In a real company, the decision should be based on:

* operational burden
* cost
* search requirements
* team expertise
* compliance
* AWS integration
* scale

---

# 77. Production Tool Placement

Our final logging placement:

| Component        | Where                 |
| ---------------- | --------------------- |
| Application logs | stdout/stderr         |
| Fluent Bit       | EKS DaemonSet         |
| Logstash         | EKS logging namespace |
| Elasticsearch    | EKS logging platform  |
| Kibana           | EKS logging platform  |
| Prometheus       | EKS monitoring        |
| Grafana          | EKS monitoring        |
| Alertmanager     | EKS monitoring        |
| OpenTelemetry    | EKS/platform          |
| Jaeger           | EKS/platform          |

---

# 78. Production Checklist

Before production:

```text
[ ] Kibana deployed through GitOps
[ ] Version pinned
[ ] Elasticsearch version compatibility verified
[ ] Multiple replicas where required
[ ] Internal-only access
[ ] TLS enabled
[ ] Authentication configured
[ ] RBAC configured
[ ] NetworkPolicy configured
[ ] Resource requests configured
[ ] Resource limits configured
[ ] Data Views standardized
[ ] @timestamp configured
[ ] Dashboards created
[ ] Operational searches created
[ ] Dashboard ownership defined
[ ] Saved-object strategy defined
[ ] Production log retention defined
[ ] Security/audit access restricted
[ ] Monitoring configured
[ ] Incident runbooks available
```

---

# 79. Full Logging Architecture

We now have the complete Elastic logging pipeline:

```text
                         EKS
                          │
              ┌───────────┴───────────┐
              │                       │
       Application Pods          Platform Pods
              │
              ▼
        stdout / stderr
              │
              ▼
       Container log files
              │
              ▼
    ┌─────────────────────┐
    │ Fluent Bit           │
    │ DaemonSet             │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │ Logstash              │
    │ Deployment             │
    │                       │
    │ Parse                 │
    │ Filter                │
    │ Enrich                │
    │ Normalize             │
    │ Route                 │
    └──────────┬──────────┘
               │
               ▼
    ┌──────────────────────────┐
    │ Elasticsearch Cluster    │
    │                          │
    │ Node 1 → AZ-A → EBS      │
    │ Node 2 → AZ-B → EBS      │
    │ Node 3 → AZ-C → EBS      │
    └────────────┬─────────────┘
                 │
                 ▼
           ┌──────────┐
           │  Kibana  │
           └────┬─────┘
                │
                ▼
        Engineers / SRE
```

---

# 80. Complete Observability Architecture

Adding the other observability signals:

```text
                         EKS
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       Metrics           Logs            Traces
          │               │                │
          ▼               ▼                ▼
    Prometheus         Fluent Bit      OpenTelemetry
          │               │                │
          ▼               ▼                ▼
       Grafana          Logstash          Jaeger
          │               │                │
          │               ▼                │
          │         Elasticsearch           │
          │               │                │
          │               ▼                │
          │             Kibana             │
          │                                │
          └───────────────┬────────────────┘
                          ▼
                    Incident Response
```

The three signals answer:

```text
Metrics → What is happening?

Logs → What happened?

Traces → Where did it happen?
```

---

# 81. Final Mental Model

When a production incident occurs:

```text
ALERT
  ↓
Prometheus / Alertmanager
  ↓
Grafana
  ↓
Identify affected service
  ↓
Kibana
  ↓
Search logs
  ↓
Trace ID
  ↓
Jaeger
  ↓
Find slow/failing dependency
  ↓
Kubernetes / AWS
  ↓
Fix root cause
```

For example:

```text
Alert:
payment error rate > 5%

        ↓

Grafana:
payment-service errors increased

        ↓

Kibana:
Payment gateway timeout

        ↓

Trace:
payment-service → gateway = 4.9 sec

        ↓

AWS/network/dependency investigation

        ↓

Root cause
```

That is the production observability workflow we are building.

---

# 82. Chapter Summary

The complete Elastic stack we've built is:

```text
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

with:

```text
EKS
+
ECK
+
Helm
+
GitOps
+
ArgoCD
```

and:

```text
Prometheus
+
Grafana
+
Alertmanager
```

for metrics and alerting.

The most important distinction to remember:

> **Kibana is the investigation interface. Elasticsearch is the searchable data platform underneath it.**

And during a real incident:

> **Don't start by randomly restarting Pods. Start with the signal, narrow the time range, identify the affected service, search the logs, correlate the trace, and then investigate the underlying Kubernetes/AWS dependency.**

---

## Next

**`39-OpenTelemetry.md`**

The next chapter moves from logs to **distributed tracing** and covers how requests travel through our 21 microservices:

```text
User
 ↓
ALB
 ↓
order-service
 ↓
inventory-service
 ↓
payment-service
 ↓
fraud-service
 ↓
notification-service
```

including **OpenTelemetry SDK/Collector architecture, instrumentation, trace/span/context propagation, sampling, OTLP, Kubernetes deployment, Spring Boot integration, production configuration, and how traces connect Kibana logs with Jaeger.**
