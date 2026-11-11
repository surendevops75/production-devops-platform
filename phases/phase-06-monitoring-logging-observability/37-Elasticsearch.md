# 37 — Elasticsearch

> Production-focused DevOps notes for running Elasticsearch as the storage and search layer of the EKS logging platform.

---

## 1. What Is Elasticsearch?

Elasticsearch is a distributed search and analytics engine.

In our logging architecture, Elasticsearch is responsible for:

* storing application logs
* indexing log fields
* searching logs
* filtering logs
* aggregating logs
* supporting Kibana queries
* retaining logs according to lifecycle policies

Our logging pipeline is:

```text
21 Microservices
       │
       ▼
stdout / stderr
       │
       ▼
Kubernetes container logs
       │
       ▼
Fluent Bit
       │
       ▼
Logstash
       │
       ▼
Elasticsearch
       │
       ▼
Kibana
```

The important distinction is:

```text
Fluent Bit  → collection
Logstash    → processing
Elasticsearch → storage + search
Kibana      → visualization
```

---

# 2. Why Elasticsearch?

Suppose our `order-service` produces:

```json
{
  "timestamp": "2026-09-10T12:30:00Z",
  "level": "ERROR",
  "service": "order-service",
  "environment": "production",
  "trace_id": "abc123",
  "order_id": "ORD-1001",
  "message": "Payment processing failed"
}
```

We want questions like:

```text
Show all ERROR logs from order-service.
```

or:

```text
Find logs for order_id = ORD-1001.
```

or:

```text
Show payment failures during the last 30 minutes.
```

or:

```text
Which services generated the most errors today?
```

Elasticsearch is designed for this type of search and aggregation workload.

---

# 3. Elasticsearch Is Not RDS

This distinction is extremely important.

RDS:

```text
Application
    │
    ▼
PostgreSQL
    │
    └── source of truth
```

Elasticsearch:

```text
Applications
    │
    ▼
Logs
    │
    ▼
Elasticsearch
    │
    └── search/analytics
```

Do not treat Elasticsearch as the transactional database for:

* orders
* payments
* customers
* inventory

Our architecture remains:

```text
Order Service
     │
     ▼
RDS PostgreSQL
```

while:

```text
Order Service
     │
     ▼
stdout
     │
     ▼
Fluent Bit
     │
     ▼
Logstash
     │
     ▼
Elasticsearch
```

---

# 4. Elasticsearch in EKS

Our logging namespace:

```text
logging
│
├── Fluent Bit
│
├── Logstash
│
├── Elasticsearch
│
└── Kibana
```

A production Elasticsearch deployment requires careful planning around:

* CPU
* memory
* JVM heap
* storage
* IOPS
* shards
* replicas
* availability zones
* snapshots
* retention
* security
* network
* upgrades

It should not be treated like an ordinary stateless Deployment.

---

# 5. Why Elasticsearch Is Stateful

A normal application Pod can be replaced:

```text
Pod A
  ↓
deleted

Pod B
  ↓
starts
```

The application can retrieve its data from RDS.

Elasticsearch stores its own data.

Therefore:

```text
Elasticsearch
     │
     ▼
persistent storage
```

is fundamental.

For EKS:

```text
Elasticsearch Pod
       │
       ▼
PVC
       │
       ▼
EBS gp3
```

---

# 6. StatefulSet vs Operator

You could theoretically create Elasticsearch using:

```text
StatefulSet
PVC
ConfigMap
Services
```

But manually operating a production Elasticsearch cluster is complex.

A common Kubernetes approach is to use an operator.

For our project:

```text
ECK
=
Elastic Cloud on Kubernetes
```

The operator manages Elasticsearch resources using Kubernetes-native objects.

Conceptually:

```text
GitOps
   │
   ▼
ECK configuration
   │
   ▼
ECK Operator
   │
   ▼
Elasticsearch cluster
```

---

# 7. Why Use an Operator?

Without an operator:

```text
DevOps Engineer
    │
    ├── StatefulSets
    ├── PVCs
    ├── Services
    ├── certificates
    ├── configuration
    ├── upgrades
    └── cluster operations
```

With an operator:

```text
Desired Elasticsearch configuration
             │
             ▼
        ECK Operator
             │
             ▼
      Elasticsearch cluster
```

The operator handles much of the Kubernetes orchestration.

It does **not** eliminate the need for Elasticsearch expertise.

You still need to understand:

* shards
* replicas
* mappings
* disk watermarks
* JVM
* indexing
* query performance
* lifecycle management
* snapshots

---

# 8. Elasticsearch Cluster

A production cluster might look like:

```text
                 Elasticsearch Cluster
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
   ES Node 1          ES Node 2          ES Node 3
     AZ-A               AZ-B               AZ-C
       │                 │                 │
       ▼                 ▼                 ▼
     EBS                EBS                EBS
```

This provides multiple failure domains.

But simply having three Pods does **not** automatically guarantee proper availability.

You must configure:

* roles
* replicas
* shard allocation
* zone awareness
* storage
* resource limits
* cluster settings

---

# 9. Elasticsearch Node Roles

Elasticsearch supports different node responsibilities.

Common roles include:

```text
Master
Data
Ingest
Coordinating
```

Modern Elasticsearch can also use specialized data roles.

For a small/medium logging cluster, nodes can have multiple roles.

For example:

```text
Node 1
 ├── master
 ├── data
 └── ingest

Node 2
 ├── master
 ├── data
 └── ingest

Node 3
 ├── master
 ├── data
 └── ingest
```

For larger environments, roles can be separated.

---

# 10. Master Nodes

Master-eligible nodes participate in cluster management.

They handle things such as:

* cluster state
* node membership
* shard allocation decisions
* index metadata

They are not simply "the database primary."

This is a common misunderstanding.

```text
Master
   ≠
application database primary
```

---

# 11. Data Nodes

Data nodes store shards.

Conceptually:

```text
Index
  │
  ├── Primary shard 0
  ├── Primary shard 1
  ├── Primary shard 2
  └── ...
```

Data nodes host these shards.

Therefore storage and I/O performance matter heavily.

---

# 12. Ingest Nodes

Ingest nodes can perform preprocessing pipelines before documents are indexed.

However, our architecture already has:

```text
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
```

Therefore we should avoid unnecessarily duplicating processing in Elasticsearch ingest pipelines.

Use each component deliberately.

---

# 13. Coordinating Nodes

A coordinating node receives a request and distributes it to the appropriate nodes.

Example:

```text
Kibana
   │
   ▼
Coordinating node
   │
   ├── shard A
   ├── shard B
   └── shard C
```

The responses are combined and returned.

In small clusters, dedicated coordinating nodes are usually unnecessary.

---

# 14. Elasticsearch Index

Think of an index as a logical collection of documents.

For example:

```text
logs-production-order-service
```

contains documents such as:

```json
{
  "service": "order-service",
  "level": "INFO",
  "message": "Order created"
}
```

Another:

```text
logs-production-payment-service
```

contains payment-service logs.

---

# 15. Document

A log event becomes a document.

Example:

```json
{
  "@timestamp": "2026-09-10T12:00:00Z",
  "service": {
    "name": "order-service"
  },
  "log": {
    "level": "ERROR"
  },
  "message": "Database connection timeout"
}
```

Elasticsearch indexes fields so they can be searched and aggregated.

---

# 16. Shards

An Elasticsearch index is divided into shards.

Suppose:

```text
logs-production
```

has:

```text
3 primary shards
```

Conceptually:

```text
Index
 │
 ├── Primary shard 0
 ├── Primary shard 1
 └── Primary shard 2
```

The shards can be distributed across nodes:

```text
Node 1 → shard 0
Node 2 → shard 1
Node 3 → shard 2
```

This allows Elasticsearch to distribute data and workload.

---

# 17. Why Sharding Matters

Suppose:

```text
100 GB/day
```

of logs arrive.

One enormous shard can become problematic.

Instead:

```text
100 GB
 ↓
multiple shards
 ↓
multiple nodes
```

allows distributed indexing and searching.

But:

> More shards does not automatically mean better performance.

Too many shards create overhead.

---

# 18. Primary Shards

Primary shards contain the authoritative copy for their shard.

Example:

```text
Primary 0
Primary 1
Primary 2
```

A document is routed to one primary shard based on Elasticsearch's routing mechanism, typically using the document's routing value.

---

# 19. Replica Shards

Replicas provide additional copies.

Example:

```text
Primary 0
   │
   └── Replica 0

Primary 1
   │
   └── Replica 1

Primary 2
   │
   └── Replica 2
```

Now:

```text
3 primary
+
3 replica
=
6 shard copies
```

Replicas provide:

* redundancy
* failover
* additional search capacity

---

# 20. Never Put Primary and Replica on the Same Failure Domain

Bad:

```text
Node 1
 ├── Primary 0
 └── Replica 0
```

If Node 1 fails:

```text
Primary + Replica
       ↓
      LOST
```

Elasticsearch shard allocation should distribute copies across failure domains.

For AWS:

```text
AZ-A
 └── primary

AZ-B
 └── replica
```

and potentially:

```text
AZ-C
 └── other shard copies
```

Zone-aware allocation should be designed deliberately.

---

# 21. Elasticsearch and EBS

Our EKS storage architecture:

```text
Elasticsearch Pod
       │
       ▼
PVC
       │
       ▼
EBS gp3
```

Why gp3?

It allows independent configuration of:

* storage size
* IOPS
* throughput

compared with older storage approaches.

Actual sizing must come from workload testing.

---

# 22. Important EBS Limitation

EBS volumes are AZ-scoped.

Example:

```text
PVC
 │
 ▼
EBS volume
 │
 ▼
AZ-A
```

The Pod cannot simply attach that same EBS volume simultaneously to a different AZ like shared network storage.

Therefore:

```text
Elasticsearch node
        +
PVC
        +
AZ placement
```

must be designed together.

This is another reason Elasticsearch operators and proper topology configuration matter.

---

# 23. StorageClass

Our EKS cluster can use an EBS CSI StorageClass.

Conceptually:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

`WaitForFirstConsumer` is particularly useful for topology-aware storage provisioning.

The volume can be provisioned considering the Pod's scheduling location.

---

# 24. Elasticsearch Storage Sizing

Suppose:

```text
logs = 150 GB/day
retention = 7 days
```

Raw storage:

```text
150 × 7
=
1,050 GB
```

But we cannot simply provision:

```text
1,050 GB
```

We must account for:

```text
replicas
+
index overhead
+
segment overhead
+
merges
+
metadata
+
temporary space
+
growth
+
safety headroom
```

Therefore actual required storage can be substantially higher.

---

# 25. Disk Watermarks

Elasticsearch uses disk watermarks to protect the cluster.

Conceptually:

```text
Disk usage
   │
   ├── normal
   │
   ├── high watermark
   │
   └── flood-stage
```

As disks become full, Elasticsearch can take protective actions such as preventing allocation or marking indices read-only under severe conditions.

This is why:

> **Free disk space is a critical Elasticsearch production metric.**

---

# 26. Disk Full Scenario

Imagine:

```text
EBS = 500 GB
```

and:

```text
used = 490 GB
```

Now:

```text
Logstash
   ↓
Elasticsearch
```

continues receiving logs.

Eventually:

```text
disk pressure
      ↓
shard allocation problems
      ↓
indexing failures
      ↓
logs stop appearing
```

Kibana might look broken even though the actual root cause is disk capacity.

---

# 27. JVM Heap

Elasticsearch is JVM-based.

Memory is therefore extremely important.

Conceptually:

```text
Node memory
│
├── JVM heap
├── off-heap memory
├── filesystem cache
└── OS/container overhead
```

Do not blindly allocate the entire container memory limit to heap.

Filesystem cache is particularly important for Elasticsearch performance.

---

# 28. Why Filesystem Cache Matters

Elasticsearch uses Lucene underneath.

Lucene relies heavily on filesystem access.

Therefore:

```text
more memory
   ↓
more useful filesystem cache
   ↓
potentially faster searches
```

This is one reason giving Elasticsearch a huge heap is not always the correct optimization.

---

# 29. Heap Pressure

If JVM heap is constantly high:

```text
heap usage ↑
      ↓
GC activity ↑
      ↓
CPU ↑
      ↓
latency ↑
```

Eventually:

```text
OutOfMemoryError
```

or severe instability can occur.

Monitor:

* heap usage
* GC
* old generation
* CPU
* search latency
* indexing latency

---

# 30. Resource Example

A small production starting point might look conceptually like:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "4Gi"

  limits:
    cpu: "2"
    memory: "8Gi"
```

This is **not a universal production size**.

A real cluster should be sized based on:

```text
events/sec
average event size
retention
queries/sec
shard count
replica count
storage performance
```

---

# 31. Elasticsearch on Dedicated Nodes

For larger production workloads, Elasticsearch may use dedicated Kubernetes capacity.

Example:

```text
Application nodes
    │
    └── microservices

Logging nodes
    │
    ├── Elasticsearch
    └── Logstash
```

Use:

```text
node labels
taints
tolerations
affinity
topology spread
```

to control placement.

---

# 32. Why Not Mix Elasticsearch With Applications?

Imagine Elasticsearch suddenly performs a large merge:

```text
Elasticsearch
    ↓
CPU + disk I/O increases
```

If it shares nodes with:

```text
payment-service
order-service
```

the applications may suffer.

Example:

```text
Elasticsearch heavy I/O
        ↓
Node disk latency ↑
        ↓
Application latency ↑
```

Dedicated capacity reduces this blast radius.

---

# 33. Elasticsearch Availability Zones

Production target:

```text
ap-south-1a
    │
    └── Elasticsearch node

ap-south-1b
    │
    └── Elasticsearch node

ap-south-1c
    │
    └── Elasticsearch node
```

The cluster should understand the topology.

Conceptually:

```text
zone awareness
```

helps Elasticsearch distribute shard copies across zones.

---

# 34. Three Nodes Does Not Mean Three AZs Automatically

This is an important interview and production concept.

You can have:

```text
3 Elasticsearch Pods
```

all scheduled on:

```text
AZ-A
```

unless scheduling constraints prevent it.

Therefore:

```text
replicas
+
topology constraints
+
zone awareness
```

must work together.

---

# 35. Kubernetes Topology

Example conceptual constraint:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: elasticsearch
```

This encourages distribution across zones.

The exact constraint should be tested with the cluster's node/AZ capacity.

---

# 36. Index Mapping

Mappings tell Elasticsearch how fields should be interpreted.

Example:

```json
{
  "properties": {
    "service": {
      "properties": {
        "name": {
          "type": "keyword"
        }
      }
    },
    "message": {
      "type": "text"
    }
  }
}
```

Now:

```text
service.name
```

can be used efficiently for exact filtering.

---

# 37. Keyword vs Text

This is extremely important.

### `keyword`

Used for exact values:

```text
service.name
environment
log.level
order.id
```

Example:

```text
service.name = "order-service"
```

### `text`

Used for full-text search:

```text
message
exception
stack_trace
```

Example:

```text
message contains "database timeout"
```

Do not blindly make every field `text`.

---

# 38. Mapping Explosion

A dangerous scenario:

```text
Every application
    ↓
creates arbitrary fields
    ↓
thousands/millions of mappings
```

This can cause:

```text
mapping explosion
```

and cluster instability.

Applications should follow a controlled schema.

---

# 39. ECS-Style Fields

For our logging platform, a standardized event model is valuable.

Conceptually:

```text
@timestamp
log.level
message
service.name
service.version
deployment.environment
trace.id
span.id
host.name
kubernetes.namespace
kubernetes.pod.name
```

Elastic Common Schema (ECS) is one option for standardizing fields.

Using a consistent schema makes cross-service investigation easier.

---

# 40. Timestamp

Every log should have a reliable timestamp.

Example:

```json
{
  "@timestamp": "2026-09-10T12:30:00Z"
}
```

This allows Kibana to perform:

```text
last 5 minutes
last 1 hour
today
custom time range
```

Time synchronization is therefore important across:

* EKS nodes
* applications
* Fluent Bit
* Logstash
* Elasticsearch

---

# 41. Data Streams

Modern Elasticsearch logging architectures can use data streams for time-series-oriented logs.

Conceptually:

```text
logs-production-order-service
       │
       ├── backing index 000001
       ├── backing index 000002
       └── backing index 000003
```

This works well with:

* rollover
* lifecycle management
* time-based retention

For a modern production design, evaluate data streams rather than blindly creating daily indexes.

---

# 42. Index Lifecycle Management

We don't want logs to remain forever.

Example:

```text
Day 0
 ↓
hot
 ↓
Day 7
 ↓
warm
 ↓
Day 30
 ↓
delete
```

The exact phases depend on:

* business requirements
* compliance
* query requirements
* storage budget

---

# 43. Retention Example

Suppose:

```text
Production logs = 7 days
Security logs = 90 days
Audit logs = 365 days
```

Do not apply:

```text
delete everything after 7 days
```

globally.

Different log classes have different requirements.

---

# 44. Hot / Warm / Cold Architecture

Large environments may use different storage tiers.

Conceptually:

```text
New logs
   ↓
Hot
   ↓
Warm
   ↓
Cold
   ↓
Archive/Delete
```

Hot:

```text
frequent searches
fast storage
```

Warm:

```text
less frequent queries
```

Cold/archive:

```text
long-term retention
lower-cost storage
```

The actual Elastic features and AWS storage choices should be selected according to the deployed Elastic version and architecture.

---

# 45. Snapshots

Elasticsearch data should have a recovery strategy.

Snapshots are the primary backup mechanism for Elasticsearch.

Conceptually:

```text
Elasticsearch
      │
      ▼
Snapshot
      │
      ▼
Object storage
```

For AWS environments, S3 is commonly used as the repository target.

---

# 46. Snapshot ≠ Replica

This is extremely important.

Replica:

```text
Primary shard
     │
     └── Replica
```

protects against certain node failures.

Snapshot:

```text
Cluster
   ↓
Snapshot
   ↓
Object storage
```

protects against larger recovery scenarios.

For example:

```text
accidental deletion
cluster corruption
major infrastructure failure
```

You need both availability and recovery planning.

---

# 47. Snapshot Testing

Never say:

> "We have backups."

until restoration has been tested.

Production process:

```text
Snapshot
   ↓
Restore test
   ↓
Validate documents
   ↓
Validate mappings
   ↓
Validate indexes
   ↓
Measure recovery time
```

This gives confidence in:

```text
RPO
RTO
```

---

# 48. RPO

Recovery Point Objective:

> How much data can we afford to lose?

Example:

```text
RPO = 1 hour
```

means the recovery design should aim to lose no more than approximately one hour of acceptable data, subject to the actual backup design.

For logs, acceptable RPO may differ from transactional data.

---

# 49. RTO

Recovery Time Objective:

> How quickly must logging become operational again?

Example:

```text
RTO = 2 hours
```

This drives:

* snapshot strategy
* infrastructure automation
* restore procedures
* capacity planning

---

# 50. Elasticsearch Security

Production Elasticsearch should not be exposed publicly.

Bad:

```text
Internet
   │
   ▼
Elasticsearch :9200
```

Correct:

```text
Private network
      │
      ▼
Elasticsearch
```

Access should be restricted to approved components such as:

```text
Logstash
Kibana
authorized administrators
```

---

# 51. Network Flow

Our intended flow:

```text
Fluent Bit
    │
    ▼
Logstash
    │
    ▼
Elasticsearch :9200
    │
    ▼
Kibana
```

Applications should generally **not** directly write arbitrary documents into Elasticsearch just because Elasticsearch is reachable.

Centralizing ingestion through the logging pipeline gives us better control.

---

# 52. TLS

Production communication should use TLS:

```text
Logstash
   │
   │ HTTPS
   ▼
Elasticsearch
```

and:

```text
User/Admin
    │
    │ HTTPS
    ▼
Kibana
```

Certificates should be managed systematically.

With ECK, certificate management can be integrated into the Kubernetes deployment model.

---

# 53. Authentication

Elasticsearch should require authentication.

Conceptually:

```text
Logstash
    │
    ├── credentials / approved identity
    ▼
Elasticsearch
```

Kibana also authenticates to Elasticsearch.

Credentials should not be:

```text
hard-coded
stored in Git
printed in logs
```

Use:

```text
Kubernetes Secrets
+
External Secrets Operator
+
AWS Secrets Manager
```

where appropriate.

---

# 54. Authorization

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What can you access?
```

For example:

```text
Logstash
  ↓
write logs

Kibana
  ↓
read logs

Administrator
  ↓
cluster administration
```

Do not give Logstash unrestricted administrative privileges simply because it needs to index logs.

---

# 55. Kibana Access

Kibana should normally be internal.

Example:

```text
Corporate network / VPN
          │
          ▼
       Internal ALB
          │
          ▼
        Kibana
```

Avoid:

```text
Internet
   ↓
Kibana
```

unless there is a very deliberate security architecture.

---

# 56. Elasticsearch and Secrets

The security chain:

```text
AWS Secrets Manager
       │
       ▼
External Secrets Operator
       │
       ▼
Kubernetes Secret
       │
       ▼
Logstash / Kibana
```

This is consistent with our earlier secrets-management chapter.

---

# 57. Elasticsearch Monitoring

Prometheus should monitor the Elasticsearch cluster.

Important categories:

```text
Cluster health
Node health
JVM
CPU
Memory
Disk
Indexing
Search
Shards
Thread pools
GC
```

---

# 58. Critical Metrics

Watch:

```text
JVM heap usage
CPU
disk usage
disk watermarks
indexing rate
search rate
search latency
indexing latency
pending tasks
unassigned shards
cluster health
GC
thread-pool rejection
```

---

# 59. Cluster Health

Elasticsearch health commonly has states such as:

```text
green
yellow
red
```

### Green

All primary and replica shards are allocated.

### Yellow

Primary shards are available, but one or more replicas are not allocated.

### Red

At least one primary shard is unavailable.

A yellow cluster may still serve traffic, but it indicates reduced redundancy.

Red requires immediate investigation.

---

# 60. Unassigned Shards

Check cluster allocation when shards are unassigned.

Conceptually:

```text
Index
 ↓
Primary/Replica
 ↓
UNASSIGNED
```

Possible causes:

```text
not enough nodes
disk watermark
allocation rules
AZ constraints
failed node
storage problem
```

Don't simply restart Elasticsearch.

Find the allocation reason first.

---

# 61. Node Failure

Suppose:

```text
Node 2
   ↓
crashes
```

If replicas are correctly configured:

```text
Primary
   +
Replica
```

Elasticsearch can promote an appropriate replica when conditions permit.

Then:

```text
cluster recovers
```

and may redistribute shards.

---

# 62. But Recovery Uses Resources

After a node failure:

```text
Node failure
     ↓
shard recovery
     ↓
disk I/O ↑
network ↑
CPU ↑
```

During recovery:

```text
application logging
+
shard recovery
```

may compete for resources.

This is another reason to size storage and nodes correctly.

---

# 63. Elasticsearch Overload

Symptoms:

```text
high CPU
high JVM
slow searches
slow indexing
rejected requests
large queues
```

Possible causes:

```text
too many logs
too many shards
expensive queries
heavy aggregations
slow EBS
insufficient nodes
large documents
mapping problems
```

---

# 64. Too Many Shards

Suppose:

```text
21 services
×
3 environments
×
daily indexes
×
5 shards
×
2 replicas
```

Shard copies can grow rapidly.

This can create:

```text
cluster-state overhead
memory pressure
recovery overhead
slow management
```

Therefore shard count should be based on data size and workload, not arbitrary defaults.

---

# 65. Large Shards

Very large shards can cause:

```text
slow recovery
long merges
long relocation
large queries
```

Very small shards cause:

```text
too many shards
```

The goal is a sensible shard sizing strategy based on:

* daily ingestion
* retention
* query workload
* node count
* expected growth

---

# 66. Logstash Backpressure + Elasticsearch

Our entire pipeline:

```text
Applications
     ↓
Fluent Bit
     ↓
Logstash
     ↓
Elasticsearch
```

If Elasticsearch becomes slow:

```text
Elasticsearch
     ↓
output slows
     ↓
Logstash queue grows
     ↓
Fluent Bit buffers
     ↓
node disk usage grows
```

Therefore Elasticsearch capacity directly affects the entire logging system.

---

# 67. Example Production Incident

Imagine at 14:00:

```text
Elasticsearch disk = 92%
```

At 14:05:

```text
disk = 95%
```

Then:

```text
indexing errors
```

At the same time:

```text
Logstash queue ↑
```

and:

```text
Fluent Bit buffer ↑
```

Developers report:

> "Kibana isn't showing new logs."

The real incident is:

```text
Elasticsearch storage exhaustion
```

not:

```text
Kibana failure
```

---

# 68. Correct Troubleshooting

Start at the user-visible symptom:

```text
Kibana has no recent logs
```

Trace backwards:

```text
Kibana
  ↓
Elasticsearch
  ↓
Logstash
  ↓
Fluent Bit
  ↓
Kubernetes
  ↓
Application
```

Check:

```text
1. Elasticsearch cluster health
2. disk usage
3. unassigned shards
4. indexing errors
5. Logstash output errors
6. Logstash queue
7. Fluent Bit buffering
8. application log generation
```

---

# 69. Elasticsearch Upgrade

Never treat an Elasticsearch upgrade as:

```bash
kubectl set image ...
```

only.

A production upgrade requires checking:

```text
version compatibility
ECK version
Kubernetes compatibility
plugin compatibility
index compatibility
deprecated APIs
snapshot availability
resource capacity
rollback/recovery plan
```

---

# 70. Upgrade Strategy

Conceptually:

```text
Current cluster
      │
      ▼
Snapshot
      │
      ▼
Compatibility checks
      │
      ▼
Upgrade operator
      │
      ▼
Controlled Elasticsearch upgrade
      │
      ▼
Validate
      │
      ▼
Kibana compatibility
      │
      ▼
Logstash compatibility
```

Do not upgrade components independently without checking the Elastic stack compatibility matrix for the versions you are deploying.

---

# 71. GitOps Upgrade

Our desired flow:

```text
Platform Engineer
       │
       ▼
Git change
       │
       ▼
Pull Request
       │
       ▼
Review
       │
       ▼
Merge
       │
       ▼
ArgoCD
       │
       ▼
EKS
```

The version is pinned.

Avoid:

```yaml
image: elasticsearch:latest
```

Use an approved version.

---

# 72. ECK Architecture

Conceptually:

```text
                   EKS
                    │
                    ▼
               ECK Operator
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
   Elasticsearch            Kibana
     resource                resource
          │
          ▼
 Elasticsearch cluster
```

ECK manages Elasticsearch and Kibana resources through Kubernetes APIs.

---

# 73. Conceptual ECK Resource

Example:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: logging
  namespace: logging
spec:
  version: "<approved-version>"

  nodeSets:
    - name: default
      count: 3

      config:
        node.roles:
          - master
          - data
          - ingest

      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            storageClassName: gp3
            resources:
              requests:
                storage: 100Gi
```

This is a conceptual starting point, not a production-ready configuration.

Production configuration should additionally address:

* resources
* topology
* security
* TLS
* storage
* JVM
* availability
* monitoring
* snapshot repository
* lifecycle policies

---

# 74. Storage Configuration

Conceptually:

```yaml
volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data
    spec:
      storageClassName: gp3
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
```

Again:

```text
100Gi
```

is only an example.

Actual capacity must be calculated.

---

# 75. Production Capacity Formula

A useful planning model:

```text
Daily raw log volume
        ×
retention days
        ×
replication factor
        +
overhead
        +
growth/headroom
```

For example:

```text
150 GB/day
×
7 days
=
1,050 GB raw
```

If two copies are required:

```text
1,050 × 2
=
2,100 GB
```

Then add:

```text
index/segment overhead
+
operational headroom
+
growth
```

Therefore:

```text
~2.1 TB
```

is still not necessarily enough.

---

# 76. Elasticsearch and 21 Microservices

Our platform:

```text
21 services
```

doesn't necessarily mean:

```text
21 Elasticsearch clusters
```

That would be operationally expensive.

A shared production cluster can store:

```text
order logs
payment logs
catalog logs
inventory logs
...
```

with proper:

* index/data-stream design
* access control
* retention
* lifecycle policies

---

# 77. Environment Separation

We should avoid mixing everything without controls.

Conceptually:

```text
Development
   ↓
logs-development-*

QA
   ↓
logs-qa-*

Production
   ↓
logs-production-*
```

Production logs should have stricter:

* retention
* access
* security
* monitoring
* resource allocation

---

# 78. Multi-Cluster Strategy

At larger scale, you may use:

```text
Dev Elasticsearch
        +
Non-prod Elasticsearch
        +
Production Elasticsearch
```

instead of one cluster for everything.

Why?

```text
production isolation
blast-radius reduction
different retention
different capacity
different access controls
```

The correct decision depends on scale and organizational requirements.

---

# 79. Elasticsearch vs OpenSearch

Both are distributed search/analytics technologies with Elasticsearch/Lucene heritage, but they are separate projects/products with different versioning and feature ecosystems.

For this project we are using:

```text
Elasticsearch
+
Logstash
+
Kibana
```

because the learning target is the Elastic stack.

In AWS environments, Amazon OpenSearch Service is another valid architectural option, but it should not be mixed into this exact stack without understanding compatibility differences.

---

# 80. Why We Are Learning Self-Managed Elasticsearch

Production organizations have several choices:

```text
Option 1
Self-managed Elasticsearch on EKS

Option 2
Elastic Cloud

Option 3
Another managed search platform
```

Learning ECK on EKS teaches:

* Kubernetes stateful workloads
* storage
* AZ distribution
* JVM tuning
* shard management
* lifecycle management
* snapshots
* cluster operations

These concepts are valuable even if the organization later chooses a managed service.

---

# 81. Production Tool Placement

For our architecture:

| Component       | Location               |
| --------------- | ---------------------- |
| Fluent Bit      | EKS DaemonSet          |
| Logstash        | EKS logging namespace  |
| Elasticsearch   | EKS logging platform   |
| Kibana          | EKS logging platform   |
| Prometheus      | EKS monitoring         |
| Grafana         | EKS monitoring         |
| Alertmanager    | EKS monitoring         |
| ArgoCD          | EKS platform           |
| Terraform       | CI/control environment |
| ECR             | AWS managed            |
| RDS             | AWS managed            |
| Secrets Manager | AWS managed            |
| S3              | AWS managed            |

The important distinction:

> Elasticsearch is a stateful platform workload, not an ordinary business microservice.

---

# 82. Production Checklist

Before calling Elasticsearch production-ready:

```text
[ ] Multi-node cluster
[ ] Multi-AZ distribution
[ ] Correct node roles
[ ] Persistent EBS storage
[ ] gp3 storage appropriately sized
[ ] EBS CSI configured
[ ] Storage headroom
[ ] Disk watermarks monitored
[ ] JVM heap sized
[ ] CPU/memory requests configured
[ ] Dedicated nodes where justified
[ ] Shard strategy defined
[ ] Replica strategy defined
[ ] Index/data-stream naming standardized
[ ] Mapping strategy defined
[ ] ECS/schema strategy considered
[ ] ILM/retention configured
[ ] Snapshots configured
[ ] Restore tested
[ ] TLS enabled
[ ] Authentication enabled
[ ] Authorization configured
[ ] NetworkPolicy configured
[ ] Kibana not publicly exposed
[ ] Prometheus monitoring
[ ] Alerts configured
[ ] Upgrade procedure documented
[ ] Incident runbook documented
```

---

# 83. Final Production Architecture

Our complete logging stack now looks like:

```text
                         EKS
                          │
          ┌───────────────┴───────────────┐
          │                               │
    Application Pods                Logging Platform
          │                               │
          ▼                               │
    stdout/stderr                        │
          │                               │
          ▼                               │
    Fluent Bit DaemonSet                 │
          │                               │
          ▼                               │
     Logstash Service ◄──────────────────┘
          │
          ▼
   Logstash Processing
          │
          ├── Parse
          ├── Normalize
          ├── Filter
          ├── Enrich
          └── Route
          │
          ▼
 ┌─────────────────────────────┐
 │ Elasticsearch Cluster       │
 │                             │
 │ Node 1 → AZ-A → EBS         │
 │ Node 2 → AZ-B → EBS         │
 │ Node 3 → AZ-C → EBS         │
 └─────────────────────────────┘
          │
          ▼
        Kibana
```

Alongside:

```text
Elasticsearch
      │
      ├── Prometheus → Grafana
      │
      └── Snapshots → S3
```

---

# 84. Final Mental Model

Think of Elasticsearch as **a distributed log-search platform**, not simply "a database running in Kubernetes."

```text
                    Elasticsearch
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
    Shards             Nodes            Storage
       │                 │                 │
       ▼                 ▼                 ▼
   distribution       compute            EBS
       │
       ▼
    Replicas
       │
       ▼
   availability
```

And the complete log journey:

```text
Application
    ↓
stdout
    ↓
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

While recovery is:

```text
Elasticsearch
    ↓
Snapshots
    ↓
S3
    ↓
Restore
```

And deployment is:

```text
Git
 ↓
Helm / Kubernetes manifests
 ↓
ArgoCD
 ↓
EKS
 ↓
ECK
 ↓
Elasticsearch
```

**Next file: `38-Kibana.md`**

That chapter will cover how Kibana sits on top of Elasticsearch: **installation through ECK/GitOps, data views/data streams, Discover, KQL, dashboards, visualizations, alerting, RBAC, SSO, production access, troubleshooting, and how a DevOps engineer actually uses Kibana during a production incident.**
