# 144 — OOMKilled Production Incident

> **Phase 18 — Incident Management**
> **Project:** Enterprise E-Commerce Platform
> **Primary application stack:** Java 21 + Spring Boot
> **Scope:** Kubernetes/container-level troubleshooting applicable to **any programming language**, followed by Java-specific investigation.

---

## 1. Objective

In production, a DevOps Engineer should not think:

> "`OOMKilled` is a Java problem."

`OOMKilled` is primarily a **Linux/container/Kubernetes memory problem**.

The application could be:

```text
Java
Node.js
Python
Go
.NET
PHP
Ruby
Rust
```

The initial Kubernetes troubleshooting process is largely the same.

The runtime-specific investigation comes afterward.

This chapter therefore follows:

```text
Kubernetes / Linux
        ↓
Language-independent investigation
        ↓
Application/runtime investigation
        ↓
Java-specific investigation
        ↓
Mitigation
        ↓
Permanent fix
        ↓
RCA
```

---

# 2. What Is OOMKilled?

`OOM` means:

> **Out Of Memory**

`OOMKilled` means a process/container was killed because it exceeded an available memory boundary enforced by the Linux/container environment.

In Kubernetes, you may see:

```text
Reason: OOMKilled
Exit Code: 137
```

Example:

```text
order-service-7d89f8f9d7-x2k9m

State:
  Terminated:
    Reason: OOMKilled
    Exit Code: 137
```

---

# 3. Kubernetes Does Not Care About the Programming Language

Consider:

```text
                    Kubernetes
                         |
                    Container
                         |
             +-----------+-----------+
             |           |           |
            Java       Node.js     Python
             |           |           |
            JVM       V8 runtime   Python
```

Kubernetes primarily manages:

```text
CPU
Memory
Processes
Network
Storage
```

It does not need to understand:

```text
Java objects
Python dictionaries
Node.js promises
Go garbage collection
.NET objects
```

Therefore, the first investigation is:

```text
Container
   ↓
Memory limit
   ↓
Actual memory usage
   ↓
Why did memory increase?
```

---

# 4. Example

Suppose:

```yaml
resources:
  requests:
    memory: "512Mi"

  limits:
    memory: "1Gi"
```

The container has a configured memory limit of:

```text
1Gi
```

If the application consumes memory beyond what the container can use under its cgroup limit, the process can be killed.

Kubernetes then reports:

```text
OOMKilled
```

---

# 5. What Is Exit Code 137?

A common OOMKilled signature is:

```text
Exit Code: 137
```

Linux signal:

```text
SIGKILL = 9
```

The conventional shell representation is:

```text
128 + 9 = 137
```

Therefore:

```text
137
```

often indicates the process was forcibly terminated with `SIGKILL`.

When combined with:

```text
Reason: OOMKilled
```

it is strong evidence of a memory-limit-related termination.

---

# 6. OOMKilled Is a Symptom, Not the Root Cause

This is one of the most important concepts.

Do not write an RCA like:

```text
Root Cause:
OOMKilled
```

That's not the root cause.

Instead:

```text
OOMKilled
    ↓
Why?
    ↓
Memory continuously increased
    ↓
Unbounded cache
    ↓
Application defect
```

The root cause is:

```text
Unbounded cache
```

not:

```text
OOMKilled
```

---

# 7. Common Causes of OOM

The same broad categories apply across programming languages.

### Application memory leak

```text
Objects keep accumulating
        ↓
Memory increases
        ↓
OOM
```

### Traffic increase

```text
RPS increases
        ↓
Concurrent requests increase
        ↓
Memory increases
```

### Large payloads

```text
Large request/response
        ↓
Large in-memory objects
        ↓
Memory pressure
```

### Large database results

```text
SELECT millions of rows
        ↓
Application loads them into memory
        ↓
Memory spike
```

### Unbounded cache

```text
Cache
  ↓
No maximum size
  ↓
Continuous growth
  ↓
OOM
```

### Excessive concurrency

```text
Requests
   ↓
Threads/tasks/workers
   ↓
Memory per worker
   ↓
Memory exhaustion
```

### Large queues

```text
Producer
   ↓
Consumer cannot keep up
   ↓
In-memory queue grows
   ↓
OOM
```

### Runtime configuration

```text
JVM heap
Node.js heap
.NET GC heap
Python process memory
```

may be incorrectly sized.

### Native/off-heap memory

The application may consume memory outside the language-managed heap.

---

# 8. Three Different Memory Problems

A DevOps Engineer must distinguish these.

## Problem 1 — Application/runtime OOM

Example Java:

```text
java.lang.OutOfMemoryError
```

Example Node.js:

```text
JavaScript heap out of memory
```

The runtime itself reports an allocation problem.

---

## Problem 2 — Container OOMKilled

Example:

```text
Reason: OOMKilled
Exit Code: 137
```

The container/process was killed because of memory exhaustion at the container/Linux level.

---

## Problem 3 — Node Memory Pressure / Eviction

Example:

```text
Node
 |
 +-- Pod A
 +-- Pod B
 +-- Pod C
 +-- Pod D
       ↓
Node memory pressure
       ↓
Kubernetes eviction
```

This is different from the application container itself being OOMKilled.

---

# 9. Container OOM vs Node OOM

This distinction is critical.

### Container-level problem

```text
Node
 |
 +-- order-service
 |       |
 |       +-- memory limit = 1Gi
 |       +-- usage > limit
 |       +-- OOMKilled
 |
 +-- other Pods
```

### Node-level pressure

```text
Node
 |
 +-- Pod A
 +-- Pod B
 +-- Pod C
 +-- Pod D
       |
       ↓
Node memory exhausted
       ↓
MemoryPressure
       ↓
Eviction
```

Therefore always investigate:

```text
Pod
+
Container
+
Node
```

---

# 10. First Production Rule

When an OOM alert fires:

### Do not immediately:

```bash
kubectl delete pod
```

and consider the incident solved.

Also don't immediately:

```text
1Gi → 2Gi → 4Gi → 8Gi
```

without understanding why memory increased.

The correct approach is:

```text
Confirm
   ↓
Measure
   ↓
Investigate
   ↓
Mitigate
   ↓
Fix
   ↓
Validate
```

---

# 11. Production Incident Flow

Use this standard flow:

```text
Alert
  ↓
Confirm customer impact
  ↓
Confirm OOMKilled
  ↓
Check Pod
  ↓
Check container
  ↓
Check node
  ↓
Check memory limits
  ↓
Check memory trend
  ↓
Check traffic
  ↓
Check recent deployment
  ↓
Check application behavior
  ↓
Runtime-specific investigation
  ↓
Root cause
  ↓
Mitigation
  ↓
Permanent fix
  ↓
Validation
  ↓
RCA
```

---

# 12. Step 1 — Check Pods

```bash
kubectl get pods -n ecommerce
```

Example:

```text
NAME                             READY   STATUS       RESTARTS
order-service-7d89f8f9d7-x2k9m   0/1     OOMKilled    5
order-service-7d89f8f9d7-k8p2q   1/1     Running      0
```

We immediately know:

```text
One replica is repeatedly failing.
```

---

# 13. Step 2 — Describe the Pod

```bash
kubectl describe pod order-service-7d89f8f9d7-x2k9m \
  -n ecommerce
```

Look for:

```text
Last State:
  Terminated:
    Reason: OOMKilled
    Exit Code: 137
```

Also inspect:

```text
Events:
```

---

# 14. Step 3 — Check Previous Logs

When a container restarts:

```bash
kubectl logs <pod> -n ecommerce --previous
```

This is extremely important.

For example:

```text
ERROR processing request
ERROR database timeout
ERROR retrying operation
...
```

Or:

```text
java.lang.OutOfMemoryError
```

Or the application may show no useful error because it was killed externally.

That is why:

> **absence of an application OOM message does not rule out OOMKilled.**

---

# 15. Step 4 — Check Memory Usage

```bash
kubectl top pod -n ecommerce
```

Example:

```text
NAME                             CPU     MEMORY
order-service-7d89f8f9d7-x2k9m   720m    990Mi
order-service-7d89f8f9d7-k8p2q   410m    620Mi
```

If the first container has:

```text
Memory limit = 1Gi
Usage        ≈ 990Mi
```

we have strong evidence of memory pressure.

---

# 16. Step 5 — Check the Resource Configuration

```bash
kubectl get deployment order-service \
  -n ecommerce \
  -o yaml
```

Find:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

---

# 17. Requests vs Limits

## Request

```yaml
requests:
  memory: "512Mi"
```

Used primarily for:

```text
scheduling
resource reservation
capacity planning
```

## Limit

```yaml
limits:
  memory: "1Gi"
```

Defines the container's memory boundary.

So:

```text
512Mi
  |
  | request
  |
  |
1Gi
  |
  | limit
  |
  X
OOM
```

---

# 18. Step 6 — Check the Node

First identify the node:

```bash
kubectl get pod <pod> \
  -n ecommerce \
  -o wide
```

Then:

```bash
kubectl describe node <node-name>
```

Look for:

```text
MemoryPressure
Conditions
Allocatable
Allocated resources
Events
```

Also:

```bash
kubectl top node
```

Example:

```text
NAME       CPU     MEMORY
node-a     55%     62%
node-b     71%     91%
node-c     48%     58%
```

If the node is heavily pressured, investigate the node as well.

---

# 19. Step 7 — Check Memory Trend

One point-in-time measurement is not enough.

You need:

```text
Memory
  |
1Gi|                    X
   |                 __/
   |              __/
   |           __/
   |        __/
   |     __/
   |____/________________ Time
```

This pattern suggests continuously increasing memory.

Another pattern:

```text
Memory
  |
1Gi|       /\      /\      /\
   |      /  \    /  \    /  \
   |_____/____\__/____\__/____\__
   |
   +----------------------------- Time
```

This may represent workload/traffic-driven memory variation rather than an obvious leak.

---

# 20. Step 8 — Check Traffic

Compare:

```text
RPS
+
memory
```

Example:

```text
RPS       Memory

100       450Mi
200       500Mi
400       620Mi
800       820Mi
1000      980Mi
```

This suggests workload is contributing to memory growth.

---

# 21. Step 9 — Check Recent Deployments

```bash
kubectl rollout history deployment/order-service \
  -n ecommerce
```

Also check:

```text
Git commit
ECR image
GitOps commit
ArgoCD sync
deployment timestamp
```

Correlate:

```text
Deployment
     ↓
Memory change
     ↓
OOM
```

But remember:

> Correlation is evidence for investigation, not automatically proof of root cause.

---

# 22. Language-Independent Root Cause Analysis

Before looking at Java, ask these questions:

### Question 1

Did traffic increase?

### Question 2

Did request size increase?

### Question 3

Did response size increase?

### Question 4

Did database result size increase?

### Question 5

Did concurrency increase?

### Question 6

Did queue backlog increase?

### Question 7

Did a deployment happen?

### Question 8

Did memory continuously grow?

### Question 9

Did another dependency become slow?

### Question 10

Did retries increase?

### Question 11

Did cache size increase?

### Question 12

Is the container simply undersized?

---

# 23. Memory Leak Pattern

A generic application memory leak can look like:

```text
Request
   ↓
Create objects
   ↓
Store references
   ↓
References never released
   ↓
Memory grows
   ↓
GC/runtime cannot reclaim
   ↓
Memory exhaustion
```

This concept applies across languages.

---

# 24. Unbounded Cache

Generic example:

```text
Cache
 |
 +-- item 1
 +-- item 2
 +-- item 3
 ...
 +-- item 1,000,000
```

If there is no:

```text
maximum size
TTL
eviction
```

memory can continuously increase.

---

# 25. Large Database Result

Bad architecture:

```text
Database
   ↓
1 million rows
   ↓
Application
   ↓
Load everything into memory
```

Better:

```text
Database
   ↓
Pagination / streaming / aggregation
   ↓
Application
```

For example:

```text
page=0&size=100
```

rather than:

```text
SELECT everything
```

---

# 26. Large Payload

Consider:

```text
Client
  ↓
100 MB request
  ↓
Application
  ↓
parse
  ↓
deserialize
  ↓
process
```

During processing, the application may temporarily hold multiple representations of the same data.

Therefore:

```text
Request size
≠
Actual memory required
```

A 100 MB payload could require substantially more than 100 MB of process memory depending on serialization and processing.

---

# 27. Excessive Concurrency

Suppose each active request requires:

```text
10 MB
```

and the application processes:

```text
100 concurrent requests
```

Approximate request-related memory:

```text
100 × 10MB = 1GB
```

Actual memory behavior is more complex, but the principle is important:

```text
Concurrency
   ↓
More active state
   ↓
More memory
```

---

# 28. In-Memory Queue

Consider:

```text
Producer
   ↓
1000 messages/sec
```

Consumer:

```text
500 messages/sec
```

Backlog:

```text
+500 messages/sec
```

If those messages remain in application memory:

```text
Memory
   ↑
   ↑
   ↑
OOM
```

A better architecture may use:

```text
SQS
Kafka
RabbitMQ
```

so the durable queue is outside the application process.

---

# 29. Retry Storm

Another language-independent cause:

```text
Dependency becomes slow
        ↓
Requests timeout
        ↓
Application retries
        ↓
More concurrent requests
        ↓
More memory
        ↓
More timeouts
        ↓
More retries
```

This can create:

```text
Retry storm
```

and eventually:

```text
OOMKilled
```

This is why our platform uses:

```text
timeouts
+
retry budgets
+
exponential backoff
+
jitter
+
circuit breakers
```

where appropriate.

---

# 30. Runtime-Specific Investigation

Once Kubernetes-level investigation is complete:

```text
                    OOM
                     |
       +-------------+-------------+
       |             |             |
      Java         Node.js       Python
       |             |             |
      JVM         V8/runtime    Python/runtime
```

The DevOps Engineer works with the application team to investigate runtime-specific behavior.

---

# 31. Java Investigation

Our project uses:

```text
Java 21
Spring Boot
```

Java process memory includes more than the Java heap.

Conceptually:

```text
Java Process
 |
 +-- Heap
 +-- Metaspace
 +-- Thread stacks
 +-- Code cache
 +-- Direct buffers
 +-- Native memory
 +-- JVM runtime
```

Therefore:

```text
Container memory
≠
Java heap
```

---

# 32. Java Heap

Example:

```text
Container limit = 1Gi
```

Do not automatically configure:

```text
-Xmx1g
```

because the JVM needs additional memory outside the heap.

You need headroom for:

```text
Metaspace
Threads
Direct memory
Native allocations
JVM overhead
```

---

# 33. Java Example

Illustrative configuration:

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxRAMPercentage=65"
```

Or an explicitly sized heap:

```text
-Xms512m
-Xmx650m
```

The correct value must be determined through:

```text
load testing
+
JVM metrics
+
GC behavior
+
actual container memory
```

There is no universal `Xmx` value.

---

# 34. Java Memory Leak

Example:

```java
private static final List<Order> orders = new ArrayList<>();
```

If objects continuously accumulate:

```text
Order
Order
Order
Order
...
```

and references remain reachable:

```text
GC cannot reclaim them
```

Memory keeps growing.

---

# 35. Java GC Investigation

Look at:

```text
heap usage
GC frequency
GC pause duration
old-generation occupancy
allocation rate
```

Potential pattern:

```text
Allocation ↑
     ↓
GC ↑
     ↓
Objects survive
     ↓
Old generation ↑
     ↓
Memory pressure
```

---

# 36. Java Threads

Threads also consume memory.

Check:

```text
live threads
peak threads
thread creation
thread pools
```

A thread explosion can cause significant native memory consumption.

---

# 37. Java Diagnostics

Depending on the production diagnostic procedure:

```bash
jcmd
jstack
jmap
JFR
```

can help investigate:

```text
heap
threads
allocation
GC
locks
```

Be careful with heap dumps in production because they can:

```text
consume disk
contain sensitive data
increase system pressure
```

Use controlled diagnostic procedures.

---

# 38. Node.js Investigation

Node.js has its own runtime memory model.

Useful areas include:

```text
V8 heap
external memory
Buffer usage
event-loop behavior
worker processes
```

Possible symptoms:

```text
JavaScript heap out of memory
```

Node.js-specific investigation may involve:

```text
heap snapshots
V8 metrics
GC behavior
process memory
```

But the first Kubernetes checks remain:

```bash
kubectl describe pod
kubectl logs --previous
kubectl top pod
```

---

# 39. Python Investigation

Python applications can experience:

```text
object retention
large collections
unbounded caches
worker/process multiplication
native library allocations
```

For example:

```text
Gunicorn
   |
   +-- Worker 1
   +-- Worker 2
   +-- Worker 3
   +-- Worker 4
```

More workers can mean more process memory.

Therefore:

```text
workers
×
memory per worker
```

must be considered.

---

# 40. Go Investigation

Go has its own garbage collector and runtime behavior.

Investigate:

```text
heap
goroutines
GC
allocations
goroutine leaks
```

A goroutine leak can create growing memory usage.

Again, Kubernetes investigation remains the same.

---

# 41. .NET Investigation

.NET applications can experience:

```text
managed heap growth
GC pressure
large object allocations
native memory
thread growth
```

Useful runtime diagnostics can include:

```text
GC metrics
dotnet counters
dump analysis
runtime metrics
```

The container-level diagnosis remains:

```text
OOMKilled
→
measure
→
investigate
```

---

# 42. The DevOps Engineer's Responsibility

The DevOps Engineer does **not** necessarily need to debug every application-level memory bug.

Instead:

```text
DevOps
  |
  +-- Detect
  +-- Confirm
  +-- Measure
  +-- Correlate
  +-- Protect platform
  +-- Mitigate
  +-- Provide evidence
  |
  +-------- Application Team
                 |
                 +-- Code-level root cause
```

For example:

```text
DevOps:
"order-service memory increased from 500Mi to 980Mi
after deployment 1.9.0."

Developer:
"The new cache retains every order object."

DevOps:
"Let's reproduce under load and validate the fix."
```

That's a realistic production workflow.

---

# 43. Prometheus Investigation

Important generic metric:

```promql
container_memory_working_set_bytes
```

Example:

```promql
container_memory_working_set_bytes{
  namespace="ecommerce",
  pod=~"order-service-.*"
}
```

Compare it with:

```text
container_spec_memory_limit_bytes
```

Conceptually:

```text
Memory utilization
=
working_set
/
memory_limit
```

---

# 44. Application Metrics

For Java:

```text
JVM heap
JVM non-heap
GC
threads
```

For other languages:

```text
runtime memory
process memory
workers
threads/goroutines
GC
application queues
```

This gives:

```text
Infrastructure metrics
+
runtime metrics
+
business metrics
```

---

# 45. Business Metrics Matter

Suppose:

```text
Order requests ↑
```

and:

```text
Memory ↑
```

This tells us much more than memory alone.

Our dashboard should correlate:

```text
RPS
HTTP latency
HTTP 5xx
memory
CPU
Pod count
RDS connections
Redis
Kafka lag
SQS backlog
```

---

# 46. HPA and OOM

HPA can help when increased traffic is the cause.

Example:

```text
Traffic ↑
   ↓
Memory ↑
   ↓
HPA scales
   ↓
More Pods
   ↓
Traffic distributed
```

But if the problem is a memory leak:

```text
Pod 1 leaks
Pod 2 leaks
Pod 3 leaks
Pod 4 leaks
```

HPA may simply create more leaking Pods.

Therefore:

> **Autoscaling is not a fix for a memory leak.**

---

# 47. Dependency Protection

During an OOM incident, don't look only at the affected service.

Check:

```text
RDS
Redis
Kafka
SQS
external APIs
```

Example:

```text
RDS slows
   ↓
Application requests wait
   ↓
More concurrent requests remain active
   ↓
Memory increases
   ↓
OOM
```

The apparent OOM problem may actually have started with a dependency failure.

---

# 48. Correct Mitigation

There is no single mitigation for every OOM incident.

Choose based on the evidence.

### Bad deployment

```text
Rollback
```

### Traffic-driven memory

```text
Scale horizontally
+
rate limiting/backpressure if required
```

### Undersized container

```text
Right-size resources
```

### Memory leak

```text
Fix application
```

### Large query

```text
Pagination/streaming/query optimization
```

### Unbounded cache

```text
Bound cache
+
TTL
+
eviction
```

### Retry storm

```text
Timeout
+
backoff
+
jitter
+
circuit breaker
```

---

# 49. Why Increasing Memory Is Sometimes Correct

Increasing memory is not inherently wrong.

It is wrong when it is used **without understanding the workload**.

Suppose testing shows:

```text
Normal:
500Mi

Peak:
750Mi

Container limit:
512Mi
```

and there is:

```text
No leak
No abnormal behavior
Expected traffic
Correct runtime configuration
```

Then the container is undersized.

Increasing the limit may be the correct fix.

---

# 50. Right-Sizing Process

Use:

```text
Load test
   ↓
Measure peak memory
   ↓
Determine safety margin
   ↓
Set request
   ↓
Set limit
   ↓
Validate under peak traffic
```

Do not choose:

```text
1Gi
```

because:

> "1Gi is common."

Choose it because:

> "Testing demonstrated the service's expected peak memory requirement and the configured limit provides appropriate headroom."

---

# 51. Production Example

Suppose:

```text
order-service
```

has:

```text
Normal RPS = 200
Peak RPS   = 1,000
```

Memory:

```text
200 RPS  → 450Mi
500 RPS  → 600Mi
800 RPS  → 750Mi
1000 RPS → 820Mi
```

Limit:

```text
1Gi
```

No memory leak is observed.

This may indicate:

```text
Workload-driven memory
```

rather than a leak.

Possible solution:

```text
HPA
+
appropriate resource sizing
+
concurrency controls
```

---

# 52. Production Example — Memory Leak

Another service:

```text
100 RPS consistently
```

Memory:

```text
500Mi
550Mi
600Mi
650Mi
700Mi
750Mi
800Mi
900Mi
```

Traffic is unchanged.

After restart:

```text
500Mi
```

Then it begins increasing again.

This strongly suggests investigating:

```text
memory retention
cache
object lifecycle
queue
connection/resource leak
```

---

# 53. Production Example — Dependency Failure

Consider:

```text
RDS latency ↑
     ↓
Requests wait longer
     ↓
Concurrent requests ↑
     ↓
Memory ↑
     ↓
OOMKilled
```

If we only increase memory:

```text
1Gi → 2Gi
```

we may delay the failure without fixing:

```text
RDS latency
```

Therefore:

> Always investigate the surrounding system.

---

# 54. GitOps Mitigation

Our platform uses:

```text
Git
 ↓
GitOps
 ↓
ArgoCD
 ↓
EKS
```

Suppose version `1.9.0` caused OOM.

Preferred approach:

```text
Revert GitOps change
       ↓
Commit
       ↓
ArgoCD
       ↓
Rollback desired state
       ↓
EKS
```

Avoid making an unmanaged change such as:

```bash
kubectl edit deployment
```

and leaving Git in the old state.

Otherwise:

```text
Git desired state
       ≠
Cluster state
```

and ArgoCD may reconcile it back.

---

# 55. Validate After Mitigation

Don't say:

> "Pods are running, incident solved."

Check:

```text
Pod status
Pod restarts
Memory
CPU
RPS
Latency
5xx
RDS
Redis
Kafka
SQS
Business transaction
```

For our application:

```text
Login
  ↓
Product
  ↓
Cart
  ↓
Order
  ↓
Payment
  ↓
Inventory
  ↓
Notification
```

must be validated where applicable.

---

# 56. OOM Alerting

Create alerts for:

```text
Memory utilization
Pod restarts
OOMKilled events
Node MemoryPressure
JVM/runtime memory
```

Example concept:

```yaml
- alert: ContainerMemoryNearLimit
  expr: |
    (
      container_memory_working_set_bytes
      /
      container_spec_memory_limit_bytes
    ) > 0.90
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Container memory usage is near its limit"
```

The production query should exclude infrastructure containers and handle containers without configured limits appropriately.

---

# 57. Runbook

Create:

```text
runbooks/
└── oomkilled.md
```

Recommended structure:

```markdown
# OOMKilled Runbook

## Symptoms

- Pod restart
- OOMKilled
- Exit code 137
- Memory near limit

## Immediate Checks

kubectl get pods
kubectl describe pod
kubectl logs --previous
kubectl top pod

## Node Checks

kubectl top node
kubectl describe node

## Investigation

- Memory trend
- Traffic
- Recent deployment
- Request size
- Response size
- Database queries
- Cache
- Queue backlog
- Retries
- Runtime metrics

## Runtime Investigation

- Java/JVM
- Node.js
- Python
- Go
- .NET

## Mitigation

- Rollback
- Scale
- Right-size
- Rate limit
- Reduce workload

## Validation

- Memory stable
- Restarts stopped
- Error rate normal
- Latency normal
- Business flow successful

## RCA

Document actual root cause.
```

---

# 58. Incident Timeline Example

```text
18:00
Normal traffic

18:15
Version 1.9.0 deployed

18:20
Memory begins increasing

18:25
Memory reaches 80%

18:28
Memory reaches 90%

18:30
OOMKilled

18:31
Alert fires

18:33
Incident declared

18:35
Deployment correlated with memory increase

18:38
Rollback initiated

18:40
Pods stabilize

18:45
Business transaction validated

19:00
Permanent investigation begins
```

---

# 59. RCA Example

### Incident

```text
order-service OOMKilled
```

### Impact

```text
Some order requests returned errors
during the affected period.
```

### Detection

```text
Prometheus memory alert
+
Pod restart/OOM signal
```

### Root Cause

```text
Version 1.9.0 introduced an unbounded
in-memory cache.
```

### Contributing Factors

```text
No cache maximum
No memory regression test
Insufficient canary validation
```

### Mitigation

```text
Rollback to version 1.8.0
```

### Permanent Fix

```text
Bounded cache
TTL
Eviction policy
Load testing
Memory alerts
```

---

# 60. Prevention

Implement:

```text
Application
   |
   +-- Memory-safe code
   +-- Bounded caches
   +-- Pagination
   +-- Streaming
   +-- Concurrency controls
   +-- Timeouts
   +-- Retry limits
   |
Kubernetes
   |
   +-- Requests
   +-- Limits
   +-- HPA
   +-- PDB
   +-- ResourceQuota
   |
Observability
   |
   +-- Prometheus
   +-- Grafana
   +-- ELK
   +-- Jaeger
   |
Process
   |
   +-- Load testing
   +-- Canary
   +-- Runbooks
   +-- RCA
```

---

# 61. Universal OOM Decision Tree

This is the most important section to remember.

```text
                    OOMKilled
                        |
                        v
              Confirm container OOM
                        |
                        v
             Check Pod + Container
                        |
                        v
                 Check Node
                        |
                        v
              Check memory trend
                        |
          +-------------+-------------+
          |                           |
      Stable pattern             Continuous growth
          |                           |
          v                           v
   Traffic/workload?            Memory leak?
          |                           |
     +----+----+                 +----+----+
     |         |                 |         |
    YES       NO                YES       NO
     |         |                 |         |
     v         v                 v         v
Scale/      Check sizing      Fix app   Runtime/
optimize    + runtime                   native memory
     |
     v
Check dependencies
     |
     v
Recent deployment?
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

# 62. What Changes by Programming Language?

| Area             | Kubernetes            | Java                  | Node.js             | Python                | Go                | .NET           |
| ---------------- | --------------------- | --------------------- | ------------------- | --------------------- | ----------------- | -------------- |
| Pod memory       | Same                  | Same                  | Same                | Same                  | Same              | Same           |
| Requests/limits  | Same                  | Same                  | Same                | Same                  | Same              | Same           |
| OOMKilled        | Same                  | Same                  | Same                | Same                  | Same              | Same           |
| Exit 137         | Same concept          | Same                  | Same                | Same                  | Same              | Same           |
| Memory leak      | Generic investigation | Heap/object retention | V8/object retention | Object/process/native | Heap/goroutine    | Managed/native |
| Runtime metrics  | Generic               | JVM                   | V8/process          | Process/runtime       | Go runtime        | CLR            |
| GC investigation | Runtime-specific      | JVM GC                | V8 GC               | Python GC             | Go GC             | .NET GC        |
| Heap dump        | Runtime-specific      | Common                | Heap snapshot       | Tool-dependent        | Runtime profiling | Dump analysis  |
| JVM flags        | N/A                   | Important             | N/A                 | N/A                   | N/A               | N/A            |

The key DevOps skill is:

> **Understand the Kubernetes/container problem first, then involve the application team's runtime expertise.**

---

# 63. Final Production Architecture

```text
                         Internet
                            |
                         Route 53
                            |
                        CloudFront
                            |
                           WAF
                            |
                           ALB
                            |
                    +-------+-------+
                    |     EKS      |
                    |              |
                    |  21 Services |
                    |              |
                    +-------+------+
                            |
                    Application Runtime
                    /       |        \
                 Java    Node.js    Python
                    \       |        /
                     Container
                         |
                 CPU / Memory Limits
                         |
                 Prometheus Metrics
                         |
                       Grafana
                         |
                    Alertmanager
                         |
                   Incident Team
```

For the Java-based project:

```text
order-service
      |
      +-- JVM
      +-- Spring Boot
      +-- PostgreSQL
      +-- Redis
      +-- Kafka
      +-- SQS
      +-- OpenTelemetry
      +-- ELK
```

But the **OOM investigation begins above the Java layer**, at:

```text
Linux
  ↓
Container
  ↓
Kubernetes
```

---

# 64. Final Rules

Remember these seven rules:

### Rule 1

```text
OOMKilled is not a Java-specific problem.
```

### Rule 2

```text
OOMKilled is a symptom, not automatically the root cause.
```

### Rule 3

```text
Always check Pod + Container + Node.
```

### Rule 4

```text
Don't blindly increase memory.
```

### Rule 5

```text
Don't use HPA to hide a memory leak.
```

### Rule 6

```text
Investigate runtime-specific behavior after Kubernetes-level investigation.
```

### Rule 7

```text
Fix the root cause and validate the entire business flow.
```

---

## 65. Interview/Production Mental Model

If someone asks:

> **"A production Pod is OOMKilled. What will you do?"**

Your answer should be:

```text
First I confirm whether it is actually OOMKilled
using kubectl describe and the container termination state.

Then I check the previous container logs, Pod events,
current memory usage, memory limits, and the node's
memory pressure.

I compare the memory trend with traffic, deployments,
application metrics, and dependency behavior.

Then I determine whether the cause is a memory leak,
traffic/concurrency, large payload or query, unbounded
cache/queue, incorrect resource sizing, runtime-specific
memory behavior, or a dependency-induced problem.

For immediate customer impact I mitigate, for example by
rolling back a recent bad deployment or scaling safely.

After identifying the root cause, I implement the permanent
fix, deploy it through GitOps, validate memory stability,
restarts, latency, errors, dependencies, and the business
transaction, and finally document the RCA and preventive
actions.
```

That is the **language-independent DevOps answer**, while the Java/JVM section gives you the deeper knowledge required for this particular project.

---

**Phase 18 progress:**

```text
143 — CrashLoopBackOff       ✅
144 — OOMKilled              ✅
145 — CPU Saturation         ⏭️
146 — Memory Saturation
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

**Next: `145 — CPU Saturation Production Incident`**.
