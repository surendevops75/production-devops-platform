# 102 — Kubernetes Resource Management

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Region:** `ap-south-1`
> **Scope:** Production-grade CPU, memory, and ephemeral-storage management for 21 microservices

---

## 1. Purpose

Kubernetes resource management determines:

* how much CPU a Pod is guaranteed to receive
* how much memory a Pod can consume
* where Kubernetes schedules the Pod
* when a Pod becomes `Pending`
* when a container gets throttled
* when a container gets `OOMKilled`
* how HPA calculates utilization
* how Karpenter decides what node capacity is required
* how efficiently EC2 nodes are packed
* how much unused capacity the cluster carries

Our production flow is:

```text
Application
    ↓
Container
    ↓
Pod resources
    ↓
Kubernetes Scheduler
    ↓
Node selection
    ↓
Karpenter / Node Group
    ↓
EC2 capacity
```

Resource management therefore affects **application reliability, scaling, scheduling, and AWS cost simultaneously**.

---

# 2. CPU and Memory Are Different

Kubernetes primarily manages two compute resources:

```text
CPU
Memory
```

They behave differently.

### CPU

CPU is generally compressible.

If a container reaches its CPU limit, the Linux cgroup can throttle it.

Example:

```text
CPU limit = 1 CPU

Application wants = 1.5 CPU

Result:

Application does not normally get killed.
It can be CPU throttled.
```

### Memory

Memory is not compressible.

If a container exceeds its memory limit, the container can be killed.

Example:

```text
Memory limit = 512Mi

Application needs = 700Mi

Result:

Container may be OOMKilled.
```

This distinction is extremely important during production incidents.

---

# 3. Kubernetes Resource Terminology

We use:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

There are four values:

| Resource | Request                | Limit          |
| -------- | ---------------------- | -------------- |
| CPU      | scheduling reservation | maximum CPU    |
| Memory   | scheduling reservation | maximum memory |

---

# 4. CPU Units

Kubernetes CPU is measured in cores.

```text
1 CPU = 1 core
```

Examples:

```text
1000m = 1 CPU
500m  = 0.5 CPU
250m  = 0.25 CPU
100m  = 0.1 CPU
50m   = 0.05 CPU
```

So:

```yaml
cpu: "500m"
```

means:

```text
0.5 CPU
```

---

# 5. Memory Units

Memory normally uses binary units:

```text
Mi
Gi
```

Examples:

```text
128Mi
256Mi
512Mi
1Gi
2Gi
4Gi
```

For example:

```yaml
memory: "1Gi"
```

means approximately:

```text
1 GiB
```

---

# 6. Requests vs Limits

This is one of the most important Kubernetes concepts.

## Request

The request tells Kubernetes:

> "This Pod needs at least this much capacity for scheduling purposes."

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
```

If a node has:

```text
Available CPU: 200m
Available Memory: 300Mi
```

the Pod cannot be scheduled there.

---

# 7. Limits

The limit tells Kubernetes the maximum resource boundary for the container.

Example:

```yaml
resources:
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

CPU can be throttled above the limit.

Memory can result in OOM termination when the container exceeds its allowed memory.

---

# 8. Why Requests Are Extremely Important

Consider a node:

```text
CPU capacity: 4 CPU
Memory: 16Gi
```

Suppose we have:

```text
Pod A:
request = 1 CPU

Pod B:
request = 1 CPU

Pod C:
request = 1 CPU
```

Kubernetes sees:

```text
Requested CPU = 3 CPU
```

Therefore approximately:

```text
1 CPU remaining
```

for scheduling purposes.

It does **not** simply look at current instantaneous CPU usage.

This is why incorrect requests can cause:

```text
Pending Pods
poor bin packing
unnecessary nodes
higher AWS cost
```

---

# 9. Example — Incorrectly Large Requests

Suppose our application actually uses:

```text
CPU: 150m
Memory: 400Mi
```

but we configure:

```yaml
requests:
  cpu: "2"
  memory: "4Gi"
```

The scheduler sees:

```text
2 CPU
4Gi
```

even if the application is currently using only:

```text
150m
400Mi
```

This can cause inefficient scheduling.

---

# 10. Example — Requests Too Small

Now suppose:

```yaml
requests:
  cpu: "50m"
  memory: "128Mi"
```

but the application normally needs:

```text
CPU: 400m
Memory: 800Mi
```

Problems may include:

```text
CPU contention
memory pressure
OOMKilled
poor latency
frequent scaling
unstable performance
```

Therefore we need measured values.

---

# 11. Production Resource Tuning

Do not choose resources randomly.

Our process should be:

```text
Deploy
   ↓
Generate realistic traffic
   ↓
Measure CPU
   ↓
Measure memory
   ↓
Measure latency/errors
   ↓
Review peak usage
   ↓
Set requests
   ↓
Set limits
   ↓
Load test
   ↓
Tune again
```

---

# 12. Example — Order Service

Suppose production measurements show:

```text
Normal CPU:
150m–300m

Peak CPU:
700m

Normal memory:
400Mi–550Mi

Peak memory:
750Mi
```

A starting configuration might be:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

This is only a starting point.

We validate it through load testing and production telemetry.

---

# 13. Order Service Helm Values

In:

```text
helm/order-service/values.yaml
```

we can define:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

Then:

```text
DEV
QA
STAGING
PROD
```

can override resources through environment-specific values.

Example:

```yaml
# environments/prod/order-service/values.yaml

resources:
  requests:
    cpu: "500m"
    memory: "1Gi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

---

# 14. Helm Deployment Template

Our Deployment should contain:

```yaml
resources:
  requests:
    cpu: {{ .Values.resources.requests.cpu | quote }}
    memory: {{ .Values.resources.requests.memory | quote }}

  limits:
    cpu: {{ .Values.resources.limits.cpu | quote }}
    memory: {{ .Values.resources.limits.memory | quote }}
```

This lets the same Helm chart work across environments.

---

# 15. CPU Request and HPA

This becomes particularly important when we use HPA.

Suppose:

```yaml
requests:
  cpu: "500m"
```

and the container is currently using:

```text
250m
```

CPU utilization is approximately:

```text
250 / 500 = 50%
```

If HPA target is:

```yaml
targetCPUUtilizationPercentage: 70
```

the application is below the target.

But if request is incorrectly set to:

```yaml
requests:
  cpu: "2"
```

while usage remains:

```text
250m
```

then utilization becomes:

```text
250 / 2000 = 12.5%
```

The HPA may therefore behave very differently.

**Incorrect requests can therefore distort autoscaling behavior.**

---

# 16. Memory and HPA

Memory-based HPA can also use resource requests.

Example:

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

Again, incorrect memory requests can influence the calculated utilization.

For many production applications, CPU-based HPA is easier to reason about, while memory should still be monitored carefully.

---

# 17. Kubernetes QoS Classes

Kubernetes assigns Pods a Quality of Service class.

There are three primary classes:

```text
Guaranteed
Burstable
BestEffort
```

---

# 18. Guaranteed QoS

A Pod generally gets Guaranteed QoS when every container has CPU and memory requests and limits set, and requests equal limits for those resources.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"

  limits:
    cpu: "500m"
    memory: "1Gi"
```

This provides predictable resource boundaries.

But it does not automatically mean:

```text
best performance
best architecture
best cost
```

It is simply a QoS classification.

---

# 19. Burstable QoS

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

Requests and limits differ.

This is common for application workloads.

Our production services will often use this model.

---

# 20. BestEffort QoS

If containers have no CPU or memory requests/limits:

```yaml
resources: {}
```

the Pod can be BestEffort.

This is generally undesirable for production workloads because the scheduler has little information about the workload's resource requirements.

---

# 21. Check QoS

Run:

```bash
kubectl get pod order-service-xxxxx \
  -n ecommerce-prod \
  -o jsonpath='{.status.qosClass}'
```

Example:

```text
Burstable
```

---

# 22. CPU Throttling

Suppose:

```yaml
limits:
  cpu: "500m"
```

The application tries to use:

```text
900m
```

The Linux cgroup can throttle the container.

Symptoms may include:

```text
increased latency
slow requests
higher response time
CPU throttling metrics
```

Importantly:

```text
High CPU usage != automatically CPU problem
```

We must determine whether the application is actually CPU constrained.

---

# 23. CPU Troubleshooting

First:

```bash
kubectl top pod order-service-xxxxx \
  -n ecommerce-prod
```

Then:

```bash
kubectl describe pod order-service-xxxxx \
  -n ecommerce-prod
```

Check:

```text
Requests
Limits
Events
Container state
Restart count
```

Prometheus can provide deeper historical analysis.

---

# 24. Memory Pressure

Memory is more dangerous because exceeding available memory can terminate workloads.

Example:

```yaml
limits:
  memory: "1Gi"
```

Application gradually grows:

```text
500Mi
600Mi
700Mi
800Mi
900Mi
1024Mi
```

Eventually:

```text
OOMKilled
```

may occur.

---

# 25. Check OOMKilled

Run:

```bash
kubectl describe pod order-service-xxxxx \
  -n ecommerce-prod
```

Look for:

```text
Reason: OOMKilled
```

Or:

```bash
kubectl get pod order-service-xxxxx \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

Expected:

```text
OOMKilled
```

---

# 26. Java/Spring Boot Memory

Our application uses Java.

Therefore Kubernetes memory limits must be considered together with JVM memory.

Example:

```yaml
resources:
  requests:
    memory: "1Gi"

  limits:
    memory: "2Gi"
```

We should not blindly assume:

```text
-Xmx2g
```

is safe.

The Java process also consumes memory outside the Java heap.

Memory can include:

```text
Java heap
Metaspace
thread stacks
direct buffers
JIT/code cache
native allocations
JVM overhead
```

---

# 27. Container-Aware JVM Configuration

For example:

```bash
JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70"
```

This allows the JVM to calculate heap sizing relative to the container memory limit.

The exact percentage must be validated through testing.

For a:

```text
2Gi
```

container memory limit, 70% is approximately:

```text
1.4Gi
```

for the relevant JVM heap sizing calculation, while the remaining memory is available for non-heap/native components.

Do not treat this as a universal production value.

---

# 28. Ephemeral Storage

CPU and memory are not the only resources.

Pods can consume node-local ephemeral storage through:

```text
container writable layer
emptyDir
temporary files
logs
```

Example:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"

  limits:
    ephemeral-storage: "2Gi"
```

This can help Kubernetes reason about ephemeral-storage usage.

---

# 29. Why Ephemeral Storage Matters

Suppose an application generates:

```text
large temporary files
```

or excessive:

```text
application logs
```

The node's disk can fill.

Symptoms:

```text
DiskPressure
Pod eviction
container failures
log collection problems
node instability
```

This connects directly to our future incident:

```text
147 — Disk Full
```

---

# 30. Node Capacity vs Allocatable

A node may show:

```text
Capacity
Allocatable
```

These are not always identical.

Example:

```text
Capacity:
CPU      4
Memory   16Gi
```

But:

```text
Allocatable:
CPU      3.8
Memory   14.5Gi
```

Why?

Because some resources are reserved for:

```text
kubelet
container runtime
OS
system DaemonSets
EKS components
```

---

# 31. Inspect Node Resources

Run:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Capacity
Allocatable
Allocated resources
```

You can also use:

```bash
kubectl top nodes
```

Example:

```text
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-a        850m         21%    5Gi             31%
```

---

# 32. Allocated Resources

`kubectl describe node` also gives an important section:

```text
Allocated resources
```

Example:

```text
Resource           Requests       Limits
CPU                2500m (65%)    5000m (130%)
Memory             8Gi (55%)      12Gi (82%)
```

Notice:

```text
Limits can exceed node capacity.
```

This is possible because limits are not necessarily reserved capacity.

Requests are the key scheduling reservation.

---

# 33. Bin Packing

Suppose a node has:

```text
4 CPU
16Gi memory
```

We have Pods:

```text
Pod A = 1 CPU / 2Gi
Pod B = 1 CPU / 2Gi
Pod C = 500m / 1Gi
Pod D = 500m / 1Gi
```

Total requests:

```text
CPU    = 3 CPU
Memory = 6Gi
```

These Pods can potentially fit on one node.

Good resource sizing improves:

```text
bin packing
node utilization
AWS cost
```

---

# 34. Over-Requesting

Example:

```text
Actual CPU = 100m
Request    = 1 CPU
```

Across:

```text
20 replicas
```

we reserve:

```text
20 CPU
```

even though actual average consumption might be:

```text
2 CPU
```

This can cause Karpenter to provision unnecessary capacity.

---

# 35. Under-Requesting

Opposite problem:

```text
Actual CPU = 800m
Request    = 100m
```

Kubernetes thinks the workload needs very little capacity.

This can cause:

```text
node contention
poor latency
CPU pressure
unexpected performance degradation
```

---

# 36. ResourceQuota

At namespace level we can define a quota.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ecommerce-prod-quota
  namespace: ecommerce-prod
spec:
  hard:
    requests.cpu: "40"
    requests.memory: "80Gi"
    limits.cpu: "80"
    limits.memory: "160Gi"
    pods: "200"
```

This protects the namespace from uncontrolled resource consumption.

---

# 37. LimitRange

We can also define defaults.

Example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: ecommerce-defaults
  namespace: ecommerce-prod
spec:
  limits:
    - type: Container
      default:
        cpu: "1"
        memory: "1Gi"
      defaultRequest:
        cpu: "250m"
        memory: "512Mi"
```

This provides defaults when developers don't specify resources.

However, production teams should still explicitly define resources for important workloads rather than relying blindly on defaults.

---

# 38. ResourceQuota vs LimitRange

| Feature       | Purpose                                |
| ------------- | -------------------------------------- |
| ResourceQuota | namespace-wide maximum                 |
| LimitRange    | container/Pod defaults and constraints |

Think:

```text
LimitRange
    ↓
"What should individual workloads default to?"

ResourceQuota
    ↓
"How much can this namespace consume?"
```

---

# 39. Karpenter and Resource Requests

This is extremely important for our architecture.

Suppose a Pod is:

```text
Pending
```

because no existing node can satisfy:

```text
CPU request = 2 CPU
Memory request = 4Gi
```

Karpenter observes the unschedulable Pod.

It evaluates:

```text
Pod requests
NodePool requirements
EC2 instance availability
AZ
architecture
capacity type
```

Then provisions suitable capacity.

Therefore:

```text
Bad requests
      ↓
Bad scheduling
      ↓
Bad Karpenter decisions
      ↓
Bad cost/performance
```

---

# 40. HPA + Karpenter + Resources

Our scaling architecture is:

```text
Traffic increases
      ↓
CPU / custom metric increases
      ↓
HPA increases replicas
      ↓
New Pods
      ↓
Existing nodes lack capacity
      ↓
Pods become Pending
      ↓
Karpenter observes Pending Pods
      ↓
Karpenter provisions EC2
      ↓
Scheduler places Pods
```

Resource requests are the bridge between these systems.

---

# 41. Resource Classes for Our Platform

We can establish initial workload classes.

### Small

```yaml
requests:
  cpu: "100m"
  memory: "256Mi"

limits:
  cpu: "500m"
  memory: "512Mi"
```

### Medium

```yaml
requests:
  cpu: "250m"
  memory: "512Mi"

limits:
  cpu: "1"
  memory: "1Gi"
```

### Large

```yaml
requests:
  cpu: "500m"
  memory: "1Gi"

limits:
  cpu: "2"
  memory: "2Gi"
```

These are **starting templates**, not final production values.

Actual values must come from workload measurement.

---

# 42. 21 Microservices Resource Strategy

We should not manually invent completely different resource settings for every service.

Instead:

```text
Standard Helm resource structure
        ↓
Service-specific values
        ↓
Environment-specific overrides
```

Example:

```text
order-service
    medium

catalog-service
    medium

search-service
    large

recommendation-service
    large

notification-service
    small
```

The actual classification will be validated during Phase 16 performance testing.

---

# 43. Helm Resource Profiles

We can eventually create:

```yaml
resourceProfiles:
  small:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

  medium:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "1"
      memory: "1Gi"

  large:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "2Gi"
```

But the application chart should still allow explicit overrides.

---

# 44. Resource Management and Multi-AZ

Our previous chapter implemented:

```text
Multi-AZ placement
```

Resource management connects directly to it.

Suppose:

```text
3 replicas
```

and each requires:

```text
500m CPU
1Gi memory
```

We need enough capacity across AZs to maintain availability.

If AZ-A fails:

```text
Replica A -> lost
```

the remaining AZs must have enough capacity to run the replacement.

Therefore:

```text
resource requests
+
topology spread
+
capacity headroom
```

must be designed together.

---

# 45. Resource Management and Spot

Our previous Spot architecture also depends on resource requests.

Suppose:

```text
Pending Pod:
2 CPU
4Gi
```

Karpenter may choose a compatible Spot instance.

If requests are inaccurate:

```text
too large
```

we waste capacity.

If too small:

```text
node becomes crowded
```

Therefore Spot optimization is impossible without good resource sizing.

---

# 46. Resource Management and PDB

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

If we have:

```text
3 replicas
```

and:

```text
minAvailable = 2
```

Kubernetes voluntary disruption should preserve at least two available replicas.

But remember:

> A PDB does not protect against an actual AZ failure or sudden node crash.

---

# 47. Resource Management and Rolling Deployments

Our rolling deployment uses:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Suppose:

```text
Current:
3 Pods
```

During deployment:

```text
3 old
+
1 new
```

may temporarily require:

```text
4 Pods
```

Therefore we need enough capacity for the surge.

Otherwise:

```text
new Pod Pending
```

and deployment progress may stall.

---

# 48. Resource Management and Blue-Green

Blue-Green is even more capacity-intensive.

Suppose:

```text
Blue:
3 replicas
```

During deployment:

```text
Blue: 3
Green: 3
```

Total:

```text
6 replicas
```

If each requires:

```text
500m CPU
1Gi
```

additional capacity may be:

```text
1.5 CPU
3Gi memory
```

Therefore deployment strategy must be included in capacity planning.

---

# 49. Resource Management and Canary

Canary also changes resource demand.

Example:

```text
Stable:
10 replicas

Canary:
2 replicas
```

During rollout:

```text
12 replicas
```

may temporarily run.

This must be accounted for in:

```text
HPA
Karpenter
PDB
node capacity
```

---

# 50. Measuring Actual Usage

For basic current usage:

```bash
kubectl top pods -n ecommerce-prod
```

Sort by CPU:

```bash
kubectl top pods -n ecommerce-prod --sort-by=cpu
```

Sort by memory:

```bash
kubectl top pods -n ecommerce-prod --sort-by=memory
```

Nodes:

```bash
kubectl top nodes
```

---

# 51. Prometheus Gives Historical Data

`kubectl top` is useful for immediate observation.

Prometheus is better for:

```text
historical usage
peak usage
percentiles
trend analysis
capacity planning
alerts
```

Examples of useful metrics include:

```text
container_cpu_usage_seconds_total
container_memory_working_set_bytes
kube_pod_container_resource_requests
kube_pod_container_resource_limits
```

Exact metric names can vary depending on the installed exporters/version.

---

# 52. Resource Utilization Dashboard

Our Grafana platform should eventually show:

```text
Cluster CPU utilization
Cluster memory utilization
Node CPU
Node memory
Pod CPU
Pod memory
Requests vs usage
Limits vs usage
CPU throttling
OOMKilled
Evictions
Pending Pods
```

This becomes part of:

```text
118 → Prometheus Production Implementation
119 → Grafana Production Dashboards
```

---

# 53. Finding a Pending Pod

Run:

```bash
kubectl get pods -n ecommerce-prod
```

Example:

```text
order-service-abc   Pending
```

Then:

```bash
kubectl describe pod order-service-abc \
  -n ecommerce-prod
```

Look at:

```text
Events:
  FailedScheduling
```

Example:

```text
0/6 nodes are available:
Insufficient cpu
```

---

# 54. Pending Due to Memory

You may see:

```text
0/6 nodes are available:
Insufficient memory
```

Possible causes:

```text
requests too large
nodes too small
cluster capacity exhausted
other Pods consuming capacity
Karpenter constraints
AZ capacity constraints
```

Do not immediately increase node size.

First understand the scheduling decision.

---

# 55. Pending Due to Resources — Investigation

Run:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

Then:

```bash
kubectl describe nodes
```

Check:

```text
requests
allocatable
taints
labels
conditions
```

Then inspect Karpenter:

```bash
kubectl get nodeclaims
kubectl describe nodeclaim <nodeclaim>
```

Exact Karpenter resources/commands depend on the installed Karpenter version.

---

# 56. OOMKilled Investigation

Start:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

Then:

```bash
kubectl logs <pod> \
  -n ecommerce-prod \
  --previous
```

Check:

```text
lastState
reason
exit code
restart count
memory usage
```

Then inspect historical metrics.

Questions:

```text
Was traffic increased?
Was there a memory leak?
Did deployment change?
Did JVM configuration change?
Did payload size increase?
Did concurrency increase?
Did cache usage increase?
```

---

# 57. CPU Saturation Investigation

Check:

```bash
kubectl top pod -n ecommerce-prod
```

Then:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

Check:

```text
CPU request
CPU limit
restarts
HPA
```

Then inspect Prometheus/Grafana:

```text
CPU usage
CPU throttling
request rate
latency
error rate
```

The goal is to determine:

```text
application workload
vs
resource limit
vs
node contention
vs
insufficient replicas
```

---

# 58. Memory Saturation Investigation

Check:

```bash
kubectl top pod -n ecommerce-prod
kubectl top nodes
```

Then:

```bash
kubectl get pods -n ecommerce-prod -o wide
```

Determine whether memory pressure is:

```text
one container
one node
one workload
entire cluster
```

This distinction is important.

---

# 59. Resource Management Failure Example

Production incident:

```text
Order API latency increased.
```

Initial observation:

```text
CPU = 95%
```

Engineer increases:

```text
CPU limit
```

But latency remains.

Further investigation shows:

```text
HPA = 3 replicas
CPU request = 2 CPU
actual usage = 1.5 CPU
```

HPA sees:

```text
75%
```

Target:

```text
70%
```

Scaling is slow.

After reviewing historical metrics, request is tuned to a realistic value and HPA behavior becomes more responsive.

The lesson:

> Resource configuration must be analyzed together with autoscaling, not independently.

---

# 60. Resource Management and Node Cost

Suppose our workload needs:

```text
20 CPU
```

If requests are inflated to:

```text
40 CPU
```

Kubernetes/Karpenter may provision substantially more capacity than required.

That impacts:

```text
EC2 cost
Spot usage
On-Demand usage
scaling time
node count
bin packing
```

Resource optimization is therefore also **cost optimization**.

---

# 61. Production Resource Policy

For our project, every production application Pod should have:

```text
CPU request
CPU limit
Memory request
Memory limit
```

and, where relevant:

```text
ephemeral-storage request
ephemeral-storage limit
```

We should also define:

```text
HPA
PDB
probes
topology constraints
```

as appropriate.

---

# 62. Resource Configuration Example

Our Order Service production values could initially look like:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
    ephemeral-storage: "256Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
    ephemeral-storage: "1Gi"
```

Again, these values must be validated through performance testing.

---

# 63. Complete Production Relationship

Our architecture is now:

```text
                 Traffic
                    │
                    ▼
                  ALB
                    │
                    ▼
                 Service
                    │
                    ▼
                   Pods
                    │
          ┌─────────┴─────────┐
          │                   │
     Resource Requests    Resource Limits
          │                   │
          ▼                   ▼
      Scheduler          cgroups/runtime
          │                   │
          ▼                   │
      Node Capacity           │
          │                   │
          ▼                   │
      Karpenter               │
          │                   │
          ▼                   │
       EC2 Nodes ◄────────────┘
```

And scaling:

```text
Metrics
   ↓
HPA
   ↓
Replica count
   ↓
Pending Pods
   ↓
Karpenter
   ↓
EC2 capacity
   ↓
Scheduler
   ↓
Pods
```

---

# 64. Production Checklist

Before calling resource management production-ready:

### Application

```text
[ ] CPU request defined
[ ] CPU limit defined
[ ] Memory request defined
[ ] Memory limit defined
[ ] Ephemeral storage reviewed
[ ] JVM/container memory reviewed
```

### Kubernetes

```text
[ ] QoS reviewed
[ ] ResourceQuota considered
[ ] LimitRange defined where appropriate
[ ] HPA configured
[ ] PDB configured
[ ] Topology constraints configured
```

### EKS

```text
[ ] Node allocatable reviewed
[ ] System overhead understood
[ ] Karpenter configured
[ ] Node pools sized correctly
[ ] Multi-AZ capacity available
[ ] Spot interruption strategy tested
```

### Observability

```text
[ ] CPU metrics
[ ] Memory metrics
[ ] CPU throttling
[ ] OOMKilled alerts
[ ] Pending Pod alerts
[ ] Node pressure alerts
[ ] Resource dashboards
```

### Performance

```text
[ ] Load testing completed
[ ] Peak usage measured
[ ] Requests tuned
[ ] Limits tuned
[ ] HPA behavior tested
[ ] Rolling deployment surge tested
[ ] Failure scenarios tested
```

---

# 65. Golden Commands

### Pod resources

```bash
kubectl describe pod <pod> -n <namespace>
```

### Current Pod usage

```bash
kubectl top pods -n <namespace>
```

### Current node usage

```bash
kubectl top nodes
```

### Node capacity

```bash
kubectl describe node <node>
```

### Pod scheduling events

```bash
kubectl get events \
  -n <namespace> \
  --sort-by='.lastTimestamp'
```

### Check QoS

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.status.qosClass}'
```

### Check container termination reason

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

### Check resource configuration

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.containers[*].resources}'
```

---

# 66. Production Mental Model

When a Pod has a resource problem, think in this order:

```text
1. What does the Pod request?
          ↓
2. What does the Pod actually use?
          ↓
3. What is the Pod limit?
          ↓
4. What is the node allocatable capacity?
          ↓
5. Is the node under pressure?
          ↓
6. Is HPA scaling correctly?
          ↓
7. Is Karpenter provisioning correctly?
          ↓
8. Are topology/AZ constraints restricting placement?
          ↓
9. Is the application itself inefficient?
          ↓
10. What does historical Prometheus data show?
```

Do not simply increase CPU or memory without understanding the cause.

---

# 67. Final Architecture

Our resource-management architecture is now:

```text
                    ┌─────────────────────┐
                    │      Traffic        │
                    └──────────┬──────────┘
                               │
                               ▼
                         ┌───────────┐
                         │    ALB    │
                         └─────┬─────┘
                               │
                               ▼
                         ┌───────────┐
                         │  Service  │
                         └─────┬─────┘
                               │
             ┌─────────────────┼─────────────────┐
             ▼                 ▼                 ▼
          Pod AZ-A          Pod AZ-B          Pod AZ-C
             │                 │                 │
             └─────────────────┼─────────────────┘
                               │
                     Resource Requests
                               │
                               ▼
                         Kubernetes
                         Scheduler
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
          Existing Nodes                 Pending Pods
                                              │
                                              ▼
                                          Karpenter
                                              │
                               ┌──────────────┼──────────────┐
                               ▼              ▼              ▼
                            Spot EC2      Spot EC2      On-Demand
                               │              │              │
                               └──────────────┴──────────────┘
```

The key principle for our project is:

> **Resource requests are not just Kubernetes configuration. They are the contract connecting the application, scheduler, HPA, Karpenter, node capacity, Multi-AZ availability, deployment strategy, and AWS cost.**

---

## Next chapter

**103 — HPA / VPA / KEDA Production Implementation**

We will connect resource management to actual autoscaling:

```text
CPU/Memory
   ↓
HPA
   ↓
Pod replicas
   ↓
Pending Pods
   ↓
Karpenter
   ↓
EC2

SQS/Kafka/RabbitMQ backlog
   ↓
KEDA
   ↓
Pod replicas
   ↓
Karpenter
   ↓
EC2
```

and implement the scaling architecture for the 21-microservice platform.
