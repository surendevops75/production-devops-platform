# 48 — Cost Optimization

> **Enterprise E-Commerce Platform — AWS EKS Production Guide**

This chapter explains how to control AWS and Kubernetes costs **without damaging availability, performance, security, or disaster recovery**.

The goal is not simply to make the infrastructure cheaper. The goal is:

> **Run the required production workload at the lowest sustainable cost while maintaining the required reliability and performance.**

---

## 1. Platform Cost Architecture

Our platform contains:

```text
                         AWS
                          |
          +---------------+----------------+
          |               |                |
         EKS             RDS          ElastiCache
          |               |                |
     EC2/Karpenter     PostgreSQL        Redis
          |
   +------+------+ 
   |             |
App Nodes     System Nodes
   |
21 Microservices
```

Additional cost sources:

```text
Internet
   |
CloudFront
   |
WAF
   |
ALB
   |
EKS
   |
   +--> ECR
   +--> S3
   +--> SQS/SNS
   +--> CloudWatch
   +--> Elasticsearch
   +--> Logstash
   +--> Kibana
   +--> Prometheus
   +--> Grafana
   +--> OpenTelemetry
   +--> Jaeger
   +--> NAT Gateway
   +--> Data Transfer
```

The important point is:

**EKS worker nodes are only one part of the bill.**

---

# 2. Where Does the Money Go?

For this platform, major cost categories are:

| Component     | Main cost driver                     |
| ------------- | ------------------------------------ |
| EKS           | Cluster/control-plane fee            |
| EC2           | Worker-node compute                  |
| Karpenter     | Node capacity                        |
| RDS           | DB instance + storage + I/O          |
| ElastiCache   | Cache nodes                          |
| NAT Gateway   | Hourly + processed data              |
| ALB           | Load balancer + traffic              |
| CloudFront    | Data transfer/requests               |
| WAF           | Requests/rules                       |
| EBS           | Persistent storage                   |
| Elasticsearch | Compute + storage                    |
| Logstash      | Compute                              |
| Prometheus    | Compute + storage                    |
| Grafana       | Compute                              |
| Jaeger        | Compute + trace storage              |
| CloudWatch    | Logs/metrics ingestion and retention |
| ECR           | Image storage                        |
| S3            | Storage + requests + transfer        |
| SQS/SNS       | Requests/data                        |
| Data transfer | Network traffic                      |

---

# 3. First Rule — Don't Optimize Blindly

Before changing infrastructure:

```text
Measure
   ↓
Identify expensive resource
   ↓
Understand why it is expensive
   ↓
Check performance/reliability impact
   ↓
Optimize
   ↓
Measure again
```

Never do:

```text
AWS bill high
    ↓
Reduce everything
```

That can create:

```text
Cost ↓
Reliability ↓
Performance ↓
Incidents ↑
Customer impact ↑
```

---

# 4. EKS Control Plane Cost

EKS has a cluster management/control-plane charge.

If you have:

```text
DEV EKS
QA EKS
STAGING EKS
PROD EKS
```

you are paying for multiple clusters.

For example, a company might decide:

```text
Development
    ↓
one EKS cluster

QA
    ↓
one EKS cluster

Staging
    ↓
one EKS cluster

Production
    ↓
dedicated production EKS cluster
```

Production should not be merged into a development environment merely to save the cluster fee.

Production isolation can be more important than the relatively small control-plane saving.

---

# 5. Biggest EKS Cost — Worker Nodes

Suppose we have:

```text
21 microservices

Each:
requests:
  cpu: 250m
  memory: 512Mi
```

If we blindly configure:

```text
cpu: 2
memory: 4Gi
```

for every service, Kubernetes may reserve significantly more capacity than applications actually need.

That causes:

```text
Low utilization
      ↓
More nodes
      ↓
Higher EC2 cost
```

---

# 6. Resource Requests Are Extremely Important

Example:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

The scheduler primarily uses **requests** when deciding where Pods can run.

Therefore:

```text
Bad requests
    ↓
Poor bin packing
    ↓
More nodes
    ↓
Higher cost
```

But don't artificially reduce requests just to save money.

Example:

```yaml
requests:
  cpu: 10m
  memory: 32Mi
```

for a Java service that actually needs 500m CPU and 700Mi memory is dangerous.

It can cause:

* CPU throttling
* memory pressure
* OOMKills
* unstable scheduling
* poor latency

---

# 7. Measure Actual Usage

Use Kubernetes metrics:

```bash
kubectl top pods -n production
```

Example:

```text
NAME                         CPU     MEMORY
order-service-abc            180m    420Mi
order-service-def            210m    450Mi
payment-service-abc          600m    900Mi
catalog-service-abc           90m    300Mi
```

Compare:

```text
Requested
vs
Actual
vs
Peak
```

Don't size only from today's average.

Consider:

```text
Normal traffic
Peak traffic
Deployment overhead
Node failure
AZ failure
Autoscaling
```

---

# 8. Requests vs Limits

A common production pattern:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

But there is no universal ratio.

Each service should be measured.

For Java services especially:

```text
Container memory
    |
    +--> JVM heap
    +--> Metaspace
    +--> Threads
    +--> Native memory
    +--> JVM overhead
```

Therefore:

```text
-Xmx ≠ container memory limit
```

You need headroom.

---

# 9. Bin Packing

Suppose:

```text
Node capacity:
4 CPU
16 GiB memory
```

Pod A:

```text
1 CPU
2 GiB
```

Pod B:

```text
1 CPU
2 GiB
```

Pod C:

```text
1 CPU
2 GiB
```

Pod D:

```text
1 CPU
2 GiB
```

They fit reasonably well.

But if requests are:

```text
Pod A = 2 CPU
Pod B = 2 CPU
Pod C = 2 CPU
```

you may need:

```text
3 nodes
```

instead of:

```text
2 nodes
```

This is called **bin packing efficiency**.

Kubernetes scheduling + Karpenter configuration should work together to improve utilization.

---

# 10. Karpenter

Karpenter can dynamically provision nodes based on pending workload requirements.

Architecture:

```text
Pod Pending
     |
     v
Scheduler
     |
     v
Karpenter
     |
     v
Select suitable EC2 capacity
     |
     v
Create node
     |
     v
Pod scheduled
```

When capacity is no longer needed, Karpenter can consolidate nodes depending on configuration and disruption policies.

This can significantly improve elasticity.

---

# 11. Karpenter Cost Controls

A production NodePool should constrain:

```text
Instance families
Instance sizes
Architecture
Capacity type
Availability zones
Operating system
Node limits
Disruption policy
```

For example:

```text
Allowed:
m/c/r families
amd64
On-Demand + Spot
3 AZs
```

Don't allow every EC2 instance type without reason.

Otherwise Karpenter can make choices that don't match your operational standards.

---

# 12. On-Demand vs Spot

Use:

```text
On-Demand
```

for workloads that require stable capacity.

Use:

```text
Spot
```

for workloads that tolerate interruption.

Good Spot candidates:

```text
CI runners
Batch jobs
Stateless workers
Non-critical processing
Some asynchronous consumers
```

Be careful with:

```text
Critical production workloads
Stateful databases
Critical singleton components
```

unless the architecture explicitly supports interruption.

---

# 13. System Nodes vs Application Nodes

A useful architecture:

```text
EKS
 |
 +-- System NodePool
 |      |
 |      +-- CoreDNS
 |      +-- AWS Load Balancer Controller
 |      +-- monitoring components
 |      +-- logging components
 |
 +-- Application NodePool
        |
        +-- order-service
        +-- payment-service
        +-- catalog-service
        +-- ...
```

This gives better control over:

* capacity
* scheduling
* disruption
* scaling
* cost allocation

Use:

```text
taints
+
tolerations
+
node selectors
+
affinity
```

where appropriate.

---

# 14. Don't Overprovision System Nodes

A common mistake:

```text
3 large nodes
```

for system workloads that actually require very little capacity.

Measure:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -A
```

Then size system capacity appropriately.

But maintain enough redundancy for:

```text
node failure
AZ failure
rolling upgrades
```

---

# 15. NAT Gateway — Hidden Cost

NAT Gateway can become surprisingly expensive.

Architecture:

```text
Private EKS Node
      |
      v
NAT Gateway
      |
      v
Internet Gateway
      |
      v
Internet
```

If thousands of Pods continuously download:

```text
Docker layers
packages
AWS APIs
external APIs
```

traffic can pass through NAT.

You pay for:

```text
NAT hourly usage
+
data processing
```

---

# 16. VPC Endpoints

For AWS services used heavily from private subnets, VPC endpoints can reduce unnecessary NAT traffic.

Examples include:

```text
S3
ECR
CloudWatch
Secrets Manager
STS
SQS
SNS
```

Exact endpoint requirements depend on the AWS services and connectivity pattern.

Architecture:

```text
EKS
 |
Private Subnet
 |
 +------> VPC Endpoint ------> AWS Service
 |
 +------> NAT Gateway -------> Internet
```

This can reduce NAT data processing and improve private connectivity.

But:

> **VPC endpoints themselves also have costs for some endpoint types.**

Therefore measure the actual traffic before adding dozens of endpoints.

---

# 17. ALB Cost

Our architecture:

```text
Internet
   |
CloudFront
   |
WAF
   |
ALB
   |
EKS
```

An ALB has:

* hourly/usage components
* Load Balancer Capacity Unit-related usage
* data processing considerations

Avoid creating an ALB for every microservice.

Prefer:

```text
One/few external ALBs
       |
Ingress rules
       |
Services
```

For example:

```text
shop.example.com/orders
shop.example.com/products
shop.example.com/cart
```

rather than:

```text
21 services
   ↓
21 internet-facing ALBs
```

---

# 18. CloudFront

CloudFront can reduce origin load by caching appropriate content.

For:

```text
Static assets
Images
JavaScript
CSS
Product images
```

CloudFront can serve content from edge locations.

Architecture:

```text
User
 |
CloudFront
 |
 +--> Cache HIT
 |
 +--> Cache MISS
          |
          v
         ALB
```

Caching can reduce:

```text
ALB traffic
EKS traffic
application CPU
```

But don't cache dynamic/private responses incorrectly.

---

# 19. WAF

WAF adds security and cost.

Don't remove security controls simply because they cost money.

Instead:

```text
Optimize rules
Remove unnecessary rules
Tune logging
Use managed rules appropriately
Monitor request volume
```

Security and cost need to be optimized together.

---

# 20. RDS Cost

RDS costs can come from:

```text
DB instance
Storage
IOPS
Backups
Read replicas
Data transfer
Monitoring
```

Our primary database design:

```text
EKS
 |
Order Service
 |
RDS PostgreSQL
```

Don't automatically create:

```text
21 large RDS instances
```

because there are 21 microservices.

Database-per-service means:

> **logical ownership/isolation**, not necessarily one large dedicated database instance per service.

---

# 21. RDS Right-Sizing

Monitor:

```text
CPUUtilization
FreeableMemory
DatabaseConnections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
FreeStorageSpace
NetworkThroughput
```

Example:

```text
RDS instance:
CPU average = 8%
Memory = plenty
Connections = low
IOPS = low
```

This may indicate overprovisioning.

But check:

```text
peak traffic
failover requirements
connection bursts
query latency
maintenance
replica requirements
```

before resizing.

---

# 22. RDS Storage

Don't simply allocate:

```text
1 TB
```

because "production needs more space."

Measure:

```text
Current usage
Growth/day
Growth/month
Retention
Backup requirements
```

Then calculate capacity with headroom.

Also monitor:

```text
FreeStorageSpace
```

continuously.

---

# 23. RDS Read Replicas

If read traffic is high:

```text
Application
 |
 +----> Primary
 |
 +----> Read Replica
```

Read replicas increase cost.

Don't create one unless:

```text
read workload
+
latency
+
primary resource pressure
```

justify it.

---

# 24. ElastiCache Cost

Redis/Valkey cost comes from:

```text
node count
node size
replicas
Multi-AZ
data transfer
```

Don't use Redis for everything.

Correct:

```text
RDS
 ↓
Source of truth

Redis
 ↓
Cache
```

If cache hit rate is poor:

```text
Redis cost ↑
Application DB load remains high
```

Optimize:

```text
TTL
key design
cache hit ratio
payload size
eviction strategy
```

before simply increasing Redis capacity.

---

# 25. EBS Cost

Every persistent workload can consume:

```text
EBS storage
IOPS
throughput
snapshots
```

Examples:

```text
Elasticsearch
PostgreSQL in EKS
RabbitMQ
Kafka
Prometheus
Jaeger
```

Storage sizing must consider:

```text
used space
growth
replicas
retention
snapshots
recovery
```

Do not treat:

```text
100Gi PVC
```

as free capacity.

It is billable storage.

---

# 26. Elasticsearch Is a Major Cost Area

Our ELK architecture:

```text
Fluent Bit
    |
Logstash
    |
Elasticsearch
    |
Kibana
```

Logs can grow extremely quickly.

For example:

```text
21 services
×
3 replicas
×
many requests
×
multiple log lines
```

can generate huge volumes.

---

# 27. Log Cost Control

Don't solve log growth only by increasing Elasticsearch.

Instead:

```text
Application
   ↓
Structured logs
   ↓
Remove unnecessary DEBUG logs
   ↓
Filter noisy logs
   ↓
Retention policy
   ↓
Delete/archive old data
```

Example:

```text
Production:
INFO

Troubleshooting:
temporary DEBUG

After incident:
return to INFO
```

Avoid:

```text
Permanent DEBUG in production
```

unless justified.

---

# 28. Elasticsearch Retention

Example policy:

```text
Hot:
7 days

Warm:
30 days

Archive:
S3

Delete:
after compliance-defined period
```

Actual retention depends on:

* compliance
* incident investigation
* business requirements
* storage budget

Don't choose retention solely by cost.

---

# 29. Logstash Cost

Logstash runs JVM workloads.

If log volume increases:

```text
CPU ↑
Memory ↑
Workers ↑
Replicas ↑
```

Monitor:

```text
events in
events out
queue size
pipeline latency
CPU
memory
GC
Elasticsearch output failures
```

Persistent queues can require additional disk.

---

# 30. Prometheus Cost

Prometheus cost is strongly affected by:

```text
Metric count
Scrape interval
Retention
Cardinality
Number of targets
Storage
```

Bad example:

```text
http_request_duration_seconds{
  user_id="123456"
}
```

If there are millions of users:

```text
Cardinality explosion
```

Avoid user IDs as metric labels.

Prefer:

```text
service
method
route
status
```

with controlled values.

---

# 31. Scrape Interval

Suppose:

```text
scrape_interval: 15s
```

For critical application metrics this may be appropriate.

But not every target needs very aggressive scraping.

For some infrastructure metrics:

```text
30s
60s
```

may be sufficient.

Reducing scrape frequency can reduce:

```text
samples
storage
CPU
network
```

but don't compromise monitoring requirements.

---

# 32. Grafana Cost

Grafana itself is generally not the largest cost.

The expensive part is often the systems behind it:

```text
Prometheus
CloudWatch
Elasticsearch
Jaeger
```

Poor dashboards can create expensive queries.

Avoid dashboards that continuously execute:

```text
very large time range
+
high-cardinality queries
+
many panels
+
short refresh interval
```

Example:

```text
Refresh every 5 seconds
```

for a dashboard querying months of Elasticsearch data is unnecessary.

---

# 33. Jaeger / OpenTelemetry Cost

Tracing can become expensive because every request can generate multiple spans.

Example:

```text
User request
 |
API Gateway
 |
Order Service
 |
Catalog Service
 |
Payment Service
 |
Inventory Service
 |
RDS
 |
SQS
```

One request can produce many spans.

At high traffic:

```text
Requests ↑
Spans ↑
Storage ↑
Network ↑
```

---

# 34. Sampling

Instead of retaining every trace:

```text
100% traces
```

we can use sampling.

For example:

```text
Normal successful traffic
    ↓
small percentage

Errors
    ↓
higher priority

Slow requests
    ↓
higher priority
```

This significantly reduces tracing cost while retaining useful diagnostic information.

Exact sampling strategy should be based on operational requirements.

---

# 35. CloudWatch Cost

CloudWatch can become expensive through:

```text
Logs
Metrics
Alarms
Dashboards
Data ingestion
Retention
```

Avoid logging everything forever.

Set retention deliberately:

```text
Development → shorter
QA → shorter
Production → longer
Compliance logs → according to policy
```

Don't confuse:

```text
"stored somewhere"
```

with:

```text
"required forever"
```

---

# 36. ECR Cost

We have:

```text
21 ECR repositories
```

Images accumulate.

Example:

```text
order-service:
  1.0.0
  1.0.1
  1.0.2
  ...
  500 images
```

Old images consume storage.

Use lifecycle policies.

Example concept:

```text
Keep production release images
Keep recent rollback images
Delete old untagged images
Delete obsolete temporary tags
```

Be careful not to delete images required for rollback or compliance.

---

# 37. S3 Cost

S3 costs depend on:

```text
Storage
Requests
Data transfer
Storage class
Replication
```

Potential data:

```text
Terraform state
Backups
Elasticsearch snapshots
Application assets
Reports
Artifacts
```

Use lifecycle policies where appropriate.

Example:

```text
Frequent access
    ↓
Standard

Older data
    ↓
Infrequent Access

Long-term archive
    ↓
Glacier class
```

Choose storage class based on retrieval requirements.

---

# 38. SQS/SNS

SQS/SNS are generally usage-driven.

Cost optimization comes mainly from:

```text
Reducing unnecessary messages
Batching where appropriate
Avoiding retry storms
Correct visibility timeout
Correct DLQ handling
```

A broken consumer can create:

```text
Message
 ↓
failure
 ↓
retry
 ↓
failure
 ↓
retry
 ↓
...
```

This can create unnecessary workload and cost.

---

# 39. Retry Storm

Example:

```text
Payment Service
      |
      v
External API DOWN
      |
      v
10,000 messages
      |
      v
Immediate retry
      |
      v
External API still DOWN
```

Bad.

Use:

```text
Exponential backoff
+
jitter
+
DLQ
+
bounded retries
```

This protects both reliability and cost.

---

# 40. Environment Cost Strategy

Development does not need production capacity.

Example:

```text
DEV

replicas: 1
small nodes
small RDS
short log retention
limited observability retention
```

Production:

```text
PROD

replicas: 3+
Multi-AZ
larger RDS
HA Redis
longer retention
full monitoring
DR
```

Don't copy production infrastructure blindly into DEV.

---

# 41. Non-Production Scheduling

For development environments:

```text
Business hours:
running

Night/weekend:
scaled down
```

For example:

```text
DEV
Monday-Friday
09:00-20:00
```

could use reduced capacity outside working hours.

However:

> Don't automatically shut down shared environments if other teams or integration tests depend on them.

---

# 42. Namespace-Level Cost Allocation

Use labels:

```yaml
metadata:
  labels:
    team: backend
    service: order-service
    environment: production
    cost-center: ecommerce
```

Then costs can be attributed to:

```text
Team
Service
Environment
Business unit
```

This is much better than:

```text
AWS bill = $20,000
```

with nobody knowing who consumed it.

---

# 43. AWS Cost Allocation Tags

Standardize tags:

```text
Environment
Application
Service
Team
Owner
CostCenter
ManagedBy
Criticality
DataClassification
```

Example:

```hcl
default_tags {
  tags = {
    Environment = "prod"
    Application = "ecommerce"
    ManagedBy   = "terraform"
    CostCenter  = "digital-commerce"
  }
}
```

---

# 44. Cost Explorer

Use AWS Cost Explorer to answer:

```text
Which service costs the most?

Which environment increased cost?

Why did this month's bill increase?

Which region costs the most?

Which service generated the increase?
```

Don't just look at:

```text
Total AWS bill
```

Look at:

```text
Trend
Service
Account
Region
Tag
Usage type
```

---

# 45. AWS Budgets

Create budgets such as:

```text
DEV budget
QA budget
PROD budget
Total AWS budget
```

Configure alerts.

For example:

```text
80% → warning
90% → warning
100% → critical
```

The thresholds are organizational decisions.

---

# 46. Cost Anomaly Detection

Cost anomaly detection can identify unusual spending patterns.

Example:

```text
Normal:
$500/day

Suddenly:
$1,500/day
```

Potential causes:

```text
Log storm
NAT traffic
Data transfer
Unexpected EC2 scaling
Elasticsearch growth
Runaway workload
```

Investigate the cause rather than immediately shutting down resources.

---

# 47. Savings Plans / Reserved Capacity

For predictable workloads, AWS commitment-based pricing can reduce compute costs.

Candidates may include:

```text
Stable production EC2 workload
Predictable RDS usage
Predictable compute demand
```

But don't commit based on temporary usage.

First:

```text
Right-size
Measure baseline
Understand growth
Then commit
```

---

# 48. Cost Optimization Hierarchy

Use this order:

```text
1. Remove waste
       ↓
2. Right-size
       ↓
3. Improve utilization
       ↓
4. Autoscale
       ↓
5. Use appropriate capacity types
       ↓
6. Optimize storage
       ↓
7. Optimize network
       ↓
8. Optimize observability
       ↓
9. Commit to predictable usage
```

This is safer than:

```text
"Buy cheaper instances"
```

---

# 49. Production Cost Scenario

Suppose:

```text
21 microservices
3 replicas each
63 Pods
```

Average requests:

```text
250m CPU
512Mi memory
```

Potential node capacity:

```text
Application nodes:
6 × medium/large nodes
```

During traffic increase:

```text
Pods increase
      ↓
Karpenter
      ↓
new nodes
```

During traffic decrease:

```text
Pods decrease
      ↓
unused capacity
      ↓
Karpenter consolidation
      ↓
nodes removed
```

This is much more efficient than permanently running maximum capacity.

---

# 50. Low / Medium / High Traffic

## Low traffic

```text
Pods:
1–2

Nodes:
minimum capacity

Database:
small

Redis:
small

Logs:
short retention
```

## Medium traffic

```text
Pods:
2–5

Nodes:
scale according to requests

HPA:
active

Karpenter:
adds nodes when required

RDS:
normal production capacity
```

## High traffic

```text
Pods:
5–20+

Nodes:
rapid scale-out

Karpenter:
provision additional capacity

Redis:
higher capacity

RDS:
monitor CPU/connections/IOPS

SQS:
consumer scaling

CloudFront:
higher cache utilization
```

The exact values must come from load testing rather than arbitrary configuration.

---

# 51. Cost Optimization vs Dangerous Cost Cutting

### Safe optimization

```text
Remove unused EBS volumes
Remove unused load balancers
Right-size EC2
Tune HPA
Improve node utilization
Use Spot for tolerant workloads
Optimize log retention
Use lifecycle policies
Use appropriate S3 storage classes
Reduce unnecessary NAT traffic
```

### Dangerous cost cutting

```text
Remove Multi-AZ
Remove backups
Disable monitoring
Remove WAF
Run one production replica
Remove PDBs
Disable encryption
Delete rollback images
Disable DR
Use tiny RDS instance without testing
Store production DB inside an unmanaged Pod
```

The objective is:

> **Cost efficiency, not minimum cost at any price.**

---

# 52. Production Cost Review

A monthly review can look like:

```text
AWS COST REVIEW
----------------------------

EKS:
  cluster cost
  EC2 cost
  Karpenter utilization

RDS:
  instance
  storage
  IOPS
  replicas

Redis:
  node utilization
  memory
  hit ratio

Networking:
  NAT
  ALB
  CloudFront
  data transfer

Observability:
  CloudWatch
  Elasticsearch
  Prometheus
  Jaeger

Storage:
  EBS
  S3
  ECR

Messaging:
  SQS
  SNS
```

Then:

```text
Current cost
vs
previous month
vs
traffic
vs
business volume
```

---

# 53. Cost per Business Transaction

A very useful production metric is:

```text
Infrastructure Cost
-------------------
Number of Orders
```

For example:

```text
AWS platform cost = ₹10,00,000

Orders processed = 10,00,000

Cost/order = ₹1
```

The exact calculation should include whatever infrastructure scope the organization chooses.

This is more meaningful than looking only at:

```text
EC2 bill
```

because business traffic may also have increased.

---

# 54. Cost and Reliability Trade-Off

Consider:

```text
One NAT Gateway
```

vs:

```text
NAT Gateway per AZ
```

One can reduce cost.

Multiple AZ-local NAT gateways can improve resilience and reduce cross-AZ dependencies.

Therefore:

```text
Cost
+
Availability
+
Latency
+
Failure behavior
```

must be considered together.

---

# 55. FinOps Model

DevOps + Finance + Engineering should collaborate.

```text
Engineering
    |
    +---- Usage
    |
    +---- Architecture
    |
Finance
    |
    +---- Budget
    |
    +---- Forecast
    |
Business
    |
    +---- Revenue
    |
    +---- Customer volume
```

The goal is:

```text
Engineering decisions
        ↓
measurable infrastructure cost
        ↓
business value
```

---

# 56. Production Cost Optimization Checklist

```text
[ ] AWS resources are tagged
[ ] Cost allocation is configured
[ ] Cost Explorer reviewed
[ ] Budgets configured
[ ] Cost anomaly detection configured
[ ] EKS nodes right-sized
[ ] Pod requests measured
[ ] HPA configured
[ ] Karpenter/consolidation evaluated
[ ] Spot workloads identified
[ ] NAT traffic reviewed
[ ] VPC endpoints evaluated
[ ] ALBs reviewed
[ ] CloudFront caching reviewed
[ ] RDS right-sized
[ ] RDS storage reviewed
[ ] Redis utilization reviewed
[ ] EBS volumes reviewed
[ ] ECR lifecycle configured
[ ] S3 lifecycle configured
[ ] Log retention reviewed
[ ] Elasticsearch shard/storage usage reviewed
[ ] Prometheus cardinality reviewed
[ ] Trace sampling reviewed
[ ] CloudWatch retention reviewed
[ ] Unused resources removed
[ ] Production HA requirements preserved
[ ] Backup/DR requirements preserved
```

---

# 57. Final Production Cost Architecture

```text
                         AWS
                          |
              +-----------+-----------+
              |                       |
          CloudFront                 WAF
              |                       |
              +-----------+-----------+
                          |
                         ALB
                          |
                         EKS
                          |
             +------------+-------------+
             |                          |
       System NodePool            Application NodePool
             |                          |
       Platform Pods             21 Microservices
             |                          |
             |                 +--------+--------+
             |                 |        |        |
             |                RDS     Redis     SQS/SNS
             |
       Observability
             |
     +-------+-------+
     |       |       |
 Prometheus ELK     OTel/Jaeger
     |
   Grafana
```

Cost controls operate across the entire architecture:

```text
Kubernetes
    ↓
Right-sized requests
    ↓
HPA
    ↓
Karpenter
    ↓
Efficient EC2 capacity
    ↓
AWS network optimization
    ↓
Right-sized databases
    ↓
Controlled logs/traces
    ↓
Storage lifecycle
    ↓
Cost allocation
    ↓
Continuous FinOps review
```

## Key takeaway

For this project, the biggest practical cost lessons are:

1. **Don't over-request CPU/memory.**
2. **Use HPA + Karpenter for elastic workloads.**
3. **Use Spot where interruption is acceptable.**
4. **Watch NAT Gateway and data-transfer costs.**
5. **Right-size RDS and Redis using real metrics.**
6. **Control Elasticsearch/log volume aggressively but safely.**
7. **Control Prometheus cardinality and trace sampling.**
8. **Use ECR/S3 lifecycle policies.**
9. **Tag resources for team/service/environment cost attribution.**
10. **Never sacrifice HA, backups, security, or observability blindly for cost savings.**

**Next chapter: `49-Production-Troubleshooting.md`** — we'll build a practical troubleshooting framework starting from **user request → CloudFront/WAF → ALB → Ingress → Service → Pod → application → Redis/RDS/SQS**, including the commands you'd actually run during a production incident.
