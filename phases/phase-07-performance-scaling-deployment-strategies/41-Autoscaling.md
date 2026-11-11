# `41-Autoscaling.md`

````markdown
# 41. Autoscaling — HPA, VPA, Karpenter and Cluster Capacity in AWS EKS

## 1. Why Autoscaling Matters

Our e-commerce platform contains 21 microservices:

- auth-service
- user-service
- customer-service
- catalog-service
- product-service
- inventory-service
- pricing-service
- promotion-service
- cart-service
- order-service
- payment-service
- fraud-service
- shipping-service
- delivery-service
- notification-service
- search-service
- recommendation-service
- review-service
- tax-service
- invoice-service
- reporting-service

Traffic is not constant.

For example:

```text
02:00 AM
100 requests/sec

10:00 AM
1,500 requests/sec

01:00 PM
4,000 requests/sec

08:00 PM
12,000 requests/sec
````

We don't want to permanently run enough infrastructure for:

```text
12,000 requests/sec
```

if most of the day only needs:

```text
1,000 requests/sec
```

Autoscaling allows Kubernetes and AWS to dynamically adjust capacity.

---

# 2. The Most Important Concept

There are multiple layers of scaling.

```text
Application workload
        |
        v
Pod scaling
        |
        v
Node scaling
        |
        v
AWS infrastructure scaling
```

Kubernetes and AWS solve different problems.

The simplified architecture is:

```text
                    Traffic
                       |
                       v
                  Kubernetes
                       |
                 HPA scales Pods
                       |
                       v
              More Pods required
                       |
                       v
              Insufficient Nodes
                       |
                       v
                 Karpenter
                       |
                       v
               New EC2 Nodes
```

Remember:

```text
HPA = "How many Pods do I need?"

Karpenter = "How many nodes do I need?"
```

---

# 3. Four Important Autoscaling Technologies

For EKS, understand these separately:

```text
HPA
VPA
Cluster Autoscaler
Karpenter
```

They have different responsibilities.

| Technology         | Main Responsibility                         |
| ------------------ | ------------------------------------------- |
| HPA                | Increase/decrease Pod replicas              |
| VPA                | Recommend/change Pod CPU/memory requests    |
| Cluster Autoscaler | Adjust node group size                      |
| Karpenter          | Provision right-sized EC2 nodes dynamically |

Our production architecture will primarily use:

```text
HPA + Karpenter
```

with VPA used carefully for selected workloads.

---

# 4. HPA

HPA means:

```text
Horizontal Pod Autoscaler
```

It changes the number of Pod replicas.

Example:

```text
Current:
3 Pods

Traffic increases

        ↓

HPA

        ↓

6 Pods
```

Later:

```text
Traffic decreases

        ↓

HPA

        ↓

3 Pods
```

---

# 5. HPA Does Not Create Nodes

This is a common misunderstanding.

Suppose:

```text
EKS nodes:
3
```

and each node can run:

```text
10 application Pods
```

Current:

```text
20 Pods
```

Traffic increases.

HPA calculates:

```text
40 Pods required
```

But only 30 Pods can fit.

Then:

```text
HPA
 |
 +-- desired replicas = 40
 |
 v
Kubernetes scheduler
 |
 +-- 10 Pods remain Pending
 |
 v
Karpenter
 |
 v
New node(s)
 |
 v
Pending Pods scheduled
```

Therefore:

```text
HPA -> Pod capacity

Karpenter -> Node capacity
```

---

# 6. HPA Architecture

```text
                Prometheus / Metrics Server
                         |
                         v
                       HPA
                         |
                 +-------+-------+
                 |               |
                 v               v
              Deployment      Deployment
                 |               |
                 v               v
               Pods            Pods
```

HPA can use:

* CPU utilization
* memory utilization
* custom metrics
* external metrics

The correct metric depends on the workload.

---

# 7. CPU-Based HPA

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

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

This means approximately:

```text
minimum = 3 Pods
maximum = 20 Pods
target CPU = 70%
```

---

# 8. Why Requests Matter for HPA

Suppose a Pod has:

```yaml
resources:
  requests:
    cpu: "500m"
```

and the Pod consumes:

```text
350m
```

CPU utilization for HPA purposes is approximately:

```text
350 / 500 × 100
=
70%
```

Therefore:

```text
CPU request affects HPA utilization calculations.
```

This is why resource requests must be realistic.

---

# 9. Bad Resource Requests

Suppose:

```yaml
requests:
  cpu: "2"
```

but the application normally needs:

```text
200m
```

Then the application may appear underutilized.

Alternatively, if we configure:

```yaml
requests:
  cpu: "50m"
```

while the application normally uses:

```text
500m
```

the HPA can react too aggressively.

Therefore:

```text
Resource requests
+
HPA configuration
```

must be designed together.

---

# 10. Memory-Based HPA

Example:

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
```

Memory-based scaling can be useful, but it needs careful analysis.

For Java applications especially:

```text
JVM heap
+
off-heap memory
+
native memory
+
container overhead
```

all matter.

Do not blindly use memory as the primary scaling signal.

---

# 11. Why CPU Is Not Always the Best Metric

Imagine:

```text
search-service
```

receives:

```text
1,000 requests/sec
```

but CPU remains relatively low because the service waits on:

```text
Elasticsearch
```

CPU-based HPA might see:

```text
CPU = 30%
```

and decide:

```text
No scaling required
```

while latency is actually:

```text
5 seconds
```

A better metric could be:

```text
requests/sec per Pod
```

or:

```text
request latency
```

or:

```text
in-flight requests
```

depending on the service.

---

# 12. Custom Metrics

Prometheus can provide application-level metrics.

Example:

```text
http_requests_total
```

or a rate:

```promql
sum(rate(http_server_request_duration_count{service="order-service"}[2m]))
```

We can derive:

```text
requests/sec
```

and use a custom/external metrics adapter where appropriate.

The architecture becomes:

```text
Application
   |
   v
Micrometer
   |
   v
Prometheus
   |
   v
Metrics adapter
   |
   v
HPA
   |
   v
Deployment
```

---

# 13. Request-Based Scaling

Suppose:

```text
Target:
100 requests/sec/Pod
```

Current traffic:

```text
600 requests/sec
```

Then approximately:

```text
600 / 100 = 6 Pods
```

HPA can scale toward that target when configured with an appropriate custom metric.

This can be more representative than CPU for some APIs.

---

# 14. HPA Example for Order Service

Suppose production starts with:

```text
minReplicas = 3
maxReplicas = 15
```

Target:

```text
CPU = 65%
```

Traffic increases.

```text
3 Pods
  |
  v
CPU > 65%
  |
  v
HPA
  |
  v
5 Pods
```

If traffic continues:

```text
5 Pods
  |
  v
CPU > target
  |
  v
8 Pods
```

Eventually:

```text
8 -> 12 -> 15
```

until the maximum is reached.

---

# 15. Scale-Up and Scale-Down Are Different

Production systems should not necessarily scale up and down at the same speed.

Example:

```text
Scale up:
fast

Scale down:
slow
```

Why?

Suppose:

```text
Traffic:
1000 -> 5000 -> 1000
```

If Pods immediately scale down:

```text
3 -> 15 -> 3
```

we can create oscillation.

Instead:

```text
3 -> 15
```

quickly, then:

```text
15 -> 12 -> 8 -> 5 -> 3
```

more slowly.

This is called stabilization.

---

# 16. HPA Behavior

Example:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0

  scaleDown:
    stabilizationWindowSeconds: 300
```

This is only an example.

Real production values should be based on:

* traffic pattern
* startup time
* request latency
* downstream dependencies
* Pod startup time
* cost
* business SLA

---

# 17. Pod Startup Time Matters

Suppose:

```text
Application startup = 45 seconds
```

Traffic suddenly increases.

HPA creates:

```text
10 new Pods
```

But they aren't immediately ready.

Therefore:

```text
HPA decision
      |
      v
Pod creation
      |
      v
Image pull
      |
      v
Container startup
      |
      v
Readiness probe
      |
      v
Traffic
```

Autoscaling must account for startup latency.

---

# 18. Readiness Probe and HPA

A Pod should not receive traffic before it is ready.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Without proper readiness:

```text
New Pod
   |
   v
Not actually ready
   |
   v
Receives traffic
   |
   v
Errors
```

Correct:

```text
New Pod
   |
   v
Starting
   |
   v
Readiness = false
   |
   v
Application ready
   |
   v
Readiness = true
   |
   v
Receives traffic
```

---

# 19. HPA and PDB

These solve different problems.

HPA:

```text
Scale based on workload
```

PDB:

```text
Protect availability during voluntary disruption
```

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

HPA:

```text
3 -> 10 Pods
```

PDB:

```text
Don't voluntarily disrupt too many simultaneously.
```

---

# 20. HPA and Deployment

The normal relationship is:

```text
HPA
 |
 v
Deployment
 |
 v
ReplicaSet
 |
 v
Pods
```

Do not manually set:

```text
kubectl scale deployment order-service --replicas=10
```

while HPA owns scaling.

The HPA will continuously reconcile the replica count.

---

# 21. VPA

VPA means:

```text
Vertical Pod Autoscaler
```

Instead of changing:

```text
number of Pods
```

VPA changes or recommends:

```text
CPU/memory requests
```

Example:

```text
Pod currently:

CPU request = 250m
Memory request = 512Mi
```

VPA observes actual usage.

It may recommend:

```text
CPU = 500m
Memory = 1Gi
```

---

# 22. HPA vs VPA

```text
HPA
Horizontal
3 Pods -> 10 Pods

VPA
Vertical
500m CPU -> 1 CPU
```

Think:

```text
HPA = more workers

VPA = bigger worker
```

---

# 23. Do Not Blindly Combine HPA and VPA

Suppose HPA scales based on CPU utilization while VPA continuously changes CPU requests.

Then:

```text
VPA changes request
       |
       v
HPA utilization calculation changes
       |
       v
HPA decision changes
       |
       v
Scaling behavior becomes difficult to reason about
```

Therefore, avoid blindly combining them on the same resource metric.

A common approach is:

```text
HPA -> production horizontally scalable APIs

VPA -> recommendation/right-sizing
```

Use VPA carefully for workloads where vertical adjustment is appropriate.

---

# 24. Karpenter

Karpenter solves node provisioning.

Suppose HPA creates more Pods:

```text
HPA
 |
 v
20 new Pods
 |
 v
Scheduler
 |
 v
No available node capacity
 |
 v
Pods Pending
 |
 v
Karpenter
 |
 v
Provision EC2 nodes
```

Karpenter watches for unschedulable Pods and can provision suitable nodes according to its configuration.

---

# 25. Why Karpenter Is Useful

Traditional node groups might say:

```text
Node Group:
m6i.large

Desired:
5

Min:
3

Max:
20
```

Karpenter can select capacity based on constraints such as:

```text
CPU
memory
architecture
instance category
generation
availability zone
capacity type
```

This can improve:

* provisioning flexibility
* bin packing
* workload-specific capacity
* cost optimization

---

# 26. Karpenter Does Not Replace HPA

Incorrect:

```text
Karpenter -> application Pod scaling
```

Correct:

```text
HPA
 |
 +-- Pod replicas

Karpenter
 |
 +-- EC2 node capacity
```

Together:

```text
Traffic
  |
  v
HPA
  |
  v
More Pods
  |
  v
Scheduler
  |
  v
Karpenter
  |
  v
More EC2 nodes
```

---

# 27. Karpenter Architecture

Conceptually:

```text
                 Kubernetes API
                       |
                       v
                  Karpenter
                       |
                       v
              Pending/unschedulable Pods
                       |
                       v
              Node requirements
                       |
                       v
                    AWS EC2
```

Karpenter runs inside the EKS cluster but interacts with AWS APIs to provision and manage compute capacity.

---

# 28. NodePools

Modern Karpenter configurations use concepts such as:

```text
NodePool
NodeClass
```

The exact APIs and fields depend on the Karpenter version selected.

A conceptual NodePool might define:

```text
architecture:
amd64

capacity:
on-demand / spot

instance families:
m6i
m7i
c6i
c7i
r6i

AZ:
multiple

limits:
cluster capacity constraints
```

Always pin and validate the Karpenter version against the EKS version and its current API.

---

# 29. Workload-Specific Node Pools

Not every workload should use identical nodes.

For example:

```text
general applications
    |
    v
general-purpose nodes

memory-heavy workloads
    |
    v
memory-optimized nodes

CPU-heavy workloads
    |
    v
compute-optimized nodes
```

Example:

```text
recommendation-service
        |
        v
memory/CPU optimized capacity
```

while:

```text
notification-service
        |
        v
general-purpose capacity
```

---

# 30. System Nodes vs Application Nodes

A production cluster often separates system workloads from applications.

Example:

```text
System Node Pool
----------------
CoreDNS
VPC CNI
kube-proxy
monitoring agents
other system workloads
```

Application capacity:

```text
Application Node Pool
---------------------
21 microservices
```

This prevents application spikes from starving critical cluster components.

---

# 31. Taints and Tolerations

We can protect system capacity.

Example node taint:

```text
workload=system:NoSchedule
```

Only Pods with the matching toleration can run there.

Conceptually:

```text
System Node
   |
   +-- CoreDNS
   +-- critical platform Pods
```

Application Pods are prevented from consuming all system capacity.

---

# 32. Critical Workloads

Some workloads should have higher scheduling priority.

Examples:

```text
CoreDNS
CNI components
Ingress
monitoring
logging
ArgoCD
```

If the cluster becomes resource constrained, these components must remain operational.

Use:

```text
PriorityClass
resource requests
taints/tolerations
dedicated capacity
```

appropriately.

---

# 33. Node Resource Calculation

Suppose a node has:

```text
8 vCPU
32 GiB memory
```

Do not assume the full amount is available for applications.

Some resources are consumed by:

```text
kubelet
container runtime
DaemonSets
CNI
kube-proxy
monitoring agents
logging agents
system processes
```

Therefore:

```text
Node capacity
      -
System overhead
      =
Allocatable capacity
```

Application scheduling uses allocatable capacity.

---

# 34. DaemonSet Overhead

This is particularly important in our platform.

Every node may run:

```text
Fluent Bit
Node Exporter
AWS components
security agents
```

If Karpenter ignores their resource requirements, node utilization estimates can be wrong.

Therefore resource requests for platform DaemonSets must be realistic.

---

# 35. Bin Packing

Suppose we have:

```text
Node 1:
CPU 40%
Memory 30%

Node 2:
CPU 30%
Memory 40%
```

There may be unused capacity.

Good node provisioning attempts to pack Pods efficiently.

```text
Pod A
Pod B
Pod C
Pod D
     |
     v
Node
```

Better packing can reduce:

```text
EC2 count
```

and therefore:

```text
AWS cost
```

But excessive packing can reduce failure headroom.

---

# 36. Headroom

Production clusters should not operate at:

```text
100% capacity
```

Suppose:

```text
Allocatable:
100 CPU

Normal usage:
70 CPU
```

This gives:

```text
30 CPU headroom
```

Headroom helps absorb:

* traffic spikes
* node failures
* rolling deployments
* DaemonSet changes
* rescheduling
* AZ disruptions

---

# 37. Three Traffic Scenarios

Now apply autoscaling to our project.

## Low Traffic

Example:

```text
100 requests/sec
```

Architecture:

```text
ALB
 |
 v
3 order-service Pods
 |
 v
3-5 EKS nodes
```

Only the minimum application replicas run.

---

# 38. Medium Traffic

Traffic:

```text
2,000 requests/sec
```

HPA:

```text
3 -> 8 order-service Pods
```

Other services may scale independently.

Example:

```text
order-service:
8 Pods

catalog-service:
6 Pods

product-service:
5 Pods

payment-service:
7 Pods

notification-service:
4 Pods
```

We don't scale all 21 services equally.

This is an important microservices principle.

---

# 39. High Traffic

Suppose:

```text
12,000 requests/sec
```

HPA:

```text
order-service:
3 -> 20

payment-service:
3 -> 15

inventory-service:
3 -> 18
```

Kubernetes scheduler detects insufficient capacity.

```text
Pending Pods
      |
      v
Karpenter
      |
      v
Additional EC2 nodes
```

The cluster expands.

---

# 40. Why All 21 Services Should Not Scale Together

Suppose only:

```text
catalog-service
```

receives a traffic spike.

Bad design:

```text
catalog traffic increases
       |
       v
scale all 21 services
```

This wastes money.

Correct:

```text
catalog-service
       |
       v
HPA
       |
       v
more catalog Pods
```

Other services remain near their normal replica counts.

---

# 41. Different Services Need Different Scaling Metrics

Example:

| Service                | Possible Primary Scaling Signal |
| ---------------------- | ------------------------------- |
| auth-service           | requests/sec / CPU              |
| catalog-service        | requests/sec                    |
| search-service         | requests/sec / latency          |
| order-service          | requests/sec / CPU              |
| payment-service        | requests/sec / CPU              |
| notification-service   | queue depth                     |
| reporting-service      | queue depth / workload          |
| recommendation-service | CPU/memory/request latency      |

There is no universal HPA metric for all microservices.

---

# 42. Queue-Based Autoscaling

Consider:

```text
notification-service
```

Messages arrive in SQS.

```text
SQS
 |
 | 10,000 messages
 v
notification-service
```

CPU may still be:

```text
40%
```

But backlog is increasing.

CPU HPA may not scale.

A better signal is:

```text
ApproximateNumberOfMessagesVisible
```

Then:

```text
Queue depth
     |
     v
KEDA/HPA
     |
     v
More consumer Pods
```

---

# 43. KEDA

KEDA means:

```text
Kubernetes Event-driven Autoscaling
```

It is useful when scaling should react to external/event metrics.

Examples:

```text
SQS queue depth
Kafka consumer lag
RabbitMQ queue length
```

Architecture:

```text
SQS
 |
 v
KEDA
 |
 v
HPA
 |
 v
Consumer Deployment
```

KEDA can create/manage the scaling behavior needed for event-driven workloads.

---

# 44. KEDA in Our Project

Excellent candidates include:

```text
notification-service
invoice-service
reporting-service
shipping-service
delivery-service
```

depending on how these services consume messages.

Example:

```text
SQS queue:
order-events

Messages:
50
```

Keep:

```text
2 consumer Pods
```

Messages increase:

```text
5,000
```

Scale:

```text
2 -> 10 Pods
```

Messages decrease:

```text
5,000 -> 100
```

Scale down gradually.

---

# 45. Kafka Autoscaling

For Kafka consumers, the important signal is often:

```text
consumer lag
```

Example:

```text
Kafka topic
 |
 +-- partition 0
 +-- partition 1
 +-- partition 2
 +-- partition 3
```

Consumer group:

```text
consumer-1
consumer-2
consumer-3
consumer-4
```

If lag increases:

```text
Kafka lag
   |
   v
KEDA/custom scaling
   |
   v
More consumers
```

But scaling is constrained by partition count.

If there are only:

```text
4 partitions
```

adding:

```text
20 consumers
```

does not provide 20-way parallelism for that topic/consumer group.

---

# 46. RabbitMQ Autoscaling

For RabbitMQ:

```text
queue depth
```

can be a scaling signal.

Example:

```text
RabbitMQ queue:
100 messages
```

Consumers:

```text
3
```

Queue grows:

```text
10,000
```

Scale:

```text
3 -> 10
```

Again, application processing capacity and downstream dependencies must be considered.

---

# 47. Scaling the Database

This is where many beginners make a mistake.

Suppose:

```text
HPA:
3 -> 30 order-service Pods
```

That does not automatically mean the database can handle 10x more connections.

If:

```text
Each Pod:
20 DB connections
```

then:

```text
30 × 20
=
600 connections
```

The database may become the bottleneck.

---

# 48. HPA and HikariCP

Our Spring Boot service may use HikariCP.

Suppose:

```text
maximumPoolSize = 20
```

and:

```text
Pods = 10
```

Potential maximum connections:

```text
10 × 20
=
200
```

If HPA scales:

```text
10 -> 30
```

then:

```text
30 × 20
=
600
```

Therefore:

```text
HPA design
+
connection pool design
+
RDS capacity
```

must be considered together.

---

# 49. Redis Scaling Impact

The same concept applies to Redis.

More Pods can mean:

```text
more Redis clients
more connections
more commands/sec
```

Therefore:

```text
HPA
 |
 v
more application Pods
 |
 v
Redis load increases
```

Do not treat Redis as infinitely scalable.

---

# 50. HPA and Downstream Services

Suppose:

```text
order-service
```

scales:

```text
3 -> 30 Pods
```

and every request calls:

```text
inventory-service
```

Then inventory traffic can increase dramatically.

```text
order-service
30 Pods
   |
   +----> inventory-service
                 |
                 +----> RDS
```

Scaling one service can create a cascading load increase.

This is why autoscaling must be designed across service dependencies.

---

# 51. Cascading Scaling

Example:

```text
Traffic
  |
  v
order-service
  |
  v
payment-service
  |
  v
RDS
```

Order scales:

```text
3 -> 20
```

Payment receives more requests:

```text
3 -> 15
```

Database receives:

```text
many more connections
```

Now:

```text
RDS saturated
```

Latency increases.

HPA sees:

```text
CPU/latency high
```

and may scale more Pods.

This can make the situation worse.

Therefore autoscaling needs:

```text
rate limiting
connection limits
backpressure
timeouts
circuit breakers
queueing
appropriate maxReplicas
```

---

# 52. Maximum Replicas Are a Safety Mechanism

Don't configure:

```yaml
maxReplicas: 500
```

without analysis.

Maximum replicas protect downstream systems.

Example:

```text
order-service:
maxReplicas = 30

payment-service:
maxReplicas = 20

inventory-service:
maxReplicas = 25
```

These values must be derived from capacity testing.

---

# 53. Autoscaling and Rate Limiting

Suppose an external client sends:

```text
100,000 requests/sec
```

If HPA tries to create enough Pods to process everything:

```text
3 -> 500 Pods
```

that could be disastrous.

Rate limiting protects the platform.

```text
Client
 |
 v
WAF / ALB / API Gateway
 |
 v
Rate limit
 |
 v
Application
```

Autoscaling handles legitimate workload growth.

Rate limiting protects against excessive workload.

---

# 54. Autoscaling and Circuit Breakers

Suppose:

```text
payment-service
```

is failing.

If:

```text
order-service
```

continues retrying aggressively, traffic can increase.

Then HPA may scale order-service.

Result:

```text
More Pods
   |
   v
More retries
   |
   v
Payment overloaded
   |
   v
More failures
```

This is a retry storm.

Use:

```text
timeouts
bounded retries
exponential backoff
circuit breakers
```

alongside autoscaling.

---

# 55. Graceful Shutdown

When Pods scale down:

```text
Pod
 |
 v
Termination signal
 |
 v
Stop accepting new work
 |
 v
Finish in-flight requests
 |
 v
Close connections
 |
 v
Exit
```

Spring Boot should be configured for graceful shutdown.

For HTTP services, Kubernetes readiness and termination handling must work together.

---

# 56. Pod Lifecycle During Scale Down

Example:

```text
HPA:
10 -> 5
```

Kubernetes chooses Pods to terminate.

Correct sequence:

```text
Pod becomes unavailable for new traffic
        |
        v
Endpoint removed
        |
        v
Graceful termination
        |
        v
Existing request completes
        |
        v
Container exits
```

Poor shutdown handling can cause:

```text
5xx
connection resets
lost work
```

---

# 57. Pod Topology

Scaling from:

```text
3 -> 15 Pods
```

doesn't mean all Pods should run on one node.

Use:

```text
topologySpreadConstraints
```

and/or:

```text
podAntiAffinity
```

to distribute replicas.

Conceptually:

```text
AZ-A       AZ-B       AZ-C

Pod-1      Pod-2      Pod-3
Pod-4      Pod-5      Pod-6
```

This improves resilience.

---

# 58. Availability Zone Failure

Suppose:

```text
AZ-A fails
```

Before failure:

```text
AZ-A: 5 Pods
AZ-B: 5 Pods
AZ-C: 5 Pods
```

After:

```text
AZ-B: 5
AZ-C: 5
```

We may need:

```text
HPA
+
Karpenter
```

to restore capacity.

This is why capacity headroom and multi-AZ node provisioning matter.

---

# 59. Karpenter and Availability Zones

Karpenter should be configured so that suitable capacity can be obtained across multiple AZs.

Avoid designing:

```text
all application nodes -> one AZ
```

Instead:

```text
AZ-A
AZ-B
AZ-C
```

with appropriate subnet and capacity configuration.

---

# 60. Spot Instances

Karpenter can use Spot capacity where workloads tolerate interruption.

Good candidates can include:

```text
batch processing
reporting
some asynchronous workers
non-critical workloads
```

Be careful with:

```text
payment-service
critical stateful systems
single-instance critical components
```

unless the architecture explicitly handles interruptions.

Spot is a capacity/cost decision, not automatically a production default.

---

# 61. On-Demand + Spot

A practical architecture can be:

```text
Critical workloads
      |
      v
On-Demand

Fault-tolerant workers
      |
      v
Spot
```

Example:

```text
payment-service
    -> On-Demand

notification-service
    -> Spot-tolerant

reporting-service
    -> Spot-tolerant
```

Actual placement should be based on workload behavior.

---

# 62. Cluster Autoscaler

Cluster Autoscaler is another node autoscaling mechanism.

Conceptually:

```text
Pending Pods
     |
     v
Cluster Autoscaler
     |
     v
Increase node group desired size
```

It works primarily with configured node groups.

Traditional model:

```text
Node Group A:
min 3
desired 5
max 20
```

Cluster Autoscaler changes:

```text
5 -> 10
```

---

# 63. Karpenter vs Cluster Autoscaler

|                    | Cluster Autoscaler        | Karpenter              |
| ------------------ | ------------------------- | ---------------------- |
| Scaling unit       | Node group                | Node                   |
| Main model         | Existing node groups      | Dynamic provisioning   |
| Instance selection | Mostly node-group-defined | Constraint-driven      |
| Flexibility        | Lower                     | Higher                 |
| Bin packing        | Good                      | Strong                 |
| AWS EKS use        | Mature/common             | Modern flexible option |
| Best fit           | Traditional node groups   | Dynamic workloads      |

Don't run both casually.

Choose an architecture intentionally.

---

# 64. Recommended Project Architecture

For this project:

```text
                    EKS
                     |
        +------------+------------+
        |                         |
        v                         v
      HPA                       KEDA
        |                         |
        v                         v
 HTTP microservices        Queue consumers
        |                         |
        +------------+------------+
                     |
                     v
                Kubernetes
                  Scheduler
                     |
                     v
                 Karpenter
                     |
                     v
                  EC2 Nodes
```

This gives us:

```text
HPA
+
KEDA
+
Karpenter
```

for application and event-driven scaling.

---

# 65. Recommended Scaling Strategy for 21 Services

Example:

```text
auth-service
HPA

user-service
HPA

customer-service
HPA

catalog-service
HPA

product-service
HPA

inventory-service
HPA

pricing-service
HPA

promotion-service
HPA

cart-service
HPA

order-service
HPA

payment-service
HPA

fraud-service
HPA

shipping-service
HPA/KEDA depending on architecture

delivery-service
HPA/KEDA depending on architecture

notification-service
KEDA

search-service
HPA

recommendation-service
HPA

review-service
HPA

tax-service
HPA

invoice-service
KEDA

reporting-service
KEDA
```

This is a starting architecture, not a mandatory configuration.

---

# 66. Low Traffic Architecture

Example:

```text
                    ALB
                     |
                     v
              3 replicas/service
                     |
                     v
               EKS Nodes
               3-5 nodes
```

Platform components remain available.

Application Pods stay near minimum replicas.

---

# 67. Medium Traffic Architecture

Traffic increases.

```text
                   ALB
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
      order       catalog     payment
      HPA          HPA          HPA
        |           |           |
      8 Pods       6 Pods      7 Pods
        \           |           /
         +----------+----------+
                    |
                    v
               Karpenter
                    |
                    v
            Additional Nodes
```

Only required services scale.

---

# 68. High Traffic Architecture

```text
                        ALB
                         |
                         v
                 21 Microservices
                         |
             +-----------+-----------+
             |                       |
             v                       v
            HPA                    KEDA
             |                       |
             v                       v
       More API Pods           More Worker Pods
             |                       |
             +-----------+-----------+
                         |
                         v
                      Scheduler
                         |
                         v
                     Karpenter
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
            AZ-A        AZ-B        AZ-C
          EC2 Nodes    EC2 Nodes    EC2 Nodes
```

---

# 69. What Happens During a Traffic Spike

Let's follow one complete event.

Traffic:

```text
2,000 -> 10,000 requests/sec
```

## Step 1

ALB sends more requests.

## Step 2

Application latency increases.

## Step 3

Prometheus metrics increase.

## Step 4

HPA evaluates metrics.

## Step 5

HPA increases replicas.

```text
order-service:
3 -> 10
```

## Step 6

Scheduler attempts placement.

## Step 7

Some Pods cannot fit.

```text
Pending
```

## Step 8

Karpenter detects unschedulable Pods.

## Step 9

Karpenter provisions suitable EC2 nodes.

## Step 10

Nodes join EKS.

## Step 11

Pending Pods are scheduled.

## Step 12

Readiness probes pass.

## Step 13

Service endpoints increase.

## Step 14

ALB/Service distributes traffic across more Pods.

Final:

```text
Traffic handled
+
latency controlled
+
capacity increased
```

---

# 70. What Happens When Traffic Falls

Traffic:

```text
10,000 -> 1,000 requests/sec
```

HPA detects reduced demand.

```text
20 Pods
   |
   v
10 Pods
   |
   v
5 Pods
   |
   v
3 Pods
```

Kubernetes now has underutilized nodes.

Karpenter can consolidate/remove unnecessary capacity according to its configured disruption and consolidation policies.

Conceptually:

```text
Traffic decreases
      |
      v
HPA removes Pods
      |
      v
Nodes become underutilized
      |
      v
Karpenter consolidation
      |
      v
Fewer EC2 nodes
```

The exact consolidation behavior depends on Karpenter version and configuration.

---

# 71. Autoscaling and Cost

Without autoscaling:

```text
Peak capacity
24 hours/day
```

With autoscaling:

```text
Low traffic:
small cluster

Medium traffic:
medium cluster

High traffic:
large cluster
```

This can significantly reduce compute waste.

But don't optimize cost so aggressively that:

```text
availability
```

is sacrificed.

---

# 72. Autoscaling and SLOs

Suppose:

```text
SLO:
99.9% requests under 500ms
```

If HPA scales too slowly:

```text
latency > 500ms
```

and SLO suffers.

If HPA scales too aggressively:

```text
cost increases
```

Therefore autoscaling is a reliability + cost optimization problem.

---

# 73. Capacity Testing

Before choosing:

```text
minReplicas
maxReplicas
target CPU
requests/sec target
node limits
```

run load tests.

Example:

```text
100 RPS
500 RPS
1,000 RPS
5,000 RPS
10,000 RPS
```

Measure:

```text
latency
CPU
memory
DB connections
Redis commands/sec
SQS backlog
Kafka lag
Pod startup time
node utilization
```

Then tune autoscaling.

---

# 74. Don't Load Test Only the Application

A realistic test must include dependencies.

Example:

```text
Load generator
      |
      v
ALB
      |
      v
order-service
      |
      +----> inventory
      |
      +----> payment
      |
      +----> RDS
      |
      +----> Redis
      |
      +----> SQS
```

Otherwise the test may produce misleading results.

---

# 75. Autoscaling Observability

Monitor HPA itself.

Useful information:

```bash
kubectl -n production get hpa
```

Detailed:

```bash
kubectl -n production describe hpa order-service
```

Check:

```text
current replicas
desired replicas
current metric
target metric
min replicas
max replicas
conditions
events
```

---

# 76. Troubleshooting HPA

If HPA does not scale:

Check:

```text
1. HPA exists
2. Target Deployment exists
3. Resource requests exist
4. Metrics are available
5. Current metric is above target
6. maxReplicas is not already reached
7. HPA events
```

Commands:

```bash
kubectl -n production get hpa

kubectl -n production describe hpa order-service

kubectl -n production get deployment order-service

kubectl -n production get pods
```

---

# 77. HPA Shows `<unknown>`

Possible causes:

```text
Metrics unavailable
```

Check:

```bash
kubectl get apiservice
```

and:

```bash
kubectl top pods -n production
```

If:

```text
kubectl top
```

doesn't work, investigate the metrics pipeline.

Remember:

```text
Metrics Server
```

and:

```text
Prometheus
```

are different systems.

---

# 78. HPA at Max Replicas

Suppose:

```text
current:
20

max:
20
```

and latency is still high.

Do not immediately increase:

```text
20 -> 100
```

First determine why.

Possible bottleneck:

```text
RDS
Redis
payment gateway
downstream service
CPU
memory
network
thread pool
connection pool
external API
```

Scaling Pods may make the bottleneck worse.

---

# 79. Pods Pending After HPA

Example:

```text
HPA:
3 -> 15

Pods:
10 Running
5 Pending
```

Check:

```bash
kubectl -n production get pods
kubectl -n production describe pod <pending-pod>
```

Look for:

```text
Insufficient cpu
Insufficient memory
node selector mismatch
taint/toleration mismatch
topology constraints
affinity rules
```

Then investigate Karpenter/node capacity.

---

# 80. Karpenter Troubleshooting

Check Karpenter Pods:

```bash
kubectl -n kube-system get pods
```

Check logs:

```bash
kubectl -n kube-system logs <karpenter-pod>
```

Investigate:

```text
NodePool constraints
NodeClass configuration
subnets
security groups
IAM permissions
instance availability
capacity type
AZ constraints
resource limits
```

---

# 81. Nodes Not Joining

If Karpenter provisions EC2 instances but they don't join EKS, investigate:

```text
IAM
bootstrap/user data
networking
subnets
security groups
EKS cluster connectivity
instance profile
VPC CNI
```

Architecture:

```text
Karpenter
   |
   v
EC2 created
   |
   X
Node doesn't join
```

The problem is now node/bootstrap infrastructure, not HPA.

---

# 82. Autoscaling Failure Chain

A useful troubleshooting chain is:

```text
Traffic
  |
  v
ALB
  |
  v
Service
  |
  v
Pods
  |
  v
Metrics
  |
  v
HPA
  |
  v
Desired replicas
  |
  v
Scheduler
  |
  v
Pending Pods?
  |
  v
Karpenter
  |
  v
EC2 Nodes
  |
  v
Pods scheduled
```

Check each layer instead of guessing.

---

# 83. Production Rules

### Rule 1

Never use autoscaling without resource requests.

### Rule 2

Never choose `maxReplicas` arbitrarily.

### Rule 3

Understand downstream capacity before increasing replicas.

### Rule 4

Use custom/event metrics where CPU is not representative.

### Rule 5

Use KEDA for queue-driven workloads where appropriate.

### Rule 6

Use multi-AZ capacity.

### Rule 7

Protect system workloads.

### Rule 8

Leave capacity headroom.

### Rule 9

Test scale-up and scale-down.

### Rule 10

Monitor autoscaling itself.

---

# 84. Final Production Architecture

Our recommended architecture is:

```text
                         Internet
                            |
                            v
                     CloudFront/WAF
                            |
                            v
                           ALB
                            |
                            v
                    21 Microservices
                            |
              +-------------+-------------+
              |                           |
              v                           v
             HPA                         KEDA
              |                           |
              v                           v
        API Pod scaling            Worker scaling
              |                           |
              +-------------+-------------+
                            |
                            v
                       Scheduler
                            |
                    +-------+-------+
                    |               |
              Capacity exists?    No capacity
                    |               |
                    v               v
                  Nodes          Karpenter
                                    |
                                    v
                               EC2 capacity
                                    |
                     +--------------+--------------+
                     |              |              |
                    AZ-A           AZ-B           AZ-C
```

Dependencies:

```text
Applications
    |
    +----> RDS
    |
    +----> Redis
    |
    +----> SQS/SNS
    |
    +----> Kafka/RabbitMQ where applicable
```

Autoscaling must consider all of them.

---

# 85. Final Mental Model

Remember:

```text
HPA
=
How many Pods?

VPA
=
How much CPU/memory should each Pod request?

KEDA
=
How many event-driven Pods?

Cluster Autoscaler
=
How many nodes in configured node groups?

Karpenter
=
What EC2 nodes should exist to satisfy unscheduled workloads?
```

And the complete scaling chain:

```text
                    TRAFFIC
                       |
                       v
                 APPLICATION
                       |
                       v
                    METRICS
                       |
          +------------+------------+
          |                         |
          v                         v
         HPA                       KEDA
          |                         |
          v                         v
     API replicas             Worker replicas
          |                         |
          +------------+------------+
                       |
                       v
                   SCHEDULER
                       |
                       v
                 Pending Pods?
                    /     \
                  No       Yes
                  |         |
                  v         v
                Done     Karpenter
                            |
                            v
                         EC2 Nodes
                            |
                            v
                       Pods Scheduled
```

The most important production lesson is:

```text
Autoscaling is not simply "add more Pods."

Autoscaling is:

Traffic
  -> Metrics
  -> Scaling decision
  -> Pod capacity
  -> Node capacity
  -> Dependency capacity
  -> Cost
  -> Reliability
```

A correctly designed autoscaling system scales the application **without accidentally overwhelming RDS, Redis, queues, external APIs, or other microservices**.

```
```
