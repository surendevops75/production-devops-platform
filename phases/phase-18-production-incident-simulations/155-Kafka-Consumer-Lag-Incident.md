# 155 — Kafka Consumer Lag Production Incident

> **Phase 18 — Incident Management**  
> **Incident #155**  
> **Primary services:** `inventory-service`, `payment-service`, `reporting-service`  
> **Environment:** Production  
> **Messaging platform:** Amazon MSK / Kafka-compatible production cluster

---

# 1. Objective

This chapter explains how to troubleshoot a **Kafka consumer lag incident** in a production EKS environment.

The goal is not simply:

```text
Kafka lag is high
        ↓
Add consumers
```

Instead, determine:

```text
Why is lag increasing?
        ↓
Are consumers running?
        ↓
Are consumers polling Kafka?
        ↓
Are records processing successfully?
        ↓
Is the consumer group balanced?
        ↓
Are partitions sufficient?
        ↓
Is the application slow?
        ↓
Is a downstream dependency slow?
        ↓
Can consumers safely scale?
```

A Kafka lag incident can originate from:

- application failure
- consumer rebalance
- insufficient partitions
- insufficient consumer instances
- slow message processing
- database bottlenecks
- external API throttling
- CPU/memory saturation
- serialization problems
- poison records
- incorrect consumer configuration
- Kafka broker problems

---

# 2. Production Kafka Architecture

Our ecommerce event architecture:

```text
                     ┌──────────────────────┐
                     │    order-service     │
                     └──────────┬───────────┘
                                │
                                │ Publish
                                ▼
                     ┌──────────────────────┐
                     │    Kafka / MSK       │
                     │                      │
                     │ order-events topic   │
                     └──────────┬───────────┘
                                │
                     ┌──────────┼──────────┐
                     │          │          │
                     ▼          ▼          ▼
               inventory     payment   reporting
               consumer      consumer   consumer
                     │          │          │
                     ▼          ▼          ▼
                    RDS        RDS        S3
                   Redis     Provider     /RDS
```

Each service can have its own consumer group.

For example:

```text
order-events
     │
     ├── inventory-service-group
     ├── payment-service-group
     ├── notification-service-group
     └── reporting-service-group
```

This means one slow consumer group does not automatically stop the others.

---

# 3. What Is Kafka Consumer Lag?

Consumer lag represents how far behind a consumer group is compared with the latest available records.

Conceptually:

```text
Lag = Latest Offset - Consumer Committed Offset
```

Example:

```text
Partition 0:

Latest offset       = 1,500,000
Consumer offset     = 1,490,000

Lag                 = 10,000
```

If lag continuously increases:

```text
Producer rate > Consumer processing rate
```

The important word is **continuously**.

A temporary lag spike during a deployment may be normal.

A steadily increasing lag with growing message age is an incident.

---

# 4. Kafka Partition Model

Suppose:

```text
Topic: order-events

Partitions:
0
1
2
3
4
5
```

A consumer group:

```text
inventory-service-group
```

has:

```text
6 consumer instances
```

Kafka can distribute approximately:

```text
Consumer 1 → Partition 0
Consumer 2 → Partition 1
Consumer 3 → Partition 2
Consumer 4 → Partition 3
Consumer 5 → Partition 4
Consumer 6 → Partition 5
```

Now suppose we deploy:

```text
12 consumers
```

but only have:

```text
6 partitions
```

The additional consumers cannot all actively consume unique partitions simultaneously.

Therefore:

> **Kafka consumer scaling is constrained by partition count.**

---

# 5. Important Kafka Metrics

Monitor at least:

```text
Consumer lag
Records consumed/sec
Records produced/sec
Consumer processing latency
Consumer error rate
Consumer rebalance rate
Consumer group members
Partition distribution
Broker health
Request latency
Under-replicated partitions
ISR changes
```

For the application:

```text
CPU
Memory
GC
Thread pools
DB connections
DB latency
Redis latency
External API latency
```

---

# 6. Lag vs Throughput

Suppose:

```text
Producer:
1,000 records/sec

Consumer:
1,000 records/sec
```

Lag should remain approximately stable.

Now:

```text
Producer:
1,500 records/sec

Consumer:
1,000 records/sec
```

Lag grows:

```text
500 records/sec
```

If this continues for 10 minutes:

```text
500 × 600
= 300,000 records
```

The backlog becomes significant.

---

# 7. First Question During an Incident

Ask:

> **Is Kafka itself slow, or are consumers unable to keep up?**

This distinction is critical.

Possible scenarios:

### Scenario A — Consumer Pods down

```text
Kafka
  ↓
Consumer group
  X
Pods unavailable
```

### Scenario B — Consumers running but slow

```text
Kafka
  ↓
Consumer
  ↓
RDS slow
  ↓
Processing latency
```

### Scenario C — Too few partitions

```text
Kafka
  ↓
6 partitions
  ↓
6 active consumers
```

Adding:

```text
20 Pods
```

does not provide 20-way partition parallelism.

### Scenario D — Rebalance storm

```text
Consumer joins/leaves
        ↓
Rebalance
        ↓
Consumption pauses
        ↓
Lag increases
        ↓
Consumer becomes unhealthy
        ↓
More rebalance
```

---

# 8. Production Incident Flow

Use:

```text
Alert
  ↓
Confirm lag
  ↓
Identify topic
  ↓
Identify consumer group
  ↓
Check partition count
  ↓
Check consumer group members
  ↓
Check consumer Pods
  ↓
Check processing rate
  ↓
Check rebalance
  ↓
Check application errors
  ↓
Check downstream dependencies
  ↓
Check Kafka brokers
  ↓
Determine root cause
  ↓
Mitigate
  ↓
Drain lag
  ↓
Validate
  ↓
RCA
```

---

# 9. Identify the Affected Consumer Group

Example:

```text
inventory-service-group
```

The first thing to establish is:

```text
Topic:
order-events

Consumer Group:
inventory-service-group
```

Do not investigate Kafka globally when only one consumer group is affected.

---

# 10. Check Kafka Consumer Group

With Kafka tooling:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --describe \
  --group inventory-service-group
```

Example:

```text
GROUP                    TOPIC          PARTITION   CURRENT-OFFSET   LOG-END-OFFSET   LAG
inventory-service-group order-events   0           1250000          1258000          8000
inventory-service-group order-events   1           1261000          1261000          0
inventory-service-group order-events   2           1249000          1257000          8000
```

This tells us that lag is concentrated on specific partitions.

That is important.

---

# 11. Partition-Level Lag

Suppose:

```text
Partition 0 → lag 100
Partition 1 → lag 100
Partition 2 → lag 50,000
Partition 3 → lag 100
```

This could indicate:

- hot partition
- uneven key distribution
- slow processing for specific records
- one consumer responsible for the hot partition
- problematic partition key

Do not automatically scale every consumer.

Investigate the hot partition.

---

# 12. Check Consumer Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=inventory-service
```

Example:

```text
NAME                              READY   STATUS
inventory-service-6d5d8c-x1      1/1     Running
inventory-service-6d5d8c-x2      1/1     Running
inventory-service-6d5d8c-x3      0/1     CrashLoopBackOff
inventory-service-6d5d8c-x4      1/1     Running
```

Check:

```bash
kubectl describe pod \
  inventory-service-6d5d8c-x3 \
  -n ecommerce
```

And:

```bash
kubectl logs \
  inventory-service-6d5d8c-x3 \
  -n ecommerce \
  --previous
```

---

# 13. Check Deployment

```bash
kubectl get deployment inventory-service \
  -n ecommerce
```

Then:

```bash
kubectl describe deployment inventory-service \
  -n ecommerce
```

Check:

```text
Desired replicas
Available replicas
Unavailable replicas
Rolling update
Recent image
Resource limits
Environment changes
```

---

# 14. Check CPU and Memory

```bash
kubectl top pods \
  -n ecommerce \
  -l app=inventory-service
```

Example:

```text
NAME                       CPU      MEMORY
inventory-service-x1       890m     700Mi
inventory-service-x2       920m     720Mi
inventory-service-x3       950m     710Mi
```

High CPU could mean:

- expensive processing
- serialization
- JSON processing
- encryption
- compression
- inefficient algorithm
- retry loop

High memory could indicate:

- large batches
- memory leak
- unbounded buffers
- large Kafka records
- excessive concurrency

---

# 15. Check Consumer Processing Latency

Suppose:

```text
Normal processing:
50 ms

Current:
800 ms
```

Then even if Pods are healthy:

```text
Consumer throughput ↓
        ↓
Kafka lag ↑
```

Use:

```text
Prometheus
Grafana
ELK
Jaeger
```

to determine where the time is being spent.

---

# 16. Check Downstream Dependencies

For `inventory-service`:

```text
Kafka
  ↓
inventory-service
  ├── PostgreSQL
  ├── Redis
  └── Kafka producer
```

Suppose PostgreSQL latency increases:

```text
Kafka
 ↓
Consumer
 ↓
RDS
 ↓
Slow query
 ↓
Processing latency
 ↓
Consumer throughput decreases
 ↓
Kafka lag increases
```

In this case Kafka is not necessarily the root cause.

---

# 17. Consumer Group Rebalancing

Kafka consumer groups rebalance when membership or partition assignment changes.

Example:

```text
Consumer 1
Consumer 2
Consumer 3
```

Consumer 2 crashes:

```text
Consumer 1
Consumer 3
```

Kafka reassigns partitions.

During repeated membership changes:

```text
Consumer joins
      ↓
Rebalance
      ↓
Consumer leaves
      ↓
Rebalance
      ↓
Consumer joins
      ↓
Rebalance
```

Consumption can be repeatedly interrupted.

---

# 18. Causes of Rebalance Storms

Common causes:

- Pods repeatedly restarting
- insufficient CPU
- insufficient memory
- long GC pauses
- network instability
- incorrect consumer timeout settings
- deployment rolling updates
- aggressive autoscaling
- consumer processing longer than allowed polling interval
- broker/network issues

---

# 19. Kafka Polling Problem

Kafka consumers need to continue polling appropriately.

If message processing takes too long:

```text
poll()
 ↓
Process large batch
 ↓
Very long processing
 ↓
Consumer fails to poll
```

Kafka may consider the consumer unhealthy depending on configuration.

Important settings include:

```text
max.poll.interval.ms
max.poll.records
session.timeout.ms
heartbeat.interval.ms
```

Exact values must be tuned to workload behavior.

Do not blindly increase them.

---

# 20. `max.poll.interval.ms`

Suppose:

```text
max.poll.interval.ms = 300000
```

which is:

```text
5 minutes
```

If application processing takes longer than that between polls, the consumer can leave the group and trigger a rebalance.

For example:

```text
Poll
 ↓
Receive 500 records
 ↓
Processing takes 7 minutes
 ↓
max.poll.interval exceeded
 ↓
Consumer removed
 ↓
Rebalance
```

---

# 21. `max.poll.records`

If a consumer retrieves too many records at once:

```text
500 records
```

and each takes:

```text
500 ms
```

the batch could take substantial time.

Reducing:

```text
max.poll.records
```

may improve processing stability.

But smaller batches can also reduce throughput.

This must be load tested.

---

# 22. Partition Count

Suppose:

```text
Topic partitions = 6
Consumer Pods = 20
```

Maximum useful parallelism for a single consumer group is constrained by those partitions.

Conceptually:

```text
6 partitions
↓
At most 6 actively assigned consumers
```

Additional consumers may remain idle.

Therefore:

> **Increasing Pod count does not automatically increase Kafka throughput.**

---

# 23. Partition Key and Hot Partition

Suppose the producer uses:

```text
customerId
```

as the partition key.

If one customer generates a huge amount of traffic:

```text
customer-123
     ↓
Partition 2
```

Partition 2 becomes hot.

Example:

```text
Partition 0 → 500 msg/sec
Partition 1 → 450 msg/sec
Partition 2 → 5,000 msg/sec
Partition 3 → 400 msg/sec
```

Overall Kafka capacity may appear healthy while one consumer remains overloaded.

---

# 24. Why We Cannot Simply Change the Partition Key

Partition keys often provide ordering.

For example:

```text
orderId
```

may guarantee:

```text
Order Created
Order Paid
Order Shipped
Order Delivered
```

remain ordered within the same partition.

Changing partition strategy can affect:

- ordering
- consumer distribution
- replay behavior
- stateful processing
- downstream assumptions

Therefore partition-key changes require design and migration planning.

---

# 25. Scaling Kafka Consumers

Suppose:

```text
Partitions = 12
Consumers = 3
```

Scaling:

```text
3 → 6
```

may improve throughput.

Scaling:

```text
12 → 20
```

does not necessarily improve partition-level parallelism.

A reasonable model:

```text
Partitions
    ↓
Consumer parallelism
    ↓
Application processing capacity
    ↓
Downstream capacity
```

All four must be considered.

---

# 26. KEDA for Kafka

KEDA can scale workloads based on Kafka lag.

Conceptually:

```text
Kafka
  ↓
Consumer lag
  ↓
KEDA
  ↓
HPA
  ↓
EKS Pods
```

Example conceptual configuration:

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
  maxReplicaCount: 12

  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-bootstrap.ecommerce.svc:9092
        consumerGroup: inventory-service-group
        topic: order-events
        lagThreshold: "1000"
```

These values are illustrative.

Production values should be derived from:

- partition count
- records/sec per Pod
- downstream capacity
- load testing
- failure scenarios

---

# 27. Dangerous Scaling Pattern

Never assume:

```text
Kafka lag = high
        ↓
maxReplicaCount = 100
```

Suppose:

```text
Kafka partitions = 12
```

and:

```text
RDS safe capacity = 100 connections
```

Scaling to 100 Pods may cause:

```text
Kafka
 ↓
12 useful consumers
+
88 idle consumers
```

while potentially creating:

```text
100 application processes
+
100 DB pools
+
100 Redis clients
```

This can create a second incident.

---

# 28. Poison Record

Kafka does not have the same SQS-style DLQ mechanism built into the basic topic model.

A consumer can encounter:

```text
Invalid event
```

and repeatedly fail.

Example:

```text
Partition 3
   ↓
Offset 8,450,221
   ↓
Bad payload
   ↓
Consumer fails
   ↓
Offset not committed
   ↓
Consumer retries
   ↓
Same record
```

This can effectively block progress for that partition.

---

# 29. Poison Record Handling

Production pattern:

```text
Kafka
 ↓
Consumer
 ↓
Processing failure
 ↓
Classify error
 ├── transient → retry
 └── permanent → failure topic/DLQ
                         ↓
                    investigation
```

A failure topic might be:

```text
order-events.inventory.failed
```

Include:

```text
eventId
originalTopic
partition
offset
errorType
errorMessage
failedAt
originalPayload
correlationId
```

Be careful not to store sensitive data unnecessarily.

---

# 30. Offset Commit Strategy

A critical principle:

> **Commit the offset only when the application has safely processed the record according to its delivery semantics.**

Unsafe pattern:

```text
Receive
 ↓
Commit
 ↓
Process
 ↓
Application crashes
```

The message may not be processed again.

Safer conceptual flow:

```text
Receive
 ↓
Process
 ↓
Persist business result
 ↓
Commit offset
```

But database + Kafka offset atomicity is a distributed consistency problem.

Production designs may use:

- transactional processing
- Kafka transactions where appropriate
- idempotent consumers
- transactional outbox
- inbox/idempotency patterns

---

# 31. Idempotency

Kafka consumers must generally tolerate duplicate processing.

Example:

```text
eventId = EVT-10045
```

Consumer checks:

```text
processed_events
```

If already processed:

```text
Skip business operation
Commit offset
```

If not:

```text
Process
Persist result
Record event ID
Commit offset
```

The exact transactional implementation depends on the application's database and Kafka semantics.

---

# 32. Consumer Lag During Deployment

Suppose:

```text
10 Pods
```

are processing normally.

A rolling deployment begins:

```text
Pod 1 terminated
Pod 2 terminated
...
```

During the transition:

```text
Consumer capacity ↓
Kafka lag ↑
```

A short lag spike may be expected.

But if:

```text
Lag continues increasing
```

after deployment completes, investigate.

This is why deployments should be correlated with Kafka dashboards.

---

# 33. Blue-Green Deployment Risk

Suppose:

```text
Blue = 10 consumers
Green = 10 consumers
```

Both may connect to the same consumer group.

Depending on deployment strategy, this can cause:

- rapid rebalances
- temporary assignment churn
- increased connection count
- duplicate processing concerns if application semantics are incorrect

Kafka deployment strategy must be designed together with consumer-group behavior.

---

# 34. Check Kafka Broker Health

If multiple consumer groups experience lag simultaneously:

```text
inventory-group lag ↑
payment-group lag ↑
notification-group lag ↑
reporting-group lag ↑
```

suspect a broader Kafka issue.

Check:

```text
Broker CPU
Broker memory
Network
Request latency
Under-replicated partitions
Offline partitions
ISR
Controller health
Disk utilization
```

For MSK, inspect the appropriate AWS/CloudWatch/MSK metrics and broker events.

---

# 35. Distinguish Consumer vs Broker Incident

### Only one group affected

Likely:

```text
Application
Consumer
Partition assignment
Downstream dependency
```

### Many groups affected

Investigate:

```text
Kafka cluster
Broker
Network
Partition replication
```

This is a powerful incident shortcut.

---

# 36. Example Investigation

Suppose:

```text
inventory-service lag = 500,000
payment-service lag   = 0
notification lag      = 0
```

Kafka itself is probably not globally unavailable.

Investigate:

```text
inventory-service
```

instead of immediately investigating all brokers.

---

# 37. Production Example

`inventory-service` consumes:

```text
order-events
```

Normal:

```text
Producer = 2,000 msg/sec
Consumer = 2,000 msg/sec
Lag ≈ stable
```

After a deployment:

```text
Producer = 2,000 msg/sec
Consumer = 800 msg/sec
```

Lag increases.

Application metrics show:

```text
DB query latency:
20 ms → 800 ms
```

RDS metrics show:

```text
DatabaseConnections ↑
DB load ↑
```

Therefore:

```text
Kafka lag
   ↓
Consumer slow
   ↓
RDS slow
```

Kafka is only exposing the downstream bottleneck.

---

# 38. Immediate Mitigation

Depending on the root cause:

### Consumer crash

```text
Rollback deployment
```

### CPU saturation

```text
Scale consumers if partitions/downstream allow
```

### RDS bottleneck

```text
Protect RDS
Reduce consumer concurrency
Optimize DB
```

### Hot partition

```text
Investigate key distribution
```

### Poison record

```text
Isolate failing event
```

### Rebalance storm

```text
Stabilize consumer Pods/configuration
```

### Broker issue

```text
Follow Kafka/MSK recovery procedure
```

---

# 39. Backlog Drain

Suppose:

```text
Lag = 500,000 records
```

Do not simply maximize replicas.

Determine:

```text
Safe processing capacity
```

Example:

```text
Normal production:
2,000 records/sec

Safe temporary capacity:
4,000 records/sec
```

Then:

```text
Net drain:
4,000 - 2,000
= 2,000 records/sec
```

Approximate drain time:

```text
500,000 / 2,000
= 250 seconds
≈ 4.2 minutes
```

Real drain time may be longer.

---

# 40. Monitoring During Drain

Watch simultaneously:

```text
Kafka lag
Kafka processing rate
Consumer errors
Consumer CPU
Consumer memory
RDS connections
RDS latency
Redis latency
External API latency
Kafka broker health
```

Do not watch only:

```text
Lag ↓
```

because:

```text
Lag ↓
RDS ↓
```

may be happening at the same time.

---

# 41. DevOps vs Application vs DBA

## DevOps / Platform

Responsible for:

- EKS
- KEDA
- Kafka/MSK infrastructure
- networking
- IAM
- deployment
- scaling
- monitoring
- alerting
- node capacity
- Terraform
- GitOps

## Application Team

Responsible for:

- Kafka consumer code
- partition key design
- processing logic
- offset behavior
- idempotency
- retry strategy
- serialization
- batching
- consumer configuration
- event schema

## DBA

If PostgreSQL is involved:

- slow queries
- locks
- connections
- indexes
- DB resource pressure
- query plans

## Kafka/MSK Platform Owner

Depending on organization:

- brokers
- partitions
- replication
- ISR
- broker capacity
- Kafka upgrades
- cluster configuration

In smaller teams, DevOps/SRE may own the Kafka platform as well.

---

# 42. Example RCA — Kafka Consumer Lag

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-155` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 16:05–16:48 IST |
| Affected Service | `inventory-service` |
| Topic | `order-events` |
| Consumer Group | `inventory-service-group` |
| Root Cause | Slow PostgreSQL queries reduced consumer throughput |
| Customer Impact | Inventory updates delayed |
| Permanent Data Loss | No |

---

## Customer Impact

Orders continued to be accepted.

However, inventory updates were delayed.

This created a temporary risk of:

```text
Order
 ↓
Inventory event
 ↓
Kafka
 ↓
inventory-service
 ↓
Delayed processing
```

Inventory availability shown to customers could therefore become temporarily stale.

No permanent event loss occurred.

---

## Timeline

```text
16:00  Normal operation
       Kafka lag ≈ 2,000

16:05  RDS query latency begins increasing

16:08  inventory-service processing latency increases

16:10  Consumer throughput decreases

16:12  Kafka lag reaches 25,000

16:15  Lag alert fires

16:17  DevOps checks consumer group

16:19  Only inventory-service group affected

16:21  Consumer Pods confirmed healthy

16:23  Consumer CPU normal

16:25  Application traces show slow PostgreSQL queries

16:27  DBA confirms database query latency increase

16:30  High-cost query identified

16:32  Consumer concurrency temporarily reduced
       to protect RDS

16:35  Query optimization applied

16:38  DB latency begins recovering

16:40  Consumer throughput increases

16:43  Kafka lag begins draining

16:47  Lag returns close to normal

16:48  Incident resolved
```

---

# 43. Investigation Evidence

### Kafka

```text
Lag:
2,000 → 180,000
```

### Consumer Pods

```text
All Pods:
Running / Ready
```

Therefore:

```text
Not a consumer availability problem
```

### CPU

```text
CPU:
45–60%
```

Therefore:

```text
Not primarily CPU saturation
```

### RDS

```text
Query latency:
25 ms → 900 ms
```

### Application

Tracing showed:

```text
Kafka consume
   ↓
Inventory processing
   ↓
PostgreSQL query
   ↓
900 ms
```

Therefore consumer throughput decreased.

---

# 44. Root Cause

A new database query introduced by an application change was inefficient for the production dataset.

The query caused increased PostgreSQL latency.

That increased Kafka record-processing time.

Therefore:

```text
Slow DB query
     ↓
Consumer processing latency
     ↓
Consumer throughput ↓
     ↓
Producer rate > Consumer rate
     ↓
Kafka lag ↑
```

Kafka itself was healthy.

---

# 45. Amplification Mechanism

The original problem:

```text
Slow PostgreSQL query
```

was amplified by:

```text
Kafka lag
     ↓
KEDA observes lag
     ↓
Consumer replicas increase
     ↓
More DB queries
     ↓
RDS pressure increases
     ↓
Query latency increases further
     ↓
Consumer throughput decreases
     ↓
Kafka lag increases further
```

This created a feedback loop.

---

# 46. Contributing Factors

1. Inefficient SQL query.
2. Production dataset was much larger than test data.
3. KEDA scaling was based on Kafka lag.
4. Consumer scaling did not account sufficiently for RDS capacity.
5. Missing query-performance release gate.
6. No alert linking Kafka lag and RDS latency.
7. Consumer concurrency was higher than the safe DB capacity.

---

# 47. What Went Well

- Kafka retained the records.
- No event loss occurred.
- Consumer Pods remained healthy.
- Jaeger identified the slow database span.
- DBA quickly identified the expensive query.
- Reducing consumer concurrency protected RDS.

---

# 48. What Went Wrong

- Query performance regression reached production.
- KEDA amplified the downstream bottleneck.
- Kafka lag alert did not immediately identify the RDS relationship.
- Performance testing used smaller datasets.
- Database capacity was not included in the Kafka scaling policy.

---

# 49. Immediate Mitigation

```text
1. Confirm Kafka cluster health
2. Confirm consumer health
3. Identify RDS bottleneck
4. Reduce consumer concurrency
5. Protect RDS
6. Optimize/fix SQL
7. Verify RDS latency
8. Restore safe consumer capacity
9. Drain Kafka lag
10. Validate inventory consistency
```

---

# 50. Corrective Actions

| Action | Owner |
|---|---|
| Optimize problematic SQL | Application + DBA |
| Add query performance testing | Application/SRE |
| Add Kafka-to-RDS dependency dashboard | SRE |
| Set safe KEDA max replicas | Platform |
| Add DB-aware consumer capacity limits | Platform + DBA |
| Add Kafka lag burn-rate style alerting | SRE |
| Add production-sized datasets to load tests | SRE |
| Add slow-query monitoring | DBA |
| Add consumer processing latency SLO | SRE |

---

# 51. Five Whys

### Why did inventory updates become delayed?

Because `inventory-service` processed Kafka records slowly.

### Why was processing slow?

PostgreSQL queries became slow.

### Why did PostgreSQL queries become slow?

A new query performed poorly against the production-sized dataset.

### Why did the query create a larger incident?

KEDA increased consumer replicas based on Kafka lag.

### Why did scaling make the problem worse?

Consumer scaling increased database concurrency without considering the safe RDS capacity.

---

# 52. Final Root Cause Statement

> The Kafka consumer lag incident was caused by a production database query regression in `inventory-service`. Increased PostgreSQL latency reduced consumer throughput, causing Kafka lag. KEDA then amplified the incident by increasing consumer concurrency and additional database load. The incident was resolved by protecting RDS, fixing the query, and safely draining the Kafka backlog.

---

# 53. Prevention Architecture

The improved architecture becomes:

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
               Consumer Pods
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
       Processing              Capacity
        latency                limits
          │                       │
          └───────────┬───────────┘
                      ▼
                    RDS
                      │
                      ▼
              Database Capacity
```

Scaling must consider **both Kafka and downstream capacity**.

---

# 54. Production Kafka Dashboard

Create a dashboard containing:

```text
┌─────────────────────────────────────────────┐
│             KAFKA OPERATIONS                │
├─────────────────────────────────────────────┤
│ Consumer Lag                                 │
│ Lag by Partition                             │
│ Records Produced/sec                         │
│ Records Consumed/sec                         │
│ Consumer Processing Latency                  │
│ Consumer Error Rate                          │
├─────────────────────────────────────────────┤
│ Consumer Group Members                       │
│ Rebalances                                  │
│ Partition Distribution                       │
│ Poll/Processing Metrics                      │
├─────────────────────────────────────────────┤
│ Kafka Broker Health                          │
│ Under-Replicated Partitions                 │
│ ISR                                          │
│ Broker CPU / Memory / Network               │
├─────────────────────────────────────────────┤
│ RDS Connections                              │
│ RDS Query Latency                            │
│ Redis Latency                                │
│ External API Errors                          │
└─────────────────────────────────────────────┘
```

---

# 55. Recommended Alerts

### Consumer lag

```text
Lag continuously increasing
```

### Consumer unavailable

```text
Consumer group members = 0
```

### Rebalance

```text
Unexpected rebalance frequency
```

### Processing latency

```text
Consumer processing latency above threshold
```

### Partition imbalance

```text
One partition has significantly higher lag
```

### Kafka infrastructure

```text
Under-replicated partitions
Offline partitions
Broker health issues
```

### Dependencies

```text
RDS latency
RDS connections
Redis latency
External API errors
```

---

# 56. Runbook

Create:

```text
runbooks/kafka-consumer-lag.md
```

Recommended flow:

```text
1. Identify topic
2. Identify consumer group
3. Check total lag
4. Check partition-level lag
5. Check producer rate
6. Check consumer rate
7. Check consumer Pods
8. Check rebalances
9. Check max.poll behavior
10. Check processing latency
11. Check CPU/memory
12. Check RDS/Redis/external APIs
13. Check broker health
14. Determine root cause
15. Mitigate
16. Drain lag
17. Validate events
18. RCA
```

---

# 57. Production Checklist

### Kafka

- [ ] Topic identified
- [ ] Consumer group identified
- [ ] Partition count checked
- [ ] Partition-level lag checked
- [ ] Producer rate checked
- [ ] Consumer rate checked
- [ ] Broker health checked
- [ ] Replication health checked

### Consumers

- [ ] Pods healthy
- [ ] Consumer group members healthy
- [ ] Rebalances checked
- [ ] Processing latency checked
- [ ] CPU checked
- [ ] Memory checked
- [ ] Error rate checked

### Dependencies

- [ ] RDS checked
- [ ] Redis checked
- [ ] External APIs checked
- [ ] DB connections checked
- [ ] Network checked

### Recovery

- [ ] Root cause identified
- [ ] Downstream protected
- [ ] Consumer scaling controlled
- [ ] Lag draining
- [ ] No data loss
- [ ] Duplicate processing checked
- [ ] Business state validated

---

# 58. Key Production Lessons

### Lesson 1

> **Kafka lag is a symptom, not automatically a Kafka problem.**

### Lesson 2

> **Always inspect lag by partition and consumer group.**

### Lesson 3

> **More consumer Pods do not help beyond partition parallelism.**

### Lesson 4

> **Consumer scaling can overload RDS, Redis, or external APIs.**

### Lesson 5

> **Rebalance storms can create significant lag.**

### Lesson 6

> **Slow database queries can appear as Kafka incidents.**

### Lesson 7

> **Partition keys directly influence scalability and hot partitions.**

### Lesson 8

> **Consumers must be idempotent.**

### Lesson 9

> **Offset commits must align with processing semantics.**

### Lesson 10

> **The RCA must identify both the original bottleneck and any autoscaling amplification.**

---

# 59. Interview Answer

If asked:

> **"How do you troubleshoot Kafka consumer lag in production?"**

A strong answer is:

> "First I identify the affected topic and consumer group and check total and partition-level lag. Then I compare producer throughput with consumer throughput and verify consumer group membership, Pod health, CPU, memory, processing latency and rebalance frequency. I also check whether the lag is isolated to one consumer group or affects multiple groups, because that helps distinguish an application problem from a Kafka platform problem. If consumers are healthy but slow, I trace the processing path into dependencies such as RDS, Redis or external APIs. I also check partition count and hot partitions before increasing replicas. If KEDA is scaling consumers, I make sure scaling does not exceed partition parallelism or downstream database capacity. After fixing the root cause, I drain the lag in a controlled way, verify idempotency and event consistency, and document the incident with an RCA."

---

# 60. Final Incident Architecture

```text
                         ┌───────────────┐
                         │   Producer    │
                         │ order-service │
                         └───────┬───────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │    Kafka / MSK   │
                       │  order-events    │
                       └────────┬─────────┘
                                │
                     ┌──────────┴──────────┐
                     │                     │
                 Lag Metrics          Partition Lag
                     │                     │
                     └──────────┬──────────┘
                                ▼
                              KEDA
                                │
                                ▼
                               HPA
                                │
                                ▼
                       ┌────────────────┐
                       │ EKS Consumers  │
                       └───────┬────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
             RDS             Redis        External APIs
              │                │                │
              └────────────────┼────────────────┘
                               ▼
                         Observability
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
          Prometheus          ELK             Jaeger
              │                │                │
              └────────────────┼────────────────┘
                               ▼
                         Alertmanager
                               │
                               ▼
                         Incident Team
```

---

## Phase 18 Progress

```text
143 CrashLoopBackOff              ✅
144 OOMKilled                     ✅
145 CPU Saturation                ✅
146 Memory Saturation             ✅
147 Disk Full                     ✅
148 ImagePullBackOff              ✅
149 ALB 5xx                       ✅
150 Service Connectivity Failure  ✅
151 DNS Failure                   ✅
152 RDS Connection Exhaustion     ✅
153 Redis Failure                 ✅
154 SQS Backlog                   ✅
155 Kafka Consumer Lag            ✅
```

**Next: `156 — Node Failure Production Incident`**