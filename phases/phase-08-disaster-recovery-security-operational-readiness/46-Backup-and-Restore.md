# 46 — Backup & Restore

> **Enterprise E-Commerce Platform | AWS | EKS | RDS PostgreSQL | ElastiCache | S3 | SQS/SNS | ECR | Secrets Manager | KMS | Terraform | ArgoCD**

Backup and Restore is the practical side of Disaster Recovery.

In the previous chapter, we established:

```text
DR
├── RTO → How quickly can we recover?
└── RPO → How much data can we afford to lose?
```

Now we answer:

> **What exactly do we back up, where do we store it, how do we restore it, and how do we prove that the restore actually works?**

---

# 1. Backup Is Not DR

A backup is only one component of DR.

```text
Backup
   ↓
Restore
   ↓
Validate
   ↓
Recover application
   ↓
Switch traffic
```

A production backup strategy is incomplete if:

```text
❌ Backup exists but cannot be restored
❌ Encryption key is unavailable
❌ Backup is in the same failure domain
❌ Application configuration is missing
❌ Secrets are missing
❌ Terraform state is missing
❌ Docker image is unavailable
```

---

# 2. What We Need to Protect

Our platform contains several different types of data.

```text
Enterprise E-Commerce
│
├── Application Source
├── Infrastructure Code
├── GitOps Configuration
├── Docker Images
├── Database Data
├── Object Data
├── Messaging Configuration
├── Secrets
├── Encryption Keys
├── Kubernetes State
└── Observability Data
```

Each requires a different recovery mechanism.

---

# 3. Backup Responsibility Matrix

| Component              | Primary Recovery Mechanism                   |
| ---------------------- | -------------------------------------------- |
| GitHub repositories    | Git history / repository backup policy       |
| Terraform code         | Git                                          |
| Terraform state        | S3 versioning + controlled recovery          |
| Helm charts            | Git                                          |
| GitOps manifests       | Git                                          |
| ECR images             | ECR retention/replication strategy           |
| RDS                    | Automated backups + PITR + snapshots         |
| Redis                  | Service-specific backup/replication strategy |
| S3                     | Versioning + replication where required      |
| SQS/SNS                | Terraform configuration                      |
| Secrets Manager        | Secret replication/recovery strategy         |
| KMS                    | Key architecture / regional strategy         |
| EKS workloads          | GitOps                                       |
| EKS persistent volumes | Snapshot/backup strategy where required      |
| Logs                   | Elasticsearch/S3 backup strategy             |
| Metrics                | Prometheus/managed metrics strategy          |
| Traces                 | Jaeger backend/storage strategy              |

---

# 4. Three Backup Questions

For every critical component ask:

### What?

```text
What data/configuration is being protected?
```

### Where?

```text
Where is the backup stored?
```

### How?

```text
How do we restore it?
```

Then add a fourth:

### Have we tested it?

```text
Can we actually restore it?
```

---

# 5. RDS Is Our Most Important Backup

Our order/customer/payment data lives primarily in:

```text
RDS PostgreSQL
```

Example:

```text
orders
customers
payments
inventory
products
```

We should use:

```text
Automated Backups
+
Point-in-Time Recovery
+
Snapshots
```

where appropriate.

---

# 6. RDS Automated Backups

Conceptually:

```text
RDS
 │
 ├── Automated backup
 │
 ├── Transaction/log data
 │
 └── Recovery window
```

The recovery window determines how far back we can perform supported point-in-time recovery.

Example:

```text
Retention = 7 days
```

could allow recovery within the configured backup window, subject to AWS service behavior.

Production retention should be selected based on:

```text
RPO
compliance
business requirements
storage/cost
```

—not simply "7 days because the example says 7."

---

# 7. Point-in-Time Recovery

Imagine:

```text
10:00 → normal
11:00 → normal
12:00 → normal
13:00 → application bug
13:15 → bad data discovered
```

We may need to recover to:

```text
12:59
```

rather than restoring only the latest snapshot.

Conceptually:

```text
Backup history
     │
     ▼
Choose recovery point
     │
     ▼
Restore database
```

---

# 8. PITR Is Not the Same as Snapshot Restore

### Snapshot

```text
RDS
 ↓
Snapshot
 ↓
Restore
```

### PITR

```text
RDS
 ↓
automated backup/log history
 ↓
selected timestamp
 ↓
restore
```

PITR is especially useful for:

```text
accidental deletion
logical corruption
bad migration
bad application deployment
```

---

# 9. Example RDS Backup Configuration

Our RDS Terraform can include:

```hcl
resource "aws_db_instance" "this" {
  identifier = var.identifier

  engine         = "postgres"
  engine_version = var.engine_version

  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage
  storage_type      = "gp3"

  db_name  = var.db_name
  username = var.username

  manage_master_user_password = true

  backup_retention_period = 7

  backup_window = "18:00-19:00"

  maintenance_window = "sun:19:00-sun:20:00"

  multi_az = true

  storage_encrypted = true

  publicly_accessible = false

  deletion_protection = true

  skip_final_snapshot = false
}
```

The exact supported arguments should match the AWS provider version used by our implementation.

---

# 10. Backup Window

Avoid concentrating backups and maintenance with peak application traffic if the service's operational behavior makes that undesirable.

For example:

```text
Application peak:
10:00–22:00
```

Potential maintenance/backup windows can be placed during a lower-traffic period, subject to the actual AWS service behavior and business requirements.

---

# 11. RDS Snapshot

A snapshot can be created before major changes.

For example:

```text
Before major DB migration
        ↓
Create snapshot
        ↓
Run migration
        ↓
Validate
```

This gives us an additional recovery point.

But:

> A snapshot should not be treated as the only backup mechanism.

---

# 12. Snapshot Before Dangerous Migration

Suppose we're changing:

```text
orders table
```

from:

```text
old schema
```

to:

```text
new schema
```

A safe approach can include:

```text
1. Test migration
2. Verify rollback/restore procedure
3. Snapshot if appropriate
4. Run migration
5. Validate
6. Monitor
```

But database migrations should still be designed to be backward compatible.

A snapshot is not a substitute for good migration design.

---

# 13. Restore to a New RDS Instance

One important operational principle:

> Don't restore over the production database blindly.

Instead:

```text
Backup
   ↓
Restore
   ↓
New RDS instance
   ↓
Validate
   ↓
Decide next action
```

For example:

```text
production-db
      │
      ▼
backup
      │
      ▼
restore-db-validation
```

Then verify:

```text
schema
data
indexes
constraints
application connectivity
```

---

# 14. Database Restore Validation

After restore:

```text
psql
  │
  ├── connect
  ├── list databases
  ├── inspect schema
  ├── check important tables
  ├── check recent records
  └── run validation queries
```

For example:

```bash
psql -h <endpoint> -U <user> -d orders
```

The PostgreSQL client should normally be available from a controlled admin/ops environment rather than installed in every application Pod.

---

# 15. Application-Level Restore Validation

Database restore is not enough.

Deploy:

```text
order-service
```

against the restored database.

Then:

```text
GET /actuator/health
```

and:

```text
POST /api/v1/orders
```

Validate:

```text
API
 ↓
Spring Boot
 ↓
RDS
 ↓
transaction
```

---

# 16. Full Business Validation

For an order:

```text
Create Order
      │
      ▼
RDS
      │
      ▼
Order Event
      │
      ▼
SQS
      │
      ▼
Consumer
      │
      ▼
Inventory / Payment
```

A proper recovery test should validate the business flow, not only the database connection.

---

# 17. RDS Restore Scenario

Suppose an engineer accidentally executes:

```sql
DELETE FROM orders;
```

at:

```text
13:40
```

and the problem is discovered at:

```text
13:45
```

The recovery process may be:

```text
Incident
   ↓
Stop further destructive activity
   ↓
Determine recovery point
   ↓
PITR to new DB
   ↓
Validate data
   ↓
Determine missing records
   ↓
Plan application reconnection
   ↓
Controlled recovery
```

The exact recovery procedure depends on the incident and business requirements.

---

# 18. Don't Immediately Destroy the Original Database

During a serious recovery:

```text
Original DB
     │
     ├── Preserve evidence
     └── Recovery source
```

Avoid making the situation worse by immediately deleting or overwriting the original system.

Incident response and data recovery should be coordinated.

---

# 19. Terraform State Backup

Our Terraform state lives in S3:

```text
S3
└── terraform.tfstate
```

Enable:

```text
versioning
encryption
restricted access
audit logging
state locking
```

S3 versioning is especially valuable because it can help recover a previous state version after an accidental overwrite/deletion, subject to bucket protection and operational procedures.

---

# 20. Why Terraform State Is Sensitive

Terraform state can contain:

```text
resource IDs
network details
configuration
potentially sensitive values
```

Even if an output is marked:

```hcl
sensitive = true
```

the underlying value may still exist in state.

Therefore:

```text
Terraform State
       ↓
Highly protected
```

---

# 21. Terraform State Recovery

Suppose:

```text
Current state
   ↓
corrupted
```

Because S3 versioning exists:

```text
S3
 │
 ├── version A
 ├── version B
 └── version C
```

we can investigate and recover an appropriate previous state version.

Do this carefully.

Never randomly replace production state without understanding what resources actually exist.

---

# 22. Git Repository Recovery

Our source repositories are:

```text
company-ecommerce
```

with:

```text
21 application repositories
+
infrastructure
+
gitops
+
platform
```

Git itself gives us history.

Example:

```text
Commit A
Commit B
Commit C
```

If a bad change is merged:

```text
revert
```

can restore the desired source state.

---

# 23. GitOps Is Recoverable State

Our GitOps repository contains:

```text
Deployments
Services
Helm values
ArgoCD configuration
environment configuration
```

Therefore:

```text
EKS cluster destroyed
        ↓
new EKS
        ↓
ArgoCD
        ↓
GitOps
        ↓
applications recreated
```

This is why GitOps dramatically improves rebuildability.

---

# 24. ECR Image Recovery

Suppose:

```text
Git commit
```

is available but:

```text
ECR image
```

is gone.

We could rebuild, but that violates the strongest interpretation of:

> Build once, promote the same immutable artifact.

For DR, critical production images should remain available through an appropriate retention/replication strategy.

---

# 25. ECR Retention

Don't delete production images immediately.

Example lifecycle strategy:

```text
Production images
       ↓
retain according to rollback/compliance needs
```

while:

```text
untagged intermediate images
       ↓
short retention
```

This balances:

```text
recovery
+
rollback
+
cost
```

---

# 26. ECR Cross-Region Strategy

For regional DR:

```text
Primary ECR
    │
    ▼
replication
    │
    ▼
DR ECR
```

Then:

```text
DR EKS
   │
   ▼
DR ECR
```

This reduces dependency on the failed Region.

The exact ECR replication rules should be defined through Terraform and validated during implementation.

---

# 27. S3 Backup Strategy

S3 may contain:

```text
invoices
reports
exports
documents
backup artifacts
```

Use:

```text
Versioning
+
Encryption
+
Lifecycle policies
```

where appropriate.

For critical cross-region requirements:

```text
S3
 ↓
replication
 ↓
DR bucket
```

---

# 28. S3 Versioning

Without versioning:

```text
file.pdf
   ↓
overwrite
   ↓
old version unavailable
```

With versioning:

```text
file.pdf
 ├── version 1
 ├── version 2
 └── version 3
```

This helps recover from:

```text
accidental overwrite
accidental deletion
application bugs
```

It is not a complete backup solution by itself, but it provides an important recovery layer.

---

# 29. S3 Lifecycle

Versioning can increase storage consumption.

Therefore:

```text
Versioning
   +
Lifecycle policy
```

should be designed together.

For example:

```text
Current objects
       ↓
retain

Old versions
       ↓
transition/expire according to policy
```

Don't blindly delete old versions if they are part of the required recovery window.

---

# 30. S3 Replication

For regional recovery:

```text
ap-south-1
    │
    ▼
S3 bucket
    │
    ▼
replication
    │
    ▼
DR Region
```

Validate:

```text
replication status
permissions
KMS permissions
object ownership
versioning requirements
```

---

# 31. SQS/SNS Backup

We generally don't "backup messages" in the same way we backup database rows.

Instead, protect:

```text
queue configuration
topic configuration
subscriptions
policies
KMS settings
DLQs
redrive policies
```

through Terraform.

Example:

```text
Terraform
   │
   ├── SQS queue
   ├── DLQ
   ├── SNS topic
   └── subscriptions
```

---

# 32. What About Messages During a Disaster?

This is a business decision.

Suppose:

```text
SQS
│
├── 10,000 pending messages
```

and primary EKS fails.

We need to determine:

```text
Can DR consumers process the same queue?
```

or:

```text
Do we need another queue?
```

For AWS managed services, the exact regional architecture must be designed rather than assuming a queue automatically behaves like a globally replicated database.

---

# 33. Idempotency During Recovery

Suppose a message was processed:

```text
payment captured
```

but the consumer crashed before acknowledging it.

The message may be delivered again.

Without idempotency:

```text
payment
payment
```

could occur.

With idempotency:

```text
payment
   ↓
processed
   ↓
duplicate detected
   ↓
ignored safely
```

This is critical for DR.

---

# 34. Secrets Backup

Our runtime architecture:

```text
Application
   │
   ▼
External Secrets Operator
   │
   ▼
AWS Secrets Manager
```

Secrets should not be treated like ordinary files.

Instead protect:

```text
secret values
secret versions
IAM permissions
KMS keys
regional availability
rotation configuration
```

---

# 35. Secrets Manager Recovery

For a DR environment:

```text
DR EKS
   │
   ▼
External Secrets Operator
   │
   ▼
DR-available secret
```

The secret must be accessible with the DR workload's identity.

Otherwise:

```text
Pod
 ↓
secret fetch
 ↓
AccessDenied
 ↓
application startup failure
```

---

# 36. KMS Is Part of Backup Recovery

Suppose:

```text
RDS backup
```

is encrypted.

Recovery requires:

```text
Backup
+
KMS permissions
+
appropriate key
```

Similarly for:

```text
S3
Secrets Manager
EBS
```

where customer-managed KMS keys are involved.

Therefore:

> Encryption keys are part of the recovery dependency chain.

---

# 37. EBS/PVC Backups

Inside EKS, stateful workloads may use:

```text
PVC
 ↓
EBS volume
```

Example:

```text
Elasticsearch
PostgreSQL inside EKS
RabbitMQ
Kafka
```

If we host stateful systems ourselves, volume snapshots/backup mechanisms become part of DR.

But remember:

> An EBS snapshot is a storage-level recovery mechanism, not necessarily a database-consistent backup.

Database-native backup is generally preferred for database correctness where available.

---

# 38. Database-Native vs Volume Backup

For PostgreSQL:

```text
Preferred
RDS automated backup / PITR
```

rather than relying only on:

```text
EBS snapshot
```

For Elasticsearch:

```text
Elasticsearch snapshot
```

is generally more appropriate than treating raw EBS snapshots as the only recovery method.

For Kafka:

```text
Kafka replication
+
appropriate backup/export strategy
```

needs to be considered.

The backup mechanism should understand the application's consistency model.

---

# 39. Kubernetes Resource Backup

Kubernetes objects include:

```text
Deployment
Service
ConfigMap
Secret
Ingress
HPA
PDB
NetworkPolicy
```

For our architecture, most desired state should already exist in:

```text
GitOps repository
```

Therefore we don't need to treat every Deployment YAML as a unique backup artifact.

The Git repository is the source of desired state.

---

# 40. What About Kubernetes Secrets?

We should **not** solve DR by committing secret values to Git.

Instead:

```text
GitOps
   │
   └── references secret
          │
          ▼
External Secrets
          │
          ▼
Secrets Manager
```

This preserves GitOps while keeping secret values outside Git.

---

# 41. Observability Backup

Our observability stack includes:

```text
Prometheus
Grafana
Alertmanager
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
Jaeger
```

Not all observability data needs the same retention.

---

# 42. Grafana

Dashboards should be treated as code where practical.

```text
Git
 └── dashboards
```

Then:

```text
ArgoCD
   ↓
Grafana
```

If Grafana is destroyed:

```text
reinstall
 ↓
provision dashboards
```

This is much better than manually recreating dashboards after a disaster.

---

# 43. Alertmanager

Alert rules should live in:

```text
GitOps
```

Example:

```text
monitoring/
├── rules/
│   ├── kubernetes.yaml
│   ├── application.yaml
│   ├── rds.yaml
│   └── infrastructure.yaml
```

Then:

```text
Git
 ↓
ArgoCD
 ↓
Prometheus/Alertmanager
```

---

# 44. Elasticsearch

Elasticsearch data can become very large.

Don't automatically attempt to retain every log forever.

Instead:

```text
Hot
 ↓
Warm
 ↓
Cold/Object storage
 ↓
Delete
```

depending on the selected architecture.

For important long-term recovery:

```text
Elasticsearch
      ↓
Snapshots
      ↓
S3
```

Snapshots should be tested.

---

# 45. Elasticsearch Snapshot Restore

Conceptually:

```text
S3 Snapshot
      │
      ▼
Elasticsearch
      │
      ▼
Restore index
      │
      ▼
Validate
```

Check:

```text
index count
documents
mappings
timestamps
dashboards/data views
```

---

# 46. Jaeger

Jaeger recovery depends on its storage backend.

In our design, if Elasticsearch is used:

```text
Jaeger
  ↓
Elasticsearch
  ↓
Snapshots
```

Therefore the trace recovery strategy is connected to Elasticsearch recovery.

This demonstrates an important DR concept:

> Dependencies create dependency chains.

---

# 47. Backup Dependency Chain

For example:

```text
Application
   ↓
RDS
   ↓
KMS
```

Another:

```text
Application
   ↓
Secrets Manager
   ↓
KMS
```

Another:

```text
Jaeger
   ↓
Elasticsearch
   ↓
EBS/S3 snapshots
```

Recovery must consider the entire chain.

---

# 48. Backup Isolation

A backup should ideally not be vulnerable to the same failure as the production system.

Bad:

```text
Production
   │
   ▼
same cluster
   │
   ▼
backup
```

If the cluster is destroyed:

```text
backup lost
```

Better:

```text
Production
   │
   ▼
separate backup storage
```

For major DR:

```text
Primary Region
       │
       ▼
DR / secondary storage
```

where appropriate.

---

# 49. Backup Security

Backups contain sensitive information.

Therefore protect them with:

```text
Encryption
IAM
KMS
access logging
retention policies
restricted deletion
```

A backup bucket should not be:

```text
public
```

or accessible by every developer.

---

# 50. Backup Immutability

For critical backups, consider controls that prevent attackers or accidental operations from deleting backups too easily.

Examples include:

```text
object lock / immutable retention
restricted IAM
separate backup account
separate security boundary
```

The exact mechanism should match the organization's AWS backup architecture and compliance requirements.

---

# 51. The 3-2-1 Principle

A traditional backup concept is:

```text
3 copies
2 different storage/media types
1 offsite copy
```

In cloud environments, the exact implementation can differ, but the underlying idea remains useful:

```text
Don't keep every copy in one failure domain.
```

Example:

```text
Primary RDS
     │
     ├── automated backup
     ├── snapshot
     └── secondary-region recovery copy
```

---

# 52. Backup Retention

Retention should be based on:

```text
RPO
RTO
compliance
business requirements
cost
```

Example:

```text
Daily backups → 30 days
Weekly backups → 3 months
Monthly backups → 1 year
```

This is only an illustrative policy.

Real retention should come from the organization's requirements.

---

# 53. Don't Keep Everything Forever

More backups mean:

```text
storage cost ↑
management complexity ↑
security exposure ↑
```

Therefore:

```text
Retention
+
Lifecycle
+
Compliance
```

must be balanced.

---

# 54. Backup Monitoring

We need alerts for:

```text
backup failed
backup missing
backup too old
replication failed
snapshot failed
storage nearly full
restore test failed
```

Example:

```text
Last successful backup
        │
        ▼
Age > RPO
        │
        ▼
CRITICAL ALERT
```

This is much more useful than simply checking:

```text
backup job exists = true
```

---

# 55. Backup Inventory

Maintain an inventory:

| System          | Backup                | Frequency                  | Retention         | Restore Method         |
| --------------- | --------------------- | -------------------------- | ----------------- | ---------------------- |
| RDS             | Automated/PITR        | Continuous/service-managed | Requirement-based | PITR                   |
| RDS             | Snapshot              | On demand/scheduled        | Requirement-based | New DB                 |
| S3              | Versioning            | Continuous                 | Requirement-based | Version restore        |
| Terraform state | S3 versioning         | Continuous                 | Long              | Previous state version |
| Elasticsearch   | Snapshot              | Scheduled                  | Requirement-based | Snapshot restore       |
| EKS PVC         | Snapshot/backup       | Scheduled                  | Requirement-based | New volume             |
| ECR             | Retention/replication | Continuous                 | Requirement-based | Pull image             |
| Secrets         | Managed/replicated    | Service-managed            | Requirement-based | Secret recovery        |

---

# 56. Restore Testing

This is the most important section.

A backup strategy without restore testing is incomplete.

Test:

```text
Backup
   ↓
Restore
   ↓
Application
   ↓
Business transaction
```

---

# 57. RDS Restore Exercise

Example exercise:

```text
1. Select backup
2. Restore to new RDS instance
3. Wait until available
4. Retrieve endpoint
5. Configure test application
6. Run schema checks
7. Run data checks
8. Execute API tests
9. Validate business transaction
10. Record recovery time
```

Measure:

```text
RTO
RPO
```

---

# 58. EKS Recovery Exercise

Destroying production isn't necessary for every test.

Instead use a dedicated recovery environment.

```text
GitOps
   │
   ▼
DR / Recovery EKS
```

Then:

```text
Terraform
   ↓
VPC
   ↓
EKS
   ↓
Add-ons
   ↓
ArgoCD
   ↓
Applications
```

---

# 59. EKS Restore Validation

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get pods -A
```

Then:

```bash
kubectl get ingress -A
```

Then:

```bash
kubectl get svc -A
```

Then application smoke tests.

---

# 60. Full Platform Restore Exercise

A serious test should look like:

```text
Terraform
   ↓
VPC
   ↓
EKS
   ↓
AWS Load Balancer Controller
   ↓
ArgoCD
   ↓
Applications
   ↓
Secrets
   ↓
RDS
   ↓
Redis
   ↓
SQS/SNS
   ↓
Observability
   ↓
Smoke tests
```

This gives us a real recovery exercise rather than a theoretical one.

---

# 61. Measure Recovery Time

Record timestamps:

```text
Disaster declared
       ↓
Infrastructure started
       ↓
EKS ready
       ↓
RDS ready
       ↓
Secrets ready
       ↓
Applications ready
       ↓
Smoke tests passed
       ↓
Traffic switched
```

Then:

```text
Actual RTO =
Traffic restored time
-
Incident start time
```

---

# 62. Measure Data Loss

Suppose:

```text
Last confirmed transaction:
14:00
```

After recovery:

```text
Latest recoverable transaction:
13:58
```

Then approximate data gap:

```text
2 minutes
```

This is the practical measurement of RPO for that test.

---

# 63. Recovery Test Report

After every exercise:

```text
DR TEST REPORT

Scenario:
EKS regional failure simulation

Target RTO:
30 minutes

Actual RTO:
24 minutes

Target RPO:
5 minutes

Actual RPO:
2 minutes

Failures:
- DNS update took longer than expected
- one secret permission missing

Actions:
- update IAM
- automate DNS validation
```

This is how DR improves over time.

---

# 64. Backup Failure Scenario

Suppose:

```text
RDS backup
    ↓
FAILED
```

Alert:

```text
CRITICAL
```

SRE investigates:

```text
CloudWatch
RDS events
AWS console/API
IAM
storage
backup configuration
```

Do not wait until disaster day.

---

# 65. Restore Failure Scenario

Suppose:

```text
Backup exists
      ↓
Restore
      ↓
FAILED
```

Possible reasons:

```text
KMS access
incorrect permissions
corrupt/incompatible backup
network configuration
quota/capacity
service configuration
```

This is why restore testing is necessary.

---

# 66. Terraform Recovery Scenario

Suppose someone accidentally deletes:

```text
Terraform state object
```

Recovery:

```text
S3 versioning
   ↓
identify previous version
   ↓
validate state
   ↓
recover carefully
   ↓
terraform plan
```

Never jump directly to:

```bash
terraform apply
```

after state recovery.

First:

```bash
terraform plan
```

and inspect what Terraform thinks exists.

---

# 67. ECR Recovery Scenario

Suppose production EKS needs:

```text
order-service@sha256:ABC
```

but primary ECR is unavailable.

If we have:

```text
DR ECR
```

then:

```text
DR EKS
   ↓
DR ECR
   ↓
same image digest
```

This preserves the build-once/promote-same-artifact principle.

---

# 68. GitOps Recovery Scenario

Suppose ArgoCD itself is lost.

Recovery:

```text
New EKS
   ↓
Install ArgoCD
   ↓
Connect GitOps repository
   ↓
Applications recreated
```

Therefore ArgoCD installation instructions should themselves be reproducible.

---

# 69. Platform Components Need Recovery Too

Our EKS platform includes:

```text
ArgoCD
AWS Load Balancer Controller
External Secrets Operator
Prometheus
Grafana
Alertmanager
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
Jaeger
```

These should be deployed through:

```text
Helm
+
GitOps
```

where appropriate.

Then the platform itself becomes reproducible.

---

# 70. Don't Backup Running Containers

Containers are ephemeral.

Bad DR model:

```text
"Let's backup the Pods."
```

Better:

```text
Container image → ECR
Desired state → Git
Infrastructure → Terraform
Data → backups
Secrets → Secrets Manager
```

Then:

```text
rebuild
```

instead of:

```text
restore random containers
```

---

# 71. Immutable Infrastructure Principle

Our recovery model should be:

```text
Destroy
   ↓
Recreate
   ↓
Configure
   ↓
Restore data
```

rather than:

```text
Manually repair everything
```

This is the DevOps advantage of:

```text
Terraform
+
GitOps
+
immutable images
```

---

# 72. Backup vs Replication

These are different.

### Backup

```text
Recover previous state
```

### Replication

```text
Keep another copy synchronized
```

Example:

```text
RDS
 ├── Multi-AZ
 └── backup
```

Multi-AZ improves availability.

Backup improves recovery.

Neither should automatically be assumed to replace the other.

---

# 73. Backup vs High Availability

Remember:

```text
HA
 ↓
keep service running
```

while:

```text
Backup
 ↓
recover data/system
```

Example:

```text
RDS Multi-AZ
```

can recover from an infrastructure failure.

But:

```text
DELETE FROM orders;
```

is a logical data problem.

Backup/PITR may be needed.

---

# 74. Backup vs Blue-Green

Blue-Green:

```text
Bad application release
      ↓
switch back
```

Backup:

```text
Data corruption
      ↓
restore
```

They solve different problems.

Production resilience combines them:

```text
Deployment safety
+
High availability
+
Backup
+
DR
```

---

# 75. Recommended Backup Architecture

```text
                         PRODUCTION
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
         RDS                S3                EKS
          │                  │                  │
          ▼                  ▼                  ▼
     PITR/Snapshot       Versioning          GitOps
          │                  │                  │
          ▼                  ▼                  ▼
       Backup            Replication        Terraform
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                      Recovery Storage
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
              Same Region       DR Region
```

---

# 76. Complete Recovery Flow

```text
                         DISASTER
                            │
                            ▼
                         Detect
                            │
                            ▼
                       Declare Incident
                            │
                            ▼
                      Determine RPO/RTO
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Restore          Rebuild       Recover
           Data         Infrastructure   Config
             │              │              │
             ▼              ▼              ▼
            RDS            EKS          Secrets
            S3            VPC           KMS
          Storage        Terraform      Messaging
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                       Applications
                            │
                            ▼
                       Smoke Tests
                            │
                            ▼
                    Business Validation
                            │
                            ▼
                       Traffic Switch
                            │
                            ▼
                         Monitor
```

---

# 77. Production Backup Checklist

## RDS

```text
☐ Automated backups enabled
☐ PITR available
☐ Retention meets requirement
☐ Multi-AZ configured where required
☐ Encryption enabled
☐ KMS recovery understood
☐ Snapshot procedure documented
☐ Restore tested
```

## S3

```text
☐ Versioning enabled where required
☐ Encryption enabled
☐ Lifecycle configured
☐ Replication configured where required
☐ Restore tested
```

## Terraform

```text
☐ S3 state
☐ Encryption
☐ Versioning
☐ Locking
☐ Restricted IAM
☐ Recovery procedure
```

## ECR

```text
☐ Immutable tags
☐ Production retention
☐ Critical image availability
☐ Cross-region strategy where required
```

## Secrets

```text
☐ Secrets Manager
☐ KMS
☐ IAM
☐ DR availability
☐ Rotation understood
```

## EKS

```text
☐ Terraform reproducibility
☐ GitOps repository
☐ ArgoCD installation procedure
☐ Platform components reproducible
☐ PVC backup strategy where required
```

---

# 78. Restore Test Checklist

```text
☐ Select recovery scenario
☐ Identify backup
☐ Record start time
☐ Restore
☐ Verify encryption
☐ Verify data
☐ Deploy application
☐ Connect application
☐ Test APIs
☐ Test database
☐ Test Redis
☐ Test SQS/SNS
☐ Test secrets
☐ Test observability
☐ Run business transaction
☐ Record RTO
☐ Record RPO
☐ Document failures
☐ Create remediation actions
```

---

# 79. Final Mental Model

The complete backup strategy for our project is:

```text
                 PROTECT
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
    Code           Data         Config
      │             │             │
     Git            RDS        Secrets Manager
     GitOps         S3         Terraform
     ECR            Redis      Kubernetes
                    │
                    ▼
                 BACKUP
                    │
                    ▼
              ISOLATED STORAGE
                    │
                    ▼
                 RESTORE
                    │
                    ▼
                 VALIDATE
                    │
                    ▼
                 RECOVER
                    │
                    ▼
                  TEST
                    │
                    ▼
               MEASURE RTO/RPO
```

The key principle is:

> **A backup is not valuable because it exists. It is valuable because you have high confidence that you can restore it correctly when production is unavailable.**

And our overall resilience model now becomes:

```text
                    Production Resilience
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
      HA              Deployment Safety       DR
       │                   │                   │
    Multi-AZ          Blue-Green             Backup
    Replicas          Canary                 Restore
    Failover          Rolling                RPO/RTO
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                       Recovery
```

**Next: `47-Security-Hardening.md`** — we'll take the entire platform we've built so far and harden it: **EKS security, IAM, Pod Identity, RBAC, NetworkPolicy, security groups, Secrets Manager, KMS, container security, Trivy, admission controls, image signing, supply-chain security, API/Ingress security, logging/auditing, Terraform security, GitHub/GitLab/Jenkins security, and production security checks.**
