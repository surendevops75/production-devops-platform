# 153. Redis Failure — Production Incident

> **Phase 18 — Incident Management**  
> **Incident #153**  
> **Environment:** Production  
> **Cache:** Amazon ElastiCache for Redis/Valkey-compatible workloads  
> **Application:** 21-microservice E-Commerce Platform

---

## 1. Incident Overview

Redis is normally used as a **performance and temporary-state layer**, not the primary source of truth for critical business data.

Our architecture follows:

```text
Application
    |
    +---- PostgreSQL → Source of truth
    |
    +---- Redis → Cache / temporary state
```

Therefore, when Redis fails, our first principle is:

> **A Redis outage should degrade application performance, not automatically become an order/data-loss outage.**

However, Redis failure can become serious if the application is poorly designed.

For example:

```text
Redis failure
     ↓
Every request goes to PostgreSQL
     ↓
RDS traffic increases
     ↓
RDS connections increase
     ↓
RDS latency increases
     ↓
Application latency increases
     ↓
HPA scales Pods
     ↓
More database connections
     ↓
RDS outage
```

This is the **Redis → RDS cascading failure** we must prevent.

---

# 2. Our Production Architecture

For our project:

```text
                         Internet
                            |
                       Route 53
                            |
                     CloudFront/WAF
                            |
                           ALB
                            |
                         EKS
                            |
                  +---------+---------+
                  |                   |
             Microservices       Async workers
                  |
          +-------+--------+
          |                |
       PostgreSQL         Redis
       Source of          Cache
        Truth
```

More specifically:

```text
order-service
     |
     +----------> PostgreSQL
     |
     +----------> Redis
```

Redis may be used for:

- Product caching
- Session-related data where appropriate
- Cart data depending on design
- Frequently accessed customer information
- Pricing cache
- Promotion cache
- Rate limiting
- Temporary state
- Distributed coordination where appropriate

Critical durable business data remains in PostgreSQL or another durable system of record.

---

# 3. What Redis Failure Looks Like

Possible application errors:

```text
RedisConnectionException
RedisTimeoutException
Connection refused
Connection reset
Unable to connect to Redis
Read timed out
Command timed out
Pool exhausted
```

Customer symptoms:

```text
Product pages become slow
Cart becomes slow
Login becomes slow
Order creation latency increases
HTTP 5xx increases
```

But there is an important distinction:

```text
Redis unavailable
```

does **not necessarily mean:

```text
Application unavailable
```

A well-designed application can fall back to PostgreSQL for cacheable data.

---

# 4. Incident Scenario

Suppose our `catalog-service` normally behaves like this:

```text
Request
   |
   ↓
Redis
   |
   +-- Cache HIT → return quickly
   |
   +-- Cache MISS → PostgreSQL
```

Normal traffic:

```text
100 requests/sec

Redis:
90 cache hits

PostgreSQL:
10 database requests
```

Now Redis fails.

Every request becomes:

```text
100 requests/sec
      ↓
Redis unavailable
      ↓
PostgreSQL
```

PostgreSQL traffic increases:

```text
10 RPS → 100 RPS
```

That's a **10× increase**.

At large scale this can overwhelm the database.

---

# 5. The Correct Incident Flow

Use:

```text
Alert
  ↓
Confirm customer impact
  ↓
Check Redis health
  ↓
Check application Redis errors
  ↓
Check Redis connections/latency
  ↓
Check ElastiCache events
  ↓
Determine cache failure vs network failure
  ↓
Protect PostgreSQL
  ↓
Enable/verify fallback
  ↓
Control cache-miss storm
  ↓
Restore Redis
  ↓
Validate cache behavior
  ↓
Warm cache if appropriate
  ↓
Verify PostgreSQL recovery
  ↓
RCA
```

---

# 6. Step 1 — Confirm Application Impact

Check Pods:

```bash
kubectl get pods -n ecommerce
```

Check order-service:

```bash
kubectl get pods \
  -n ecommerce \
  -l app=order-service
```

Check logs:

```bash
kubectl logs \
  -n ecommerce \
  -l app=order-service \
  --tail=200
```

Look for:

```text
RedisConnectionException
RedisTimeoutException
Connection refused
Read timed out
Command timed out
```

---

# 7. Step 2 — Check Redis Metrics

Check ElastiCache metrics.

Important signals include:

```text
CPU utilization
Memory usage
Current connections
Network traffic
Cache hits
Cache misses
Evictions
Replication/engine health
Latency
```

Exact available metrics depend on the Redis/Valkey service configuration and deployment mode.

Look for:

```text
Redis latency ↑
Connections ↑
Errors ↑
Cache hits ↓
Cache misses ↑
Evictions ↑
```

---

# 8. Check Redis Availability

Determine whether Redis is:

```text
Healthy
Degraded
Failing over
Unavailable
```

Check the AWS resource state through your approved AWS tooling.

For example:

```bash
aws elasticache describe-replication-groups \
  --region ap-south-1
```

If your environment uses a different ElastiCache topology, inspect the corresponding cache cluster/resource information.

---

# 9. Application Logs

Search ELK for:

```text
service.name: order-service
```

and:

```text
Redis
```

Typical errors:

```text
Unable to connect to Redis
Connection timed out
Connection refused
Read timed out
Redis command timeout
```

Also search by:

```text
correlationId
traceId
```

to determine which customer requests are affected.

---

# 10. Check Network Connectivity

Redis might be healthy while applications cannot reach it.

The architecture is:

```text
Pod
 ↓
DNS
 ↓
Redis endpoint
 ↓
VPC networking
 ↓
Security Group
 ↓
Redis
```

Check DNS from a Pod:

```bash
kubectl exec -it <pod> -n ecommerce -- \
  nslookup <redis-endpoint>
```

Test TCP connectivity:

```bash
kubectl exec -it <pod> -n ecommerce -- \
  nc -vz <redis-endpoint> 6379
```

For TLS-enabled Redis/Valkey, test using the appropriate TLS client configuration rather than assuming plaintext port behavior.

---

# 11. Distinguish Redis Failure From Network Failure

### Case 1

```text
DNS fails
```

Likely:

```text
CoreDNS
VPC DNS
endpoint
configuration
```

### Case 2

```text
DNS works
TCP connection times out
```

Investigate:

```text
Security Groups
Network ACLs
NetworkPolicy
routing
VPC connectivity
```

### Case 3

```text
TCP works
Redis commands timeout
```

Investigate:

```text
Redis health
CPU
memory
slow commands
connection saturation
failover
```

### Case 4

```text
Redis itself is unavailable
```

Investigate:

```text
ElastiCache events
failover
maintenance
node health
replication
```

---

# 12. Check Application Redis Pool

Depending on the Java Redis client configuration, the application may also have a Redis connection pool.

For example, Spring Boot commonly uses Lettuce.

Conceptually:

```text
Spring Boot
    |
  Lettuce
    |
Connection pool
    |
 Redis
```

Monitor:

```text
Active connections
Idle connections
Pending requests
Connection errors
Command latency
```

The exact metrics depend on your client and Micrometer configuration.

---

# 13. Why Redis Connection Pooling Matters

Suppose:

```text
20 Pods
```

and each Pod can create:

```text
20 Redis connections
```

Then potential connections are:

```text
20 × 20
=
400
```

Again:

```text
Pods × pool size
```

can become important.

The same capacity-planning principle applies to:

```text
PostgreSQL
Redis
Kafka
RabbitMQ
External APIs
```

---

# 14. Most Important Question

When Redis fails, ask:

> **Can the application continue without Redis?**

If yes:

```text
Redis failure
     ↓
Cache miss/fallback
     ↓
Database
```

If no:

```text
Redis failure
     ↓
Application failure
```

The second architecture is riskier.

---

# 15. Cache-Aside Pattern

Our preferred caching pattern is:

```text
Application
    |
    ↓
Redis
    |
    +---- HIT → return data
    |
    +---- MISS
           |
           ↓
       PostgreSQL
           |
           ↓
       Put into Redis
           |
           ↓
       Return data
```

Example:

```text
GET product/123
```

First:

```text
Redis GET product:123
```

If not found:

```text
PostgreSQL SELECT product 123
```

Then:

```text
Redis SET product:123
```

Then return response.

---

# 16. What Happens During Redis Failure?

The application should behave like:

```text
GET product/123
      |
      ↓
Redis
      |
      X
  unavailable
      |
      ↓
Fallback
      |
      ↓
PostgreSQL
      |
      ↓
Return product
```

But we must protect PostgreSQL.

Otherwise:

```text
1000 RPS
   ↓
Redis failure
   ↓
1000 RPS directly to PostgreSQL
   ↓
RDS overloaded
```

---

# 17. The Dangerous Cache Stampede

Suppose Redis comes back empty after failure.

At the same moment:

```text
10,000 requests
```

request:

```text
product:123
```

All see:

```text
CACHE MISS
```

Then all hit PostgreSQL:

```text
10,000 requests
      ↓
PostgreSQL
```

This is a **cache stampede**.

---

# 18. Cache Stampede Protection

Possible techniques:

### Request coalescing

Only one request loads the value:

```text
Request A
   ↓
Load product
   ↓
Redis

Requests B/C/D
   ↓
Wait for same result
```

### Distributed locking

One worker becomes the loader.

### Cache warming

Preload frequently accessed data.

### Stale-while-revalidate

Serve slightly stale data while refreshing in the background where business requirements permit.

### Rate limiting

Protect the database during recovery.

---

# 19. Redis Failure + RDS Protection

During an incident, we want:

```text
Redis failure
     ↓
Fallback
     ↓
Rate limited
     ↓
PostgreSQL
```

not:

```text
Redis failure
     ↓
Everything → PostgreSQL
     ↓
RDS overload
```

This is one of the most important production lessons.

---

# 20. Check RDS During Redis Incident

Even though this is a Redis incident, immediately check:

```text
RDS DatabaseConnections
RDS CPU
RDS DB load
RDS latency
API latency
Hikari pending
```

Because the secondary failure may already be starting.

Architecture:

```text
Redis failure
     ↓
Cache misses
     ↓
RDS load
     ↓
Hikari pools
     ↓
RDS connections
```

---

# 21. Redis Failure Can Become RDS Connection Exhaustion

This connects directly to Chapter 152.

Example:

```text
Normal:

Redis:
90% cache hit

RDS:
100 connections
```

Redis fails:

```text
Redis:
0% hit

RDS:
500 connections
```

Then:

```text
Hikari:
pending ↑

API:
latency ↑

5xx:
↑
```

Now we have two incidents:

```text
Primary:
Redis failure

Secondary:
RDS connection exhaustion
```

This is why incident responders must understand dependencies.

---

# 22. Immediate Mitigation

Depending on the failure and application design:

### 1. Protect RDS

Reduce cache-fallback concurrency.

### 2. Reduce noncritical workloads

For example:

```text
reporting
recommendation
background processing
```

### 3. Apply rate limits

Protect database-dependent endpoints.

### 4. Disable expensive features

If business-approved:

```text
recommendations
personalized suggestions
noncritical analytics
```

### 5. Stop retry storms

Don't repeatedly hammer Redis.

---

# 23. Don't Do This

Bad code pattern:

```text
Redis GET
  ↓
failure
  ↓
retry immediately
  ↓
retry immediately
  ↓
retry immediately
  ↓
retry immediately
```

With thousands of requests:

```text
Redis
 ↑↑↑↑↑
```

This can make the incident worse.

Use:

```text
timeout
+
bounded retry
+
backoff
+
jitter
+
circuit breaker
```

where appropriate.

---

# 24. Redis Timeout

Suppose application configuration has:

```text
Redis timeout = 30 seconds
```

and Redis is unavailable.

Every request could wait:

```text
30 seconds
```

before falling back.

At high traffic:

```text
1000 requests
×
30 second wait
```

can consume application resources.

Therefore cache operations should generally have **short, measured timeouts** appropriate to the application's latency budget.

---

# 25. Circuit Breaker

Conceptually:

```text
             Redis
               |
        +------+------+
        |             |
      Healthy       Failed
        |             |
     normal       circuit opens
                      |
                      ↓
                   fallback
                      |
                      ↓
                  PostgreSQL
```

When Redis repeatedly fails:

```text
CLOSED
   ↓
failures
   ↓
OPEN
   ↓
don't keep calling Redis
```

After a controlled period:

```text
OPEN
 ↓
HALF-OPEN
 ↓
test request
 ↓
healthy?
```

This prevents repeated Redis calls during an outage.

---

# 26. Redis Recovery

Suppose ElastiCache has recovered.

Don't immediately send all traffic at full intensity if the application has been operating in fallback mode.

First verify:

```text
Redis health
DNS
TCP connectivity
authentication
TLS
application connections
latency
```

Then:

```text
Redis
  ↓
Application
  ↓
Cache writes
  ↓
Cache hits recover
```

---

# 27. Cache Warming

After recovery, important keys may be missing.

We can warm selected high-value keys.

For example:

```text
Popular products
Popular categories
Pricing
Promotions
Configuration
```

But don't blindly load the entire database into Redis.

Otherwise:

```text
Cache warming
      ↓
Millions of DB queries
      ↓
RDS overload
```

Warm gradually.

---

# 28. Redis Failover

With a managed Redis/Valkey-compatible service, there may be replicas and automatic failover depending on the deployment mode.

During failover:

```text
Primary
   X
   ↓
Replica promoted
   ↓
New primary
```

Applications should tolerate brief connection errors and reconnect according to the client/service behavior.

Do not design the application to permanently bind to a specific Redis node address when the managed service provides a stable endpoint.

---

# 29. Redis Data Classification

Not every Redis key has the same recovery importance.

Classify:

### Cache

```text
Product cache
Pricing cache
Catalog cache
```

Can usually be rebuilt.

### Temporary state

```text
Short-lived session state
Rate limits
locks
```

Recovery behavior depends on business design.

### Business-critical state

If something is truly business-critical and must survive Redis loss:

> It should generally have a durable source of truth outside the cache.

For example:

```text
Order
Payment
Inventory reservation
```

should not exist only in Redis.

---

# 30. Redis and Database Consistency

Our preferred model:

```text
PostgreSQL
    |
    | Source of truth
    ↓
Redis
    |
    | Cached representation
    ↓
Application
```

Not:

```text
Redis
   |
   | Source of truth
   ↓
PostgreSQL
```

unless a specific architecture deliberately uses Redis as a durable primary data store with the appropriate durability and recovery design.

---

# 31. Cache Invalidation

Suppose product price changes:

```text
PostgreSQL
    |
    ↓
Price updated
```

The old Redis value may remain.

We need a cache invalidation strategy.

For example:

```text
Update PostgreSQL
       ↓
Invalidate Redis key
```

or:

```text
Update PostgreSQL
       ↓
Publish event
       ↓
Consumers invalidate/update cache
```

---

# 32. Redis Failure and Stale Data

During recovery, you might choose:

```text
Serve stale cache
```

for some noncritical information.

For example:

```text
Product recommendations
```

might tolerate stale information.

But:

```text
Payment status
Inventory availability
Order status
```

may require much stricter consistency.

So fallback behavior must be **business-specific**.

---

# 33. DevOps vs Application Team

This is similar to your previous DBA question.

### DevOps/Platform

Usually responsible for:

- ElastiCache provisioning
- Terraform
- networking
- security groups
- monitoring
- alerts
- infrastructure health
- IAM
- Secrets Manager/ESO
- capacity
- failover architecture
- incident coordination

### Application team

Usually responsible for:

- Redis client
- cache logic
- TTL
- cache keys
- serialization
- fallback behavior
- retry logic
- circuit breaker
- cache invalidation

### DBA/Data team

May help with:

- RDS impact
- database load
- query performance
- database protection

---

# 34. What DevOps Does During Redis Incident

You don't start modifying application Java code during a live incident unless your role includes that and the change is approved.

Your immediate job is:

```text
1. Confirm impact
2. Check Redis health
3. Check network
4. Check application errors
5. Check RDS impact
6. Stop cascading failure
7. Coordinate with application team
8. Coordinate with DBA if RDS is affected
9. Restore infrastructure
10. Validate
11. Monitor
12. RCA
```

---

# 35. Example Incident

### 10:00

Normal:

```text
Redis hit ratio: 92%
RDS connections: 120
API P95: 150ms
```

### 10:05

Redis starts failing.

```text
Redis errors ↑
```

### 10:06

Cache hits:

```text
92% → 5%
```

### 10:07

Fallback begins.

```text
RDS traffic ↑
```

### 10:08

RDS:

```text
Connections:
120 → 350
```

### 10:09

Hikari:

```text
Pending ↑
```

### 10:10

API:

```text
P95 ↑
5xx ↑
```

Now the incident commander recognizes:

```text
Redis failure
+
RDS cascading failure
```

---

# 36. Immediate Response

The team:

```text
Stop unnecessary background workloads
        ↓
Limit fallback concurrency
        ↓
Reduce retry storm
        ↓
Protect RDS
        ↓
Investigate Redis
```

Redis infrastructure is restored.

Then:

```text
Redis
 ↓
Cache repopulation
 ↓
Cache hit ratio ↑
 ↓
RDS load ↓
 ↓
Hikari pending ↓
 ↓
API latency ↓
```

---

# 37. Observability Dashboard

Create:

```text
Redis Production Dashboard
```

Include:

### Redis

```text
CPU
Memory
Connections
Latency
Cache hits
Cache misses
Evictions
Network
Replication/failover status
```

### Application

```text
Redis errors
Redis command latency
Redis pool active
Redis pool pending
Fallback requests
Cache hit ratio
```

### Database

```text
RDS connections
DB load
CPU
Latency
Hikari pending
```

### Business

```text
Product API latency
Cart API latency
Order API latency
Checkout success rate
```

---

# 38. Important Alerts

Create alerts for:

```text
Redis unavailable
```

```text
Redis command latency high
```

```text
Redis connection errors high
```

```text
Cache hit ratio unexpectedly low
```

```text
Redis memory high
```

```text
Redis evictions high
```

```text
Application Redis fallback rate high
```

And most importantly:

```text
Redis fallback + RDS load high
```

That catches the cascading failure.

---

# 39. Runbook

Create:

```text
runbooks/redis-failure.md
```

Structure:

```markdown
# Redis Failure

## Symptoms

## Customer Impact

## Detection

## Redis Health Checks

## Kubernetes Checks

## DNS Checks

## Network Checks

## Application Checks

## RDS Impact

## Immediate Mitigation

## Cache Fallback

## Cache Stampede Protection

## Redis Recovery

## Cache Warming

## Validation

## RCA

## Prevention
```

---

You're right. **Chapter 153 should include a concrete production RCA example**, not just an RCA template. Since these chapters are meant to be practical, we should show exactly how the incident is documented from **symptom → investigation → root cause → contributing factors → corrective actions**.

Add the following section to **`153-Redis-Failure.md`**, preferably after the incident scenario/recovery sections and before the final checklist.

---

# 39. Example Production RCA — Redis Failure

## Incident Title

```text
Redis Failure Caused Increased RDS Load and API Latency
```

## Incident ID

```text
INC-2026-153
```

## Severity

```text
SEV-2
```

## Date

```text
2026-10-04
```

## Duration

```text
10:05 IST – 10:24 IST
```

## Affected Services

```text
catalog-service
product-service
cart-service
order-service
```

---

## 39.1 Customer Impact

Customers experienced:

- Slow product pages
- Slow cart operations
- Increased API latency
- Some HTTP 5xx responses
- Intermittent order failures

Example:

```text
Product API P95:
180ms → 4.8s
```

```text
Cart API P95:
250ms → 5.2s
```

```text
HTTP 5xx:
0.3% → 6.5%
```

No permanent order or payment data was lost because PostgreSQL remained the system of record.

---

# 39.2 Detection

The incident was detected by:

```text
Alert:
Redis command latency above threshold
```

Shortly afterward:

```text
Redis connection errors ↑
Cache hit ratio ↓
RDS DatabaseConnections ↑
API latency ↑
HTTP 5xx ↑
```

This indicated that Redis failure was beginning to affect PostgreSQL.

---

# 39.3 Timeline

```text
10:00
System operating normally.

Redis cache hit ratio:
92%

RDS connections:
125

Product API P95:
180ms
```

```text
10:05
Redis latency starts increasing.
```

```text
10:06
Application Redis connection errors begin.
```

```text
10:07
Redis cache hit ratio:
92% → 15%
```

```text
10:08
Applications begin falling back to PostgreSQL.
```

```text
10:09
RDS connections:
125 → 280
```

```text
10:10
Database load increases.

Hikari pending connections begin increasing.
```

```text
10:11
Product API P95:
180ms → 2.5s
```

```text
10:12
HTTP 5xx increases to 4%.
```

```text
10:13
Incident declared SEV-2.
```

```text
10:14
DevOps confirms Redis endpoint/network connectivity
and ElastiCache health is degraded.
```

```text
10:16
Application team confirms Redis fallback is working,
but fallback traffic is too high.
```

```text
10:17
Noncritical reporting/background workloads reduced.
```

```text
10:18
Fallback concurrency reduced to protect RDS.
```

```text
10:20
Redis recovery/failover completes.
```

```text
10:21
Redis connections recover.
```

```text
10:22
Cache hit ratio begins increasing.
```

```text
10:23
RDS connections decrease.
```

```text
10:24
API latency and 5xx return to normal.
```

---

# 39.4 Initial Investigation

DevOps checked:

```bash
kubectl get pods -n ecommerce
```

Application logs showed:

```text
RedisConnectionException
Redis command timeout
Unable to connect to Redis
```

The next question was:

> Is this a Redis problem or an application/network problem?

---

# 39.5 DNS Check

From an application Pod:

```bash
kubectl exec -it <pod> -n ecommerce -- \
  nslookup <redis-endpoint>
```

Result:

```text
Redis hostname resolved successfully.
```

Therefore:

```text
CoreDNS
   ↓
Healthy
```

---

# 39.6 Network Connectivity Check

```bash
kubectl exec -it <pod> -n ecommerce -- \
  nc -vz <redis-endpoint> 6379
```

Result:

```text
Connection timeout
```

Further investigation confirmed the application Pods could resolve the Redis endpoint but could not establish a successful Redis connection.

The team then checked:

- Security Groups
- NetworkPolicy
- VPC routing
- ElastiCache status
- Redis events

---

# 39.7 Redis Investigation

The infrastructure team found that the Redis deployment was experiencing a managed failover event.

During the event:

```text
Redis endpoint
      ↓
Temporary connection failures
      ↓
Application Redis errors
```

Redis eventually recovered through the configured failover mechanism.

---

# 39.8 Why Did the Incident Become Larger?

Redis failure alone should have produced:

```text
Redis failure
     ↓
Cache miss
     ↓
PostgreSQL lookup
```

But the traffic volume was high.

For example:

```text
Normal:

1,000 requests/sec
↓
Redis:
920 cache hits
80 DB requests
```

During Redis failure:

```text
1,000 requests/sec
↓
Redis unavailable
↓
1,000 DB requests
```

Database traffic increased:

```text
80 RPS → 1,000 RPS
```

That's:

```text
12.5× increase
```

---

# 39.9 RDS Investigation

The DBA checked PostgreSQL.

Before the incident:

```text
DatabaseConnections:
125
```

During the incident:

```text
DatabaseConnections:
380
```

The DBA confirmed:

```text
PostgreSQL itself was healthy.
```

There were no major:

- Locking problems
- Database failures
- Storage problems
- Database corruption issues

The database was being overloaded by **application fallback traffic**.

---

# 39.10 Hikari Investigation

DevOps checked application metrics.

Before:

```text
order-service Pods:
10

Hikari active:
35

Hikari pending:
0
```

During incident:

```text
order-service Pods:
15

Hikari active:
145

Hikari pending:
32
```

The increased PostgreSQL traffic caused more application threads to wait for database connections.

---

# 39.11 Root Cause

### Primary Root Cause

```text
Redis became temporarily unavailable during a
managed failover event.
```

This caused:

```text
Redis failure
     ↓
Cache misses
     ↓
Fallback to PostgreSQL
     ↓
Large increase in database traffic
```

---

# 39.12 Contributing Factors

The incident became customer-visible because of several contributing factors.

### 1. High cache dependency

A large percentage of product/catalog reads depended on Redis.

### 2. Fallback amplification

Redis failure caused a large increase in PostgreSQL requests.

### 3. Insufficient fallback rate limiting

The application allowed too much fallback traffic to reach PostgreSQL.

### 4. Cache stampede risk

Multiple Pods attempted to reload the same popular keys.

### 5. Hikari connection pressure

More PostgreSQL requests consumed the application connection pools.

### 6. HPA amplification

Higher request processing pressure caused additional Pods to be created.

### 7. Insufficient Redis-failure alerting

Redis errors were detected, but the **Redis fallback → RDS pressure** relationship was not alerted early enough.

---

# 39.13 What Went Well

```text
✓ PostgreSQL remained the source of truth.

✓ No permanent order data was lost.

✓ Redis failover eventually restored service.

✓ Prometheus/Grafana showed the relationship
  between Redis failure and RDS load.

✓ ELK provided application Redis errors.

✓ The incident team reduced noncritical workloads.

✓ RDS was protected before complete connection exhaustion.

✓ Business validation confirmed order processing
  after recovery.
```

---

# 39.14 What Went Wrong

```text
✗ Redis fallback generated excessive database traffic.

✗ Fallback concurrency was not sufficiently bounded.

✗ Cache stampede protection was incomplete.

✗ Redis failure did not immediately trigger
  an RDS-protection response.

✗ HPA scaling increased database pressure.

✗ Redis fallback metrics were not part of the
  primary production dashboard.
```

---

# 39.15 Immediate Corrective Actions

### Action 1 — Protect RDS

Implemented a controlled fallback limit.

```text
Redis unavailable
      ↓
Fallback requests
      ↓
Rate/concurrency limit
      ↓
PostgreSQL
```

---

### Action 2 — Reduce Retry Storm

Redis retries were changed from:

```text
Immediate retry
Immediate retry
Immediate retry
```

to:

```text
Retry
 ↓
Exponential backoff
 ↓
Jitter
 ↓
Bounded attempts
```

---

### Action 3 — Circuit Breaker

Added/verified circuit breaker behavior:

```text
Redis failures
     ↓
Circuit OPEN
     ↓
Stop repeatedly calling Redis
     ↓
Controlled fallback
```

---

### Action 4 — Cache Stampede Protection

Implemented request coalescing for high-demand keys.

Instead of:

```text
100 requests
    ↓
100 PostgreSQL queries
```

we target:

```text
100 requests
    ↓
1 database query
    ↓
Redis
    ↓
100 responses
```

where the business/data pattern permits it.

---

# 39.16 Long-Term Corrective Actions

| Action | Owner | Priority |
|---|---|---|
| Add Redis fallback metric | Platform | High |
| Add Redis → RDS correlation dashboard | SRE | High |
| Add fallback rate limiting | Backend | High |
| Review Redis timeout | Backend | High |
| Review retry/backoff | Backend | High |
| Add cache stampede protection | Backend | High |
| Review HPA max replicas | Platform | Medium |
| Add Redis failure game day | SRE | Medium |
| Test Redis failover | Platform | High |
| Document Redis runbook | DevOps | High |
| Add Redis dependency to capacity planning | Platform | Medium |

---

# 39.17 Prevention Architecture

After the RCA, the target architecture becomes:

```text
                  Application
                       |
                 +-----+-----+
                 |           |
                 ↓           ↓
              Redis       PostgreSQL
                 |
                 X
              Failure
                 |
                 ↓
          Short Timeout
                 |
                 ↓
          Circuit Breaker
                 |
                 ↓
       Controlled Fallback
                 |
                 ↓
        Rate/Concurrency Limit
                 |
                 ↓
             PostgreSQL
```

The important protection is:

```text
Redis failure
     ↓
Controlled degradation
```

not:

```text
Redis failure
     ↓
Unlimited PostgreSQL traffic
```

---

# 39.18 RCA Root Cause Statement

The final RCA should have a concise statement:

> **A temporary Redis failover caused application Redis connections to fail. The affected services fell back to PostgreSQL, resulting in a significant increase in database traffic and Hikari connection utilization. HPA scaling and insufficient fallback concurrency controls amplified the load, causing elevated API latency and HTTP 5xx responses. PostgreSQL remained healthy and no persistent business data was lost.**

---

# 39.19 Five Whys

### Why did customers experience high API latency?

Because application requests were waiting for database resources.

### Why were database resources heavily utilized?

Because Redis failures caused cache misses and fallback requests to PostgreSQL.

### Why did fallback traffic become excessive?

Because fallback concurrency and cache stampede protection were insufficient.

### Why did the application scale further?

HPA responded to increased workload/resource pressure, adding more Pods and therefore more database demand.

### Why wasn't this prevented?

Our capacity model and alerts did not sufficiently account for the **Redis failure → cache miss → PostgreSQL load → Hikari → HPA** failure chain.

---

# 39.20 Final RCA Lesson

The important lesson isn't:

> "Redis failed."

The deeper lesson is:

```text
Redis failure
     ↓
Application fallback
     ↓
Database dependency
     ↓
Connection pool
     ↓
Kubernetes scaling
     ↓
Customer impact
```

A production RCA must therefore identify **both the original failure and the amplification mechanism**.

In this example:

```text
Primary failure:
Redis failover/connectivity issue

Amplification:
Uncontrolled fallback traffic

Secondary pressure:
PostgreSQL + Hikari

Customer impact:
High latency + 5xx
```

---

# 40. Production Decision Tree

```text
Redis alert
    |
    ↓
Application impacted?
    |
   YES
    |
    ↓
Redis endpoint resolves?
    |
  NO → DNS investigation
    |
   YES
    |
    ↓
TCP connection works?
    |
  NO → Network/SG investigation
    |
   YES
    |
    ↓
Redis commands succeed?
    |
  NO → Redis health/failover investigation
    |
   YES
    |
    ↓
Application client/pool issue?
    |
   YES → Application investigation
    |
    NO
    |
    ↓
Fallback active?
    |
   NO → Enable/restore fallback if designed
    |
   YES
    |
    ↓
RDS load increasing?
    |
   YES
    |
    ↓
Protect RDS
    |
    ↓
Restore Redis
    |
    ↓
Warm cache carefully
    |
    ↓
Validate business flow
    |
    ↓
RCA
```

---

# 41. Production Prevention Checklist

## Redis

- [ ] Multi-AZ/failover architecture where appropriate
- [ ] Private networking
- [ ] Encryption enabled
- [ ] Authentication configured
- [ ] Monitoring enabled
- [ ] Capacity tested
- [ ] Memory limits understood
- [ ] Eviction policy reviewed
- [ ] Backup/recovery strategy defined where needed

## Application

- [ ] Short Redis timeouts
- [ ] Bounded retries
- [ ] Exponential backoff
- [ ] Jitter
- [ ] Circuit breaker where appropriate
- [ ] Database fallback where appropriate
- [ ] Cache stampede protection
- [ ] Cache invalidation strategy
- [ ] Correct TTLs

## PostgreSQL protection

- [ ] Connection budget
- [ ] Hikari limits
- [ ] HPA limits
- [ ] Fallback concurrency limits
- [ ] RDS monitoring
- [ ] Rate limiting
- [ ] Backpressure

## Incident management

- [ ] Redis runbook
- [ ] DBA escalation path
- [ ] Application team escalation path
- [ ] Redis recovery tested
- [ ] Cache warming procedure
- [ ] Business validation procedure

---

# 42. Final Mental Model

Remember this:

```text
                  APPLICATION
                       |
                 +-----+-----+
                 |           |
                 ↓           ↓
              Redis       PostgreSQL
              CACHE       SOURCE OF TRUTH
                 |
                 X
              FAILURE
                 |
                 ↓
            FALLBACK
                 |
                 ↓
            PostgreSQL
                 |
                 ↓
          Protect database
```

The dangerous architecture is:

```text
Redis failure
     ↓
Every request → PostgreSQL
     ↓
RDS overloaded
     ↓
Hikari exhausted
     ↓
Application outage
```

The production architecture should instead be:

```text
Redis failure
     ↓
Detect quickly
     ↓
Short timeout
     ↓
Controlled fallback
     ↓
Rate limit/backpressure
     ↓
Protect PostgreSQL
     ↓
Restore Redis
     ↓
Warm cache gradually
     ↓
Validate
```

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
153 — Redis Failure                    ✅
154 — SQS Backlog                      ⏭️ NEXT
155 — Kafka Consumer Lag
156 — Node Failure
157 — AZ Failure
158 — ArgoCD Drift
159 — Bad Production Deployment
160 — Security Incident
```

**Next: `154-SQS-Backlog.md` — how DevOps detects an SQS backlog, distinguishes producer/consumer problems, investigates visibility timeout and DLQ behavior, handles KEDA scaling, protects downstream services, and performs controlled message recovery.**