# 109-RDS-Failover-Testing.md

````markdown
# 109 — RDS Multi-AZ Failover Testing

## 1. Purpose

This chapter implements and tests Amazon RDS PostgreSQL Multi-AZ failover.

The goal is to understand what actually happens when the RDS primary database becomes unavailable and AWS promotes the standby.

We will test:

- RDS Multi-AZ architecture
- controlled failover
- application connection behavior
- DNS behavior
- HikariCP behavior
- transaction failures
- retry behavior
- Kubernetes readiness
- application recovery
- monitoring
- measured recovery time
- production failover runbook

The target architecture is:

```text
                    EKS
                     |
                     |
                     v
             RDS PostgreSQL
                     |
              +------+------+
              |             |
              v             v
             AZ-A          AZ-B
           Primary        Standby
````

The application uses the normal RDS endpoint.

It should NOT connect directly to the individual DB instance endpoint.

---

# 2. Multi-AZ Is High Availability

Multi-AZ provides a standby database in another Availability Zone.

Conceptually:

```text
                  RDS Endpoint
                       |
                       v
                Current Primary
                       |
                synchronous/
                service-managed
                replication
                       |
                       v
                    Standby
```

The exact implementation details depend on the RDS deployment configuration and AWS service behavior.

The important application principle is:

```text
Application
    |
    v
RDS endpoint
```

not:

```text
Application
    |
    v
Specific EC2-style database host
```

---

# 3. Why We Test Failover

A configuration saying:

```text
multi_az = true
```

does not prove that the application recovers correctly.

The real system is:

```text
RDS fails over
       |
       v
Endpoint resolves to new primary
       |
       v
Existing connections may fail
       |
       v
Application must reconnect
       |
       v
Requests recover
```

Every layer must be tested.

---

# 4. Production Architecture

Our production flow:

```text
Internet
   |
   v
Route 53
   |
   v
ALB
   |
   v
EKS
   |
   v
order-service
   |
   v
RDS endpoint
   |
   +-------------------+
   |                   |
   v                   v
AZ-A                AZ-B
Primary             Standby
```

The RDS endpoint remains the application's logical database destination.

---

# 5. Verify Multi-AZ

Before testing, verify the configuration.

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].MultiAZ'
```

Expected:

```text
true
```

Also inspect:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].DBInstanceStatus'
```

Expected:

```text
available
```

---

# 6. Verify the RDS Endpoint

Run:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].Endpoint.Address'
```

Example:

```text
ecommerce-prod-postgres.xxxxx.ap-south-1.rds.amazonaws.com
```

The application configuration should reference this endpoint.

---

# 7. Application Configuration

Our Spring Boot application uses:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Example environment value:

```text
DB_URL=jdbc:postgresql://ecommerce-prod-postgres.xxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

The endpoint is obtained from the approved configuration/secret mechanism.

Do not hardcode credentials into Git.

---

# 8. What Happens During Failover?

Conceptually:

```text
Normal:

Application
    |
    v
Primary
    |
    v
Standby


Failure:

Application
    |
    v
Primary unavailable
    |
    v
AWS detects failure
    |
    v
Standby promoted
    |
    v
RDS endpoint points to new primary
    |
    v
Application reconnects
```

There is normally a period where connections or requests can fail.

That period must be measured.

---

# 9. Existing Connections

An important concept:

The application may have existing TCP/database connections to the old primary.

During failover:

```text
Old connection
      |
      X
Primary changed
```

Existing connections may fail.

The application must establish new connections to the current primary.

This is why connection pool behavior matters.

---

# 10. HikariCP During Failover

Suppose:

```text
Hikari pool:

Connection 1
Connection 2
Connection 3
...
Connection 15
```

During failover:

```text
Existing connections
        ↓
Some become invalid
        ↓
Queries fail/retry according to application behavior
        ↓
Pool creates/replaces connections
        ↓
New primary
        ↓
Normal operation
```

The exact timing depends on:

* RDS failover
* DNS
* network behavior
* JDBC driver
* connection pool configuration
* application retry behavior

---

# 11. Important: DNS Is Involved

The application uses the RDS DNS endpoint.

Example:

```text
ecommerce-prod-postgres.xxxxx.rds.amazonaws.com
```

The endpoint resolves to the current database infrastructure.

During failover, AWS updates the endpoint's underlying destination.

Therefore:

```text
RDS endpoint
```

should be treated as the stable application endpoint.

Do not build application logic around a specific RDS host/IP.

---

# 12. Failover Test Safety

This is a controlled production-impacting operation.

Do not run:

```bash
aws rds reboot-db-instance ... --force-failover
```

against production casually.

A failover test should have:

```text
Change ticket
Test window
Incident/operations owner
Application owner
Database owner
Rollback/abort procedure
Monitoring
Business approval
Communication plan
```

For initial testing, use a non-production environment.

---

# 13. Test Environment

Recommended progression:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
Production
```

Production should only be tested after the lower environments demonstrate expected recovery behavior.

---

# 14. Pre-Test Checklist

Before failover:

```text
[ ] RDS status = available
[ ] Multi-AZ enabled
[ ] Application healthy
[ ] Pods healthy
[ ] HPA stable
[ ] No active deployment
[ ] No active migration
[ ] No database maintenance
[ ] Monitoring working
[ ] Alerts working
[ ] Logs available
[ ] Incident/change ticket created
[ ] Stakeholders notified
```

---

# 15. Record Baseline

Record:

```text
RDS endpoint
RDS instance identifier
Current AZ
Database engine/version
Pod count
Hikari pool size
Current connections
Application request rate
Error rate
P95 latency
P99 latency
RDS CPU
RDS memory
```

Example:

```text
Pods:              5
Pool size:         15
RDS connections:   72
Request rate:      120 req/s
P95:               180 ms
Error rate:        0.1%
```

---

# 16. Baseline Database Connection

Run:

```sql
SELECT
    now(),
    count(*)
FROM pg_stat_activity;
```

Record the result.

Also record:

```sql
SELECT
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY count(*) DESC;
```

---

# 17. Baseline Application Test

Run a controlled request:

```bash
curl -f \
  https://api.example.com/api/v1/orders
```

Or test a known test endpoint.

Record:

```text
HTTP status
Latency
Response
Timestamp
```

---

# 18. Start Continuous Traffic

For a controlled test environment, generate a small predictable request rate.

Example:

```text
10 requests/sec
```

The purpose is to observe:

```text
Before failover
During failover
After failover
```

Do not generate unnecessary production load during a failover test.

---

# 19. Trigger Controlled Failover

AWS provides a controlled reboot/failover operation for Multi-AZ DB instances.

Example:

```bash
aws rds reboot-db-instance \
  --db-instance-identifier ecommerce-prod-postgres \
  --force-failover
```

This operation is disruptive.

Use it only during an approved test.

The exact AWS behavior and CLI options should be verified against the currently installed AWS CLI/service behavior before execution.

---

# 20. Immediately Start Timing

Record:

```text
Failover started:
2026-09-28 11:00:00 IST
```

Then monitor:

```text
RDS status
Application errors
Database connections
HTTP latency
Pod readiness
Hikari metrics
```

---

# 21. Monitor RDS Status

Run:

```bash
watch -n 5 \
'aws rds describe-db-instances \
 --db-instance-identifier ecommerce-prod-postgres \
 --query "DBInstances[0].DBInstanceStatus"'
```

Possible transitional states can include:

```text
rebooting
available
```

The exact status progression can vary.

---

# 22. Monitor Application Pods

```bash
watch -n 5 \
'kubectl get pods -n ecommerce-prod'
```

You may see:

```text
Running
Ready
```

The Pods may remain running while database requests temporarily fail.

This is important.

A Kubernetes Pod being:

```text
Running
```

does NOT mean the application dependency is healthy.

---

# 23. Check Application Logs

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200 \
  -f
```

During failover, you may see errors such as:

```text
connection reset
connection refused
I/O error
SQL exception
connection is closed
timeout
```

The exact error depends on the JDBC driver and failure timing.

---

# 24. Check Hikari Metrics

Observe:

```text
Active
Idle
Pending
Maximum
```

Expected behavior may resemble:

```text
Before:

Active = 4
Idle   = 11
Pending = 0

During failover:

Active = changes
Idle   = decreases
Pending = increases

After recovery:

Active = normal
Idle   = normal
Pending = 0
```

Actual behavior must be measured.

---

# 25. Monitor HTTP Errors

Observe:

```text
HTTP 2xx
HTTP 4xx
HTTP 5xx
Latency
```

The most important question is:

> How much application traffic was affected during the database failover?

---

# 26. Test a Database Operation

During the failover window, requests may fail.

After recovery:

```bash
curl -X POST \
  https://api.example.com/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "customer-001",
    "productId": "product-001",
    "quantity": 1
  }'
```

Then verify the order.

---

# 27. Important Transaction Behavior

Suppose:

```text
Transaction starts
      ↓
SQL statement 1
      ↓
RDS failover
      ↓
Connection breaks
```

That transaction may fail.

The application must not assume:

> "The database failed over, therefore the transaction automatically continued."

The application should treat the transaction as failed and determine whether it is safe to retry.

---

# 28. Retry Is Not Always Safe

Consider:

```text
POST /orders
```

The application sends:

```text
Create order
```

The database may commit the transaction immediately before the network connection breaks.

The client may receive:

```text
timeout
```

Now the client doesn't know whether the order was created.

Blind retry could create a duplicate.

Therefore retries must consider:

```text
Idempotency
Transaction semantics
Business operation
Request identity
```

---

# 29. Idempotency

For order creation, use an idempotency key where appropriate.

Example:

```http
Idempotency-Key: 6f7c2d3e-...
```

Conceptually:

```text
Request 1
   ↓
Create order
   ↓
DB commits
   ↓
Network fails
   ↓
Client retries
   ↓
Same idempotency key
   ↓
Return existing result
```

This is especially important during transient infrastructure failures.

---

# 30. JDBC Driver Considerations

The PostgreSQL JDBC driver handles database connections, but failover behavior still depends on:

```text
Driver configuration
DNS behavior
Connection pool
Network
Application retry logic
```

Do not assume a generic:

```text
retry=true
```

setting will solve database failover.

Test the actual driver and version used by the service.

---

# 31. DNS Cache Considerations

Java applications and the underlying JVM/network stack can cache DNS results.

Therefore failover testing should verify:

```text
DNS resolution
Connection establishment
New connection behavior
```

Do not build a failover design based on a hardcoded IP address.

---

# 32. Verify Endpoint After Failover

Run:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].Endpoint.Address'
```

The endpoint name should remain the application endpoint.

You can also inspect DNS resolution from an environment with network access:

```bash
getent hosts ecommerce-prod-postgres.xxxxx.ap-south-1.rds.amazonaws.com
```

or:

```bash
nslookup ecommerce-prod-postgres.xxxxx.ap-south-1.rds.amazonaws.com
```

---

# 33. Identify New Primary AZ

Check:

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres \
  --query 'DBInstances[0].AvailabilityZone'
```

Compare:

```text
Before:
ap-south-1a

After:
ap-south-1b
```

The exact AZs will depend on the deployment.

---

# 34. Failover Completion

Define a clear recovery condition.

For example:

```text
RDS available
+
Application DB connections successful
+
Hikari pending = 0
+
HTTP error rate normal
+
Database queries successful
+
Business smoke test successful
```

Do not declare recovery solely because:

```text
RDS status = available
```

---

# 35. Measure Failover Duration

Example:

```text
Failover started:
11:00:00

RDS available:
11:01:12

Application DB connection recovered:
11:01:20

HTTP errors returned to baseline:
11:01:28

Business smoke test passed:
11:01:35
```

Measured application recovery:

```text
~95 seconds
```

This is a test result, not a universal AWS guarantee.

---

# 36. Measure Application Impact

Record:

```text
Total requests during test
Successful requests
Failed requests
5xx requests
P95 latency
P99 latency
Duration of elevated errors
```

Example:

```text
Requests: 10,000
5xx: 35
Error rate: 0.35%
Recovery: 95 seconds
```

The actual values should come from the test.

---

# 37. RTO for Database Failover

Our target may be:

```text
RTO < 2 minutes
```

After testing:

```text
Measured:
95 seconds
```

This provides evidence about the tested architecture.

Do not advertise an RTO until repeated testing confirms it under realistic conditions.

---

# 38. Connection Recovery Test

After failover, check:

```sql
SELECT
    client_addr,
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY client_addr, application_name, state
ORDER BY count(*) DESC;
```

Confirm that new application connections are being established.

---

# 39. Hikari Recovery

Check the application's metrics.

Expected:

```text
Pending connections
       ↓
increase during disruption
       ↓
return toward baseline
```

If pending remains high:

```text
Investigate:
- database availability
- network
- connection pool
- credentials
- DNS
- application retries
```

---

# 40. Application Readiness

Our readiness endpoint:

```text
/actuator/health/readiness
```

should reflect the application's intended readiness semantics.

Be careful about making readiness depend directly on every external dependency.

For example:

```text
Pod
  |
  +-- readiness
  |
  +-- RDS
```

If readiness is too tightly coupled to a transient dependency, all Pods may be removed from Service endpoints during a short database event.

The health-check design should reflect the application's actual failure behavior.

---

# 41. Liveness vs Readiness

### Liveness

Answers:

> Is the application process alive?

### Readiness

Answers:

> Should Kubernetes send traffic to this Pod?

They are different.

A database failover should not automatically cause Kubernetes to restart every Pod.

Avoid using liveness probes that kill healthy application processes simply because a temporary dependency is unavailable.

---

# 42. Graceful Recovery

A resilient application should behave like:

```text
RDS failover
    ↓
Temporary DB errors
    ↓
Requests fail/retry appropriately
    ↓
RDS becomes available
    ↓
Connection pool reconnects
    ↓
Application resumes
```

Not:

```text
RDS failover
    ↓
Every Pod crashes
    ↓
Kubernetes restarts everything
    ↓
Traffic spike
    ↓
Database overloaded
```

---

# 43. Retry Storm

A dangerous design:

```text
Database unavailable
       ↓
100 requests fail
       ↓
Each retries 10 times immediately
       ↓
1000 requests
       ↓
Database recovers
       ↓
Huge connection/request spike
```

This is a retry storm.

Use:

```text
Exponential backoff
Jitter
Retry limits
Circuit breakers where appropriate
Idempotency
Timeouts
```

---

# 44. Example Retry Strategy

Conceptually:

```text
Attempt 1
   ↓
wait 100ms

Attempt 2
   ↓
wait 250ms

Attempt 3
   ↓
wait 500ms

Stop
```

Actual values should be determined by workload.

Do not retry indefinitely.

---

# 45. Circuit Breaker

For downstream database-dependent operations, application-level resilience patterns may include circuit breakers.

Conceptually:

```text
Normal
  ↓
Closed

Failures increase
  ↓
Open

Wait
  ↓
Half-open

Successful
  ↓
Closed
```

Database failure handling should be designed carefully because database operations have transaction and consistency semantics.

---

# 46. What We Should NOT Do

Avoid:

```text
while(true) retry
```

Avoid:

```text
sleep 1 second
retry forever
```

Avoid:

```text
restart all Pods when DB fails
```

Avoid:

```text
hardcode RDS IP
```

Avoid:

```text
increase connection pool blindly
```

---

# 47. Monitoring During Failover

Our Grafana dashboard should show:

```text
APPLICATION
────────────────────────────
Request Rate
Error Rate
P95
P99

DATABASE
────────────────────────────
Connections
CPU
Latency
IOPS

HIKARI
────────────────────────────
Active
Idle
Pending
Maximum

KUBERNETES
────────────────────────────
Pod count
Ready pods
Restarts
HPA replicas
```

---

# 48. Cloud Monitoring

Also observe AWS RDS metrics/events.

Examples include:

```text
Database connections
CPU utilization
Freeable memory
Read latency
Write latency
IOPS
Storage
RDS events
```

RDS events are particularly useful during failover tests.

---

# 49. RDS Events

Use:

```bash
aws rds describe-events \
  --source-type db-instance \
  --source-identifier ecommerce-prod-postgres
```

Review events around the failover timestamp.

This provides AWS-side evidence for the test timeline.

---

# 50. CloudTrail

RDS API operations can also be investigated through CloudTrail.

For example:

```text
Who initiated the failover?
When?
From which identity?
Which API operation?
```

This is useful for auditability.

---

# 51. Incident Timeline

Create:

```text
RDS Failover Test
=================

10:55
Baseline captured

11:00
Failover initiated

11:00:05
Database connections begin failing

11:01:12
RDS available

11:01:20
Application reconnects

11:01:28
HTTP errors return to baseline

11:01:35
Business smoke test passes
```

This becomes evidence for the recovery test.

---

# 52. Test Scenarios

Do not perform only one failover test.

Test:

```text
Scenario 1:
Normal application traffic

Scenario 2:
High application traffic

Scenario 3:
HPA-scaled application

Scenario 4:
Long-running query

Scenario 5:
Deployment during failover

Scenario 6:
Message-consuming application

Scenario 7:
Multiple microservices using databases
```

Some scenarios should be simulated separately rather than combined into one risky test.

---

# 53. Failover During Deployment

This is especially important.

Bad combination:

```text
Database failover
+
Production deployment
+
Schema migration
```

at the same time.

Avoid combining unrelated changes during a failover test.

Instead test independently:

```text
Failover
```

then:

```text
Deployment
```

then:

```text
Migration compatibility
```

---

# 54. Failover During Migration

A migration could be running:

```text
ALTER TABLE ...
```

while RDS fails over.

The transaction may fail.

Therefore database migrations need:

```text
Retry/recovery procedure
Migration status verification
Forward-fix strategy
```

This is one reason our migration chapter uses controlled migration Jobs and backward-compatible changes.

---

# 55. Message Processing During Failover

Suppose:

```text
Order Service
   ↓
Database
   ↓
Commit
   ↓
Publish event
```

If the database fails between steps, consistency becomes important.

Later, with SQS/SNS/Kafka, we will design:

```text
Database
+
Messaging
+
Idempotency
+
Retry
+
Outbox/event patterns where required
```

This prevents database failover from producing inconsistent business events.

---

# 56. Failover and Kubernetes

The important point:

```text
RDS failover
```

does not mean:

```text
EKS failure
```

The Pods may continue running.

The dependency changes underneath them.

Architecture:

```text
             EKS
              |
        order-service
              |
              X
         temporary DB
         connection loss
              |
              v
        new RDS primary
              |
              v
         connections recover
```

---

# 57. Failure Injection

For learning, we can test:

```text
RDS Multi-AZ failover
```

without destroying the database.

The purpose is to validate application resilience.

Never simulate failure by deleting production data or infrastructure unless the test explicitly requires it and has appropriate controls.

---

# 58. Post-Failover Validation

After failover:

```text
[ ] RDS available
[ ] New primary confirmed
[ ] Endpoint resolves
[ ] New DB connections successful
[ ] Hikari pending returned to baseline
[ ] HTTP errors returned to baseline
[ ] Latency returned to baseline
[ ] Application health normal
[ ] Database queries successful
[ ] Orders can be created
[ ] Orders can be read
[ ] Messaging works
[ ] Monitoring normal
```

---

# 59. Data Integrity Validation

Run business validation.

Example:

```sql
SELECT COUNT(*)
FROM orders;
```

Then validate a known order:

```sql
SELECT *
FROM orders
WHERE id = 10001;
```

Also verify:

```text
No unexpected duplicate
No missing committed record
No application corruption
No failed migration state
```

---

# 60. Application Recovery Test

Test:

```text
POST /orders
GET /orders/{id}
PUT /orders/{id}
```

Then validate database state.

The goal is to prove:

```text
Infrastructure recovered
+
Application recovered
+
Business operation recovered
```

---

# 61. Failover Test Report

Example:

```text
RDS Multi-AZ Failover Test
==========================

Environment:
STAGING

Database:
ecommerce-staging-postgres

Region:
ap-south-1

Before AZ:
ap-south-1a

After AZ:
ap-south-1b

Failover initiated:
11:00:00

RDS available:
11:01:12

Application recovery:
11:01:20

Business validation:
11:01:35

Observed errors:
35 requests

Measured recovery:
95 seconds

Result:
Successful

Follow-up:
Review retry behavior
```

Use actual observed values in the real test.

---

# 62. Production Runbook

## Step 1 — Confirm Incident

Determine:

```text
Is database unavailable?
Is this an application problem?
Is this a network problem?
Is this an RDS event?
```

---

## Step 2 — Check RDS

```bash
aws rds describe-db-instances \
  --db-instance-identifier ecommerce-prod-postgres
```

---

## Step 3 — Check RDS Events

```bash
aws rds describe-events \
  --source-type db-instance \
  --source-identifier ecommerce-prod-postgres
```

---

## Step 4 — Check Application

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200
```

---

## Step 5 — Check Connection Pool

Look at:

```text
Hikari active
Hikari idle
Hikari pending
```

---

## Step 6 — Check Database Recovery

Verify:

```text
RDS available
Endpoint working
New connections succeed
```

---

## Step 7 — Business Validation

Test:

```text
Create order
Read order
Payment flow
Inventory flow
```

---

## Step 8 — Continue Monitoring

Watch:

```text
Error rate
Latency
Connections
CPU
Memory
Queue backlog
```

---

# 63. Production Alert

An alert could conceptually be:

```text
RDS failover event
       ↓
Alert
       ↓
On-call engineer
       ↓
Check application error rate
       ↓
Check DB connections
       ↓
Verify recovery
```

Not every RDS event needs an incident.

The severity depends on:

```text
Customer impact
Duration
Data impact
Application recovery
```

---

# 64. Common Failure Modes

## Failure 1

Application never reconnects.

Investigate:

```text
JDBC driver
Hikari configuration
DNS
Network
Credentials
Connection lifecycle
```

---

## Failure 2

Application reconnects but latency remains high.

Investigate:

```text
RDS CPU
Slow queries
Connection pressure
Locks
Cache behavior
```

---

## Failure 3

Massive 5xx spike after recovery.

Investigate:

```text
Retry storm
Connection pool
HPA
Load
Application exceptions
```

---

## Failure 4

Duplicate orders.

Investigate:

```text
Retry logic
Idempotency
Transaction outcome uncertainty
Client retries
```

---

## Failure 5

Pods restart unnecessarily.

Investigate:

```text
Liveness probe
Application exception handling
Container restart policy
Dependency health-check design
```

---

# 65. Production Design Principles

Our final database resilience model:

```text
Multi-AZ
   +
Connection pooling
   +
Retry with backoff
   +
Idempotency
   +
Health checks
   +
Monitoring
   +
PITR
   +
Failover testing
```

---

# 66. Relationship With Previous Chapters

We now have:

```text
105
RDS Production
       ↓
106
Database Migration
       ↓
107
Backup/PITR
       ↓
108
Performance/Connection Pooling
       ↓
109
Multi-AZ Failover
```

Together:

```text
RDS
 ├── HA
 ├── Backup
 ├── PITR
 ├── Performance
 ├── Connection Management
 └── Failover
```

---

# 67. Production Checklist

## Infrastructure

```text
[ ] Multi-AZ enabled
[ ] Private subnet
[ ] Correct DB subnet group
[ ] Security Group restricted
[ ] Encryption enabled
[ ] Backup enabled
```

## Application

```text
[ ] RDS endpoint used
[ ] No hardcoded IP
[ ] Hikari configured
[ ] Connection timeout configured
[ ] Retry strategy defined
[ ] Idempotency implemented where required
```

## Kubernetes

```text
[ ] Readiness probe reviewed
[ ] Liveness probe reviewed
[ ] HPA reviewed
[ ] Rolling update capacity reviewed
[ ] PodDisruptionBudget reviewed
```

## Observability

```text
[ ] RDS metrics
[ ] Hikari metrics
[ ] HTTP metrics
[ ] Logs
[ ] RDS events
[ ] Alerts
```

## Testing

```text
[ ] DEV failover tested
[ ] QA failover tested
[ ] STAGING failover tested
[ ] Production test approved
[ ] RTO measured
[ ] Application impact measured
[ ] Business validation completed
```

---

# 68. Final Architecture

```text
                        Internet
                            |
                            v
                           ALB
                            |
                            v
                     ┌──────────────┐
                     │     EKS      │
                     │              │
                     │ order-service│
                     │              │
                     │  HikariCP    │
                     └──────┬───────┘
                            |
                            |
                     RDS Endpoint
                            |
               ┌────────────┴────────────┐
               |                         |
               v                         v
          AZ-A Primary              AZ-B Standby
               |                         |
               └───────────┬─────────────┘
                           |
                     Multi-AZ HA
                           |
                     Backup / PITR
```

---

# 69. Key Takeaways

Remember these production principles:

```text
1. Multi-AZ provides high availability.
2. Existing database connections may fail during failover.
3. Applications must establish new connections.
4. RDS endpoint should be used instead of a hardcoded IP.
5. Connection pools must be capacity-planned.
6. Retries must use limits and backoff.
7. Database writes need idempotency where retries can duplicate operations.
8. Kubernetes should not blindly restart every Pod during a DB failover.
9. RTO must be measured, not assumed.
10. Failover must be tested before relying on it.
```

The most important chain is:

```text
RDS Failover
     ↓
Existing connections fail
     ↓
Hikari detects failures
     ↓
Application establishes new connections
     ↓
Requests recover
     ↓
Business validation
```

---

# 70. Next Chapter

**110-Redis-Production-Implementation.md**

We will now move from PostgreSQL to the caching layer:

```text
Application
     |
     +------------> RDS PostgreSQL
     |
     +------------> Redis / ElastiCache
```

We will implement:

* Redis architecture
* ElastiCache
* primary/replica
* Multi-AZ
* cache-aside
* TTL
* cache invalidation
* Spring Boot Redis integration
* EKS networking
* Redis security
* connection pooling
* Redis metrics
* cache hit/miss monitoring
* failure behavior
* HPA interaction
* cache stampede prevention
* production troubleshooting

```
```
