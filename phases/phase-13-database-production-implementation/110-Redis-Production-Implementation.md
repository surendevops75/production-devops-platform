# 110-Redis-Production-Implementation.md

````markdown
# 110 — Redis Production Implementation

## 1. Purpose

Redis is used in our e-commerce platform as a high-speed data store for workloads where repeatedly querying PostgreSQL would create unnecessary latency or database load.

Our primary production design uses:

```text
EKS
  |
  v
Amazon ElastiCache for Redis/Valkey-compatible workloads
````

rather than running Redis inside application Pods.

The main goals are:

* reduce database read pressure
* reduce application latency
* cache frequently accessed data
* support short-lived state where appropriate
* understand cache failures
* implement TTL correctly
* secure Redis
* monitor cache behavior
* prevent cache stampede
* integrate Redis with Spring Boot
* understand Redis behavior during scaling and failures

---

# 2. Where Redis Fits

Our order-service architecture becomes:

```text
                    Client
                       |
                       v
                      ALB
                       |
                       v
                      EKS
                       |
                 order-service
                  /          \
                 /            \
                v              v
             Redis            RDS
          ElastiCache       PostgreSQL
```

The normal request path for cached data is:

```text
Request
   |
   v
Application
   |
   v
Redis
   |
   +---- HIT ----> Return data
   |
   +---- MISS ---> PostgreSQL
                    |
                    v
                  Redis
                    |
                    v
                 Response
```

---

# 3. Why Use Redis?

Suppose the application receives:

```text
10,000 requests/sec
```

and many requests repeatedly query the same product information.

Without caching:

```text
10,000 requests
       |
       v
PostgreSQL
```

This can create unnecessary database load.

With Redis:

```text
10,000 requests
       |
       v
Redis
       |
       +---- most requests served from cache
       |
       +---- cache misses go to PostgreSQL
```

The database receives fewer reads.

---

# 4. Redis Is Not Automatically a Database Replacement

A common mistake is:

> "Redis is faster, so store everything in Redis."

Our architecture generally treats Redis as a cache unless the specific workload requires Redis as a primary data store.

For example:

```text
PostgreSQL
    ↓
Source of truth

Redis
    ↓
Performance layer
```

If Redis disappears:

```text
Redis failure
     ↓
Cache unavailable
     ↓
Application falls back to PostgreSQL
```

The application should not lose the authoritative order data merely because the cache is unavailable.

---

# 5. Production Choice

For our production AWS architecture:

```text
Application
    |
    v
Amazon ElastiCache
```

Advantages include:

* AWS-managed infrastructure
* managed failover options
* AWS networking/security integration
* monitoring integration
* reduced operational burden compared with self-managing Redis

The exact engine/version and supported features should be pinned and verified against the AWS service currently selected.

---

# 6. Why Not Redis Inside EKS?

We could technically deploy:

```text
Redis StatefulSet
PVC
EBS
Redis replicas
Sentinel/Cluster
```

but then the platform team becomes responsible for:

```text
Redis upgrades
Failover
Replication
Storage
Backups
Node failures
AZ placement
Cluster recovery
Operational tuning
```

For a production AWS environment, managed ElastiCache is usually simpler when its feature set meets the workload requirements.

EKS-hosted Redis remains useful for:

```text
Learning
Development
Testing
Specialized workloads
```

---

# 7. Redis Deployment Models

There are several concepts to distinguish.

## Single node

```text
Application
     |
     v
 Redis
```

Simple, but a single node represents a failure domain.

---

## Primary + replica

```text
             Redis Primary
                  |
            replication
                  |
                  v
             Redis Replica
```

If the primary fails, a replica can be promoted depending on the selected ElastiCache configuration.

---

## Cluster mode

Conceptually:

```text
Redis Cluster
   |
   +-- Shard 1
   |     +-- Primary
   |     +-- Replica
   |
   +-- Shard 2
   |     +-- Primary
   |     +-- Replica
   |
   +-- Shard 3
         +-- Primary
         +-- Replica
```

This provides horizontal data distribution for appropriate workloads.

Do not enable cluster mode simply because the application has high traffic. First determine whether the workload needs:

```text
More memory
More throughput
Horizontal sharding
Higher concurrency
```

---

# 8. Multi-AZ

For production availability, Redis should not depend on one Availability Zone when the selected ElastiCache architecture supports Multi-AZ/failover.

Conceptually:

```text
              ElastiCache
                   |
          +--------+--------+
          |                 |
         AZ-A              AZ-B
       Primary            Replica
```

The application connects through the appropriate managed endpoint.

---

# 9. VPC Placement

Redis should normally be private.

Example:

```text
Internet
   |
   X
   |
   X
Redis

EKS
 |
 +------> Redis
```

There should not be a public Internet path directly to Redis.

---

# 10. Security Groups

Example:

```text
EKS Application SG
        |
        | TCP 6379
        v
Redis SG
```

Redis Security Group should allow inbound Redis traffic only from the approved application security group.

Conceptually:

```text
Redis SG inbound:

Protocol: TCP
Port: 6379
Source: EKS application SG
```

Do not use:

```text
0.0.0.0/0
```

for Redis access.

---

# 11. Network Flow

Our production path:

```text
order-service Pod
      |
      v
Node ENI / VPC networking
      |
      v
Redis Security Group
      |
      v
ElastiCache
```

If connectivity fails, investigate:

```text
Pod
 ↓
Node
 ↓
VPC CNI
 ↓
Subnet
 ↓
Route
 ↓
Security Group
 ↓
Redis endpoint
```

---

# 12. DNS

The application should use the managed Redis endpoint.

Example conceptual configuration:

```text
redis-prod.xxxxxx.cache.amazonaws.com
```

Do not hardcode an IP address.

Redis infrastructure may change during:

```text
Failover
Maintenance
Replacement
Scaling
```

The endpoint is the stable application abstraction.

---

# 13. Redis Port

The common Redis TCP port is:

```text
6379
```

But the application should use the actual endpoint/port exposed by the deployed ElastiCache configuration.

Do not assume every Redis-compatible deployment uses the default port.

---

# 14. Encryption

Production Redis should use encryption where supported and required.

Two important categories:

```text
Encryption at rest
Encryption in transit
```

Conceptually:

```text
EKS
 |
 | TLS
 v
Redis
 |
 | encrypted storage
 v
ElastiCache
```

The exact encryption options depend on the selected engine/version and AWS service configuration.

---

# 15. Redis Authentication

The application should not connect anonymously to a production Redis cluster when authentication is required.

Credentials/authentication material should be stored in:

```text
AWS Secrets Manager
```

and injected through our existing secret architecture.

Our pattern is:

```text
Secrets Manager
      |
      v
External Secrets Operator
      |
      v
Kubernetes Secret
      |
      v
order-service
```

Never put Redis credentials in:

```text
Git
Dockerfile
Helm values
ConfigMap
Application source
```

---

# 16. IAM vs Redis Authentication

Remember:

```text
IAM
```

controls AWS API access.

Redis authentication controls:

```text
Application
     |
     v
Redis
```

These are different security layers.

Our application may use:

```text
EKS Pod Identity
```

to access AWS Secrets Manager, while Redis itself uses its configured authentication/TLS mechanism.

---

# 17. Terraform Structure

Our infrastructure repository can contain:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── redis/
    │       ├── main.tf
    │       ├── variables.tf
    │       ├── outputs.tf
    │       └── security-groups.tf
    │
    └── environments/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

---

# 18. Terraform Module Inputs

Example:

```hcl
variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "node_type" {
  type = string
}

variable "num_cache_nodes" {
  type = number
}

variable "subnet_ids" {
  type = list(string)
}

variable "security_group_ids" {
  type = list(string)
}
```

---

# 19. ElastiCache Subnet Group

Conceptually:

```hcl
resource "aws_elasticache_subnet_group" "redis" {
  name       = "ecommerce-${var.environment}-redis"
  subnet_ids = var.subnet_ids

  tags = {
    Environment = var.environment
    Service     = "redis"
  }
}
```

Use private subnets.

For production, provide subnets across the required Availability Zones.

---

# 20. Redis Security Group

Conceptually:

```hcl
resource "aws_security_group" "redis" {
  name        = "ecommerce-${var.environment}-redis"
  description = "Redis access"
  vpc_id      = var.vpc_id

  tags = {
    Environment = var.environment
    Service     = "redis"
  }
}
```

Then permit Redis traffic only from the approved application security group.

---

# 21. Example Redis Ingress

Conceptually:

```hcl
resource "aws_vpc_security_group_ingress_rule" "redis_from_apps" {
  security_group_id            = aws_security_group.redis.id
  referenced_security_group_id = var.application_security_group_id

  from_port = 6379
  to_port   = 6379
  ip_protocol = "tcp"
}
```

The exact Terraform resource model depends on the AWS provider version used by the project.

---

# 22. Redis Production Parameters

The production module should define things such as:

```text
Engine
Engine version
Node type
Number of replicas
Subnet group
Security groups
Encryption
TLS
Authentication
Maintenance window
Backup/snapshot settings where applicable
Automatic failover
Multi-AZ
```

Do not hardcode these values into application repositories.

---

# 23. Environment Values

Example:

```hcl
redis_node_type = "cache.r7g.large"

redis_replicas = 2
```

Development might use a smaller configuration.

Production should be capacity-tested.

Do not copy production node sizes from an unrelated workload.

---

# 24. Application Configuration

Our Spring Boot application can use environment variables:

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: ${REDIS_SSL_ENABLED:true}
```

The exact property names depend on the Spring Boot version and Redis client configuration.

Verify them against the version used by the application.

---

# 25. Redis Client

Spring Boot can use a Redis client such as:

```text
Lettuce
```

or another supported client.

For a typical Spring Boot application, Lettuce is a common choice.

Architecture:

```text
Spring Boot
     |
     v
Redis Client
     |
     v
ElastiCache
```

---

# 26. Maven Dependency

For Spring Boot:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

This provides the Spring Data Redis integration.

---

# 27. RedisTemplate

A basic configuration can use:

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {

        RedisTemplate<String, Object> template =
                new RedisTemplate<>();

        template.setConnectionFactory(connectionFactory);

        return template;
    }
}
```

For production, serializers should be explicitly designed rather than relying on unsafe or ambiguous Java serialization.

---

# 28. Cache-Aside Pattern

Our preferred basic caching pattern:

```text
Application
    |
    v
Check Redis
    |
    +---- HIT ----> Return
    |
    +---- MISS ---> PostgreSQL
                     |
                     v
                   Redis
                     |
                     v
                  Return
```

This is called:

```text
Cache-Aside
```

---

# 29. Cache Read Example

Conceptually:

```java
public Product getProduct(String productId) {

    String key = "product:" + productId;

    Product cached = redisTemplate.opsForValue()
            .get(key);

    if (cached != null) {
        return cached;
    }

    Product product = productRepository
            .findById(productId)
            .orElseThrow();

    redisTemplate.opsForValue()
            .set(key, product, Duration.ofMinutes(10));

    return product;
}
```

The actual serialization and error handling should be productionized.

---

# 30. Cache Key Design

Use predictable names.

Example:

```text
product:1001
product:1002
customer:1001
cart:customer-123
pricing:product-1001
```

A good key should communicate:

```text
Domain
Entity
Identifier
Version where necessary
```

---

# 31. Avoid Ambiguous Keys

Bad:

```text
1001
```

Better:

```text
product:1001
```

Better when versioning matters:

```text
product:v2:1001
```

The correct strategy depends on the application.

---

# 32. TTL

TTL means:

```text
Time To Live
```

Example:

```text
product:1001
TTL = 10 minutes
```

After the TTL expires:

```text
Redis
  |
  X
expired
```

The next request becomes a cache miss and can retrieve fresh data.

---

# 33. TTL Must Match Data Freshness

Do not choose:

```text
TTL = 24 hours
```

for highly dynamic data simply because Redis is fast.

Examples:

```text
Product description:
30 minutes

Product price:
shorter TTL or explicit invalidation

Inventory:
very short TTL or carefully designed cache strategy

User profile:
minutes

Static configuration:
longer TTL
```

These are examples, not universal values.

---

# 34. Cache Hit

Example:

```text
Request
  ↓
Redis
  ↓
product:1001 exists
  ↓
HIT
  ↓
Return immediately
```

No database query is needed.

---

# 35. Cache Miss

```text
Request
  ↓
Redis
  ↓
product:1001 missing
  ↓
MISS
  ↓
PostgreSQL
  ↓
Get product
  ↓
Redis SET
  ↓
Return
```

---

# 36. Cache Hit Ratio

A critical Redis metric is:

```text
Cache Hit Ratio
```

Conceptually:

```text
hits
-----------------------
hits + misses
```

Example:

```text
90,000 hits
10,000 misses
```

Hit ratio:

```text
90%
```

A high hit ratio is not automatically proof that the cache is useful, but it is an important signal.

---

# 37. Why Cache Hit Ratio Matters

Suppose:

```text
1,000,000 requests
```

Without Redis:

```text
1,000,000 DB reads
```

With:

```text
95% cache hit
```

approximately:

```text
950,000 served by cache
50,000 DB reads
```

This can substantially reduce database read pressure.

Actual performance depends on query cost, cache latency, payload size, and workload.

---

# 38. Cache Invalidation

One of the hardest caching problems is:

> What happens when the source data changes?

Example:

```text
PostgreSQL:
Product price = ₹100

Redis:
product:1001
price = ₹100
```

Price changes:

```text
PostgreSQL:
₹120
```

But Redis may still contain:

```text
₹100
```

Now the cache is stale.

---

# 39. Invalidation Strategy

When data changes:

```text
Update PostgreSQL
      |
      v
Invalidate Redis
```

Example:

```java
productRepository.save(product);

redisTemplate.delete(
    "product:" + product.getId()
);
```

The next read loads fresh data.

---

# 40. Write-Through vs Cache-Aside

### Cache-aside

```text
Application
    |
    +--> DB
    |
    +--> Cache
```

Application manages both.

### Write-through

```text
Application
    |
    v
Cache
    |
    v
Database
```

The caching layer participates in writes.

The appropriate pattern depends on the workload.

Our basic e-commerce implementation starts with cache-aside because it is straightforward and widely applicable.

---

# 41. Cache Stampede

Suppose:

```text
product:1001
```

expires.

At exactly the same time:

```text
1000 requests
```

arrive.

All see:

```text
MISS
```

Then:

```text
1000 requests
      ↓
PostgreSQL
```

This creates a:

```text
Cache Stampede
```

---

# 42. Preventing Cache Stampede

Techniques include:

```text
TTL jitter
Request coalescing
Distributed locking
Background refresh
Early refresh
Stale-while-revalidate
```

The correct technique depends on the data and traffic pattern.

---

# 43. TTL Jitter

Instead of:

```text
TTL = 10 minutes
```

for every key, use a small randomized variation.

Conceptually:

```text
Product A:
9m 42s

Product B:
10m 13s

Product C:
9m 51s
```

This prevents thousands of related keys from expiring simultaneously.

---

# 44. Cache Penetration

Cache penetration happens when requests repeatedly ask for data that does not exist.

Example:

```text
product:999999999
```

does not exist.

Without protection:

```text
Request
 ↓
Redis MISS
 ↓
PostgreSQL
 ↓
No record
```

Repeated requests keep hitting PostgreSQL.

---

# 45. Preventing Cache Penetration

Options include:

```text
Negative caching
Input validation
Bloom filters where appropriate
Rate limiting
```

Example negative cache:

```text
product:999999999 = NOT_FOUND
TTL = short
```

Use this carefully so temporary database errors are not cached as "not found."

---

# 46. Cache Avalanche

An avalanche can happen when many cached entries become unavailable or expire together.

Example:

```text
Redis restart
   ↓
Large portion of cache disappears
   ↓
Many requests miss
   ↓
PostgreSQL receives sudden load
```

This is why the database must remain capable of handling a cache-loss scenario, at least to an acceptable degraded level.

---

# 47. Redis Failure Design

Our application should decide:

```text
What happens when Redis is unavailable?
```

For non-critical cache data:

```text
Redis failure
    ↓
Log/metric
    ↓
Read PostgreSQL
```

This is often preferable to:

```text
Redis failure
    ↓
Entire application unavailable
```

---

# 48. Cache Failure Is Different From Database Failure

If Redis fails:

```text
Application
    |
    X
Redis
    |
    v
Fallback
    |
    v
RDS
```

If RDS fails:

```text
Application
    |
    X
RDS
```

Redis generally should not become the only source of truth for business-critical relational data unless the architecture intentionally makes Redis authoritative.

---

# 49. Redis Circuit Breaker

If Redis becomes unhealthy, repeatedly attempting Redis operations can increase application latency.

Conceptually:

```text
Redis failure
    ↓
Many timeout calls
    ↓
Application threads blocked
```

A resilience strategy may:

```text
detect Redis failures
      ↓
short-circuit cache calls
      ↓
use fallback
      ↓
periodically test recovery
```

This must be implemented according to the application's resilience library and failure semantics.

---

# 50. Redis Timeout

A Redis timeout should not be so large that every HTTP request waits several seconds for a cache that is supposed to be a fast optimization.

Conceptually:

```text
HTTP request
   |
   v
Redis
   |
   X
timeout
   |
   v
Fallback
```

The exact timeout should be load-tested.

---

# 51. Redis Connection Pooling

The application also maintains Redis connections.

Therefore:

```text
EKS Pods
   ↓
Redis client connections
   ↓
ElastiCache
```

Scaling Pods can increase Redis connection count.

For example:

```text
20 Pods
×
client connection usage
```

must be considered.

Do not assume Redis has unlimited connections.

---

# 52. HPA Interaction

Our architecture now has two shared data dependencies:

```text
HPA
 ↓
More Pods
 ↓
More PostgreSQL connections
 ↓
More Redis connections
```

Therefore HPA tuning must consider:

```text
CPU
Memory
RDS
Redis
```

not only application CPU.

---

# 53. Redis and RDS Load Relationship

Without caching:

```text
Application
    ↓
RDS
```

With caching:

```text
Application
   |
   +----> Redis HIT
   |
   +----> RDS on MISS
```

If Redis fails:

```text
Application
   |
   +----> Redis unavailable
   |
   v
RDS
```

This means:

> Redis capacity and availability directly influence RDS traffic.

---

# 54. Cache Failure Load Test

We should eventually test:

```text
Normal
 ↓
Redis healthy
 ↓
High cache hit rate
```

Then:

```text
Redis unavailable
 ↓
Cache misses
 ↓
RDS traffic increases
```

Measure:

```text
RDS CPU
RDS connections
RDS latency
HTTP latency
HTTP errors
```

This is important because a Redis failure can indirectly create an RDS incident.

---

# 55. Redis Metrics

Important metrics include:

```text
Cache hits
Cache misses
Hit ratio
Connected clients
Memory usage
Evictions
CPU
Network bytes
Commands/sec
Latency
Replication health
```

The exact CloudWatch/ElastiCache metric names depend on the engine and configuration.

---

# 56. Evictions

If Redis memory becomes full, keys may be evicted depending on the configured eviction policy.

Conceptually:

```text
Redis memory
     ↓
Full
     ↓
Eviction policy
     ↓
Keys removed
```

Then:

```text
More cache misses
     ↓
More database traffic
```

Therefore eviction metrics matter.

---

# 57. Eviction Policy

Redis supports different eviction strategies.

Examples conceptually include:

```text
allkeys-lru
allkeys-lfu
volatile-lru
volatile-lfu
noeviction
```

The correct choice depends on the workload.

Do not select an eviction policy without understanding:

```text
Which keys have TTL
Which keys are disposable
Whether writes can fail
Whether stale data is acceptable
```

---

# 58. Cache Memory Planning

Suppose:

```text
Average cached object:
2 KB

1 million objects:
≈ 2 GB raw payload
```

But actual Redis memory usage also includes:

```text
Key overhead
Object metadata
Data structures
Replication
Fragmentation
Client buffers
Operational overhead
```

Therefore:

> Redis capacity must be based on measured memory usage, not payload size alone.

---

# 59. Serialization Size

Suppose a Product object contains:

```text
id
name
description
price
category
metadata
```

Serializing it as JSON may produce:

```text
3 KB
```

A million entries could require approximately:

```text
3 GB
```

before Redis overhead and replicas.

Therefore payload design matters.

Do not cache huge objects unnecessarily.

---

# 60. Cache Only What Helps

Bad:

```text
Cache entire 500 KB object
```

if the application only needs:

```text
price
availability
name
```

Prefer smaller representations where appropriate.

This reduces:

```text
Memory
Network
Serialization
Latency
```

---

# 61. Cache Key Namespace

For our e-commerce platform:

```text
product:v1:<id>
customer:v1:<id>
pricing:v1:<id>
inventory:v1:<id>
cart:v1:<customerId>
```

This helps prevent collisions.

---

# 62. Versioned Keys

Suppose Product schema changes:

```text
product:v1:1001
```

becomes:

```text
product:v2:1001
```

This can allow controlled rollout without trying to interpret old cached objects using a new schema.

---

# 63. Redis Serialization

Avoid blindly using native Java serialization for externally managed/shared cache data.

Prefer a deliberate format such as:

```text
JSON
```

or another explicitly defined serialization format.

Benefits:

```text
Readable
Versionable
Language-independent
Easier debugging
```

The choice depends on performance and schema requirements.

---

# 64. Cache Data Should Be Disposable

A useful design rule:

```text
Can I delete Redis completely
and rebuild the application state
from the source of truth?
```

For normal cache entries, the answer should generally be:

```text
Yes
```

If the answer is:

```text
No
```

then Redis may be serving as a primary data store or stateful system, which requires a different architecture and recovery strategy.

---

# 65. Cart Data

Shopping carts require special consideration.

A cart may be stored in:

```text
Redis
```

for fast access, but business requirements determine whether Redis can be the authoritative store.

Possible architecture:

```text
Cart Service
    |
    +--> Redis
    |
    +--> durable store
```

or:

```text
Cart Service
    |
    v
Redis as primary state
```

The second design requires:

```text
Persistence
Backup
Recovery
Replication
Data-loss analysis
```

Do not automatically classify every Redis workload as "just cache."

---

# 66. Product Cache Example

Recommended conceptual flow:

```text
GET /products/1001

       |
       v
Redis:
product:v1:1001
       |
   +---+---+
   |       |
 HIT      MISS
   |       |
   |       v
   |     RDS
   |       |
   |       v
   |     Redis SET
   |       |
   +---+---+
       |
       v
    Response
```

---

# 67. Product Update

```text
PUT /products/1001
       |
       v
PostgreSQL UPDATE
       |
       v
Redis DELETE
       |
       v
Return success
```

Next GET:

```text
Redis MISS
   ↓
RDS
   ↓
Fresh data
   ↓
Redis SET
```

This avoids keeping the old cached object indefinitely.

---

# 68. Cache-Aside Race Condition

Consider:

```text
Request A:
Read Redis → MISS

Request B:
Update PostgreSQL → new price

Request A:
Read old DB value
       ↓
SET old value into Redis
```

Now Redis may contain stale data.

This demonstrates that cache invalidation is not merely:

```text
GET → SET
```

It can involve concurrency and ordering.

For highly sensitive data such as pricing/inventory, design the cache strategy carefully.

---

# 69. Pricing and Inventory

These are not ordinary cache candidates.

For example:

```text
Product description
```

can tolerate some staleness.

But:

```text
Inventory = 0
```

versus:

```text
Inventory = 10
```

can affect whether an order can be accepted.

Therefore inventory caching requires stronger consistency considerations.

---

# 70. Redis for Sessions

Redis is commonly used for session/state data.

Example:

```text
session:<id>
```

Architecture:

```text
User
 ↓
ALB
 ↓
Any EKS Pod
 ↓
Redis
 ↓
Session
```

This can avoid sticky sessions.

But session data still needs:

```text
TTL
Security
Memory planning
Failover behavior
```

---

# 71. Redis for Rate Limiting

Redis can also support rate limiting.

Conceptually:

```text
user:123:requests
```

with expiration.

Example:

```text
100 requests
per
60 seconds
```

This can be implemented using Redis atomic operations or appropriate libraries.

The exact implementation depends on the API gateway/application architecture.

---

# 72. Redis Security Checklist

```text
[ ] Private subnet
[ ] Security Group restricted
[ ] TLS where required
[ ] Authentication enabled where required
[ ] Secrets Manager
[ ] No credentials in Git
[ ] No public endpoint
[ ] Encryption at rest
[ ] IAM access restricted
[ ] Monitoring enabled
```

---

# 73. Redis Connectivity Test

From an approved EKS debugging environment:

```bash
kubectl exec -it \
  -n ecommerce-prod \
  deploy/order-service \
  -- sh
```

Then test DNS:

```bash
getent hosts "$REDIS_HOST"
```

Test TCP:

```bash
nc -vz "$REDIS_HOST" "$REDIS_PORT"
```

If TLS is enabled, use the appropriate TLS-aware Redis client/test command rather than assuming plaintext Redis.

---

# 74. Redis Troubleshooting Flow

```text
Redis request failing
        |
        v
DNS resolves?
        |
       Yes
        |
        v
TCP connection?
        |
       Yes
        |
        v
TLS/authentication?
        |
       Yes
        |
        v
Redis available?
        |
       Yes
        |
        v
Application/client issue
```

---

# 75. If DNS Fails

Check:

```bash
kubectl exec \
  -n ecommerce-prod \
  deploy/order-service \
  -- getent hosts "$REDIS_HOST"
```

Then investigate:

```text
CoreDNS
VPC DNS
Network configuration
Endpoint value
```

---

# 76. If TCP Fails

Check:

```text
Redis Security Group
EKS node/pod networking
Subnet
Route
Network ACL
Redis endpoint
Port
```

---

# 77. If Authentication Fails

Check:

```text
Secret value
Secret version
Redis auth configuration
TLS configuration
Application environment
External Secrets Operator
```

Do not print Redis credentials into logs.

---

# 78. If Redis Is Slow

Investigate:

```text
CPU
Memory
Network
Commands/sec
Large values
Slow commands
Hot keys
Evictions
Client connections
```

---

# 79. Hot Key

A hot key is a single key accessed extremely frequently.

Example:

```text
product:homepage
```

If:

```text
100,000 requests/sec
```

all target one key, that key can become a bottleneck depending on the architecture.

Mitigations may include:

```text
Local caching
Replication/read scaling
Key distribution
Request coalescing
Content design
```

Do not blindly duplicate keys without understanding consistency implications.

---

# 80. Large Key / Big Value Problem

A single huge Redis value can cause:

```text
Memory pressure
Network latency
Serialization overhead
Long command execution
```

Prefer appropriately sized objects.

Monitor large keys using approved Redis diagnostic tooling.

Avoid running expensive diagnostic commands against production without assessing their impact.

---

# 81. Redis Commands

Basic diagnostics may include:

```text
PING
INFO
MEMORY USAGE
TTL
GET
SET
DEL
```

Avoid using expensive operations such as broad key scans or blocking commands indiscriminately on production.

For key discovery, prefer:

```text
SCAN
```

over:

```text
KEYS *
```

because `KEYS *` can block the server while scanning a large keyspace.

---

# 82. Redis TTL Inspection

For a key:

```text
TTL product:v1:1001
```

Interpret the result according to Redis semantics.

For example:

```text
positive number
```

means remaining TTL.

A special negative result can indicate:

```text
key does not exist
```

or:

```text
key exists without expiration
```

depending on the returned value.

---

# 83. Keys Without TTL

If cache keys are intended to expire, keys without TTL may indicate a bug.

Example:

```text
product:v1:1001
TTL = -1
```

If this key was supposed to have a TTL, investigate the application code.

---

# 84. Redis Observability

Our final observability architecture:

```text
ElastiCache
    |
    +---- CloudWatch metrics
    |
    +---- AWS events
    |
    +---- Application Redis metrics
    |
    +---- Application logs
    |
    v
Grafana / Alerting
```

---

# 85. Important Dashboard

Create a Redis dashboard with:

```text
REDIS
────────────────────────────

Hit Ratio
Misses
Hits

Memory
────────────────────────────

Used Memory
Free Memory
Evictions

Connections
────────────────────────────

Connected Clients
New Connections

Performance
────────────────────────────

Commands/sec
CPU
Network
Latency

Application
────────────────────────────

Redis Errors
Redis Timeouts
Cache Fallbacks
```

---

# 86. Alerts

Potential alerts:

```text
Redis memory high
Redis evictions increasing
Redis CPU sustained high
Redis connection count high
Redis latency high
Redis replication unhealthy
Cache hit ratio unexpectedly low
Redis endpoint unavailable
```

Thresholds should be based on measured baseline and service requirements.

---

# 87. Cache Fallback Metric

The application should expose a metric such as:

```text
redis_fallback_total
```

Conceptually:

```text
Redis unavailable
      ↓
Application falls back to RDS
      ↓
Increment metric
```

This is extremely useful.

You may see:

```text
Redis errors = increasing
RDS reads = increasing
RDS CPU = increasing
```

This connects the Redis incident to its downstream impact.

---

# 88. Redis Failure Cascade

A production incident may look like:

```text
Redis failure
     ↓
Cache misses
     ↓
RDS read traffic increases
     ↓
RDS CPU increases
     ↓
RDS latency increases
     ↓
Application latency increases
     ↓
HPA scales Pods
     ↓
More RDS connections
     ↓
RDS becomes overloaded
```

This is why caching must be treated as part of the entire architecture.

---

# 89. Preventing the Cascade

Possible protections:

```text
Cache fallback limits
Request rate limiting
Circuit breaker
RDS capacity headroom
Connection pool limits
HPA limits
Query optimization
Cache warming
Graceful degradation
```

The correct combination depends on the service.

---

# 90. Cache Warming

After a Redis restart/failure:

```text
Cache empty
```

Instead of waiting for normal traffic to repopulate it, selected high-value data can be preloaded.

Example:

```text
Popular products
Configuration
Frequently accessed catalog data
```

Be careful:

```text
Cache warming
      ↓
Huge number of requests
      ↓
Database spike
```

Therefore warming itself must be rate-limited.

---

# 91. Redis Failure Test

In a controlled environment:

```text
Normal traffic
      ↓
Redis healthy
      ↓
Disable/fail Redis
      ↓
Observe application
      ↓
Measure RDS impact
      ↓
Restore Redis
      ↓
Observe recovery
```

We will implement the detailed recovery procedure in:

```text
111-Redis-Failure-Cache-Recovery.md
```

---

# 92. Production Redis Architecture

Our target:

```text
                         EKS
                          |
              +-----------+-----------+
              |                       |
              v                       v
        order-service          other services
              |
              |
              v
       Redis Client
              |
              v
      Amazon ElastiCache
              |
        +-----+-----+
        |           |
       AZ-A        AZ-B
      Primary     Replica
        |
        +---- managed failover
```

---

# 93. Redis and Database Architecture

```text
                 Application
                 /          \
                /            \
               v              v
            Redis            RDS
             |                |
       Performance        Source of Truth
             |
       Cache / Session
```

This separation is critical.

---

# 94. Production Checklist

## Infrastructure

```text
[ ] ElastiCache selected
[ ] Private subnets
[ ] Multi-AZ/failover architecture where required
[ ] Appropriate node type
[ ] Capacity tested
[ ] Security Group configured
[ ] Encryption enabled
[ ] TLS configured where required
[ ] Authentication configured
```

## Application

```text
[ ] Redis client configured
[ ] Cache-aside implemented
[ ] TTL implemented
[ ] Cache invalidation implemented
[ ] Serialization defined
[ ] Timeout configured
[ ] Retry policy defined
[ ] Redis fallback defined
[ ] Idempotency considered
```

## Performance

```text
[ ] Hit ratio monitored
[ ] Misses monitored
[ ] Evictions monitored
[ ] Memory monitored
[ ] Connections monitored
[ ] Latency monitored
[ ] Hot keys considered
[ ] Large values considered
```

## Operations

```text
[ ] Redis failure runbook
[ ] Cache recovery runbook
[ ] RDS fallback tested
[ ] Redis failure tested
[ ] Monitoring verified
[ ] Alerts verified
```

---

# 95. Final Mental Model

Remember:

```text
Redis is not just "a fast database."
```

In our architecture it is a performance and state-management component whose failure can affect the database.

The complete dependency chain is:

```text
Traffic
   ↓
EKS
   ↓
Application
   ↓
Redis
   |
   +---- HIT ----> Fast response
   |
   +---- MISS ---> RDS
                   |
                   v
                 Redis
```

And during Redis failure:

```text
Redis failure
     ↓
Cache misses
     ↓
RDS load increases
     ↓
Database capacity becomes important
     ↓
Application resilience becomes important
```

Therefore Redis implementation must be designed together with:

```text
RDS
HPA
Connection pools
Observability
Failure handling
```

---

# 96. Next Chapter

**111-Redis-Failure-Cache-Recovery.md**

We will perform the complete Redis failure scenario:

```text
Normal traffic
      ↓
Redis healthy
      ↓
Redis failure
      ↓
Cache misses
      ↓
RDS load increases
      ↓
Application degradation
      ↓
Fallback protection
      ↓
Redis recovery
      ↓
Cache repopulation
      ↓
System returns to normal
```

We will also cover:

* Redis failure testing
* cache fallback
* circuit breaker
* cache stampede during recovery
* RDS protection
* cache warming
* Redis failover
* application retry behavior
* monitoring
* alerts
* incident timeline
* recovery validation
* production runbook

```
```
