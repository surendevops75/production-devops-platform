# 97 — EKS Backup and Recovery Architecture

> Production-grade recovery guide for rebuilding the EKS platform when the cluster, nodes, workloads, or supporting AWS services fail.

---

# 1. Why EKS Backup Is Different

A common mistake is thinking:

```text
EKS backup
    =
backup the EKS cluster
```

That is incomplete.

Our platform contains many independent layers:

```text
                    AWS ACCOUNT
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       VPC              IAM             KMS
        │                │                │
        ▼                ▼                ▼
       EKS             Roles           Keys
        │
 ┌──────┼─────────┐
 │      │         │
Nodes  Pods     Add-ons
 │      │
 │      ├── Applications
 │      ├── ConfigMaps
 │      ├── Secrets references
 │      └── PVCs
 │
 └───────────────────────┐
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
      RDS              Redis             SQS/SNS
       │
       ▼
    Business Data
```

There is no single backup operation that magically restores all of this.

---

# 2. What Should Be Recoverable?

For our enterprise e-commerce platform, recovery should cover:

```text
Infrastructure
├── VPC
├── Subnets
├── Route tables
├── NAT
├── VPC endpoints
├── Security Groups
├── IAM
├── KMS
└── EKS

Platform
├── EKS add-ons
├── ArgoCD
├── Kyverno
├── Argo Rollouts
├── Prometheus
├── Fluent Bit
├── Logstash
├── Elasticsearch
└── Grafana

Applications
├── Helm charts
├── GitOps configuration
├── Docker images
└── Kubernetes manifests

Data
├── RDS
├── Redis
├── S3
├── SQS
└── SNS

Security
├── Secrets Manager
├── KMS
├── CloudTrail
└── Security configuration
```

---

# 3. Recovery Philosophy

Our architecture follows:

> **Rebuild infrastructure from code and recover stateful data from managed backups.**

Instead of trying to manually recreate a broken cluster:

```text
Old EKS
   │
   X
   │
   └── manually repair everything
```

we aim for:

```text
Terraform
    │
    ▼
New VPC/EKS
    │
    ▼
Platform installation
    │
    ▼
ArgoCD
    │
    ▼
GitOps
    │
    ▼
Applications
    │
    ▼
Recovered data
```

This is much more repeatable.

---

# 4. Two Different Things: Backup and Rebuild

These are not the same.

## Backup

Preserve data/configuration so it can be restored.

## Rebuild

Create infrastructure again.

Example:

```text
Terraform
```

is primarily a **rebuild mechanism**, not a backup of application data.

Similarly:

```text
GitHub
```

is the source of truth for code/configuration, but it is not a replacement for database backups.

---

# 5. What Terraform Can Recover

Terraform can recreate resources such as:

```text
VPC
Subnets
Route Tables
NAT
Security Groups
IAM
ECR
EKS
RDS configuration
Redis configuration
SQS
SNS
S3
KMS
```

depending on how the infrastructure is defined.

But Terraform cannot magically recover:

```text
PostgreSQL rows
Redis cached state
EBS filesystem contents
Elasticsearch historical data
Secrets values
```

unless those are separately backed up/recovered.

---

# 6. Recovery Layers

We divide recovery into six layers:

```text
Layer 1
AWS Foundation
      ↓
Layer 2
EKS Platform
      ↓
Layer 3
GitOps
      ↓
Layer 4
Applications
      ↓
Layer 5
Data
      ↓
Layer 6
Traffic
```

This gives us a predictable recovery sequence.

---

# 7. Layer 1 — AWS Foundation

Recover:

```text
VPC
├── CIDR
├── Public subnets
├── Private subnets
├── Route tables
├── Internet Gateway
├── NAT
├── VPC endpoints
└── DNS

Security
├── IAM
├── KMS
└── Security Groups
```

Our Terraform repository already contains most of this.

```text
infrastructure/
└── terraform/
    ├── modules/
    └── environments/
```

Therefore:

```bash
terraform init
terraform plan
terraform apply
```

can rebuild the foundation, assuming the required Terraform state/backend and external dependencies are available.

---

# 8. Terraform State Is Critical

Our Terraform backend is:

```text
S3
 +
S3 versioning
 +
S3 native state locking
```

Example:

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

If this bucket is lost, recovering Terraform becomes much harder.

Therefore:

> Terraform state itself requires protection and recovery planning.

---

# 9. Protecting Terraform State

Our state bucket should have:

```text
Versioning
Encryption
Public access blocked
Restricted IAM
Bucket owner enforced
Audit logging
Recovery process
```

Check:

```bash
aws s3api get-bucket-versioning \
  --bucket company-ecommerce-prod-terraform-state
```

---

# 10. Terraform State Is Not Application Backup

Suppose:

```text
Terraform state
```

contains:

```text
RDS instance
EKS cluster
VPC
```

That does **not** mean:

```text
Terraform state
=
PostgreSQL backup
```

For example, Terraform may know:

```text
RDS identifier = ecommerce-prod-db
```

but it doesn't contain every PostgreSQL row.

Therefore:

```text
Terraform
    +
RDS backups
    +
Git
```

are all required.

---

# 11. Layer 2 — Recreate EKS

After VPC and IAM:

```text
VPC
 ↓
IAM
 ↓
EKS
 ↓
Node capacity
 ↓
EKS add-ons
```

Terraform recreates the EKS cluster.

Example:

```bash
terraform plan
```

Review carefully.

Then:

```bash
terraform apply
```

---

# 12. Verify New EKS Cluster

Configure access:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-prod
```

Check:

```bash
kubectl get nodes
```

Initially you may see:

```text
NAME                     STATUS
ip-10-0-xx-xx            Ready
ip-10-0-xx-xx            Ready
```

Then:

```bash
kubectl get pods -A
```

---

# 13. Reinstall EKS Add-ons

Our required add-ons include:

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI
Pod Identity Agent
AWS Load Balancer Controller
Metrics Server
```

The exact versions must be selected for the recreated cluster's Kubernetes version.

Validate:

```bash
aws eks list-addons \
  --cluster-name ecommerce-prod \
  --region ap-south-1
```

---

# 14. Layer 3 — Restore GitOps

This is where our architecture becomes powerful.

We don't want to manually recreate:

```text
21 Deployments
21 Services
21 HPAs
21 PDBs
Ingresses
ConfigMaps
Rollouts
Policies
```

Instead:

```text
GitHub
   │
   ▼
GitOps repository
   │
   ▼
ArgoCD
   │
   ▼
Kubernetes
```

ArgoCD reconciles the cluster toward the desired state stored in Git.

---

# 15. Install ArgoCD

Create namespace:

```bash
kubectl create namespace argocd
```

Install the approved ArgoCD version through the platform's normal Helm/GitOps process.

Then:

```bash
kubectl get pods -n argocd
```

Expected components include:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
```

Exact components depend on the ArgoCD deployment/version.

---

# 16. Connect ArgoCD to Git

ArgoCD needs access to:

```text
order-service repository
gitops repository
platform repository
```

Private repository credentials must be recreated securely.

Do **not** put GitHub private keys directly into:

```text
Git repository
Helm values
ConfigMaps
```

Use the approved secret-management mechanism.

---

# 17. Restore Applications

Our ApplicationSet architecture becomes extremely useful.

Remember:

```text
gitops/
└── environments/
    ├── dev/
    ├── qa/
    ├── staging/
    └── prod/
```

ArgoCD/ApplicationSet reads:

```text
app.yaml
values.yaml
```

and recreates the applications.

For example:

```text
order-service
catalog-service
payment-service
inventory-service
...
```

---

# 18. 21 Microservices Recovery

The recovery process becomes:

```text
GitOps
  │
  ▼
ApplicationSet
  │
  ├── auth-service
  ├── user-service
  ├── customer-service
  ├── catalog-service
  ├── product-service
  ├── inventory-service
  ├── pricing-service
  ├── promotion-service
  ├── cart-service
  ├── order-service
  ├── payment-service
  ├── fraud-service
  ├── shipping-service
  ├── delivery-service
  ├── notification-service
  ├── search-service
  ├── recommendation-service
  ├── review-service
  ├── tax-service
  ├── invoice-service
  └── reporting-service
```

No manual deployment of 21 services should be required.

---

# 19. Layer 4 — Container Images

Applications need their images.

Our images are stored in:

```text
Amazon ECR
```

Example:

```text
ECR
├── order-service
├── payment-service
├── inventory-service
└── ...
```

Check:

```bash
aws ecr describe-repositories \
  --region ap-south-1
```

---

# 20. Why ECR Is Part of DR

Suppose:

```text
EKS cluster deleted
```

but ECR still exists.

We can rebuild:

```text
EKS
 ↓
ArgoCD
 ↓
GitOps
 ↓
ECR image
 ↓
Pods
```

without rebuilding the application artifacts.

This is another reason to separate:

```text
build
```

from:

```text
deployment
```

---

# 21. ECR Cross-Region DR

For stronger DR, images can be replicated to another AWS region.

Conceptually:

```text
Mumbai
ECR
 │
 │ replication
 ▼
DR Region
ECR
```

Then if the primary region becomes unavailable:

```text
GitOps
  ↓
DR EKS
  ↓
DR ECR
```

The exact ECR replication design should be managed through Terraform.

---

# 22. Layer 5 — RDS Recovery

RDS contains our business data.

This is one of the most important recovery components.

Our database architecture:

```text
EKS
 │
 ▼
RDS PostgreSQL
 │
 ├── Automated backups
 ├── PITR
 ├── Snapshots
 └── Multi-AZ
```

---

# 23. RDS Point-in-Time Recovery

Suppose:

```text
14:00
Everything healthy

14:35
Bad deployment corrupts data

15:00
Incident discovered
```

If PITR is available:

```text
Backup history
      │
      ▼
14:34:59
      │
      ▼
Restore
```

The exact recovery point depends on the available backup/transaction-log retention and AWS capabilities.

---

# 24. RDS Snapshot Recovery

Another recovery method:

```text
RDS
 ↓
Snapshot
 ↓
New RDS instance
```

Then update application configuration to point to the recovered database.

Example:

```text
old:
ecommerce-prod-db

new:
ecommerce-prod-db-recovery
```

Then:

```text
Secrets Manager
 ↓
database endpoint
 ↓
application
```

must be updated appropriately.

---

# 25. Database Recovery Principle

Do not immediately point production applications at a recovered database.

First:

```text
Restore
 ↓
Validate
 ↓
Check schema
 ↓
Check data
 ↓
Check application compatibility
 ↓
Controlled cutover
```

---

# 26. Database Schema Recovery

Our application uses:

```text
Flyway / Liquibase
```

for controlled schema changes.

Therefore:

```text
Application version
+
Database schema version
```

must be compatible.

Example:

```text
Application v2
requires
DB migration V15
```

A database restored to a state before V15 may not support the application.

---

# 27. Expand-Contract and Recovery

For production deployments:

```text
Old application
+
New application
```

may temporarily coexist.

Therefore database changes should generally follow:

```text
Expand
 ↓
Deploy compatible application
 ↓
Migrate data
 ↓
Switch application behavior
 ↓
Contract
```

This makes both deployment and recovery safer.

---

# 28. Redis Recovery

Our primary Redis architecture is:

```text
Amazon ElastiCache
```

But Redis is generally treated differently from RDS.

Why?

Because Redis often contains:

```text
Cache
Session
Temporary state
Rate-limit state
```

depending on the application.

If it is purely cache:

```text
RDS = source of truth
Redis = rebuildable cache
```

Then during DR:

```text
Restore EKS
 ↓
Restore application
 ↓
Redis starts empty
 ↓
Applications repopulate cache
```

---

# 29. When Redis Is Not Just Cache

If Redis contains business-critical state:

```text
Redis
=
source of truth
```

then it requires its own backup and recovery design.

Do not assume Redis is always disposable.

Document exactly what each keyspace represents.

---

# 30. SQS Recovery

SQS configuration can be recreated through Terraform.

For example:

```text
Queue
DLQ
Redrive policy
Encryption
IAM
```

Terraform:

```text
Terraform
   ↓
SQS
```

But existing messages are a separate concern.

A recreated queue does not automatically contain messages that existed in a deleted queue.

Therefore determine:

```text
Do messages need to survive regional disaster?
```

before choosing the architecture.

---

# 31. SNS Recovery

Similarly:

```text
SNS Topic
 ↓
Subscriptions
```

can be recreated from infrastructure code.

But verify:

```text
Topic
Subscriptions
IAM
KMS
Consumers
```

after recreation.

---

# 32. S3 Recovery

Our platform may use S3 for:

```text
Invoices
Reports
Documents
Exports
Artifacts
Terraform state
```

These are potentially business-critical.

For important buckets consider:

```text
Versioning
Encryption
Lifecycle
Replication
Access controls
Object Lock where appropriate
```

---

# 33. S3 Cross-Region Replication

For critical data:

```text
Primary Region
     │
     │ replication
     ▼
DR Region
```

Example:

```text
ap-south-1
   │
   ▼
DR region
```

This protects against a regional failure, not merely accidental deletion in the same region.

---

# 34. Secrets Recovery

Our runtime secrets come from:

```text
AWS Secrets Manager
```

Examples:

```text
prod/ecommerce/order-service/database
prod/ecommerce/payment-service/credentials
prod/ecommerce/notification-service/credentials
```

These secrets need their own recovery plan.

---

# 35. Why Secrets Are Special

Suppose:

```text
EKS
```

is recreated.

The Kubernetes Secret objects may not exist yet.

Our architecture:

```text
Secrets Manager
       │
       ▼
External Secrets Operator
       │
       ▼
Kubernetes Secret
       │
       ▼
Pod
```

Therefore after recovery:

```text
Secrets Manager
 ↓
ESO
 ↓
Kubernetes Secret
 ↓
Application
```

must be verified.

---

# 36. KMS Recovery

Encryption depends on KMS.

Examples:

```text
RDS
S3
EBS
Secrets Manager
ECR
Terraform state
```

may depend on KMS keys.

Therefore:

> KMS keys and their policies are part of the DR design.

Do not casually delete encryption keys.

---

# 37. EKS Cluster Recovery Scenario

Now let's simulate the major incident.

## Scenario

```text
02:00 AM

Production EKS cluster becomes unrecoverable.
```

We need to restore:

```text
Applications
Data
Networking
Security
Traffic
Observability
```

---

# 38. Step 1 — Declare Incident

Create incident:

```text
INC-20501
```

Then identify:

```text
Incident Commander
Technical Lead
Communications
AWS/Cloud owner
Database owner
Application owners
```

Use the incident-management process from our earlier chapters.

---

# 39. Step 2 — Stop Changes

Immediately freeze unrelated deployments.

```text
No application deployments
No infrastructure changes
No database migrations
```

except changes required for recovery.

---

# 40. Step 3 — Determine Blast Radius

Ask:

```text
Is only EKS broken?

Or:

VPC?
IAM?
RDS?
S3?
ECR?
KMS?
AWS Region?
```

This determines the recovery path.

---

# 41. Step 4 — Verify RDS

Check:

```bash
aws rds describe-db-instances \
  --region ap-south-1
```

If RDS is healthy:

```text
Do not restore it unnecessarily.
```

This is important.

Recovery does not mean rebuilding everything.

---

# 42. Step 5 — Verify ECR

```bash
aws ecr describe-repositories \
  --region ap-south-1
```

Check the required image:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

---

# 43. Step 6 — Verify S3

Check critical buckets.

```bash
aws s3api head-bucket \
  --bucket <BUCKET_NAME>
```

Terraform state is particularly important.

---

# 44. Step 7 — Rebuild Infrastructure

Run the approved Terraform recovery process:

```text
VPC
 ↓
IAM
 ↓
Security Groups
 ↓
EKS
 ↓
Node capacity
 ↓
EKS add-ons
```

---

# 45. Step 8 — Install Platform

```text
ArgoCD
Kyverno
External Secrets
AWS Load Balancer Controller
Metrics Server
Argo Rollouts
Monitoring
Logging
```

The platform itself should be defined as code/GitOps as much as practical.

---

# 46. Step 9 — Restore Secrets

Verify:

```text
Secrets Manager
 ↓
External Secrets
 ↓
Kubernetes Secrets
```

Example:

```bash
kubectl get externalsecret -A
```

Then:

```bash
kubectl get secret -n ecommerce-prod
```

Do not print secret values into logs.

---

# 47. Step 10 — Restore Applications

ArgoCD:

```text
GitOps
 ↓
ApplicationSet
 ↓
Applications
 ↓
Helm
 ↓
Deployments
```

Check:

```bash
argocd app list
```

Then:

```bash
kubectl get pods -A
```

---

# 48. Step 11 — Restore Data Connectivity

Applications should connect to:

```text
RDS
Redis
SQS
SNS
S3
```

Validate IAM and network access.

For example:

```text
Order Service
     │
     ├── RDS
     ├── Redis
     ├── SQS
     ├── S3
     └── Secrets Manager
```

---

# 49. Step 12 — Recreate ALB

AWS Load Balancer Controller processes the Ingress.

```text
Ingress
 ↓
AWS Load Balancer Controller
 ↓
ALB
```

Check:

```bash
kubectl get ingress -A
```

Wait for the ALB to become healthy.

---

# 50. Step 13 — Route 53

Once the new ALB is healthy:

```text
Route 53
    ↓
ALB
    ↓
EKS
```

If the ALB DNS name changed, DNS routing must point to the correct target.

With CloudFront:

```text
Route 53
 ↓
CloudFront
 ↓
WAF
 ↓
ALB
 ↓
EKS
```

The external architecture can remain stable while the backend cluster is rebuilt.

---

# 51. Step 14 — Smoke Test

Start with:

```text
Health
 ↓
Authentication
 ↓
Catalog
 ↓
Cart
 ↓
Order
 ↓
Payment
 ↓
Notification
```

Don't immediately run every possible business operation.

Validate the critical path first.

---

# 52. Step 15 — Observe

Watch:

```text
Prometheus
Grafana
ELK
Jaeger
AWS CloudWatch
ALB metrics
RDS metrics
Redis metrics
SQS metrics
```

For at least the defined post-recovery observation period.

---

# 53. Step 16 — Re-enable Normal Deployments

Only after:

```text
Infrastructure healthy
+
Applications healthy
+
Data healthy
+
Monitoring healthy
+
Smoke tests passed
```

should the deployment freeze be removed.

---

# 54. Step 17 — Close Incident

Document:

```text
What happened?
When?
Why?
What was affected?
What was recovered?
How long?
What data was lost?
What data was restored?
Were RTO/RPO met?
What failed during recovery?
What should change?
```

Then create:

```text
Problem Record
```

if a longer-term root-cause investigation is required.

---

# 55. Recovery Architecture

The complete recovery path:

```text
                   INCIDENT
                       │
                       ▼
                Incident Command
                       │
                       ▼
              Assess Blast Radius
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
   AWS Foundation OK            Foundation Lost
         │                           │
         └─────────────┬─────────────┘
                       ▼
                 Terraform
                       │
                       ▼
                    VPC/IAM
                       │
                       ▼
                     EKS
                       │
                       ▼
                 EKS Add-ons
                       │
                       ▼
                    ArgoCD
                       │
                       ▼
                    GitOps
                       │
                       ▼
                  Applications
                       │
            ┌──────────┼───────────┐
            ▼          ▼           ▼
           RDS       Redis       Messaging
            │
            ▼
         Validate
            │
            ▼
          ALB
            │
            ▼
       Route 53/CF
            │
            ▼
        Smoke Tests
            │
            ▼
         Monitoring
            │
            ▼
          Recovery
```

---

# 56. RTO and RPO

Two important DR measurements:

## RTO

**Recovery Time Objective**

How quickly must the service be restored?

Example:

```text
RTO = 60 minutes
```

Means the recovery target is to restore service within the defined 60-minute objective.

---

## RPO

**Recovery Point Objective**

How much data loss is acceptable?

Example:

```text
RPO = 5 minutes
```

means the architecture targets recovery to a point within approximately five minutes of the incident, subject to the actual backup/replication mechanisms.

---

# 57. Different Services Can Have Different RTO/RPO

Do not automatically assign one value to everything.

Example:

| Component       | Example RTO |     Example RPO |
| --------------- | ----------: | --------------: |
| Order DB        |      30 min |           5 min |
| Product catalog |      60 min |          15 min |
| Redis cache     |      60 min |     Rebuildable |
| S3 invoices     |      60 min |        Very low |
| Metrics         |        4 hr |   Less critical |
| Logs            |        4 hr |         Depends |
| EKS platform    |      60 min | Git-rebuildable |

These are **example targets**, not universal requirements. Actual values should come from business requirements.

---

# 58. Backup vs Replication

These are different.

### Backup

```text
Database
 ↓
Backup
```

Useful for:

```text
Accidental deletion
Corruption
Point-in-time recovery
```

### Replication

```text
Primary
 ↓
Secondary
```

Useful for:

```text
Regional failure
Lower recovery time
```

Replication does not automatically protect against logical corruption because bad changes may also replicate.

---

# 59. Same-Region DR

Example:

```text
ap-south-1
       │
       ▼
EKS failure
       │
       ▼
Rebuild EKS
       │
       ▼
Same RDS
```

This is simpler.

---

# 60. Cross-Region DR

More resilient architecture:

```text
             Primary
          ap-south-1
               │
      ┌────────┼────────┐
      │        │        │
     ECR      S3       RDS
      │        │        │
      └────────┼────────┘
               │
          Replication
               │
               ▼
             DR
        <DR REGION>
               │
              EKS
```

The DR region should not be treated as complete simply because EKS can be created there.

You must validate:

```text
IAM
KMS
ECR
S3
RDS
Secrets
DNS
Networking
```

---

# 61. Route 53 Failover

A regional DR design can use DNS-based failover.

Conceptually:

```text
                Route 53
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Primary             DR
          │                 │
      CloudFront/ALB     CloudFront/ALB
          │                 │
         EKS               EKS
```

Exact routing depends on the CloudFront/ALB architecture.

DNS failover also requires health-check and TTL considerations.

---

# 62. Velero

Velero can be useful for Kubernetes resource and volume backup/recovery scenarios.

Conceptually:

```text
Kubernetes
    │
    ▼
Velero
    │
    ├── Kubernetes resources
    │
    └── Backup storage
```

But don't misunderstand its role.

Velero does not replace:

```text
RDS backups
Terraform
ECR
Secrets Manager
S3 replication
```

It can complement them.

---

# 63. When Velero Is Useful

Potential use cases:

```text
Namespace recovery
Kubernetes resource recovery
Migration
Cluster-to-cluster recovery
Certain PVC/volume recovery workflows
```

For our architecture:

```text
Terraform
+
GitOps
+
AWS managed backups
+
Velero where justified
```

is more appropriate than expecting Velero alone to recover the whole platform.

---

# 64. What Should Be Rebuilt vs Restored?

| Component     | Strategy                                  |
| ------------- | ----------------------------------------- |
| VPC           | Terraform rebuild                         |
| IAM           | Terraform rebuild                         |
| EKS           | Terraform rebuild                         |
| Node groups   | Terraform rebuild                         |
| Karpenter     | Terraform/GitOps                          |
| ArgoCD        | GitOps/Helm                               |
| Applications  | GitOps                                    |
| Docker images | ECR                                       |
| RDS           | Backup/PITR/snapshot                      |
| Redis         | Recreate/restore based on role            |
| SQS           | Terraform + message strategy              |
| SNS           | Terraform                                 |
| S3            | Replication/backup                        |
| Secrets       | Secrets Manager                           |
| KMS           | Protected key architecture                |
| Monitoring    | GitOps/rebuild                            |
| Logs          | Depends on retention/storage architecture |

---

# 65. The Golden Rule

Do not build your DR process around:

```text
SSH into server
kubectl manually
copy YAML
hope it works
```

Instead:

```text
Code
 ↓
Terraform
 ↓
GitOps
 ↓
ArgoCD
 ↓
Automated reconciliation
```

This makes disaster recovery repeatable.

---

# 66. DR Drill

A production-grade team should periodically perform a controlled DR exercise.

Example:

```text
DR GAME DAY
```

Scenario:

```text
Production EKS unavailable
```

Team must recover the service without relying on undocumented manual steps.

Measure:

```text
Start time
First infrastructure recovery
EKS ready
ArgoCD ready
Applications ready
Database accessible
ALB healthy
Smoke tests passed
Traffic restored
```

---

# 67. DR Runbook

Create:

```text
docs/runbooks/eks-disaster-recovery.md
```

Example:

```markdown
# EKS Disaster Recovery

## Incident
Production EKS unavailable.

## Preconditions
- Terraform repository available
- Terraform state available
- GitOps repository available
- ECR available
- RDS recovery available
- Secrets Manager available

## Recovery

1. Declare incident.
2. Freeze unrelated changes.
3. Determine blast radius.
4. Validate RDS.
5. Validate ECR.
6. Validate Terraform state.
7. Rebuild infrastructure.
8. Create EKS.
9. Install add-ons.
10. Install ArgoCD.
11. Restore secrets integration.
12. Reconcile GitOps.
13. Validate applications.
14. Validate ALB.
15. Validate DNS.
16. Run smoke tests.
17. Monitor.
18. Close incident.
```

---

# 68. Recovery Verification Checklist

## Infrastructure

```text
[ ] VPC
[ ] Subnets
[ ] Routes
[ ] NAT
[ ] Security Groups
[ ] IAM
[ ] KMS
[ ] EKS
```

## Kubernetes

```text
[ ] Nodes
[ ] VPC CNI
[ ] CoreDNS
[ ] kube-proxy
[ ] EBS CSI
[ ] Pod Identity
```

## Platform

```text
[ ] ArgoCD
[ ] ApplicationSets
[ ] Kyverno
[ ] Argo Rollouts
[ ] External Secrets
[ ] AWS Load Balancer Controller
```

## Applications

```text
[ ] 21 services
[ ] Images
[ ] Config
[ ] Secrets
[ ] Services
[ ] Ingress
[ ] HPA
[ ] PDB
```

## Data

```text
[ ] RDS
[ ] Redis
[ ] SQS
[ ] SNS
[ ] S3
```

## Observability

```text
[ ] Prometheus
[ ] Grafana
[ ] Fluent Bit
[ ] Logstash
[ ] Elasticsearch
[ ] Kibana
[ ] OTel
[ ] Jaeger
```

---

# 69. What We Should Automate

Eventually, recovery should become:

```text
./recover-production.sh
```

or preferably an approved automation pipeline.

Conceptually:

```text
Recovery Pipeline
       │
       ├── Validate Terraform state
       ├── Validate AWS prerequisites
       ├── Terraform plan
       ├── Terraform apply
       ├── Wait for EKS
       ├── Install platform
       ├── Validate ArgoCD
       ├── Sync GitOps
       ├── Validate workloads
       ├── Validate data
       └── Run smoke tests
```

Do not make a fully automatic production recovery pipeline blindly destructive. Human approval should exist at appropriate points.

---

# 70. Final Disaster Recovery Architecture

```text
                       GIT
                        │
          ┌─────────────┴─────────────┐
          │                           │
     Terraform                      GitOps
          │                           │
          ▼                           ▼
      AWS Infra                    ArgoCD
          │                           │
     ┌────┼────┐                ┌─────┼─────┐
     │    │    │                │     │     │
     VPC IAM  EKS             Apps  Policies Secrets
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
     Nodes   Addons  Pods
                       │
       ┌───────────────┼──────────────┐
       ▼               ▼              ▼
      RDS             Redis          SQS
       │
       ▼
    Backups
       │
       ▼
   DR Region
```

---

# 71. The Most Important Mental Model

For this project, remember:

```text
                CODE
                  │
                  ▼
             GITHUB/GIT
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
    Terraform             GitOps
        │                   │
        ▼                   ▼
 AWS Infrastructure       ArgoCD
        │                   │
        ▼                   ▼
      EKS                Workloads
        │                   │
        └─────────┬─────────┘
                  ▼
                DATA
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
       RDS       S3        SQS
                  │
                  ▼
              BACKUPS
                  │
                  ▼
               RECOVERY
```

**EKS is replaceable. Business data is not.**

That principle drives the entire architecture.

---

## Next Chapter

### `98-EKS-Multi-Region-Disaster-Recovery.md`

Next we can move from **“rebuild the EKS cluster”** to a much deeper **multi-region production DR architecture**, covering:

* Primary Mumbai → DR region architecture
* active/passive vs active/active
* Route 53 failover
* CloudFront/WAF
* cross-region ECR
* RDS cross-region strategy
* S3 replication
* Secrets Manager/KMS considerations
* EKS cluster provisioning in the DR region
* GitOps for two clusters
* ArgoCD multi-cluster
* database/application consistency
* SQS/SNS regional considerations
* regional failure simulation
* exact failover sequence
* failback sequence
* RTO/RPO testing
* a complete **“Mumbai region is down”** production incident scenario.
