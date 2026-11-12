# 122 — Log Correlation

> **Goal:** Implement end-to-end log correlation across the 21-microservice E-Commerce Platform so that a single customer request can be traced through synchronous HTTP calls, asynchronous Kafka/SQS events, databases, and background workers.

---

# 1. Why Log Correlation Matters

Suppose a customer clicks:

```text
Place Order
```

The request may travel through:

```text
Client
  ↓
ALB
  ↓
order-service
  ↓
customer-service
  ↓
product-service
  ↓
pricing-service
  ↓
promotion-service
  ↓
tax-service
  ↓
payment-service
  ↓
fraud-service
  ↓
inventory-service
  ↓
Kafka/SQS
  ↓
notification-service
  ↓
shipping-service
```

If each service generates independent logs, investigating the request becomes difficult.

We want:

```text
correlationId = 7f4c9d...
```

to follow the request.

Then Kibana can search:

```text
correlationId:"7f4c9d..."
```

and return the relevant logs across services.

---

# 2. The Problem Without Correlation

Without correlation:

```text
order-service
10:21:01 ERROR Database timeout

payment-service
10:21:02 ERROR Request failed

inventory-service
10:21:04 ERROR Reservation failed

notification-service
10:21:05 WARN Retry
```

We don't immediately know whether these logs belong to the same customer transaction.

With correlation:

```text
10:21:01 order-service
correlationId=abc123

10:21:02 payment-service
correlationId=abc123

10:21:04 inventory-service
correlationId=abc123

10:21:05 notification-service
correlationId=abc123
```

Now the request chain becomes visible.

---

# 3. Correlation ID

A correlation ID is an identifier used to associate related operations.

Example:

```text
correlationId=01JXYZ123ABC
```

It should be:

* unique enough for the required scope
* generated at the edge if one does not already exist
* propagated across service boundaries
* included in structured logs
* included in relevant asynchronous messages

---

# 4. Correlation ID vs Request ID

These terms are sometimes used interchangeably, but define them clearly in your platform.

### Request ID

Identifies one HTTP request.

```text
requestId=req-123
```

### Correlation ID

Groups related operations belonging to one broader transaction/request flow.

```text
correlationId=order-flow-456
```

For example:

```text
HTTP Request
    │
    ├── requestId = req-001
    │
    └── correlationId = corr-001
```

Then an internal service call:

```text
HTTP Request
    │
    ├── requestId = req-002
    │
    └── correlationId = corr-001
```

The request IDs differ, but the correlation ID remains the same.

---

# 5. Correlation ID vs Trace ID

This distinction becomes especially important when we introduce distributed tracing.

### Correlation ID

Application/business-level grouping.

```text
correlationId=abc123
```

### Trace ID

Distributed tracing identifier.

```text
traceId=4bf92f3577b34da6...
```

### Span ID

Identifies an individual operation within a trace.

```text
spanId=00f067aa0ba902b7
```

Conceptually:

```text
Correlation ID
      │
      ▼
Business/request flow
      │
      ▼
Trace ID
      │
      ├── Span: order-service
      ├── Span: payment-service
      ├── Span: inventory-service
      └── Span: database
```

A mature platform can carry both.

---

# 6. Recommended Header

For our application-level correlation:

```http
X-Correlation-ID: abc123
```

Example:

```http
POST /api/v1/orders
X-Correlation-ID: 01JXYZ123ABC
```

The exact header name should be standardized across the organization.

Do not allow each microservice team to invent:

```text
X-Request-ID
X-Correlation
Correlation-ID
Request-Correlation
X-Trace-Reference
```

with no common convention.

---

# 7. Request Flow

Our target:

```text
Client
  │
  │ X-Correlation-ID
  ▼
ALB
  │
  ▼
order-service
  │
  │ X-Correlation-ID
  ▼
payment-service
  │
  │ X-Correlation-ID
  ▼
inventory-service
```

Every service logs:

```json
{
  "service": "payment-service",
  "correlationId": "01JXYZ123ABC",
  "message": "Payment authorization started"
}
```

---

# 8. Where Should the ID Be Generated?

There are two situations.

### Client already provides one

Validate it and apply organizational rules.

### Client doesn't provide one

Generate it at the application's trusted entry point.

Example:

```text
Request
  │
  ├── X-Correlation-ID exists
  │       ↓
  │    validate
  │
  └── doesn't exist
          ↓
      generate ID
```

Don't blindly trust arbitrary client-supplied identifiers for security-sensitive purposes.

---

# 9. Correlation ID Requirements

A good correlation ID should not contain:

```text
Email
Phone number
Customer name
Order amount
Payment information
```

Bad:

```text
X-Correlation-ID: customer@example.com
```

Better:

```text
X-Correlation-ID: 01JXYZ123ABC
```

The ID is a reference, not business data.

---

# 10. Spring Boot Implementation

Our services use Spring Boot.

A common implementation is:

```text
HTTP Request
     │
     ▼
Filter
     │
     ▼
Read/generate correlation ID
     │
     ▼
MDC
     │
     ▼
Controller
     │
     ▼
Service
     │
     ▼
Repository
```

MDC = **Mapped Diagnostic Context**.

It allows logging frameworks to automatically include contextual fields.

---

# 11. Correlation ID Filter

Example:

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-ID";
    private static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain)
            throws ServletException, IOException {

        String correlationId = request.getHeader(HEADER);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put(MDC_KEY, correlationId);
        response.setHeader(HEADER, correlationId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

Important:

```text
MDC.put()
   ↓
request processing
   ↓
MDC.remove()
```

The cleanup is important because application servers reuse threads.

---

# 12. Why MDC Cleanup Matters

Suppose thread `T1` processes:

```text
Request A
correlationId=A
```

Then later processes:

```text
Request B
```

If the context isn't cleared:

```text
Request B
correlationId=A
```

That is a serious observability bug.

Therefore:

```java
try {
    filterChain.doFilter(request, response);
} finally {
    MDC.remove("correlationId");
}
```

---

# 13. Logging

With MDC configured, logs can include:

```text
correlationId
```

Example:

```java
log.info("Order creation started");
```

Structured output:

```json
{
  "level": "INFO",
  "service": "order-service",
  "correlationId": "01JXYZ123ABC",
  "message": "Order creation started"
}
```

The application code doesn't need to manually pass the ID into every logging statement.

---

# 14. Response Header

Return the correlation ID:

```http
X-Correlation-ID: 01JXYZ123ABC
```

Why?

If a customer or support engineer reports:

```text
Request failed.
Correlation ID: 01JXYZ123ABC
```

you can search:

```text
correlationId:"01JXYZ123ABC"
```

in Kibana.

---

# 15. Internal HTTP Calls

Suppose:

```text
order-service
    ↓
payment-service
```

The order service must propagate:

```http
X-Correlation-ID: 01JXYZ123ABC
```

to payment-service.

Architecture:

```text
Incoming Request
       │
       ▼
order-service
       │
       │ X-Correlation-ID
       ▼
payment-service
```

---

# 16. RestClient / WebClient

For Spring HTTP clients, use a reusable interceptor/filter rather than manually setting the header in every business method.

Conceptually:

```java
headers.set(
    "X-Correlation-ID",
    MDC.get("correlationId")
);
```

This gives:

```text
Service code
     │
     ▼
HTTP client interceptor
     │
     ▼
X-Correlation-ID
```

This is safer and easier to maintain.

---

# 17. Don't Duplicate Propagation Logic

Bad:

```java
paymentClient.call(...)
    .withHeader(...)

inventoryClient.call(...)
    .withHeader(...)

customerClient.call(...)
    .withHeader(...)
```

across hundreds of methods.

Instead:

```text
HTTP Client
    ↓
Reusable interceptor
    ↓
Correlation propagation
```

Centralize cross-cutting behavior.

---

# 18. Asynchronous Messaging

HTTP propagation is straightforward.

But our architecture also contains:

```text
Kafka
SQS
SNS
RabbitMQ
```

Now we need to propagate correlation context through messages.

Example:

```text
order-service
      │
      ▼
Kafka
      │
      ▼
inventory-service
```

The event should contain correlation information.

---

# 19. Event Envelope

Use a standard event envelope:

```json
{
  "eventId": "evt-123",
  "eventType": "OrderCreated",
  "version": "1",
  "occurredAt": "2026-09-29T10:30:00Z",
  "source": "order-service",
  "correlationId": "01JXYZ123ABC",
  "payload": {
    "orderId": "ORD-123"
  }
}
```

We already introduced this pattern in Chapter 117.

Now we connect it to logging.

---

# 20. Kafka Correlation

Producer:

```text
order-service
    │
    │ correlationId=abc123
    ▼
Kafka
    │
    ▼
inventory-service
```

Consumer extracts:

```text
correlationId=abc123
```

and places it into MDC during message processing.

Then:

```text
inventory-service logs
```

contain:

```json
{
  "correlationId": "abc123",
  "service": "inventory-service"
}
```

---

# 21. Kafka Consumer Context

Conceptually:

```java
String correlationId =
        event.getCorrelationId();

MDC.put("correlationId", correlationId);

try {
    process(event);
} finally {
    MDC.remove("correlationId");
}
```

Again:

```text
put
 ↓
process
 ↓
remove
```

is important.

---

# 22. Kafka Headers

Correlation metadata can also be carried using Kafka headers.

Conceptually:

```text
Kafka Record
├── key
├── value
└── headers
     ├── correlationId
     └── trace context
```

This can be useful when you don't want operational metadata duplicated in every payload.

For this project, define clearly which metadata belongs in:

```text
event envelope
```

versus:

```text
transport headers
```

and use the convention consistently.

---

# 23. SQS Correlation

For SQS:

```text
order-service
      │
      ▼
SQS
      │
      ▼
notification-service
```

Carry correlation information in the message.

For example:

```json
{
  "eventId": "evt-100",
  "eventType": "OrderCreated",
  "correlationId": "abc123",
  "payload": {}
}
```

Alternatively, message attributes can carry metadata.

Choose one standard for the platform.

---

# 24. SQS Consumer

Consumer:

```text
Receive message
      │
      ▼
Extract correlationId
      │
      ▼
MDC.put()
      │
      ▼
Process message
      │
      ▼
Log
      │
      ▼
MDC.remove()
```

Example:

```java
MDC.put("correlationId", event.getCorrelationId());

try {
    processOrder(event);
} finally {
    MDC.remove("correlationId");
}
```

---

# 25. RabbitMQ

RabbitMQ can carry correlation information through message properties/headers.

Conceptually:

```text
Message
├── body
└── headers
     └── correlationId
```

Consumer:

```text
RabbitMQ
    ↓
Consumer
    ↓
Extract correlationId
    ↓
MDC
```

---

# 26. Correlation Through a Saga

Our order flow uses a Saga.

Example:

```text
OrderCreated
     │
     ▼
InventoryReservation
     │
     ▼
PaymentAuthorization
     │
     ▼
FraudCheck
     │
     ▼
OrderConfirmed
```

All events belonging to the same business workflow can carry:

```text
correlationId=abc123
```

Then Kibana can reconstruct:

```text
OrderCreated
InventoryReserved
PaymentAuthorized
FraudApproved
OrderConfirmed
```

---

# 27. Correlation ID and Event ID

Don't confuse:

```text
correlationId
```

with:

```text
eventId
```

Example:

```text
correlationId = ORDER-FLOW-123
```

Events:

```text
eventId=EVENT-001
eventType=OrderCreated

eventId=EVENT-002
eventType=InventoryReserved

eventId=EVENT-003
eventType=PaymentAuthorized
```

All three:

```text
correlationId=ORDER-FLOW-123
```

This allows:

```text
one business flow
    ↓
multiple events
```

while keeping each event individually identifiable.

---

# 28. Parent/Child Relationships

For deeper observability:

```text
correlationId
       │
       ▼
traceId
       │
       ├── span A
       │
       ├── span B
       │
       └── span C
```

Later, distributed tracing will provide stronger parent/child relationships.

Log correlation is useful even before tracing is implemented.

---

# 29. Async Context Problem

Thread-local MDC works naturally for:

```text
HTTP request
```

But asynchronous execution can break that assumption.

Example:

```text
Thread A
   │
   └── correlationId=abc
          │
          ▼
      Async task
          │
       Thread B
```

Thread B may not automatically inherit the same context depending on the execution model.

Therefore, explicitly propagate context across:

```text
Executor
CompletableFuture
@Async
Reactive pipelines
Message consumers
Scheduled tasks
```

---

# 30. Thread Pool Example

Suppose:

```text
Request thread
    ↓
ExecutorService
    ↓
worker thread
```

Don't assume MDC automatically propagates correctly.

Use an appropriate context propagation mechanism or task decorator.

Conceptually:

```text
Submit task
    │
    ├── capture MDC
    │
    ▼
Worker thread
    │
    ├── restore MDC
    │
    ▼
Execute
    │
    ▼
Clear MDC
```

---

# 31. Scheduled Jobs

Our reporting service may run:

```text
Every 15 minutes
```

There may be no HTTP request.

So:

```text
correlationId
```

should be generated for the job execution.

Example:

```text
jobCorrelationId=reporting-20260929-1030
```

Then all logs generated by that execution use that context.

---

# 32. Background Workers

Example:

```text
notification-service
     │
     ▼
SQS
     │
     ▼
worker
```

Each message processing attempt should have sufficient context.

Recommended fields:

```text
correlationId
eventId
messageId
service
consumer
attempt
```

This helps distinguish:

```text
same business flow
```

from:

```text
same message retry
```

---

# 33. Retry Correlation

Suppose a message fails three times.

```text
Attempt 1
correlationId=abc
eventId=evt-100

Attempt 2
correlationId=abc
eventId=evt-100

Attempt 3
correlationId=abc
eventId=evt-100
```

Now Kibana can show the complete retry history.

Add:

```text
attempt=1
attempt=2
attempt=3
```

where available.

---

# 34. Dead-Letter Queue Correlation

Suppose:

```text
order-events
     ↓
inventory-service
     ↓
failure
     ↓
retry
     ↓
retry
     ↓
DLQ
```

The DLQ message should preserve relevant metadata:

```text
eventId
correlationId
eventType
original source
failure information
```

Then:

```text
DLQ message
     ↓
correlationId
     ↓
Kibana
     ↓
original request
```

This dramatically improves troubleshooting.

---

# 35. Kibana Search

Example:

```text
correlationId:"01JXYZ123ABC"
```

Then filter:

```text
environment:"prod"
```

Then:

```text
service:*
```

You might see:

```text
10:30:01 order-service
Order creation started

10:30:02 customer-service
Customer validated

10:30:02 pricing-service
Price calculated

10:30:03 inventory-service
Stock reservation started

10:30:04 payment-service
Payment authorization started

10:30:05 payment-service
Payment provider timeout

10:30:06 order-service
Order failed
```

Now the request's story is visible.

---

# 36. Kibana Dashboard — Correlation Investigation

Create a dashboard:

```text
Correlation Investigation
│
├── Correlation ID
├── Event ID
├── Trace ID
├── Services involved
├── Request timeline
├── ERROR logs
├── WARN logs
└── Dependency failures
```

Engineer enters:

```text
Correlation ID
```

and sees the complete flow.

---

# 37. Search by Event ID

Sometimes you don't know the correlation ID.

But you have:

```text
eventId=evt-123
```

Search:

```text
eventId:"evt-123"
```

This is useful for:

```text
Kafka
SQS
RabbitMQ
SNS
```

events.

---

# 38. Search by Order ID

Business users may provide:

```text
Order ID = ORD-123
```

If the order ID is permitted in logs according to data-handling policy, search:

```text
orderId:"ORD-123"
```

Then find:

```text
correlationId
```

and continue the investigation.

Important:

> Business identifiers should not automatically become Prometheus labels just because they are useful for logs.

This is exactly why logs and metrics serve different purposes.

---

# 39. Correlation Through 21 Services

Our final architecture:

```text
                     Client
                       │
                       ▼
                      ALB
                       │
                 correlationId
                       │
                       ▼
                order-service
                       │
        ┌──────────────┼───────────────┐
        ▼              ▼               ▼
 customer-service product-service pricing-service
        │              │               │
        └──────────────┼───────────────┘
                       ▼
               promotion-service
                       │
                       ▼
                  tax-service
                       │
                       ▼
                 payment-service
                       │
                       ▼
                  fraud-service
                       │
                       ▼
               inventory-service
                       │
                       ▼
                    Kafka
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
 notification      shipping       reporting
```

Every relevant component carries enough context to connect the operation.

---

# 40. Standard Log Fields

For all 21 services, standardize:

```json
{
  "@timestamp": "...",
  "level": "INFO",
  "service": "order-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "order-service-...",
  "version": "1.5.2",

  "correlationId": "...",
  "requestId": "...",
  "traceId": "...",
  "spanId": "...",

  "eventId": "...",
  "eventType": "...",

  "message": "..."
}
```

Not every field will exist for every log.

For example:

```text
HTTP request
```

may have:

```text
requestId
```

while:

```text
scheduled job
```

may not.

---

# 41. HTTP-Specific Fields

For HTTP:

```text
method
route
status
durationMs
client
userAgent
```

Example:

```json
{
  "method": "POST",
  "route": "/api/v1/orders",
  "status": 201,
  "durationMs": 145
}
```

Avoid putting raw URLs with sensitive query parameters into logs.

---

# 42. Messaging-Specific Fields

For Kafka/SQS/RabbitMQ:

```text
messageId
eventId
eventType
topic
queue
partition
consumerGroup
attempt
```

Example:

```json
{
  "service": "inventory-service",
  "eventId": "evt-123",
  "eventType": "OrderCreated",
  "topic": "order-events",
  "partition": 4,
  "consumerGroup": "inventory-service"
}
```

---

# 43. Database Correlation

Don't add sensitive data unnecessarily.

Instead, log useful operational information:

```text
database.operation=SELECT
database.system=postgresql
database.durationMs=40
```

Do not log:

```text
SQL password
full customer records
credit-card data
```

For tracing, database spans will later provide richer context.

---

# 44. Error Correlation

When an exception occurs:

```json
{
  "level": "ERROR",
  "service": "payment-service",
  "correlationId": "abc123",
  "eventId": "evt-456",
  "error.type": "TimeoutException",
  "error.message": "Payment provider timeout"
}
```

Avoid dumping sensitive request bodies into logs just to make troubleshooting easier.

---

# 45. Correlation ID Validation

A malicious or careless client could send:

```text
X-Correlation-ID: 10MB_OF_DATA
```

or values containing unexpected characters.

Therefore:

```text
Validate
Normalize
Limit length
```

For example:

```text
maximum length = reasonable fixed limit
```

and reject or replace invalid values.

---

# 46. Prevent Log Injection

Never blindly trust values inserted into logs.

For example:

```text
username = "normal\nFAKE ERROR"
```

could create misleading log entries.

Structured JSON logging greatly reduces some of these problems, but input validation and proper encoding remain important.

---

# 47. Correlation ID and Security

Correlation IDs should help investigation but should not become authentication credentials.

Never use:

```text
correlationId
```

as:

```text
password
token
authorization credential
```

Anyone who knows a correlation ID should not gain access to anything.

---

# 48. Correlation ID Storage

Don't store correlation IDs as a long-term business database field unless there is a genuine business requirement.

Usually:

```text
correlationId
```

belongs to:

```text
logs
traces
events
```

rather than being treated as a primary business identifier.

---

# 49. Correlation ID in CI/CD

Deployment systems can also generate useful context.

For example:

```text
deploymentId=deploy-2026-09-29-001
```

Then logs can include:

```text
version=1.5.2
deploymentId=deploy-2026-09-29-001
```

This allows:

```text
Deployment
   ↓
Version
   ↓
Error spike
   ↓
Logs
```

to be correlated.

---

# 50. Correlation ID + Git SHA

Every container should have an immutable version.

Example:

```text
image:
order-service:9f2e31a
```

Logs:

```json
{
  "service": "order-service",
  "version": "9f2e31a"
}
```

Now when an error occurs:

```text
correlationId
+
version
```

helps identify exactly which build generated the log.

This connects:

```text
Git
 ↓
CI
 ↓
ECR
 ↓
Deployment
 ↓
Logs
```

---

# 51. Incident Example — Complete Correlation

Customer:

> My order failed.

Support provides:

```text
Order ID: ORD-123
```

Engineer searches Kibana:

```text
orderId:"ORD-123"
```

Finds:

```text
correlationId=abc123
```

Now searches:

```text
correlationId:"abc123"
```

Timeline:

```text
10:30:00 order-service
Order creation started

10:30:01 customer-service
Customer validated

10:30:01 pricing-service
Price calculated

10:30:02 inventory-service
Reservation successful

10:30:03 payment-service
Authorization started

10:30:05 payment-service
Provider timeout

10:30:05 payment-service
Retry 1

10:30:07 payment-service
Retry 2

10:30:12 payment-service
Payment failed

10:30:12 order-service
Order marked PAYMENT_FAILED
```

Now investigate the payment provider rather than the entire platform.

---

# 52. Incident Example — Kafka

Suppose:

```text
Order created
```

but:

```text
Inventory wasn't updated.
```

Search:

```text
orderId:"ORD-123"
```

Find:

```text
correlationId=abc123
eventId=evt-100
```

Then search:

```text
eventId:"evt-100"
```

Find:

```text
order-service
OrderCreated published

Kafka
event accepted

inventory-service
event received

inventory-service
database timeout

inventory-service
retry

inventory-service
retry failed

inventory-service
message sent to DLQ
```

This gives the complete event lifecycle.

---

# 53. Incident Example — Duplicate Processing

Suppose a customer appears to have been charged twice.

Search:

```text
correlationId:"abc123"
```

Then:

```text
eventId:"evt-100"
```

You may discover:

```text
Payment attempt 1
Payment timeout
Consumer retry
Payment attempt 2
```

Now correlate:

```text
idempotency key
eventId
payment provider request ID
correlationId
```

This is where logging becomes critical for financial workflows.

---

# 54. Don't Log Payment Secrets

For payment troubleshooting, log safe references:

```text
paymentRequestId
providerTransactionId
correlationId
eventId
```

Do not log:

```text
CVV
full card number
authentication secret
provider credentials
```

---

# 55. Correlation Contract

Create a platform-wide document:

```text
platform/
└── observability/
    └── correlation-standard.md
```

Define:

```text
Header:
X-Correlation-ID

Required:
correlationId

Messaging:
eventId
correlationId

Tracing:
traceId
spanId

HTTP:
requestId

Deployment:
version
deploymentId
```

All 21 services follow this standard.

---

# 56. Reusable Java Library

Instead of implementing correlation separately in all 21 services, create a reusable internal library.

Example:

```text
platform/
└── java-observability-library/
    ├── correlation/
    ├── logging/
    ├── metrics/
    └── tracing/
```

It can provide:

```text
CorrelationIdFilter
HTTP client propagation
MDC utilities
Kafka context propagation
SQS context propagation
structured logging configuration
```

Then:

```text
order-service
payment-service
inventory-service
...
```

all consume the same library version.

---

# 57. Benefits of Shared Library

Without shared library:

```text
21 services
×
custom implementation
```

Potentially:

```text
21 different behaviors
```

With a platform library:

```text
Platform library
       ↓
standard implementation
       ↓
21 services
```

But don't make the library so large that every application becomes tightly coupled to platform implementation details.

Keep it focused on cross-cutting concerns.

---

# 58. GitOps + Observability

Deployment metadata should be standardized.

For example:

```yaml
metadata:
  labels:
    app: order-service
    version: "1.5.2"
    environment: prod
```

This helps:

```text
Prometheus
Grafana
Fluent Bit
Elasticsearch
Kibana
```

all use consistent metadata.

---

# 59. Standard Kubernetes Labels

Use a consistent label scheme.

Example:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/instance: order-service
  app.kubernetes.io/version: "1.5.2"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: ecommerce-platform
```

These labels are useful for correlation and filtering.

---

# 60. Correlation Architecture

Final model:

```text
                    Customer Request
                           │
                           ▼
                     X-Correlation-ID
                           │
                           ▼
                     order-service
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          customer      pricing      product
              │            │            │
              └────────────┼────────────┘
                           ▼
                      payment
                           │
                           ▼
                         Kafka
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         inventory     notification   reporting
```

Logs:

```text
                    correlationId
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         service A     service B    service C
             │            │            │
             └────────────┼────────────┘
                          ▼
                    Elasticsearch
                          │
                          ▼
                       Kibana
```

---

# 61. Production Checklist

## HTTP

* [ ] Correlation ID standard defined
* [ ] Header standardized
* [ ] Generate if absent
* [ ] Validate incoming ID
* [ ] Return ID in response
* [ ] Propagate downstream
* [ ] MDC configured
* [ ] MDC cleared

## Messaging

* [ ] Kafka correlation defined
* [ ] SQS correlation defined
* [ ] RabbitMQ correlation defined
* [ ] Event ID defined
* [ ] Correlation ID defined
* [ ] Retry preserves correlation
* [ ] DLQ preserves correlation

## Logging

* [ ] Structured JSON logs
* [ ] Correlation ID
* [ ] Request ID
* [ ] Event ID
* [ ] Trace ID later
* [ ] Version
* [ ] Environment
* [ ] Service
* [ ] No secrets
* [ ] PII reviewed

## Platform

* [ ] Shared Java library
* [ ] Standard Kubernetes labels
* [ ] Deployment version metadata
* [ ] Kibana correlation dashboard
* [ ] Runbooks updated

---

# 62. Final Observability Flow

We now have:

```text
                         REQUEST
                            │
                            ▼
                    X-Correlation-ID
                            │
          ┌─────────────────┼──────────────────┐
          │                 │                  │
          ▼                 ▼                  ▼
       Metrics             Logs              Events
          │                 │                  │
          ▼                 ▼                  ▼
     Prometheus          Fluent Bit         Kafka/SQS
          │                 │                  │
          ▼                 ▼                  │
       Grafana          Logstash              │
                            │                  │
                            ▼                  │
                      Elasticsearch            │
                            │                  │
                            ▼                  │
                         Kibana ◄──────────────┘
```

The investigation path becomes:

```text
Customer problem
      ↓
Grafana
      ↓
Affected service
      ↓
Correlation ID
      ↓
Kibana
      ↓
Related logs
      ↓
Event ID
      ↓
Kafka/SQS
      ↓
Dependency
      ↓
Root cause
```

That is the foundation for distributed observability.

---

# Next Chapter

## `123-Distributed-Tracing.md`

Next we move beyond log correlation and implement actual distributed tracing:

```text
Client
  │
  ▼
ALB
  │
  ▼
order-service
  │
  ├──────────► customer-service
  │
  ├──────────► pricing-service
  │
  └──────────► payment-service
                    │
                    ▼
              payment provider
```

We will cover:

* trace vs span
* trace ID
* span ID
* parent/child spans
* HTTP propagation
* W3C Trace Context
* synchronous service tracing
* asynchronous Kafka/SQS tracing
* database spans
* Redis spans
* external API spans
* sampling
* trace volume/cost
* trace-to-log correlation
* trace-to-metric correlation
* identifying the slowest dependency
* distributed trace incident walkthroughs
* production tracing architecture for all 21 microservices.
