# 124. OpenTelemetry Production Implementation

> **Project:** Enterprise E-Commerce Platform
> **Environment:** AWS EKS
> **Application standard:** Java + Spring Boot
> **Tracing:** OpenTelemetry
> **Trace backend:** Tempo / compatible tracing backend
> **Collection:** OpenTelemetry Collector
> **Deployment model:** Kubernetes + Helm + ArgoCD
> **Scope:** 21 microservices

---

## 1. Objective

In Chapter 123, we introduced distributed tracing.

Now we will implement **OpenTelemetry in our production-style EKS environment**.

The target architecture is:

```text
                    CLIENT
                       |
                       v
                CloudFront / WAF
                       |
                       v
                      ALB
                       |
                       v
                  EKS Cluster
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
   order-service   payment-service  inventory-service
   Spring Boot     Spring Boot      Spring Boot
        |              |              |
        +--------------+--------------+
                       |
                       v
              OpenTelemetry SDK
                 / Java Agent
                       |
                       v
             OpenTelemetry Collector
                       |
                       v
                Trace Backend
                   (Tempo)
                       |
                       v
                    Grafana
```

The goal is that when an order request fails, we can follow the complete request across multiple services.

---

# 2. What OpenTelemetry Actually Does

OpenTelemetry does **not** store traces itself.

It is primarily an **observability instrumentation and telemetry collection standard/framework**.

Think:

```text
Application
    |
    | generates telemetry
    v
OpenTelemetry
    |
    | collects/processes
    v
Collector
    |
    | exports
    v
Backend
```

For our tracing implementation:

```text
Spring Boot
      |
      | OTLP
      v
OpenTelemetry Collector
      |
      v
Tempo
      |
      v
Grafana
```

---

# 3. OpenTelemetry Components

Our implementation contains several separate components.

### Application instrumentation

Creates:

* traces
* spans
* trace context
* HTTP telemetry
* database telemetry
* messaging telemetry

### OpenTelemetry Collector

Receives telemetry and performs:

* batching
* processing
* filtering
* enrichment
* retry
* exporting

### Trace backend

Stores traces.

Examples:

* Grafana Tempo
* Jaeger
* AWS X-Ray

For our project, we will use:

```text
Tempo
```

as the primary example.

### Grafana

Provides visualization and investigation.

---

# 4. Why We Need the Collector

A common beginner implementation is:

```text
Application
    |
    v
Tempo
```

That can work for small environments, but we want a production-oriented architecture.

Instead:

```text
Application
    |
    v
OTel Collector
    |
    +--> batching
    +--> retry
    +--> filtering
    +--> enrichment
    |
    v
Tempo
```

This gives the platform team a central telemetry layer.

If we later change the backend:

```text
Tempo
```

to:

```text
AWS X-Ray
```

or:

```text
another OTLP-compatible backend
```

the applications do not necessarily need to be redesigned.

---

# 5. Our Production Architecture

For our 21-service platform:

```text
                         INTERNET
                            |
                            v
                       CloudFront
                            |
                            v
                           ALB
                            |
                            v
                         EKS
                            |
       +--------------------+--------------------+
       |                    |                    |
       v                    v                    v
 order-service       payment-service      inventory-service
       |                    |                    |
       |                    |                    |
       +--------------------+--------------------+
                            |
                     OTEL instrumentation
                            |
                         OTLP/gRPC
                            |
                            v
                +-----------------------+
                | OpenTelemetry         |
                | Collector             |
                +-----------------------+
                            |
                            v
                         Tempo
                            |
                            v
                         Grafana
```

---

# 6. Telemetry Protocol

OpenTelemetry supports several protocols.

The most important for our implementation is:

```text
OTLP
```

OpenTelemetry Protocol.

We can use:

```text
OTLP/gRPC
```

or:

```text
OTLP/HTTP
```

For example:

```text
Spring Boot
    |
    | OTLP/gRPC
    v
OTel Collector
```

The exact collector endpoint, ports and protocol configuration should always match the versions deployed in our cluster.

---

# 7. Instrumentation Options

There are two major approaches for our Java services.

## Option 1 — OpenTelemetry Java Agent

Architecture:

```text
Spring Boot
     |
     +---- Java Application
     |
     +---- OpenTelemetry Java Agent
                    |
                    v
             OTel Collector
```

The Java agent can automatically instrument many common libraries.

This is very useful because we don't have to manually add tracing code everywhere.

---

# 8. Java Agent Example

Our Docker container could contain:

```text
/app/
├── app.jar
└── opentelemetry-javaagent.jar
```

The application starts approximately like:

```bash
java \
  -javaagent:/app/opentelemetry-javaagent.jar \
  -jar /app/app.jar
```

The exact Java agent version should be pinned rather than using a floating version.

---

# 9. Kubernetes Configuration

We can provide environment variables through the Deployment.

Example:

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: order-service

  - name: OTEL_RESOURCE_ATTRIBUTES
    value: service.namespace=ecommerce,service.version=1.0.0,deployment.environment=production

  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector.observability.svc.cluster.local:4317

  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: grpc
```

These values tell the agent:

```text
Service name = order-service
Environment = production
Collector = otel-collector
Protocol = OTLP/gRPC
```

The exact environment-variable support and defaults should be verified against the Java agent version we pin.

---

# 10. Service Naming

Service names are extremely important.

We should use consistent names:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

Do not randomly use:

```text
order
orders
OrderService
order-api
orders-prod
```

for the same service.

Use one canonical service identity.

---

# 11. Resource Attributes

We want additional metadata attached to telemetry.

Example:

```text
service.name=order-service
service.version=1.4.2
deployment.environment=production
service.namespace=ecommerce
```

We can also associate Kubernetes metadata such as:

```text
namespace
pod
node
container
cluster
```

This becomes extremely useful during incident investigation.

---

# 12. Trace Example

Suppose the customer creates an order.

The request:

```http
POST /api/v1/orders
```

might produce:

```text
Trace ID
|
+-- order-service
|      |
|      +-- customer-service
|      |
|      +-- product-service
|      |
|      +-- pricing-service
|      |
|      +-- payment-service
|      |
|      +-- inventory-service
|      |
|      +-- notification-service
|
+-- PostgreSQL
```

One trace can therefore show the entire transaction.

---

# 13. Trace and Span

Example:

```text
Trace: 8f92ab...
```

contains:

```text
Span 1: POST /orders
    |
    +-- Span 2: GET customer
    |
    +-- Span 3: GET product
    |
    +-- Span 4: calculate price
    |
    +-- Span 5: authorize payment
    |
    +-- Span 6: reserve inventory
```

Each span has:

```text
trace_id
span_id
parent_span_id
operation
start_time
duration
status
attributes
events
```

---

# 14. HTTP Propagation

Suppose:

```text
order-service
```

calls:

```text
payment-service
```

The trace context must travel with the request.

Modern distributed tracing commonly uses:

```text
W3C Trace Context
```

with headers such as:

```text
traceparent
```

Conceptually:

```text
order-service
     |
     | traceparent
     v
payment-service
```

Payment service creates a child span rather than starting an unrelated trace.

---

# 15. Why This Matters

Without propagation:

```text
Trace A
order-service

Trace B
payment-service

Trace C
inventory-service
```

We cannot easily connect them.

With propagation:

```text
                 Trace A
                    |
       +------------+------------+
       |            |            |
     Order        Payment     Inventory
      Span         Span         Span
```

Now we can follow one business transaction.

---

# 16. Database Tracing

OpenTelemetry instrumentation can create spans for database operations.

Example:

```text
order-service
    |
    +-- HTTP request
    |
    +-- PostgreSQL query
```

We might see:

```text
POST /api/v1/orders
    |
    +-- SELECT customer
    |
    +-- SELECT product
    |
    +-- INSERT order
```

This helps distinguish:

```text
application latency
```

from:

```text
database latency
```

---

# 17. Redis Tracing

For Redis operations:

```text
order-service
      |
      +-- Redis GET
      |
      +-- Redis SET
```

This helps identify:

```text
Redis latency
Redis connection problems
slow operations
```

Again, instrumentation capabilities depend on the library and agent/SDK version being used.

---

# 18. External API Tracing

Suppose:

```text
payment-service
```

calls an external payment provider.

The trace can become:

```text
order-service
     |
     v
payment-service
     |
     v
External Payment API
```

If the external API takes:

```text
4.8 seconds
```

we can see that latency inside the overall trace.

---

# 19. Messaging Tracing

Our platform uses:

* SQS
* SNS
* RabbitMQ
* Kafka

Tracing must also work across asynchronous boundaries.

Example:

```text
order-service
      |
      v
Kafka
      |
      v
inventory-service
```

The event should carry trace context where supported by the instrumentation and messaging client.

We should also retain our explicit business identifiers:

```json
{
  "eventId": "evt-123",
  "eventType": "OrderCreated",
  "version": "1",
  "correlationId": "corr-456",
  "traceId": "trace-789"
}
```

Important distinction:

```text
eventId
```

identifies the event.

```text
correlationId
```

identifies the business/request journey.

```text
traceId
```

identifies the distributed trace.

They should not automatically be treated as the same identifier.

---

# 20. OpenTelemetry Collector Deployment

We will deploy the Collector into:

```text
observability
```

namespace.

Example:

```bash
kubectl create namespace observability
```

In our actual project, namespace creation and deployment should be managed through GitOps rather than manually.

---

# 21. Helm Installation

A common approach is to deploy the OpenTelemetry Collector using the OpenTelemetry Helm chart.

Our GitOps repository can contain:

```text
gitops/
└── platform/
    └── opentelemetry/
        ├── namespace.yaml
        ├── values.yaml
        └── application.yaml
```

ArgoCD then manages the deployment.

---

# 22. Example Collector Configuration

Conceptually:

```yaml
mode: deployment

config:
  receivers:
    otlp:
      protocols:
        grpc:
        http:

  processors:
    memory_limiter:
      check_interval: 1s
      limit_mib: 512

    batch:
      timeout: 5s
      send_batch_size: 1024

  exporters:
    otlp:
      endpoint: tempo.observability.svc.cluster.local:4317

  service:
    pipelines:
      traces:
        receivers:
          - otlp
        processors:
          - memory_limiter
          - batch
        exporters:
          - otlp
```

This is an **illustrative configuration**. Collector chart/version-specific configuration keys should be validated against the exact version we deploy.

---

# 23. Why Memory Limiter?

Imagine:

```text
100 traces/sec
```

normally.

Suddenly:

```text
20,000 traces/sec
```

during a flash sale.

The Collector could consume excessive memory.

A memory limiter provides protection.

Conceptually:

```text
Traffic spike
     |
     v
Collector
     |
     +-- memory protection
     |
     v
Exporter
```

We don't want our observability platform to become the reason the cluster becomes unstable.

---

# 24. Why Batch?

Instead of exporting every span independently:

```text
Span
Span
Span
Span
Span
```

the Collector can batch telemetry:

```text
Span
Span
Span
Span
      ↓
   Batch
      ↓
  Backend
```

This can reduce network and processing overhead.

---

# 25. Retry and Export Failure

Suppose Tempo becomes temporarily unavailable.

We don't want the Collector to immediately lose every trace.

Production configuration should account for:

```text
retry
queueing
batching
backpressure
memory protection
```

depending on the deployment model and reliability requirements.

But there is an important principle:

> **Observability must not consume unlimited resources while trying to recover from its own failure.**

---

# 26. Collector Architecture for Production

For higher availability, we can run multiple Collector replicas:

```text
                 Applications
                      |
             +--------+--------+
             |        |        |
             v        v        v
          Collector Collector Collector
             |        |        |
             +--------+--------+
                      |
                      v
                    Tempo
```

Kubernetes Deployment:

```yaml
replicas: 2
```

is a starting point, not a universal production sizing rule.

Sizing depends on:

* spans/sec
* attributes
* payload size
* processors
* exporters
* retention architecture
* CPU
* memory

---

# 27. Collector Service

Applications should normally communicate with a Kubernetes Service rather than a specific Collector Pod.

Example:

```text
otel-collector.observability.svc.cluster.local
```

Flow:

```text
order-service
      |
      v
Kubernetes Service
      |
      +---- Collector Pod 1
      |
      +---- Collector Pod 2
```

Kubernetes provides service-level load balancing.

---

# 28. Agent vs Gateway Collector

There are multiple deployment patterns.

### Pattern 1 — Gateway

```text
Applications
     |
     v
Collector Deployment
     |
     v
Backend
```

### Pattern 2 — Agent

A Collector runs close to each node/workload.

```text
Application
    |
    v
Node Collector
    |
    v
Gateway Collector
    |
    v
Backend
```

### Pattern 3 — Hybrid

```text
Pods
 |
 v
DaemonSet Collector
 |
 v
Gateway Collector
 |
 v
Tempo
```

For our initial implementation, we can use a **Collector Deployment** and evolve to an agent/gateway architecture if scale or network topology requires it.

---

# 29. Trace Backend

We use:

```text
Grafana Tempo
```

as our example backend.

Architecture:

```text
Application
     |
     v
OTel Collector
     |
     v
Tempo
     |
     v
Grafana
```

Tempo stores trace data.

Grafana provides investigation and visualization.

---

# 30. Why Not Store Traces in Elasticsearch?

We already use Elasticsearch/OpenSearch-style infrastructure for logs.

Logs:

```text
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

Traces:

```text
OpenTelemetry
    ↓
Collector
    ↓
Tempo
    ↓
Grafana
```

This separation allows each backend to be optimized for its workload.

---

# 31. Connect Grafana to Tempo

Grafana can have:

```text
Prometheus datasource
Elasticsearch datasource
Tempo datasource
```

So our observability platform becomes:

```text
                  Grafana
              /      |       \
             /       |        \
            v        v         v
      Prometheus  Elasticsearch  Tempo
          |           |            |
       Metrics       Logs        Traces
```

This gives us the three pillars:

```text
Metrics
Logs
Traces
```

---

# 32. Trace-to-Log Correlation

We already implemented correlation in Chapter 122.

Now combine it with tracing.

Example log:

```json
{
  "timestamp": "2026-09-30T10:10:00Z",
  "level": "ERROR",
  "service": "payment-service",
  "message": "Payment authorization failed",
  "correlationId": "corr-123",
  "traceId": "abc123",
  "spanId": "def456"
}
```

Now we can:

```text
Grafana Trace
      |
      v
Trace ID
      |
      v
Elasticsearch
      |
      v
Relevant logs
```

This is extremely useful during incidents.

---

# 33. Trace-to-Metric Correlation

Suppose Prometheus shows:

```text
payment-service
HTTP 5xx = 8%
```

We investigate.

Grafana:

```text
Metric
  ↓
Service
  ↓
Trace
  ↓
Slow span
  ↓
Database
  ↓
Logs
```

This creates an investigation workflow rather than looking at every system independently.

---

# 34. Sampling

Tracing every request can become expensive.

Suppose:

```text
100,000 requests/sec
```

and every request generates:

```text
10 spans
```

That's:

```text
1,000,000 spans/sec
```

So we need a sampling strategy.

---

# 35. Head Sampling

Decision is made near the beginning of the trace.

For example:

```text
Sample 10%
```

means approximately:

```text
100 requests
 ↓
10 traces retained
```

This is relatively simple but may miss interesting traces.

---

# 36. Tail Sampling

Tail sampling allows the system to inspect a trace before deciding whether to retain it.

For example:

```text
Successful + fast
    → low sampling

Error
    → retain

Very slow
    → retain

Important business transaction
    → retain
```

This is often more useful for production troubleshooting.

It also increases Collector complexity and resource requirements.

---

# 37. Example Sampling Policy

Conceptually:

```text
Normal successful request
        ↓
     sample %

HTTP 5xx
        ↓
     keep

Latency > threshold
        ↓
     keep

Critical payment transaction
        ↓
     keep
```

Exact sampling percentages should be based on traffic volume, cost, storage and troubleshooting requirements.

---

# 38. Never Put Secrets in Traces

Do NOT record:

```text
password
JWT token
credit card number
CVV
database password
AWS credentials
Secrets Manager values
```

Also avoid blindly recording:

```text
request body
response body
authorization headers
```

especially when they contain sensitive data.

---

# 39. PII Protection

Our ecommerce platform may process:

```text
customer information
email
phone number
address
payment information
```

Tracing should minimize sensitive data.

Instead of:

```text
customerEmail=john@example.com
```

we might record:

```text
customer_id=customer-123
```

where appropriate.

Data classification and compliance requirements should determine the exact policy.

---

# 40. OpenTelemetry + Spring Boot

Our Java services can use:

```text
Spring Boot
+
OpenTelemetry Java Agent
```

This can instrument common operations automatically.

For example:

```text
HTTP
JDBC
Redis
Kafka
HTTP clients
```

depending on supported libraries and instrumentation.

This is useful because we don't want developers manually writing tracing logic for every database call.

---

# 41. Manual Instrumentation

Sometimes automatic instrumentation is not enough.

Suppose we have business logic:

```java
processOrder()
```

We may want a custom span:

```text
processOrder
```

Then:

```text
Trace
 |
 +-- HTTP /orders
       |
       +-- processOrder
             |
             +-- validateOrder
             +-- calculatePrice
             +-- reserveInventory
```

Manual instrumentation should be used selectively.

Don't create hundreds of unnecessary spans for every method.

---

# 42. Business Spans

Good candidates:

```text
createOrder
authorizePayment
reserveInventory
generateInvoice
publishOrderEvent
```

Poor candidates:

```text
getVariable
calculateInteger
simpleGetter
setter
```

The objective is useful operational visibility, not tracing every line of code.

---

# 43. Kubernetes Metadata

We want traces associated with:

```text
cluster
namespace
pod
container
node
deployment
service
version
environment
```

Example:

```text
service.name = order-service
deployment.environment = production
k8s.namespace.name = ecommerce
k8s.pod.name = order-service-7f8d9
```

This helps answer:

> Which pod generated this problematic trace?

---

# 44. Deployment Strategy

Our OpenTelemetry infrastructure belongs in the platform/GitOps layer.

Example:

```text
gitops/
├── platform/
│   ├── prometheus/
│   ├── grafana/
│   ├── alertmanager/
│   ├── fluent-bit/
│   ├── logstash/
│   ├── opentelemetry/
│   └── tempo/
│
└── applications/
    ├── order-service/
    ├── payment-service/
    └── ...
```

ArgoCD manages these components.

---

# 45. GitOps Flow

```text
Developer
    |
    v
GitHub
    |
    v
GitOps Repository
    |
    v
Pull Request
    |
    v
Review
    |
    v
Merge
    |
    v
ArgoCD
    |
    v
EKS
```

No manual:

```bash
kubectl apply
```

for normal production changes.

---

# 46. OpenTelemetry Deployment Responsibility

The platform team owns:

```text
Collector
Tempo
Grafana integration
sampling
resource sizing
networking
security
retention
upgrades
```

Application teams own:

```text
service name
version metadata
business instrumentation where necessary
sensitive-data filtering
application-specific trace attributes
```

This is a realistic platform engineering boundary.

---

# 47. Observability Namespace

We can isolate observability components:

```text
observability
```

Example:

```text
kubectl get pods -n observability
```

could show:

```text
prometheus-...
grafana-...
alertmanager-...
otel-collector-...
tempo-...
```

Logging components may use:

```text
logging
```

depending on the platform organization.

---

# 48. Network Security

We should not expose the Collector publicly.

Bad:

```text
Internet
   |
   v
OTel Collector
```

Preferred:

```text
EKS workloads
     |
     v
Private Kubernetes Service
     |
     v
Collector
```

Similarly:

```text
Tempo
Grafana
```

should be private and access-controlled.

---

# 49. NetworkPolicy

We can restrict traffic.

Conceptually:

```text
application namespaces
        |
        | OTLP
        v
observability/collector
```

And:

```text
collector
   |
   | trace export
   v
tempo
```

Only required paths should be permitted.

---

# 50. Resource Requests and Limits

Observability workloads need resources too.

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

These are starting examples, not production sizing values.

Actual sizing requires:

```text
requests/sec
spans/sec
attributes/span
sampling
batch size
export rate
backend performance
```

---

# 51. Monitor the Collector

We must monitor the monitoring system.

Important signals:

```text
Collector CPU
Collector memory
received spans
exported spans
export failures
dropped spans
queue size
processor errors
receiver errors
```

Otherwise we might have:

```text
Application healthy
        |
        v
Collector broken
        |
        v
No traces
```

and not realize our observability system is failing.

---

# 52. Alert Examples

Potential alerts:

```text
CollectorDown
CollectorHighMemory
CollectorExportFailures
CollectorDroppedSpans
CollectorQueueGrowing
TempoUnavailable
HighTraceIngestionLatency
```

These should be owned by the platform/SRE team.

---

# 53. Failure Scenario — Collector Down

Suppose:

```text
order-service
     |
     X
Collector unavailable
```

The business application should ideally continue operating.

Tracing is important, but:

> **The observability pipeline should not become a hard dependency for business traffic.**

The application must not fail an order simply because the tracing backend is unavailable.

---

# 54. Failure Scenario — Tempo Down

```text
Application
    |
    v
Collector
    |
    X
Tempo
```

Collector should use appropriate buffering/retry behavior.

But buffering cannot be unlimited.

Eventually:

```text
memory limit
queue limit
retention limit
```

must protect the platform.

---

# 55. Failure Scenario — High Traffic

Suppose a flash sale causes:

```text
1,000 req/sec
     ↓
20,000 req/sec
```

Tracing volume also increases.

We need:

```text
sampling
+
Collector scaling
+
backend capacity
+
resource limits
+
monitoring
```

This is why tracing is part of **capacity planning**, not just application configuration.

---

# 56. Trace Investigation Example

Customer reports:

> "Order creation is taking 8 seconds."

We start with Grafana.

### Step 1

Check:

```text
order-service latency
```

### Step 2

Find a slow trace.

### Step 3

Open the trace.

We discover:

```text
POST /orders       8.1 sec
 |
 +-- customer-service     50 ms
 |
 +-- product-service      70 ms
 |
 +-- pricing-service      80 ms
 |
 +-- payment-service     120 ms
 |
 +-- inventory-service  7.6 sec
```

### Step 4

Open inventory span.

```text
inventory-service
      |
      +-- PostgreSQL query
             |
             +-- 7.4 sec
```

### Step 5

Correlate the trace ID with logs.

```text
Elasticsearch
```

shows:

```text
database connection pool exhausted
```

Now the investigation has gone:

```text
Metric
  ↓
Trace
  ↓
Service
  ↓
Database
  ↓
Logs
```

This is exactly why we built the three-pillar observability architecture.

---

# 57. OpenTelemetry + Our Existing Stack

Our complete observability platform now looks like:

```text
                    EKS
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
    Metrics         Logs         Traces
       |             |             |
       v             v             v
 Prometheus      Fluent Bit    OpenTelemetry
       |             |             |
       v          Logstash       Collector
       |             |             |
       v             v             v
    Grafana    Elasticsearch     Tempo
       |             |             |
       +-------------+-------------+
                     |
                  Grafana
```

---

# 58. Three Pillars

## Metrics

Answers:

> **Is something wrong?**

Examples:

```text
CPU
memory
latency
error rate
request rate
pod restarts
queue depth
```

## Logs

Answers:

> **What happened?**

Example:

```text
Payment authorization failed
```

## Traces

Answers:

> **Where did the request spend time and where did it fail?**

Together:

```text
Metrics → Detect
Logs → Understand
Traces → Locate
```

---

# 59. 21 Microservices

Our tracing architecture should cover all 21 services:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

We should **not manually create a completely different tracing architecture for every service**.

Instead:

```text
Standard instrumentation
        +
Standard resource attributes
        +
Standard Collector
        +
Standard propagation
        +
Standard dashboards
```

This is the platform engineering approach.

---

# 60. Standard Service Configuration

Every Spring Boot service should follow the same basic pattern:

```text
SERVICE_NAME
SERVICE_VERSION
ENVIRONMENT
OTEL_ENDPOINT
OTEL_PROTOCOL
```

Example:

```text
order-service
payment-service
inventory-service
```

all send telemetry to:

```text
otel-collector
```

---

# 61. Application Repository Responsibility

Each application repository contains its application-specific configuration.

Example:

```text
order-service/
├── src/
├── Dockerfile
├── pom.xml
├── helm/
│   └── order-service/
└── .github/
    └── workflows/
```

Tracing configuration belongs in the standardized deployment template.

The platform team should avoid requiring every developer to manually reinvent:

```text
OTEL environment variables
ServiceMonitor
labels
annotations
resource attributes
```

---

# 62. Reusable Helm Pattern

Our platform Helm template can provide:

```yaml
otel:
  enabled: true

  serviceName: order-service

  endpoint: http://otel-collector.observability.svc.cluster.local:4317

  protocol: grpc
```

Then:

```text
order-service
payment-service
inventory-service
```

can use the same pattern.

---

# 63. Versioning

We should track:

```text
Application version
OpenTelemetry Java Agent version
Collector version
Helm chart version
Tempo version
Grafana version
```

Example:

```text
order-service: 1.4.2
otel-javaagent: pinned-version
otel-collector: pinned-version
tempo: pinned-version
```

Do not blindly deploy:

```text
latest
```

for production observability components.

---

# 64. Upgrade Strategy

Never upgrade all observability components simultaneously without testing.

A safer process:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
Production canary
 ↓
Production rollout
```

Validate:

```text
traces received
trace attributes
sampling
export
Grafana queries
resource consumption
application latency
```

---

# 65. Production Troubleshooting

### Problem: No traces

Check:

```bash
kubectl get pods -n observability
```

Then:

```bash
kubectl logs -n observability deploy/otel-collector
```

Check service:

```bash
kubectl get svc -n observability
```

Check application configuration:

```bash
kubectl describe pod <pod> -n ecommerce
```

Check:

```text
OTEL endpoint
protocol
service name
network policy
Collector health
backend availability
```

---

# 66. Problem: Traces Exist but Service Name Is Wrong

Check:

```text
OTEL_SERVICE_NAME
```

and:

```text
service.name
```

Don't try to fix this by creating random Grafana transformations.

Fix the telemetry source.

---

# 67. Problem: Traces Stop at One Service

Example:

```text
order-service
    |
    v
payment-service
    X
```

Investigate:

```text
trace propagation
HTTP headers
instrumentation
HTTP client
framework version
async boundaries
```

For messaging:

```text
Kafka/SQS/RabbitMQ propagation
```

must also be checked.

---

# 68. Problem: Collector CPU Is Too High

Investigate:

```text
span rate
sampling
attributes
processors
exporter
batch size
backend latency
```

Then consider:

```text
scale Collector replicas
```

or:

```text
adjust sampling
```

or:

```text
optimize instrumentation
```

Do not simply keep increasing CPU indefinitely.

---

# 69. Problem: Trace Storage Growing Too Fast

Check:

```text
request volume
sampling rate
trace duration
span count
retention
attribute size
```

Potential actions:

```text
reduce unnecessary spans
adjust sampling
reduce retention
control high-cardinality attributes
increase backend capacity
```

---

# 70. Production Security Checklist

```text
[ ] Collector is private
[ ] Tempo is private
[ ] Grafana authentication enabled
[ ] RBAC configured
[ ] NetworkPolicy configured
[ ] TLS used where required
[ ] Sensitive headers filtered
[ ] Passwords excluded
[ ] Tokens excluded
[ ] Payment data excluded
[ ] PII minimized
[ ] IAM follows least privilege
[ ] Trace retention defined
[ ] Access logs/audit requirements defined
```

---

# 71. Production Reliability Checklist

```text
[ ] Collector has multiple replicas where required
[ ] Resource requests configured
[ ] Resource limits configured
[ ] Memory protection configured
[ ] Batching configured
[ ] Retry strategy configured
[ ] Export queue/buffering considered
[ ] Backend capacity planned
[ ] Sampling configured
[ ] Collector monitored
[ ] Backend monitored
[ ] Alerts configured
[ ] Failure testing completed
```

---

# 72. End-to-End Production Architecture

Our final observability flow becomes:

```text
                         USER
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
          +----------------+----------------+
          |                |                |
          v                v                v
       Order           Payment          Inventory
       Service         Service          Service
          |                |                |
          +----------------+----------------+
                           |
              +------------+------------+
              |            |            |
              v            v            v
           Metrics        Logs        Traces
              |            |            |
              v            v            v
         Prometheus     Fluent Bit   OpenTelemetry
              |            |         Collector
              v            v            |
           Grafana     Logstash          |
                           |             v
                           v           Tempo
                      Elasticsearch       |
                           |              |
                           +-------> Grafana
```

---

# 73. Final Investigation Workflow

When production has an incident:

```text
1. Alert fires
       ↓
2. Grafana metrics
       ↓
3. Identify affected service
       ↓
4. Open distributed trace
       ↓
5. Find slow/error span
       ↓
6. Obtain traceId
       ↓
7. Search Elasticsearch
       ↓
8. Correlate application logs
       ↓
9. Check database/cache/messaging
       ↓
10. Identify root cause
       ↓
11. Mitigate
       ↓
12. Verify recovery through metrics
       ↓
13. Verify traces
       ↓
14. Document incident/RCA
```

This connects everything we have built so far.

---

# 74. What We Have Now

At this stage our platform has:

```text
                    OBSERVABILITY
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
      Metrics           Logs            Traces
        |                |                |
   Prometheus          ELK             OpenTelemetry
        |                |                |
        v                v                v
     Grafana        Elasticsearch        Tempo
                         |
                         v
                       Kibana
```

And the application layer:

```text
21 Microservices
       |
       +-- Metrics
       +-- Logs
       +-- Traces
       +-- Correlation ID
       +-- Trace ID
       +-- Business events
```

This gives us a proper production-style observability foundation rather than simply installing Prometheus and Grafana.

---

# 75. Chapter Completion Checklist

```text
[ ] Understand OpenTelemetry
[ ] Understand OTLP
[ ] Understand traces/spans
[ ] Understand W3C Trace Context
[ ] Understand Java Agent
[ ] Understand Spring Boot instrumentation
[ ] Understand Collector
[ ] Understand Collector processors
[ ] Understand batching
[ ] Understand retry/buffering
[ ] Understand sampling
[ ] Understand Tempo
[ ] Understand Grafana trace integration
[ ] Understand trace-to-log correlation
[ ] Understand trace-to-metric correlation
[ ] Understand async messaging tracing
[ ] Understand PII protection
[ ] Understand Collector scaling
[ ] Understand production troubleshooting
[ ] Understand GitOps deployment
[ ] Understand observability ownership
```

---

## 76. Current Architecture After Chapter 124

We now have:

```text
                 AWS EKS E-COMMERCE PLATFORM
                              |
        +---------------------+----------------------+
        |                     |                      |
        v                     v                      v
     Metrics                Logs                   Traces
        |                     |                      |
  Prometheus             Fluent Bit             OpenTelemetry
        |                     |                  Java Agent
        |                  Logstash                   |
        |                     |                      |
        v                     v                      v
     Grafana             Elasticsearch          Collector
                               |                      |
                               v                      v
                            Kibana                 Tempo
                                                      |
                                                      v
                                                   Grafana
```

The next chapter is:

# **125 — SLO / SLA / SLI Implementation**

There we will connect our **metrics + logs + traces** to actual production reliability objectives such as:

```text
Availability
Latency
Error Rate
SLO
Error Budget
Burn Rate
```

and define SLOs for our 21-microservice ecommerce platform.
