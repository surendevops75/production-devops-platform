# 116-Kafka-Production-Implementation.md

````markdown
# 116-Kafka-Production-Implementation.md

# Kafka Production Implementation on AWS EKS

**Project:** Enterprise E-Commerce Platform  
**Platform:** AWS EKS  
**Kafka Platform:** Amazon MSK  
**Infrastructure:** Terraform  
**Deployment:** Helm + ArgoCD  
**Observability:** Prometheus + Grafana + Alertmanager  
**Primary Region:** `ap-south-1`  
**Environment:** Production

---

# 1. Objective

In the previous chapters we implemented:

- SQS
- SQS DLQ / Retry / Idempotency
- SNS
- RabbitMQ

In this chapter we implement **Apache Kafka in a production-style EKS environment**.

The goal is not simply to install Kafka.

We need to understand:

```text
Why Kafka?
How Kafka works
How Kafka differs from SQS/SNS/RabbitMQ
How to deploy Kafka
How to create topics
How partitions work
How replication works
How producers work
How consumers work
How consumer groups work
How offsets work
How to secure Kafka
How EKS workloads authenticate to Kafka
How to scale consumers
How to monitor consumer lag
How to handle failures
How to handle retries
How to handle DLQ
How to replay events
How to use Kafka with PostgreSQL
How to use the transactional outbox
How Kafka fits into our 21-microservice architecture
````

---

# 2. Kafka in Our Enterprise E-Commerce Platform

Our platform contains 21 microservices:

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

Kafka is not automatically required by every service.

We use Kafka where we need:

* High-throughput event streaming
* Durable event retention
* Multiple independent consumers
* Event replay
* Partition-based parallelism
* Event-driven processing
* Decoupling between services
* Consumer-specific processing speeds

Example:

```text
                    +-------------------+
                    |   Order Service   |
                    +---------+---------+
                              |
                              | OrderCreated
                              v
                    +-------------------+
                    |    Amazon MSK     |
                    |                   |
                    | orders.created.v1 |
                    +---------+---------+
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
       Inventory          Payment           Fraud
       Consumer            Consumer         Consumer
       Group               Group            Group
             |                |                |
             v                v                v
        Inventory DB      Payment DB       Fraud DB
```

Each consumer group receives the event independently.

---

# 3. Why Kafka?

Consider this business event:

```text
OrderCreated
```

Several services may need the same event.

```text
OrderCreated
     |
     +----> Inventory
     |
     +----> Payment
     |
     +----> Fraud
     |
     +----> Notification
     |
     +----> Reporting
```

With Kafka, each service can maintain its own consumer group.

For example:

```text
Topic:
ecommerce.orders.created.v1

Consumer Groups:

inventory-order-consumers
payment-order-consumers
fraud-order-consumers
notification-order-consumers
reporting-order-consumers
```

Every consumer group maintains its own position in the topic.

This means Reporting can consume historical events without affecting Payment.

---

# 4. Kafka vs SQS vs SNS vs RabbitMQ

We already implemented other messaging technologies.

The architecture should deliberately choose the appropriate technology.

| Technology | Main Purpose                                       |
| ---------- | -------------------------------------------------- |
| SQS        | Managed queue / asynchronous work                  |
| SNS        | Pub/Sub fan-out                                    |
| RabbitMQ   | Broker routing / queues / messaging                |
| Kafka      | Durable event streaming / replay / high throughput |

## Example

### SQS

```text
Producer
   |
   v
SQS Queue
   |
   +--> Consumer
```

Generally the message represents work that needs to be processed.

---

### SNS + SQS

```text
                 SNS
                  |
       +----------+----------+
       |          |          |
       v          v          v
     SQS-A      SQS-B      SQS-C
       |          |          |
       v          v          v
   Service A  Service B  Service C
```

Good for managed event fan-out.

---

### RabbitMQ

```text
Producer
   |
   v
Exchange
   |
   +--> Queue A
   +--> Queue B
   +--> Queue C
```

Routing is a major strength.

---

### Kafka

```text
Producer
   |
   v
Kafka Topic
   |
   +---- Partition 0
   +---- Partition 1
   +---- Partition 2
   +---- Partition 3
   +---- Partition 4
   +---- Partition 5
          |
          +--> Consumer Group A
          +--> Consumer Group B
          +--> Consumer Group C
```

Kafka is particularly useful when the event stream itself is an important durable data source.

---

# 5. Kafka Architecture

A Kafka environment consists of:

```text
Producer
    |
    v
Kafka Cluster
    |
    +--> Topics
          |
          +--> Partitions
                 |
                 +--> Replicas
    |
    v
Consumer Groups
    |
    +--> Consumers
```

In our AWS implementation:

```text
EKS
 |
 +--> order-service
 +--> payment-service
 +--> inventory-service
 +--> fraud-service
 +--> notification-service
 +--> reporting-service
 |
 v
Amazon MSK
```

---

# 6. Amazon MSK

For this project we use:

```text
Amazon MSK
```

Amazon MSK is AWS's managed service for Apache Kafka.

Instead of operating Kafka brokers ourselves, AWS manages significant portions of the broker infrastructure.

Our platform team remains responsible for:

* Kafka topics
* Partitions
* Retention
* Client configuration
* IAM
* Application integration
* Consumer groups
* Capacity planning
* Monitoring
* Alerting
* Replay procedures
* Event contracts
* Disaster recovery

---

# 7. Why We Use MSK Instead of Running Kafka Inside EKS

Self-managed Kafka on Kubernetes is possible.

For example:

```text
EKS
 |
 +--> Strimzi Operator
       |
       +--> Kafka Brokers
       +--> Kafka Controllers
       +--> Persistent Volumes
```

However, self-managed Kafka means our team must manage:

* Kafka upgrades
* Broker configuration
* Storage
* Controller quorum
* Broker failures
* Persistent volumes
* Network failures
* Capacity
* Recovery
* Kafka security
* Operational tooling

For our production reference architecture:

```text
Application workloads
        |
        v
       EKS
        |
        v
     Amazon MSK
```

This separates application platform operations from Kafka broker operations.

For organizations that require Kafka inside Kubernetes, Strimzi is a viable alternative.

---

# 8. Production Kafka Network Architecture

Our production architecture:

```text
AWS Region: ap-south-1

                    Internet
                       |
                       v
                 CloudFront/WAF
                       |
                       v
                      ALB
                       |
                       v
                     EKS
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
  Private AZ-A    Private AZ-B     Private AZ-C
       |               |                |
       +---------------+----------------+
                       |
                       v
                 Amazon MSK
                 Private Only
```

Kafka brokers are not exposed directly to the public internet.

Applications communicate with MSK over private networking.

---

# 9. Availability Zones

Production Kafka should be distributed across multiple Availability Zones.

Example:

```text
AZ-A
 |
 +--> MSK Broker 1

AZ-B
 |
 +--> MSK Broker 2

AZ-C
 |
 +--> MSK Broker 3
```

This provides broker-level fault tolerance for AZ failures.

A three-broker cluster is a starting point, not a universal production capacity recommendation.

Broker count and instance type must be sized according to:

* Throughput
* Storage
* Network traffic
* Partition count
* Replication traffic
* Consumer load
* Retention
* Recovery requirements

---

# 10. Kafka Brokers

A Kafka broker stores and serves partitions.

Example:

```text
Kafka Cluster

Broker 1
 ├── orders partition 0
 ├── payments partition 1
 └── inventory partition 2

Broker 2
 ├── orders partition 1
 ├── payments partition 2
 └── inventory partition 0

Broker 3
 ├── orders partition 2
 ├── payments partition 0
 └── inventory partition 1
```

The exact partition placement is managed by Kafka.

---

# 11. Kafka Topics

A topic is a logical stream of events.

Examples:

```text
ecommerce.orders.created.v1
ecommerce.orders.cancelled.v1
ecommerce.payments.completed.v1
ecommerce.payments.failed.v1
ecommerce.inventory.reserved.v1
ecommerce.inventory.released.v1
ecommerce.shipments.created.v1
ecommerce.shipments.updated.v1
```

Topic names should be consistent across the organization.

---

# 12. Topic Naming Convention

Recommended:

```text
<domain>.<entity>.<event>.v<version>
```

Example:

```text
ecommerce.orders.created.v1
```

Breakdown:

```text
ecommerce
   |
domain

orders
   |
entity

created
   |
event

v1
   |
schema/event contract version
```

---

# 13. Kafka Partitions

Partitions provide Kafka's parallelism.

Example:

```text
Topic:
ecommerce.orders.created.v1

Partitions:

Partition 0
Partition 1
Partition 2
Partition 3
Partition 4
Partition 5
```

Each partition contains an ordered sequence of records.

Example:

```text
Partition 0

Offset 100 -> Event A
Offset 101 -> Event B
Offset 102 -> Event C
Offset 103 -> Event D
```

Kafka guarantees ordering within a partition.

It does not provide global ordering across all partitions.

---

# 14. Partition Key

When publishing an event, the producer can provide a key.

For orders:

```text
Key = orderId
```

Example:

```text
Order 1001 -> Partition 2
Order 1002 -> Partition 5
Order 1003 -> Partition 1
Order 1001 -> Partition 2
```

Events with the same key are normally routed to the same partition as long as the partitioning scheme and partition count remain stable.

This allows us to preserve ordering for a specific order.

Example:

```text
OrderCreated
OrderPaid
OrderShipped
OrderDelivered
```

If all use:

```text
key = ORD-1001
```

they can remain ordered within the same partition.

However, do not assume Kafka gives ordering across multiple topics.

---

# 15. Choosing Partition Count

Partition count affects:

* Consumer parallelism
* Throughput
* Broker resources
* Rebalancing
* Storage
* Network traffic

Example:

```text
6 partitions

Consumer Group
|
+-- Consumer 1 -> P0
+-- Consumer 2 -> P1
+-- Consumer 3 -> P2
+-- Consumer 4 -> P3
+-- Consumer 5 -> P4
+-- Consumer 6 -> P5
```

If we have:

```text
6 partitions
10 consumers
```

we cannot get 10-way partition parallelism for that topic within that consumer group.

Some consumers will have no partition assigned.

Therefore:

```text
Consumer parallelism <= Partition count
```

for a single topic/consumer group under normal assignment.

Partition count should be planned before production traffic.

---

# 16. Kafka Replication

Kafka replicates partitions across brokers.

Example:

```text
Partition 0

Broker 1 -> Leader
Broker 2 -> Replica
Broker 3 -> Replica
```

Replication factor:

```text
3
```

means three copies of the partition exist across brokers.

A common production starting point is:

```text
replication.factor = 3
min.insync.replicas = 2
```

The exact values must be selected according to availability, throughput, storage and recovery requirements.

---

# 17. Leader and Followers

For each partition:

```text
Partition 0

Broker 1
  |
  +--> Leader

Broker 2
  |
  +--> Follower

Broker 3
  |
  +--> Follower
```

Producers normally write through the partition leader.

Consumers normally fetch records from the appropriate broker serving the partition.

If the leader fails, Kafka can elect another eligible replica.

---

# 18. ISR

ISR means:

```text
In-Sync Replicas
```

Example:

```text
Partition 0

Broker 1 -> Leader -> ISR
Broker 2 -> Follower -> ISR
Broker 3 -> Follower -> ISR
```

If Broker 3 falls significantly behind:

```text
Broker 1 -> Leader -> ISR
Broker 2 -> Follower -> ISR
Broker 3 -> Follower -> OUT
```

This matters because production durability depends on having enough healthy replicas.

---

# 19. Important Producer Configuration

For important business events:

```properties
acks=all
enable.idempotence=true
```

Conceptually:

```text
Producer
   |
   | acks=all
   v
Kafka Leader
   |
   +--> Replica
   +--> Replica
```

The producer waits for the required replication acknowledgment according to the cluster configuration.

Idempotent producer behavior helps prevent duplicate writes caused by producer retries under supported conditions.

It does not eliminate application-level duplicate business processing.

---

# 20. Kafka Terraform Structure

Add Kafka infrastructure to our existing repository.

```text
infrastructure/
|
└── terraform/
    |
    ├── modules/
    |   |
    |   ├── vpc/
    |   ├── eks/
    |   ├── rds/
    |   ├── redis/
    |   ├── ecr/
    |   ├── sqs/
    |   ├── sns/
    |   |
    |   └── msk/
    |
    └── environments/
        |
        ├── dev/
        ├── qa/
        └── prod/
```

---

# 21. MSK Terraform Module

Representative implementation:

```hcl
resource "aws_cloudwatch_log_group" "msk" {
  name              = "/aws/msk/${var.cluster_name}"
  retention_in_days = 30

  tags = var.tags
}

resource "aws_msk_cluster" "this" {
  cluster_name           = var.cluster_name
  kafka_version          = var.kafka_version
  number_of_broker_nodes = var.number_of_broker_nodes

  broker_node_group_info {
    instance_type   = var.instance_type
    client_subnets  = var.private_subnet_ids
    security_groups = [aws_security_group.msk.id]

    storage_info {
      ebs_storage_info {
        volume_size = var.broker_volume_gib
      }
    }
  }

  encryption_info {
    encryption_at_rest_kms_key_arn = var.kms_key_arn

    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }

  client_authentication {
    sasl {
      iam = true
    }

    unauthenticated = false
  }

  enhanced_monitoring = "PER_BROKER"

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk.name
      }
    }
  }

  tags = var.tags
}
```

The exact Kafka version, instance type, monitoring options and authentication capabilities must be validated against the MSK service capabilities available in the selected AWS region.

---

# 22. Terraform Variables

```hcl
variable "cluster_name" {
  type = string
}

variable "kafka_version" {
  type = string
}

variable "number_of_broker_nodes" {
  type    = number
  default = 3
}

variable "instance_type" {
  type = string
}

variable "broker_volume_gib" {
  type    = number
  default = 500
}

variable "vpc_id" {
  type = string
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "eks_application_security_group_id" {
  type = string
}

variable "kms_key_arn" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

---

# 23. MSK Security Group

Kafka should not accept traffic from arbitrary networks.

```hcl
resource "aws_security_group" "msk" {
  name        = "${var.cluster_name}-sg"
  description = "Security group for Amazon MSK"
  vpc_id      = var.vpc_id

  tags = var.tags
}
```

For IAM-authenticated TLS client connections, configure the appropriate MSK listener and source security group.

Example:

```hcl
resource "aws_vpc_security_group_ingress_rule" "msk_client" {
  security_group_id = aws_security_group.msk.id

  referenced_security_group_id =
    var.eks_application_security_group_id

  from_port   = 9098
  to_port     = 9098
  ip_protocol = "tcp"

  description = "EKS workloads to MSK IAM TLS"
}
```

The exact broker ports depend on the authentication mode and MSK configuration.

Do not blindly open:

```text
0.0.0.0/0
```

to Kafka.

---

# 24. MSK Outputs

```hcl
output "cluster_arn" {
  value = aws_msk_cluster.this.arn
}

output "bootstrap_brokers_sasl_iam" {
  value = aws_msk_cluster.this.bootstrap_brokers_sasl_iam
}

output "security_group_id" {
  value = aws_security_group.msk.id
}
```

Applications should receive the bootstrap broker endpoints through controlled configuration.

---

# 25. Production Terraform Workflow

```text
Developer
    |
    v
Terraform PR
    |
    v
terraform fmt
    |
    v
terraform validate
    |
    v
terraform plan
    |
    v
Security checks
    |
    v
Peer Review
    |
    v
Approval
    |
    v
Terraform Apply
```

Commands:

```bash
terraform init

terraform fmt -check

terraform validate

terraform plan -out=msk.tfplan
```

After approval:

```bash
terraform apply msk.tfplan
```

Do not blindly run:

```bash
terraform apply -auto-approve
```

against production infrastructure.

---

# 26. Getting MSK Bootstrap Brokers

AWS CLI:

```bash
aws kafka get-bootstrap-brokers \
  --cluster-arn "$MSK_CLUSTER_ARN" \
  --region ap-south-1
```

You may receive different bootstrap broker strings depending on the enabled authentication mechanism.

For IAM authentication, use the IAM bootstrap broker endpoint.

---

# 27. Kafka Topic Management

We should not allow applications to create production topics dynamically.

Instead:

```text
Git
 |
 v
Topic Definition
 |
 v
Review
 |
 v
Platform CI/CD
 |
 v
Kafka Administration
 |
 v
Topic
```

Example repository:

```text
platform/
|
└── kafka/
    |
    ├── topics/
    |   ├── orders.yaml
    |   ├── payments.yaml
    |   ├── inventory.yaml
    |   └── shipping.yaml
    |
    ├── schemas/
    |
    └── acl/
```

---

# 28. Topic Definition

Example:

```yaml
name: ecommerce.orders.created.v1

partitions: 6

replicationFactor: 3

config:
  min.insync.replicas: "2"
  retention.ms: "604800000"
  cleanup.policy: delete
```

This example specifies:

```text
Partitions: 6
Replication: 3
Minimum ISR for writes: 2
Retention: 7 days
Cleanup: delete
```

Retention must be selected based on business requirements, storage cost and replay requirements.

---

# 29. Create Kafka Topic

From an authorized Kafka administration environment:

```bash
kafka-topics.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --command-config client.properties \
  --create \
  --topic ecommerce.orders.created.v1 \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000
```

Verify:

```bash
kafka-topics.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --command-config client.properties \
  --describe \
  --topic ecommerce.orders.created.v1
```

Check:

```text
Leader
Replicas
ISR
Partition count
```

---

# 30. Disable Automatic Topic Creation

Production environments should normally prevent accidental topic creation.

Conceptually:

```properties
auto.create.topics.enable=false
```

Why?

Suppose an application has a typo:

```text
ecommerce.order.created.v1
```

instead of:

```text
ecommerce.orders.created.v1
```

If automatic topic creation is enabled, a new topic might appear.

That can create:

* Silent data loss
* Wrong routing
* Monitoring gaps
* Storage growth
* Operational confusion

Therefore topics should be explicitly managed.

---

# 31. Event Contract

Our Order Service publishes:

```json
{
  "eventId": "evt-9b8d21",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-09-28T09:00:00Z",
  "source": "order-service",
  "correlationId": "corr-78123",
  "payload": {
    "orderId": "ORD-100245",
    "customerId": "CUS-102",
    "amount": 1499.00,
    "currency": "INR",
    "status": "CREATED"
  }
}
```

Important fields:

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

# 32. Why eventId matters

Suppose:

```text
eventId = evt-123
```

is delivered twice.

```text
Delivery 1
     |
     v
Consumer
     |
     v
Process

Delivery 2
     |
     v
Consumer
     |
     v
Same event
```

The consumer must detect:

```text
evt-123 already processed
```

and avoid applying the business operation twice.

Kafka does not make our business operation automatically exactly-once.

---

# 33. Schema Evolution

For a production platform we need schema governance.

Possible technologies:

```text
Avro
Protobuf
JSON Schema
Schema Registry
```

Example evolution:

### Version 1

```json
{
  "orderId": "ORD-100",
  "amount": 1000
}
```

### Version 2

```json
{
  "orderId": "ORD-100",
  "amount": 1000,
  "currency": "INR"
}
```

Adding an optional field can be backward compatible depending on the schema rules.

Breaking changes require more careful coordination.

Do not simply change an existing event contract and assume all consumers will immediately understand it.

---

# 34. Producer Application

Our Order Service uses Spring Boot.

Add:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

---

# 35. Kafka Producer Configuration

Example:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}

    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: AWS_MSK_IAM
      sasl.jaas.config: >
        software.amazon.msk.auth.iam.IAMLoginModule required;
      sasl.client.callback.handler.class: >
        software.amazon.msk.auth.iam.IAMClientCallbackHandler

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

      acks: all

      properties:
        enable.idempotence: true
        delivery.timeout.ms: 120000
```

The AWS MSK IAM authentication library must also be included using a version compatible with the application and Kafka client.

---

# 36. Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderCreatedEvent>
            kafkaTemplate;

    @Value("${app.kafka.topics.order-created}")
    private String orderCreatedTopic;

    public CompletableFuture<SendResult<String, OrderCreatedEvent>>
            publish(OrderCreatedEvent event) {

        return kafkaTemplate.send(
                orderCreatedTopic,
                event.orderId(),
                event
        );
    }
}
```

Key:

```text
event.orderId()
```

This ensures that events for the same order normally go to the same partition.

---

# 37. The Dangerous Database + Kafka Pattern

Avoid this:

```text
BEGIN
 |
 +--> INSERT order
 |
COMMIT
 |
 +--> Publish Kafka event
 |
 X Application crashes
```

The order exists.

The event may not.

Now:

```text
Order DB = SUCCESS
Kafka     = FAILURE
```

This creates inconsistent state.

---

# 38. Transactional Outbox Pattern

Use:

```text
Order API
   |
   v
PostgreSQL Transaction
   |
   +--> orders
   |
   +--> outbox_events
   |
   v
COMMIT
   |
   v
Outbox Publisher
   |
   v
Kafka
```

Now the order and event intent are persisted together.

---

# 39. Outbox Table

Example:

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    event_version INTEGER NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,
    published_at TIMESTAMP NULL,
    status VARCHAR(30) NOT NULL
);
```

Example record:

```text
id            = evt-123
aggregate     = order
aggregate_id  = ORD-1001
event_type    = OrderCreated
status        = PENDING
```

---

# 40. Outbox Publisher

Conceptually:

```text
Every few seconds

SELECT pending events

        |
        v

Publish to Kafka

        |
        v

Kafka ACK

        |
        v

Mark event published
```

However, this introduces another important failure case.

Suppose:

```text
Kafka publish = SUCCESS
Application crashes
Before outbox status update
```

The same event may be published again.

Therefore:

```text
Outbox
+
Kafka
+
Consumer Idempotency
```

must work together.

---

# 41. Consumer Configuration

Inventory Service:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}

    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: AWS_MSK_IAM
      sasl.jaas.config: >
        software.amazon.msk.auth.iam.IAMLoginModule required;
      sasl.client.callback.handler.class: >
        software.amazon.msk.auth.iam.IAMClientCallbackHandler

    consumer:
      group-id: inventory-order-consumers
      enable-auto-commit: false
      auto-offset-reset: earliest

      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

---

# 42. Consumer Group

Inventory Service:

```text
Consumer Group:

inventory-order-consumers
```

Payment Service:

```text
payment-order-consumers
```

Fraud:

```text
fraud-order-consumers
```

Notification:

```text
notification-order-consumers
```

Reporting:

```text
reporting-order-consumers
```

Do not use the same consumer group when independent services each need to receive every event.

---

# 43. Consumer Example

```java
@Component
@RequiredArgsConstructor
public class OrderCreatedConsumer {

    private final InventoryService inventoryService;

    @KafkaListener(
        topics = "${app.kafka.topics.order-created}",
        groupId = "inventory-order-consumers"
    )
    public void consume(
        ConsumerRecord<String, OrderCreatedEvent> record,
        Acknowledgment acknowledgment
    ) {

        inventoryService.processOrderCreated(
            record.value()
        );

        acknowledgment.acknowledge();
    }
}
```

The application should acknowledge or commit progress only after successful business processing according to the chosen listener and transaction model.

---

# 44. Consumer Idempotency

Example database table:

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(200) NOT NULL,
    processed_at TIMESTAMP NOT NULL
);
```

Processing:

```text
Receive event
      |
      v
Check eventId
      |
      +---- Already exists ---> Skip business operation
      |
      +---- Does not exist
                    |
                    v
             Business operation
                    |
                    v
             Record eventId
                    |
                    v
             Commit transaction
```

The business operation and idempotency record should be coordinated transactionally where possible.

---

# 45. Kafka Offsets

Kafka stores consumer offsets.

Example:

```text
Partition 0

Offset:
100
101
102
103
104
```

Consumer committed:

```text
103
```

The consumer's position represents its progress through the partition.

If the consumer crashes, Kafka can reassign the partition to another consumer.

Depending on exactly where the offset was committed relative to business processing, records may be processed again.

Therefore:

```text
Kafka offset management
+
Business idempotency
```

must be designed together.

---

# 46. Consumer Group Rebalancing

Suppose:

```text
Partitions = 6

Consumers:
C1
C2
C3
```

Assignment:

```text
C1 -> P0 P3
C2 -> P1 P4
C3 -> P2 P5
```

Now C2 crashes.

Kafka rebalances:

```text
C1 -> P0 P3 P1
C3 -> P2 P5 P4
```

Rebalancing creates overhead.

Frequent rebalances can reduce throughput and increase processing latency.

Monitor:

```text
Consumer restarts
Consumer group membership
Rebalance frequency
Processing latency
Poll interval behavior
```

---

# 47. Consumer Lag

Lag:

```text
Latest Kafka offset
        -
Consumer committed offset
```

Example:

```text
Latest offset = 10,000
Consumer offset = 9,000

Lag = 1,000
```

Lag indicates unprocessed or uncommitted records.

But lag alone is not enough.

We also need:

```text
Lag
+
Lag growth rate
+
Event processing latency
+
Consumer throughput
+
Downstream health
```

---

# 48. Kafka Consumer Group Commands

Check group:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --command-config client.properties \
  --describe \
  --group inventory-order-consumers
```

Example output:

```text
GROUP
inventory-order-consumers

TOPIC
ecommerce.orders.created.v1

PARTITION
0

CURRENT-OFFSET
9000

LOG-END-OFFSET
10000

LAG
1000
```

---

# 49. KEDA Kafka Autoscaling

Kafka consumers can scale based on lag.

Architecture:

```text
Kafka
 |
 | Consumer Lag
 v
KEDA
 |
 v
Deployment replicas
 |
 v
EKS
 |
 v
Karpenter
 |
 v
EC2 Nodes
```

Example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: inventory-kafka-scaler
  namespace: ecommerce

spec:
  scaleTargetRef:
    name: inventory-service

  minReplicaCount: 2
  maxReplicaCount: 6

  pollingInterval: 30
  cooldownPeriod: 300

  triggers:
    - type: kafka
      metadata:
        bootstrapServers: REPLACE_WITH_MSK_BOOTSTRAP
        consumerGroup: inventory-order-consumers
        topic: ecommerce.orders.created.v1
        lagThreshold: "100"
```

The exact authentication configuration must be added for the installed KEDA version and MSK authentication method.

---

# 50. Kafka Scaling Limitation

Suppose:

```text
Partitions = 6
```

and:

```text
Consumer replicas = 20
```

Only approximately six consumers can actively own partitions for that topic within the consumer group at one time.

Therefore:

```text
Partition count
```

is an important scalability decision.

Do not blindly increase:

```text
maxReplicaCount
```

without understanding partition count.

---

# 51. Kafka + Database Scaling Problem

Suppose:

```text
Inventory Pods = 2
Hikari max pool = 20
```

Potential connections:

```text
2 × 20 = 40
```

Now KEDA scales:

```text
2 -> 6 pods
```

Potential connections:

```text
6 × 20 = 120
```

If PostgreSQL cannot support that load:

```text
Kafka lag
    |
    v
KEDA scaling
    |
    v
More pods
    |
    v
More DB connections
    |
    v
Database saturation
    |
    v
Processing slows further
```

This is a classic scaling failure.

Kafka scaling must be coordinated with:

* PostgreSQL
* Redis
* External APIs
* CPU
* Memory
* Network
* Connection pools

---

# 52. Retry Architecture

Kafka retry behavior must be explicitly designed.

Example:

```text
Main Topic

ecommerce.orders.created.v1
          |
          v
Inventory Consumer
          |
       Failure
          |
          v
Retry Topic
          |
          v
Inventory Consumer
          |
       Failure
          |
          v
DLQ Topic
```

Example topics:

```text
ecommerce.orders.created.v1

ecommerce.orders.created.v1.retry.1

ecommerce.orders.created.v1.retry.2

ecommerce.orders.created.v1.dlq
```

---

# 53. Retry Categories

## Transient failure

Example:

```text
PostgreSQL timeout
```

Retry.

---

## Permanent failure

Example:

```text
Invalid event schema
```

Do not retry indefinitely.

Send to DLQ.

---

## Business rejection

Example:

```text
Insufficient inventory
```

This may be a valid business outcome rather than a technical failure.

The service might publish:

```text
InventoryReservationFailed
```

instead of sending the event to a technical DLQ.

---

# 54. Retry Backoff

Avoid:

```text
Retry immediately
Retry immediately
Retry immediately
Retry immediately
```

This can create a retry storm.

Use:

```text
1 second
2 seconds
4 seconds
8 seconds
16 seconds
```

with jitter.

Example:

```text
Failure
 |
 +--> Retry 1
 |
 +--> Retry 2
 |
 +--> Retry 3
 |
 +--> DLQ
```

The exact retry policy depends on the downstream dependency and business SLA.

---

# 55. Dead-Letter Topic

Example:

```text
ecommerce.orders.created.v1.dlq
```

DLQ message should preserve:

```text
eventId
originalTopic
originalPartition
originalOffset
eventType
failureReason
consumerGroup
firstFailureTime
lastFailureTime
retryCount
originalPayload
correlationId
```

This makes investigation much easier.

---

# 56. DLQ Is Not a Trash Bin

Bad design:

```text
Application
    |
    v
DLQ
    |
    v
Nobody checks it
```

Production design:

```text
DLQ
 |
 v
Alert
 |
 v
Investigation
 |
 +--> Fix application
 |
 +--> Fix data
 |
 +--> Replay
 |
 +--> Permanently reject
```

---

# 57. Controlled Replay

Never blindly replay a production DLQ.

Process:

```text
DLQ
 |
 v
Identify failed event
 |
 v
Determine root cause
 |
 v
Fix problem
 |
 v
Validate in lower environment
 |
 v
Approve replay
 |
 v
Replay selected events
 |
 v
Monitor
```

Before replaying payment or inventory events, verify idempotency.

---

# 58. Kafka Authentication

Production authentication options include:

```text
IAM
TLS certificates
SASL mechanisms
```

Our AWS reference implementation uses:

```text
MSK + IAM authentication + TLS
```

Architecture:

```text
EKS Pod
   |
   v
EKS Pod Identity
   |
   v
IAM Role
   |
   v
MSK IAM Authentication
   |
   v
Kafka Topic Authorization
```

No AWS access keys should be stored in:

```text
Dockerfile
GitHub
Helm values
Kubernetes Secret
application.yaml
```

---

# 59. EKS Pod Identity

ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: inventory-service
  namespace: ecommerce
```

Terraform:

```hcl
resource "aws_eks_pod_identity_association" "inventory" {
  cluster_name    = module.eks.cluster_name
  namespace       = "ecommerce"
  service_account = "inventory-service"

  role_arn = aws_iam_role.inventory_kafka.arn
}
```

Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: inventory-service
```

The IAM role should contain only the Kafka permissions required by Inventory Service.

---

# 60. Producer IAM

Order Service needs permissions appropriate to publishing.

Conceptually:

```json
{
  "Effect": "Allow",
  "Action": [
    "kafka-cluster:Connect",
    "kafka-cluster:DescribeCluster",
    "kafka-cluster:DescribeTopic",
    "kafka-cluster:WriteData"
  ],
  "Resource": [
    "MSK_CLUSTER_ARN",
    "ORDER_TOPIC_ARN"
  ]
}
```

The exact resource ARNs and permissions must match the MSK IAM authorization model being used.

---

# 61. Consumer IAM

Inventory Service requires:

```text
Connect
Describe cluster
Describe topic
Read data
Consumer group permissions
```

Example conceptual policy:

```json
{
  "Effect": "Allow",
  "Action": [
    "kafka-cluster:Connect",
    "kafka-cluster:DescribeCluster",
    "kafka-cluster:DescribeTopic",
    "kafka-cluster:ReadData",
    "kafka-cluster:DescribeGroup",
    "kafka-cluster:AlterGroup"
  ],
  "Resource": [
    "MSK_CLUSTER_ARN",
    "ORDER_TOPIC_ARN",
    "INVENTORY_GROUP_ARN"
  ]
}
```

Use least privilege.

---

# 62. Spring Boot Graceful Shutdown

Consumer applications should handle Kubernetes termination correctly.

Lifecycle:

```text
SIGTERM
   |
   v
Stop receiving new work
   |
   v
Finish safe in-flight processing
   |
   v
Commit completed offsets
   |
   v
Leave consumer group
   |
   v
Pod terminates
```

Kubernetes:

```yaml
terminationGracePeriodSeconds: 90
```

This value must be based on actual processing duration.

Do not assume a 90-second grace period automatically makes Kafka consumers safe.

---

# 63. Kafka Monitoring

Our monitoring architecture:

```text
Kafka / MSK
     |
     +--> CloudWatch
     |
     +--> Prometheus-compatible metrics
     |
     v
Prometheus
     |
     v
Grafana
     |
     v
Alertmanager
```

Application metrics:

```text
Spring Boot
    |
    v
Micrometer
    |
    v
Prometheus
```

---

# 64. Kafka Dashboard

Create Grafana dashboards for:

### Broker

```text
Broker CPU
Broker memory
Broker storage
Network
Request latency
```

### Replication

```text
Under-replicated partitions
Offline partitions
ISR changes
Leader elections
```

### Producer

```text
Records produced
Producer errors
Retries
Latency
Request rate
```

### Consumer

```text
Consumer lag
Records consumed
Processing latency
Consumer errors
Rebalances
```

---

# 65. Most Important Kafka Alert

One of the most useful application-level alerts is sustained consumer lag.

But do not use:

```text
lag > 1000
```

universally.

Instead define thresholds based on:

```text
Service
Traffic
SLA
Partition count
Processing time
Business criticality
```

Example:

```text
Payment consumer

WARNING:
Lag increasing continuously for 5 minutes

CRITICAL:
Estimated processing delay exceeds payment SLA
```

---

# 66. Under-Replicated Partitions

Alert:

```text
UnderReplicatedPartitions > 0
```

for a sustained period.

Potential causes:

```text
Broker failure
Network problem
Disk pressure
Broker overload
Replication lag
```

Investigation:

```text
MSK metrics
Kafka partition state
Broker health
Storage
Network
AWS events
```

---

# 67. Offline Partitions

An offline partition means the partition is unavailable.

This is more severe than ordinary consumer lag.

Example:

```text
Producer
   |
   X
Partition unavailable
```

Investigate immediately.

---

# 68. Kafka Storage

Kafka storage planning depends on:

```text
Incoming throughput
Retention
Replication factor
Compression
Consumer traffic
Peak traffic
Recovery requirements
```

Approximate raw storage concept:

```text
Daily data
×
Retention days
×
Replication factor
```

For example, if logical event data is:

```text
100 GB/day
```

retention:

```text
7 days
```

replication:

```text
3
```

raw replicated storage is approximately:

```text
100 × 7 × 3
=
2100 GB
```

Actual storage consumption differs because of compression, Kafka overhead, segment behavior and other factors.

---

# 69. Kafka Compression

Producers can use compression:

```properties
compression.type=zstd
```

or another supported compression codec.

Compression can reduce:

```text
Network usage
Storage
Broker I/O
```

but requires CPU.

Benchmark using realistic event sizes and workloads before selecting the production configuration.

---

# 70. Message Size

Do not treat Kafka as a file-transfer system.

Keep events relatively small.

Bad:

```text
OrderCreated
+
10 MB product image
+
large PDF
+
huge customer profile
```

Better:

```text
Kafka event
   |
   +--> orderId
   +--> metadata
   +--> S3 object reference
```

Large binary objects should generally live in object storage rather than inside Kafka records.

---

# 71. Kafka and S3

Example:

```text
Order Service
     |
     +--> Generate invoice
     |
     +--> Store PDF in S3
     |
     +--> Publish event
             |
             v
      invoice.created
             |
             v
       Notification
```

Kafka event:

```json
{
  "eventType": "InvoiceCreated",
  "invoiceId": "INV-1001",
  "orderId": "ORD-1001",
  "s3ObjectKey": "invoices/2026/09/INV-1001.pdf"
}
```

The consumer retrieves the object from S3 using its IAM permissions.

---

# 72. Kafka and Redis

Kafka should not replace Redis.

Redis:

```text
Fast state/cache
```

Kafka:

```text
Durable event stream
```

Example:

```text
Product Service
    |
    +--> PostgreSQL
    |
    +--> Redis cache
    |
    +--> Kafka ProductUpdated
```

Search Service receives:

```text
ProductUpdated
```

and updates its search index.

---

# 73. Kafka and PostgreSQL

Recommended architecture:

```text
PostgreSQL
    |
    +--> Business state
    |
    +--> Outbox
             |
             v
           Kafka
             |
      +------+------+------+
      |      |      |      |
      v      v      v      v
   Search  Fraud  Report  Notification
```

PostgreSQL remains the source of truth for the service's transactional state.

Kafka carries events representing state changes.

---

# 74. Kafka Failure Scenario

Suppose Kafka becomes temporarily unavailable.

Order Service:

```text
Order API
   |
   v
PostgreSQL
   |
   +--> Order saved
   |
   +--> Outbox saved
```

Kafka:

```text
UNAVAILABLE
```

The order is still committed.

Outbox remains:

```text
PENDING
```

When Kafka recovers:

```text
Outbox Publisher
       |
       v
Kafka
       |
       v
Event published
```

This is one major reason for using the transactional outbox.

---

# 75. Consumer Failure Scenario

Inventory Service crashes.

```text
Kafka
 |
 v
Consumer
 |
 X Crash
```

Kafka retains the records.

Another consumer instance can take ownership after the group rebalance.

Potentially:

```text
Event processed
Offset not committed
      |
      v
Event delivered again
```

Therefore:

```text
Idempotency required
```

---

# 76. PostgreSQL Failure Scenario

Inventory consumer receives:

```text
OrderCreated
```

PostgreSQL is unavailable.

Bad behavior:

```text
Kafka
 |
 v
Consumer
 |
 v
DB failure
 |
 v
Immediate retry
 |
 v
DB failure
 |
 v
Immediate retry
```

This creates a retry storm.

Better:

```text
Kafka
 |
 v
Consumer
 |
 v
DB failure
 |
 v
Backoff
 |
 v
Circuit breaker
 |
 v
Controlled retry
```

Kafka lag may temporarily increase.

That is acceptable if the system is designed to recover safely.

---

# 77. Poison Message

Example:

```json
{
  "eventType": "OrderCreated",
  "orderId": null
}
```

Consumer repeatedly fails.

Without bounded retry:

```text
same message
same failure
same partition
same consumer
```

Potentially the consumer makes no progress.

Therefore:

```text
Validate
 |
 +--> Invalid
       |
       v
DLQ
```

and continue according to the ordering requirements of the workflow.

---

# 78. Ordering vs Retry Trade-Off

Suppose:

```text
Partition 0

Offset 100 -> OrderCreated
Offset 101 -> OrderPaid
Offset 102 -> OrderShipped
```

If Offset 100 fails and we immediately skip it:

```text
Offset 101
Offset 102
```

may be processed before:

```text
Offset 100
```

This can violate business ordering.

Therefore the retry architecture must explicitly decide whether:

```text
Strict partition ordering
```

or:

```text
Maximum throughput
```

has priority for each event stream.

---

# 79. Kafka Connect

Kafka Connect allows integration with external systems.

Examples:

```text
Kafka -> Elasticsearch
Kafka -> S3
Kafka -> Database
Database -> Kafka
```

Architecture:

```text
PostgreSQL
    |
    v
Debezium
    |
    v
Kafka Connect
    |
    v
Amazon MSK
```

or:

```text
Amazon MSK
    |
    v
Kafka Connect
    |
    v
Elasticsearch
```

Use connectors carefully.

A connector is another production component requiring:

* Monitoring
* Capacity
* Security
* Configuration management
* Failure recovery
* Version management

---

# 80. CDC with Debezium

Example:

```text
Order DB
    |
    v
Outbox Table
    |
    v
Debezium
    |
    v
Kafka
```

This can remove the need for an application polling publisher in some architectures.

However, CDC introduces additional operational considerations:

```text
Replication slots
WAL retention
Connector offsets
Schema history
Connector failures
Database impact
```

Therefore we should not introduce CDC simply because it is popular.

Use it when its operational and throughput characteristics fit the platform.

---

# 81. Kafka Disaster Recovery

Kafka replication within one region does not automatically provide regional disaster recovery.

Our DR architecture may look like:

```text
Region A
ap-south-1
 |
 +--> MSK Primary
 |
 +--> EKS Primary

        |
        | Replication
        v

Region B
ap-south-2
 |
 +--> MSK DR
 |
 +--> EKS DR
```

Possible technologies and approaches should be selected based on the current AWS/MSK capabilities and the organization's RPO/RTO.

Cross-region event replication must address:

```text
Topics
Schemas
Consumer groups
Offsets
IAM
Network
Applications
Databases
Secrets
```

---

# 82. Kafka Backup vs Retention

Important distinction:

```text
Retention != Backup
```

Kafka retention controls how long records remain available in the cluster.

A disaster recovery strategy may require:

```text
Cross-region replication
External archival
Infrastructure recreation
Topic configuration restoration
Schema restoration
Consumer recovery
```

Do not rely solely on:

```text
retention.ms
```

for disaster recovery.

---

# 83. Production Kafka Incident Example

Incident:

```text
Payment consumer lag increasing
```

Timeline:

```text
10:00
Normal

10:05
Payment traffic increases

10:08
Consumer lag begins increasing

10:10
KEDA scales consumers

10:12
Payment DB connection count increases

10:15
Database latency increases

10:17
Consumer processing slows

10:20
Lag reaches critical threshold
```

Wrong response:

```text
Increase replicas again
```

Potential result:

```text
More consumers
      |
      v
More DB connections
      |
      v
More DB contention
      |
      v
Higher lag
```

Correct investigation:

```text
Kafka lag
   |
   v
Consumer processing time
   |
   v
Database metrics
   |
   v
Connection pool
   |
   v
Slow queries
```

---

# 84. Kafka Troubleshooting Workflow

When consumer lag increases:

```text
1. Confirm lag
        |
2. Determine whether lag is increasing
        |
3. Check consumer pod health
        |
4. Check consumer errors
        |
5. Check consumer processing latency
        |
6. Check rebalances
        |
7. Check Kafka broker health
        |
8. Check database/downstream dependencies
        |
9. Check KEDA scaling
        |
10. Check node capacity
        |
11. Apply controlled remediation
        |
12. Monitor recovery
```

Do not immediately restart every consumer.

---

# 85. Useful Kubernetes Commands

Check deployment:

```bash
kubectl get deployment inventory-service \
  -n ecommerce
```

Check pods:

```bash
kubectl get pods \
  -n ecommerce \
  -l app=inventory-service
```

Check logs:

```bash
kubectl logs \
  deployment/inventory-service \
  -n ecommerce \
  --since=15m
```

Check previous container:

```bash
kubectl logs \
  deployment/inventory-service \
  -n ecommerce \
  --previous
```

Check events:

```bash
kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

---

# 86. KEDA Troubleshooting

```bash
kubectl get scaledobject \
  -n ecommerce
```

```bash
kubectl describe scaledobject \
  inventory-kafka-scaler \
  -n ecommerce
```

Check HPA:

```bash
kubectl get hpa \
  -n ecommerce
```

Check KEDA:

```bash
kubectl get pods \
  -n keda
```

If Kafka lag increases but replicas remain unchanged, investigate:

```text
KEDA scaler
Kafka authentication
Kafka connectivity
Consumer group name
Topic name
Trigger configuration
KEDA operator logs
```

---

# 87. Kafka Failure Testing

Test in a non-production environment first.

## Test 1 — Consumer pod failure

```text
Delete one consumer pod
```

Expected:

```text
Pod terminates
Consumer group rebalances
Partition ownership changes
Processing continues
```

Validate:

```text
No permanent event loss
No uncontrolled duplicates
Lag recovers
```

---

# 88. Test 2 — Database Failure

Temporarily make the consumer's database unavailable.

Expected:

```text
Consumer errors
Retry/backoff
Lag increases
Database recovery
Processing resumes
Lag decreases
```

Validate:

```text
No retry storm
No DB connection exhaustion
No permanent event loss
```

---

# 89. Test 3 — Duplicate Event

Publish the same event twice:

```text
eventId = evt-123
```

Expected:

```text
First delivery
    |
    v
Business operation

Second delivery
    |
    v
Duplicate detected
    |
    v
Business operation skipped
```

Validate database state.

---

# 90. Test 4 — Poison Message

Publish an invalid event.

Expected:

```text
Validation failure
       |
       v
Bounded retry
       |
       v
DLQ
```

Verify:

```text
DLQ record exists
Original metadata preserved
Alert generated
Consumer continues where appropriate
```

---

# 91. Test 5 — Traffic Spike

Generate a controlled increase in events.

Observe:

```text
Kafka throughput
Consumer lag
KEDA
Pod count
Karpenter
Node count
CPU
Memory
Database connections
Database latency
```

This is especially important before our later chapters on:

```text
Low traffic
Medium traffic
High traffic
Flash sale
```

---

# 92. Production Kafka Security Checklist

```text
[ ] Kafka brokers private
[ ] TLS enabled
[ ] IAM authentication enabled where appropriate
[ ] Least-privilege IAM roles
[ ] No static AWS keys
[ ] Topic access restricted
[ ] Consumer groups restricted
[ ] Security groups restricted
[ ] KMS encryption configured
[ ] Topic creation controlled
[ ] Administrative access restricted
[ ] Kafka UI not publicly exposed
[ ] Secrets not committed to Git
[ ] Audit logging enabled where applicable
```

---

# 93. Production Kafka Reliability Checklist

```text
[ ] Multi-AZ brokers
[ ] Replication factor defined
[ ] min.insync.replicas defined
[ ] Producer acks configured
[ ] Idempotent producer enabled
[ ] Consumer idempotency implemented
[ ] Outbox implemented for DB + event consistency
[ ] Retry policy defined
[ ] DLQ defined
[ ] Replay procedure documented
[ ] Consumer lag monitored
[ ] Under-replication monitored
[ ] Offline partitions monitored
[ ] Storage monitored
[ ] Broker capacity planned
[ ] Partition count planned
[ ] DR strategy documented
[ ] Recovery tested
```

---

# 94. Production Kafka Architecture

Our final reference architecture:

```text
                         Developer
                             |
                             v
                       GitHub PR
                             |
                             v
                  CI + Security Scans
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
       +---------------------+----------------------+
       |                     |                      |
       v                     v                      v
 Order Service         Payment Service       Inventory Service
       |                     |                      |
       |                     |                      |
       +---------------------+----------------------+
                             |
                             v
                       PostgreSQL
                             |
                       Transaction
                             |
                  +----------+----------+
                  |                     |
                  v                     v
               Business             Outbox
                State                  |
                                       v
                                  Outbox Publisher
                                       |
                                       v
                                  Amazon MSK
                                       |
             +-------------------------+--------------------------+
             |            |             |             |           |
             v            v             v             v           v
         Inventory     Payment        Fraud      Notification  Reporting
         Consumer      Consumer      Consumer      Consumer     Consumer
             |            |             |             |           |
             v            v             v             v           v
          Database     Database      Database      External     Analytics
                                                     APIs
```

---

# 95. Kafka Operational Model

Our production operating model is:

```text
Terraform
    |
    +--> MSK infrastructure

GitOps
    |
    +--> Application configuration

Helm
    |
    +--> Consumer deployment

IAM
    |
    +--> Authentication / authorization

Prometheus
    |
    +--> Metrics

Grafana
    |
    +--> Dashboards

Alertmanager
    |
    +--> Alerts

ArgoCD
    |
    +--> Deployment reconciliation
```

Kafka itself is not manually configured from random developer laptops.

Production changes should be:

```text
Version controlled
Reviewed
Audited
Automated where practical
Reversible where possible
```

---

# 96. Complete Kafka Event Flow

Example: customer places an order.

```text
Customer
   |
   v
ALB
   |
   v
Order Service
   |
   +-----------------------------+
   |                             |
   v                             v
PostgreSQL                    Outbox
   |                             |
   |                             v
   +------------------------> Publisher
                                  |
                                  v
                              Amazon MSK
                                  |
                     +------------+------------+
                     |            |            |
                     v            v            v
                 Inventory     Payment       Fraud
                     |            |            |
                     v            v            v
                 Inventory     Payment       Fraud
                   DB            DB            DB
                     |
                     v
              InventoryReserved
                     |
                     v
                   Kafka
                     |
                     v
                 Order Service
                     |
                     v
                 Shipping
                     |
                     v
                 Delivery
                     |
                     v
               Notification
```

This becomes the foundation for the next chapter.

---

# 97. What We Have Implemented

By the end of this chapter we understand:

```text
Kafka architecture
Amazon MSK
Kafka brokers
Topics
Partitions
Replication
Leaders
Followers
ISR
Producer acknowledgements
Idempotent producers
Consumer groups
Offsets
Consumer lag
KEDA scaling
IAM authentication
EKS Pod Identity
Kafka security
Retry topics
DLQ topics
Event schemas
Transactional outbox
Consumer idempotency
Kafka Connect
CDC
Monitoring
Troubleshooting
Failure testing
Disaster recovery concepts
```

---

# 98. Important Production Principles

## Principle 1

Kafka is not just another queue.

```text
Kafka = durable distributed event stream
```

---

## Principle 2

Kafka does not eliminate duplicate business processing.

```text
Kafka
+
Idempotent consumers
=
Reliable business processing
```

---

## Principle 3

Kafka replication is not disaster recovery.

```text
Replication
!=
Backup
!=
Cross-region DR
```

---

## Principle 4

More consumers do not always mean more throughput.

```text
Partitions
+
Downstream capacity
+
Database capacity
+
Network
```

all limit scaling.

---

## Principle 5

The transactional outbox solves an important consistency problem:

```text
Database transaction
        +
Event publication intent
```

---

## Principle 6

Consumer lag is a symptom, not necessarily the root cause.

Possible root causes:

```text
Database
CPU
Memory
Network
External API
Consumer bugs
Kafka broker pressure
Partition imbalance
Rebalances
Insufficient partitions
```

---

# 99. Final Production Checklist

```text
Infrastructure
[ ] MSK provisioned
[ ] Private subnets
[ ] Multi-AZ
[ ] Encryption at rest
[ ] TLS in transit
[ ] IAM authentication
[ ] KMS configured
[ ] CloudWatch logging
[ ] Monitoring enabled

Kafka
[ ] Topics defined
[ ] Partition count planned
[ ] Replication configured
[ ] min.insync.replicas configured
[ ] Retention defined
[ ] Topic creation controlled

Applications
[ ] Producer configured
[ ] Consumer configured
[ ] Consumer groups defined
[ ] Idempotency implemented
[ ] Event schemas versioned
[ ] Correlation IDs implemented

Reliability
[ ] Transactional outbox
[ ] Retry strategy
[ ] DLQ
[ ] Replay procedure
[ ] Failure testing

Scaling
[ ] KEDA
[ ] Partition-aware scaling
[ ] Database connection budget
[ ] Karpenter integration
[ ] Downstream capacity considered

Observability
[ ] Consumer lag
[ ] Broker metrics
[ ] Under-replicated partitions
[ ] Offline partitions
[ ] Producer errors
[ ] Consumer errors
[ ] Processing latency
[ ] Grafana dashboards
[ ] Alertmanager alerts

Security
[ ] Least privilege
[ ] Pod Identity
[ ] No static AWS credentials
[ ] Private Kafka
[ ] Restricted security groups
[ ] Restricted administrative access

DR
[ ] RPO defined
[ ] RTO defined
[ ] Cross-region strategy
[ ] Replay strategy
[ ] Recovery tested
```

---

# 100. Chapter Summary

The production architecture we have built is:

```text
                    EKS
                     |
         +-----------+-----------+
         |           |           |
       Order      Payment    Inventory
         |           |           |
         +-----------+-----------+
                     |
                     v
                 PostgreSQL
                     |
                     v
                   Outbox
                     |
                     v
              Amazon MSK
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
   Inventory      Payment        Fraud
   Consumer       Consumer      Consumer
       |             |             |
       v             v             v
      DB            DB            DB

                     |
                     v
            Prometheus/Grafana
                     |
                     v
                Alertmanager
```

The most important production chain is:

```text
Database Transaction
        |
        v
Transactional Outbox
        |
        v
Kafka
        |
        v
Consumer Group
        |
        v
Idempotent Business Processing
        |
        v
Database Transaction
```

This pattern becomes extremely important when we build the complete event-driven architecture in Chapter 117.

---

# Next Chapter

## 117-Event-Driven-Microservice-Architecture.md

The next chapter will combine everything we have implemented:

```text
Order
  |
  +--> PostgreSQL
  +--> Outbox
  |
  v
Kafka
  |
  +--> Inventory
  +--> Payment
  +--> Fraud
  +--> Notification
  +--> Reporting
  |
  v
Saga / Business Workflow
  |
  +--> Success path
  |
  +--> Payment failure
  |
  +--> Inventory failure
  |
  +--> Shipping failure
  |
  +--> Compensation
  |
  +--> Retry
  |
  +--> DLQ
  |
  +--> Recovery
```

It will show **exactly how the 21 microservices communicate in a real production e-commerce workflow**, including where we use **Kafka vs SQS vs SNS vs RabbitMQ**, transactional outbox, Saga orchestration/choreography, compensating transactions, idempotency, correlation IDs, failure scenarios, and end-to-end recovery.

```
```
