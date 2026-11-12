# 129 — KUBERNETES CAPACITY PLANNING — PRODUCTION IMPLEMENTATION

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Region example:** `ap-south-1`
> **Workloads:** 21 Java + Spring Boot microservices
> **Autoscaling:** HPA + KEDA + Karpenter
> **Database:** Amazon RDS PostgreSQL
> **Cache:** Amazon ElastiCache Redis/Valkey-compatible
> **Messaging:** SQS + SNS + Kafka + RabbitMQ
> **Observability:** Prometheus + Grafana + ELK + OpenTelemetry + Jaeger

---

# 1. Objective

Chapters 127 and 128 taught us how to measure:

```text
Traffic
Latency
Errors
CPU
Memory
Database
Redis
Messaging
JVM
```

Now we answer:

> **How much Kubernetes infrastructure do we actually need to run this workload safely?**

Capacity planning connects application performance to infrastructure sizing:

```text
Traffic
   ↓
Requests per second
   ↓
Requests handled per pod
   ↓
Required replicas
   ↓
CPU / memory per pod
   ↓
Node capacity
   ↓
Number of nodes
   ↓
AZ distribution
   ↓
Cluster capacity
   ↓
Cost + headroom
```

---

# 2. Capacity Planning Is Not Guesswork

Bad approach:

```text
21 services
↓
Create 20 EC2 nodes
↓
Done
```

Why?

Because we don't know:

* how much CPU each service needs
* how much memory each service needs
* how many requests one pod can handle
* how much traffic each service receives
* which services scale independently
* how much database capacity is available
* how much spare capacity is required

Instead:

```text
Measure
   ↓
Model
   ↓
Test
   ↓
Validate
   ↓
Size
```

---

# 3. The Main Capacity Questions

For our EKS cluster we need to answer:

### Application

```text
How many pods are required?
```

### Kubernetes

```text
How much CPU?
How much memory?
```

### Nodes

```text
How many EC2 nodes?
```

### Availability

```text
How many nodes per AZ?
```

### Scaling

```text
What happens during traffic spikes?
```

### Failure

```text
What happens if one node fails?
```

### Cost

```text
How much infrastructure do we need?
```

---

# 4. Start With Traffic

Suppose performance testing gives us:

```text
Normal traffic = 300 RPS
Peak traffic   = 600 RPS
```

But that doesn't mean every microservice receives 600 RPS.

Example:

```text
Customer request
       │
       ├── catalog-service
       │
       ├── product-service
       │
       ├── pricing-service
       │
       └── order-service
```

Different services can have completely different traffic patterns.

---

# 5. Traffic Distribution

Example:

| Service              | Traffic |
| -------------------- | ------: |
| catalog-service      | 300 RPS |
| product-service      | 250 RPS |
| search-service       | 200 RPS |
| pricing-service      | 150 RPS |
| cart-service         |  80 RPS |
| order-service        |  40 RPS |
| payment-service      |  20 RPS |
| notification-service |   async |
| reporting-service    |   async |

These are examples only.

Real values should come from:

* ALB metrics
* application metrics
* Prometheus
* access logs
* business analytics
* load tests

---

# 6. Requests Per Pod

This is one of the most useful capacity concepts.

Suppose:

```text
product-service
```

performance testing shows:

```text
1 pod
=
50 RPS
```

at the required SLO.

Production traffic:

```text
250 RPS
```

Then:

```text
250 / 50
= 5 pods
```

So approximately:

```text
5 pods
```

are needed for steady-state capacity.

But we shouldn't stop there.

---

# 7. Add Headroom

If we need:

```text
5 pods
```

we may run:

```text
6–8 pods
```

depending on:

* SLO
* failure requirements
* traffic variability
* deployment surge
* autoscaling behavior
* AZ failure requirements

For example:

```text
Required = 5
Headroom = 30%

5 × 1.30 = 6.5
```

Round appropriately:

```text
7 pods
```

The exact headroom should come from the service's operating model, not a universal percentage.

---

# 8. Replica Calculation

A simple model:

```text
Required replicas =
Peak RPS / sustainable RPS per pod
```

Example:

```text
Peak RPS = 300

Pod capacity = 50 RPS
```

Therefore:

```text
300 / 50
= 6 pods
```

Then account for:

```text
HA
headroom
rolling deployment
failure
autoscaling
```

---

# 9. CPU-Based Capacity

We also need CPU.

Suppose:

```text
1 pod
CPU request = 500m
```

At normal traffic:

```text
CPU usage = 300m
```

At peak:

```text
CPU usage = 450m
```

If we require:

```text
300 RPS
```

and six pods:

```text
6 × 450m
= 2700m
```

approximately:

```text
2.7 CPU
```

for that service.

---

# 10. Memory-Based Capacity

Suppose each pod requires:

```text
memory request = 512Mi
```

Six pods:

```text
6 × 512Mi
= 3072Mi
```

approximately:

```text
3Gi
```

But we must also account for:

* JVM behavior
* native memory
* sidecars
* startup overhead
* memory spikes
* Kubernetes system workloads

---

# 11. Requests vs Limits

Remember:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

### Request

Used primarily for:

```text
scheduling
capacity planning
```

### Limit

Defines an upper boundary for the container's resource usage, subject to Kubernetes behavior.

Capacity planning should primarily start with:

```text
requests
```

because Kubernetes schedules based on requested resources.

---

# 12. Why Requests Matter

Suppose:

```text
Node:
4 CPU
```

Pods:

```text
Pod A request = 1 CPU
Pod B request = 1 CPU
Pod C request = 1 CPU
Pod D request = 1 CPU
```

Then:

```text
4 CPU requested
```

The node is fully allocated from the scheduler's perspective.

Even if actual usage happens to be:

```text
1.5 CPU
```

you cannot safely treat the remaining requested capacity as guaranteed scheduling capacity.

---

# 13. Node Capacity Is Not Equal to EC2 vCPU

Suppose an EC2 instance has:

```text
8 vCPU
```

We should not assume:

```text
8 CPU
```

is available for application pods.

Some resources are consumed by:

```text
kubelet
kube-proxy
CNI
CoreDNS
monitoring
DaemonSets
system services
```

Therefore calculate:

```text
Allocatable CPU
```

rather than blindly using instance vCPU.

Check:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Capacity
Allocatable
```

---

# 14. Node Memory

Same principle.

A node might have:

```text
32 GiB RAM
```

but application workloads cannot safely consume the entire 32 GiB.

System components require memory.

Therefore use:

```text
Allocatable memory
```

for planning.

---

# 15. Example Node

Suppose:

```text
Node capacity:
8 CPU
32Gi memory
```

and after Kubernetes/system overhead:

```text
Allocatable:
7.5 CPU
29Gi memory
```

If application workloads require:

```text
30 CPU
```

then:

```text
30 / 7.5
= 4 nodes
```

But we still need:

```text
AZ distribution
failure headroom
DaemonSets
system overhead
autoscaling
deployment surge
```

So four is not necessarily the production node count.

---

# 16. Node Packing

Suppose:

```text
Node allocatable:
7.5 CPU
29Gi
```

Service pods:

```text
order-service:
500m CPU
512Mi

product-service:
500m CPU
512Mi

catalog-service:
250m CPU
256Mi
```

The scheduler tries to place pods based on resource requests and constraints.

Conceptually:

```text
Node
├── order pod       500m
├── order pod       500m
├── product pod     500m
├── product pod     500m
├── catalog pod     250m
└── ...
```

---

# 17. CPU vs Memory as the Limiting Resource

Sometimes CPU determines node count.

Example:

```text
Total CPU required = 30
Allocatable CPU/node = 7.5

30 / 7.5 = 4 nodes
```

But memory might require:

```text
Total memory = 150Gi
Allocatable memory/node = 29Gi

150 / 29 = 5.17
```

Therefore:

```text
CPU requires ≈ 4 nodes
Memory requires ≈ 6 nodes
```

Memory becomes the limiting resource.

So initial node count must satisfy the larger requirement.

---

# 18. Example With 21 Microservices

Let's create an illustrative model.

| Service        | Peak Pods | CPU Request | Memory Request |
| -------------- | --------: | ----------: | -------------: |
| auth           |         4 |        300m |          512Mi |
| user           |         4 |        300m |          512Mi |
| customer       |         4 |        300m |          512Mi |
| catalog        |         8 |        500m |          768Mi |
| product        |         8 |        500m |          768Mi |
| inventory      |         6 |        500m |          768Mi |
| pricing        |         6 |        500m |          768Mi |
| promotion      |         4 |        300m |          512Mi |
| cart           |         6 |        500m |          768Mi |
| order          |         6 |        500m |          768Mi |
| payment        |         4 |        500m |          768Mi |
| fraud          |         4 |        500m |          768Mi |
| shipping       |         4 |        300m |          512Mi |
| delivery       |         4 |        300m |          512Mi |
| notification   |         4 |        300m |          512Mi |
| search         |         8 |        500m |            1Gi |
| recommendation |         4 |        500m |            1Gi |
| review         |         4 |        300m |          512Mi |
| tax            |         4 |        300m |          512Mi |
| invoice        |         4 |        500m |          768Mi |
| reporting      |         4 |        500m |            1Gi |

These are **planning examples**, not production sizing values.

---

# 19. Calculate Total CPU

We can sum:

```text
Total requested CPU
=
Σ(pods × CPU request)
```

For example:

```text
catalog:
8 × 500m = 4 CPU

product:
8 × 500m = 4 CPU

order:
6 × 500m = 3 CPU
```

Repeat for all 21 services.

This gives the application CPU requirement.

Then add:

```text
platform workloads
system DaemonSets
observability
ingress
DNS
autoscaling components
```

---

# 20. Calculate Total Memory

Same concept:

```text
Total memory
=
Σ(pods × memory request)
```

Example:

```text
catalog:
8 × 768Mi
= 6144Mi
≈ 6Gi
```

Then calculate all services.

After that:

```text
Application memory
+
platform memory
+
system overhead
+
headroom
```

---

# 21. Don't Forget DaemonSets

This is a common capacity-planning mistake.

We may run:

```text
Fluent Bit
node-exporter
AWS networking components
security agents
```

as DaemonSets.

That means:

```text
1 node
→
one instance of each DaemonSet
```

If we have:

```text
10 nodes
```

we may have:

```text
10 Fluent Bit pods
10 node-exporter pods
...
```

Therefore every additional node can add platform overhead.

---

# 22. Observability Capacity

Our observability stack itself consumes resources.

For example:

```text
Prometheus
Grafana
Alertmanager
kube-state-metrics
node-exporter
Fluent Bit
Logstash
OpenTelemetry Collector
Jaeger
```

These workloads must be included in capacity planning.

Don't calculate:

```text
21 services only
```

Calculate:

```text
Application
+
Platform
+
Observability
+
System
```

---

# 23. Three Types of Capacity

For this project, think about:

### 1. Steady-state capacity

Normal traffic.

```text
300 RPS
```

### 2. Peak capacity

Expected high traffic.

```text
600 RPS
```

### 3. Failure capacity

Capacity after infrastructure loss.

Example:

```text
600 RPS
+
one node unavailable
```

or:

```text
one AZ unavailable
```

---

# 24. Failure Headroom

Suppose:

```text
Normal requirement:
6 nodes
```

Three AZ architecture:

```text
AZ-A
AZ-B
AZ-C
```

If nodes are:

```text
2 / 2 / 2
```

and one AZ disappears:

```text
2 nodes lost
```

Remaining:

```text
4 nodes
```

Can the remaining four nodes handle the required workload?

If not:

```text
capacity plan is not resilient enough
```

This is why:

> **Availability requirements affect capacity planning.**

---

# 25. Multi-AZ Capacity

For production:

```text
AZ-A
AZ-B
AZ-C
```

Example:

```text
Node group / Karpenter
        │
 ┌──────┼──────┐
 ▼      ▼      ▼
AZ-A   AZ-B   AZ-C
```

Workloads should be distributed across AZs.

Use:

```text
topologySpreadConstraints
```

and/or:

```text
podAntiAffinity
```

where appropriate.

---

# 26. Topology Spread

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

If we have:

```text
6 replicas
```

the scheduler aims for approximately:

```text
AZ-A = 2
AZ-B = 2
AZ-C = 2
```

subject to the other scheduling constraints.

---

# 27. Node Failure Example

Suppose:

```text
9 nodes
```

distributed:

```text
AZ-A = 3
AZ-B = 3
AZ-C = 3
```

One node fails:

```text
8 nodes
```

Remaining capacity should still be sufficient for the workload.

One AZ fails:

```text
6 nodes
```

The platform should be tested against the required workload and SLOs.

This is much stronger than simply saying:

> "We have three AZs."

---

# 28. Pod Disruption Budgets

PDB protects availability during voluntary disruptions.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

This helps protect replicas during:

```text
node drain
maintenance
cluster operations
```

PDB does not protect against every type of failure.

---

# 29. HPA Capacity Planning

Suppose:

```yaml
minReplicas: 3
maxReplicas: 20
```

Capacity planning asks:

```text
Why 3?
Why 20?
```

We should derive these from:

```text
minimum HA requirement
traffic
pod capacity
SLO
dependency limits
failure scenarios
```

---

# 30. Example HPA

Conceptually:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
spec:
  minReplicas: 3
  maxReplicas: 20

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 300

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

The exact values should come from performance testing.

---

# 31. Why maxReplicas Matters

Suppose:

```text
maxReplicas = 20
```

but at peak:

```text
25 pods required
```

Then:

```text
HPA reaches 20
↓
Traffic continues increasing
↓
Pods cannot scale further
↓
Latency increases
```

Therefore:

> HPA `maxReplicas` is a capacity boundary.

---

# 32. But Don't Set maxReplicas to 1000

Suppose:

```text
maxReplicas = 1000
```

and each pod can create:

```text
20 DB connections
```

Potential database connections:

```text
1000 × 20
= 20,000
```

That could be catastrophic.

Every autoscaling boundary must respect downstream capacity.

---

# 33. KEDA Capacity Planning

For asynchronous services:

```text
SQS
Kafka
```

CPU may not represent workload accurately.

Example:

```text
notification-service
```

might scale based on:

```text
SQS queue depth
```

Conceptually:

```text
Queue depth ↑
     ↓
KEDA
     ↓
More consumer pods
```

Capacity planning must determine:

```text
messages/sec per consumer pod
```

---

# 34. KEDA Example

Conceptual:

```text
SQS:
1000 messages

Consumer:
100 messages/sec/pod
```

Then:

```text
1000 / 100
= 10 pods
```

But actual scaling also depends on:

* processing time
* visibility timeout
* downstream capacity
* concurrency
* CPU
* memory
* max replicas

---

# 35. Karpenter Capacity Planning

HPA creates pods.

Karpenter provides nodes when required.

```text
Traffic
 ↓
HPA
 ↓
Pods
 ↓
Pending
 ↓
Karpenter
 ↓
Node
 ↓
Pods scheduled
```

Capacity planning therefore includes:

```text
Pod requirements
+
Node selection
+
Node startup time
+
AZ availability
+
instance availability
```

---

# 36. NodePool Strategy

Our project can use different capacity classes.

For example:

```text
system workloads
general application
memory-heavy workloads
compute-heavy workloads
```

Conceptually:

```text
NodePool: system
NodePool: general
NodePool: compute
NodePool: memory
```

This avoids forcing every workload onto one instance type.

---

# 37. Instance Selection

Suppose a service requires:

```text
CPU-heavy
```

A compute-optimized instance may be appropriate.

Another service may be:

```text
memory-heavy
```

and benefit from a memory-oriented instance family.

Capacity planning should use actual benchmark data rather than selecting instance types purely by name.

---

# 38. Bin Packing

Suppose:

```text
Node allocatable:
7.5 CPU
29Gi
```

Pod requirements:

```text
Pod A:
1 CPU / 1Gi

Pod B:
1 CPU / 2Gi

Pod C:
500m / 1Gi
```

The scheduler packs workloads subject to:

```text
CPU
memory
affinity
topology
taints/tolerations
architecture
node selectors
```

Good packing can reduce cost.

But excessive packing can reduce resilience.

---

# 39. Utilization vs Headroom

Suppose nodes run at:

```text
95% CPU
```

Cost is efficient.

But:

```text
traffic spike
```

could cause immediate saturation.

At:

```text
30% CPU
```

we have plenty of headroom but may be wasting money.

Capacity planning seeks a reasonable operating point based on:

```text
workload
SLO
autoscaling
failure requirements
cost
```

There is no universal "correct" utilization percentage.

---

# 40. Deployment Capacity

This is often forgotten.

Suppose:

```text
Current:
10 pods
```

During a rolling deployment:

```text
maxSurge = 25%
```

we might temporarily have:

```text
10 + 3
= 13 pods
```

Capacity planning must support the temporary surge.

Otherwise:

```text
deployment
 ↓
new pods Pending
 ↓
rollout slows
 ↓
availability risk
```

---

# 41. Blue/Green Capacity

If using blue/green:

```text
Blue:
10 pods

Green:
10 pods
```

Temporary capacity:

```text
20 pods
```

approximately twice the application footprint.

Therefore blue/green deployments have different capacity requirements from rolling deployments.

---

# 42. Canary Capacity

Canary:

```text
Stable:
10 pods

Canary:
1–2 pods
```

Traffic:

```text
95% → stable
5%  → canary
```

Canary usually needs less additional capacity than full blue/green, but still requires room for the additional pods.

---

# 43. Cluster Autoscaler vs Karpenter

Our architecture uses Karpenter for node provisioning.

Conceptually:

```text
HPA
 ↓
Pod demand
 ↓
Karpenter
 ↓
EC2 capacity
```

Traditional Cluster Autoscaler works differently by scaling predefined node groups.

The capacity-planning principle remains:

```text
Pod resource requests
        ↓
Node capacity
```

---

# 44. IP Capacity

This is particularly important for EKS using the AWS VPC CNI.

Nodes and pods consume VPC IP addresses.

Therefore:

```text
More pods
 ↓
More IP addresses
```

If the subnet runs out of available addresses:

```text
Pod scheduling / networking
        ↓
problems
```

Capacity planning must include:

```text
subnet CIDRs
node count
pod density
AWS VPC CNI configuration
```

---

# 45. Example IP Planning

Suppose a subnet has:

```text
/24
```

There are roughly 256 addresses before AWS/Kubernetes/networking reservations and other allocations.

Don't assume all addresses are available for pods.

If your architecture eventually requires:

```text
1000+ pod IPs
```

a small subnet plan will become a problem.

This is why we selected:

```text
VPC:
10.0.0.0/16
```

and planned subnet sizes deliberately.

---

# 46. Capacity Planning Across AZs

Suppose:

```text
Total pods = 90
```

Three AZs:

```text
AZ-A = 30
AZ-B = 30
AZ-C = 30
```

Now lose AZ-A:

```text
60 pods remain
```

Can 60 pods handle the traffic?

If not, options include:

```text
larger node capacity
more baseline replicas
cross-AZ failover capacity
autoscaling headroom
traffic reduction
```

The answer depends on the required failure scenario.

---

# 47. Resource Calculation Model

Create a spreadsheet or document with:

```text
Service
Peak RPS
RPS/pod
Required pods
CPU/pod
Memory/pod
Total CPU
Total memory
Min replicas
Max replicas
Scaling metric
Dependency
```

Example:

| Service | Peak RPS | RPS/Pod | Pods | CPU/Pod | Memory/Pod |
| ------- | -------: | ------: | ---: | ------: | ---------: |
| product |      300 |      50 |    6 |    500m |      768Mi |
| catalog |      200 |      50 |    4 |    500m |      768Mi |
| order   |       60 |      20 |    3 |    500m |      768Mi |
| search  |      300 |      75 |    4 |    500m |        1Gi |

Again, the numbers are illustrative.

---

# 48. Capacity Spreadsheet Formula

For each service:

```text
required_pods =
ceil(peak_rps / tested_rps_per_pod)
```

Then:

```text
cpu =
required_pods × cpu_request
```

and:

```text
memory =
required_pods × memory_request
```

Then add:

```text
headroom
+
platform
+
observability
+
failure capacity
```

---

# 49. Example

Suppose:

```text
order-service
```

Performance testing:

```text
Sustainable:
20 RPS/pod
```

Peak:

```text
60 RPS
```

Therefore:

```text
60 / 20
= 3 pods
```

Minimum HA requirement:

```text
3
```

If one pod fails:

```text
2 remaining
```

Capacity becomes:

```text
2 × 20
= 40 RPS
```

So if we must survive one pod failure without violating the target:

```text
3 pods is insufficient
```

We may need:

```text
4 pods
```

because:

```text
4 pods = 80 RPS
```

and after one failure:

```text
3 × 20
= 60 RPS
```

This is a much better way to derive replica counts.

---

# 50. Node Calculation

Suppose total workload requests:

```text
CPU = 30
Memory = 120Gi
```

Node allocatable:

```text
CPU = 7.5
Memory = 29Gi
```

CPU:

```text
30 / 7.5
= 4 nodes
```

Memory:

```text
120 / 29
≈ 4.14
```

So:

```text
CPU → 4
Memory → 5
```

Initial capacity:

```text
5 nodes
```

Then apply:

```text
AZ requirements
failure headroom
system overhead
deployment surge
```

The final node count could be higher.

---

# 51. Failure-Aware Node Calculation

Suppose normal requirement:

```text
5 nodes
```

But we want to survive:

```text
1 node failure
```

without violating workload capacity.

We need enough spare capacity so that after losing one node:

```text
remaining capacity >= required workload
```

If each node contributes:

```text
7.5 CPU
```

then:

```text
5 nodes = 37.5 CPU
```

Lose one:

```text
4 nodes = 30 CPU
```

If workload requires:

```text
30 CPU
```

there is effectively no remaining headroom after the failure.

So five nodes may be technically sufficient but operationally tight.

---

# 52. AZ Failure Calculation

For stronger resilience:

```text
3 AZ
```

Suppose we need:

```text
30 CPU
```

after one AZ loss.

If we have:

```text
3 nodes/AZ
```

then:

```text
9 nodes total
```

Lose one AZ:

```text
6 nodes remain
```

Capacity:

```text
6 × 7.5
= 45 CPU
```

This provides:

```text
45 - 30
= 15 CPU
```

of remaining allocatable capacity before considering other constraints.

Again, exact sizing must be based on actual workload and failure requirements.

---

# 53. Cluster Capacity Model

Our production EKS planning should consider:

```text
                    EKS
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         AZ-A       AZ-B       AZ-C
          │          │          │
       Nodes       Nodes      Nodes
          │          │          │
        Pods        Pods       Pods
          │          │          │
       Services   Services   Services
```

Then:

```text
RDS
Redis
Kafka
SQS
```

must be checked as separate dependency capacity domains.

---

# 54. RDS Can Become the Cluster's Effective Limit

Suppose EKS can support:

```text
1000 RPS
```

but RDS safely supports only:

```text
500 RPS
```

Then the e-commerce platform cannot safely claim:

```text
1000 RPS capacity
```

for an RDS-dependent workflow.

Capacity must be expressed by workload:

```text
Catalog:
1000 RPS

Search:
2000 RPS

Order:
500 RPS
```

This is much more realistic than assigning one number to the entire platform.

---

# 55. Redis Capacity

Similarly:

```text
Redis capacity
```

may be much higher than:

```text
RDS
```

If cache hit ratio is high:

```text
RDS load decreases
```

But if Redis fails:

```text
RDS load increases
```

Therefore capacity planning should include:

```text
normal Redis
+
Redis degraded mode
```

---

# 56. Messaging Capacity

For asynchronous workloads:

```text
Producer capacity
Consumer capacity
Queue capacity
Broker capacity
Database capacity
```

must align.

Example:

```text
Order creation:
100 orders/sec
```

generates:

```text
OrderCreated:
100 events/sec
```

But downstream processing may generate:

```text
Payment:
100/sec
Inventory:
100/sec
Notification:
100/sec
Reporting:
100/sec
```

The architecture needs to handle these independently.

---

# 57. Capacity Headroom

Headroom protects against:

```text
traffic spikes
node failure
AZ failure
deployment surge
background jobs
unexpected workload
autoscaling delay
```

But too much headroom increases cost.

So define headroom based on:

```text
traffic profile
SLO
failure model
scaling speed
business criticality
```

---

# 58. Scaling Speed Matters

Suppose:

```text
Current:
3 pods
```

Traffic suddenly jumps:

```text
100 → 500 RPS
```

HPA might need time to:

```text
observe
calculate
create pods
schedule
start containers
```

Karpenter might additionally need time to:

```text
provision EC2
join node
schedule pods
```

Therefore:

> Capacity planning must include enough baseline capacity to bridge autoscaling delay.

---

# 59. Baseline vs Elastic Capacity

A good architecture often has:

```text
Baseline capacity
+
Elastic capacity
```

Example:

```text
Baseline:
6 nodes
```

During normal traffic:

```text
6 nodes
```

During peak:

```text
10 nodes
```

After traffic drops:

```text
6 nodes
```

This gives:

```text
cost efficiency
+
scalability
```

---

# 60. Overprovisioning

Sometimes we intentionally maintain spare capacity.

Example:

```text
6 application nodes
```

but normal workload requires:

```text
4 nodes
```

The spare:

```text
2 nodes
```

helps absorb:

```text
spikes
rolling deployments
node failure
```

This can be preferable for latency-sensitive workloads.

---

# 61. Capacity Planning for Batch Workloads

Not everything is online API traffic.

We also have:

```text
reporting-service
invoice-service
```

and potentially:

```text
CronJobs
data processing
migration Jobs
```

These can consume CPU/memory unexpectedly.

For example:

```text
02:00 AM
report generation
 ↓
CPU ↑
 ↓
application latency ↑
```

Therefore schedule batch jobs carefully and assign appropriate resource requests.

---

# 62. Resource Quotas

In larger environments, use:

```text
ResourceQuota
```

and:

```text
LimitRange
```

to prevent one namespace/team from consuming uncontrolled resources.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: application-quota
spec:
  requests.cpu: "20"
  requests.memory: 40Gi
```

The exact limits should be based on the environment.

---

# 63. Namespace Capacity

Our EKS cluster could contain:

```text
platform
monitoring
logging
tracing
applications
performance
```

Each namespace can have appropriate policies.

Example:

```text
platform
 ├── ArgoCD
 ├── ingress
 └── controllers

monitoring
 ├── Prometheus
 ├── Grafana
 └── Alertmanager

logging
 ├── Fluent Bit
 ├── Logstash
 └── Elasticsearch/OpenSearch

tracing
 ├── OTel Collector
 └── Jaeger

applications
 └── 21 services
```

Capacity planning should include all of them.

---

# 64. Cluster-Wide Resource Inventory

Run:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -A
```

Also:

```bash
kubectl get nodes
```

and:

```bash
kubectl describe node <node>
```

Use these to compare:

```text
Requested
Allocated
Actual
```

---

# 65. Resource Requests Inventory

A useful report should show:

```text
Namespace
Pod
CPU request
Memory request
CPU usage
Memory usage
Node
```

This allows us to find:

```text
over-requested workloads
under-requested workloads
memory-heavy services
CPU-heavy services
```

---

# 66. Over-Requesting

Suppose:

```text
Actual CPU:
100m
```

but:

```text
Request:
1000m
```

That pod reserves:

```text
1 CPU
```

even though it usually uses:

```text
0.1 CPU
```

This can lead to poor cluster utilization.

But don't immediately reduce requests.

Validate:

```text
peak usage
startup usage
GC behavior
traffic spikes
failure behavior
```

first.

---

# 67. Under-Requesting

Opposite problem:

```text
Actual:
800m

Request:
100m
```

This can cause:

```text
poor scheduling decisions
resource contention
unpredictable performance
```

Resource requests should represent a realistic workload requirement.

---

# 68. Capacity Planning Dashboard

Create a Grafana dashboard:

```text
EKS Capacity Planning
```

Panels:

```text
Total CPU requested
Total CPU usage
Total memory requested
Total memory usage
Allocatable CPU
Allocatable memory
Node count
Pod count
Pending pods
HPA desired replicas
HPA current replicas
Karpenter nodes
Node utilization
Pod distribution by AZ
```

---

# 69. Capacity Alerts

Useful alerts include:

```text
Pending pods
```

because:

```text
Pending pods
→ insufficient capacity or scheduling constraint
```

Also:

```text
Node CPU saturation
Node memory pressure
Node disk pressure
Insufficient IPs
HPA maxed out
KEDA maxed out
Karpenter provisioning failures
```

---

# 70. HPA Maxed-Out Alert

A useful condition:

```text
currentReplicas == maxReplicas
```

for a sustained period.

Why?

Because:

```text
HPA at max
```

may mean:

```text
traffic exceeds planned capacity
```

or:

```text
resource target too low/high
```

or:

```text
dependency bottleneck
```

This should trigger investigation.

---

# 71. Pending Pods Are a Capacity Signal

Example:

```bash
kubectl get pods -A | grep Pending
```

Then:

```bash
kubectl describe pod <pod>
```

Possible reasons:

```text
Insufficient CPU
Insufficient memory
No matching node
Taint
Affinity
Topology constraint
IP capacity
PVC issue
```

Never assume:

> Pending = not enough EC2 nodes.

The scheduler reason must be checked.

---

# 72. Capacity Planning With Karpenter

Suppose:

```text
HPA:
10 → 30 pods
```

Karpenter should evaluate:

```text
Pod requests
NodePool constraints
instance availability
AZ constraints
capacity type
architecture
```

and provision appropriate nodes.

Test:

```text
traffic spike
 ↓
HPA
 ↓
Pending pods
 ↓
Karpenter
 ↓
nodes
 ↓
pods ready
```

Measure the entire time.

---

# 73. Karpenter Provisioning Time

Record:

```text
T0:
Traffic spike

T1:
HPA increases desired replicas

T2:
Pods Pending

T3:
Karpenter creates node

T4:
Node Ready

T5:
Pod scheduled

T6:
Pod Ready
```

Calculate:

```text
T6 - T0
```

This tells us how long elastic scaling takes.

That number matters when deciding baseline capacity.

---

# 74. Capacity Planning With Deployment

Suppose:

```text
10 replicas
```

and:

```yaml
maxSurge: 25%
```

During rollout:

```text
up to ~13 pods
```

Depending on rollout configuration and readiness.

Therefore the cluster needs enough capacity for:

```text
normal workload
+
rollout surge
```

---

# 75. Pod Density

Nodes have practical pod limits.

EKS with AWS VPC CNI is affected by:

```text
ENIs
IP addresses
instance networking limits
CNI configuration
```

Therefore:

```text
Node CPU capacity
```

is not the only node capacity metric.

We must also consider:

```text
maximum pod density
```

and available IP addresses.

---

# 76. Example Capacity Constraint

Suppose:

```text
CPU allows:
40 pods/node
```

but networking supports only:

```text
25 pods/node
```

Then:

```text
25
```

becomes the practical pod-density limit.

This is another reason why simple:

```text
CPU / CPU-per-pod
```

calculations are insufficient.

---

# 77. Capacity Model

For our project, use this model:

```text
                Workload
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
         CPU     Memory    Pods
          │        │        │
          └────────┼────────┘
                   ▼
             Node Capacity
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
      CPU         RAM         IPs
       │           │           │
       └───────────┼───────────┘
                   ▼
              AZ Capacity
                   │
                   ▼
             Failure Model
                   │
                   ▼
            Final Capacity
```

---

# 78. Capacity Planning Workbook

Create:

```text
performance/
└── capacity/
    ├── service-capacity.md
    ├── node-capacity.md
    ├── az-capacity.md
    ├── dependency-capacity.md
    ├── ip-capacity.md
    └── capacity-summary.md
```

---

# 79. Service Capacity Template

```markdown
# Service Capacity

## Service

order-service

## Peak RPS

TBD

## Tested RPS per Pod

TBD

## Required Replicas

TBD

## CPU Request

TBD

## Memory Request

TBD

## Minimum Replicas

TBD

## Maximum Replicas

TBD

## HPA Metric

TBD

## Dependency Limits

- RDS
- Redis
- Kafka

## Failure Requirement

TBD

## Notes

TBD
```

---

# 80. Node Capacity Template

```markdown
# Node Capacity

## Instance Type

TBD

## vCPU

TBD

## Memory

TBD

## Allocatable CPU

TBD

## Allocatable Memory

TBD

## Maximum Pod Density

TBD

## AZ Distribution

TBD

## Normal Node Count

TBD

## Peak Node Count

TBD

## Failure Node Count

TBD
```

---

# 81. Dependency Capacity

Create:

```markdown
# Dependency Capacity

## RDS

Maximum validated workload:
TBD

## Redis

Maximum validated workload:
TBD

## Kafka

Maximum validated throughput:
TBD

## SQS

Consumer processing capacity:
TBD

## External APIs

Rate limits:
TBD
```

This prevents us from saying:

```text
EKS supports 1000 RPS
```

when:

```text
RDS supports only 500 RPS
```

for that workflow.

---

# 82. Production Capacity Planning Workflow

The complete process:

```text
1. Collect production traffic
        ↓
2. Build workload model
        ↓
3. Load test
        ↓
4. Determine RPS/pod
        ↓
5. Determine CPU/memory/pod
        ↓
6. Determine replicas
        ↓
7. Determine node capacity
        ↓
8. Add platform overhead
        ↓
9. Add failure requirements
        ↓
10. Validate AZ distribution
        ↓
11. Validate IP capacity
        ↓
12. Validate RDS/Redis/messaging
        ↓
13. Test autoscaling
        ↓
14. Test node failure
        ↓
15. Document capacity
```

---

# 83. Capacity Planning Example

Suppose testing shows:

```text
Peak customer traffic:
600 RPS
```

For `catalog-service`:

```text
100 RPS/pod
```

Therefore:

```text
600 / 100
= 6 pods
```

Suppose:

```text
CPU request = 500m
Memory = 768Mi
```

Total:

```text
CPU:
6 × 500m
= 3 CPU

Memory:
6 × 768Mi
= 4608Mi
≈ 4.5Gi
```

Then consider:

```text
3 AZ
pod distribution
one-pod failure
deployment surge
HPA scaling
node failure
```

Only then choose final capacity.

---

# 84. Capacity Planning Is Iterative

Initial model:

```text
6 pods
```

Load test:

```text
P95 violates SLO
```

Investigation:

```text
Database latency
```

Optimize database.

Retest:

```text
P95 passes
```

Now:

```text
6 pods
```

may be sufficient.

Alternatively:

```text
7 pods
```

may be required.

The numbers evolve as the system evolves.

---

# 85. Capacity Planning Mistakes

## Mistake 1

Sizing from:

```text
CPU only
```

Wrong.

Need:

```text
CPU
memory
pods
network
IPs
dependencies
```

---

## Mistake 2

Using average traffic.

Capacity should account for:

```text
peak
spike
failure
```

---

## Mistake 3

Ignoring system workloads.

Include:

```text
DaemonSets
monitoring
logging
networking
controllers
```

---

## Mistake 4

Ignoring deployment surge.

Include:

```text
rolling
canary
blue/green
```

---

## Mistake 5

Ignoring database capacity.

More pods can mean:

```text
more DB connections
```

---

## Mistake 6

Ignoring AZ failure.

Three AZs alone do not guarantee that remaining capacity is sufficient.

---

## Mistake 7

Ignoring IP capacity.

EKS networking can become the limiting resource.

---

# 86. Capacity Planning and Cost

Once we know:

```text
required nodes
```

we can calculate infrastructure cost.

Example:

```text
6 nodes
```

versus:

```text
10 nodes
```

But cost should not be optimized by reducing capacity below the requirements needed for:

```text
SLO
availability
failure recovery
scaling
```

Capacity planning therefore becomes:

```text
Performance
+
Reliability
+
Cost
```

---

# 87. Final Production Architecture

```text
                         Traffic
                            │
                            ▼
                          ALB
                            │
                            ▼
                   ┌────────────────┐
                   │      EKS       │
                   └───────┬────────┘
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
              AZ-A       AZ-B       AZ-C
                │          │          │
             Nodes       Nodes      Nodes
                │          │          │
             Pods        Pods       Pods
                │          │          │
                └──────────┼──────────┘
                           │
                     HPA / KEDA
                           │
                           ▼
                       Karpenter
                           │
                           ▼
                     EC2 Capacity

Dependencies:
    ├── RDS
    ├── Redis
    ├── SQS
    ├── SNS
    ├── Kafka
    └── RabbitMQ

Observability:
    ├── Prometheus
    ├── Grafana
    ├── ELK
    ├── OpenTelemetry
    └── Jaeger
```

---

# 88. Final Capacity Calculation Model

For every service:

```text
Peak traffic
     ↓
Tested requests/pod
     ↓
Required replicas
     ↓
CPU/pod
Memory/pod
     ↓
Total workload
     ↓
Node allocatable capacity
     ↓
Platform overhead
     ↓
Deployment surge
     ↓
Failure headroom
     ↓
AZ distribution
     ↓
IP/pod-density limits
     ↓
Final EKS capacity
```

And then:

```text
Final EKS capacity
        ↓
Check RDS capacity
        ↓
Check Redis capacity
        ↓
Check messaging capacity
        ↓
Check external dependency limits
        ↓
Validate end-to-end SLO
```

---

# 89. What You Should Be Able to Explain After Chapter 129

If an interviewer or engineer asks:

### "How did you decide the number of EKS nodes?"

You should be able to explain:

> We first measured workload traffic and sustainable requests per pod through performance testing. We calculated required replicas from peak workload, then multiplied replicas by CPU and memory requests. We compared the aggregate requirements against node allocatable capacity, while also accounting for platform DaemonSets, observability workloads, pod density/IP limits, deployment surge, AZ distribution, and node/AZ failure requirements. Finally, we validated the model through load and failure testing.

### "Why not just use CPU?"

Because:

```text
CPU
Memory
Pod density
IP capacity
Network
Database
Redis
Messaging
AZ failure
```

can all become limiting resources.

### "How does HPA affect capacity planning?"

```text
HPA
 ↓
Replica count
 ↓
CPU/memory requirement
 ↓
Pending pods
 ↓
Karpenter
 ↓
Node capacity
```

### "How do you know the capacity is sufficient?"

Not from Terraform alone.

We validate it with:

```text
Load tests
Performance tests
Failure tests
Observability
SLO validation
```

---

# 90. Connection to the Next Chapters

We now have:

```text
127 — Load Testing
        ↓
Generate realistic traffic

128 — Performance Testing
        ↓
Find bottlenecks

129 — Kubernetes Capacity Planning
        ↓
Convert workload into infrastructure capacity
```

The next chapter is:

# **130 — 21-MICROSERVICE SCALING**

There we will take the actual **21 microservices individually** and build the scaling model:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

For each service, we will determine **traffic pattern → synchronous/asynchronous behavior → CPU/memory profile → HPA/KEDA strategy → min/max replicas → RDS/Redis/messaging dependency → scaling bottleneck → failure behavior → observability**, and then combine all 21 into one production EKS scaling architecture.
