# 136 — RDS RECOVERY

## 1. Objective

This chapter covers **production PostgreSQL/RDS recovery** for our 21-microservice e-commerce platform.

The objective is to recover the database safely after:

* Accidental deletion
* Database corruption
* Bad schema migration
* Application-induced data corruption
* RDS instance failure
* Multi-AZ failure
* Region failure
* Security incident
* Incorrect database configuration
* Failed deployment
* Recovery into a new VPC
* Recovery into a new RDS instance

The recovery architecture must protect:

```text
Data
Availability
Consistency
Security
RTO
RPO
Application compatibility
```

---

# 2. Our RDS Architecture

Production architecture:

```text
                         EKS
                          |
              +-----------+-----------+
              |           |           |
        order-service  user-service  ...
              |
              v
       EKS Application SG
              |
              v
       RDS PostgreSQL
              |
      +-------+-------+
      |               |
   AZ-A            AZ-B
  Primary          Standby
```

RDS is located in private subnets.

Example:

```text
VPC: 10.0.0.0/16

Private DB subnet A
10.0.11.0/24

Private DB subnet B
10.0.12.0/24

Private DB subnet C
10.0.13.0/24
```

The DB subnet group spans multiple AZs.

---

# 3. Important Principle

EKS recovery and RDS recovery are separate.

If EKS is destroyed:

```text
EKS
  X
```

RDS can still exist:

```text
RDS
 |
 +--> PostgreSQL data
```

Therefore:

```text
EKS Recovery
     |
     +----> Reconnect to existing RDS
```

not:

```text
EKS Recovery
     |
     +----> Automatically recreate database
```

---

# 4. RDS Recovery Scenarios

We will handle these scenarios separately.

```text
1. Application cannot connect
2. RDS instance failure
3. Multi-AZ failover
4. Accidental database deletion
5. Data corruption
6. Bad migration
7. Wrong data update/delete
8. Database configuration problem
9. Security incident
10. Restore from snapshot
11. Point-in-time recovery
12. Restore into another VPC
13. Region-level recovery
```

---

# 5. Recovery Decision Tree

```text
Database problem
       |
       v
Is RDS available?
       |
   +---+---+
   |       |
  YES      NO
   |       |
   v       v
Investigate  RDS failure
connection   |
/data         v
             Failover /
             Restore
```

If RDS is available:

```text
Do NOT restore unnecessarily.
```

For example:

```text
Application cannot connect
```

does not automatically mean:

```text
Restore database
```

It could be:

```text
Security Group
Network
DNS
Credentials
IAM
Connection Pool
Application configuration
```

---

# 6. First Recovery Rule

Before changing anything:

```text
STOP
 |
 v
Understand the failure
 |
 v
Preserve evidence
 |
 v
Determine impact
 |
 v
Choose recovery method
```

Avoid immediately running:

```bash
terraform apply
```

or:

```bash
aws rds delete-db-instance
```

during an incident.

Recovery actions must be deliberate.

---

# 7. Check RDS Status

Example:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --region ap-south-1
```

Look at:

```text
DBInstanceStatus
Engine
EngineVersion
AvailabilityZone
MultiAZ
DBInstanceClass
Endpoint
DBSubnetGroup
VpcSecurityGroups
BackupRetentionPeriod
LatestRestorableTime
```

---

# 8. Check RDS Events

RDS events can reveal:

```text
Failover
Maintenance
Storage problems
Backup events
Configuration changes
Recovery
Availability events
```

Example:

```bash
aws rds describe-events \
  --source-identifier ecommerce-prod-postgres \
  --source-type db-instance \
  --duration 1440 \
  --region ap-south-1
```

During an incident, capture the relevant events rather than relying on memory.

---

# 9. Check CloudTrail

RDS configuration changes should also be investigated through CloudTrail.

Questions:

```text
Who changed the database?
When?
What changed?
Was the change approved?
Was it automated?
Was Terraform involved?
Was the change made manually?
```

Examples of important actions:

```text
ModifyDBInstance
ModifyDBCluster
DeleteDBInstance
RestoreDBInstanceFromDBSnapshot
RestoreDBInstanceToPointInTime
RebootDBInstance
```

---

# 10. RDS Monitoring

Important database metrics include:

```text
CPUUtilization
FreeableMemory
FreeStorageSpace
DatabaseConnections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
DiskQueueDepth
NetworkThroughput
```

For PostgreSQL also investigate:

```text
Active sessions
Idle sessions
Locks
Long-running queries
Transactions
Replication state
```

---

# 11. Determine the Failure Type

Example:

```text
Customer reports checkout failure
```

Check:

```text
ALB
 ↓
order-service
 ↓
Database connection
 ↓
RDS
```

If order-service reports:

```text
Connection refused
```

investigate:

```text
RDS status
DNS
Security Group
Route
Port 5432
Credentials
Connection pool
```

---

# 12. Test Database Connectivity From EKS

Do not test only from your laptop.

The real test is from the application network.

Example temporary diagnostic Pod:

```bash
kubectl run postgres-debug \
  --rm -it \
  --restart=Never \
  --image=postgres:16 \
  -n ecommerce-prod \
  -- bash
```

Inside:

```bash
pg_isready \
  -h "$DB_HOST" \
  -p 5432
```

Then:

```bash
psql \
  -h "$DB_HOST" \
  -U "$DB_USERNAME" \
  -d orders
```

Use the approved diagnostic credentials and do not expose passwords in shell history or incident logs.

---

# 13. DNS Troubleshooting

RDS provides a DNS endpoint.

Example:

```text
ecommerce-prod.xxxxxx.ap-south-1.rds.amazonaws.com
```

Test:

```bash
nslookup ecommerce-prod.xxxxxx.ap-south-1.rds.amazonaws.com
```

or:

```bash
dig ecommerce-prod.xxxxxx.ap-south-1.rds.amazonaws.com
```

From a Pod:

```bash
kubectl run dns-test \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -n ecommerce-prod \
  -- nslookup ecommerce-prod.xxxxxx.ap-south-1.rds.amazonaws.com
```

---

# 14. Security Group Recovery

Our intended architecture:

```text
EKS Application SG
        |
        | TCP 5432
        v
RDS Security Group
```

RDS should **not** allow:

```text
0.0.0.0/0 : 5432
```

Instead:

```text
Source:
EKS application security group

Destination:
RDS security group

Port:
5432
```

---

# 15. RDS Endpoint Stability

Applications should use the RDS endpoint:

```text
PostgreSQL DNS endpoint
```

not:

```text
Primary IP address
```

This matters during failover.

Architecture:

```text
Application
    |
    v
RDS DNS endpoint
    |
    v
Current database primary
```

When RDS performs a supported failover, the endpoint continues to represent the database service while the underlying instance role changes.

Applications still need sensible connection retry/recovery behavior.

---

# 16. RDS Multi-AZ Failover

Our production database should use Multi-AZ where appropriate.

Architecture:

```text
AZ-A
Primary
  |
  | synchronous replication
  |
AZ-B
Standby
```

If the primary fails:

```text
Primary failure
       |
       v
RDS detects failure
       |
       v
Standby promoted
       |
       v
Endpoint points to active primary
```

This is an availability mechanism.

It is **not** a backup mechanism.

---

# 17. Multi-AZ Is Not Backup

This distinction is critical.

```text
Multi-AZ
    =
High Availability
```

while:

```text
Backup / PITR
    =
Data Recovery
```

Example:

```text
Bad SQL:
DELETE FROM orders;
```

Multi-AZ will faithfully replicate the bad change.

Therefore:

```text
Multi-AZ
does NOT protect against logical corruption.
```

Backups and PITR are required.

---

# 18. Recovery Options

There are several recovery mechanisms.

```text
                 RDS Recovery
                      |
        +-------------+-------------+
        |             |             |
     Failover      Snapshot        PITR
        |             |             |
   Availability    Known state   Time-based
                   restore        restore
```

### Failover

Use for infrastructure/availability failure.

### Snapshot restore

Use when you need to restore from a known snapshot.

### Point-in-time recovery

Use when you need to recover to a specific time before corruption.

---

# 19. RPO and RTO

For our database:

```text
RPO
=
How much database data can we afford to lose?
```

Example:

```text
RPO = 5 minutes
```

RTO:

```text
How long can the database/application remain unavailable?
```

Example:

```text
RTO = 60 minutes
```

These are targets and must be validated through actual recovery exercises.

---

# 20. Automated Backups

Production RDS should have automated backups enabled.

Conceptually:

```text
RDS
 |
 +--> Automated backups
       |
       +--> Restore to point in time
```

The retention period should be selected according to:

```text
Business requirements
Compliance
Storage cost
Recovery requirements
```

Do not blindly choose a retention value simply because it is common elsewhere.

---

# 21. Manual Snapshots

A manual snapshot is useful before high-risk changes.

Example:

```text
Production database
       |
       v
Pre-migration snapshot
       |
       v
Schema migration
```

If the migration causes a severe problem:

```text
Snapshot
   |
   v
Restore
```

However, a snapshot should not be the only rollback mechanism for every schema change.

For application schema migrations, the preferred strategy is usually:

```text
Backward-compatible migration
+
Forward fix
```

rather than restoring the entire production database.

---

# 22. Point-in-Time Recovery

Suppose:

```text
10:00  Normal
10:30  Deployment
10:42  Bad application release
10:45  Data corruption begins
10:50  Incident detected
```

We may need:

```text
Restore database
to approximately 10:41
```

rather than:

```text
Restore yesterday's snapshot
```

PITR is designed for this type of recovery.

---

# 23. PITR Recovery Architecture

```text
Existing RDS
     |
     +--> Automated backups
              |
              v
      Point-in-time restore
              |
              v
       New RDS instance
              |
              v
       Validate database
```

Important:

> PITR normally creates a new database instance rather than magically rewinding the existing production instance in place.

---

# 24. Identify Recovery Timestamp

This is one of the most important incident decisions.

Suppose corruption started at:

```text
10:42:17
```

We might select a restore point shortly before the first bad transaction.

But don't guess.

Use:

```text
Application logs
Database audit logs
CloudTrail
Deployment timeline
Kibana
Jaeger
Business records
```

to establish the corruption window.

---

# 25. Example Recovery Investigation

```text
10:30
Deployment v1.8.4

10:35
Order-service logs normal

10:41
Database query behavior changes

10:42
First invalid update

10:45
Customer reports incorrect orders

10:48
Incident declared
```

Possible recovery target:

```text
10:41:xx
```

The exact timestamp must be determined from evidence.

---

# 26. PITR Recovery Process

Conceptually:

```text
1. Identify corruption time
2. Stop further destructive writes
3. Preserve current database
4. Select recovery timestamp
5. Start PITR
6. Wait for new RDS instance
7. Validate database
8. Validate schema
9. Validate application compatibility
10. Redirect application
11. Monitor
```

---

# 27. Freeze Writes

During serious corruption:

```text
Application
   |
   X
writes
```

Possible controlled approaches:

```text
Scale write-producing workloads down
Disable affected endpoints
Enable maintenance mode
Block specific workflows
Stop consumers
```

Do not arbitrarily shut down the entire platform if only one service is causing corruption.

The objective is:

> Stop the source of damage while preserving as much service as possible.

---

# 28. Example: Bad Deployment

Suppose a new order-service version contains:

```java
order.setStatus("CANCELLED");
```

when it should set:

```java
order.setStatus("CONFIRMED");
```

The application starts corrupting order state.

Recovery:

```text
Detect
 ↓
Rollback application
 ↓
Stop bad writes
 ↓
Assess data corruption
 ↓
Determine recovery point
 ↓
PITR if necessary
```

Do not immediately restore the database without first stopping the application that caused the corruption.

Otherwise:

```text
Restore
 ↓
Bad application reconnects
 ↓
Corruption happens again
```

---

# 29. Restore Into an Isolated Environment

A very important production practice:

```text
PITR
 |
 v
Temporary RDS
 |
 v
Validation
```

Do not immediately point production applications at the newly restored database.

First verify:

```text
Schema
Tables
Rows
Critical business data
Users
Indexes
Extensions
Constraints
Sequences
Application compatibility
```

---

# 30. Database Validation

Example:

```sql
SELECT COUNT(*) FROM orders;
```

Check important records:

```sql
SELECT *
FROM orders
ORDER BY created_at DESC
LIMIT 20;
```

Check statuses:

```sql
SELECT status, COUNT(*)
FROM orders
GROUP BY status;
```

Check migrations:

```sql
SELECT *
FROM flyway_schema_history
ORDER BY installed_rank DESC;
```

---

# 31. Schema Validation

Verify:

```text
Tables
Indexes
Constraints
Foreign keys
Sequences
Extensions
Functions
Triggers
Views
```

Example:

```sql
SELECT version();
```

Also verify the expected PostgreSQL extensions used by the application.

---

# 32. Flyway Validation

Our production application uses controlled schema migrations.

After restoring:

```text
RDS
 |
 v
Flyway metadata
 |
 v
Application schema
```

Run validation in the appropriate controlled environment:

```bash
./mvnw flyway:validate
```

or through the project's migration Job/process.

The exact command depends on how Flyway is integrated into the project.

---

# 33. Important Flyway Recovery Rule

Never casually run:

```text
flyway clean
```

against production.

`clean` can remove database objects and is generally inappropriate as a production recovery technique.

For a restored database:

```text
Validate
 |
 v
Compare schema
 |
 v
Forward-fix if necessary
```

---

# 34. Sequence Validation

PostgreSQL sequences can matter after data restoration.

Example:

```sql
SELECT MAX(id) FROM orders;
```

Then inspect the sequence where applicable.

A sequence that is behind the maximum ID can produce conflicts during future inserts.

This should be explicitly checked during recovery validation.

---

# 35. Application Compatibility

Suppose the application expects:

```text
orders.customer_id
```

but the restored schema doesn't contain it.

Then:

```text
Database = healthy
Application = broken
```

Therefore recovery validation must include:

```text
Database
+
Application version
```

---

# 36. Blue/Green Database Recovery

For a controlled recovery:

```text
                    RDS
                     |
          +----------+----------+
          |                     |
      Current DB            Restored DB
          |                     |
      Production            Validation
```

After validation:

```text
Application
     |
     v
Restored DB
```

The exact cutover mechanism depends on the application's architecture and AWS RDS capabilities.

---

# 37. Connection String Cutover

Our application should not hard-code database endpoints into Docker images.

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
Environment variables
       |
       v
Spring Boot
```

Example:

```yaml
env:
  - name: DB_URL
    valueFrom:
      secretKeyRef:
        name: order-db
        key: url
```

After recovery, the database endpoint/configuration can be changed through the controlled secret/configuration workflow.

---

# 38. Restart Application Pods

If the connection configuration changes:

```bash
kubectl rollout restart deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

For all affected services, use the controlled GitOps deployment process rather than ad-hoc production changes whenever possible.

---

# 39. Connection Pool Recovery

Spring Boot uses HikariCP in our architecture.

Example:

```text
order-service
 |
 +--> Pod 1 --> Hikari pool
 +--> Pod 2 --> Hikari pool
 +--> Pod 3 --> Hikari pool
```

If the application was connected to the old RDS instance:

```text
Old connections
      X
      |
New database
```

The application must establish new connections.

Short, sensible connection and validation timeouts help prevent stale connections from creating prolonged failures.

---

# 40. Connection Budget

Suppose:

```text
10 Pods
```

and:

```text
maximumPoolSize = 20
```

Potential connections:

```text
10 × 20 = 200
```

Across 21 services:

```text
Pods × pool size
```

can become significant.

After recovery, monitor:

```text
DatabaseConnections
```

and PostgreSQL:

```sql
SELECT state, COUNT(*)
FROM pg_stat_activity
GROUP BY state;
```

---

# 41. RDS Restore Into a New VPC

Sometimes the original VPC itself is unavailable or a recovery environment is required.

Architecture:

```text
Original VPC
     |
     X
     |
Recovery VPC
     |
     v
Restored RDS
```

Recovery VPC must provide:

```text
Private subnets
DB subnet group
Route tables
Security groups
DNS
KMS access
Monitoring
Application connectivity
```

---

# 42. Security During Restore

A restored database must not accidentally become public.

Verify:

```text
PubliclyAccessible = false
```

where appropriate.

Also verify:

```text
Security Group
Encryption
KMS
Parameter group
Subnet group
IAM
Monitoring
Backup retention
```

Never use:

```text
0.0.0.0/0 : 5432
```

as a quick recovery workaround.

---

# 43. KMS Considerations

If the original database uses KMS encryption:

```text
RDS
 |
 v
KMS key
```

The recovery process must account for:

```text
KMS key availability
Key policy
IAM permissions
Region
Cross-account permissions if applicable
```

A backup is not useful if the recovery environment cannot legitimately use the required encryption key.

---

# 44. Snapshot Recovery

Conceptually:

```text
Snapshot
   |
   v
New RDS
   |
   +--> subnet group
   +--> security group
   +--> KMS
   +--> parameter group
   |
   v
Validation
```

Snapshot recovery is useful when a known-good snapshot is the appropriate recovery point.

---

# 45. Snapshot Selection

Never blindly select:

```text
latest snapshot
```

Determine:

```text
Which snapshot?
When created?
Before which change?
Was the snapshot successful?
Was it encrypted?
Which engine version?
Which configuration?
```

Example:

```text
Snapshot A
09:00
Known good

Snapshot B
10:30
After problematic deployment
```

Snapshot A may be the more appropriate recovery candidate.

---

# 46. RDS Engine Compatibility

Before recovery, verify:

```text
PostgreSQL version
Extensions
Parameter group
Instance compatibility
Application JDBC driver
Flyway compatibility
```

Example:

```text
Production:
PostgreSQL 16.x
```

Do not casually restore into a completely different major version and assume the application will work.

---

# 47. Data Integrity Validation

Business-level checks are important.

Examples:

```sql
SELECT COUNT(*) FROM orders;

SELECT COUNT(*) FROM customers;

SELECT COUNT(*) FROM payments;
```

But counts alone are insufficient.

Validate:

```text
Foreign keys
Order/payment consistency
Inventory consistency
Duplicate orders
Missing orders
Invalid statuses
Sequence values
Recent transactions
```

---

# 48. Example Business Consistency Check

Suppose:

```text
orders.order_id = 1001
```

Verify:

```text
Order exists
Payment exists
Inventory reservation exists
Customer exists
```

If:

```text
Order = CONFIRMED
Payment = FAILED
```

the database may be technically healthy while business state is inconsistent.

Therefore recovery requires both:

```text
Technical validation
+
Business validation
```

---

# 49. RDS Recovery and Messaging

Suppose order creation writes:

```text
Order DB
+
Kafka event
```

During recovery, the database and messaging system may temporarily diverge.

Example:

```text
Order inserted
      |
      X
Kafka publish failed
```

Therefore our architecture uses the transactional outbox pattern.

Recovery should inspect:

```text
outbox table
Kafka events
consumer offsets
processed events
```

and safely replay outstanding events where required.

---

# 50. RDS Recovery and Idempotency

Recovery can cause retries.

Example:

```text
Order created
   |
   v
Application timeout
   |
   v
Client retries
```

Without idempotency:

```text
Order 1001
Order 1002
```

could accidentally represent the same customer request.

Therefore:

```text
Idempotency key
+
Unique constraint
+
Business validation
```

are part of recovery safety.

---

# 51. Bad Migration Recovery

Scenario:

```text
V1
 ↓
V2
 ↓
V3
 ↓
V4 bad migration
```

Do not automatically delete:

```text
V4
```

from source control if it has already been applied.

Instead:

```text
Identify failure
   ↓
Stop bad deployment
   ↓
Assess schema/data impact
   ↓
Restore if necessary
   OR
Forward-fix schema/data
   ↓
Validate
```

This preserves migration history.

---

# 52. Logical Corruption Example

Suppose an application accidentally runs:

```sql
UPDATE orders
SET status = 'CANCELLED';
```

without a `WHERE` clause.

The database itself may remain:

```text
AVAILABLE
```

and:

```text
Multi-AZ = healthy
```

But the data is wrong.

This is a **logical corruption** scenario.

Recovery:

```text
Stop bad writer
 ↓
Identify corruption time
 ↓
Determine recovery point
 ↓
PITR to temporary instance
 ↓
Validate
 ↓
Cut over
```

---

# 53. Accidental Deletion

Suppose:

```text
DROP TABLE orders;
```

occurs.

Immediately:

```text
Stop further destructive activity
```

Then determine:

```text
Was the table recoverable through migration?
Was data backed up?
Was PITR enabled?
Is there a recent snapshot?
```

For major production data loss:

```text
PITR / snapshot restore
```

may be required.

---

# 54. Recovery Into Temporary RDS

Example:

```text
Production RDS
     |
     v
PITR
     |
     v
rds-recovery-validation
     |
     +--> schema validation
     +--> data validation
     +--> business validation
```

Do not immediately replace production.

---

# 55. Recovery Validation Environment

The temporary recovery environment should have:

```text
Private subnet
Restricted SG
Restricted IAM
Restricted access
No public endpoint
Audit logging
Monitoring
```

Access should be limited to the recovery team.

---

# 56. Compare Production vs Recovery

Compare:

```text
Engine version
Schema
Migration version
Table counts
Critical records
Indexes
Constraints
Extensions
Parameter group
```

Example:

```text
Production migration:
V42

Recovery migration:
V42
```

Then inspect critical business data.

---

# 57. Cutover Planning

Before switching:

```text
Recovery DB
      |
      v
Validated
      |
      v
Application stopped/frozen
      |
      v
Final consistency check
      |
      v
Connection configuration updated
      |
      v
Application restarted
      |
      v
Business validation
```

Cutover should have a rollback plan.

---

# 58. Rollback Planning

Suppose restored DB is deployed but validation fails.

Do not immediately destroy the original database.

Keep:

```text
Original DB
```

available until the recovery is confirmed.

Potential state:

```text
Original RDS
     |
     | preserved
     |
     +------> Recovery RDS
                    |
                    +--> validation
```

Only retire resources after the incident/recovery process explicitly approves it.

---

# 59. RDS Recovery Monitoring

During recovery monitor:

```text
CPU
Memory
Storage
IOPS
Latency
Connections
Locks
Transactions
Replication
Application errors
```

Application metrics:

```text
HTTP 5xx
HTTP latency
DB query latency
Connection acquisition time
Connection timeout
```

---

# 60. Observability During Recovery

Use all three pillars.

### Metrics

```text
Prometheus
Grafana
RDS metrics
```

### Logs

```text
ELK
Kibana
```

### Traces

```text
OpenTelemetry
Jaeger
```

Example:

```text
Customer
 ↓
ALB
 ↓
order-service
 ↓
PostgreSQL
```

Jaeger can reveal whether latency is coming from:

```text
Application
Database
External API
```

---

# 61. Recovery Timeline Example

```text
10:00
Normal operation

10:20
Bad migration deployed

10:27
Application errors begin

10:30
Incident detected

10:32
Migration stopped

10:35
Database corruption confirmed

10:40
Recovery timestamp determined

10:45
PITR started

11:05
Recovery DB available

11:10
Schema validation complete

11:15
Business data validation complete

11:20
Application cutover

11:25
Checkout validated

11:30
Incident resolved
```

Measured:

```text
RTO = 63 minutes
```

If target was:

```text
60 minutes
```

we have a recovery gap of:

```text
3 minutes
```

The purpose of the exercise is to discover this.

---

# 62. RDS Recovery Game Day

Dedicated DR environment:

```text
ecommerce-dr
```

Scenario:

> Production database has suffered logical corruption.

Exercise:

```text
1. Generate test traffic
2. Introduce controlled corruption
3. Detect incident
4. Stop bad writer
5. Identify corruption timestamp
6. Start PITR
7. Validate recovery DB
8. Validate Flyway
9. Validate business data
10. Redirect application
11. Run checkout
12. Measure RTO/RPO
13. Document gaps
```

---

# 63. Recovery Test Data

Use synthetic/test data.

Example:

```text
Customer:
DR-CUSTOMER-001

Product:
DR-PRODUCT-001

Order:
DR-ORDER-001
```

This makes business validation easy.

Example:

```text
Create order
 ↓
Payment
 ↓
Inventory
 ↓
Notification
```

---

# 64. RDS Recovery Checklist

## Detection

```text
[ ] Incident declared
[ ] Impact identified
[ ] Database status checked
[ ] RDS events checked
[ ] CloudTrail checked
[ ] Application logs checked
```

## Protection

```text
[ ] Bad writer stopped
[ ] Evidence preserved
[ ] Existing RDS protected
[ ] No unnecessary destructive action
```

## Recovery

```text
[ ] Recovery method selected
[ ] Snapshot/PITR point identified
[ ] Temporary DB created
[ ] Security groups verified
[ ] Encryption verified
[ ] Network verified
```

## Validation

```text
[ ] PostgreSQL version verified
[ ] Schema verified
[ ] Flyway verified
[ ] Tables verified
[ ] Indexes verified
[ ] Constraints verified
[ ] Sequences verified
[ ] Critical business data verified
```

## Application

```text
[ ] DB endpoint configured
[ ] Secrets updated
[ ] Pods restarted if required
[ ] Hikari connections healthy
[ ] HTTP errors normal
[ ] Business transaction successful
```

## Completion

```text
[ ] RTO measured
[ ] RPO measured
[ ] Monitoring normal
[ ] Original DB preserved until safe
[ ] Incident documented
[ ] RCA created
[ ] Preventive actions identified
```

---

# 65. RDS Recovery Architecture

Final design:

```text
                         Production
                             |
                             v
                     RDS PostgreSQL
                             |
              +--------------+--------------+
              |                             |
        Multi-AZ HA                   Automated Backup
              |                             |
              |                             v
              |                         PITR
              |                             |
              |                             v
              |                    Recovery RDS
              |                             |
              +-----------------------------+
                            |
                         Validation
                            |
                            v
                     Application Cutover
                            |
                            v
                    Business Validation
```

---

# 66. The Most Important Distinction

There are three different problems:

### Availability problem

```text
RDS primary failed
```

Use:

```text
Multi-AZ failover
```

### Infrastructure recovery problem

```text
RDS instance/environment lost
```

Use:

```text
Snapshot / PITR
```

### Logical corruption problem

```text
Application damaged valid data
```

Use:

```text
Stop writer
+
PITR/snapshot
+
Business validation
```

These must not be treated as the same incident.

---

# 67. Production Recovery Principle

The complete RDS recovery process is:

```text
Detect
  ↓
Protect
  ↓
Investigate
  ↓
Choose recovery point
  ↓
Restore
  ↓
Validate
  ↓
Cut over
  ↓
Verify business
  ↓
Monitor
  ↓
RCA
```

The most important rule is:

> **Never confuse high availability with data recovery.**

Multi-AZ helps keep PostgreSQL available during infrastructure failures.

Backups and PITR protect the ability to recover from logical mistakes, corruption, deletion, and other data-loss scenarios.

---

## Next Chapter

**137 — S3 Recovery**

We will cover:

```text
S3 accidental deletion
Versioning
Object recovery
Delete markers
MFA/Delete protection concepts
Cross-Region Replication
S3 backup strategy
KMS-encrypted objects
Bucket recovery
Terraform recovery
Application dependency recovery
EKS + S3 IAM recovery
RPO/RTO
Controlled S3 disaster-recovery game day
```
