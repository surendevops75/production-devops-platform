# 113-SQS-DLQ-Retry-Idempotency.md

````markdown
# 113 — SQS DLQ, Retry & Idempotency

## 1. Purpose

The previous chapter implemented the basic SQS production architecture.

This chapter focuses on what happens when message processing fails.

The core problem is:

```text
SQS
 ↓
Consumer
 ↓
Processing
 ↓
FAILURE
````

A production system must answer:

* How many times should we retry?
* When should a message go to DLQ?
* What happens if the same message is delivered twice?
* How do we prevent duplicate business operations?
* How do we handle poison messages?
* How do we safely replay DLQ messages?
* How do we avoid retry storms?
* How do we tune visibility timeout?
* What happens when the consumer crashes?
* What happens when the database is unavailable?

The target architecture is:

```text
                    +----------------+
                    |    Producer    |
                    +-------+--------+
                            |
                            v
                    +---------------+
                    |   SQS Queue   |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    |   Consumer    |
                    +-------+-------+
                            |
                   +--------+--------+
                   |                 |
                SUCCESS           FAILURE
                   |                 |
                   v                 v
                DELETE            RETRY
                                     |
                                     v
                              maxReceiveCount
                                     |
                                     v
                                   DLQ
```

---

# 2. The Most Important SQS Rule

Never design a consumer assuming:

```text
Message is delivered exactly once.
```

Instead design for:

```text
At-least-once delivery
```

That means:

```text
Message A
   ↓
Consumer receives A
   ↓
Consumer processes A
   ↓
DeleteMessage fails
   ↓
A becomes visible again
   ↓
Consumer receives A again
```

Therefore:

```text
Duplicate delivery
      +
Idempotent processing
```

must be part of the application design.

---

# 3. What Does "Exactly Once" Mean?

There are two different concepts.

### Delivery exactly once

This means the messaging system guarantees that a message is delivered only once.

Do not assume this for a general SQS Standard queue design.

### Business effect exactly once

This means:

```text
Duplicate messages
       ↓
Duplicate processing attempts
       ↓
Only one business effect
```

For example:

```text
Message delivered twice
       ↓
Payment operation executed once
```

This is achieved through application design.

---

# 4. Example: Duplicate Payment

Suppose:

```text
eventId = evt-1001
orderId = ORD-5001
amount = 5000
```

Payment service receives it.

```text
Consumer
   ↓
Charge ₹5000
   ↓
Success
```

But before deleting the SQS message:

```text
Consumer
   X
Network failure
```

The message becomes visible again.

Second attempt:

```text
Consumer
   ↓
Charge ₹5000
```

Without idempotency:

```text
₹5000 charged
+
₹5000 charged
=
₹10000 charged
```

This is a business-critical failure.

---

# 5. Idempotency

An operation is idempotent when repeating the same operation does not create an additional business effect.

For example:

```text
Process event evt-1001
```

first time:

```text
Payment created
```

second time:

```text
Existing result returned
```

rather than creating another payment.

---

# 6. Idempotency Key

The event should contain a stable identifier.

Example:

```json
{
  "eventId": "evt-1001",
  "eventType": "OrderCreated",
  "eventVersion": "1",
  "orderId": "ORD-5001"
}
```

The consumer uses:

```text
eventId
```

as the idempotency key.

Do not generate a new random ID each time the message is processed.

Bad:

```text
Receive message
 ↓
generate UUID
 ↓
process
```

because every retry gets a different key.

---

# 7. Idempotency Store

A consumer can maintain a table such as:

```text
processed_events
```

Example:

| Column       | Purpose                   |
| ------------ | ------------------------- |
| event_id     | Unique event identifier   |
| consumer     | Consumer name             |
| status       | PROCESSING / COMPLETED    |
| result       | Business result/reference |
| created_at   | First processing time     |
| completed_at | Completion time           |

Example:

```text
event_id      consumer          status
------------------------------------------------
evt-1001      payment-service   COMPLETED
evt-1002      payment-service   COMPLETED
evt-1003      payment-service   PROCESSING
```

---

# 8. Database Constraint

The strongest protection is usually a unique constraint.

Example PostgreSQL:

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(255) NOT NULL,
    consumer VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL,
    result_reference VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,

    PRIMARY KEY (event_id, consumer)
);
```

Why include consumer?

Because:

```text
evt-1001
```

may legitimately be processed by:

```text
payment-service
```

and:

```text
notification-service
```

as independent consumers.

---

# 9. Idempotency Scope

The key might be:

```text
eventId
```

or:

```text
consumer + eventId
```

depending on the architecture.

Example:

```text
payment-service + evt-1001
```

and:

```text
inventory-service + evt-1001
```

are different processing operations.

---

# 10. Incorrect Idempotency Flow

A common mistake is:

```text
Check event
     ↓
Not found
     ↓
Perform business operation
     ↓
Insert idempotency record
```

Two consumers could race:

```text
Consumer A
   ↓
Check → not found

Consumer B
   ↓
Check → not found

A → process
B → process
```

Now the business operation may happen twice.

---

# 11. Better Approach

Use a database uniqueness constraint.

Conceptually:

```text
BEGIN
   |
   v
Insert event_id
   |
   +---- duplicate key ---> Already processing/processed
   |
   v
Business operation
   |
   v
Mark COMPLETED
   |
   v
COMMIT
```

The exact implementation depends on whether the business operation occurs in the same database transaction.

---

# 12. Transactional Idempotency

Suppose payment state and idempotency record live in the same PostgreSQL database.

We can use:

```text
BEGIN TRANSACTION

Insert processed_event

Create/update payment

Mark event COMPLETED

COMMIT
```

If the transaction fails:

```text
ROLLBACK
```

Both changes roll back.

The SQS message remains available for retry.

This provides a strong consistency boundary.

---

# 13. Important Limitation

You cannot automatically make an external API call transactional with PostgreSQL.

Example:

```text
BEGIN DB transaction
   ↓
Insert idempotency record
   ↓
Call external payment provider
   ↓
Payment succeeds
   ↓
DB transaction fails
```

Now:

```text
Payment provider:
SUCCESS

Database:
ROLLBACK
```

The external operation cannot simply be rolled back by PostgreSQL.

This is a distributed systems problem.

---

# 14. External Payment Provider

For payment processing, use provider-supported idempotency if available.

Example:

```text
provider request
Idempotency-Key: payment-ORD-5001
```

Then:

```text
Attempt 1
   ↓
Provider processes payment

Attempt 2
   ↓
Same idempotency key
   ↓
Provider returns existing result
```

This is much safer than creating a new payment request for every retry.

---

# 15. Retry Categories

Not every failure should be retried.

Classify failures.

### Transient

Examples:

```text
Database temporarily unavailable
Network timeout
HTTP 503
Temporary AWS API throttling
```

Usually retryable.

### Permanent

Examples:

```text
Invalid schema
Missing required field
Unsupported event version
Invalid business data
```

Usually not retryable.

---

# 16. Retry Decision

Consumer logic:

```text
Receive
  ↓
Validate
  ↓
Processing
  ↓
Failure
  |
  +---- Transient ----> Retry
  |
  +---- Permanent ----> DLQ
```

Do not waste resources repeatedly retrying a permanently invalid message.

---

# 17. Example

Message:

```json
{
  "eventType": "OrderCreated",
  "orderId": null
}
```

Validation:

```text
orderId required
```

This is a permanent data problem.

Retrying 10 times will not fix it.

Better:

```text
Validation failure
      ↓
DLQ
      ↓
Investigation
```

---

# 18. Transient Failure Example

Suppose:

```text
payment-service
      ↓
PostgreSQL
      X
connection timeout
```

The database may recover in seconds.

Retrying can make sense.

```text
Attempt 1
 ↓
DB timeout
 ↓
Retry

Attempt 2
 ↓
DB success
 ↓
Delete
```

---

# 19. Retry Storm

A retry storm occurs when a dependency fails and all consumers immediately retry.

Example:

```text
10,000 messages
     ↓
Database unavailable
     ↓
10,000 failures
     ↓
Immediate retry
     ↓
10,000 failures
     ↓
Immediate retry
```

This can make the original outage worse.

---

# 20. Exponential Backoff

Instead of:

```text
1 sec
1 sec
1 sec
1 sec
```

use increasing delays:

```text
1 sec
2 sec
4 sec
8 sec
16 sec
```

with a maximum cap.

Example:

```text
delay = min(base * 2^attempt, maxDelay)
```

Add jitter where appropriate.

---

# 21. Jitter

Without jitter:

```text
100 consumers
   ↓
all retry at 10 seconds
```

This creates another spike.

With jitter:

```text
Consumer 1 → 9.2 sec
Consumer 2 → 10.7 sec
Consumer 3 → 11.3 sec
Consumer 4 → 8.9 sec
```

The load is distributed.

---

# 22. Where Should Backoff Happen?

Possible approaches:

```text
Application retry
```

or:

```text
Queue visibility timeout
```

or:

```text
Dedicated retry queue
```

or:

```text
Delayed message architecture
```

SQS itself does not provide arbitrary per-message delayed retry scheduling in the same way as a dedicated delayed-job system.

Choose the retry design based on workload and operational requirements.

---

# 23. Simple SQS Retry Model

A simple architecture:

```text
Main Queue
    |
    v
Consumer
    |
    +---- success ----> Delete
    |
    +---- failure ----> Don't Delete
                         |
                         v
                   Visibility Timeout
                         |
                         v
                    Retry receive
```

This is simple and often sufficient for transient failures.

---

# 24. DLQ Configuration

Terraform:

```hcl
resource "aws_sqs_queue" "order_events_dlq" {
  name = "ecommerce-${var.environment}-order-events-dlq"

  message_retention_seconds = 1209600

  tags = {
    Environment = var.environment
    Service     = "order"
    Type        = "dlq"
  }
}
```

Example retention:

```text
1209600 seconds = 14 days
```

Choose retention according to investigation and recovery requirements.

---

# 25. Main Queue + DLQ

```hcl
resource "aws_sqs_queue" "order_events" {
  name = "ecommerce-${var.environment}-order-events"

  visibility_timeout_seconds = 120

  message_retention_seconds = 345600

  receive_wait_time_seconds = 20

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_events_dlq.arn
    maxReceiveCount     = 5
  })
}
```

The exact visibility timeout and receive count should be based on the consumer behavior.

---

# 26. Visibility Timeout Calculation

Suppose:

```text
Typical processing:
5 seconds

P95:
15 seconds

P99:
30 seconds

Maximum expected:
60 seconds
```

A reasonable initial visibility timeout could be:

```text
120 seconds
```

Then measure actual processing behavior.

Do not choose values solely from generic recommendations.

---

# 27. Visibility Timeout Too Short

Example:

```text
Processing = 90 seconds
Visibility = 30 seconds
```

Timeline:

```text
0 sec
Receive

30 sec
Message visible again

90 sec
Original consumer finishes
```

Potential duplicate:

```text
Consumer A
+
Consumer B
```

---

# 28. Visibility Timeout Too Long

Example:

```text
Processing = 10 seconds
Visibility = 2 hours
```

Consumer crashes:

```text
Message
 ↓
Invisible for 2 hours
```

Recovery is delayed.

Therefore:

```text
Visibility timeout
≈ enough for normal processing
```

with appropriate handling for unusually long processing.

---

# 29. Maximum Processing Time

Measure:

```text
P50
P95
P99
Maximum
```

Example:

```text
P50 = 1.2 sec
P95 = 4.5 sec
P99 = 8 sec
Max = 25 sec
```

This gives a better basis for visibility timeout tuning.

---

# 30. Consumer Timeout

The application should also have timeouts.

For example:

```text
HTTP timeout = 5 sec
DB connection timeout = 3 sec
DB query timeout = 10 sec
```

Avoid:

```text
SQS visibility = 60 sec
DB query = unlimited
```

An operation that can hang forever is dangerous for queue consumers.

---

# 31. Processing Contract

A good consumer should behave like:

```text
Receive
 ↓
Validate
 ↓
Idempotency
 ↓
Business operation
 ↓
Success
 ↓
Delete
```

Not:

```text
Receive
 ↓
Delete
 ↓
Business operation
```

If you delete first:

```text
Delete succeeds
Business operation fails
```

the message is lost from the queue.

---

# 32. Never Delete Before Processing

Bad:

```java
receiveMessage();

deleteMessage();

processMessage();
```

Good:

```java
receiveMessage();

processMessage();

deleteMessage();
```

with appropriate error handling.

---

# 33. Consumer Pseudocode

```java
while (running) {

    List<Message> messages = receiveMessages();

    for (Message message : messages) {

        try {

            Event event = deserialize(message);

            validate(event);

            if (alreadyProcessed(event.getEventId())) {
                deleteMessage(message);
                continue;
            }

            process(event);

            markProcessed(event.getEventId());

            deleteMessage(message);

        } catch (RetryableException e) {

            log.error("Retryable failure", e);

        } catch (PermanentException e) {

            log.error("Permanent failure", e);

            // Depending on the architecture:
            // allow redrive to DLQ or explicitly route to a failure workflow.

        }
    }
}
```

This is conceptual pseudocode.

Production implementation must carefully handle transaction boundaries.

---

# 34. The Hard Problem

Consider:

```text
process(event)
```

succeeds.

Then:

```text
markProcessed(event)
```

fails.

The message is not deleted.

It will be retried.

If the business operation is not itself protected by idempotency:

```text
Business operation happens twice
```

Therefore idempotency must surround the business effect, not simply be added as an afterthought.

---

# 35. Database Transaction Example

For a database-backed operation:

```text
BEGIN
   |
   v
Insert idempotency record
   |
   v
Business update
   |
   v
Mark completed
   |
   v
COMMIT
```

Then:

```text
SQS Delete
```

after the database transaction succeeds.

---

# 36. If SQS Delete Fails

Suppose:

```text
DB transaction = SUCCESS
SQS Delete = FAILURE
```

The message is redelivered.

Second attempt:

```text
Idempotency record exists
      ↓
Recognize already processed
      ↓
Do not repeat business operation
      ↓
Delete message
```

This is exactly why idempotency is necessary.

---

# 37. Idempotency State Machine

Use:

```text
NEW
 ↓
PROCESSING
 ↓
COMPLETED
```

Potential failure:

```text
PROCESSING
 ↓
Consumer crash
```

Then on retry:

```text
PROCESSING
```

must be interpreted carefully.

Do not automatically assume:

```text
PROCESSING = failed
```

because the previous consumer may still be running.

---

# 38. Stale Processing Records

A processing record can include:

```text
locked_at
lease_until
```

Example:

```text
event_id = evt-1001
status = PROCESSING
lease_until = 10:35
```

If the consumer crashes and the lease expires:

```text
Retry can safely take ownership
```

This is one possible implementation.

The exact design depends on the database and processing model.

---

# 39. Idempotency TTL

Not every event needs to be stored forever.

For some workloads:

```text
idempotency retention = 7 days
```

may be sufficient.

For financial operations:

```text
longer retention
```

may be required.

Retention should match:

```text
Maximum possible duplicate-delivery window
+
Business audit requirements
```

---

# 40. Idempotency Storage Options

Possible stores:

```text
PostgreSQL
DynamoDB
Redis
```

### PostgreSQL

Good when:

```text
Business transaction already uses PostgreSQL
```

### DynamoDB

Useful for:

```text
High-scale idempotency keys
Simple key-value access
```

### Redis

Fast, but be careful about using it as the only durable source for critical business idempotency.

For financial correctness, durable persistence is generally preferred.

---

# 41. DynamoDB Example

Conceptual table:

```text
Table:
payment-idempotency
```

Partition key:

```text
eventId
```

Record:

```json
{
  "eventId": "evt-1001",
  "status": "COMPLETED",
  "paymentId": "PAY-5001",
  "processedAt": "2026-09-28T10:30:00Z"
}
```

Use conditional writes to prevent duplicate processing.

---

# 42. Conditional Write

Conceptually:

```text
Put eventId
ONLY IF
eventId does not already exist
```

First consumer:

```text
SUCCESS
```

Second consumer:

```text
ConditionalCheckFailed
```

Second consumer knows:

```text
Already processed
```

and can safely acknowledge/delete the SQS message.

---

# 43. DLQ Is Not a Garbage Bin

A common production mistake:

```text
DLQ
 ↓
Ignore
```

A DLQ should be treated as:

```text
Operational signal
```

It means:

```text
A message could not be processed normally.
```

---

# 44. DLQ Monitoring

Monitor:

```text
DLQ visible message count
Oldest DLQ message age
Messages moved to DLQ
Replay success rate
Replay failure rate
```

Create alerts.

Example:

```text
DLQ messages > 0
```

may be a warning or critical condition depending on the business process.

---

# 45. DLQ Investigation Workflow

```text
DLQ alert
   ↓
Identify queue
   ↓
Identify message
   ↓
Identify eventId
   ↓
Identify consumer
   ↓
Search logs
   ↓
Search trace
   ↓
Check dependency
   ↓
Identify root cause
```

Then decide:

```text
Fix + replay
```

or:

```text
Discard according to approved policy
```

---

# 46. DLQ Message Inspection

Example:

```bash
aws sqs receive-message \
  --queue-url "$DLQ_URL" \
  --max-number-of-messages 1 \
  --wait-time-seconds 5
```

Be careful:

Receiving a message changes its visibility.

Do not repeatedly inspect production DLQ messages without understanding the operational effect.

---

# 47. Safe DLQ Investigation

Better practice:

```text
1. Receive controlled number
2. Capture message ID/event ID
3. Inspect attributes
4. Record root cause
5. Decide replay/discard
6. Continue according to runbook
```

For highly sensitive production systems, use approved tooling and access controls rather than ad-hoc CLI inspection.

---

# 48. DLQ Replay

Conceptually:

```text
DLQ
 |
 v
Replay process
 |
 +----> validation
 |
 +----> rate limit
 |
 v
Main queue
```

Do not blindly move all messages back at once.

---

# 49. Replay Rate

Suppose:

```text
DLQ = 500,000 messages
```

Consumer capacity:

```text
1000 messages/sec
```

Database capacity:

```text
300 messages/sec
```

If replay sends:

```text
5000 messages/sec
```

the database may fail.

Therefore replay should respect the slowest critical dependency.

---

# 50. Replay Workflow

```text
DLQ
 ↓
Select batch
 ↓
Replay 100
 ↓
Monitor
 ↓
Replay 500
 ↓
Monitor
 ↓
Replay 1000
 ↓
Monitor
```

Stop if:

```text
Error rate increases
DB saturation increases
Queue latency increases
Downstream API starts failing
```

---

# 51. Poison Message During Replay

Suppose:

```text
DLQ:
1000 messages

999 valid
1 malformed
```

Replay:

```text
999 succeed
1 fails
```

The failed message may return to the DLQ.

That is acceptable.

Do not repeatedly replay the same poison message.

---

# 52. Redrive Allow Policy

For controlled DLQ workflows, configure queue permissions appropriately so only intended sources can use redrive operations.

The exact AWS configuration should be reviewed against the current SQS redrive/redrive-allow features and account architecture.

---

# 53. Retry Queue Pattern

Sometimes one DLQ is not enough.

Architecture:

```text
Main Queue
    |
    v
Consumer
    |
 failure
    v
Retry Queue
    |
    v
Consumer
    |
 failure
    v
DLQ
```

This can be useful when explicit retry delays are required.

---

# 54. Multiple Retry Tiers

Example:

```text
Main Queue
    |
    v
Retry-1
    |
    v
Retry-5m
    |
    v
Retry-30m
    |
    v
Retry-2h
    |
    v
DLQ
```

This provides more controlled retry scheduling.

But it also increases architectural complexity.

Use it when the business workload justifies it.

---

# 55. Retry Design Decision

Use simple visibility-timeout retry when:

```text
Transient failures are short
Processing is simple
No long retry schedule is required
```

Use dedicated retry workflows when:

```text
Failures can last minutes/hours
Retry delay must be explicit
Backoff policy is business-critical
```

---

# 56. Backoff Example

Suppose:

```text
Attempt 1 → immediate
Attempt 2 → 10 sec
Attempt 3 → 30 sec
Attempt 4 → 2 min
Attempt 5 → 10 min
```

After final failure:

```text
DLQ
```

The exact numbers should come from operational testing.

---

# 57. Circuit Breaker

Suppose:

```text
Payment provider
      ↓
DOWN
```

Without protection:

```text
100 consumers
   ↓
100 requests
   ↓
fail
   ↓
retry
   ↓
100 requests
```

Use a circuit breaker:

```text
Healthy
  ↓
Failures
  ↓
Open
  ↓
Stop downstream calls temporarily
  ↓
Half-open
  ↓
Test
  ↓
Recover
```

This reduces pressure on failing dependencies.

---

# 58. Queue Consumer + Circuit Breaker

Architecture:

```text
SQS
 ↓
Consumer
 ↓
Circuit Breaker
 ↓
Payment API
```

If payment API is unavailable:

```text
Circuit opens
 ↓
Consumer avoids unnecessary calls
 ↓
Messages remain retryable
```

This must be coordinated carefully with visibility timeout and consumer concurrency.

---

# 59. Concurrency Control

More consumers are not always better.

Suppose:

```text
Database capacity = 100 concurrent operations
```

and:

```text
Consumers = 100
```

Each creates:

```text
5 DB connections
```

Potential connection demand:

```text
100 × 5 = 500
```

This can overload PostgreSQL.

Therefore:

```text
SQS scaling
```

must consider:

```text
Database capacity
External API capacity
Redis capacity
CPU
Memory
Network
```

---

# 60. KEDA Maximum Replicas

Example:

```yaml
minReplicaCount: 2
maxReplicaCount: 20
```

Do not blindly set:

```text
maxReplicaCount: 1000
```

because queue depth is high.

Set the maximum based on downstream capacity.

---

# 61. Backpressure

Backpressure means:

```text
Producer/consumer system
```

does not overwhelm downstream dependencies.

Example:

```text
SQS
 ↓
KEDA
 ↓
Consumer
 ↓
PostgreSQL
```

If PostgreSQL can support:

```text
50 concurrent operations
```

we should not scale consumers to create:

```text
1000 concurrent DB operations
```

just because the queue contains millions of messages.

---

# 62. Queue Backlog Does Not Automatically Mean "Scale Forever"

A large queue can mean:

```text
Producer spike
```

or:

```text
Consumer failure
```

or:

```text
Database failure
```

or:

```text
Malformed messages
```

or:

```text
Downstream dependency outage
```

First determine the cause.

---

# 63. Incident Example

Situation:

```text
09:00
Normal traffic

09:05
Payment provider starts returning 503

09:06
Payment consumer errors increase

09:07
SQS backlog increases

09:08
KEDA increases replicas

09:09
More Pods call payment provider

09:10
Payment provider remains down

09:11
Retry pressure increases
```

Blind scaling makes the incident worse.

---

# 64. Correct Response

```text
Payment provider failure
        ↓
Circuit breaker
        ↓
Limit downstream requests
        ↓
Allow controlled queue accumulation
        ↓
Monitor oldest message age
        ↓
Recover dependency
        ↓
Scale consumers gradually
        ↓
Drain queue
```

This is an important production lesson:

> Scaling is not always the correct response to backlog.

---

# 65. SQS Failure Scenario Matrix

| Failure                    | Expected behavior                     |
| -------------------------- | ------------------------------------- |
| Consumer crash             | Message becomes visible again         |
| Temporary DB failure       | Retry                                 |
| Permanent validation error | DLQ                                   |
| SQS delete failure         | Duplicate possible                    |
| Duplicate message          | Idempotency prevents duplicate effect |
| Consumer deployment        | Graceful shutdown                     |
| Downstream outage          | Backlog + controlled retry            |
| Poison message             | DLQ                                   |
| DLQ replay                 | Controlled batch                      |
| KEDA failure               | Fixed/minimum consumers remain        |
| Karpenter failure          | Pods may remain Pending               |

---

# 66. Consumer Crash Test

In DEV:

```bash
kubectl delete pod \
  -n ecommerce-dev \
  -l app=payment-service
```

Observe:

```text
Pod terminated
   ↓
Deployment creates replacement
   ↓
SQS message becomes visible after timeout
   ↓
New consumer processes message
```

Measure:

```text
Recovery time
```

---

# 67. Database Failure Test

In a controlled environment:

```text
Consumer
   ↓
PostgreSQL unavailable
```

Expected:

```text
Processing failure
 ↓
Message not deleted
 ↓
Retry
 ↓
Eventually DLQ if failure persists
```

After restoring PostgreSQL:

```text
Messages process successfully
```

---

# 68. Duplicate Message Test

Create a test event:

```json
{
  "eventId": "duplicate-test-001",
  "eventType": "OrderCreated",
  "orderId": "TEST-001"
}
```

Send it twice:

```text
Message 1
Message 2
```

Expected:

```text
First:
Business operation executed

Second:
Idempotency detects duplicate
Business operation NOT repeated
```

---

# 69. Poison Message Test

Send:

```json
{
  "eventId": "poison-test-001",
  "eventType": "OrderCreated",
  "orderId": null
}
```

Expected:

```text
Validation failure
 ↓
Retry policy
 ↓
DLQ
```

Verify:

```text
DLQ count increases
```

---

# 70. DLQ Recovery Test

After fixing the consumer:

```text
DLQ
 ↓
Replay one message
 ↓
Consumer
 ↓
Success
```

Then verify:

```text
DLQ decreases
Main queue processes message
Business state correct
```

---

# 71. Production Incident Runbook

## Alert

```text
SQS DLQ > 0
```

## Step 1

Identify queue:

```bash
aws sqs list-queues
```

## Step 2

Check consumer:

```bash
kubectl get pods -n ecommerce-prod
```

## Step 3

Check logs:

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/payment-service \
  --tail=300
```

## Step 4

Check dependency:

```text
RDS
Redis
External API
Secrets
Network
```

## Step 5

Identify failure type:

```text
Transient
Permanent
Unknown
```

## Step 6

Fix root cause.

## Step 7

Test one message.

## Step 8

Replay controlled batches.

---

# 72. Production Decision Tree

```text
SQS backlog increasing
        |
        v
Are consumers healthy?
   |
   +---- NO ----> Fix consumer
   |
   +---- YES
        |
        v
Is processing latency increasing?
   |
   +---- YES ---> Check dependencies
   |
   +---- NO
        |
        v
Is producer rate higher than consumer capacity?
   |
   +---- YES ---> Scale within safe limits
   |
   +---- NO
        |
        v
Check KEDA/Karpenter
```

---

# 73. DLQ Decision Tree

```text
DLQ message
    |
    v
Why failed?
    |
    +---- Invalid data
    |       |
    |       v
    |     Fix/transform/discard
    |
    +---- Consumer bug
    |       |
    |       v
    |     Fix deployment
    |
    +---- Dependency outage
            |
            v
          Restore dependency
            |
            v
          Controlled replay
```

---

# 74. Logging Requirements

Every consumer processing attempt should log:

```text
eventId
eventType
consumer
orderId
correlationId
attempt
processing duration
result
error category
```

Example:

```text
eventId=evt-1001
consumer=payment-service
orderId=ORD-5001
attempt=2
result=SUCCESS
duration_ms=420
```

Do not log:

```text
card number
password
AWS credentials
database password
secret values
```

---

# 75. Metrics

Application metrics should include:

```text
sqs_messages_processed_total
sqs_messages_failed_total
sqs_messages_duplicate_total
sqs_processing_duration_seconds
sqs_processing_errors_total
sqs_dlq_events_total
```

The exact metric names are implementation choices.

Use low-cardinality labels.

Avoid:

```text
eventId
orderId
customerId
```

as Prometheus labels because this can create enormous cardinality.

---

# 76. Tracing

For a request:

```text
POST /orders
```

trace:

```text
HTTP
 ↓
order-service
 ↓
SQS publish
 ↓
payment-service
 ↓
database
```

should ideally allow correlation across asynchronous boundaries.

This will be implemented later in:

```text
123-Distributed-Tracing.md
124-OpenTelemetry-Production-Implementation.md
```

---

# 77. Alert Categories

### Warning

```text
Queue depth increasing
```

### Critical

```text
Oldest message age exceeds business SLO
```

### Critical

```text
DLQ contains messages
```

### Warning

```text
Consumer processing error rate increasing
```

### Critical

```text
Consumer replicas = 0 while queue has backlog
```

---

# 78. Security Checklist

```text
[ ] SQS encrypted
[ ] KMS configured where required
[ ] Producer least privilege
[ ] Consumer least privilege
[ ] Pod Identity
[ ] No AWS keys in Pods
[ ] Queue policies reviewed
[ ] Cross-account access restricted
[ ] Sensitive payloads minimized
[ ] Logs do not expose sensitive data
```

---

# 79. Reliability Checklist

```text
[ ] At-least-once delivery assumed
[ ] Idempotency implemented
[ ] Visibility timeout tuned
[ ] Long polling enabled
[ ] Retry strategy defined
[ ] DLQ configured
[ ] DLQ alerts configured
[ ] Poison messages handled
[ ] Controlled replay process
[ ] Graceful consumer shutdown
[ ] Downstream limits considered
```

---

# 80. Scaling Checklist

```text
[ ] KEDA configured where appropriate
[ ] Minimum replicas defined
[ ] Maximum replicas defined
[ ] Downstream capacity considered
[ ] Karpenter capacity available
[ ] Resource requests configured
[ ] Database connection limits considered
[ ] External API rate limits considered
[ ] Queue backlog monitored
[ ] Oldest message age monitored
```

---

# 81. Production Architecture

The complete SQS architecture now looks like:

```text
                       ┌───────────────────┐
                       │   order-service   │
                       └─────────┬─────────┘
                                 │
                                 │ SendMessage
                                 v
                    ┌────────────────────────┐
                    │  SQS Order Events      │
                    │                        │
                    │ Encryption             │
                    │ Long Polling           │
                    │ Retention              │
                    │ Visibility Timeout     │
                    └───────────┬────────────┘
                                │
                                │ Receive
                                v
                    ┌────────────────────────┐
                    │    Consumer Pods       │
                    │                        │
                    │ Idempotency            │
                    │ Business Processing     │
                    │ Graceful Shutdown       │
                    └───────────┬────────────┘
                                │
                     ┌──────────┴──────────┐
                     │                     │
                  SUCCESS               FAILURE
                     │                     │
                     v                     v
                  DELETE                 RETRY
                                           │
                                           v
                                  maxReceiveCount
                                           │
                                           v
                                  ┌────────────────┐
                                  │      DLQ       │
                                  └───────┬────────┘
                                          │
                                          v
                                   Investigation
                                          │
                                          v
                                   Controlled Replay
```

---

# 82. Complete Reliability Model

Our SQS production design is:

```text
                 SQS
                  |
        +---------+---------+
        |                   |
   Visibility            Long Polling
    Timeout
        |
        v
    Consumer
        |
        +-------------------+
        |                   |
   Idempotency           Processing
        |                   |
        |              +----+----+
        |              |         |
        |           Success    Failure
        |              |         |
        |              v         v
        |           Delete     Retry
        |                         |
        |                         v
        |                        DLQ
        |
        v
 Duplicate Protection
```

---

# 83. Key Production Lessons

### Lesson 1

SQS does not remove distributed-systems problems.

It changes how we handle them.

### Lesson 2

Assume duplicate delivery.

### Lesson 3

Idempotency is a business correctness mechanism.

### Lesson 4

A DLQ is an operational recovery mechanism, not a garbage bin.

### Lesson 5

More consumers do not always mean better performance.

### Lesson 6

Queue scaling must consider downstream capacity.

### Lesson 7

Visibility timeout must be based on real processing behavior.

### Lesson 8

Retry storms can make outages worse.

### Lesson 9

DLQ replay must be controlled.

### Lesson 10

Event contracts must be backward compatible.

---

# 84. Final SQS Failure Model

The complete mental model is:

```text
              MESSAGE
                 |
                 v
              RECEIVE
                 |
                 v
              VALIDATE
                 |
          +------+------+
          |             |
       Invalid        Valid
          |             |
          v             v
         DLQ        IDEMPOTENCY
                        |
                +-------+-------+
                |               |
             Duplicate       New Event
                |               |
                v               v
              DELETE         PROCESS
                                |
                         +------+------+
                         |             |
                      Success       Failure
                         |             |
                         v             v
                       DELETE        Retry
                                       |
                                +------+------+
                                |             |
                             Temporary     Permanent
                                |             |
                                v             v
                              Retry          DLQ
```

This is the reliability pattern we will reuse throughout the 21-microservice platform.

---

# 85. Next Chapter

## 114-SNS-Event-Architecture.md

The next chapter introduces the fan-out architecture:

```text
                         order-service
                              |
                              v
                        SNS / Event Bus
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
        payment SQS     inventory SQS    notification SQS
             |                |                |
             v                v                v
       payment-service  inventory-service  notification-service
```

We will implement:

* SNS topics
* topic subscriptions
* SNS → SQS
* fan-out architecture
* queue policies
* IAM
* KMS
* event envelopes
* message filtering
* subscription filters
* retries
* DLQs
* cross-account event delivery
* event ownership
* ordering considerations
* SQS vs SNS
* SNS vs EventBridge
* production Terraform
* Kubernetes consumers
* observability
* failure scenarios
* complete `order-service → SNS → multiple SQS → consumers` implementation

```
```
