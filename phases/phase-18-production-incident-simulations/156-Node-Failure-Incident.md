# 156 — Kubernetes Node Failure Production Incident

> **Phase 18 — Incident Management**  
> **Incident #156**  
> **Environment:** Production  
> **Platform:** Amazon EKS  
> **Primary infrastructure:** EC2 worker nodes / Karpenter-managed nodes  
> **Example affected services:** `order-service`, `payment-service`, `inventory-service`

---

# 1. Objective

This chapter explains how to troubleshoot a **Kubernetes worker node failure** in a production EKS environment.

The goal is not simply:

```text
Node NotReady
    ↓
Restart node
```

Instead, determine:

```text
What failed?
    ↓
Node?
Kubelet?
Container runtime?
Network?
EC2 instance?
EBS?
Memory?
Disk?
AWS infrastructure?
    ↓
Which Pods were affected?
    ↓
Did Kubernetes reschedule them?
    ↓
Did PDB/topology constraints prevent recovery?
    ↓
Did downstream systems become overloaded?
    ↓
Is replacement capacity available?
    ↓
Recover
    ↓
RCA
```

---

# 2. Production EKS Architecture

Our production architecture:

```text
                         AWS EKS
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
          AZ-1           AZ-2           AZ-3
             │              │              │
        ┌─────────┐    ┌─────────┐    ┌─────────┐
        │ Node 1  │    │ Node 3  │    │ Node 5  │
        │ Node 2  │    │ Node 4  │    │ Node 6  │
        └────┬────┘    └────┬────┘    └────┬────┘
             │              │              │
          Pods           Pods           Pods
```

Kubernetes should distribute critical workloads across:

```text
AZ-1
AZ-2
AZ-3
```

using:

- topology spread constraints
- pod anti-affinity where appropriate
- PodDisruptionBudget
- multiple nodes
- autoscaling
- Karpenter/node groups

---

# 3. What Is a Node Failure?

A node failure occurs when a Kubernetes worker node becomes unable to reliably run workloads.

Possible states include:

```text
Ready
NotReady
Unknown
SchedulingDisabled
```

A node can fail because of:

- EC2 instance failure
- kernel crash
- kubelet failure
- container runtime failure
- disk exhaustion
- memory pressure
- CPU starvation
- network failure
- CNI failure
- EBS problems
- node termination
- spot interruption
- OS issue
- incorrect node configuration
- hardware/infrastructure failure

---

# 4. Important Principle

A Kubernetes node is **replaceable infrastructure**.

Do not design the application around:

```text
"This particular EC2 instance must stay alive."
```

Instead:

```text
Node fails
   ↓
Pods become unavailable
   ↓
Kubernetes reschedules
   ↓
New node becomes available
   ↓
Pods become Ready
```

The application should recover without depending on a specific worker node.

---

# 5. Node Failure Detection

Typical alerts:

```text
Kubernetes node NotReady
Node unavailable
Node heartbeat missing
Node memory pressure
Node disk pressure
Node PID pressure
Node network unavailable
```

Example Prometheus alert:

```yaml
groups:
  - name: kubernetes-node-alerts
    rules:

      - alert: KubernetesNodeNotReady
        expr: kube_node_status_condition{
          condition="Ready",
          status="true"
        } == 0
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Kubernetes node is not Ready"
          runbook: "runbooks/node-failure.md"
```

The exact expression depends on the Prometheus/kube-state-metrics version and label set.

---

# 6. First Incident Command

Start with:

```bash
kubectl get nodes -o wide
```

Example:

```text
NAME                     STATUS     ROLES    AGE
ip-10-0-11-25            Ready      <none>   42d
ip-10-0-12-31            NotReady   <none>   42d
ip-10-0-13-42            Ready      <none>   41d
```

Immediately identify:

```text
Failed node
AZ
Instance
Instance type
Pod count
```

---

# 7. Check Node Details

```bash
kubectl describe node ip-10-0-12-31
```

Look at:

```text
Conditions
Events
Capacity
Allocatable
Pod count
Taints
Kubelet version
Container runtime
Network information
```

Important conditions:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

---

# 8. Node Conditions

### Ready=False

The node is not considered healthy.

### MemoryPressure=True

The node is experiencing memory pressure.

### DiskPressure=True

The node has insufficient disk resources.

### PIDPressure=True

The node is approaching process ID exhaustion.

### NetworkUnavailable=True

Networking may not be correctly configured.

---

# 9. Check Node Events

```bash
kubectl describe node ip-10-0-12-31
```

Look at:

```text
Events:
  Warning
  Normal
```

Examples:

```text
MemoryPressure
DiskPressure
NodeNotReady
Kubelet stopped posting status
Container runtime unavailable
```

Events can give the first clue, but they are not always the complete root cause.

---

# 10. Check Pods on Failed Node

```bash
kubectl get pods \
  -A \
  -o wide \
  --field-selector spec.nodeName=ip-10-0-12-31
```

This tells us which workloads were running there.

Example:

```text
NAMESPACE    NAME                         STATUS
ecommerce    order-service-abc123         Running
ecommerce    payment-service-def456       Running
ecommerce    inventory-service-xyz789     Running
monitoring   fluent-bit-abcde             Running
```

---

# 11. Why This Is Important

Suppose:

```text
Node failure
   ↓
order-service Pod lost
payment-service Pod lost
inventory-service Pod lost
```

If all replicas were on the same node:

```text
order-service:
3 replicas
    ↓
Node failure
    ↓
3 replicas lost
```

That is an architecture problem.

With proper topology:

```text
order-service:

AZ-1 → Pod 1
AZ-2 → Pod 2
AZ-3 → Pod 3
```

one node failure may only remove one replica.

---

# 12. Check Deployments

```bash
kubectl get deployment \
  -n ecommerce
```

Example:

```text
NAME                  READY   UP-TO-DATE   AVAILABLE
order-service         2/3     3            2
payment-service       2/3     3            2
inventory-service     3/3     3            3
```

This tells us whether Kubernetes is already replacing lost Pods.

---

# 13. Check ReplicaSets

```bash
kubectl get rs \
  -n ecommerce
```

Determine:

```text
Desired
Current
Ready
Available
```

If desired replicas are 3 but only 2 are available:

```text
2/3
```

we need to determine why the replacement Pod has not become Ready.

---

# 14. Check Pending Pods

```bash
kubectl get pods \
  -n ecommerce \
  --field-selector=status.phase=Pending
```

Example:

```text
NAME                           STATUS
order-service-7fd88c-x9        Pending
payment-service-8df99d-y4      Pending
```

Now inspect:

```bash
kubectl describe pod \
  order-service-7fd88c-x9 \
  -n ecommerce
```

Look at events:

```text
Insufficient cpu
Insufficient memory
No nodes available
Topology constraints
Taint/toleration mismatch
PVC unavailable
Node affinity mismatch
```

---

# 15. Node Failure vs Capacity Failure

This distinction matters.

### Node failure

```text
Node 2
   ↓
NotReady
```

but:

```text
Node 1
Node 3
Node 4
```

have enough capacity.

Kubernetes can reschedule.

### Capacity failure

```text
Node 2 failed
   ↓
Remaining nodes full
   ↓
Replacement Pods Pending
```

Now we need:

```text
Karpenter
or
managed node group scaling
```

to create capacity.

---

# 16. Karpenter Recovery

If Karpenter is used:

```text
Pod Pending
   ↓
Karpenter observes unschedulable Pod
   ↓
Calculates required capacity
   ↓
Creates EC2 instance
   ↓
Node joins EKS
   ↓
Pod scheduled
```

Architecture:

```text
Failed Node
    ↓
Pods unavailable
    ↓
ReplicaSet creates replacement
    ↓
Pod Pending
    ↓
Karpenter
    ↓
New EC2 Node
    ↓
Pod scheduled
    ↓
Readiness probe
    ↓
Traffic restored
```

---

# 17. Check Karpenter

Depending on the deployed Karpenter version/configuration, inspect its controller and node lifecycle resources.

For example:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter
```

Check logs:

```bash
kubectl logs \
  -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  --tail=200
```

Look for:

```text
No matching NodePool
Insufficient capacity
Subnet issue
Security group issue
IAM issue
Instance launch failure
EC2 capacity issue
```

Exact labels and resource names depend on the Karpenter version deployed.

---

# 18. Managed Node Groups

If using EKS Managed Node Groups:

```text
Node failure
    ↓
ASG / managed node group
    ↓
Replacement EC2 instance
    ↓
EKS node joins
    ↓
Pods scheduled
```

Check AWS-side node group state and EC2 instance health.

---

# 19. Check EC2 Instance

Identify the EC2 instance corresponding to the node.

Then inspect:

```text
Instance state
System status checks
Instance status checks
Termination
Reboot
Spot interruption
Network interface
EBS volumes
```

Example:

```bash
aws ec2 describe-instance-status \
  --instance-ids i-0123456789abcdef0
```

A failed system status check may indicate an underlying infrastructure issue.

---

# 20. EC2 Failure Types

### System status check failure

Potential:

- host infrastructure
- networking
- power
- hardware

### Instance status check failure

Potential:

- OS
- kernel
- boot issue
- networking configuration

### Spot interruption

The instance is intentionally being reclaimed by AWS.

### Terminated instance

Node disappears completely.

---

# 21. Spot Instance Failure

If workloads use Spot:

```text
Spot interruption
    ↓
Node termination notice
    ↓
Pods need relocation
```

Production workloads should use:

- multiple AZs
- multiple instance types
- On-Demand capacity for critical baseline
- Spot for suitable workloads
- Karpenter diversification

Do not place the entire critical order path on Spot-only capacity.

---

# 22. PodDisruptionBudget

A PDB can protect availability during voluntary disruptions.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: ecommerce
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

But remember:

> A PDB does not prevent every involuntary failure.

A sudden EC2/node failure can still remove Pods.

PDB primarily helps control **voluntary disruptions** such as node draining.

---

# 23. Topology Spread

For critical services:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

This encourages distribution across AZs.

For three replicas:

```text
AZ-1 → Pod 1
AZ-2 → Pod 2
AZ-3 → Pod 3
```

This is much safer than:

```text
AZ-1 → Pod 1
AZ-1 → Pod 2
AZ-1 → Pod 3
```

---

# 24. Pod Anti-Affinity

Another option:

```yaml
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

This encourages replicas to be placed on different nodes.

Use topology rules based on the actual availability requirements.

---

# 25. Node Failure and Stateful Workloads

Stateless application:

```text
Pod
 ↓
Node failure
 ↓
Replacement Pod
```

Usually straightforward.

Stateful workload:

```text
Pod
 ↓
PVC
 ↓
EBS
```

requires additional consideration.

For example:

```text
PostgreSQL StatefulSet
```

may depend on:

- EBS volume
- AZ
- CSI driver
- volume attachment
- StatefulSet identity
- database recovery

This is one reason production ecommerce workloads often use:

```text
RDS
```

instead of operating PostgreSQL directly on EKS.

---

# 26. EBS Volume Attachment Problem

Suppose a StatefulSet Pod is rescheduled to another AZ.

An EBS volume may be AZ-specific.

Then:

```text
Old Node
  ↓
AZ-1
  ↓
EBS volume
```

cannot simply attach to:

```text
New Node
  ↓
AZ-2
```

without appropriate architecture.

Therefore storage topology matters.

---

# 27. Node Failure and DaemonSets

Some workloads run one Pod per node.

Examples:

```text
Fluent Bit
Node exporter
Security agents
```

When a node fails:

```text
DaemonSet Pod disappears
```

It should automatically appear on the replacement node.

For example:

```bash
kubectl get daemonset -A
```

Check:

```text
DESIRED
CURRENT
READY
AVAILABLE
```

---

# 28. Node Failure and Logging

If Fluent Bit was running on the failed node:

```text
Node failure
   ↓
Fluent Bit disappears
   ↓
Logs from that node stop
```

Logs already shipped to Elasticsearch/OpenSearch remain available.

Logs not yet shipped may be lost if only stored locally.

This is why logging pipelines should have:

- buffering
- retry
- appropriate persistence where needed
- centralized storage
- monitoring

---

# 29. Node Failure and Monitoring

Prometheus itself may be unaffected if running elsewhere.

But:

```text
node-exporter
```

on the failed node disappears.

Prometheus should show:

```text
up{instance="failed-node"} = 0
```

Kube-state-metrics can show:

```text
node ready = false
```

This allows correlation between:

```text
Node failure
+
Pod failures
+
Customer impact
```

---

# 30. Node Failure and ALB

Suppose ALB routes to:

```text
Node 1
Node 2
Node 3
```

If Node 2 disappears:

```text
ALB
 ├── Node 1 → healthy
 ├── Node 2 → unavailable
 └── Node 3 → healthy
```

The AWS Load Balancer Controller and Kubernetes endpoint updates should remove unavailable targets according to the configured target model.

With IP targets, the ALB generally targets Pod IPs directly.

The exact behavior depends on ingress/service configuration.

---

# 31. Service Endpoint Changes

Check:

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

When a Pod disappears:

```text
EndpointSlice
     ↓
Pod endpoint removed
```

Traffic should move to healthy endpoints.

---

# 32. Node Failure and HPA

HPA may see:

```text
Available replicas ↓
```

and attempt to maintain desired capacity.

But HPA cannot create physical node capacity.

The chain is:

```text
HPA
 ↓
Desired replicas
 ↓
Scheduler
 ↓
Pending Pods
 ↓
Karpenter / Node Group
 ↓
EC2 node
```

Therefore:

> **HPA scales Pods; Karpenter/node groups provide the infrastructure capacity.**

---

# 33. Node Failure and Cluster Autoscaling

If using Cluster Autoscaler instead of Karpenter:

```text
Pending Pod
   ↓
Cluster Autoscaler
   ↓
Node Group scale-out
   ↓
EC2
   ↓
Pod scheduled
```

The exact autoscaling technology is a platform architecture decision.

---

# 34. Node Failure and IP Capacity

EKS commonly uses AWS VPC CNI.

Pods consume VPC IP addresses.

Suppose:

```text
Node capacity available
```

but:

```text
Subnet IP capacity = exhausted
```

New Pods may remain Pending.

Therefore node recovery can fail because of:

```text
Subnet/IP capacity
```

even though EC2 capacity is available.

Check:

```text
VPC subnet free IPs
ENI capacity
Pod density
instance type limits
```

---

# 35. Node Failure Investigation Tree

```text
                 Node NotReady
                      │
                      ▼
              Check node conditions
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Memory       Disk        Network
       Pressure     Pressure    Failure
          │           │           │
          └───────────┼───────────┘
                      ▼
              Check EC2 health
                      │
              ┌───────┴────────┐
              ▼                ▼
         EC2 healthy       EC2 failed
              │                │
              ▼                ▼
       Check kubelet      Replace node
       runtime/network         │
              │                ▼
              └────────→ Reschedule Pods
                              │
                              ▼
                     Are Pods Pending?
                              │
                              ▼
                      Check capacity
                              │
                       ┌──────┴──────┐
                       ▼             ▼
                    Capacity      Constraints
                       │             │
                       ▼             ▼
                   Karpenter    PDB/topology/
                   /MNG         affinity/PVC
```

---

# 36. Immediate Mitigation

If a node is unhealthy but still responding:

```bash
kubectl cordon <node>
```

This prevents new workloads from being scheduled there.

If appropriate and safe:

```bash
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

**Do not blindly run drain in production.**

Before draining, check:

- PDBs
- replica count
- topology
- stateful workloads
- local storage
- customer impact
- replacement capacity

---

# 37. Why `kubectl drain` Can Be Dangerous

Suppose:

```text
order-service replicas = 2
```

and both are already on:

```text
Node A
```

If you drain Node A:

```text
Pod 1 → terminated
Pod 2 → terminated
```

You could temporarily lose the entire service.

Therefore first verify:

```bash
kubectl get pods -n ecommerce -l app=order-service -o wide
```

---

# 38. GitOps Consideration

Node recovery itself is infrastructure/runtime behavior.

But permanent configuration changes should go through GitOps.

For example:

```text
Temporary:
kubectl cordon
kubectl drain
```

Permanent:

```text
Terraform
Karpenter NodePool
Helm
Topology constraints
PDB
Resource requests
```

should be committed to Git.

---

# 39. Example Production Scenario

Suppose:

```text
Node count = 6

AZ-1:
Node 1
Node 2

AZ-2:
Node 3
Node 4

AZ-3:
Node 5
Node 6
```

Critical workloads:

```text
order-service = 6 replicas
payment-service = 6 replicas
inventory-service = 6 replicas
```

Topology spread distributes them across AZs.

Now:

```text
Node 3
```

fails.

Some Pods disappear.

Kubernetes creates replacement Pods.

Karpenter creates:

```text
Node 7
```

Pods move onto Node 7.

Customer impact should be minimal.

---

# 40. Bad Architecture Example

Suppose all order Pods are placed on one node:

```text
Node 1
 ├── order Pod 1
 ├── order Pod 2
 ├── order Pod 3
 └── order Pod 4
```

Node failure:

```text
Node 1
   ↓
All 4 Pods lost
   ↓
Order service unavailable
```

The root problem isn't just the node failure.

The architecture also lacked workload distribution.

---

# 41. Example RCA — Node Failure

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-156` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 11:10–11:43 IST |
| Failed Node | `ip-10-0-12-31` |
| AZ | `ap-south-1b` |
| Affected Services | `order-service`, `payment-service` |
| Root Cause | EC2 underlying infrastructure failure |
| Permanent Data Loss | No |

---

# 42. Customer Impact

During the incident:

```text
order-service
```

temporarily lost one or more Pods.

Some requests experienced:

```text
5xx
timeout
latency
```

while Kubernetes rescheduled the workloads.

Payment processing was protected by idempotency.

No duplicate successful payment was recorded.

No permanent database data was lost.

---

# 43. Timeline

```text
11:00  Cluster healthy
       6 worker nodes
       All critical services healthy

11:10  Node ip-10-0-12-31 stops responding

11:11  Node heartbeat unavailable

11:12  Kubernetes marks node NotReady

11:13  Node failure alert fires

11:14  order-service Pods on node become unavailable

11:15  ALB target health begins changing

11:16  Customer API latency increases

11:18  SEV-1 declared

11:20  DevOps checks node conditions

11:22  EC2 system status check indicates infrastructure failure

11:24  Replacement capacity requested

11:27  Karpenter launches replacement node

11:31  New node joins EKS

11:33  Replacement Pods scheduled

11:36  Pods pass readiness probes

11:38  ALB target health stabilizes

11:40  Order/payment transactions validated

11:43  Incident resolved
```

---

# 44. Investigation

### Kubernetes

```bash
kubectl get nodes -o wide
```

Result:

```text
ip-10-0-11-25    Ready
ip-10-0-12-31    NotReady
ip-10-0-13-42    Ready
```

### Node Conditions

```text
Ready=False
```

### Pods

Several critical Pods were running on the failed node.

### EC2

AWS-side status checks indicated an underlying instance/system problem.

The kubelet was not the primary cause.

---

# 45. Root Cause

The EC2 instance hosting the Kubernetes node experienced an underlying infrastructure failure.

The node stopped responding.

Therefore:

```text
EC2 infrastructure failure
        ↓
Worker node unavailable
        ↓
Kubernetes Node NotReady
        ↓
Pods unavailable
        ↓
ReplicaSet creates replacements
        ↓
Karpenter provides new capacity
        ↓
Pods rescheduled
```

---

# 46. Amplification Mechanism

The node failure itself was not the only reason customer impact occurred.

A contributing architectural weakness was:

```text
order-service replicas were not perfectly distributed
```

Several critical replicas were concentrated on the affected node.

Therefore:

```text
Node failure
    ↓
Multiple replicas lost simultaneously
    ↓
Available capacity temporarily drops
    ↓
ALB sends traffic to fewer healthy targets
    ↓
Latency/5xx increases
```

---

# 47. Contributing Factors

1. EC2 infrastructure failure.
2. Multiple critical Pods were colocated on the failed node.
3. Node replacement took several minutes.
4. Capacity headroom was lower than desired.
5. Topology distribution could be improved.
6. Customer-facing service had limited temporary redundancy during the failure window.

---

# 48. What Went Well

- Kubernetes detected the node failure.
- ReplicaSets recreated missing Pods.
- Karpenter provided replacement infrastructure.
- RDS remained healthy.
- Redis remained healthy.
- Payment idempotency prevented duplicate business operations.
- Monitoring detected the node quickly.
- GitOps configuration remained consistent.

---

# 49. What Went Wrong

- Critical replicas were not optimally distributed.
- Some services had insufficient temporary capacity.
- Replacement node provisioning added recovery time.
- Node-level failure had a measurable customer impact.

---

# 50. Immediate Mitigation

```text
1. Confirm node failure
2. Confirm affected Pods
3. Check remaining cluster capacity
4. Verify critical service replicas
5. Provide replacement capacity
6. Reschedule workloads
7. Monitor ALB
8. Validate RDS/Redis
9. Validate order/payment flow
10. Confirm recovery
```

---

# 51. Corrective Actions

| Action | Owner |
|---|---|
| Improve topology spread | Platform |
| Review pod anti-affinity | Platform |
| Increase critical service replica minimums | Application + Platform |
| Review Karpenter capacity headroom | Platform |
| Add node failure game day | SRE |
| Add node-level capacity alerts | SRE |
| Review AZ distribution | Platform |
| Validate PDBs | Platform |
| Review Spot/On-Demand mix | Platform |
| Test node termination scenarios | SRE |

---

# 52. Five Whys

### Why did customers experience increased latency?

Critical application replicas were lost when the node failed.

### Why were multiple replicas lost?

Several replicas were colocated on the same node.

### Why were replicas colocated?

Workload topology distribution was not strict enough.

### Why did the cluster not immediately recover?

Replacement node capacity had to be provisioned.

### Why did recovery take longer than expected?

The cluster did not have sufficient spare capacity to immediately absorb the failed node's workloads.

---

# 53. Final Root Cause Statement

> The production incident was caused by an underlying EC2 infrastructure failure that made an EKS worker node unavailable. Customer impact was amplified because multiple replicas of critical workloads were concentrated on the failed node and the cluster required additional capacity before all workloads could be rescheduled.

---

# 54. Improved Architecture

Before:

```text
AZ-1
 └── Node 1
      ├── order-1
      ├── order-2
      └── order-3
```

After:

```text
AZ-1                    AZ-2                    AZ-3

Node 1                  Node 3                  Node 5
 └── order-1             └── order-2             └── order-3

Node 2                  Node 4                  Node 6
 └── payment-1           └── payment-2           └── payment-3
```

Combined with:

```text
Topology Spread
+
PDB
+
Karpenter
+
Multi-AZ
+
Capacity Headroom
```

node failure becomes much less disruptive.

---

# 55. Node Failure Runbook

Create:

```text
runbooks/node-failure.md
```

Recommended procedure:

```text
1. Confirm NodeNotReady
2. Identify AZ
3. Identify EC2 instance
4. Check node conditions
5. Check node events
6. List affected Pods
7. Check critical service availability
8. Check Pending Pods
9. Check cluster capacity
10. Check Karpenter/MNG
11. Check EC2 health
12. Cordon if required
13. Drain only when safe
14. Replace node
15. Verify Pod scheduling
16. Verify readiness
17. Verify ALB
18. Verify business flow
19. Remove temporary mitigation
20. RCA
```

---

# 56. Production Checklist

### Node

- [ ] Node status checked
- [ ] Conditions checked
- [ ] Events checked
- [ ] EC2 status checked
- [ ] AZ identified
- [ ] Disk/memory/network checked

### Workloads

- [ ] Affected Pods identified
- [ ] Replica count checked
- [ ] Pending Pods checked
- [ ] PDB checked
- [ ] Topology distribution checked
- [ ] Stateful workloads identified

### Capacity

- [ ] Remaining node capacity checked
- [ ] Karpenter checked
- [ ] Node group checked
- [ ] Subnet IP capacity checked
- [ ] Instance capacity checked

### Recovery

- [ ] Replacement node available
- [ ] Pods rescheduled
- [ ] Readiness verified
- [ ] ALB targets healthy
- [ ] Application metrics normal
- [ ] Business transaction validated

---

# 57. Important Commands

### Nodes

```bash
kubectl get nodes -o wide
```

```bash
kubectl describe node <node>
```

### Pods on node

```bash
kubectl get pods -A \
  -o wide \
  --field-selector spec.nodeName=<node>
```

### Pending Pods

```bash
kubectl get pods -A \
  --field-selector=status.phase=Pending
```

### Deployment

```bash
kubectl get deployment -A
```

### PDB

```bash
kubectl get pdb -A
```

### EndpointSlices

```bash
kubectl get endpointslice -A
```

### Karpenter

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=karpenter
```

---

# 58. What DevOps Does During Node Failure

DevOps/Platform typically owns:

```text
Node
EKS
EC2
Karpenter
Node Groups
Networking
CNI
Capacity
Scheduling constraints
PDB
Topology
Monitoring
Alerting
Recovery
```

Application teams own:

```text
Application health
Readiness/liveness
Graceful shutdown
State handling
Idempotency
Business validation
```

DBA owns database internals if the node failure causes database-side issues.

---

# 59. Interview Answer

If asked:

> **"How do you handle an EKS node failure in production?"**

Answer:

> "First I confirm the node is actually NotReady and identify the AZ, EC2 instance and affected Pods. I check node conditions, events, kubelet/runtime symptoms and AWS instance health to determine whether the problem is disk, memory, network, kubelet or underlying EC2 infrastructure. Then I check which critical Pods were lost and whether Kubernetes can reschedule them. If Pods are Pending, I check available cluster capacity, topology constraints, PDBs, taints, PVCs and Karpenter or managed node-group scaling. I avoid blindly draining a production node because that can remove the last healthy replicas. Once replacement capacity is available, I verify Pod readiness, ALB targets, application metrics and an actual business transaction. Finally I perform an RCA covering both the node failure and any architectural amplification such as poor replica distribution or insufficient capacity headroom."

---

# 60. Key Production Lessons

### Lesson 1

> **EKS worker nodes are replaceable infrastructure.**

### Lesson 2

> **A node failure should not equal an application outage.**

### Lesson 3

> **Multi-AZ distribution is necessary but not sufficient; replicas must actually be distributed.**

### Lesson 4

> **HPA creates Pods; Karpenter or node groups create capacity.**

### Lesson 5

> **PDB protects voluntary disruption, not every unexpected failure.**

### Lesson 6

> **Do not blindly run `kubectl drain` in production.**

### Lesson 7

> **Node recovery must consider IP, storage and scheduling capacity.**

### Lesson 8

> **Stateful workloads require a different recovery strategy from stateless services.**

### Lesson 9

> **Capacity headroom is part of high availability.**

### Lesson 10

> **The RCA should identify both the infrastructure failure and the architectural weakness that amplified the customer impact.**

---

# 61. Final Architecture

```text
                         AWS EKS
                            │
             ┌──────────────┼──────────────┐
             │              │              │
            AZ-1           AZ-2           AZ-3
             │              │              │
        ┌─────────┐    ┌─────────┐    ┌─────────┐
        │ Nodes   │    │ Nodes   │    │ Nodes   │
        └────┬────┘    └────┬────┘    └────┬────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                     Critical Pods
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
      Topology            PDB             Capacity
       Spread                              Headroom
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                       Karpenter
                            │
                            ▼
                     Replacement Nodes
                            │
                            ▼
                     Pod Rescheduling
                            │
                            ▼
                     Readiness Checks
                            │
                            ▼
                       ALB / Traffic
                            │
                            ▼
                    Business Validation
```

---

## Phase 18 Progress

```text
143 CrashLoopBackOff              ✅
144 OOMKilled                     ✅
145 CPU Saturation                ✅
146 Memory Saturation             ✅
147 Disk Full                     ✅
148 ImagePullBackOff              ✅
149 ALB 5xx                       ✅
150 Service Connectivity Failure  ✅
151 DNS Failure                   ✅
152 RDS Connection Exhaustion     ✅
153 Redis Failure                 ✅
154 SQS Backlog                   ✅
155 Kafka Consumer Lag            ✅
156 Node Failure                  ✅
```

**Next: `157 — Availability Zone (AZ) Failure Production Incident`**