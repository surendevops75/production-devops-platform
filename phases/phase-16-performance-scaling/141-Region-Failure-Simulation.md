# 141 — REGION FAILURE SIMULATION

## 1. Objective

In the previous chapters, we built the individual recovery mechanisms:

```text
135 — EKS Recovery
136 — RDS Recovery
137 — S3 Recovery
138 — ECR Recovery
139 — Secrets/KMS Recovery
140 — DNS Failover
```

Now we combine them.

The goal of this chapter is to simulate:

> **What happens when the primary AWS region becomes unavailable and the e-commerce platform must operate from the DR region?**

This is a **controlled disaster-recovery simulation**, not a production outage.

We will test:

```text
Primary Region Failure
        ↓
Detection
        ↓
Incident Declaration
        ↓
DR Readiness
        ↓
Data Recovery
        ↓
EKS Recovery
        ↓
DNS Failover
        ↓
Business Validation
        ↓
RTO/RPO Measurement
        ↓
Failback
```

---

# 2. Scenario

Our primary region:

```text
ap-south-1
```

Our DR region:

```text
ap-southeast-1
```

Example:

```text
                    Internet
                       |
                       v
                   Route 53
                       |
                Failover DNS
                 /          \
                /            \
               v              v
        ap-south-1       ap-southeast-1
          PRIMARY              DR
             |                  |
            ALB                ALB
             |                  |
            EKS                EKS
             |                  |
       21 Microservices   21 Microservices
             |                  |
            RDS                RDS
             |                  |
           Redis              Redis
             |                  |
         Messaging          Messaging
```

---

# 3. What Does "Region Failure" Mean?

We need to define the failure.

A region failure simulation could represent:

```text
AWS regional outage
```

or:

```text
Primary EKS unavailable
```

or:

```text
Primary networking unavailable
```

or:

```text
Primary application completely unavailable
```

We should **not** randomly destroy production resources.

Instead, we create a controlled failure condition that produces the same operational symptoms.

---

# 4. Important Safety Rule

Never perform this:

```text
terraform destroy
```

against the production region just to simulate disaster recovery.

Never intentionally delete:

```text
RDS production
EKS production
S3 production
ECR production
KMS production keys
```

for a DR exercise.

The simulation should use:

```text
DR/test environment
```

or controlled failure injection.

---

# 5. DR Game Day Architecture

We will have:

```text
Primary Environment
        |
        | simulated failure
        v
Primary unavailable
        |
        v
Route 53 detects failure
        |
        v
DR Environment
        |
        v
Customer traffic
```

The DR environment should already have enough infrastructure to recover within the required RTO.

---

# 6. DR Models

There are several possible DR models.

## Backup and Restore

```text
Primary
   |
   | backups
   v
DR
```

Cheaper but slower.

---

## Pilot Light

Critical infrastructure exists in DR:

```text
VPC
IAM
KMS
basic EKS
database recovery capability
```

Application capacity is increased during disaster.

---

## Warm Standby

DR is already running:

```text
DR EKS
DR applications
DR database
DR dependencies
```

but at reduced capacity.

---

## Active/Active

Both regions serve traffic:

```text
Primary ← traffic
DR      ← traffic
```

This is more complex.

For our learning project, we will primarily simulate:

```text
Primary production
+
Warm/Pilot-Light DR
```

and document how the architecture can evolve toward active/active.

---

# 7. Define Business Requirements First

Before performing the simulation, define:

```text
RTO
RPO
```

Example:

```text
RTO = 30 minutes
RPO = 5 minutes
```

These are **example targets**, not automatically achieved values.

The game day determines whether the architecture actually meets them.

---

# 8. Define Critical Business Functions

The application contains 21 services.

But not every service has the same importance during a regional outage.

Critical order path:

```text
Customer
   |
   v
Auth
   |
   v
Catalog/Product
   |
   v
Cart
   |
   v
Order
   |
   +----> Inventory
   |
   +----> Payment
   |
   +----> Fraud
   |
   v
Shipping
   |
   v
Notification
```

Other workloads:

```text
Reporting
Recommendation
Invoice
Analytics
```

may have different recovery priorities.

---

# 9. Recovery Priority

Example:

| Priority | Workloads                              |
| -------- | -------------------------------------- |
| P0       | Order, Payment, Inventory              |
| P1       | Auth, Customer, Cart, Product, Pricing |
| P1       | Shipping, Delivery, Fraud              |
| P2       | Notification, Search, Tax              |
| P2       | Invoice, Review                        |
| P3       | Recommendation, Reporting              |

These priorities are architectural examples.

The actual business must define them.

---

# 10. Before the Game Day

Record the normal state.

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

Record:

```text
RPS
latency
5xx
active orders
queue depth
Kafka lag
RDS connections
Redis health
```

---

# 11. Capture Baseline Application Health

Example:

```bash
curl -I https://api.example.com
```

Check:

```text
HTTP status
latency
TLS
DNS
```

Business test:

```text
Login
Browse product
Add to cart
Create order
Payment
Inventory reservation
```

Record the successful transaction IDs.

These become our baseline.

---

# 12. Capture DNS Baseline

Run:

```bash
dig api.example.com
```

Record:

```text
Primary endpoint
TTL
```

Also test through multiple resolvers:

```bash
dig @1.1.1.1 api.example.com
```

```bash
dig @8.8.8.8 api.example.com
```

This helps demonstrate DNS caching behavior during failover.

---

# 13. Capture RDS Baseline

Record:

```text
DB instance status
DB endpoint
connections
CPU
memory-related metrics
storage
replication/recovery state
```

For example:

```text
Current connections = 240
Maximum configured capacity = X
```

The exact values must come from the actual environment.

---

# 14. Capture Redis Baseline

Record:

```text
Redis availability
memory usage
hit ratio
connections
evictions
latency
```

---

# 15. Capture Messaging Baseline

For SQS:

```text
ApproximateNumberOfMessagesVisible
ApproximateAgeOfOldestMessage
```

For Kafka:

```text
Consumer lag
partition status
ISR
broker health
```

For RabbitMQ:

```text
queue depth
consumer count
unacknowledged messages
```

---

# 16. Capture Observability Baseline

Make sure:

```text
Prometheus
Grafana
ELK
OpenTelemetry Collector
Jaeger
Alertmanager
```

are operational.

Otherwise, during the game day you won't be able to determine what happened.

---

# 17. Game Day Timeline

We create a timeline:

```text
T0
|
| Failure begins
|
T1
|
| Monitoring detects failure
|
T2
|
| Incident declared
|
T3
|
| DR validation
|
T4
|
| DNS failover
|
T5
|
| First successful DR request
|
T6
|
| Business validation complete
|
T7
|
| Recovery stabilized
```

---

# 18. T0 — Simulate Primary Failure

Instead of destroying infrastructure, introduce a controlled failure.

For example:

```text
Primary application endpoint
        ↓
intentionally made unavailable
```

or:

```text
Primary ALB
        ↓
test health endpoint
        ↓
unhealthy
```

The objective is to create:

```text
Primary = unavailable
DR = available
```

---

# 19. T1 — Detection

Monitoring should detect:

```text
HTTP 5xx
```

or:

```text
HTTP latency
```

or:

```text
ALB unhealthy
```

or:

```text
Route 53 health check failure
```

or:

```text
AWS service event
```

Alert:

```text
PRIMARY REGION HEALTH DEGRADED
```

---

# 20. Incident Declaration

The incident commander decides:

```text
Is this:
service incident
AZ incident
regional incident
?
```

If regional criteria are satisfied:

```text
SEV-1 / regional DR event
```

according to the organization's incident policy.

---

# 21. Stop Risky Changes

Once DR is declared:

```text
Pause deployments
Pause Terraform changes
Pause database migrations
Pause non-essential configuration changes
```

Why?

Because during disaster recovery we want:

```text
Known state
```

not:

```text
Failure
+
deployment
+
migration
+
configuration change
```

all happening simultaneously.

---

# 22. Validate DR Infrastructure

Check the DR EKS cluster:

```bash
kubectl --context=dr get nodes
```

Expected:

```text
NAME       STATUS
node-1     Ready
node-2     Ready
node-3     Ready
```

Then:

```bash
kubectl --context=dr get pods -A
```

---

# 23. Validate EKS Add-ons

Check:

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI
AWS Load Balancer Controller
Karpenter
External Secrets Operator
```

For example:

```bash
kubectl --context=dr -n kube-system get pods
```

---

# 24. Validate Application Deployment

Check:

```bash
kubectl --context=dr get deployments -n ecommerce
```

Then:

```bash
kubectl --context=dr get pods -n ecommerce
```

We want:

```text
auth-service
user-service
customer-service
...
reporting-service
```

available according to the DR design.

---

# 25. Validate ECR

Confirm required images exist in the DR artifact strategy.

For example:

```text
order-service
payment-service
inventory-service
```

must be deployable.

Remember:

```text
Git repository
       ≠
Docker image artifact
```

A Git repository alone does not guarantee that the exact tested image still exists.

---

# 26. Validate Secrets

Check External Secrets:

```bash
kubectl --context=dr get externalsecret -A
```

Check:

```bash
kubectl --context=dr get secret -n ecommerce
```

Do not print secret values.

We want to confirm:

```text
Secret exists
Secret synced
Pod Identity works
KMS works
Application can authenticate
```

---

# 27. Validate RDS

The DR database must be in an appropriate recovery state.

Check:

```text
Database available
Correct version
Correct schema
Required data
Required credentials
Security groups
Network connectivity
```

Do not simply assume:

```text
RDS = available
```

means:

```text
Application = ready
```

---

# 28. Validate Database Data

Test important records:

```text
Customer
Product
Inventory
Order
Payment
```

Example:

```text
Known order:
ORD-100001
```

Check that the expected state exists in DR.

---

# 29. Validate Flyway

Check schema migration state.

For example:

```text
flyway_schema_history
```

Ensure the DR database contains the expected migrations.

You do not want:

```text
Application version = 50
Database schema = 45
```

during failover.

---

# 30. Validate Redis

Check:

```text
Redis reachable
authentication works
TLS works
required cache configuration exists
```

Remember:

> Redis is generally a cache, not the source of truth for critical business data.

If the DR Redis cache is cold, the application should recover by reading from the database and rebuilding cache entries.

---

# 31. Validate Messaging

Check:

```text
SQS
Kafka
RabbitMQ
```

according to the workload.

Important questions:

```text
Where are messages currently located?
What happens to messages in primary?
Can events be replayed?
What is the expected duplicate behavior?
```

---

# 32. Activate DR Application

If the DR application is already running:

```text
Scale to required capacity
```

If using pilot light:

```text
Deploy application
        ↓
Wait for Ready
        ↓
Run smoke tests
        ↓
Scale
```

For example:

```bash
kubectl --context=dr scale deployment order-service \
  -n ecommerce \
  --replicas=3
```

Actual replica counts must come from capacity testing.

---

# 33. Validate ALB

Check:

```bash
kubectl --context=dr get ingress -n ecommerce
```

Verify:

```text
DR ALB
listeners
TLS certificate
target groups
healthy targets
security groups
```

Then test the DR endpoint directly if the architecture permits.

---

# 34. Validate Business Workflow Before DNS

Before sending customers to DR:

```text
DR ALB
   |
   v
DR EKS
```

test:

```text
GET /health
```

Then:

```text
Login
Product
Cart
Order
Payment
Inventory
```

Only after this should we proceed to DNS failover.

---

# 35. DNS Failover

Now Route 53 should evaluate:

```text
Primary = unhealthy
Secondary = healthy
```

The public endpoint:

```text
api.example.com
```

should begin resolving toward the DR architecture according to the configured failover design.

Run:

```bash
dig api.example.com
```

from multiple locations/resolvers.

---

# 36. DNS Convergence

Do not expect every client to switch simultaneously.

Example:

```text
10:00:00
Primary fails

10:00:20
Route 53 detects unhealthy

10:00:30
New DNS queries receive DR

10:00:40
Some clients still use cached primary answer

10:01:00+
More clients begin using DR
```

This is why the RTO measurement should use real client/request observations rather than simply:

```text
Route 53 status = healthy
```

---

# 37. T5 — First Successful DR Request

Record:

```text
T5 = first successful customer request through DR
```

For example:

```text
T0 = 10:00:00
T5 = 10:08:30
```

Then:

```text
Recovery time so far
= 8 minutes 30 seconds
```

---

# 38. Business Validation

Now execute a real business journey.

Example:

```text
Customer
   |
   v
Login
   |
   v
Browse product
   |
   v
Add to cart
   |
   v
Create order
   |
   v
Reserve inventory
   |
   v
Authorize payment
   |
   v
Create invoice
   |
   v
Send notification
```

Record:

```text
Order ID
Payment ID
Inventory reservation
Event IDs
```

This gives us evidence that the platform is actually operating.

---

# 39. Verify Idempotency

This is especially important during DR.

Suppose the customer submitted:

```text
POST /orders
```

just before the outage.

The request may have succeeded in the primary region, but the customer may not have received the response.

During recovery, the client may retry.

Without idempotency:

```text
Retry
  ↓
New order
  ↓
Duplicate order
```

With idempotency:

```text
Same idempotency key
       ↓
Existing operation detected
       ↓
Original result returned
```

---

# 40. Verify Payment Safety

Payment must receive special treatment.

Check:

```text
Payment authorization
Payment capture
Payment provider request
Idempotency key
Transaction state
```

Never assume:

```text
DNS failover
=
payment state automatically safe
```

Payment providers are external systems and require their own recovery logic.

---

# 41. Verify Inventory

Check:

```text
Inventory before
Inventory reservation
Inventory after
```

Example:

```text
Before:
100 units

Order:
2 units

Expected:
98 units
```

Verify no duplicate reservation occurred.

---

# 42. Verify Messaging

After creating an order, inspect:

```text
Kafka
SQS
RabbitMQ
```

depending on which workflow is being tested.

Check:

```text
eventId
correlationId
traceId
```

Make sure events are not duplicated unexpectedly.

If duplicates occur, verify consumer idempotency.

---

# 43. Verify Observability in DR

The DR environment must produce:

```text
Metrics
Logs
Traces
Alerts
```

Check Grafana:

```text
RPS
5xx
latency
CPU
memory
pod restarts
RDS
Redis
Kafka/SQS
```

Check ELK:

```text
service
environment
region
pod
correlationId
traceId
```

Check Jaeger:

```text
Customer request
   ↓
Order
   ↓
Inventory
   ↓
Payment
```

---

# 44. Trace Example

A customer submits:

```text
POST /api/v1/orders
```

Trace:

```text
Trace ID:
abc123
```

Spans:

```text
order-service
   |
   +--- customer-service
   |
   +--- product-service
   |
   +--- pricing-service
   |
   +--- inventory-service
   |
   +--- payment-service
```

This lets the incident team prove that the DR workflow is working end-to-end.

---

# 45. RPO Measurement

Suppose:

```text
Primary failure:
10:00:00
```

Latest transaction safely available in DR:

```text
09:57:30
```

Then:

```text
Measured RPO
= 2 minutes 30 seconds
```

This is the actual observed result for this test.

---

# 46. RTO Measurement

Suppose:

```text
T0 = 10:00:00
```

Customer order successfully completed through DR:

```text
T6 = 10:12:40
```

Then:

```text
Measured RTO
= 12 minutes 40 seconds
```

Compare that with the business target.

---

# 47. Example Game Day Result

Example only:

| Metric           |            Target |             Measured |
| ---------------- | ----------------: | -------------------: |
| RTO              |            30 min |              12m 40s |
| RPO              |             5 min |               2m 30s |
| DNS convergence  | Defined in design | Measured during test |
| First DR request |          Required |              10m 20s |
| Order workflow   |          Required |               Passed |
| Payment          |          Required |               Passed |
| Inventory        |          Required |               Passed |

These values must be replaced by actual measurements from your environment.

---

# 48. What If RTO Fails?

Suppose target:

```text
RTO = 30 minutes
```

but measured:

```text
RTO = 52 minutes
```

Don't simply say:

```text
Need faster DNS
```

Break down the timeline:

```text
Detection          5 min
Incident decision  10 min
DR activation      20 min
DNS convergence     5 min
Business testing   12 min
-------------------------
Total              52 min
```

Now we know where the problem actually exists.

---

# 49. Improving RTO

Possible improvements:

### Detection

```text
Better alerts
Synthetic monitoring
Regional health signals
```

### DR infrastructure

```text
Warm standby
Pre-provisioned EKS
Pre-created ALB
Pre-created IAM
```

### Application

```text
Pre-deployed applications
```

### Data

```text
Continuous replication
Automated recovery
```

### DNS

```text
Appropriate TTL
Automated health detection
```

### Operations

```text
Clear runbook
Automation
Practiced incident response
```

---

# 50. What If RPO Fails?

Suppose target:

```text
RPO = 5 minutes
```

but measured:

```text
RPO = 20 minutes
```

Potential causes:

```text
Database replication lag
Backup frequency
S3 replication delay
Kafka replication
SQS regional message handling
Application writes not replicated
```

The solution isn't necessarily DNS.

You need to improve the affected data recovery mechanism.

---

# 51. Failback Preparation

After DR is stable:

```text
Do NOT immediately send traffic back.
```

First recover primary.

```text
Primary Region
      |
      v
Repair
      |
      v
Validate
      |
      v
Synchronize data
      |
      v
Smoke tests
      |
      v
Controlled failback
```

---

# 52. Primary Recovery

Restore:

```text
VPC
EKS
ALB
21 services
Secrets
ECR access
RDS
Redis
Messaging
Observability
```

using:

```text
Terraform
+
GitOps
+
Recovery procedures
```

---

# 53. Data Synchronization

This is one of the hardest parts.

During DR operation:

```text
DR
 |
 +---- new orders
 +---- payments
 +---- inventory changes
```

Primary must not simply start serving old data.

We need a defined data reconciliation process.

Possible approaches depend on the database architecture:

```text
Replication
Backup/PITR
Logical replication
Application event replay
Dual-write architecture
```

The correct method depends on the actual DR design.

---

# 54. Failback

After data synchronization:

```text
DR
 |
 | stable
 v
Primary
 |
 | validated
 v
Route 53
 |
 v
Controlled traffic return
```

Monitor:

```text
5xx
latency
orders
payments
inventory
DB
queues
Kafka
```

---

# 55. Game Day Incident Timeline

Create a document:

```text
dr-game-day-2026-10-02.md
```

Example:

```text
10:00:00
Primary failure injected

10:00:20
Monitoring detected failure

10:01:00
Incident declared

10:03:00
DR infrastructure validated

10:05:00
DR application validated

10:06:30
DNS failover initiated

10:08:00
First DR traffic observed

10:10:00
Login validated

10:11:00
Order workflow validated

10:12:40
Payment + inventory validated

10:15:00
RTO measurement complete
```

---

# 56. Evidence Collection

Save:

```text
kubectl outputs
Route 53 state
ALB metrics
RDS metrics
Redis metrics
Kafka/SQS metrics
Grafana screenshots
Kibana logs
Jaeger traces
Git commits
ArgoCD state
incident timeline
```

The objective is to have evidence rather than memory.

---

# 57. Failure Injection Matrix

For the simulation program, we can progressively test:

| Test  | Failure                         |
| ----- | ------------------------------- |
| DR-01 | Primary application unavailable |
| DR-02 | Primary EKS unavailable         |
| DR-03 | Primary ALB unavailable         |
| DR-04 | Primary RDS unavailable         |
| DR-05 | Primary Redis unavailable       |
| DR-06 | Primary ECR unavailable         |
| DR-07 | Primary Secrets unavailable     |
| DR-08 | Primary S3 unavailable          |
| DR-09 | Primary messaging unavailable   |
| DR-10 | Complete regional simulation    |

We should not perform all failures simultaneously initially.

---

# 58. Recovery Dependency Matrix

| Component       | Primary                 | DR Strategy                    |
| --------------- | ----------------------- | ------------------------------ |
| Route 53        | Global                  | Failover                       |
| EKS             | Regional                | Rebuild/standby                |
| ECR             | Regional                | Replication/artifact retention |
| RDS             | Regional                | Replication/backup/PITR        |
| Redis           | Regional                | Replication/rebuild            |
| S3              | Regional/global service | Versioning/replication         |
| Secrets Manager | Regional                | Regional secret strategy       |
| KMS             | Regional                | DR key strategy                |
| SQS             | Regional                | Replay/recovery strategy       |
| Kafka           | Regional                | Replication/replay             |
| Git             | Global                  | GitHub                         |
| Terraform state | S3                      | Protected state + recovery     |
| ArgoCD          | Regional                | Reinstall/reconcile            |

---

# 59. Disaster Recovery Decision Tree

During an actual incident:

```text
Primary failure detected
        |
        v
Is customer traffic affected?
        |
      Yes
        |
        v
Is failure isolated?
      /   \
    Yes    No
     |      |
     v      v
Normal     Regional
incident   DR assessment
              |
              v
        Is DR ready?
          /     \
        No       Yes
        |         |
        v         v
Recover       Validate data
primary           |
                  v
             Activate DR
                  |
                  v
             DNS failover
                  |
                  v
          Business validation
```

---

# 60. Production Game Day Rules

### Rule 1

Have an incident commander.

### Rule 2

Have a recovery owner.

### Rule 3

Have a communications owner.

### Rule 4

Have a database owner.

### Rule 5

Have a platform owner.

### Rule 6

Have a security owner.

### Rule 7

Define stop conditions.

Example:

```text
STOP TEST IF:
- unexpected customer impact
- uncontrolled data corruption
- payment duplication
- irreversible data loss
- security boundary violated
```

---

# 61. Post-Game-Day Review

After the simulation, ask:

```text
Did detection work?
Did alerting work?
Did DNS failover work?
Did DR EKS work?
Did images exist?
Did secrets work?
Did KMS work?
Did RDS recover?
Did Redis recover?
Did messaging recover?
Did applications start?
Did business workflows work?
Was RTO achieved?
Was RPO achieved?
Could we fail back?
```

---

# 62. Create Action Items

Example:

```text
DR-001
Problem:
DR ECR missing payment-service image

Action:
Enable cross-region artifact replication

Owner:
Platform Team
```

```text
DR-002
Problem:
DR database migration was two versions behind

Action:
Add migration validation to DR readiness check

Owner:
Database Team
```

```text
DR-003
Problem:
DNS convergence exceeded expected window

Action:
Review DNS architecture and resolver behavior

Owner:
Network Team
```

---

# 63. Final Regional DR Architecture

After this chapter, the architecture becomes:

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
              PRIMARY REGION     DR REGION
              ap-south-1        ap-southeast-1
                    |                  |
                   ALB                ALB
                    |                  |
                   EKS                EKS
                    |                  |
              21 Services        21 Services
                    |                  |
          +---------+--------+ +-------+--------+
          |         |        | |       |        |
         RDS      Redis     MQ RDS    Redis     MQ
          |                   | 
          +------ DR ---------+
                    |
              Data Recovery
                    |
       +------------+-------------+
       |            |             |
      S3           ECR         Secrets/KMS
       |            |             |
       +------------+-------------+
                    |
             Observability
                    |
       +------------+-------------+
       |            |             |
   Prometheus     ELK          Jaeger
       |
    Grafana
```

---

# 64. The Most Important Lesson

A regional disaster is **not a Kubernetes problem**.

It is a complete platform problem:

```text
DNS
+
Networking
+
EKS
+
IAM
+
ECR
+
RDS
+
Redis
+
S3
+
Secrets
+
KMS
+
SQS/Kafka/RabbitMQ
+
21 Microservices
+
Observability
+
Incident Management
+
Business Validation
```

The actual DR objective is:

```text
                REGION FAILURE
                      |
                      v
                Detect quickly
                      |
                      v
                Recover safely
                      |
                      v
                 Redirect traffic
                      |
                      v
             Process real business
                      |
                      v
                 Protect data
                      |
                      v
                Measure RTO/RPO
                      |
                      v
              Recover primary
                      |
                      v
                Fail back safely
```

**This is the point where the individual recovery chapters become one complete disaster-recovery system.**

### Next: **142 — Full DR Game Day**

That chapter will turn this simulation into a **complete end-to-end DR exercise**, including the incident commander process, exact game-day phases, failure injection, communication, RTO/RPO measurement, business validation, evidence collection, failback, and final DR report.
