# 25-Kafka.md

# Apache Kafka — Production Event Streaming Architecture

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Kubernetes:** Amazon EKS
> **Primary AWS messaging:** SQS + SNS
> **Messaging alternative:** RabbitMQ
> **Event streaming platform:** Apache Kafka
> **Managed AWS option:** Amazon MSK
> **Self-managed option:** Kafka on EKS

---

# 1. What Is Kafka?

Kafka is a distributed event-streaming platform.

The easiest way to think about Kafka is:

> **Kafka stores an ordered stream of events, allowing multiple independent consumers to read those events at their own pace.**

The basic model is:

```text
Producer
   │
   ▼
Kafka Topic
   │
   ▼
Partition
   │
   ▼
Consumer Group
   │
   ▼
Consumer
```

Kafka is fundamentally different from a traditional work queue.

---

# 2. RabbitMQ vs Kafka Mental Model

RabbitMQ:

```text
Producer
   │
   ▼
Exchange
   │
   ▼
Queue
   │
   ▼
Consumer
   │
   ▼
ACK
```

Kafka:

```text
Producer
   │
   ▼
Topic
   │
   ▼
Partition
   │
   ▼
Offset
   │
   ▼
Consumer Group
```

RabbitMQ emphasizes message delivery and broker-side routing.

Kafka emphasizes durable event streams, partitions, offsets, retention and independent consumers.

---

# 3. Why Do We Need Kafka?

Imagine our e-commerce platform generates:

```text
OrderCreated
PaymentCompleted
InventoryReserved
ShipmentCreated
OrderDelivered
CustomerUpdated
ProductUpdated
PriceChanged
```

Many systems may want those events.

For example:

```text
OrderCreated
      │
      ▼
Kafka
      │
      ├── Analytics
      ├── Fraud
      ├── Recommendation
      ├── Reporting
      ├── Data Platform
      └── Search
```

Each consumer can independently process the event stream.

---

# 4. Kafka Does Not Behave Like SQS

With a typical SQS work queue:

```text
Message
   │
   ▼
Consumer
   │
   ▼
Delete
```

The message is primarily work waiting to be processed.

Kafka:

```text
Event
 │
 ▼
Partition
 │
 ├── Consumer Group A
 ├── Consumer Group B
 └── Consumer Group C
```

The event remains available according to the topic's retention policy.

Different consumer groups can read the same event independently.

---

# 5. Example

Suppose:

```text
OrderCreated
```

is published.

Kafka:

```text
orders topic
      │
      ▼
Partition
      │
      ├── analytics-group
      ├── fraud-group
      ├── recommendation-group
      └── reporting-group
```

The analytics system can process it.

The fraud system can also process it.

The reporting system can process it later.

They don't have to consume the same physical message from a single work queue.

---

# 6. Kafka Architecture

A production Kafka cluster contains brokers.

```text
Kafka Cluster
 │
 ├── Broker 1
 ├── Broker 2
 └── Broker 3
```

Topics are distributed across brokers through partitions.

Example:

```text
orders topic

Partition 0 → Broker 1
Partition 1 → Broker 2
Partition 2 → Broker 3
```

With replication:

```text
Partition 0
 ├── Leader   → Broker 1
 ├── Replica  → Broker 2
 └── Replica  → Broker 3
```

---

# 7. Kafka Broker

A broker is a Kafka server.

Example:

```text
broker-1
broker-2
broker-3
```

The cluster combines them into one distributed Kafka system.

Applications don't need to think of each broker as a completely separate messaging system.

---

# 8. Kafka Topic

A topic represents a logical stream of events.

Examples:

```text
orders
payments
inventory
shipments
customers
products
```

Our project could use:

```text
ecommerce.order-events
ecommerce.payment-events
ecommerce.inventory-events
```

Avoid creating random topics without ownership.

---

# 9. Topic Ownership

Define clear ownership.

Example:

```text
ecommerce.order-events
       │
       └── owned by order domain

ecommerce.payment-events
       │
       └── owned by payment domain
```

A platform team can manage Kafka infrastructure while application teams own event contracts.

---

# 10. Partition

A Kafka topic is divided into partitions.

Example:

```text
orders topic
 │
 ├── Partition 0
 ├── Partition 1
 ├── Partition 2
 └── Partition 3
```

Partitions provide:

* scalability
* parallelism
* ordering within a partition

---

# 11. Partition Is the Key to Kafka Scaling

Suppose:

```text
orders
```

has:

```text
1 partition
```

You have limited parallelism.

With:

```text
10 partitions
```

multiple consumers can process different partitions concurrently.

```text
P0 → Consumer 1
P1 → Consumer 2
P2 → Consumer 3
P3 → Consumer 4
...
```

This is one of Kafka's most important scaling concepts.

---

# 12. Ordering

Kafka guarantees ordering within a partition.

Suppose:

```text
Partition 0

Offset 100 → OrderCreated
Offset 101 → PaymentCompleted
Offset 102 → OrderShipped
```

A consumer reading that partition sees the events in that order.

But Kafka does **not** provide one global ordering across all partitions.

---

# 13. Global Ordering

Suppose:

```text
Partition 0
Order A

Partition 1
Order B
```

There is no single global ordering between:

```text
Order A
Order B
```

across those partitions.

Therefore don't say:

> "Kafka guarantees ordering."

Instead say:

> **Kafka guarantees ordering within a partition.**

---

# 14. Choosing a Partition Key

This is extremely important.

Suppose we use:

```text
orderId
```

as the partition key.

Then:

```text
ORD-1001
```

always maps to the same partition under the normal partitioning model.

Example:

```text
ORD-1001
   │
   ▼
Partition 3

OrderCreated
PaymentCompleted
OrderShipped
```

This preserves ordering for that order.

---

# 15. Bad Partition Key

Suppose we randomly distribute events for the same order:

```text
OrderCreated      → P1
PaymentCompleted  → P3
OrderShipped      → P0
```

Now a consumer cannot rely on partition ordering for that order.

Therefore partition-key design must come from business ordering requirements.

---

# 16. Partition Key and Hot Partitions

There is another problem.

Suppose:

```text
99% of events
```

use the same key:

```text
customerId = VIP-CUSTOMER
```

Then one partition can become extremely hot.

```text
Partition 0 → 1,000,000 events
Partition 1 → 10,000
Partition 2 → 9,000
Partition 3 → 11,000
```

This is poor distribution.

Partition-key design must balance:

```text
ordering requirement
+
distribution
```

---

# 17. Kafka Record

A Kafka record conceptually contains:

```text
key
value
timestamp
headers
```

Example:

```text
key = ORD-1001

value = {
  "eventType": "OrderCreated",
  "orderId": "ORD-1001"
}
```

The key influences partition selection.

---

# 18. Offset

Every record within a partition has an offset.

Example:

```text
Partition 0

100 → OrderCreated
101 → PaymentCompleted
102 → OrderShipped
103 → OrderDelivered
```

The offset identifies the record's position within that partition.

---

# 19. Why Offsets Matter

Suppose a consumer has processed:

```text
100
101
102
```

Its position can be represented through offsets.

If the consumer crashes, it can continue from the appropriate committed position rather than starting blindly from the beginning.

---

# 20. Consumer Group

A consumer group is a set of consumers cooperating to process topic partitions.

Example:

```text
orders topic
 │
 ├── P0
 ├── P1
 ├── P2
 └── P3

consumer-group-payment
 │
 ├── payment-1
 ├── payment-2
 ├── payment-3
 └── payment-4
```

Each partition is assigned to one consumer within that group at a time under normal group operation.

---

# 21. Consumer Groups Enable Parallelism

Suppose:

```text
4 partitions
```

and:

```text
4 consumers
```

you can have:

```text
P0 → C1
P1 → C2
P2 → C3
P3 → C4
```

If you have:

```text
8 consumers
```

with only 4 partitions, some consumers will have no partition assigned.

Therefore:

> Consumer parallelism within a group is fundamentally constrained by partition count.

---

# 22. Important Scaling Rule

Suppose:

```text
Topic = 6 partitions
Consumer group = 10 pods
```

You don't automatically get 10-way partition parallelism.

Maximum active partition consumers:

```text
6
```

under the normal one-consumer-per-partition assignment model.

So:

```text
Partitions ≥ desired consumer parallelism
```

must be considered during topic design.

---

# 23. Consumer Group Isolation

Suppose we have:

```text
orders topic
```

and:

```text
payment-group
fraud-group
analytics-group
```

Architecture:

```text
                   orders
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
       payment      fraud      analytics
        group       group         group
```

Each group gets its own logical consumption position.

---

# 24. Same Group vs Different Group

### Same group

```text
payment-1
payment-2
payment-3
```

They share work.

### Different groups

```text
payment-group
fraud-group
analytics-group
```

They independently consume the stream.

This distinction is critical.

---

# 25. Kafka Retention

Kafka normally retains records according to topic retention policies.

For example:

```text
retention = 7 days
```

means records can remain available for that retention period subject to the topic's configured policies and storage constraints.

This enables replay.

---

# 26. Replay

Suppose analytics-service had a bug.

```text
analytics-service
      X
```

The events remain in Kafka according to retention.

After fixing the service:

```text
Kafka
  │
  ▼
Replay events
  │
  ▼
analytics-service
```

This is one of Kafka's biggest advantages.

---

# 27. RabbitMQ Replay vs Kafka Replay

RabbitMQ:

```text
Message
  │
  ▼
Consumer
  │
  ▼
ACK
```

Kafka:

```text
Event
  │
  ▼
Consumer
  │
  ▼
Offset
```

The event remains available according to retention.

Kafka therefore makes historical event consumption a first-class capability.

---

# 28. Consumer Offset Commit

A consumer generally commits its progress.

Conceptually:

```text
Process record
     │
     ▼
Business operation
     │
     ▼
Commit offset
```

But the exact order matters.

Bad design:

```text
Commit offset
     ↓
Database operation
     ↓
Crash
```

Now Kafka believes the event has been processed even though the business operation failed.

---

# 29. Better Processing Model

A common approach:

```text
Read event
   │
   ▼
Validate
   │
   ▼
Process business operation
   │
   ▼
Persist result
   │
   ▼
Commit offset
```

But even this has distributed transaction considerations.

You need to understand:

```text
Kafka
+
Database
```

are separate systems.

---

# 30. Duplicate Processing

Suppose:

```text
Database commit
      │
      ▼
Consumer crashes
      X
      │
      ▼
Offset not committed
```

Kafka can deliver the event again.

Therefore:

> Kafka consumers should also be designed to be idempotent.

Same principle we learned with SQS and RabbitMQ.

---

# 31. Idempotency

Example:

```text
eventId = EVT-10001
```

Consumer checks:

```text
processed_events
```

If already processed:

```text
skip business operation
```

Otherwise:

```text
process
+
record event
```

This protects against duplicate business effects.

---

# 32. Producer Acknowledgement

Kafka producers have acknowledgement settings.

Conceptually:

```text
acks=0
acks=1
acks=all
```

### `acks=0`

Producer doesn't wait for broker acknowledgement.

Fast, but weaker durability guarantees.

### `acks=1`

Leader acknowledges the record.

### `acks=all`

Leader waits for the required in-sync replicas according to Kafka's replication configuration.

For important production events, stronger acknowledgement settings are generally preferred.

---

# 33. Replication

Suppose:

```text
Topic
Partition 0
```

has replication factor 3:

```text
P0
 ├── Broker 1 → Leader
 ├── Broker 2 → Replica
 └── Broker 3 → Replica
```

If one broker fails, another replica may be able to take over depending on cluster state.

---

# 34. Replication Factor

Example:

```text
replication.factor = 3
```

means each partition has three replicas.

This improves fault tolerance but increases:

```text
storage
network traffic
replication overhead
```

Therefore don't blindly choose the largest number.

---

# 35. ISR

ISR means:

> In-Sync Replicas

Conceptually:

```text
Partition 0

Leader
   │
   ├── Replica 1 → in sync
   └── Replica 2 → in sync
```

If a replica falls behind sufficiently, it may leave the ISR.

ISR health is an important production metric.

---

# 36. Leader

Each partition has a leader.

Producers and consumers interact with partition leaders according to Kafka's protocol.

Example:

```text
Partition 0
   │
   └── Leader → Broker 1

Partition 1
   │
   └── Leader → Broker 2

Partition 2
   │
   └── Leader → Broker 3
```

Leadership should be distributed across brokers to avoid concentrating all workload on one broker.

---

# 37. Broker Failure

Suppose:

```text
Broker 1
   X
```

and it was leader for:

```text
Partition 0
```

Kafka can elect an eligible replica as the new leader depending on the cluster configuration and replica state.

Conceptually:

```text
Broker 1
   X

Partition 0
   │
   ▼
Broker 2 → new leader
```

This is why replication and healthy ISR are critical.

---

# 38. `min.insync.replicas`

A useful production concept is:

```text
min.insync.replicas
```

Combined with:

```text
acks=all
```

it helps enforce stronger durability requirements.

Example:

```text
replication factor = 3
min.insync.replicas = 2
acks = all
```

The producer requires the configured minimum number of in-sync replicas to acknowledge the write.

Exact settings must be chosen based on availability and durability requirements.

---

# 39. Idempotent Producer

Kafka supports idempotent producer behavior.

This helps prevent certain duplicate records caused by producer retries.

Conceptually:

```text
Producer
   │
   ├── send
   │
   X network issue
   │
   └── retry
```

Without appropriate protection, retries can potentially create duplicates.

Idempotent producer support helps address this class of problem.

But:

> Idempotent producer does not mean your entire business workflow is magically exactly-once.

---

# 40. Kafka Exactly-Once Semantics

Kafka supports transactions and exactly-once processing patterns in specific Kafka-to-Kafka scenarios.

But be careful when your workflow includes:

```text
Kafka
+
RDS
+
external payment provider
```

You cannot casually claim the entire business transaction is exactly once.

External systems require their own idempotency/transaction design.

---

# 41. Compression

Kafka producers can compress records.

Common codecs include:

```text
gzip
snappy
lz4
zstd
```

Compression can reduce:

```text
network traffic
storage
```

at the cost of CPU.

For high-volume workloads, compression is often worth evaluating.

---

# 42. Batch Processing

Kafka producers can batch records.

Instead of:

```text
send
send
send
send
```

they can accumulate records and send batches.

This improves throughput.

But excessive batching can increase:

```text
latency
```

Therefore production tuning is always a trade-off:

```text
throughput
vs
latency
```

---

# 43. Topic Design

Avoid creating topics like:

```text
topic1
topic2
events
data
```

Use meaningful names:

```text
ecommerce.order-events
ecommerce.payment-events
ecommerce.inventory-events
```

Include ownership and purpose where useful.

---

# 44. Event Schema

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

This follows the same event-envelope principles we introduced with SQS/SNS.

---

# 45. Schema Evolution

Suppose version 1:

```json
{
  "orderId": "ORD-1001",
  "amount": 4999
}
```

Version 2:

```json
{
  "orderId": "ORD-1001",
  "amount": 4999,
  "currency": "INR"
}
```

Consumers shouldn't unexpectedly break.

Use compatible evolution rules.

For larger organizations, a schema registry and formal compatibility policy can become important.

---

# 46. Event Contract Ownership

Don't allow:

```text
order-service
   │
   └── changes event schema
          │
          X
   10 consumers break
```

Instead:

```text
Producer
   │
   ▼
Event Contract
   │
   ├── versioning
   ├── compatibility
   └── ownership
```

Platform and architecture teams can define organization-wide standards.

---

# 47. Kafka on EKS

We can run Kafka inside EKS.

Conceptually:

```text
EKS
 │
 ├── Kafka Operator
 │
 └── Kafka Cluster
      ├── Broker 0
      ├── Broker 1
      └── Broker 2
```

But Kafka is stateful and operationally complex.

This is why a managed service such as Amazon MSK is often preferable for production AWS deployments.

---

# 48. Kafka Operator

A Kubernetes operator can manage Kafka resources.

One widely used ecosystem is:

```text
Strimzi
```

Conceptually:

```text
EKS
 │
 ├── Strimzi Operator
 │
 └── Kafka Cluster
      ├── broker-0
      ├── broker-1
      └── broker-2
```

The operator manages Kafka-related Kubernetes resources and lifecycle operations.

---

# 49. Managed MSK

AWS provides Amazon Managed Streaming for Apache Kafka (Amazon MSK).

Architecture:

```text
EKS
 │
 └── Applications
       │
       ▼
      MSK
       │
       ├── Broker
       ├── Broker
       └── Broker
```

This removes much of the broker-management burden compared with running Kafka ourselves.

---

# 50. MSK vs Kafka on EKS

| Area                          | Amazon MSK       | Kafka on EKS   |
| ----------------------------- | ---------------- | -------------- |
| Broker management             | AWS-managed      | You            |
| Kubernetes integration        | Client-side      | Native         |
| Storage operations            | AWS-managed      | You            |
| Upgrades                      | Managed workflow | You            |
| Operator                      | Not required     | Usually useful |
| Infrastructure learning       | Lower            | Higher         |
| Production operational burden | Lower            | Higher         |
| AWS-native architecture       | Excellent        | Excellent      |

For our learning project, we should understand both.

---

# 51. Kafka Storage on EKS

Kafka is heavily dependent on storage performance.

Conceptually:

```text
Kafka broker
     │
     ▼
PVC
     │
     ▼
EBS
```

Storage sizing depends on:

```text
incoming data rate
retention period
replication factor
compression
headroom
segment files
recovery requirements
```

---

# 52. Storage Calculation

Suppose:

```text
100 GB/day incoming
```

Retention:

```text
7 days
```

Approximate logical data:

```text
700 GB
```

With replication factor 3:

```text
~2.1 TB
```

before considering operational overhead, compression effects, partition distribution, free-space requirements and other factors.

So don't size storage from CPU alone.

---

# 53. AZ Distribution

For production:

```text
AZ-A
  │
Broker 0

AZ-B
  │
Broker 1

AZ-C
  │
Broker 2
```

The goal is to avoid having all replicas in one AZ.

A failure-domain-aware design is essential.

---

# 54. Dedicated Kafka Nodes

For self-managed Kafka:

```text
Kafka node group
      │
      ├── taint
      └── label
```

Kafka pods can be scheduled onto dedicated nodes.

This avoids:

```text
application pod
      │
      ▼
same node
      │
      ▼
Kafka disk/network contention
```

---

# 55. Anti-Affinity

Avoid:

```text
Node-1
 ├── broker-0
 ├── broker-1
 └── broker-2
```

Better:

```text
Node-1 → broker-0
Node-2 → broker-1
Node-3 → broker-2
```

Use topology spread and anti-affinity appropriate to the operator and cluster design.

---

# 56. Kafka Networking

Applications need connectivity to Kafka bootstrap endpoints.

Conceptually:

```text
EKS Pod
   │
   ▼
Kafka bootstrap
   │
   ▼
Kafka brokers
```

Kafka clients need to be able to reach the broker addresses returned by the cluster metadata.

This means:

> Kafka networking is more than simply opening one port.

Advertised broker addresses must be reachable from clients.

---

# 57. Common Networking Failure

Application:

```text
Connected to bootstrap
```

then:

```text
cannot connect to broker-2
```

Why?

Possible causes:

```text
security group
NetworkPolicy
DNS
advertised listener
routing
TLS
port
private networking
```

The bootstrap connection succeeding does not guarantee every broker connection will work.

---

# 58. Kafka Security

Production Kafka should use:

```text
TLS
+
authentication
+
authorization
```

Depending on the deployment, authentication mechanisms may include:

```text
SASL
IAM-based authentication for supported AWS MSK configurations
```

Authorization should restrict which applications can:

```text
produce
consume
create
describe
alter
```

specific topics/resources.

---

# 59. Application Permissions

Example:

```text
payment-service
```

should have:

```text
WRITE → payment-events
READ  → order-events
```

It doesn't need:

```text
WRITE → every topic
READ  → every topic
```

This is least privilege.

---

# 60. Kafka Secrets

Don't put:

```text
username
password
certificate
private key
```

directly into:

```text
Git repository
Docker image
source code
```

Use:

```text
AWS Secrets Manager
Kubernetes Secrets
External Secrets
```

depending on the platform architecture.

Our next chapter after Kafka will cover secrets in detail.

---

# 61. Kafka Consumer Deployment

Example:

```text
payment-service
 │
 └── Deployment
      ├── Pod 1
      ├── Pod 2
      └── Pod 3
```

Each pod belongs to:

```text
consumer group:
payment-service
```

Kafka distributes partitions among the pods.

---

# 62. Kubernetes HPA

CPU-based HPA:

```text
CPU ↑
 ↓
more pods
```

But Kafka consumers often need scaling based on:

```text
consumer lag
```

rather than CPU alone.

This is where event-driven scaling becomes useful.

---

# 63. Consumer Lag

Suppose Kafka has:

```text
Latest offset = 10,000
```

Consumer has processed:

```text
Offset = 8,000
```

Approximate lag:

```text
2,000 records
```

Conceptually:

```text
Producer
   │
   ▼
Kafka
   │
   │ 2,000 pending
   ▼
Consumer
```

If lag continuously increases:

```text
incoming rate > processing rate
```

---

# 64. Lag Is a Critical Metric

A consumer may have:

```text
CPU = 40%
```

and still be unhealthy.

Why?

Because:

```text
Kafka incoming = 10,000 msg/sec
Consumer = 5,000 msg/sec
```

CPU may not be the limiting factor.

Therefore monitor:

```text
consumer lag
```

alongside CPU, memory and processing latency.

---

# 65. KEDA and Kafka

KEDA can scale consumers using Kafka lag metrics.

Conceptually:

```text
Kafka
 │
 ▼
Consumer Lag
 │
 ▼
KEDA
 │
 ▼
Deployment
 │
 ├── Pod 1
 ├── Pod 2
 ├── Pod 3
 └── ...
```

This can be more meaningful than CPU-based scaling for event-driven workloads.

---

# 66. Rebalancing

Suppose:

```text
payment group
 │
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

Pod 2 crashes:

```text
Pod 2
  X
```

Kafka detects the membership change and reassigns partitions.

This is called:

```text
Consumer Group Rebalancing
```

---

# 67. Why Rebalancing Matters

During rebalancing:

```text
Consumers
   │
   ▼
Partition reassignment
   │
   ▼
Processing can pause/change
```

Frequent rebalances can hurt throughput and increase latency.

---

# 68. Causes of Rebalancing

Common causes include:

```text
Consumer crashes
Pod restarts
Scaling up
Scaling down
Long processing pauses
Session timeout
Deployment rollout
Network problems
```

Therefore Kubernetes deployment strategy matters.

---

# 69. Kubernetes Rolling Deployment

Suppose:

```text
payment-service
3 pods
```

During deployment:

```text
Old Pod 1
Old Pod 2
Old Pod 3
```

are gradually replaced.

Kafka consumer groups may rebalance as membership changes.

Production deployments should therefore consider:

```text
graceful shutdown
consumer poll behavior
termination grace period
readiness
lifecycle hooks
```

---

# 70. Graceful Shutdown

A Kafka consumer shouldn't simply disappear.

Better:

```text
SIGTERM
   │
   ▼
Stop accepting new work
   │
   ▼
Finish safe processing
   │
   ▼
Commit appropriate offsets
   │
   ▼
Close consumer
```

This reduces unnecessary reprocessing.

---

# 71. Poison Events

Suppose:

```text
OrderCreated
```

always crashes the consumer.

Without proper handling:

```text
consume
 ↓
failure
 ↓
retry
 ↓
failure
 ↓
retry
```

You need an application-level failure strategy.

Unlike a simple queue DLQ pattern, Kafka retention and replay require careful handling of failed records.

---

# 72. Kafka DLQ / Dead-Letter Topic

A common pattern is:

```text
orders
  │
  ▼
consumer
  │
  X
  │
  ▼
orders.dlq
```

The consumer records:

```text
original topic
partition
offset
error
timestamp
```

along with the original event where appropriate.

---

# 73. Why DLQ Design Matters More in Kafka

If you simply keep retrying one bad record:

```text
Partition
 │
 ▼
Poison record
 │
 X
```

you may prevent subsequent records from being processed depending on your consumer implementation.

Therefore:

```text
retry policy
+
DLQ strategy
+
ordering requirements
```

must be designed together.

---

# 74. Retry Topic Pattern

A common architecture:

```text
orders
 │
 ▼
consumer
 │
 ├── success
 │
 └── transient failure
        │
        ▼
   orders.retry
        │
        ▼
     retry
        │
        ▼
      orders
```

Permanent failure:

```text
orders
  │
  ▼
orders.dlq
```

The exact topology should be designed to preserve required ordering and avoid uncontrolled retry loops.

---

# 75. Log Compaction

Kafka also supports log compaction.

Instead of retaining every historical record indefinitely, Kafka can retain the latest value for a key under compaction semantics.

Example:

```text
customerId = C100

name = A
name = B
name = C
```

Compaction can eventually retain the latest state:

```text
C100 → name=C
```

This is useful for certain state/event patterns.

---

# 76. Compaction vs Retention

### Time/size retention

```text
Keep events for 7 days
```

### Compaction

```text
Keep latest record for each key
```

They solve different problems.

Some topics can use both.

---

# 77. Kafka Streams

Kafka can also support stream-processing architectures.

Example:

```text
orders topic
    │
    ▼
Kafka Streams
    │
    ├── aggregation
    ├── filtering
    └── transformation
    │
    ▼
sales-summary topic
```

This is useful for real-time data processing.

For our initial DevOps project, we should understand the infrastructure before adding complex stream-processing workloads.

---

# 78. Kafka Connect

Kafka Connect provides a framework for moving data between Kafka and external systems.

Conceptually:

```text
RDS
 │
 ▼
Kafka Connect
 │
 ▼
Kafka
```

or:

```text
Kafka
 │
 ▼
Kafka Connect
 │
 ▼
Data Warehouse
```

This becomes relevant in data-platform architectures.

---

# 79. Kafka Monitoring

Monitor:

### Brokers

```text
CPU
Memory
Disk
Network
Request latency
Under-replicated partitions
Offline partitions
ISR changes
```

### Topics

```text
Bytes in
Bytes out
Messages in
Partition distribution
Retention
```

### Consumers

```text
Consumer lag
Rebalances
Commit failures
Processing latency
```

---

# 80. Critical Kafka Alerts

Examples:

```text
Broker unavailable
Under-replicated partitions > 0
Offline partitions > 0
Consumer lag above threshold
Disk usage high
Request latency high
ISR shrinking
Frequent consumer rebalances
Controller/quorum issues
```

The exact thresholds should be based on workload and SLA.

---

# 81. Under-Replicated Partitions

Suppose:

```text
Replication factor = 3
```

but only:

```text
2 replicas
```

are currently in sync.

That indicates a degraded replication state.

Don't ignore:

```text
under-replicated partitions
```

in production.

A cluster may still be serving traffic while its fault tolerance is degraded.

---

# 82. Disk Pressure

Kafka is storage-intensive.

Suppose:

```text
Disk = 90%
```

and incoming data continues.

Eventually:

```text
Disk = 95%
Disk = 98%
```

This can become a serious incident.

Monitor:

```text
disk utilization
retention
partition growth
segment cleanup
broker distribution
```

---

# 83. Uneven Partition Distribution

Bad:

```text
Broker 1 → 100 partitions
Broker 2 → 20
Broker 3 → 20
```

Broker 1 may become overloaded.

Good:

```text
Broker 1 → ~47
Broker 2 → ~46
Broker 3 → ~47
```

The exact balance depends on workload, partition sizes and leadership distribution.

---

# 84. Partition Count Is Difficult to Change

You can increase partition count, but increasing partitions can have consequences for:

```text
ordering
consumer parallelism
resource usage
file descriptors
metadata
rebalance behavior
```

You generally cannot simply reduce a topic's partition count in place.

Therefore:

> Partition planning should happen before production-scale workloads arrive.

---

# 85. Too Many Partitions

More partitions aren't always better.

Excessive partitions can increase:

```text
broker memory
metadata overhead
open files
controller workload
rebalancing complexity
recovery time
```

Choose partitions from measured throughput and parallelism requirements.

---

# 86. Low Traffic

Example:

```text
100 events/sec
```

You may only need a small cluster.

Focus on:

```text
HA
security
backup/recovery
monitoring
correct topic design
```

Don't create hundreds of partitions for a tiny workload.

---

# 87. Medium Traffic

Example:

```text
5,000 events/sec
```

Now evaluate:

```text
partition throughput
broker CPU
disk throughput
network throughput
consumer lag
replication traffic
```

Potential architecture:

```text
3 brokers
+
appropriate partitions
+
RF=3
```

Exact sizing requires load testing.

---

# 88. High Traffic

Example:

```text
100,000+ events/sec
```

Now Kafka's strengths become more relevant.

You must model:

```text
producer throughput
consumer throughput
partition count
message size
retention
replication
storage
network
compression
consumer groups
```

At this point, Kafka can be significantly more appropriate than a traditional work queue for the right event-streaming workload.

---

# 89. Kafka vs RabbitMQ — Practical Example

### Payment command

```text
"Process payment for order 1001"
```

RabbitMQ can be a natural fit:

```text
payment exchange
      │
      ▼
payment queue
      │
      ▼
payment-service
```

### Business event stream

```text
"Order 1001 was created"
```

Kafka can be a natural fit:

```text
order-events
      │
      ├── fraud
      ├── analytics
      ├── recommendation
      └── reporting
```

The distinction comes from requirements, not from a simplistic "Kafka is newer" rule.

---

# 90. Kafka vs SQS/SNS

Our AWS-native design:

```text
order-service
     │
     ▼
SNS
     │
     ├── payment SQS
     ├── inventory SQS
     └── notification SQS
```

Kafka alternative:

```text
order-service
     │
     ▼
Kafka
     │
     ▼
order-events
     │
     ├── payment-group
     ├── fraud-group
     ├── analytics-group
     └── reporting-group
```

The second architecture is better when retained event streams and independent replay are important.

---

# 91. MSK vs SQS/SNS

Amazon MSK:

```text
Event streaming
```

SQS/SNS:

```text
Managed queue + pub/sub
```

Choose SQS/SNS when:

```text
simple async workloads
AWS-native integration
low operational burden
```

Choose MSK when:

```text
event streaming
high throughput
long retention
replay
multiple independent consumers
stream processing
```

---

# 92. Kafka on EKS vs MSK

For our learning:

```text
Self-managed Kafka
        │
        ▼
EKS
 │
 └── Operator
      │
      └── Kafka
```

For many production AWS environments:

```text
EKS
 │
 └── Applications
       │
       ▼
      MSK
```

This lets the platform team focus on application/platform concerns rather than operating the Kafka brokers themselves.

---

# 93. Terraform Boundary

Terraform can provision:

```text
VPC
MSK
Security Groups
IAM
KMS
Networking
```

For self-managed Kafka:

```text
Terraform
   │
   ▼
EKS infrastructure
```

Then:

```text
ArgoCD
   │
   ▼
Kafka Operator
   │
   ▼
Kafka resources
```

Don't make Terraform and ArgoCD fight over the same Kubernetes resources.

---

# 94. GitOps Boundary

Our architecture:

```text
Terraform
   │
   ├── VPC
   ├── EKS
   ├── IAM
   └── MSK if selected

ArgoCD
   │
   ├── applications
   ├── Kafka operator
   ├── Kafka resources
   └── Kubernetes configuration
```

This keeps infrastructure and Kubernetes desired state separated.

---

# 95. Complete Kafka Production Architecture

```text
                         AWS
                          │
                         VPC
                          │
             ┌────────────┴────────────┐
             │                         │
            EKS                       MSK
             │                         │
      ┌──────┼──────┐                  │
      │      │      │                  │
 order   payment inventory             │
 service  service service              │
      │      │      │                  │
      └──────┴──────┴──────────────────┘
                     │
                     ▼
              Kafka Topics
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Fraud      Analytics   Reporting
       Group       Group       Group
```

For self-managed Kafka:

```text
EKS
 │
 ├── Kafka Operator
 │
 └── Kafka cluster
      ├── Broker 0
      ├── Broker 1
      └── Broker 2
```

---

# 96. Kafka Troubleshooting Flow

When an application can't consume:

```text
Application
    │
    ▼
DNS
    │
    ▼
Network/Security Group
    │
    ▼
TLS/Auth
    │
    ▼
Kafka bootstrap
    │
    ▼
Broker metadata
    │
    ▼
Topic
    │
    ▼
Partition
    │
    ▼
Consumer group
    │
    ▼
Offset
    │
    ▼
Business processing
```

Don't start by restarting Kafka.

---

# 97. Troubleshooting: Consumer Lag Increasing

Check:

```text
1. Incoming message rate
2. Consumer processing rate
3. Consumer pod count
4. Partition count
5. Consumer assignment
6. CPU/memory
7. Database latency
8. External API latency
9. Consumer errors
10. Rebalances
```

If:

```text
10 partitions
2 consumers
```

and:

```text
incoming > processing
```

scaling from:

```text
2 → 4 → 8
```

may help.

But if:

```text
partitions = 4
```

then scaling to 20 pods won't create 20-way partition parallelism.

---

# 98. Troubleshooting: Frequent Rebalances

Check:

```text
Pod restarts
Long GC pauses
Processing time
Consumer configuration
Session timeout
Heartbeat behavior
Deployment strategy
Network connectivity
```

Kubernetes:

```text
CrashLoopBackOff
```

can indirectly become:

```text
Kafka consumer rebalance
```

So Kafka troubleshooting must include Kubernetes troubleshooting.

---

# 99. Troubleshooting: Under-Replicated Partitions

Check:

```text
Broker health
Disk latency
Network
CPU
Memory
Replication traffic
Broker restarts
```

If one broker is slow:

```text
Broker 2
   ↓
replication falls behind
   ↓
ISR shrinks
   ↓
under-replicated partitions
```

This is a cluster health issue, not merely an application issue.

---

# 100. Troubleshooting: Disk Full

Check:

```text
df
PVC usage
topic growth
retention
partition sizes
broker distribution
log cleanup
```

Do not simply delete Kafka log files manually.

Kafka storage is part of the distributed data system.

---

# 101. Production Kafka Checklist

### Cluster

* [ ] Multi-broker
* [ ] Multi-AZ
* [ ] Appropriate replication factor
* [ ] ISR healthy
* [ ] Balanced partition leadership
* [ ] Storage capacity
* [ ] Network capacity

### Topics

* [ ] Clear naming
* [ ] Ownership
* [ ] Partition count
* [ ] Partition key
* [ ] Retention
* [ ] Compression
* [ ] Schema versioning
* [ ] Compaction where appropriate

### Producers

* [ ] Appropriate `acks`
* [ ] Idempotence
* [ ] Retry strategy
* [ ] Compression
* [ ] Batching
* [ ] Timeouts

### Consumers

* [ ] Consumer groups
* [ ] Offset management
* [ ] Idempotency
* [ ] Graceful shutdown
* [ ] Error handling
* [ ] DLQ/retry strategy
* [ ] Consumer lag monitoring
* [ ] Rebalance behavior

### Security

* [ ] TLS
* [ ] Authentication
* [ ] Authorization
* [ ] Topic-level permissions
* [ ] Secrets management
* [ ] Network isolation

### Observability

* [ ] Broker metrics
* [ ] Consumer lag
* [ ] Disk
* [ ] CPU
* [ ] Memory
* [ ] Network
* [ ] ISR
* [ ] Under-replicated partitions
* [ ] Rebalances
* [ ] Logs

---

# 102. Final Technology Decision for Our Project

We now have three messaging architectures:

```text
                  Messaging
                      │
       ┌──────────────┼──────────────┐
       │              │              │
      SQS          RabbitMQ        Kafka
       │              │              │
     Queue         Broker          Stream
       │              │              │
   AWS managed    MQ/EKS          MSK/EKS
```

And SNS adds:

```text
SNS
 │
 ├── SQS
 ├── SQS
 └── SQS
```

---

# 103. What We Will Actually Build

For the main enterprise e-commerce architecture:

```text
                         EKS
                          │
            ┌─────────────┼─────────────┐
            │             │             │
       Microservices    Redis          RDS
            │
            ▼
           SNS
            │
       ┌────┼────┐
       ▼    ▼    ▼
      SQS  SQS  SQS
       │    │    │
      Payment
      Inventory
      Notification
```

Then separately:

```text
RabbitMQ
   │
   └── Dedicated learning/alternative architecture
```

and:

```text
Kafka
   │
   └── Dedicated event-streaming architecture
```

This gives you the ability to explain **why** a technology was selected rather than simply saying:

> "I know Kafka, RabbitMQ and SQS."

---

# 104. The DevOps Perspective

As a DevOps engineer, don't stop at:

```text
Producer → Kafka
```

You need to think:

```text
How is Kafka provisioned?
        ↓
Where does it run?
        ↓
How is storage provisioned?
        ↓
How are AZs handled?
        ↓
How is networking configured?
        ↓
How is TLS configured?
        ↓
How are applications authenticated?
        ↓
How are topics managed?
        ↓
How are partitions sized?
        ↓
How are consumers scaled?
        ↓
How is lag monitored?
        ↓
How are failures handled?
        ↓
How is data recovered?
        ↓
How are upgrades performed?
        ↓
How is it integrated with GitOps?
```

That is the production DevOps perspective.

---

# 105. Final Mental Model

Remember Kafka like this:

```text
                         KAFKA
                           │
                    ┌──────┴──────┐
                    │             │
                 Topic         Topic
                    │
              ┌─────┼─────┐
              │     │     │
             P0    P1    P2
              │     │     │
              ▼     ▼     ▼
           offsets offsets offsets
              │     │     │
              └─────┼─────┘
                    │
             Consumer Group
              ┌─────┼─────┐
              ▼     ▼     ▼
             Pod   Pod   Pod
```

And production reliability:

```text
Producer
   │
   ▼
Kafka Topic
   │
   ▼
Partitions
   │
   ▼
Replication
   │
   ▼
Consumer Group
   │
   ▼
Processing
   │
   ▼
Offset Commit
```

The most important Kafka concepts to master are:

**Topic → Partition → Key → Offset → Consumer Group → Replication → ISR → Producer ACK → Consumer Lag → Rebalancing → Retention.**

---

## Messaging section completed

You now have:

```text
22-Redis.md             ✅
23-SQS-SNS.md           ✅
24-RabbitMQ.md          ✅
25-Kafka.md              ✅
```

The next chapter is:

# `26-Secrets-Management.md`

That chapter will connect **AWS Secrets Manager + KMS + EKS + Kubernetes Secrets + External Secrets + IAM/Pod Identity + ArgoCD + GitHub Actions + Spring Boot**, including the important production question:

> **Where does a database password actually live from the moment it is created in AWS until an application running in an EKS pod uses it—and how do we prevent that secret from appearing in Git, Terraform state, CI logs, Docker images, Kubernetes manifests, or application logs?**
