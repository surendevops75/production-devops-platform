# 114-SNS-Event-Architecture.md

````markdown
# 114 — SNS Event Architecture

## 1. Purpose

In the previous chapters, we implemented Amazon SQS as the asynchronous messaging layer.

SQS works well when one queue has a producer and one logical consumer group.

But our e-commerce platform has a different requirement.

When an order is created, multiple independent services may need to react:

```text
OrderCreated
     |
     +----> Payment
     |
     +----> Inventory
     |
     +----> Notification
     |
     +----> Fraud
     |
     +----> Reporting
````

If we use only one SQS queue:

```text
order-service
      |
      v
order-events queue
      |
      +----> payment-service
      +----> inventory-service
      +----> notification-service
```

the consumers compete for messages.

That is not what we want.

We want every interested service to receive its own copy.

Therefore:

```text
                         OrderCreated
                              |
                              v
                         SNS Topic
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
          SQS-Payment    SQS-Inventory    SQS-Notification
              |               |               |
              v               v               v
       payment-service  inventory-service notification-service
```

This is the **fan-out architecture**.

---

# 2. SNS vs SQS

The easiest way to remember the difference:

```text
SNS = distribute
SQS = buffer/process
```

SNS is primarily a publisher/subscriber service.

SQS is primarily a durable queue for consumers.

Together:

```text
Producer
   |
   v
SNS
   |
   +----> SQS
   |
   +----> SQS
   |
   +----> SQS
```

---

# 3. Why We Need SNS

Suppose:

```text
order-service
```

publishes:

```text
OrderCreated
```

Three services need the event:

```text
payment-service
inventory-service
notification-service
```

If they all consume one SQS queue:

```text
             SQS
          /   |   \
         /    |    \
    payment inventory notification
```

only one consumer receives a particular message.

Instead:

```text
                   SNS
                    |
       +------------+------------+
       |            |            |
       v            v            v
   SQS-Payment  SQS-Inventory  SQS-Notify
       |            |            |
       v            v            v
   Payment      Inventory     Notification
```

Each service gets an independent copy.

---

# 4. Fan-Out

This architecture is called:

```text
Fan-Out
```

One event:

```text
OrderCreated
```

is distributed to:

```text
Payment
Inventory
Notification
Fraud
Reporting
```

Each consumer can process independently.

---

# 5. Our Production Event Architecture

For the e-commerce platform:

```text
                         ┌─────────────────┐
                         │  order-service  │
                         └────────┬────────┘
                                  |
                                  | Publish
                                  v
                         ┌─────────────────┐
                         │ SNS Topic       │
                         │ order-events    │
                         └────────┬────────┘
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
       ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
       │ SQS Payment │     │SQS Inventory│     │ SQS Notify  │
       └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
              |                   |                   |
              v                   v                   v
       payment-service     inventory-service    notification-service
```

This is the architecture we will build.

---

# 6. Event Ownership

The event producer owns the event contract.

For example:

```text
OrderCreated
Producer:
order-service
```

Consumers:

```text
payment-service
inventory-service
notification-service
fraud-service
```

The producer should not need to know every consumer implementation.

This creates loose coupling.

---

# 7. Loose Coupling

Without events:

```text
order-service
   |
   +----> payment-service
   |
   +----> inventory-service
   |
   +----> notification-service
   |
   +----> fraud-service
```

The order service becomes tightly coupled to downstream services.

With SNS:

```text
order-service
      |
      v
   SNS topic
      |
      +----> consumers
```

The order service publishes an event.

It does not need to synchronously call every consumer.

---

# 8. Event Contract

Our standard event envelope:

```json
{
  "eventId": "evt-12345",
  "eventType": "OrderCreated",
  "eventVersion": "1",
  "occurredAt": "2026-09-28T10:30:00Z",
  "source": "order-service",
  "correlationId": "corr-12345",
  "payload": {
    "orderId": "ORD-10001",
    "customerId": "CUS-1001"
  }
}
```

Every event should have a stable contract.

---

# 9. Why eventId Matters

Suppose SNS delivers:

```text
evt-12345
```

to payment-service.

The consumer processes it.

If SQS subsequently redelivers the message:

```text
evt-12345
```

payment-service can recognize the duplicate.

Therefore:

```text
eventId
```

must remain stable across retries.

---

# 10. SNS Topic Naming

Use a predictable naming convention.

Example:

```text
ecommerce-prod-order-events
```

Other environments:

```text
ecommerce-dev-order-events
ecommerce-qa-order-events
ecommerce-staging-order-events
ecommerce-prod-order-events
```

Avoid:

```text
topic1
mytopic
orders-topic-final
```

---

# 11. Topic Categories

Possible topics:

```text
order-events
payment-events
inventory-events
customer-events
shipment-events
notification-events
```

Avoid creating one giant topic:

```text
all-events
```

unless the architecture has a clear reason for doing so.

A topic should represent a coherent event domain.

---

# 12. Terraform Architecture

Our infrastructure repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── sns/
    │   └── sqs/
    │
    └── environments/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

---

# 13. SNS Topic Terraform

Example:

```hcl
resource "aws_sns_topic" "order_events" {
  name = "ecommerce-${var.environment}-order-events"

  tags = {
    Environment = var.environment
    Domain      = "order"
    ManagedBy   = "terraform"
  }
}
```

---

# 14. SNS Encryption

For production:

```hcl
resource "aws_sns_topic" "order_events" {
  name = "ecommerce-${var.environment}-order-events"

  kms_master_key_id = aws_kms_key.messaging.arn
}
```

The exact KMS key architecture depends on whether messaging uses:

```text
one shared key
```

or:

```text
domain-specific keys
```

For sensitive environments, key separation may be appropriate.

---

# 15. SQS Queues

Create one queue per logical consumer.

Example:

```text
ecommerce-prod-payment-events
ecommerce-prod-inventory-events
ecommerce-prod-notification-events
```

And their DLQs:

```text
ecommerce-prod-payment-events-dlq
ecommerce-prod-inventory-events-dlq
ecommerce-prod-notification-events-dlq
```

---

# 16. Complete Terraform Flow

Conceptually:

```text
SNS Topic
   |
   +---- Subscription ----> Payment SQS
   |
   +---- Subscription ----> Inventory SQS
   |
   +---- Subscription ----> Notification SQS
```

---

# 17. SNS to SQS Subscription

Terraform:

```hcl
resource "aws_sns_topic_subscription" "payment" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.payment_events.arn
}
```

Inventory:

```hcl
resource "aws_sns_topic_subscription" "inventory" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.inventory_events.arn
}
```

Notification:

```hcl
resource "aws_sns_topic_subscription" "notification" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.notification_events.arn
}
```

---

# 18. Critical Piece: SQS Queue Policy

Creating the subscription is not enough.

SNS must be allowed to send messages to the SQS queue.

Therefore:

```text
SNS Topic
   |
   | SendMessage
   v
SQS Queue
```

requires an appropriate queue policy.

---

# 19. Queue Policy

Conceptual example:

```hcl
data "aws_iam_policy_document" "payment_queue" {

  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["sns.amazonaws.com"]
    }

    actions = [
      "sqs:SendMessage"
    ]

    resources = [
      aws_sqs_queue.payment_events.arn
    ]

    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values = [
        aws_sns_topic.order_events.arn
      ]
    }
  }
}
```

The important protection is:

```text
Only the intended SNS topic
```

should be allowed to publish into the queue.

---

# 20. Attach Queue Policy

Example:

```hcl
resource "aws_sqs_queue_policy" "payment" {
  queue_url = aws_sqs_queue.payment_events.id
  policy    = data.aws_iam_policy_document.payment_queue.json
}
```

Repeat for each consumer queue.

---

# 21. Why SourceArn Matters

Without restriction, the policy could accidentally permit too much.

We want:

```text
SNS order-events
       |
       v
payment-events queue
```

not:

```text
Any SNS topic
       |
       v
payment-events queue
```

Therefore:

```text
aws:SourceArn
```

is important.

---

# 22. Raw Message Delivery

SNS-to-SQS subscriptions can wrap the published message in an SNS envelope.

Conceptually:

```json
{
  "Type": "Notification",
  "TopicArn": "...",
  "Message": "{...event...}"
}
```

The consumer then extracts:

```text
Message
```

before processing the event.

---

# 23. Raw Message Delivery

SNS supports raw message delivery for supported subscription types.

Conceptually:

```hcl
resource "aws_sns_topic_subscription" "payment" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.payment_events.arn
  raw_message_delivery = true
}
```

With raw delivery:

```text
SQS
 ↓
Event JSON directly
```

rather than:

```text
SQS
 ↓
SNS envelope
 ↓
Event JSON
```

Choose deliberately based on the consumer contract and metadata requirements.

---

# 24. SNS Message Attributes

SNS supports message attributes.

Example:

```text
eventType = OrderCreated
eventVersion = 1
domain = order
```

These can be used for subscription filtering.

---

# 25. Subscription Filtering

Suppose the topic receives:

```text
OrderCreated
OrderCancelled
OrderShipped
PaymentCompleted
```

Payment queue may only need:

```text
PaymentCompleted
```

or another defined subset.

SNS subscription filtering can reduce unnecessary delivery.

Architecture:

```text
                    SNS
                     |
        +------------+------------+
        |                         |
    Filter A                  Filter B
        |                         |
        v                         v
     Queue A                    Queue B
```

---

# 26. Example Filter Policy

Conceptually:

```json
{
  "eventType": [
    "OrderCreated",
    "OrderCancelled"
  ]
}
```

Only matching messages are delivered to that subscription.

This is useful when a topic contains multiple event types.

---

# 27. Filter Policy Design

Do not rely on filtering as a security boundary.

Filtering is primarily an event-routing mechanism.

Authorization still comes from:

```text
IAM
Queue policy
SNS topic policy
KMS
```

---

# 28. Topic Policy

SNS topic policies can restrict who can publish.

For example:

```text
order-service IAM role
```

may be allowed to:

```text
sns:Publish
```

to:

```text
ecommerce-prod-order-events
```

Only.

---

# 29. Producer IAM

Order service:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity
 ↓
IAM role
 ↓
SNS
```

IAM:

```hcl
actions = [
  "sns:Publish"
]

resources = [
  aws_sns_topic.order_events.arn
]
```

Do not grant:

```text
sns:*
Resource = "*"
```

to application roles.

---

# 30. Consumer IAM

Consumers do not normally need permission to receive from SNS when SNS is delivering to SQS.

The consumer needs permissions to read its SQS queue:

```text
payment-service
    |
    +--> ReceiveMessage
    +--> DeleteMessage
    +--> GetQueueAttributes
```

SNS handles the publication to SQS.

---

# 31. End-to-End IAM

```text
                    order-service
                         |
                         v
                    IAM Role A
                         |
                  sns:Publish
                         |
                         v
                     SNS Topic
                         |
                         v
                    SQS Queue
                         |
                         v
                  IAM Role B
                         |
                         v
                 payment-service
```

Role A and Role B should normally be different.

---

# 32. Cross-Account Architecture

Enterprise example:

```text
Application Account
       |
       v
SNS Topic
       |
       v
Shared Services Account
       |
       v
SQS Queue
```

This requires:

```text
SNS topic policy
SQS queue policy
IAM
KMS policies
Account IDs
```

Cross-account permissions must be tested carefully.

---

# 33. SNS Message Flow

Complete flow:

```text
1. order-service creates order

2. order-service publishes OrderCreated

3. SNS receives event

4. SNS evaluates subscriptions

5. SNS sends copy to payment SQS

6. SNS sends copy to inventory SQS

7. SNS sends copy to notification SQS

8. Each consumer processes independently
```

---

# 34. Failure Isolation

Suppose:

```text
payment-service = DOWN
```

Payment queue:

```text
payment-events
    |
    v
backlog increases
```

But:

```text
inventory-service = HEALTHY
notification-service = HEALTHY
```

Their queues can continue processing.

This is one of the biggest advantages of the fan-out design.

---

# 35. What Happens Without Independent Queues?

If all consumers share one queue:

```text
Queue
 |
 +--> payment
 +--> inventory
 +--> notification
```

a consumer failure can affect which messages are consumed by the other consumers.

With independent queues:

```text
SNS
 |
 +--> payment queue
 |
 +--> inventory queue
 |
 +--> notification queue
```

each service has its own backlog and scaling policy.

---

# 36. Independent Scaling

Payment:

```text
2 replicas
```

Inventory:

```text
5 replicas
```

Notification:

```text
10 replicas
```

Each scales independently.

For example:

```text
payment queue
    ↓
KEDA
    ↓
payment-service
```

and:

```text
notification queue
    ↓
KEDA
    ↓
notification-service
```

---

# 37. Independent DLQs

Each queue should have its own DLQ.

```text
SNS
 |
 +--> Payment Queue --> Payment DLQ
 |
 +--> Inventory Queue --> Inventory DLQ
 |
 +--> Notification Queue --> Notification DLQ
```

Do not automatically put every consumer's failed messages into one generic DLQ.

Separate DLQs make ownership and recovery clearer.

---

# 38. Example

Payment processing fails:

```text
Payment Queue
     |
     v
Payment DLQ
```

Inventory remains:

```text
Inventory Queue
     |
     v
Inventory Service
```

Notification remains:

```text
Notification Queue
     |
     v
Notification Service
```

This provides failure isolation.

---

# 39. SNS Durability vs Consumer Processing

A successful SNS publish does not mean:

```text
Payment processed
```

It means the event was accepted by the messaging layer.

Business processing occurs later:

```text
SNS
 ↓
SQS
 ↓
Consumer
 ↓
Business operation
```

Therefore the order API should not claim:

```text
Payment completed
```

merely because:

```text
SNS publish succeeded
```

---

# 40. Eventual Consistency

With asynchronous architecture:

```text
Order Created
```

may become visible immediately while:

```text
Payment
Inventory
Notification
```

complete later.

This is:

```text
Eventual consistency
```

The business workflow must explicitly support this.

---

# 41. Order State Machine

For example:

```text
PENDING
   |
   v
PAYMENT_PROCESSING
   |
   +----> PAYMENT_FAILED
   |
   v
PAYMENT_COMPLETED
   |
   v
INVENTORY_RESERVED
   |
   v
CONFIRMED
```

The exact state machine depends on business requirements.

---

# 42. Do Not Hide Asynchronous State

An API response could return:

```json
{
  "orderId": "ORD-10001",
  "status": "PENDING"
}
```

rather than falsely returning:

```json
{
  "status": "COMPLETED"
}
```

before downstream processing actually completes.

---

# 43. Event Correlation

The event should carry:

```text
correlationId
```

Example:

```text
corr-12345
```

Then:

```text
order-service
   |
   +--> SNS
         |
         +--> payment
         +--> inventory
         +--> notification
```

All logs can reference:

```text
corr-12345
```

This makes incident investigation much easier.

---

# 44. Message Attributes vs Event Body

Use message attributes for routing metadata when appropriate.

Example:

```text
eventType = OrderCreated
domain = order
version = 1
```

Keep business data in the event body.

Do not create an architecture where consumers need to inspect arbitrary infrastructure metadata to understand the business event.

---

# 45. Event Versioning

Example:

```text
OrderCreated v1
OrderCreated v2
```

A new field can be added:

```json
{
  "orderId": "ORD-10001",
  "customerId": "CUS-1001",
  "currency": "INR"
}
```

Existing consumers should continue functioning if the change is backward compatible.

---

# 46. Breaking Event Change

Bad:

```text
v1:
orderId

v2:
id
```

if consumers only understand:

```text
orderId
```

and the old field disappears.

Better:

```text
v2:
orderId
id
```

during a migration period, or introduce a compatible versioning strategy.

---

# 47. Event Schema Ownership

The producer should document:

```text
eventType
version
required fields
optional fields
field types
meaning
compatibility rules
```

Example:

```text
OrderCreated v1
```

Documentation:

```text
eventId: required
orderId: required
customerId: required
currency: optional
```

---

# 48. SNS vs EventBridge

Both can be used for event-driven architectures.

SNS is commonly useful for:

```text
High-throughput fan-out
Simple pub/sub
SNS → SQS
Application notifications
```

EventBridge is useful for:

```text
Event buses
AWS service events
Event routing
Rules
Cross-account event architectures
SaaS integrations
Complex event patterns
```

They are not identical replacements.

Our current design uses:

```text
SNS → SQS
```

because our primary requirement is application event fan-out.

---

# 49. SNS vs SQS

| Capability                   | SNS                                                 | SQS              |
| ---------------------------- | --------------------------------------------------- | ---------------- |
| Main role                    | Pub/Sub                                             | Queue            |
| Fan-out                      | Yes                                                 | No               |
| Consumer buffering           | Limited role                                        | Yes              |
| Independent consumer backlog | Through subscribed queues                           | Yes              |
| DLQ                          | Supported in relevant subscription/delivery designs | Native queue DLQ |
| Polling                      | No                                                  | Yes              |
| Long polling                 | No                                                  | Yes              |
| KEDA queue scaling           | Not directly                                        | Yes              |

---

# 50. SNS + SQS Pattern

The recommended architecture for our use case:

```text
Producer
   |
   v
SNS
   |
   +--> SQS A --> Consumer A
   |
   +--> SQS B --> Consumer B
   |
   +--> SQS C --> Consumer C
```

This gives us:

```text
Fan-out
+
Durable buffering
+
Independent scaling
+
Independent retries
+
Independent DLQs
```

---

# 51. SQS-Only Pattern

Use SQS alone when:

```text
One logical consumer group
```

Example:

```text
order-service
     |
     v
order-processing-queue
     |
     v
order-worker replicas
```

The workers compete for messages.

This is appropriate.

---

# 52. SNS + SQS Pattern

Use SNS + SQS when:

```text
One event
+
Multiple independent consumers
```

Example:

```text
OrderCreated
     |
     +--> payment
     +--> inventory
     +--> notification
```

---

# 53. Terraform Module Design

Recommended module:

```text
modules/
└── messaging/
    ├── sns-topic/
    ├── sqs-queue/
    └── sns-sqs-subscription/
```

Or a domain-specific module:

```text
modules/
└── event-topic/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

The module should standardize:

```text
Encryption
Tags
Retention
DLQ
Queue policies
Subscriptions
```

---

# 54. Example Module Inputs

```hcl
variable "topic_name" {
  type = string
}

variable "subscriptions" {
  type = map(object({
    queue_arn = string
    filter_policy = optional(string)
  }))
}
```

This lets us create:

```text
one topic
+
multiple queues
+
multiple subscriptions
```

using reusable Terraform.

---

# 55. Example Environment Configuration

Conceptually:

```hcl
module "order_events" {

  source = "../../modules/event-topic"

  topic_name = "ecommerce-prod-order-events"

  subscriptions = {
    payment = {
      queue_arn = module.payment_queue.arn
    }

    inventory = {
      queue_arn = module.inventory_queue.arn
    }

    notification = {
      queue_arn = module.notification_queue.arn
    }
  }
}
```

This is much better than duplicating infrastructure manually.

---

# 56. GitOps Boundary

SNS and SQS infrastructure belongs in:

```text
infrastructure repo
```

not:

```text
application repo
```

Application code publishes to:

```text
SNS
```

but Terraform manages:

```text
SNS
SQS
DLQs
Policies
KMS
Subscriptions
```

---

# 57. Kubernetes Boundary

Kubernetes manages:

```text
Deployment
Service
ServiceAccount
HPA/KEDA
NetworkPolicy
ConfigMap
ExternalSecret
```

AWS Terraform manages:

```text
SNS
SQS
KMS
IAM
```

ArgoCD manages:

```text
Kubernetes desired state
```

This keeps responsibilities clear.

---

# 58. Configuration

Application configuration should reference:

```text
SNS topic ARN
```

or:

```text
topic name
```

without hardcoding environment-specific infrastructure throughout the code.

Example:

```yaml
messaging:
  order-events-topic-arn: ${ORDER_EVENTS_TOPIC_ARN}
```

Environment values can come from:

```text
GitOps values
ConfigMap
Secrets where appropriate
```

Do not store secrets in ConfigMaps.

---

# 59. Producer Application Flow

Spring Boot:

```text
OrderController
      |
      v
OrderService
      |
      v
OrderRepository
      |
      v
PostgreSQL
      |
      v
Event Publisher
      |
      v
SNS
```

The event publisher should be isolated behind an application abstraction.

Example:

```java
public interface EventPublisher {

    void publish(OrderCreatedEvent event);
}
```

---

# 60. SNS Publisher

Conceptually:

```java
@Service
public class SnsEventPublisher implements EventPublisher {

    private final SnsClient snsClient;

    public void publish(OrderCreatedEvent event) {

        PublishRequest request =
            PublishRequest.builder()
                .topicArn(topicArn)
                .message(toJson(event))
                .build();

        snsClient.publish(request);
    }
}
```

The exact AWS SDK configuration should use the application's EKS workload identity.

---

# 61. No AWS Credentials in Code

Never:

```java
AwsBasicCredentials.create(
    "ACCESS_KEY",
    "SECRET_KEY"
);
```

inside the application.

Instead:

```text
EKS Pod
 ↓
Pod Identity
 ↓
IAM Role
 ↓
AWS SDK default credential provider chain
```

---

# 62. AWS SDK Region

Configure the AWS region using environment/configuration where appropriate:

```text
AWS_REGION=ap-south-1
```

The AWS SDK can then use the workload's runtime environment and credential provider chain.

Avoid hardcoding:

```text
us-east-1
```

into business logic.

---

# 63. Publishing Failure

Suppose:

```text
Order saved
   |
   v
SNS publish fails
```

Now:

```text
Database:
Order exists

SNS:
No event
```

This is the same dual-write problem discussed earlier.

---

# 64. The Outbox Pattern

A robust solution is:

```text
Order transaction
      |
      +--> Orders table
      |
      +--> Outbox table
```

Both are written in one database transaction.

Then:

```text
Outbox Publisher
      |
      v
SNS
```

Architecture:

```text
order-service
     |
     v
PostgreSQL
  |       |
  |       +--> outbox_events
  |
  +--> orders
          |
          v
     Outbox Publisher
          |
          v
         SNS
```

This will become an important pattern in our event-driven architecture chapter.

---

# 65. SNS Delivery Failure

Suppose:

```text
SNS
 |
 +--> Payment SQS
 |
 X
Inventory SQS
```

SNS handles delivery according to the subscription/delivery model.

The important architectural point is:

```text
Payment queue
```

should not prevent:

```text
Notification queue
```

from independently processing its copy.

For robust SQS consumers, configure appropriate failure handling and DLQs at the queue/subscription level.

---

# 66. Queue-Level DLQ

Each queue:

```text
payment-events
```

has:

```text
payment-events-dlq
```

Then:

```text
SNS
 |
 +--> payment queue --> payment DLQ
 |
 +--> inventory queue --> inventory DLQ
 |
 +--> notification queue --> notification DLQ
```

This isolates failures.

---

# 67. Monitoring Architecture

We will monitor:

```text
SNS
 |
 +--> publish activity
 |
 +--> delivery failures where applicable
 |
 v
SQS
 |
 +--> queue depth
 +--> oldest message
 +--> not-visible messages
 +--> DLQ
 |
 v
Consumer
 |
 +--> processing latency
 +--> errors
 +--> duplicate events
```

---

# 68. Grafana View

Later we can build a dashboard:

```text
ORDER EVENT PIPELINE

SNS publishes                 12,500/min

Payment queue                 120
Payment oldest message        4 sec

Inventory queue               45
Inventory oldest message      2 sec

Notification queue            320
Notification oldest message   15 sec

Payment DLQ                   0
Inventory DLQ                 0
Notification DLQ              3
```

This immediately shows where the problem is.

---

# 69. Alert Examples

### Payment backlog

```text
Payment queue age > SLO
```

### Inventory backlog

```text
Inventory queue depth > threshold
```

### Notification DLQ

```text
Notification DLQ > 0
```

### Consumer failure

```text
Consumer error rate > threshold
```

---

# 70. Failure Scenario: Payment Service Down

Architecture:

```text
SNS
 |
 +--> Payment Queue ----> X Consumer
 |
 +--> Inventory Queue --> Inventory
 |
 +--> Notification -----> Notification
```

Expected:

```text
Payment queue backlog increases
```

while:

```text
Inventory continues
Notification continues
```

This is correct isolation behavior.

---

# 71. Failure Scenario: SNS Publisher Down

If order-service cannot publish:

```text
Order API
   |
   v
Order DB
   |
   X
SNS
```

The outbox pattern becomes important.

Without an outbox:

```text
Order exists
Event missing
```

With an outbox:

```text
Order exists
Outbox event exists
```

The publisher can retry later.

---

# 72. Failure Scenario: Consumer Crash

```text
Payment Queue
      |
      v
Payment Pod
      |
      X
Crash
```

Message becomes available again after visibility timeout.

New Pod:

```text
Payment Pod
      |
      v
Receive
      |
      v
Idempotency
      |
      v
Process
```

---

# 73. Failure Scenario: Poison Event

```text
SNS
 |
 v
Payment Queue
 |
 v
Consumer
 |
 X
Invalid event
 |
 v
Retry
 |
 v
DLQ
```

Payment team investigates.

Inventory and notification remain unaffected.

---

# 74. Failure Scenario: Consumer Overload

Suppose queue depth increases.

KEDA:

```text
Queue depth
    ↓
More replicas
```

If nodes are full:

```text
Pods Pending
    ↓
Karpenter
    ↓
New nodes
```

If the database is already saturated:

```text
DO NOT
```

scale consumers indefinitely.

Investigate downstream capacity first.

---

# 75. Event-Driven Scaling

Full chain:

```text
Order traffic increases
       ↓
More events
       ↓
SNS
       ↓
SQS queues
       ↓
Queue backlog
       ↓
KEDA
       ↓
More Pods
       ↓
Karpenter
       ↓
More nodes
       ↓
Consumers drain queues
```

This is one of the central scaling patterns of the platform.

---

# 76. Security Model

```text
Producer
   |
   | sns:Publish
   v
SNS Topic
   |
   | subscription
   v
SQS Queue
   |
   | receive/delete
   v
Consumer
```

Security controls:

```text
IAM
Queue policies
SNS topic policies
KMS
Pod Identity
Network controls where applicable
CloudTrail
Application authorization
```

---

# 77. Least Privilege

Order service:

```text
sns:Publish
```

Payment service:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Inventory service:

```text
same permissions
```

but against:

```text
inventory queue
```

Notification service:

```text
same permissions
```

against:

```text
notification queue
```

Do not share broad roles unnecessarily.

---

# 78. Event Data Security

Do not publish:

```text
Credit card number
CVV
Password
AWS credentials
Database password
Session tokens
```

into events.

Instead publish identifiers:

```json
{
  "orderId": "ORD-10001",
  "customerId": "CUS-1001"
}
```

Consumers retrieve sensitive information through authorized systems when necessary.

---

# 79. Event Payload Size

If the event contains a large document:

```text
Don't:
SNS → huge document
```

Prefer:

```text
S3
 |
 +--> object
 |
 +--> SNS event with S3 reference
```

Example:

```json
{
  "eventType": "InvoiceGenerated",
  "invoiceId": "INV-10001",
  "bucket": "ecommerce-prod-invoices",
  "key": "2026/09/INV-10001.pdf"
}
```

---

# 80. Auditability

Every important event should be traceable:

```text
Order ID
Event ID
Correlation ID
Producer
Timestamp
Event version
Consumer
Processing result
```

Then an incident can answer:

```text
Where did this event originate?
Which consumers received it?
Which consumer failed?
Was it retried?
Did it reach DLQ?
Was it replayed?
```

---

# 81. Production Topic Inventory

Example:

| Topic            | Producer          | Consumers                               |
| ---------------- | ----------------- | --------------------------------------- |
| order-events     | order-service     | payment, inventory, notification, fraud |
| payment-events   | payment-service   | order, notification, reporting          |
| inventory-events | inventory-service | order, notification                     |
| shipment-events  | shipping-service  | order, notification, reporting          |

This becomes our event catalog.

---

# 82. Event Catalog

For every event document:

```text
Event Name:
OrderCreated

Producer:
order-service

Topic:
ecommerce-prod-order-events

Consumers:
payment-service
inventory-service
notification-service
fraud-service

Version:
v1

Required Fields:
eventId
orderId
customerId

Compatibility:
Backward compatible
```

This prevents undocumented event dependencies.

---

# 83. Ownership Model

```text
Event
 ↓
Producer team owns contract
 ↓
Consumer teams own processing
 ↓
Platform team owns messaging infrastructure
 ↓
Security team reviews IAM/KMS/policies
```

This matches our existing team structure.

---

# 84. Repository Responsibilities

### Application repo

```text
order-service/
├── src/
├── Dockerfile
├── helm/
└── .github/
```

contains:

```text
Event publisher code
Event models
Consumer code
```

### Infrastructure repo

```text
infrastructure/
```

contains:

```text
SNS
SQS
DLQ
IAM
KMS
Subscriptions
Policies
```

### GitOps repo

```text
gitops/
```

contains:

```text
Deployment values
KEDA
ServiceAccount
ExternalSecret
NetworkPolicy
```

---

# 85. Complete Repository Flow

```text
Developer
    |
    v
order-service repository
    |
    v
CI
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
    v
order-service
    |
    v
SNS
    |
    +------> SQS Payment
    |
    +------> SQS Inventory
    |
    +------> SQS Notification
```

Infrastructure:

```text
Terraform
    |
    +--> SNS
    +--> SQS
    +--> DLQ
    +--> IAM
    +--> KMS
```

---

# 86. Hands-On Implementation Order

Implement in this order:

```text
1. Create KMS key
2. Create SNS topic
3. Create payment DLQ
4. Create payment queue
5. Create inventory DLQ
6. Create inventory queue
7. Create notification DLQ
8. Create notification queue
9. Configure queue policies
10. Create SNS subscriptions
11. Configure filters if required
12. Configure IAM
13. Deploy producer
14. Deploy consumers
15. Publish test event
16. Verify all queues
17. Verify consumers
18. Test duplicate delivery
19. Test consumer failure
20. Test DLQ
```

---

# 87. Terraform Validation

Run:

```bash
terraform fmt -check
```

Then:

```bash
terraform validate
```

Then:

```bash
terraform plan
```

Review:

```text
SNS topic
SQS queues
DLQs
subscriptions
IAM
KMS
queue policies
```

Only then:

```bash
terraform apply
```

for the appropriate environment.

---

# 88. Verify SNS

AWS CLI:

```bash
aws sns list-topics \
  --region ap-south-1
```

Find:

```text
ecommerce-prod-order-events
```

---

# 89. Verify Subscriptions

```bash
aws sns list-subscriptions \
  --region ap-south-1
```

Expected:

```text
order-events
   |
   +--> payment queue
   +--> inventory queue
   +--> notification queue
```

---

# 90. Verify Queues

```bash
aws sqs list-queues \
  --region ap-south-1
```

Expected:

```text
ecommerce-prod-payment-events
ecommerce-prod-inventory-events
ecommerce-prod-notification-events
```

---

# 91. Publish Test Event

Example:

```bash
aws sns publish \
  --topic-arn "$TOPIC_ARN" \
  --message '{
    "eventId": "test-001",
    "eventType": "OrderCreated",
    "eventVersion": "1",
    "source": "order-service",
    "correlationId": "test-correlation-001",
    "payload": {
      "orderId": "TEST-001"
    }
  }'
```

---

# 92. Expected Result

After publication:

```text
SNS
 |
 +--> Payment Queue
 |
 +--> Inventory Queue
 |
 +--> Notification Queue
```

Each queue should receive a copy.

Then:

```text
Payment consumer
Inventory consumer
Notification consumer
```

process independently.

---

# 93. Verify Payment Queue

```bash
aws sqs receive-message \
  --queue-url "$PAYMENT_QUEUE_URL" \
  --max-number-of-messages 1 \
  --wait-time-seconds 5
```

Verify the event.

Repeat for:

```text
inventory queue
notification queue
```

---

# 94. Verify Fan-Out

The test is successful when:

```text
One SNS publish
       |
       +--> Payment received
       |
       +--> Inventory received
       |
       +--> Notification received
```

This confirms fan-out.

---

# 95. Test Failure Isolation

Stop payment-service in DEV:

```bash
kubectl scale deployment/payment-service \
  --replicas=0 \
  -n ecommerce-dev
```

Publish events.

Expected:

```text
Payment queue:
BACKLOG

Inventory:
PROCESSING

Notification:
PROCESSING
```

This proves independent consumer queues.

Restore:

```bash
kubectl scale deployment/payment-service \
  --replicas=2 \
  -n ecommerce-dev
```

Then verify the payment queue drains.

---

# 96. Test Duplicate Handling

Publish the same:

```text
eventId=test-duplicate-001
```

twice.

Expected:

```text
First:
Business operation executes

Second:
Idempotency detects duplicate
```

This test should be automated later.

---

# 97. Test DLQ

In DEV:

```text
Send invalid event
```

Consumer:

```text
Validation fails
```

After configured retries:

```text
Main queue
   ↓
DLQ
```

Verify:

```bash
aws sqs receive-message \
  --queue-url "$DLQ_URL" \
  --max-number-of-messages 1
```

---

# 98. Troubleshooting: No Message in Queue

Check:

```text
1. Topic ARN
2. Subscription
3. Subscription confirmation where relevant
4. Queue policy
5. SNS topic policy
6. KMS
7. Filter policy
8. Region
```

A filter policy can silently explain why a message is not delivered to a particular subscription.

---

# 99. Troubleshooting: AccessDenied Publishing

Check:

```text
Pod Identity
 ↓
IAM role
 ↓
sns:Publish
 ↓
Correct topic ARN
```

Use:

```bash
kubectl describe pod \
  -n ecommerce-prod \
  <pod-name>
```

and inspect the workload identity configuration.

---

# 100. Troubleshooting: Queue AccessDenied

Check:

```text
Consumer ServiceAccount
IAM role
sqs:ReceiveMessage
sqs:DeleteMessage
queue ARN
region
```

Do not solve authorization problems with AdministratorAccess.

---

# 101. Troubleshooting: SNS to SQS Not Working

Check:

```text
SNS topic
       |
       v
Subscription
       |
       v
SQS queue policy
       |
       v
SourceArn
       |
       v
KMS
```

The queue policy is one of the most common configuration points to inspect.

---

# 102. Troubleshooting: Messages Repeatedly Appear

Possible causes:

```text
Consumer processing takes too long
DeleteMessage fails
Consumer crashes
Visibility timeout too short
Network problems
Idempotency incorrectly implemented
```

Start with:

```text
CloudWatch
+
consumer logs
+
processing latency
+
visibility timeout
```

---

# 103. Troubleshooting: Queue Backlog

Check:

```text
Producer rate
Consumer rate
Consumer replicas
KEDA
Karpenter
Pod Pending
Database
External APIs
Redis
CPU
Memory
```

Do not immediately increase replicas.

Find the bottleneck first.

---

# 104. Production Architecture Summary

Our event-driven architecture is now:

```text
                         ┌────────────────────┐
                         │    order-service   │
                         └─────────┬──────────┘
                                   |
                                   | sns:Publish
                                   v
                         ┌────────────────────┐
                         │    SNS Topic       │
                         │   order-events     │
                         └─────────┬──────────┘
                                   |
                 +-----------------+-----------------+
                 |                 |                 |
                 v                 v                 v
        ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
        │ Payment Queue  │ │Inventory Queue │ │Notification Q. │
        └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
                |                  |                  |
                v                  v                  v
        payment-service    inventory-service   notification-service
                |                  |                  |
                v                  v                  v
           payment DB         inventory DB       notification API
```

Each queue has:

```text
DLQ
Retry
Idempotency
KEDA
Monitoring
```

---

# 105. Final Mental Model

Remember:

```text
SNS = FAN-OUT
SQS = BUFFER
DLQ = FAILURE ISOLATION
IDEMPOTENCY = DUPLICATE PROTECTION
KEDA = EVENT-DRIVEN SCALING
KARPENTER = NODE CAPACITY
IAM = AUTHORIZATION
KMS = ENCRYPTION
GITOPS = KUBERNETES DELIVERY
TERRAFORM = AWS INFRASTRUCTURE
```

Together:

```text
                  APPLICATION
                      |
                      v
                    SNS
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
         SQS         SQS         SQS
          |           |           |
        KEDA        KEDA        KEDA
          |           |           |
        Pods        Pods        Pods
          |           |           |
        DB/API      DB/API      DB/API
          |
        Failure
          |
          v
         DLQ
          |
          v
      Investigation
          |
          v
     Controlled Replay
```

---

# 106. Production Checklist

## SNS

```text
[ ] Topic naming standardized
[ ] Encryption enabled
[ ] Topic policy restricted
[ ] Producer IAM least privilege
[ ] Event contract documented
[ ] Event versioning defined
[ ] Correlation ID included
[ ] Event ID included
[ ] Subscription filters reviewed
```

## SQS

```text
[ ] One queue per independent consumer
[ ] Queue encryption enabled
[ ] Long polling enabled
[ ] Visibility timeout tuned
[ ] Retention configured
[ ] DLQ configured
[ ] Queue policy restricted
[ ] Consumer IAM restricted
```

## Consumers

```text
[ ] Idempotency
[ ] Retry strategy
[ ] Graceful shutdown
[ ] Timeouts
[ ] Circuit breaker where appropriate
[ ] Resource requests/limits
[ ] KEDA
[ ] Karpenter capacity
```

## Operations

```text
[ ] Queue depth monitoring
[ ] Oldest message monitoring
[ ] DLQ monitoring
[ ] Consumer error alerts
[ ] Event correlation
[ ] Runbook
[ ] Controlled replay procedure
[ ] Ownership documented
```

---

# 107. Next Chapter

## 115-RabbitMQ-Production-Implementation.md

We now move to the third messaging technology in our platform study:

```text
RabbitMQ
```

The next chapter will implement RabbitMQ as a production messaging platform and cover:

```text
RabbitMQ
    |
    +-- Exchanges
    +-- Queues
    +-- Bindings
    +-- Routing Keys
    +-- Direct Exchange
    +-- Topic Exchange
    +-- Fanout Exchange
    +-- Headers Exchange
    +-- ACK/NACK
    +-- Prefetch
    +-- Publisher Confirms
    +-- Retry
    +-- Dead Letter Exchange
    +-- Quorum Queues
    +-- HA
    +-- TLS
    +-- Authentication
    +-- Kubernetes Operator
    +-- Amazon MQ
    +-- KEDA
    +-- Monitoring
    +-- Failure Testing
```

We will also compare:

```text
SQS
SNS + SQS
RabbitMQ
Kafka
```

and determine **which messaging pattern fits which workload** in our 21-microservice e-commerce platform.

```
```
