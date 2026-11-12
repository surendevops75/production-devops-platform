# 104 — Kubernetes Cluster Autoscaling

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Region:** `ap-south-1`
> **Scope:** Cluster-level capacity management using EKS Managed Node Groups, Karpenter, and Cluster Autoscaler

---

## 1. Purpose

We have already implemented workload-level autoscaling:

```text
100 → Spot Instances
101 → Multi-AZ Workload Placement
102 → Kubernetes Resource Management
103 → HPA / VPA / KEDA
```

Now we need to answer a different question:

> **Where do the additional Pods run when the existing EKS nodes do not have enough capacity?**

The complete scaling chain is:

```text
Traffic
   ↓
HPA / KEDA
   ↓
More Pods
   ↓
Scheduler
   ↓
No capacity
   ↓
Cluster capacity controller
   ↓
New EC2 node
   ↓
Node joins EKS
   ↓
Scheduler places Pod
```

For our architecture, **Karpenter is the primary dynamic node provisioning mechanism**.

Managed Node Groups provide the stable baseline capacity.

---

# 2. Pod Scaling vs Node Scaling

This distinction must be clear.

### HPA

Answers:

> How many application Pods do I need?

```text
3 Pods → 8 Pods
```

### KEDA

Answers:

> How many worker Pods do I need based on an external event source?

```text
SQS backlog
   ↓
2 Pods → 15 Pods
```

### Karpenter

Answers:

> What EC2 capacity is needed to run the unscheduled Pods?

```text
Pending Pods
   ↓
EC2 node
```

### Cluster Autoscaler

Answers:

> Which configured node group should grow or shrink?

```text
Node Group
3 nodes → 5 nodes
```

So:

```text
HPA/KEDA
    ↓
Pods

Karpenter / Cluster Autoscaler
    ↓
Nodes
```

---

# 3. Our Production Capacity Architecture

We will use a mixed strategy:

```text
                    EKS Cluster
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
   Baseline Capacity             Dynamic Capacity
          │                             │
          ▼                             ▼
 Managed Node Groups                Karpenter
          │                             │
          ├── System                  ├── Spot
          ├── Platform                └── On-Demand
          └── Critical
```

This gives us:

```text
stable baseline
+
dynamic scaling
+
Spot optimization
+
On-Demand fallback
+
Multi-AZ capacity
```

---

# 4. Why We Need Baseline Nodes

We should not make every workload depend on Karpenter.

Some components need to exist before dynamic application capacity is available.

Examples:

```text
CoreDNS
VPC CNI
kube-proxy
EBS CSI
AWS Load Balancer Controller
ArgoCD
monitoring components
logging agents
Karpenter itself
```

Therefore we maintain a baseline system capacity.

Example:

```text
System Node Group
    ↓
3 nodes
    ↓
AZ-A
AZ-B
AZ-C
```

---

# 5. Node Pool Design

Our cluster can have:

```text
System Node Group
Application On-Demand
Karpenter Spot
Karpenter On-Demand
```

Conceptually:

```text
┌───────────────────────────────┐
│ EKS Cluster                   │
│                               │
│  System Nodes                 │
│  ├── CoreDNS                  │
│  ├── Karpenter                │
│  └── platform components      │
│                               │
│  Karpenter                    │
│  ├── Spot application nodes   │
│  └── On-Demand application    │
└───────────────────────────────┘
```

---

# 6. Why Not Only Karpenter?

Karpenter itself requires some Kubernetes infrastructure to run.

If:

```text
Karpenter Pod
```

is waiting for:

```text
Karpenter to create a node
```

we have a bootstrap dependency problem.

Therefore Karpenter should run on stable capacity.

Typical architecture:

```text
Managed Node Group
        ↓
Karpenter Pod
        ↓
Karpenter provisions
        ↓
Application nodes
```

---

# 7. Why Not Only Managed Node Groups?

Managed Node Groups work well, but they generally require you to define:

```text
instance type
min size
desired size
max size
```

and Cluster Autoscaler grows or shrinks those configured groups.

Karpenter can dynamically select suitable EC2 capacity based on Pod requirements and NodePool constraints.

This gives us greater flexibility for:

```text
instance types
Spot pools
architecture
AZ
capacity type
workload requirements
```

---

# 8. Cluster Autoscaler vs Karpenter

| Capability                 | Cluster Autoscaler   | Karpenter                 |
| -------------------------- | -------------------- | ------------------------- |
| Scales nodes               | Yes                  | Yes                       |
| Uses node groups           | Yes                  | Not required              |
| Dynamic instance selection | More limited         | Strong                    |
| Spot diversification       | Possible             | Strong                    |
| Pod-driven provisioning    | Yes                  | Yes                       |
| Node consolidation         | Group-based          | Dynamic                   |
| Best use                   | Existing node groups | Flexible dynamic capacity |

The exact capabilities depend on the installed versions and AWS/Kubernetes architecture.

---

# 9. Recommended Architecture for This Project

For our enterprise e-commerce platform:

```text
Managed Node Groups
        │
        ├── System
        └── Critical baseline

Karpenter
        │
        ├── Spot application capacity
        └── On-Demand application capacity
```

We are therefore not using:

```text
Cluster Autoscaler + Karpenter
```

to independently manage the same capacity pool.

Avoid overlapping controllers managing the same nodes unless there is a deliberate, documented architecture.

---

# 10. Node Lifecycle

A typical scale-up looks like:

```text
HPA
 ↓
Replica count increases
 ↓
Pod created
 ↓
Scheduler attempts placement
 ↓
Pod Pending
 ↓
Karpenter detects unschedulable Pod
 ↓
Karpenter evaluates requirements
 ↓
EC2 instance selected
 ↓
EC2 starts
 ↓
EKS node joins
 ↓
Node becomes Ready
 ↓
Scheduler places Pod
 ↓
Application becomes Ready
```

---

# 11. Example

Current state:

```text
3 nodes
```

Each:

```text
4 CPU
16Gi memory
```

Available capacity:

```text
CPU: 1 CPU
Memory: 3Gi
```

HPA scales:

```text
3 Pods → 8 Pods
```

New Pods require:

```text
CPU: 2 CPU each
Memory: 2Gi each
```

Existing nodes cannot fit them.

Therefore:

```text
Pending Pods
    ↓
Karpenter
    ↓
New EC2 capacity
```

---

# 12. Scheduler Comes First

An important detail:

Karpenter does not simply watch CPU utilization.

It reacts to unschedulable Pods and their scheduling requirements.

Example Pod:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
```

Scheduler says:

```text
No existing node can satisfy this request.
```

That creates the condition for Karpenter to provision suitable capacity.

---

# 13. Why Requests Matter Again

This connects directly to Chapter 102.

Suppose actual usage is:

```text
CPU = 200m
```

but request is:

```text
CPU = 2
```

Kubernetes schedules based on:

```text
2 CPU
```

not:

```text
200m
```

Therefore Karpenter may provision larger or additional nodes unnecessarily.

This is another reason accurate resource requests are critical.

---

# 14. Karpenter NodePool

A conceptual NodePool for application Spot capacity:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: application-spot
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64

        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - spot

        - key: karpenter.k8s.aws/instance-category
          operator: In
          values:
            - m
            - c
            - r

        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values:
            - "4"

  limits:
    cpu: "100"
```

**Important:** Karpenter APIs and requirement keys can change between versions. Pin the Karpenter version and validate the manifest against the installed CRDs before applying it.

---

# 15. Why Multiple Instance Categories?

We don't want:

```text
Only m6i.large
```

as the entire Spot strategy.

Instead:

```text
m
c
r
```

gives Karpenter more options.

This improves the ability to find available capacity.

---

# 16. Why Multiple Instance Types?

Suppose our application accepts:

```text
m6i.large
m6a.large
m7i.large
m7a.large
c6i.large
c7i.large
```

If one Spot pool has no capacity:

```text
m6i.large / AZ-A
```

Karpenter can potentially select another compatible pool.

This is called **capacity diversification**.

---

# 17. Spot Capacity Is a Pool

A Spot capacity pool is effectively tied to:

```text
instance type
+
Availability Zone
```

For example:

```text
m6i.large / ap-south-1a
m6i.large / ap-south-1b
m6a.large / ap-south-1a
c6i.large / ap-south-1c
```

These are different capacity opportunities.

Therefore:

```text
more compatible pools
        ↓
more placement options
        ↓
better Spot resilience
```

---

# 18. On-Demand NodePool

For workloads that require stronger capacity assurance:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: application-on-demand
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64

        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - on-demand

        - key: karpenter.k8s.aws/instance-category
          operator: In
          values:
            - m
            - c
            - r

  limits:
    cpu: "100"
```

Again, validate against the installed Karpenter version.

---

# 19. Spot + On-Demand Strategy

Our application capacity can be:

```text
                   Application
                       │
              ┌────────┴────────┐
              │                 │
           Spot              On-Demand
              │                 │
       cost optimized       critical/fallback
```

We do not necessarily hard-code:

```text
70% Spot
30% On-Demand
```

for every workload.

The actual split depends on:

```text
workload criticality
interruption tolerance
capacity availability
cost objectives
business requirements
```

---

# 20. Critical Workloads

Examples that may need On-Demand capacity:

```text
payment-service
critical platform components
stateful workloads
highly sensitive transaction processors
```

Even then, the final choice should be based on application resilience and business requirements.

---

# 21. Stateless Workloads

Good candidates for Spot generally have:

```text
multiple replicas
stateless architecture
graceful shutdown
retry handling
PDB
Multi-AZ placement
```

Examples:

```text
catalog-service
search-service
notification-service
recommendation-service
```

provided their specific workload characteristics support interruption.

---

# 22. Stateful Workloads

Avoid treating:

```text
PostgreSQL
Kafka
RabbitMQ
Redis
```

as ordinary stateless Spot workloads.

Our architecture already prefers managed services where appropriate:

```text
RDS
ElastiCache
Amazon MSK / Amazon MQ where selected
```

When stateful components are self-managed, their storage and failure model must be explicitly designed.

---

# 23. Node Labels

We can classify nodes.

Example:

```text
workload=system
workload=application
capacity=spot
capacity=on-demand
```

Then Pods can request suitable nodes.

---

# 24. Taints and Tolerations

For example, we may taint specialized system nodes:

```text
workload=system:NoSchedule
```

Only Pods with the matching toleration can run there.

This prevents application Pods from consuming capacity intended for critical platform components.

---

# 25. Node Affinity

Application Pods can use:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: workload
              operator: In
              values:
                - application
```

This tells the scheduler:

> Prefer/require application nodes according to the specified affinity rule.

---

# 26. Topology Still Matters

Even if Karpenter creates nodes, we still need:

```text
topology.kubernetes.io/zone
```

constraints.

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

Karpenter provides capacity.

The scheduler decides where the Pods go.

---

# 27. Karpenter Does Not Replace the Scheduler

This distinction is important.

```text
Kubernetes Scheduler
    ↓
decides where Pod runs

Karpenter
    ↓
provides capacity when existing capacity is insufficient
```

Karpenter doesn't replace Kubernetes scheduling.

---

# 28. EC2NodeClass

Karpenter also needs to know how to launch EC2 instances.

Conceptually:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: application
spec:
  amiFamily: AL2023

  role: EcommerceKarpenterNodeRole

  subnetSelectorTerms:
    - tags:
        kubernetes.io/cluster/ecommerce-prod: owned

  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/ecommerce-prod: owned

  tags:
    Project: ecommerce
    Environment: prod
```

Exact API fields and discovery behavior are version-sensitive.

---

# 29. Node IAM Role

The node needs AWS permissions required for EKS node operation.

This is different from the application Pod's IAM role.

Architecture:

```text
EC2 Node
    ↓
Node IAM Role

Application Pod
    ↓
ServiceAccount
    ↓
EKS Pod Identity
    ↓
Application IAM Role
```

Do not give application Pods the node role.

---

# 30. Why IAM Separation Matters

Suppose:

```text
order-service
```

needs:

```text
SQS SendMessage
```

It should receive:

```text
OrderService IAM Role
```

not:

```text
Node IAM Role
```

Otherwise any workload capable of accessing the node role could potentially obtain broader permissions.

Least privilege remains essential.

---

# 31. Karpenter Limits

We should protect the cluster from runaway provisioning.

Example:

```yaml
limits:
  cpu: "100"
```

This establishes a boundary for the NodePool.

We should also consider:

```text
AWS service quotas
EC2 limits
subnet IP availability
budget
business traffic
```

---

# 32. Why Cost Guardrails Matter

Imagine:

```text
Traffic spike
   ↓
HPA
   ↓
100 Pods
   ↓
Karpenter
   ↓
20 EC2 nodes
```

If something is wrong:

```text
bad HPA
bot traffic
application retry storm
metric failure
```

capacity can grow rapidly.

Therefore autoscaling needs:

```text
max replicas
+
NodePool limits
+
rate limiting
+
alerts
+
budgets
```

---

# 33. Node Consolidation

Karpenter can identify underutilized capacity and consolidate workloads where appropriate.

Example:

```text
Node A:
20% utilization

Node B:
25% utilization
```

If Pods can safely move:

```text
Node A + Node B
        ↓
Node C
```

one node may be removed.

This can reduce cost.

But consolidation must respect:

```text
PDB
topology
affinity
anti-affinity
taints
Pod disruption constraints
```

---

# 34. Consolidation Risk

Suppose:

```text
3 replicas
```

are deliberately spread across:

```text
AZ-A
AZ-B
AZ-C
```

A naive consolidation strategy must not collapse them into one failure domain.

Therefore:

```text
cost optimization
```

must not defeat:

```text
availability architecture
```

---

# 35. Karpenter + PDB

Suppose:

```yaml
minAvailable: 2
```

for:

```text
3 replicas
```

Karpenter/node disruption must respect voluntary disruption constraints where supported.

But remember:

```text
PDB != protection against sudden node/AZ failure
```

It mainly controls voluntary disruptions.

---

# 36. Karpenter + Spot Interruption

Our previous Spot architecture was:

```text
AWS Spot interruption
        ↓
EventBridge
        ↓
SQS interruption queue
        ↓
Karpenter
        ↓
Node termination handling
        ↓
Pod disruption
        ↓
Replacement capacity
```

Therefore:

```text
Spot
+
Karpenter
+
PDB
+
Multi-AZ
+
multiple instance pools
```

form one resilience design.

---

# 37. What Happens When Spot Capacity Is Unavailable?

Suppose:

```text
Spot:
unavailable
```

Karpenter has multiple options depending on configured NodePools and workload requirements.

For example:

```text
Spot Pool A
   ↓
unavailable

Spot Pool B
   ↓
unavailable

On-Demand Pool
   ↓
available
```

A workload designed to tolerate the fallback can continue using On-Demand capacity.

The exact behavior depends on NodePool weighting/requirements and scheduling configuration.

---

# 38. Capacity Type Separation

We should explicitly classify workloads.

Example:

```text
application-spot
application-on-demand
system-on-demand
```

Then:

```text
system
   ↓
stable baseline

application
   ↓
dynamic Karpenter capacity
```

---

# 39. Node Group Baseline

Example Terraform concept:

```hcl
resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "system"
  node_role_arn   = aws_iam_role.eks_node.arn

  subnet_ids = var.private_subnet_ids

  scaling_config {
    desired_size = 3
    min_size     = 3
    max_size     = 6
  }

  instance_types = ["m6i.large"]

  labels = {
    workload = "system"
  }
}
```

Exact instance types should be selected based on current AWS availability, pricing, architecture, and workload requirements.

---

# 40. System Node Taint

A system node group can optionally use a taint:

```text
workload=system:NoSchedule
```

Then platform workloads explicitly tolerate it.

Example:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: system
    effect: NoSchedule
```

This is one way to keep system capacity isolated.

---

# 41. Don't Put Everything on System Nodes

We don't want:

```text
ArgoCD
+
Prometheus
+
Grafana
+
ELK
+
21 application services
```

all competing for the same tiny node pool.

Instead:

```text
System nodes
    ↓
critical cluster/platform components

Application capacity
    ↓
Karpenter-managed nodes
```

---

# 42. Capacity Planning

We should calculate:

```text
Required capacity =
application requests
+
system overhead
+
deployment surge
+
autoscaling headroom
+
failure headroom
```

For example:

```text
Application requests:
20 CPU

Deployment surge:
3 CPU

AZ/node failure headroom:
5 CPU

System:
3 CPU

Approximate planned capacity:
31 CPU
```

This is an example methodology, not a universal sizing formula.

---

# 43. AZ Failure

Suppose we have:

```text
AZ-A
AZ-B
AZ-C
```

and:

```text
AZ-A fails
```

We lose:

```text
nodes
+
Pods
+
capacity
```

Remaining:

```text
AZ-B
AZ-C
```

must have enough capacity to recover.

Therefore our Karpenter configuration must permit capacity across multiple AZs.

---

# 44. Node Failure

Suppose:

```text
Node-A
```

fails.

Pods are rescheduled if possible.

If there isn't enough existing capacity:

```text
Pending Pods
    ↓
Karpenter
    ↓
replacement node
```

This is why capacity controllers are part of availability.

---

# 45. Node Drain

For controlled maintenance:

```bash
kubectl cordon <node>
```

prevents new Pods from scheduling there.

Then:

```bash
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

**Use drain carefully in production.**

Before draining, inspect:

```text
PDB
replica count
topology
critical workloads
local storage
DaemonSets
```

Never blindly run drain against a production node during an incident.

---

# 46. Node Troubleshooting

First:

```bash
kubectl get nodes
```

Then:

```bash
kubectl describe node <node>
```

Check:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
conditions
taints
allocatable
events
```

Then:

```bash
kubectl top node <node>
```

---

# 47. Check Pending Pods

```bash
kubectl get pods -A \
  --field-selector=status.phase=Pending
```

Then:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
FailedScheduling
```

Examples:

```text
Insufficient cpu
Insufficient memory
node affinity mismatch
untolerated taint
topology constraint
```

---

# 48. Check Karpenter

Depending on the installed Karpenter version and deployment:

```bash
kubectl get pods -n kube-system
```

Find Karpenter:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=karpenter
```

Then inspect logs:

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  --tail=200
```

Exact labels may differ by installation/version.

---

# 49. Check NodeClaims

With modern Karpenter versions, NodeClaims are useful for understanding provisioning.

```bash
kubectl get nodeclaims
```

Then:

```bash
kubectl describe nodeclaim <name>
```

Investigate:

```text
requirements
capacity type
instance selection
launch status
conditions
```

The exact resource names depend on the installed Karpenter CRDs.

---

# 50. Common Provisioning Failure

### Symptom

```text
Pod Pending
```

### Event

```text
Insufficient cpu
```

### Karpenter cannot provision.

Possible reasons:

```text
NodePool requirement too restrictive
no compatible instance types
AZ capacity unavailable
subnet IP exhaustion
IAM problem
security group discovery problem
EC2 quota
NodeClass configuration
```

Do not assume the problem is simply "not enough EC2."

---

# 51. Subnet IP Exhaustion

AWS VPC CNI allocates Pod IP addresses from VPC subnets.

Suppose:

```text
Private subnet:
10.0.11.0/24
```

and the subnet runs out of available addresses.

Karpenter may successfully launch an EC2 instance, but Pods can still have networking problems because there aren't enough Pod IPs.

Check:

```text
subnet available IPs
VPC CNI allocation
prefix delegation configuration
Pod density
```

This connects node scaling to our VPC/IP planning.

---

# 52. EC2 Quotas

Another failure:

```text
Karpenter requests:
new EC2 capacity
```

AWS responds:

```text
quota exceeded
```

Potentially:

```text
instance family quota
vCPU quota
Spot quota
```

Therefore production capacity planning must include AWS service quotas.

---

# 53. Node Bootstrap Failure

Suppose EC2 launches but never becomes:

```text
Ready
```

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl describe node <node>
```

If the node never registers, investigate:

```text
IAM
network
security groups
EKS bootstrap
AMI
user data
DNS
VPC endpoints/NAT
cluster endpoint connectivity
```

This is an infrastructure problem, not an application problem.

---

# 54. System Node Failure

Suppose one system node fails.

We should have:

```text
system node A
system node B
system node C
```

across multiple AZs.

The remaining system capacity should continue supporting:

```text
CoreDNS
Karpenter
platform services
```

This is another reason not to run the entire cluster on a single node.

---

# 55. Karpenter Failure

What if Karpenter itself fails?

Our baseline application nodes may continue running.

Existing Pods:

```text
continue
```

New capacity:

```text
may not provision
```

Therefore Karpenter must be monitored.

We should alert on:

```text
Karpenter unavailable
Node provisioning failures
Pending Pods
Node provisioning latency
```

---

# 56. HPA + Karpenter Failure Scenario

Production:

```text
Traffic increases
```

HPA:

```text
3 → 10 replicas
```

Pods:

```text
4 Pending
```

Karpenter:

```text
cannot provision
```

Users experience:

```text
higher latency
5xx
timeouts
```

Investigation:

```text
1. HPA working?
2. Pods Pending?
3. Why Pending?
4. Karpenter seeing them?
5. NodePool matches?
6. NodeClass valid?
7. EC2 capacity available?
8. IAM valid?
9. subnet IPs available?
10. AWS quota available?
```

This is a real production troubleshooting sequence.

---

# 57. Cluster Autoscaling Monitoring

Our dashboards should include:

```text
Cluster:
  total nodes
  ready nodes
  pending nodes
  allocatable CPU
  allocatable memory

Karpenter:
  NodeClaims
  provisioning failures
  node launch latency
  consolidation activity

Kubernetes:
  Pending Pods
  unschedulable Pods
  scheduling failures

AWS:
  EC2 capacity
  Spot interruption
  service quotas
```

---

# 58. Alerts

Useful alerts include:

```text
PendingPodsHigh
NodeNotReady
NodeProvisioningFailure
KarpenterUnavailable
ClusterCPUCapacityLow
ClusterMemoryCapacityLow
NodeDiskPressure
NodeMemoryPressure
```

Alerts should include:

```text
severity
cluster
namespace
service
runbook
dashboard
```

---

# 59. Example Pending Pod Alert

Conceptually:

```yaml
alert: PendingPodsHigh
expr: kube_pod_status_phase{phase="Pending"} > 0
for: 10m
labels:
  severity: warning
annotations:
  summary: "Pods remain pending"
  description: "Pods have remained pending for more than 10 minutes."
```

In production, the expression should be scoped and refined to avoid alert noise.

---

# 60. Cluster Autoscaler Architecture — Alternative

If we used Cluster Autoscaler instead of Karpenter:

```text
HPA
 ↓
Pending Pods
 ↓
Cluster Autoscaler
 ↓
Node Group
 ↓
ASG
 ↓
EC2
```

Cluster Autoscaler generally scales configured node groups.

For example:

```text
application-m5
min=2
desired=4
max=10
```

CA might change:

```text
4 → 7 nodes
```

---

# 61. When Cluster Autoscaler Is Useful

Cluster Autoscaler can be appropriate when:

```text
fixed node groups
predictable instance types
existing ASG architecture
simple scaling requirements
```

are sufficient.

It remains a valid Kubernetes autoscaling technology.

---

# 62. Why Our Project Uses Karpenter

Our project has:

```text
21 microservices
Spot
multiple instance families
Multi-AZ
variable workloads
HPA
KEDA
dynamic capacity
```

Karpenter fits this flexible architecture well.

But the design principle is more important than the specific product:

> Choose one clear node provisioning strategy per capacity domain.

---

# 63. Production Node Architecture

Our EKS architecture now looks like:

```text
                         EKS
                          │
           ┌──────────────┴──────────────┐
           │                             │
           ▼                             ▼
   Managed Node Groups              Karpenter
           │                             │
      ┌────┴────┐                 ┌──────┴──────┐
      │         │                 │             │
   System    Critical          Spot          On-Demand
      │         │                 │             │
      └─────────┴─────────────────┴─────────────┘
                          │
                          ▼
                       Pods
```

---

# 64. Full Scaling Architecture

```text
                         USER
                           │
                           ▼
                          ALB
                           │
                           ▼
                       Kubernetes
                        Service
                           │
                           ▼
                          Pods
                           │
             ┌─────────────┴─────────────┐
             │                           │
          HPA/KEDA                     VPA
             │                           │
             ▼                           ▼
       Replica Count              Resource tuning
             │
             ▼
       Pending Pods
             │
             ▼
         Karpenter
             │
       ┌─────┴─────┐
       │           │
     Spot       On-Demand
       │           │
       └─────┬─────┘
             ▼
          EC2 Nodes
             │
             ▼
         Kubernetes
         Scheduler
             │
             ▼
            Pods
```

---

# 65. Capacity Strategy by Workload

### System

```text
Managed Node Group
On-Demand
Multi-AZ
Stable baseline
```

### Normal stateless applications

```text
Karpenter
Spot where appropriate
Multi-AZ
Multiple instance types
```

### Critical workloads

```text
Karpenter On-Demand
or dedicated baseline capacity
```

### Event-driven workers

```text
KEDA
    ↓
Karpenter
```

### Stateful managed services

```text
RDS
ElastiCache
managed messaging where selected
```

rather than treating them as ordinary EKS node workloads.

---

# 66. Capacity During Rolling Deployment

Suppose:

```text
3 replicas
```

and:

```yaml
maxSurge: 1
```

Deployment temporarily requires:

```text
4 replicas
```

If existing capacity is full:

```text
4th Pod Pending
```

Karpenter may provision a node.

Therefore deployment capacity must be included in:

```text
NodePool
resource requests
HPA
PDB
```

planning.

---

# 67. Capacity During Blue-Green

Blue:

```text
3 Pods
```

Green:

```text
3 Pods
```

Temporary total:

```text
6 Pods
```

Karpenter must be capable of providing additional capacity.

Therefore:

```text
deployment strategy
+
cluster autoscaling
```

must be designed together.

---

# 68. Capacity During AZ Failure

Normal:

```text
AZ-A = 3 nodes
AZ-B = 3 nodes
AZ-C = 3 nodes
```

AZ-A fails:

```text
AZ-B = 3
AZ-C = 3
```

Karpenter must be able to provision additional capacity in surviving AZs.

If our NodePool only permits:

```text
AZ-A
```

the recovery fails.

Therefore avoid overly restrictive topology requirements.

---

# 69. Production Checklist

### Baseline capacity

```text
[ ] System node group exists
[ ] Multi-AZ baseline
[ ] Karpenter runs on stable nodes
[ ] Critical platform components protected
```

### Karpenter

```text
[ ] NodePool defined
[ ] NodeClass defined
[ ] Spot capacity configured
[ ] On-Demand capacity configured
[ ] Multiple instance types
[ ] Multiple AZs
[ ] CPU/memory limits
[ ] IAM configured
[ ] subnet discovery validated
[ ] security groups validated
```

### Kubernetes

```text
[ ] HPA
[ ] KEDA where required
[ ] resource requests
[ ] resource limits
[ ] topology spread
[ ] PDB
[ ] affinity/tolerations
```

### AWS

```text
[ ] EC2 quotas
[ ] subnet IP capacity
[ ] Spot interruption handling
[ ] CloudWatch/metrics
[ ] AWS cost controls
```

---

# 70. Golden Troubleshooting Flow

When application Pods are Pending:

```text
             Pod Pending
                  │
                  ▼
        kubectl describe pod
                  │
                  ▼
         FailedScheduling?
                  │
         ┌────────┴────────┐
         │                 │
        Yes                No
         │                 │
         ▼                 ▼
  Check scheduling      Check Pod
     constraints        lifecycle
         │
         ▼
   CPU / memory?
         │
         ▼
   Node affinity?
         │
         ▼
   Taint/toleration?
         │
         ▼
   Topology?
         │
         ▼
   Karpenter?
         │
         ▼
   NodePool?
         │
         ▼
   NodeClass?
         │
         ▼
   AWS capacity?
         │
         ▼
   IAM / subnet / quota?
```

---

# 71. Production Incident Example

### Incident

Users report:

```text
Order API latency increased.
```

### Step 1

```bash
kubectl get hpa -n ecommerce-prod
```

Result:

```text
3 → 15 replicas
```

HPA is working.

### Step 2

```bash
kubectl get pods -n ecommerce-prod
```

Result:

```text
8 Running
7 Pending
```

### Step 3

```bash
kubectl describe pod <pending-pod> \
  -n ecommerce-prod
```

Result:

```text
FailedScheduling
```

### Step 4

Check Karpenter.

```bash
kubectl get nodeclaims
```

No new NodeClaims.

### Step 5

Inspect Karpenter logs.

Possible finding:

```text
No instance type satisfies requirements
```

### Root cause

The NodePool had overly restrictive requirements:

```text
only one instance type
only one AZ
```

### Resolution

Expand compatible capacity options through a reviewed GitOps change.

Then:

```text
Karpenter
   ↓
new EC2
   ↓
node Ready
   ↓
Pending Pods scheduled
   ↓
latency recovers
```

This is the kind of incident we will later reproduce deliberately in Phase 18.

---

# 72. Important Production Principle

Never troubleshoot:

```text
HPA
Karpenter
EC2
Kubernetes Scheduler
```

as completely separate systems.

They form one chain:

```text
Application load
      ↓
Metrics
      ↓
HPA/KEDA
      ↓
Pod count
      ↓
Resource requests
      ↓
Scheduler
      ↓
Karpenter
      ↓
EC2
      ↓
Pod placement
      ↓
Application capacity
```

A failure anywhere can appear to the user as:

```text
"Application is slow."
```

---

# 73. Current Platform Architecture

We have now completed:

```text
100 — Spot Instances
101 — Multi-AZ Workload Placement
102 — Kubernetes Resource Management
103 — HPA / VPA / KEDA
104 — Cluster Autoscaling
```

Our EKS capacity architecture is now:

```text
                 ┌─────────────────────────┐
                 │      EKS Cluster        │
                 │                         │
                 │  Managed Node Groups    │
                 │       │                 │
                 │       ├── System        │
                 │       └── Critical      │
                 │                         │
                 │  Karpenter              │
                 │       │                 │
                 │       ├── Spot          │
                 │       └── On-Demand     │
                 │                         │
                 └───────────┬─────────────┘
                             │
                             ▼
                         Application
                            Pods
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
        HPA                KEDA                VPA
     HTTP/API             Events          Recommendations
          │                  │
          └──────────┬───────┘
                     ▼
                Replica Count
                     │
                     ▼
              Pending Pods
                     │
                     ▼
                 Karpenter
                     │
                     ▼
               EC2 Capacity
```

---

## Key Takeaways

1. **HPA/KEDA scale Pods.**
2. **Karpenter scales dynamic EC2 capacity.**
3. **Managed Node Groups provide stable baseline capacity.**
4. **Karpenter should run on stable capacity, not depend entirely on itself.**
5. **Resource requests drive scheduling and influence Karpenter decisions.**
6. **Multiple AZs and instance types improve capacity flexibility.**
7. **Spot should be used with interruption-tolerant workloads.**
8. **Critical workloads need an explicit On-Demand/baseline strategy.**
9. **PDB, topology spread, affinity, and autoscaling must be designed together.**
10. **Never let autoscaling ignore downstream limits such as RDS connections.**
11. **AWS subnet IP capacity and service quotas are part of cluster capacity planning.**
12. **GitOps remains the source of truth for production configuration.**

### Next chapter

**105 — RDS Production Implementation**

We now move from **EKS compute capacity** into the production database layer:

```text
EKS
 ↓
Order Service
 ↓
RDS PostgreSQL
 ↓
Private Subnets
 ↓
Security Groups
 ↓
DB Subnet Group
 ↓
Multi-AZ
 ↓
Backups / PITR
 ↓
Secrets Manager
 ↓
Connection Pooling
```

We will implement the RDS side rather than only describing it, including Terraform, networking, security groups, Secrets Manager, application connectivity, migrations, monitoring, and production validation.
