# 120 — Alertmanager Production Alerts

> **Goal:** Build a production-grade alerting system for the 21-microservice E-Commerce Platform using Prometheus rules, Alertmanager, routing, grouping, inhibition, silences, escalation, and SLO-based alerts.

---

# 1. Objective

In Chapter 118 we built Prometheus.

In Chapter 119 we built Grafana dashboards.

Now we need the system that **actively tells engineers when something requires attention**.

The production flow becomes:

```text
Applications
     │
     ▼
Prometheus
     │
     ▼
PrometheusRule
     │
     ▼
Alert
     │
     ▼
Alertmanager
     │
     ├── Slack
     ├── PagerDuty
     ├── Email
     └── Other notification channels
```

The key distinction is:

```text
Grafana
    = "Let me investigate"

Alertmanager
    = "Something requires attention"
```

---

# 2. Why Alerting Is Different From Monitoring

Monitoring tells us:

```text
What is happening?
```

Alerting tells us:

```text
What requires action?
```

Example:

```text
CPU = 75%
```

doesn't automatically mean there is an incident.

But:

```text
CPU = 95%
+
request latency increasing
+
error rate increasing
```

may justify an alert.

Therefore:

> **Do not alert on every metric that changes. Alert on conditions that require human action.**

---

# 3. Alert Architecture

Our production architecture:

```text
                   ┌─────────────────┐
                   │ 21 Microservices│
                   └────────┬────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │ Prometheus  │
                     └──────┬──────┘
                            │
                     PrometheusRule
                            │
                            ▼
                     ┌─────────────┐
                     │ Alertmanager│
                     └──────┬──────┘
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
           Slack         PagerDuty       Email
```

Alertmanager handles:

```text
Grouping
Routing
Deduplication
Inhibition
Silencing
Notification delivery
```

---

# 4. Alert Lifecycle

A typical alert goes through:

```text
Metric
  ↓
PromQL condition
  ↓
Pending
  ↓
Firing
  ↓
Alertmanager
  ↓
Routing
  ↓
Notification
  ↓
Engineer
  ↓
Investigation
  ↓
Condition clears
  ↓
Resolved notification
```

For example:

```text
Error rate > 5%
      │
      ▼
Pending for 5 minutes
      │
      ▼
Firing
      │
      ▼
PagerDuty
      │
      ▼
On-call engineer
```

---

# 5. Pending vs Firing

Suppose:

```yaml
expr: error_rate > 5%
for: 5m
```

The condition must remain true for five minutes.

Conceptually:

```text
Error %
  10 |          ┌──────
   8 |          │
   6 |          │
   5 |----------┘
   4 |
     +------------------
       0  1  2  3  4  5
                   minutes
```

This prevents a one-second spike from waking up the on-call engineer.

---

# 6. Alert Severity

Use a small, consistent severity model.

Recommended:

```text
critical
warning
info
```

### Critical

Requires immediate action.

Examples:

```text
Production availability severely degraded
RDS unavailable
Multiple critical services down
Large Kafka lag affecting order processing
```

### Warning

Requires investigation but may not require immediate paging.

Examples:

```text
Disk > 75%
CPU sustained > 80%
Queue backlog increasing
Certificate approaching expiration
```

### Info

Useful operational information.

Examples:

```text
Deployment completed
Node entered maintenance
Scaling event
```

Avoid creating hundreds of informational notifications.

---

# 7. Alert Ownership

Every production alert should have an owner.

Example:

```yaml
labels:
  team: payments
```

or:

```yaml
labels:
  team: platform
```

Possible teams:

```text
platform
devops
sre
backend
security
database
```

Example:

```yaml
labels:
  severity: critical
  team: payments
  service: payment-service
```

Now Alertmanager can route the alert appropriately.

---

# 8. Basic PrometheusRule

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ecommerce-application-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:

    - name: ecommerce.application

      rules:

        - alert: OrderServiceHighErrorRate

          expr: |
            (
              sum(rate(http_server_requests_seconds_count{
                application="order-service",
                status=~"5.."
              }[5m]))
              /
              sum(rate(http_server_requests_seconds_count{
                application="order-service"
              }[5m]))
            ) > 0.05

          for: 5m

          labels:
            severity: critical
            team: backend
            service: order-service

          annotations:
            summary: "Order service error rate is high"
            description: |
              Order service has a 5xx error rate above 5%
              for more than 5 minutes.
```

---

# 9. Why `for:` Matters

Without:

```yaml
for: 5m
```

a temporary spike could trigger:

```text
Alert
 ↓
PagerDuty
 ↓
Engineer wakes up
 ↓
Problem already gone
```

With:

```yaml
for: 5m
```

we require sustained failure.

But don't blindly use long durations.

For a critical availability failure:

```text
30s–2m
```

may be appropriate.

For capacity:

```text
10m–30m
```

may be more appropriate.

The correct duration depends on the failure mode and business impact.

---

# 10. Alert Annotations

Annotations explain the alert.

Useful fields:

```yaml
annotations:
  summary: "Order service error rate is high"

  description: |
    The order service has exceeded the configured
    5xx error threshold for 5 minutes.

  runbook_url: "..."

  dashboard_url: "..."
```

A good alert should tell the engineer:

```text
What happened?
Where?
How serious?
What should I look at next?
```

---

# 11. Runbooks

Every important production alert should have a runbook.

Example:

```text
alerts/
└── runbooks/
    ├── order-service-high-error-rate.md
    ├── order-service-down.md
    ├── rds-connection-exhaustion.md
    ├── redis-failure.md
    ├── kafka-consumer-lag.md
    ├── sqs-backlog.md
    └── node-not-ready.md
```

The alert points to the runbook.

Example:

```yaml
annotations:
  runbook_url: "https://github.example.com/platform/runbooks/order-service-high-error-rate"
```

In a real company, use the organization's approved internal documentation system rather than exposing internal URLs publicly.

---

# 12. What a Runbook Contains

Example:

```markdown
# Order Service High Error Rate

## Symptoms

5xx responses exceed 5%.

## First checks

1. Open Order Service Grafana dashboard.
2. Check deployment history.
3. Check pod restarts.
4. Check RDS connections.
5. Check Redis.
6. Check dependent services.

## Commands

kubectl get pods -n ecommerce
kubectl describe pod <pod>
kubectl logs <pod> --previous

## Possible causes

- Bad deployment
- Database connectivity
- Database connection exhaustion
- Dependency failure
- Application bug
- Resource exhaustion

## Rollback

Follow the approved deployment rollback procedure.

## Escalation

Backend team → SRE → platform/database team as appropriate.
```

---

# 13. Alertmanager Configuration

Conceptually:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default

receivers:
  - name: default
```

Then add routing.

Example:

```yaml
route:
  receiver: default

  routes:

    - matchers:
        - severity="critical"
      receiver: pagerduty-critical

    - matchers:
        - severity="warning"
      receiver: slack-warning
```

The exact Alertmanager configuration syntax should be validated against the Alertmanager version deployed in the cluster.

---

# 14. Routing by Team

Suppose:

```text
payment-service
```

fires an alert.

Labels:

```yaml
labels:
  severity: critical
  team: payments
```

Alertmanager can route:

```text
team=payments
        │
        ▼
Payments on-call
```

Similarly:

```text
team=database
        │
        ▼
Database on-call
```

and:

```text
team=platform
        │
        ▼
Platform/SRE on-call
```

---

# 15. Routing Tree

A production routing tree might look like:

```text
Alertmanager
│
├── severity=critical
│      │
│      └── PagerDuty
│
├── severity=warning
│      │
│      └── Slack
│
├── team=database
│      │
│      └── Database on-call
│
├── team=payments
│      │
│      └── Payment on-call
│
└── default
       │
       └── Platform team
```

Routing rules should be designed around the organization's actual on-call model.

---

# 16. Grouping

Imagine 20 pods restart simultaneously.

Without grouping:

```text
Pod 1 failed
Pod 2 failed
Pod 3 failed
Pod 4 failed
...
Pod 20 failed
```

The engineer could receive 20 notifications.

Instead:

```text
20 pod failures
      ↓
Alertmanager
      ↓
Grouped notification
      ↓
"20 instances affected"
```

This is one of Alertmanager's most important functions.

---

# 17. Group By

Example:

```yaml
route:
  group_by:
    - alertname
    - cluster
    - namespace
```

This can group related alerts.

For example:

```text
alertname=KubernetesPodCrashLooping
cluster=ecommerce-prod
namespace=ecommerce
```

becomes one group rather than one notification per pod.

Choose grouping keys carefully.

Too broad:

```text
group_by:
  - severity
```

could hide useful context.

Too narrow:

```text
group_by:
  - pod
  - instance
  - container
  - request
```

can create notification floods.

---

# 18. `group_wait`

Example:

```yaml
group_wait: 30s
```

When the first alert arrives, Alertmanager waits briefly for related alerts before sending the initial notification.

Useful scenario:

```text
Deployment
   ↓
10 pods restart
   ↓
10 alerts
   ↓
group_wait
   ↓
single grouped notification
```

---

# 19. `group_interval`

Example:

```yaml
group_interval: 5m
```

After a group has already been notified, Alertmanager waits before sending updates for newly added alerts in the same group.

---

# 20. `repeat_interval`

Example:

```yaml
repeat_interval: 4h
```

If the alert remains active, Alertmanager can repeat the notification after the configured interval.

Be careful.

A broken alert with:

```text
repeat every 5 minutes
```

can create alert fatigue very quickly.

---

# 21. Alert Fatigue

One of the biggest problems with poorly designed monitoring is:

```text
Too many alerts
       ↓
Engineer ignores alerts
       ↓
Important alert gets missed
```

Therefore:

> **An alert should represent a condition that requires action.**

Bad:

```text
CPU > 60%
```

Better:

```text
CPU > 90%
AND
sustained for 15 minutes
AND
service is experiencing elevated latency
```

Even then, validate whether paging is actually required.

---

# 22. Symptom vs Cause Alerts

Consider:

```text
RDS CPU = 90%
```

and:

```text
Checkout error rate = 8%
```

The second is closer to the customer-facing symptom.

You may still want the RDS CPU alert as a warning.

But the customer-impacting alert may deserve higher urgency.

---

# 23. Kubernetes Alerts

Important alerts:

```text
NodeNotReady
PodCrashLooping
PodNotReady
PodPending
DeploymentReplicasMismatch
ContainerOOMKilled
ContainerRestarting
PersistentVolumeSpaceLow
NodeDiskPressure
NodeMemoryPressure
```

Example:

```yaml
- alert: KubernetesNodeNotReady

  expr: |
    kube_node_status_condition{
      condition="Ready",
      status="true"
    } == 0

  for: 5m

  labels:
    severity: critical
    team: platform

  annotations:
    summary: "Kubernetes node is not ready"
```

---

# 24. Pod CrashLoop Alert

Example:

```yaml
- alert: KubernetesPodCrashLooping

  expr: |
    increase(
      kube_pod_container_status_restarts_total[15m]
    ) > 3

  for: 5m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Pod is restarting repeatedly"
```

In production, tune this around actual workload behavior so legitimate restart patterns don't page unnecessarily.

---

# 25. OOMKilled Alert

Detect containers that were terminated because of memory pressure.

Example metric:

```promql
kube_pod_container_status_last_terminated_reason{
  reason="OOMKilled"
}
```

This should usually trigger investigation rather than simply restarting the workload repeatedly.

Investigate:

```text
Memory limit
Memory request
JVM heap
Application leak
Traffic increase
Cache growth
Node pressure
```

---

# 26. Application Availability Alert

For a service:

```promql
sum(rate(http_server_requests_seconds_count[5m]))
```

versus successful requests.

Example availability calculation:

```promql
(
  1 -
  (
    sum(rate(http_server_requests_seconds_count{
      application="order-service",
      status=~"5.."
    }[5m]))
    /
    sum(rate(http_server_requests_seconds_count{
      application="order-service"
    }[5m]))
  )
)
```

The exact query should match the application's metric labels and semantics.

---

# 27. High Error Rate Alert

Example:

```yaml
- alert: OrderServiceHighErrorRate

  expr: |
    (
      sum(rate(http_server_requests_seconds_count{
        application="order-service",
        status=~"5.."
      }[5m]))
      /
      sum(rate(http_server_requests_seconds_count{
        application="order-service"
      }[5m]))
    ) > 0.05

  for: 5m

  labels:
    severity: critical
    team: backend
    service: order-service
```

---

# 28. High Latency Alert

Example:

```promql
histogram_quantile(
  0.95,
  sum by(le) (
    rate(
      http_server_requests_seconds_bucket{
        application="order-service"
      }[5m]
    )
  )
) > 1
```

This means:

```text
P95 latency > 1 second
```

for the configured evaluation period.

Don't choose arbitrary thresholds.

Base latency objectives on:

```text
SLO
business requirement
baseline
dependency behavior
user experience
```

---

# 29. RDS Alerts

Important alerts:

```text
High CPU
Low Free Storage
High Connections
Low Freeable Memory
High Read Latency
High Write Latency
Replication Lag
Database unavailable
```

Example concept:

```text
RDS Connections
       │
       ▼
80%
       │
       ▼
Warning

95%
       │
       ▼
Critical
```

Thresholds should reflect actual connection capacity and workload characteristics.

---

# 30. RDS Connection Exhaustion

From Chapter 108:

```text
Pods × connection pool
```

must be considered.

If:

```text
order-service
10 pods × 20
```

then:

```text
200 possible connections
```

If RDS capacity is:

```text
300
```

and other services use:

```text
150
```

you have a capacity problem.

An alert should be based on actual connection headroom, not simply a generic percentage.

---

# 31. Redis Alerts

Monitor:

```text
Memory utilization
Evictions
Connection count
Replication health
CPU
Latency
Failover
Cache hit ratio
```

Example:

```text
Redis memory > 85%
```

could be:

```text
warning
```

while:

```text
Redis unavailable
```

could be:

```text
critical
```

depending on whether applications have safe fallback behavior.

---

# 32. Kafka Alerts

Critical Kafka conditions:

```text
Offline partitions
Under-replicated partitions
Consumer lag
Broker unavailable
ISR shrink
Disk pressure
```

Example:

```yaml
- alert: KafkaUnderReplicatedPartitions

  expr: |
    kafka_server_replicamanager_underreplicatedpartitions > 0

  for: 5m

  labels:
    severity: critical
    team: platform
```

Metric names depend on the Kafka exporter/monitoring implementation used.

Always validate against the metrics actually exposed by your Kafka deployment.

---

# 33. Kafka Consumer Lag Alert

Example concept:

```promql
kafka_consumer_group_lag > 10000
```

But a fixed lag threshold is often insufficient.

Consider:

```text
Lag = 10,000
```

If producer rate is:

```text
100/sec
```

this could represent significant delay.

But if:

```text
Producer = 50,000/sec
Consumer = 49,900/sec
```

the lag behavior needs different interpretation.

Better alerting can combine:

```text
absolute lag
+
lag growth
+
consumer processing rate
+
business impact
```

---

# 34. SQS Alerts

Important:

```text
ApproximateAgeOfOldestMessage
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
DLQ message count
```

Example:

```text
Oldest message age > 5 minutes
```

may be:

```text
warning
```

while:

```text
DLQ messages increasing
```

could require immediate investigation depending on the queue's purpose.

---

# 35. SQS DLQ Alert

A DLQ should generally not silently accumulate messages.

Example:

```yaml
- alert: SQSDeadLetterMessages

  expr: |
    aws_sqs_approximate_number_of_messages_visible{
      queue_name=~".*-dlq"
    } > 0

  for: 5m

  labels:
    severity: critical
    team: platform
```

The exact metric name depends on how AWS metrics are exported into Prometheus.

---

# 36. SLO-Based Alerting

This is where production alerting becomes more mature.

Instead of:

```text
CPU > 80%
```

you can alert on:

```text
Reliability budget is being consumed too quickly.
```

Example:

```text
SLO = 99.9%
```

If error budget is being consumed rapidly, alert.

---

# 37. Burn Rate

Conceptually:

```text
Error budget
     │
     ▼
Consumption rate
     │
     ▼
Burn rate
```

If:

```text
Burn rate = 1
```

the service is consuming error budget at approximately the planned rate.

If:

```text
Burn rate = 10
```

the budget is being consumed much faster.

A common advanced strategy uses multiple windows:

```text
short window
+
long window
```

to detect both severe incidents and sustained degradation.

---

# 38. Multi-Window Burn Rate

Conceptual model:

```text
Short window
   +
Long window
   ↓
Burn-rate alert
```

For example:

```text
5-minute window
+
1-hour window
```

or:

```text
30-minute window
+
6-hour window
```

The exact thresholds depend on the SLO and desired alert sensitivity.

This avoids paging on a tiny transient spike while still detecting serious incidents quickly.

---

# 39. Alert Inhibition

Suppose an entire Kubernetes node fails.

You might get:

```text
NodeNotReady
PodNotReady
PodCrashLooping
ServiceDown
```

If the root cause is:

```text
NodeNotReady
```

you may not want all downstream alerts to page independently.

Alertmanager can use inhibition.

Conceptually:

```text
NodeNotReady
    │
    ├── suppress lower-level pod alerts
    ├── suppress some workload alerts
    └── keep root-cause alert
```

This reduces noise.

---

# 40. Example Inhibition

Conceptual configuration:

```yaml
inhibit_rules:

  - source_matchers:
      - alertname="KubernetesNodeNotReady"

    target_matchers:
      - alertname=~"KubernetesPodNotReady|KubernetesPodPending"

    equal:
      - cluster
```

Validate matcher syntax against the deployed Alertmanager version.

Do not create overly broad inhibition rules that hide genuine independent failures.

---

# 41. Silences

A silence means:

```text
Don't notify me about this alert for a defined period.
```

Example:

```text
Planned node maintenance
        ↓
Create silence
        ↓
Perform maintenance
        ↓
No unnecessary pages
        ↓
Silence expires
```

A silence should have:

```text
Reason
Owner
Start time
End time
Scope
```

Never create:

```text
silence everything for 7 days
```

as a workaround.

---

# 42. Maintenance Example

Suppose:

```text
RDS maintenance
```

is scheduled.

Before maintenance:

```text
Silence specific expected alerts
```

Then:

```text
Maintenance
   ↓
Monitor manually
   ↓
Validate recovery
   ↓
Silence expires
```

Unexpected alerts should still be able to fire.

---

# 43. Notification Channels

Typical architecture:

```text
Critical
   ↓
PagerDuty
   ↓
On-call engineer

Warning
   ↓
Slack
   ↓
Team channel

Non-urgent
   ↓
Email
```

Do not send every warning to the paging system.

---

# 44. Slack Notification

Example concept:

```text
🚨 CRITICAL

Alert: OrderServiceHighErrorRate

Service: order-service
Environment: prod

Error Rate: 8.2%
Duration: 7 minutes

Summary:
Order service 5xx rate is above threshold.

Runbook:
<internal runbook>

Dashboard:
<internal dashboard>
```

The actual notification template should be standardized across teams.

---

# 45. PagerDuty

Critical alerts can be routed to the on-call system.

Example:

```text
Alertmanager
     │
     ▼
PagerDuty
     │
     ▼
Primary On-Call
     │
     ├── Acknowledge
     ├── Investigate
     └── Escalate
```

If the engineer does not acknowledge within the configured period:

```text
Primary
   ↓
Secondary
   ↓
Escalation team
```

The exact escalation policy belongs to the organization's incident-management process.

---

# 46. Alert Deduplication

Suppose:

```text
order-service
```

has five replicas.

All five expose:

```text
error rate = 10%
```

You don't want:

```text
5 separate pages
```

Alertmanager grouping can produce:

```text
OrderServiceHighErrorRate
5 instances affected
```

This is deduplication at the notification layer.

---

# 47. Alert Labels

Use labels for machine routing:

```yaml
labels:
  severity: critical
  team: backend
  service: order-service
  environment: production
```

Use annotations for human-readable details:

```yaml
annotations:
  summary: "Order service has high error rate"
  description: "5xx rate is above 5% for 5 minutes."
```

Do not put rapidly changing unique values into labels unnecessarily.

---

# 48. Alert Naming Convention

Use consistent names:

```text
OrderServiceHighErrorRate
OrderServiceHighLatency
OrderServiceDown

PaymentServiceHighErrorRate
PaymentServiceProviderTimeout

RedisHighMemory
RedisUnavailable

RDSHighConnections
RDSLowFreeStorage

KafkaConsumerLagHigh
KafkaUnderReplicatedPartitions

SQSBacklogHigh
SQSDeadLetterMessages
```

Readable names are easier to search during incidents.

---

# 49. Alert Repository

Recommended:

```text
gitops/
└── monitoring/
    ├── prometheus-rules/
    │   ├── kubernetes.yaml
    │   ├── applications.yaml
    │   ├── databases.yaml
    │   ├── redis.yaml
    │   ├── kafka.yaml
    │   ├── sqs.yaml
    │   └── slo.yaml
    │
    ├── alertmanager/
    │   ├── values.yaml
    │   └── templates/
    │
    └── runbooks/
        ├── kubernetes/
        ├── applications/
        ├── databases/
        ├── messaging/
        └── incidents/
```

---

# 50. Alert Review Process

A production alert should go through:

```text
Developer/SRE
      ↓
Create alert
      ↓
Test PromQL
      ↓
Test firing condition
      ↓
Test notification
      ↓
Write runbook
      ↓
PR
      ↓
Review
      ↓
Merge
      ↓
ArgoCD
      ↓
Production
```

Never deploy a critical production alert without testing the notification path.

---

# 51. Testing Alerts

You should test:

```text
Normal
Warning
Critical
Resolved
Notification failure
Routing
Grouping
Inhibition
Silence
Escalation
```

Example controlled test:

```text
Deploy a temporary test workload
        ↓
Generate known failure
        ↓
Prometheus evaluates rule
        ↓
Alert becomes Pending
        ↓
Alert becomes Firing
        ↓
Alertmanager receives alert
        ↓
Slack/PagerDuty notification
        ↓
Fix workload
        ↓
Alert resolves
        ↓
Resolved notification
```

Use a controlled non-production environment for initial testing.

---

# 52. Testing a PrometheusRule

Check:

```bash
kubectl get prometheusrule -n monitoring
```

Then:

```bash
kubectl describe prometheusrule \
  ecommerce-application-alerts \
  -n monitoring
```

Check Prometheus targets/rules through the Prometheus UI or API as appropriate.

Also inspect:

```bash
kubectl get pods -n monitoring
```

---

# 53. Checking Alertmanager

```bash
kubectl get pods -n monitoring
```

Look for:

```text
alertmanager
```

Then inspect logs:

```bash
kubectl logs \
  -n monitoring \
  <alertmanager-pod>
```

For configuration problems, look for:

```text
configuration parsing
receiver configuration
routing
authentication
notification delivery
```

---

# 54. Common Alerting Failure

Imagine:

```text
Prometheus
   │
   ├── Rule works
   └── Alert firing
```

but:

```text
PagerDuty
   │
   └── no notification
```

Investigate:

```text
Prometheus
 ↓
Alertmanager
 ↓
Routing
 ↓
Receiver
 ↓
Network
 ↓
Authentication
 ↓
Destination
```

Don't assume:

```text
Alert firing = notification delivered
```

These are separate stages.

---

# 55. Alertmanager Troubleshooting

Check:

```text
1. Is Alertmanager running?
2. Is Prometheus connected to Alertmanager?
3. Is the alert firing?
4. Is it routed to the expected receiver?
5. Is the receiver configured correctly?
6. Is authentication valid?
7. Is outbound connectivity available?
8. Is the notification suppressed by inhibition?
9. Is the alert silenced?
10. Is grouping delaying notification?
```

---

# 56. Real Incident — Order Service

### 10:00

Deployment:

```text
order-service v1.5.0
```

### 10:03

Alert:

```text
OrderServiceHighErrorRate
```

### 10:04

PagerDuty:

```text
CRITICAL
```

### 10:05

Engineer checks Grafana:

```text
5xx ↑
P95 ↑
```

### 10:06

Pod logs:

```text
Database schema mismatch
```

### 10:07

Deployment introduced incompatible schema expectations.

### 10:08

Rollback initiated.

### 10:10

Metrics recover.

### 10:11

Alert resolves.

The alerting system helped connect:

```text
Deployment
   ↓
Error rate
   ↓
PagerDuty
   ↓
Investigation
   ↓
Rollback
```

---

# 57. Real Incident — Kafka Lag

Suppose:

```text
order-service
    ↓
Kafka
    ↓
inventory-consumer
```

Suddenly:

```text
Consumer lag ↑
```

Alert:

```text
KafkaConsumerLagHigh
```

Engineer checks:

```text
Kafka dashboard
      ↓
Consumer lag
      ↓
inventory-service
      ↓
pod CPU
      ↓
pod logs
```

Finds:

```text
Inventory database queries slowed down.
```

Then:

```text
Kafka
 ↓
Inventory
 ↓
RDS
```

The alert points to the affected pipeline.

---

# 58. Real Incident — Node Failure

Suppose one node becomes unavailable.

Without proper grouping:

```text
NodeNotReady
PodNotReady x 15
PodPending x 8
ServiceUnavailable x 4
```

Potentially:

```text
28 notifications
```

With grouping/inhibition:

```text
NodeNotReady
     ↓
Primary notification

Related pod alerts
     ↓
Grouped/suppressed where appropriate
```

The engineer sees the likely root cause first.

---

# 59. Alert Quality Rules

Every production alert should answer:

```text
1. What happened?
2. Where did it happen?
3. How severe is it?
4. Who owns it?
5. Does it require action?
6. What dashboard should I open?
7. What runbook should I follow?
```

If the alert doesn't answer these questions, improve it.

---

# 60. Anti-Patterns

Avoid:

```text
❌ Alert on every metric
❌ Alert on CPU spikes lasting seconds
❌ Page on every warning
❌ No runbooks
❌ No ownership
❌ No grouping
❌ No inhibition
❌ Infinite silences
❌ Hardcoded secrets in Alertmanager config
❌ Public Alertmanager
❌ Public Prometheus
❌ Public Grafana without proper authentication
❌ Different alert naming conventions for every team
```

---

# 61. Production Alert Categories

Our platform should eventually have:

```text
Infrastructure
    │
    ├── Nodes
    ├── EKS
    └── Storage

Application
    │
    ├── Availability
    ├── Errors
    └── Latency

Database
    │
    ├── Connections
    ├── CPU
    └── Storage

Cache
    │
    ├── Redis memory
    └── Failover

Messaging
    │
    ├── Kafka lag
    ├── Kafka replication
    └── SQS backlog

Business
    │
    ├── Order failures
    ├── Payment failures
    └── Inventory failures

Reliability
    │
    ├── SLO
    └── Error budget
```

---

# 62. Alert Severity Matrix

A practical structure:

| Condition                         | Severity         | Typical destination                |
| --------------------------------- | ---------------- | ---------------------------------- |
| Production service unavailable    | Critical         | PagerDuty                          |
| Severe customer-facing error rate | Critical         | PagerDuty                          |
| RDS unavailable                   | Critical         | PagerDuty                          |
| Kafka partition unavailable       | Critical         | PagerDuty                          |
| SQS DLQ growing                   | Critical/Warning | Team + on-call depending on impact |
| Moderate latency degradation      | Warning          | Slack                              |
| Disk approaching threshold        | Warning          | Slack                              |
| CPU sustained high                | Warning          | Slack                              |
| Capacity approaching limit        | Warning          | Slack                              |
| Informational deployment event    | Info             | Optional team channel              |

These are examples; the actual severity should follow the service's documented impact and SLO.

---

# 63. Complete Alert Flow

The final architecture is:

```text
                 ┌──────────────────┐
                 │ 21 Microservices │
                 └────────┬─────────┘
                          │
                          ▼
                    Prometheus
                          │
                          ▼
                  PrometheusRule
                          │
                          ▼
                   Alertmanager
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       Grouping       Inhibition       Silencing
          │
          ▼
       Routing
          │
     ┌────┼─────┐
     ▼    ▼     ▼
  Slack PagerDuty Email
     │    │     │
     ▼    ▼     ▼
  Teams On-call Reports
```

---

# 64. Final Production Checklist

### Prometheus

* [ ] PrometheusRules deployed through GitOps
* [ ] Alert expressions tested
* [ ] `for:` durations configured
* [ ] Alert labels standardized
* [ ] Runbook annotations added

### Alertmanager

* [ ] HA configuration considered
* [ ] Routing configured
* [ ] Grouping configured
* [ ] Inhibition configured
* [ ] Silencing process defined
* [ ] Repeat intervals defined
* [ ] Notification receivers configured
* [ ] Secrets managed securely

### Notifications

* [ ] Critical → on-call
* [ ] Warning → team channel
* [ ] Email where appropriate
* [ ] Escalation policy documented
* [ ] Resolved notifications tested

### Alert quality

* [ ] No unnecessary alerts
* [ ] No high-cardinality alert labels
* [ ] Every critical alert has a runbook
* [ ] Every critical alert has an owner
* [ ] Customer impact considered
* [ ] SLO alerts implemented
* [ ] Alert fatigue reviewed regularly

---

# 65. End-to-End Observability So Far

We now have:

```text
                  ┌─────────────────────┐
                  │    Applications     │
                  │    21 Services      │
                  └──────────┬──────────┘
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
        Metrics           Logs            Traces
             │
             ▼
        Prometheus
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
    Grafana    Alertmanager
       │           │
       │      ┌────┼─────┐
       │      ▼    ▼     ▼
       │    Slack Pager Email
       │
       ├── Business
       ├── Kubernetes
       ├── Services
       ├── RDS
       ├── Redis
       ├── Kafka
       ├── SQS
       ├── SLO
       └── Incidents
```

The next step is to complete the **logging side** of the observability platform.

---

# Next Chapter

## `121-ELK-Production-Implementation.md`

We will implement:

```text
EKS Nodes
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

and cover:

* where Fluent Bit actually runs in production
* DaemonSet architecture
* container log files
* Kubernetes metadata enrichment
* Logstash deployment and pipelines
* Elasticsearch production topology
* index strategy
* index lifecycle/retention
* application JSON logging
* Spring Boot structured logs
* correlation IDs
* namespace/pod/container metadata
* multiline Java stack traces
* log levels
* PII/security considerations
* Elasticsearch storage
* shard/replica planning
* Kibana access control
* troubleshooting missing logs
* high-volume logging
* backpressure
* log loss vs log duplication
* production incident investigation using ELK.
