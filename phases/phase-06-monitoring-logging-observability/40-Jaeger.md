# `40-Jaeger.md`

````markdown
# 40. Jaeger — Distributed Tracing in Production on AWS EKS

## 1. Why Jaeger Exists

In our e-commerce platform, a single customer request can travel through many microservices.

For example:

```text
Customer
   |
   v
CloudFront / WAF
   |
   v
ALB
   |
   v
auth-service
   |
   v
product-service
   |
   v
inventory-service
   |
   v
pricing-service
   |
   v
order-service
   |
   +----> payment-service
   |
   +----> fraud-service
   |
   +----> notification-service
   |
   v
RDS PostgreSQL
````

Suppose an order request takes 8 seconds.

Application logs may show:

```text
order-service request started
payment-service request started
fraud-service request started
inventory-service request started
database query started
```

But logs alone make it difficult to answer:

* Which service caused the delay?
* Which downstream call took the most time?
* Which database query was slow?
* Where did the request fail?
* Did the payment service call complete?
* Did the SQS message get published?
* Which services participated in this request?

Distributed tracing solves this.

Jaeger allows us to follow a single request across multiple services.

---

# 2. What Distributed Tracing Means

A distributed trace represents the complete journey of one request.

Example:

```text
Trace ID:
4bf92f3577b34da6a3ce929d0e0e4736
```

The request creates multiple spans.

```text
Trace
|
+-- ALB -> auth-service
|     120 ms
|
+-- auth-service -> product-service
|     80 ms
|
+-- product-service -> inventory-service
|     500 ms
|
+-- inventory-service -> PostgreSQL
|     450 ms
|
+-- product-service -> pricing-service
|     70 ms
|
+-- order-service
      300 ms
```

Jaeger can visualize this as a timeline.

The important concept is:

```text
Trace
  |
  +-- Span
       |
       +-- Child Span
            |
            +-- Child Span
```

---

# 3. Jaeger vs OpenTelemetry

This distinction is extremely important.

## OpenTelemetry

OpenTelemetry is responsible for:

```text
Instrumentation
      |
      v
Generate telemetry
      |
      v
Collect telemetry
      |
      v
Process telemetry
      |
      v
Export telemetry
```

## Jaeger

Jaeger is primarily used for:

```text
Trace collection/storage/querying/visualization
```

In our architecture:

```text
Spring Boot Application
        |
        | OTLP
        v
OpenTelemetry Collector
        |
        | OTLP
        v
Jaeger
        |
        v
Storage
        |
        v
Jaeger UI
```

Therefore:

```text
OpenTelemetry = telemetry framework
Jaeger        = tracing backend/UI
```

They are complementary, not competing products.

---

# 4. Production Architecture

Our production tracing architecture looks like:

```text
                    Internet
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
                 Microservices
                       |
              +--------+--------+
              |        |        |
              v        v        v
           auth     catalog    order
              |        |        |
              +--------+--------+
                       |
                       v
             OpenTelemetry SDK/
                Java Agent
                       |
                       | OTLP
                       v
             OpenTelemetry Collector
                  /           \
                 /             \
                v               v
          Metrics backend     Jaeger
                                  |
                                  v
                           Elasticsearch
                                  |
                                  v
                              Jaeger UI
```

At the same time:

```text
Application
     |
     +---- Metrics ----> Prometheus ----> Grafana
     |
     +---- Logs -------> Fluent Bit
     |                       |
     |                       v
     |                   Logstash
     |                       |
     |                       v
     |                  Elasticsearch
     |                       |
     |                       v
     |                    Kibana
     |
     +---- Traces -----> OTel Collector
                             |
                             v
                           Jaeger
```

This gives us three observability signals:

```text
Metrics
   |
   +--> What is happening?

Logs
   |
   +--> What happened?

Traces
   |
   +--> Where did the request travel?
```

---

# 5. Jaeger Components

A Jaeger deployment commonly contains these logical components:

```text
Application
    |
    v
Jaeger Collector
    |
    v
Storage
    |
    v
Jaeger Query
    |
    v
Jaeger UI
```

Depending on the Jaeger version and deployment architecture, components and deployment patterns can differ.

Always use the deployment model supported by the exact Jaeger version selected for the platform.

Do not blindly copy manifests from an old Jaeger tutorial.

---

# 6. Jaeger Collector

The collector receives tracing data.

Example:

```text
OTel Collector
      |
      | OTLP
      v
Jaeger Collector
```

The collector then writes trace data to the configured storage backend.

Production requirements include:

* multiple replicas
* resource requests/limits
* PodDisruptionBudget
* topology spread
* internal Kubernetes Service
* TLS where required
* NetworkPolicy
* monitoring
* graceful shutdown

---

# 7. Jaeger Query

Jaeger Query reads trace information from storage.

```text
User
 |
 v
Jaeger UI
 |
 v
Jaeger Query
 |
 v
Storage
```

Query should normally be deployed as multiple replicas in production.

Example:

```text
jaeger-query-0
jaeger-query-1
```

Both can serve requests through:

```text
jaeger-query.monitoring.svc.cluster.local
```

or another dedicated tracing namespace/service.

---

# 8. Jaeger UI

The UI allows engineers to search and inspect traces.

Typical workflow:

```text
Open Jaeger UI
      |
      v
Select service
      |
      v
Select operation
      |
      v
Select time range
      |
      v
Find trace
      |
      v
Open trace
      |
      v
Inspect spans
```

Example:

```text
Service:
order-service

Operation:
POST /api/v1/orders

Duration:
4.8 seconds
```

We can then inspect the complete trace.

---

# 9. Trace ID

Every distributed trace gets a unique Trace ID.

Example:

```text
trace_id=4bf92f3577b34da6a3ce929d0e0e4736
```

All spans belonging to the same request share this Trace ID.

Example logs:

```text
2026-09-11 09:10:01 INFO
order request received
trace_id=abc123
```

Downstream:

```text
payment request started
trace_id=abc123
```

Another service:

```text
fraud check started
trace_id=abc123
```

This allows us to correlate logs and traces.

---

# 10. Span

A span represents one unit of work.

For example:

```text
POST /api/v1/orders
```

may produce:

```text
Span: order-service HTTP request
```

Then:

```text
Span: payment-service HTTP request
```

Then:

```text
Span: PostgreSQL query
```

A span can contain:

* operation name
* start time
* duration
* status
* attributes
* events
* parent span ID
* trace ID
* errors

---

# 11. Parent and Child Spans

Example:

```text
order-service
|
+-- POST /orders
      |
      +-- inventory-service
      |
      +-- payment-service
      |      |
      |      +-- PostgreSQL
      |
      +-- notification-service
```

The parent-child relationship allows Jaeger to reconstruct the request tree.

---

# 12. W3C Trace Context

Distributed tracing requires context propagation.

Modern OpenTelemetry deployments commonly use W3C Trace Context.

Example HTTP header:

```text
traceparent:
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

The downstream service extracts this context.

```text
Service A
   |
   | traceparent
   v
Service B
   |
   | traceparent
   v
Service C
```

Without propagation:

```text
Service A -> Trace A

Service B -> Trace B

Service C -> Trace C
```

The distributed trace becomes disconnected.

With proper propagation:

```text
Trace A
 |
 +-- Service A
 |
 +-- Service B
 |
 +-- Service C
```

---

# 13. OpenTelemetry Collector to Jaeger

Our preferred architecture is:

```text
Applications
     |
     | OTLP
     v
OpenTelemetry Collector
     |
     | OTLP
     v
Jaeger
```

The application should not need to know the Jaeger storage implementation.

This creates a useful abstraction:

```text
Application
     |
     v
OTel Collector
     |
     +----> Jaeger
     |
     +----> other observability backend
```

If the backend changes later, application configuration can remain stable.

---

# 14. OTLP Ports

Common OTLP endpoints are:

```text
4317 -> OTLP/gRPC

4318 -> OTLP/HTTP
```

For example:

```text
otel-collector.monitoring.svc.cluster.local:4317
```

Applications can export traces to the Collector using OTLP.

---

# 15. Jaeger Storage

Jaeger requires a storage backend.

The appropriate backend depends on the Jaeger version and deployment architecture.

For our project, Elasticsearch is especially relevant because we already operate Elasticsearch for centralized logging.

Conceptually:

```text
Jaeger
  |
  v
Elasticsearch
```

However, do not automatically assume that sharing the same Elasticsearch cluster is always the right production decision.

Tracing can generate significant write volume.

Therefore we must evaluate:

* trace volume
* retention
* storage capacity
* shard count
* query load
* log volume
* recovery requirements
* failure domains

---

# 16. Shared Elasticsearch vs Dedicated Elasticsearch

## Option A — Shared Elasticsearch

```text
                  Elasticsearch
                  /            \
                 /              \
                v                v
             Logs             Traces
```

Advantages:

* fewer infrastructure components
* simpler learning environment
* lower operational overhead

Disadvantages:

* logs can compete with traces
* trace ingestion can affect logging
* retention requirements may differ
* noisy-neighbor risk

---

## Option B — Dedicated Elasticsearch

```text
Logstash
   |
   v
Logging Elasticsearch


Jaeger
   |
   v
Tracing Elasticsearch
```

Advantages:

* workload isolation
* independent sizing
* independent retention
* independent failure domain

Disadvantages:

* higher cost
* more infrastructure
* more operational work

For a larger production platform, dedicated capacity may be preferable when observability workloads are substantial.

---

# 17. Elasticsearch Capacity Planning

If Jaeger uses Elasticsearch, trace volume must be estimated.

Example:

```text
Requests/sec = 500
Average spans/request = 8

Approximate spans/sec:

500 × 8 = 4,000 spans/sec
```

If:

```text
4,000 spans/sec × 86,400 sec
```

then daily span volume is approximately:

```text
345,600,000 spans/day
```

That is a significant amount of data.

Therefore tracing should not automatically be configured to capture every request indefinitely.

---

# 18. Sampling

Sampling is one of the most important production tracing concepts.

Suppose:

```text
10,000 requests/sec
```

and every request generates:

```text
10 spans
```

Then:

```text
100,000 spans/sec
```

This can become extremely expensive.

Sampling reduces the volume.

---

# 19. Head Sampling

Head sampling decides early whether a trace should be sampled.

Example:

```text
100 requests

10 selected for tracing
```

Sampling rate:

```text
10%
```

Advantages:

* simple
* reduces data early
* reduces downstream processing

Disadvantage:

A trace may be discarded before we know whether it becomes interesting.

---

# 20. Tail Sampling

Tail sampling waits until enough information about the trace is available.

For example:

```text
Normal request
   |
   v
Low priority

Failed request
   |
   v
Keep trace

Slow request
   |
   v
Keep trace
```

This is particularly useful for production.

Example policy:

```text
Errors:
100%

Slow requests:
100%

Normal requests:
5%
```

The exact policy must be based on traffic and capacity.

---

# 21. Why Error Traces Matter

Suppose normal traffic is:

```text
200 ms
```

but one request takes:

```text
12 seconds
```

and fails.

That trace is extremely valuable.

Sampling policies should prioritize:

* errors
* high latency
* important business operations
* selected customers/workflows where appropriate
* important payment/order failures

Avoid tracing sensitive customer information merely because it is useful for debugging.

---

# 22. Jaeger UI — Searching Traces

Typical search fields include:

```text
Service
Operation
Tags
Duration
Time range
Trace ID
```

Example:

```text
Service:
order-service

Operation:
POST /api/v1/orders

Duration:
> 2s
```

This can identify slow order requests.

---

# 23. Inspecting a Trace

Suppose Jaeger shows:

```text
order-service
  5.8s
    |
    +-- inventory-service
    |      200ms
    |
    +-- payment-service
    |      5.1s
    |        |
    |        +-- payment DB
    |             4.8s
    |
    +-- notification-service
           100ms
```

The investigation immediately focuses on:

```text
payment-service
      |
      v
payment DB
```

Instead of investigating all 21 services.

This is one of the biggest benefits of distributed tracing.

---

# 24. Trace-to-Log Correlation

Our logging architecture should include trace identifiers.

Example structured log:

```json
{
  "timestamp": "2026-09-11T09:10:10Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace.id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span.id": "00f067aa0ba902b7",
  "message": "Payment gateway timeout"
}
```

Now the engineer can:

```text
Jaeger
   |
   | trace.id
   v
Kibana
   |
   v
All related logs
```

This is extremely useful during incidents.

---

# 25. Trace-to-Metrics Correlation

Suppose Grafana shows:

```text
order-service
p95 latency = 4.5s
```

We investigate the service.

Then:

```text
Grafana
   |
   v
High latency
   |
   v
Jaeger
   |
   v
Slow trace
   |
   v
payment-service
   |
   v
Kibana
   |
   v
payment timeout logs
```

This creates the complete observability workflow.

---

# 26. Service Dependency Graph

Distributed traces can help identify service relationships.

Example:

```text
auth-service
      |
      +----> user-service
      |
      +----> customer-service

order-service
      |
      +----> inventory-service
      +----> pricing-service
      +----> payment-service
      +----> fraud-service
      +----> notification-service
```

This helps engineers understand:

* dependencies
* high-traffic services
* bottlenecks
* failure propagation
* unexpected dependencies

---

# 27. HTTP Tracing

For Spring Boot services:

```text
order-service
     |
     | HTTP
     v
payment-service
```

OpenTelemetry can capture:

```text
HTTP method
HTTP route
status code
duration
peer service
trace ID
span ID
```

Avoid storing sensitive request/response bodies by default.

For example, do not blindly capture:

```text
credit card number
password
authorization token
customer secrets
```

---

# 28. Database Tracing

Tracing can also show database activity.

Example:

```text
order-service
    |
    +-- SQL query
          |
          +-- PostgreSQL
```

A trace might reveal:

```text
HTTP request: 3.2s

Database query: 2.9s
```

Now we know that application processing is not necessarily the bottleneck.

The database may be the real issue.

---

# 29. Messaging Tracing

Tracing becomes even more valuable with asynchronous systems.

Example:

```text
order-service
      |
      v
SQS
      |
      v
payment-service
```

The tracing context should be propagated through supported messaging instrumentation.

Conceptually:

```text
Trace
 |
 +-- HTTP POST /orders
 |
 +-- publish order event
 |
 +-- consume order event
 |
 +-- payment processing
```

Messaging tracing requires correct context propagation and instrumentation.

Do not assume an HTTP trace automatically continues across an asynchronous boundary without proper instrumentation.

---

# 30. Java Instrumentation

For Spring Boot services, OpenTelemetry can be introduced through:

1. OpenTelemetry SDK
2. OpenTelemetry Java agent
3. Spring/OpenTelemetry integrations

The Java agent is often attractive because it can automatically instrument common libraries.

Conceptually:

```text
Container
|
+-- Java application
|
+-- OpenTelemetry Java Agent
```

The agent produces telemetry and exports it to:

```text
OTel Collector
```

---

# 31. Recommended Application Pattern

For the 21 microservices:

```text
Microservice
     |
     v
OpenTelemetry Java instrumentation
     |
     v
OTLP
     |
     v
OTel Collector
     |
     v
Jaeger
```

Standardize:

* service name
* environment
* region
* namespace
* cluster
* version

Example:

```text
service.name=order-service
deployment.environment=prod
cloud.region=ap-south-1
k8s.namespace.name=production
```

---

# 32. Service Naming

Consistent service names are critical.

Use:

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
orders-api
orderapp
orderservice
order-service-prod
```

for the same service.

Standard naming makes dashboards and traces much easier to use.

---

# 33. Kubernetes Deployment

Jaeger should run as platform infrastructure rather than inside application Pods.

Example namespaces:

```text
production
monitoring
logging
tracing
argocd
ingress-system
```

A dedicated namespace is useful:

```text
tracing
```

Example:

```bash
kubectl create namespace tracing
```

In a GitOps environment, however, the namespace and deployment should ultimately be managed through Git.

---

# 34. Helm/GitOps Installation

Production installation should follow:

```text
Git
 |
 v
Helm values
 |
 v
ArgoCD
 |
 v
EKS
 |
 v
Jaeger
```

Do not rely on:

```text
Engineer laptop
      |
      v
kubectl apply
```

for repeatable production platform deployment.

---

# 35. GitOps Structure

Example:

```text
gitops/
├── applications/
│   └── jaeger.yaml
│
├── environments/
│   ├── dev/
│   │   └── tracing/
│   │
│   ├── qa/
│   │   └── tracing/
│   │
│   └── prod/
│       └── tracing/
│
└── helm/
    └── jaeger/
```

The exact structure can differ according to the team's GitOps convention.

---

# 36. Example Jaeger Application

Conceptual ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: jaeger
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    path: environments/prod/tracing/jaeger
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: tracing
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

For production, synchronization policy should follow the platform team's approval model.

Do not blindly enable automatic production changes for every platform component.

---

# 37. High Availability

A production Jaeger deployment should avoid a single replica where availability matters.

Conceptually:

```text
                 Load Balancer / Service
                         |
              +----------+----------+
              |                     |
              v                     v
        Jaeger Collector 1   Jaeger Collector 2
              |                     |
              +----------+----------+
                         |
                         v
                      Storage
```

Query:

```text
Jaeger UI
    |
    v
Service
 /    \
v      v
Query1 Query2
```

Use:

* multiple replicas
* PodDisruptionBudget
* topology spread
* anti-affinity where appropriate
* resource requests/limits
* graceful termination

---

# 38. PodDisruptionBudget

Example concept:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: jaeger-query
```

The exact selector and deployment structure must match the selected Jaeger deployment.

---

# 39. Multi-AZ Placement

Our EKS production cluster spans:

```text
AZ-A
AZ-B
AZ-C
```

Tracing components should avoid unnecessary concentration in one AZ.

Conceptually:

```text
AZ-A             AZ-B             AZ-C

Collector-1      Collector-2      Collector-3
Query-1          Query-2
```

Kubernetes topology constraints can help distribute Pods.

---

# 40. Resource Requests and Limits

Tracing components require capacity planning.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"
```

These are examples, not universal production values.

Actual values depend on:

```text
requests/sec
spans/request
sampling rate
attribute size
retention
storage performance
query workload
```

---

# 41. Collector Backpressure

Suppose applications generate:

```text
20,000 spans/sec
```

but Jaeger accepts only:

```text
10,000 spans/sec
```

Then the pipeline can experience backpressure.

```text
Applications
      |
      v
OTel Collector
      |
      X
Jaeger overloaded
```

The Collector must be configured with appropriate:

* batching
* memory limits
* queues
* retry behavior
* timeout
* sampling

The goal is to prevent tracing from destabilizing applications.

---

# 42. Observability Must Not Become an Outage Source

This is a critical production principle.

Bad design:

```text
Application
   |
   | synchronous dependency
   v
Tracing backend
```

If Jaeger goes down:

```text
Jaeger DOWN
     |
     v
Application DOWN
```

That is unacceptable.

The application should be able to continue serving traffic even if the tracing backend becomes temporarily unavailable.

Telemetry pipelines should be designed for graceful degradation.

---

# 43. NetworkPolicy

Tracing components should not be reachable from everywhere.

Example conceptual flow:

```text
production namespace
        |
        | OTLP
        v
OTel Collector
        |
        | OTLP
        v
Jaeger
        |
        v
Elasticsearch
```

NetworkPolicy should allow only required paths.

For example:

```text
production -> OTel Collector
OTel Collector -> Jaeger
Jaeger -> Elasticsearch
users/admins -> Jaeger Query/UI
```

Avoid:

```text
* -> Jaeger
```

---

# 44. Jaeger UI Security

The UI should generally be treated as an internal platform tool.

Possible access model:

```text
Engineer
   |
   v
SSO / Identity Provider
   |
   v
Internal ALB
   |
   v
Jaeger UI
```

Avoid exposing the tracing UI directly to the public Internet unless there is a specific, reviewed architecture for doing so.

---

# 45. TLS

Production environments should use encryption for sensitive telemetry paths where appropriate.

Potential connections:

```text
Application
    |
    | TLS
    v
OTel Collector
    |
    | TLS
    v
Jaeger
    |
    | TLS
    v
Elasticsearch
```

Certificates and credentials should be managed through the platform's approved secret-management mechanism.

Do not hardcode:

```text
password: admin123
```

inside Git.

---

# 46. RBAC

Jaeger and the surrounding components should use minimal Kubernetes permissions.

Example principle:

```text
ServiceAccount
      |
      v
Only required Kubernetes resources
```

Do not give:

```text
cluster-admin
```

unless there is a genuine requirement.

---

# 47. Trace Attributes

Useful attributes include:

```text
service.name
service.version
deployment.environment
k8s.namespace.name
k8s.pod.name
k8s.node.name
http.request.method
http.response.status_code
server.address
db.system
messaging.system
```

But avoid uncontrolled attributes.

Bad example:

```text
customer_email
full_request_body
credit_card_number
authorization_header
```

High-cardinality and sensitive attributes can create both security and storage problems.

---

# 48. Trace Cardinality

Suppose we create an attribute:

```text
customer_id
```

with millions of unique values.

This can dramatically increase the number of unique combinations in observability systems.

Prefer bounded attributes:

```text
service.name
operation
status
region
environment
```

Use high-cardinality identifiers such as order IDs primarily where they are operationally useful and supported by the storage/query model.

---

# 49. Business Correlation

For an e-commerce platform, a trace can be correlated with:

```text
order_id
payment_id
shipment_id
```

But these should be handled carefully.

A useful model is:

```text
Trace ID
   |
   +-- Order ID
   |
   +-- Payment ID
```

This lets engineers search an incident without putting sensitive customer data into traces.

---

# 50. Trace Sampling Strategy for E-Commerce

A possible production strategy:

```text
Normal requests:
5%

Errors:
100%

Slow requests:
100%

Payment operations:
higher sampling

Checkout/order operations:
higher sampling
```

This is illustrative.

Sampling should be based on real traffic and business importance.

---

# 51. Retention

Do not retain every trace forever.

Example conceptual policy:

```text
Hot traces:
7 days

Longer-term:
only selected/important data
```

Actual retention depends on:

* incident requirements
* compliance
* storage cost
* trace volume
* debugging requirements

Logs, metrics, and traces often need different retention periods.

---

# 52. Elasticsearch and Jaeger Retention

If Elasticsearch stores traces:

```text
Jaeger
   |
   v
Elasticsearch
   |
   +-- Trace indexes/data streams
   |
   +-- Retention policy
   |
   +-- Snapshots
```

Storage planning should consider:

```text
primary shards
replica shards
segment overhead
index metadata
filesystem cache
snapshot requirements
```

Avoid creating excessive tiny shards.

---

# 53. Backup and Recovery

Tracing data is usually less critical than transactional business data.

Compare:

```text
RDS
 |
 +-- Business source of truth
 |
 +-- Strong recovery requirements
```

versus:

```text
Jaeger
 |
 +-- Observability data
 |
 +-- Usually lower recovery priority
```

Therefore, the RPO/RTO requirements can differ.

However, if traces are required for compliance or forensic purposes, retention and backup requirements must be adjusted accordingly.

---

# 54. Failure Scenario — No Traces

Application is healthy:

```text
HTTP 200
```

but Jaeger shows nothing.

Troubleshoot:

```text
Application
    |
    v
Instrumentation
    |
    v
OTLP export
    |
    v
Collector
    |
    v
Jaeger
    |
    v
Storage
    |
    v
Query
    |
    v
UI
```

Check from left to right.

---

# 55. Check Application Instrumentation

Check:

```text
service.name
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_EXPORTER_OTLP_PROTOCOL
```

For example:

```bash
kubectl -n production exec deploy/order-service -- env | grep OTEL
```

Then inspect logs:

```bash
kubectl -n production logs deploy/order-service
```

Look for exporter errors.

---

# 56. Check OTel Collector

Check Pods:

```bash
kubectl -n monitoring get pods
```

Check logs:

```bash
kubectl -n monitoring logs deploy/otel-collector
```

Check Service:

```bash
kubectl -n monitoring get svc
```

Check endpoints:

```bash
kubectl -n monitoring get endpoints
```

or the corresponding EndpointSlice resources.

---

# 57. Check OTLP Connectivity

From an appropriate diagnostic environment:

```bash
nc -vz otel-collector.monitoring.svc.cluster.local 4317
```

If connectivity fails, investigate:

```text
Service
DNS
NetworkPolicy
Port
TLS
routing
```

---

# 58. Check Jaeger

Check:

```bash
kubectl -n tracing get pods
```

Then:

```bash
kubectl -n tracing logs <jaeger-collector-pod>
```

Look for:

```text
storage errors
connection errors
authentication errors
TLS errors
```

---

# 59. Check Elasticsearch

If Elasticsearch is the storage backend:

```bash
kubectl -n logging get pods
```

Check cluster health through the approved administrative interface.

Potential problems:

```text
red cluster
unassigned shards
disk watermark
insufficient storage
high JVM pressure
network failures
authentication failure
```

---

# 60. Jaeger UI Has No Data

If:

```text
Collector receives spans
```

but:

```text
Jaeger UI = no traces
```

then investigate:

```text
Collector
    |
    v
Storage
    |
    v
Query
    |
    v
UI
```

The problem may be in storage indexing or Query rather than application instrumentation.

---

# 61. Slow Jaeger UI

Possible causes:

```text
Huge trace
Large search window
Too many traces
Large Elasticsearch indexes
Poor shard design
High Elasticsearch load
Insufficient Query resources
```

First narrow the query:

```text
Service
Operation
Time range
Duration
Tags
```

Then inspect storage performance.

---

# 62. Missing Downstream Span

Suppose:

```text
order-service
   |
   +-- inventory-service
```

but the inventory span is missing.

Investigate:

```text
HTTP client instrumentation
        |
        v
Context propagation
        |
        v
inventory-service instrumentation
```

Common causes:

* unsupported client library
* instrumentation disabled
* context lost
* custom thread/executor handling
* asynchronous code
* messaging context propagation problem
* incorrect instrumentation configuration

---

# 63. Trace IDs Not Matching Logs

Suppose Jaeger shows:

```text
trace_id=abc123
```

but Kibana logs show:

```text
trace.id=xyz789
```

Investigate:

```text
trace context propagation
logging MDC integration
OpenTelemetry logging instrumentation
structured logging configuration
```

The goal is:

```text
Jaeger trace ID
        =
Kibana trace.id
```

for the same operation.

---

# 64. Production Incident Example

Customer reports:

```text
Checkout is slow.
```

Grafana:

```text
order-service p95:
800ms -> 5.2s
```

Open Jaeger.

Find:

```text
order-service
   |
   +-- payment-service
          |
          +-- payment gateway
                 4.4s
```

Open Kibana.

Search:

```text
trace.id=<trace-id>
```

Find:

```text
payment gateway timeout
```

Now the investigation is:

```text
Customer
   |
   v
Order latency
   |
   v
Jaeger
   |
   v
Payment service
   |
   v
Kibana
   |
   v
Gateway timeout
```

Without distributed tracing, engineers might spend hours checking unrelated services.

---

# 65. Jaeger vs Logs

Logs answer:

```text
What happened?
```

Example:

```text
Payment gateway timeout.
```

Tracing answers:

```text
Where did the request spend its time?
```

Example:

```text
Order request
 |
 +-- auth: 50ms
 +-- inventory: 100ms
 +-- payment: 4.8s
```

Both are required.

---

# 66. Jaeger vs Prometheus

Prometheus:

```text
order-service request duration
```

might tell us:

```text
p95 = 5 seconds
```

Jaeger tells us:

```text
Why this particular request took 5 seconds.
```

Therefore:

```text
Prometheus -> aggregate behavior
Jaeger     -> individual request path
```

---

# 67. Complete Observability Workflow

A production engineer should be able to move between all three.

```text
                    Grafana
                       |
                       | High latency
                       v
                     Jaeger
                       |
                       | Trace ID
                       v
                     Kibana
                       |
                       | Logs
                       v
                 Root cause
```

Example:

```text
Grafana:
payment-service p95 increased

        ↓

Jaeger:
payment DB span = 4.5s

        ↓

Kibana:
database timeout / connection pool exhaustion

        ↓

RDS:
DatabaseConnections high
```

This is real observability.

---

# 68. Jaeger Monitoring

Monitor the tracing system itself.

Useful metrics include:

```text
collector received spans
collector exported spans
export failures
queue size
dropped spans
collector CPU
collector memory
query request latency
query errors
storage errors
Elasticsearch health
```

Never assume observability infrastructure does not need monitoring.

---

# 69. Alerting

Possible alerts:

```text
Jaeger Collector export failures high

Jaeger Collector dropped spans high

Jaeger Query error rate high

Jaeger Query latency high

Tracing storage unhealthy

Tracing Elasticsearch disk usage high
```

But alerts should represent actionable problems.

---

# 70. Kubernetes Tool Placement

| Component                | Production Location                                |
| ------------------------ | -------------------------------------------------- |
| OpenTelemetry Java Agent | Application container                              |
| OpenTelemetry SDK        | Application                                        |
| OpenTelemetry Collector  | EKS platform                                       |
| Jaeger Collector         | EKS tracing/platform                               |
| Jaeger Query             | EKS tracing/platform                               |
| Jaeger UI                | EKS tracing/platform                               |
| Elasticsearch            | EKS logging/tracing platform or managed service    |
| Kibana                   | EKS logging/platform                               |
| Prometheus               | EKS monitoring                                     |
| Grafana                  | EKS monitoring/platform                            |
| Fluent Bit               | EKS nodes as DaemonSet                             |
| Logstash                 | EKS logging                                        |
| RDS                      | AWS managed                                        |
| Redis/Valkey             | AWS managed preferred for production when suitable |
| SQS/SNS                  | AWS managed                                        |
| ECR                      | AWS managed                                        |
| ArgoCD                   | EKS platform                                       |

The key principle:

```text
Application containers
    |
    +-- Application code
    +-- Application instrumentation

Platform
    |
    +-- OTel Collector
    +-- Jaeger
    +-- Prometheus
    +-- Grafana
    +-- Logging stack
```

---

# 71. Terraform vs ArgoCD Boundary

Terraform should generally manage AWS infrastructure such as:

```text
VPC
EKS
RDS
ElastiCache
ECR
IAM
S3
SQS
SNS
KMS
```

ArgoCD should manage Kubernetes workloads such as:

```text
Jaeger
OpenTelemetry Collector
Prometheus
Grafana
Fluent Bit
Logstash
Elasticsearch
Kibana
Applications
```

Therefore:

```text
Terraform
    |
    v
AWS infrastructure

ArgoCD
    |
    v
Kubernetes desired state
```

Avoid having both systems manage the same resource.

---

# 72. Production Deployment Flow

The complete platform flow becomes:

```text
Developer
   |
   v
GitHub
   |
   v
CI
   |
   +-- Build
   +-- Test
   +-- SonarQube
   +-- Security scan
   +-- Docker
   +-- Trivy
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
EKS
   |
   +-------------------+
   |                   |
   v                   v
Application          Platform
Pods                 Components
   |                   |
   |                   +-- Prometheus
   |                   +-- Grafana
   |                   +-- Alertmanager
   |                   +-- OTel Collector
   |                   +-- Jaeger
   |                   +-- Fluent Bit
   |                   +-- Logstash
   |                   +-- Elasticsearch
   |                   +-- Kibana
   |
   +---- Metrics ------> Prometheus
   |
   +---- Logs ---------> Fluent Bit -> Logstash -> Elasticsearch -> Kibana
   |
   +---- Traces -------> OTel Collector -> Jaeger -> Storage
```

---

# 73. End-to-End Incident Investigation

Suppose checkout latency increases.

## Step 1 — Grafana

Check:

```text
request rate
error rate
p50
p95
p99
CPU
memory
```

Find:

```text
payment-service p99 increased
```

## Step 2 — Jaeger

Search:

```text
payment-service
```

Find:

```text
POST /payments
```

Duration:

```text
5.7s
```

Inspect spans.

Find:

```text
database = 5.1s
```

## Step 3 — Kibana

Search:

```text
trace.id=<trace-id>
```

Find:

```text
connection pool timeout
```

## Step 4 — RDS

Check:

```text
DatabaseConnections
CPU
FreeableMemory
latency
IOPS
locks
```

Now the investigation has moved from:

```text
Customer complaint
```

to:

```text
payment-service
      |
      v
database
      |
      v
connection pool
      |
      v
RDS
```

That is the real purpose of observability.

---

# 74. Production Checklist

## Jaeger

* [ ] Supported Jaeger version selected
* [ ] Deployment method matches selected version
* [ ] Multiple Collector replicas
* [ ] Multiple Query replicas
* [ ] Resources configured
* [ ] PDB configured
* [ ] Multi-AZ placement
* [ ] Internal access only
* [ ] TLS configured where required
* [ ] Authentication configured
* [ ] RBAC configured
* [ ] NetworkPolicy configured
* [ ] Storage configured
* [ ] Retention configured
* [ ] Sampling configured
* [ ] Monitoring configured
* [ ] Backup requirements defined
* [ ] Recovery procedure documented

## OpenTelemetry

* [ ] Standard service names
* [ ] OTLP configured
* [ ] Context propagation verified
* [ ] Error traces retained
* [ ] Slow traces retained
* [ ] Sensitive attributes excluded
* [ ] Collector HA configured
* [ ] Collector backpressure handled
* [ ] Export failures monitored

## Elasticsearch

* [ ] Capacity planned
* [ ] Storage sized
* [ ] Shards planned
* [ ] Replicas configured
* [ ] Disk watermarks monitored
* [ ] Retention configured
* [ ] Snapshots configured where required
* [ ] Cluster health monitored

## Correlation

* [ ] Trace ID appears in logs
* [ ] Trace ID searchable in Kibana
* [ ] Grafana links to traces where supported
* [ ] Jaeger links/correlation to logs configured where supported
* [ ] Incident runbooks document the workflow

---

# 75. Final Mental Model

Remember the entire observability stack like this:

```text
                 ┌─────────────────────┐
                 │       Grafana       │
                 │      Metrics        │
                 └──────────┬──────────┘
                            │
                            v
                       Prometheus
                            ^
                            |
                       Application
                            |
        +-------------------+-------------------+
        |                   |                   |
        | Metrics           | Logs              | Traces
        |                   |                   |
        v                   v                   v
   Prometheus          Fluent Bit          OTel Collector
                            |                   |
                            v                   v
                         Logstash             Jaeger
                            |                   |
                            v                   v
                     Elasticsearch        Trace Storage
                            |
                            v
                         Kibana
```

The three signals answer different questions:

```text
METRICS
"What is wrong?"

LOGS
"What happened?"

TRACES
"Where did this request go, and where did it spend time?"
```

And the production debugging workflow is:

```text
Grafana
   |
   | Detect
   v
Jaeger
   |
   | Isolate request/service
   v
Kibana
   |
   | Inspect detailed logs
   v
AWS / Kubernetes
   |
   | Validate infrastructure
   v
Root Cause
```

That is the observability architecture we want for the 21-service EKS e-commerce platform.

```

**Next chapter:** `41-Autoscaling.md` — HPA, VPA, Karpenter, Cluster Autoscaler, requests/limits, scale-up/scale-down behavior, and how the 21 microservices handle low, medium, and high traffic in production.
```
