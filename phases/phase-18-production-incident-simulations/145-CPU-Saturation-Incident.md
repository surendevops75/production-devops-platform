# 145 — CPU Saturation Production Incident

> **Phase 18 — Incident Management**
> **Project:** Enterprise E-Commerce Platform
> **Primary stack:** AWS EKS + Java 21 + Spring Boot
> **Scope:** Kubernetes/container CPU troubleshooting applicable to **any programming language/runtime**, followed by Java-specific investigation.

---

## 1. Objective

CPU saturation is one of the most common production incidents in Kubernetes.

A service may suddenly experience:

```text
CPU ↑
Latency ↑
5xx ↑
Requests timeout
Pods become unresponsive
HPA scales
Node CPU ↑
```

The important point is:

> **High CPU is a symptom. It is not automatically the root cause.**

CPU can increase because of:

* increased traffic
* inefficient application code
* infinite loops
* excessive garbage collection
* expensive database/query processing
* serialization/deserialization
* encryption/compression
* excessive logging
* retry storms
* thread contention
* CPU throttling
* insufficient CPU limits
* inefficient regular expressions
* background jobs
* malicious/abnormal traffic

The investigation must determine **why CPU increased**.

---

# 2. CPU Saturation vs High CPU

These terms are related but not identical.

### High CPU

A container is using a large percentage of its available CPU.

Example:

```text
CPU usage = 80%
```

This does not necessarily mean the application is unhealthy.

### CPU saturation

The workload is consuming or competing for CPU such that application performance is affected.

Example:

```text
CPU = 100%
Latency ↑
Requests queue ↑
5xx ↑
```

So:

```text
High CPU
    ≠
Automatically an incident

High CPU
    +
Customer impact / performance degradation
    =
Potential CPU saturation incident
```

---

# 3. CPU Is Different From Memory

Memory:

```text
Too much memory
       ↓
OOMKilled
```

CPU:

```text
Too much CPU demand
       ↓
CPU contention/throttling
       ↓
Application slows
```

CPU exhaustion generally doesn't produce an equivalent `CPUKilled` event.

Instead, the application may continue running but become slow.

---

# 4. Production Scenario

Suppose:

```text
order-service
```

normally runs at:

```text
CPU = 35%
RPS = 200
P95 = 150ms
```

Suddenly:

```text
CPU = 98%
RPS = 400
P95 = 2.5s
```

Then:

```text
HTTP 5xx ↑
```

Grafana shows:

```text
CPU
100% |                 /---------
     |              __/
     |           __/
     |        __/
     |_______/
     +--------------------------> time
```

This is a production incident.

---

# 5. First Production Rule

Do **not** immediately conclude:

> "The application needs more CPU."

First determine:

```text
Why did CPU increase?
```

Use:

```text
Alert
 ↓
Confirm impact
 ↓
Measure CPU
 ↓
Check Pod
 ↓
Check container
 ↓
Check node
 ↓
Check throttling
 ↓
Check traffic
 ↓
Check application/runtime
 ↓
Check dependencies
 ↓
Check recent deployment
 ↓
Identify root cause
 ↓
Mitigate
 ↓
Fix
 ↓
Validate
 ↓
RCA
```

---

# 6. Step 1 — Check Pods

```bash
kubectl get pods -n ecommerce
```

Example:

```text
NAME                             READY   STATUS    RESTARTS
order-service-7d89f8f9d7-x2k9m   1/1     Running   0
order-service-7d89f8f9d7-k8p2q   1/1     Running   0
```

Unlike OOMKilled, the Pods may appear perfectly healthy:

```text
Running
Ready
```

This is why CPU incidents can be harder to detect through Kubernetes status alone.

---

# 7. Step 2 — Check CPU Usage

```bash
kubectl top pod -n ecommerce
```

Example:

```text
NAME                             CPU     MEMORY
order-service-7d89f8f9d7-x2k9m   980m    620Mi
order-service-7d89f8f9d7-k8p2q   950m    610Mi
```

If the container has:

```yaml
limits:
  cpu: "1"
```

then:

```text
980m ≈ 0.98 CPU
```

and the container is close to its CPU limit.

---

# 8. CPU Units

Kubernetes CPU is normally expressed in:

```text
CPU cores
millicores
```

Examples:

```text
1000m = 1 CPU
500m  = 0.5 CPU
250m  = 0.25 CPU
100m  = 0.1 CPU
```

Therefore:

```text
750m = 0.75 CPU
```

---

# 9. Requests vs Limits for CPU

Example:

```yaml
resources:
  requests:
    cpu: "500m"

  limits:
    cpu: "1"
```

This means:

```text
Request = 0.5 CPU
Limit   = 1 CPU
```

The request is important for:

```text
scheduling
capacity planning
```

The limit establishes a CPU ceiling.

---

# 10. CPU Throttling

This is one of the most important Kubernetes CPU concepts.

Suppose:

```yaml
limits:
  cpu: "1"
```

The application suddenly needs:

```text
1.8 CPU
```

It cannot continuously consume 1.8 CPU from that container if its configured limit is 1 CPU.

The Linux CPU control mechanism can throttle the container.

Conceptually:

```text
Application demand
       |
       v
    1.8 CPU
       |
       X
 CPU limit = 1 CPU
       |
       v
CPU throttling
```

The application may remain healthy but become slower.

---

# 11. CPU Throttling vs CPU Usage

This is an important distinction.

You might see:

```text
CPU usage = 60%
```

but still have performance problems if the workload is experiencing throttling under its CPU limit.

Therefore investigate:

```text
CPU usage
+
CPU throttling
+
latency
```

not CPU usage alone.

---

# 12. Why CPU Limits Need Careful Design

Suppose:

```yaml
limits:
  cpu: "500m"
```

The application normally needs:

```text
400m
```

But during a burst it needs:

```text
900m
```

The container can become throttled.

The result may be:

```text
Request
 ↓
CPU constrained
 ↓
Processing slows
 ↓
Request remains active longer
 ↓
Concurrency increases
 ↓
Latency increases
```

So CPU limits should be based on measured workload behavior.

---

# 13. Step 3 — Check the Node

Identify the node:

```bash
kubectl get pod <pod-name> \
  -n ecommerce \
  -o wide
```

Then:

```bash
kubectl top node
```

Example:

```text
NAME       CPU(cores)   CPU%
node-a     3.2          80%
node-b     3.8          95%
node-c     2.1          53%
```

Then:

```bash
kubectl describe node node-b
```

Check:

```text
Conditions
Allocated resources
Events
```

---

# 14. Pod CPU vs Node CPU

This distinction is important.

### Pod-level CPU problem

```text
Node
 |
 +-- order-service → 100%
 +-- other Pods    → normal
```

Potential application/service problem.

### Node-level CPU saturation

```text
Node
 |
 +-- Pod A → high
 +-- Pod B → high
 +-- Pod C → high
 +-- Pod D → high
```

Potential:

```text
insufficient node capacity
too many workloads
bad scheduling
DaemonSet overhead
unexpected workload
```

---

# 15. Step 4 — Check Traffic

This is usually one of the first correlations.

Suppose:

```text
09:00
RPS = 200
CPU = 30%

09:15
RPS = 400
CPU = 55%

09:30
RPS = 800
CPU = 95%
```

That tells us CPU is strongly correlated with traffic.

But we still need to determine whether:

```text
800 RPS
```

is expected and within the tested capacity.

---

# 16. CPU per Request

A useful concept is:

```text
CPU per request
```

For example:

```text
200 RPS → 0.4 CPU
400 RPS → 0.8 CPU
800 RPS → 1.6 CPU
```

If CPU scales approximately with traffic, this can indicate normal workload scaling.

If:

```text
200 RPS → 0.4 CPU
400 RPS → 1.5 CPU
```

something changed.

Potential causes:

```text
application regression
database behavior
larger payloads
cache miss
serialization
retry
```

---

# 17. Step 5 — Check HPA

```bash
kubectl get hpa -n ecommerce
```

Example:

```text
NAME            REFERENCE                   TARGETS
order-service   Deployment/order-service    92%/70%
```

Describe:

```bash
kubectl describe hpa order-service -n ecommerce
```

Check:

```text
Current replicas
Desired replicas
Min replicas
Max replicas
CPU target
Events
```

---

# 18. HPA Flow

Our normal architecture is:

```text
Traffic
   ↓
ALB
   ↓
Service
   ↓
Pods
   ↓
CPU increases
   ↓
HPA
   ↓
More Pods
```

For example:

```text
2 Pods
  ↓
CPU 90%
  ↓
HPA
  ↓
4 Pods
  ↓
CPU 50%
```

This is a normal scaling response.

---

# 19. But HPA May Not Solve Everything

Suppose CPU increases because of:

```text
infinite loop
```

HPA may create more Pods:

```text
Pod 1 → CPU 100%
Pod 2 → CPU 100%
Pod 3 → CPU 100%
Pod 4 → CPU 100%
```

The problem gets multiplied.

Similarly, if the bottleneck is:

```text
RDS
```

adding more Pods may increase:

```text
DB connections
DB queries
DB CPU
```

and make the incident worse.

Therefore:

> **Never scale blindly.**

---

# 20. Step 6 — Check CPU Throttling

Prometheus can expose container CPU throttling metrics.

A common metric is:

```promql
container_cpu_cfs_throttled_seconds_total
```

You can investigate the rate:

```promql
rate(
  container_cpu_cfs_throttled_seconds_total[5m]
)
```

The exact metric availability depends on your node/container metrics setup.

Look for:

```text
CPU usage ↑
+
throttling ↑
+
latency ↑
```

That is a strong indication that CPU limits may be contributing to the problem.

---

# 21. CPU Saturation vs CPU Throttling

They are not the same.

### CPU saturation

The workload genuinely needs a lot of CPU.

```text
Demand ↑
```

### CPU throttling

The container is being constrained by its configured CPU limit.

```text
Demand > allowed CPU
```

You can have:

```text
High CPU
without
significant throttling
```

and:

```text
Throttling
with
CPU usage patterns that don't appear to be 100%
```

Therefore check both.

---

# 22. Step 7 — Check Recent Deployment

Run:

```bash
kubectl rollout history deployment/order-service \
  -n ecommerce
```

Check:

```text
Git commit
Image tag
Image digest
ArgoCD sync
Deployment time
```

Example:

```text
10:00
version 1.8.0

10:30
version 1.9.0 deployed

10:35
CPU increases

10:40
latency increases
```

This is a strong signal to investigate version `1.9.0`.

---

# 23. Common Application-Level Causes

CPU can increase because of:

```text
Expensive algorithm
Infinite loop
Large data processing
Serialization
Deserialization
Compression
Encryption
Regex processing
Excessive logging
Garbage collection
Retry loops
Thread contention
Busy polling
Background jobs
```

These concepts apply regardless of programming language.

---

# 24. Example — Infinite Loop

Generic application:

```text
while condition:
    process()
```

If the condition never becomes false:

```text
CPU
 ↑
 ↑
 ↑
100%
```

No amount of:

```text
HPA
```

fixes the underlying defect.

The application needs to be corrected.

---

# 25. Example — Expensive Algorithm

Suppose an API performs:

```text
100,000-item sorting operation
```

for every request.

At:

```text
10 RPS
```

it may be acceptable.

At:

```text
1,000 RPS
```

CPU may become saturated.

Solution may involve:

```text
better algorithm
caching
precomputation
pagination
database-side processing
asynchronous processing
```

rather than simply adding CPU.

---

# 26. Example — Excessive Logging

Suppose a service logs:

```text
10,000 lines/sec
```

CPU may be consumed by:

```text
string formatting
serialization
JSON encoding
I/O processing
```

During an incident, check:

```text
log volume
```

and correlate with:

```text
CPU
```

---

# 27. Example — Retry Storm

Suppose:

```text
payment-service
```

is slow.

`order-service` retries:

```text
Request
 ↓
payment
 ↓
timeout
 ↓
retry
 ↓
timeout
 ↓
retry
 ↓
timeout
```

Now one user request generates multiple operations.

CPU increases:

```text
CPU ↑
Threads ↑
Requests ↑
Latency ↑
```

This is a classic cascading failure pattern.

---

# 28. Example — Database Problem

Suppose PostgreSQL becomes slow.

Application behavior:

```text
DB request
 ↓
wait
 ↓
timeout
 ↓
retry
 ↓
DB request
```

CPU may increase because of:

```text
connection handling
request processing
retry logic
serialization
logging
```

Meanwhile:

```text
RDS CPU ↑
RDS connections ↑
```

The application CPU may be a secondary symptom.

---

# 29. Java/Spring Boot CPU Investigation

Our project uses:

```text
Java 21
Spring Boot
```

Java-specific CPU causes include:

```text
High GC activity
Excessive object allocation
Infinite loops
Thread contention
Lock contention
Serialization
JSON processing
Regex
Database processing
High request concurrency
```

---

# 30. JVM CPU Investigation

Look at:

```text
JVM CPU
GC CPU
threads
thread states
allocation rate
```

Spring Boot/Micrometer can expose useful JVM metrics.

For example:

```text
jvm_threads_live_threads
jvm_threads_peak_threads
jvm_gc_pause_seconds
```

These should be correlated with container CPU and application latency.

---

# 31. Garbage Collection Can Consume CPU

Consider:

```text
Application
    ↓
Creates millions of objects
    ↓
Allocation rate ↑
    ↓
GC runs frequently
    ↓
CPU ↑
    ↓
Application processing ↓
```

Grafana may show:

```text
CPU ↑
GC ↑
Latency ↑
```

This suggests investigating allocation/GC behavior.

---

# 32. Java Thread Investigation

If CPU is high:

```bash
kubectl exec -it <pod> -n ecommerce -- \
  jcmd 1 Thread.print
```

Depending on the container image and process PID.

Look for:

```text
RUNNABLE
BLOCKED
WAITING
```

High numbers of:

```text
RUNNABLE
```

threads can indicate CPU-consuming work.

But thread state alone is not enough; correlate with thread CPU and application behavior.

---

# 33. Thread Dump

A thread dump helps identify:

```text
which code paths are active
```

For example:

```text
Thread-1 → JSON serialization
Thread-2 → database processing
Thread-3 → retry loop
Thread-4 → business calculation
```

This gives the application team evidence about where CPU is being consumed.

---

# 34. Java Flight Recorder

JFR can help identify:

```text
CPU hotspots
allocation
GC
threads
locks
```

For example:

```text
CPU
 |
 +-- JSON serialization 35%
 +-- business logic     25%
 +-- GC                 20%
 +-- database handling  10%
 +-- other              10%
```

Now the team can investigate the biggest contributor.

---

# 35. Node.js CPU Investigation

For Node.js, investigate:

```text
event loop
V8 CPU
worker threads
synchronous/blocking code
JSON processing
GC
```

A common problem is CPU-heavy synchronous code blocking the event loop.

Example:

```text
HTTP request
 ↓
CPU-heavy synchronous operation
 ↓
Event loop blocked
 ↓
Other requests wait
 ↓
Latency ↑
```

---

# 36. Python CPU Investigation

Python applications can experience:

```text
CPU-heavy computation
worker/process multiplication
large serialization
regex
data processing
native library processing
```

Depending on architecture:

```text
Gunicorn workers
Celery workers
background jobs
```

can all contribute to CPU consumption.

---

# 37. Go CPU Investigation

For Go:

```text
goroutines
GC
CPU profiles
mutex contention
busy loops
```

can be investigated.

Go's profiling ecosystem is particularly useful for identifying CPU hotspots.

Again:

```text
Kubernetes investigation
```

comes first.

---

# 38. .NET CPU Investigation

For .NET:

```text
GC
thread pool
CPU-heavy methods
async/thread behavior
serialization
```

can contribute to CPU.

Runtime diagnostics can help identify the hot code path.

---

# 39. Node CPU Saturation

Suppose:

```text
Node
8 CPU
```

and:

```text
Pod A → 2 CPU
Pod B → 2 CPU
Pod C → 2 CPU
Pod D → 2 CPU
```

Node:

```text
8 / 8 CPU
```

Now another workload is scheduled or bursts.

The node can become CPU constrained.

This is why node capacity planning matters.

---

# 40. Kubernetes Requests and Scheduling

Suppose:

```yaml
resources:
  requests:
    cpu: "500m"
```

Kubernetes uses the request when making scheduling decisions.

For example:

```text
Node allocatable = 4 CPU

Pod A request = 1 CPU
Pod B request = 1 CPU
Pod C request = 1 CPU
Pod D request = 1 CPU
```

The node is effectively fully allocated according to requests.

Even if actual CPU usage is lower at the moment, Kubernetes capacity planning must account for requests.

---

# 41. CPU Limits and Burst Behavior

Suppose:

```yaml
requests:
  cpu: "500m"

limits:
  cpu: "2"
```

The application normally requests:

```text
500m
```

but can burst up to:

```text
2 CPU
```

if node capacity and scheduling/runtime behavior allow it.

This can be useful for bursty workloads.

But the correct configuration depends on:

```text
workload
node capacity
SLO
HPA
cost
```

---

# 42. HPA Strategy

For our customer-facing services:

```text
ALB
 ↓
Service
 ↓
HPA
 ↓
Pods
```

HPA can use:

```text
CPU
Memory
custom metrics
external metrics
```

CPU is often a useful signal for CPU-bound workloads.

But it should not automatically be the only scaling signal.

---

# 43. CPU vs RPS

For HTTP services, a useful architecture is often:

```text
CPU
+
RPS
+
Latency
```

For example:

```text
RPS ↑
CPU ↑
Latency ↑
```

indicates workload pressure.

Whereas:

```text
RPS stable
CPU suddenly ↑
```

suggests investigating:

```text
deployment
application regression
dependency behavior
GC
logging
retry
```

---

# 44. CPU Saturation and Downstream Systems

Scaling a service from:

```text
5 Pods
```

to:

```text
20 Pods
```

may increase:

```text
RDS connections
Redis connections
Kafka consumers
external API requests
```

Therefore:

```text
CPU scaling
```

must be coordinated with:

```text
dependency capacity
```

This is especially important for:

```text
order-service
payment-service
inventory-service
```

---

# 45. Correct Production Mitigation

### Case 1 — Traffic spike

```text
Scale horizontally
```

using HPA/KEDA as appropriate.

---

### Case 2 — CPU throttling

Investigate:

```text
CPU limit
```

and right-size based on measured workload.

---

### Case 3 — Bad deployment

```text
Rollback
```

through GitOps.

---

### Case 4 — Application CPU bug

```text
Fix application
```

---

### Case 5 — Retry storm

```text
Reduce retries
+
backoff
+
jitter
+
circuit breaker
```

---

### Case 6 — Node capacity problem

```text
Scale nodes
```

using Karpenter or the configured node autoscaling mechanism.

---

# 46. Example CPU Incident

### 10:00

```text
RPS = 200
CPU = 30%
P95 = 150ms
```

### 10:30

Deployment:

```text
order-service 1.9.0
```

### 10:35

```text
CPU = 70%
P95 = 400ms
```

### 10:40

```text
CPU = 95%
P95 = 2s
```

### 10:42

Alert:

```text
High CPU
```

---

# 47. Investigation

Team checks:

```text
kubectl top pod
Grafana
Prometheus
ELK
Jaeger
ArgoCD
Git
```

Findings:

```text
RPS unchanged
CPU increased immediately after deployment
GC normal
RDS normal
Redis normal
Kafka normal
```

This strongly points toward:

```text
application regression
```

---

# 48. Further Investigation

JFR/thread profiling shows:

```text
CPU hotspot:
JSON transformation logic
```

Version `1.9.0` introduced:

```text
large repeated object transformation
```

The application was doing expensive processing for every request.

---

# 49. Mitigation

Since customers are affected:

```text
GitOps
   ↓
Revert 1.9.0
   ↓
ArgoCD
   ↓
1.8.0
```

CPU returns:

```text
95%
 ↓
35%
```

Latency:

```text
2s
 ↓
150ms
```

---

# 50. Permanent Fix

Development changes:

```text
Remove repeated transformation
+
optimize algorithm
+
add regression test
```

Then:

```text
Build
 ↓
Unit tests
 ↓
SonarQube
 ↓
Trivy
 ↓
Docker
 ↓
ECR
 ↓
GitOps
 ↓
Staging
 ↓
Load test
 ↓
Canary
 ↓
Production
```

---

# 51. CPU Saturation Alert

Example:

```yaml
- alert: ContainerHighCPU
  expr: |
    (
      rate(container_cpu_usage_seconds_total[5m])
      /
      container_spec_cpu_quota
    ) > 0.90
  for: 10m
  labels:
    severity: warning
```

The exact PromQL must match how CPU limits are exposed by the cluster's metrics stack.

For production, prefer a tested recording-rule/query rather than copying a generic expression blindly.

---

# 52. Better Alerting

Don't alert only:

```text
CPU > 80%
```

Also consider:

```text
CPU
+
latency
+
error rate
+
throttling
```

For example:

```text
High CPU
+
P95 latency high
=
more actionable
```

Because a service can legitimately use:

```text
80% CPU
```

without any customer impact.

---

# 53. CPU Incident Runbook

Create:

```text
runbooks/
└── cpu-saturation.md
```

Example:

```markdown
# CPU Saturation Runbook

## Symptoms

- High CPU
- Increased latency
- Increased errors
- CPU throttling
- HPA scaling

## Immediate Checks

kubectl top pod -n ecommerce
kubectl top node
kubectl get hpa -n ecommerce
kubectl describe hpa <name> -n ecommerce

## Investigate

- RPS
- latency
- error rate
- CPU throttling
- recent deployments
- application CPU
- GC
- threads
- retries
- database
- Redis
- Kafka
- external APIs

## Mitigation

- scale horizontally
- rollback
- increase capacity
- reduce workload
- protect dependencies

## Validation

- CPU stable
- latency normal
- errors normal
- dependencies healthy

## RCA

Document actual root cause.
```

---

# 54. CPU Saturation Decision Tree

```text
                    High CPU
                       |
                       v
             Is customer impact present?
                  /          \
                NO            YES
                |              |
        Continue observe       v
                         Check throttling
                               |
                    +----------+----------+
                    |                     |
                  YES                     NO
                    |                     |
             Review CPU limits      Check traffic
                    |                     |
                    |             +-------+-------+
                    |             |               |
                    |            ↑               =
                    |          Traffic        Traffic stable
                    |             |               |
                    |             v               v
                    |         Scale/            Check
                    |         optimize          deployment
                    |                             |
                    |                     +-------+-------+
                    |                     |               |
                    |                  Recent          No recent
                    |                  change            change
                    |                     |               |
                    |                     v               v
                    |                  Rollback       Runtime/app
                    |                                  investigation
                    |                                       |
                    +-------------------+-------------------+
                                        |
                                        v
                                  Check dependencies
                                        |
                                        v
                                  Root cause
                                        |
                                        v
                                   Mitigate
                                        |
                                        v
                                    Validate
                                        |
                                        v
                                      RCA
```

---

# 55. Production Checklist

## Kubernetes

* [ ] Pod CPU checked
* [ ] Node CPU checked
* [ ] CPU requests checked
* [ ] CPU limits checked
* [ ] HPA checked
* [ ] Karpenter/node scaling checked
* [ ] CPU throttling checked

## Application

* [ ] RPS checked
* [ ] Latency checked
* [ ] Error rate checked
* [ ] Recent deployment checked
* [ ] Runtime metrics checked
* [ ] Threads/workers checked
* [ ] GC checked where applicable
* [ ] CPU hotspot investigated

## Dependencies

* [ ] RDS checked
* [ ] Redis checked
* [ ] Kafka checked
* [ ] SQS checked
* [ ] External APIs checked

## Mitigation

* [ ] Scale if appropriate
* [ ] Rollback if deployment-related
* [ ] Protect dependencies
* [ ] Avoid blind scaling

## Validation

* [ ] CPU recovered
* [ ] Latency recovered
* [ ] 5xx recovered
* [ ] HPA stable
* [ ] Dependencies healthy
* [ ] Business transaction successful

## RCA

* [ ] Timeline
* [ ] Root cause
* [ ] Impact
* [ ] Mitigation
* [ ] Permanent fix
* [ ] Prevention

---

# 56. Final Architecture

```text
                    Internet
                       |
                    Route53
                       |
                   CloudFront
                       |
                      WAF
                       |
                      ALB
                       |
                 Kubernetes Service
                       |
                +------+------+
                |             |
              Pod 1         Pod 2
                |             |
             CPU ↑          CPU ↑
                \             /
                 +-----+-----+
                       |
                      HPA
                       |
                More replicas
                       |
                   Karpenter
                       |
                  More nodes
```

But remember the complete system:

```text
Traffic
   ↓
Application
   ↓
CPU
   ↓
HPA
   ↓
Pods
   ↓
Nodes
   ↓
Dependencies
   ↓
RDS / Redis / Kafka / SQS
```

Every layer can contribute to the incident.

---

# 57. Final Mental Model

For any language:

```text
High CPU
   ↓
Is it real saturation?
   ↓
Pod or Node?
   ↓
CPU throttling?
   ↓
Traffic increased?
   ↓
Recent deployment?
   ↓
Application CPU hotspot?
   ↓
Runtime issue?
   ↓
Dependency issue?
   ↓
Choose mitigation
   ↓
Validate
   ↓
RCA
```

For our Java application:

```text
CPU ↑
 ↓
Spring Boot metrics
 ↓
JVM threads
 ↓
GC
 ↓
JFR/thread dump
 ↓
identify CPU hotspot
```

The key rule is:

> **Do not solve CPU saturation by automatically adding CPU. First determine whether the workload, application, runtime, deployment, node capacity, or dependency is responsible.**

---

## Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ⏭️
147 — Disk Full
148 — ImagePullBackOff
149 — ALB 5xx
150 — Service Connectivity Failure
151 — DNS Failure
152 — RDS Connection Exhaustion
153 — Redis Failure
154 — SQS Backlog
155 — Kafka Consumer Lag
156 — Node Failure
157 — AZ Failure
158 — ArgoCD Drift
159 — Bad Production Deployment
160 — Security Incident
```

**Next: `146 — Memory Saturation Production Incident`.**
