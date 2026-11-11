# 39 — OpenTelemetry

> Production-focused DevOps notes for implementing distributed tracing across the 21-microservice AWS EKS e-commerce platform.

---

## 1. What Is OpenTelemetry?

OpenTelemetry, commonly called **OTel**, is an open-source observability framework for generating, collecting, and exporting telemetry.

Telemetry generally includes:

```text
Metrics
Logs
Traces
```

In this chapter our primary focus is:

```text
Distributed Tracing
```

Our architecture will eventually look like:

```text
User
 │
 ▼
CloudFront / WAF
 │
 ▼
ALB
 │
 ▼
order-service
 │
 ├── inventory-service
 │
 ├── pricing-service
 │
 ├── payment-service
 │      │
 │      └── fraud-service
 │
 └── notification-service
```

OpenTelemetry helps us understand this entire request path.

---

# 2. Why Do We Need Distributed Tracing?

With 21 microservices, a single customer request may cross many services.

For example:

```text
POST /api/v1/orders
```

could become:

```text
Client
  ↓
order-service
  ↓
inventory-service
  ↓
pricing-service
  ↓
payment-service
  ↓
fraud-service
  ↓
notification-service
```

Suppose the request takes:

```text
5 seconds
```

The question is:

> Which service caused the delay?

Without tracing:

```text
order-service = 5 sec
```

doesn't tell us enough.

With tracing:

```text
Total request = 5 sec

order-service       500 ms
inventory-service   200 ms
pricing-service     100 ms
payment-service    3,900 ms
fraud-service       250 ms
notification-service 50 ms
```

Now the bottleneck is obvious:

```text
payment-service
```

---

# 3. Logs vs Metrics vs Traces

These three signals complement each other.

### Metrics

Answer:

> What is happening?

Example:

```text
payment error rate = 18%
```

### Logs

Answer:

> What happened?

Example:

```text
Payment gateway timeout
```

### Traces

Answer:

> Where did it happen?

Example:

```text
order-service
    ↓
payment-service
    ↓
external gateway
       ↑
    4.2 sec
```

Together:

```text
Metrics
   +
Logs
   +
Traces
```

provide much stronger observability.

---

# 4. OpenTelemetry Is Not Jaeger

This distinction is very important.

OpenTelemetry:

```text
Generate
Collect
Process
Export
```

Jaeger:

```text
Store / query / visualize traces
```

Conceptually:

```text
Application
     │
     ▼
OpenTelemetry
     │
     ▼
OTel Collector
     │
     ▼
Jaeger
```

We will cover Jaeger in detail in:

```text
40-Jaeger.md
```

---

# 5. OpenTelemetry Architecture

Our target architecture:

```text
                   EKS
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
 Microservice Pods       OTel Collector
          │                   ▲
          │                   │
          └──── telemetry ────┘
                              │
                              ▼
                           Jaeger
```

A more complete production design:

```text
Application
    │
    ▼
OTel SDK / Agent
    │
    │ OTLP
    ▼
OTel Collector
    │
    ├── traces → Jaeger
    │
    ├── metrics → Prometheus
    │
    └── logs → selected backend
```

---

# 6. Why Use an OTel Collector?

We could theoretically make every application send traces directly to Jaeger:

```text
order-service ───────► Jaeger
payment-service ─────► Jaeger
catalog-service ──────► Jaeger
...
```

But this couples every application directly to the backend.

Instead:

```text
Applications
     │
     ▼
OTel Collector
     │
     ├── Jaeger
     ├── Prometheus
     └── other systems
```

Now applications only need to know:

```text
OTel Collector
```

The backend can change later.

---

# 7. Benefits of the Collector

The Collector provides:

* batching
* retries
* filtering
* enrichment
* sampling
* routing
* authentication
* TLS
* buffering
* telemetry fan-out

For example:

```text
Application
    │
    ▼
Collector
    │
    ├── production traces → Jaeger
    │
    └── selected metrics → Prometheus
```

This provides an abstraction layer between workloads and observability backends.

---

# 8. Collector Components

An OpenTelemetry Collector pipeline generally contains:

```text
Receivers
    ↓
Processors
    ↓
Exporters
```

Conceptually:

```text
INPUT
  ↓
RECEIVER
  ↓
PROCESSOR
  ↓
EXPORTER
  ↓
BACKEND
```

---

# 9. Receivers

Receivers accept telemetry.

Common examples include:

```text
OTLP
Jaeger
Zipkin
Prometheus
```

Our applications will primarily use:

```text
OTLP
```

So:

```text
Spring Boot
    │
    │ OTLP
    ▼
OTel Collector
```

---

# 10. OTLP

OTLP means:

```text
OpenTelemetry Protocol
```

It is the native protocol for OpenTelemetry telemetry.

Transport options include:

```text
OTLP/gRPC
OTLP/HTTP
```

Conceptually:

```text
Application
   │
   ├── OTLP/gRPC
   │
   └── OTLP/HTTP
         │
         ▼
   OTel Collector
```

The exact endpoint and protocol should be standardized across the platform.

---

# 11. Processors

Processors modify or control telemetry.

Examples include:

```text
batch
memory_limiter
resource
attributes
filter
tail_sampling
```

For production, a common pipeline concept is:

```text
Receiver
   ↓
memory_limiter
   ↓
batch
   ↓
sampling
   ↓
Exporter
```

The exact processor order depends on the desired behavior.

---

# 12. Batch Processor

Without batching:

```text
Trace 1 → backend
Trace 2 → backend
Trace 3 → backend
Trace 4 → backend
```

With batching:

```text
Trace 1
Trace 2
Trace 3
Trace 4
   │
   ▼
Batch
   │
   ▼
Backend
```

This can reduce network overhead and improve throughput.

---

# 13. Memory Limiter

Telemetry pipelines must protect themselves from memory exhaustion.

Conceptually:

```text
Telemetry volume
      ↓
Collector memory
      ↓
memory limiter
      ↓
protect process
```

Without protection:

```text
traffic spike
   ↓
telemetry spike
   ↓
collector memory ↑
   ↓
OOMKilled
```

Therefore the Collector should have resource limits and memory protection.

---

# 14. Resource Attributes

The Collector can enrich telemetry with metadata such as:

```text
service.name
service.version
deployment.environment
cloud.region
k8s.cluster.name
```

Example:

```text
service.name = order-service
deployment.environment = production
cloud.region = ap-south-1
```

This makes cross-service analysis easier.

---

# 15. Trace

A **trace** represents one end-to-end request.

Example:

```text
Trace ID = abc123
```

Everything associated with that request belongs to the same trace.

```text
Trace abc123
│
├── order-service
│
├── inventory-service
│
├── payment-service
│
└── fraud-service
```

---

# 16. Span

A span represents one operation inside a trace.

Example:

```text
Trace abc123
│
├── Span: order-service HTTP
│
├── Span: inventory HTTP
│
├── Span: payment HTTP
│
└── Span: PostgreSQL query
```

A trace consists of multiple related spans.

---

# 17. Parent and Child Spans

Suppose:

```text
order-service
   ↓
payment-service
```

We can have:

```text
Parent Span
order-service
      │
      └── Child Span
          payment-service
```

Then:

```text
payment-service
      │
      └── Child Span
          PostgreSQL
```

This forms a hierarchy.

---

# 18. Trace Example

```text
Trace: abc123

order-service
│
├── HTTP POST /orders       5000 ms
│
├── inventory-service        200 ms
│
├── pricing-service          100 ms
│
├── payment-service         4000 ms
│   │
│   └── payment-gateway     3800 ms
│
└── notification-service     50 ms
```

Immediately:

```text
payment gateway
```

looks suspicious.

---

# 19. Trace Context Propagation

This is one of the most important concepts in distributed tracing.

Suppose:

```text
Client
  ↓
order-service
```

The request receives:

```text
trace-id = abc123
```

When order-service calls payment-service, it propagates the trace context.

```text
order-service
    │
    │ trace-id = abc123
    ▼
payment-service
```

Payment service continues the same trace.

---

# 20. Without Context Propagation

Bad:

```text
order-service
   │
   ▼
payment-service
```

but:

```text
order trace = abc123
payment trace = xyz789
```

Now Jaeger sees two unrelated traces.

We lose the end-to-end relationship.

---

# 21. With Context Propagation

Correct:

```text
order-service
trace = abc123
    │
    ▼
payment-service
trace = abc123
    │
    ▼
fraud-service
trace = abc123
```

Now:

```text
one request
=
one distributed trace
```

---

# 22. W3C Trace Context

OpenTelemetry commonly uses the W3C Trace Context standard.

A request can carry headers such as:

```text
traceparent
```

Conceptually:

```text
Client
  │
  │ traceparent
  ▼
order-service
  │
  │ traceparent
  ▼
payment-service
```

This allows tracing context to travel between services.

---

# 23. Spring Boot Instrumentation

Our services use Spring Boot.

There are multiple ways to instrument them.

Common approaches include:

```text
Java agent
OpenTelemetry Spring Boot starter
manual SDK instrumentation
```

For a large microservice platform, automatic instrumentation is often the preferred starting point.

Why?

Because manually adding tracing code to 21 services creates unnecessary development and maintenance work.

---

# 24. Automatic Instrumentation

Conceptually:

```text
Spring Boot
    │
    ▼
OpenTelemetry Java Agent
    │
    ├── HTTP
    ├── JDBC
    ├── messaging
    └── supported libraries
```

The application can automatically generate spans for supported operations.

For example:

```text
HTTP request
     ↓
Controller
     ↓
HTTP client
     ↓
JDBC
```

---

# 25. Java Agent Placement

The OpenTelemetry Java Agent runs alongside the application JVM.

Conceptually:

```text
Order Pod
│
└── Container
     │
     ├── Java application
     │
     └── OpenTelemetry Java Agent
```

The agent is not a separate Kubernetes Pod.

This is different from:

```text
OTel Collector
```

which is normally a separate platform component.

---

# 26. Application vs Collector

Remember:

```text
OTel Java Agent
=
instrument application
```

while:

```text
OTel Collector
=
receive/process/export telemetry
```

So:

```text
Spring Boot
    │
    ▼
OTel Java Agent
    │
    ▼
OTLP
    │
    ▼
OTel Collector
    │
    ▼
Jaeger
```

---

# 27. Kubernetes Collector Deployment Models

There are several possible deployment patterns.

### Deployment

```text
OTel Collector
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

Useful for centralized collection.

### DaemonSet

```text
Node 1 → Collector
Node 2 → Collector
Node 3 → Collector
```

Useful when node-local collection is desirable.

### Sidecar

```text
Application Pod
 ├── Application
 └── Collector
```

Usually more expensive at scale.

---

# 28. Which Pattern for Our Project?

For our platform, a centralized Collector **Deployment** is a good starting architecture:

```text
21 Microservices
       │
       ▼
OTLP
       │
       ▼
OTel Collector Deployment
       │
       ▼
Jaeger
```

For larger environments, a two-tier Collector architecture can be useful:

```text
Application
    ↓
node-local Collector
    ↓
gateway Collector
    ↓
backend
```

This can improve scaling and isolation.

---

# 29. Centralized Collector

Example:

```text
EKS
│
├── production
│   ├── order-service
│   ├── payment-service
│   └── ...
│
└── observability
    └── OTel Collector
          ├── Pod 1
          ├── Pod 2
          └── Pod 3
```

Applications send telemetry to:

```text
otel-collector.observability.svc.cluster.local
```

---

# 30. Collector Service

Conceptually:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    app: otel-collector

  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317

    - name: otlp-http
      port: 4318
      targetPort: 4318
```

These are the commonly used OTLP ports.

Use only the protocols actually required by your deployment.

---

# 31. Collector Configuration

A simplified conceptual configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  memory_limiter:
  batch:

exporters:
  otlp:
    endpoint: jaeger-collector:4317

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

This is intentionally simplified.

Exact exporter configuration depends on the Jaeger deployment and supported protocol.

---

# 32. Configuration Flow

Our GitOps model:

```text
Git
 │
 ▼
OTel Collector configuration
 │
 ▼
ArgoCD
 │
 ▼
EKS
 │
 ▼
OTel Collector
```

Do not manually change the production Collector configuration with:

```bash
kubectl edit configmap
```

as the normal operating process.

---

# 33. Helm

The Collector can be installed using Helm.

Production model:

```text
Helm chart
   ↓
values.yaml
   ↓
Git
   ↓
ArgoCD
   ↓
EKS
```

Pin:

```text
chart version
image version
configuration
```

Avoid unpinned `latest`.

---

# 34. Collector Namespace

We can use:

```text
observability
```

or:

```text
tracing
```

depending on platform conventions.

For this project:

```text
monitoring
├── Prometheus
├── Grafana
└── Alertmanager

tracing
├── OpenTelemetry Collector
└── Jaeger
```

is a clean separation.

---

# 35. Resource Requests

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

These values are only starting points.

Real sizing depends on:

```text
spans/sec
average span size
sampling rate
processors
export destinations
```

---

# 36. Trace Volume

Suppose:

```text
21 services
```

and:

```text
500 requests/sec
```

If each request creates:

```text
20 spans
```

then approximately:

```text
500 × 20
=
10,000 spans/sec
```

This is a very different workload from:

```text
100 spans/sec
```

Therefore tracing requires capacity planning.

---

# 37. Sampling

Tracing every request may be expensive.

Suppose:

```text
100,000 requests/sec
```

and each request creates:

```text
30 spans
```

That's:

```text
3,000,000 spans/sec
```

Sending every span to the backend may be unnecessarily expensive.

Sampling controls what gets retained.

---

# 38. Head Sampling

Head sampling makes the decision near the beginning of the trace.

Example:

```text
100 requests
     ↓
10 sampled
```

approximately:

```text
10%
```

This reduces telemetry volume.

But the decision is made before we know whether the request will fail later.

---

# 39. Tail Sampling

Tail sampling waits until more of the trace is available.

Example:

```text
Request
   ↓
trace completes
   ↓
Collector evaluates
   │
   ├── error? → keep
   ├── slow? → keep
   └── normal? → maybe sample
```

This is powerful.

For example:

```text
Keep 100% of errors
Keep 100% of very slow traces
Keep 5% of normal traces
```

This can preserve valuable debugging information while reducing volume.

---

# 40. Why Tail Sampling Is Useful

Imagine:

```text
1,000 normal requests
```

and:

```text
5 failed requests
```

If random head sampling misses those 5:

```text
no trace
```

Tail sampling can prioritize:

```text
error traces
```

after seeing the completed request.

---

# 41. Production Sampling Strategy

A possible strategy:

```text
Successful normal traffic
    ↓
5%

Slow requests
    ↓
100%

Errors
    ↓
100%
```

This is only an example.

Real sampling should be based on:

* traffic
* storage
* cost
* debugging requirements
* compliance
* service criticality

---

# 42. Do Not Sample Blindly

For:

```text
payment-service
```

we may want higher retention than:

```text
health-check endpoints
```

Sampling can therefore be service-aware.

For example:

```text
payment-service → higher sampling
health endpoints → aggressive sampling
```

---

# 43. Health Checks

Kubernetes can generate many requests:

```text
/actuator/health
/actuator/health/readiness
```

If every health probe creates a trace:

```text
trace volume ↑
```

without much diagnostic value.

Consider filtering or sampling health-check traces appropriately.

Do not accidentally remove meaningful traffic from monitoring.

---

# 44. Trace Attributes

A span may contain:

```text
service.name
http.request.method
http.response.status_code
server.address
db.system
db.namespace
```

and other semantic attributes depending on instrumentation/version.

These allow queries like:

```text
service.name = payment-service
```

or:

```text
HTTP status = 500
```

---

# 45. Don't Add Sensitive Data

Never put:

```text
password
credit card
JWT
secret
AWS credential
```

into span attributes.

Bad:

```text
user.password = ...
```

Even if logs are protected, telemetry backends must also be treated as sensitive operational data stores.

---

# 46. High Cardinality in Traces

Avoid arbitrary high-cardinality attributes when they are not needed.

For example:

```text
customer_email
full_request_body
huge JSON response
```

can make telemetry expensive.

Use identifiers carefully.

For debugging:

```text
order.id
```

may be useful.

But don't blindly attach the entire request payload to every span.

---

# 47. Trace ID and Logs

This is where OpenTelemetry becomes extremely powerful.

Suppose:

```text
Trace ID:
abc123
```

The application log contains:

```text
trace.id = abc123
```

Now:

```text
Kibana
   │
   ▼
search trace.id = abc123
   │
   ▼
logs
```

and:

```text
Jaeger
   │
   ▼
trace abc123
```

Both point to the same request.

---

# 48. Trace ID and Metrics

Prometheus metrics might show:

```text
payment-service latency ↑
```

Then:

```text
Grafana
   ↓
identify time window
```

Then:

```text
Kibana
   ↓
find errors
```

Then:

```text
Jaeger
   ↓
find slow span
```

This is the full observability workflow.

---

# 49. Complete Correlation

```text
                   Incident
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
       Metrics       Logs       Traces
          │           │           │
      Prometheus   Elasticsearch  Jaeger
          │           │           │
          ▼           ▼           ▼
       Grafana      Kibana       Trace UI
```

All three can be correlated using:

```text
service.name
timestamp
trace.id
```

and other shared metadata.

---

# 50. Database Spans

Suppose:

```text
order-service
    ↓
PostgreSQL
```

OpenTelemetry instrumentation can create a database span.

Example:

```text
order-service
│
└── SELECT orders WHERE id = ?
       │
       └── 850 ms
```

This immediately indicates database latency.

---

# 51. HTTP Client Spans

Suppose:

```text
order-service
   ↓
payment-service
```

Tracing can show:

```text
HTTP POST /payments
```

with:

```text
duration = 1.8 sec
status = 200
```

Then we can investigate the downstream service.

---

# 52. Messaging Traces

Our project also uses:

```text
SQS/SNS
RabbitMQ
Kafka
```

Depending on the chosen integration and instrumentation, messaging spans can preserve trace context across asynchronous boundaries.

Conceptually:

```text
order-service
    │
    ▼
SQS
    │
    ▼
notification-service
```

Trace context can flow with the message.

This is different from ordinary HTTP propagation because messaging is asynchronous.

---

# 53. Asynchronous Tracing

HTTP:

```text
Service A
   │
   ▼
Service B
   │
   ▼
response
```

Messaging:

```text
Service A
   │
   ▼
Queue
   │
   │ time passes
   ▼
Service B
```

The trace may contain separate producer/consumer operations.

This helps identify:

```text
queue delay
consumer processing time
downstream processing
```

---

# 54. Trace Through Order Creation

Our example:

```text
POST /api/v1/orders
```

could produce:

```text
Trace abc123
│
├── ALB/request
│
├── order-service
│   │
│   ├── PostgreSQL
│   │
│   ├── inventory-service
│   │
│   ├── pricing-service
│   │
│   └── payment-service
│       │
│       └── fraud-service
│
└── SQS → notification-service
```

Now one customer operation can be visualized end-to-end.

---

# 55. Collector High Availability

Don't run:

```text
1 Collector Pod
```

in production and assume everything is fine.

If:

```text
Collector
   ↓
crashes
```

applications may lose their telemetry or experience exporter/backpressure issues depending on their SDK/export configuration.

Use multiple replicas where required:

```text
OTel Collector
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

---

# 56. Collector Service Load Balancing

Applications can send to:

```text
otel-collector.tracing.svc.cluster.local
```

The Kubernetes Service distributes connections/traffic to Collector Pods.

But understand the semantics of your chosen protocol and client behavior.

Some clients may maintain long-lived connections, so simply adding Pods doesn't guarantee perfectly even load distribution.

---

# 57. Collector Failure

If:

```text
Collector
   ↓
unavailable
```

the application exporter may:

```text
retry
buffer
drop
block
```

depending on configuration.

Therefore configure telemetry export carefully.

Observability should not bring down the application.

---

# 58. Critical Rule

> **Telemetry must never become a single point of failure for the application.**

Bad:

```text
Application
   ↓
Collector unavailable
   ↓
application requests fail
```

Good:

```text
Application
   ↓
telemetry export failure
   ↓
application continues serving business traffic
```

The exact behavior depends on the SDK/exporter configuration.

---

# 59. Collector Backpressure

Suppose:

```text
Applications
   ↓
10,000 spans/sec
```

but:

```text
Jaeger
   ↓
supports only 5,000 spans/sec
```

Then:

```text
Collector queue ↑
```

Eventually:

```text
memory ↑
disk/buffer ↑
dropped telemetry ↑
```

Monitor the Collector itself.

---

# 60. Collector Monitoring

Monitor:

```text
received spans
exported spans
dropped spans
export failures
queue size
CPU
memory
GC
```

Prometheus can scrape Collector metrics.

Architecture:

```text
OTel Collector
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

---

# 61. Collector Security

Production:

```text
Application
    │
    │ TLS
    ▼
OTel Collector
    │
    │ TLS
    ▼
Jaeger
```

Where authentication is supported/required, configure it.

Don't expose OTLP endpoints publicly.

---

# 62. NetworkPolicy

Allow:

```text
production namespaces
       │
       ▼
OTel Collector
```

But restrict:

```text
random namespace
       X
       ▼
OTel Collector
```

Similarly:

```text
OTel Collector
       │
       ▼
Jaeger
```

should be controlled.

---

# 63. ServiceAccount

The Collector should use a dedicated ServiceAccount.

Avoid:

```text
default
```

for production platform components.

If the Collector needs Kubernetes metadata:

```text
Collector
   ↓
Kubernetes API
```

grant only the required RBAC permissions.

---

# 64. Kubernetes Metadata

The Collector can enrich telemetry with Kubernetes information such as:

```text
cluster
namespace
Pod
container
node
```

Then a trace can be associated with:

```text
payment-service
Pod = payment-service-abc123
Node = ip-10-0-x-x
```

This is very useful during incidents.

---

# 65. Kubernetes Metadata + Logs

Now:

```text
Trace
   ↓
payment-service
   ↓
Pod abc123
```

and Kibana:

```text
kubernetes.pod.name = payment-service-abc123
```

can find the corresponding logs.

---

# 66. Production Configuration Pattern

Our configuration hierarchy:

```text
Application
│
├── OTEL_SERVICE_NAME
├── OTEL_EXPORTER_OTLP_ENDPOINT
├── OTEL_EXPORTER_OTLP_PROTOCOL
├── OTEL_RESOURCE_ATTRIBUTES
└── sampling configuration
```

Example conceptually:

```text
OTEL_SERVICE_NAME=order-service

OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.tracing:4317
```

For production, use the appropriate secure endpoint and protocol.

---

# 67. Environment Variables

A Kubernetes Deployment might conceptually include:

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: order-service

  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector.tracing.svc.cluster.local:4317
```

Environment-specific values should come from Helm values/GitOps rather than being duplicated manually across 21 services.

---

# 68. One Configuration Pattern for 21 Services

Instead of writing 21 completely different configurations:

```text
order
payment
inventory
catalog
...
```

standardize:

```text
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_EXPORTER_OTLP_PROTOCOL
OTEL_RESOURCE_ATTRIBUTES
```

and only vary:

```text
OTEL_SERVICE_NAME
```

This makes platform operations much easier.

---

# 69. Helm Values

Example:

```yaml
observability:
  otel:
    enabled: true
    endpoint: "otel-collector.tracing.svc.cluster.local:4317"
    protocol: grpc
```

Then:

```text
values-dev.yaml
values-qa.yaml
values-prod.yaml
```

can provide environment-specific configuration.

---

# 70. GitOps Flow

```text
Application Repository
        │
        ▼
CI builds image
        │
        ▼
ECR
        │
        ▼
GitOps repository
        │
        ▼
Helm values
        │
        ▼
ArgoCD
        │
        ▼
EKS
```

OTel configuration becomes part of the standard platform deployment model.

---

# 71. OpenTelemetry Collector Deployment Flow

```text
Platform GitOps
       │
       ▼
Helm
       │
       ▼
ArgoCD
       │
       ▼
EKS
       │
       ▼
OTel Collector
```

This is different from installing OTel manually on individual EC2/EKS nodes.

---

# 72. OpenTelemetry on Every Microservice

Eventually:

```text
order-service
payment-service
catalog-service
inventory-service
...
```

all emit telemetry through a standardized platform configuration.

Example:

```text
             OTel Collector
                  ▲
                  │
       ┌──────────┼──────────┐
       │          │          │
       │          │          │
    order      payment    catalog
    service    service    service
```

---

# 73. What Gets Instrumented?

Typical automatic instrumentation can cover supported:

```text
HTTP servers
HTTP clients
JDBC
Spring
messaging
database clients
```

Manual instrumentation can be added for business operations.

Example:

```text
createOrder()
processPayment()
reserveInventory()
```

But don't manually instrument everything.

Use automatic instrumentation first.

---

# 74. Business Spans

Suppose:

```java
createOrder()
```

is important.

A custom span can represent:

```text
create-order
```

with attributes such as:

```text
order.id
```

when safe and appropriate.

Then Jaeger shows:

```text
order-service
   └── create-order
```

This gives business-level context.

---

# 75. Don't Over-Instrument

Bad:

```text
Every method
   ↓
new span
```

This can create:

```text
millions of spans
```

Use spans where they provide meaningful diagnostic value.

Good candidates:

```text
external HTTP calls
database operations
messaging
important business operations
```

---

# 76. Error Status

Tracing should capture failures.

Example:

```text
payment-service
   ↓
HTTP 502
```

The span should reflect an error condition according to OpenTelemetry semantic conventions/instrumentation behavior.

This allows:

```text
Jaeger
   ↓
find error traces
```

more effectively.

---

# 77. Slow Request

Suppose:

```text
order request = 6 seconds
```

Jaeger shows:

```text
order-service       6 sec
 ├── inventory      100 ms
 ├── pricing        150 ms
 ├── payment        5.5 sec
 │    └── gateway   5.3 sec
 └── notification   100 ms
```

Root cause is likely:

```text
external payment gateway
```

instead of:

```text
order-service itself
```

---

# 78. Production Incident Workflow

Example alert:

```text
payment-service p95 latency > SLO
```

### Step 1

Grafana:

```text
payment-service latency increased
```

### Step 2

Kibana:

```text
payment-service
ERROR
```

shows:

```text
gateway timeout
```

### Step 3

Jaeger:

```text
payment-service
   ↓
gateway
   ↓
4.8 sec
```

### Step 4

Investigate:

```text
network
gateway
DNS
external dependency
```

This is observability working as a system.

---

# 79. OpenTelemetry and SLOs

Suppose:

```text
Checkout availability SLO = 99.9%
```

Metrics can calculate:

```text
error rate
latency
availability
```

Traces can help explain:

```text
which dependency is causing SLO violations
```

Traces are generally the investigation signal, while metrics are often the alerting signal.

---

# 80. OpenTelemetry and Kubernetes HPA

Tracing does not normally directly replace CPU/memory metrics for HPA.

Instead:

```text
HPA
 ↓
resource/custom metrics
```

while:

```text
OpenTelemetry
 ↓
trace analysis
```

Tracing helps us understand whether scaling is solving the problem.

Example:

```text
HPA increases replicas
       ↓
application CPU decreases
       ↓
but payment gateway latency remains
```

The bottleneck is external.

---

# 81. Trace Sampling and Cost

If:

```text
1 million requests/day
```

and:

```text
20 spans/request
```

then:

```text
20 million spans/day
```

before considering retries or additional spans.

At larger scale, tracing storage becomes significant.

Therefore:

```text
instrumentation
+
sampling
+
retention
+
storage
```

must be designed together.

---

# 82. Trace Retention

Not every trace needs to remain forever.

Example conceptual policy:

```text
Normal traces
   ↓
7 days

Error traces
   ↓
30 days
```

Exact retention depends on:

* incident requirements
* compliance
* storage
* cost

Jaeger/backend lifecycle design determines actual retention.

---

# 83. Collector vs Jaeger Responsibilities

| Component      | Responsibility               |
| -------------- | ---------------------------- |
| OTel SDK/Agent | Generate telemetry           |
| OTel Collector | Receive/process/export       |
| Jaeger         | Store/query/visualize traces |
| Prometheus     | Metrics                      |
| Grafana        | Metrics visualization        |
| Elasticsearch  | Logs/search                  |
| Kibana         | Log visualization            |

This separation is critical.

---

# 84. Common Failure — No Traces

Application:

```text
healthy
```

but:

```text
Jaeger
↓
no traces
```

Check:

```text
1. OTel agent loaded?
2. service name configured?
3. OTLP endpoint correct?
4. Collector Service exists?
5. Collector endpoints exist?
6. NetworkPolicy?
7. TLS?
8. Collector receiver enabled?
9. Collector exporter working?
10. sampling?
```

---

# 85. Common Failure — Collector Has No Receivers

If the Collector isn't configured with:

```text
OTLP receiver
```

then:

```text
Application
   ↓
OTLP
   ↓
Collector
```

cannot work.

Check the Collector configuration first.

---

# 86. Common Failure — Wrong Port

Typical OTLP ports:

```text
4317 → OTLP gRPC
4318 → OTLP HTTP
```

If application uses:

```text
4317
```

but Collector only exposes:

```text
4318
```

you have a protocol/endpoint mismatch.

Always verify:

```text
protocol
port
Service
targetPort
receiver
```

together.

---

# 87. Common Failure — Collector Running but No Export

Collector:

```text
received spans > 0
```

but:

```text
exported spans = 0
```

Investigate:

```text
Jaeger endpoint
TLS
authentication
exporter configuration
network
backend availability
```

---

# 88. Common Failure — Trace Exists but Is Broken

You see:

```text
order-service
```

but:

```text
payment-service
```

is missing.

Possible causes:

```text
context propagation broken
payment instrumentation disabled
HTTP client not instrumented
different protocol
sampling
messaging propagation issue
```

This is why distributed tracing requires consistent instrumentation across services.

---

# 89. Common Failure — Separate Trace IDs

If:

```text
order-service
trace = abc123
```

and:

```text
payment-service
trace = xyz789
```

then context propagation is probably broken.

Investigate:

```text
traceparent
HTTP client
proxy/gateway
instrumentation
messaging headers
```

---

# 90. Common Failure — Too Many Traces

Symptoms:

```text
Collector CPU ↑
Collector memory ↑
Jaeger storage ↑
network ↑
```

Possible causes:

```text
sampling disabled
health checks traced
debug traffic traced
duplicate instrumentation
too many custom spans
```

Fix:

```text
sampling
filtering
instrumentation review
```

---

# 91. Common Failure — Application Performance Impact

If tracing causes:

```text
application latency ↑
CPU ↑
memory ↑
```

investigate:

```text
sampling
span count
exporter behavior
agent configuration
Collector endpoint
batching
```

Telemetry must remain lightweight enough for production workloads.

---

# 92. Production Security Checklist

```text
[ ] OTLP endpoints private
[ ] TLS enabled where required
[ ] Authentication configured where required
[ ] NetworkPolicy
[ ] Least-privilege RBAC
[ ] Dedicated ServiceAccount
[ ] No secrets in spans
[ ] No sensitive request payloads
[ ] Sampling configured
[ ] Retention defined
[ ] Collector resources configured
[ ] Jaeger access restricted
```

---

# 93. Production Architecture

Our tracing architecture becomes:

```text
                         EKS
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
    order-service    payment-service   catalog-service
          │               │                │
          └───────────────┼────────────────┘
                          │
                    OTel Agent/SDK
                          │
                          │ OTLP
                          ▼
               ┌─────────────────────┐
               │ OTel Collector      │
               │                     │
               │ Receiver            │
               │ Processor           │
               │ Batch               │
               │ Sampling            │
               │ Exporter            │
               └──────────┬──────────┘
                          │
                          ▼
                       Jaeger
```

---

# 94. Complete Three-Pillar Architecture

We now have:

```text
                         EKS
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
    Metrics              Logs              Traces
       │                  │                  │
       ▼                  ▼                  ▼
  Prometheus          Fluent Bit        OpenTelemetry
       │                  │                  │
       ▼                  ▼                  ▼
    Grafana            Logstash          Collector
       │                  │                  │
       │                  ▼                  ▼
       │             Elasticsearch         Jaeger
       │                  │                  │
       ▼                  ▼                  ▼
  Alertmanager          Kibana          Trace analysis
```

---

# 95. Complete Request Flow

Let's follow one order request:

```text
Customer
   │
   ▼
CloudFront / WAF
   │
   ▼
ALB
   │
   ▼
order-service
   │
   ├──── trace ──────► OTel Collector
   │
   ├──── logs ───────► Fluent Bit
   │                      │
   │                      ▼
   │                   Logstash
   │                      │
   │                      ▼
   │                 Elasticsearch
   │
   └──── metrics ─────► Prometheus
```

Downstream:

```text
order-service
   │
   ├── inventory-service
   ├── pricing-service
   ├── payment-service
   │       │
   │       └── fraud-service
   │
   └── notification-service
```

Tracing connects these calls into one distributed trace.

---

# 96. Production Tool Placement

| Component           | Location                                    |
| ------------------- | ------------------------------------------- |
| OTel SDK/Java Agent | Application containers                      |
| OTel Collector      | EKS tracing/observability namespace         |
| Jaeger              | EKS tracing namespace / managed alternative |
| Prometheus          | EKS monitoring                              |
| Grafana             | EKS monitoring                              |
| Fluent Bit          | EKS DaemonSet                               |
| Logstash            | EKS logging namespace                       |
| Elasticsearch       | EKS logging platform                        |
| Kibana              | EKS logging platform                        |
| ArgoCD              | EKS platform                                |
| Terraform           | CI/control environment                      |

---

# 97. Final Troubleshooting Chain

When a request is slow:

```text
ALERT
  ↓
Grafana
  ↓
Identify service
  ↓
Kibana
  ↓
Find error logs
  ↓
trace.id
  ↓
Jaeger
  ↓
Identify slow span
  ↓
Kubernetes / AWS / dependency
  ↓
Root cause
```

This is the workflow we want to be able to execute confidently in production.

---

# 98. Production Checklist

```text
[ ] OTel standardized across services
[ ] Service names standardized
[ ] Resource attributes standardized
[ ] Trace context propagation verified
[ ] OTLP protocol standardized
[ ] Collector deployed through GitOps
[ ] Collector HA configured
[ ] Collector resources sized
[ ] Memory limiter configured
[ ] Batch processor configured
[ ] Sampling strategy defined
[ ] Health-check traces controlled
[ ] TLS configured
[ ] NetworkPolicy configured
[ ] RBAC configured
[ ] Jaeger connectivity verified
[ ] Logs contain trace IDs
[ ] Metrics/logs/traces correlation tested
[ ] Sensitive attributes removed
[ ] Trace retention defined
[ ] Incident runbook created
```

---

# 99. Final Mental Model

Remember these four layers:

```text
Application
    ↓
Instrumentation
    ↓
OpenTelemetry Collector
    ↓
Observability backend
```

For our project:

```text
Spring Boot
    ↓
OTel Java Agent
    ↓
OTLP
    ↓
OTel Collector
    ↓
Jaeger
```

And the most important concept:

> **A trace follows one request across multiple services. A span represents one operation within that request. Trace context propagation connects those spans together.**

---

# 100. What We Have Built So Far

Our observability platform is now becoming:

```text
                 ┌─────────────────────────────┐
                 │       21 Microservices      │
                 └──────────────┬──────────────┘
                                │
               ┌────────────────┼────────────────┐
               │                │                │
               ▼                ▼                ▼
            Metrics            Logs             Traces
               │                │                │
               ▼                ▼                ▼
          Prometheus        Fluent Bit       OTel Agent
               │                │                │
               ▼                ▼                ▼
            Grafana          Logstash        OTel Collector
               │                │                │
               │                ▼                ▼
               │           Elasticsearch       Jaeger
               │                │                │
               │                ▼                │
               │              Kibana             │
               │                │                │
               └────────────────┼────────────────┘
                                ▼
                       Incident Investigation
```

This is the foundation for the next stage.

---

## Next File — `40-Jaeger.md`

The next chapter will complete the tracing stack by going deep into **Jaeger**:

```text
OpenTelemetry Agent
        ↓
OTLP
        ↓
OTel Collector
        ↓
Jaeger
        ↓
Trace Storage
        ↓
Jaeger UI
```

It will cover **Jaeger architecture, collector/query components, storage, Elasticsearch integration, Kubernetes/EKS deployment, HA, sampling, trace retention, UI investigation, trace-to-log correlation with Kibana, trace-to-metrics correlation with Grafana, security, scaling, and production troubleshooting.**
