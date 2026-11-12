# 142 — FULL DR GAME DAY

## 1. Objective

This chapter combines everything we built in Chapters **135–141** into one complete disaster-recovery exercise.

We will simulate a major failure of the primary AWS region and execute the entire operational lifecycle:

```text
Failure Injection
      ↓
Detection
      ↓
Incident Declaration
      ↓
Incident Command
      ↓
Primary Assessment
      ↓
DR Readiness Validation
      ↓
DR Activation
      ↓
DNS Failover
      ↓
Business Validation
      ↓
RTO/RPO Measurement
      ↓
Stabilization
      ↓
Primary Recovery
      ↓
Data Synchronization
      ↓
Controlled Failback
      ↓
RCA
      ↓
Corrective Actions
```

This is the **final DR exercise** before we move into the incident-simulation phase.

---

# 2. What Is a DR Game Day?

A DR Game Day is a controlled exercise where the engineering team intentionally simulates a serious failure and practices recovering the platform.

The objective is not:

```text
"Let's see if Kubernetes survives."
```

The objective is:

```text
"Can the business continue operating after a major infrastructure failure?"
```

---

# 3. Our Scenario

Primary:

```text
Region:
ap-south-1

Role:
PRIMARY
```

DR:

```text
Region:
ap-southeast-1

Role:
SECONDARY / DR
```

Architecture:

```text
                         CUSTOMERS
                             |
                             v
                         Route 53
                             |
                       Failover DNS
                       /           \
                      /             \
                     v               v
              ap-south-1       ap-southeast-1
                PRIMARY              DR
                   |                  |
                  ALB                ALB
                   |                  |
                  EKS                EKS
                   |                  |
            21 Microservices   21 Microservices
                   |                  |
          +--------+-------+   +------+--------+
          |        |       |   |      |        |
         RDS     Redis    MQ  RDS   Redis      MQ
```

Supporting systems:

```text
GitHub
Terraform
ArgoCD
ECR
S3
Secrets Manager
KMS
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
Alertmanager
```

---

# 4. Game Day Objectives

We want to prove:

* [ ] Primary failure is detected
* [ ] Incident process works
* [ ] DR team knows what to do
* [ ] DR infrastructure is usable
* [ ] Required images exist
* [ ] Secrets are available
* [ ] KMS works
* [ ] Database recovery works
* [ ] Redis recovery works
* [ ] Messaging recovery works
* [ ] Applications start
* [ ] Route 53 failover works
* [ ] Customers can reach DR
* [ ] Orders can be created
* [ ] Payments are safe
* [ ] Inventory remains consistent
* [ ] Observability works
* [ ] RTO is measured
* [ ] RPO is measured
* [ ] Primary can be recovered
* [ ] Failback is controlled
* [ ] Evidence is collected
* [ ] Corrective actions are created

---

# 5. Game Day Participants

A realistic exercise should have defined roles.

## Incident Commander

Responsible for:

```text
Decision making
Incident coordination
Failover approval
Communication
Stop/go decisions
```

---

## Platform Engineer

Responsible for:

```text
EKS
Karpenter
ALB
Kubernetes
ArgoCD
Networking
```

---

## Database Engineer

Responsible for:

```text
RDS
PITR
Replication
Schema
Data validation
```

---

## DevOps Engineer

Responsible for:

```text
Terraform
CI/CD
ECR
GitOps
AWS infrastructure
```

---

## Application Engineer

Responsible for:

```text
Microservices
Application errors
Business workflows
```

---

## Security Engineer

Responsible for:

```text
IAM
KMS
Secrets
Security controls
```

---

## SRE/Observability Engineer

Responsible for:

```text
Prometheus
Grafana
Alertmanager
ELK
Jaeger
Incident metrics
```

---

# 6. Communication Channel

Create a dedicated incident channel.

Example:

```text
#incident-dr-game-day
```

All important events should be recorded there.

Example:

```text
10:00 — Failure injected
10:02 — Alert received
10:04 — Incident declared
10:08 — DR validation started
10:15 — DNS failover approved
10:18 — DR traffic observed
```

This becomes part of the incident evidence.

---

# 7. Change Freeze

Before starting:

```text
FREEZE
```

Pause:

```text
Production deployments
Terraform changes
Database migrations
Infrastructure modifications
Non-essential configuration changes
```

The purpose is to avoid introducing unrelated changes during the exercise.

---

# 8. Pre-Game Checklist

Before failure injection:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

```bash
kubectl get ingress -A
```

```bash
kubectl get svc -A
```

Verify:

```text
EKS
ALB
RDS
Redis
SQS/Kafka
ECR
Secrets
KMS
S3
Route 53
```

---

# 9. Application Baseline

Perform a complete customer workflow.

```text
Login
  ↓
Browse product
  ↓
Search
  ↓
Add to cart
  ↓
Create order
  ↓
Inventory reservation
  ↓
Payment
  ↓
Invoice
  ↓
Notification
```

Record:

```text
Order ID
Payment ID
Inventory state
Event IDs
Correlation ID
Trace ID
```

Example:

```text
Order ID:
ORD-DR-000001
```

---

# 10. Capture Baseline Metrics

Before failure:

```text
RPS
P50 latency
P95 latency
P99 latency
HTTP 5xx
CPU
Memory
Pod count
RDS connections
Redis memory
Redis hit ratio
SQS backlog
Kafka lag
```

Example:

```text
RPS: 120
P95: 180 ms
5xx: 0.02%
RDS connections: 180
Kafka lag: 12
```

These are example values only.

The actual game day uses measured values.

---

# 11. Capture DNS Baseline

Run:

```bash
dig api.example.com
```

Record:

```text
Primary endpoint
TTL
Resolver
Timestamp
```

Also:

```bash
dig @1.1.1.1 api.example.com
```

and:

```bash
dig @8.8.8.8 api.example.com
```

---

# 12. Capture DR Baseline

Before the test, verify DR.

```bash
kubectl --context=dr get nodes
```

```bash
kubectl --context=dr get pods -A
```

Check:

```text
DR ALB
DR EKS
DR RDS
DR Redis
DR messaging
DR secrets
DR ECR
DR S3
```

The DR environment must not be assumed healthy simply because it exists.

---

# 13. DR Readiness Test

Create a simple checklist:

```text
DR READY?

EKS                 YES/NO
ALB                 YES/NO
21 services         YES/NO
RDS                 YES/NO
Redis               YES/NO
Messaging           YES/NO
Secrets             YES/NO
KMS                 YES/NO
ECR                 YES/NO
S3                  YES/NO
Observability       YES/NO
DNS                 YES/NO
```

If critical dependencies are unavailable, record the failure before beginning.

---

# 14. T-15 Minutes

Fifteen minutes before the exercise:

```text
Incident team assembled
Communication channel open
Monitoring dashboards open
Runbooks open
Recovery credentials verified
Game-day plan reviewed
Stop conditions reviewed
```

No destructive action has occurred yet.

---

# 15. T-5 Minutes

Confirm:

```text
Primary healthy
DR healthy
Monitoring healthy
Business workflow healthy
```

Incident commander announces:

```text
"Game Day starting in 5 minutes."
```

---

# 16. T0 — Failure Injection

We now simulate a major primary-region failure.

Important:

> Use a controlled failure mechanism in the DR/test environment. Do not destroy production resources.

Example simulation:

```text
Primary regional application endpoint
          ↓
Health endpoint intentionally unavailable
```

or an approved controlled failure that causes:

```text
PRIMARY = UNHEALTHY
```

while:

```text
DR = HEALTHY
```

---

# 17. T+1 — Initial Detection

Monitoring should detect:

```text
HTTP failures
ALB health problems
Application failures
Synthetic monitoring failure
Route 53 health degradation
```

Alert example:

```text
CRITICAL

Primary regional endpoint unavailable.

Region:
ap-south-1

Service:
ecommerce-api
```

---

# 18. T+2 — Incident Commander Takes Control

The IC announces:

```text
"Regional DR incident declared."
```

The team switches from normal operations to incident operations.

---

# 19. T+3 — Establish Incident Severity

Example:

```text
SEV-1
```

because:

```text
Customer-facing platform unavailable
```

The actual severity definitions should follow your organization's incident policy.

---

# 20. T+4 — Scope the Failure

Ask:

```text
Is one service affected?
```

```text
Is one AZ affected?
```

```text
Is EKS affected?
```

```text
Is networking affected?
```

```text
Is RDS affected?
```

```text
Is the entire region affected?
```

Use:

```text
AWS health information
CloudWatch
Prometheus
Grafana
ELK
Jaeger
kubectl
```

---

# 21. T+5 — Protect Data

Before failover:

```text
Stop unnecessary writes
Stop migrations
Stop deployments
Preserve logs
Preserve metrics
Record timestamps
```

The objective is to avoid making the recovery state harder to understand.

---

# 22. T+6 — Validate RPO

Determine:

```text
What is the latest recoverable state?
```

For example:

```text
Primary last known successful transaction:
10:00:00

DR latest recoverable transaction:
09:57:30
```

Measured RPO:

```text
2m 30s
```

---

# 23. T+7 — Validate DR EKS

```bash
kubectl --context=dr get nodes
```

Expected:

```text
node-1   Ready
node-2   Ready
node-3   Ready
```

Then:

```bash
kubectl --context=dr get pods -A
```

Check:

```text
CoreDNS
VPC CNI
EBS CSI
AWS Load Balancer Controller
Karpenter
External Secrets
ArgoCD
```

---

# 24. T+8 — Validate ArgoCD

Check:

```bash
argocd app list
```

Confirm:

```text
Synced
Healthy
```

for required DR applications.

If ArgoCD itself is unavailable:

```text
GitOps repository
        ↓
Reinstall/recover ArgoCD
        ↓
Reconcile applications
```

---

# 25. T+9 — Validate Application Pods

```bash
kubectl --context=dr get pods -n ecommerce
```

Check:

```text
READY
STATUS
RESTARTS
AGE
```

Example:

```text
order-service-xxxxx       2/2   Running
inventory-service-xxxxx   2/2   Running
payment-service-xxxxx     2/2   Running
```

---

# 26. T+10 — Validate Secrets

```bash
kubectl --context=dr get externalsecret -n ecommerce
```

Expected:

```text
STATUS = SecretSynced
```

Verify:

```text
Pod Identity
IAM
Secrets Manager
KMS
```

Do not expose secret values during the game day.

---

# 27. T+11 — Validate RDS

Check:

```text
Database available
Network reachable
Credentials valid
Schema correct
Migration state correct
Data available
```

Application connectivity should be tested from DR EKS.

---

# 28. T+12 — Validate Redis

Check:

```text
Redis reachable
TLS/authentication
Connection count
Memory
```

If Redis is empty:

```text
Application
    ↓
RDS
    ↓
Rebuild cache
```

should work where the architecture supports it.

---

# 29. T+13 — Validate Messaging

Check:

```text
SQS queues
DLQs
Kafka brokers
Kafka consumer groups
RabbitMQ queues
```

Determine:

```text
Messages available?
Messages lost?
Messages duplicated?
Messages replayable?
```

---

# 30. T+14 — Validate ECR

Confirm the required image digests exist.

For example:

```text
order-service
digest:
sha256:xxxx
```

The DR deployment should use the tested immutable artifact.

Avoid:

```text
order-service:latest
```

for disaster recovery.

---

# 31. T+15 — Validate DR ALB

Check:

```bash
kubectl --context=dr get ingress -n ecommerce
```

Confirm:

```text
ALB
HTTPS
TLS certificate
Target groups
Healthy targets
Security groups
```

---

# 32. T+16 — Direct DR Smoke Test

Before DNS failover:

```text
DR ALB
   |
   v
/api/health
```

Then:

```text
Login
Product
Cart
Order
```

If direct DR traffic doesn't work:

```text
DO NOT FAIL OVER DNS.
```

Fix the DR environment first.

---

# 33. T+17 — Failover Approval

Incident Commander asks:

```text
Is DR healthy?
```

Platform:

```text
YES
```

Database:

```text
YES
```

Application:

```text
YES
```

Security:

```text
YES
```

Business validation:

```text
YES
```

IC:

```text
APPROVED — FAIL OVER TO DR
```

---

# 34. T+18 — Route 53 Failover

Route 53 changes health evaluation:

```text
PRIMARY
   ↓
UNHEALTHY

SECONDARY
   ↓
HEALTHY
```

New DNS lookups should begin receiving the DR endpoint according to the configured failover architecture.

---

# 35. T+19 — Monitor DNS

Run:

```bash
dig api.example.com
```

from:

```text
Engineering workstation
Monitoring system
External resolver
```

For example:

```bash
dig @1.1.1.1 api.example.com
```

and:

```bash
dig @8.8.8.8 api.example.com
```

---

# 36. T+20 — First DR Customer Traffic

Monitor:

```text
ALB requests
HTTP status
RPS
latency
```

Expected:

```text
DR ALB
   ↓
DR EKS
   ↓
21 services
```

Record:

```text
T_FIRST_DR_TRAFFIC
```

---

# 37. T+21 — Business Validation

Execute:

```text
Login
   ↓
Product
   ↓
Search
   ↓
Cart
   ↓
Order
   ↓
Inventory
   ↓
Payment
   ↓
Invoice
   ↓
Notification
```

---

# 38. T+22 — Create Test Order

Create:

```text
ORD-DR-000001
```

Record:

```text
Order ID
Customer ID
Payment ID
Inventory reservation ID
Event ID
Correlation ID
Trace ID
```

This gives us an end-to-end recovery proof.

---

# 39. T+23 — Validate Payment

Check:

```text
Payment request
Payment provider response
Idempotency key
Payment status
Order status
```

Make sure:

```text
One order
=
One payment
```

unless the business workflow explicitly allows another state.

---

# 40. T+24 — Validate Inventory

Example:

```text
Before:
100 units

Order:
2 units

After:
98 units
```

Check:

```text
No duplicate reservation
No negative inventory
No lost event
```

---

# 41. T+25 — Validate Event Flow

Check:

```text
OrderCreated
       ↓
InventoryReserved
       ↓
PaymentAuthorized
       ↓
InvoiceCreated
       ↓
NotificationRequested
```

Use:

```text
Kafka
SQS
RabbitMQ
```

according to the workflow.

---

# 42. T+26 — Check Logs

Search Kibana using:

```text
correlationId
```

Example:

```text
correlationId = dr-game-000001
```

Expected:

```text
order-service
inventory-service
payment-service
invoice-service
notification-service
```

all show the same correlation context.

---

# 43. T+27 — Check Distributed Trace

Search Jaeger:

```text
traceId = abc123
```

Expected:

```text
Client
  |
  v
ALB
  |
  v
order-service
  |
  +--> inventory-service
  |
  +--> payment-service
  |
  +--> invoice-service
```

This proves observability survived the regional transition.

---

# 44. T+28 — Check Metrics

Grafana should show:

```text
DR RPS
DR latency
DR 5xx
DR CPU
DR memory
DR pod count
RDS connections
Redis
Kafka/SQS
```

Watch for:

```text
Error spike
Connection exhaustion
HPA instability
KEDA instability
Retry storms
```

---

# 45. T+30 — Stabilization

At this point:

```text
DNS → DR
```

and:

```text
DR → serving customers
```

The team should stop making unnecessary changes.

The objective changes from:

```text
RECOVERY
```

to:

```text
STABILIZATION
```

---

# 46. Scale DR According to Traffic

The DR environment may have started with reduced capacity.

Now observe:

```text
RPS
CPU
Memory
HPA
KEDA
Karpenter
RDS
Redis
```

Scale only as required.

Example:

```text
Traffic increases
      ↓
HPA/KEDA
      ↓
More Pods
      ↓
Karpenter
      ↓
More nodes
```

---

# 47. Watch for Autoscaling Problems

During DR, don't blindly allow unlimited scaling.

Potential cascade:

```text
Traffic
  ↓
HPA
  ↓
More Pods
  ↓
More DB connections
  ↓
RDS exhaustion
  ↓
Requests fail
  ↓
Retries
  ↓
More traffic
```

This is why:

```text
HPA maxReplicas
+
Hikari pool size
+
RDS capacity
```

must be designed together.

---

# 48. RTO Calculation

Suppose:

```text
T0:
10:00:00
```

First successful customer request:

```text
10:18:30
```

Business workflow fully validated:

```text
10:25:00
```

Then:

```text
Operational recovery:
18m 30s

Business recovery:
25m
```

For the game day, define clearly which timestamp represents the organization's RTO.

---

# 49. RPO Calculation

Suppose:

```text
Failure:
10:00:00
```

Last recoverable transaction:

```text
09:57:30
```

Then:

```text
RPO = 2m 30s
```

---

# 50. Compare Against Target

Example:

```text
Target RTO = 30 minutes
Measured = 25 minutes

Target RPO = 5 minutes
Measured = 2m 30s
```

This means the test met those example targets.

If it doesn't:

```text
Target RTO = 30 minutes
Measured = 45 minutes
```

then create an engineering action.

---

# 51. Don't Hide Failed Tests

A real DR exercise should be allowed to fail.

For example:

```text
❌ DR ECR image missing
❌ RDS schema outdated
❌ DNS convergence slower than expected
❌ Payment workflow failed
```

These are valuable findings.

The objective is:

```text
Find problems during the game day
```

rather than:

```text
Find problems during a real regional outage
```

---

# 52. T+60 — Recovery Review

Once DR is stable, review:

```text
Customer impact
Order success
Payment success
Inventory consistency
Messaging
Database
Observability
Security
```

Do not start failback immediately.

---

# 53. Primary Recovery

Begin rebuilding/recovering primary.

Use:

```text
Terraform
+
GitOps
+
Recovery runbooks
```

Validate:

```text
VPC
EKS
ALB
21 services
RDS
Redis
Messaging
Secrets
KMS
ECR
S3
Observability
```

---

# 54. Primary Health Check

Before failback:

```text
Primary API
        ↓
Healthy
```

Then:

```text
Primary database
        ↓
Healthy
```

Then:

```text
Primary application
        ↓
Healthy
```

Then:

```text
Primary business workflow
        ↓
Healthy
```

---

# 55. Data Reconciliation

While DR was active:

```text
DR received writes.
```

Therefore primary must be brought to a consistent state before accepting traffic.

Possible mechanisms:

```text
Database replication
Event replay
Logical replication
Application reconciliation
Backup/restore
```

The exact mechanism depends on the architecture.

---

# 56. Verify No Lost Orders

Compare:

```text
Primary
vs
DR
```

for:

```text
Orders
Payments
Inventory
Customers
Invoices
Events
```

Example:

```text
DR orders:
10,251

Primary orders after synchronization:
10,251
```

Any mismatch requires investigation before failback.

---

# 57. Verify No Duplicate Payments

This is especially important.

Search payment records using:

```text
orderId
idempotencyKey
paymentId
providerTransactionId
```

Expected:

```text
One business payment
=
One valid provider transaction
```

for the tested workflow.

---

# 58. Failback Approval

The Incident Commander should explicitly approve:

```text
PRIMARY READY FOR FAILBACK
```

Only after:

```text
Primary healthy
Data synchronized
Business validation passed
Security validated
Observability healthy
Rollback plan ready
```

---

# 59. Controlled Failback

Change DNS routing:

```text
DR
 ↓
SECONDARY

PRIMARY
 ↓
PRIMARY
```

Monitor:

```text
DNS
ALB
RPS
5xx
latency
orders
payments
inventory
```

---

# 60. Gradual Failback

Where the architecture permits controlled traffic shifting, avoid an abrupt all-at-once change.

For example:

```text
DR
100%
```

then controlled movement:

```text
DR 75%
Primary 25%
```

then:

```text
DR 50%
Primary 50%
```

then:

```text
DR 25%
Primary 75%
```

then:

```text
Primary 100%
```

The exact mechanism depends on whether the architecture uses Route 53 weighted routing, CloudFront origin behavior, or another traffic-management layer.

---

# 61. Monitor During Failback

Watch:

```text
HTTP 5xx
P95/P99 latency
RDS connections
Redis
Kafka lag
SQS backlog
Payment failures
Order failures
```

If something goes wrong:

```text
STOP
```

and return traffic to the known-good state if the failback plan permits.

---

# 62. Close the Incident

Only after:

```text
Primary stable
Customer traffic stable
Data verified
DR state documented
Monitoring stable
```

should the incident be closed.

---

# 63. Final Game Day Timeline

Example:

```text
10:00:00  Failure injected
10:01:00  Alert received
10:02:00  Incident declared
10:04:00  Scope confirmed
10:06:00  DR validation started
10:12:00  DR application validated
10:15:00  DNS failover approved
10:18:30  First DR traffic
10:22:00  Customer login validated
10:24:00  Order validated
10:25:00  Payment validated
10:26:00  Inventory validated
10:30:00  DR stabilized
10:40:00  RTO/RPO calculated
11:30:00  Primary recovery started
12:30:00  Data synchronization complete
13:00:00  Primary validated
13:15:00  Failback approved
13:20:00  Traffic returned to primary
13:40:00  Incident closed
```

These timestamps are examples for the exercise design.

---

# 64. Final DR Report

Create:

```text
dr-game-day-report.md
```

Structure:

```text
# DR Game Day Report

## Date

## Participants

## Scenario

## Primary Region

## DR Region

## Failure Injected

## Detection

## Incident Timeline

## Recovery Steps

## DNS Failover

## Application Validation

## Database Validation

## Messaging Validation

## Security Validation

## Observability Validation

## RTO

## RPO

## Customer Impact

## What Worked

## What Failed

## Findings

## Corrective Actions

## Owner

## Due Date

## Next DR Test
```

---

# 65. Example Findings

```text
Finding 1
---------
DR ECR was missing one application image.

Impact:
Payment service could not start.

Action:
Enable artifact replication and DR readiness validation.
```

```text
Finding 2
---------
DR database schema was one migration behind.

Impact:
Order service failed during startup.

Action:
Add automated Flyway version validation.
```

```text
Finding 3
---------
DNS convergence differed between test clients.

Impact:
Some clients continued reaching primary.

Action:
Document DNS caching behavior and improve client-side failover strategy.
```

---

# 66. DR Readiness Scorecard

Instead of assigning an overall score, track concrete evidence:

| Area              | Evidence                      |
| ----------------- | ----------------------------- |
| Detection         | Alert generated               |
| Incident response | IC declared incident          |
| EKS               | DR cluster healthy            |
| ECR               | Required digests available    |
| Secrets           | ESO synchronized              |
| KMS               | Keys usable                   |
| RDS               | Data recovered                |
| Redis             | Cache available/rebuildable   |
| Messaging         | Recovery/replay tested        |
| DNS               | Failover observed             |
| Application       | Business workflow passed      |
| Observability     | Metrics/logs/traces available |
| RTO               | Measured                      |
| RPO               | Measured                      |
| Failback          | Completed                     |

This is much more useful than simply saying:

```text
"DR passed."
```

---

# 67. DR Automation Opportunities

After completing the manual game day, automate repetitive checks.

For example:

```text
DR Readiness Script
```

could validate:

```text
EKS
ECR
Secrets
RDS
Redis
S3
Messaging
ALB
DNS
ArgoCD
```

Conceptually:

```text
DR Readiness
      |
      +--> EKS       PASS
      +--> ECR       PASS
      +--> Secrets   PASS
      +--> RDS       PASS
      +--> Redis     PASS
      +--> S3        PASS
      +--> Kafka     PASS
      +--> ALB       PASS
      +--> DNS       PASS
      +--> ArgoCD    PASS
```

---

# 68. Automated DR Readiness Gate

Before approving failover:

```text
DR readiness check
       |
       +---- EKS             PASS
       +---- Applications    PASS
       +---- Database        PASS
       +---- Secrets         PASS
       +---- ECR             PASS
       +---- Messaging      PASS
       +---- ALB             PASS
       +---- Observability  PASS
       |
       v
   FAILOVER APPROVED
```

If a critical dependency fails:

```text
FAILOVER BLOCKED
```

until the incident commander makes an informed decision.

---

# 69. What We Have Proven

After completing the full DR game day, we should be able to demonstrate:

```text
Primary Region
      |
      | failure
      v
Incident Detection
      |
      v
Incident Command
      |
      v
DR Readiness
      |
      v
Data Recovery
      |
      v
DR EKS
      |
      v
21 Microservices
      |
      v
Route 53
      |
      v
Customer Traffic
      |
      v
Order
      |
      +--> Inventory
      |
      +--> Payment
      |
      +--> Invoice
      |
      +--> Notification
```

That is a **real end-to-end DR workflow**, rather than just an infrastructure diagram.

---

# 70. Final Production Architecture After Phase 18

At this point our platform has:

```text
                    INTERNET
                       |
                       v
                    Route 53
                       |
                 DNS Failover
                  /          \
                 /            \
                v              v
           PRIMARY             DR
          ap-south-1       ap-southeast-1
               |                 |
              ALB               ALB
               |                 |
              EKS               EKS
               |                 |
        21 Microservices   21 Microservices
               |                 |
        +------+------+    +------+------+
        |      |      |    |      |      |
       RDS   Redis   MQ   RDS   Redis    MQ
        |                     |
        +------ DR/Data ------+
               |
        +------+------+------+
        |      |      |      |
       S3     ECR  Secrets  KMS
               |
        +------+------+------+
        |      |      |      |
   Prometheus Grafana ELK  Jaeger
               |
          Alertmanager
```

And the operational lifecycle is:

```text
Build
  ↓
Deploy
  ↓
Monitor
  ↓
Detect
  ↓
Respond
  ↓
Recover
  ↓
Failover
  ↓
Validate
  ↓
Failback
  ↓
Improve
```

---

# 71. Phase 18 Complete

The disaster-recovery section is now complete:

```text
135 — EKS Recovery
136 — RDS Recovery
137 — S3 Recovery
138 — ECR Recovery
139 — Secrets/KMS Recovery
140 — DNS Failover
141 — Region Failure Simulation
142 — Full DR Game Day
```

The next phase moves from **planned disaster recovery** into **realistic incident troubleshooting simulations**.

# Next: 143 — CrashLoopBackOff Production Incident

We will simulate a real production incident:

```text
Deployment
   ↓
New Pods
   ↓
CrashLoopBackOff
   ↓
Customer impact
   ↓
Prometheus alert
   ↓
Grafana
   ↓
kubectl investigation
   ↓
Logs
   ↓
Events
   ↓
Root cause
   ↓
Fix
   ↓
Rollback / redeploy
   ↓
Validation
   ↓
RCA
```

This will be the beginning of **Phase 19 — Production Incident Simulations**.
