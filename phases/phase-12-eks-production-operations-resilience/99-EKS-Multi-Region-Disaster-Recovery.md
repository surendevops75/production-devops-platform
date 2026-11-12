# 99 — EKS Multi-Region Disaster Recovery

> Production-grade architecture for recovering the e-commerce platform when the **entire primary AWS Region becomes unavailable**, including EKS, ECR, RDS, S3, Secrets Manager, Route 53, CloudFront/WAF, GitOps, ArgoCD, and application traffic.

---

## 1. Why Multi-Region DR?

Our previous chapter handled:

```text
EKS cluster failure
```

inside the same AWS Region.

Now consider a much larger failure:

```text
ap-south-1
Mumbai
   X
Entire region unavailable
```

Our production platform must be able to operate from another AWS Region.

Conceptually:

```text
             GLOBAL USERS
                  |
                  v
             Route 53
                  |
          +-------+-------+
          |               |
          v               v
       PRIMARY           DR
    ap-south-1        DR Region
          |               |
        EKS             EKS
          |               |
        Apps             Apps
          |               |
        Data             Data
```

---

# 2. Important: DR Is Not Automatically Active-Active

There are several architectures.

### Active/Passive

```text
Primary = serving traffic
DR      = standby
```

### Warm Standby

```text
Primary = serving traffic
DR      = smaller running environment
```

### Active/Active

```text
Primary = serving traffic
DR      = also serving traffic
```

For our learning project, we will primarily design:

> **Active/Passive with a warm or partially warm DR environment.**

This provides a practical balance between complexity and recovery capability.

---

# 3. Primary Architecture

Primary:

```text
ap-south-1
```

Example:

```text
Route 53
   |
CloudFront
   |
WAF
   |
ALB
   |
EKS
   |
21 Microservices
   |
+--------+--------+--------+
|        |        |        |
RDS    Redis     SQS      S3
```

---

# 4. DR Architecture

Suppose our DR Region is:

```text
<DR_REGION>
```

For example, the actual region should be selected based on:

* AWS service availability
* latency
* compliance
* cost
* data residency
* customer requirements

Architecture:

```text
<DR_REGION>
     |
     v
   VPC
     |
     v
   EKS
     |
     v
21 Microservices
     |
+----+----+----+
|    |    |    |
RDS Redis SQS S3
```

---

# 5. Global Architecture

```text
                         USERS
                           |
                           v
                       Route 53
                           |
                    DNS / Failover
                           |
              +------------+------------+
              |                         |
              v                         v
          PRIMARY                       DR
       ap-south-1                   DR Region
              |                         |
          CloudFront                 CloudFront
              |                         |
             WAF                       WAF
              |                         |
             ALB                       ALB
              |                         |
             EKS                       EKS
              |                         |
        21 Services               21 Services
              |                         |
        +-----+-----+             +-----+-----+
        |     |     |             |     |     |
       RDS  Redis  SQS            RDS  Redis  SQS
```

---

# 6. What Must Be Replicated?

We need to think about every layer.

```text
Infrastructure
├── VPC
├── Subnets
├── Routes
├── NAT
├── Security Groups
├── IAM
└── EKS

Artifacts
├── ECR images
└── Helm charts/Git

Data
├── RDS
├── S3
├── Redis
└── Messaging

Secrets
├── Secrets Manager
└── KMS

Traffic
├── Route 53
├── CloudFront
└── WAF

Platform
├── ArgoCD
├── Kyverno
├── Monitoring
└── Logging
```

---

# 7. Git Is Our Global Source of Truth

Our application code exists in GitHub.

```text
GitHub
   |
   +── application repositories
   |
   +── infrastructure
   |
   +── gitops
   |
   +── platform
```

This is extremely useful for DR.

If:

```text
Primary Region = DOWN
```

GitHub remains available.

Then:

```text
GitHub
   |
   v
Terraform
   |
   v
DR AWS Region
```

and:

```text
GitHub
   |
   v
ArgoCD
   |
   v
DR EKS
```

---

# 8. Terraform Multi-Region Design

We should not duplicate Terraform manually.

Use reusable modules.

Example:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   ├── rds/
    │   ├── redis/
    │   ├── ecr/
    │   ├── s3/
    │   └── ...
    │
    └── environments/
        ├── primary/
        │   └── ap-south-1/
        │
        └── dr/
            └── <DR_REGION>/
```

---

# 9. Primary Terraform

Example:

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

---

# 10. DR Terraform

Conceptually:

```hcl
provider "aws" {
  region = var.aws_region
}
```

Then:

```hcl
aws_region = "<DR_REGION>"
```

The actual target region should be selected according to the business DR requirements.

---

# 11. Separate Terraform State

Do not use one ambiguous state key for both regions.

Example:

```text
S3 Terraform State
│
├── primary/ap-south-1/terraform.tfstate
│
└── dr/<DR_REGION>/terraform.tfstate
```

This makes regional infrastructure independently manageable.

---

# 12. Why Separate State?

Imagine:

```text
Primary infrastructure
```

and:

```text
DR infrastructure
```

are accidentally managed by the same incorrect state configuration.

A mistake could affect both environments.

Separate state gives clearer boundaries.

---

# 13. DR EKS

The DR cluster should be reproducible.

Terraform creates:

```text
DR VPC
   |
   v
DR EKS
   |
   +── System node capacity
   +── Application capacity
   +── Karpenter
   +── EKS add-ons
```

---

# 14. Don't Treat DR EKS as a Manual Cluster

Bad:

```text
Primary EKS
    |
    | manually copy settings
    v
DR EKS
```

Better:

```text
Terraform
    |
    +------> Primary
    |
    +------> DR
```

Same modules.

Different environment/region inputs.

---

# 15. GitOps for Multiple Clusters

Our GitOps repository can represent:

```text
gitops/
├── clusters/
│   ├── primary/
│   └── dr/
│
└── environments/
    ├── primary/
    └── dr/
```

Another valid model is separate cluster/environment directories while keeping application definitions reusable.

The important principle is:

> **Git must clearly identify which cluster receives which configuration.**

---

# 16. ArgoCD Multi-Cluster

ArgoCD can manage multiple Kubernetes clusters.

Conceptually:

```text
                    ArgoCD
                       |
          +------------+------------+
          |                         |
          v                         v
    Primary EKS                  DR EKS
    ap-south-1                  DR Region
          |                         |
       Apps                       Apps
```

---

# 17. Where Should ArgoCD Run?

There are two major approaches.

### Option A

ArgoCD runs in the primary cluster.

```text
Primary EKS
   |
 ArgoCD
   |
   +----> Primary EKS
   |
   +----> DR EKS
```

Problem:

If the primary region disappears:

```text
Primary EKS
    X
    |
ArgoCD unavailable
```

DR automation becomes more difficult.

---

# 18. Better DR Architecture

For stronger resilience, the GitOps control plane itself should not depend solely on the region it is supposed to recover.

Possible architecture:

```text
Management / Dedicated Platform
             |
             v
           ArgoCD
             |
       +-----+-----+
       |           |
       v           v
    Primary       DR
      EKS          EKS
```

Alternatively, maintain an independently recoverable ArgoCD instance in the DR environment.

The correct architecture depends on your organization's platform strategy.

---

# 19. GitHub Remains Outside AWS Region

This is valuable.

```text
AWS Primary
     X

GitHub
     |
     v
Terraform
     |
     v
DR AWS
```

Your source control and deployment definitions remain available even if the primary AWS region is unavailable.

---

# 20. ECR Replication

Primary:

```text
ECR Primary
     |
     | replication
     v
ECR DR
```

Example:

```text
order-service
```

exists in both regions.

Therefore the DR cluster doesn't need to rebuild the image.

---

# 21. Why Image Digest Matters

Suppose production runs:

```text
order-service@sha256:ABC123
```

DR should deploy:

```text
order-service@sha256:ABC123
```

not:

```text
order-service:latest
```

The same artifact should be used.

---

# 22. ECR DR Validation

Before an incident:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region <DR_REGION>
```

Confirm the expected image exists.

For each critical service, verify replication.

---

# 23. RDS Is the Hard Part

EKS can be recreated relatively easily.

Database recovery is more complicated.

Primary:

```text
RDS PostgreSQL
```

DR:

```text
RDS PostgreSQL
```

Possible mechanisms include:

* cross-region read replica
* snapshot copy
* backup/PITR-based recovery
* other AWS-supported database DR architectures

The right choice depends on RPO/RTO requirements.

---

# 24. Cross-Region RDS Read Replica

Conceptually:

```text
Primary RDS
ap-south-1
     |
     | replication
     v
DR RDS
DR Region
```

During normal operation:

```text
Primary = source
DR = replica
```

During a disaster:

```text
promote DR
```

The exact promotion/failover procedure must be tested and documented.

---

# 25. RDS DR Is Not Just "Create Another Database"

Suppose:

```text
Primary DB
```

contains:

```text
10 million orders
```

Creating a blank DR database gives you:

```text
0 orders
```

That isn't DR.

We need:

```text
Data replication / backup
+
schema compatibility
+
credentials
+
networking
+
application connection
```

---

# 26. Database Connection During Failover

Our applications should not hard-code the database endpoint.

Instead:

```text
Secrets Manager
      |
      v
External Secrets
      |
      v
Kubernetes Secret
      |
      v
Application
```

Then the database endpoint can be changed through controlled configuration.

---

# 27. Database Failover Example

Normal:

```text
order-service
      |
      v
prod DB endpoint
      |
      v
Primary RDS
```

After DR:

```text
order-service
      |
      v
DR DB endpoint
      |
      v
DR RDS
```

The change must be controlled, validated, and observable.

---

# 28. S3 Cross-Region Replication

Important S3 data:

```text
Invoices
Reports
Documents
Exports
```

can use cross-region replication where required.

```text
Primary S3
    |
    | CRR
    v
DR S3
```

Remember:

> Terraform state and business-data buckets may have different DR requirements.

---

# 29. Secrets Manager DR

Secrets require special planning.

Conceptually:

```text
Primary Secrets Manager
          |
          v
DR secret strategy
```

AWS supports different approaches depending on the secret and architecture.

Do not assume every secret automatically exists in another region.

For critical secrets, explicitly test:

```text
Secret exists
KMS works
IAM works
External Secrets works
Application retrieves secret
```

---

# 30. KMS

Encryption introduces another dependency.

Example:

```text
RDS
 |
KMS

S3
 |
KMS

Secrets Manager
 |
KMS
```

DR therefore requires an appropriate encryption-key strategy in the DR region.

Never build a DR process that assumes a regional KMS dependency will magically work after a regional outage.

---

# 31. Redis DR

For Redis:

```text
Primary ElastiCache
        |
        v
DR strategy
```

But first determine:

```text
Is Redis cache?
```

If yes:

```text
RDS = source of truth
Redis = rebuildable
```

DR can recreate the cache.

If Redis contains critical state:

```text
Redis
=
business data
```

then it needs a much stronger replication/recovery strategy.

---

# 32. SQS DR

Queues are regional.

Therefore:

```text
Primary SQS
```

and:

```text
DR SQS
```

are separate resources.

Terraform should be able to create both.

But messages already present in the primary queue require explicit DR design.

---

# 33. Messaging DR Strategy

For critical event-driven systems:

```text
Producer
   |
   v
Primary Queue
```

and:

```text
DR Producer
   |
   v
DR Queue
```

The system must define:

```text
What happens to unprocessed messages?
How are duplicates handled?
Can events be replayed?
Is idempotency implemented?
```

---

# 34. Route 53 Failover

This is how users can move between regions.

Normal:

```text
User
 |
 v
Route 53
 |
 v
Primary
```

Disaster:

```text
User
 |
 v
Route 53
 |
 v
DR
```

The exact mechanism can use health checks or other routing strategies appropriate to the architecture.

---

# 35. CloudFront Consideration

Our architecture uses:

```text
Route 53
   |
CloudFront
   |
WAF
   |
ALB
```

CloudFront is global.

The origin can be designed so that regional failure can be handled without changing every client.

For example:

```text
CloudFront
   |
   +---- Primary ALB
   |
   +---- DR ALB
```

The exact failover design depends on the origin architecture.

---

# 36. WAF

WAF rules should be managed as code where practical.

```text
Terraform
   |
   v
WAF configuration
```

This means DR can reproduce the same security policy.

Verify:

```text
Managed rules
Custom rules
Rate limits
IP rules
Logging
```

---

# 37. DR Application Architecture

Primary:

```text
21 Services
```

DR:

```text
21 Services
```

The same container images should be used.

Environment-specific differences should primarily be:

```text
region
endpoints
capacity
data dependencies
traffic configuration
```

not application binaries.

---

# 38. Same Artifact Principle

Example:

```text
Git commit:
8f31a2c

Image:
order-service:8f31a2c

Digest:
sha256:ABC...
```

Primary:

```text
order-service@sha256:ABC...
```

DR:

```text
order-service@sha256:ABC...
```

This gives artifact consistency.

---

# 39. DR Capacity Choices

There are three common approaches.

### Cold

```text
No EKS running
```

Cheapest, slower recovery.

### Warm

```text
EKS exists
small capacity
```

Faster recovery.

### Hot

```text
Full application capacity
```

Expensive but faster.

For our project:

> A **warm standby** is a useful practical target.

---

# 40. Warm Standby

Normal state:

```text
PRIMARY
Full capacity
    |
    v
Users

DR
Reduced capacity
    |
    v
Ready to scale
```

During disaster:

```text
DR
 |
 +-- scale nodes
 +-- scale applications
 +-- promote data
 +-- switch traffic
```

---

# 41. Karpenter Helps DR

Suppose DR starts with:

```text
small baseline capacity
```

When traffic switches:

```text
Traffic increases
      |
      v
HPA
      |
      v
Pending Pods
      |
      v
Karpenter
      |
      v
New EC2 nodes
```

This allows DR to scale without permanently running the entire primary capacity.

---

# 42. But Karpenter Is Not Instantaneous

Do not design:

```text
Traffic suddenly arrives
   ↓
Karpenter instantly gives 100 nodes
```

There are delays:

```text
Pod scaling
Node provisioning
EC2 startup
Image pull
Application startup
Readiness
ALB health
```

Therefore DR capacity planning must account for startup time.

---

# 43. DR Application Scaling

Before failover:

```text
DR replicas:
2
```

After failover:

```text
DR replicas:
10
```

or use HPA/Karpenter.

This should be controlled by GitOps or an approved DR automation process.

---

# 44. DR Failover Sequence

Now the important part.

Suppose:

```text
Mumbai region unavailable.
```

Runbook:

```text
1. Declare regional disaster
2. Freeze normal changes
3. Confirm primary outage
4. Confirm DR infrastructure
5. Promote/recover database
6. Validate DR secrets
7. Validate DR ECR
8. Validate DR EKS
9. Scale DR capacity
10. Deploy/sync applications
11. Validate ALB
12. Validate application
13. Switch traffic
14. Run smoke tests
15. Monitor
```

---

# 45. Step 1 — Declare Disaster

Create:

```text
INC-DR-001
```

Assign:

```text
Incident Commander
DR Technical Lead
Database Lead
Application Lead
Network Lead
Communications Lead
```

---

# 46. Step 2 — Freeze Changes

Stop:

```text
normal deployments
Terraform changes
database migrations
nonessential GitOps changes
```

Only approved recovery actions continue.

---

# 47. Step 3 — Confirm Primary Failure

Do not fail over simply because one Pod is down.

Confirm:

```text
AWS regional impact
EKS
ALB
RDS
networking
AWS service health
```

This prevents unnecessary failovers.

---

# 48. Step 4 — Validate DR Infrastructure

```bash
aws eks describe-cluster \
  --name ecommerce-dr \
  --region <DR_REGION>
```

Then:

```bash
kubectl get nodes
```

and:

```bash
kubectl get pods -A
```

---

# 49. Step 5 — Validate DR Database

Before allowing application traffic:

```text
Database available
Replication state acceptable
Expected recovery point
Schema valid
Credentials valid
```

Then perform application-level validation.

---

# 50. Step 6 — Promote Database

For architectures using a cross-region replica, perform the documented promotion process.

Do not improvise during the incident.

Record:

```text
Promotion start
Promotion completion
Recovered timestamp
Data-loss assessment
```

---

# 51. Step 7 — Update Application Database Endpoint

Update the DR configuration through GitOps/Secrets Manager.

Example:

```text
Secrets Manager
      |
      v
DR DB endpoint
      |
      v
ExternalSecret
      |
      v
Kubernetes Secret
      |
      v
Pods
```

Restart/redeploy workloads only as required by the secret-consumption method.

---

# 52. Step 8 — Validate ECR

Verify critical images exist in DR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region <DR_REGION>
```

Repeat for critical services.

---

# 53. Step 9 — Scale DR

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get hpa -A
```

If using Karpenter:

```bash
kubectl get nodepool
```

Ensure capacity can handle the expected production load.

---

# 54. Step 10 — Sync GitOps

ArgoCD should reconcile:

```text
DR GitOps
   |
   v
DR EKS
```

Check:

```bash
argocd app list
```

Then:

```bash
argocd app get order-service-dr
```

---

# 55. Step 11 — Validate All Services

Start with critical services:

```text
auth
catalog
cart
order
payment
inventory
notification
```

Then validate the remaining services.

---

# 56. Step 12 — Validate ALB

```bash
kubectl get ingress -A
```

Then:

```bash
kubectl describe ingress <NAME> \
  -n <NAMESPACE>
```

Verify:

```text
ALB exists
target groups healthy
listeners correct
security groups correct
TLS valid
```

---

# 57. Step 13 — Switch Traffic

Only after:

```text
DR infrastructure healthy
+
database healthy
+
applications healthy
+
ALB healthy
+
smoke tests passed
```

change the traffic routing.

Conceptually:

```text
Route 53
   |
   X Primary
   |
   v
DR
```

---

# 58. Step 14 — Smoke Test

Test:

```text
DNS
HTTPS
Authentication
Catalog
Cart
Order
Payment
```

For example:

```bash
curl -I https://api.example.com/health
```

Then execute the critical business workflow.

---

# 59. Step 15 — Monitor

Immediately watch:

```text
5xx
4xx
Latency
Request rate
Pod restarts
HPA
Karpenter
RDS
Redis
SQS
ALB
```

Do not declare success merely because the homepage opens.

---

# 60. Failback

Once the primary region recovers:

```text
Primary
   |
   v
Rebuild/validate
   |
   v
Data synchronization
   |
   v
Application validation
   |
   v
Traffic migration
   |
   v
Primary becomes active again
```

Failback is another change event.

---

# 61. Never Immediately Fail Back

Bad:

```text
Primary is back
   ↓
switch traffic immediately
```

Better:

```text
Primary recovered
   ↓
validate infrastructure
   ↓
validate database
   ↓
validate applications
   ↓
validate observability
   ↓
test
   ↓
controlled traffic migration
```

---

# 62. Data Consistency During Failback

This is one of the hardest problems.

During DR:

```text
DR becomes writer
```

When primary returns:

```text
Primary database may be stale.
```

You need a documented process for:

```text
DR → Primary synchronization
```

before switching users back.

---

# 63. Split-Brain Risk

Never allow:

```text
Primary
  ↓
writes

AND

DR
  ↓
writes
```

unless the application/database architecture explicitly supports active-active writes.

Otherwise:

```text
Two writers
    |
    v
Data divergence
```

can become catastrophic.

---

# 64. Active/Passive Simpler Model

Normal:

```text
PRIMARY
Writer
  |
Users

DR
Standby
```

Disaster:

```text
PRIMARY
X

DR
Writer
  |
Users
```

Failback:

```text
DR
Writer
  |
sync
  |
PRIMARY
```

Then controlled traffic migration.

---

# 65. DR Testing

A DR design that has never been tested is only a hypothesis.

Perform:

```text
DR Game Day
```

Example:

```text
Simulated regional outage
```

Measure:

```text
Detection time
Decision time
Infrastructure recovery
Database recovery
Application recovery
Traffic switch
Total RTO
Data loss
```

---

# 66. Example RTO/RPO Test

Suppose target:

```text
RTO = 60 minutes
RPO = 5 minutes
```

Test:

```text
10:00
Primary failure simulated

10:05
Incident declared

10:15
DR database promoted

10:25
EKS validated

10:35
Applications healthy

10:45
Traffic switched

10:50
Smoke tests complete
```

Measured:

```text
RTO = 50 minutes
```

Then determine actual data recovery point and compare it to the RPO target.

---

# 67. DR Test Failure

Suppose:

```text
DR EKS works
but payment-service fails.
```

That means the DR test discovered a dependency problem.

Investigate:

```text
payment secrets
payment endpoint
KMS
network
ECR
external provider
```

Fix the architecture.

Do not simply mark the DR test as successful because EKS itself started.

---

# 68. DR Dependency Matrix

For every service document:

| Service      | Database | Redis | SQS | S3    | External API | Secrets |
| ------------ | -------- | ----- | --- | ----- | ------------ | ------- |
| order        | Yes      | Yes   | Yes | Maybe | No           | Yes     |
| payment      | Yes      | Maybe | Yes | No    | Yes          | Yes     |
| catalog      | Yes      | Yes   | No  | Maybe | No           | Yes     |
| notification | Maybe    | No    | Yes | No    | Yes          | Yes     |

This matrix becomes part of the DR runbook.

---

# 69. DR Ownership

| Component        | Owner             |
| ---------------- | ----------------- |
| Terraform        | DevOps            |
| EKS              | Platform          |
| ArgoCD           | Platform          |
| RDS              | DBA/Platform      |
| Redis            | Platform          |
| Messaging        | Platform          |
| Application      | Backend team      |
| DNS              | Network/Platform  |
| WAF              | Security/Platform |
| Secrets          | Security/Platform |
| Observability    | SRE               |
| Incident command | SRE/Engineering   |

---

# 70. Multi-Region Security

DR must have equivalent security controls.

Do not create:

```text
Primary:
secure

DR:
wide-open
```

DR should have:

```text
IAM
Security Groups
NetworkPolicy
Pod Security
Kyverno
Secrets
KMS
WAF
TLS
Logging
Audit
```

---

# 71. DR Observability

The DR cluster must have monitoring **before** the disaster.

At minimum:

```text
Prometheus
Grafana
CloudWatch
CloudTrail
logging
alerts
```

If observability only exists in the failed region, you may lose visibility during recovery.

---

# 72. DR Alerting

Alerts should reach an external channel:

```text
EKS
 |
Alertmanager
 |
External notification
 |
On-call engineer
```

Don't rely only on a dashboard inside the failed region.

---

# 73. DR Logging

Critical audit/security logs should have a retention strategy independent of the primary EKS cluster.

Examples:

```text
CloudTrail
VPC Flow Logs
security findings
application audit logs
```

For critical logs, use appropriate centralized storage/aggregation.

---

# 74. Cost of Multi-Region DR

Multi-region DR increases cost:

```text
Second VPC
Second EKS
Second nodes
ECR replication
RDS replication
S3 replication
Load balancers
NAT
observability
storage
data transfer
```

Therefore choose the architecture based on:

```text
Business impact
RTO
RPO
Compliance
Cost
```

---

# 75. What Should Always Be Available?

For a strong DR design:

```text
GitHub
Terraform
GitOps
ECR artifacts
Terraform state
Recovery documentation
Secrets recovery process
Database backups/replication
DNS
```

These are recovery dependencies.

---

# 76. Complete Multi-Region Architecture

```text
                              USERS
                                |
                                v
                            Route 53
                                |
                       Global Traffic Control
                                |
                +---------------+---------------+
                |                               |
                v                               v
          PRIMARY REGION                  DR REGION
          ap-south-1                      <DR_REGION>
                |                               |
          CloudFront                         CloudFront
                |                               |
               WAF                             WAF
                |                               |
               ALB                             ALB
                |                               |
               EKS                             EKS
                |                               |
          21 Services                      21 Services
                |                               |
       +--------+--------+              +-------+-------+
       |        |        |              |       |       |
      RDS     Redis     SQS            RDS    Redis    SQS
       |        |        |              |       |       |
       +--------+--------+              +-------+-------+
                |
           Data replication
                |
                v
             DR DATA


                 GLOBAL CONTROL
                      |
        +-------------+-------------+
        |                           |
        v                           v
   Terraform                      GitHub
        |                           |
        +-------------+-------------+
                      |
                      v
                  GitOps
                      |
                 +----+----+
                 |         |
                 v         v
             Primary      DR
              ArgoCD     ArgoCD
                 |         |
                 v         v
                EKS       EKS
```

---

# 77. The Most Important Principle

Multi-region DR is not:

```text
"Create another EKS cluster."
```

It is:

```text
Infrastructure
+
Artifacts
+
Data
+
Secrets
+
Networking
+
Security
+
Traffic
+
Observability
+
Automation
+
People
+
Runbooks
```

all working together.

---

# 78. Final Failover Checklist

```text
[ ] Incident declared
[ ] Primary outage confirmed
[ ] Change freeze active
[ ] DR infrastructure healthy
[ ] DR EKS healthy
[ ] EKS add-ons healthy
[ ] ECR images available
[ ] Database promoted/recovered
[ ] RPO verified
[ ] Secrets available
[ ] KMS available
[ ] Redis available/rebuildable
[ ] SQS/SNS available
[ ] GitOps available
[ ] ArgoCD healthy
[ ] 21 services healthy
[ ] ALB healthy
[ ] WAF healthy
[ ] DNS ready
[ ] Smoke tests passed
[ ] Traffic switched
[ ] Monitoring active
[ ] RTO measured
[ ] Incident timeline updated
```

---

# 79. Final Failback Checklist

```text
[ ] Primary region recovered
[ ] Primary infrastructure validated
[ ] Primary EKS validated
[ ] Primary database validated
[ ] Data synchronized
[ ] Primary applications validated
[ ] Primary ALB validated
[ ] Primary observability validated
[ ] Failback change approved
[ ] Traffic moved gradually
[ ] Smoke tests passed
[ ] Primary confirmed healthy
[ ] DR returned to standby
[ ] Incident closed
[ ] Postmortem completed
```

---

# 80. Final Mental Model

Your entire production platform now looks like:

```text
                    GLOBAL USERS
                         |
                         v
                     Route 53
                         |
             +-----------+-----------+
             |                       |
             v                       v
          PRIMARY                   DR
             |                       |
            EKS                     EKS
             |                       |
         Applications           Applications
             |                       |
          Data Layer              Data Layer
             |                       |
             +-----------+-----------+
                         |
                    Replication
                         |
                         v
                  Backup / Recovery


                 ENGINEERING CONTROL
                         |
          +--------------+--------------+
          |              |              |
       GitHub        Terraform       GitOps
          |              |              |
          |              v              |
          |             AWS             |
          |                             |
          +-------------+---------------+
                        |
                      ArgoCD
                        |
                 Kubernetes API
                        |
                  EKS Workloads
```

> **The real goal of multi-region DR is not to keep a second cluster alive. The goal is to make the entire production service recoverable from an independent region using tested infrastructure, immutable artifacts, recoverable data, controlled traffic switching, and documented operational procedures.**

---