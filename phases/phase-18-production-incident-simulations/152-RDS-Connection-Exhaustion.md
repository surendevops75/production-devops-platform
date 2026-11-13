# 152. RDS Connection Exhaustion — Production Incident

> **Phase 18 — Incident Management**  
> **Incident #152**  
> **Environment:** Production  
> **Database:** Amazon RDS PostgreSQL  
> **Application:** 21-microservice E-Commerce Platform

---

## 1. Incident Overview

RDS connection exhaustion occurs when the application workloads collectively consume most or all PostgreSQL connection slots.

This is different from:

- CPU exhaustion
- Memory exhaustion
- Disk exhaustion
- Network failure

The application may have healthy CPU and memory while requests are failing because **no database connection is available**.

Typical application errors:

```text
HikariPool-1 - Connection is not available, request timed out after 3000ms
```

or:

```text
org.postgresql.util.PSQLException:
FATAL: remaining connection slots are reserved for non-replication superuser connections
```

or:

```text
FATAL: too many clients already
```

Typical customer symptoms:

```text
HTTP 500
HTTP 503
HTTP 504
High API latency
Order creation failures
Login failures
Cart failures
Payment failures
```

---

# 2. Production Architecture

Our application contains 21 microservices.

The database connection path is:

```text
                    Internet
                       |
                    Route 53
                       |
                  CloudFront/WAF
                       |
                      ALB
                       |
                 EKS Services
                       |
              +--------+--------+
              |        |        |
           Pod       Pod       Pod
              |        |        |
           Hikari   Hikari    Hikari
            Pool     Pool      Pool
              |        |        |
              +--------+--------+
                       |
                  RDS Proxy
                  (optional)
                       |
                RDS PostgreSQL
```

Without RDS Proxy:

```text
Pod
 |
Hikari
 |
PostgreSQL connection
 |
RDS
```

With RDS Proxy:

```text
Pod
 |
Hikari
 |
RDS Proxy
 |
PostgreSQL
```

---

# 3. Most Important Concept

## Connection pools belong to Pods

Suppose:

```text
order-service
Replicas = 10

Hikari maximumPoolSize = 10
```

Potential connection capacity:

```text
10 Pods × 10 connections
=
100 connections
```

Now HPA scales to:

```text
30 Pods
```

Then:

```text
30 × 10
=
300 connections
```

The application didn't change.

The number of database connections changed because **the number of Pods changed**.

---

# 4. The 21-Service Problem

Our platform has 21 services.

Each service can have:

```text
Pods × connection pool
```

Therefore total possible application connections are approximately:

```text
Total DB connections
≈
Σ(Pods per service × Pool size per Pod)
+
Other clients
```

Other clients include:

- Admin connections
- Migration jobs
- Monitoring tools
- Reporting jobs
- ETL jobs
- BI tools
- Batch workloads
- RDS Proxy connections if applicable

---

# 5. Example Connection Explosion

Suppose:

| Service | Pods | Pool/Pod | Potential connections |
|---|---:|---:|---:|
| auth | 10 | 10 | 100 |
| customer | 10 | 10 | 100 |
| catalog | 15 | 10 | 150 |
| product | 15 | 10 | 150 |
| cart | 10 | 10 | 100 |
| order | 20 | 10 | 200 |
| payment | 10 | 10 | 100 |
| inventory | 15 | 10 | 150 |

Already:

```text
1,050 potential connections
```

And we haven't even counted all 21 services.

This is why blindly configuring:

```yaml
maximum-pool-size: 50
```

on every microservice is dangerous.

---

# 6. RDS `max_connections`

PostgreSQL has a limit on concurrent connections.

Check it:

```sql
SHOW max_connections;
```

or:

```sql
SELECT name, setting
FROM pg_settings
WHERE name = 'max_connections';
```

But **do not simply increase `max_connections` whenever connections are exhausted**.

Increasing the connection limit can increase:

- Memory consumption
- CPU overhead
- Context switching
- Lock contention
- Query concurrency
- Database instability

The correct question is:

> Why are we consuming so many connections?

---

# 7. Incident Scenario

Assume production normally has:

```text
order-service:
8 Pods

Hikari:
maximumPoolSize = 20
```

Potential capacity:

```text
8 × 20 = 160
```

During a traffic spike:

```text
8 Pods
      ↓
15 Pods
      ↓
30 Pods
```

Now:

```text
30 × 20
=
600 potential connections
```

Suppose RDS can safely support fewer than that after reserving connections for other workloads.

Suddenly:

```text
DatabaseConnections ↑
Hikari pending ↑
API latency ↑
HTTP 5xx ↑
```

The incident begins.

---

# 8. Alert

Example alert:

```text
CRITICAL: RDS connection utilization is above safe threshold.

DatabaseConnections: 480
Safe connection budget: 450
RDS max_connections: 500
Environment: production
```

At the same time Grafana shows:

```text
Order API P95 latency: ↑
Order API 5xx: ↑
Hikari active connections: ↑
Hikari pending threads: ↑
HPA replicas: ↑
RDS connections: ↑
```

This correlation is extremely important.

---

# 9. Production Incident Flow

Use:

```text
Alert
  ↓
Confirm customer impact
  ↓
Check RDS connections
  ↓
Check Hikari pools
  ↓
Check pg_stat_activity
  ↓
Check active / idle / idle-in-transaction
  ↓
Check HPA
  ↓
Check recent deployment
  ↓
Check long-running queries
  ↓
Check connection leaks
  ↓
Protect RDS
  ↓
Reduce connection pressure
  ↓
Restore application traffic
  ↓
Find root cause
  ↓
Fix permanently
  ↓
Validate
  ↓
RCA
```

---

# 10. Step 1 — Confirm Customer Impact

Check application health:

```bash
kubectl get pods -n ecommerce
```

Check services:

```bash
kubectl get svc -n ecommerce
```

Check recent events:

```bash
kubectl get events -n ecommerce --sort-by=.lastTimestamp
```

Check API Pods:

```bash
kubectl get pods -n ecommerce \
  -l app=order-service
```

Check logs:

```bash
kubectl logs -n ecommerce \
  -l app=order-service \
  --tail=200
```

Look for:

```text
Connection is not available
request timed out
too many clients
remaining connection slots
PSQLException
HikariPool
```

---

# 11. Step 2 — Check RDS Connections

From the PostgreSQL database, run:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Then:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

Example:

```text
active                  120
idle                    340
idle in transaction      25
```

This tells us much more than simply saying:

```text
340 connections
```

---

# 12. Active vs Idle Connections

## Active

```text
state = active
```

The connection is currently executing a query.

Possible causes:

- Heavy traffic
- Slow SQL
- Missing indexes
- Long-running transactions
- Lock contention

---

## Idle

```text
state = idle
```

The connection is currently not executing a query.

Idle connections aren't automatically bad.

Connection pools intentionally keep connections available.

The question is:

> Is the idle connection count consistent with our expected pool configuration?

---

# 13. Idle in Transaction

This is much more concerning:

```text
idle in transaction
```

Example:

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    xact_start,
    query_start,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

A long-running transaction can:

- Hold locks
- Prevent cleanup
- Increase database pressure
- Keep a connection occupied
- Cause application requests to queue

---

# 14. Find Which Applications Consume Connections

Run:

```sql
SELECT
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY
    usename,
    application_name,
    client_addr,
    state
ORDER BY connections DESC;
```

Example:

```text
order-service       280
catalog-service     120
customer-service     80
payment-service      60
admin                 15
migration              5
```

Now we know where the connection pressure is coming from.

---

# 15. Find Long-Running Sessions

Use a read-only diagnostic query:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    xact_start,
    query_start,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Be careful with displaying SQL text because queries may contain:

- PII
- customer information
- tokens
- sensitive business data

---

# 16. Find Long-Running Queries

Example:

```sql
SELECT
    pid,
    application_name,
    state,
    now() - query_start AS query_duration,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE query_start IS NOT NULL
ORDER BY query_duration DESC;
```

This helps determine whether connection exhaustion is actually caused by slow database work.

---

# 17. Check Hikari Metrics

Spring Boot applications expose Hikari metrics through Micrometer.

Useful metrics include:

```text
hikaricp_connections
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_max
hikaricp_connections_min
```

Depending on the Micrometer version/export configuration, exact metric names may differ.

The important signals are:

```text
Active
Idle
Pending
Maximum
```

---

# 18. Important Hikari Pattern

Suppose:

```text
maximumPoolSize = 20
```

and Grafana shows:

```text
Active = 20
Idle = 0
Pending = 45
```

That means:

```text
All 20 connections are busy
+
45 application threads are waiting
```

This is a major signal.

The application isn't necessarily CPU-bound.

It may be **database-bound**.

---

# 19. Hikari Connection Timeout

Example:

```text
HikariPool-1 -
Connection is not available,
request timed out after 3000ms
```

This means an application thread requested a connection but could not obtain one within the configured timeout.

For example:

```yaml
connection-timeout: 3000
```

means approximately:

```text
Wait up to 3 seconds
↓
No connection available
↓
Request fails
```

---

# 20. Spring Boot Hikari Configuration

Example:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 3000
      validation-timeout: 1000
      max-lifetime: 1500000
      idle-timeout: 600000
```

These numbers are **illustrative**.

Do not copy:

```text
maximumPoolSize = 10
```

into every production service automatically.

Pool size must be derived from:

```text
Database capacity
+
Query latency
+
Traffic
+
Pod count
+
Service criticality
+
Connection budget
```

and validated with load testing.

---

# 21. Connection Pool Per Pod

This is one of the most important Kubernetes concepts.

Suppose:

```text
Deployment:
order-service

Replicas:
10

Hikari maximumPoolSize:
10
```

Potential pool capacity:

```text
10 × 10
=
100
```

If HPA increases replicas:

```text
10 → 20
```

then:

```text
20 × 10
=
200
```

If:

```text
20 → 40
```

then:

```text
40 × 10
=
400
```

Autoscaling can therefore create a database incident.

---

# 22. HPA Can Amplify a Database Incident

This is a classic production failure pattern.

```text
Traffic ↑
   ↓
CPU ↑
   ↓
HPA scales Pods
   ↓
More Pods
   ↓
More Hikari pools
   ↓
More DB connections
   ↓
RDS saturation
   ↓
Queries slow
   ↓
Requests remain active longer
   ↓
Connections remain occupied
   ↓
Hikari pending ↑
   ↓
Latency ↑
   ↓
5xx ↑
```

The system can enter a positive feedback loop.

---

# 23. Deployment Surge Can Also Increase Connections

Kubernetes rolling deployments can temporarily create extra Pods.

For example:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0
```

Suppose:

```text
Current replicas = 20
```

During deployment:

```text
20 existing Pods
+
5 new Pods
```

Potentially:

```text
25 connection pools
```

If every Pod can use 10 DB connections:

```text
25 × 10
=
250
```

instead of:

```text
20 × 10
=
200
```

---

# 24. Blue/Green Deployment

Blue/green can be even more obvious:

```text
BLUE:
20 Pods

GREEN:
20 Pods
```

During the transition:

```text
40 Pods
```

If:

```text
Pool = 10
```

then:

```text
40 × 10
=
400 potential connections
```

This must be included in capacity planning.

---

# 25. Canary Deployments

Canary deployments also add connection consumers.

Example:

```text
Stable:
20 Pods

Canary:
5 Pods
```

Total:

```text
25 Pods
```

Therefore database capacity must account for:

- HPA
- max replicas
- maxSurge
- Canary
- Blue/green
- Maintenance jobs

---

# 26. Connection Budget

We should define a database connection budget.

For example:

```text
RDS safe connection budget = X
```

Not:

```text
max_connections = X
```

We reserve capacity for:

- PostgreSQL internal requirements
- Admin access
- Monitoring
- Migration
- Operational recovery
- Other workloads

Then distribute the remaining budget.

---

# 27. Service Connection Budget

Example:

```text
Database safe budget = 400

order-service:
maximum replicas = 20
pool = 10

20 × 10 = 200
```

That means order-service alone could consume:

```text
50% of the entire database budget
```

That may or may not be acceptable.

This needs architectural review.

---

# 28. HPA and Connection Budget

We can derive an upper bound:

```text
safe_max_replicas
≤
service_connection_budget / pool_size
```

Example:

```text
Service DB budget = 100
Pool size = 10
```

Then:

```text
safe max replicas ≤ 100 / 10

safe max replicas ≤ 10
```

Therefore:

```yaml
maxReplicas: 10
```

may be a database protection mechanism.

Again, the exact value must come from testing.

---

# 29. Check HPA During Incident

```bash
kubectl get hpa -n ecommerce
```

Example:

```text
NAME            REFERENCE                  TARGETS
order-service   Deployment/order-service   92%/70%
```

Inspect:

```bash
kubectl describe hpa order-service -n ecommerce
```

Check current replicas:

```bash
kubectl get deployment order-service -n ecommerce
```

---

# 30. Check Recent Deployment

```bash
kubectl rollout history \
  deployment/order-service \
  -n ecommerce
```

Check:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce
```

Also inspect ArgoCD.

Ask:

```text
Did pool size change?
Did query behavior change?
Did a new endpoint increase DB traffic?
Did replica limits change?
Did HPA thresholds change?
Did maxSurge change?
```

---

# 31. Connection Leak

Another possible root cause is a connection leak.

Conceptually:

```text
Application requests connection
       ↓
Connection used
       ↓
Connection should be returned
       ↓
BUG
       ↓
Connection remains occupied
```

Over time:

```text
Available connections ↓
```

Eventually:

```text
Pool exhausted
```

In normal Spring Data JPA usage, Spring manages most JDBC resource lifecycle.

But problems can still occur through:

- Manual JDBC usage
- Incorrect transaction handling
- Long-running transactions
- Streaming results
- Custom connection management
- External libraries
- Misconfigured transaction boundaries

---

# 32. Spring Transaction Problem

Consider:

```java
@Transactional
public void processOrder() {

    // database operation

    callExternalPaymentAPI();

    // another database operation
}
```

If the transaction remains open while waiting for an external API:

```text
DB connection
     ↓
Transaction begins
     ↓
External API call
     ↓
5 seconds
     ↓
Transaction still open
     ↓
Connection occupied
```

At high traffic, this can consume many connections.

Prefer keeping transaction boundaries as small as practical.

---

# 33. Slow Query Connection Exhaustion

Suppose:

```text
Normal query:
50 ms
```

Then a missing index causes:

```text
Query:
5 seconds
```

Requests now hold connections for much longer.

Even if request rate stays constant:

```text
Connection occupancy ↑
```

Eventually:

```text
Pool exhausted
```

Therefore connection exhaustion can actually be a **query performance problem**.

---

# 34. Check Database Locks

During investigation, also inspect waits and locks.

Useful PostgreSQL diagnostic views include:

```sql
pg_stat_activity
pg_locks
```

Example:

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

If many sessions are waiting on locks, increasing the connection pool may make the situation worse.

---

# 35. Retry Storm

Another common cause:

```text
RDS becomes slow
      ↓
Application request times out
      ↓
Client retries
      ↓
Application retries
      ↓
More database requests
      ↓
More connections
      ↓
RDS becomes slower
```

This is a retry storm.

During an incident:

```text
More retries ≠ more reliability
```

Retries must have:

- bounded attempts
- exponential backoff
- jitter
- timeout
- circuit breaker where appropriate

---

# 36. RDS Metrics

Check RDS metrics in CloudWatch/Performance Insights.

Important signals include:

```text
DatabaseConnections
CPUUtilization
FreeableMemory
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
Database load / DBLoad
```

Exact available metrics depend on the RDS engine and monitoring configuration.

Look for correlation:

```text
DatabaseConnections ↑
DBLoad ↑
Latency ↑
5xx ↑
```

---

# 37. DatabaseConnections vs CPU

Example:

```text
CPU = 40%
Connections = 490/500
```

The database may appear healthy from CPU alone.

But the application is effectively out of connection capacity.

Therefore:

> CPU is not a substitute for connection monitoring.

---

# 38. Performance Insights

If enabled, use Performance Insights to determine:

```text
Which SQL is consuming DB load?
Which application is generating load?
Are sessions waiting?
Are queries blocked?
```

This is particularly useful when:

```text
connections high
+
CPU moderate
```

because the problem may be:

- locks
- waits
- long queries
- connection behavior

rather than raw CPU.

---

# 39. RDS Proxy

For workloads where it is appropriate, consider:

```text
Application Pods
      |
Hikari
      |
RDS Proxy
      |
RDS PostgreSQL
```

RDS Proxy can help manage and multiplex database connections.

But:

> RDS Proxy is not a magic solution for bad queries or unlimited traffic.

You still need:

- correct pool sizes
- query optimization
- transaction discipline
- connection budgets
- application limits

---

# 40. Connection Multiplexing

Conceptually:

```text
100 application connections
        |
     RDS Proxy
        |
Fewer backend DB connections
        |
       RDS
```

The actual multiplexing benefit depends on transaction/session behavior.

Long-lived transactions, session state, prepared statements, and other behaviors can reduce multiplexing efficiency.

Therefore test your workload.

---

# 41. Read Replicas

A read replica can help with:

```text
Read scalability
```

For example:

```text
Primary
  ↑
Writes

Read Replica
  ↑
Read workloads
```

But a read replica does **not automatically solve writer connection exhaustion**.

If every application connection still goes to the primary:

```text
Primary connection pressure remains.
```

Connection management and read scaling are separate concerns.

---

# 42. Incident Mitigation

During a live production incident:

## Step 1 — Stop amplification

If HPA is rapidly increasing replicas:

```text
Don't blindly allow unlimited scaling.
```

Temporarily control:

```text
maxReplicas
```

or scale down noncritical workloads where safe.

---

## Step 2 — Stop bad deployments

If the issue started immediately after deployment:

```text
Freeze further deployment
```

Then use GitOps rollback.

---

## Step 3 — Reduce noncritical database workloads

Examples:

```text
reporting-service
batch jobs
analytics jobs
background workers
```

Critical customer flows get priority.

---

## Step 4 — Reduce retry pressure

Temporarily control:

```text
retry count
retry rate
consumer concurrency
batch concurrency
```

---

# 43. Do Not Blindly Kill Connections

A dangerous response is:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity;
```

Never blindly terminate production connections.

You may kill:

- customer transactions
- payment operations
- order transactions
- migrations
- important background jobs

Instead:

```text
Identify
→ Confirm
→ Coordinate
→ Terminate only clearly unsafe/stale sessions
```

Prefer DBA-approved, targeted termination.

---

# 44. Idle-in-Transaction Cleanup

If a session is clearly stuck:

```text
idle in transaction
```

and has exceeded the approved operational threshold, a DBA may terminate the specific backend after confirming business impact.

Example:

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    xact_start
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

Then investigate **why** it happened.

Terminating it only treats the symptom.

---

# 45. GitOps Rollback

If the root cause is a bad deployment:

```text
GitOps repository
      ↓
Revert bad commit
      ↓
ArgoCD detects desired-state change
      ↓
Deployment rolls back
      ↓
Pods reduce
      ↓
Connection pressure reduces
```

Avoid treating:

```bash
kubectl rollout undo
```

as the final GitOps solution.

ArgoCD may reconcile the deployment back to the bad desired state.

---

# 46. Example Incident

### Timeline

```text
10:00
Normal traffic

10:05
Traffic increases

10:07
HPA:
8 → 20 Pods

10:08
RDS connections:
180 → 350

10:09
Deployment begins

10:10
Temporary surge Pods created

10:11
RDS connections:
350 → 480

10:12
Hikari pending connections increase

10:13
Order API P95 increases

10:14
HTTP 5xx increases

10:15
Incident declared
```

---

# 47. Initial Diagnosis

Grafana:

```text
Order Pods:
8 → 20

Hikari:
active ↑
pending ↑

RDS:
connections ↑

API:
latency ↑
5xx ↑
```

This strongly suggests:

```text
Traffic
+
HPA scaling
+
connection pool size
```

rather than a network failure.

---

# 48. PostgreSQL Investigation

Run:

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

Result:

```text
active                  150
idle                    320
idle in transaction      20
```

Then:

```sql
SELECT
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY count(*) DESC;
```

Result:

```text
order-service       300
catalog-service     100
customer-service     50
others               40
```

Now order-service becomes the primary suspect.

---

# 49. Check Hikari

Grafana:

```text
order-service
-------------------------
Pods:       20
Max pool:   20
Active:     20
Idle:        0
Pending:    80
```

Potential pool capacity:

```text
20 × 20
=
400
```

This is already near the database's entire safe budget.

---

# 50. Root Cause

Suppose Git history shows:

```text
Previous:
maximumPoolSize = 10

New:
maximumPoolSize = 20
```

And HPA changed:

```text
maxReplicas:
20 → 30
```

The combination created a dangerous capacity multiplication.

The root cause becomes:

```text
Pool size increased
+
HPA max replicas increased
+
Deployment surge
+
Insufficient DB connection budget
```

---

# 51. Immediate Recovery

The incident commander decides:

```text
1. Freeze deployment
2. Stop further HPA amplification
3. Reduce order-service replicas safely
4. Reduce noncritical workloads
5. Reduce retry pressure
6. Restore RDS connection headroom
7. Verify order API
```

Then:

```text
RDS connections:
480 → 350
```

Hikari:

```text
Pending:
80 → 0
```

API:

```text
5xx:
high → normal
```

---

# 52. Permanent Fix

Do not simply say:

```text
Increase RDS max_connections
```

Instead:

### Application

- Reduce oversized pool
- Optimize slow queries
- Fix transaction boundaries
- Fix connection leaks
- Add timeouts
- Review retry behavior

### Kubernetes

- Recalculate HPA max replicas
- Review maxSurge
- Protect database-dependent workloads
- Separate batch workloads

### Database

- Review connection capacity
- Review query performance
- Review locks
- Review instance sizing
- Consider RDS Proxy where appropriate

### Observability

Add:

```text
RDS connections
Hikari active
Hikari idle
Hikari pending
Hikari acquisition time
HPA replicas
API latency
API 5xx
```

---

# 53. Connection Pool Design Rules

Follow these principles:

```text
Small
Measured
Tested
Bounded
Observable
```

Avoid:

```text
maximumPoolSize: 100
```

simply because:

> "More connections means more performance."

It doesn't.

At some point:

```text
More connections
       ↓
More concurrency
       ↓
More DB contention
       ↓
More latency
       ↓
Less stability
```

---

# 54. Connection Pool Formula

A useful starting point:

```text
Total potential connections
≈
Σ(
  max replicas × max pool size
)
+
operational connections
```

For example:

```text
order:
20 × 10 = 200

catalog:
10 × 5 = 50

customer:
10 × 5 = 50

payment:
10 × 5 = 50

other:
50

Total:
400
```

If the safe DB budget is:

```text
450
```

we have only:

```text
50
```

connections of remaining headroom.

That may not be enough for:

- rollout surge
- batch jobs
- unexpected scaling
- operational access

So capacity planning must include headroom.

---

# 55. Kubernetes Deployment Strategy

For database-heavy services, review:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

This can reduce sudden connection multiplication.

But don't blindly use these values.

Deployment speed, availability, Pod startup time and database capacity all matter.

---

# 56. Hikari Pool Sizing

A common mistake is:

```text
CPU cores = pool size
```

or:

```text
100 RPS = 100 DB connections
```

Neither is a universal rule.

Pool size depends on:

```text
Query latency
Concurrent DB work
Transaction duration
DB capacity
Request rate
Number of replicas
Read/write mix
```

Measure it through load testing.

---

# 57. Database Connection Budget Must Include All Workloads

Our architecture isn't only:

```text
21 microservices
```

We may also have:

```text
Flyway migration
Reporting jobs
Admin tools
Monitoring
Data exports
Batch processing
Debug sessions
RDS Proxy
Other applications
```

Therefore:

```text
Application pool capacity
≠
Entire DB capacity
```

---

# 58. Monitoring Dashboard

Create a dedicated:

```text
RDS Connection Health
```

dashboard.

Include:

### RDS

```text
DatabaseConnections
CPU
Memory
DB load
Read latency
Write latency
IOPS
```

### Hikari

```text
Active
Idle
Pending
Maximum
Connection acquisition time
```

### Kubernetes

```text
Pod count
HPA desired replicas
HPA current replicas
Deployment surge
Pod restarts
```

### Application

```text
RPS
P50
P95
P99
5xx
Timeouts
```

---

# 59. Recommended Alerts

Example:

```text
RDS connection count approaching safe budget
```

```text
Hikari pending connections > 0 for sustained period
```

```text
Hikari acquisition latency increasing
```

```text
Idle-in-transaction sessions above threshold
```

```text
Database connection errors increasing
```

```text
HPA replicas approaching DB-safe maximum
```

```text
API latency + DB connections increasing together
```

The final alert is especially useful because it connects:

```text
Customer impact
+
database saturation
```

---

# 60. Prometheus Investigation

Depending on your Micrometer/Prometheus configuration, query metrics for:

```text
Hikari active
Hikari idle
Hikari pending
```

For example, conceptually:

```promql
sum by (service) (
  hikaricp_connections_active
)
```

Pending:

```promql
sum by (service) (
  hikaricp_connections_pending
)
```

The exact metric labels/names must match the metrics actually exposed by your Spring Boot/Micrometer version.

---

# 61. Correlate With HPA

Compare:

```text
HPA replicas
```

against:

```text
Hikari active
```

and:

```text
RDS DatabaseConnections
```

A classic graph looks like:

```text
HPA replicas       ↑↑↑
Hikari active      ↑↑↑
RDS connections    ↑↑↑
API latency        ↑↑↑
5xx                ↑↑↑
```

This is strong evidence of scaling-induced database pressure.

---

# 62. ELK Investigation

Search application logs for:

```text
Connection is not available
```

or:

```text
remaining connection slots
```

or:

```text
too many clients
```

Use:

```text
service.name = order-service
```

and:

```text
environment = production
```

Then correlate:

```text
correlationId
traceId
```

to identify affected customer requests.

---

# 63. Jaeger Investigation

A trace may look like:

```text
POST /api/v1/orders
        |
        +-- customer-service
        |
        +-- product-service
        |
        +-- pricing-service
        |
        +-- order-service
                |
                +-- PostgreSQL
                       |
                       4.8 seconds
```

This helps determine whether database latency caused the request slowdown.

---

# 64. Important Distinction

Don't confuse:

```text
Connection exhaustion
```

with:

```text
Query performance problem
```

They can be related.

For example:

```text
Slow query
   ↓
Connection held longer
   ↓
Pool fills
   ↓
Connection exhaustion
```

So the real root cause may be:

```text
Missing database index
```

rather than:

```text
Hikari pool too small
```

---

# 65. Don't Increase Pool Size During an Incident

Suppose:

```text
maximumPoolSize = 10
```

and connections are exhausted.

A developer might say:

> Increase it to 30.

That could make the incident much worse:

```text
20 Pods × 10 = 200
```

becomes:

```text
20 Pods × 30 = 600
```

The database may collapse further.

---

# 66. Don't Increase HPA During DB Saturation

Similarly:

```text
CPU high
```

doesn't automatically mean:

```text
increase maxReplicas
```

If the actual bottleneck is RDS:

```text
More Pods
→ More DB connections
→ More contention
```

Scaling the application tier can amplify the database incident.

---

# 67. Incident Decision Tree

```text
API 5xx / latency
        |
        v
Hikari pending > 0?
        |
       YES
        |
        v
RDS DatabaseConnections high?
        |
       YES
        |
        v
Check pg_stat_activity
        |
        +----------------------+
        |          |           |
      active      idle    idle in transaction
        |          |           |
        v          v           v
slow queries   pool sizing   transaction issue
locks          /too many     investigate
DB load        Pods
        |
        v
Check HPA / deployment
        |
        v
Connection pool size
        |
        v
Recent code/config change?
        |
        v
Mitigate
        |
        v
Validate
        |
        v
Permanent fix
```

---

# 68. Production Runbook

Create:

```text
runbooks/rds-connection-exhaustion.md
```

Recommended structure:

```markdown
# RDS Connection Exhaustion

## Symptoms

## Customer Impact

## Alerts

## First 5 Minutes

## Kubernetes Checks

## Hikari Checks

## PostgreSQL Checks

## RDS Checks

## HPA Checks

## Deployment Checks

## Immediate Mitigation

## Rollback

## Validation

## Root Cause

## Recovery

## Prevention

## Escalation
```

---

# 69. First 5 Minutes Runbook

### 1. Check application

```bash
kubectl get pods -n ecommerce
```

### 2. Check HPA

```bash
kubectl get hpa -n ecommerce
```

### 3. Check logs

```bash
kubectl logs \
  -n ecommerce \
  -l app=order-service \
  --tail=200
```

### 4. Check RDS connections

```text
CloudWatch / Performance Insights
```

### 5. Check PostgreSQL

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

### 6. Stop amplification

```text
HPA
Deployments
Batch jobs
Retries
```

### 7. Restore connection headroom

### 8. Validate customer flow

---

# 70. Business Validation

Never finish the incident merely because:

```text
DatabaseConnections ↓
```

Test the actual business flow:

```text
Login
 ↓
Product search
 ↓
Product details
 ↓
Add to cart
 ↓
Create order
 ↓
Inventory reservation
 ↓
Payment authorization
 ↓
Order confirmation
```

Then verify:

```text
HTTP 2xx
+
acceptable latency
+
no new database errors
+
no growing Hikari pending queue
```

---

# 71. RCA Template

```markdown
# RDS Connection Exhaustion RCA

## Incident

RDS connection exhaustion

## Date

YYYY-MM-DD

## Severity

SEV-1 / SEV-2 / SEV-3

## Customer Impact

Describe affected APIs and users.

## Detection

Which alert detected the issue?

## Timeline

10:00 - Normal
10:05 - Traffic increased
10:07 - HPA scaled
10:10 - Connections increased
10:12 - 5xx increased
10:15 - Mitigation started

## Root Cause

Describe the technical root cause.

## Contributing Factors

- Oversized connection pool
- HPA configuration
- Deployment surge
- Slow SQL
- Retry amplification

## Mitigation

Describe actions taken.

## Permanent Fix

Describe code/infrastructure changes.

## Prevention

- Connection budgets
- Hikari monitoring
- HPA guardrails
- Load testing
- Alerts
- Query optimization

## RTO

Measured value.

## Customer Impact Duration

Measured value.

## Action Items

- [ ] Fix pool configuration
- [ ] Review HPA
- [ ] Review maxSurge
- [ ] Add connection alerts
- [ ] Load test
- [ ] Update runbook
```

---

# 72. Production Prevention Checklist

## Application

- [ ] Hikari pool sized from measurements
- [ ] Connection timeout configured
- [ ] Query timeout configured where appropriate
- [ ] Transactions are bounded
- [ ] JDBC resources are properly managed
- [ ] Slow queries monitored
- [ ] Retry policy bounded
- [ ] Circuit breakers configured where appropriate

## Kubernetes

- [ ] HPA max replicas reviewed
- [ ] DB connection budget considered
- [ ] maxSurge considered
- [ ] Canary capacity considered
- [ ] Blue/green capacity considered
- [ ] Batch workloads isolated
- [ ] PDB configured appropriately

## RDS

- [ ] `max_connections` understood
- [ ] DatabaseConnections monitored
- [ ] Performance Insights configured as appropriate
- [ ] Slow queries investigated
- [ ] Lock monitoring available
- [ ] Capacity tested
- [ ] Backup/PITR tested

## Observability

- [ ] Hikari active
- [ ] Hikari idle
- [ ] Hikari pending
- [ ] Hikari acquisition latency
- [ ] RDS connections
- [ ] API latency
- [ ] API 5xx
- [ ] HPA replicas
- [ ] Idle-in-transaction monitoring

## Incident Management

- [ ] Runbook exists
- [ ] DBA escalation defined
- [ ] Rollback procedure tested
- [ ] Connection termination procedure controlled
- [ ] RCA template available
- [ ] Load test results documented

---

# 73. Final Production Mental Model

The most important relationship is:

```text
Traffic
   ↓
HPA
   ↓
Pod Count
   ↓
Hikari Pool Per Pod
   ↓
Total Application Connections
   ↓
RDS Connection Capacity
   ↓
Query Concurrency
   ↓
Database Performance
   ↓
API Latency
   ↓
Customer Experience
```

Therefore:

> **Kubernetes autoscaling and database connection capacity must be designed together.**

You cannot independently configure:

```text
HPA
Hikari
RDS
```

and expect the system to remain safe.

They form one capacity system.

---

# 74. The Golden Rule

Never think:

```text
More Pods = More Capacity
```

Think:

```text
More Pods
   ↓
More CPU capacity
   +
More memory capacity
   +
More database connections
   +
More Redis connections
   +
More downstream traffic
   +
More network traffic
   +
More observability traffic
```

Every scaling decision has downstream consequences.

---

# 75. Final Incident Architecture

```text
                    Customer
                       |
                    Route53
                       |
                 CloudFront/WAF
                       |
                      ALB
                       |
                    EKS
                       |
             +---------+---------+
             |                   |
          HPA/KEDA          Karpenter
             |                   |
          Pods                  Nodes
             |
        Spring Boot
             |
          Hikari
             |
       +-----+------+
       |            |
    RDS Proxy     RDS
       |            |
       +-----+------+
             |
       PostgreSQL
             |
      Performance Insights
             |
       Prometheus/Grafana
             |
       Alertmanager
             |
       Incident Response
```

---

# 76. Key Lessons

1. **RDS connection exhaustion is a capacity problem, not simply an application error.**
2. **Every Pod can create its own connection pool.**
3. **HPA can unintentionally amplify database pressure.**
4. **Deployment surge must be included in connection planning.**
5. **Blue/green and canary deployments can temporarily multiply connections.**
6. **Slow queries can cause connection exhaustion.**
7. **`idle in transaction` deserves immediate investigation.**
8. **Do not blindly increase Hikari pool size.**
9. **Do not blindly increase RDS `max_connections`.**
10. **RDS Proxy can help with connection management but does not fix bad queries.**
11. **Read replicas solve read scalability, not every connection problem.**
12. **Database connection capacity must be included in Kubernetes capacity planning.**
13. **GitOps rollback is preferred when a deployment caused the incident.**
14. **Always validate the actual business transaction after recovery.**
15. **The permanent solution is measured capacity + observability + guardrails.**

---

## Phase 18 Progress

```text
143 — CrashLoopBackOff                 ✅
144 — OOMKilled                        ✅
145 — CPU Saturation                   ✅
146 — Memory Saturation                ✅
147 — Disk Full                        ✅
148 — ImagePullBackOff                 ✅
149 — ALB 5xx                          ✅
150 — Service Connectivity Failure     ✅
151 — DNS Failure                      ✅
152 — RDS Connection Exhaustion        ✅
153 — Redis Failure                    ⏭️ NEXT
154 — SQS Backlog
155 — Kafka Consumer Lag
156 — Node Failure
157 — AZ Failure
158 — ArgoCD Drift
159 — Bad Production Deployment
160 — Security Incident
```

**Next chapter: `153-Redis-Failure.md` — production Redis/ElastiCache failure, application fallback to RDS, cache stampede, connection failures, failover, recovery, and preventing Redis failure from becoming an RDS outage.**