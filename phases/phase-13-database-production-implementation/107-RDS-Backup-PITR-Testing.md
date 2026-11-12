# 107-RDS-Backup-PITR-Testing.md

# 107 — RDS Backup & Point-in-Time Recovery Testing

## 1. Purpose

This chapter implements and tests production-grade backup and recovery for Amazon RDS PostgreSQL.

The goal is not simply to enable automated backups.

We will prove that:

- backups are actually being created
- point-in-time recovery works
- snapshots can restore the database
- restored databases can be connected to
- application data can be validated
- recovery time can be measured
- recovery procedures can be documented
- RTO/RPO assumptions are realistic

The architecture is:

Developer
    ↓
Application
    ↓
EKS
    ↓
RDS PostgreSQL
    ↓
Automated Backups / PITR
    ↓
Snapshots
    ↓
Restored RDS Instance

---

# 2. Backup vs High Availability

These are different concepts.

## Multi-AZ

Multi-AZ primarily provides high availability.

Example:

```text
                    RDS
                     |
          +----------+----------+
          |                     |
       AZ-A                  AZ-B
       Primary              Standby
````

If the primary database becomes unavailable, AWS can fail over to the standby.

Multi-AZ does NOT mean:

> "I can recover accidentally deleted data."

---

## Backup

Backup allows recovery of database state.

Examples:

```text
Accidental DELETE
Bad deployment
Corrupted data
Application bug
Operator mistake
Database deletion
```

These require recovery mechanisms.

---

# 3. Recovery Types

RDS PostgreSQL gives us several recovery mechanisms.

## Automated backups

Used for point-in-time recovery.

Example:

```text
09:00
09:30
10:00
10:30
11:00
```

If a production problem occurs at:

```text
10:37
```

we may restore to a point before the problem.

---

## Manual snapshot

A manual DB snapshot represents a specific database state.

Example:

```text
Snapshot:
order-db-prod-before-migration
```

Useful before:

* major schema changes
* risky migrations
* application upgrades
* destructive operations

---

## Point-in-Time Recovery

PITR allows us to restore to a selected time within the available backup window.

Example:

```text
09:00 ─── 10:00 ─── 10:30 ─── 10:37 ─── 11:00
                         ↑
                   bad operation

Restore to:
10:36:30
```

---

# 4. Production Backup Configuration

A typical production RDS configuration should include:

```text
Automated backups
        +
Backup retention
        +
Multi-AZ
        +
Encryption
        +
Manual snapshots where required
        +
Restore testing
```

Example Terraform:

```hcl
resource "aws_db_instance" "postgres" {
  identifier = "ecommerce-prod-postgres"

  engine         = "postgres"
  engine_version = var.postgres_version

  instance_class = var.db_instance_class

  allocated_storage = 100
  storage_type      = "gp3"

  db_name  = "orders"
  username = "admin"

  manage_master_user_password = true

  multi_az = true

  backup_retention_period = 14

  backup_window      = "18:00-18:30"
  maintenance_window = "sun:19:00-sun:19:30"

  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  deletion_protection = true

  skip_final_snapshot = false

  final_snapshot_identifier = "ecommerce-prod-postgres-final"

  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  tags = {
    Environment = "prod"
    Service     = "database"
  }
}
```

Exact Terraform arguments should be checked against the AWS provider version being used.

---

# 5. Backup Retention

Example:

```hcl
backup_retention_period = 14
```

This means automated backup retention is configured for 14 days.

The correct production value depends on:

* business requirements
* compliance
* recovery objectives
* storage cost
* incident investigation requirements

Do not choose a retention period simply because it is commonly used.

---

# 6. RPO

RPO means:

> Recovery Point Objective

It answers:

> "How much data can we afford to lose?"

Example:

```text
RPO = 5 minutes
```

means the recovery design should aim to lose no more than approximately five minutes of data.

For another workload:

```text
RPO = 1 hour
```

may be acceptable.

RDS backup configuration should therefore be designed from the application's actual RPO.

---

# 7. RTO

RTO means:

> Recovery Time Objective

It answers:

> "How quickly must the service be restored?"

Example:

```text
RTO = 60 minutes
```

The recovery process must be designed and tested so that restoration can realistically happen within that target.

Important:

Do not claim an RTO.

Measure it.

---

# 8. Backup Architecture

Production architecture:

```text
                         Application
                              |
                             EKS
                              |
                              v
                      RDS PostgreSQL
                              |
               +--------------+--------------+
               |                             |
               v                             v
       Automated Backup                 Manual Snapshot
               |                             |
               v                             v
             PITR                    Specific Recovery Point
```

---

# 9. What Should Be Backed Up?

The database is not the only thing we need to recover.

Our complete platform contains:

```text
Terraform
GitHub
GitOps
ECR
RDS
Redis
S3
SQS
SNS
Secrets Manager
KMS
EKS
Observability
Route 53
CloudFront/WAF
```

Different components have different recovery mechanisms.

| Component       | Recovery Mechanism                                                        |
| --------------- | ------------------------------------------------------------------------- |
| Terraform       | Git + remote state versioning                                             |
| GitHub          | Git repository                                                            |
| GitOps          | Git repository                                                            |
| ECR             | Image retention/replication                                               |
| RDS             | PITR + snapshots                                                          |
| Redis           | Rebuild/cache recovery or service-specific backups                        |
| S3              | Versioning + replication where required                                   |
| SQS             | Infrastructure recreation; messages retained according to queue semantics |
| SNS             | Terraform recreation                                                      |
| Secrets Manager | Replication/backup strategy                                               |
| KMS             | Key architecture and recovery planning                                    |
| EKS             | Terraform rebuild + GitOps                                                |
| Observability   | Configuration Git + backend-specific retention/backup                     |

---

# 10. Backup Is Not Enough

A common production mistake is:

```text
Backup enabled
        ↓
Assume recovery works
```

That is insufficient.

The correct model is:

```text
Backup
  ↓
Restore
  ↓
Validate
  ↓
Measure
  ↓
Document
  ↓
Repeat
```

---

# 11. Create Test Data

Before performing a recovery test, create known test records.

Example:

```sql
INSERT INTO orders
(customer_id, product_id, quantity, status)
VALUES
('customer-001', 'product-001', 2, 'CREATED');

INSERT INTO orders
(customer_id, product_id, quantity, status)
VALUES
('customer-002', 'product-002', 1, 'PAID');

INSERT INTO orders
(customer_id, product_id, quantity, status)
VALUES
('customer-003', 'product-003', 3, 'SHIPPED');
```

Verify:

```sql
SELECT *
FROM orders
ORDER BY id;
```

Record:

```text
Order IDs
Creation timestamps
Database row count
Important business values
```

---

# 12. Record the Recovery Test Start Time

Example:

```text
Recovery test started:

2026-09-28 10:00:00 IST
```

Record this in the test document.

Also record:

```text
Current RDS identifier
RDS engine/version
Instance class
Region
AZ
Backup retention
Encryption key
Application version
Git commit
```

---

# 13. Create a Manual Snapshot Before the Test

For a controlled recovery exercise:

```bash
aws rds create-db-snapshot \
  --db-instance-identifier ecommerce-prod-postgres \
  --db-snapshot-identifier ecommerce-prod-pre-recovery-test
```

Check:

```bash
aws rds describe-db-snapshots \
  --db-snapshot-identifier ecommerce-prod-pre-recovery-test
```

Wait until:

```text
available
```

before continuing.

---

# 14. Verify Automated Backups

Run:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].BackupRetentionPeriod'
```

Expected:

```text
14
```

Check backup window:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].PreferredBackupWindow'
```

---

# 15. Verify Latest Restorable Time

Use:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].LatestRestorableTime'
```

This is important.

We want to know:

```text
Current time
       |
       | small recovery lag
       v
LatestRestorableTime
```

Do not assume the desired recovery timestamp is available until you verify it.

---

# 16. PITR Test Scenario

We will simulate:

```text
Normal application
      |
      v
Known good database
      |
      v
Bad database operation
      |
      v
Identify incident time
      |
      v
PITR before bad operation
      |
      v
Restore new RDS instance
      |
      v
Validate data
```

---

# 17. Record the Good State

Suppose:

```text
10:00:00
```

Known-good data exists.

We record:

```text
Known-good checkpoint:
10:00:00
```

Then wait enough time for the backup system to have a recoverable point after that checkpoint.

---

# 18. Simulate Bad Data Change

For a controlled test environment, perform a destructive operation.

Example:

```sql
DELETE FROM orders
WHERE customer_id = 'customer-002';
```

Verify:

```sql
SELECT *
FROM orders
WHERE customer_id = 'customer-002';
```

Expected:

```text
0 rows
```

Record the exact timestamp.

Example:

```text
Bad operation:
10:17:42 IST
```

---

# 19. Identify Recovery Target

We do not restore to the time of the incident.

We restore to a point before it.

Example:

```text
10:00  Good
10:05  Good
10:10  Good
10:17  Bad DELETE
10:20  Incident detected
```

Recovery target:

```text
10:16:30
```

The exact timestamp must be chosen based on the actual incident and available recovery point.

---

# 20. Restore Using PITR

AWS CLI example:

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier ecommerce-prod-postgres \
  --target-db-instance-identifier ecommerce-prod-postgres-pitr-test \
  --restore-time 2026-09-28T10:16:30Z \
  --db-instance-class db.r6g.large \
  --db-subnet-group-name ecommerce-prod-db-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx \
  --no-publicly-accessible
```

The exact parameters should match the production architecture.

The restore creates a new RDS instance.

It does NOT overwrite the existing production database.

---

# 21. Important Recovery Principle

Never use the production database itself as your first recovery experiment.

Prefer:

```text
Production RDS
      |
      v
PITR
      |
      v
New temporary RDS
      |
      v
Validate
```

This prevents the recovery test from becoming another incident.

---

# 22. Monitor Restore

Check:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres-pitr-test \
  --query 'DBInstances[0].DBInstanceStatus'
```

Possible status:

```text
creating
```

Eventually:

```text
available
```

---

# 23. Get the Restored Endpoint

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres-pitr-test \
  --query 'DBInstances[0].Endpoint.Address'
```

Example:

```text
ecommerce-prod-postgres-pitr-test.xxxxx.ap-south-1.rds.amazonaws.com
```

---

# 24. Test Network Connectivity

From an approved environment that has network access:

```bash
nc -vz \
ecommerce-prod-postgres-pitr-test.xxxxx.ap-south-1.rds.amazonaws.com \
5432
```

Expected:

```text
Connection succeeded
```

If this fails, investigate:

```text
Security Group
Route
Subnet
NACL
DNS
Network path
Client location
```

---

# 25. Connect to Restored Database

Example:

```bash
psql \
  -h ecommerce-prod-postgres-pitr-test.xxxxx.ap-south-1.rds.amazonaws.com \
  -U orders_app \
  -d orders
```

Do not place production passwords directly in shell history.

Use the approved Secrets Manager/secret retrieval mechanism.

---

# 26. Validate Recovered Data

Run:

```sql
SELECT *
FROM orders
ORDER BY id;
```

The deleted record should exist if the recovery target is before the deletion.

Example:

```sql
SELECT *
FROM orders
WHERE customer_id = 'customer-002';
```

Expected:

```text
customer-002 record exists
```

---

# 27. Validate Row Counts

Record before incident:

```sql
SELECT COUNT(*)
FROM orders;
```

Record after recovery:

```sql
SELECT COUNT(*)
FROM orders;
```

Compare.

Example:

```text
Before incident: 125,430
Bad state:        125,429
Recovered:        125,430
```

---

# 28. Validate Application Connectivity

Do not stop at:

```text
Database is available
```

Test the application.

For example:

```text
EKS
 ↓
order-service
 ↓
restored database
```

For a controlled test, temporarily configure a test instance/environment to use the restored database.

Then test:

```bash
curl http://order-service:8080/actuator/health
```

And:

```bash
curl http://order-service:8080/api/v1/orders
```

---

# 29. Validate Schema

Run:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

Check:

```text
Tables
Indexes
Constraints
Sequences
Extensions
Permissions
```

---

# 30. Validate Database Migrations

Check:

```sql
SELECT *
FROM flyway_schema_history
ORDER BY installed_rank;
```

Confirm:

```text
Expected migration versions
Expected checksum
Expected application compatibility
```

Recovery is not complete if the database is restored but the application cannot use its schema.

---

# 31. Measure RTO

Record:

```text
Incident detected:
10:20

Recovery started:
10:25

RDS available:
10:48

Application connected:
10:55

Business validation completed:
11:05
```

Recovery duration:

```text
45 minutes
```

If the target RTO was:

```text
60 minutes
```

the measured test result is:

```text
45 minutes
```

Do not convert this into a permanent guarantee.

Repeat the test under realistic conditions.

---

# 32. Measure RPO

Suppose:

```text
Last recoverable point:
10:16:30

Bad operation:
10:17:42
```

The recovery point difference is approximately:

```text
1 minute 12 seconds
```

This gives evidence about the actual recovery point achieved during the test.

---

# 33. PITR Recovery Checklist

```text
[ ] Incident timestamp identified
[ ] Known-good timestamp identified
[ ] LatestRestorableTime checked
[ ] Recovery target selected
[ ] New RDS instance created
[ ] Network connectivity tested
[ ] Database login tested
[ ] Schema validated
[ ] Migration history validated
[ ] Data validated
[ ] Application connectivity tested
[ ] Business validation completed
[ ] RTO measured
[ ] RPO measured
[ ] Recovery evidence recorded
[ ] Temporary instance removed
```

---

# 34. Snapshot Restore Test

PITR is not the only recovery test.

Test a snapshot as well.

Example:

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ecommerce-prod-postgres-snapshot-test \
  --db-snapshot-identifier ecommerce-prod-pre-recovery-test \
  --db-subnet-group-name ecommerce-prod-db-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx \
  --no-publicly-accessible
```

Wait for:

```text
available
```

Then repeat:

```text
Connectivity
Authentication
Schema validation
Data validation
Application validation
```

---

# 35. Snapshot vs PITR

| Capability                          | Snapshot | PITR                               |
| ----------------------------------- | -------- | ---------------------------------- |
| Specific database state             | Yes      | Yes                                |
| Restore to arbitrary supported time | No       | Yes                                |
| Useful before risky migration       | Yes      | Yes                                |
| Accidental data deletion recovery   | Limited  | Yes                                |
| Fast operational backup             | Yes      | Yes                                |
| Long-term archival strategy         | Depends  | Not normally the primary mechanism |

---

# 36. Production Recovery Procedure

A production incident might look like:

```text
Application incident
       |
       v
Determine database impact
       |
       v
Stop further destructive activity
       |
       v
Identify recovery target
       |
       v
Check available PITR window
       |
       v
Create recovery database
       |
       v
Validate data
       |
       v
Determine cutover strategy
       |
       v
Application reconnect
       |
       v
Business validation
       |
       v
Resume traffic
```

---

# 37. Do Not Immediately Delete the Bad Database

During an actual incident, preserve evidence.

The original database may be required for:

```text
Forensics
Root cause analysis
Data comparison
Audit
Incident investigation
```

The exact retention strategy should follow the organization's incident and compliance procedures.

---

# 38. Production Cutover Options

After restoring:

```text
ecommerce-prod-postgres-recovery
```

we need a controlled application cutover.

Possible approaches:

### Option 1 — Configuration update

Update the database endpoint consumed by the application.

### Option 2 — Secret update

Update the database connection configuration in Secrets Manager.

### Option 3 — DNS abstraction

Use an internal database DNS abstraction where appropriate.

### Option 4 — Controlled application redeployment

Update GitOps configuration and allow ArgoCD to deploy the new configuration.

The chosen method must be tested before an incident.

---

# 39. Application Connection Configuration

The application should not hardcode:

```text
production-db.example.com
```

inside the image.

Instead:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Then:

```text
Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Pod environment
       ↓
Spring Boot
```

---

# 40. Recovery Does Not Mean Data Is Automatically Correct

A restored database may technically be healthy while business data is still wrong.

Validation should include:

```text
Database availability
Schema
Application compatibility
Orders
Payments
Inventory
Customers
Transactions
Indexes
Sequences
Critical business tables
```

For e-commerce:

```text
Order
Payment
Inventory
Shipment
Invoice
```

must be checked carefully.

---

# 41. PostgreSQL Sequence Validation

After data recovery, verify important sequences.

Example:

```sql
SELECT
    sequence_name
FROM information_schema.sequences
WHERE sequence_schema = 'public';
```

If IDs are sequence-based, validate that future inserts will not generate conflicting IDs.

---

# 42. Application Smoke Test

After recovery:

```text
Create order
        ↓
Read order
        ↓
Update order
        ↓
Payment flow
        ↓
Inventory interaction
        ↓
Notification event
```

Do this in a controlled recovery environment before production cutover.

---

# 43. Monitoring During Recovery

Watch:

```text
RDS CPU
RDS memory
Database connections
Read/write latency
IOPS
Throughput
Storage
Application errors
Application latency
Connection pool utilization
```

Example:

```bash
kubectl top pods -n ecommerce-prod
```

And inspect application logs:

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200
```

---

# 44. Connection Pool Consideration

After database recovery, applications may reconnect simultaneously.

Example:

```text
100 Pods
    ×
20 connections
    =
2000 connections
```

This may overwhelm the database.

Therefore recovery should include:

```text
Connection pool limits
Connection retry behavior
Backoff
Startup behavior
RDS capacity
RDS Proxy where appropriate
```

---

# 45. Recovery Thundering Herd

A common failure pattern:

```text
Database becomes available
        ↓
100 application Pods reconnect
        ↓
Each opens many connections
        ↓
Database connection spike
        ↓
Database becomes overloaded
        ↓
Application failures
```

Mitigation:

```text
Connection pool limits
Exponential backoff
Jitter
Controlled rollout
Readiness probes
RDS Proxy where appropriate
```

---

# 46. Backup Monitoring

Backup operations themselves should be observable.

Track:

```text
Backup success
Backup failure
Latest restorable time
Snapshot availability
Backup retention
Storage
Recovery test results
```

A backup failure should generate an operational signal.

---

# 47. Recovery Evidence

Store recovery test evidence in an approved location.

Example:

```text
DR/
└── RDS/
    └── 2026-09-28-pitr-test/
        ├── test-plan.md
        ├── timestamps.md
        ├── validation-results.md
        ├── screenshots/
        └── incident-notes.md
```

Do not store:

```text
Passwords
Database credentials
Access keys
Secret values
```

in the evidence repository.

---

# 48. Backup Security

Backups contain production data.

Therefore:

```text
Encryption
IAM controls
KMS
Audit logging
Access control
Retention
Deletion protection
```

are important.

Do not give every engineer permission to:

```text
Delete snapshots
Restore production databases
Access backup contents
```

---

# 49. IAM Separation

Example operational roles:

```text
Developer
   ↓
Read application data only

DevOps
   ↓
Infrastructure operations

DBA / Database Operations
   ↓
Database administration

Incident Commander
   ↓
Coordinates recovery

Security
   ↓
Audits access
```

Actual permissions should follow organizational RBAC and least privilege.

---

# 50. Terraform and Recovery

Terraform should manage the desired infrastructure configuration.

Terraform should NOT be treated as the database backup mechanism.

Correct separation:

```text
Terraform
    ↓
Infrastructure configuration

RDS
    ↓
Database data

RDS Backup/PITR
    ↓
Database recovery
```

---

# 51. Disaster Recovery Architecture

For regional DR:

```text
                Primary Region
                ap-south-1
                     |
             RDS PostgreSQL
                     |
                     | replication/
                     | snapshot strategy
                     v
                DR Region
```

The exact implementation depends on:

```text
RTO
RPO
Cost
Database size
Compliance
Business requirements
```

This will be implemented later in the DR phase.

---

# 52. Automated Recovery Test

A mature platform should periodically test recovery.

Example process:

```text
1. Select recovery point
2. Restore temporary RDS
3. Validate connectivity
4. Validate schema
5. Validate critical tables
6. Run application smoke tests
7. Record RTO/RPO
8. Destroy temporary resources
9. Publish report
```

This can be partially automated.

---

# 53. Example Recovery Test Script Structure

```text
scripts/
└── rds-recovery-test/
    ├── restore.sh
    ├── validate.sh
    ├── smoke-test.sh
    ├── cleanup.sh
    └── README.md
```

Example:

```bash
#!/bin/bash

set -euo pipefail

SOURCE_DB="ecommerce-prod-postgres"
TARGET_DB="ecommerce-prod-pitr-test"

echo "Starting RDS recovery test..."

aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier "$SOURCE_DB" \
  --target-db-instance-identifier "$TARGET_DB" \
  --use-latest-restorable-time

echo "Restore initiated."
```

The script should be expanded with:

```text
status polling
timeouts
validation
logging
cleanup
failure handling
```

before being considered production-grade.

---

# 54. Recovery Test Failure Scenarios

## Scenario 1 — PITR unavailable

Possible causes:

```text
Backup retention disabled
Backup configuration issue
Incorrect source DB
Recovery target outside available window
```

---

## Scenario 2 — Restored DB cannot be reached

Check:

```text
Security Group
Subnet group
Route
NACL
DNS
Client network
```

---

## Scenario 3 — Application cannot authenticate

Check:

```text
Secret
Username
Password
Database permissions
Authentication configuration
```

---

## Scenario 4 — Application starts but queries fail

Check:

```text
Schema
Migration version
Database user permissions
Indexes
Sequences
Connection pool
```

---

## Scenario 5 — Recovery is too slow

Investigate:

```text
Database size
Instance class
Storage
Backup restore behavior
Network
Application startup
Migration duration
Smoke test duration
```

---

# 55. Recovery Runbook

## Incident

```text
Database corruption / accidental data deletion
```

## Step 1

Stop the operation causing damage.

## Step 2

Record incident timestamp.

## Step 3

Identify affected data.

## Step 4

Determine required recovery point.

## Step 5

Check:

```bash
aws rds describe-db-instances ...
```

for available restore information.

## Step 6

Restore to a new RDS instance.

## Step 7

Validate:

```text
Network
Authentication
Schema
Data
Application
```

## Step 8

Obtain approval for cutover.

## Step 9

Update application configuration through the approved GitOps/change process.

## Step 10

Monitor application.

## Step 11

Validate business transactions.

## Step 12

Preserve original database/evidence according to incident procedure.

## Step 13

Complete RCA.

---

# 56. Recovery Test Report

Example:

```text
RDS Recovery Test
=================

Date:
2026-09-28

Database:
ecommerce-prod-postgres

Engine:
PostgreSQL

Primary Region:
ap-south-1

Recovery Type:
Point-in-Time Recovery

Incident Simulation:
Accidental DELETE

Bad Operation:
10:17:42 IST

Recovery Target:
10:16:30 IST

Restore Started:
10:25:00 IST

Database Available:
10:48:00 IST

Application Validation:
10:55:00 IST

Business Validation:
11:05:00 IST

Measured Recovery Duration:
40 minutes

Measured Recovery Point:
~1 minute

Result:
Successful

Follow-up:
Review connection pool behavior
```

---

# 57. Important Production Rule

Never write:

```text
RDS backups are enabled, therefore DR is complete.
```

Instead:

```text
RDS backups are enabled
        ↓
Recovery has been tested
        ↓
RTO measured
        ↓
RPO measured
        ↓
Runbook documented
        ↓
Recovery periodically retested
```

---

# 58. Production Checklist

## RDS

```text
[ ] Multi-AZ configured where required
[ ] Automated backups enabled
[ ] Appropriate retention configured
[ ] Encryption enabled
[ ] KMS configured
[ ] Deletion protection enabled
[ ] Final snapshot configured
[ ] Backup window defined
[ ] Maintenance window defined
```

## Recovery

```text
[ ] PITR tested
[ ] Snapshot restore tested
[ ] Recovery database connectivity tested
[ ] Schema validated
[ ] Data validated
[ ] Application tested
[ ] RTO measured
[ ] RPO measured
[ ] Runbook documented
```

## Security

```text
[ ] Backup access restricted
[ ] KMS permissions restricted
[ ] Database credentials protected
[ ] No secrets stored in Git
[ ] Audit logging enabled
```

## Operations

```text
[ ] Backup failures monitored
[ ] Recovery tests scheduled
[ ] Evidence retained
[ ] Ownership documented
[ ] Incident procedure documented
```

---

# 59. Final Architecture

```text
                  ┌───────────────────────┐
                  │      EKS Cluster      │
                  │                       │
                  │   order-service       │
                  └───────────┬───────────┘
                              │
                              │ PostgreSQL
                              ▼
                    ┌───────────────────┐
                    │   RDS PostgreSQL  │
                    │                   │
                    │     Multi-AZ      │
                    └─────────┬─────────┘
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
          Automated Backup          Manual Snapshot
                  │                       │
                  ▼                       ▼
                 PITR              Recovery Point
                  │                       │
                  └───────────┬───────────┘
                              ▼
                    Temporary RDS Restore
                              │
                              ▼
                    Validation + Testing
                              │
                              ▼
                     Controlled Cutover
```

---

# 60. Key Takeaways

The production mindset is:

```text
Backup ≠ Recovery
```

Instead:

```text
Backup
+
Restore
+
Validation
+
Measured RTO
+
Measured RPO
+
Runbook
+
Regular Testing
=
Recoverable Database
```

For our e-commerce platform, RDS recovery is one layer of the complete disaster-recovery architecture.

The next database implementation will focus on another major production problem:

```text
Too many application Pods
        ↓
Too many database connections
        ↓
Connection pool exhaustion
        ↓
RDS performance degradation
```

That is covered in:

**108-RDS-Performance-and-Connection-Pooling.md**

```
```
