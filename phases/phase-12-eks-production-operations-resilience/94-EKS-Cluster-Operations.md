# 94 — EKS Cluster Operations

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Workloads:** 21 microservices
> **Purpose:** Operate the EKS platform reliably in production after the infrastructure has been built.

---

# 1. Objective

Until now, we focused heavily on **building and deploying** the platform.

Now we move into the operational phase.

A production DevOps engineer must know what happens when:

* a node becomes unhealthy
* a node runs out of memory
* pods become pending
* an AZ has problems
* an EKS version needs upgrading
* an add-on needs upgrading
* a node must be replaced
* capacity suddenly increases
* a workload must be drained
* a deployment conflicts with a PDB
* Karpenter cannot provision capacity
* the Kubernetes API becomes unavailable
* a node group needs to be changed

The operational model is:

```text
Build
  ↓
Deploy
  ↓
Operate
  ↓
Monitor
  ↓
Troubleshoot
  ↓
Upgrade
  ↓
Scale
  ↓
Recover
```

---

# 2. Our EKS Architecture

Our production cluster is conceptually:

```text
                         AWS
                          │
                          ▼
                       VPC
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
       AZ-A              AZ-B              AZ-C
        │                 │                 │
        ▼                 ▼                 ▼
   Private Subnet    Private Subnet    Private Subnet
        │                 │                 │
        ▼                 ▼                 ▼
     EKS Node          EKS Node          EKS Node
        │                 │                 │
        ├─────────────────┼─────────────────┤
        │                 │                 │
        ▼                 ▼                 ▼
                    Kubernetes
                        Pods
```

We want workloads distributed across multiple AZs.

---

# 3. EKS Control Plane vs Data Plane

This distinction is extremely important.

## Control Plane

AWS manages the EKS control plane.

It contains components such as:

```text
API Server
Scheduler
Controller Manager
etcd
```

Conceptually:

```text
Developer / kubectl
        │
        ▼
EKS API Server
        │
        ├── Scheduler
        ├── Controllers
        └── etcd
```

You don't SSH into EKS managed control-plane hosts.

---

# 4. Data Plane

The data plane runs our workloads.

Example:

```text
EKS Cluster
    │
    ├── Node 1
    │     ├── order-service
    │     ├── payment-service
    │     └── inventory-service
    │
    ├── Node 2
    │     ├── catalog-service
    │     └── user-service
    │
    └── Node 3
          ├── shipping-service
          └── notification-service
```

The nodes are where Kubernetes Pods actually execute.

---

# 5. Node Types

We should not necessarily put everything on the same node pool.

A production cluster may have:

```text
system nodes
application nodes
specialized nodes
Spot nodes
```

Example:

```text
NodePool: system
    │
    ├── CoreDNS
    ├── kube-system components
    └── monitoring infrastructure

NodePool: application
    │
    └── Microservices

NodePool: spot
    │
    └── Fault-tolerant workloads
```

---

# 6. Why Separate System and Application Capacity?

Imagine our application consumes all CPU:

```text
Application Pods
       │
       ▼
100% CPU
       │
       ▼
CoreDNS cannot schedule
```

Now cluster operations themselves can become unhealthy.

We therefore want enough reserved capacity for essential platform components.

---

# 7. Kubernetes Labels

Node labels help control scheduling.

Example:

```yaml id="3w8zbb"
metadata:
  labels:
    workload-type: application
```

Then:

```yaml id="2vkm2j"
nodeSelector:
  workload-type: application
```

The pod will prefer nodes matching that label.

---

# 8. Taints and Tolerations

For stronger isolation:

```text id="0j5u1v"
System Node
   │
   └── taint
```

Example:

```bash id="h6t25h"
kubectl taint nodes <node-name> \
  workload=system:NoSchedule
```

Only pods with the appropriate toleration can schedule there.

---

# 9. Don't Taint Everything Blindly

Taints are powerful.

Incorrect configuration can result in:

```text id="r7q3v1"
Pod
 │
 ▼
No matching node
 │
 ▼
Pending
```

Always verify:

```bash id="g6u8c4"
kubectl describe pod <pod> -n <namespace>
```

Look at:

```text
Events:
  FailedScheduling
```

---

# 10. Check Cluster Nodes

Basic operational command:

```bash id="5ryj5e"
kubectl get nodes -o wide
```

Example:

```text
NAME          STATUS   ROLES    AGE   VERSION
node-a        Ready    <none>   20d   v1.xx
node-b        Ready    <none>   20d   v1.xx
node-c        Ready    <none>   20d   v1.xx
```

The exact Kubernetes version depends on the EKS version currently selected.

---

# 11. Check Node Details

```bash id="9b8j12"
kubectl describe node <node-name>
```

Important sections:

```text
Conditions
Capacity
Allocatable
Allocated resources
Pods
Taints
Labels
Events
```

---

# 12. Capacity vs Allocatable

A node might have:

```text id="xq2b9w"
CPU:
8 vCPU
```

But not all CPU is available to application Pods.

Conceptually:

```text
Node capacity
      │
      ├── Kubernetes/system reservation
      ├── kubelet
      ├── DaemonSets
      └── Application allocatable capacity
```

This is why:

```text id="u9o9t2"
Capacity ≠ Application capacity
```

---

# 13. Resource Requests

Suppose:

```yaml id="a8s4cv"
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The scheduler uses requests when determining where a Pod can run.

If nodes don't have enough allocatable resources:

```text id="4dy4sa"
Pod
 ↓
Scheduler
 ↓
No suitable node
 ↓
Pending
```

---

# 14. Resource Limits

Example:

```yaml id="5rv0ks"
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "1"
    memory: "512Mi"
```

Conceptually:

```text
Request
 ↓
Scheduling requirement

Limit
 ↓
Runtime upper boundary
```

---

# 15. Requests Are Extremely Important

Our HPA and Karpenter decisions depend heavily on resource requests.

For example:

```text id="v7k5p8"
Deployment
  replicas: 10

CPU request:
500m

Total requested CPU:
10 × 500m
= 5 CPU
```

The cluster must have enough allocatable capacity.

---

# 16. Check Resource Usage

Use:

```bash id="whu8jd"
kubectl top nodes
```

and:

```bash id="w4o4js"
kubectl top pods -A
```

Remember:

> Metrics Server provides resource usage metrics for operational purposes; it is not a replacement for Prometheus.

---

# 17. Node Conditions

Important node conditions include:

```text id="n8eybc"
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Check:

```bash id="u7f8cb"
kubectl describe node <node-name>
```

---

# 18. MemoryPressure

Example:

```text id="v6xw9x"
MemoryPressure=True
```

Potential causes:

* too many Pods
* large memory usage
* memory leaks
* insufficient node capacity
* incorrect requests/limits

Investigate:

```bash id="j6y1s4"
kubectl top node <node-name>
```

Then:

```bash id="4w1jlo"
kubectl top pods -A --sort-by=memory
```

---

# 19. DiskPressure

Example:

```text id="w44o4c"
DiskPressure=True
```

Potential causes:

```text
Container images
Container logs
Temporary files
Application files
EBS filesystem usage
```

Investigate the node.

If you have node access for the chosen node-management model, inspect filesystem usage appropriately.

---

# 20. Node NotReady

A major incident:

```text id="g8oyr0"
node-a
STATUS = NotReady
```

First:

```bash id="8x9njr"
kubectl describe node node-a
```

Then:

```bash id="4mb1c6"
kubectl get events -A \
  --sort-by=.lastTimestamp
```

Look at:

```text
kubelet
network
container runtime
disk
memory
CNI
```

---

# 21. Node Failure

Suppose:

```text id="w0gqpi"
AZ-A
 │
 └── node-a
       │
       X
```

Kubernetes may reschedule Pods if enough capacity exists elsewhere.

Architecture:

```text id="1x8v0c"
             AZ-A
              X
              │
          Pod disappears
              │
              ▼
          Scheduler
          /       \
       AZ-B      AZ-C
        │          │
        ▼          ▼
      Pod        Pod
```

This is why multi-AZ architecture matters.

---

# 22. Replica Distribution

If all three replicas land on one node:

```text id="8v5l8n"
Node A
 ├── Pod 1
 ├── Pod 2
 └── Pod 3
```

then:

```text id="8n1z6n"
Node A failure
      ↓
All replicas affected
```

Better:

```text id="4yqv0v"
Node A → Pod 1
Node B → Pod 2
Node C → Pod 3
```

---

# 23. Topology Spread Constraints

We can ask Kubernetes to spread Pods.

Example:

```yaml id="m73jv4"
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

Now Kubernetes attempts to maintain a balanced distribution across AZs.

---

# 24. Pod Anti-Affinity

Another technique:

```yaml id="3q1m86"
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app: order-service
```

This can reduce concentration of replicas on the same node.

---

# 25. PDB Interaction

We already created:

```text id="0n4j0c"
PodDisruptionBudget
```

Example:

```yaml id="8i1zq0"
minAvailable: 2
```

Suppose:

```text
3 replicas
minAvailable = 2
```

A voluntary disruption should not intentionally reduce healthy availability below the PDB requirement.

This becomes important during:

```text
node drain
node upgrades
node replacement
maintenance
```

---

# 26. Cordon

Before draining a node:

```bash id="b3v8rm"
kubectl cordon <node-name>
```

Meaning:

> Don't schedule new Pods onto this node.

Existing Pods remain.

Check:

```bash id="l9k6ut"
kubectl get nodes
```

You should see:

```text
SchedulingDisabled
```

---

# 27. Drain

Then:

```bash id="5e1m1c"
kubectl drain <node-name> \
  --ignore-daemonsets
```

Kubernetes attempts to evict Pods gracefully.

PDBs can prevent eviction if doing so would violate availability requirements.

---

# 28. Why Drain Matters

Never casually terminate a production node containing application Pods.

Correct operational flow:

```text id="qk4p2z"
Cordon
   ↓
Drain
   ↓
Pods rescheduled
   ↓
Verify workloads
   ↓
Replace node
```

---

# 29. Drain Troubleshooting

If drain hangs:

```bash id="m9w4rf"
kubectl get pods -A \
  --field-selector spec.nodeName=<node-name>
```

Then inspect:

```text
PDB
DaemonSet
Local storage
Pod termination
Finalizers
```

---

# 30. PDB Blocking Drain

Example:

```text id="8ezvwu"
3 replicas
PDB minAvailable=3
```

Trying to drain:

```text id="6w9p6x"
Pod eviction
    ↓
PDB says minimum 3
    ↓
Cannot evict
```

This is not necessarily a Kubernetes failure.

It may be the expected behavior of the availability policy.

---

# 31. Cluster Autoscaler vs Karpenter

We previously covered autoscaling.

Operationally:

### Cluster Autoscaler

Works primarily with node groups.

```text id="q5nyha"
Pending Pod
    │
    ▼
Cluster Autoscaler
    │
    ▼
Increase node group
```

### Karpenter

Can provision appropriate capacity based on unschedulable workloads and configured constraints.

```text id="4o1xv7"
Pending Pod
    │
    ▼
Karpenter
    │
    ▼
Select instance/capacity
    │
    ▼
Provision node
```

---

# 32. Karpenter Operational Model

Conceptually:

```text id="h1p4h0"
Pod Pending
     │
     ▼
Scheduler
     │
     ▼
No capacity
     │
     ▼
Karpenter
     │
     ├── CPU requirement
     ├── memory
     ├── architecture
     ├── AZ
     ├── instance constraints
     └── capacity type
     │
     ▼
EC2 capacity
     │
     ▼
Node joins EKS
     │
     ▼
Pod scheduled
```

---

# 33. Karpenter Troubleshooting

If a Pod remains Pending:

```bash id="z8xj2b"
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
FailedScheduling
```

Then inspect Karpenter resources/logs according to the version installed.

Possible causes:

```text
No matching instance type
No available capacity
NodePool constraints
Architecture mismatch
AZ constraint
Taint/toleration mismatch
Subnet/security-group discovery issue
IAM issue
```

---

# 34. Node Group Upgrade

Suppose we have:

```text
Old:
EKS node AMI/version

New:
Updated AMI/version
```

We don't simply terminate every old node.

Instead:

```text id="6g8pmb"
Create/update capacity
       ↓
New nodes join
       ↓
Cordon old nodes
       ↓
Drain old nodes
       ↓
Pods reschedule
       ↓
Verify
       ↓
Remove old nodes
```

Managed node group update mechanisms can automate much of this, subject to configuration and workload constraints.

---

# 35. EKS Version Upgrade

EKS upgrades should be treated as a planned production change.

Flow:

```text id="q21w1s"
Check supported versions
       ↓
Review deprecated APIs
       ↓
Upgrade non-prod
       ↓
Test
       ↓
Upgrade staging
       ↓
Test
       ↓
Jira change
       ↓
Production upgrade
       ↓
Verify
```

Never treat a Kubernetes version upgrade as:

```text id="q4xv6d"
terraform apply
```

and walk away.

---

# 36. Pre-Upgrade Checklist

Before upgrading:

```text
[ ] Current EKS version
[ ] Target supported version
[ ] Kubernetes API deprecations
[ ] Add-on compatibility
[ ] AWS Load Balancer Controller compatibility
[ ] EBS CSI compatibility
[ ] VPC CNI compatibility
[ ] CoreDNS compatibility
[ ] kube-proxy compatibility
[ ] Helm charts
[ ] ArgoCD
[ ] Kyverno
[ ] Argo Rollouts
[ ] Workload tests
[ ] Backup verification
[ ] Rollback/contingency plan
```

---

# 37. Check Current EKS Version

```bash id="7a2n4g"
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1 \
  --query 'cluster.version'
```

---

# 38. Check Kubernetes API Usage

Before a Kubernetes upgrade, search manifests and workloads for APIs that may be removed or deprecated.

Examples:

```text id="mgp4qk"
Ingress APIs
PodSecurityPolicy
old autoscaling APIs
old CRD versions
```

Use current Kubernetes/EKS upgrade tooling and documentation to identify version-specific issues.

---

# 39. Add-On Compatibility

Our EKS cluster depends on:

```text id="5esb0h"
VPC CNI
CoreDNS
kube-proxy
EBS CSI
Pod Identity Agent
AWS Load Balancer Controller
Metrics Server
```

Upgrading the cluster doesn't mean we can blindly leave every add-on unchanged forever.

Plan:

```text id="8w3v2k"
EKS version
   ↓
Add-on compatibility
   ↓
Upgrade sequence
   ↓
Validation
```

---

# 40. EBS CSI Operational Check

Check:

```bash id="0z2p6e"
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Depending on the installation method/version, labels may differ.

Also:

```bash id="9ohb6k"
kubectl get daemonset,statefulset \
  -n kube-system
```

---

# 41. CoreDNS Operational Check

```bash id="y01e0v"
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

Then:

```bash id="x1b9ab"
kubectl logs \
  -n kube-system \
  -l k8s-app=kube-dns
```

If DNS fails:

```text
Application
    │
    ▼
CoreDNS
    │
    ▼
Service discovery
```

Many seemingly unrelated application failures can actually be DNS problems.

---

# 42. VPC CNI

AWS VPC CNI assigns networking to Pods.

Check:

```bash id="1b9jjj"
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Logs:

```bash id="b8j6jt"
kubectl logs \
  -n kube-system \
  -l k8s-app=aws-node
```

Potential issues:

```text
IP exhaustion
Subnet exhaustion
ENI limits
Security group configuration
CNI errors
```

---

# 43. Pod IP Exhaustion

One common EKS operational problem:

```text id="8r7c1m"
Pod Pending
   │
   ▼
CNI cannot allocate IP
```

You may see networking-related scheduling/runtime errors.

Investigate:

```text id="1ih0g3"
Subnet free IPs
ENI capacity
instance type limits
VPC CNI configuration
```

This is why IP planning was important in our VPC chapter.

---

# 44. Kubernetes API Connectivity

Check:

```bash id="35n9pq"
kubectl cluster-info
```

Then:

```bash id="0i7oqp"
kubectl get --raw='/readyz?verbose'
```

If the API isn't reachable, investigate:

```text
AWS authentication
network path
EKS endpoint configuration
DNS
credentials
cluster health
```

---

# 45. Authentication vs Authorization

When:

```bash id="m1lqg2"
kubectl get pods
```

fails, distinguish:

### Authentication

> Who are you?

### Authorization

> Are you allowed to perform this action?

Example:

```text id="e8q8yz"
Authentication
    ↓
AWS identity / EKS access
    ↓
Kubernetes RBAC
    ↓
Authorization
```

---

# 46. Check AWS Identity

```bash id="t0m1i4"
aws sts get-caller-identity
```

Then:

```bash id="0s8jjd"
aws eks update-kubeconfig \
  --name ecommerce-prod \
  --region ap-south-1
```

---

# 47. Check Kubernetes Authorization

```bash id="qf4n0b"
kubectl auth can-i get pods \
  -n ecommerce-prod
```

For a deployment:

```bash id="9qaq8m"
kubectl auth can-i get deployments \
  -n ecommerce-prod
```

This quickly separates many access problems from application problems.

---

# 48. Node Replacement

Suppose an EC2 node is unhealthy.

Operational process:

```text id="q8y9jv"
Identify node
    ↓
Cordon
    ↓
Drain
    ↓
Verify workloads
    ↓
Terminate/replace node
    ↓
New node joins
    ↓
Verify
```

Check:

```bash id="z6m1k3"
kubectl get nodes
```

Then:

```bash id="l7qv9b"
kubectl get pods -A -o wide
```

---

# 49. Verify Pod Rescheduling

After draining:

```bash id="q39kgl"
kubectl get pods \
  -n ecommerce-prod \
  -o wide
```

Check:

```text
NODE
```

You want replicas to be distributed appropriately.

---

# 50. AZ Failure

Imagine:

```text id="4kpr9b"
AZ-A
  │
  X
```

We want:

```text id="jv65gf"
AZ-B → workloads
AZ-C → workloads
```

This requires:

* multiple AZs
* sufficient capacity
* replicas
* topology constraints
* correct load balancing
* database HA
* resilient dependencies

Multi-AZ EKS alone does not guarantee application availability.

---

# 51. AZ Failure and RDS

Our architecture:

```text id="m42b9s"
EKS
 │
 ├── AZ-A
 ├── AZ-B
 └── AZ-C
       │
       ▼
      RDS
```

RDS Multi-AZ provides database availability according to the selected RDS architecture.

The application must also handle connection failures and retries appropriately.

---

# 52. AZ Failure and Redis

Redis/ElastiCache also needs its own HA configuration.

Remember:

```text id="4znfpk"
EKS HA
   ≠
Redis HA
```

Every dependency needs its own resilience model.

---

# 53. AZ Failure and SQS

SQS is managed by AWS and abstracts much of the underlying infrastructure.

Application architecture should still account for:

```text
Retries
Visibility timeout
DLQ
Idempotency
Consumer scaling
```

---

# 54. Node Pressure Incident

Example:

```text id="b6q6r3"
Node:
MemoryPressure=True
```

Investigation:

```bash id="iv22d0"
kubectl top node <node-name>
```

Then:

```bash id="y4i2n9"
kubectl top pods -A --sort-by=memory
```

Find:

```text id="2e6u7r"
order-service
1.8 GiB
```

Then inspect:

```bash id="78b8b7"
kubectl describe pod <pod> -n ecommerce-prod
```

---

# 55. OOMKilled

Pod:

```text id="6h8h2f"
STATUS:
OOMKilled
```

Check:

```bash id="r9h0y8"
kubectl describe pod <pod> \
  -n ecommerce-prod
```

Then:

```bash id="3p5u1v"
kubectl logs <pod> \
  -n ecommerce-prod \
  --previous
```

Compare:

```text
memory request
memory limit
actual usage
application behavior
```

---

# 56. CPU Throttling

A Pod may have:

```yaml id="g39x1y"
limits:
  cpu: "500m"
```

If the application needs sustained higher CPU, it may experience throttling.

Investigate:

```text id="m8ujpo"
CPU usage
CPU limits
request/limit ratio
latency
application behavior
```

Don't automatically respond by simply increasing limits.

First understand why the application needs more CPU.

---

# 57. Pending Pods

When a Pod is Pending:

```bash id="9o5n7e"
kubectl describe pod <pod> \
  -n ecommerce-prod
```

Look at:

```text
Events
```

Common causes:

```text
Insufficient CPU
Insufficient memory
No matching node
Taints
Affinity
Topology constraints
PVC unavailable
CNI/IP limitations
Karpenter constraints
```

---

# 58. Production Troubleshooting Decision Tree

```text id="gck2xm"
Pod Pending?
     │
     ├── YES
     │    │
     │    ▼
     │  describe pod
     │    │
     │    ├── resources?
     │    ├── taints?
     │    ├── affinity?
     │    ├── PVC?
     │    └── Karpenter?
     │
     └── NO
          │
          ▼
       Pod Running?
          │
          ▼
       Ready?
          │
          ├── NO → probes/logs/events
          │
          └── YES
                │
                ▼
             Application
                │
                ├── Logs
                ├── Metrics
                ├── Traces
                └── Dependencies
```

---

# 59. Cluster Capacity Planning

We have 21 services.

Suppose each service initially has:

```text
3 replicas
```

Then:

```text id="z1u8mx"
21 × 3
= 63 application Pods
```

But we also need:

```text id="d3r2v8"
CoreDNS
AWS CNI
kube-proxy
EBS CSI
Load Balancer Controller
Monitoring
Logging
ArgoCD
Kyverno
OTel
```

Therefore:

```text id="b9f44e"
Application capacity
+
Platform capacity
+
Failure headroom
```

must be considered.

---

# 60. Failure Headroom

Suppose we normally require:

```text id="1g1g75"
100 vCPU
```

If we run exactly:

```text
100 vCPU
```

we have little room for:

```text
node failure
deployment surge
autoscaling
maintenance
```

We therefore need appropriate capacity headroom.

The exact percentage should be determined from workload behavior and availability requirements rather than using a universal number.

---

# 61. Deployment Surge

Our rolling strategy:

```yaml id="8v5q6s"
maxSurge: 1
maxUnavailable: 0
```

means the deployment may temporarily require an additional Pod.

If there is no spare capacity:

```text id="w6v7h9"
New Pod
   ↓
Pending
   ↓
Karpenter / Cluster Autoscaler
   ↓
New node
```

Therefore:

> Deployment strategy and cluster autoscaling are connected.

---

# 62. HPA + Karpenter

Example:

```text id="p3x1pb"
Traffic increases
      │
      ▼
HPA
      │
      ▼
More Pods
      │
      ▼
Insufficient node capacity
      │
      ▼
Karpenter
      │
      ▼
New node
      │
      ▼
Pods schedule
```

This is one of the most important production scaling chains.

---

# 63. HPA + PDB + Rolling Deployment

These three must work together.

```text id="i5q7cj"
HPA
 │
 └── number of Pods

Rolling Deployment
 │
 └── replacement behavior

PDB
 │
 └── disruption protection
```

Poorly configured values can conflict.

Example:

```text
2 replicas
minAvailable: 2
maxUnavailable: 0
```

A node drain may not be able to proceed.

---

# 64. Karpenter + Spot

Spot capacity can reduce cost for workloads that tolerate interruption.

Potential workloads:

```text id="o0h7tk"
Stateless
Retryable
Queue consumers
Batch jobs
```

Avoid assuming every workload can safely use Spot.

For critical stateful/control-plane-dependent workloads, evaluate capacity and disruption characteristics carefully.

---

# 65. NodePool Strategy

A conceptual production model:

```text id="o8e3mb"
NodePool
│
├── on-demand
│     └── critical workloads
│
├── spot
│     └── interruptible workloads
│
└── system
      └── platform components
```

The exact design depends on Karpenter version and organization requirements.

---

# 66. Cluster Upgrade Runbook

A production EKS upgrade should follow:

```text id="u0j3lq"
1. Read AWS/EKS upgrade guidance
2. Check current version
3. Check target version
4. Review deprecated APIs
5. Review add-ons
6. Backup/verify recovery
7. Upgrade DEV
8. Test
9. Upgrade QA
10. Test
11. Upgrade STAGING
12. Test
13. Create Jira change
14. Production approval
15. Upgrade PROD
16. Validate nodes
17. Validate workloads
18. Validate add-ons
19. Validate observability
20. Close change
```

---

# 67. Upgrade Validation

After upgrade:

```bash id="e5i3g0"
kubectl get nodes
```

Then:

```bash id="3h75f5"
kubectl get pods -A
```

Then:

```bash id="a3p7hm"
kubectl get events -A \
  --sort-by=.lastTimestamp
```

Then check:

```text id="l7b5wq"
Ingress
Services
DNS
CNI
CSI
ArgoCD
Monitoring
Logging
Application traffic
```

---

# 68. EKS Add-On Upgrade

For managed add-ons, use the appropriate EKS APIs/Terraform configuration rather than manually editing internal Kubernetes resources that AWS manages.

Conceptually:

```text id="x8p3oj"
Terraform
   │
   ▼
EKS Add-on
   │
   ▼
AWS-managed lifecycle
```

For Helm-installed components:

```text id="6a3l3f"
Helm chart
   │
   ▼
GitOps
   │
   ▼
ArgoCD
```

This preserves our desired-state model.

---

# 69. What Should Not Be Done Manually?

Avoid manually modifying production platform components with:

```text id="m5kq3f"
kubectl edit
kubectl apply
helm install from laptop
```

for resources that are supposed to be GitOps-managed.

Instead:

```text id="3m6qk1"
Git
 ↓
PR
 ↓
Review
 ↓
ArgoCD
```

Emergency operational commands may still be necessary during incidents, but afterward reconcile the Git desired state.

---

# 70. Daily EKS Operations

A DevOps/SRE engineer might routinely check:

```bash id="i5v48s"
kubectl get nodes
```

```bash id="j4h8q0"
kubectl get pods -A
```

```bash id="c1q74k"
kubectl get events -A \
  --sort-by=.lastTimestamp
```

```bash id="8q8nml"
kubectl top nodes
```

```bash id="4v8w0j"
kubectl top pods -A
```

```bash id="r2z2hi"
argocd app list
```

and monitor:

```text
Grafana
Kibana
CloudWatch
Security Hub
```

---

# 71. Weekly/Regular Operations

Regular platform maintenance may include:

```text id="s9m1f7"
EKS version review
Add-on updates
Node AMI updates
Security patches
Unused resources
Capacity review
Cost review
Backup verification
DR readiness
Certificate review
IAM access review
Security findings
```

The exact cadence should be defined by your organization's operational policy.

---

# 72. Cluster Health Dashboard

Our Grafana cluster dashboard should show:

```text id="9p0m6r"
EKS Cluster

Nodes
 ├── Ready
 ├── NotReady
 └── Pressure

Pods
 ├── Running
 ├── Pending
 ├── Failed
 └── Restarts

Resources
 ├── CPU
 ├── Memory
 └── Disk

Workloads
 ├── Deployments
 ├── StatefulSets
 └── DaemonSets

Autoscaling
 ├── HPA
 ├── Karpenter
 └── Node capacity
```

---

# 73. Alerts

Useful operational alerts include:

```text id="2v1lbg"
NodeNotReady
NodeMemoryPressure
NodeDiskPressure
PodCrashLooping
PodPending
OOMKilled
High restart count
High API error rate
High application latency
Insufficient capacity
SQS backlog
RDS connection exhaustion
```

---

# 74. Security Alerts

Separate security alerts include:

```text id="8n9o5v"
GuardDuty finding
IAM policy modification
Unexpected security-group change
Unexpected Secrets Manager access
Unexpected S3 access
Unauthorized Kubernetes API activity
Image admission rejection
Suspicious network activity
```

---

# 75. Incident Example — Node Failure

Let's walk through a real scenario.

```text id="4f7q8j"
09:10
node-a becomes NotReady
```

Alert:

```text id="e9l6bj"
NodeNotReady
```

Engineer:

```bash id="z4q4jj"
kubectl get nodes
```

Find:

```text
node-a NotReady
```

Then:

```bash id="jzqf0q"
kubectl describe node node-a
```

---

# 76. Determine Impact

```bash id="r4wq0f"
kubectl get pods -A \
  -o wide | grep node-a
```

Determine:

```text
Which services were on node-a?
How many replicas remain?
Are replicas running in other AZs?
```

---

# 77. Check Application Health

```bash id="y6x5cc"
kubectl get pods \
  -n ecommerce-prod
```

Then Grafana:

```text
Error rate?
Latency?
Traffic?
```

Then:

```text id="xjv4l2"
ELK
```

for application errors.

---

# 78. Recovery

If capacity exists:

```text id="2xq3jv"
Pods reschedule
```

If not:

```text id="8rr9gr"
Karpenter
   │
   ▼
New node
   │
   ▼
Pods schedule
```

Then:

```text id="n0s4si"
Verify service
```

---

# 79. Incident Example — Node Drain

Change request:

```text id="h75f0d"
CHG-10501
Node maintenance
```

Engineer:

```bash id="n7m0ag"
kubectl cordon node-a
```

Then:

```bash id="ql4g4k"
kubectl drain node-a \
  --ignore-daemonsets
```

Monitor:

```bash id="1m8p4v"
kubectl get pods -A -o wide
```

Then replace node.

Finally:

```bash id="8r9s1c"
kubectl uncordon node-a
```

if the node is intentionally returned to service.

---

# 80. Production EKS Golden Rules

### Rule 1

Don't treat nodes as permanent.

```text
Node = replaceable capacity
```

### Rule 2

Don't manually change GitOps-managed workloads without an emergency reason.

### Rule 3

Always understand PodDisruptionBudgets before draining.

### Rule 4

Always consider AZ distribution.

### Rule 5

Requests and limits are part of capacity planning.

### Rule 6

Autoscaling must be designed with workload behavior.

### Rule 7

Upgrade EKS systematically.

### Rule 8

Monitor the monitoring infrastructure itself.

### Rule 9

Have enough capacity for maintenance and failure.

### Rule 10

Test recovery instead of assuming it works.

---

# 81. Complete EKS Operations Model

```text id="5e7h1w"
                    EKS
                     │
        ┌────────────┼─────────────┐
        │            │             │
     Control       Nodes         Add-ons
      Plane          │             │
        │            │             │
        │       ┌────┼────┐       │
        │       │    │    │       │
        │      AZ-A AZ-B AZ-C     │
        │       │    │    │       │
        │       └────┼────┘       │
        │            │            │
        │          Pods           │
        │            │            │
        └────────────┼────────────┘
                     │
              Observability
                     │
       ┌─────────────┼──────────────┐
       │             │              │
    Prometheus      ELK            OTel
       │             │              │
       ▼             ▼              ▼
    Grafana        Kibana          Jaeger
                     │
                  Security
                     │
          CloudTrail / GuardDuty
```

---

# 82. Practical Troubleshooting Command Set

Keep this command set available.

### Cluster

```bash id="k0e9u2"
kubectl cluster-info
```

### Nodes

```bash id="1s5xhv"
kubectl get nodes -o wide
```

### Node details

```bash id="x3ax1d"
kubectl describe node <node>
```

### Pods

```bash id="u5v3xm"
kubectl get pods -A -o wide
```

### Pod details

```bash id="g8f7d4"
kubectl describe pod <pod> -n <namespace>
```

### Logs

```bash id="x7x3i4"
kubectl logs <pod> -n <namespace>
```

### Previous container logs

```bash id="0qj8yx"
kubectl logs <pod> \
  -n <namespace> \
  --previous
```

### Events

```bash id="7z4s9w"
kubectl get events -A \
  --sort-by=.lastTimestamp
```

### Resources

```bash id="v9p3om"
kubectl top nodes
```

```bash id="qj5u0g"
kubectl top pods -A
```

### Authorization

```bash id="6v9e4p"
kubectl auth can-i get pods \
  -n ecommerce-prod
```

### AWS identity

```bash id="5x7q5b"
aws sts get-caller-identity
```

### EKS

```bash id="3z7w1y"
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

---

# 83. EKS Production Readiness Checklist

## Cluster

* [ ] Multi-AZ
* [ ] Private worker nodes
* [ ] Correct security groups
* [ ] Correct subnet tags
* [ ] Cluster logging configured
* [ ] API endpoint access reviewed

## Nodes

* [ ] System capacity
* [ ] Application capacity
* [ ] Node update strategy
* [ ] AMI patching
* [ ] Capacity headroom
* [ ] Multi-AZ distribution

## Workloads

* [ ] Requests
* [ ] Limits
* [ ] Probes
* [ ] PDB
* [ ] Topology spread
* [ ] Anti-affinity where required

## Autoscaling

* [ ] HPA
* [ ] Karpenter/Cluster Autoscaler
* [ ] Node capacity
* [ ] Scale-up testing
* [ ] Scale-down testing

## Add-ons

* [ ] VPC CNI
* [ ] CoreDNS
* [ ] kube-proxy
* [ ] EBS CSI
* [ ] Pod Identity Agent
* [ ] AWS Load Balancer Controller
* [ ] Metrics Server

## Operations

* [ ] Node drain procedure
* [ ] Node replacement procedure
* [ ] EKS upgrade runbook
* [ ] Add-on upgrade runbook
* [ ] AZ failure test
* [ ] Backup/DR procedure
* [ ] Incident runbook

---

# 84. Final Production Mental Model

The most important thing to understand is:

```text id="5a7t9x"
                  APPLICATION
                       │
                       ▼
                      POD
                       │
                       ▼
                     NODE
                       │
                       ▼
                     AZ
                       │
                       ▼
                      VPC
                       │
                       ▼
                      AWS
```

But operations works in the opposite direction:

```text id="u7x8n2"
ALERT
  │
  ▼
What failed?
  │
  ▼
Application?
  │
  ▼
Pod?
  │
  ▼
Node?
  │
  ▼
AZ?
  │
  ▼
Network?
  │
  ▼
AWS dependency?
```

A strong EKS operator doesn't immediately restart Pods.

They identify **which layer is actually failing**.

---

# 85. Chapter 94 Completion

We have now moved from:

```text
BUILD THE CLUSTER
```

to:

```text
OPERATE THE CLUSTER
```

The next logical chapter is:

```text
95-EKS-Node-Groups-and-Karpenter-Deep-Dive.md
```

There we will go much deeper into the actual compute layer:

```text
Managed Node Groups
       │
       ├── Launch Templates
       ├── AMIs
       ├── Instance Types
       ├── On-Demand
       ├── Spot
       ├── Taints
       ├── Labels
       ├── Scaling
       └── Upgrade

Karpenter
       │
       ├── NodePools
       ├── NodeClasses
       ├── Instance Selection
       ├── Consolidation
       ├── Disruption
       ├── Spot
       ├── Capacity Types
       ├── AZ Selection
       └── Production Troubleshooting
```

This will be the **hands-on compute-capacity chapter**, including what happens internally when one of our 21 microservices suddenly needs more Pods and the cluster doesn't have enough capacity.
