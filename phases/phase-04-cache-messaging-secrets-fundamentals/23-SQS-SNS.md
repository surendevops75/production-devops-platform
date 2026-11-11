# 23-SQS-SNS.md

# Amazon SQS & SNS — Production Messaging Architecture

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Consumers:** EKS microservices
> **Messaging options covered:** SQS, SNS, RabbitMQ, Kafka
> **Primary AWS design:** SQS + SNS
> **Alternatives:** RabbitMQ and Kafka, covered in dedicated chapters

Amazon SQS is a fully managed message-queuing service designed to decouple distributed application components, while SNS provides publish/subscribe fan-out to multiple subscribers. ([AWS Documentation][1])

---

# 1. Why Do We Need Messaging?

Our microservices should not depend on every other service being available at the exact same moment.

Without asynchronous messaging:

```text
order-service
     │
     ├──── HTTP ────► payment-service
     │
     ├──── HTTP ────► inventory-service
     │
     ├──── HTTP ────► notification-service
     │
     └──── HTTP ────► invoice-service
```

Now imagine:

```text
notification-service
        X
```

The order request may become slow or fail even though the notification system is not actually required to complete the order.

Messaging allows us to decouple these workloads.

---

# 2. Asynchronous Architecture

Instead:

```text
order-service
      │
      ▼
     SNS
      │
      ├──► SQS payment
      ├──► SQS inventory
      ├──► SQS notification
      └──► SQS invoice
```

The order service publishes an event and consumers process it independently.

AWS specifically supports the SNS → SQS pattern for fan-out and asynchronous processing. ([AWS Documentation][2])

---

# 3. SQS vs SNS

This distinction must be very clear.

## SQS

SQS is primarily:

```text
Queue
```

Think:

> "Someone needs to process this message."

Example:

```text
order-service
      │
      ▼
payment-queue
      │
      ▼
payment-service
```

---

## SNS

SNS is primarily:

```text
Pub/Sub
```

Think:

> "Something happened, and multiple systems may be interested."

Example:

```text
             SNS Topic
                 │
       ┌─────────┼─────────┐
       │         │         │
      SQS       SQS       SQS
       │         │         │
    Payment   Inventory Notification
```

SNS supports fan-out to multiple subscribers, including SQS queues. ([AWS Documentation][3])

---

# 4. Our E-Commerce Example

Suppose:

```text
POST /api/v1/orders
```

creates:

```json
{
  "orderId": "ORD-1001",
  "customerId": "C-100",
  "amount": 4999
}
```

Order service:

```text
order-service
      │
      ▼
PostgreSQL
      │
      ▼
OrderCreated event
      │
      ▼
SNS
```

SNS then fans out:

```text
                       SNS
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
     payment-queue inventory-queue notification-queue
          │             │             │
          ▼             ▼             ▼
       payment       inventory      notification
```

This is a very common event-driven architecture pattern.

---

# 5. Why Not Call Everything Through REST?

Imagine:

```text
order-service
      │
      ├── payment-service
      ├── inventory-service
      ├── notification-service
      ├── invoice-service
      └── shipping-service
```

If every call is synchronous:

```text
Order API
   │
   ├── Payment → 500ms
   ├── Inventory → 300ms
   ├── Invoice → 400ms
   ├── Notification → 800ms
   └── Shipping → 300ms
```

The request can become slow and fragile.

With asynchronous messaging:

```text
Order API
   │
   ▼
Save Order
   │
   ▼
Publish Event
   │
   ▼
Return response
```

Consumers process their work independently.

---

# 6. Synchronous vs Asynchronous

### Synchronous

```text
Client
  │
  ▼
Order
  │
  ▼
Payment
  │
  ▼
Response
```

The caller waits.

### Asynchronous

```text
Client
  │
  ▼
Order
  │
  ▼
Queue
  │
  └────► Consumer
```

The caller doesn't need to wait for the consumer to finish.

---

# 7. When NOT to Use SQS

Don't automatically make every operation asynchronous.

For example:

```text
GET /products/P1001
```

usually needs an immediate response.

A synchronous path is appropriate:

```text
Client
  │
  ▼
catalog-service
  │
  ▼
Redis
  │
  ▼
Database
```

Messaging is better for work that can happen asynchronously.

---

# 8. Good SQS Use Cases

Examples:

```text
Payment processing
Email sending
Invoice generation
Image processing
Order fulfillment
Report generation
Background jobs
Retryable work
```

Architecture:

```text
Producer
   │
   ▼
SQS
   │
   ▼
Consumer
```

---

# 9. Good SNS Use Cases

SNS is useful when one event has multiple independent subscribers.

Example:

```text
OrderCreated
     │
     ▼
    SNS
     │
     ├── Payment
     ├── Inventory
     ├── Notification
     ├── Fraud
     └── Analytics
```

Each subscriber can have its own SQS queue.

---

# 10. SNS + SQS Is More Powerful Than SNS Alone

A common production pattern is:

```text
                  SNS
                   │
       ┌───────────┼───────────┐
       │           │           │
      SQS         SQS         SQS
       │           │           │
   Payment      Inventory   Notification
```

Why?

Each consumer gets an independent queue.

Suppose notification-service is down:

```text
notification-service
        X
```

Its queue can retain messages while payment processing continues through its own queue.

---

# 11. Important Isolation Principle

Bad:

```text
SNS
 │
 └── one shared queue
       │
       ├── payment
       ├── inventory
       └── notification
```

Now consumers can interfere with each other.

Better:

```text
SNS
 │
 ├── payment-queue
 ├── inventory-queue
 └── notification-queue
```

Each service owns its consumption workload.

---

# 12. Our Messaging Architecture

For the e-commerce project:

```text
                        SNS
                         │
             ecommerce-events
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
payment-queue      inventory-queue   notification-queue
       │                 │                  │
       ▼                 ▼                  ▼
payment-service    inventory-service  notification-service
```

Additional consumers:

```text
SNS
 │
 ├── fraud-queue
 ├── invoice-queue
 ├── shipping-queue
 └── reporting-queue
```

---

# 13. Queue Ownership

A useful rule:

> Each consuming microservice should normally have its own queue.

Example:

```text
payment-service
     │
     └── payment-queue

inventory-service
     │
     └── inventory-queue

notification-service
     │
     └── notification-queue
```

This gives each service:

* independent retry behavior
* independent scaling
* independent backlog
* independent DLQ
* independent monitoring

---

# 14. Standard vs FIFO SQS

SQS provides different queue semantics.

### Standard Queue

Designed for very high throughput and distributed processing.

Use when:

```text
Ordering not strictly required
```

### FIFO Queue

Use when:

```text
Ordering matters
+
Duplicate handling requirements matter
```

For example:

```text
OrderCreated
OrderConfirmed
OrderShipped
```

may require carefully designed ordering semantics.

Do not choose FIFO simply because "FIFO sounds safer."

Choose based on business requirements.

---

# 15. Ordering Is a Business Requirement

Suppose messages:

```text
M1 = OrderCreated
M2 = OrderCancelled
```

If the consumer processes:

```text
M2
M1
```

you may have a problem.

But ordering everything globally can severely limit scalability.

Therefore ask:

> **What exactly needs to be ordered?**

Often you need ordering per:

```text
orderId
customerId
productId
```

rather than globally.

---

# 16. Message Groups

For FIFO-style workloads, a common concept is:

```text
MessageGroupId
```

Example:

```text
Order 1001 → group = ORD-1001
Order 1002 → group = ORD-1002
```

Then messages for the same order can be serialized while different orders can potentially process concurrently.

This is a much better scaling model than:

```text
One global queue
One global consumer
```

---

# 17. Visibility Timeout

One of the most important SQS concepts.

Suppose:

```text
Message
   │
   ▼
SQS
```

Consumer receives it:

```text
Consumer
   │
   ▼
Message
```

The message becomes temporarily invisible to other consumers.

This is the:

```text
Visibility Timeout
```

---

# 18. Why Visibility Timeout Exists

Suppose payment-service receives:

```text
PaymentRequested
```

and needs:

```text
30 seconds
```

to process it.

We don't want another consumer immediately processing the same message.

Conceptually:

```text
Receive
   ↓
Visibility timeout starts
   ↓
Process
   ↓
Delete message
```

If processing fails:

```text
Receive
   ↓
Visibility timeout
   ↓
Consumer crashes
   ↓
Timeout expires
   ↓
Message becomes available again
```

---

# 19. Visibility Timeout Must Be Sized Correctly

Suppose:

```text
Typical processing = 20 sec
```

but:

```text
Visibility timeout = 5 sec
```

Then:

```text
Consumer A
   │
   ├── processing
   │
   └── 5 sec passes

Message becomes visible

Consumer B
   │
   └── receives same message
```

Now two consumers may process the same business operation.

Therefore:

> Visibility timeout should be comfortably larger than normal processing time, with an explicit strategy for unusually long processing.

---

# 20. Long-Running Processing

If processing can take longer than the visibility timeout, the consumer may need to extend the visibility timeout while it works.

Conceptually:

```text
Receive
  │
  ▼
Visibility = 60 sec
  │
  ▼
Processing still running
  │
  ▼
Extend visibility
  │
  ▼
Continue
  │
  ▼
Delete after success
```

---

# 21. Message Acknowledgement

SQS does not use RabbitMQ's exact ACK mechanism.

The consumer:

```text
ReceiveMessage
      │
      ▼
Process
      │
      ▼
DeleteMessage
```

The deletion is effectively the successful completion signal.

If the consumer crashes before deletion:

```text
Consumer
   X
```

the message can become visible again after the visibility timeout.

---

# 22. At-Least-Once Processing

A critical production rule:

> Your consumer must be idempotent.

Why?

A message can potentially be delivered more than once.

Example:

```text
PaymentRequested
      │
      ▼
payment-service
      │
      ▼
Payment succeeds
      │
      X
Network failure before message deletion
```

SQS doesn't know the payment succeeded.

The message may be delivered again.

Without idempotency:

```text
Charge ₹5,000
Charge ₹5,000
```

This could become a serious business problem.

---

# 23. Idempotency

Use a unique business operation identifier.

Example:

```json
{
  "eventId": "EVT-12345",
  "orderId": "ORD-1001",
  "operation": "PAYMENT"
}
```

Consumer:

```text
Receive event
     │
     ▼
Check processed-event store
     │
 ┌───┴────┐
 │        │
Exists   New
 │        │
ACK      Process
          │
          ▼
       Record event
          │
          ▼
          ACK
```

The exact implementation can use a database table, conditional write, or another durable idempotency mechanism.

---

# 24. Never Use "Exactly Once" Casually

In distributed systems:

```text
Exactly once
```

is a much stronger claim than:

```text
At least once
```

For application business operations, design assuming duplicate delivery can happen.

Then make the operation idempotent.

This is much safer.

---

# 25. Dead-Letter Queue

Suppose:

```text
Payment message
```

fails repeatedly.

We don't want:

```text
Retry
Retry
Retry
Retry
Retry
...
```

forever.

Instead:

```text
payment-queue
      │
      ├── success → payment-service
      │
      └── repeated failure
               │
               ▼
             DLQ
```

Dead-letter queues are essential for production troubleshooting.

---

# 26. DLQ Architecture

```text
payment-queue
      │
      ├── successful → delete
      │
      └── failures
             │
             ▼
       payment-dlq
```

Now engineers can investigate:

```text
Why did this message fail?
```

without blocking the normal queue indefinitely.

---

# 27. Redrive Policy

A queue can be configured so that after a message has been received a certain number of times:

```text
Receive count
    │
    ▼
Threshold reached
    │
    ▼
Move to DLQ
```

Example:

```text
maxReceiveCount = 5
```

The exact number should be based on the workload.

---

# 28. DLQ Is Not a Trash Bin

This is important.

Don't create:

```text
DLQ
 ↓
Ignore
```

Instead:

```text
DLQ
 │
 ▼
Monitoring
 │
 ▼
Alert
 │
 ▼
Investigate
 │
 ├── application bug
 ├── invalid data
 ├── dependency outage
 ├── poison message
 └── schema mismatch
```

Then decide whether the message should be safely redriven.

---

# 29. Poison Messages

A poison message is a message that repeatedly fails processing.

Example:

```json
{
  "orderId": null
}
```

Consumer:

```text
Deserialize
   ↓
Validation fails
   ↓
Retry
   ↓
Validation fails
```

Eventually:

```text
DLQ
```

This prevents one bad message from consuming processing capacity forever.

---

# 30. Retry Strategy

Not every error should be retried equally.

### Transient error

```text
RDS timeout
Redis timeout
Temporary network issue
```

Retry may make sense.

### Permanent error

```text
Invalid schema
Missing required field
Invalid order ID
```

Retrying may be pointless.

Therefore:

```text
Error
 │
 ├── transient → retry
 │
 └── permanent → DLQ
```

---

# 31. Exponential Backoff

Don't retry:

```text
1 sec
1 sec
1 sec
1 sec
1 sec
```

forever.

Use backoff:

```text
1 sec
2 sec
4 sec
8 sec
16 sec
```

with appropriate jitter.

This prevents synchronized retries from creating another outage.

---

# 32. SNS Filtering

SNS can route messages based on message attributes/filter policies.

Example:

```text
SNS
 │
 ├── payment queue
 │      filter: eventType = PAYMENT
 │
 ├── inventory queue
 │      filter: eventType = INVENTORY
 │
 └── notification queue
        filter: eventType = NOTIFICATION
```

This prevents every consumer from receiving every event unnecessarily.

---

# 33. Event Envelope

Use a consistent event format.

Example:

```json
{
  "eventId": "EVT-10001",
  "eventType": "OrderCreated",
  "eventVersion": "1",
  "occurredAt": "2026-09-08T12:00:00Z",
  "source": "order-service",
  "correlationId": "REQ-10001",
  "data": {
    "orderId": "ORD-1001",
    "customerId": "C-100",
    "amount": 4999
  }
}
```

This gives us:

* traceability
* versioning
* debugging
* correlation
* consistent consumers

---

# 34. Event Versioning

Suppose version 1:

```json
{
  "orderId": "ORD-1001",
  "amount": 4999
}
```

Later:

```json
{
  "orderId": "ORD-1001",
  "amount": 4999,
  "currency": "INR"
}
```

Existing consumers shouldn't immediately break.

Therefore:

```text
eventVersion
```

should be part of the design.

---

# 35. Correlation ID

A request may travel through:

```text
Client
 ↓
ALB
 ↓
order-service
 ↓
SNS
 ↓
payment-service
 ↓
SQS
 ↓
notification-service
```

Use:

```text
correlationId
```

to connect the entire workflow.

Example:

```text
REQ-10001
```

appears in:

```text
application logs
message attributes
events
traces
```

This becomes extremely useful when we later implement:

```text
OpenTelemetry
Jaeger
ELK
```

---

# 36. Message Payload Size

Don't put huge objects into messages unnecessarily.

Bad:

```json
{
  "order": {
    "...": "massive nested object..."
  }
}
```

Better:

```json
{
  "eventId": "EVT-10001",
  "eventType": "OrderCreated",
  "orderId": "ORD-1001"
}
```

Then the consumer can retrieve additional data when appropriate.

But don't create a design where every event causes expensive synchronous database lookups either.

Choose payload size based on:

* consumer needs
* consistency
* performance
* message limits
* coupling

---

# 37. SQS Queue Naming

Use predictable names.

Examples:

```text
ecommerce-prod-payment
ecommerce-prod-inventory
ecommerce-prod-notification
ecommerce-prod-invoice
```

DLQs:

```text
ecommerce-prod-payment-dlq
ecommerce-prod-inventory-dlq
```

Environment should be obvious.

---

# 38. SNS Topic Naming

Example:

```text
ecommerce-prod-order-events
ecommerce-prod-payment-events
ecommerce-prod-inventory-events
```

Avoid vague names:

```text
events
topic1
queue2
```

Production naming should communicate ownership and environment.

---

# 39. Terraform Architecture

Infrastructure repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── sqs/
    │   └── sns/
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

---

# 40. Terraform SQS Queue

Example:

```hcl
resource "aws_sqs_queue" "payment" {
  name = "${var.name}-payment"

  visibility_timeout_seconds = 60

  message_retention_seconds = 345600

  receive_wait_time_seconds = 20

  sqs_managed_sse_enabled = true
}
```

These values are examples and should be sized according to the workload.

---

# 41. Long Polling

Instead of continuously asking:

```text
Is there a message?

Is there a message?

Is there a message?
```

use long polling.

Conceptually:

```text
Consumer
   │
   ▼
SQS
   │
   └── waits for message
```

This reduces unnecessary empty receives.

Example:

```hcl
receive_wait_time_seconds = 20
```

---

# 42. Why Long Polling Matters

Without effective polling:

```text
100 consumers
   ×
many empty polls
   ↓
unnecessary API calls
```

With long polling:

```text
Consumer
   │
   ▼
Wait
   │
   ▼
Message arrives
   │
   ▼
Process
```

This is both more efficient and often reduces unnecessary request overhead.

---

# 43. SQS Retention

SQS messages have a retention period.

The right value depends on:

```text
Expected outage duration
+
Recovery strategy
+
Business requirements
```

If notification-service may be down for several hours, the queue should retain messages long enough for recovery according to the business SLA.

---

# 44. Queue Backlog

One of the most important production metrics is:

```text
ApproximateNumberOfMessagesVisible
```

Suppose:

```text
Queue backlog
10
20
50
100
500
5000
```

This indicates consumers are not keeping up.

---

# 45. Queue Backlog and HPA

This is where Kubernetes and AWS integrate.

Architecture:

```text
                  SQS
                   │
              queue backlog
                   │
                   ▼
             KEDA / metrics
                   │
                   ▼
                 HPA
                   │
             ┌─────┴─────┐
             │           │
           Pod         Pod
             │           │
             ▼           ▼
       payment-service consumers
```

For queue-driven workloads, scaling on CPU alone may be inadequate.

We can scale consumers based on queue backlog using an appropriate event-driven autoscaling mechanism such as KEDA.

---

# 46. Example Scaling Scenario

Queue:

```text
100 messages
```

Consumers:

```text
2 pods
```

Backlog grows:

```text
100
500
2000
```

KEDA can react:

```text
2 pods
 ↓
5 pods
 ↓
10 pods
```

As the backlog falls:

```text
2000
 ↓
1000
 ↓
100
```

consumer capacity can scale down according to the configured policy.

---

# 47. Consumer Application

Our Spring Boot service:

```text
payment-service
```

can consume messages.

Conceptually:

```java
@SqsListener("ecommerce-prod-payment")
public void processPayment(PaymentEvent event) {

    // validate

    // idempotency check

    // process payment

    // persist result

    // successful return = message completion
}
```

The exact Spring Cloud AWS dependency/configuration depends on the selected project version.

---

# 48. Consumer Processing Flow

Production consumer:

```text
Receive message
      │
      ▼
Validate schema
      │
      ▼
Check idempotency
      │
      ▼
Process business operation
      │
      ▼
Persist result
      │
      ▼
Successful completion
      │
      ▼
Message removed
```

If failure:

```text
Failure
   │
   ├── transient → retry
   │
   └── repeated/permanent → DLQ
```

---

# 49. IAM

We don't give every service full SQS access.

For example:

```text
payment-service
```

might have:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

on:

```text
payment queue
```

Only.

It shouldn't automatically have:

```text
sqs:*
```

on:

```text
*
```

---

# 50. EKS Pod Identity

Our AWS access architecture:

```text
payment-service Pod
       │
       ▼
Kubernetes ServiceAccount
       │
       ▼
EKS Pod Identity
       │
       ▼
IAM Role
       │
       ▼
SQS
```

No static AWS access keys inside the container.

This continues the IAM architecture we established earlier.

---

# 51. SNS Publish Permissions

For:

```text
order-service
```

we may grant:

```text
sns:Publish
```

only on:

```text
ecommerce-prod-order-events
```

Not:

```text
sns:*
```

globally.

---

# 52. SQS Encryption

Production queues should use encryption according to organizational requirements.

Options include:

```text
SQS-managed encryption
```

or:

```text
AWS KMS customer-managed key
```

where appropriate.

If using KMS, IAM must also allow the necessary key operations.

---

# 53. SNS Encryption

Similarly:

```text
SNS Topic
    │
    ▼
KMS encryption
```

where required.

Don't treat encryption as simply:

```text
Create KMS key
```

You must also correctly configure:

* key policy
* IAM permissions
* service permissions
* rotation strategy
* access boundaries

---

# 54. Dead-Letter Architecture

Production:

```text
                  SNS
                   │
                   ▼
             payment-queue
                   │
              ┌────┴────┐
              │         │
          success      failure
              │         │
              ▼         ▼
            Done     payment-dlq
                         │
                         ▼
                      Alert
```

Monitoring should detect abnormal DLQ growth.

---

# 55. Alerting

Alert on:

```text
Queue backlog
DLQ messages
Age of oldest message
Consumer errors
Consumer count
Processing latency
Visibility timeout issues
```

One particularly useful metric is:

```text
Age of oldest message
```

because a queue can have only a small number of messages but those messages may have been waiting for a very long time.

---

# 56. Message Age and SLA

Suppose:

```text
notification queue
```

has:

```text
10 messages
```

That sounds healthy.

But:

```text
Oldest message = 45 minutes
```

while the business SLA is:

```text
< 5 minutes
```

Then the system is unhealthy.

So:

```text
Queue depth
```

alone is not enough.

---

# 57. SQS Standard Queue Architecture

```text
                  SNS
                   │
                   ▼
             Standard SQS
                   │
       ┌───────────┼───────────┐
       │           │           │
     Pod-1       Pod-2       Pod-3
       │           │           │
       └───────────┼───────────┘
                   │
             Processing
```

This is excellent for scalable asynchronous workloads where strict ordering is not required.

---

# 58. FIFO Architecture

For workflows requiring ordered processing:

```text
                 FIFO SQS
                    │
             Message Group
                    │
                    ▼
               consumer
```

Example:

```text
Order 1001:
Created
Paid
Shipped
Delivered
```

You need to carefully determine what ordering is actually required and how the consumer handles duplicates.

---

# 59. SQS vs RabbitMQ

Now we can compare the technologies you've asked about.

| Capability            | SQS                     | RabbitMQ                         |
| --------------------- | ----------------------- | -------------------------------- |
| Managed by AWS        | Yes                     | Not unless using Amazon MQ       |
| Queue                 | Yes                     | Yes                              |
| Routing               | Simpler                 | Very powerful exchanges/bindings |
| Consumer ACK          | Delete after processing | Explicit acknowledgements        |
| DLQ                   | Yes                     | Dead-letter exchanges/queues     |
| Scaling               | AWS managed             | Broker/cluster capacity          |
| Operational burden    | Low                     | Higher if self-managed           |
| Kubernetes deployment | Consumer only usually   | Broker can run in EKS            |
| Best fit              | AWS-native async work   | Complex messaging/routing        |

RabbitMQ uses exchanges, queues and bindings, with acknowledgements and publisher confirms for delivery safety. ([RabbitMQ][4])

---

# 60. SQS vs Kafka

| Capability        | SQS                          | Kafka                             |
| ----------------- | ---------------------------- | --------------------------------- |
| Primary model     | Queue                        | Event log/stream                  |
| Message retention | Queue retention              | Configurable topic retention      |
| Replay            | Limited compared with Kafka  | Core feature                      |
| Consumer groups   | Application pattern          | Native                            |
| Partitioning      | FIFO/message-group semantics | Core scaling mechanism            |
| Event history     | Not primary                  | Primary                           |
| Operations        | Low with AWS                 | Higher unless using managed Kafka |
| Best fit          | Async jobs/work queues       | Event streaming                   |

Kafka stores records in partitioned logs with offsets and configurable retention; consumer groups allow parallel consumption while preserving ordering within a partition. ([Apache Kafka][5])

---

# 61. SQS/SNS vs RabbitMQ vs Kafka

The simplest mental model:

```text
SQS
 ↓
"Please process this work."

SNS
 ↓
"Something happened; notify subscribers."

RabbitMQ
 ↓
"Route and process messages with broker-level messaging patterns."

Kafka
 ↓
"Record this event stream so multiple consumers can process it."
```

None is universally better.

---

# 62. What We Will Use in Our AWS Project

Our main AWS architecture will use:

```text
SQS
+
SNS
```

because we're building an AWS-native platform and want to learn the managed-service model.

We'll then separately learn:

```text
RabbitMQ
```

and:

```text
Kafka
```

so you understand both **managed AWS messaging** and **self-managed/clustered messaging infrastructure**.

---

# 63. Complete Order Flow

Our production order flow can eventually look like:

```text
Customer
   │
   ▼
ALB
   │
   ▼
order-service
   │
   ├── PostgreSQL
   │
   ├── Redis
   │
   └── SNS
         │
         ├── payment-queue
         │       │
         │       ▼
         │   payment-service
         │
         ├── inventory-queue
         │       │
         │       ▼
         │   inventory-service
         │
         ├── notification-queue
         │       │
         │       ▼
         │   notification-service
         │
         ├── fraud-queue
         │       │
         │       ▼
         │   fraud-service
         │
         └── invoice-queue
                 │
                 ▼
             invoice-service
```

This is much closer to a real enterprise microservice architecture.

---

# 64. Failure Scenario — Payment Service Down

Suppose:

```text
payment-service
     X
```

Order service has already published:

```text
OrderCreated
```

The payment queue retains the work.

```text
SNS
 │
 ▼
payment-queue
 │
 │ waiting
 ▼
payment-service
```

When the service recovers:

```text
payment-service
      │
      ▼
payment-queue
      │
      ▼
Process messages
```

The order service doesn't need to continuously retry an HTTP call.

---

# 65. Failure Scenario — Notification Service Slow

Suppose:

```text
notification-service
```

can process only:

```text
100 messages/sec
```

but incoming traffic is:

```text
500 messages/sec
```

Queue backlog grows:

```text
100
500
1000
5000
```

Instead of immediately failing order creation:

```text
Order creation continues
```

while the notification consumer scales.

```text
5 consumers
   ↓
10 consumers
   ↓
20 consumers
```

This is one of the major benefits of asynchronous architecture.

---

# 66. Failure Scenario — Poison Message

Suppose:

```text
OrderCreated
```

contains invalid data.

Consumer:

```text
payment-service
```

fails repeatedly.

Eventually:

```text
payment-queue
      │
      ▼
retry limit
      │
      ▼
payment-dlq
```

Alert:

```text
DLQ > 0
```

Engineer investigates.

The healthy queue continues processing other messages.

---

# 67. Failure Scenario — Database Down

Suppose:

```text
payment-service
      │
      ▼
RDS
      X
```

Consumer attempts processing.

Transient DB error:

```text
Process fails
     ↓
Don't delete message
     ↓
Retry
```

If the outage continues:

```text
Retry
Retry
Retry
     ↓
DLQ / controlled failure handling
```

This prevents silent message loss.

---

# 68. Important: Messaging Does Not Eliminate Distributed-System Problems

Messaging introduces new concerns:

```text
Duplicate messages
Ordering
Retries
Dead letters
Schema evolution
Poison messages
Backpressure
Consumer lag
Idempotency
Observability
```

So:

> Adding SQS doesn't automatically make the system reliable.

You must design the entire message lifecycle.

---

# 69. Production Message Lifecycle

The complete lifecycle:

```text
Producer
   │
   ▼
Publish
   │
   ▼
SNS
   │
   ▼
SQS
   │
   ▼
Receive
   │
   ▼
Validate
   │
   ▼
Idempotency
   │
   ▼
Business processing
   │
   ├── Success → Delete
   │
   └── Failure
          │
          ▼
        Retry
          │
          ▼
       DLQ if exhausted
```

This is the workflow you should visualize whenever you troubleshoot asynchronous systems.

---

# 70. Production Checklist

### SNS

* [ ] Topic naming convention
* [ ] Encryption
* [ ] Least-privilege publish permissions
* [ ] Subscription filtering where appropriate
* [ ] Monitoring
* [ ] Event schema/versioning

### SQS

* [ ] Standard vs FIFO decision
* [ ] Visibility timeout
* [ ] Long polling
* [ ] Message retention
* [ ] DLQ
* [ ] Redrive policy
* [ ] Encryption
* [ ] IAM
* [ ] Queue monitoring
* [ ] Consumer scaling

### Application

* [ ] Idempotency
* [ ] Retry policy
* [ ] Backoff
* [ ] Validation
* [ ] Correlation IDs
* [ ] Structured logging
* [ ] Error classification

### Operations

* [ ] Queue depth alerts
* [ ] Oldest message age alerts
* [ ] DLQ alerts
* [ ] Consumer health
* [ ] Processing latency
* [ ] Runbooks
* [ ] Replay/redrive procedure

---

# 71. Final Architecture

Our messaging architecture is now:

```text
                           EKS
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         order-service  payment-service inventory
              │
              ├──────► Redis
              │
              ├──────► RDS
              │
              └──────► SNS
                         │
             ┌───────────┼───────────┐
             │           │           │
            SQS         SQS         SQS
             │           │           │
          Payment     Inventory   Notification
             │           │           │
             ▼           ▼           ▼
          Service     Service      Service
```

And now our architecture has three different communication patterns:

```text
REST/gRPC
   ↓
Immediate synchronous communication

Redis
   ↓
Fast shared cache/state

SQS/SNS
   ↓
Asynchronous messaging/event fan-out
```

Later:

```text
RabbitMQ
   ↓
Advanced broker-based messaging

Kafka
   ↓
High-volume event streaming
```

That gives you a much more complete understanding of **how 20+ microservices communicate in a real production platform**, rather than treating every communication problem as REST.

[AWS SQS documentation](https://docs.aws.amazon.com/sqs/?utm_source=chatgpt.com)
[AWS SNS documentation](https://docs.aws.amazon.com/sns/?utm_source=chatgpt.com)
[RabbitMQ documentation](https://www.rabbitmq.com/docs?utm_source=chatgpt.com)
[Apache Kafka documentation](https://kafka.apache.org/documentation/?utm_source=chatgpt.com)

### Next file: `24-RabbitMQ.md`

We'll go deep into **RabbitMQ as a production platform**, including:

```text
RabbitMQ architecture
        ↓
Exchange
        ↓
Queue
        ↓
Binding
        ↓
Routing key
        ↓
Publisher confirms
        ↓
Consumer acknowledgements
        ↓
Prefetch
        ↓
Retries
        ↓
Dead-letter exchange
        ↓
Quorum queues
        ↓
RabbitMQ clustering
        ↓
Persistent storage
        ↓
EKS deployment
        ↓
Operator
        ↓
HA across AZs
        ↓
Monitoring
        ↓
Production troubleshooting
```

RabbitMQ's current documentation specifically emphasizes queues, durability, acknowledgements, publisher confirms, prefetch, and replicated queue types as core reliability concepts. ([RabbitMQ][6])

[1]: https://docs.aws.amazon.com/sqs/?utm_source=chatgpt.com "Amazon Simple Queue Service Documentation"
[2]: https://docs.aws.amazon.com/sns/latest/dg/sns-sqs-as-subscriber.html?utm_source=chatgpt.com "Fanout Amazon SNS notifications to Amazon SQS queues for asynchronous processing - Amazon Simple Notification Service"
[3]: https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-integrating-microservices/sns.html?utm_source=chatgpt.com "Amazon SNS - AWS Prescriptive Guidance"
[4]: https://www.rabbitmq.com/tutorials/amqp-concepts?utm_source=chatgpt.com "AMQP 0-9-1 Model Explained | RabbitMQ"
[5]: https://kafka.apache.org/08/documentation.html?utm_source=chatgpt.com "Introduction | Apache Kafka"
[6]: https://www.rabbitmq.com/docs/queues?utm_source=chatgpt.com "Queues | RabbitMQ"
