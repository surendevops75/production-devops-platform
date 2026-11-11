# 24-RabbitMQ.md

# RabbitMQ — Production Messaging on EKS

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Kubernetes:** Amazon EKS
> **Primary AWS messaging:** SQS + SNS
> **Alternative broker:** RabbitMQ
> **Deployment options:** Amazon MQ for RabbitMQ / RabbitMQ on EKS
> **Purpose:** Deep production understanding of broker-based messaging

---

# 1. What Is RabbitMQ?

RabbitMQ is a message broker.

The simplest mental model is:

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
```

Unlike SQS, where the producer generally sends directly to a queue, RabbitMQ introduces an important routing layer:

```text
Producer
    │
    ▼
 Exchange
    │
    ├── Queue A
    ├── Queue B
    └── Queue C
```

The exchange decides where messages should go.

RabbitMQ implements the AMQP messaging model with concepts such as exchanges, queues, bindings, acknowledgements and routing keys.

---

# 2. RabbitMQ vs SQS

This distinction is extremely important.

### SQS

```text
Producer
   │
   ▼
 SQS Queue
   │
   ▼
Consumer
```

### RabbitMQ

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
```

RabbitMQ provides much richer broker-side routing.

---

# 3. Why Would We Use RabbitMQ?

RabbitMQ is particularly useful when applications need:

* complex routing
* acknowledgements
* routing keys
* exchanges
* work queues
* priority handling
* controlled consumer flow
* request/reply patterns
* dead-letter exchanges
* broker-level messaging semantics

For example:

```text
order-service
     │
     ▼
order-events exchange
     │
     ├── payment queue
     ├── fraud queue
     ├── inventory queue
     └── notification queue
```

---

# 4. RabbitMQ Architecture

A production RabbitMQ architecture looks conceptually like:

```text
                         RabbitMQ Cluster
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 Broker 1            Broker 2
                    │                   │
                    └─────────┬─────────┘
                              │
                         Broker 3
```

Applications connect to the cluster:

```text
order-service ──────────┐
payment-service ────────┤
inventory-service ──────┤
notification-service ──┤
                         ▼
                  RabbitMQ Cluster
```

---

# 5. Core RabbitMQ Components

You need to understand these six concepts extremely well:

```text
1. Producer
2. Exchange
3. Queue
4. Binding
5. Routing Key
6. Consumer
```

Then:

```text
ACK
NACK
Publisher Confirm
Prefetch
DLX
TTL
Quorum Queue
```

become the reliability layer.

---

# 6. Producer

A producer creates a message.

Example:

```text
order-service
```

creates:

```json
{
  "eventType": "OrderCreated",
  "orderId": "ORD-1001",
  "customerId": "C-100"
}
```

The producer normally publishes to an exchange.

```text
order-service
      │
      ▼
RabbitMQ Exchange
```

---

# 7. Exchange

The exchange receives the message and determines which queues should receive it.

```text
Producer
   │
   ▼
Exchange
   │
   ├── Queue A
   ├── Queue B
   └── Queue C
```

RabbitMQ supports multiple exchange types.

The important ones are:

```text
Direct
Topic
Fanout
Headers
```

---

# 8. Direct Exchange

A direct exchange routes using an exact routing-key match.

Example:

```text
Exchange: ecommerce.events
```

Messages:

```text
routing key = payment.created
```

Binding:

```text
payment.created → payment-queue
```

Architecture:

```text
Producer
   │
   ▼
Direct Exchange
   │
   ├── payment.created → payment-queue
   └── inventory.created → inventory-queue
```

---

# 9. Topic Exchange

Topic exchanges support pattern-based routing.

Example:

```text
order.created
order.updated
order.cancelled
payment.created
payment.failed
```

A binding could be:

```text
order.*
```

which receives:

```text
order.created
order.updated
order.cancelled
```

Another:

```text
payment.*
```

receives payment events.

This is extremely useful for event-driven architectures.

---

# 10. Topic Wildcards

RabbitMQ topic patterns support:

```text
*
```

and:

```text
#
```

Conceptually:

```text
order.*
```

matches one word after `order`.

While:

```text
order.#
```

can match deeper routing paths.

Example:

```text
order.created
order.payment.created
order.payment.failed
```

This gives much more flexible routing than a simple queue.

---

# 11. Fanout Exchange

Fanout means:

> Send the message to all bound queues.

Architecture:

```text
                 Fanout Exchange
                       │
            ┌──────────┼──────────┐
            │          │          │
            ▼          ▼          ▼
        payment     inventory   notification
         queue        queue        queue
```

Routing key is effectively irrelevant for fanout routing.

This is similar conceptually to SNS fan-out.

---

# 12. Headers Exchange

Headers exchange routing is based on message headers rather than routing keys.

For example:

```text
department = payment
priority = high
```

This is less commonly used than direct/topic/fanout exchanges in many microservice systems.

---

# 13. Queue

The queue stores messages until consumers process them.

Example:

```text
payment-queue
```

Architecture:

```text
Exchange
   │
   ▼
payment-queue
   │
   ├── payment-pod-1
   ├── payment-pod-2
   └── payment-pod-3
```

---

# 14. Binding

A binding connects:

```text
Exchange
    ↓
Queue
```

Example:

```text
order-events
      │
      │ routing key:
      │ order.created
      ▼
order-processing-queue
```

The binding says:

> Messages matching this routing rule should be delivered to this queue.

---

# 15. Routing Key

A routing key provides information used by exchanges for routing.

Example:

```text
order.created
```

Another:

```text
payment.completed
```

Another:

```text
inventory.reserved
```

A useful naming scheme:

```text
<domain>.<entity>.<event>
```

For example:

```text
order.order.created
payment.payment.completed
inventory.stock.reserved
```

Keep the naming convention simple and consistent.

---

# 16. Complete Message Path

A complete RabbitMQ flow:

```text
order-service
      │
      │ publish
      ▼
order-events exchange
      │
      │ routing key
      ▼
order-created queue
      │
      ▼
order-consumer
```

With multiple consumers:

```text
                         Exchange
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
          payment-q     inventory-q   notification-q
              │             │             │
              ▼             ▼             ▼
          payment       inventory     notification
```

---

# 17. One Queue vs Multiple Queues

This is a major architectural decision.

Suppose:

```text
payment-service
inventory-service
notification-service
```

Need the same event.

Don't necessarily put all consumers against one queue:

```text
                    Queue
                  /   |   \
            payment inventory notification
```

RabbitMQ's competing-consumer behavior means consumers on the same queue compete for messages.

Instead:

```text
              Exchange
             /    |    \
            /     |     \
     payment-q inventory-q notification-q
```

Now every service gets its own copy.

---

# 18. Competing Consumers

Suppose:

```text
payment-queue
```

has:

```text
Pod-1
Pod-2
Pod-3
```

Messages are distributed among them.

```text
Message 1 → Pod-1
Message 2 → Pod-2
Message 3 → Pod-3
Message 4 → Pod-1
```

This allows horizontal scaling.

---

# 19. RabbitMQ Consumer Scaling

In EKS:

```text
payment-queue
      │
      ▼
payment-service Deployment
      │
      ├── Pod 1
      ├── Pod 2
      ├── Pod 3
      └── Pod 4
```

Increasing replicas increases consumer capacity.

But simply increasing pods doesn't always solve the problem.

You must consider:

```text
RabbitMQ capacity
Database connections
Downstream API capacity
Prefetch
CPU
Memory
Message processing time
```

---

# 20. Prefetch

Prefetch controls how many unacknowledged messages a consumer can receive.

Conceptually:

```text
prefetch = 10
```

means the consumer can have roughly:

```text
10 unacknowledged messages
```

in flight under the relevant RabbitMQ QoS configuration.

---

# 21. Why Prefetch Matters

Too low:

```text
prefetch = 1
```

may reduce throughput.

Too high:

```text
prefetch = 10,000
```

can cause:

* memory pressure
* unfair distribution
* slow consumers holding many messages
* increased recovery complexity

So prefetch is a tuning parameter.

---

# 22. Example

Suppose:

```text
RabbitMQ
   │
   ▼
1000 messages
```

Consumers:

```text
Pod-1
Pod-2
Pod-3
```

If each consumer prefetches too many messages, one pod may reserve a huge portion of the queue.

A more appropriate prefetch value can improve work distribution.

The correct number depends on:

```text
message size
processing time
consumer concurrency
workload variability
```

---

# 23. Acknowledgement

RabbitMQ consumers can explicitly acknowledge successful processing.

Conceptually:

```text
RabbitMQ
    │
    ▼
Consumer
    │
    ▼
Process
    │
    ▼
ACK
```

The broker can then consider that delivery successfully handled.

---

# 24. What If Consumer Crashes?

Suppose:

```text
RabbitMQ
   │
   ▼
Consumer
   X
```

before acknowledgement.

The broker can requeue the unacknowledged message according to the configured behavior.

Conceptually:

```text
Message
   │
   ▼
Consumer
   X
   │
   ▼
Requeue
   │
   ▼
Another consumer
```

This is one reason acknowledgements matter.

---

# 25. ACK Only After Successful Processing

Bad:

```text
Receive
  ↓
ACK
  ↓
Process
  ↓
Crash
```

Now the message may already be considered complete.

Better:

```text
Receive
  ↓
Validate
  ↓
Process
  ↓
Persist result
  ↓
ACK
```

The exact transaction boundary must be designed carefully.

---

# 26. Publisher Confirms

Consumer acknowledgements solve one side of the reliability problem.

What about the producer?

Suppose:

```text
order-service
      │
      ▼
RabbitMQ
```

The producer wants confidence that the broker accepted the message.

Publisher confirms provide that mechanism.

Conceptually:

```text
Producer
   │
   ▼
RabbitMQ
   │
   ▼
Confirm
   │
   ▼
Producer knows broker accepted publication
```

This is different from consumer ACK.

---

# 27. Publisher Confirm vs Consumer ACK

Remember:

```text
Publisher Confirm
Producer ←→ Broker
```

while:

```text
Consumer ACK
Consumer ←→ Broker
```

They solve different problems.

```text
Producer
   │
   │ Publisher Confirm
   ▼
RabbitMQ
   │
   │ Consumer ACK
   ▼
Consumer
```

---

# 28. Durable Queue

Production queues generally need appropriate durability.

Conceptually:

```text
durable queue
```

means the queue definition survives broker restart.

But:

> Durable queue does not automatically mean every message is safely persisted under every failure scenario.

Message persistence and queue durability are separate concepts.

---

# 29. Persistent Messages

A producer can publish persistent messages.

Conceptually:

```text
Queue = durable
Message = persistent
```

This is much more appropriate for important business events than:

```text
non-durable
+
transient
```

But persistence has performance/storage implications.

---

# 30. RabbitMQ Reliability Has Multiple Layers

Think:

```text
                 Reliability
                     │
       ┌─────────────┼─────────────┐
       │             │             │
 Durable Queue   Persistent     Publisher
                 Message        Confirms
       │                           │
       └──────────────┬────────────┘
                      │
               Consumer ACK
                      │
                      ▼
               Idempotent app
```

You need to understand all of these independently.

---

# 31. Dead Letter Exchange

RabbitMQ has a powerful dead-letter mechanism.

Conceptually:

```text
payment-queue
      │
      ├── success
      │
      └── rejected/expired
              │
              ▼
        Dead Letter Exchange
              │
              ▼
          payment-dlq
```

DLX is the RabbitMQ equivalent of an important dead-letter architecture pattern.

---

# 32. Retry Design

Avoid:

```text
Consumer
   ↓
failure
   ↓
immediate retry
   ↓
failure
   ↓
immediate retry
   ↓
...
```

This can create a retry storm.

Instead:

```text
Main Queue
    │
    ▼
Failure
    │
    ▼
Retry Queue
    │
    ▼
Delay/TTL
    │
    ▼
Main Queue
```

or use an appropriate retry topology.

---

# 33. Retry Queue Example

```text
payment-queue
      │
      ▼
payment-consumer
      │
      X
      │
      ▼
payment-retry-1
      │
      ▼
delay
      │
      ▼
payment-queue
```

Then:

```text
retry-1
retry-2
retry-3
      │
      ▼
payment-dlq
```

This provides controlled retry behavior.

---

# 34. Exponential Retry

Example:

```text
Attempt 1 → 5 sec
Attempt 2 → 15 sec
Attempt 3 → 30 sec
Attempt 4 → 60 sec
```

Add jitter where appropriate.

This prevents many consumers from retrying simultaneously after a shared dependency failure.

---

# 35. TTL

RabbitMQ supports message and queue TTL mechanisms.

For example:

```text
retry queue
TTL = 30 seconds
```

The message remains there for the configured lifetime before being dead-lettered according to the topology.

TTL can be useful for retry architectures.

---

# 36. Important TTL Warning

Don't blindly use TTL for business data.

Example:

```text
OrderCreated
TTL = 30 sec
```

If the consumer is down for 2 minutes:

```text
message expires
```

You may lose an important business event.

TTL should reflect an intentional business/technical requirement.

---

# 37. Quorum Queues

For important production queues, RabbitMQ's quorum queues are a key concept.

They are designed around replicated state using a consensus-based approach and provide stronger data safety characteristics than classic mirrored-queue approaches.

Conceptually:

```text
             Quorum Queue
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Node 1    Node 2    Node 3
```

A quorum queue is not simply:

```text
same queue copied randomly
```

It has defined replication and leader/follower behavior.

---

# 38. Why Quorum Queues Matter

Suppose:

```text
RabbitMQ Node 1
       X
```

A replicated quorum queue can continue operating when the cluster retains the required quorum.

This is much more suitable for important workloads than treating one broker as the only copy of business messages.

---

# 39. RabbitMQ Cluster

Production RabbitMQ should not normally depend on a single broker for critical workloads.

Conceptually:

```text
AWS VPC
│
└── EKS
    │
    └── RabbitMQ
        ├── Node 1 → AZ-A
        ├── Node 2 → AZ-B
        └── Node 3 → AZ-C
```

The exact topology depends on the RabbitMQ deployment/operator and AWS/EKS design.

---

# 40. Availability Zones

For production:

```text
AZ-A
RabbitMQ node 1

AZ-B
RabbitMQ node 2

AZ-C
RabbitMQ node 3
```

This protects against a single-AZ failure.

But remember:

> Three pods in three AZs do not automatically guarantee application-level HA.

You must configure RabbitMQ clustering, queue replication and client failover correctly.

---

# 41. RabbitMQ Operator on Kubernetes

Running RabbitMQ directly through manually written StatefulSets is not normally how I would build a serious production platform.

A RabbitMQ Kubernetes Operator can manage RabbitMQ resources and lifecycle.

Conceptually:

```text
EKS
 │
 ├── RabbitMQ Operator
 │
 └── RabbitMQ Cluster
      ├── node-0
      ├── node-1
      └── node-2
```

The operator becomes part of the platform layer.

---

# 42. Production Tool Placement

For our project:

```text
Terraform
   │
   └── AWS infrastructure

ArgoCD
   │
   └── Kubernetes desired state

RabbitMQ Operator
   │
   └── RabbitMQ cluster

RabbitMQ
   │
   └── messaging

Applications
   │
   └── RabbitMQ clients
```

This maintains clear ownership.

---

# 43. RabbitMQ on EKS vs Amazon MQ

We have two major approaches.

### Option 1 — Amazon MQ for RabbitMQ

```text
EKS applications
       │
       ▼
Amazon MQ
       │
       ▼
RabbitMQ
```

AWS manages much of the underlying broker infrastructure.

### Option 2 — RabbitMQ on EKS

```text
EKS
 │
 ├── Operator
 │
 └── RabbitMQ cluster
```

You own considerably more operational responsibility.

---

# 44. Amazon MQ Advantage

Managed broker:

```text
Less infrastructure management
```

You still need to understand:

* queues
* exchanges
* bindings
* authentication
* permissions
* topology
* monitoring
* application behavior

Managed does not mean zero engineering responsibility.

---

# 45. RabbitMQ on EKS Advantage

Running it ourselves teaches:

```text
Kubernetes
+
Stateful workloads
+
Storage
+
Networking
+
HA
+
Operators
+
Broker operations
```

This is excellent for DevOps learning.

But production operation becomes significantly more complex.

---

# 46. RabbitMQ Storage on EKS

RabbitMQ is stateful.

Therefore:

```text
RabbitMQ Pod
    │
    ▼
PVC
    │
    ▼
EBS
```

The EBS CSI driver provides persistent storage integration.

But remember:

```text
PVC ≠ HA
```

Storage persistence alone does not make RabbitMQ highly available.

---

# 47. EBS and AZs

EBS volumes are AZ-scoped.

Therefore:

```text
Node in AZ-A
   │
   ▼
EBS volume in AZ-A
```

Moving a workload to AZ-B may require a different storage attachment/topology strategy.

This is why StatefulSet storage topology and scheduling are important.

---

# 48. Resource Requests

RabbitMQ should have explicit resources.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

These are illustrative values only.

Real sizing should come from:

```text
message rate
message size
connections
queues
consumer count
retention
disk I/O
replication
```

---

# 49. Dedicated Nodes

For serious broker workloads, consider dedicated capacity.

Example:

```text
RabbitMQ nodes
      │
      ▼
Dedicated node group
      │
      ├── taint
      └── labels
```

Then RabbitMQ pods use:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: messaging
    effect: NoSchedule
```

and:

```yaml
nodeSelector:
  workload: messaging
```

This prevents noisy application workloads from competing directly with the broker.

---

# 50. Anti-Affinity

Don't put all RabbitMQ replicas on one node.

Bad:

```text
Node-1
 ├── rabbit-0
 ├── rabbit-1
 └── rabbit-2
```

If Node-1 fails:

```text
everything gone
```

Better:

```text
Node-1 → rabbit-0
Node-2 → rabbit-1
Node-3 → rabbit-2
```

Use topology spread/anti-affinity according to the operator's supported configuration.

---

# 51. Network Security

RabbitMQ commonly uses:

```text
5672
```

for AMQP.

TLS commonly uses:

```text
5671
```

when configured.

Management UI/API commonly uses:

```text
15672
```

depending on deployment/configuration.

Do not expose the management interface publicly.

---

# 52. NetworkPolicy

Example concept:

```text
payment namespace
      │
      ▼
RabbitMQ
```

Allow only approved workloads to connect.

```text
order-service ─────► RabbitMQ
payment-service ───► RabbitMQ
inventory-service ─► RabbitMQ
```

Avoid:

```text
any pod ─────► RabbitMQ
```

---

# 53. Security Architecture

Production:

```text
Application Pod
      │
      ▼
TLS
      │
      ▼
RabbitMQ
      │
      ├── Authentication
      ├── Authorization
      └── Virtual Host
```

Credentials should be stored through a proper secret-management mechanism rather than committed to Git.

Our later:

```text
26-Secrets-Management.md
```

will cover this in depth.

---

# 54. RabbitMQ Virtual Hosts

RabbitMQ supports virtual hosts.

Conceptually:

```text
RabbitMQ
 │
 ├── /production
 │
 ├── /qa
 │
 └── /development
```

They provide logical separation within a RabbitMQ deployment.

However, don't assume virtual hosts alone are sufficient isolation for environments requiring strong security boundaries.

Separate infrastructure/accounts/clusters may be preferable for strong isolation.

---

# 55. RabbitMQ Permissions

Permissions can control:

```text
configure
write
read
```

for users against virtual hosts/resources.

Example:

```text
payment-service
   │
   ├── write → payment exchange
   └── read  → payment queue
```

It shouldn't automatically have access to every queue.

---

# 56. Application Connection Pooling

Applications maintain RabbitMQ connections/channels.

Don't create a new broker connection for every message.

Bad:

```text
message
 ↓
create connection
 ↓
publish
 ↓
close
```

Better:

```text
Application
    │
    └── long-lived connection
            │
            ├── channel
            ├── channel
            └── channel
```

Use the client library/framework's recommended connection management.

---

# 57. Connection Explosion

Suppose:

```text
20 microservices
```

each has:

```text
50 pods
```

and every pod creates many connections/channels.

You can suddenly have:

```text
1000+ application instances
```

connecting to RabbitMQ.

This can become a broker bottleneck.

Always monitor:

```text
connections
channels
consumers
```

---

# 58. Kubernetes HPA and RabbitMQ

CPU-based HPA:

```text
CPU ↑
 ↓
more pods
```

doesn't necessarily mean:

```text
queue backlog ↓
```

For consumers, a better signal may be:

```text
queue depth
message age
processing latency
```

KEDA can be used for event-driven scaling where appropriate.

---

# 59. Consumer Lag

RabbitMQ doesn't use Kafka's offset/consumer-lag model.

Instead, monitor things such as:

```text
ready messages
unacknowledged messages
consumer count
message rates
processing time
```

Example:

```text
Ready = 10,000
Unacked = 100
Consumers = 3
```

This suggests the queue is significantly behind.

---

# 60. Ready vs Unacked

Very useful distinction.

### Ready

```text
messages waiting for delivery
```

### Unacked

```text
messages delivered but not yet acknowledged
```

Architecture:

```text
Queue
 │
 ├── Ready
 │     └── waiting
 │
 └── Unacked
       └── consumers processing
```

This distinction helps diagnose whether the bottleneck is:

```text
consumer capacity
```

or:

```text
slow processing
```

---

# 61. Troubleshooting: Queue Growing

Suppose:

```text
Ready messages
100
500
1000
5000
```

Check:

```text
1. Consumer count
2. Consumer health
3. Consumer errors
4. Processing latency
5. Prefetch
6. Database performance
7. Downstream APIs
8. RabbitMQ resource usage
```

Don't immediately increase replicas.

---

# 62. Troubleshooting: Consumers Connected but No Progress

You see:

```text
Consumers = 10
```

but:

```text
Ready messages = 20,000
```

Check:

```text
Unacked messages
```

If:

```text
Unacked = 10,000
```

consumers may be processing slowly or holding too many messages because of prefetch.

---

# 63. Troubleshooting: Messages Reappearing

Symptoms:

```text
Message delivered
   ↓
Consumer crashes
   ↓
Message appears again
```

Check:

```text
Application exceptions
Pod restarts
OOMKilled
Network disconnects
ACK logic
RabbitMQ connection logs
```

---

# 64. Troubleshooting: Duplicate Business Operation

Suppose:

```text
OrderCreated
```

causes two invoices.

RabbitMQ may not be the only issue.

Investigate:

```text
Publisher retry
Consumer redelivery
Application retry
Database transaction
ACK timing
Idempotency
```

Then implement a durable idempotency mechanism.

---

# 65. Troubleshooting: RabbitMQ Memory Alarm

RabbitMQ can apply flow-control/resource protection when memory or other resources become constrained.

Symptoms can include:

```text
Publishers slowing/stalling
Consumer behavior changes
High memory
```

Check:

```text
memory usage
queue depth
message size
unacked messages
connection count
prefetch
disk space
```

Don't simply restart the broker.

---

# 66. Troubleshooting: Disk Alarm

If disk capacity becomes dangerously low:

```text
RabbitMQ
   │
   ▼
disk pressure
```

RabbitMQ may block operations to protect itself.

Check:

```text
PVC capacity
disk usage
message persistence
queue backlog
retention
logs
```

This is one reason persistent storage monitoring is essential.

---

# 67. Troubleshooting Chain

When RabbitMQ processing fails:

```text
Application
    │
    ▼
Network
    │
    ▼
RabbitMQ connection
    │
    ▼
Exchange
    │
    ▼
Binding
    │
    ▼
Queue
    │
    ▼
Consumer
    │
    ▼
Business processing
    │
    ▼
Database/API
```

Troubleshoot from the layer where the symptom actually occurs.

---

# 68. Monitoring

Monitor RabbitMQ itself.

Important metrics include:

```text
Messages ready
Messages unacked
Publish rate
Deliver rate
Ack rate
Consumer count
Connection count
Channel count
Memory
Disk
CPU
Network
Queue count
Node health
```

RabbitMQ provides a management interface and metrics mechanisms for operational monitoring.

Later we will integrate these into:

```text
Prometheus
Grafana
Alertmanager
```

---

# 69. Logging

RabbitMQ logs should go into our centralized logging architecture:

```text
RabbitMQ
   │
   ▼
stdout/files
   │
   ▼
Fluent Bit
   │
   ▼
Logstash
   │
   ▼
Elasticsearch
   │
   ▼
Kibana
```

This connects directly with the ELK architecture we already designed.

---

# 70. Alerting

Useful alerts:

```text
RabbitMQ node unavailable
Queue backlog high
Message age high
DLQ messages > 0
Consumer count unexpectedly low
Memory high
Disk low
Connection count high
Channel count high
Replication/quorum problems
PVC almost full
```

---

# 71. Low Traffic

Example:

```text
10 messages/sec
```

Architecture:

```text
RabbitMQ
   │
   ├── Node 1
   ├── Node 2
   └── Node 3

payment-service
   └── 2 pods
```

You may not need huge broker capacity.

Focus on:

```text
HA
security
correctness
observability
```

rather than over-sizing.

---

# 72. Medium Traffic

Example:

```text
500 messages/sec
```

Now evaluate:

```text
message size
processing time
consumer count
database throughput
broker CPU
disk I/O
network
```

Potential architecture:

```text
RabbitMQ cluster
      │
      ▼
payment queue
      │
 ┌────┼────┐
 ▼    ▼    ▼
Pod  Pod  Pod
```

Scale consumers based on actual backlog and processing capacity.

---

# 73. High Traffic

Example:

```text
10,000+ messages/sec
```

Now the architecture needs serious capacity planning.

Evaluate:

```text
RabbitMQ node count
queue distribution
message size
publisher rate
consumer rate
storage throughput
network
replication overhead
connection management
prefetch
```

At very high event-streaming workloads, Kafka may become a more appropriate technology depending on the requirements.

---

# 74. RabbitMQ Is Not Kafka

This is one of the most important concepts.

RabbitMQ:

```text
Message broker
```

Kafka:

```text
Distributed event streaming platform
```

RabbitMQ focuses heavily on:

```text
message delivery
routing
queues
acknowledgements
work distribution
```

Kafka focuses heavily on:

```text
event logs
partitions
retention
replay
consumer groups
stream processing
```

---

# 75. RabbitMQ vs Kafka Example

Suppose:

```text
OrderCreated
```

RabbitMQ:

```text
OrderCreated
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
OrderCreated
     │
     ▼
Kafka Topic
     │
     ▼
Partition
     │
 ┌───┼────┐
 ▼   ▼    ▼
CG1 CG2  CG3
```

Multiple consumer groups can independently process the retained event stream.

---

# 76. RabbitMQ vs SQS/SNS vs Kafka

| Requirement             | SQS/SNS                   | RabbitMQ                        | Kafka                  |
| ----------------------- | ------------------------- | ------------------------------- | ---------------------- |
| Simple async jobs       | Excellent                 | Excellent                       | Usually overkill       |
| AWS managed queue       | Excellent                 | Amazon MQ                       | MSK                    |
| Complex routing         | Moderate                  | Excellent                       | Different model        |
| Work queues             | Excellent                 | Excellent                       | Possible but different |
| Event replay            | Limited                   | Not primary                     | Excellent              |
| Long event retention    | Not primary               | Not primary                     | Excellent              |
| Very high event streams | Possible                  | Possible but evaluate carefully | Excellent              |
| Operational simplicity  | Excellent                 | Moderate                        | Moderate/High          |
| Broker routing          | SNS/filtering             | Very strong                     | Topic/partition model  |
| Consumer groups         | Application/KEDA patterns | Competing consumers             | Native                 |

---

# 77. Where RabbitMQ Fits in Our Project

Our **primary AWS production messaging design** remains:

```text
EKS
 │
 ├── SNS
 │    ├── payment SQS
 │    ├── inventory SQS
 │    ├── notification SQS
 │    └── ...
 │
 └── Applications
```

RabbitMQ is our alternative architecture:

```text
EKS
 │
 └── RabbitMQ cluster
       │
       ├── exchanges
       ├── queues
       ├── DLX
       └── consumers
```

We'll learn both.

---

# 78. Should We Deploy Both?

Not necessarily.

Don't build:

```text
SQS
+
SNS
+
RabbitMQ
+
Kafka
```

into every application just to demonstrate knowledge.

That creates unnecessary operational complexity.

Instead:

```text
Primary architecture
        │
        ▼
SQS + SNS

Alternative architecture
        │
        ▼
RabbitMQ

Streaming architecture
        │
        ▼
Kafka
```

This is a much more realistic design.

---

# 79. Production RabbitMQ Architecture

Our conceptual production design:

```text
                           AWS
                            │
                            ▼
                           VPC
                            │
                           EKS
                            │
              ┌─────────────┴─────────────┐
              │                           │
       Application Pods             RabbitMQ Cluster
              │                     ┌─────┼─────┐
              │                     │     │     │
              └────────────────────►Node1 Node2 Node3
                                      │     │     │
                                      └─────┼─────┘
                                            │
                                       Quorum Queues
```

With observability:

```text
RabbitMQ
   │
   ├── Metrics → Prometheus
   ├── Logs → Fluent Bit → Logstash → Elasticsearch
   └── Alerts → Alertmanager
```

---

# 80. RabbitMQ Production Checklist

### Architecture

* [ ] Cluster design
* [ ] AZ distribution
* [ ] Quorum queues where appropriate
* [ ] Dedicated capacity where needed
* [ ] Storage design
* [ ] Network design

### Messaging

* [ ] Exchange types
* [ ] Routing keys
* [ ] Bindings
* [ ] Queue ownership
* [ ] Durable queues
* [ ] Persistent messages
* [ ] Publisher confirms
* [ ] Consumer ACKs
* [ ] Idempotency

### Reliability

* [ ] DLX
* [ ] DLQ
* [ ] Retry strategy
* [ ] Backoff
* [ ] Poison message handling
* [ ] Recovery/runbook

### Kubernetes

* [ ] Operator
* [ ] Stateful architecture
* [ ] PVCs
* [ ] EBS CSI
* [ ] topology spread
* [ ] anti-affinity
* [ ] resource requests/limits
* [ ] PodDisruptionBudget
* [ ] controlled upgrades

### Security

* [ ] TLS
* [ ] authentication
* [ ] authorization
* [ ] NetworkPolicy
* [ ] secrets management
* [ ] least privilege
* [ ] no public management UI

### Observability

* [ ] Prometheus metrics
* [ ] Grafana dashboards
* [ ] Alertmanager
* [ ] centralized logs
* [ ] queue backlog alerts
* [ ] DLQ alerts
* [ ] disk alerts
* [ ] memory alerts

---

# 81. Final Mental Model

Remember RabbitMQ like this:

```text
                    RABBITMQ
                       │
                       ▼
                   Exchange
                       │
              ┌────────┼────────┐
              │        │        │
              ▼        ▼        ▼
            Queue    Queue    Queue
              │        │        │
              ▼        ▼        ▼
          Consumer Consumer Consumer
              │        │        │
             ACK      ACK      ACK
```

And reliability:

```text
Producer
   │
   ├── Publisher Confirm
   │
   ▼
Exchange
   │
   ▼
Durable Queue
   │
   ▼
Consumer
   │
   ├── success → ACK
   │
   └── failure → retry/DLX/DLQ
```

And production:

```text
AWS
 │
 ▼
EKS
 │
 ├── RabbitMQ Operator
 │
 └── RabbitMQ Cluster
      ├── AZ-A
      ├── AZ-B
      └── AZ-C
```

**The key lesson:** RabbitMQ isn't simply "another queue." Its value comes from the combination of **broker-side routing + queues + acknowledgements + publisher confirms + retry/dead-letter topology + controlled consumer concurrency + clustered operation**.

---

## Next file: `25-Kafka.md`

The next chapter will be substantially different because Kafka is **not just another RabbitMQ/SQS replacement**. We'll cover:

```text
Kafka architecture
        ↓
Broker
        ↓
Topic
        ↓
Partition
        ↓
Leader
        ↓
Follower
        ↓
Replication
        ↓
ISR
        ↓
Producer ACK
        ↓
Idempotent producer
        ↓
Consumer group
        ↓
Offsets
        ↓
Rebalancing
        ↓
Retention
        ↓
Compaction
        ↓
Ordering
        ↓
Kafka on EKS
        ↓
Kafka Operator
        ↓
Storage/AZ design
        ↓
MSK vs self-managed Kafka
        ↓
Monitoring
        ↓
Production troubleshooting
```
