# 45 — Disaster Recovery

> **Enterprise E-Commerce Platform | AWS | EKS | RDS PostgreSQL | Redis | SQS/SNS | S3 | ECR | ArgoCD | GitOps**

Disaster Recovery (DR) is the ability to **restore the business after a major failure**.

Deployment strategies such as Rolling, Blue-Green, and Canary protect us from **bad releases**.

Disaster Recovery protects us from **infrastructure and regional failures**.

The distinction is important:

```text
Bad application release
        ↓
Blue-Green / Canary
        ↓
Rollback
```

versus:

```text
AWS / infrastructure disaster
        ↓
Disaster Recovery
        ↓
Restore / Failover
```

---

# 1. What Is a Disaster?

A disaster is an event that prevents the production platform from operating normally.

Examples:

```text
EKS cluster failure
AWS Availability Zone failure
RDS failure
Database corruption
Accidental deletion
S3 data loss
IAM misconfiguration
Bad Terraform change
Security incident
Large-scale networking failure
AWS Region failure
```

Not every incident is a disaster.

For example:

```text
One Pod crashes
```

is normally:

```text
Kubernetes self-healing
```

while:

```text
Entire production Region unavailable
```

is:

```text
Disaster Recovery event
```

---

# 2. Disaster Recovery Is Bigger Than Kubernetes

A common beginner mistake is:

> "We use EKS, so if EKS fails we just recreate the cluster."

That is incomplete.

Our application depends on:

```text
                    Application
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
      EKS               RDS              Redis
       │                 │                 │
       ▼                 ▼                 ▼
    Services          Database          Cache
       │
       ▼
      SQS/SNS
       │
       ▼
      S3
       │
       ▼
     Secrets
```

DR must cover **all important dependencies**.

---

# 3. Our Production Architecture

Primary Region:

```text
AWS ap-south-1
        │
        ▼
       VPC
        │
 ┌──────┼──────┐
 ▼      ▼      ▼
AZ-A   AZ-B   AZ-C
 │      │      │
 └──────┼──────┘
        │
       EKS
        │
 ┌──────┼─────────────┐
 ▼      ▼             ▼
Apps   Platform    Observability
 │
 ├── RDS
 ├── Redis
 ├── SQS/SNS
 ├── S3
 └── Secrets Manager
```

The primary goal is:

> Survive failures without losing more data or downtime than the business can tolerate.

---

# 4. RTO

**RTO = Recovery Time Objective**

It answers:

> How long can the application be unavailable?

Example:

```text
RTO = 1 hour
```

Means:

```text
Disaster
   ↓
Recovery should complete
within approximately 1 hour
```

---

# 5. RPO

**RPO = Recovery Point Objective**

It answers:

> How much data loss can the business tolerate?

Example:

```text
RPO = 15 minutes
```

means the business accepts that, in the worst case, approximately the latest 15 minutes of data may need to be recovered/reconstructed, depending on the architecture.

---

# 6. RTO vs RPO

Think:

```text
RTO → Time
RPO → Data
```

Example:

```text
Database disaster at 14:00

RTO = 1 hour
→ service restored by ~15:00

RPO = 15 minutes
→ recovery point around ~13:45 or better
```

These are business requirements, not values DevOps should invent.

---

# 7. Example RTO/RPO Targets

Illustrative targets:

| Component       |     RTO |    RPO |
| --------------- | ------: | -----: |
| Order service   |  30 min |  5 min |
| Payment service |  30 min |  5 min |
| Catalog         |  1 hour | 15 min |
| Reporting       | 4 hours | 1 hour |
| Logs            | 4 hours | 1 hour |

These are **example targets**.

The actual values must come from business requirements.

---

# 8. Why RTO/RPO Matter

Suppose management says:

> "We need zero downtime and zero data loss."

That sounds simple.

But achieving it can require:

```text
multi-region
synchronous replication
duplicate infrastructure
global traffic management
database failover
continuous testing
large operational investment
```

Therefore the first question should always be:

```text
What RTO?
What RPO?
```

---

# 9. Disaster Recovery Tiers

Common DR patterns include:

```text
Backup & Restore
       ↓
Pilot Light
       ↓
Warm Standby
       ↓
Active/Passive
       ↓
Active/Active
```

As resilience increases:

```text
Cost ↑
Complexity ↑
Recovery speed ↑
```

---

# 10. Backup and Restore

Simplest model:

```text
Production
    │
    ▼
Backups
    │
    ▼
S3 / AWS Backup / service backups
```

During disaster:

```text
Disaster
   │
   ▼
Restore
   │
   ▼
Infrastructure
   │
   ▼
Application
```

This is cheaper but usually has longer recovery time.

---

# 11. Pilot Light

In Pilot Light:

```text
Primary Region
    │
    ├── EKS
    ├── RDS
    └── Services

DR Region
    │
    └── Minimal infrastructure
```

Critical data replication may continue.

During disaster:

```text
DR infrastructure
      ↓
scale up
      ↓
deploy applications
      ↓
restore/failover
      ↓
traffic switch
```

---

# 12. Warm Standby

Warm standby means the DR environment is already running.

```text
PRIMARY                    DR

EKS running                EKS running
Apps running               Apps scaled down
RDS primary                DB replica
Redis primary              replica/standby
```

During failure:

```text
DR
 ↓
scale
 ↓
promote
 ↓
traffic
```

Recovery is faster than Backup/Restore.

---

# 13. Active/Active

Both regions serve production traffic.

```text
             Users
                │
          Global Routing
          ┌─────┴─────┐
          ▼           ▼
       Region A     Region B
          │           │
         EKS         EKS
          │           │
         DB            DB
```

This can provide very strong resilience but is significantly more complicated.

Database consistency becomes a major architectural problem.

---

# 14. Our Recommended Learning Progression

For this project:

### Stage 1

```text
Multi-AZ primary Region
+
automated backups
+
tested restore
```

### Stage 2

```text
Infrastructure as Code
+
rebuild EKS
+
restore databases
```

### Stage 3

```text
Cross-region DR
+
replicated data
+
DNS/global traffic failover
```

This gives us realistic learning without immediately building an unnecessarily expensive multi-region active/active platform.

---

# 15. Multi-AZ Is Not Multi-Region

Very important.

Our production EKS:

```text
Region: ap-south-1

AZ-A
AZ-B
AZ-C
```

This protects against:

```text
single AZ failure
```

It does **not** protect against:

```text
entire AWS Region failure
```

Architecture:

```text
Region
 ├── AZ-A
 ├── AZ-B
 └── AZ-C
```

is:

> Multi-AZ.

Architecture:

```text
Region A
      +
Region B
```

is:

> Multi-Region.

---

# 16. EKS Disaster Recovery

Kubernetes workloads are mostly declarative.

If GitOps contains:

```text
Deployment
Service
Ingress
ConfigMap
HPA
PDB
NetworkPolicy
ServiceAccount
```

we can recreate them.

This is one of the major benefits of:

```text
GitOps + Infrastructure as Code
```

---

# 17. EKS DR Principle

Don't treat the EKS cluster itself as something that must be manually reconstructed from memory.

Instead:

```text
Terraform
    │
    ▼
VPC
    │
    ▼
EKS
    │
    ▼
Node capacity
```

and:

```text
GitOps
    │
    ▼
ArgoCD
    │
    ▼
Kubernetes workloads
```

Therefore:

```text
Cluster lost
   ↓
Terraform rebuilds infrastructure
   ↓
ArgoCD rebuilds workloads
```

---

# 18. What Must Be Backed Up?

Not everything has the same importance.

### Infrastructure

```text
Terraform code
Terraform state
```

### Application

```text
Git repositories
Docker images
Helm charts
GitOps manifests
```

### Data

```text
RDS
Redis where persistence matters
S3
message state where required
```

### Configuration

```text
Secrets Manager
KMS
parameter/configuration data
```

### Observability

```text
important dashboards
alert rules
logging configuration
tracing configuration
```

---

# 19. Git Is Part of DR

Suppose EKS is completely destroyed.

If we have:

```text
GitHub
Terraform
GitOps
Helm
```

we can recreate much of the platform.

This is why:

> **GitOps is also a disaster-recovery mechanism.**

But Git alone does not recover:

```text
RDS production data
customer records
orders
payments
```

Data backups are still essential.

---

# 20. Terraform State Is Critical

Our Terraform backend:

```text
S3
```

contains:

```text
terraform.tfstate
```

If the state is lost or corrupted, infrastructure management can become difficult.

Therefore state should have:

```text
encryption
versioning
restricted IAM access
audit logging
locking
backup/recovery considerations
```

Our S3 backend uses native state locking:

```hcl
terraform {
  backend "s3" {
    bucket       = "company-ecommerce-prod-terraform-state"
    key          = "eks/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

---

# 21. RDS DR

RDS is one of the most important DR components.

Our architecture:

```text
EKS
 │
 ▼
RDS PostgreSQL
 │
 ├── Multi-AZ
 ├── Automated backups
 ├── Point-in-time recovery
 └── Snapshots
```

Multi-AZ provides availability.

Backups provide recovery.

These solve different problems.

---

# 22. Multi-AZ RDS

Normal production architecture:

```text
              RDS
               │
        ┌──────┴──────┐
        ▼             ▼
   Primary AZ       Standby AZ
```

If the primary DB instance/AZ fails:

```text
RDS
 ↓
failover
 ↓
standby becomes primary
```

Application connection should use the RDS endpoint rather than hardcoding an individual instance address.

---

# 23. RDS Backup

Automated backups allow:

```text
Point-in-time recovery
```

For example:

```text
10:00
11:00
12:00
13:00
14:00
```

If corruption occurs at:

```text
13:42
```

we may restore to an appropriate point before the incident, subject to the configured retention and AWS service capabilities.

---

# 24. Snapshot vs PITR

### Snapshot

A snapshot is a point-in-time backup.

```text
Snapshot
   ↓
restore
```

### PITR

Point-in-time recovery uses backups/logs to restore to a selected point within the available recovery window.

```text
Backups + transaction/log history
              ↓
        chosen restore time
```

Both are useful.

---

# 25. RDS Cross-Region DR

For regional DR, options can include:

```text
Primary Region
      │
      ▼
Cross-region replication / backup strategy
      │
      ▼
DR Region
```

The exact mechanism depends on the chosen RDS engine, architecture, and required RPO/RTO.

Don't assume that Multi-AZ automatically provides regional DR.

It does not.

---

# 26. Database Restore Test

A backup that has never been restored is only an assumption.

We need:

```text
Backup
   ↓
Restore
   ↓
Connect
   ↓
Validate schema
   ↓
Validate data
   ↓
Validate application
```

This should be periodically tested.

---

# 27. Database Corruption Scenario

Imagine:

```text
14:00
Application bug modifies customer/order data.
```

Multi-AZ won't help.

Why?

The bad write can be replicated to the standby.

So:

```text
HA ≠ protection from logical corruption
```

Recovery may require:

```text
PITR
```

or another appropriate backup/recovery mechanism.

---

# 28. Redis DR

Redis is different from RDS.

Our architecture prefers:

```text
EKS
 │
 ▼
ElastiCache
```

Redis is generally a cache rather than the system of record.

Therefore:

```text
Redis failure
    ↓
cache miss
    ↓
application
    ↓
RDS
    ↓
rebuild cache
```

is often acceptable.

But if Redis stores important state such as sessions or other durable business-critical information, its recovery requirements must be explicitly designed.

---

# 29. Never Treat Redis as the Only Source of Truth

Bad architecture:

```text
Order
 ↓
Redis only
```

Better:

```text
Order
 │
 ├── RDS = source of truth
 │
 └── Redis = cache
```

Then Redis failure doesn't mean:

```text
customer data lost
```

---

# 30. SQS DR

SQS is a managed AWS service.

Our architecture:

```text
EKS
 │
 ▼
SQS
 │
 ├── messages
 └── DLQ
```

The important DR considerations include:

```text
queue configuration
IAM
KMS
DLQ
redrive configuration
consumer deployment
idempotency
message retention
```

We should recreate queue infrastructure through Terraform rather than manually.

---

# 31. SNS DR

Likewise:

```text
SNS Topic
   │
   ├── SQS subscription
   ├── another SQS subscription
   └── notification consumers
```

Terraform should define:

```text
topic
subscriptions
policies
KMS configuration
```

so the messaging topology can be rebuilt.

---

# 32. S3 DR

S3 can contain:

```text
reports
invoices
exports
documents
backups
logs
artifacts
```

Important capabilities include:

```text
versioning
encryption
lifecycle
access control
replication where required
```

For critical cross-region requirements, use an appropriate S3 replication strategy.

---

# 33. ECR DR

Suppose the primary ECR registry is inaccessible because of a regional incident.

Can the DR cluster pull images?

This needs planning.

Possible architecture:

```text
Primary Region
ECR
 │
 ▼
replication
 │
 ▼
DR Region ECR
```

Then:

```text
DR EKS
   │
   ▼
DR ECR
```

The exact ECR replication design should be configured according to the required regional recovery model.

---

# 34. Secrets DR

Our runtime secret flow:

```text
Application
    │
    ▼
External Secrets Operator
    │
    ▼
Secrets Manager
```

For regional DR, we need to ensure required secrets are available in the recovery environment.

Don't assume:

```text
DR EKS
+
no secrets
=
working application
```

The application will fail during startup if required secrets are unavailable.

---

# 35. KMS DR

Encryption introduces another dependency:

```text
RDS
 │
 ▼
KMS
```

and:

```text
S3
 │
 ▼
KMS
```

During DR:

```text
backup
 ↓
restore
 ↓
KMS permissions
```

must all work.

A backup that cannot be decrypted is not useful.

---

# 36. DNS Failover

Suppose:

```text
Primary Region
   ↓
production
```

fails.

We need a mechanism to route traffic toward DR.

Conceptually:

```text
                    Route 53
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Primary Region     DR Region
              │                 │
             EKS               EKS
```

Possible routing approaches include:

```text
failover routing
health checks
weighted routing
latency-based routing
```

The correct design depends on the DR architecture.

---

# 37. DNS Is Not Instantaneous

DNS failover isn't necessarily:

```text
failure
 ↓
0 seconds
 ↓
everyone moved
```

Caching and TTL behavior matter.

Therefore:

```text
DNS TTL
health checks
client caching
connection reuse
```

must be considered when defining RTO.

---

# 38. CloudFront During DR

Our application may use:

```text
Route 53
   ↓
CloudFront
   ↓
WAF
   ↓
ALB
```

If the origin changes during regional recovery:

```text
CloudFront
   ↓
DR ALB
```

we must ensure the distribution/origin configuration supports the desired failover architecture.

Don't design DNS failover without considering CloudFront's position in the traffic path.

---

# 39. Global Traffic Architecture

A possible multi-region design:

```text
                   Users
                     │
                     ▼
                Route 53
                     │
                     ▼
                CloudFront
                     │
              ┌──────┴──────┐
              ▼             ▼
         Region A       Region B
              │             │
             ALB           ALB
              │             │
             EKS           EKS
```

But the data layer is the hard part.

---

# 40. Multi-Region EKS Is the Easy Part

We can create:

```text
Region A EKS
Region B EKS
```

using Terraform.

ArgoCD can deploy workloads.

But the real questions are:

```text
Where is the database?
How is data replicated?
How do we avoid split brain?
How do payments remain consistent?
How are orders synchronized?
What happens to SQS?
What happens to Redis?
```

DR architecture is primarily a **data and dependency problem**, not just a Kubernetes problem.

---

# 41. Active/Passive Recommendation

For our learning project, a sensible advanced architecture is:

```text
PRIMARY
ap-south-1
    │
    ├── EKS
    ├── RDS
    ├── Redis
    ├── SQS/SNS
    └── S3

              ↓ DR

SECONDARY
another AWS Region
    │
    ├── EKS
    ├── replicated/restorable data
    ├── ECR
    └── required AWS services
```

Primary serves production.

Secondary is prepared for recovery.

---

# 42. DR GitOps

The GitOps repository should be capable of defining both environments.

Example:

```text
gitops/
├── environments/
│   ├── prod-ap-south-1/
│   └── dr-region/
│
├── applications/
│   ├── order-service/
│   ├── payment-service/
│   └── ...
│
└── argocd/
```

Then:

```text
ArgoCD
   │
   ├── Primary cluster
   └── DR cluster
```

---

# 43. DR Terraform

Infrastructure should also be reproducible.

```text
infrastructure/
└── terraform/
    ├── modules/
    └── environments/
        ├── prod-ap-south-1/
        └── dr-region/
```

Terraform provisions:

```text
VPC
EKS
IAM
ECR
RDS
S3
SQS
SNS
KMS
```

where appropriate.

---

# 44. Terraform and ArgoCD Boundary

Keep the responsibility clear.

### Terraform

```text
AWS infrastructure
```

### ArgoCD

```text
Kubernetes workloads
```

Don't make Terraform responsible for continuously deploying all application Pods.

Don't make ArgoCD responsible for creating the entire AWS account/VPC foundation.

---

# 45. Disaster Scenario — Single Pod

Scenario:

```text
order-service Pod crashes
```

Kubernetes:

```text
Deployment
   ↓
ReplicaSet
   ↓
new Pod
```

No DR required.

---

# 46. Disaster Scenario — Node Failure

```text
Node-A fails
```

Kubernetes:

```text
Pods unavailable
      ↓
scheduler
      ↓
new node / existing capacity
      ↓
Pods rescheduled
```

Karpenter can help provision replacement capacity where configured.

Again:

> This is resilience, not necessarily a disaster-recovery event.

---

# 47. Disaster Scenario — AZ Failure

Suppose:

```text
AZ-A unavailable
```

We designed:

```text
AZ-A
AZ-B
AZ-C
```

with replicas distributed across AZs.

Traffic:

```text
ALB
 │
 ├── AZ-B
 └── AZ-C
```

Application continues if sufficient capacity remains.

This is why:

```text
multi-AZ
+
topology spreading
+
capacity headroom
```

matter.

---

# 48. Disaster Scenario — RDS Primary Failure

```text
Application
     │
     ▼
RDS
     │
Primary fails
     │
     ▼
Multi-AZ failover
     │
     ▼
Standby
```

Applications should reconnect gracefully.

Connection pools must be configured with realistic:

```text
timeouts
connection validation
retry behavior
```

---

# 49. Disaster Scenario — Database Corruption

This is different.

```text
Application
     │
     ▼
Bad write
     │
     ▼
Database corrupted
```

Multi-AZ doesn't necessarily save us.

Recovery:

```text
PITR / backup
     ↓
restored database
     ↓
validation
     ↓
application reconnection
```

This is a true DR/data-recovery procedure.

---

# 50. Disaster Scenario — Entire EKS Cluster Lost

Suppose:

```text
Production EKS cluster
      ↓
destroyed
```

Recovery:

```text
Terraform
   ↓
VPC
   ↓
EKS
   ↓
Node capacity
   ↓
Add-ons
   ↓
ArgoCD
   ↓
Applications
```

Then:

```text
RDS
Redis
SQS
Secrets
S3
```

must be available.

---

# 51. Disaster Scenario — Region Failure

Suppose:

```text
ap-south-1
     ↓
unavailable
```

Recovery:

```text
DNS / Global routing
        ↓
DR Region
        ↓
EKS
        ↓
ArgoCD
        ↓
Applications
        ↓
DR data services
```

This requires much more preparation.

---

# 52. The DR Runbook

A production DR plan should not be:

> "Everyone meet in a call and figure it out."

It should be a runbook.

Example:

```text
1. Declare incident
2. Confirm disaster scope
3. Freeze normal deployments
4. Determine RTO/RPO impact
5. Protect remaining data
6. Activate DR
7. Provision/scale infrastructure
8. Validate database
9. Validate secrets
10. Validate messaging
11. Deploy applications
12. Run smoke tests
13. Validate business transactions
14. Switch traffic
15. Monitor
16. Communicate recovery
17. Investigate primary failure
18. Plan failback
```

---

# 53. Freeze Deployments

During major DR:

```text
GitOps changes
+
Terraform changes
+
application deployments
```

should normally be controlled/frozen unless required for recovery.

Why?

Because we don't want:

```text
disaster
+
deployment
+
Terraform change
+
database migration
```

all happening simultaneously.

---

# 54. Recovery Validation

Never say:

> "Pods are Running, DR is complete."

We need:

```text
Infrastructure
      ↓
Kubernetes
      ↓
Application
      ↓
Database
      ↓
Messaging
      ↓
External integrations
      ↓
Business transaction
```

For order-service:

```text
Create order
   ↓
RDS record
   ↓
event published
   ↓
inventory/payment workflow
   ↓
notification
```

---

# 55. DR Smoke Tests

Example:

```text
GET /actuator/health
```

Then:

```text
POST /api/v1/orders
```

Validate:

```text
HTTP response
database record
event
downstream processing
logs
metrics
trace
```

Only then declare the service operational.

---

# 56. Data Validation

After database recovery:

```text
Count important tables
Check recent orders
Check payment state
Check inventory state
Check constraints
Check indexes
Check application migrations
```

Don't blindly assume:

```text
restore successful
=
data correct
```

---

# 57. Message Validation

After recovery:

```text
SQS
 ↓
consumer
```

Check:

```text
queue depth
old messages
DLQ
visibility timeout
consumer health
duplicate processing
```

For Kafka/RabbitMQ, recovery requires corresponding:

```text
offsets
partitions
queues
consumer groups
replication
```

to be validated.

---

# 58. DR and Idempotency

During recovery, messages may be:

```text
replayed
retried
redelivered
```

Therefore consumers need idempotency.

Example:

```text
Order Event
orderId = 123
```

If processed twice:

```text
first → create inventory reservation
second → detect already processed
```

rather than:

```text
first → reserve 1
second → reserve another 1
```

DR makes idempotency even more important.

---

# 59. Failback

Recovering into DR is only half the story.

Eventually we may want:

```text
DR Region
    ↓
Primary Region
```

again.

That's called:

> **Failback.**

A proper runbook includes:

```text
Primary repaired
     ↓
Synchronize data
     ↓
Validate primary
     ↓
Test
     ↓
Gradual traffic movement
     ↓
Primary active
```

---

# 60. Don't Rush Failback

A common mistake:

```text
Primary restored
      ↓
Immediately switch traffic
```

Instead:

```text
Primary restored
      ↓
Infrastructure validation
      ↓
Data validation
      ↓
Application validation
      ↓
Smoke tests
      ↓
Controlled traffic switch
```

---

# 61. DR and Security

During disaster recovery, security controls must remain intact.

Don't weaken:

```text
IAM
NetworkPolicy
TLS
Secrets
KMS
authentication
authorization
```

just to make recovery faster.

A DR environment with excessive permissions can create a security incident on top of the original disaster.

---

# 62. DR and Observability

The DR region needs observability too.

Don't wait until disaster day to discover:

```text
Prometheus not installed
Grafana inaccessible
logs not flowing
alerts not configured
Jaeger unavailable
```

DR must include:

```text
metrics
logs
traces
alerts
dashboards
```

appropriate to the recovery objectives.

---

# 63. DR Monitoring

We should monitor:

```text
DR cluster
DR nodes
DR Pods
DR database
DR Redis
DR queues
DR application
DR traffic
```

And also:

```text
replication lag
backup age
backup success
restore readiness
```

---

# 64. Backup Monitoring

A useful alert is:

```text
Last successful backup
      ↓
too old
      ↓
ALERT
```

For example:

```text
Backup age > required RPO
```

This is more useful than simply monitoring whether the backup job exists.

---

# 65. DR Testing

This is critical.

A DR plan that exists only in a README is not a DR plan.

We need periodic tests.

Examples:

```text
Restore RDS snapshot
Rebuild EKS
Restore S3 data
Deploy applications into DR
Test DNS failover
Validate messaging
Run business transactions
```

---

# 66. Game Days

A **DR Game Day** is a controlled disaster simulation.

Example:

```text
Scenario:

Primary EKS unavailable
```

Team performs:

```text
detect
→ declare
→ recover
→ validate
→ failover
→ monitor
```

Measure:

```text
actual RTO
actual RPO
```

Then compare against requirements.

---

# 67. Example Game Day

```text
09:00
Simulated production disaster

09:05
Incident declared

09:10
DR activated

09:25
EKS available

09:35
Database recovered

09:45
Applications healthy

09:50
Smoke tests passed

09:55
Traffic switched

10:00
Recovery complete
```

Measured:

```text
RTO = 55 minutes
```

If target was:

```text
RTO = 30 minutes
```

we have a gap.

---

# 68. DR Gap Analysis

After every test:

```text
Expected
   vs
Actual
```

Example:

| Metric       | Target | Actual | Result |
| ------------ | -----: | -----: | ------ |
| RTO          | 30 min | 55 min | ❌      |
| RPO          |  5 min |  3 min | ✅      |
| EKS recovery | 20 min | 18 min | ✅      |
| DB recovery  | 15 min | 30 min | ❌      |

Then improve the system.

---

# 69. Common DR Mistakes

### Mistake 1

```text
"I have backups."
```

But nobody tested restoring them.

---

### Mistake 2

```text
"We have Multi-AZ."
```

But no regional DR.

---

### Mistake 3

```text
"EKS can be recreated."
```

But RDS data cannot.

---

### Mistake 4

```text
"Terraform can rebuild everything."
```

But Terraform state is unavailable.

---

### Mistake 5

```text
"DR has EKS."
```

But:

```text
Secrets missing
ECR missing
KMS permissions missing
DNS missing
```

---

# 70. Another Common Mistake — DR Environment Is Stale

Suppose production runs:

```text
order-service v1.6.0
```

but DR only has:

```text
order-service v1.1.0
```

Recovery now becomes a deployment problem.

GitOps should keep DR configuration aligned with the intended recovery state.

---

# 71. Immutable Images Help DR

If ECR contains:

```text
order-service:1.6.0
```

but the tag can be overwritten, recovery could accidentally pull a different image.

Prefer:

```text
image digest
```

for strong immutability.

Example:

```text
order-service@sha256:...
```

Then DR gets the exact same artifact.

---

# 72. DR Audit Trail

A good recovery should answer:

```text
Which Git commit?
Which Terraform version?
Which image digest?
Which database backup?
Which recovery point?
Which GitOps revision?
Which ArgoCD revision?
Who approved failover?
When did traffic switch?
```

This gives us an auditable recovery chain.

---

# 73. Complete DR Architecture

```text
                         USERS
                           │
                           ▼
                    Global DNS / Routing
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
          PRIMARY REGION          DR REGION
                │                     │
               ALB                   ALB
                │                     │
               EKS                   EKS
                │                     │
       ┌────────┼────────┐            │
       ▼        ▼        ▼            │
      RDS     Redis     SQS            │
       │        │        │             │
       └────────┼────────┘             │
                │                      │
          Backup / Replication ───────┘
                │
                ▼
          S3 / Backup Storage
```

---

# 74. DR Control Plane

The recovery system itself should be automated where practical:

```text
GitHub
   │
   ├── Terraform
   │
   └── GitOps
         │
         ▼
       ArgoCD
         │
         ▼
      DR EKS
```

This reduces manual recovery steps.

---

# 75. DR Responsibility Matrix

| Component    | Primary DR Mechanism                          |
| ------------ | --------------------------------------------- |
| VPC          | Terraform                                     |
| EKS          | Terraform                                     |
| Nodes        | Managed Node Groups/Karpenter                 |
| Applications | GitOps/ArgoCD                                 |
| Helm         | Git                                           |
| ECR          | Replication/rebuild strategy                  |
| RDS          | Multi-AZ + backups/PITR + regional strategy   |
| Redis        | ElastiCache HA/recovery strategy              |
| SQS          | Terraform + managed service                   |
| SNS          | Terraform + managed service                   |
| S3           | Versioning/replication/backup strategy        |
| Secrets      | Secrets Manager replication/recovery strategy |
| KMS          | Regional key/recovery design                  |
| DNS          | Route 53                                      |
| Logs         | Elasticsearch/S3 retention strategy           |
| Metrics      | Prometheus/managed metrics strategy           |
| Traces       | Jaeger/storage strategy                       |

---

# 76. Our DR Design — Practical Version

For the first production implementation, use:

```text
PRIMARY REGION
────────────────────────

3 AZ VPC
    │
    ▼
EKS
    │
    ├── 21 services
    ├── ArgoCD
    ├── Prometheus
    ├── Grafana
    ├── ELK
    └── OTel/Jaeger

RDS PostgreSQL
    │
    └── Multi-AZ

ElastiCache
SQS/SNS
S3
Secrets Manager
KMS
ECR

Backups
    │
    ▼
Protected recovery storage
```

Then add:

```text
DR REGION
────────────────────────

Terraform-reproducible VPC/EKS
+
GitOps
+
required AWS services
+
replicated/restorable data
```

---

# 77. Recovery Flow

When a major disaster occurs:

```text
                    DISASTER
                       │
                       ▼
                  Incident
                  Declared
                       │
                       ▼
                 Assess Scope
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Recover Primary    Activate DR
              │                 │
              │                 ▼
              │              EKS Ready
              │                 │
              │                 ▼
              │             Data Ready
              │                 │
              │                 ▼
              │          Applications Ready
              │                 │
              │                 ▼
              │            Smoke Tests
              │                 │
              │                 ▼
              └────────────► Traffic
                              │
                              ▼
                           DR Active
```

---

# 78. Most Important DR Principles

Remember these:

```text
1. Multi-AZ ≠ Multi-Region

2. HA ≠ Backup

3. Backup ≠ Tested Recovery

4. EKS recovery ≠ Data recovery

5. Redis ≠ RDS

6. GitOps helps rebuild applications

7. Terraform helps rebuild infrastructure

8. RPO measures acceptable data loss

9. RTO measures acceptable downtime

10. DR must be tested
```

---

# 79. Final Mental Model

Think of DR in four layers:

```text
                 DISASTER RECOVERY
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
 Infrastructure         Data          Application
        │                │                │
     Terraform          RDS             GitOps
     VPC                S3              ArgoCD
     EKS                Redis           Helm
     IAM                SQS             Images
     KMS                backups         Secrets
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                    Traffic / DNS
                         │
                         ▼
                     Users
```

And the ultimate goal is:

```text
Disaster
   ↓
Detect
   ↓
Assess
   ↓
Recover infrastructure
   ↓
Recover data
   ↓
Recover applications
   ↓
Validate
   ↓
Switch traffic
   ↓
Monitor
   ↓
Failback when safe
```

> **Production-grade DR means you don't merely have a backup or a second EKS cluster—you have a tested, measurable process that can restore the business within the agreed RTO/RPO.**

**Next: `46-Backup-and-Restore.md`** — we will go one level deeper into the actual backup/recovery implementation: **RDS automated backups, PITR, snapshots, S3 versioning/replication, EBS/PVC considerations, EKS recovery, Terraform state recovery, ECR recovery, Secrets Manager/KMS, SQS/SNS configuration recovery, backup retention, restore procedures, validation, and a realistic DR restore exercise.**
