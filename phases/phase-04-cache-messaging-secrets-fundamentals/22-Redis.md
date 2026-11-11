# 22-Redis.md

# Redis in Production — AWS ElastiCache and EKS

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Primary use:** Caching, session/state where appropriate, hot data, rate limiting, distributed coordination where justified
> **Primary production design:** Amazon ElastiCache for Redis/Valkey-compatible workloads
> **Alternative:** Redis/Valkey deployed inside EKS when there is a specific operational reason

---

# 1. Why Do We Need Redis?

Our applications use databases such as:

```text
order-service  → PostgreSQL
customer       → PostgreSQL
inventory      → PostgreSQL
catalog        → PostgreSQL / MongoDB
```

A database is excellent for durable data.

But imagine:

```text
1000 requests/sec
       │
       ▼
Every request queries PostgreSQL
       │
       ▼
Database CPU increases
       │
       ▼
Latency increases
```

For frequently requested data, we can introduce a cache:

```text
Client
  │
  ▼
Application
  │
  ▼
Redis
  │
  ├── HIT  → return data
  │
  └── MISS → Database
```

Redis is extremely useful for reducing repeated database reads and improving application latency.

---

# 2. Redis Is Not a Replacement for RDS

This distinction is critical.

PostgreSQL:

```text
Orders
Payments
Customers
Invoices
```

are durable business data.

Redis:

```text
Cached product
Cached pricing
Session data
Rate-limit counters
Temporary state
```

can often be reconstructed or allowed to expire, depending on the use case.

Do not design:

```text
Order
  ↓
Redis only
```

when the order must be durable.

Instead:

```text
Order request
      │
      ▼
PostgreSQL
      │
      └── Redis cache where useful
```

---

# 3. Redis Mental Model

Think of Redis as:

> **A very fast in-memory data store that can be used for caching and other low-latency data patterns.**

Typical operations:

```text
SET
GET
DEL
EXPIRE
INCR
HSET
LPUSH
SADD
```

Example:

```text
SET product:P1001 "{product-json}" EX 300
```

Then:

```text
GET product:P1001
```

returns the cached value.

---

# 4. Our E-Commerce Use Cases

Redis can be useful for:

### Catalog caching

```text
catalog-service
      │
      ▼
    Redis
      │
      ▼
 product data
```

### Pricing cache

```text
pricing-service
      │
      ▼
    Redis
```

### Session/state

Where session-based architecture is required:

```text
User
 │
 ▼
Application Pod
 │
 ▼
Redis
```

### Rate limiting

```text
Client
  │
  ▼
API
  │
  ▼
Redis counter
```

### Frequently accessed configuration

```text
Application
     │
     ▼
Redis
```

But Redis should not become a dumping ground for everything.

---

# 5. Why Redis Is Important With Kubernetes

Suppose we have:

```text
order-service
├── pod-1
├── pod-2
├── pod-3
├── pod-4
└── pod-5
```

If session/state exists only inside Pod 1:

```text
User
  │
  ▼
Pod-1
  │
  └── local memory
```

and the next request reaches:

```text
Pod-3
```

the state may not be available.

Externalizing appropriate state gives:

```text
                Redis
              /   |   \
             /    |    \
          Pod-1 Pod-2 Pod-3
```

Now the application replicas can share the relevant state.

---

# 6. Two Ways to Run Redis

We have two major choices.

### Option A — Managed Redis/Valkey

```text
EKS
 │
 └── Applications
        │
        ▼
AWS ElastiCache
```

### Option B — Redis/Valkey inside EKS

```text
EKS
 │
 ├── Applications
 │
 └── Redis/Valkey
        │
        └── Persistent storage if required
```

For our primary AWS production architecture, we prefer the managed service where it satisfies the requirements.

---

# 7. Why Managed Redis?

Running Redis inside EKS means we must think about:

```text
Pod
Storage
Replication
Failover
Cluster topology
Upgrades
Backups
Monitoring
Node failures
AZ failures
```

With a managed AWS service, AWS handles much of the underlying infrastructure.

Our architecture becomes:

```text
Application
     │
     ▼
ElastiCache
     │
     ├── HA
     ├── Replication
     ├── Failover
     └── Monitoring integration
```

We still own configuration, security, application behavior, capacity planning, and operational policies.

---

# 8. Redis vs Valkey

AWS has supported both Redis OSS-compatible offerings and Valkey-based offerings.

For a new AWS deployment, the exact engine choice should follow:

* current AWS service support
* organizational standards
* application/client compatibility
* licensing considerations
* operational requirements

For this project, we'll use **Redis terminology throughout the application discussion**, while recognizing that a modern AWS architecture may use **Valkey** where appropriate.

---

# 9. Production Architecture

Our target:

```text
                         AWS VPC
                            │
             ┌──────────────┴──────────────┐
             │                             │
            EKS                       ElastiCache
             │                             │
      ┌──────┼──────┐                ┌────┴────┐
      │      │      │                │         │
    order  catalog pricing         AZ-A      AZ-B
      │      │      │                │         │
      └──────┼──────┘                Node      Replica
             │
             ▼
           Redis
```

The cache should remain private.

---

# 10. Redis Should Not Be Public

Bad:

```text
Internet
   │
   ▼
Redis
```

Absolutely not.

Redis should be reachable through private networking:

```text
EKS Pod
   │
   ▼
Private VPC
   │
   ▼
ElastiCache
```

No public Redis endpoint should be exposed to the Internet.

---

# 11. Redis Security Group

Example architecture:

```text
order-service
      │
      │ TCP 6379
      ▼
Redis Security Group
```

Only approved workloads should be able to connect.

Conceptually:

```text
Redis-SG

Inbound:
TCP 6379
Source:
approved application workload SG
```

Not:

```text
0.0.0.0/0 → 6379
```

---

# 12. Redis Port

Default Redis port:

```text
6379
```

Example:

```text
redis.example.internal:6379
```

The application uses:

```text
REDIS_HOST
REDIS_PORT
```

with authentication configured according to the selected AWS Redis/Valkey service mode.

---

# 13. Redis Authentication

Production Redis should not be unauthenticated.

Depending on the selected AWS service/configuration, authentication can involve:

* AUTH/password-based access
* Redis ACLs
* TLS
* IAM authentication where supported by the selected service mode

The exact mechanism should be selected based on current AWS engine/service capabilities.

Credentials should be stored in:

```text
AWS Secrets Manager
```

not:

```text
Git
Helm values
Dockerfile
source code
```

---

# 14. Encryption

Production Redis should use encryption in transit and at rest where supported and required.

Architecture:

```text
Application
     │
     │ TLS
     ▼
ElastiCache
     │
     ▼
Encrypted storage
```

Encryption keys can be managed using AWS KMS according to the service's supported encryption configuration.

---

# 15. Redis Connection From EKS

Spring Boot application:

```text
order-service
      │
      ▼
Redis client
      │
      ▼
ElastiCache endpoint
```

Example conceptual configuration:

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

The exact Spring Boot property names depend on the Spring Boot version and selected Redis client configuration.

---

# 16. Add Redis Dependency

For Spring Boot, a typical dependency is:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Spring Boot commonly uses Lettuce as the Redis client.

Architecture:

```text
Spring Boot
    │
    ▼
Spring Data Redis
    │
    ▼
Lettuce
    │
    ▼
Redis
```

---

# 17. Redis Cache Pattern

The most common pattern:

```text
Application
     │
     ▼
Check Redis
     │
 ┌───┴────┐
 │        │
 HIT     MISS
 │        │
 ▼        ▼
Return   DB
          │
          ▼
        Redis
          │
          ▼
        Return
```

This is called **cache-aside**.

---

# 18. Cache-Aside Example

Suppose:

```text
GET /products/P1001
```

First request:

```text
Application
     │
     ▼
Redis GET product:P1001
     │
     ▼
MISS
     │
     ▼
PostgreSQL/MongoDB
     │
     ▼
Product
     │
     ├────► Redis SET
     │
     ▼
Response
```

Second request:

```text
Application
     │
     ▼
Redis GET product:P1001
     │
     ▼
HIT
     │
     ▼
Response
```

The database isn't queried for every request.

---

# 19. Spring Cache

Spring provides cache abstractions.

Example:

```java
@Cacheable(
    value = "products",
    key = "#productId"
)
public Product getProduct(Long productId) {

    return productRepository
        .findById(productId)
        .orElseThrow();
}
```

Conceptually:

```text
getProduct(1001)
       │
       ▼
Cache
       │
   ┌───┴────┐
   │        │
  HIT      MISS
   │        │
 return    DB
```

---

# 20. Cache Keys

Good key design is extremely important.

Example:

```text
product:P1001
product:P1002
pricing:P1001
inventory:P1001
customer:C1001
```

Avoid ambiguous keys:

```text
1001
```

Better:

```text
product:1001
customer:1001
order:1001
```

because different domains can use the same numeric ID.

---

# 21. Key Naming Convention

For our project:

```text
<domain>:<resource>:<id>
```

Examples:

```text
product:details:P1001
price:product:P1001
inventory:product:P1001
customer:profile:C1001
order:summary:O1001
```

This makes troubleshooting much easier.

---

# 22. TTL

Never cache everything forever.

Example:

```text
product:P1001
TTL = 300 seconds
```

After five minutes:

```text
Redis
  │
  ▼
Key expires
```

The next request loads fresh data.

Example:

```bash
SET product:P1001 "..." EX 300
```

TTL helps prevent permanently stale data.

---

# 23. Different TTLs

Not every object should have the same TTL.

Example:

```text
Product catalog → 10 minutes
Pricing → 30 seconds
Promotions → 60 seconds
Reference data → 1 hour
Session → application-specific
```

These are illustrative values.

TTL should come from:

```text
Business freshness requirement
+
Update frequency
+
Database load
```

---

# 24. Cache Invalidation

One of the hardest cache problems is:

> How do we keep cached data consistent with the source of truth?

Suppose:

```text
Database:
price = 100
```

Redis:

```text
price = 100
```

Then price changes:

```text
Database:
price = 120
```

but Redis still contains:

```text
price = 100
```

Application returns stale data.

---

# 25. Write-Through / Cache-Aside Strategies

Several strategies exist.

### Cache-aside

```text
Write DB
   ↓
Invalidate cache
```

### Write-through

```text
Application
   │
   ▼
Cache
   │
   ▼
Database
```

### Event-driven invalidation

```text
Database change
      │
      ▼
Event
      │
      ▼
Cache invalidation
```

The correct pattern depends on consistency requirements.

---

# 26. Event-Driven Cache Invalidation

Our architecture already uses messaging.

Later:

```text
Product updated
      │
      ▼
product-service
      │
      ▼
SQS/SNS
      │
      ▼
Cache invalidation
      │
      ▼
Redis
```

For example:

```text
product:P1001
```

could be deleted when the product changes.

This keeps the cache closer to the source of truth.

---

# 27. Cache Stampede

Consider a very popular key:

```text
product:P1001
```

It expires.

Suddenly:

```text
10,000 requests
```

arrive.

All requests see:

```text
CACHE MISS
```

Then:

```text
10,000 requests
      ↓
10,000 database queries
```

The database can become overloaded.

This is called a **cache stampede** or thundering herd problem.

---

# 28. Preventing Cache Stampede

Possible techniques:

* TTL jitter
* request coalescing
* distributed locking where justified
* background refresh
* stale-while-revalidate
* prewarming hot keys

Example:

```text
Cache expires
     │
     ▼
One request refreshes
     │
     ▼
Other requests use existing/stale value
```

The implementation must match the consistency requirement.

---

# 29. Cache Penetration

Suppose users request:

```text
product:INVALID-123
product:INVALID-456
product:INVALID-789
```

Every request misses Redis and queries the database.

Attackers can intentionally exploit this.

Possible protections:

* validate IDs
* negative caching
* rate limiting
* request validation
* application-level controls

---

# 30. Cache Avalanche

Imagine thousands of keys all expire at:

```text
12:00:00
```

Then:

```text
Redis
 │
 └── thousands of keys expire
              │
              ▼
        Database load spike
```

Avoid synchronized TTLs where possible.

TTL jitter can help:

```text
300 ± random seconds
```

rather than every key expiring at exactly the same moment.

---

# 31. Redis and HPA

This is another important production relationship.

Suppose:

```text
order-service
3 pods
```

and traffic increases.

HPA:

```text
3 → 10 pods
```

Now ten pods connect to Redis.

Redis may experience:

```text
More connections
More commands
More memory usage
More network traffic
```

So:

```text
Application scaling
      ↓
Redis load
      ↓
Redis capacity
```

must be considered together.

---

# 32. Redis Memory

Redis is primarily memory-oriented.

Important metric:

```text
Used Memory
```

If Redis approaches its memory limit:

```text
Memory
████████████████████
       HIGH
```

you need an eviction policy or capacity expansion depending on the use case.

---

# 33. Eviction Policies

Redis can remove keys according to configured policies.

Conceptually:

```text
Memory full
    ↓
Eviction policy
    ↓
Remove eligible keys
```

Possible policies include:

* `noeviction`
* `allkeys-lru`
* `allkeys-lfu`
* TTL-based policies such as `volatile-lru`
* other Redis-supported policies

The correct policy depends on what Redis stores.

---

# 34. Do Not Use the Same Redis Policy for Everything

Suppose Redis stores:

```text
cache
```

Eviction can be acceptable.

But if Redis stores:

```text
critical application state
```

blind eviction may be dangerous.

Therefore:

> **Redis configuration depends on the data semantics.**

Cache and durable state should not be treated identically.

---

# 35. Redis Cluster vs Replication

These are often confused.

### Replication

```text
Primary
   │
   ├── Replica
   └── Replica
```

Purpose:

* high availability
* redundancy
* failover

### Cluster/sharding

```text
Node 1 → keys A-F
Node 2 → keys G-M
Node 3 → keys N-Z
```

Purpose:

* distribute data
* scale memory
* scale throughput

You can have architectures combining replication and sharding.

---

# 36. Scaling Redis

Suppose:

```text
Redis memory
    │
    ▼
80% used
```

Possible solutions:

### Vertical scaling

Larger Redis node:

```text
10 GB → 20 GB
```

### Horizontal/sharded scaling

```text
Redis
 ├── Shard 1
 ├── Shard 2
 └── Shard 3
```

Which approach is appropriate depends on:

* data size
* throughput
* command patterns
* availability
* client support
* workload characteristics

---

# 37. Redis High Availability

A production architecture may look like:

```text
                Redis
                 │
          ┌──────┴──────┐
          │             │
       Primary        Replica
          │             │
         AZ-A          AZ-B
```

If the primary fails:

```text
Primary
   X
   │
   ▼
Replica
   │
   ▼
Promoted
```

The managed service can handle much of the failover orchestration.

---

# 38. Redis Inside EKS

Now let's compare with running Redis ourselves.

```text
EKS
 │
 ├── redis-0
 ├── redis-1
 └── redis-2
```

You need:

```text
StatefulSet/operator
+
Persistent storage if persistence is required
+
Replication
+
Failover
+
Monitoring
+
Backup
```

Again:

```text
3 Redis Pods
≠
Redis HA
```

You need Redis-aware clustering/failover architecture.

---

# 39. Redis StatefulSet Example

A simplified learning example:

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: redis
  namespace: database

spec:
  serviceName: redis
  replicas: 1

  selector:
    matchLabels:
      app: redis

  template:
    metadata:
      labels:
        app: redis

    spec:
      containers:
        - name: redis
          image: redis:7

          ports:
            - containerPort: 6379

          volumeMounts:
            - name: redis-data
              mountPath: /data

  volumeClaimTemplates:
    - metadata:
        name: redis-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 20Gi
```

This is useful for learning.

It is **not automatically a production HA Redis architecture**.

---

# 40. Redis Service in EKS

```yaml
apiVersion: v1
kind: Service

metadata:
  name: redis
  namespace: database

spec:
  type: ClusterIP

  selector:
    app: redis

  ports:
    - port: 6379
      targetPort: 6379
```

Application:

```text
order-service
      │
      ▼
redis.database.svc.cluster.local:6379
      │
      ▼
Redis
```

---

# 41. Redis Persistent Storage

If Redis persistence is required:

```text
Redis
 │
 ▼
/data
 │
 ▼
PVC
 │
 ▼
EBS
```

But whether persistence is required depends entirely on the workload.

For a disposable cache:

```text
Redis data
    ↓
Can be rebuilt from database
```

persistent storage may not provide much business value.

For state that must survive restarts:

```text
Persistence
```

becomes much more important.

---

# 42. Cache vs Persistent Redis

### Cache

```text
Source of truth → PostgreSQL
Cache → Redis
```

If Redis disappears:

```text
Redis failure
    ↓
Rebuild cache
    ↓
Database remains source of truth
```

### Stateful Redis

```text
Redis
  ↓
Important state
```

Now Redis failure can have business impact.

The reliability requirements become much higher.

---

# 43. Redis Backup

For managed Redis, use the backup/snapshot capabilities supported by the selected AWS service configuration.

For self-managed Redis:

```text
Redis
 │
 ├── RDB
 ├── AOF where appropriate
 └── backup system
```

Then:

```text
Backup
  ↓
S3 / backup storage
```

But again:

> Backup requirements depend on whether Redis is a cache or a system holding important state.

---

# 44. Redis Monitoring

Important metrics include:

```text
CPU
Memory
Connections
Commands
Network throughput
Cache hit rate
Cache miss rate
Evictions
Replication health
Replication lag
Latency
```

For caching workloads, one of the most important application-level metrics is:

```text
Cache Hit Ratio
```

Example:

```text
Cache hits = 9,000
Cache misses = 1,000

Hit ratio = 90%
```

---

# 45. Cache Hit Ratio

A high hit ratio generally means the cache is serving a large proportion of requests.

Conceptually:

```text
             Requests
                │
        ┌───────┴────────┐
        │                │
      Redis HIT        Redis MISS
        │                │
        ▼                ▼
      Return             DB
```

If hit ratio drops suddenly:

```text
95%
 ↓
70%
 ↓
40%
```

investigate:

* TTL changes
* key changes
* cache invalidation
* deployment changes
* traffic pattern
* cache capacity
* eviction
* application bugs

---

# 46. Redis Latency

Monitor:

```text
Redis command latency
```

If Redis becomes slow:

```text
Application
    │
    ▼
Redis
    │
    ▼
High latency
```

the application can become slow even though PostgreSQL is healthy.

Therefore Redis is itself a production dependency.

---

# 47. Redis Failure

Suppose:

```text
order-service
      │
      ▼
Redis
      X
```

What should happen?

For a normal cache:

```text
Redis failure
      ↓
Fallback to database
      ↓
Application continues
```

This is often called **cache failure tolerance**.

But be careful.

If every request immediately falls back to PostgreSQL:

```text
Redis failure
     ↓
10,000 requests
     ↓
10,000 DB requests
     ↓
RDS overloaded
```

So Redis failure can create a **secondary database outage**.

---

# 48. Designing for Redis Failure

Possible protections:

* timeouts
* circuit breakers
* bounded retries
* fallback behavior
* rate limiting
* request coalescing
* database capacity planning
* graceful degradation

Don't use:

```text
Retry Redis forever
```

or:

```text
Retry database forever
```

because retries can amplify outages.

---

# 49. Redis Timeout

Application configuration should have sensible timeouts.

Example conceptually:

```text
Redis connection timeout
Redis command timeout
```

Avoid extremely long waits.

Otherwise:

```text
Redis slow
   ↓
Application threads wait
   ↓
Thread pool exhausted
   ↓
Requests queue
   ↓
Application becomes unhealthy
```

A dependency timeout is therefore also an application availability control.

---

# 50. Redis and Rate Limiting

Redis is very useful for distributed rate limiting.

Example:

```text
Client IP
   │
   ▼
INCR rate:user:1001
   │
   ▼
Redis
```

Suppose:

```text
100 requests/minute
```

Redis can maintain counters with expiration.

Conceptually:

```text
INCR rate:user:1001
EXPIRE rate:user:1001 60
```

For production, use an atomic/race-safe design appropriate to the chosen algorithm.

---

# 51. Distributed Locks

Redis can also be used for distributed coordination.

Example:

```text
Pod-1
Pod-2
Pod-3
```

All want to process the same work.

A distributed lock can coordinate them.

But:

> Don't introduce distributed locks casually.

Distributed locking has failure, expiry, ownership, and correctness considerations.

For critical business transactions, database transactions or durable messaging may be more appropriate.

---

# 52. Redis and Kubernetes Sessions

Suppose:

```text
User
 │
 ▼
ALB
 │
 ├── Pod-1
 ├── Pod-2
 └── Pod-3
```

Without shared session state:

```text
Pod-1 → session exists
Pod-2 → session missing
```

With Redis:

```text
               Redis
             /   |   \
            /    |    \
         Pod-1 Pod-2 Pod-3
```

The application can remain horizontally scalable.

However, modern applications can also avoid server-side sessions entirely using other authentication/state models, so Redis should not be introduced solely because Kubernetes has multiple replicas.

---

# 53. Redis Security Checklist

Production:

```text
[ ] Private networking
[ ] No public endpoint
[ ] TLS enabled where required
[ ] Authentication enabled
[ ] Secrets Manager
[ ] Least-privilege access
[ ] Security groups
[ ] Network controls
[ ] Encryption at rest
[ ] Encryption in transit
[ ] Monitoring
[ ] Audit/logging strategy
```

---

# 54. Redis Infrastructure as Code

For AWS managed Redis/Valkey, Terraform should provision the infrastructure.

Architecture:

```text
Terraform
    │
    ▼
ElastiCache
    │
    ├── Subnet Group
    ├── Security Group
    ├── Parameter Group
    ├── Replication / Cluster configuration
    └── Encryption
```

Example structure:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── redis/
    │       ├── main.tf
    │       ├── variables.tf
    │       └── outputs.tf
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

---

# 55. Terraform Redis Subnet Group

The exact Terraform resource depends on whether you're using the current ElastiCache engine/mode.

Conceptually:

```hcl
resource "aws_elasticache_subnet_group" "this" {
  name = "${var.name}-subnet-group"

  subnet_ids = var.private_subnet_ids
}
```

Redis remains inside private subnets.

---

# 56. Terraform Security Group

Conceptually:

```hcl
resource "aws_security_group" "redis" {
  name        = "${var.name}-redis-sg"
  description = "Redis access from approved workloads"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Redis"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = var.allowed_security_group_ids
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Production should restrict access as tightly as the chosen networking model allows.

---

# 57. Don't Hardcode Redis Passwords

Bad:

```hcl
auth_token = "RedisPassword123"
```

Better:

```text
Secrets Manager
       │
       ▼
Approved Terraform/application integration
```

Be aware that even sensitive Terraform values can still appear in Terraform state depending on the resource and configuration.

Treat Terraform state as sensitive.

---

# 58. Application Configuration

Production:

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

Kubernetes:

```text
Pod
 │
 ├── REDIS_HOST
 ├── REDIS_PORT
 └── REDIS_PASSWORD
```

The password comes through the approved secret-management mechanism.

---

# 59. Production Traffic Example

Suppose:

```text
100,000 product requests
```

Without caching:

```text
100,000
    │
    ▼
Database
```

With effective caching:

```text
100,000 requests
       │
       ▼
     Redis
       │
   ┌───┴────┐
   │        │
 95,000    5,000
   │        │
   ▼        ▼
 Return     DB
```

This is illustrative, not a promise of a specific hit rate.

The important architectural idea is:

> Redis absorbs repeated hot reads so the database does not have to answer every request.

---

# 60. Low Traffic

Example:

```text
10 requests/sec
```

Architecture:

```text
Application
     │
     ▼
Redis
     │
     ▼
RDS
```

Redis may be relatively lightly utilized.

Don't massively overprovision simply because production architecture diagrams look impressive.

---

# 61. Medium Traffic

Suppose:

```text
500 requests/sec
```

Now:

```text
Application replicas
       ↑
       │
      HPA
       │
       ▼
      Redis
       │
       ▼
      RDS
```

We monitor:

```text
Redis hit ratio
Redis memory
Redis latency
RDS CPU
RDS connections
Application latency
```

Scaling one layer can affect another.

---

# 62. High Traffic

Suppose:

```text
10,000+ requests/sec
```

Now architecture becomes more deliberate:

```text
                 ALB
                  │
              Kubernetes
                  │
        ┌─────────┼─────────┐
        │         │         │
      Pod       Pod       Pod
        │         │         │
        └─────────┼─────────┘
                  │
               Redis
            ┌─────┴─────┐
            │           │
         Shard-1     Shard-2
            │           │
            └─────┬─────┘
                  │
                 RDS
```

But high traffic does **not automatically mean Redis Cluster**.

Measure first.

---

# 63. Capacity Planning

Before scaling Redis, collect:

```text
Current memory usage
Peak memory usage
Requests/sec
Commands/sec
Connections
Latency
Cache hit ratio
Evictions
Network throughput
```

Then determine:

```text
Current capacity
       ↓
Peak requirement
       ↓
Safety headroom
       ↓
Target capacity
```

---

# 64. Redis Troubleshooting — Connection Failure

If application reports:

```text
Cannot connect to Redis
```

check:

```text
Pod
 ↓
DNS
 ↓
Network
 ↓
Security Group
 ↓
Redis endpoint
 ↓
TLS
 ↓
Authentication
 ↓
Redis health
```

Don't immediately restart the application.

---

# 65. Redis Troubleshooting — High Memory

Symptom:

```text
Redis memory > expected
```

Check:

```text
Number of keys
Key sizes
TTL
Evictions
Hot keys
Unexpected data
Application changes
```

A common application bug is accidentally caching massive objects.

For example:

```text
Expected:
Product = 10 KB

Actual:
Product = 5 MB
```

Across:

```text
100,000 keys
```

memory consumption becomes enormous.

---

# 66. Redis Troubleshooting — Low Hit Ratio

Suppose:

```text
Previous hit ratio = 95%
Current = 40%
```

Investigate:

```text
Did key format change?
Did TTL change?
Did cache get flushed?
Did deployment change?
Did traffic pattern change?
Are keys expiring too quickly?
Are evictions occurring?
```

Do not immediately increase Redis capacity.

The application may simply have a cache bug.

---

# 67. Redis Troubleshooting — Latency Spike

```text
Redis latency
     │
     ▼
High
```

Check:

```text
CPU
Memory pressure
Network
Command patterns
Large commands
Connection count
Hot keys
Evictions
Cluster topology
Application client behavior
```

Also inspect whether the application is sending inefficient commands.

---

# 68. Avoid Huge Redis Commands

Avoid designs where the application sends enormous payloads or performs expensive operations against massive data structures.

For example:

```text
GET 5 MB object
```

across thousands of requests can create:

```text
Network load
+
Serialization cost
+
Redis memory pressure
+
Application memory pressure
```

Redis being fast doesn't mean every Redis operation is free.

---

# 69. Redis and Serialization

Spring applications often serialize objects before storing them.

For example:

```text
Java Object
   ↓
JSON
   ↓
Redis
```

or another serialization format.

Be careful with:

* payload size
* compatibility across deployments
* serialization changes
* security
* schema evolution

Don't blindly cache Java-specific serialized objects if interoperability or long-term compatibility matters.

---

# 70. Redis Cache Consistency

Always define:

```text
Source of truth?
Cache freshness requirement?
TTL?
Invalidation strategy?
What happens if cache fails?
```

For example:

```text
Product price
```

may require much stricter freshness than:

```text
Product description
```

Therefore:

```text
One Redis strategy
```

does not necessarily fit every microservice.

---

# 71. Production Architecture

Our final managed-cache architecture:

```text
                         Internet
                            │
                         CloudFront
                            │
                           WAF
                            │
                           ALB
                            │
                           EKS
                            │
             ┌──────────────┼──────────────┐
             │              │              │
         order-service  catalog-service pricing-service
             │              │              │
             └──────────────┼──────────────┘
                            │
                     ElastiCache
                       Redis/Valkey
                            │
                            ▼
                         RDS
```

Notice the difference:

```text
RDS = source of truth for relational data
Redis = performance/cache layer
```

---

# 72. Production vs Learning

### Learning

You can run:

```text
Docker Compose
    │
    ├── PostgreSQL
    └── Redis
```

or:

```text
EKS
 ├── application
 └── Redis StatefulSet
```

### Production

Prefer the appropriate managed service when operational requirements justify it:

```text
EKS
 │
 └── Applications
       │
       ▼
ElastiCache Redis/Valkey
```

This keeps the production platform simpler.

---

# 73. Where Redis Fits in Our 21-Microservice Platform

We don't necessarily connect all 21 services to the same Redis instance blindly.

Example:

```text
catalog-service
       │
       ▼
catalog cache

pricing-service
       │
       ▼
pricing cache

customer-service
       │
       ▼
customer/session cache
```

Depending on scale and isolation requirements, we might use:

```text
Shared Redis
```

or:

```text
Separate Redis clusters
```

or:

```text
Separate logical namespaces/keyspaces
```

The correct design depends on workload and isolation requirements.

---

# 74. Important Rule — Don't Cache Everything

Bad architecture:

```text
Every DB query
      ↓
Redis
```

without understanding why.

Good architecture:

```text
Identify expensive/hot reads
        ↓
Measure database load
        ↓
Determine freshness requirement
        ↓
Introduce cache
        ↓
Measure improvement
```

Caching is an optimization, not a default replacement for good database design.

---

# 75. Final Redis Mental Model

When Redis is introduced:

```text
                Request
                   │
                   ▼
              Application
                   │
             ┌─────┴─────┐
             │           │
           Redis        DB
             │           │
           Cache       Source
```

And production engineering asks:

```text
Where does Redis run?
        ↓
How is it secured?
        ↓
How much memory?
        ↓
What is cached?
        ↓
What is the TTL?
        ↓
How is invalidation handled?
        ↓
What happens on cache failure?
        ↓
How does scaling work?
        ↓
How is HA implemented?
        ↓
How is it monitored?
        ↓
How does it affect RDS?
```

That last question is especially important.

**Redis can protect the database from load, but a poorly designed Redis failure can also overload the database.**

---

# 76. Chapter Summary

We now have:

```text
20-RDS.md
      ↓
AWS managed relational database

21-EKS-Database-Management.md
      ↓
How stateful databases actually run inside Kubernetes

22-Redis.md
      ↓
Caching + fast state + Redis/Valkey architecture
```

Our production data architecture is becoming:

```text
                    EKS
                     │
       ┌─────────────┼─────────────┐
       │             │             │
   Microservice  Microservice  Microservice
       │             │             │
       └─────────────┼─────────────┘
                     │
             ┌───────┴────────┐
             │                │
          Redis             RDS
        Cache layer      Source of truth
```

## Next: `23-SQS-SNS.md`

We'll add the **asynchronous/event-driven layer**:

```text
order-service
      │
      ▼
    SQS/SNS
      │
      ├── payment-service
      ├── inventory-service
      ├── notification-service
      └── shipping-service
```

and cover **queues vs topics, standard vs FIFO, visibility timeout, DLQ, retries, idempotency, ordering, message duplication, consumer scaling, IAM, encryption, Terraform, Spring Boot integration, EKS consumers, low/medium/high traffic, and real production failure scenarios**.
