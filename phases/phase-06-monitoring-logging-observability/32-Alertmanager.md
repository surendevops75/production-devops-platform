# 32 — Alertmanager

> Production-grade alerting architecture for the Enterprise E-Commerce Platform on AWS EKS

---

## 1. Purpose

Prometheus answers:

> **“What is happening?”**

Grafana answers:

> **“How can I visualize and investigate it?”**

Alertmanager answers:

> **“Who should be notified, when, and how?”**

Our production flow is:

```text
Application / Kubernetes / AWS
            │
            ▼
       Prometheus
            │
            ▼
       Alert Rules
            │
            ▼
       Alertmanager
            │
     ┌──────┼────────┐
     ▼      ▼        ▼
   Slack  PagerDuty  Email
            │
            ▼
       On-call Engineer
```

The goal is **not to alert on everything**.

The goal is to alert on conditions that require human action.

---

# 2. Alerting Architecture

Our EKS monitoring architecture:

```text
                       EKS
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
   Applications       Nodes          Kubernetes
        │               │                │
        ▼               ▼                ▼
   /actuator       Node Exporter   kube-state-metrics
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                   Prometheus
                        │
                        ▼
                  Alerting Rules
                        │
                        ▼
                  Alertmanager
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
        Slack       PagerDuty        Email
          │             │              │
          └─────────────┼──────────────┘
                        ▼
                   On-call Team
```

---

# 3. Prometheus vs Alertmanager

This distinction is extremely important.

### Prometheus

Responsible for:

```text
Collect metrics
Store metrics
Evaluate alert rules
Generate alerts
```

### Alertmanager

Responsible for:

```text
Receive alerts
Group alerts
Deduplicate alerts
Silence alerts
Inhibit related alerts
Route alerts
Send notifications
```

Therefore:

```text
Prometheus
    ↓
"Alert condition is true"

Alertmanager
    ↓
"How should this alert be handled?"
```

---

# 4. What Is an Alert?

An alert is essentially a rule that says:

```text
IF condition remains true
THEN generate alert
```

Example:

```promql
rate(http_server_requests_seconds_count{
  status=~"5.."
}[5m]) > 1
```

This could indicate:

```text
More than 1 server error/second
```

But we usually shouldn't immediately page someone.

Add:

```yaml
for: 5m
```

Meaning:

```text
Condition must remain true for 5 minutes
```

This reduces noise from short spikes.

---

# 5. Alert Lifecycle

The lifecycle is:

```text
Normal
  │
  │ condition becomes true
  ▼
Pending
  │
  │ remains true for "for" duration
  ▼
Firing
  │
  ▼
Alertmanager
  │
  ▼
Notification
```

When the condition clears:

```text
Firing
  │
  ▼
Resolved
```

---

# 6. Why `for` Matters

Without:

```yaml
for: 5m
```

a temporary spike can generate:

```text
CPU 91%
   ↓
Alert
   ↓
CPU 78%
   ↓
Resolved
   ↓
CPU 92%
   ↓
Alert
```

This creates alert noise.

With:

```yaml
for: 10m
```

the system requires sustained failure.

This is much more appropriate for many infrastructure alerts.

---

# 7. Alert Severity

A production platform should distinguish severity.

Example:

```text
critical
warning
info
```

---

## Critical

Human intervention is required immediately.

Examples:

```text
Production availability severely degraded
Database unavailable
Entire service down
Massive error rate
Critical SLO burn
```

Typical destination:

```text
PagerDuty / on-call
```

---

## Warning

Something requires investigation but may not need immediate paging.

Examples:

```text
CPU consistently high
Memory trending high
Queue backlog increasing
Disk utilization high
```

Typical destination:

```text
Slack
```

---

## Info

Useful operational information.

Examples:

```text
Deployment completed
Node added
Pod rescheduled
```

Usually:

```text
Dashboard/event
```

rather than paging.

---

# 8. Alert Quality

A good alert should answer:

```text
What is broken?
Where is it broken?
How severe is it?
Why should I care?
What should I do?
```

Bad:

```text
CPU High
```

Better:

```text
order-service production CPU saturation
```

Best:

```text
order-service in production has sustained CPU saturation
above 90% for 10 minutes and may be unable to handle
current traffic.
```

---

# 9. Alert Rule Structure

Example:

```yaml
groups:
  - name: ecommerce-application-alerts
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
          service: order-service
          environment: production

        annotations:
          summary: "Order service error rate is high"
          description: >
            Order service has had more than 5% server errors
            for the last 5 minutes.
```

---

# 10. Labels vs Annotations

This distinction matters.

### Labels

Used for:

```text
Grouping
Routing
Filtering
Deduplication
```

Example:

```yaml
labels:
  severity: critical
  service: order-service
  environment: production
```

### Annotations

Used for human-readable information.

Example:

```yaml
annotations:
  summary: "Order service error rate is high"
  description: "More than 5% of requests are returning 5xx."
```

---

# 11. Alert Routing

Alertmanager receives alerts from Prometheus.

Example:

```text
                 Alertmanager
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    critical       warning         info
        │             │
        ▼             ▼
   PagerDuty       Slack
```

Routing can use labels.

For example:

```yaml
route:
  receiver: default

  routes:
    - matchers:
        - severity="critical"
      receiver: pagerduty

    - matchers:
        - severity="warning"
      receiver: slack
```

---

# 12. Team-Based Routing

Our organization has:

```text
backend-team
platform-team
devops-team
security-team
sre-team
```

We can route based on ownership.

Example:

```yaml
labels:
  team: backend
  service: order-service
```

Then:

```text
team=backend
      ↓
backend Slack channel
```

and:

```text
team=platform
      ↓
platform Slack channel
```

---

# 13. Service Ownership

Every production alert should ideally have an owner.

Example:

```text
order-service
     ↓
backend-team
     ↓
SRE escalation
```

For infrastructure:

```text
EKS
 ↓
platform-team
 ↓
SRE escalation
```

For security:

```text
Security finding
 ↓
security-team
```

---

# 14. Grouping

Imagine 30 pods fail because a node disappears.

Without grouping:

```text
PodDown
PodDown
PodDown
PodDown
PodDown
...
```

The engineer gets flooded.

With grouping:

```text
EKS Node Failure
  ├── pod-1
  ├── pod-2
  ├── pod-3
  └── pod-4
```

One notification can represent the incident.

---

# 15. Group By

Example:

```yaml
route:
  group_by:
    - alertname
    - cluster
    - environment
```

Alerts with the same grouping labels are combined.

The exact grouping strategy should match the organization's incident model.

---

# 16. Deduplication

Suppose Prometheus generates:

```text
OrderServiceDown
```

every evaluation interval.

Alertmanager should not send:

```text
10:00 Alert
10:01 Alert
10:02 Alert
10:03 Alert
10:04 Alert
```

as five separate incidents.

Instead:

```text
10:00 → notification
10:01 → same alert maintained
10:02 → no duplicate notification
```

This is one of Alertmanager's major responsibilities.

---

# 17. Group Wait

Example:

```yaml
group_wait: 30s
```

When the first alert arrives:

```text
Alert A
  ↓
wait 30 seconds
```

If related alerts arrive during that period:

```text
Alert A
Alert B
Alert C
```

they can be sent together.

---

# 18. Group Interval

Example:

```yaml
group_interval: 5m
```

After sending a group, Alertmanager waits before sending another notification for newly added alerts in the same group.

---

# 19. Repeat Interval

Example:

```yaml
repeat_interval: 4h
```

If an alert remains firing:

```text
10:00 → notification
14:00 → reminder
18:00 → reminder
```

This prevents both:

```text
constant spam
```

and:

```text
forgotten unresolved incidents
```

---

# 20. Silences

Sometimes an alert is expected.

Example:

```text
Database maintenance
```

During planned maintenance:

```text
RDSHighCPU
RDSUnavailable
ConnectionErrors
```

may be expected.

Create a silence:

```text
Start
 ↓
Maintenance
 ↓
Silence alerts
 ↓
Maintenance ends
 ↓
Silence expires
```

Never leave permanent broad silences.

---

# 21. Silence Best Practice

A silence should have:

```text
Who
Why
What
When
Expiration
```

Example:

```text
Match:
  alertname=RDSMaintenance

Comment:
  Planned PostgreSQL maintenance

Created by:
  devops-team

Expires:
  20:00 IST
```

---

# 22. Inhibition

Inhibition suppresses secondary alerts when a higher-level alert is already firing.

Example:

```text
NodeDown
   ↓
PodDown
PodDown
PodDown
PodDown
```

If the node itself is down, many pod alerts may not be useful.

We can configure:

```text
NodeDown
   ↓
inhibit
   ↓
PodNotReady
```

The exact inhibition rules must be carefully designed.

---

# 23. Why Inhibition Matters

Without inhibition:

```text
Node failure
 ↓
20 pod alerts
 ↓
20 notifications
```

With appropriate inhibition:

```text
Node failure
 ↓
NodeDown
 ↓
one primary incident
```

This dramatically improves signal-to-noise ratio.

---

# 24. Alertmanager HA

Production should not depend on a single Alertmanager instance.

Architecture:

```text
                Prometheus
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
   Alertmanager-1       Alertmanager-2
          │                   │
          └─────────┬─────────┘
                    ▼
              Notification
```

Alertmanager instances can form a cluster for alert state coordination.

Run across separate nodes/AZs where practical.

---

# 25. Kubernetes Deployment

For our EKS cluster:

```text
namespace: monitoring
```

Example:

```bash
kubectl get pods -n monitoring
```

Expected:

```text
grafana
prometheus
alertmanager
kube-state-metrics
node-exporter
```

The exact names depend on the installed chart/release.

---

# 26. kube-prometheus-stack

A common Kubernetes approach is to deploy the Prometheus Operator ecosystem through the `kube-prometheus-stack` Helm chart.

It can provide:

```text
Prometheus
Alertmanager
Grafana
Prometheus Operator
kube-state-metrics
Node Exporter
ServiceMonitors
PodMonitors
```

In our architecture, we conceptually separate the responsibilities even if they are installed by the same stack.

---

# 27. Alert Rules as Code

Production alert rules belong in Git.

Example:

```text
gitops/
└── monitoring/
    ├── prometheus/
    │   └── rules/
    │       ├── applications.yaml
    │       ├── kubernetes.yaml
    │       ├── nodes.yaml
    │       ├── databases.yaml
    │       └── queues.yaml
    │
    └── alertmanager/
        └── values.yaml
```

Then:

```text
Git
 ↓
PR
 ↓
Review
 ↓
ArgoCD
 ↓
Prometheus / Alertmanager
```

---

# 28. Application Alerts

For our 21 microservices, important alerts include:

```text
HighErrorRate
HighLatency
ServiceDown
HighRestartRate
PodCrashLooping
HighCPU
HighMemory
HPAAtMaximum
```

But not every service needs a completely different alert definition.

Use labels and reusable recording rules where possible.

---

# 29. Service Down

Example:

```promql
up{
  job="order-service"
} == 0
```

Alert:

```yaml
- alert: OrderServiceDown
  expr: up{job="order-service"} == 0
  for: 5m

  labels:
    severity: critical
    service: order-service
```

Be careful:

A single pod being down does not necessarily mean the **service** is down.

For a Deployment with 10 replicas:

```text
1 pod down
9 pods healthy
```

is very different from:

```text
0 pods healthy
```

---

# 30. Availability Alert

Better service-level thinking:

```text
healthy replicas
----------------
desired replicas
```

Example:

```promql
(
  kube_deployment_status_replicas_available{
    deployment="order-service"
  }
/
  kube_deployment_spec_replicas{
    deployment="order-service"
  }
) < 0.5
```

This means less than half the desired replicas are available.

---

# 31. High Error Rate

Example:

```promql
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
```

Alert:

```text
Error rate > 5%
for 5 minutes
```

The threshold should be based on service requirements and SLOs.

---

# 32. High Latency

Example p95:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_server_requests_seconds_bucket{
      application="order-service"
    }[5m])
  )
) > 1
```

Meaning:

```text
p95 latency > 1 second
```

for a sustained period.

Again, the correct threshold is service-specific.

---

# 33. CPU Alert

Example:

```promql
(
  sum by (pod) (
    rate(container_cpu_usage_seconds_total{
      namespace="production",
      container!="POD",
      container!=""
    }[5m])
  )
)
>
(
  sum by (pod) (
    kube_pod_container_resource_limits{
      namespace="production",
      resource="cpu",
      unit="core"
    }
  ) * 0.9
)
```

But CPU alerts should be designed carefully because:

```text
High CPU
```

is not automatically a customer-impacting incident.

If HPA is functioning correctly and latency remains healthy:

```text
high CPU ≠ emergency
```

---

# 34. Memory Alert

Memory is often more dangerous because hitting limits can result in:

```text
OOMKilled
```

Monitor:

```text
Working set
Memory limit
OOMKilled events
Restart count
```

Example:

```promql
container_memory_working_set_bytes
/
container_spec_memory_limit_bytes
> 0.9
```

for a sustained period.

---

# 35. CrashLoopBackOff

Use kube-state-metrics/container metrics to detect repeated restarts.

Example concept:

```promql
increase(
  kube_pod_container_status_restarts_total{
    namespace="production"
  }[15m]
) > 5
```

This identifies containers restarting repeatedly.

---

# 36. HPA Maximum Alert

An important scaling alert:

```text
Current replicas = max replicas
```

Example:

```text
HPA:
minReplicas = 3
maxReplicas = 10

Current:
10
```

If traffic continues increasing:

```text
HPA cannot scale further
```

This can be more operationally important than CPU itself.

---

# 37. Node Alerts

Important node alerts:

```text
NodeNotReady
NodeCPUHigh
NodeMemoryPressure
NodeDiskPressure
NodeFilesystemFull
NodeNetworkErrors
```

Example:

```text
NodeNotReady
   ↓
Pods rescheduled
   ↓
Capacity decreases
```

---

# 38. Kubernetes Capacity Alerts

Monitor:

```text
CPU requests / allocatable CPU
Memory requests / allocatable memory
Pod capacity
DaemonSet overhead
IP availability
```

EKS VPC CNI adds an important dimension:

```text
Available Pod IPs
```

A node can have:

```text
CPU available
Memory available
```

but still be unable to schedule another Pod because:

```text
ENI/IP capacity exhausted
```

---

# 39. EBS Alerts

For workloads using EBS:

```text
Volume health
Volume throughput
Volume IOPS
Filesystem usage
PVC capacity
```

Remember:

```text
PVC full
```

can become:

```text
Application failure
```

before CPU or memory shows anything unusual.

---

# 40. RDS Alerts

For our PostgreSQL RDS:

```text
CPU high
Free storage low
Memory pressure
Connections high
Read latency high
Write latency high
Replica lag
```

Connection exhaustion is particularly important.

Example:

```text
Pods
 ×
Hikari pool size
 =
potential DB connections
```

If:

```text
20 pods × 30 connections
```

we could potentially request:

```text
600 connections
```

from RDS.

Therefore:

```text
Application scaling
        ↓
DB connection scaling
```

must be considered together.

---

# 41. Redis Alerts

Important:

```text
Memory usage
Evictions
Cache hit ratio
Connections
CPU
Replication health
```

Example incident:

```text
Redis memory ↑
 ↓
Evictions ↑
 ↓
Cache hit ratio ↓
 ↓
RDS traffic ↑
 ↓
RDS connections ↑
 ↓
Application latency ↑
```

A good alerting system should detect the early signals.

---

# 42. SQS Alerts

For our queues:

```text
ApproximateAgeOfOldestMessage
ApproximateNumberOfMessagesVisible
```

are especially important.

Example:

```text
Oldest message age > 5 minutes
```

for a critical order-processing queue could trigger a warning or critical alert depending on business requirements.

---

# 43. Queue Backlog

Suppose:

```text
Messages:
100
200
500
1,000
5,000
```

The queue is growing.

But:

```text
Queue size alone
```

doesn't tell us whether the situation is acceptable.

We should also evaluate:

```text
message age
consumer throughput
producer rate
consumer errors
processing latency
```

---

# 44. SLO-Based Alerting

This is where production alerting becomes much better.

Suppose:

```text
Order API SLO = 99.9%
```

The question isn't:

```text
CPU > 80?
```

The question is:

```text
Are we consuming our error budget too quickly?
```

---

# 45. Error Budget

For:

```text
99.9% availability
```

allowed failure is:

```text
0.1%
```

That becomes the error budget.

If the service is burning that budget rapidly:

```text
Alert
```

This aligns engineering alerts with customer impact.

---

# 46. Burn Rate

Conceptually:

```text
Burn Rate =
current error consumption
/
allowed error consumption
```

If:

```text
burn rate = 1x
```

we are consuming the budget at the expected rate.

If:

```text
burn rate = 20x
```

we are consuming it extremely quickly.

That deserves urgent investigation.

---

# 47. Multi-Window Burn Rate

A mature SRE setup can use:

```text
short window
+
long window
```

For example:

```text
5-minute window
+
1-hour window
```

This avoids paging because of a tiny short-lived anomaly while still detecting serious incidents quickly.

---

# 48. Alert Fatigue

One of the biggest production monitoring failures is:

```text
Too many alerts
```

Example:

```text
Engineer receives 150 alerts/day
```

Eventually:

```text
Alert
 ↓
Ignore
```

That is dangerous.

The correct goal is:

```text
Fewer
+
Actionable
+
High-confidence
alerts
```

---

# 49. Alert Should Have a Runbook

Every important alert should point to a runbook.

Example:

```yaml
annotations:
  summary: "Order service high error rate"

  runbook_url: "internal-runbook-reference"

  description: >
    Order service is returning more than 5% 5xx
    responses for the last 5 minutes.
```

The runbook should explain:

```text
1. Check Grafana
2. Check pods
3. Check logs
4. Check RDS
5. Check recent deployment
6. Roll back if necessary
```

---

# 50. Alert Example — Production Incident

Suppose:

```text
payment-service
```

starts returning 20% errors.

Prometheus:

```text
5xx > threshold
```

generates:

```text
PaymentServiceHighErrorRate
```

Alertmanager:

```text
severity=critical
team=backend
environment=production
```

routes:

```text
PagerDuty
+
backend Slack channel
```

Engineer receives:

```text
CRITICAL

Payment service error rate > 20%
Duration: 5 minutes
Environment: production

Runbook: Payment Service High Error Rate
```

Engineer opens Grafana:

```text
Error rate ↑
Latency ↑
Pods healthy
```

Then logs:

```text
Elasticsearch
```

show:

```text
Database connection timeout
```

Then RDS dashboard:

```text
DatabaseConnections ↑
```

Now we have a causal investigation path.

---

# 51. Alertmanager Configuration Concept

A simplified configuration:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by:
    - alertname
    - cluster
    - environment

  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  receiver: default

  routes:

    - matchers:
        - severity="critical"
      receiver: pagerduty

    - matchers:
        - severity="warning"
      receiver: slack

receivers:

  - name: default

  - name: slack
    # webhook configuration managed securely

  - name: pagerduty
    # integration configuration managed securely
```

This is intentionally simplified.

Production credentials must never be committed directly.

---

# 52. Secret Management

Notification credentials are secrets.

Examples:

```text
Slack webhook
PagerDuty integration key
SMTP credentials
```

Do not commit:

```yaml
webhook_url: https://...
```

to Git.

Use:

```text
Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Alertmanager
```

This follows the secret architecture from chapter 26.

---

# 53. Alertmanager and GitOps

Our deployment flow:

```text
Developer
   │
   ▼
Git commit
   │
   ▼
GitOps PR
   │
   ▼
Review
   │
   ▼
ArgoCD
   │
   ▼
Alertmanager
```

Configuration changes are auditable.

Example:

```text
Commit abc123
 ↓
Changed RDS alert threshold
 ↓
PR approved
 ↓
ArgoCD sync
 ↓
New alert rule active
```

---

# 54. Alert Testing

Never assume an alert works because YAML is valid.

Test:

```text
Rule syntax
Condition
Notification routing
Silence
Inhibition
Escalation
Resolution
```

A practical test:

```text
Generate controlled failure
        ↓
Prometheus evaluates
        ↓
Alert fires
        ↓
Alertmanager receives
        ↓
Notification arrives
        ↓
Recovery
        ↓
Resolved notification
```

---

# 55. Alertmanager Troubleshooting

## Problem — Alert not firing

Check:

```text
Prometheus rule
PromQL expression
Labels
for duration
Prometheus targets
Metric availability
```

First test the expression directly in Prometheus.

---

# 56. Alert Exists in Prometheus but Not Alertmanager

Check:

```text
Prometheus Alertmanager URL
Network connectivity
Alertmanager service
Prometheus configuration
```

Example:

```bash
kubectl get svc -n monitoring
```

Then inspect Prometheus configuration/status.

---

# 57. Alertmanager Received Alert but No Notification

Check:

```text
Route matching
Receiver
Secrets
Webhook
Integration
Silence
Inhibition
```

A common mistake:

```text
severity=critical
```

but routing expects:

```text
severity="Critical"
```

Labels must match exactly according to the configured matcher semantics.

---

# 58. Too Many Notifications

Investigate:

```text
group_by
group_wait
group_interval
repeat_interval
deduplication
inhibition
alert rule design
```

Don't simply increase:

```text
repeat_interval
```

without understanding why the alerts are noisy.

---

# 59. Alert Is Constantly Firing

Possible causes:

```text
Threshold is wrong
Metric is wrong
Alert condition is expected
No recovery condition
Service is actually unhealthy
```

Ask:

```text
Is the alert actionable?
```

If the answer is:

```text
No
```

redesign the alert.

---

# 60. Alertmanager Failure

If Alertmanager becomes unavailable:

```text
Prometheus
   │
   ▼
Alert rules still evaluate
```

But:

```text
Notifications cannot be delivered
```

Therefore Alertmanager itself must be monitored.

Example:

```text
Alertmanager unavailable
```

should be a high-priority platform alert through an independent path where possible.

---

# 61. Production Alert Categories

Our platform should eventually have:

```text
01 - Application
02 - Kubernetes
03 - EKS
04 - Nodes
05 - Network
06 - RDS
07 - Redis
08 - SQS/SNS
09 - ALB
10 - Security
11 - GitOps
12 - Observability
13 - SLO
14 - Capacity
```

---

# 62. Recommended Alert Philosophy

### Page humans for:

```text
Customer impact
+
High confidence
+
Immediate action required
```

### Notify but don't page for:

```text
Potential degradation
+
Capacity risk
+
Early warning
```

### Dashboard only:

```text
Interesting metrics
+
Normal operational information
```

---

# 63. Full Production Flow

Our observability system now looks like:

```text
                     USERS
                       │
                       ▼
                CloudFront / WAF
                       │
                       ▼
                     ALB
                       │
                       ▼
                      EKS
                       │
             ┌─────────┼──────────┐
             │         │          │
             ▼         ▼          ▼
        Applications  Nodes    Kubernetes
             │         │          │
             └─────────┼──────────┘
                       ▼
                  Prometheus
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
        Grafana             Alert Rules
             │                   │
             │                   ▼
             │              Alertmanager
             │                   │
             │          ┌────────┼─────────┐
             │          ▼        ▼         ▼
             │       Slack   PagerDuty   Email
             │
             ▼
        Investigation
             │
       ┌─────┼──────┐
       ▼     ▼      ▼
     Logs  Metrics Traces
       │     │      │
       ▼     ▼      ▼
 Elasticsearch  Prometheus  Jaeger
       │
       ▼
     Kibana
```

---

# 64. Final Production Checklist

### Prometheus

* [ ] Alert rules stored in Git
* [ ] Rules reviewed
* [ ] `for` duration configured appropriately
* [ ] SLO alerts implemented for critical services
* [ ] Recording rules used where useful

### Alertmanager

* [ ] HA configured
* [ ] Routing configured
* [ ] Grouping configured
* [ ] Deduplication verified
* [ ] Inhibition configured
* [ ] Silences controlled
* [ ] Repeat interval configured
* [ ] Notification integrations tested

### Security

* [ ] Secrets not stored in Git
* [ ] IAM least privilege
* [ ] Internal access
* [ ] TLS
* [ ] RBAC
* [ ] Auditability

### Operations

* [ ] Runbook for critical alerts
* [ ] Service ownership defined
* [ ] Escalation path defined
* [ ] Alert testing performed
* [ ] Alertmanager itself monitored
* [ ] Alert fatigue reviewed regularly

---

# 65. The Most Important Mental Model

Don't memorize individual alert names.

Think:

```text
                    CONDITION
                       │
                       ▼
                  Prometheus
                       │
                       ▼
                     ALERT
                       │
                       ▼
                 Alertmanager
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       GROUP         ROUTE        INHIBIT
          │            │            │
          └────────────┼────────────┘
                       ▼
                  NOTIFICATION
                       │
                       ▼
                  HUMAN ACTION
                       │
                       ▼
                    RUNBOOK
                       │
                       ▼
                   RESOLUTION
```

The objective is not:

> **“Generate more alerts.”**

The objective is:

> **“Get the right alert to the right engineer at the right time with enough context to take action.”**

---

## Next file

**`33-Node-Exporter.md`**

We will go one layer deeper into EKS node-level monitoring:

```text
EC2 Worker Node
      │
      ▼
Node Exporter
      │
      ▼
Prometheus
      │
      ▼
Grafana
      │
      ▼
Alertmanager
```

That chapter will cover **DaemonSet deployment, node CPU/memory/filesystem/network metrics, disk pressure, inode exhaustion, EC2/EKS node troubleshooting, Node Exporter security, resource impact, PromQL, production dashboards, and real node-failure scenarios**.
