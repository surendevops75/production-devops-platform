# 154 — SQS Backlog Production Incident

> **Phase 18 — Incident Management**  
> **Incident #154**  
> **Service:** SQS-based asynchronous processing  
> **Environment:** Production  
> **Primary example:** `notification-service`

---

# 1. Objective

This chapter covers how to troubleshoot and resolve an **Amazon SQS backlog incident** in a production EKS environment.

The goal is not simply:

```text
Queue has many messages
        ↓
Increase Pods
```

The real objective is to determine:

```text
Why is the queue growing?
        ↓
Are consumers healthy?
        ↓
Can consumers process messages successfully?
        ↓
Is the downstream dependency healthy?
        ↓
Can we safely increase consumer capacity?
        ↓
Can we drain the backlog without creating another incident?
```

A queue backlog is often a **symptom**, not the root cause.

---

# 2. Production Architecture

Our asynchronous architecture:

```text
                    ┌──────────────────────┐
                    │      Producer        │
                    │    order-service     │
                    └──────────┬───────────┘
                               │
                               │ SendMessage
                               ▼
                    ┌──────────────────────┐
                    │    Amazon SQS        │
                    │   order-events       │
                    └──────────┬───────────┘
                               │
                         Poll / Receive
                               │
                               ▼
                    ┌──────────────────────┐
                    │ notification-service │
                    │      EKS Pods         │
                    └──────────┬───────────┘
                               │
                  ┌────────────┼────────────┐
                  ▼            ▼            ▼
                Email         SMS       PostgreSQL
              Provider      Provider       / Redis
```

KEDA can scale the consumers:

```text
SQS Queue
    │
    │ queue metrics
    ▼
  KEDA
    │
    ▼
Deployment replicas
    │
    ▼
Consumer Pods
```

---

# 3. What Is an SQS Backlog?

An SQS backlog occurs when:

```text
Messages arriving > Messages successfully processed
```

For example:

```text
Producer rate       = 1,000 messages/sec
Consumer capacity   =   600 messages/sec

Net backlog growth  =   400 messages/sec
```

If this continues:

```text
1 minute  → 24,000 messages
5 minutes → 120,000 messages
10 minutes → 240,000 messages
```

The important metric is therefore not only:

```text
Queue depth
```

but also:

```text
Incoming rate
Processing rate
Backlog age
Consumer success rate
Consumer failure rate
```

---

# 4. Important SQS Metrics

For an SQS incident, check:

### 4.1 ApproximateNumberOfMessagesVisible

Messages available for consumers.

```text
Queue depth increasing
        ↓
Consumers cannot keep up
```

---

### 4.2 ApproximateNumberOfMessagesNotVisible

Messages currently being processed or inside the visibility timeout.

A high value can indicate:

- many active consumers
- long processing time
- slow downstream dependency
- visibility timeout problems
- stuck consumers

---

### 4.3 ApproximateAgeOfOldestMessage

One of the most important operational metrics.

Example:

```text
Queue depth: 100,000
Oldest message age: 45 minutes
```

This means customers/events may be experiencing significant processing delay.

---

### 4.4 NumberOfMessagesSent

Producer rate.

---

### 4.5 NumberOfMessagesReceived

Consumer receive activity.

---

### 4.6 NumberOfMessagesDeleted

Successful processing completion.

Remember:

```text
Receive ≠ successful processing
```

A consumer can receive a message, fail processing, and leave it for retry.

---

### 4.7 DLQ Metrics

Monitor:

```text
DLQ message count
DLQ oldest message age
DLQ receive/send activity
```

A growing DLQ can indicate a **poison message** or permanent processing failure.

---

# 5. First Question During the Incident

Do not immediately scale consumers.

Ask:

> **Is the backlog growing because consumers are too few, or because consumers cannot successfully process messages?**

There are several possibilities.

### Case 1 — Consumers are down

```text
SQS
 ↓
Consumers = 0
 ↓
Backlog grows
```

### Case 2 — Consumers are too few

```text
SQS
 ↓
5 Pods
 ↓
Processing capacity insufficient
```

### Case 3 — Consumers are healthy but slow

```text
SQS
 ↓
Consumers
 ↓
RDS slow
 ↓
Processing takes longer
 ↓
Backlog grows
```

### Case 4 — Poison messages

```text
Message
 ↓
Consumer
 ↓
Processing fails
 ↓
Retry
 ↓
Processing fails
 ↓
Retry
 ↓
DLQ
```

### Case 5 — Visibility timeout problem

```text
Consumer receives message
 ↓
Processing takes 5 minutes
 ↓
Visibility timeout = 60 seconds
 ↓
Message becomes visible again
 ↓
Another consumer receives same message
```

This can create duplicate processing and additional load.

---

# 6. Incident Detection

Example Prometheus/CloudWatch alert:

```text
SQS queue oldest message age > 300 seconds
```

or:

```text
Queue depth continuously increasing for 5 minutes
```

or:

```text
DLQ messages > 0
```

Example alert:

```yaml
groups:
  - name: sqs-alerts
    rules:

      - alert: SQSBacklogGrowing
        expr: aws_sqs_approximate_number_of_messages_visible > 10000
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "SQS backlog is growing"
          runbook: "runbooks/sqs-backlog.md"

      - alert: SQSOldestMessageTooOld
        expr: aws_sqs_approximate_age_of_oldest_message > 300
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SQS oldest message is too old"
```

Exact metric names depend on the CloudWatch exporter/integration used.

---

# 7. Production Incident Flow

Use:

```text
Alert
  ↓
Confirm backlog
  ↓
Check oldest message age
  ↓
Check producer rate
  ↓
Check consumer Pods
  ↓
Check KEDA
  ↓
Check consumer logs
  ↓
Check processing latency
  ↓
Check downstream dependencies
  ↓
Check DLQ
  ↓
Determine root cause
  ↓
Protect downstream systems
  ↓
Scale safely / fix consumers
  ↓
Drain backlog
  ↓
Validate
  ↓
RCA
```

---

# 8. Step 1 — Check Queue Attributes

Example:

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names \
  ApproximateNumberOfMessages \
  ApproximateNumberOfMessagesNotVisible \
  ApproximateNumberOfMessagesDelayed \
  VisibilityTimeout \
  MessageRetentionPeriod
```

Check:

```text
Visible
NotVisible
Delayed
VisibilityTimeout
Retention
```

---

# 9. Step 2 — Check Queue Depth

Example:

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages
```

Example:

```text
Visible messages = 180,000
```

Do not immediately conclude:

> "We need more Pods."

Continue investigating.

---

# 10. Step 3 — Check Oldest Message

Check:

```text
ApproximateAgeOfOldestMessage
```

Example:

```text
Oldest message = 27 minutes
```

This means the consumer system is falling behind significantly.

---

# 11. Step 4 — Check Consumer Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=notification-service
```

Example:

```text
NAME                                  READY   STATUS
notification-service-7c9d8b5d4-x1     1/1     Running
notification-service-7c9d8b5d4-x2     1/1     Running
notification-service-7c9d8b5d4-x3     0/1     CrashLoopBackOff
```

Immediately inspect:

```bash
kubectl describe pod \
  notification-service-7c9d8b5d4-x3 \
  -n ecommerce
```

And:

```bash
kubectl logs \
  notification-service-7c9d8b5d4-x3 \
  -n ecommerce \
  --previous
```

---

# 12. Step 5 — Check Deployment

```bash
kubectl get deployment notification-service \
  -n ecommerce
```

Then:

```bash
kubectl describe deployment notification-service \
  -n ecommerce
```

Check:

```text
Desired replicas
Current replicas
Available replicas
Unavailable replicas
Rolling update
Recent changes
```

---

# 13. Step 6 — Check KEDA

If KEDA is used:

```bash
kubectl get scaledobject \
  -n ecommerce
```

Example:

```text
NAME                    READY
notification-sqs        True
```

Inspect:

```bash
kubectl describe scaledobject notification-sqs \
  -n ecommerce
```

Check:

```text
Queue name
Polling configuration
Target queue length
Min replicas
Max replicas
Authentication
Current metric
Scaling status
```

Also check:

```bash
kubectl get hpa \
  -n ecommerce
```

KEDA commonly creates/manages an HPA behind the scenes.

---

# 14. KEDA Architecture

Example:

```text
                 SQS
                  │
                  │ queue length
                  ▼
                KEDA
                  │
                  ▼
                 HPA
                  │
          ┌───────┴────────┐
          ▼                ▼
       Pod 1             Pod N
          │                │
          └──────┬─────────┘
                 ▼
              SQS Consumer
```

Example conceptual configuration:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-sqs
  namespace: ecommerce
spec:
  scaleTargetRef:
    name: notification-service

  minReplicaCount: 2
  maxReplicaCount: 20

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123456789012/order-events
        queueLength: "100"
        awsRegion: ap-south-1
```

Treat these values as examples.

Production values must come from load testing and downstream capacity.

---

# 15. Why Maximum Replicas Matter

Suppose:

```text
SQS backlog = 500,000
```

You might think:

```text
Increase consumers:
10 → 100 Pods
```

But suppose every consumer Pod has:

```text
Hikari pool = 10
```

Then:

```text
100 Pods × 10 DB connections
= 1,000 potential DB connections
```

If the consumer writes to PostgreSQL, you may turn:

```text
SQS incident
```

into:

```text
RDS connection exhaustion
```

Therefore:

> **Queue scaling must respect downstream capacity.**

---

# 16. Consumer Capacity Calculation

Suppose one consumer Pod processes:

```text
50 messages/sec
```

Current:

```text
10 Pods
```

Capacity:

```text
10 × 50
= 500 messages/sec
```

Incoming:

```text
800 messages/sec
```

Backlog grows:

```text
800 - 500
= 300 messages/sec
```

If we scale to 16 Pods:

```text
16 × 50
= 800 messages/sec
```

Now the system approximately keeps up.

But if RDS can only support 12 Pods safely:

```text
12 × 50
= 600 messages/sec
```

then we need to solve the downstream bottleneck rather than blindly scaling.

---

# 17. Backlog Drain Rate

Suppose:

```text
Backlog = 120,000
```

Incoming:

```text
500 msg/sec
```

Consumer processing:

```text
1,000 msg/sec
```

Net drain:

```text
1,000 - 500
= 500 msg/sec
```

Approximate drain time:

```text
120,000 / 500
= 240 seconds
= 4 minutes
```

This is an approximation.

Real systems must account for:

- processing variability
- retries
- failures
- visibility timeout
- throttling
- autoscaling delay
- downstream latency
- duplicate deliveries

---

# 18. Check Consumer Logs

```bash
kubectl logs \
  -n ecommerce \
  -l app=notification-service \
  --tail=200
```

Look for:

```text
Timeout
Connection refused
Connection pool exhausted
HTTP 429
HTTP 500
SerializationException
Authentication failure
AccessDenied
Message processing failed
Database timeout
Redis timeout
```

Example:

```text
ERROR Failed to process message
ERROR PostgreSQL connection timeout
ERROR HikariPool - Connection is not available
```

This changes the diagnosis.

The SQS queue may not be the real problem.

The actual problem could be:

```text
SQS
 ↓
Consumer
 ↓
RDS
 ↓
Connection exhaustion
```

---

# 19. Check Processing Latency

Suppose:

```text
Normal processing time = 100 ms
Current processing time = 4 seconds
```

Even though Pods are healthy, throughput collapses.

Possible causes:

```text
RDS latency
Redis latency
External API
Slow email provider
Network problems
CPU saturation
GC
Lock contention
```

Use:

```text
Prometheus
Grafana
ELK
Jaeger
```

to trace the processing path.

---

# 20. Check Downstream Dependencies

For `notification-service`:

```text
SQS
 ↓
notification-service
 ├── Email provider
 ├── SMS provider
 ├── Redis
 └── PostgreSQL
```

For an order event consumer:

```text
SQS
 ↓
inventory-service
 ↓
PostgreSQL
Redis
Kafka
```

Check the dependency before increasing replicas.

---

# 21. Common Root Cause — Consumers Down

Example:

```text
SQS backlog
       ↑
       │
Consumers = 0
       │
Deployment failed
       │
Bad image
       │
ImagePullBackOff
```

The correct fix is:

```text
Fix consumer deployment
```

not:

```text
Increase KEDA max replicas
```

---

# 22. Common Root Cause — Insufficient Consumers

Example:

```text
Incoming = 1,000 msg/sec
Processing = 500 msg/sec
```

Consumers are healthy.

Then:

```text
Scale consumers
```

may be appropriate.

But first confirm:

```text
RDS capacity
API provider limits
CPU
memory
network
Kafka
Redis
```

---

# 23. Common Root Cause — Poison Message

Example:

```text
Message #A123
       ↓
Consumer
       ↓
Invalid payload
       ↓
Processing fails
       ↓
Message returns after visibility timeout
       ↓
Retry
       ↓
Failure
```

Eventually:

```text
Message
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
DLQ
```

A poison message can consume consumer capacity repeatedly.

---

# 24. Idempotency

SQS provides **at-least-once delivery** for Standard queues.

Therefore the consumer must tolerate duplicate messages.

Example:

```text
eventId = EVT-12345
```

Before processing:

```text
Check processed_events
```

If already processed:

```text
ACK/delete message
```

If not:

```text
Process
 ↓
Persist result
 ↓
Record event ID
 ↓
Delete message
```

Never assume:

```text
One message = exactly one delivery
```

---

# 25. Visibility Timeout

Suppose:

```text
Processing time = 3 minutes
Visibility timeout = 60 seconds
```

Then:

```text
Consumer receives message
        ↓
60 seconds
        ↓
Message becomes visible
        ↓
Another consumer receives same message
```

Now multiple Pods may process the same message.

The visibility timeout should be designed around realistic processing time, with a safety margin.

Long-running processing may require extending visibility timeout.

---

# 26. Long Polling

SQS consumers should generally use long polling rather than continuously making empty receive requests.

Conceptually:

```text
Short polling:

Consumer → SQS
Consumer → SQS
Consumer → SQS
Consumer → SQS
```

Long polling:

```text
Consumer ───────────────→ SQS
                          waits
                          message arrives
                          ↓
Consumer ←─────────────── message
```

Benefits include:

- fewer empty receives
- lower API call volume
- more efficient consumers

---

# 27. Retry Storm

A dangerous situation:

```text
SQS
 ↓
Consumer
 ↓
External API fails
 ↓
Retry immediately
 ↓
API fails
 ↓
Retry
 ↓
API fails
```

This can create:

```text
SQS backlog
+
consumer CPU
+
external API load
+
network load
```

Use:

```text
Exponential backoff
Jitter
Bounded retries
Circuit breaker
DLQ
```

---

# 28. Protect the Downstream System

During a backlog incident, the goal is **not necessarily to drain the queue as fast as possible**.

Suppose:

```text
RDS safe capacity = 500 writes/sec
```

and:

```text
200 consumers
```

could generate:

```text
2,000 writes/sec
```

That would create another incident.

Therefore:

```text
SQS backlog
     ↓
Controlled consumer scaling
     ↓
Downstream capacity
     ↓
Safe processing
```

is better than uncontrolled scaling.

---

# 29. Controlled Incident Mitigation

Example:

```bash
kubectl scale deployment notification-service \
  --replicas=10 \
  -n ecommerce
```

However, in GitOps production environments, do not treat this as the permanent solution.

The proper permanent change should be made in Git:

```text
Git
 ↓
PR
 ↓
Review
 ↓
GitOps
 ↓
ArgoCD
 ↓
EKS
```

Emergency manual actions should be documented and reconciled afterward.

---

# 30. DLQ Investigation

Check the DLQ:

```bash
aws sqs get-queue-attributes \
  --queue-url "$DLQ_URL" \
  --attribute-names ApproximateNumberOfMessages
```

If DLQ messages increase:

```text
Do NOT immediately replay everything.
```

First determine:

```text
Why did messages fail?
```

Otherwise:

```text
DLQ
 ↓
Replay
 ↓
Same failure
 ↓
DLQ
 ↓
Replay
```

creates an operational loop.

---

# 31. Controlled DLQ Replay

Safe process:

```text
DLQ
 ↓
Inspect messages
 ↓
Identify failure reason
 ↓
Fix consumer
 ↓
Test with small batch
 ↓
Replay controlled batch
 ↓
Monitor
 ↓
Continue
```

Never replay hundreds of thousands of messages blindly.

---

# 32. IAM Failure

The consumer needs permission to receive/delete messages.

Typical permissions include:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Check workload identity architecture.

For EKS:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
SQS
```

If IAM breaks:

```text
Consumer healthy
        ↓
Cannot receive/delete
        ↓
Backlog grows
```

Application logs may show:

```text
AccessDenied
```

---

# 33. Network Failure

If the consumer cannot reach SQS:

```text
Consumer
   ↓
AWS endpoint
   X
Network/DNS
```

Check:

- DNS
- NAT Gateway
- VPC endpoints
- NetworkPolicy
- security controls
- AWS regional connectivity

For private EKS architectures, consider appropriate AWS VPC endpoints where they make architectural and cost sense.

---

# 34. Producer vs Consumer Diagnosis

This distinction is extremely important.

### Producer Problem

```text
Producer suddenly sends
10× normal traffic
```

Example:

```text
Normal = 500 msg/sec
Current = 5,000 msg/sec
```

Consumers may be healthy but unable to match the sudden load.

### Consumer Problem

```text
Normal producer = 500 msg/sec
Consumer capacity drops
500 → 100 msg/sec
```

Backlog grows.

### Both

Sometimes:

```text
Producer ↑
Consumer ↓
```

This produces a very rapid backlog increase.

---

# 35. SQS Backlog Decision Tree

```text
                SQS backlog?
                     │
                     ▼
             Check queue age
                     │
                     ▼
             Are consumers running?
                /           \
              No             Yes
              │               │
         Fix deployment       ▼
                          Check processing
                              │
                              ▼
                     Is processing slow?
                        /          \
                      Yes          No
                       │            │
                 Check RDS/API      ▼
                 Redis/network   Check capacity
                                      │
                                      ▼
                               Is arrival rate
                               > processing rate?
                                  /       \
                                Yes        No
                                 │          │
                           Scale safely    Check
                           / optimize      retries,
                                          poison msgs,
                                          visibility
```

---

# 36. DevOps vs Application Team

During an SQS incident, ownership should be clear.

## DevOps / Platform Team

Usually owns:

- SQS infrastructure
- Terraform
- queue configuration
- IAM
- KEDA
- EKS
- autoscaling
- monitoring
- dashboards
- alerts
- networking
- DLQ infrastructure
- incident coordination

## Application Team

Usually owns:

- message handler
- business processing
- event schema
- idempotency
- retry logic
- visibility-timeout behavior
- application concurrency
- downstream API usage
- database transactions

## DBA

Gets involved when:

```text
SQS
 ↓
Consumer
 ↓
PostgreSQL
 ↓
DB bottleneck
```

DBA investigates:

- connections
- locks
- slow queries
- database load
- indexes
- transactions

---

# 37. Example Production Scenario

Suppose:

```text
Queue:
order-events

Normal:
2,000 messages/min

Incident:
20,000 messages/min
```

Consumer:

```text
10 Pods
100 msg/sec total
```

Incoming:

```text
333 msg/sec
```

Backlog grows rapidly.

KEDA increases:

```text
10 → 20 Pods
```

But then RDS connections increase:

```text
100 → 300
```

RDS begins slowing down.

Processing falls:

```text
100 msg/sec → 60 msg/sec
```

Now the original SQS incident becomes:

```text
SQS backlog
      ↓
Consumer scale-out
      ↓
RDS overload
      ↓
Consumer slowdown
      ↓
Even larger SQS backlog
```

This is a **scaling feedback loop**.

---

# 38. Production Mitigation for This Scenario

Instead of unlimited scaling:

```text
1. Identify RDS bottleneck
2. Cap consumer replicas
3. Reduce retry pressure
4. Optimize/restore DB capacity
5. Process messages at safe rate
6. Monitor queue age
7. Drain backlog gradually
```

This is much safer than:

```text
SQS backlog → 100 Pods
```

---

# 39. Example RCA — SQS Consumer Deployment Failure

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-154` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 14:05–14:42 IST |
| Affected Service | `notification-service` |
| Queue | `order-notification-events` |
| Root Cause | Bad container image caused consumer Pods to fail |
| Customer Impact | Notification processing delayed |
| Permanent Data Loss | No |

---

## Customer Impact

Order processing itself continued successfully.

However:

```text
Order completed
      ↓
Notification event
      ↓
SQS
      ↓
notification-service
      X
```

Customers experienced delayed:

- email notifications
- SMS notifications
- order status notifications

No order or payment data was lost.

---

## Timeline

```text
14:00  Notification system normal
       Queue depth = 1,200
       Consumer Pods = 10

14:05  New notification-service version deployed

14:07  New Pods begin failing

14:08  Existing Pods terminate during rollout

14:09  Consumer replicas fall to 0

14:10  SQS backlog begins increasing

14:12  Oldest message age exceeds 2 minutes

14:14  SQS backlog alert fires

14:15  SEV-1 declared

14:17  DevOps checks SQS metrics

14:18  Queue depth = 18,000

14:19  Consumer deployment inspected

14:20  Pods show ImagePullBackOff

14:22  ECR image/tag investigation begins

14:24  GitOps manifest references incorrect image tag

14:26  Previous image digest identified

14:28  GitOps reverted to known-good digest

14:30  ArgoCD synchronization completes

14:32  Consumer Pods become Ready

14:33  KEDA begins scaling consumers

14:35  Processing rate increases

14:38  Queue backlog starts draining

14:40  Oldest message age decreases

14:42  Queue returns to normal operating range
```

---

## Investigation

### Queue

```text
Initial:
1,200 messages

Peak:
18,000 messages
```

### Consumer Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=notification-service
```

Result:

```text
notification-service-abc123   0/1   ImagePullBackOff
notification-service-def456   0/1   ImagePullBackOff
```

### Deployment

```bash
kubectl describe deployment notification-service \
  -n ecommerce
```

The deployment referenced:

```text
notification-service:2026.10.04.2
```

But ECR contained:

```text
notification-service:2026.10.04.1
```

---

## Root Cause

The production GitOps configuration referenced an image tag that did not exist in ECR.

Therefore:

```text
GitOps
  ↓
ArgoCD
  ↓
EKS Deployment
  ↓
Invalid ECR image
  ↓
ImagePullBackOff
  ↓
Consumer replicas unavailable
  ↓
SQS messages accumulate
```

---

## Amplification Mechanism

The direct failure was:

```text
Bad image
```

But the customer-impacting symptom was:

```text
SQS backlog
```

The amplification was:

```text
Consumer Pods = 0
        ↓
Processing rate = 0
        ↓
Producer continues
        ↓
Queue grows
        ↓
Oldest message age increases
        ↓
Customer notification delay
```

---

## Contributing Factors

1. Incorrect image tag.
2. Manual GitOps image update.
3. No automated ECR image existence validation.
4. No canary deployment for notification-service.
5. Deployment allowed all consumers to be replaced.
6. SQS backlog alert threshold was too high.
7. No automated rollback when consumer availability reached zero.

---

## What Went Well

- SQS retained the messages.
- No permanent data loss.
- Queue metrics clearly showed increasing backlog.
- GitOps allowed quick rollback.
- Previous image digest was available.
- KEDA automatically recovered consumer scaling after Pods became healthy.

---

## What Went Wrong

- Production deployment removed the last healthy consumers.
- Invalid image reference passed the deployment process.
- No minimum healthy consumer protection existed.
- Notification delay became customer-visible before mitigation.

---

## Immediate Mitigation

```text
1. Stop the faulty deployment
2. Restore known-good image
3. Synchronize ArgoCD
4. Verify consumer Pods
5. Verify SQS message processing
6. Allow KEDA to scale consumers
7. Monitor downstream systems
8. Monitor queue drain
```

---

## Why We Did Not Immediately Increase KEDA Max Replicas

The queue had thousands of messages, but increasing:

```text
maxReplicaCount
```

before restoring the consumer application would not help.

The correct sequence was:

```text
Fix consumer
    ↓
Confirm healthy Pods
    ↓
Confirm message processing
    ↓
Scale safely
    ↓
Drain backlog
```

---

## Corrective Actions

| Action | Owner | Priority |
|---|---|---|
| Validate ECR image before GitOps deployment | Platform | P0 |
| Use image digest promotion | Platform | P0 |
| Add canary deployment | Platform | P1 |
| Keep minimum healthy consumers during rollout | Platform | P1 |
| Add zero-consumer alert | SRE | P1 |
| Lower SQS oldest-message alert threshold | SRE | P1 |
| Add consumer smoke test | Application | P1 |
| Add automated rollback | Platform | P1 |
| Test DLQ/replay process | SRE | P2 |

---

# 40. Five Whys

### Why did customers receive notifications late?

Because notification events remained in SQS.

### Why did messages remain in SQS?

No consumer Pods were successfully processing them.

### Why were there no healthy consumer Pods?

The new deployment referenced an invalid ECR image.

### Why was the invalid image deployed?

GitOps accepted the image reference without validating that the image existed.

### Why wasn't this caught before production?

The CI/CD promotion process did not enforce an immutable image promotion and ECR existence check.

---

# 41. Final Root Cause Statement

> The SQS backlog incident was caused by a production deployment of `notification-service` that referenced a non-existent ECR image. The resulting ImagePullBackOff reduced consumer capacity to zero, causing SQS backlog and increasing message age. The backlog was safely drained after restoring the known-good image and allowing controlled KEDA scaling.

---

# 42. Prevention Architecture

The improved deployment flow becomes:

```text
Developer
    ↓
Git PR
    ↓
CI
    ↓
Build
    ↓
Test
    ↓
SonarQube
    ↓
Trivy
    ↓
Docker Build
    ↓
ECR Push
    ↓
Verify Image Exists
    ↓
Capture Image Digest
    ↓
GitOps Update
    ↓
ArgoCD
    ↓
Canary
    ↓
Smoke Test
    ↓
Full Consumer Deployment
```

For SQS:

```text
SQS
 ↓
KEDA
 ↓
Consumer Pods
 ↓
Downstream Capacity
 ↓
Controlled Backlog Drain
```

---

# 43. Monitoring Dashboard

The SQS dashboard should show:

```text
┌────────────────────────────────────────────┐
│             SQS OPERATIONS                 │
├────────────────────────────────────────────┤
│ Queue Depth                                │
│ Oldest Message Age                        │
│ Messages Sent/sec                         │
│ Messages Deleted/sec                      │
│ Messages Received/sec                     │
│ NotVisible Messages                       │
│ DLQ Messages                              │
├────────────────────────────────────────────┤
│ Consumer Pods                              │
│ KEDA Desired Replicas                      │
│ HPA Current Replicas                       │
│ Consumer Processing Latency                │
│ Consumer Error Rate                        │
├────────────────────────────────────────────┤
│ RDS Connections                            │
│ RDS CPU                                    │
│ Redis Latency                              │
│ External API Errors                        │
└────────────────────────────────────────────┘
```

The important point is that the SQS dashboard should include **downstream dependency capacity**, not just queue depth.

---

# 44. Alerts

Recommended alert categories:

### Queue

```text
Queue depth increasing
Oldest message age high
DLQ messages increasing
```

### Consumers

```text
Consumer replicas = 0
Consumer CrashLoopBackOff
Consumer processing errors
Consumer processing latency high
```

### KEDA

```text
ScaledObject not ready
Authentication failure
Scaler error
Unexpected max replicas
```

### Dependencies

```text
RDS connections high
RDS latency high
Redis latency high
External provider 429/5xx
```

---

# 45. Runbook

Create:

```text
runbooks/sqs-backlog.md
```

Recommended sections:

```text
1. Alert
2. Queue identification
3. Queue depth
4. Oldest message age
5. Producer rate
6. Consumer health
7. KEDA health
8. Consumer logs
9. DLQ
10. Visibility timeout
11. IAM
12. Network
13. Downstream dependencies
14. Safe scaling
15. Backlog drain
16. Validation
17. RCA
```

---

# 46. Production Checklist

Before closing an SQS incident:

### Queue

- [ ] Queue depth normalized
- [ ] Oldest message age normalized
- [ ] DLQ checked
- [ ] No unexpected delayed messages

### Consumers

- [ ] Consumer Pods healthy
- [ ] Desired replicas stable
- [ ] KEDA healthy
- [ ] Processing latency normal
- [ ] Error rate normal

### Dependencies

- [ ] RDS healthy
- [ ] Redis healthy
- [ ] External APIs healthy
- [ ] No connection exhaustion

### Data

- [ ] No lost messages
- [ ] Duplicate processing investigated
- [ ] Idempotency verified
- [ ] DLQ messages investigated

### Deployment

- [ ] GitOps state correct
- [ ] ArgoCD synchronized
- [ ] Image digest verified
- [ ] Rollout completed

### Incident

- [ ] Timeline captured
- [ ] Root cause identified
- [ ] Amplification mechanism identified
- [ ] Corrective actions created
- [ ] Owners assigned

---

# 47. Key Production Lessons

### Lesson 1

> **SQS backlog is a symptom, not necessarily the root cause.**

### Lesson 2

> **Do not blindly increase consumer replicas.**

### Lesson 3

> **KEDA scaling must respect downstream capacity.**

### Lesson 4

> **Queue age can be more meaningful than queue depth.**

### Lesson 5

> **At-least-once delivery requires idempotent consumers.**

### Lesson 6

> **Visibility timeout must match realistic processing behavior.**

### Lesson 7

> **DLQ is an operational signal, not a trash bin.**

### Lesson 8

> **Backlog recovery must be controlled.**

### Lesson 9

> **A consumer outage can create a backlog without losing messages.**

### Lesson 10

> **The best RCA identifies both the primary failure and the amplification mechanism.**

---

# 48. Interview Explanation

If an interviewer asks:

> **"How would you troubleshoot an SQS backlog in production?"**

Answer:

> "First I would confirm the queue depth and, more importantly, the age of the oldest message. Then I would determine whether the issue is on the producer side, consumer side, or downstream dependency side. I would check consumer Pod health, KEDA/HPA status, processing latency, application logs, DLQ, visibility timeout, IAM and network connectivity. If consumers are healthy but processing is slow, I would investigate dependencies such as RDS, Redis or external APIs. I would not blindly increase replicas because consumer concurrency can overload downstream systems. Once the root cause is fixed, I would scale consumers within the safe capacity limit, monitor the backlog drain rate, verify idempotency and DLQ behavior, and finally document the incident with an RCA and preventive actions."

---

# 49. Final Architecture

```text
                         ┌──────────────┐
                         │   Producer   │
                         │ order-service│
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │     SQS      │
                         │    Queue     │
                         └──────┬───────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
        Queue Metrics                       DLQ
                │
                ▼
              KEDA
                │
                ▼
              HPA
                │
                ▼
        ┌───────────────┐
        │ EKS Consumers │
        │   Pods        │
        └───────┬───────┘
                │
       ┌────────┼─────────┐
       ▼        ▼         ▼
      RDS     Redis    External API
       │        │         │
       └────────┼─────────┘
                │
                ▼
          Prometheus
                │
                ▼
             Grafana
                │
                ▼
           Alertmanager
                │
                ▼
           Incident Team
```

**Phase 18 progress:**

```text
143 CrashLoopBackOff       ✅
144 OOMKilled              ✅
145 CPU Saturation         ✅
146 Memory Saturation      ✅
147 Disk Full              ✅
148 ImagePullBackOff       ✅
149 ALB 5xx                ✅
150 Service Connectivity   ✅
151 DNS Failure            ✅
152 RDS Connection Exhaustion ✅
153 Redis Failure          ✅
154 SQS Backlog            ✅
```

**Next chapter: `155 — Kafka Consumer Lag Production Incident`.**