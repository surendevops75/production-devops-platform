# 108-RDS-Performance-and-Connection-Pooling.md

````markdown
# 108 — RDS Performance & Connection Pooling

## 1. Purpose

In our 21-microservice e-commerce platform, RDS PostgreSQL is a shared critical dependency.

A common production failure is not:

> "RDS is down."

Instead:

```text
More Pods
   ↓
More application connections
   ↓
Connection pool growth
   ↓
RDS connection exhaustion
   ↓
Requests wait for connections
   ↓
Latency increases
   ↓
Timeouts
   ↓
5xx errors
````

This chapter explains how to design, calculate, monitor, tune, and troubleshoot database connections in production.

---

# 2. Architecture

Our application architecture is:

```text
Internet
   ↓
ALB
   ↓
EKS
   ↓
order-service
   ↓
HikariCP
   ↓
RDS PostgreSQL
```

For multiple services:

```text
order-service ──────┐
payment-service ────┤
inventory-service ──┤
customer-service ───┤
catalog-service ────┤
                    ↓
              RDS PostgreSQL
```

This creates an important capacity-planning problem.

Every application Pod may maintain database connections.

---

# 3. What Is a Database Connection?

When the application needs PostgreSQL, it establishes a connection.

Conceptually:

```text
Application Pod
      |
      | TCP connection
      v
PostgreSQL
```

Creating a database connection has cost.

The connection involves:

```text
TCP connection
      ↓
TLS if enabled
      ↓
PostgreSQL authentication
      ↓
Backend session
      ↓
Application uses connection
```

We therefore normally do not create a new connection for every HTTP request.

Instead, we use a connection pool.

---

# 4. Connection Pool

Our Spring Boot application uses HikariCP.

Architecture:

```text
HTTP Requests
     |
     v
Order Service
     |
     v
HikariCP
 ┌───────────────┐
 │ Connection 1  │
 │ Connection 2  │
 │ Connection 3  │
 │ Connection 4  │
 │ ...           │
 └───────────────┘
     |
     v
PostgreSQL
```

The application borrows a connection:

```text
Request
   ↓
Get connection
   ↓
Execute SQL
   ↓
Return connection
```

The connection usually remains available in the pool.

---

# 5. Why Connection Pooling Matters

Without pooling:

```text
Request
 ↓
Create DB connection
 ↓
Authenticate
 ↓
Query
 ↓
Close connection
```

for every request.

With pooling:

```text
Application startup
       ↓
Create pool
       ↓
Request
       ↓
Borrow connection
       ↓
Query
       ↓
Return connection
```

This reduces connection establishment overhead.

---

# 6. Spring Boot + HikariCP

Spring Boot commonly uses HikariCP as its JDBC connection pool.

Example:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

These values are examples.

Do not copy `maximum-pool-size: 20` into every production service without capacity planning.

---

# 7. Maximum Pool Size

This is one of the most important settings.

Example:

```yaml
maximum-pool-size: 20
```

A single Pod can maintain up to approximately:

```text
20 database connections
```

Now consider Kubernetes.

```text
1 Pod  × 20 = 20 connections

5 Pods × 20 = 100 connections

10 Pods × 20 = 200 connections

50 Pods × 20 = 1000 connections
```

Therefore:

> Kubernetes autoscaling directly affects database connection pressure.

---

# 8. The Most Important Formula

For a simple first-order estimate:

```text
Maximum DB connections
≈
Number of Pods × Maximum Pool Size
```

Example:

```text
10 Pods
×
20 connections
=
200 connections
```

But this is only an application-level estimate.

The database has other connection consumers.

---

# 9. Multiple Microservices

Suppose:

```text
order-service:
10 Pods × 20 = 200

payment-service:
8 Pods × 15 = 120

inventory-service:
6 Pods × 15 = 90

customer-service:
5 Pods × 10 = 50
```

Total:

```text
200 + 120 + 90 + 50
= 460 connections
```

And that is only four services.

Our platform has 21 services.

Therefore we must calculate database connection capacity globally.

---

# 10. RDS max_connections

PostgreSQL/RDS has a maximum connection capacity determined by the database configuration and instance characteristics.

Check the current value rather than assuming it.

From PostgreSQL:

```sql
SHOW max_connections;
```

Also:

```sql
SELECT
    setting
FROM pg_settings
WHERE name = 'max_connections';
```

Do not assume the value is identical across RDS instance classes or PostgreSQL versions.

---

# 11. Current Connections

Check:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Example:

```text
350
```

This tells us the current number of PostgreSQL sessions represented in `pg_stat_activity`.

---

# 12. Connections by Application

A better query is:

```sql
SELECT
    application_name,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name
ORDER BY connections DESC;
```

This helps identify which application is consuming connections.

---

# 13. Connections by Database

```sql
SELECT
    datname,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

---

# 14. Connections by State

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

Possible states include:

```text
active
idle
idle in transaction
```

`idle in transaction` deserves particular attention.

---

# 15. Idle Connections

An idle connection is not necessarily a problem.

Connection pooling intentionally keeps connections available.

Example:

```text
20 connections

5 active
15 idle
```

This can be completely normal.

The important question is:

> Are connections available for requests, and is the database operating within safe capacity?

---

# 16. Idle in Transaction

This is more concerning.

Example:

```text
BEGIN;
SELECT ...
```

and then the transaction remains open.

The session may show:

```text
idle in transaction
```

Long-running transactions can:

* hold locks
* prevent vacuum progress
* increase table bloat
* interfere with maintenance
* consume connections

Investigate long-running sessions.

---

# 17. Find Long-Running Sessions

Example:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    now() - xact_start AS transaction_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Use carefully in production because query text may contain sensitive information.

---

# 18. Connection Timeout

Example:

```yaml
hikari:
  connection-timeout: 30000
```

This means the application waits up to approximately 30 seconds to obtain a connection from the pool.

If all pool connections are busy:

```text
Request
  ↓
Hikari pool
  ↓
No free connection
  ↓
Wait
  ↓
Timeout
```

The application may then return an error.

---

# 19. Connection Pool Exhaustion

Typical application error:

```text
HikariPool-1 - Connection is not available,
request timed out after 30000ms
```

This does not automatically mean:

> "RDS is down."

Possible causes:

```text
Slow SQL
Long transactions
Database locks
Too many requests
Pool too small
Database overloaded
Connection leak
Network problems
RDS connection exhaustion
```

---

# 20. Connection Leak

A connection leak happens when application code obtains a connection but does not properly return/release it.

With JPA/Spring this is less common when using normal managed repository patterns, but custom JDBC/native code can still introduce problems.

Typical pattern:

```text
Get connection
   ↓
Exception
   ↓
Connection not released
   ↓
Pool shrinks
   ↓
Eventually exhausted
```

Use proper resource management and application instrumentation.

---

# 21. Pool Size Is Not "Bigger Is Better"

A common mistake:

```yaml
maximum-pool-size: 100
```

because:

> "The application needs to be fast."

Suppose:

```text
20 Pods × 100
=
2000 connections
```

If RDS cannot safely handle that workload, increasing the pool makes the situation worse.

A connection pool controls concurrency.

It does not create database capacity.

---

# 22. Database Concurrency

Imagine:

```text
1000 requests
       ↓
100 DB connections
       ↓
100 SQL operations concurrently
```

The remaining requests wait.

That can be acceptable if the database can process those 100 operations efficiently.

But:

```text
500 DB connections
```

may cause:

```text
CPU contention
Memory pressure
Lock contention
Context switching
I/O contention
Query latency
```

The optimal connection count depends on workload and database capacity.

---

# 23. Connection Pool and CPU

A useful mental model:

```text
More connections
       ↓
More possible concurrent DB work
       ↓
More database CPU/I/O pressure
```

Eventually:

```text
More connections
       ↓
More contention
       ↓
Less throughput
```

Therefore connection tuning requires load testing.

---

# 24. RDS Proxy

RDS Proxy can be used as a connection-management layer in appropriate architectures.

Architecture:

```text
EKS Pods
   |
   | many client connections
   v
RDS Proxy
   |
   | managed DB connections
   v
RDS PostgreSQL
```

The proxy can help with:

* connection pooling/multiplexing scenarios
* connection management
* failover handling
* reducing direct connection pressure in suitable workloads

It does not magically fix:

```text
Bad SQL
Huge transactions
Database CPU saturation
Missing indexes
Application bugs
```

---

# 25. Direct RDS vs RDS Proxy

### Direct

```text
Pod
 ↓
RDS
```

Advantages:

* simple
* fewer components
* lower architectural complexity

### With Proxy

```text
Pod
 ↓
RDS Proxy
 ↓
RDS
```

Advantages may include:

* connection management
* smoother connection behavior during scaling
* reduced direct connection pressure in suitable workloads

The choice should be based on workload and testing.

---

# 26. When Pods Scale

Our HPA may scale:

```text
3 Pods
 ↓
10 Pods
 ↓
30 Pods
```

If every Pod has:

```text
maximum-pool-size = 20
```

then potential connections become:

```text
3 × 20 = 60

10 × 20 = 200

30 × 20 = 600
```

This is why HPA configuration and database capacity cannot be designed independently.

---

# 27. HPA + Database Connection Planning

Architecture:

```text
CPU increases
      ↓
HPA increases Pods
      ↓
More Hikari pools
      ↓
More possible DB connections
      ↓
RDS connection pressure
```

Therefore:

```text
HPA maximum replicas
+
Pool size
+
Database connection capacity
```

must be considered together.

---

# 28. Example Capacity Calculation

Suppose:

```text
RDS max_connections = 500
```

Do not allocate all 500 to applications.

Reserve capacity for:

```text
DB administration
Monitoring
Maintenance
Migration
Unexpected connections
Other services
```

Suppose the engineering capacity target is:

```text
350 application connections
```

Then:

```text
order-service:
10 Pods × 15 = 150

payment-service:
5 Pods × 15 = 75

inventory-service:
5 Pods × 10 = 50

customer-service:
5 Pods × 10 = 50

Total = 325
```

This leaves some headroom.

The exact safe threshold must be established through testing.

---

# 29. Connection Budget

For our platform, we should maintain a connection budget.

Example:

| Service           | Max Pods | Pool | Potential Connections |
| ----------------- | -------: | ---: | --------------------: |
| order-service     |       10 |   15 |                   150 |
| payment-service   |        5 |   15 |                    75 |
| inventory-service |        5 |   10 |                    50 |
| customer-service  |        5 |   10 |                    50 |
| catalog-service   |        5 |   10 |                    50 |

Potential total:

```text
375
```

This should be reviewed whenever:

```text
HPA maxReplicas changes
Pool size changes
RDS instance changes
New service is added
```

---

# 30. Hikari Minimum Idle

Example:

```yaml
minimum-idle: 5
```

This can maintain a number of idle connections.

But across many Pods:

```text
50 Pods × 5 idle connections
=
250 idle connections
```

That can be significant.

Therefore minimum idle should also be considered globally.

---

# 31. maxLifetime

Example:

```yaml
max-lifetime: 1800000
```

which is approximately:

```text
30 minutes
```

Connections are periodically retired and replaced.

The exact value should be chosen considering:

```text
Database/network behavior
Proxy behavior
Connection lifetime policies
Application workload
```

Do not blindly copy values from another system.

---

# 32. Keepalive

Hikari can support connection keepalive behavior.

This may help in environments where network infrastructure closes idle connections.

But unnecessary aggressive keepalive can create additional traffic.

Use it only when there is an actual requirement.

---

# 33. Connection Timeout vs Query Timeout

These are different.

### Connection timeout

```text
How long do I wait for a pool connection?
```

### Query timeout

```text
How long should SQL execution be allowed to run?
```

Example:

```text
Request
 ↓
Wait for connection
 ↓
Get connection
 ↓
Execute query
 ↓
Query takes too long
```

Both stages need appropriate controls.

---

# 34. Slow Query Problem

Suppose:

```text
Pool = 20
```

and each query normally takes:

```text
50 ms
```

But a bad query takes:

```text
20 seconds
```

Now connections remain occupied for much longer.

Result:

```text
20 connections
 ↓
slow queries
 ↓
pool exhausted
 ↓
requests wait
 ↓
timeouts
```

The underlying problem may be SQL performance, not pool size.

---

# 35. PostgreSQL Query Investigation

Use PostgreSQL tools appropriate to your environment.

Example:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

For production, protect sensitive query text and restrict database access.

---

# 36. EXPLAIN ANALYZE

For a controlled query investigation:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 'customer-001';
```

This can show:

```text
Execution time
Planning time
Index usage
Rows scanned
Rows returned
```

Do not run expensive `EXPLAIN ANALYZE` queries blindly against a busy production database.

Test against a representative environment where possible.

---

# 37. Indexes

Suppose:

```sql
SELECT *
FROM orders
WHERE customer_id = 'customer-001';
```

If `customer_id` is frequently queried, an appropriate index may be required.

Example:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Index decisions should be based on workload and query plans.

Indexes improve some reads but add:

```text
Storage
Write overhead
Maintenance overhead
```

---

# 38. Connection Pool Metrics

Application metrics should expose pool behavior.

Hikari metrics can provide concepts such as:

```text
active connections
idle connections
pending threads
maximum connections
minimum connections
```

These are highly useful for diagnosing:

```text
Pool exhaustion
Slow queries
Traffic spikes
Database degradation
```

---

# 39. Prometheus Integration

Our Spring Boot service already exposes:

```text
/actuator/prometheus
```

Prometheus can scrape application metrics.

Architecture:

```text
order-service
      |
      | Hikari metrics
      v
Prometheus
      |
      v
Grafana
```

---

# 40. Important Dashboard Panels

For order-service:

```text
HTTP request rate
HTTP error rate
HTTP latency
Hikari active connections
Hikari idle connections
Hikari pending threads
Hikari maximum pool
JVM heap
CPU
Memory
```

For RDS:

```text
CPU
Database connections
Freeable memory
Read latency
Write latency
IOPS
Throughput
Storage
```

---

# 41. Correlation

The important production skill is correlation.

Example:

```text
10:05
HTTP traffic increases

10:06
HPA increases Pods

10:07
Hikari connections increase

10:08
RDS connections increase

10:09
RDS CPU increases

10:10
Query latency increases

10:11
Hikari pending threads increase

10:12
HTTP 5xx increases
```

This gives us a causal investigation path.

---

# 42. Grafana Dashboard

Example structure:

```text
ORDER SERVICE
─────────────────────────────
Request Rate
Error Rate
P95 Latency
P99 Latency

DATABASE CONNECTIONS
─────────────────────────────
Active
Idle
Pending
Maximum

RDS
─────────────────────────────
CPU
Connections
Latency
IOPS
Memory

KUBERNETES
─────────────────────────────
Pods
CPU
Memory
HPA replicas
```

---

# 43. Alerts

Potential application alert:

```text
Hikari pending threads > threshold
```

Potential RDS alert:

```text
Database connections approaching safe capacity
```

Potential combined alert:

```text
DB connections high
AND
application latency high
```

Alert thresholds should be based on measured capacity.

---

# 44. Connection Exhaustion Incident

Suppose an alert fires:

```text
order-service DB pool saturation
```

Start with:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl get hpa -n ecommerce-prod
```

Check replicas:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod
```

---

# 45. Check Application Logs

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200
```

Search for:

```text
Connection is not available
timeout
HikariPool
SQLException
connection refused
```

---

# 46. Check RDS Connections

From an approved DB session:

```sql
SELECT count(*)
FROM pg_stat_activity;
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

---

# 47. Check RDS CPU

If connections are high and CPU is also high:

```text
Connections
   +
CPU
   +
Query latency
```

investigate SQL/database capacity.

If connections are high but CPU is low:

```text
Possible:
Idle pools
Connection leak
Long transactions
Application pool configuration
```

---

# 48. Check Long Transactions

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    now() - xact_start AS transaction_age
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Long transactions may be contributing to the problem.

---

# 49. Check HPA

```bash
kubectl get hpa \
  -n ecommerce-prod
```

Check:

```bash
kubectl describe hpa \
  order-service \
  -n ecommerce-prod
```

Look for:

```text
Current replicas
Desired replicas
CPU utilization
Memory utilization
Scaling events
```

---

# 50. Check Recent Deployment

```bash
kubectl rollout history \
  deployment/order-service \
  -n ecommerce-prod
```

If the problem started immediately after deployment:

```text
Deployment
   ↓
New application version
   ↓
New SQL behavior
   ↓
Slow queries
   ↓
Pool exhaustion
```

Investigate the release.

---

# 51. Do Not Immediately Increase Pool Size

Bad response:

```text
Hikari timeout
       ↓
Increase pool from 20 to 100
```

This can transform:

```text
10 Pods × 20 = 200
```

into:

```text
10 Pods × 100 = 1000
```

which may make the incident worse.

First identify the bottleneck.

---

# 52. Possible Immediate Mitigations

Depending on the root cause:

```text
Stop problematic deployment
Rollback application
Reduce HPA max replicas
Fix runaway query
Terminate problematic sessions
Temporarily reduce traffic
Scale RDS where appropriate
Enable/adjust RDS Proxy where justified
```

Every production action should follow the incident/change process.

---

# 53. Connection Exhaustion Decision Tree

```text
DB connection alert
        |
        v
Are connections high?
        |
       Yes
        |
        v
Are Pods unusually high?
     /       \
   Yes        No
    |          |
Check HPA     Check pool
              configuration
                   |
                   v
          Are queries slow?
             /        \
           Yes         No
            |           |
       Investigate     Check
       SQL/locks       leaks/
                       transactions
```

---

# 54. Database Locks

A query may wait because another transaction holds a lock.

This can cause:

```text
Query waits
   ↓
Connection remains occupied
   ↓
Pool connections become busy
   ↓
Pool exhaustion
```

Investigate lock waits using PostgreSQL activity and lock views.

Example:

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

---

# 55. Transaction Design

Avoid unnecessarily long transactions.

Bad conceptual flow:

```text
BEGIN
 ↓
Database query
 ↓
External API call
 ↓
Wait 5 seconds
 ↓
Another query
 ↓
COMMIT
```

The database connection and transaction may remain occupied while waiting on an external dependency.

Prefer:

```text
Short transaction
 ↓
Commit
 ↓
External processing
```

where the business workflow allows it.

---

# 56. Order-Service Example

Our order flow may involve:

```text
Create Order
   ↓
PostgreSQL
   ↓
Commit
   ↓
Publish order event
   ↓
Payment service
```

We should not hold a database transaction open while waiting synchronously for:

```text
Payment
Notification
Shipping
External API
```

For reliable event workflows, later chapters will introduce messaging and event-driven patterns.

---

# 57. Hikari Configuration Example

A starting configuration for a controlled environment:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 15
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

This is not a universal production configuration.

The final values should come from:

```text
Load testing
RDS capacity
Query latency
Pod count
Traffic patterns
Connection usage
```

---

# 58. Environment-Specific Pool Configuration

Do not necessarily use the same pool configuration everywhere.

Example:

```text
DEV
3 Pods
Pool = 5

QA
5 Pods
Pool = 10

STAGING
10 Pods
Pool = 10

PROD
Autoscaling
Capacity-derived pool
```

The actual values must be calculated and tested.

---

# 59. HPA Maximum Replica Guardrail

Suppose:

```text
RDS application connection budget = 350
```

and:

```text
order-service pool = 15
```

If order-service alone can scale to:

```text
30 Pods
```

potential connections:

```text
30 × 15
=
450
```

That exceeds the service's assumed connection budget.

Therefore:

```text
HPA maxReplicas
```

must be part of database capacity planning.

---

# 60. 21-Service Connection Budget

Eventually our platform needs a table like:

```text
Service
  ↓
Expected Pods
  ↓
Maximum Pods
  ↓
Pool Size
  ↓
Potential Connections
  ↓
RDS Connection Budget
```

Example:

```text
order-service
maxPods = 10
pool = 15
budget = 150
```

Repeat for all services that use the database.

---

# 61. Shared Database vs Separate Database

Our architecture may eventually use:

```text
order-service → orders DB
payment-service → payments DB
inventory-service → inventory DB
```

rather than:

```text
21 services
     ↓
One giant database
```

Database ownership is an architectural decision.

Separate databases can improve:

```text
Isolation
Ownership
Scaling
Failure boundaries
Security
```

but introduce:

```text
Operational complexity
Cross-service consistency challenges
More infrastructure
More migrations
```

---

# 62. Database-per-Service Principle

For microservices, the logical ownership model should generally be:

```text
Service
   ↓
Owns its data
```

For example:

```text
Order Service
    ↓
orders data

Payment Service
    ↓
payment data

Inventory Service
    ↓
inventory data
```

A service should not directly modify another service's tables.

Use APIs/events for cross-service interaction.

---

# 63. Connection Pooling with Read Replicas

If the architecture introduces PostgreSQL read replicas:

```text
Application
    |
    +------ writes ------> Primary
    |
    +------ reads -------> Replica
```

The application must intentionally route read traffic.

Read replicas do not automatically solve every database scaling problem.

Consider:

```text
Replication lag
Read-after-write consistency
Connection pools
Query routing
Failover
```

---

# 64. Connection Pooling During Deployment

Suppose:

```text
Old version:
10 Pods × 15 = 150

Rolling deployment:
Old Pods + New Pods
```

During rollout, temporary Pod count can exceed the normal replica count.

For example:

```text
10 old
+
2 surge
=
12 Pods
```

Potential connections:

```text
12 × 15
=
180
```

Therefore deployment strategy affects database connection capacity.

This is another reason:

```yaml
maxSurge: 1
```

and capacity planning matter.

---

# 65. Blue-Green Connection Impact

Blue-Green can be more significant.

Example:

```text
Blue:
10 Pods × 15 = 150

Green:
10 Pods × 15 = 150
```

Potential:

```text
300 connections
```

before considering other services.

Therefore Blue-Green deployments require database capacity planning.

---

# 66. Canary Connection Impact

Canary:

```text
Stable:
10 Pods

Canary:
1 Pod
```

may temporarily add only a small number of connections.

But traffic routing and HPA behavior still need to be considered.

---

# 67. Database Connection SLO Signals

Useful signals:

```text
Connection utilization
Pool wait time
Pool saturation
Query latency
Transaction duration
RDS CPU
RDS memory
Lock waits
Error rate
```

A database dashboard should correlate these rather than showing only CPU.

---

# 68. Example Prometheus Alert Concept

Conceptually:

```yaml
groups:
  - name: database
    rules:
      - alert: OrderServiceConnectionPoolSaturation
        expr: |
          hikari_connections_pending > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Order service DB connection pool is saturated"
          runbook: "https://runbooks.example.com/order-db-pool"
```

The actual metric name depends on the Spring Boot/Micrometer version and metric configuration.

Verify the metric exposed by the deployed application before creating the final alert.

---

# 69. RDS Connection Alert

A conceptual alert:

```text
Current DB connections
---------------------- > threshold
Connection capacity
```

Do not use an arbitrary threshold such as 80% without understanding:

```text
Application budget
Admin reserve
Maintenance behavior
Workload characteristics
```

---

# 70. Load Test Before Final Pool Configuration

The correct workflow is:

```text
Choose initial pool
      ↓
Load test
      ↓
Observe RDS
      ↓
Observe Hikari
      ↓
Observe latency
      ↓
Change configuration
      ↓
Repeat
```

Do not tune database pools from theory alone.

---

# 71. Example Load Test

Suppose traffic:

```text
100 requests/sec
```

Test:

```text
Pool = 5
Pool = 10
Pool = 15
Pool = 20
```

Measure:

```text
P95 latency
P99 latency
Throughput
RDS CPU
RDS connections
Hikari pending
Error rate
```

The goal is to identify the useful operating range.

---

# 72. Connection Pool Capacity Test

Example table:

| Pool | Pods | Potential Connections |     P95 |  Errors |
| ---: | ---: | --------------------: | ------: | ------: |
|    5 |   10 |                    50 | Measure | Measure |
|   10 |   10 |                   100 | Measure | Measure |
|   15 |   10 |                   150 | Measure | Measure |
|   20 |   10 |                   200 | Measure | Measure |

Do not select the largest pool simply because it gives a higher number.

---

# 73. Production Design

Our target architecture becomes:

```text
                  ┌─────────────────────┐
                  │        EKS          │
                  │                     │
                  │ order-service Pods  │
                  │                     │
                  │ HikariCP            │
                  └──────────┬──────────┘
                             │
                             │
                       Optional Proxy
                             │
                             ▼
                  ┌─────────────────────┐
                  │   RDS PostgreSQL    │
                  │                     │
                  │ Multi-AZ            │
                  │ Backups             │
                  │ PITR                │
                  └─────────────────────┘
```

---

# 74. Golden Troubleshooting Flow

When users report:

```text
Orders are slow
```

Do not immediately blame RDS.

Check:

```text
1. HTTP latency
2. Pod CPU/memory
3. HPA replicas
4. Hikari active/idle/pending
5. RDS connections
6. RDS CPU/memory
7. Query latency
8. Locks
9. Recent deployment
10. Recent traffic increase
```

---

# 75. Example Production Incident

## Symptoms

```text
Order API P99 latency:
300ms → 8s

HTTP 5xx:
0.2% → 12%

HPA:
5 → 20 Pods

Hikari pending:
0 → 40

RDS connections:
150 → 480
```

Investigation:

```text
Traffic spike
     ↓
HPA scaled
     ↓
20 Pods
     ↓
20 × pool 20
     ↓
400 potential connections
     ↓
RDS became connection constrained
     ↓
Requests waited
```

But this is still only a hypothesis.

We verify it using metrics and database state.

---

# 76. Mitigation

Possible controlled mitigation:

```text
1. Stop further scaling if appropriate
2. Reduce traffic if necessary
3. Investigate expensive queries
4. Check database capacity
5. Roll back recent application changes if relevant
6. Adjust pool/HPA configuration through approved change
7. Scale database if justified
```

Avoid making multiple uncontrolled changes simultaneously.

---

# 77. Root Cause

A final RCA might identify:

```text
Root cause:
HPA maxReplicas allowed 20 Pods while
connection-pool configuration allowed each Pod
to maintain 20 database connections.

The resulting connection demand exceeded the
tested RDS application connection budget.
```

Contributing factors may include:

```text
No connection budget
No Hikari saturation alert
No HPA/database capacity guardrail
Insufficient load testing
```

---

# 78. Preventive Actions

Examples:

```text
[ ] Define per-service DB connection budgets
[ ] Define HPA max replicas based on DB capacity
[ ] Tune Hikari pool sizes
[ ] Add Hikari metrics
[ ] Add RDS connection alerts
[ ] Add long-transaction monitoring
[ ] Load test database limits
[ ] Review deployment surge capacity
[ ] Document database scaling procedure
```

---

# 79. Production Checklist

## Application

```text
[ ] HikariCP configured
[ ] Pool size capacity-tested
[ ] Connection timeout configured
[ ] Query timeout strategy defined
[ ] Transaction boundaries reviewed
[ ] Connection metrics exposed
```

## Kubernetes

```text
[ ] HPA maxReplicas reviewed
[ ] Rolling update surge reviewed
[ ] Blue-Green connection impact reviewed
[ ] Canary capacity reviewed
[ ] Resource requests configured
```

## RDS

```text
[ ] max_connections understood
[ ] Current connections monitored
[ ] CPU monitored
[ ] Memory monitored
[ ] IOPS monitored
[ ] Latency monitored
[ ] Backup/PITR enabled
[ ] Multi-AZ configured where required
```

## Operations

```text
[ ] Connection exhaustion runbook
[ ] Pool saturation alert
[ ] RDS connection alert
[ ] Long transaction investigation
[ ] Load test completed
[ ] Capacity budget documented
```

---

# 80. Final Mental Model

Remember this chain:

```text
Traffic
   ↓
HTTP Requests
   ↓
HPA
   ↓
Pod Count
   ↓
Hikari Pool
   ↓
Database Connections
   ↓
SQL Concurrency
   ↓
RDS CPU / Memory / I/O
   ↓
Query Latency
   ↓
Application Latency
   ↓
User Experience
```

This is why:

> Kubernetes autoscaling and database capacity planning must be designed together.

The next chapter will test what happens when the database itself fails over.

---

# 81. Next Chapter

**109-RDS-Failover-Testing.md**

We will perform a controlled RDS Multi-AZ failover test and trace:

```text
Application
    ↓
RDS primary
    ↓
Failure
    ↓
AWS Multi-AZ failover
    ↓
New primary
    ↓
DNS endpoint
    ↓
Connection recovery
    ↓
Application recovery
```

We will measure:

* failover duration
* application impact
* connection behavior
* Hikari recovery
* transaction failures
* retry behavior
* health checks
* monitoring signals
* RTO
* post-failover validation

```
```
