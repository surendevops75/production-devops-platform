# 111-Redis-Failure-Cache-Recovery.md

````markdown
# 111 — Redis Failure & Cache Recovery

## 1. Purpose

This chapter tests what happens when Redis becomes unavailable in our production-style e-commerce platform.

The objective is not simply to prove that Redis can fail over.

We need to prove that the application can survive a cache failure without turning the Redis incident into an RDS incident.

The scenario is:

Traffic
   ↓
EKS
   ↓
Application
   ↓
Redis
   X
Redis failure
   ↓
Cache misses / fallback
   ↓
RDS load increases
   ↓
Protection mechanisms
   ↓
Redis recovery
   ↓
Cache repopulation
   ↓
Normal operation

We will test:

- Redis failure
- Redis timeout behavior
- cache fallback
- RDS impact
- connection pool behavior
- cache stampede
- circuit breaker behavior
- Redis recovery
- cache warming
- monitoring
- alerts
- recovery validation
- incident response

---

# 2. Why Redis Failure Is Dangerous

Redis is often introduced as:

> "Just a cache."

But Redis can significantly reduce database traffic.

Normal:

```text
10,000 requests
      |
      v
Redis
      |
      +---- 9,000 HIT
      |
      +---- 1,000 MISS
                  |
                  v
                 RDS
````

If Redis disappears:

```text
10,000 requests
      |
      X
    Redis
      |
      v
10,000 potential database requests
      |
      v
RDS
```

This can create a sudden database load increase.

---

# 3. Failure Cascade

The dangerous sequence is:

```text
Redis failure
     ↓
Cache misses
     ↓
RDS traffic increases
     ↓
RDS CPU increases
     ↓
RDS latency increases
     ↓
Application latency increases
     ↓
HPA scales Pods
     ↓
More DB connections
     ↓
RDS becomes overloaded
```

Therefore:

> Redis failure protection must include RDS protection.

---

# 4. Normal Architecture

```text
                    Internet
                       |
                       v
                      ALB
                       |
                       v
                      EKS
                       |
                order-service
                  /        \
                 /          \
                v            v
             Redis          RDS
          ElastiCache     PostgreSQL
```

Normal request:

```text
Request
   |
   v
Redis
   |
   +---- HIT ---> Response
   |
   +---- MISS --> RDS --> Redis --> Response
```

---

# 5. Failure Architecture

During Redis failure:

```text
                 order-service
                 /           \
                /             \
               X               v
            Redis              RDS
          unavailable       fallback
```

The application should preferably degrade from:

```text
Fast cached response
```

to:

```text
Database-backed response
```

rather than:

```text
Entire service unavailable
```

for data that can safely be fetched from the source of truth.

---

# 6. Important Distinction

Not every Redis workload should fall back to RDS.

For example:

```text
Product cache
    ↓
RDS fallback
```

may be reasonable.

But:

```text
Rate limiter
    ↓
Redis unavailable
```

may require a different strategy.

Likewise:

```text
Session state
```

may require application-specific behavior.

Therefore define fallback behavior per Redis use case.

---

# 7. Failure Classification

We should distinguish:

### Redis completely unavailable

```text
Connection refused
Timeout
Endpoint unavailable
```

### Redis slow

```text
Connection succeeds
but commands are slow
```

### Redis partially unhealthy

```text
Some operations succeed
Some fail
```

### Redis failover

```text
Primary changes
connections temporarily fail
new connections recover
```

Each scenario should be tested.

---

# 8. Pre-Test Environment

Start with:

```text
DEV
```

then:

```text
QA
```

then:

```text
STAGING
```

Production failure testing requires:

```text
Approved change
Test window
Monitoring
Incident owner
Rollback/recovery plan
Business approval
```

Never intentionally break a production Redis cluster without explicit authorization.

---

# 9. Baseline Before Failure

Record:

```text
Redis:
- memory
- CPU
- connections
- hit ratio
- misses
- commands/sec
- latency

Application:
- request rate
- P95
- P99
- error rate
- Redis errors
- Redis fallback count

RDS:
- CPU
- connections
- read latency
- write latency
- IOPS
```

---

# 10. Verify Redis Health

Check the ElastiCache status using the AWS console or CLI appropriate to the selected Redis/Valkey-compatible deployment.

Conceptually:

```bash
aws elasticache describe-cache-clusters
```

For replication-group based deployments, inspect:

```bash
aws elasticache describe-replication-groups
```

Verify:

```text
available
```

and confirm the expected nodes/replicas.

The exact CLI response depends on the ElastiCache deployment model.

---

# 11. Baseline Cache Hit Ratio

Suppose:

```text
Hits   = 950,000
Misses = 50,000
```

Then:

```text
Hit Ratio =
950,000
---------
1,000,000

= 95%
```

Record the baseline.

---

# 12. Baseline RDS Load

Suppose before the test:

```text
RDS CPU:
35%

RDS connections:
120

Read latency:
8 ms

Application P95:
180 ms
```

These numbers become the comparison point.

---

# 13. Baseline Application Test

Test:

```bash
curl -f \
  https://api.example.com/api/v1/products/1001
```

Record:

```text
HTTP status
Latency
Response
Timestamp
```

Repeat multiple times.

The first request may be a cache miss.

Subsequent requests should normally become cache hits.

---

# 14. Verify Cache Hit Behavior

Conceptually:

```text
Request 1
   ↓
Redis MISS
   ↓
RDS
   ↓
Redis SET

Request 2
   ↓
Redis HIT
   ↓
Response
```

Observe the application metrics.

Expected:

```text
Redis hits increase
RDS read traffic remains lower
```

---

# 15. Start Controlled Traffic

For a test environment:

```text
10 req/sec
```

or another controlled load appropriate to the environment.

We want enough traffic to observe the impact without creating unrelated overload.

---

# 16. Trigger Redis Failure

There are multiple ways to test Redis failure.

### Method 1

Use a controlled ElastiCache failover operation.

### Method 2

Temporarily block connectivity from a test application environment.

### Method 3

Use a test Redis endpoint/configuration that is intentionally unavailable.

### Method 4

Use a controlled maintenance/failure simulation.

The safest initial exercise is usually to simulate application connectivity failure without modifying production infrastructure.

---

# 17. Application-Side Failure Injection

For learning, we can temporarily change:

```text
REDIS_HOST
```

to an intentionally invalid endpoint in DEV/QA.

Example:

```text
REDIS_HOST=redis-does-not-exist.internal
```

Then redeploy the test workload.

This tests:

```text
DNS failure
Redis timeout
Fallback
Application behavior
```

without destroying Redis infrastructure.

---

# 18. Controlled Network Failure

Another test can block the application-to-Redis path in a test namespace/environment.

Conceptually:

```text
order-service
      |
      X
Redis
```

This is useful for testing:

```text
Network timeout
Connection errors
Fallback
Circuit breaker
```

Use a reversible NetworkPolicy or approved test mechanism.

Do not accidentally block production traffic.

---

# 19. Observe the Application

Immediately watch:

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200 \
  -f
```

Look for:

```text
RedisConnectionException
timeout
connection refused
Redis command failure
fallback
```

---

# 20. Expected Application Behavior

A resilient application might log:

```text
Redis unavailable
Falling back to PostgreSQL
```

and continue serving requests.

A bad implementation might:

```text
Redis timeout
   ↓
HTTP request waits 30 seconds
   ↓
HTTP 500
```

for every request.

This defeats the purpose of using Redis as a cache.

---

# 21. Redis Timeout Is Critical

Suppose Redis timeout is:

```text
30 seconds
```

Then every cache operation could potentially block the request for a long time.

If:

```text
1000 requests
```

hit the failed Redis connection:

```text
1000 requests
   ↓
30-second waits
   ↓
Thread/connection pressure
   ↓
Application degradation
```

Therefore Redis timeout should be short enough for the application's latency budget.

The exact value must be load-tested.

---

# 22. Cache Fallback

Conceptually:

```java
public Product getProduct(String productId) {

    String key = "product:v1:" + productId;

    try {
        Product cached = redisTemplate
                .opsForValue()
                .get(key);

        if (cached != null) {
            return cached;
        }

    } catch (Exception e) {
        metrics.incrementRedisFallback();
    }

    Product product = productRepository
            .findById(productId)
            .orElseThrow();

    try {
        redisTemplate.opsForValue()
                .set(
                    key,
                    product,
                    Duration.ofMinutes(10)
                );
    } catch (Exception e) {
        metrics.incrementRedisWriteFailure();
    }

    return product;
}
```

The important principle is:

```text
Redis failure should not necessarily prevent
the source-of-truth operation.
```

---

# 23. Do Not Hide All Redis Errors

This is bad:

```java
catch (Exception e) {
}
```

It silently hides failures.

Instead:

```text
Log appropriately
Increment metric
Record failure reason
Use fallback
```

Do not log secrets or excessive sensitive payloads.

---

# 24. Cache Fallback Metric

Create a metric such as:

```text
redis_fallback_total
```

Example:

```text
redis_fallback_total{
    service="order-service",
    cache="product"
}
```

This lets us detect:

```text
Redis failure
       ↓
Fallback increasing
```

---

# 25. RDS Impact

Once Redis fails:

```text
Redis MISS
   ↓
RDS
```

RDS read traffic should increase.

Monitor:

```text
RDS CPU
Connections
Read IOPS
Read latency
Throughput
```

---

# 26. Example Failure Timeline

```text
10:00
Redis healthy

10:01
Cache hit ratio = 95%

10:02
Redis failure injected

10:02:05
Redis errors increase

10:02:10
Fallback requests increase

10:02:20
RDS read traffic increases

10:02:30
RDS CPU increases

10:03
Application latency increases
```

The exact values come from the actual test.

---

# 27. Why RDS May Not Immediately Fail

Suppose normal traffic is:

```text
10,000 req/sec
```

with:

```text
95% cache hit
```

Approximately:

```text
5000? 
```

No.

Correctly:

```text
10,000 × 5%
=
500 RDS reads/sec
```

If Redis fails:

```text
10,000 × 100%
=
10,000 potential RDS reads/sec
```

That is:

```text
20×
```

the previous cache-miss load.

This illustrates why cache failure can be dangerous.

---

# 28. RDS Protection

We need several protections.

### Connection pool limits

```text
Prevent unlimited DB connections
```

### HPA limits

```text
Prevent uncontrolled Pod scaling
```

### Query optimization

```text
Reduce DB work per request
```

### Rate limiting

```text
Prevent excessive traffic
```

### Circuit breakers

```text
Prevent repeated failing calls
```

### Graceful degradation

```text
Reduce non-essential operations
```

---

# 29. Cache Stampede During Failure Recovery

Suppose Redis comes back empty.

Then:

```text
100,000 requests
      ↓
Redis
      ↓
MISS
      ↓
RDS
```

This can overload RDS.

This is sometimes called a cache stampede or cold-cache surge.

Therefore Redis recovery itself is a traffic event.

---

# 30. Cache Warming

After Redis recovery, selected keys can be loaded gradually.

Example:

```text
Top 100 products
Top categories
Frequently accessed configuration
```

Process:

```text
RDS
 ↓
Controlled read
 ↓
Redis SET
```

Do not immediately warm millions of keys at maximum speed.

---

# 31. Rate-Limited Warming

Conceptually:

```text
100 keys/sec
```

rather than:

```text
1,000,000 keys/sec
```

This protects RDS.

The appropriate warming rate depends on:

```text
RDS capacity
Redis capacity
Object size
Traffic
Business priority
```

---

# 32. Request Coalescing

Suppose:

```text
product:1001
```

is missing.

Instead of:

```text
1000 requests
     ↓
1000 RDS queries
```

we want:

```text
1000 requests
      ↓
One database fetch
      ↓
Redis SET
      ↓
All requests receive result
```

This is request coalescing.

---

# 33. Distributed Lock

A distributed lock can be used carefully for cache population.

Conceptually:

```text
Request A
   ↓
Acquire lock
   ↓
Read RDS
   ↓
Populate Redis
   ↓
Release lock

Request B
   ↓
Wait / re-check
```

Locks introduce their own failure modes.

Do not use distributed locking without understanding:

```text
Lock timeout
Owner failure
Deadlock
Retry
Lease expiration
```

---

# 34. Stale-While-Revalidate

Another strategy:

```text
Cached data exists but is slightly stale
        |
        v
Return cached data
        |
        v
Refresh asynchronously
```

This can avoid a large synchronous cache-miss spike.

It is particularly useful for data where a small amount of staleness is acceptable.

---

# 35. Not All Data Can Use Stale Data

Do not apply stale-while-revalidate blindly to:

```text
Payment status
Inventory availability
Fraud decision
Financial balance
```

For these workloads, consistency requirements may be stricter.

---

# 36. Circuit Breaker

Suppose Redis is unavailable.

Without a circuit breaker:

```text
Request
 ↓
Redis
 ↓
Timeout
 ↓
Request
 ↓
Redis
 ↓
Timeout
```

This continues for every request.

With a circuit breaker:

```text
Redis failures
      ↓
Threshold reached
      ↓
Circuit OPEN
      ↓
Skip Redis
      ↓
Use fallback
```

This reduces repeated timeout overhead.

---

# 37. Circuit States

Typical model:

```text
CLOSED
  |
  | failures
  v
OPEN
  |
  | wait
  v
HALF-OPEN
  |
  +---- success ----> CLOSED
  |
  +---- failure ----> OPEN
```

The exact library and configuration should be chosen according to the application platform.

---

# 38. Retry + Circuit Breaker

Avoid:

```text
Circuit breaker
+
10 retries
+
30-second timeout
```

per request.

That can create:

```text
Huge request delay
```

A better pattern is:

```text
Short timeout
+
Small retry count
+
Exponential backoff
+
Circuit breaker
```

when retries are actually appropriate.

---

# 39. Retry Jitter

Without jitter:

```text
1000 requests
     ↓
retry after 1 second
     ↓
1000 requests simultaneously
```

With jitter:

```text
1000 requests
     ↓
slightly different retry times
     ↓
spread load
```

This reduces synchronized retry spikes.

---

# 40. Redis Recovery

When Redis becomes available:

```text
Redis healthy
    ↓
Circuit half-open
    ↓
Test request succeeds
    ↓
Circuit closes
    ↓
Cache operations resume
```

The application should not assume recovery until it has successfully completed a real operation.

---

# 41. Verify Redis Recovery

Check:

```text
Redis status
Connections
Memory
Commands/sec
Latency
Errors
```

Then application:

```text
Redis errors ↓
Fallback ↓
Cache hits ↑
RDS reads ↓
```

This should be visible in the metrics.

---

# 42. Recovery Timeline

Example:

```text
10:02:00
Redis failure

10:02:05
Circuit opens

10:02:10
Application uses RDS fallback

10:03:00
Redis recovered

10:03:05
Circuit half-open

10:03:06
Test succeeds

10:03:07
Circuit closes

10:03:10
Cache population begins

10:04:00
Hit ratio begins recovering
```

These are example timestamps.

---

# 43. Cache Warmup Strategy

A controlled sequence:

```text
Redis available
     ↓
Warm highest-value keys
     ↓
Monitor RDS
     ↓
Increase warmup rate
     ↓
Normal traffic repopulates remaining keys
```

Do not attempt to rebuild the entire cache blindly.

---

# 44. RDS Protection During Warmup

Monitor:

```text
RDS CPU
RDS connections
Read IOPS
Read latency
```

If these exceed the safe operating range:

```text
Slow/stop warming
```

This is important.

---

# 45. Cache Recovery Validation

Validate:

```text
[ ] Redis available
[ ] Application can connect
[ ] Authentication works
[ ] Cache writes work
[ ] Cache reads work
[ ] Hit ratio increasing
[ ] Fallback decreasing
[ ] RDS load decreasing
[ ] Application latency normal
[ ] Error rate normal
```

---

# 46. Redis Failover vs Redis Failure

These are different tests.

### Failover

```text
Primary
  X
Replica promoted
```

Redis service remains available with a temporary interruption.

### Complete service failure

```text
Redis
  X
```

Application must use fallback.

Test both.

---

# 47. Redis Failover Test

For an appropriate ElastiCache replication group, perform a controlled failover according to AWS's currently supported operational procedure.

The test should measure:

```text
Failover start
Primary unavailable
Replica promoted
New connections
Application recovery
Error rate
```

Do not assume the failover duration.

Measure it.

---

# 48. Redis Failure Injection Test

For a safe application-level test:

```text
Change test REDIS_HOST
       ↓
Application cannot connect
       ↓
Observe fallback
       ↓
Restore correct REDIS_HOST
       ↓
Observe recovery
```

This tests application resilience without requiring an actual Redis infrastructure failure.

---

# 49. Redis Network Failure

Another test:

```text
order-service
      |
      X
      |
Redis
```

Observe:

```text
DNS
TCP
TLS
timeout
fallback
circuit breaker
```

This is useful because many production Redis incidents are connectivity-related rather than actual Redis process failures.

---

# 50. Security Group Failure

In a controlled environment, temporarily remove the application-to-Redis inbound rule.

Result:

```text
EKS
  |
  X
Redis SG
```

Application should:

```text
Detect failure
Fallback
Generate metrics
Avoid excessive retries
```

Restore the rule afterward.

---

# 51. Redis Failure Incident

Example:

```text
ALERT:
Redis availability degraded
```

Start:

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

Look for:

```text
Redis timeout
Connection refused
DNS error
Authentication error
```

---

# 52. Check Redis Endpoint

From an approved debugging environment:

```bash
getent hosts "$REDIS_HOST"
```

Then:

```bash
nc -vz "$REDIS_HOST" "$REDIS_PORT"
```

If TLS is required, use an appropriate TLS-aware client.

---

# 53. Check Application Metrics

Look at:

```text
redis_errors_total
redis_fallback_total
redis_hit_total
redis_miss_total
```

The exact metric names depend on how the application instrumentation is implemented.

---

# 54. Check RDS

Because Redis failure may increase RDS load:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Also inspect:

```text
CPU
connections
latency
IOPS
```

This is a critical part of the incident.

---

# 55. Check HPA

```bash
kubectl get hpa \
  -n ecommerce-prod
```

If HPA has scaled from:

```text
5 Pods
```

to:

```text
20 Pods
```

database connection pressure may have increased substantially.

---

# 56. HPA During Redis Failure

Potential chain:

```text
Redis failure
   ↓
Requests become slower
   ↓
CPU increases
   ↓
HPA scales
   ↓
More Pods
   ↓
More RDS connections
   ↓
RDS pressure
```

Therefore CPU-based HPA can amplify a dependency failure.

This is why:

```text
HPA
+
database capacity
+
cache fallback
```

must be designed together.

---

# 57. Rate Limiting During Redis Failure

If traffic becomes too high for the fallback database path:

```text
Redis unavailable
      ↓
RDS capacity limited
      ↓
Rate limit requests
```

Possible behavior:

```text
Normal traffic
    ↓
Serve normally

Excess traffic
    ↓
429 Too Many Requests
```

This is preferable to allowing uncontrolled traffic to take down the entire service.

---

# 58. Graceful Degradation

For example:

If product recommendations depend on Redis:

```text
Recommendation unavailable
        ↓
Return default recommendations
```

instead of:

```text
Entire product page = HTTP 500
```

For optional functionality, graceful degradation protects the core business path.

---

# 59. Critical vs Non-Critical Cache

Classify cached data.

### Critical path

```text
Order
Payment
Inventory
```

### Non-critical

```text
Recommendations
Popular products
Analytics cache
UI personalization
```

The failure behavior can differ.

For example:

```text
Recommendation Redis failure
       ↓
Show default recommendations
```

while:

```text
Payment state Redis failure
       ↓
Do not guess
       ↓
Use authoritative source
```

---

# 60. Redis Data Classification

Maintain a table:

| Data            | Source of Truth              | Redis Role                 | TTL           | Fallback          |
| --------------- | ---------------------------- | -------------------------- | ------------- | ----------------- |
| Product         | PostgreSQL                   | Cache                      | 10m           | RDS               |
| Customer        | PostgreSQL                   | Cache                      | 5m            | RDS               |
| Pricing         | PostgreSQL/service           | Cache                      | Short         | Source service    |
| Inventory       | Inventory service            | Carefully controlled cache | Short         | Inventory service |
| Recommendations | Recommendation system        | Cache                      | 30m           | Default response  |
| Session         | Defined by auth architecture | Session state              | Minutes/hours | App-specific      |

The actual values must come from business requirements.

---

# 61. Cache Failure Runbook

## Step 1

Confirm Redis alert.

## Step 2

Check Redis service status.

## Step 3

Check application Redis errors.

## Step 4

Check Redis DNS/connectivity.

## Step 5

Check RDS load.

## Step 6

Check HPA.

## Step 7

Determine whether fallback is functioning.

## Step 8

If RDS is approaching capacity:

```text
Control traffic
Reduce non-critical work
Review HPA
Protect database
```

## Step 9

Restore Redis.

## Step 10

Validate cache recovery.

## Step 11

Monitor RDS load returning toward baseline.

---

# 62. Do Not Immediately Flush Redis

A dangerous action:

```text
FLUSHALL
```

This deletes the entire Redis keyspace.

Never use it casually in production.

If Redis is a cache, losing the entire cache can cause a huge database surge.

If Redis contains state, the consequences may be much more serious.

---

# 63. Do Not Delete All Keys During an Incident

Even:

```text
SCAN
DEL
```

performed indiscriminately can create a large cache-miss wave.

If cache cleanup is required:

```text
Identify affected keys
Delete selectively
Rate-limit cleanup
Monitor RDS
```

---

# 64. Cache Warmup After Full Cache Loss

If the cache is completely empty:

```text
Cache hit ratio:
95% → 0%
```

Do not immediately expect:

```text
0% → 95%
```

Instead:

```text
0%
 ↓
10%
 ↓
25%
 ↓
50%
 ↓
75%
 ↓
95%
```

depending on traffic and warmup strategy.

---

# 65. Monitoring Recovery

The most useful graph is not only:

```text
Redis available
```

but:

```text
Redis availability
+
Cache hit ratio
+
RDS load
+
Application latency
```

Together they show whether the system actually recovered.

---

# 66. Example Recovery Graph

Conceptually:

```text
Cache Hit Ratio

100% |                 ________
 95% |                /
 75% |              /
 50% |            /
 25% |          /
  0% |________/
       Failure  Recovery
```

At the same time:

```text
RDS Read Load

Low  |________
     |        \
High |         \____
        Failure Recovery
```

The actual graph should come from monitoring data.

---

# 67. Incident Timeline Example

```text
14:00
Redis healthy

14:05
Redis failure detected

14:05:10
Application fallback activated

14:05:20
RDS read load increases

14:06
RDS remains within safe capacity

14:07
Redis recovery begins

14:07:10
Application reconnects

14:07:30
Cache warming begins

14:10
Hit ratio returns toward baseline

14:12
RDS load returns toward baseline

14:15
Incident resolved
```

---

# 68. RTO

Measure:

```text
Redis failure
        ↓
Application fallback available
```

and:

```text
Redis failure
        ↓
Redis service recovered
        ↓
Cache restored
```

These are different recovery objectives.

For example:

```text
Application degraded-mode recovery:
5 seconds

Redis infrastructure recovery:
60 seconds

Cache warmup:
5 minutes
```

These should be measured independently.

---

# 69. RPO

For a pure cache:

```text
RPO may be effectively irrelevant
```

because the source of truth is elsewhere.

For Redis-hosted business state:

```text
RPO becomes critical.
```

This distinction must be documented per workload.

---

# 70. Redis as Cache vs Redis as State

### Cache

```text
Redis data lost
      ↓
Rebuild from source
```

### Primary state

```text
Redis data lost
      ↓
Potential business data loss
```

The second architecture requires:

```text
Persistence
Backup
Recovery
Replication
DR
RPO
RTO
```

Do not mix the two designs.

---

# 71. Production Test Matrix

| Scenario                     | Expected Behavior                      |
| ---------------------------- | -------------------------------------- |
| Redis timeout                | Fast fallback                          |
| Redis DNS failure            | Fallback                               |
| Redis SG block               | Fallback                               |
| Redis primary failover       | Temporary errors, recovery             |
| Redis restart                | Reconnect                              |
| Empty cache                  | Controlled DB load                     |
| High cache miss              | RDS remains within capacity            |
| Redis recovery               | Cache repopulates                      |
| Circuit breaker open         | Redis calls bypassed                   |
| High traffic + Redis failure | Rate limiting/degradation protects RDS |

---

# 72. Production Checklist

## Before Test

```text
[ ] Approved change
[ ] Monitoring working
[ ] Baseline recorded
[ ] RDS capacity confirmed
[ ] Redis healthy
[ ] Application healthy
[ ] Recovery plan ready
```

## During Test

```text
[ ] Redis metrics
[ ] Application errors
[ ] Fallback metrics
[ ] RDS CPU
[ ] RDS connections
[ ] HPA replicas
[ ] Application latency
```

## Recovery

```text
[ ] Redis available
[ ] Application reconnects
[ ] Cache writes succeed
[ ] Cache hit ratio increases
[ ] RDS load decreases
[ ] Application latency normal
[ ] Business requests succeed
```

---

# 73. Final Architecture

```text
                         Internet
                            |
                            v
                           ALB
                            |
                            v
                         EKS
                            |
                    +-------+-------+
                    |               |
                    v               v
             order-service       other services
                    |
             +------+------+
             |             |
             v             v
          Redis            RDS
        ElastiCache     PostgreSQL
             |
             |
        CACHE HIT
             |
             v
          Response

Redis failure:

          Redis X
             |
             v
        RDS fallback
             |
             v
       Protected database
```

---

# 74. Complete Failure Chain

The most important production scenario is:

```text
Redis failure
     ↓
Cache miss
     ↓
Database fallback
     ↓
RDS load increase
     ↓
Monitor capacity
     ↓
Protect RDS
     ↓
Redis recovery
     ↓
Controlled cache warmup
     ↓
Hit ratio recovers
     ↓
RDS load decreases
     ↓
Application returns to normal
```

---

# 75. Key Takeaways

Remember:

```text
1. Redis is often a performance layer, not the source of truth.
2. Redis failure can significantly increase RDS traffic.
3. Cache fallback must be designed explicitly.
4. Redis timeouts should not consume application threads for too long.
5. Circuit breakers can prevent repeated failed cache calls.
6. Retries need limits, backoff, and jitter.
7. Cache recovery can create a stampede.
8. Cache warming must protect RDS.
9. HPA can amplify a Redis failure if fallback traffic increases CPU.
10. Redis failure testing must include downstream RDS impact.
11. Cache data and primary Redis state require different recovery designs.
12. Measure application recovery separately from infrastructure recovery.
```

---

# 76. Next Chapter

**112-SQS-Production-Implementation.md**

We now move from the caching layer to asynchronous messaging.

The architecture becomes:

```text
                    EKS
                     |
             order-service
                     |
                     v
                  SQS Queue
                     |
          +----------+----------+
          |                     |
          v                     v
   payment-service      notification-service
```

We will implement:

* SQS architecture
* Standard vs FIFO
* queue creation
* Terraform
* IAM
* KMS encryption
* producer
* consumer
* visibility timeout
* long polling
* message retention
* retries
* DLQ
* idempotency
* Kubernetes consumer deployment
* KEDA autoscaling
* SQS backlog monitoring
* production troubleshooting
* end-to-end order event flow

```
```
