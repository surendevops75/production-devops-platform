# 121 — ELK Production Implementation

> **Goal:** Build a production-grade centralized logging platform for the 21-microservice E-Commerce Platform running on Amazon EKS, using **Fluent Bit → Logstash → Elasticsearch → Kibana**.

---

# 1. Objective

We already have:

```text
Metrics
   ↓
Prometheus
   ↓
Grafana
```

and:

```text
Alerts
   ↓
Alertmanager
```

Now we build centralized logging:

```text
                    EKS
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   order-service payment-service inventory-service
       │             │             │
       └─────────────┼─────────────┘
                     │
                     ▼
                Container Logs
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

This gives engineers the ability to answer:

```text
What happened?

Which service generated the error?

Which pod generated it?

Which node was running the pod?

Which request was affected?

What happened immediately before the error?

Was this related to a deployment?

Did another service fail first?
```

---

# 2. Why Centralized Logging?

Without centralized logging:

```text
Engineer
   │
   ├── kubectl logs pod-1
   ├── kubectl logs pod-2
   ├── kubectl logs pod-3
   ├── kubectl logs pod-4
   └── ...
```

With 21 services and many replicas, this becomes difficult.

Instead:

```text
All application logs
       ↓
Central logging
       ↓
Search
       ↓
Filter
       ↓
Correlate
       ↓
Investigate
```

---

# 3. ELK Components

ELK traditionally means:

```text
E = Elasticsearch
L = Logstash
K = Kibana
```

We add Fluent Bit:

```text
Fluent Bit
     ↓
Logstash
     ↓
Elasticsearch
     ↓
Kibana
```

Each component has a different responsibility.

| Component     | Responsibility                  |
| ------------- | ------------------------------- |
| Fluent Bit    | Collect and forward logs        |
| Logstash      | Parse, transform, enrich, route |
| Elasticsearch | Store and index logs            |
| Kibana        | Search, visualize, investigate  |

---

# 4. Why Fluent Bit?

Fluent Bit is lightweight and is well suited to running on every Kubernetes node.

We deploy it as:

```text
DaemonSet
```

Therefore:

```text
Node 1 → Fluent Bit
Node 2 → Fluent Bit
Node 3 → Fluent Bit
Node 4 → Fluent Bit
```

When a new node joins:

```text
New Node
   ↓
DaemonSet
   ↓
Fluent Bit automatically scheduled
```

This is exactly what we want.

---

# 5. Why Not Run One Fluent Bit Pod?

Bad architecture:

```text
                 ┌───────────────┐
Node 1 ─────────►│               │
Node 2 ─────────►│ Fluent Bit    │
Node 3 ─────────►│ Single Pod    │
Node 4 ─────────►│               │
                 └───────────────┘
```

If that pod fails:

```text
All log collection stops.
```

Preferred:

```text
Node 1 → Fluent Bit
Node 2 → Fluent Bit
Node 3 → Fluent Bit
Node 4 → Fluent Bit
```

Each node collects logs locally.

---

# 6. Kubernetes Container Logging

Most containers should write application logs to:

```text
stdout
stderr
```

For example:

```java
log.info("Order created successfully");
log.error("Payment processing failed");
```

The container runtime captures those streams.

Kubernetes exposes container logs through the node's logging infrastructure.

Depending on the runtime/node image, the underlying files and paths can vary, so don't hard-code assumptions without checking the EKS node configuration.

A common Kubernetes log location is:

```text
/var/log/containers/
```

which contains links to pod/container log files.

---

# 7. Logging Flow

For our EKS platform:

```text
Spring Boot Application
        │
        ▼
stdout / stderr
        │
        ▼
Container Runtime
        │
        ▼
Node log files
        │
        ▼
Fluent Bit
        │
        ▼
Logstash
```

This means application containers don't need to know where Elasticsearch lives.

---

# 8. Application Logging

The first requirement is:

> Applications should produce structured logs.

Avoid:

```text
2026-09-29 ERROR Payment failed
```

Prefer JSON:

```json
{
  "timestamp": "2026-09-29T10:30:12.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "message": "Payment processing failed",
  "orderId": "ORD-123",
  "correlationId": "abc-123"
}
```

However, sensitive identifiers should not be included merely because they are convenient.

---

# 9. Spring Boot Structured Logging

For production applications, use structured JSON logging rather than manually concatenating strings.

Conceptually:

```json
{
  "timestamp": "...",
  "level": "INFO",
  "service": "order-service",
  "environment": "prod",
  "message": "Order created",
  "correlationId": "..."
}
```

The exact Spring Boot logging configuration depends on the Spring Boot version and logging implementation.

The important design principle is:

```text
Machine-readable logs
```

rather than:

```text
Human-formatted text only
```

---

# 10. Log Levels

Use:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

Production default:

```text
INFO
```

Potentially:

```text
payment-service = INFO
```

and temporarily increase logging for a specific troubleshooting scenario if operationally safe.

Avoid permanently running production at:

```text
DEBUG
```

because logging volume can increase dramatically.

---

# 11. Logging Categories

Typical logs:

### INFO

```text
Order created
Payment authorized
Shipment created
```

### WARN

```text
Retrying dependency call
Cache unavailable, using fallback
Queue processing delayed
```

### ERROR

```text
Database connection failure
Payment provider timeout
Unhandled exception
```

Do not log every normal event as `ERROR`.

---

# 12. Never Log Secrets

Never put these into application logs:

```text
Passwords
AWS access keys
AWS secret keys
JWT secrets
Database passwords
API keys
Credit-card data
Payment authentication data
Kubernetes tokens
Secrets Manager values
```

Bad:

```java
log.info("Database password={}", password);
```

Never do this.

---

# 13. PII Protection

Be careful with:

```text
Email
Phone
Address
Payment information
Government identifiers
Customer identifiers
```

Whether these can be logged depends on business, legal, and organizational requirements.

A good default is:

```text
Log only what is required for operations.
```

Instead of:

```json
{
  "email": "customer@example.com"
}
```

use an internal identifier where appropriate:

```json
{
  "customerId": "internal-reference"
}
```

Even internal identifiers should be handled according to company policy.

---

# 14. Fluent Bit Architecture

Deploy:

```text
DaemonSet
```

in:

```text
logging
```

namespace.

Example:

```bash
kubectl create namespace logging
```

In production, create this through GitOps.

---

# 15. Fluent Bit Repository

Example:

```text
gitops/
└── infrastructure/
    └── logging/
        ├── namespace.yaml
        ├── fluent-bit/
        │   ├── values.yaml
        │   └── application.yaml
        ├── logstash/
        │   ├── values.yaml
        │   └── application.yaml
        ├── elasticsearch/
        │   ├── values.yaml
        │   └── application.yaml
        └── kibana/
            ├── values.yaml
            └── application.yaml
```

---

# 16. Fluent Bit Inputs

Conceptually:

```text
INPUT
  ↓
Tail container log files
  ↓
Parser
  ↓
Kubernetes metadata
  ↓
Filter
  ↓
Output
```

A typical input uses the `tail` plugin.

Conceptually:

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Tag               kube.*
    Parser            cri
    DB                /var/log/flb_kubernetes.db
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
```

The exact parser depends on the container runtime/log format used by the EKS node.

---

# 17. Why Fluent Bit Needs a Position Database

Suppose Fluent Bit reads:

```text
log-file.log
```

and has processed:

```text
line 1 → 10000
```

Then Fluent Bit restarts.

Without tracking its position, it may:

```text
re-read everything
```

or lose track of what was already processed.

The position database records:

```text
file
offset
```

so collection can resume appropriately.

---

# 18. Kubernetes Metadata

Fluent Bit should enrich records with Kubernetes metadata.

For example:

```json
{
  "kubernetes": {
    "namespace_name": "ecommerce",
    "pod_name": "order-service-7d8c7",
    "container_name": "order-service",
    "container_id": "..."
  }
}
```

This is extremely valuable.

You can search:

```text
namespace=ecommerce
```

or:

```text
pod=order-service-7d8c7
```

---

# 19. Metadata Flow

```text
Application Log
      │
      ▼
Fluent Bit
      │
      ├── namespace
      ├── pod
      ├── container
      ├── node
      └── labels
      │
      ▼
Logstash
```

Now Kibana can answer:

```text
Which namespace generated this error?

Which pod?

Which container?

Which service?
```

---

# 20. Fluent Bit Filters

Typical processing:

```text
Tail
 ↓
CRI parser
 ↓
Kubernetes metadata
 ↓
JSON parser
 ↓
Modify fields
 ↓
Remove unnecessary fields
 ↓
Forward
```

Avoid performing expensive transformations at every node unless needed.

---

# 21. Fluent Bit Output

Instead of sending directly to Elasticsearch:

```text
Fluent Bit → Elasticsearch
```

we are intentionally using:

```text
Fluent Bit → Logstash → Elasticsearch
```

Why?

Because Logstash gives us a centralized transformation/routing layer.

---

# 22. Why Use Logstash?

Logstash can:

```text
Parse
Transform
Enrich
Route
Normalize
Drop unwanted logs
Create derived fields
```

Example:

```text
Fluent Bit
     ↓
Logstash
     │
     ├── application logs → Elasticsearch
     ├── security logs → security index
     ├── audit logs → audit index
     └── unwanted/noisy logs → drop
```

This separation is useful in larger environments.

---

# 23. Logstash Deployment

Logstash is not a DaemonSet in our architecture.

We can run it as:

```text
Deployment
```

with multiple replicas.

Example:

```text
                 Logstash Service
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
      Logstash Pod 1        Logstash Pod 2
```

This provides horizontal processing capacity.

---

# 24. Logstash Namespace

```text
logging
```

Example:

```text
logging/
├── fluent-bit
├── logstash
├── elasticsearch
└── kibana
```

Network policies should restrict communication.

---

# 25. Logstash Pipeline

A basic pipeline:

```text
input
  ↓
filter
  ↓
output
```

Example:

```conf
input {
  forward {
    port => 24224
  }
}

filter {
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "ecommerce-logs-%{+YYYY.MM.dd}"
  }
}
```

This is a simplified learning example.

Production Elasticsearch security and index strategy should not be left at this basic configuration.

---

# 26. JSON Parsing

Suppose the application sends:

```text
{"level":"ERROR","service":"order-service","message":"DB timeout"}
```

Logstash can parse:

```text
message
```

into structured fields:

```text
level = ERROR
service = order-service
message = DB timeout
```

Now Kibana can search:

```text
level:ERROR
```

instead of:

```text
message:"ERROR"
```

---

# 27. Standard Log Schema

Create a common schema across all services.

Recommended fields:

```text
@timestamp
level
message
service
environment
namespace
pod
container
node
version
correlationId
traceId
spanId
httpMethod
httpRoute
httpStatus
durationMs
```

Not every log requires every field.

But common naming is important.

---

# 28. Example Production Log

```json
{
  "@timestamp": "2026-09-29T10:30:12.123Z",
  "level": "ERROR",
  "service": "order-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "order-service-7d8c7",
  "container": "order-service",
  "version": "1.5.2",
  "correlationId": "abc-123",
  "httpMethod": "POST",
  "httpRoute": "/api/v1/orders",
  "httpStatus": 500,
  "durationMs": 1432,
  "message": "Database operation failed"
}
```

This is highly searchable.

---

# 29. Correlation ID

This is one of the most important fields in distributed systems.

Suppose:

```text
Customer
   ↓
ALB
   ↓
order-service
   ↓
payment-service
   ↓
inventory-service
   ↓
notification-service
```

We generate:

```text
correlationId = abc-123
```

The same identifier travels across the request chain where appropriate.

Then Kibana can search:

```text
correlationId:"abc-123"
```

and show the related logs.

---

# 30. Correlation ID vs Trace ID

They are related but not identical.

### Correlation ID

Used to associate business/request activity.

```text
abc-123
```

### Trace ID

Identifies a distributed trace.

```text
4bf92f3577b34da6...
```

Later, distributed tracing will provide:

```text
trace
 ├── span
 ├── span
 ├── span
 └── span
```

Logs should eventually include trace identifiers so we can navigate:

```text
Metric
 ↓
Trace
 ↓
Log
```

---

# 31. Elasticsearch Architecture

For production, avoid thinking of Elasticsearch as:

```text
one Pod
```

Instead:

```text
                 Elasticsearch Cluster
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Node 1          Node 2         Node 3
```

Depending on workload, separate roles may be appropriate.

Examples include:

```text
Cluster-manager eligible
Data
Ingest
Coordinating
```

Modern Elasticsearch terminology and deployment architecture should follow the version being used.

---

# 32. Elasticsearch on EKS

There are two broad choices:

### Option A — Run Elasticsearch in EKS

```text
EKS
 │
 └── Elasticsearch
      ├── StatefulSet/operator
      ├── EBS volumes
      └── Kubernetes management
```

### Option B — Use managed Elasticsearch-compatible service

For AWS, Amazon OpenSearch Service is a managed alternative.

Architecture:

```text
EKS
 │
 └── Fluent Bit
       ↓
     Logstash
       ↓
Amazon OpenSearch Service
       ↓
     Kibana/OpenSearch Dashboards
```

For a hands-on Kubernetes project, running the stack inside EKS is useful for learning the operational model. For production, evaluate managed OpenSearch against self-managed Elasticsearch based on operational and organizational requirements.

---

# 33. Elasticsearch Stateful Storage

Logs require persistence.

Do not use:

```text
emptyDir
```

for production log data.

Use persistent storage.

For EKS:

```text
Elasticsearch
     ↓
PVC
     ↓
EBS
```

Example:

```yaml
resources:
  requests:
    storage: 200Gi
```

Actual sizing depends on:

```text
daily log volume
retention
replication
compression
index overhead
headroom
recovery requirements
```

---

# 34. Log Volume Calculation

Suppose:

```text
10 GB/day
```

and retention:

```text
14 days
```

Raw storage:

```text
10 × 14
= 140 GB
```

But you should not provision exactly 140 GB.

Consider:

```text
replicas
index overhead
segment merges
temporary space
growth
recovery
headroom
```

For example:

```text
140 GB raw
+
replication
+
overhead
+
headroom
```

The actual storage model must be calculated from measured production ingestion.

---

# 35. Shards

Elasticsearch indexes are divided into shards.

Conceptually:

```text
Index
│
├── Primary shard 1
├── Primary shard 2
├── Primary shard 3
│
├── Replica shard 1
├── Replica shard 2
└── Replica shard 3
```

Too many shards can cause overhead.

Too few shards can limit scale.

Therefore:

> Do not choose shard counts arbitrarily from a generic tutorial.

Base them on:

```text
index size
ingestion rate
query patterns
node resources
retention
recovery requirements
```

---

# 36. Daily Index Strategy

A simple strategy:

```text
ecommerce-logs-2026.09.29
ecommerce-logs-2026.09.30
```

But modern Elasticsearch/OpenSearch deployments may use data streams and lifecycle policies instead.

For a new production implementation, evaluate:

```text
Data Streams
+
Index Lifecycle Management
```

rather than automatically creating unlimited daily indexes.

---

# 37. Log Retention

Logs should not be kept forever.

Example:

```text
Application logs: 14 days
Security logs:    90 days
Audit logs:       1 year
```

These are examples only.

Actual retention depends on:

```text
Compliance
Security
Business requirements
Storage cost
Incident investigation requirements
```

---

# 38. Index Lifecycle

A lifecycle could look like:

```text
Hot
 │
 │ recent logs
 ▼
Warm
 │
 │ less frequently queried
 ▼
Cold
 │
 ▼
Delete
```

Example:

```text
Day 0-7
   Hot

Day 8-30
   Warm

Day 31+
   Delete/archive
```

The exact policy should be based on actual operational requirements.

---

# 39. Kibana

Kibana is the investigation interface.

Engineers can:

```text
Search logs
Filter fields
Create visualizations
Create dashboards
Investigate errors
Analyze patterns
```

Typical access:

```text
Engineer
   ↓
VPN / identity layer
   ↓
Kibana
   ↓
Elasticsearch
```

Don't expose Elasticsearch directly to the public Internet.

---

# 40. Kibana Dashboard

Create dashboards:

```text
Logging
│
├── Application Errors
├── HTTP Requests
├── Order Service
├── Payment Service
├── Inventory Service
├── Kubernetes
├── Security
└── Incident Investigation
```

---

# 41. Error Dashboard

Useful panels:

```text
Errors/min
Errors by service
Errors by namespace
Errors by HTTP status
Errors by endpoint
Top error messages
Top exception classes
```

Example:

```text
ERRORS
│
│              /\
│             /  \
│            /    \
│___________/      \____
        time
```

---

# 42. Searching Errors

Example Kibana query:

```text
level: "ERROR"
```

Then filter:

```text
service: "order-service"
```

Then:

```text
environment: "prod"
```

Then:

```text
namespace: "ecommerce"
```

This progressively narrows the investigation.

---

# 43. Searching by Correlation ID

Suppose customer support gives:

```text
correlationId = abc-123
```

Search:

```text
correlationId:"abc-123"
```

You may discover:

```text
10:20:01 order-service
10:20:02 payment-service
10:20:04 inventory-service
10:20:05 notification-service
```

Now you can reconstruct the request journey from logs.

---

# 44. Searching Java Exceptions

A common production search:

```text
level:"ERROR" AND service:"order-service"
```

Then search for:

```text
exception
```

Potential result:

```text
org.postgresql.util.PSQLException
```

Then:

```text
service = order-service
exception = PSQLException
```

Now check RDS dashboards.

This is where observability starts connecting together.

---

# 45. Multiline Java Stack Traces

Java exceptions often contain multiple lines:

```text
java.sql.SQLException
    at ...
    at ...
    at ...
    at ...
```

If multiline handling is incorrect, Elasticsearch may receive:

```text
Log 1
Log 2
Log 3
Log 4
```

instead of one exception event.

Therefore, configure multiline parsing according to the actual container runtime and application logging format.

Better yet, structured exception logging can reduce parsing ambiguity.

---

# 46. Fluent Bit Buffering

What happens if Logstash is temporarily unavailable?

We don't want:

```text
Application
 ↓
Fluent Bit
 ↓
Logstash unavailable
 ↓
logs immediately disappear
```

Configure buffering appropriately.

Possible architecture:

```text
Fluent Bit
   │
   ├── memory buffer
   └── filesystem buffer
           │
           ▼
       Logstash
```

Filesystem buffering provides more resilience against temporary downstream outages.

The exact settings must be sized for expected log volume and available node disk.

---

# 47. Backpressure

Imagine:

```text
Applications
   │
   │ 100 MB/s
   ▼
Fluent Bit
   │
   │ 100 MB/s
   ▼
Logstash
   │
   │ 50 MB/s
   ▼
Elasticsearch
```

Now:

```text
Incoming > Processing
```

Backlog increases.

This is:

```text
backpressure
```

We must monitor it.

---

# 48. What Happens During Elasticsearch Failure?

Suppose:

```text
Elasticsearch
     ↓
Unavailable
```

Expected architecture:

```text
Applications
     ↓
Fluent Bit
     ↓
Buffer
     ↓
Logstash
     ↓
Elasticsearch
```

During a short outage:

```text
Fluent Bit buffering
+
Logstash buffering
```

can absorb some disruption.

But buffers are finite.

Therefore:

> Logging systems themselves require capacity planning and failure testing.

---

# 49. Logging Is Not Allowed to Break Applications

A critical principle:

```text
Application
     │
     └── logging
```

should not become:

```text
Application
     ↓
Logging failure
     ↓
Application failure
```

Logging should be treated as an asynchronous operational dependency.

A logging outage should not normally cause order processing to stop.

---

# 50. Log Sampling

High-volume services may produce enormous amounts of logs.

For example:

```text
search-service
100,000 requests/sec
```

If every request generates several log lines:

```text
millions of logs
```

per minute can be generated.

Consider reducing unnecessary logs.

Examples:

```text
INFO → business events
DEBUG → disabled in production
TRACE → disabled
```

Do not simply delete error logs to reduce volume.

---

# 51. Access Logs

HTTP access logs can be useful:

```json
{
  "method": "POST",
  "route": "/api/v1/orders",
  "status": 201,
  "durationMs": 120
}
```

But if Prometheus already provides:

```text
request rate
error rate
latency
```

you don't necessarily need extremely verbose access logs for every request.

Choose based on operational needs and volume.

---

# 52. Kubernetes Audit Logs

Kubernetes audit logging is a different category.

Application logs:

```text
order-service
```

are not the same as:

```text
Kubernetes API audit logs
```

Audit events can answer:

```text
Who changed this resource?

What API operation occurred?

When?

From where?

Which object?
```

Security teams may have separate retention requirements for these logs.

---

# 53. Security Logging

Security-related logs may include:

```text
Authentication failures
Authorization failures
IAM events
Kubernetes audit events
Security scan results
Admission decisions
Network security events
```

Do not mix all security logs blindly with application logs.

You may route them to separate indexes/data streams with different access and retention policies.

---

# 54. NetworkPolicy

Logging components should have restricted communication.

Conceptually:

```text
Fluent Bit
    ↓
allowed → Logstash

Logstash
    ↓
allowed → Elasticsearch

Kibana
    ↓
allowed → Elasticsearch
```

Don't allow:

```text
Any namespace
     ↓
Elasticsearch
```

unless there is a documented reason.

---

# 55. Elasticsearch Security

Production Elasticsearch/OpenSearch should have:

```text
Authentication
Authorization
TLS
Encryption
Network restrictions
Audit controls where required
```

Avoid:

```text
http://elasticsearch:9200
```

as the final production security model.

That may be acceptable as a local learning example but not as the production target.

---

# 56. Secrets

Credentials should not be stored directly in:

```text
Git
Helm values
Docker images
Application source code
```

Use the project's existing secrets architecture:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
Logstash / Kibana / application
```

Use IAM through:

```text
EKS Pod Identity
```

where appropriate.

---

# 57. Elasticsearch Credentials

For example:

```text
Logstash
   │
   ▼
Secret
   │
   ├── username
   └── password
```

The secret is mounted/injected securely.

Don't write:

```yaml
password: myproductionpassword
```

into Git.

---

# 58. Logging Repository

Our complete GitOps structure becomes:

```text
gitops/
│
├── infrastructure/
│   │
│   ├── monitoring/
│   │   ├── prometheus/
│   │   ├── grafana/
│   │   └── alertmanager/
│   │
│   └── logging/
│       ├── fluent-bit/
│       ├── logstash/
│       ├── elasticsearch/
│       └── kibana/
│
└── applications/
```

---

# 59. Fluent Bit → Logstash Flow

Example:

```text
Node
│
├── order-service logs
├── payment-service logs
├── inventory-service logs
└── notification-service logs
        │
        ▼
   Fluent Bit
        │
        │ Kubernetes metadata
        ▼
    Logstash
        │
        ├── parse JSON
        ├── normalize fields
        ├── enrich
        ├── filter
        └── route
        │
        ▼
 Elasticsearch
        │
        ▼
     Kibana
```

---

# 60. Production Incident Example

Suppose customers report:

> Orders are failing.

Grafana shows:

```text
order-service
5xx ↑
P95 ↑
```

Engineer opens Kibana.

Search:

```text
service:"order-service"
AND level:"ERROR"
```

Finds:

```text
PSQLException
connection refused
```

Then:

```text
Kibana
   ↓
RDS error logs
```

Grafana:

```text
RDS connections ↑
RDS CPU ↑
```

Then Kubernetes:

```text
order-service replicas
   ↓
10 → 25
```

HPA scaled the service.

Connection capacity was not sufficient.

The complete chain:

```text
Traffic spike
   ↓
HPA
   ↓
More pods
   ↓
More DB connections
   ↓
RDS saturation
   ↓
Order errors
```

This is the type of production investigation the observability platform should enable.

---

# 61. Another Incident — Bad Deployment

Deployment:

```text
order-service v1.6.0
```

After deployment:

```text
5xx ↑
```

Kibana:

```text
ERROR
NoSuchMethodError
```

Engineer correlates:

```text
Deployment annotation
        ↓
Grafana error graph
        ↓
Kibana exception
        ↓
Git commit
```

Then rollback.

---

# 62. Another Incident — Missing Logs

Suppose:

```text
payment-service
```

is running but Kibana has no logs.

Investigate:

```text
Pod
 ↓
stdout/stderr
 ↓
Node log file
 ↓
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
 ↓
Kibana
```

Check each layer.

### 1. Application

```bash
kubectl logs <pod> -n ecommerce
```

If no logs:

```text
Application logging problem.
```

### 2. Fluent Bit

```bash
kubectl get pods -n logging
```

### 3. Logstash

Check:

```text
pipeline
input
filter
output
errors
```

### 4. Elasticsearch

Check:

```text
index/data stream
ingestion
disk
cluster health
```

### 5. Kibana

Check:

```text
data view
time range
filters
permissions
```

---

# 63. Another Incident — Log Volume Explosion

Suppose a developer accidentally changes:

```text
INFO
```

to:

```text
DEBUG
```

for production.

Log volume:

```text
10 GB/day
    ↓
200 GB/day
```

Consequences:

```text
Fluent Bit CPU ↑
Network ↑
Logstash CPU ↑
Elasticsearch storage ↑
Query performance ↓
Cost ↑
```

Monitoring should therefore include the logging platform itself.

---

# 64. Logging Platform Metrics

Monitor:

### Fluent Bit

```text
CPU
Memory
Records processed
Records dropped
Buffer usage
Output errors
Retry count
```

### Logstash

```text
CPU
Memory
Pipeline events
Queue depth
Events in
Events out
Failed events
Persistent queue usage
```

### Elasticsearch

```text
Cluster health
CPU
Heap
JVM pressure
Disk
Indexing rate
Search rate
Rejected requests
Shard health
```

### Kibana

```text
CPU
Memory
Availability
Request latency
```

---

# 65. Logging Dashboard

Create a Grafana dashboard:

```text
Logging Platform
│
├── Fluent Bit
│   ├── CPU
│   ├── Memory
│   ├── Records
│   ├── Errors
│   └── Buffer
│
├── Logstash
│   ├── Events In
│   ├── Events Out
│   ├── Queue
│   └── Errors
│
├── Elasticsearch
│   ├── Cluster Health
│   ├── JVM
│   ├── Disk
│   ├── Indexing
│   └── Search
│
└── Kibana
    ├── Availability
    └── Resource Usage
```

---

# 66. Logging Alerts

Add alerts such as:

```text
FluentBitOutputErrors
FluentBitBufferHigh
LogstashPipelineBacklog
LogstashOutputErrors
ElasticsearchClusterUnhealthy
ElasticsearchDiskHigh
ElasticsearchJVMPressure
ElasticsearchIndexingFailures
KibanaUnavailable
```

These alerts protect the observability system itself.

---

# 67. Important Principle

> **You need monitoring for your monitoring system.**

If Elasticsearch is full and nobody knows:

```text
Application continues
      ↓
Logs disappear
      ↓
Incident happens
      ↓
No logs available
```

This can make an incident much harder to resolve.

Therefore:

```text
Observability Platform
       ↓
must itself be observable
```

---

# 68. Log Retention and Cost

Logging can become one of the largest infrastructure costs.

Consider:

```text
Log volume
×
Retention
×
Replication
×
Storage
×
Query workload
```

Cost controls include:

```text
Reduce unnecessary logs
Use appropriate retention
Compress data
Use lifecycle policies
Archive older logs
Avoid excessive DEBUG
Control index/shard count
```

---

# 69. Production Logging Principles

### Application

```text
stdout/stderr
structured JSON
INFO default
no secrets
no unnecessary PII
```

### Collection

```text
Fluent Bit
DaemonSet
buffering
metadata enrichment
```

### Processing

```text
Logstash
parsing
normalization
routing
filtering
```

### Storage

```text
Elasticsearch/OpenSearch
persistent storage
replication
lifecycle management
```

### Visualization

```text
Kibana
authentication
RBAC
dashboards
```

---

# 70. Final Architecture

```text
                         EKS
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
 order-service      payment-service    inventory-service
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                    stdout/stderr
                          │
                          ▼
                  ┌──────────────┐
                  │ Fluent Bit   │
                  │  DaemonSet   │
                  └───────┬──────┘
                          │
                          ▼
                  ┌──────────────┐
                  │  Logstash    │
                  │  Deployment  │
                  └───────┬──────┘
                          │
                          ▼
               ┌─────────────────────┐
               │ Elasticsearch /     │
               │ OpenSearch          │
               └──────────┬──────────┘
                          │
                          ▼
                     ┌─────────┐
                     │ Kibana  │
                     └─────────┘
```

And alongside it:

```text
Application
    │
    ├── Metrics ──────→ Prometheus ─────→ Grafana
    │
    ├── Logs ─────────→ Fluent Bit
    │                       ↓
    │                    Logstash
    │                       ↓
    │                  Elasticsearch
    │                       ↓
    │                    Kibana
    │
    └── Traces ───────→ OpenTelemetry
```

This gives us the three pillars:

```text
Metrics
Logs
Traces
```

---

# 71. Production Checklist

## Fluent Bit

* [ ] DaemonSet
* [ ] Node log collection
* [ ] Kubernetes metadata
* [ ] JSON parsing
* [ ] Position database
* [ ] Filesystem buffering
* [ ] Resource limits
* [ ] NetworkPolicy
* [ ] Monitoring

## Logstash

* [ ] Multiple replicas where required
* [ ] Pipeline configuration in Git
* [ ] JSON parsing
* [ ] Field normalization
* [ ] Persistent queue if required
* [ ] Resource sizing
* [ ] Monitoring
* [ ] Failure testing

## Elasticsearch/OpenSearch

* [ ] Multi-node architecture
* [ ] Persistent storage
* [ ] Encryption
* [ ] Authentication
* [ ] Authorization
* [ ] Shard planning
* [ ] Replica planning
* [ ] Lifecycle policy
* [ ] Retention
* [ ] Disk monitoring
* [ ] Backup/recovery strategy

## Kibana

* [ ] Authentication
* [ ] RBAC
* [ ] Private access
* [ ] Data views
* [ ] Dashboards
* [ ] Incident investigation views

## Security

* [ ] No secrets in logs
* [ ] PII reviewed
* [ ] TLS
* [ ] NetworkPolicy
* [ ] Least privilege
* [ ] Access auditing where required

---

# 72. What We Have Now

Our observability platform is becoming:

```text
                       EKS
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
       Metrics         Logs         Traces
          │             │             │
          ▼             ▼             ▼
    Prometheus      Fluent Bit    OpenTelemetry
          │             │             │
          ▼             ▼             ▼
       Grafana       Logstash      Trace Backend
          │             │             │
          │             ▼             │
          │        Elasticsearch      │
          │             │             │
          │             ▼             │
          │          Kibana            │
          │                           │
          └─────────────┬─────────────┘
                        │
                        ▼
                 Incident Response
```

The next chapter connects these logs to the **request flow across the 21 microservices**.

---

# Next Chapter

## `122-Log-Correlation.md`

We will implement end-to-end log correlation:

```text
Client
  │
  │ correlationId
  ▼
ALB
  │
  ▼
order-service
  │
  │ correlationId
  ▼
payment-service
  │
  │ correlationId
  ▼
inventory-service
  │
  │ correlationId
  ▼
notification-service
```

and cover:

* correlation ID generation
* propagation between Spring Boot services
* HTTP headers
* asynchronous Kafka/SQS propagation
* correlation IDs in logs
* correlation IDs in events
* request IDs vs correlation IDs
* trace IDs vs correlation IDs
* MDC in Java
* Spring filters/interceptors
* logging the same request across 21 services
* Kibana searches
* Grafana → logs navigation
* failure correlation
* duplicate/cross-request correlation problems
* security considerations
* production incident walkthrough
* the complete **request → metric → log → dependency** investigation workflow.
