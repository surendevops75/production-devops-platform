# 106 — Database Migration with Flyway / Liquibase

> **Project:** Enterprise E-Commerce Platform
> **Primary Service:** `order-service`
> **Database:** PostgreSQL on Amazon RDS
> **Deployment:** EKS + Helm + ArgoCD
> **CI:** GitHub Actions
> **Migration Strategy:** Version-controlled, forward-only production migrations with expand-and-contract for breaking schema changes

---

## 1. Purpose

In Chapter 105 we implemented:

```text
EKS
  ↓
order-service
  ↓
RDS PostgreSQL
  ↓
Secrets Manager
  ↓
Private networking
  ↓
Multi-AZ
```

Now we need to solve a critical production problem:

> **How does the database schema change safely when application code changes?**

We cannot depend on:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

in production.

Instead:

```text
Developer
   ↓
Migration file
   ↓
Git
   ↓
PR
   ↓
CI validation
   ↓
Test database
   ↓
QA
   ↓
STAGING
   ↓
Production migration
   ↓
Application deployment
```

---

# 2. Why Database Migrations Matter

Suppose version 1 of `orders` contains:

```text
orders
├── id
├── customer_id
├── product_id
├── quantity
└── status
```

A developer adds:

```text
payment_reference
```

The application now expects:

```text
orders.payment_reference
```

But production RDS still has the old schema.

The application can fail.

Therefore application version and database schema must evolve together.

---

# 3. The Wrong Production Approach

Do not use:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

Why?

Because application startup becomes responsible for modifying production schema.

That creates risks around:

```text
automatic DDL
locking
deployment ordering
rollback
multiple replicas
schema compatibility
unexpected changes
```

With multiple Pods:

```text
Pod 1
Pod 2
Pod 3
Pod 4
```

all starting around the same time, you do not want each application instance deciding how to modify the production database.

---

# 4. Migration Ownership

The production ownership model should be:

```text
Application team
       │
       ├── writes migration
       │
       ▼
Git
       │
       ▼
CI validates migration
       │
       ▼
Release process
       │
       ▼
Migration execution
       │
       ▼
RDS
```

The database schema is therefore part of the application's version-controlled source.

---

# 5. Flyway vs Liquibase

Both are valid.

### Flyway

Typically emphasizes:

```text
versioned SQL migrations
simple migration model
developer-friendly workflow
```

Example:

```text
V1__create_orders.sql
V2__add_payment_reference.sql
V3__add_order_indexes.sql
```

### Liquibase

Provides:

```text
XML
YAML
JSON
SQL
changeSets
rollback metadata
```

Example:

```text
001-create-orders.yaml
002-add-payment-reference.yaml
```

For this project, we will use **Flyway as the primary implementation** because our PostgreSQL schema changes are easy to understand as explicit SQL.

The same production principles apply to Liquibase.

---

# 6. Migration Architecture

Our application repository becomes:

```text
order-service/
├── src/
├── db/
│   └── migration/
│       ├── V1__create_orders.sql
│       ├── V2__add_payment_reference.sql
│       └── V3__add_order_indexes.sql
├── Dockerfile
├── pom.xml
├── helm/
└── .github/
    └── workflows/
```

---

# 7. Add Flyway Dependency

For Maven:

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

For PostgreSQL, depending on the Flyway version being used, the PostgreSQL database support artifact may also be required.

The exact dependency versions should be aligned with the Spring Boot and Flyway versions used by the project.

---

# 8. Migration Directory

Spring Boot/Flyway convention:

```text
src/main/resources/db/migration/
```

Therefore our repository should actually use:

```text
src/
└── main/
    └── resources/
        └── db/
            └── migration/
                ├── V1__create_orders.sql
                ├── V2__add_payment_reference.sql
                └── V3__add_order_indexes.sql
```

This is preferable to putting migrations somewhere the application won't automatically discover.

---

# 9. Migration Naming

Flyway versioned migration format:

```text
V<version>__<description>.sql
```

Examples:

```text
V1__create_orders.sql

V2__add_payment_reference.sql

V3__create_order_status_index.sql
```

Notice:

```text
V1__
```

contains two underscores.

---

# 10. First Migration

File:

```text
V1__create_orders.sql
```

Example:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

This becomes the initial schema definition.

---

# 11. Second Migration

Suppose payment integration requires:

```text
payment_reference
```

Create:

```text
V2__add_payment_reference.sql
```

with:

```sql
ALTER TABLE orders
ADD COLUMN payment_reference VARCHAR(100);
```

We do **not** modify `V1`.

Once a migration has been applied, treat it as immutable.

---

# 12. Why We Don't Edit V1

Suppose production already executed:

```text
V1
V2
```

If we modify:

```text
V1__create_orders.sql
```

the migration checksum changes.

Flyway can detect that the migration previously applied does not match the file currently in Git.

This is intentional.

It prevents silent history rewriting.

---

# 13. Migration History

Flyway creates a metadata table.

Typically:

```text
flyway_schema_history
```

It records information such as:

```text
version
description
type
script
checksum
installed_by
installed_on
execution_time
success
```

Conceptually:

```text
flyway_schema_history
----------------------------------------
1   create_orders
2   add_payment_reference
3   add_order_indexes
```

This gives us database migration history.

---

# 14. Migration State

Suppose:

```text
Git:
V1
V2
V3
```

RDS:

```text
V1
V2
```

When migration runs:

```text
V3
```

is detected as pending.

Flyway executes it.

After success:

```text
RDS:
V1
V2
V3
```

---

# 15. Application Startup Migration

Flyway can run migrations automatically when the application starts.

Example:

```yaml
spring:
  flyway:
    enabled: true
```

But this raises an important production architecture question:

> **Should every application Pod be responsible for executing migrations?**

For simple systems, automatic Flyway startup migration can work.

For our production platform, we want clearer deployment control.

---

# 16. Recommended Production Pattern

Use a dedicated migration step/job:

```text
GitOps
   ↓
Migration Job
   ↓
Flyway
   ↓
RDS
   ↓
success
   ↓
Application Deployment
```

Instead of:

```text
10 application Pods
   ↓
10 application startup processes
   ↓
all attempting migration
```

Flyway has locking behavior, but a dedicated migration step makes the release process easier to reason about and audit.

---

# 17. Kubernetes Migration Job

Conceptually:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: order-service-migration
  namespace: ecommerce-prod
spec:
  backoffLimit: 3

  template:
    spec:
      restartPolicy: Never

      containers:
        - name: flyway
          image: flyway/flyway:<approved-version>
          command:
            - flyway
            - migrate
```

The real production Job needs:

```text
RDS credentials
RDS endpoint
network access
IAM/Secrets integration
resource requests
security context
```

and should use an approved, pinned Flyway image version.

---

# 18. Migration Image Strategy

We can use:

```text
Flyway image
```

or:

```text
order-service image
```

containing Flyway and migrations.

For this project, keeping migrations alongside the application source provides strong traceability:

```text
Application commit
+
Migration files
```

However, the migration execution mechanism should still be a distinct deployment step.

---

# 19. Better Production Flow

Our release becomes:

```text
Developer
   ↓
Application code
+
Migration
   ↓
PR
   ↓
CI
   ↓
Tests
   ↓
Migration validation
   ↓
Build image
   ↓
ECR
   ↓
STAGING
   ↓
Migration
   ↓
Application
   ↓
Production approval
   ↓
Migration
   ↓
Application deployment
```

---

# 20. Critical Rule — Schema Before Code

Suppose new application code requires:

```text
payment_reference
```

The production order should generally be:

```text
1. Add compatible schema
2. Verify migration
3. Deploy compatible application
4. Start using new field
```

Not:

```text
1. Deploy application requiring new column
2. Hope migration catches up
```

The second approach can produce:

```text
column does not exist
```

errors.

---

# 21. Expand-and-Contract

For safe production schema evolution:

```text
EXPAND
   ↓
MIGRATE
   ↓
DEPLOY
   ↓
SWITCH
   ↓
CONTRACT
```

This is one of the most important database deployment patterns.

---

# 22. Example — Rename Column

Suppose old column:

```text
customer_id
```

needs to become:

```text
buyer_id
```

Don't immediately do:

```sql
ALTER TABLE orders
RENAME COLUMN customer_id TO buyer_id;
```

if old application Pods are still running.

During a rolling deployment:

```text
Pod A → old code
Pod B → old code
Pod C → new code
```

Old Pods expect:

```text
customer_id
```

New Pods expect:

```text
buyer_id
```

Immediate rename breaks one side.

---

# 23. Safe Rename Strategy

### Step 1 — Expand

Add:

```sql
ALTER TABLE orders
ADD COLUMN buyer_id BIGINT;
```

### Step 2 — Deploy compatible application

Application can read/write both:

```text
customer_id
buyer_id
```

### Step 3 — Backfill

Copy:

```text
customer_id → buyer_id
```

### Step 4 — Switch

New code uses:

```text
buyer_id
```

### Step 5 — Contract

After all old Pods are gone and the compatibility window has passed:

```sql
ALTER TABLE orders
DROP COLUMN customer_id;
```

---

# 24. Why This Matters With Kubernetes

Our production deployment uses:

```text
RollingUpdate
```

There can temporarily be:

```text
old Pods
+
new Pods
```

Therefore database schema must support both versions during the rollout.

This connects:

```text
Database migration
+
Rolling deployment
+
Blue-Green
+
Canary
```

into one release architecture.

---

# 25. Additive Changes Are Easier

Usually safer:

```sql
ALTER TABLE orders
ADD COLUMN payment_reference VARCHAR(100);
```

because old application versions can continue operating.

Potentially dangerous changes:

```text
drop column
rename column
change type
make nullable → NOT NULL
change semantics
```

These require compatibility planning.

---

# 26. Adding NOT NULL Safely

Suppose we want:

```text
customer_email NOT NULL
```

Don't immediately run:

```sql
ALTER TABLE orders
ADD COLUMN customer_email VARCHAR(255) NOT NULL;
```

on a populated production table without understanding the database/version/locking implications.

Safer pattern:

```text
1. Add nullable column
2. Deploy application
3. Backfill
4. Verify no NULLs
5. Add constraint
```

For example:

```sql
ALTER TABLE orders
ADD COLUMN customer_email VARCHAR(255);
```

Then backfill.

Then:

```sql
ALTER TABLE orders
ALTER COLUMN customer_email SET NOT NULL;
```

The exact locking/performance impact must be tested before production.

---

# 27. Large Table Warning

Suppose:

```text
orders
=
500 million rows
```

A seemingly simple migration can become a production incident.

Before executing:

```sql
ALTER TABLE ...
```

consider:

```text
table size
index size
lock duration
query activity
replication
maintenance
available storage
IOPS
transaction duration
```

Never assume a migration that is instant in DEV will be instant in production.

---

# 28. Index Creation

Suppose we need:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

For large production PostgreSQL tables, consider whether:

```sql
CREATE INDEX CONCURRENTLY
```

is more appropriate.

Example:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id
ON orders(customer_id);
```

But there are trade-offs.

`CREATE INDEX CONCURRENTLY` has different transaction behavior and operational implications.

Test it before production.

---

# 29. Migration Transaction Behavior

Some PostgreSQL operations can run inside transactions.

Others have special restrictions.

For example:

```text
CREATE INDEX CONCURRENTLY
```

cannot be executed inside a normal transaction block.

Therefore migration tooling must understand:

```text
transactional migration
vs
non-transactional migration
```

Flyway supports configuration/options for such migrations.

Do not assume every SQL statement can be grouped into one transaction.

---

# 30. Migration Failure

Suppose:

```text
V4__add_index.sql
```

fails.

Flyway records migration state.

Do not immediately:

```text
delete flyway history
```

or manually manipulate migration metadata.

First investigate:

```text
SQL error
lock
permissions
database storage
connection
partial execution
transaction behavior
```

Then determine the correct recovery strategy.

---

# 31. Failed Migration Example

Suppose:

```sql
ALTER TABLE orders
ADD COLUMN payment_reference VARCHAR(100) NOT NULL;
```

fails because existing rows contain no value.

The correct response is not:

```text
ignore error
```

Instead design the migration correctly:

```text
1. Add nullable column
2. Deploy compatible application
3. Backfill
4. Validate
5. Add NOT NULL
```

---

# 32. Flyway Validate

Before migration:

```bash
flyway validate
```

This checks migration consistency.

For example:

```text
Git migration
     ↓
Flyway metadata
     ↓
checksum validation
```

If someone changed:

```text
V2__add_payment_reference.sql
```

after it was applied, validation can detect the mismatch.

---

# 33. Flyway Info

Useful command:

```bash
flyway info
```

This can show:

```text
Pending
Applied
Success
Failed
```

Conceptually:

```text
Version   Description             State
------------------------------------------------
1         create_orders           Success
2         add_payment_reference   Success
3         add_index               Pending
```

---

# 34. Flyway Migrate

The actual migration command:

```bash
flyway migrate
```

Flow:

```text
Connect RDS
   ↓
Read flyway history
   ↓
Find pending migrations
   ↓
Validate
   ↓
Execute
   ↓
Record success
```

---

# 35. Flyway Baseline

Suppose an existing production database already exists but has no migration history.

We cannot simply pretend it started at V1.

A baseline can establish:

```text
Current database
       ↓
Baseline version
```

Then future migrations begin after that version.

For example:

```text
baseline = 1
```

Then:

```text
V2
V3
V4
```

can be applied.

Baseline strategy must be carefully planned for existing databases.

---

# 36. Don't Baseline a New Empty Database Incorrectly

For a fresh database:

```text
empty RDS
```

we generally want:

```text
V1
V2
V3
```

to actually execute.

Don't accidentally baseline away the migrations you intended to run.

---

# 37. CI Migration Validation

Our GitHub Actions pipeline should validate:

```text
SQL syntax
migration naming
migration ordering
application tests
migration execution against PostgreSQL
```

Example pipeline:

```text
Checkout
   ↓
Build
   ↓
Unit tests
   ↓
Start PostgreSQL
   ↓
Flyway migrate
   ↓
Integration tests
   ↓
Flyway validate
   ↓
SonarQube
   ↓
Trivy
   ↓
Docker build
```

---

# 38. Test PostgreSQL in CI

GitHub Actions can use a PostgreSQL service container.

Conceptually:

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_DB: orders
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - 5432:5432
```

Then CI runs:

```text
Flyway
   ↓
PostgreSQL
   ↓
Migrations
```

before building the production image.

---

# 39. Migration Integration Test

Example:

```text
Empty PostgreSQL
      ↓
V1
      ↓
V2
      ↓
V3
      ↓
Application integration tests
```

Verify:

```text
tables exist
columns exist
indexes exist
constraints exist
application queries work
```

---

# 40. Migration Test With Existing Data

Don't test only an empty database.

Also create:

```text
representative production-like data
```

Then execute:

```text
migration
```

and verify:

```text
existing rows preserved
new columns populated
constraints valid
indexes created
queries still work
```

This is especially important for:

```text
large tables
backfills
data transformations
```

---

# 41. Migration Performance Testing

For important production migrations measure:

```text
execution time
locks
CPU
IOPS
storage
database connections
query latency
```

During staging:

```text
migration starts
       ↓
Prometheus/RDS metrics
       ↓
observe database impact
```

---

# 42. Lock Monitoring

PostgreSQL can have blocking queries.

During migration, investigate locks if required using PostgreSQL system views.

For example:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

This can help identify active sessions and waits.

Use carefully in production and avoid exposing sensitive query contents unnecessarily.

---

# 43. Blocking Query Example

Suppose migration:

```text
ALTER TABLE orders ...
```

needs a lock.

But a long transaction is holding it.

Then:

```text
Migration
   ↓
waiting for lock
   ↓
application latency
```

Potentially:

```text
API requests
   ↓
database waits
   ↓
connection pool exhausted
   ↓
5xx
```

A migration can therefore become an application incident.

---

# 44. Production Migration Safety

Before running:

```text
[ ] backup/PITR available
[ ] migration tested
[ ] staging successful
[ ] execution time known
[ ] locking behavior understood
[ ] rollback/recovery plan
[ ] monitoring active
[ ] on-call available
[ ] change approved
```

---

# 45. Migration Rollback Reality

Database rollback is not always:

```text
flyway undo
```

Many production teams prefer:

```text
forward-compatible fix
```

instead of attempting destructive rollback.

Example:

```text
V4 introduced column
```

If V4 has a bug:

```text
V5 fixes the behavior
```

rather than:

```text
delete V4
```

This is safer for distributed application deployments.

---

# 46. Forward-Only Migration Strategy

Our preferred model:

```text
V1
 ↓
V2
 ↓
V3
 ↓
V4
 ↓
V5
```

Never rewrite:

```text
V2
```

after production.

If something needs correction:

```text
V6__fix_previous_behavior.sql
```

---

# 47. Data Migration vs Schema Migration

These are different.

### Schema migration

```text
ADD COLUMN
CREATE TABLE
CREATE INDEX
ALTER CONSTRAINT
```

### Data migration

```text
UPDATE orders
SET ...
```

Data migrations can be significantly more expensive.

For large datasets, consider:

```text
batch processing
background jobs
throttling
checkpoints
monitoring
```

instead of one giant transaction.

---

# 48. Example Data Backfill

Suppose:

```text
payment_reference
```

must be populated.

Instead of:

```sql
UPDATE orders
SET payment_reference = ...
WHERE payment_reference IS NULL;
```

across hundreds of millions of rows in one transaction, consider a controlled batch process.

Conceptually:

```text
1000 rows
 ↓
commit
 ↓
1000 rows
 ↓
commit
 ↓
...
```

The exact strategy depends on data size and consistency requirements.

---

# 49. Migration and Deployment Strategy

Our deployment flow becomes:

```text
                    Git
                     │
              Application + DB
                     │
                     ▼
                    CI
                     │
          ┌──────────┴──────────┐
          │                     │
      App tests             DB tests
          │                     │
          └──────────┬──────────┘
                     ▼
                    ECR
                     │
                     ▼
                  GitOps
                     │
                     ▼
                  ArgoCD
                     │
             ┌───────┴───────┐
             ▼               ▼
        Migration Job    Application
             │               │
             ▼               ▼
             └────── RDS ────┘
```

---

# 50. ArgoCD Ordering

We need to ensure:

```text
migration
```

runs before:

```text
new application
```

when the application requires the new schema.

One mechanism is ArgoCD sync waves.

Conceptually:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Migration:

```text
wave -1
```

Application:

```text
wave 0
```

This gives:

```text
Migration
   ↓
Application
```

However, a production migration Job should also be designed for reruns/idempotency and clear success/failure handling.

---

# 51. Migration Job Idempotency

Suppose ArgoCD retries.

We don't want:

```text
migration executes destructive operation twice
```

Flyway itself tracks applied migrations.

Therefore:

```text
V1
V2
V3
```

are not executed repeatedly after successful application.

The migration Job should be safe to rerun against the same database state.

---

# 52. Migration Job Failure

Suppose:

```text
Migration Job
   ↓
FAILED
```

ArgoCD should not blindly continue to:

```text
new application
```

because the application may require the missing schema.

The desired release flow is:

```text
Migration failed
     ↓
Production deployment stops
     ↓
Engineer investigates
     ↓
Fix migration
     ↓
Test
     ↓
Retry
```

---

# 53. Production Change Management

Our previous Jira flow:

```text
CHG-10452
```

can include:

```text
Database migration:
V12__add_payment_reference.sql
```

Change record:

```text
Change ID
Service
Migration version
Database
Risk
Implementation plan
Validation plan
Rollback/recovery plan
Execution window
Approver
```

Traceability:

```text
Jira Change
   ↓
Git PR
   ↓
Migration commit
   ↓
Application image
   ↓
GitOps commit
   ↓
ArgoCD
   ↓
Migration Job
   ↓
RDS
```

---

# 54. Production Migration Example

Suppose current:

```text
Application:
v1.4.0

Database:
V8
```

New release:

```text
Application:
v1.5.0

Database:
V9
```

V9 adds:

```text
payment_reference
```

Release:

```text
1. Test V9
2. Deploy migration
3. Verify V9
4. Deploy v1.5.0
5. Smoke test
6. Monitor
```

---

# 55. Canary Deployment Consideration

Suppose:

```text
Stable:
v1.4.0
```

Canary:

```text
v1.5.0
```

If v1.5.0 requires V9:

```text
V9 must be compatible with v1.4.0
```

because:

```text
Stable Pods
+
Canary Pods
```

run simultaneously.

This is exactly why expand-and-contract is essential.

---

# 56. Blue-Green Consideration

Blue:

```text
v1.4.0
```

Green:

```text
v1.5.0
```

Both may temporarily exist.

Therefore:

```text
Database V9
```

must support both versions.

Do not perform an incompatible schema change before switching traffic.

---

# 57. Database Migration and Rollback

Application rollback:

```text
v1.5.0 → v1.4.0
```

must still work against the current database schema.

Therefore:

```text
DB schema
```

should remain backward-compatible for the rollback window.

This gives:

```text
Application rollback
without immediate database rollback
```

which is usually safer.

---

# 58. Migration Repository Strategy

We will keep migrations with the service:

```text
order-service/
└── src/
    └── main/
        └── resources/
            └── db/
                └── migration/
```

This provides:

```text
application version
+
migration version
```

in the same Git history.

---

# 59. Migration Naming Rules

Use:

```text
V1__create_orders.sql
V2__add_payment_reference.sql
V3__create_customer_index.sql
V4__add_order_total.sql
```

Avoid:

```text
update.sql
latest.sql
fix.sql
new.sql
final.sql
final2.sql
```

A migration should clearly communicate its purpose.

---

# 60. Migration Review Checklist

Every migration PR should answer:

```text
What changes?
Why?
How large is the affected table?
Will it lock?
How long should it take?
Is it backward-compatible?
Can old application Pods continue?
Does it require a backfill?
Does it require an index?
What happens if it fails?
How do we recover?
```

---

# 61. Dangerous Migration Examples

Be especially careful with:

```sql
DROP TABLE orders;
```

```sql
DROP COLUMN customer_id;
```

```sql
ALTER TABLE orders
ALTER COLUMN customer_id TYPE ...;
```

```sql
ALTER TABLE orders
ADD COLUMN x ... NOT NULL;
```

```sql
UPDATE orders SET ...
```

on very large tables.

These should receive significantly more review than a simple additive migration.

---

# 62. Migration Security

Database migration credentials should not be:

```text
GitHub repository secrets pasted into SQL
```

Instead:

```text
Migration Job
      ↓
ServiceAccount
      ↓
AWS Pod Identity
      ↓
Secrets Manager
      ↓
Database credentials
```

The migration Job gets only the database permissions it requires.

---

# 63. Migration Credential Separation

Ideally:

```text
Application user
```

and:

```text
Migration user
```

are not necessarily identical.

Why?

Application:

```text
SELECT
INSERT
UPDATE
DELETE
```

Migration:

```text
DDL
ALTER
CREATE INDEX
```

Giving the application unrestricted DDL privileges increases blast radius.

A more mature architecture separates these privileges.

---

# 64. Example Database Roles

Conceptually:

```text
orders_app
    ↓
DML only

orders_migrator
    ↓
DDL + migration permissions
```

Then:

```text
Application Pod
    ↓
orders_app

Migration Job
    ↓
orders_migrator
```

This is a stronger least-privilege model.

---

# 65. Migration Observability

We should monitor:

```text
migration duration
migration success/failure
database CPU
database connections
database latency
locks
IOPS
storage
application errors
```

A migration dashboard should allow an engineer to answer:

> "Did the migration cause the application problem?"

---

# 66. Migration Incident Example

Production:

```text
Migration starts
```

Within 30 seconds:

```text
RDS CPU:
45% → 90%

DB latency:
10ms → 700ms
```

Application:

```text
HTTP 500 increases
```

Investigation:

```text
migration executing expensive operation
```

Response:

```text
1. Stop/contain migration if safe
2. Protect application traffic
3. Evaluate locks/transactions
4. Follow migration recovery procedure
5. Stabilize database
6. Validate application
7. Record incident
```

Never kill database processes blindly during an incident.

---

# 67. Migration Testing Environments

We should validate through:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

Each environment should use:

```text
same migration files
```

The difference is:

```text
data volume
traffic
database size
configuration
```

---

# 68. Production-Like Staging

For important migrations, staging should approximate:

```text
production schema
+
representative data volume
+
similar indexes
+
similar database version
```

A migration that works against:

```text
100 rows
```

is not automatically proven safe against:

```text
500 million rows
```

---

# 69. Migration CI Pipeline

Example:

```yaml
name: Database Migration Validation

on:
  pull_request:

jobs:
  migration-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: orders
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"

      - name: Run tests
        run: ./mvnw test

      - name: Run Flyway migration
        run: ./mvnw flyway:migrate

      - name: Validate Flyway
        run: ./mvnw flyway:validate
```

Exact Maven Flyway plugin configuration must be present before these commands are used.

---

# 70. Better CI Flow

Our existing CI becomes:

```text
Checkout
   ↓
Compile
   ↓
Unit Tests
   ↓
Integration Tests
   ↓
PostgreSQL
   ↓
Flyway Migration
   ↓
Schema Validation
   ↓
SonarQube
   ↓
Dependency Scan
   ↓
Secret Scan
   ↓
Docker Build
   ↓
Trivy
   ↓
ECR
```

---

# 71. Production Deployment Flow

Our production release:

```text
Developer
    ↓
PR
    ↓
CI
    ↓
Image
    ↓
ECR
    ↓
GitOps PR
    ↓
Jira Change
    ↓
Production Approval
    ↓
ArgoCD
    ↓
Migration Job
    ↓
Migration Success
    ↓
Application Rollout
    ↓
Smoke Test
    ↓
Monitoring
```

---

# 72. What Happens If Migration Fails?

```text
             Migration
                 │
           ┌─────┴─────┐
           │           │
        Success       Fail
           │           │
           ▼           ▼
      Deploy App     STOP
           │           │
           ▼           ▼
       Validate    Investigate
                       │
                       ▼
                   Fix/Test
                       │
                       ▼
                     Retry
```

Do not proceed with an incompatible application release after migration failure.

---

# 73. What Happens If Application Deployment Fails?

Suppose:

```text
Migration V10 = successful
Application v2 = failed
```

We don't automatically roll back V10.

Instead:

```text
Database remains V10
        │
        ▼
Rollback application to compatible version
```

This is why V10 should have been backward-compatible.

---

# 74. Migration Compatibility Matrix

Example:

| Application | DB V8 | DB V9 |
| ----------- | ----: | ----: |
| v1.4        |     ✅ |     ✅ |
| v1.5        |     ❌ |     ✅ |

Desired production transition:

```text
v1.4 + V8
      ↓
v1.4 + V9
      ↓
v1.5 + V9
```

This is safe.

Bad transition:

```text
v1.4 + V8
      ↓
v1.5 + V9
```

if there is no compatibility period.

---

# 75. Migration Documentation

Each migration should be documented when it has operational risk.

Example:

```text
Migration:
V12__add_order_total.sql

Purpose:
Add calculated order total.

Affected table:
orders

Estimated rows:
150M

Expected impact:
Low/Medium

Lock:
Requires testing

Backfill:
Required

Application compatibility:
v1.7 and later

Rollback:
Forward fix preferred
```

---

# 76. Liquibase Equivalent

If we choose Liquibase instead, the architecture remains:

```text
Git
 ↓
changeset
 ↓
CI
 ↓
validate
 ↓
test DB
 ↓
STAGING
 ↓
production
```

Example:

```yaml
databaseChangeLog:
  - changeSet:
      id: 001
      author: platform-team
      changes:
        - createTable:
            tableName: orders
            columns:
              - column:
                  name: id
                  type: BIGINT
                  constraints:
                    primaryKey: true
```

Liquibase maintains its own changelog metadata.

---

# 77. Flyway vs Liquibase for Our Project

For our implementation:

```text
Flyway
   ↓
SQL-first
   ↓
simple versioning
   ↓
easy PostgreSQL ownership
```

Liquibase would be reasonable when we need:

```text
structured changesets
database abstraction
built-in rollback metadata
more declarative change management
```

We will continue with Flyway.

---

# 78. Production Migration Golden Rules

```text
1. Never edit an applied migration.
2. Never use ddl-auto=update in production.
3. Test migrations against PostgreSQL.
4. Test with realistic data.
5. Understand locks.
6. Use backward-compatible changes.
7. Prefer expand-and-contract.
8. Keep migrations version controlled.
9. Separate migration and application privileges where practical.
10. Monitor database health during migration.
11. Treat destructive changes as high-risk.
12. Prefer forward fixes over destructive rollback.
```

---

# 79. Complete Database Release Architecture

```text
                 GitHub
                    │
          ┌─────────┴─────────┐
          │                   │
    Application Code     DB Migrations
          │                   │
          └─────────┬─────────┘
                    ▼
                   CI
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     Tests       Flyway       Security
                    │
                    ▼
             PostgreSQL Test DB
                    │
                    ▼
                   ECR
                    │
                    ▼
                 GitOps
                    │
                    ▼
                 ArgoCD
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
   Migration Job        Application
          │                   │
          └─────────┬─────────┘
                    ▼
             RDS PostgreSQL
```

---

# 80. Final Production Checklist

### Migration source

```text
[ ] Migration files stored in Git
[ ] Naming convention enforced
[ ] Applied migrations immutable
[ ] Migration review required
```

### CI

```text
[ ] PostgreSQL test database
[ ] Flyway migrate
[ ] Flyway validate
[ ] Integration tests
[ ] Security scanning
```

### Production

```text
[ ] Migration tested in staging
[ ] Change approved
[ ] Backup/PITR available
[ ] Migration Job defined
[ ] Migration credentials protected
[ ] Application compatibility verified
[ ] Monitoring enabled
[ ] Recovery procedure documented
```

### Deployment

```text
[ ] Expand before contract
[ ] Migration before incompatible application
[ ] Old application remains compatible
[ ] Rollback strategy tested
[ ] Smoke tests after migration
```

---

# 81. Our Database Lifecycle

We now have:

```text
                    Database Lifecycle

Create
  ↓
Terraform
  ↓
RDS
  ↓
Schema V1
  ↓
Flyway
  ↓
V2
  ↓
V3
  ↓
V4
  ↓
Application releases
  ↓
Backup
  ↓
PITR
  ↓
Failover
  ↓
Recovery
```

This is the production database lifecycle we are building across Chapters 105–109.

---

# 82. Current Project Progress

Completed:

```text
100 — Spot Instances
101 — Multi-AZ Workload Placement
102 — Kubernetes Resource Management
103 — HPA / VPA / KEDA Production Implementation
104 — Cluster Autoscaling
105 — RDS Production Implementation
106 — Database Migration with Flyway/Liquibase
```

The next database chapter is:

# 107 — RDS Backup / PITR Testing

We will move from:

```text
"Backups are enabled"
```

to actually testing:

```text
RDS
 ↓
Automated Backup
 ↓
PITR
 ↓
Restore
 ↓
New RDS Instance
 ↓
Application Connection
 ↓
Data Validation
 ↓
Recovery Time Measurement
```

including **snapshot restore, point-in-time restore, accidental data deletion, recovery validation, RPO/RTO measurement, Terraform implications, Secrets Manager updates, DNS/endpoint changes, and a complete production recovery drill**.
