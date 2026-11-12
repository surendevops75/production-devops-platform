# 123 — Distributed Tracing

> **Goal:** Implement production-grade distributed tracing for the 21-microservice E-Commerce Platform so engineers can follow one request across services, databases, Redis, Kafka/SQS, and external dependencies.

---

# 1. Why Distributed Tracing?

Log correlation tells us:

```text id="h3wq0u"
Which logs belong together?
```

Metrics tell us:

```text id="9o3f9s"
Is the system healthy?
```

Distributed tracing tells us:

```text id="z0f0r3"
Where did the request spend its time?
Which dependency is slow?
Where did the failure originate?
Which service called which service?
```

Example:

```text id="8gl5xq"
Customer
   │
   ▼
ALB
   │
   ▼
order-service             850ms
   │
   ├── customer-service    40ms
   │
   ├── product-service     35ms
   │
   ├── pricing-service     80ms
   │
   ├── tax-service         55ms
   │
   └── payment-service    620ms
                              │
                              └── Payment Provider 580ms
```

Immediately we can see where the latency is concentrated.

---

# 2. The Three Observability Pillars

Our platform now has:

```text id="k3a4q7"
             Observability
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Metrics      Logs      Traces
       │          │          │
       ▼          ▼          ▼
 Prometheus    ELK       Trace Backend
       │          │          │
       ▼          ▼          ▼
   Grafana     Kibana      Grafana
```

They answer different questions.

| Signal  | Main question        |
| ------- | -------------------- |
| Metrics | Is something wrong?  |
| Logs    | What happened?       |
| Traces  | Where did it happen? |

---

# 3. Trace

A **trace** represents the complete journey of an operation.

Example:

```text id="x2aj9s"
Trace ID: 4bf92f3577b34da6...

Customer
   │
   ▼
order-service
   │
   ├── customer-service
   ├── product-service
   ├── pricing-service
   ├── tax-service
   └── payment-service
```

All of these spans belong to one trace.

---

# 4. Span

A span represents one operation inside a trace.

Example:

```text id="k6xw6r"
Trace
│
├── Span: HTTP POST /orders
│
├── Span: customer-service HTTP call
│
├── Span: product-service HTTP call
│
├── Span: PostgreSQL query
│
└── Span: payment provider HTTP call
```

Each span has:

```text id="fj6l0q"
traceId
spanId
parentSpanId
startTime
duration
attributes
status
events
```

---

# 5. Trace vs Span

Think of:

```text id="i8w9kz"
Trace
=
entire journey

Span
=
one step in that journey
```

Example:

```text id="m5k7a4"
Trace
│
└── order-service
      │
      ├── DB query
      ├── Redis lookup
      └── payment-service
             │
             └── external API
```

---

# 6. Parent/Child Relationship

Suppose:

```text id="2q4rpx"
order-service
      │
      ▼
payment-service
```

The trace might look like:

```text id="gk15jm"
Span A
order-service
│
└── Span B
    payment-service
```

Then:

```text id="1u6k4d"
Span A
parent = none

Span B
parent = Span A
```

This creates the call hierarchy.

---

# 7. Example Trace

Imagine a request takes:

```text id="vktd7a"
1.2 seconds
```

Trace:

```text id="3j6hsk"
order-service
├────────────────────────────── 1200ms ──────────────────────┤
│                                                            │
├── customer-service ─ 40ms                                  │
├── product-service ── 50ms                                  │
├── pricing-service ── 90ms                                  │
├── inventory-service ─ 100ms                                │
└── payment-service ───────────────────── 850ms ────────────┤
                                      │
                                      └── provider = 780ms
```

The trace immediately highlights:

```text id="sp4k2w"
payment-service
```

as the major latency contributor.

---

# 8. Distributed Trace Propagation

The trace context must travel between services.

Our target architecture:

```text id="j5l6gr"
order-service
      │
      │ trace context
      ▼
payment-service
      │
      │ trace context
      ▼
fraud-service
```

The standard mechanism is **W3C Trace Context**.

The primary header is:

```http id="4g5s2v"
traceparent
```

Example conceptually:

```http id="z5x3h8"
traceparent: 00-4bf92f3577b34da6-00f067aa0ba902b7-01
```

Applications should use supported tracing libraries/framework instrumentation rather than manually constructing trace headers.

---

# 9. Why W3C Trace Context?

Without a standard:

```text id="2y3c8v"
Service A
  │
  └── custom-header-a
       ↓
Service B
  │
  └── custom-header-b
       ↓
Service C
```

With W3C:

```text id="7xg3v2"
Service A
   │
   ▼
traceparent
   │
   ▼
Service B
   │
   ▼
traceparent
   │
   ▼
Service C
```

This gives interoperability between instrumentation systems.

---

# 10. OpenTelemetry

For modern distributed tracing, OpenTelemetry is the common instrumentation/telemetry standard.

Architecture:

```text id="h8v3cu"
Application
    │
    ▼
OpenTelemetry SDK / instrumentation
    │
    ▼
OTLP
    │
    ▼
OpenTelemetry Collector
    │
    ▼
Trace backend
```

The application does not need to know every detail about the final storage system.

---

# 11. OpenTelemetry Collector

The Collector acts as a telemetry pipeline.

```text id="t2ly6d"
21 Services
    │
    ▼
OTel Collector
    │
    ├── receive
    ├── process
    ├── batch
    ├── sample
    └── export
    │
    ▼
Trace Backend
```

This provides a central place to manage telemetry.

---

# 12. Why Use a Collector?

Without a Collector:

```text id="i6lq0k"
21 Services
   │
   ├──→ Backend
   ├──→ Backend
   ├──→ Backend
   └──→ Backend
```

Every application knows about the backend.

With Collector:

```text id="b3yn5v"
21 Services
      │
      ▼
OTel Collector
      │
      ▼
Trace Backend
```

Benefits include:

```text id="7d0e8b"
Centralized processing
Sampling
Batching
Filtering
Retry
Routing
Backend abstraction
```

---

# 13. Collector Deployment Model

There are multiple deployment patterns.

### Deployment

```text id="kz5m9x"
Services
   ↓
Collector Deployment
   ↓
Trace backend
```

### DaemonSet

```text id="5z4t0p"
Node 1 → Collector
Node 2 → Collector
Node 3 → Collector
```

### Gateway + Agent

For a larger platform:

```text id="9y4r2h"
Application
    │
    ▼
Node/Agent Collector
    │
    ▼
Gateway Collector
    │
    ▼
Trace Backend
```

The exact model depends on traffic volume, network topology, and operational requirements.

---

# 14. Our EKS Architecture

For this project, a practical production-style architecture is:

```text id="7v2e2x"
21 Microservices
       │
       ▼
OTel Collector
       │
       ▼
Trace Backend
       │
       ▼
Grafana
```

Later, if scale requires it:

```text id="c2uk9r"
Application
    ↓
Collector Agent
    ↓
Collector Gateway
    ↓
Trace Backend
```

---

# 15. Trace Backend

OpenTelemetry generates telemetry; it is not itself the long-term trace storage system.

Possible backends include:

```text id="h8ozl0"
Grafana Tempo
Jaeger
AWS X-Ray
Other compatible tracing backends
```

For our project, **Grafana Tempo** is a natural option because Grafana is already our visualization layer.

Architecture:

```text id="1q0b3w"
Application
   ↓
OpenTelemetry
   ↓
Collector
   ↓
Tempo
   ↓
Grafana
```

If the company standardizes on another backend, the Collector allows the export layer to change without redesigning application instrumentation.

---

# 16. Grafana Tracing

Grafana can visualize traces alongside:

```text id="zj6i9y"
Prometheus metrics
Elasticsearch logs
Trace backend
```

This gives:

```text id="qz5m1b"
Metric
  ↓
Trace
  ↓
Logs
```

which is one of the most powerful observability workflows.

---

# 17. Spring Boot Instrumentation

Our services are Spring Boot applications.

There are multiple ways to instrument them:

```text id="9bbn8j"
OpenTelemetry Java Agent
```

or:

```text id="2zmx3p"
OpenTelemetry SDK/libraries
```

For broad automatic instrumentation, the Java agent is often operationally attractive because it can instrument common frameworks without requiring application code changes for every span.

Always pin and test the agent version with the application's JDK and framework versions.

---

# 18. Java Agent Model

Conceptually:

```text id="r3q5t2"
Java Application
       │
       ├── Spring MVC
       ├── JDBC
       ├── HTTP client
       └── messaging
              │
              ▼
       OpenTelemetry Agent
              │
              ▼
            OTLP
```

This can automatically create spans for supported libraries.

---

# 19. Kubernetes Deployment

A conceptual Java deployment:

```yaml id="4ndx5e"
containers:
  - name: order-service
    image: order-service:<IMMUTABLE_TAG>

    env:
      - name: OTEL_SERVICE_NAME
        value: order-service

      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: http://otel-collector.monitoring.svc.cluster.local:4317
```

The exact environment variables depend on the OpenTelemetry deployment and protocol configuration.

Do not copy an example endpoint blindly; verify the actual Collector service and port.

---

# 20. Service Name

Every service needs a stable service identity.

Example:

```text id="mxxmup"
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

Don't use pod names as service names.

Bad:

```text id="2t4xxu"
order-service-7d8c7f8f6f-x9k2p
```

Good:

```text id="p6o8tu"
order-service
```

Pod identity can be a separate resource attribute.

---

# 21. Trace Resource Attributes

Useful attributes include:

```text id="l7tqpi"
service.name
service.version
deployment.environment
k8s.namespace.name
k8s.pod.name
k8s.node.name
cloud.provider
cloud.region
```

Example:

```text id="cvw7gk"
service.name=order-service
service.version=1.5.2
deployment.environment=prod
cloud.region=ap-south-1
```

These make trace filtering much easier.

---

# 22. HTTP Trace

Suppose the customer sends:

```text id="iqr6cf"
POST /api/v1/orders
```

Trace:

```text id="6n80ap"
Trace
│
└── HTTP POST /api/v1/orders
        │
        ├── customer-service
        ├── product-service
        ├── pricing-service
        └── payment-service
```

The incoming request becomes the root server span.

---

# 23. Internal HTTP Span

When:

```text id="b9l2oh"
order-service
      │
      ▼
payment-service
```

the client call creates a child span:

```text id="l2g2y4"
order-service
│
└── HTTP payment-service
       │
       ▼
   payment-service
```

The trace context is propagated automatically by supported instrumentation.

---

# 24. Database Spans

A JDBC database call can become:

```text id="a8d5c9"
order-service
   │
   └── PostgreSQL
        └── SELECT / INSERT
```

Example:

```text id="3s1o8v"
order-service request
      │
      └── SELECT order
           duration=45ms
```

This allows us to identify database latency inside a user request.

---

# 25. Redis Spans

Similarly:

```text id="8t4e3k"
order-service
      │
      └── Redis GET
            4ms
```

If Redis suddenly takes:

```text id="hm6x1y"
500ms
```

the trace will expose it.

---

# 26. External API Spans

Payment service:

```text id="c2p9a4"
payment-service
      │
      └── HTTPS payment-provider
              │
              └── 800ms
```

This can be extremely valuable for distinguishing:

```text id="s0qvbr"
application latency
```

from:

```text id="qjv7px"
external dependency latency
```

---

# 27. Trace Attributes

Useful attributes:

```text id="5xj6v7"
http.request.method
url.path
http.response.status_code
server.address
client.address
db.system
db.namespace
messaging.system
messaging.destination.name
```

But do not indiscriminately record:

```text id="1n8o7a"
password
authorization tokens
full credit-card information
sensitive request bodies
```

Trace attributes are telemetry data and must be treated accordingly.

---

# 28. Sampling

Tracing every request can be expensive.

Suppose:

```text id="4a4f5u"
100,000 requests/sec
```

If every request generates multiple spans:

```text id="1f90vi"
100,000 × several spans
```

can create enormous telemetry volume.

Therefore we need sampling.

---

# 29. Sampling Types

### Head sampling

Decision made near the beginning of the trace.

```text id="7f8m2x"
Request arrives
     ↓
Sample?
 ┌───┴───┐
Yes     No
 │       │
 ▼       ▼
Trace   Drop
```

### Tail sampling

Collector waits until enough trace information is available and then decides.

```text id="w5w6yd"
Complete trace
     ↓
Collector
     ↓
Was it interesting?
 ┌───┴────┐
Yes       No
 │         │
Store     Drop
```

Tail sampling can preserve traces that are more valuable, such as errors or slow requests.

---

# 30. Production Sampling Strategy

Do not simply say:

```text id="x1c9f8"
sample 1%
```

and consider tracing solved.

A more useful strategy may be:

```text id="9x1j3v"
Normal successful requests
        ↓
sample subset

Errors
        ↓
retain

Very slow requests
        ↓
retain

Important business operations
        ↓
higher sampling
```

Exact percentages should be based on traffic, storage, incident requirements, and cost.

---

# 31. Trace Sampling and Errors

Suppose:

```text id="7i4l9g"
1,000,000 successful requests
```

and:

```text id="q3z9az"
500 errors
```

You probably don't want to sample errors at the same rate as normal successful requests.

A useful policy can prioritize:

```text id="d6qg5h"
errors
timeouts
high latency
important business operations
```

---

# 32. Trace-to-Log Correlation

Our logs already contain:

```text id="0m0qk5"
traceId
spanId
correlationId
```

Now:

```text id="s4u8vi"
Grafana
   │
   ▼
Trace
   │
   ▼
span
   │
   ▼
Logs
```

For example:

```text id="m3b6zo"
Trace ID:
4bf92f...

Span:
00f067...

Log:
Database timeout
```

This is extremely useful during incidents.

---

# 33. Trace-to-Metric Correlation

Suppose Grafana shows:

```text id="3o9y5r"
order-service P95 = 2.5 seconds
```

Engineer selects a representative trace.

Trace:

```text id="c4o4n1"
order-service
   │
   ├── customer-service 30ms
   ├── pricing-service 50ms
   └── payment-service 2.3s
```

Now the metric has led directly to the slow dependency.

---

# 34. Metric → Trace → Log

The complete investigation:

```text id="n9a1pz"
Metric
  │
  │ P95 increased
  ▼
Trace
  │
  │ payment-service slow
  ▼
Span
  │
  │ external API = 2.1s
  ▼
Log
  │
  │ provider timeout
  ▼
Root Cause
```

This is the observability workflow we are building.

---

# 35. Asynchronous Tracing

HTTP tracing is relatively straightforward.

Async messaging is more complex.

Example:

```text id="1oh7ru"
order-service
      │
      ▼
Kafka
      │
      ▼
inventory-service
```

The trace context must be propagated across the message boundary.

Conceptually:

```text id="1glq8b"
Producer span
      │
      ▼
Kafka message headers
      │
      ▼
Consumer span
```

---

# 36. Kafka Trace Context

A message may carry W3C trace context in headers.

Conceptually:

```text id="9j8xjz"
Kafka Record
│
├── payload
├── eventId
├── correlationId
└── trace context
```

Consumer instrumentation extracts the context.

Then:

```text id="z0j7h3"
Producer Span
      │
      ▼
Kafka
      │
      ▼
Consumer Span
```

The exact producer/consumer span relationships depend on the instrumentation and messaging model.

---

# 37. SQS Trace Context

Similarly:

```text id="r7f1r3"
Producer
  │
  ▼
SQS
  │
  ▼
Consumer
```

Trace context can be carried through message attributes/metadata.

The consumer creates a new span associated with the producer/message context.

---

# 38. Why Async Tracing Matters

Suppose an order appears stuck.

Trace:

```text id="b5j4yh"
order-service
      │
      ▼
Kafka
      │
      ▼
inventory-service
```

You discover:

```text id="4x6m0v"
inventory consumer
    processing latency = 20 seconds
```

Then:

```text id="h1e7qv"
inventory → PostgreSQL
```

takes:

```text id="y2f7n4"
19.5 seconds
```

Now the root cause is visible.

---

# 39. Messaging Trace Attributes

Useful attributes:

```text id="22spwz"
messaging.system
messaging.destination.name
messaging.operation.type
messaging.message.id
messaging.kafka.partition
messaging.kafka.offset
```

Use only attributes supported by the instrumentation and telemetry conventions being used.

Don't attach huge message payloads to spans.

---

# 40. Don't Put Full Payloads in Traces

Bad:

```json id="7tr0bf"
{
  "span": {
    "requestBody": {
      "customer": "...",
      "payment": "...",
      "entireOrder": "..."
    }
  }
}
```

This can cause:

```text id="4rglv4"
high storage
PII exposure
security risk
large telemetry
```

Better:

```text id="6k6yra"
order.id = ORD-123
event.type = OrderCreated
```

where permitted.

---

# 41. Trace Status

Spans should communicate success/failure.

Example:

```text id="k2o7m1"
payment-service
status = ERROR
```

and an exception event:

```text id="nyh1n7"
TimeoutException
```

This helps trace backends filter failed spans.

---

# 42. Trace Duration

Every span has:

```text id="m29j1w"
start time
end time
duration
```

Example:

```text id="xy0s4f"
payment-service
duration = 842ms
```

Nested:

```text id="5ts3qx"
payment-service
   └── provider API
       duration = 790ms
```

This shows where the time was spent.

---

# 43. Parallel Calls

Suppose order-service calls:

```text id="5wn1ty"
customer-service
product-service
pricing-service
```

in parallel.

Trace:

```text id="1xk8k2"
order-service
│
├──── customer-service ─── 40ms
│
├──── product-service ─── 60ms
│
└──── pricing-service ─── 80ms
```

Total isn't necessarily:

```text id="1h8m5e"
40 + 60 + 80
```

because they may execute concurrently.

Tracing makes this visible.

---

# 44. Sequential Calls

If instead:

```text id="2e1lqt"
order
 ↓
customer
 ↓
product
 ↓
pricing
 ↓
tax
```

then latency accumulates.

Trace:

```text id="a3d6sj"
order-service ─────────────── 500ms
  └─ customer ─ 50ms
      └─ product ─ 80ms
          └─ pricing ─ 120ms
              └─ tax ─ 200ms
```

Tracing can reveal architectural latency amplification.

---

# 45. Trace a Slow Request

Suppose:

```text id="k8e2pl"
POST /api/v1/orders
```

takes:

```text id="j9v0qn"
4.8 seconds
```

Trace:

```text id="sv3k01"
order-service       4.8s
│
├── customer          40ms
├── product           30ms
├── pricing           50ms
├── tax               80ms
├── inventory         200ms
└── payment          4.3s
      │
      └── provider   4.1s
```

Without tracing, an engineer may spend time investigating EKS CPU, Redis, or RDS unnecessarily.

---

# 46. Trace Cardinality

Tracing generates high-cardinality data naturally.

This is okay for trace storage.

But do not automatically copy every trace attribute into Prometheus labels.

Bad:

```text id="ub2d1u"
Prometheus:
trace_id="..."
```

This would create huge metric cardinality.

Keep responsibilities separate:

```text id="qqc8ru"
Metrics → aggregation
Logs → detailed events
Traces → request relationships
```

---

# 47. Trace Backend Storage

Trace retention is usually shorter than logs.

Example:

```text id="0v7y2n"
Metrics: 15-30 days
Logs:    14-30 days
Traces:  3-7 days
```

These are example policies, not universal requirements.

Trace data can be expensive at high volume.

---

# 48. Tempo Architecture

If using Grafana Tempo:

```text id="5g6h4y"
21 Services
      │
      ▼
OpenTelemetry Collector
      │
      ▼
Tempo
      │
      ▼
Grafana
```

Tempo itself requires:

```text id="c1qf0o"
storage
resource sizing
retention
access control
monitoring
```

For larger deployments, object storage is commonly used as part of the architecture.

---

# 49. AWS Architecture

For our EKS project:

```text id="9o7jwu"
AWS
│
├── Route 53
├── CloudFront
├── WAF
├── ALB
│
└── EKS
    │
    ├── 21 Services
    │
    ├── Prometheus
    ├── Grafana
    ├── Alertmanager
    │
    ├── Fluent Bit
    ├── Logstash
    ├── Elasticsearch/OpenSearch
    │
    └── OpenTelemetry Collector
```

Trace storage can be:

```text id="l7qjv1"
Tempo
or
AWS X-Ray
or
another approved backend
```

depending on the production architecture selected.

---

# 50. Network Security

Telemetry endpoints should not be public.

Preferred:

```text id="j3w9xn"
Application Pod
    │
    ▼
Internal Collector Service
    │
    ▼
Trace Backend
```

Use:

```text id="1c4z8d"
NetworkPolicy
Security Groups
TLS
Authentication
```

where applicable.

---

# 51. Resource Requests

Telemetry components require resources.

Example Collector:

```yaml id="bq2e8v"
resources:
  requests:
    cpu: 200m
    memory: 256Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

These are starting examples only.

Actual values depend on:

```text id="v52zbc"
requests/sec
spans/request
sampling
attributes
batching
export rate
```

---

# 52. Collector Batching

Sending every span individually can create unnecessary overhead.

Collector processors can batch telemetry.

Conceptually:

```text id="7t2qbc"
Span
Span
Span
Span
  ↓
Batch
  ↓
Backend
```

This improves network efficiency.

---

# 53. Collector Retry

If the trace backend is temporarily unavailable:

```text id="d4v0z1"
Collector
    ↓
Backend unavailable
```

a retry mechanism can help.

But:

```text id="4x6wqk"
Retry forever
```

is not a solution.

Buffers are finite.

Monitor:

```text id="v80j6f"
queue size
retry count
dropped spans
export errors
```

---

# 54. Sampling + Buffering

Production tracing should balance:

```text id="j88v84"
Observability
+
Cost
+
Reliability
```

Architecture:

```text id="1d9m7w"
Applications
    │
    ▼
Collector
    │
    ├── batch
    ├── retry
    ├── sample
    └── enrich
    │
    ▼
Trace backend
```

---

# 55. Trace Alerts

You don't usually alert on individual trace IDs.

Instead, use metrics derived from tracing or application metrics:

```text id="xqg1e4"
P95 latency
Error rate
Dependency latency
Trace error rate
```

For example:

```text id="k3b8b1"
payment-service
P95 > 2s
```

may trigger an alert.

Then tracing helps investigate.

---

# 56. Metrics and Traces Work Together

```text id="6snr8j"
Prometheus
    │
    └── P95 latency alert
             │
             ▼
          Grafana
             │
             ▼
          Trace
             │
             ▼
      payment-service
             │
             ▼
     external provider
```

This is a much better workflow than expecting engineers to inspect traces continuously.

---

# 57. Trace-to-Log Configuration

The target is:

```text id="w5c7d1"
Trace
  │
  ├── traceId
  ├── spanId
  │
  └── Logs
       ├── traceId
       └── spanId
```

In Grafana, configure trace-to-log correlation so that a trace span can open related Elasticsearch logs.

The exact configuration depends on the Grafana and Elasticsearch/OpenSearch versions and datasource setup.

---

# 58. Log-to-Trace

The reverse is also useful.

Kibana shows:

```text id="0v7m4x"
traceId=4bf92f...
```

Engineer can open Grafana and search that trace.

Eventually:

```text id="b5i8fc"
Kibana
  │
  └── traceId
        ↓
     Grafana
        ↓
      Trace
```

---

# 59. Complete Investigation

Suppose:

```text id="j6l2v1"
Customer checkout is slow.
```

### Step 1 — Grafana

```text id="5r3z1b"
P95 = 3.8s
```

### Step 2 — Trace

```text id="p9x1v4"
payment-service = 3.4s
```

### Step 3 — Span

```text id="4e2g8n"
external payment provider = 3.2s
```

### Step 4 — Logs

```text id="5d7u9c"
Payment provider timeout
```

### Step 5 — Correlation

```text id="8m1c4y"
correlationId=abc123
traceId=4bf92f...
```

Now the incident is tightly scoped.

---

# 60. Trace Sampling Example

Suppose:

```text id="w8x0as"
10,000 requests/sec
```

You might initially sample a subset of successful requests while retaining errors and high-latency traces.

Conceptually:

```text id="5k9j9x"
100% errors
100% very slow requests
higher sampling for critical operations
lower sampling for normal success
```

This is often more useful than a single uniform sampling percentage.

The actual policy should be validated against traffic and storage capacity.

---

# 61. Trace Naming

Use useful span names.

Good:

```text id="n50xqk"
POST /api/v1/orders
GET /products/{id}
payment provider request
SELECT orders
Redis GET
```

Avoid:

```text id="3x7c5m"
HTTP request
operation
database
```

when more specific safe names are available.

Don't put unbounded IDs directly into span names.

Bad:

```text id="6m6q2s"
GET /orders/123456
GET /orders/123457
GET /orders/123458
```

Better:

```text id="90fxp9"
GET /orders/{id}
```

This avoids unnecessary trace aggregation fragmentation.

---

# 62. Error Handling

A failed span should record enough context to investigate.

Example:

```text id="x0r8gk"
Span:
payment provider request

Status:
ERROR

Error:
TimeoutException

Duration:
3000ms
```

Then logs provide detailed context.

---

# 63. Don't Log the Same Exception 10 Times

A single failure may generate:

```text id="a6b1d8"
HTTP layer
Service layer
Repository layer
Global exception handler
```

all logging the same exception.

This creates noise.

Prefer:

```text id="z7m4vx"
one detailed ERROR log
+
context at other layers
```

and let tracing capture the propagation.

---

# 64. Production Failure — Collector Down

Suppose:

```text id="y4y0fw"
OTel Collector
     ↓
DOWN
```

Applications should not normally become unavailable merely because telemetry collection failed.

Design for:

```text id="ax1q1b"
bounded buffers
timeouts
non-blocking telemetry export
drop policy
```

The exact SDK/exporter behavior should be tested under failure.

---

# 65. Production Failure — Trace Backend Down

```text id="3h0j8z"
Application
   ↓
Collector
   ↓
Tempo unavailable
```

Expected:

```text id="j7u5r9"
Collector retries/buffers
        ↓
temporary outage tolerated
```

If the outage continues:

```text id="4f0q0x"
buffer fills
   ↓
some traces dropped
```

That is preferable to:

```text id="d5w6j3"
application requests blocked
```

because observability should not become a hard dependency for application availability.

---

# 66. Monitoring the Tracing Platform

Monitor:

```text id="b0v2mc"
Collector
  ├── CPU
  ├── Memory
  ├── received spans
  ├── exported spans
  ├── dropped spans
  ├── retry count
  └── queue size

Trace backend
  ├── ingestion
  ├── storage
  ├── query latency
  ├── errors
  └── capacity
```

Create a Grafana dashboard:

```text id="qg9qv0"
Tracing Platform
```

---

# 67. 21-Service Tracing

All services should use the same observability standard.

```text id="25m9um"
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

Each service:

```text id="v0l4xn"
service.name
service.version
environment
trace context
correlation ID
```

---

# 68. Service Dependency Graph

Once tracing is enabled, we can derive:

```text id="qz2c8o"
order-service
   ├── customer-service
   ├── product-service
   ├── pricing-service
   ├── promotion-service
   ├── tax-service
   ├── payment-service
   └── inventory-service
```

This becomes a service dependency map.

It helps identify:

```text id="4y6o7m"
high fan-out services
slow dependencies
failure propagation
unexpected dependencies
```

---

# 69. Trace Security

Treat trace data as potentially sensitive.

Don't expose:

```text id="1t7z6a"
authorization headers
cookies
passwords
payment details
full request bodies
```

Use:

```text id="xg7j2k"
allowlists
attribute filtering
redaction
sampling
RBAC
TLS
```

---

# 70. Production Checklist

## Application

* [ ] OpenTelemetry instrumentation
* [ ] Stable service names
* [ ] Service version
* [ ] Environment
* [ ] HTTP tracing
* [ ] DB tracing
* [ ] Redis tracing
* [ ] Messaging tracing
* [ ] External API tracing

## Propagation

* [ ] W3C Trace Context
* [ ] HTTP propagation
* [ ] Kafka propagation
* [ ] SQS propagation
* [ ] RabbitMQ propagation
* [ ] Async context handling

## Collector

* [ ] Deployment model selected
* [ ] Resource limits
* [ ] Batching
* [ ] Retry
* [ ] Sampling
* [ ] NetworkPolicy
* [ ] Monitoring
* [ ] Failure testing

## Backend

* [ ] Trace backend selected
* [ ] Retention
* [ ] Storage
* [ ] Authentication
* [ ] Authorization
* [ ] Monitoring

## Correlation

* [ ] Trace → logs
* [ ] Logs → trace
* [ ] Metrics → trace
* [ ] Correlation ID retained
* [ ] Event ID retained

---

# 71. Final Architecture

```text id="j3w7f0"
                           USERS
                             │
                             ▼
                         CloudFront
                             │
                             ▼
                            ALB
                             │
                             ▼
                    ┌─────────────────┐
                    │       EKS       │
                    │                 │
                    │ 21 Microservices│
                    │                 │
                    └───────┬─────────┘
                            │
          ┌─────────────────┼──────────────────┐
          │                 │                  │
          ▼                 ▼                  ▼
       Metrics             Logs              Traces
          │                 │                  │
          ▼                 ▼                  ▼
     Prometheus         Fluent Bit       OpenTelemetry
          │                 │                  │
          ▼                 ▼                  ▼
       Grafana           Logstash          Collector
                            │                  │
                            ▼                  ▼
                     Elasticsearch          Tempo
                            │                  │
                            ▼                  │
                         Kibana ◄──────────────┘
                            │
                            ▼
                     Incident Response
```

---

# 72. Complete Investigation Workflow

We have now built the foundation for this workflow:

```text id="1f3s3z"
             CUSTOMER REPORT
                    │
                    ▼
              Grafana Metrics
                    │
                    ▼
             Error/Latency Spike
                    │
                    ▼
                Trace ID
                    │
                    ▼
             Distributed Trace
                    │
             ┌──────┴──────┐
             ▼             ▼
       Slow Service      Failed Span
             │             │
             └──────┬──────┘
                    ▼
                Trace ID
                    │
                    ▼
                  Logs
                    │
                    ▼
             Correlation ID
                    │
                    ▼
             Event / Message
                    │
                    ▼
               Dependency
                    │
                    ▼
                Root Cause
```

This is the target production observability workflow.

---

# Next Chapter

## `124-OpenTelemetry-Production-Implementation.md`

The next chapter goes deeper into the **actual OpenTelemetry platform implementation**, including:

```text
21 Spring Boot Services
        │
        ▼
OpenTelemetry Java Agent
        │
        ▼
OTLP
        │
        ▼
OpenTelemetry Collector
        │
   ┌────┼─────┐
   ▼    ▼     ▼
 Batch Sample Filter
        │
        ▼
     Tempo
        │
        ▼
     Grafana
```

We will cover the production installation and configuration of the Collector, Java instrumentation, OTLP protocols, Collector pipelines, processors, resource detection, Kubernetes metadata, sampling, security, HA, scaling, persistent buffering, trace-to-log correlation, and a complete end-to-end trace through the 21-service platform.
