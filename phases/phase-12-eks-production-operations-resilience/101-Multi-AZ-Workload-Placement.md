# 101 — Multi-AZ Workload Placement

## 1. Purpose

In Chapter 100, we implemented **Spot capacity** and learned that Spot resilience depends heavily on how Pods are distributed across the cluster.

Now we implement **Multi-AZ workload placement** for our EKS platform.

The objective is not simply:

> "Create nodes in three Availability Zones."

The real objective is:

> **Make sure application replicas are distributed across failure domains so that losing a node or an Availability Zone does not unnecessarily take the service offline.**

Our target:

```text
                    AWS Region
                   ap-south-1
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
        AZ-A          AZ-B          AZ-C
          |             |             |
       Nodes          Nodes          Nodes
          |             |             |
        Pods           Pods           Pods
          \             |             /
           \            |            /
            +-----------+-----------+
                        |
                  Kubernetes Service
                        |
                       ALB
```

---

# 2. Why Multi-AZ Matters

Consider:

```text
order-service
3 replicas
```

Bad placement:

```text
AZ-A
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

If AZ-A fails:

```text
AZ-A failure
     ↓
Pod 1 lost
Pod 2 lost
Pod 3 lost
     ↓
Service unavailable
```

Now consider:

```text
AZ-A       AZ-B       AZ-C
  |          |          |
 Pod 1      Pod 2      Pod 3
```

If AZ-A fails:

```text
AZ-A failure
     ↓
Pod 1 lost
     ↓
Pod 2 + Pod 3 remain
     ↓
Kubernetes replaces Pod 1
     ↓
Service continues
```

That is the purpose of Multi-AZ workload placement.

---

# 3. Failure Domains

We need to understand what Kubernetes is distributing.

A simplified hierarchy:

```text
AWS Region
    |
    +── Availability Zone A
    |      |
    |      +── Node
    |      +── Node
    |
    +── Availability Zone B
    |      |
    |      +── Node
    |      +── Node
    |
    +── Availability Zone C
           |
           +── Node
           +── Node
```

Then:

```text
Node
  |
  +── Pod
  +── Pod
  +── Pod
```

Failures can occur at different levels:

```text
Container failure
      ↓
Pod failure
      ↓
Node failure
      ↓
AZ failure
      ↓
Region failure
```

Multi-AZ specifically addresses the **Availability Zone failure domain**.

It does not protect against:

```text
Region failure
```

That requires the multi-region DR architecture we will implement later.

---

# 4. Our EKS Architecture

Our target production-style architecture:

```text
                        AWS Region
                       ap-south-1
                            |
       +--------------------+--------------------+
       |                    |                    |
       v                    v                    v
     AZ-A                 AZ-B                 AZ-C
       |                    |                    |
  +----+----+          +----+----+          +----+----+
  |         |          |         |          |         |
Node      Node        Node      Node        Node      Node
  |         |          |         |          |         |
Pods      Pods        Pods      Pods        Pods      Pods
```

The exact number of nodes depends on workload and autoscaling.

The important requirement is:

```text
Capacity exists in multiple AZs.
```

---

# 5. Multi-AZ Starts at the VPC Layer

Our VPC already uses three Availability Zones.

From the earlier infrastructure design:

```text
VPC
10.0.0.0/16

AZ-A
 ├── public subnet
 └── private subnet

AZ-B
 ├── public subnet
 └── private subnet

AZ-C
 ├── public subnet
 └── private subnet
```

Our EKS worker nodes should normally use:

```text
private subnets
```

The external ALB can use:

```text
public subnets
```

depending on the ingress architecture.

---

# 6. EKS Nodes Across AZs

We want:

```text
AZ-A → node capacity
AZ-B → node capacity
AZ-C → node capacity
```

For example:

```text
System NodeGroup

AZ-A → 1 node
AZ-B → 1 node
AZ-C → 1 node
```

Application capacity can then be dynamically added by Karpenter.

```text
Karpenter
    |
    +── AZ-A
    +── AZ-B
    +── AZ-C
```

---

# 7. Why Node Distribution Alone Is Not Enough

This is a common misunderstanding.

Suppose we have:

```text
AZ-A → 3 nodes
AZ-B → 3 nodes
AZ-C → 3 nodes
```

That does **not** automatically mean:

```text
order-service
Pod 1 → AZ-A
Pod 2 → AZ-B
Pod 3 → AZ-C
```

Kubernetes scheduler may place Pods based on:

* resource availability
* affinity
* taints
* topology constraints
* scheduling policies
* existing workload distribution

Therefore:

> **Multi-AZ nodes do not automatically guarantee Multi-AZ Pods.**

We need explicit workload placement rules.

---

# 8. Kubernetes Topology Labels

Kubernetes nodes normally expose topology information such as:

```text
topology.kubernetes.io/zone
```

Check:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Example:

```text
NAME          STATUS   ZONE
node-a        Ready    ap-south-1a
node-b        Ready    ap-south-1b
node-c        Ready    ap-south-1c
```

Also inspect:

```bash
kubectl describe node <NODE>
```

Look for:

```text
topology.kubernetes.io/zone
```

---

# 9. The Three Main Scheduling Mechanisms

For Multi-AZ workload placement, we will primarily use:

```text
1. Topology Spread Constraints
2. Pod Anti-Affinity
3. Node Affinity
```

They solve different problems.

```text
Topology Spread
       ↓
Distribute Pods across topology domains

Pod Anti-Affinity
       ↓
Avoid placing matching Pods together

Node Affinity
       ↓
Choose nodes based on node attributes
```

---

# 10. Topology Spread Constraints

This is one of the most important tools for our architecture.

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

The important fields are:

```text
maxSkew
topologyKey
whenUnsatisfiable
labelSelector
```

---

# 11. What Does `topologyKey` Mean?

We use:

```yaml
topologyKey: topology.kubernetes.io/zone
```

This tells Kubernetes:

> Distribute Pods across Availability Zones.

Therefore:

```text
AZ-A
AZ-B
AZ-C
```

become the topology domains.

We could also use:

```yaml
topologyKey: kubernetes.io/hostname
```

to distribute Pods across individual nodes.

That gives us:

```text
AZ-level spreading
```

or:

```text
Node-level spreading
```

depending on the key.

---

# 12. What Does `maxSkew` Mean?

Suppose we have:

```text
3 replicas
3 AZs
```

Ideal:

```text
AZ-A → 1
AZ-B → 1
AZ-C → 1
```

Skew:

```text
0
```

With:

```yaml
maxSkew: 1
```

Kubernetes allows a difference of at most one between topology domains, subject to the rest of the scheduling constraints.

For example:

```text
AZ-A → 2
AZ-B → 1
AZ-C → 1
```

can be valid.

But:

```text
AZ-A → 3
AZ-B → 0
AZ-C → 0
```

does not satisfy a balanced AZ distribution under the intended constraint.

---

# 13. `whenUnsatisfiable`

We have two important choices.

### DoNotSchedule

```yaml
whenUnsatisfiable: DoNotSchedule
```

Kubernetes will not schedule the Pod if doing so would violate the topology constraint.

This provides stronger placement guarantees but can cause:

```text
Pod Pending
```

if capacity is insufficient.

### ScheduleAnyway

```yaml
whenUnsatisfiable: ScheduleAnyway
```

Kubernetes attempts to maintain the spread but can place the Pod elsewhere when the ideal distribution is impossible.

This improves scheduling flexibility.

---

# 14. Which Should We Use?

For critical services, we may choose:

```yaml
whenUnsatisfiable: DoNotSchedule
```

when we have sufficient capacity and the availability requirement justifies it.

For less critical workloads:

```yaml
whenUnsatisfiable: ScheduleAnyway
```

may be appropriate.

There is no universal setting.

The decision depends on:

```text
SLO
replica count
cluster capacity
startup time
workload criticality
cost
```

---

# 15. Order Service Example

Our Order Service can use:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:

    metadata:
      labels:
        app: order-service

    spec:

      topologySpreadConstraints:

        - maxSkew: 1

          topologyKey: topology.kubernetes.io/zone

          whenUnsatisfiable: DoNotSchedule

          labelSelector:
            matchLabels:
              app: order-service
```

This tells Kubernetes to distribute Order Service Pods across zones.

---

# 16. What Happens With Three Replicas?

Suppose the cluster contains:

```text
AZ-A
AZ-B
AZ-C
```

and we deploy:

```text
replicas: 3
```

The scheduler attempts:

```text
Pod 1 → AZ-A
Pod 2 → AZ-B
Pod 3 → AZ-C
```

The actual result depends on all scheduling constraints and available capacity, but the topology constraint strongly guides the scheduler toward this distribution.

---

# 17. What Happens With Six Replicas?

With:

```yaml
replicas: 6
```

we ideally want:

```text
AZ-A → 2
AZ-B → 2
AZ-C → 2
```

This gives:

```text
       AZ-A     AZ-B     AZ-C

        2        2        2
        |        |        |
       Pods     Pods     Pods
```

---

# 18. HPA and Multi-AZ

This becomes more interesting when HPA scales.

Initial:

```text
3 replicas

AZ-A → 1
AZ-B → 1
AZ-C → 1
```

Traffic increases.

HPA:

```text
3 → 6
```

The scheduler attempts to maintain balanced topology:

```text
AZ-A → 2
AZ-B → 2
AZ-C → 2
```

Traffic increases again:

```text
6 → 9
```

Ideally:

```text
AZ-A → 3
AZ-B → 3
AZ-C → 3
```

This is why topology constraints become very important once we introduce autoscaling.

---

# 19. Karpenter and Multi-AZ

Karpenter must also be able to provision capacity in the required zones.

For example:

```text
Pending Pods
      |
      v
Scheduler constraints
      |
      v
Karpenter
      |
      +── AZ-A
      +── AZ-B
      +── AZ-C
```

If the workload requires:

```text
AZ-A
```

but Karpenter can only provision:

```text
AZ-B
AZ-C
```

the Pod may remain Pending.

Therefore:

> Workload placement constraints and node provisioning constraints must be designed together.

---

# 20. Karpenter Zone Requirements

A NodePool can allow multiple zones.

Conceptually:

```yaml
requirements:

  - key: topology.kubernetes.io/zone
    operator: In
    values:
      - ap-south-1a
      - ap-south-1b
      - ap-south-1c
```

This gives Karpenter multiple possible failure domains.

Again, the exact NodePool schema depends on the installed Karpenter version.

---

# 21. Pod Anti-Affinity

Another scheduling mechanism is Pod anti-affinity.

Example:

```yaml
affinity:

  podAntiAffinity:

    preferredDuringSchedulingIgnoredDuringExecution:

      - weight: 100

        podAffinityTerm:

          topologyKey: topology.kubernetes.io/zone

          labelSelector:
            matchLabels:
              app: order-service
```

This expresses:

> Prefer not to place matching Order Service Pods in the same Availability Zone.

---

# 22. Required Pod Anti-Affinity

We can also use:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
```

This is much stronger.

It means the scheduler must respect the rule.

However, overly strict anti-affinity can create:

```text
Pod Pending
```

For example:

```text
3 replicas
3 AZs
```

works well.

But:

```text
10 replicas
3 AZs
required one per AZ
```

requires careful design because the rule may become unnecessarily restrictive depending on the topology key and other constraints.

---

# 23. Topology Spread vs Anti-Affinity

### Topology Spread

Focuses on:

```text
Balance
```

Example:

```text
2 / 2 / 2
```

### Anti-Affinity

Focuses on:

```text
Avoid co-location
```

Example:

```text
Don't put these replicas together.
```

For our platform, topology spread is often the clearer primary mechanism for AZ balancing.

Anti-affinity can complement it.

---

# 24. Node Affinity

Node affinity is different.

Suppose nodes have:

```text
capacity-type=spot
```

or:

```text
workload=application
```

A Pod can select nodes based on these labels.

Example:

```yaml
nodeAffinity:

  requiredDuringSchedulingIgnoredDuringExecution:

    nodeSelectorTerms:

      - matchExpressions:

          - key: workload
            operator: In
            values:
              - application
```

This is about:

```text
Node characteristics
```

rather than directly distributing replicas across zones.

---

# 25. Combining the Three

Our Order Service could use:

```text
Node Affinity
      ↓
Choose application-capable nodes

Topology Spread
      ↓
Distribute across AZs

Pod Anti-Affinity
      ↓
Avoid undesirable co-location
```

Conceptually:

```text
                 Order Service
                       |
            +----------+----------+
            |          |          |
            v          v          v
          AZ-A       AZ-B       AZ-C
            |          |          |
           Pod        Pod        Pod
```

---

# 26. Multi-AZ and Spot

Chapter 100 introduced Spot.

Now combine them.

```text
                         EKS
                          |
              +-----------+-----------+
              |                       |
          On-Demand                  Spot
              |                       |
       +------+-------+        +------+-------+
       |      |       |        |      |       |
      AZ-A   AZ-B    AZ-C     AZ-A   AZ-B    AZ-C
```

This gives us:

```text
capacity diversity
+
AZ diversity
+
workload diversity
```

---

# 27. The Most Important Rule

Never assume:

```text
3 AZs
=
3-way application redundancy
```

You need:

```text
3 AZs
+
multiple replicas
+
topology constraints
+
sufficient capacity
+
healthy probes
+
correct service routing
```

Only then do we have meaningful Multi-AZ application resilience.

---

# 28. ALB Multi-AZ

Our external traffic path is:

```text
Internet
   ↓
Route 53
   ↓
CloudFront/WAF
   ↓
ALB
   ↓
EKS Service
   ↓
Pods
```

The ALB operates across Availability Zones where configured subnets and targets are available.

The application Pods should also be distributed.

Otherwise we can have:

```text
ALB = Multi-AZ
Pods = single AZ
```

which creates a hidden failure domain.

---

# 29. Kubernetes Service

The Service provides a stable abstraction:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service

spec:
  type: ClusterIP

  selector:
    app: order-service

  ports:
    - port: 80
      targetPort: 8080
```

The Service selects healthy Pods regardless of which AZ they are in.

---

# 30. Endpoint Distribution

Check:

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=order-service
```

We should inspect whether endpoints exist across the expected Pods.

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o wide
```

Example:

```text
NAME             NODE       ZONE
order-abc        node-a     ap-south-1a
order-def        node-b     ap-south-1b
order-ghi        node-c     ap-south-1c
```

---

# 31. Testing Multi-AZ Placement

Deploy:

```yaml
replicas: 3
```

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o wide
```

Check:

```text
NODE
```

Then map nodes to zones:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Combine the results.

We want to see:

```text
Pod 1 → AZ-A
Pod 2 → AZ-B
Pod 3 → AZ-C
```

---

# 32. Scale to Six Replicas

```bash
kubectl scale deployment order-service \
  -n ecommerce-prod \
  --replicas=6
```

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o wide
```

Check whether the Pods distribute approximately:

```text
AZ-A → 2
AZ-B → 2
AZ-C → 2
```

The exact distribution depends on all active constraints and available capacity.

---

# 33. Scale Using HPA

Later our HPA will control replicas.

For now, we can simulate:

```text
3
↓
6
↓
9
```

and observe the topology distribution.

The important interaction is:

```text
HPA
 ↓
Replica count increases
 ↓
Scheduler
 ↓
Topology constraints
 ↓
Karpenter if capacity is missing
```

---

# 34. AZ Failure Simulation

A useful production test is:

```text
AZ-A unavailable
```

We need to understand what happens.

Suppose:

```text
AZ-A → 2 Pods
AZ-B → 2 Pods
AZ-C → 2 Pods
```

AZ-A fails.

Remaining:

```text
AZ-B → 2
AZ-C → 2
```

Kubernetes detects the failure.

Then:

```text
Desired replicas = 6
Available = 4
```

The cluster needs replacement capacity.

Karpenter can provision nodes in healthy AZs if the requirements allow it.

Then:

```text
AZ-B → 3
AZ-C → 3
```

or another valid distribution.

---

# 35. Important AZ Failure Limitation

Kubernetes cannot magically recreate a Pod if:

```text
No capacity exists
```

Therefore:

```text
AZ failure
+
insufficient remaining capacity
=
possible outage
```

This is why we need:

```text
headroom
+
autoscaling
+
Karpenter
+
multiple AZs
```

---

# 36. Capacity Headroom

Suppose:

```text
Current workload:
AZ-A → 10 Pods
AZ-B → 10 Pods
AZ-C → 10 Pods
```

Total:

```text
30 Pods
```

If AZ-A disappears:

```text
20 Pods remain
10 need replacement
```

If the remaining AZs have zero spare capacity:

```text
10 Pods Pending
```

Therefore, Multi-AZ resilience requires enough capacity to absorb failures.

Possible solutions:

```text
extra node capacity
+
Karpenter
+
On-Demand baseline
+
Spot capacity
```

---

# 37. PDB During Node/AZ Disruption

PDB can protect against excessive voluntary disruption.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: order-service

spec:
  minAvailable: 4

  selector:
    matchLabels:
      app: order-service
```

If:

```text
replicas = 6
```

then we want:

```text
4 available
```

during supported voluntary disruptions.

Again:

> PDB is not a shield against an actual AZ outage.

If an entire AZ disappears, those Pods are gone regardless of PDB.

---

# 38. Stateful Workloads and AZs

This becomes more complicated for stateful applications.

For example:

```text
EBS volume
```

is associated with a specific Availability Zone.

A Pod using an EBS volume cannot simply attach that same volume to an arbitrary node in another AZ.

This is why our production database architecture uses:

```text
RDS PostgreSQL
```

rather than relying on an application-managed PostgreSQL StatefulSet for the main production database.

---

# 39. RDS Multi-AZ

Our application:

```text
EKS
 ↓
RDS
```

RDS provides its own high-availability mechanisms.

The application should not try to recreate database HA using:

```text
three PostgreSQL Pods
```

unless we have deliberately chosen an operator-managed database architecture.

This is another important principle:

> Each layer should use the appropriate HA mechanism.

---

# 40. Redis Multi-AZ

Our production Redis architecture uses:

```text
ElastiCache
```

rather than a random Redis Pod on an EKS node.

Therefore:

```text
EKS Multi-AZ
```

and:

```text
Redis Multi-AZ
```

are separate resilience layers.

---

# 41. Multi-AZ Application Dependency Chain

For an Order request:

```text
Client
  ↓
Route 53
  ↓
CloudFront/WAF
  ↓
ALB
  ↓
EKS
  ↓
order-service Pods
  ↓
RDS
```

For resilience:

```text
ALB
 ↓
Multi-AZ
 ↓
EKS Multi-AZ
 ↓
Pods Multi-AZ
 ↓
RDS Multi-AZ
```

Every critical dependency must be considered.

Having a Multi-AZ application with a single-AZ dependency still leaves a failure domain.

---

# 42. Anti-Pattern: Single-AZ NodePool

Bad:

```yaml
requirements:
  - key: topology.kubernetes.io/zone
    operator: In
    values:
      - ap-south-1a
```

This restricts capacity to one AZ.

For production application capacity, we normally want multiple zones:

```yaml
requirements:
  - key: topology.kubernetes.io/zone
    operator: In
    values:
      - ap-south-1a
      - ap-south-1b
      - ap-south-1c
```

The exact selection depends on the cluster's subnet and capacity design.

---

# 43. Anti-Pattern: All Replicas on One Node

Bad:

```text
Node-A
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

Better:

```text
Node-A → Pod 1
Node-B → Pod 2
Node-C → Pod 3
```

Best when appropriate:

```text
AZ-A → Pod 1
AZ-B → Pod 2
AZ-C → Pod 3
```

We can use:

```text
topology spread
+
anti-affinity
```

to achieve this.

---

# 44. Anti-Pattern: Overly Strict Constraints

The opposite problem is also possible.

Suppose:

```text
3 replicas
3 AZs
```

and we require:

```text
specific instance type
+
specific AZ
+
Spot
+
specific node label
+
specific taint
+
strict anti-affinity
```

The scheduler may have no legal placement.

Result:

```text
Pod Pending
```

Therefore:

> High availability does not mean adding unlimited scheduling constraints.

Constraints must be compatible.

---

# 45. Debugging a Multi-AZ Scheduling Problem

Start with:

```bash
kubectl get pods -n ecommerce-prod -o wide
```

Then:

```bash
kubectl describe pod <POD> -n ecommerce-prod
```

Look at:

```text
Events
```

Then:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Then:

```bash
kubectl get nodepool
```

Then:

```bash
kubectl describe nodepool <NODEPOOL>
```

Look for:

```text
zone requirements
instance requirements
capacity type
taints
resources
```

---

# 46. Check Pod Topology Constraints

You can inspect the Deployment:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o yaml
```

Look for:

```yaml
topologySpreadConstraints:
```

and:

```yaml
affinity:
```

---

# 47. Check Node Labels

```bash
kubectl get nodes --show-labels
```

Or more readable:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone \
  -L node.kubernetes.io/instance-type \
  -L karpenter.sh/capacity-type
```

This is one of the most useful commands for understanding placement.

---

# 48. Check Current Pod Distribution

For a service:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o wide
```

Create a simple mental table:

```text
Pod       Node       AZ
---------------------------
pod-1     node-a     AZ-A
pod-2     node-b     AZ-B
pod-3     node-c     AZ-C
```

This should become a routine troubleshooting technique.

---

# 49. Testing Node Failure

We can test a controlled node disruption.

First identify the node:

```bash
kubectl get nodes
```

Then:

```bash
kubectl cordon <NODE>
```

This prevents new Pods from being scheduled there.

Then:

```bash
kubectl drain <NODE> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

**Do this only in a controlled test environment.**

Observe:

```bash
kubectl get pods -A -o wide -w
```

Then verify Pods move to other nodes/AZs.

---

# 50. Cordon vs Drain

### Cordon

```bash
kubectl cordon <NODE>
```

means:

```text
Don't schedule new Pods here.
```

Existing Pods remain.

### Drain

```bash
kubectl drain <NODE>
```

means:

```text
Evict eligible Pods from the node.
```

This is useful during:

```text
maintenance
node replacement
planned failure testing
```

---

# 51. Testing AZ-Level Resilience

We should **not manually shut down an entire production AZ**.

For controlled testing, use:

* AWS Fault Injection Service where supported
* controlled node termination
* simulated capacity loss
* isolated test environments

The objective is to test the application's response without creating unnecessary production risk.

---

# 52. Multi-AZ Failure Test

Test scenario:

```text
Initial:
AZ-A → 2 Pods
AZ-B → 2 Pods
AZ-C → 2 Pods
```

Simulate loss of AZ-A capacity.

Observe:

```text
AZ-A → unavailable
AZ-B → 2 Pods
AZ-C → 2 Pods
```

Then:

```text
Kubernetes
     ↓
Detect failure
     ↓
Pods become unavailable
     ↓
Karpenter
     ↓
Provision capacity
     ↓
Scheduler
     ↓
Replacement Pods
```

Expected final state:

```text
AZ-B → 3 Pods
AZ-C → 3 Pods
```

or another valid distribution.

---

# 53. What We Measure During the Test

Don't only check Pod status.

Measure:

```text
Availability
Error rate
Latency
ALB 5xx
Pod readiness
Pod restart count
Pending Pods
Node provisioning time
Pod startup time
RDS connectivity
Redis connectivity
SQS processing
```

This connects Multi-AZ testing to observability.

---

# 54. Application-Level Success Criteria

For our Order Service:

```text
Before:
6 replicas

AZ-A → 2
AZ-B → 2
AZ-C → 2
```

During failure:

```text
AZ-A unavailable
```

Target:

```text
No prolonged application outage
Remaining Pods serve traffic
Replacement capacity launches
Desired replica count recovers
Error rate returns to normal
```

The exact SLO threshold will be defined later in our SLO chapter.

---

# 55. Multi-AZ and Rolling Deployments

Our rolling deployment strategy also benefits from Multi-AZ placement.

Suppose:

```text
Version 1
6 replicas
```

distributed:

```text
AZ-A → 2
AZ-B → 2
AZ-C → 2
```

During deployment:

```text
v1
 ↓
v2
```

we should avoid accidentally concentrating all new replicas in one AZ.

Topology constraints continue to influence placement.

This works together with:

```text
maxUnavailable
maxSurge
PDB
readiness probes
```

---

# 56. Multi-AZ and Blue-Green

Blue-Green deployments also need Multi-AZ thinking.

```text
Blue
 ├── AZ-A
 ├── AZ-B
 └── AZ-C

Green
 ├── AZ-A
 ├── AZ-B
 └── AZ-C
```

Do not build Green entirely in one AZ.

Otherwise:

```text
Switch traffic
   ↓
Green AZ failure
   ↓
Production outage
```

---

# 57. Multi-AZ and Canary

Canary workloads should also be distributed appropriately.

Example:

```text
Stable
  |
  +-- AZ-A
  +-- AZ-B
  +-- AZ-C

Canary
  |
  +-- AZ-A
  +-- AZ-B
  +-- AZ-C
```

A canary should not accidentally represent:

```text
one Pod
one node
one AZ
```

because then the experiment also measures an AZ-specific failure condition.

---

# 58. Multi-AZ and Spot

Now combine both chapters.

```text
                 EKS
                  |
       +----------+----------+
       |                     |
   On-Demand               Spot
       |                     |
   +---+---+             +---+---+
   |   |   |             |   |   |
  AZ-A AZ-B AZ-C         AZ-A AZ-B AZ-C
   |   |   |             |   |   |
   +---+---+-------------+---+---+
                  |
             Application Pods
```

A Spot interruption becomes less dangerous because:

```text
Pods
 ↓
multiple nodes
 ↓
multiple AZs
 ↓
replacement capacity
```

---

# 59. Multi-AZ and Cluster Autoscaling

Later, our autoscaling architecture will become:

```text
Traffic
   ↓
HPA
   ↓
More Pods
   ↓
Pending Pods
   ↓
Karpenter
   ↓
Select AZ + instance type + capacity type
   ↓
Launch EC2
   ↓
Scheduler
   ↓
Topology constraints
   ↓
Pods distributed
```

This is the complete capacity-placement loop.

---

# 60. Production Workload Placement Standard

For our platform, a standard application Deployment should generally consider:

```text
replicas
+
resources
+
readiness
+
liveness
+
startup
+
topology spread
+
PDB
+
capacity type
+
node affinity
+
tolerations
+
graceful shutdown
```

Not every service needs every feature with identical values.

The platform template should provide safe defaults.

---

# 61. Example Production Template

A standard service could contain:

```yaml
spec:

  replicas: 3

  template:

    spec:

      terminationGracePeriodSeconds: 60

      topologySpreadConstraints:

        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule

          labelSelector:
            matchLabels:
              app: order-service

      affinity:

        nodeAffinity:

          preferredDuringSchedulingIgnoredDuringExecution:

            - weight: 80

              preference:

                matchExpressions:

                  - key: karpenter.sh/capacity-type
                    operator: In
                    values:
                      - spot

      containers:

        - name: order-service

          resources:

            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: "1"
              memory: 1Gi
```

The actual platform template will be standardized later in Phase 20.

---

# 62. Production Checklist

Before declaring a service Multi-AZ ready:

```text
[ ] EKS has capacity in multiple AZs
[ ] Private subnets exist in required AZs
[ ] Node capacity can be provisioned in multiple AZs
[ ] Multiple replicas configured
[ ] topologySpreadConstraints configured
[ ] Pod anti-affinity evaluated
[ ] Node affinity reviewed
[ ] Taints/tolerations reviewed
[ ] PDB configured
[ ] Readiness probe configured
[ ] Graceful shutdown configured
[ ] HPA tested
[ ] Karpenter tested
[ ] RDS HA reviewed
[ ] Redis HA reviewed
[ ] Messaging HA reviewed
[ ] AZ failure scenario tested
[ ] Monitoring configured
[ ] Recovery time measured
```

---

# 63. Common Mistakes

## Mistake 1 — Only Creating Three AZs

```text
Three AZs
≠
Multi-AZ application
```

You must distribute workloads.

---

## Mistake 2 — No Topology Constraints

Without explicit scheduling requirements, Pods may concentrate.

---

## Mistake 3 — Too Many Required Constraints

Over-constraining scheduling can create Pending Pods.

---

## Mistake 4 — No Capacity Headroom

If one AZ fails and remaining AZs have no capacity, Pods cannot recover immediately.

---

## Mistake 5 — Ignoring Stateful Dependencies

Application Pods may be Multi-AZ while the database isn't appropriately protected.

---

## Mistake 6 — Testing Only Kubernetes

The complete path must be tested:

```text
ALB
 ↓
Service
 ↓
Pods
 ↓
RDS
 ↓
Redis
 ↓
Messaging
```

---

# 64. Troubleshooting Decision Tree

```text
Pod Pending
    |
    v
Check describe pod
    |
    v
Scheduling event?
    |
 +--+------------------+
 |                     |
YES                   NO
 |                     |
 v                     v
Check topology       Check node
constraints          readiness
 |
 +----------------------------+
 |
 v
Check NodePool
 |
 +----------------------------+
 |
 v
Check AZ capacity
 |
 +----------------------------+
 |
 v
Check instance requirements
 |
 +----------------------------+
 |
 v
Check taints / tolerations
 |
 +----------------------------+
 |
 v
Check resources
```

---

# 65. Golden Commands

### List nodes by AZ

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

### List nodes by capacity type

```bash
kubectl get nodes \
  -L karpenter.sh/capacity-type
```

### List nodes with instance type and AZ

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone \
  -L node.kubernetes.io/instance-type
```

### Check Pod distribution

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o wide
```

### Inspect scheduling failure

```bash
kubectl describe pod \
  <POD> \
  -n ecommerce-prod
```

### Check topology configuration

```bash
kubectl get deployment \
  order-service \
  -n ecommerce-prod \
  -o yaml
```

### Check PDB

```bash
kubectl get pdb \
  -n ecommerce-prod
```

### Check EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=order-service
```

---

# 66. Complete Example

Suppose our production environment has:

```text
Region:
ap-south-1
```

AZs:

```text
ap-south-1a
ap-south-1b
ap-south-1c
```

Order Service:

```text
6 replicas
```

Desired placement:

```text
AZ-A
 ├── Pod 1
 └── Pod 2

AZ-B
 ├── Pod 3
 └── Pod 4

AZ-C
 ├── Pod 5
 └── Pod 6
```

Then:

```text
AZ-A failure
```

Remaining:

```text
AZ-B → 2
AZ-C → 2
```

Kubernetes detects the lost capacity.

Karpenter provisions additional capacity.

Then:

```text
AZ-B → 3
AZ-C → 3
```

The service returns to:

```text
6 healthy replicas
```

This is the behavior we want.

---

# 67. Complete Production Architecture

Our architecture now combines Chapters 100 and 101:

```text
                         INTERNET
                            |
                            v
                   Route 53 / CloudFront
                            |
                            v
                           ALB
                            |
                            v
                       EKS CLUSTER
                            |
       +--------------------+--------------------+
       |                    |                    |
      AZ-A                 AZ-B                 AZ-C
       |                    |                    |
 +-----+-----+        +-----+-----+        +-----+-----+
 |           |        |           |        |           |
Node        Node     Node        Node     Node        Node
 |           |        |           |        |           |
Pods        Pods     Pods        Pods     Pods        Pods
       |                    |                    |
       +--------------------+--------------------+
                            |
                   Topology Spread
                            |
                   Kubernetes Service
                            |
             +--------------+--------------+
             |                             |
            RDS                         Redis
             |
          Messaging
```

---

# 68. What We Achieved

After Chapter 100:

```text
Spot capacity
```

After Chapter 101:

```text
Spot
  +
Multi-AZ
```

Our platform now has:

```text
Cost optimization
        +
Capacity diversification
        +
AZ failure-domain isolation
        +
Workload distribution
```

The next problem is:

> **How much CPU and memory should each Pod request, and how do we prevent one workload from consuming all available cluster resources?**

That is the purpose of the next chapter.

```text
100 → Spot Instances                    ✅
101 → Multi-AZ Workload Placement       ✅
102 → Kubernetes Resource Management    → NEXT
103 → HPA / VPA / KEDA
104 → Cluster Autoscaling
```

# Key Takeaway

Multi-AZ is not simply an AWS networking feature.

It is a **workload scheduling design**:

```text
Multiple AZs
     +
Multiple Nodes
     +
Multiple Replicas
     +
Topology Spread
     +
Appropriate Affinity
     +
PDB
     +
Capacity Headroom
     +
Autoscaling
     =
Production Workload Resilience
```

The goal is:

```text
Node failure
      ↓
Application survives

Spot interruption
      ↓
Application survives

AZ failure
      ↓
Application survives

Region failure
      ↓
DR architecture
```

Region-level failure will be handled later in our Disaster Recovery phase.
