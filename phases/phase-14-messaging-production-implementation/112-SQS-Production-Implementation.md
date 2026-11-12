# 112-SQS-Production-Implementation.md

````markdown
# 112 — SQS Production Implementation

## 1. Purpose

This chapter implements Amazon SQS as the asynchronous messaging layer for our e-commerce platform.

The goal is to understand how a real production system uses SQS between microservices instead of making every operation synchronous.

Our architecture will evolve from:

```text
Client
  ↓
order-service
  ↓
payment-service
  ↓
notification-service
````

to:

```text
Client
  ↓
order-service
  ↓
SQS
  ↓
Consumers
  ├── payment-service
  ├── notification-service
  └── inventory-service
```

The major concepts covered are:

* SQS Standard queues
* SQS FIFO queues
* queue creation
* Terraform
* IAM
* KMS encryption
* producers
* consumers
* visibility timeout
* long polling
* message retention
* retries
* DLQ
* idempotency
* Kubernetes consumers
* KEDA
* monitoring
* backlog management
* failure scenarios
* production troubleshooting

---

# 2. Why Asynchronous Messaging?

Consider order creation.

A synchronous architecture might be:

```text
POST /orders
      |
      v
order-service
      |
      v
payment-service
      |
      v
inventory-service
      |
      v
notification-service
      |
      v
Response
```

The order request now depends on every downstream service.

If notification-service is slow:

```text
notification-service slow
        ↓
order-service waits
        ↓
user waits
```

This increases coupling.

---

# 3. Asynchronous Architecture

With SQS:

```text
POST /orders
      |
      v
order-service
      |
      v
SQS
      |
      v
HTTP response
```

Then consumers process the event asynchronously:

```text
SQS
 |
 +----> payment-service
 |
 +----> inventory-service
 |
 +----> notification-service
```

The order service does not have to synchronously wait for every downstream operation.

---

# 4. Important SQS Principle

SQS is a queue.

A message is normally consumed by one consumer from a queue.

Example:

```text
                 SQS Queue
                    |
          +---------+---------+
          |                   |
     consumer-1          consumer-2
```

If both consumers compete for messages:

```text
Message A → consumer-1
Message B → consumer-2
Message C → consumer-1
```

They are competing consumers.

If multiple independent services each need the same event, we normally use:

```text
SNS/EventBridge
       |
   +---+---+---+
   |       |   |
   v       v   v
SQS-A   SQS-B SQS-C
```

That fan-out architecture will be covered in the SNS chapter.

---

# 5. Our Order Event

When an order is created:

```text
order-service
     |
     v
order-events queue
```

Example message:

```json
{
  "eventId": "evt-12345",
  "eventType": "OrderCreated",
  "eventVersion": "1",
  "occurredAt": "2026-09-28T10:30:00Z",
  "source": "order-service",
  "orderId": "10001",
  "customerId": "customer-001"
}
```

The event should contain enough information for the consumer to process it.

Avoid putting unnecessary sensitive information into messages.

---

# 6. Event vs Command

These concepts are different.

### Event

```text
OrderCreated
```

means:

> Something happened.

### Command

```text
ProcessPayment
```

means:

> Please perform an action.

Our SQS architecture should define clearly whether a queue carries:

```text
events
```

or:

```text
commands
```

Do not mix semantics accidentally.

---

# 7. SQS Standard Queue

The Standard queue provides high throughput and at-least-once delivery semantics.

Conceptually:

```text
Producer
   |
   v
SQS Standard
   |
   v
Consumer
```

Standard queues are appropriate when:

```text
Very high throughput
Ordering is not mandatory
Application supports duplicate messages
```

This is often a good default for asynchronous microservice workloads.

---

# 8. SQS FIFO Queue

FIFO means:

```text
First In
First Out
```

FIFO queues provide stronger ordering/deduplication capabilities.

Use FIFO when the business workflow genuinely requires them.

Examples may include:

```text
Sequential state transitions
Ordered commands
Workflows where order matters
```

Do not choose FIFO merely because:

> "FIFO sounds safer."

It introduces different throughput and design considerations.

---

# 9. Standard vs FIFO

| Feature            | Standard                   | FIFO                                    |
| ------------------ | -------------------------- | --------------------------------------- |
| Throughput         | Very high                  | Controlled by FIFO limits/configuration |
| Ordering           | Best effort                | Ordered within message group            |
| Delivery           | At least once              | At least once with FIFO features        |
| Duplicate handling | Application responsibility | Deduplication features available        |
| Message groups     | No                         | Yes                                     |
| Typical use        | General async workloads    | Ordering-sensitive workloads            |

Always verify current AWS service limits for the selected queue configuration.

---

# 10. Queue Architecture

For our platform:

```text
order-service
      |
      v
order-events
      |
      +---- consumer
```

Later:

```text
order-service
      |
      v
SNS/EventBridge
      |
      +----> payment-events SQS
      |
      +----> inventory-events SQS
      |
      +----> notification-events SQS
```

This avoids having multiple services compete for the same event when all need it.

---

# 11. SQS Queue Naming

Use predictable names.

Example:

```text
ecommerce-prod-order-events
ecommerce-prod-payment-events
ecommerce-prod-inventory-events
```

For DLQ:

```text
ecommerce-prod-order-events-dlq
```

Include:

```text
Environment
Domain
Purpose
```

Avoid ambiguous names such as:

```text
queue1
testqueue
orders
```

---

# 12. Environment Separation

Our environments:

```text
dev
qa
staging
prod
```

should have separate queues.

Example:

```text
dev:
ecommerce-dev-order-events

qa:
ecommerce-qa-order-events

staging:
ecommerce-staging-order-events

prod:
ecommerce-prod-order-events
```

Do not allow a DEV application to publish into a production queue.

---

# 13. Terraform Structure

Our infrastructure repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── sqs/
    │       ├── main.tf
    │       ├── variables.tf
    │       └── outputs.tf
    │
    └── environments/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

---

# 14. Terraform SQS Queue

Example:

```hcl
resource "aws_sqs_queue" "order_events" {
  name = "ecommerce-${var.environment}-order-events"

  visibility_timeout_seconds = 60

  message_retention_seconds = 345600

  receive_wait_time_seconds = 20

  tags = {
    Environment = var.environment
    Service     = "order"
  }
}
```

Values are examples.

Production values should be derived from the consumer's processing time and business requirements.

---

# 15. Visibility Timeout

This is one of the most important SQS concepts.

Suppose:

```text
Message
   ↓
Consumer receives it
```

SQS temporarily hides the message from other consumers.

That period is:

```text
Visibility Timeout
```

Example:

```text
60 seconds
```

---

# 16. Why Visibility Timeout Exists

Without visibility timeout:

```text
Consumer A receives message
Consumer B receives same message
```

while Consumer A is still processing it.

With visibility timeout:

```text
SQS
 ↓
Consumer A receives message
 ↓
Message hidden
 ↓
Consumer A processes
 ↓
Delete message
```

---

# 17. Visibility Timeout Is Not Processing Timeout

Suppose:

```text
Visibility timeout = 60 sec
```

but:

```text
Processing time = 120 sec
```

Then the message may become visible again before the first consumer finishes.

This can cause:

```text
Consumer A
     |
     | processing
     |
     +---- 60 sec ----+
                      |
                      v
                 Message visible
                      |
                      v
                 Consumer B
```

Now both consumers may process it.

---

# 18. Visibility Timeout Rule

A practical starting principle:

```text
Visibility Timeout
>
Expected maximum processing time
```

But do not choose an excessively large timeout.

If a consumer crashes:

```text
Message remains invisible
```

until the timeout expires.

Therefore:

```text
Too short
→ duplicate processing

Too long
→ slow retry/recovery
```

---

# 19. Long Polling

SQS supports long polling.

Example:

```hcl
receive_wait_time_seconds = 20
```

Instead of constantly asking:

```text
Any message?
No.

Any message?
No.

Any message?
No.
```

the consumer can wait for messages.

Benefits:

```text
Fewer empty receives
Lower API request volume
Lower unnecessary polling
Better efficiency
```

---

# 20. Short Polling vs Long Polling

### Short polling

```text
Consumer
 ↓
Receive
 ↓
No message
 ↓
Receive
 ↓
No message
```

### Long polling

```text
Consumer
 ↓
Receive
 ↓
Wait
 ↓
Message arrives
 ↓
Return message
```

Our production consumers should normally use long polling.

---

# 21. Message Retention

Example:

```hcl
message_retention_seconds = 345600
```

which is:

```text
4 days
```

This controls how long SQS retains a message if it is not successfully removed.

Retention should be chosen based on:

```text
Maximum outage duration
Consumer recovery time
Operational requirements
Business requirements
Compliance
```

---

# 22. Maximum Message Size

SQS has service-defined message size limits.

Do not use SQS as a large object store.

Bad:

```text
Huge PDF
Large image
Large database dump
```

Better:

```text
S3 object
   |
   v
SQS message containing object reference
```

Example:

```json
{
  "eventType": "InvoiceGenerated",
  "invoiceId": "INV-10001",
  "s3Bucket": "ecommerce-prod-invoices",
  "s3Key": "2026/09/INV-10001.pdf"
}
```

Protect the referenced object with IAM.

---

# 23. Message Envelope

Use a consistent event structure.

Example:

```json
{
  "eventId": "evt-12345",
  "eventType": "OrderCreated",
  "eventVersion": "1",
  "occurredAt": "2026-09-28T10:30:00Z",
  "source": "order-service",
  "correlationId": "corr-12345",
  "payload": {
    "orderId": "10001",
    "customerId": "customer-001"
  }
}
```

Useful metadata:

```text
eventId
eventType
eventVersion
occurredAt
source
correlationId
payload
```

---

# 24. Event ID

Every event should have a unique identifier.

Example:

```text
evt-12345
```

This is important for idempotency.

If the same message is delivered twice:

```text
evt-12345
```

the consumer can recognize that it has already processed it.

---

# 25. At-Least-Once Delivery

SQS Standard should be designed with:

```text
At-least-once delivery
```

in mind.

That means:

```text
Message
   ↓
Consumer processes
   ↓
Network problem
   ↓
DeleteMessage fails
   ↓
Message becomes visible again
```

The consumer may receive the message again.

Therefore:

> Consumers must be idempotent.

---

# 26. Producer Architecture

Our order-service:

```text
POST /orders
     |
     v
Validate
     |
     v
Save order to PostgreSQL
     |
     v
Publish event
     |
     v
SQS
```

But this introduces a consistency problem.

---

# 27. Database + SQS Dual Write Problem

Consider:

```text
Save database
     |
     v
Success
     |
     X
SQS publish fails
```

Now:

```text
Database = order exists
SQS = event missing
```

Or:

```text
SQS publish succeeds
     |
     X
Database transaction rolls back
```

Now:

```text
SQS says order exists
Database says it doesn't
```

This is a distributed consistency problem.

Later we will introduce the Outbox Pattern to address this class of problem.

---

# 28. Initial Learning Implementation

For the first implementation:

```text
Order Service
   |
   +--> PostgreSQL
   |
   +--> SQS
```

We explicitly document the dual-write limitation.

This is important because:

> A working demo is not automatically a production-safe event architecture.

---

# 29. IAM for Producer

The order-service should have an IAM role with only the permissions it needs.

For example:

```text
sqs:SendMessage
```

on:

```text
ecommerce-prod-order-events
```

Do not give:

```text
sqs:*
```

to every application.

---

# 30. IAM Architecture

Our existing EKS Pod Identity model:

```text
order-service Pod
       |
       v
ServiceAccount
       |
       v
EKS Pod Identity
       |
       v
IAM Role
       |
       v
SQS
```

This avoids putting AWS access keys inside the Pod.

---

# 31. Producer IAM Policy

Conceptual example:

```hcl
data "aws_iam_policy_document" "order_sqs" {

  statement {
    effect = "Allow"

    actions = [
      "sqs:SendMessage"
    ]

    resources = [
      aws_sqs_queue.order_events.arn
    ]
  }
}
```

This is much safer than:

```text
Resource = "*"
```

---

# 32. Consumer IAM

The consumer needs different permissions.

Typical actions:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Depending on the implementation, additional permissions may be needed.

Consumer:

```text
payment-service
      |
      v
SQS
```

should only access the queue it owns.

---

# 33. Producer and Consumer Roles

Do not automatically share one IAM role.

Better:

```text
order-service
    |
    +--> order-events:SendMessage

payment-service
    |
    +--> payment-events:Receive/Delete
```

This provides better isolation.

---

# 34. KMS Encryption

SQS supports server-side encryption.

Architecture:

```text
Producer
   |
   v
SQS
   |
   v
KMS encryption
```

For regulated/sensitive environments, use an appropriate customer-managed KMS key where required.

The exact KMS key policy must permit the necessary SQS/service principals and application identities as appropriate.

---

# 35. Encryption at Rest

The goal:

```text
SQS message
    ↓
Encrypted at rest
```

Do not assume encryption solves authorization.

We still need:

```text
IAM
KMS key policy
Queue policy
Application identity
```

---

# 36. Queue Policy

Queue policies can restrict who can send/receive.

For example:

```text
Only approved AWS account/service
Only approved SNS topic
Only approved principals
```

Use queue policies when cross-service/account permissions require them.

---

# 37. Cross-Account Messaging

Enterprise architectures may use:

```text
Account A
order-service
     |
     v
Account B
SQS
```

This requires coordinated:

```text
IAM
Queue policy
KMS policy
Network is usually not the primary concern for SQS API access
```

SQS is an AWS regional managed service accessed through AWS APIs.

---

# 38. Sending a Message

AWS CLI example:

```bash
aws sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{
    "eventId": "evt-12345",
    "eventType": "OrderCreated",
    "eventVersion": "1",
    "orderId": "10001"
  }'
```

For production applications, the SDK is normally used rather than shelling out to the CLI.

---

# 39. Receiving a Message

CLI example:

```bash
aws sqs receive-message \
  --queue-url "$QUEUE_URL" \
  --max-number-of-messages 10 \
  --wait-time-seconds 20
```

The consumer receives messages.

---

# 40. Delete After Successful Processing

Important:

Receiving a message does NOT permanently remove it.

The consumer should:

```text
Receive
  ↓
Process
  ↓
Success
  ↓
Delete
```

If processing fails:

```text
Receive
  ↓
Processing fails
  ↓
Do not delete
  ↓
Visibility timeout expires
  ↓
Message becomes available again
```

---

# 41. Consumer Flow

Production-style consumer:

```text
Poll SQS
   |
   v
Receive message
   |
   v
Validate event
   |
   v
Check idempotency
   |
   v
Process business logic
   |
   +---- Failure ----> no delete
   |
   v
Success
   |
   v
Delete message
```

---

# 42. What Happens When Consumer Crashes?

Suppose:

```text
Consumer receives message
      |
      v
Visibility timeout = 60s
      |
      X
Consumer crashes
```

The message becomes visible again after the visibility timeout.

Another consumer can process it.

This is one of the core reliability features of SQS.

---

# 43. Retry Lifecycle

Example:

```text
Attempt 1
   ↓
Failure
   ↓
Visibility timeout
   ↓
Attempt 2
   ↓
Failure
   ↓
Visibility timeout
   ↓
Attempt 3
   ↓
Failure
   ↓
DLQ
```

The exact retry count is configured using redrive policy.

---

# 44. Dead-Letter Queue

DLQ means:

```text
Dead-Letter Queue
```

Example:

```text
Main Queue
    |
    | failures
    v
Retry
    |
    | repeated failures
    v
DLQ
```

The DLQ prevents permanently failing messages from continuously blocking normal processing.

---

# 45. DLQ Architecture

```text
ecommerce-prod-order-events
             |
             |
       repeated failure
             |
             v
ecommerce-prod-order-events-dlq
```

The DLQ is operationally important.

It should be monitored.

---

# 46. Redrive Policy

Conceptually:

```hcl
redrive_policy = jsonencode({
  deadLetterTargetArn = aws_sqs_queue.order_events_dlq.arn
  maxReceiveCount     = 5
})
```

This means after the configured number of receives, the message is moved to the DLQ.

The exact count should be based on:

```text
Processing time
Transient failure duration
Expected retry behavior
Business requirements
```

---

# 47. DLQ Retention

A DLQ should usually retain failed messages long enough for investigation and replay according to operational requirements.

Example:

```text
Main queue:
4 days

DLQ:
14 days
```

These are example values.

Choose them intentionally.

---

# 48. DLQ Alert

Create an alert when:

```text
ApproximateNumberOfMessagesVisible > 0
```

or when it exceeds a defined threshold.

Example:

```text
WARNING:
Order event DLQ contains messages
```

A DLQ message should not be ignored.

---

# 49. DLQ Investigation

When a message reaches DLQ:

```text
1. Identify event
2. Identify consumer
3. Identify failure reason
4. Check application logs
5. Check dependency failures
6. Determine whether message is safe to replay
```

Do not blindly redrive everything.

---

# 50. Poison Message

A poison message is a message that repeatedly causes processing failure.

Example:

```json
{
  "eventType": "OrderCreated",
  "orderId": null
}
```

Consumer:

```text
Validation fails
 ↓
Retry
 ↓
Validation fails
 ↓
Retry
```

Eventually:

```text
DLQ
```

---

# 51. Why DLQ Matters

Without DLQ:

```text
Bad message
   ↓
Retry
   ↓
Retry
   ↓
Retry
   ↓
Retry forever
```

With DLQ:

```text
Bad message
   ↓
Retry
   ↓
Retry
   ↓
DLQ
   ↓
Normal queue continues
```

---

# 52. Idempotency

Suppose:

```text
OrderCreated
eventId = evt-123
```

Consumer processes it.

Then:

```text
DeleteMessage
```

fails.

SQS redelivers:

```text
evt-123
```

Without idempotency:

```text
Payment
↓
charged twice
```

This is unacceptable.

---

# 53. Idempotency Store

A consumer can maintain an idempotency record.

Example:

```text
processed_events
----------------
event_id
processed_at
consumer
status
```

Before processing:

```text
SELECT event_id
FROM processed_events
WHERE event_id = 'evt-123';
```

If already processed:

```text
Skip business operation
Delete message
```

---

# 54. Idempotency Must Be Atomic

A dangerous implementation:

```text
Check event
   ↓
Not found
   ↓
Process payment
   ↓
Save event
```

Two consumers could race:

```text
Consumer A checks
Consumer B checks
Both see "not found"
Both process
```

For critical operations, use a design that atomically establishes idempotency before allowing the business effect, or otherwise coordinates the effect with the idempotency record.

---

# 55. Payment Example

Never design:

```text
SQS duplicate
   ↓
Charge credit card again
```

Instead:

```text
eventId
   ↓
Idempotency check
   ↓
Payment operation
   ↓
Record result
```

The payment provider itself should also support appropriate idempotency where available.

---

# 56. Message Ordering

Standard SQS does not provide strict ordering guarantees.

Suppose:

```text
OrderCreated
OrderCancelled
```

The consumer might observe them in an unexpected order.

If ordering is required, options include:

```text
FIFO
MessageGroupId
Application-level sequencing
```

Choose based on business requirements.

---

# 57. FIFO Message Groups

Suppose:

```text
MessageGroupId = order-10001
```

Then messages for:

```text
order-10001
```

can be ordered within that message group.

Meanwhile:

```text
order-10002
```

can be processed independently.

This allows parallelism across groups while preserving ordering within a group.

---

# 58. Standard Queue + Idempotency

For many workloads:

```text
Standard SQS
+
Idempotent consumer
+
No strict ordering requirement
```

is simpler and highly scalable.

Do not use FIFO when ordering is not actually needed.

---

# 59. Consumer Deployment in EKS

Our consumer is a Kubernetes workload.

Example:

```text
payment-service
   |
   v
Deployment
   |
   v
Pods
   |
   v
SQS
```

The Pods continuously poll the queue.

---

# 60. Consumer Deployment

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: ecommerce-prod

spec:
  replicas: 2

  selector:
    matchLabels:
      app: payment-service

  template:
    metadata:
      labels:
        app: payment-service

    spec:
      serviceAccountName: payment-service

      containers:
        - name: payment-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-service:8f31a2c
```

The ServiceAccount is associated with the required AWS permissions using EKS Pod Identity.

---

# 61. No AWS Keys in the Pod

Do not put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside Kubernetes Secrets for normal EKS workload access.

Use:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity
 ↓
IAM role
 ↓
SQS
```

---

# 62. Consumer IAM

Conceptually:

```hcl
actions = [
  "sqs:ReceiveMessage",
  "sqs:DeleteMessage",
  "sqs:GetQueueAttributes"
]
```

Resource:

```text
ecommerce-prod-payment-events
```

Avoid:

```text
sqs:*
Resource = "*"
```

unless a very specific platform-level role genuinely requires it.

---

# 63. KEDA

KEDA can scale Kubernetes workloads based on external event sources such as SQS.

Architecture:

```text
SQS backlog
     |
     v
KEDA
     |
     v
Deployment replicas
     |
     v
Consumer Pods
```

This is useful because CPU is not necessarily the best signal for queue consumers.

---

# 64. Why CPU-Based Scaling Is Insufficient

Suppose:

```text
Queue:
100,000 messages
```

but:

```text
Consumer CPU:
30%
```

CPU may look healthy.

However:

```text
Queue backlog:
100,000
```

means consumers are not keeping up.

KEDA can scale based on queue depth.

---

# 65. KEDA Scaling Model

Conceptually:

```text
Queue depth increases
       ↓
KEDA observes SQS
       ↓
Desired replicas increase
       ↓
Kubernetes schedules Pods
       ↓
Karpenter provisions nodes if necessary
       ↓
Consumers process messages
       ↓
Queue depth decreases
       ↓
Replicas scale down
```

This connects:

```text
SQS
+
KEDA
+
HPA-like behavior
+
Karpenter
```

---

# 66. KEDA ScaledObject

A conceptual configuration:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: payment-service
  namespace: ecommerce-prod

spec:
  scaleTargetRef:
    name: payment-service

  minReplicaCount: 2
  maxReplicaCount: 20

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123456789012/ecommerce-prod-payment-events
        queueLength: "50"
        awsRegion: ap-south-1
```

Exact KEDA metadata/authentication configuration depends on the KEDA version and AWS authentication method.

Prefer workload identity rather than static AWS credentials.

---

# 67. Queue Length vs Replica Count

Suppose:

```text
Queue = 10,000 messages
```

and:

```text
queueLength = 50
```

Conceptually KEDA can determine that more consumers are required.

The exact scaling formula depends on KEDA's scaler behavior and configuration.

Do not interpret:

```text
queueLength = 50
```

as:

> exactly 50 Pods.

It is a scaling target signal.

---

# 68. KEDA + Karpenter

Full architecture:

```text
SQS
 ↓
Queue backlog increases
 ↓
KEDA
 ↓
More consumer Pods
 ↓
Pods Pending
 ↓
Karpenter
 ↓
New EC2 nodes
 ↓
Scheduler
 ↓
Consumer Pods
 ↓
SQS backlog decreases
```

This is one of the most useful event-driven scaling patterns in our platform.

---

# 69. Visibility Timeout + KEDA

Suppose:

```text
Processing time = 2 minutes
```

but:

```text
Visibility timeout = 30 seconds
```

Messages may reappear while processing.

This can produce:

```text
Duplicate processing
```

Therefore:

```text
Visibility timeout
+
Consumer processing time
+
Retry behavior
```

must be designed together.

---

# 70. Long-Running Messages

If processing can exceed the initial visibility timeout, the consumer can extend the visibility timeout while processing where appropriate.

Conceptually:

```text
Receive
  ↓
Process
  ↓
Extend visibility
  ↓
Continue
  ↓
Success
  ↓
Delete
```

This should be used carefully.

A permanently stuck consumer should not hide a message forever.

---

# 71. Consumer Graceful Shutdown

When Kubernetes terminates a consumer:

```text
SIGTERM
   ↓
Stop receiving new work
   ↓
Finish current message where possible
   ↓
Delete only after successful processing
   ↓
Exit
```

Configure:

```yaml
terminationGracePeriodSeconds: 60
```

as a starting example.

The correct value depends on maximum processing time.

---

# 72. Consumer Deployment Strategy

For consumers, rolling updates should avoid excessive disruption.

Example:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

But the actual strategy should consider:

```text
Message processing
Visibility timeout
Graceful shutdown
Queue depth
Consumer capacity
```

---

# 73. Consumer Health

A consumer needs health checks.

But avoid making:

```text
SQS unavailable
```

automatically kill every consumer Pod unless that is actually the desired behavior.

The application should distinguish:

```text
Process alive
```

from:

```text
Can currently consume messages
```

---

# 74. SQS Monitoring

Important CloudWatch metrics include concepts such as:

```text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
ApproximateAgeOfOldestMessage
NumberOfMessagesReceived
NumberOfMessagesDeleted
NumberOfMessagesSent
```

The exact available metrics should be verified for the queue configuration.

---

# 75. Queue Depth

Most important operational signal:

```text
Visible Messages
```

Example:

```text
10
100
1,000
10,000
100,000
```

A continuously increasing queue usually means:

```text
Incoming rate > Consumer processing rate
```

---

# 76. Age of Oldest Message

This can be more meaningful than queue depth.

Example:

```text
Queue:
100 messages

Oldest:
2 hours old
```

The queue is not healthy merely because there are only 100 messages.

The age indicates customer/business impact.

---

# 77. Queue Throughput

Suppose:

```text
Producer:
1000 messages/sec

Consumers:
800 messages/sec
```

Backlog grows:

```text
200 messages/sec
```

Eventually:

```text
Queue depth:
1000
5000
10000
50000
```

Scaling consumers is required.

---

# 78. Consumer Processing Rate

Measure:

```text
messages processed/sec
processing latency
success rate
failure rate
```

Then compare:

```text
Producer rate
vs
Consumer rate
```

This tells us whether the system is keeping up.

---

# 79. SQS Backlog Alert

Conceptually:

```text
Queue depth > threshold
for 10 minutes
```

or:

```text
Oldest message age > threshold
```

The threshold should come from:

```text
Business SLA
Processing time
Queue capacity
```

not an arbitrary number.

---

# 80. Example Alert

```yaml
- alert: OrderQueueBacklog
  expr: aws_sqs_messages_visible > 10000
  for: 10m

  labels:
    severity: warning

  annotations:
    summary: "Order queue backlog is increasing"
    runbook: "https://runbooks.example.com/sqs/order-queue"
```

The exact metric integration depends on whether AWS metrics are collected through CloudWatch exporter, Amazon Managed Service for Prometheus integration, Grafana Cloud, or another monitoring architecture.

---

# 81. Queue Consumer Incident

Symptoms:

```text
Order processing delayed
```

Start with:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=payment-service
```

Then:

```bash
kubectl get hpa -n ecommerce-prod
```

If using KEDA:

```bash
kubectl get scaledobject \
  -n ecommerce-prod
```

---

# 82. Check Queue

Inspect:

```text
Visible messages
Not-visible messages
Oldest message age
DLQ depth
```

Determine:

```text
Producer rate
Consumer rate
```

---

# 83. Check Consumer Logs

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/payment-service \
  --tail=200
```

Look for:

```text
SQS timeout
AccessDenied
Message processing failure
Database failure
External API failure
Serialization failure
```

---

# 84. AWS AccessDenied

If the consumer cannot read SQS:

```text
AccessDenied
```

Check:

```text
Pod Identity
ServiceAccount
IAM role
Trust relationship
IAM policy
Queue ARN
Region
KMS permissions
```

Do not immediately grant:

```text
AdministratorAccess
```

---

# 85. KMS Access Failure

If SQS encryption uses a customer-managed KMS key, permissions may involve:

```text
IAM policy
+
KMS key policy
```

An application may have SQS permissions but still encounter an authorization problem involving encryption configuration.

Investigate both.

---

# 86. Region Configuration

Check:

```text
SQS region
Application AWS region
Queue URL
IAM policy resource ARN
```

A common configuration mistake is:

```text
Application:
ap-south-1

Queue:
us-east-1
```

when the code assumes a different regional resource.

---

# 87. Message Processing Failure

Suppose:

```text
Payment service
    ↓
Database unavailable
    ↓
Message processing fails
```

Do not delete the message.

Allow retry.

If the failure persists:

```text
DLQ
```

Once the database recovers:

```text
Redrive/replay
```

can be performed according to the approved recovery procedure.

---

# 88. DLQ Replay

Before replay:

```text
1. Identify root cause
2. Fix consumer
3. Verify dependency
4. Test with one message
5. Replay controlled batch
6. Monitor
```

Do not immediately replay:

```text
100,000 messages
```

into a consumer that was just recovered.

---

# 89. Replay Storm

Example:

```text
DLQ:
100,000 messages
```

Replay all:

```text
100,000
    ↓
SQS
    ↓
Consumer
    ↓
Database
```

This can overload the downstream system.

Use:

```text
Controlled replay
Rate limiting
Consumer scaling
Database capacity monitoring
```

---

# 90. Message Poison Handling

If a message repeatedly fails because of malformed data:

```text
eventId = evt-123
```

do not repeatedly replay it.

Investigate:

```text
Schema
Producer version
Consumer compatibility
Data corruption
```

Then decide whether to:

```text
Fix data
Discard
Transform
Replay
```

according to business policy.

---

# 91. Schema Evolution

Suppose producer sends:

```json
{
  "eventVersion": "2",
  "orderId": "10001",
  "customerId": "customer-001",
  "currency": "INR"
}
```

Old consumer only understands:

```text
eventVersion = 1
```

This can break processing.

Therefore events need:

```text
Backward compatibility
Versioning
Schema governance
```

We should avoid breaking existing consumers during rolling deployments.

---

# 92. Backward-Compatible Event Changes

Safe example:

```text
Add optional field
```

Example:

```json
{
  "orderId": "10001",
  "currency": "INR"
}
```

Old consumer can ignore:

```text
currency
```

Risky change:

```text
Rename:
orderId
```

to:

```text
id
```

without compatibility.

---

# 93. Consumer Versioning

During deployment:

```text
Old consumer
      +
New consumer
```

may process messages simultaneously.

Therefore event contracts should support both versions during the rollout.

This is another reason:

```text
eventVersion
```

is useful.

---

# 94. Message Correlation

Use:

```text
correlationId
```

across services.

Example:

```text
HTTP request
correlationId = corr-123

order-service
      ↓
SQS
      ↓
payment-service
      ↓
notification-service
```

The same correlation ID can appear in:

```text
Logs
Metrics
Traces
Messages
```

This becomes extremely useful in our later observability chapters.

---

# 95. Trace Context

For distributed tracing, event messages can carry trace context.

Conceptually:

```text
HTTP
 ↓
order-service
 ↓
SQS
 ↓
payment-service
```

The trace context can connect asynchronous processing.

Our OpenTelemetry chapter will implement this in more detail.

---

# 96. Queue Ownership

Each queue should have a clear owner.

Example:

```text
order-events
Owner: backend-team

payment-events
Owner: payment-team

notification-events
Owner: notification-team
```

Operational ownership should be documented.

---

# 97. Queue Naming and Ownership

Maintain:

```text
Queue
Environment
Producer
Consumer
Owner
DLQ
Retention
Visibility timeout
Encryption
IAM role
SLO
```

Example:

| Field      | Value                             |
| ---------- | --------------------------------- |
| Queue      | ecommerce-prod-payment-events     |
| Producer   | order-service                     |
| Consumer   | payment-service                   |
| Owner      | payment-team                      |
| DLQ        | ecommerce-prod-payment-events-dlq |
| Visibility | 120 sec                           |
| Retention  | 4 days                            |

---

# 98. Production SQS Architecture

Our initial production architecture:

```text
                   order-service
                         |
                         |
                         v
              ecommerce-prod-order-events
                         |
                         v
                  Consumer service
                         |
                 +-------+-------+
                 |               |
               Success          Failure
                 |               |
                 v               v
             Delete           Retry
                                 |
                                 v
                                DLQ
```

---

# 99. Multi-Service Event Architecture

Eventually:

```text
                     order-service
                           |
                           v
                    Event Publisher
                           |
                           v
                     SNS/EventBridge
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
          SQS-Pay       SQS-Inv       SQS-Notify
             |             |             |
             v             v             v
       payment-service inventory-service notification-service
```

This will be implemented in:

```text
114-SNS-Event-Architecture.md
```

after we complete the SQS implementation and retry/idempotency chapters.

---

# 100. Production Checklist

## Queue

```text
[ ] Correct queue type
[ ] Naming convention
[ ] Environment isolation
[ ] Retention configured
[ ] Visibility timeout configured
[ ] Long polling enabled
[ ] Encryption enabled
[ ] DLQ configured
```

## IAM

```text
[ ] Producer role
[ ] Consumer role
[ ] Least privilege
[ ] EKS Pod Identity
[ ] No AWS keys in Pods
[ ] KMS permissions reviewed
```

## Consumer

```text
[ ] Long polling
[ ] Idempotency
[ ] Delete after success
[ ] Retry on failure
[ ] Graceful shutdown
[ ] Visibility timeout compatible with processing
[ ] Health checks
```

## Kubernetes

```text
[ ] Deployment
[ ] Resource requests
[ ] Resource limits
[ ] HPA/KEDA
[ ] Karpenter capacity
[ ] Pod disruption strategy
```

## Observability

```text
[ ] Queue depth
[ ] Oldest message age
[ ] Message receive rate
[ ] Message delete rate
[ ] Processing latency
[ ] Consumer errors
[ ] DLQ depth
[ ] Alerts
```

---

# 101. End-to-End Order Example

Customer:

```text
POST /orders
```

Order service:

```text
Validate request
     ↓
Create database record
     ↓
Publish OrderCreated
     ↓
SQS
```

Payment service:

```text
Receive OrderCreated
     ↓
Idempotency check
     ↓
Process payment
     ↓
Delete message
```

Inventory service:

```text
Receive event
     ↓
Reserve inventory
     ↓
Delete message
```

Notification service:

```text
Receive event
     ↓
Send notification
     ↓
Delete message
```

This decouples the services.

---

# 102. Final Mental Model

Remember the SQS processing contract:

```text
SEND
 ↓
QUEUE
 ↓
RECEIVE
 ↓
PROCESS
 ↓
SUCCESS?
 ├── YES → DELETE
 └── NO  → RETRY
              ↓
             DLQ
```

And the scaling model:

```text
Queue backlog
     ↓
KEDA
     ↓
Consumer Pods
     ↓
Karpenter
     ↓
EC2 capacity
     ↓
More consumers
     ↓
Queue drains
```

The production reliability model is:

```text
SQS
+
Visibility Timeout
+
Long Polling
+
DLQ
+
Idempotency
+
KEDA
+
IAM
+
Monitoring
=
Reliable Async Processing
```

---

# 103. Next Chapter

**113-SQS-DLQ-Retry-Idempotency.md**

The next chapter will go deeper into the hardest SQS production problems:

```text
Message
   ↓
Consumer
   ↓
Processing failure
   ↓
Retry
   ↓
Duplicate delivery
   ↓
Idempotency
   ↓
DLQ
   ↓
Root cause investigation
   ↓
Controlled replay
```

We will implement:

* retry strategy
* visibility timeout tuning
* exponential backoff
* DLQ
* poison messages
* duplicate messages
* idempotency store
* exactly-once misconceptions
* transactional processing
* failure scenarios
* DLQ replay
* consumer crash recovery
* message ordering
* schema evolution
* production incident simulation

```
```
