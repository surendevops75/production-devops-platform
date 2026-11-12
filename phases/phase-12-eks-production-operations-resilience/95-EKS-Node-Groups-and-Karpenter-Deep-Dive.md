# 95 — EKS Node Groups and Karpenter Deep Dive

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Workloads:** 21 microservices
> **Purpose:** Understand and implement production-grade EKS compute capacity using managed node groups and Karpenter.

---

# 1. Objective

In Chapter 94 we learned how to operate EKS nodes.

Now we go deeper into **how those nodes are created, selected, scaled, replaced, and removed**.

The core question is:

> **When a new Pod cannot be scheduled because the cluster doesn't have enough capacity, how does a production EKS platform obtain a suitable EC2 instance?**

There are two major approaches:

```text
                    EKS COMPUTE
                        │
             ┌──────────┴──────────┐
             │                     │
     Managed Node Groups        Karpenter
             │                     │
      Predefined capacity     Dynamic capacity
```

We can use both.

---

# 2. The Basic Problem

Suppose our application currently has:

```text
id="ecommerce-dev"
```

and:

```text
order-service
replicas = 3
```

Everything fits.

Then traffic increases.

HPA changes:

```text
3 replicas
     ↓
10 replicas
```

But the existing nodes don't have enough capacity.

The new Pods become:

```text
Pending
```

Now something must provide additional compute.

```text
HPA
 │
 ▼
More Pods
 │
 ▼
Scheduler
 │
 ▼
No capacity
 │
 ▼
Capacity provisioner
 │
 ▼
New EC2 node
 │
 ▼
Pod scheduled
```

That capacity provisioner can be Cluster Autoscaler or Karpenter.

---

# 3. Managed Node Groups

A managed node group is an AWS-managed way to run a group of EC2 worker nodes for EKS.

Conceptually:

```text
EKS
 │
 ▼
Managed Node Group
 │
 ├── EC2
 ├── EC2
 └── EC2
```

The nodes normally share characteristics such as:

* instance type
* AMI family
* capacity type
* scaling configuration
* subnets
* Kubernetes labels/taints

---

# 4. Example Managed Node Group

Our production cluster might have:

```text
system-ng
application-ng
```

For example:

```text
system-ng
├── m6i.large
├── m6i.large
└── m6i.large

application-ng
├── m6i.large
├── m6i.large
├── m6i.large
└── m6i.large
```

The exact instance types are workload-dependent.

---

# 5. Why Managed Node Groups Are Useful

They provide a relatively straightforward operational model.

AWS handles much of the node-group lifecycle integration with EKS.

We still control:

```text
Instance type
Subnets
Scaling
Labels
Taints
Capacity type
AMI/update configuration
```

They are a good fit for predictable baseline capacity.

---

# 6. Baseline Capacity

For our platform, we should always have some baseline capacity for critical components.

Example:

```text
System Node Group
        │
        ├── CoreDNS
        ├── VPC CNI
        ├── kube-proxy
        ├── EBS CSI
        ├── ArgoCD
        └── Monitoring
```

Then dynamic application capacity can be handled separately.

---

# 7. System vs Application Nodes

Example:

```text
                    EKS
                     │
          ┌──────────┴──────────┐
          │                     │
       System                Application
       Nodes                   Nodes
          │                     │
    Platform Pods          Microservices
```

This separation helps protect cluster infrastructure from application workload spikes.

---

# 8. Labels

Example node label:

```yaml
workload-type: system
```

or:

```yaml
workload-type: application
```

Check:

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl get nodes \
  -L workload-type
```

---

# 9. Taints

Suppose we want system nodes reserved for platform components.

A taint can be applied:

```bash
kubectl taint nodes <node-name> \
  workload=system:NoSchedule
```

Then application Pods without a matching toleration won't be scheduled there.

---

# 10. Toleration

A platform Pod can explicitly tolerate the taint:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: system
    effect: NoSchedule
```

This creates:

```text
System node
    │
    ├── taint
    │
    ▼
Only compatible workloads
```

---

# 11. Important Distinction

A node label says:

> **This node has this characteristic.**

A taint says:

> **Don't schedule here unless you explicitly tolerate me.**

A toleration says:

> **This Pod is allowed onto that tainted node.**

---

# 12. Node Group Scaling

Managed node groups normally have:

```text
minSize
desiredSize
maxSize
```

Example:

```text
min = 3
desired = 3
max = 10
```

Meaning:

```text
3 nodes
   │
   ├── normal operation
   │
   ▼
up to 10 nodes
```

But something still needs to decide when to change the node count.

That's where Cluster Autoscaler traditionally comes in.

---

# 13. Cluster Autoscaler

Conceptually:

```text
Pending Pod
     │
     ▼
Cluster Autoscaler
     │
     ▼
Managed Node Group
     │
     ▼
Increase desired capacity
     │
     ▼
EC2 instance
```

Cluster Autoscaler primarily reasons about node groups.

---

# 14. Karpenter

Karpenter takes a different approach.

Instead of primarily saying:

> "Increase node group X from 3 to 4."

it reasons more directly about:

> "What compute capacity can satisfy this unscheduled Pod?"

Conceptually:

```text
Pending Pod
    │
    ▼
Karpenter
    │
    ├── CPU
    ├── Memory
    ├── Architecture
    ├── AZ
    ├── Instance types
    ├── Capacity type
    └── Constraints
    │
    ▼
EC2 capacity
```

---

# 15. Karpenter Architecture

The exact resource names depend on the Karpenter version being used.

Modern Karpenter configurations commonly involve:

```text
NodePool
   │
   ├── Scheduling requirements
   ├── limits
   ├── disruption policy
   └── requirements
        │
        ▼
NodeClass
        │
        ├── AMI
        ├── subnet
        ├── security group
        ├── IAM role
        └── EC2 configuration
```

Think:

```text
NodePool
=
"What kind of capacity may I create?"

NodeClass
=
"How should that EC2 capacity be configured?"
```

---

# 16. Why Karpenter Is Powerful

Imagine:

```text
Pod requires:
4 CPU
8 GiB memory
```

The cluster has:

```text
m6i.large
2 CPU
8 GiB
```

A node group may need to add another fixed-size node.

Karpenter can evaluate available compatible instance types and provision an appropriate node based on its constraints.

---

# 17. Karpenter Doesn't Replace the Scheduler

This is important.

Kubernetes scheduler still decides:

> Which node should run the Pod?

Karpenter helps create capacity when the existing nodes cannot satisfy scheduling requirements.

Conceptually:

```text
Pod
 │
 ▼
Kubernetes Scheduler
 │
 ├── Fits existing node → schedule
 │
 └── Doesn't fit
          │
          ▼
       Karpenter
          │
          ▼
       New node
          │
          ▼
       Scheduler
          │
          ▼
        Pod
```

---

# 18. Complete Scaling Flow

Suppose:

```text
order-service
3 replicas
```

Traffic increases.

HPA:

```text
3
↓
8
```

Scheduler:

```text
5 new Pods
```

Some remain:

```text
Pending
```

Karpenter:

```text
Pending Pods
     ↓
Calculate requirements
     ↓
Select suitable capacity
     ↓
Launch node
```

Then:

```text
New node
   ↓
Joins EKS
   ↓
Scheduler
   ↓
Pending Pods
   ↓
Running
```

---

# 19. The Important Dependency Chain

Remember:

```text
Traffic
  ↓
HPA
  ↓
Pod count
  ↓
Scheduler
  ↓
Node capacity
  ↓
Karpenter
  ↓
EC2
```

Therefore if the application doesn't scale, don't immediately blame HPA.

The problem could be:

```text
HPA
Scheduler
NodePool
NodeClass
EC2 capacity
Networking
IAM
```

---

# 20. Karpenter NodePool

A simplified conceptual NodePool:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: application
spec:
  template:
    metadata:
      labels:
        workload-type: application
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: EC2NodeClass
        name: application
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64
        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - on-demand
        - key: topology.kubernetes.io/zone
          operator: In
          values:
            - ap-south-1a
            - ap-south-1b
            - ap-south-1c
```

> **Version note:** Karpenter APIs and fields evolve. Treat this as an architectural example and validate the exact API version and `nodeClassRef` fields against the Karpenter version installed in the cluster.

---

# 21. NodePool Requirements

We can constrain:

```text
Architecture
AZ
Capacity type
Instance family
Instance generation
Operating system
```

Example:

```yaml
requirements:
  - key: kubernetes.io/arch
    operator: In
    values:
      - amd64

  - key: karpenter.sh/capacity-type
    operator: In
    values:
      - on-demand
      - spot
```

---

# 22. Why Instance Flexibility Matters

Bad configuration:

```text
Only one instance type
```

For example:

```text
m6i.large
```

If that capacity isn't available:

```text
Karpenter
    ↓
No compatible capacity
    ↓
Pod remains Pending
```

Better:

```text
m6i.large
m6a.large
m7i.large
m7a.large
```

subject to application compatibility, pricing, availability, and organizational standards.

More flexibility can improve provisioning resilience.

---

# 23. Architecture Compatibility

A Pod might require:

```text
amd64
```

while your NodePool only provides:

```text
arm64
```

Then:

```text
Pod
 ↓
Scheduler
 ↓
No compatible node
 ↓
Pending
```

Check:

```bash
kubectl get nodes \
  -L kubernetes.io/arch
```

---

# 24. ARM vs AMD64

Our container image must support the architecture of the node.

For a multi-architecture strategy:

```text
Docker build
     │
     ├── amd64
     └── arm64
          │
          ▼
      Multi-arch image
```

Otherwise:

```text
ARM node
   ↓
AMD64-only image
   ↓
Container startup problem
```

Choose architecture intentionally.

---

# 25. On-Demand vs Spot

Karpenter can be configured to use different capacity types.

### On-Demand

```text
More predictable availability
Higher cost
```

### Spot

```text
Lower cost
Can be interrupted
Requires workload tolerance
```

---

# 26. Good Spot Candidates

Potential examples:

```text
Batch jobs
Async workers
Stateless services with sufficient replicas
Queue consumers
Non-critical workloads
```

But every workload must be evaluated individually.

---

# 27. Bad Assumption

Don't say:

> "We'll run everything on Spot because it is cheaper."

Instead:

```text
Critical
   ↓
On-Demand baseline

Interruptible
   ↓
Spot where appropriate
```

---

# 28. Mixed Capacity

Example:

```text
                    Application Capacity
                           │
                 ┌─────────┴─────────┐
                 │                   │
             On-Demand             Spot
                 │                   │
            Critical APIs       Workers / flexible
```

This can reduce cost while preserving capacity for critical services.

---

# 29. NodeClass

A NodeClass describes how EC2 capacity should be configured.

Conceptually:

```yaml
apiVersion: eks.amazonaws.com/v1
kind: EC2NodeClass
metadata:
  name: application
spec:
  role: EcommerceKarpenterNodeRole
```

> **Important:** The exact API group/version and fields depend on the Karpenter release. Use the CRD/schema installed in your cluster rather than copying an old example.

It may define things such as:

```text
IAM role
AMI selection
subnets
security groups
block devices
metadata options
tags
```

---

# 30. IAM for Karpenter

Karpenter needs permission to provision EC2 resources.

Architecture:

```text
Karpenter Controller
       │
       ▼
AWS IAM
       │
       ▼
EC2
       │
       ▼
New Node
```

Do not give:

```text
AdministratorAccess
```

just because provisioning is failing.

Instead:

```text
Karpenter IAM role
       │
       └── required EC2 provisioning permissions
```

with least privilege based on the currently supported Karpenter installation model.

---

# 31. Node IAM Role

Separate the Karpenter controller identity from the identity assumed by the node.

```text
Karpenter Controller
        │
        ▼
Controller IAM Role
        │
        ▼
AWS APIs


EC2 Node
        │
        ▼
Node IAM Role
        │
        ▼
Required AWS APIs
```

Don't confuse the two.

---

# 32. Pod Identity Still Applies

Our application Pods should not inherit broad node permissions.

Instead:

```text
Order Pod
    │
    ▼
ServiceAccount
    │
    ▼
EKS Pod Identity
    │
    ▼
Order IAM Role
    │
    ├── SQS
    ├── Secrets Manager
    └── S3
```

The node role should not become the application security boundary.

---

# 33. Karpenter Node Discovery

Karpenter needs to know which:

```text
subnets
security groups
```

it may use.

This is commonly achieved using tags/discovery mechanisms supported by the selected Karpenter version.

Conceptually:

```text
AWS VPC
 │
 ├── subnet A
 ├── subnet B
 └── subnet C
       │
       ▼
Karpenter discovers eligible resources
```

---

# 34. Why Private Subnets?

Our application nodes should generally be in private subnets:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Private EKS Nodes
```

Karpenter therefore provisions nodes into approved private subnets.

---

# 35. Node Security Groups

Nodes require appropriate network access for:

```text
EKS control plane
Pod networking
AWS services
internal dependencies
```

Don't simply allow:

```text
0.0.0.0/0
```

for node-to-everything traffic.

Follow the EKS networking architecture and least privilege.

---

# 36. Node Bootstrap

When Karpenter launches an instance:

```text
EC2
 │
 ▼
Bootstrap
 │
 ▼
Kubernetes node configuration
 │
 ▼
EKS API
 │
 ▼
Node registers
```

Then:

```bash
kubectl get nodes
```

shows the new node.

---

# 37. Node Lifecycle

A Karpenter-managed node goes through:

```text
Provision
   ↓
Register
   ↓
Schedule workloads
   ↓
Run
   ↓
Consolidation/Disruption decision
   ↓
Cordon
   ↓
Drain/Evict
   ↓
Terminate
```

This is why PodDisruptionBudgets matter.

---

# 38. Consolidation

One of Karpenter's useful capabilities is reducing inefficient capacity.

Suppose:

```text
Node A
 └── 10% utilized

Node B
 └── 15% utilized
```

Karpenter may identify opportunities to consolidate workloads onto fewer/more appropriate nodes, subject to its configured disruption policies and workload constraints.

Conceptually:

```text
Node A ─┐
        ├──→ fewer nodes
Node B ─┘
```

---

# 39. Consolidation Is Not "Delete Empty Nodes"

It is more nuanced.

Karpenter considers:

```text
Pod requirements
Node requirements
Scheduling constraints
Disruption settings
Capacity availability
Pod disruption budgets
```

The goal is efficient capacity while respecting workload constraints.

---

# 40. Disruption

Node disruption can affect workloads.

Therefore we need:

```text
PDB
Graceful shutdown
Readiness probes
Topology spread
Multiple replicas
```

For example:

```text
3 replicas
PDB minAvailable=2
```

helps protect availability during voluntary disruption.

---

# 41. Karpenter + PDB

Suppose:

```text
3 Pods
PDB:
minAvailable = 2
```

Karpenter wants to replace a node containing:

```text
Pod 1
```

Eviction:

```text
3 → 2
```

is acceptable.

But if the node contains:

```text
Pod 1
Pod 2
```

and both cannot be evicted safely:

```text
3 → 1
```

would violate the PDB.

The disruption may be blocked.

---

# 42. Karpenter + Topology

If our service uses:

```yaml
topologySpreadConstraints:
```

Karpenter must provision capacity that allows those constraints to be satisfied.

Example:

```text
AZ-A
  Pod 1

AZ-B
  Pod 2

AZ-C
  Pod 3
```

If one AZ has no eligible capacity, scheduling constraints can cause Pending Pods.

---

# 43. Karpenter Troubleshooting — Step 1

Pod is Pending.

Start:

```bash
kubectl describe pod <pod-name> \
  -n ecommerce-prod
```

Look at:

```text
Events:
  FailedScheduling
```

Don't start by restarting Karpenter.

Find the scheduling reason first.

---

# 44. Karpenter Troubleshooting — Step 2

Check nodes:

```bash
kubectl get nodes -o wide
```

Then:

```bash
kubectl top nodes
```

Determine:

```text
Current capacity
Current utilization
```

---

# 45. Karpenter Troubleshooting — Step 3

Check Karpenter resources:

```bash
kubectl get nodepool
```

Depending on the installed version, also inspect the relevant NodeClass resources.

For example:

```bash
kubectl get ec2nodeclass
```

---

# 46. Karpenter Troubleshooting — Step 4

Check Karpenter controller:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter
```

The exact labels depend on the chart/version.

Then:

```bash
kubectl logs \
  -n kube-system \
  <karpenter-pod>
```

---

# 47. Common Karpenter Failure

### NodePool doesn't match Pod

Example:

```text
Pod:
arm64

NodePool:
amd64 only
```

Result:

```text
Pending
```

---

# 48. Another Failure

Pod requests:

```text
8 CPU
```

NodePool only allows:

```text
small instance types
```

Result:

```text
No compatible instance
```

Fix:

```text
Expand allowed instance types
```

subject to cost and workload requirements.

---

# 49. Another Failure

NodePool:

```text
capacity-type=spot
```

but no suitable Spot capacity is available.

Possible strategy:

```text
Allow on-demand fallback
```

if business requirements permit it.

---

# 50. Another Failure

Node provisioning fails because of IAM.

Symptoms may include:

```text
Node never appears
```

or provisioning/controller errors.

Investigate:

```text
Karpenter controller IAM
Node IAM role
EC2 permissions
Instance profile / identity mechanism
```

Do not blindly add AdministratorAccess.

---

# 51. Another Failure

Subnet IP exhaustion.

```text
VPC
 │
 ▼
Private subnet
 │
 ▼
No available IPs
```

Then:

```text
Node provisioning or Pod networking
        ↓
failure
```

Check:

```text
subnet free IPs
VPC CNI
ENI limits
instance type
subnet sizing
```

---

# 52. Another Failure — Security Group Discovery

Karpenter may not be able to launch nodes with the required network configuration if its discovery/configuration is incorrect.

Check:

```text
subnet selection
security-group selection
tags/discovery
IAM
```

---

# 53. Another Failure — Architecture

```text
Pod:
linux/amd64

Available capacity:
linux/arm64
```

If the image isn't multi-architecture:

```text
Provisioning may succeed
but application startup can fail
```

Always distinguish:

```text
Scheduling problem
```

from:

```text
Container runtime problem
```

---

# 54. Another Failure — PVC

Suppose a workload requires:

```text
PVC
```

and a new node is created.

The node may exist, but the volume may not be attachable in the required AZ/configuration.

Architecture:

```text
Pod
 │
 ▼
PVC
 │
 ▼
EBS
 │
 ▼
AZ
```

Storage topology matters.

---

# 55. Stateful Workloads

Be especially careful with:

```text
PostgreSQL
Elasticsearch
Kafka
RabbitMQ
```

Don't treat stateful workloads like stateless APIs.

Node replacement must consider:

```text
storage
replication
quorum
AZ
PDB
startup time
data recovery
```

---

# 56. Karpenter and Stateful Workloads

Karpenter can provide compute capacity for stateful workloads, but storage and scheduling constraints become much more important.

For example:

```text
Kafka broker
   │
   ▼
EBS volume
   │
   ▼
AZ-A
```

Moving the Pod to another AZ may not be equivalent to moving a stateless HTTP Pod.

Design stateful workloads separately.

---

# 57. Managed Node Groups + Karpenter Together

We don't need to choose one universally.

A practical architecture:

```text
EKS
 │
 ├── Managed Node Group
 │      │
 │      └── Baseline/system capacity
 │
 └── Karpenter
        │
        ├── Dynamic application capacity
        ├── On-Demand
        └── Spot where appropriate
```

This is often easier to operate than trying to make every workload dynamic from day one.

---

# 58. Example Capacity Architecture

```text
                    EKS
                     │
          ┌──────────┴───────────┐
          │                      │
       Baseline               Dynamic
          │                      │
   Managed Node Group         Karpenter
          │                      │
    ┌─────┼─────┐          ┌─────┼─────┐
    │     │     │          │     │     │
   AZ-A  AZ-B  AZ-C       OD    Spot   OD
```

---

# 59. When to Use Managed Node Groups

Good for:

```text
Predictable baseline capacity
System workloads
Stable workloads
Simple environments
Organizations wanting simpler node lifecycle
```

---

# 60. When Karpenter Helps

Useful for:

```text
Variable workloads
Large number of instance options
Fast scaling
Mixed instance types
On-Demand/Spot flexibility
Bin-packing efficiency
Dynamic capacity
```

---

# 61. Don't Over-Engineer Dev

For our learning project:

```text
DEV
 │
 └── Managed Node Group
```

is perfectly reasonable.

Then implement Karpenter once the application workload is stable.

Production:

```text
PROD
 │
 ├── Baseline managed capacity
 └── Karpenter dynamic capacity
```

can be evaluated.

---

# 62. Capacity Type Strategy

Example:

```text
critical-api
    ↓
On-Demand

async-worker
    ↓
Spot + On-Demand fallback

batch-job
    ↓
Spot
```

The application owner should help determine interruption tolerance.

---

# 63. Node Lifecycle During Deployment

Suppose we deploy a new version.

```text
Deployment
replicas = 10
```

Rolling update:

```text
10 old
  ↓
11 total
  ↓
replace old
```

That temporary surge may require additional node capacity.

Karpenter can respond if configured appropriately.

Thus:

```text
Deployment strategy
        +
Resource requests
        +
Autoscaler
```

must be planned together.

---

# 64. Node Lifecycle During HPA

Traffic spike:

```text
100 requests/s
      ↓
500 requests/s
```

HPA:

```text
3 Pods
 ↓
10 Pods
```

Capacity:

```text
Current nodes
 ↓
insufficient
```

Karpenter:

```text
Provision capacity
```

Final:

```text
3 → 10 Pods
+
new nodes
```

---

# 65. Scale Down

Traffic drops:

```text
500 req/s
 ↓
100 req/s
```

HPA:

```text
10 Pods
 ↓
3 Pods
```

Now some nodes become underutilized.

Karpenter's disruption/consolidation logic may identify capacity that can be removed, subject to configuration and safety constraints.

```text
Underutilized nodes
        ↓
Consolidation
        ↓
Drain
        ↓
Terminate
```

---

# 66. Cost Optimization

Node capacity contributes significantly to EKS cost.

We should monitor:

```text
Instance utilization
CPU requests
Memory requests
Actual usage
Spot percentage
Idle nodes
Overprovisioning
```

Example:

```text
Node:
8 CPU

Requested:
2 CPU

Actual:
1 CPU
```

Potential inefficiency.

But don't optimize purely for utilization.

You must preserve:

```text
availability
latency
failure headroom
deployment capacity
```

---

# 67. Requests vs Actual Usage

Suppose:

```text
CPU request = 2 CPU
actual = 0.2 CPU
```

The scheduler reserves 2 CPU worth of placement capacity.

That can lead to:

```text
Poor bin packing
More nodes
Higher cost
```

So resource requests should be based on measured workload behavior.

---

# 68. Karpenter and Bin Packing

Imagine:

```text
Node A
CPU: 8
requested: 3

Node B
CPU: 8
requested: 3
```

Total:

```text
6 CPU requested
```

But there may be enough room to consolidate workloads onto fewer nodes if constraints allow.

Better resource sizing can improve packing.

---

# 69. But Don't Under-Request

If:

```text
request = 100m
actual = 1 CPU
```

you may pack too many workloads onto nodes.

Then:

```text
CPU contention
    ↓
Latency
    ↓
Errors
```

Therefore:

> Resource requests are a reliability setting, not merely a cost setting.

---

# 70. Node Monitoring

Prometheus should monitor:

```text
CPU
Memory
Filesystem
Network
Node conditions
Pod density
```

Node Exporter gives host-level metrics.

kube-state-metrics gives Kubernetes object state.

Together:

```text
Node Exporter
    +
kube-state-metrics
    +
Prometheus
```

give us a useful cluster view.

---

# 71. Karpenter Monitoring

Monitor:

```text
Provisioning latency
Provisioning failures
Node count
Node termination
Consolidation
Capacity type
Pending Pods
```

The exact Karpenter metrics depend on the installed version.

The important operational question is:

> **Is the autoscaling system itself healthy?**

---

# 72. Alerts for Capacity

Useful alerts:

```text
Too many Pending Pods
Node provisioning failures
NodeNotReady
High node CPU
High node memory
Low free disk
Subnet IP exhaustion
Karpenter controller errors
Excessive node churn
```

---

# 73. Node Churn

Suppose Karpenter repeatedly does:

```text
Create node
   ↓
Terminate node
   ↓
Create node
   ↓
Terminate node
```

This is undesirable.

Possible causes:

```text
Bad consolidation configuration
Resource request changes
Workload instability
PDB constraints
Rapid HPA changes
Incorrect limits
```

Monitor node churn.

---

# 74. HPA Flapping

Suppose:

```text
3 Pods
 ↓
10
 ↓
3
 ↓
10
 ↓
3
```

This is scaling oscillation.

Potential causes:

```text
unstable metrics
bad target
short stabilization window
insufficient workload capacity
```

HPA configuration and Karpenter behavior should be tuned together.

---

# 75. Production Karpenter Rollout Strategy

Don't install Karpenter and immediately move every service onto it.

Use:

```text
DEV
 ↓
Test
 ↓
QA
 ↓
STAGING
 ↓
One production workload
 ↓
Observe
 ↓
Expand gradually
```

---

# 76. First Production Workload

Choose a suitable stateless service.

For example:

```text
search-service
```

or:

```text
notification-service
```

depending on workload behavior.

Avoid starting with a critical stateful system.

---

# 77. Karpenter Migration

Existing:

```text
Managed Node Group
    │
    └── all application workloads
```

Target:

```text
Managed Node Group
    │
    └── baseline/system

Karpenter
    │
    └── application workloads
```

Migration can use labels, taints, affinities, and scheduling constraints.

---

# 78. Production Migration Safety

Before moving workloads:

```text
[ ] Karpenter controller healthy
[ ] IAM verified
[ ] NodeClass verified
[ ] NodePool verified
[ ] Subnets correct
[ ] Security groups correct
[ ] AMI correct
[ ] Architecture correct
[ ] Monitoring configured
[ ] PDB configured
[ ] Topology constraints tested
[ ] Rollback plan
```

---

# 79. Rollback

If Karpenter causes problems:

```text
Workload
   │
   ▼
Scheduling issue
   │
   ▼
Move workload back to
managed node group
```

This is why we don't immediately remove the baseline node group.

---

# 80. Production Architecture

A mature version of our platform could look like:

```text
                         EKS
                          │
              ┌───────────┴────────────┐
              │                        │
       Managed Node Groups          Karpenter
              │                        │
       ┌──────┴──────┐          ┌──────┼───────┐
       │             │          │      │       │
    System        Baseline     OD    Spot    OD
       │             │          │      │       │
       └─────────────┼──────────┴──────┴───────┘
                     │
                     ▼
                  Workloads
                     │
         ┌───────────┼────────────┐
         │           │            │
      APIs        Workers       Jobs
         │           │            │
         └───────────┼────────────┘
                     │
                 Autoscaling
                     │
              ┌──────┴──────┐
              │             │
             HPA        Karpenter
              │             │
              └──────┬──────┘
                     ▼
                   EC2
```

---

# 81. Troubleshooting Decision Tree

When a Pod is Pending:

```text
                  POD PENDING
                       │
                       ▼
                describe pod
                       │
             ┌─────────┼──────────┐
             │         │          │
          Resources  Taints    Affinity
             │         │          │
             └─────────┼──────────┘
                       │
                       ▼
               Check existing nodes
                       │
                       ▼
             Does Karpenter need
             to provision capacity?
                       │
                 ┌─────┴─────┐
                 │           │
                YES          NO
                 │           │
                 ▼           ▼
            Check NodePool  Scheduler
                 │          constraints
                 ▼
            Check NodeClass
                 │
                 ▼
             Check IAM
                 │
                 ▼
             Check subnet
                 │
                 ▼
           Check security group
                 │
                 ▼
         Check EC2 availability
```

---

# 82. Golden Commands

### NodePools

```bash
kubectl get nodepool
```

### NodeClasses

```bash
kubectl get ec2nodeclass
```

### Nodes

```bash
kubectl get nodes -o wide
```

### Pods

```bash
kubectl get pods -A -o wide
```

### Pending Pods

```bash
kubectl get pods -A \
  --field-selector=status.phase=Pending
```

### Pod events

```bash
kubectl describe pod <pod> -n <namespace>
```

### Node events

```bash
kubectl describe node <node>
```

### Resource usage

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

---

# 83. Example Production Incident

### Situation

Order traffic increases from:

```text
500 req/s
```

to:

```text
5,000 req/s
```

---

### Step 1 — HPA

```text
3 Pods
 ↓
15 Pods
```

---

### Step 2 — Scheduler

Existing nodes cannot fit all Pods.

```text
5 Pods Pending
```

---

### Step 3 — Karpenter

Karpenter sees:

```text
CPU requirement
Memory requirement
AZ requirements
Architecture
Capacity type
```

---

### Step 4 — EC2

Karpenter launches:

```text
new capacity
```

---

### Step 5 — Node Registration

```text
EC2
 ↓
EKS
 ↓
Ready
```

---

### Step 6 — Scheduler

```text
Pending Pods
 ↓
New node
 ↓
Running
```

---

### Step 7 — Application

```text
ALB
 ↓
Service
 ↓
Pods
```

Traffic is distributed.

---

# 84. Now Traffic Drops

```text
5,000 req/s
 ↓
500 req/s
```

HPA:

```text
15
 ↓
3
```

Karpenter observes excess capacity.

If consolidation/disruption policy permits:

```text
underutilized node
 ↓
drain
 ↓
terminate
```

Cost falls.

---

# 85. What Can Break This?

Several things:

```text
Bad resource requests
Bad PDB
Bad topology constraints
Bad NodePool requirements
No compatible EC2 capacity
Subnet exhaustion
IAM failure
Security group failure
Architecture mismatch
PVC topology
Spot interruption
Application startup failure
```

That's why autoscaling isn't simply:

```text
install Karpenter
```

It is an entire scheduling architecture.

---

# 86. Production Design Recommendation for Our Project

For our Enterprise E-Commerce Platform, a practical progression is:

### DEV

```text
Managed Node Group
```

### QA

```text
Managed Node Group
```

### STAGING

```text
Managed Node Group
+
optional Karpenter testing
```

### PROD

```text
Managed baseline/system capacity
+
Karpenter for dynamic application capacity
```

This allows us to learn and validate Karpenter without making the entire platform dependent on it immediately.

---

# 87. EKS Compute Checklist

## Managed Node Groups

* [ ] Correct instance types
* [ ] Private subnets
* [ ] Multi-AZ
* [ ] System capacity
* [ ] Baseline capacity
* [ ] Labels
* [ ] Taints
* [ ] Update strategy
* [ ] AMI lifecycle

## Karpenter

* [ ] Controller installed
* [ ] IAM configured
* [ ] NodePool configured
* [ ] NodeClass configured
* [ ] Subnet discovery
* [ ] Security-group discovery
* [ ] Instance types
* [ ] Architecture
* [ ] On-Demand/Spot
* [ ] Limits
* [ ] Disruption policy
* [ ] Monitoring

## Workloads

* [ ] Requests
* [ ] Limits
* [ ] HPA
* [ ] PDB
* [ ] Topology spread
* [ ] Anti-affinity
* [ ] Graceful shutdown
* [ ] Readiness probes

---

# 88. Final Mental Model

The most important chain to remember is:

```text
                   TRAFFIC
                      │
                      ▼
                     HPA
                      │
                      ▼
                 MORE PODS
                      │
                      ▼
                 K8s Scheduler
                      │
             ┌────────┴────────┐
             │                 │
        Capacity exists    Capacity missing
             │                 │
             ▼                 ▼
          Schedule          Karpenter
                               │
                               ▼
                           EC2 capacity
                               │
                               ▼
                           EKS Node
                               │
                               ▼
                          K8s Scheduler
                               │
                               ▼
                              Pod
                               │
                               ▼
                         Application
```

And when traffic decreases:

```text
Traffic ↓
   │
   ▼
HPA ↓
   │
   ▼
Pods ↓
   │
   ▼
Unused capacity
   │
   ▼
Karpenter consolidation
   │
   ▼
Node removal
```

---

# 89. The Production Principle

Don't think:

> **"Karpenter creates EC2 instances."**

Think:

> **"Karpenter is part of the Kubernetes scheduling and capacity-management system."**

It sits between:

```text
Kubernetes workload demand
```

and:

```text
AWS compute capacity
```

while Kubernetes scheduling constraints, IAM, networking, storage, PDBs, topology, and application resource definitions all influence the result.

---

# 90. Chapter 95 Completion

We now understand the complete EKS compute model:

```text
Managed Node Groups
        +
Karpenter
        +
HPA
        +
Kubernetes Scheduler
        +
Resource Requests
        +
PDB
        +
Topology
        +
AWS EC2
```

The next chapter should go one level deeper into **EKS upgrades and lifecycle management**, because once you operate production clusters, one of the most important recurring responsibilities is keeping the Kubernetes control plane, node AMIs, and EKS add-ons supported without causing downtime.

### Next file

```text
96-EKS-Upgrades-and-Cluster-Lifecycle.md
```

It will cover:

```text
EKS version lifecycle
        ↓
API deprecations
        ↓
Control-plane upgrade
        ↓
Managed node-group upgrade
        ↓
Karpenter node AMI rotation
        ↓
VPC CNI
CoreDNS
kube-proxy
EBS CSI
Pod Identity
        ↓
Application compatibility
        ↓
Staging validation
        ↓
Jira Change
        ↓
Production upgrade
        ↓
Post-upgrade verification
        ↓
Rollback / recovery
```
