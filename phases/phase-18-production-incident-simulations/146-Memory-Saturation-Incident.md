# 146 — Memory Saturation Production Incident

> **Phase 18 — Incident Management**
> **Project:** Enterprise E-Commerce Platform
> **Primary stack:** AWS EKS + Java 21 + Spring Boot
> **Scope:** Memory saturation is explained first at the **Linux/Kubernetes/container level**, independent of programming language. Java/JVM investigation is covered afterward.

---

# 1. Objective

Memory saturation is different from simply seeing `OOMKilled`.

A production system can experience:

```text
Memory usage ↑
       ↓
Available memory ↓
       ↓
Performance degradation
       ↓
OOMKilled / eviction
```

But the incident can happen at different levels:

```text
Application memory
        ↓
Container memory
        ↓
Pod
        ↓
Node memory
        ↓
Cluster capacity
```

Our objective is to identify **where memory is being exhausted and why**.

---

# 2. Memory Saturation vs OOMKilled

These are related but different.

## Memory saturation

The system is approaching or has reached a state where available memory is insufficient for normal workload.

Example:

```text
Container limit = 1Gi

Memory:
600Mi
700Mi
800Mi
900Mi
950Mi
```

The application may still be running.

---

## OOMKilled

The container actually reaches a memory boundary and is killed.

```text
Memory
  |
1Gi |                    X
    |                 __/
    |              __/
    |           __/
    |__________/
    |
    +----------------------> time

                    OOMKilled
```

Therefore:

```text
Memory saturation
        ↓
Warning condition

OOMKilled
        ↓
Actual termination
```

---

# 3. Memory Saturation Can Occur at Multiple Levels

This is extremely important.

```text
                    Cluster
                       |
          +------------+------------+
          |                         |
        Node A                    Node B
          |                         |
      +---+---+                 +---+---+
      |       |                 |       |
    Pod A   Pod B             Pod C   Pod D
      |
   Container
      |
 Application
```

Memory can become saturated at:

```text
Application
Container
Pod
Node
Cluster
```

---

# 4. Container Memory Saturation

Example:

```yaml
resources:
  requests:
    memory: "512Mi"

  limits:
    memory: "1Gi"
```

Current usage:

```text
900Mi
```

The container is approaching its configured limit.

This is a **container-level memory problem**.

---

# 5. Node Memory Saturation

Suppose the node has:

```text
16 GiB allocatable memory
```

and workloads consume:

```text
15.5 GiB
```

The node is under significant memory pressure.

Kubernetes may report:

```text
MemoryPressure=True
```

This is different from one container exceeding its memory limit.

---

# 6. Kubernetes MemoryPressure

Check:

```bash
kubectl describe node <node-name>
```

Look under:

```text
Conditions:
```

You may see:

```text
MemoryPressure   True
```

This means the kubelet considers the node to be under memory pressure.

---

# 7. Node Memory Pressure Can Cause Eviction

If the node does not have enough memory, Kubernetes may evict Pods.

Conceptually:

```text
Node memory
      ↓
Available memory falls
      ↓
MemoryPressure
      ↓
Kubelet eviction
      ↓
Pod terminated/evicted
```

This is **not the same as `OOMKilled`**.

That distinction matters during incident investigation.

---

# 8. OOMKilled vs Evicted

| Situation                      | Meaning                                           |
| ------------------------------ | ------------------------------------------------- |
| `OOMKilled`                    | Container/process exceeded memory boundary        |
| `Evicted`                      | Kubelet evicted Pod due to node resource pressure |
| `MemoryPressure=True`          | Node is under memory pressure                     |
| Application `OutOfMemoryError` | Runtime/application allocation failure            |
| High memory                    | Memory usage is high but may not yet be failing   |

Always determine which one occurred.

---

# 9. First Production Rule

Do not immediately:

```bash
kubectl delete pod
```

and consider the incident fixed.

Also don't immediately increase:

```text
512Mi → 1Gi → 2Gi → 4Gi
```

without investigation.

The correct approach is:

```text
Alert
 ↓
Confirm impact
 ↓
Determine memory level
 ↓
Measure
 ↓
Find cause
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

# 10. Production Incident Scenario

Suppose:

```text
order-service
```

normally runs:

```text
Memory = 500Mi
```

Suddenly:

```text
Memory = 900Mi
```

Then:

```text
Latency ↑
Pod restarts ↑
```

Grafana:

```text
Memory
1Gi |                    /X
    |                 __/
    |              __/
    |           __/
    |        __/
    |_______/
    +----------------------> time
```

Alert:

```text
ContainerMemoryNearLimit
```

---

# 11. Step 1 — Check Pods

```bash
kubectl get pods -n ecommerce
```

Example:

```text
NAME                             READY   STATUS    RESTARTS
order-service-7d89f8f9d7-x2k9m   1/1     Running   3
order-service-7d89f8f9d7-k8p2q   1/1     Running   0
```

A Pod can remain:

```text
Running
```

while memory is approaching the limit.

Therefore:

> Pod status alone is not enough.

---

# 12. Step 2 — Check Current Memory

```bash
kubectl top pod -n ecommerce
```

Example:

```text
NAME                             CPU     MEMORY
order-service-7d89f8f9d7-x2k9m   550m    920Mi
order-service-7d89f8f9d7-k8p2q   450m    610Mi
```

The first Pod is approaching:

```text
1Gi
```

---

# 13. Step 3 — Check Memory Configuration

```bash
kubectl get deployment order-service \
  -n ecommerce \
  -o yaml
```

Look for:

```yaml
resources:
  requests:
    memory: "512Mi"

  limits:
    memory: "1Gi"
```

Now we know:

```text
Request = 512Mi
Limit   = 1Gi
Usage   ≈ 920Mi
```

---

# 14. Step 4 — Check the Node

Find the node:

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
NAME       CPU     MEMORY
node-a     60%     55%
node-b     70%     92%
node-c     50%     57%
```

Node B requires investigation.

---

# 15. Step 5 — Describe the Node

```bash
kubectl describe node node-b
```

Check:

```text
Conditions
Allocated resources
Events
Capacity
Allocatable
```

Especially:

```text
MemoryPressure
```

---

# 16. Step 6 — Check All Pods on the Node

```bash
kubectl get pods -A \
  -o wide \
  --field-selector spec.nodeName=node-b
```

Then identify which workloads consume memory.

You may discover:

```text
node-b
 |
 +-- order-service       2Gi
 +-- payment-service     1.5Gi
 +-- inventory-service   2Gi
 +-- monitoring          1Gi
 +-- logging             2Gi
 +-- other workloads     3Gi
```

The issue may not belong to `order-service` alone.

---

# 17. Step 7 — Check Node Allocatable Memory

A node has:

```text
Physical memory
      ↓
OS reserved memory
      ↓
Kubernetes/kubelet/system reservations
      ↓
Allocatable memory
```

Therefore:

```text
Node physical memory
≠
Memory available for Pods
```

This is important when performing capacity planning.

---

# 18. Requests Matter for Capacity Planning

Suppose:

```text
Node allocatable = 16Gi
```

and Pods request:

```text
Pod A = 2Gi
Pod B = 2Gi
Pod C = 4Gi
Pod D = 4Gi
```

Total:

```text
12Gi
```

There is approximately:

```text
4Gi
```

remaining according to requested capacity.

But actual usage can differ from requests.

Therefore you must monitor both:

```text
Requested memory
+
Actual memory usage
```

---

# 19. Why Requests Should Not Be Too Low

Suppose actual workload usually requires:

```text
2Gi
```

but request is:

```yaml
requests:
  memory: "256Mi"
```

Kubernetes may schedule too many workloads onto the same node because scheduling is based on requests.

Eventually:

```text
Actual usage ↑
       ↓
Node memory pressure
       ↓
Evictions
```

This is why resource requests are part of **capacity planning**, not merely configuration.

---

# 20. Why Limits Should Not Be Arbitrarily High

Suppose every container has:

```yaml
limits:
  memory: "16Gi"
```

but nodes only have:

```text
16Gi
```

This does not mean Kubernetes reserves 16Gi for each container in the same way requests affect scheduling.

You can create dangerous overcommit situations where many containers can attempt to consume large amounts of memory simultaneously.

Therefore resource configuration must be designed intentionally.

---

# 21. Memory Overcommit

Consider:

```text
Node = 16Gi

Pod A limit = 8Gi
Pod B limit = 8Gi
Pod C limit = 8Gi
Pod D limit = 8Gi
```

The theoretical limits total:

```text
32Gi
```

on a:

```text
16Gi
```

node.

This can be risky if all workloads actually consume their limits simultaneously.

Therefore:

```text
Requests
+
Limits
+
Actual usage
+
Node capacity
```

must all be considered.

---

# 22. Step 8 — Check Memory Trend

Use Grafana rather than looking at only the current value.

Example:

```text
Pattern A:

500Mi
520Mi
510Mi
530Mi
520Mi
```

Stable.

Pattern B:

```text
500Mi
600Mi
700Mi
800Mi
900Mi
```

Continuous growth.

Pattern C:

```text
500Mi
900Mi
500Mi
900Mi
500Mi
```

Traffic/workload-driven pattern may be involved.

---

# 23. Memory Leak Pattern

A generic memory leak looks like:

```text
Application
     |
Creates objects
     |
References retained
     |
Objects cannot be released
     |
Memory grows
     |
Memory grows
     |
OOM
```

This applies to:

```text
Java
Node.js
Python
Go
.NET
```

The runtime-specific mechanism differs, but the operational symptom can be the same.

---

# 24. Traffic-Driven Memory

Suppose:

```text
100 RPS → 500Mi
200 RPS → 600Mi
400 RPS → 750Mi
800 RPS → 950Mi
```

This can be normal workload scaling.

But you must compare it against:

```text
tested capacity
```

If load testing demonstrated:

```text
800 RPS
```

requires:

```text
950Mi
```

then the solution may be:

```text
horizontal scaling
```

rather than treating it as a leak.

---

# 25. Large Payload Problem

Suppose a service receives:

```text
50MB JSON
```

The application may temporarily create:

```text
request buffer
+
parsed objects
+
validation objects
+
business objects
+
response objects
```

Therefore:

```text
50MB request
```

does not necessarily mean:

```text
50MB memory
```

It can require significantly more.

---

# 26. Large Database Result

Example:

```text
Database
   ↓
500,000 rows
   ↓
Application
   ↓
Load all rows into memory
```

This can cause memory saturation.

Better:

```text
Database
   ↓
Pagination
   ↓
100 rows
   ↓
Process
   ↓
Next 100
```

or use appropriate streaming/batch processing.

---

# 27. Unbounded Cache

Example:

```text
Cache
 |
 +-- object
 +-- object
 +-- object
 ...
 +-- millions of objects
```

Without:

```text
TTL
Maximum size
Eviction
```

memory can continuously grow.

This is one of the most common application-level causes worth investigating.

---

# 28. In-Memory Queue

Suppose:

```text
Producer = 2,000 messages/sec
Consumer = 1,000 messages/sec
```

If messages are retained in application memory:

```text
Backlog = +1,000/sec
```

Memory increases continuously.

For durable asynchronous workloads, external messaging such as:

```text
SQS
Kafka
RabbitMQ
```

can prevent the application process itself from becoming the durable queue.

---

# 29. Excessive Concurrency

Suppose every active request consumes approximately some amount of memory.

Then:

```text
Concurrency ↑
       ↓
Active state ↑
       ↓
Memory ↑
```

If the application accepts unlimited concurrent work:

```text
10 requests
100 requests
1,000 requests
10,000 requests
```

memory may become saturated.

Possible controls:

```text
Concurrency limits
Connection pool limits
Worker limits
Queueing
Backpressure
Rate limiting
```

---

# 30. Retry Storm and Memory

Consider:

```text
Dependency slow
      ↓
Requests timeout
      ↓
Retries
      ↓
More concurrent work
      ↓
Memory ↑
      ↓
Latency ↑
      ↓
More timeouts
```

This can create a feedback loop.

Therefore during a memory incident check:

```text
retry rate
timeout rate
dependency latency
request concurrency
```

---

# 31. Memory Saturation and HPA

HPA can help if memory grows because traffic increased.

Example:

```text
Traffic ↑
   ↓
Memory ↑
   ↓
HPA
   ↓
More Pods
   ↓
Traffic distributed
   ↓
Memory per Pod ↓
```

But HPA can make a memory leak more expensive:

```text
Pod 1 leaking
   ↓
HPA
   ↓
Pod 2
Pod 3
Pod 4
   ↓
4 leaking processes
```

Therefore:

> **Do not use horizontal scaling as the permanent solution to a memory leak.**

---

# 32. Java/Spring Boot Memory Investigation

Now we move into the application-specific section.

Our project uses:

```text
Java 21
Spring Boot
Maven
```

A Java process contains more than Java heap.

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
JVM heap
```

---

# 33. Java Heap

Suppose:

```text
Container limit = 1Gi
```

Don't blindly use:

```text
-Xmx1g
```

because the JVM also requires memory for:

```text
Metaspace
Thread stacks
Direct memory
Native allocations
JVM overhead
```

You need headroom.

---

# 34. Java Heap Saturation

A common pattern:

```text
Heap
 |
 |       /\      /\      /\
 |      /  \    /  \    /  \
 |_____/____\__/____\__/____\__
 |
 +----------------------------> time
```

The JVM may use memory and later reclaim it through GC.

This is different from:

```text
Heap
 |
 |       /
 |      /
 |     /
 |    /
 |___/_________________________ time
```

where memory continuously grows.

---

# 35. Java GC Pressure

High memory can lead to:

```text
Allocation ↑
    ↓
GC frequency ↑
    ↓
CPU ↑
    ↓
Latency ↑
```

This connects the previous incident:

```text
145 CPU Saturation
```

with:

```text
146 Memory Saturation
```

Memory and CPU incidents can cause each other.

---

# 36. Java Thread Memory

Each thread can consume native memory for its stack.

If the application creates excessive threads:

```text
Threads ↑
   ↓
Native memory ↑
   ↓
Container memory ↑
```

Therefore check:

```text
jvm_threads_live_threads
jvm_threads_peak_threads
```

and application thread-pool configuration.

---

# 37. Java Direct Memory

Java applications can use off-heap/direct memory.

For example:

```text
network buffers
NIO
libraries
drivers
```

So:

```text
Heap looks healthy
```

does not necessarily mean:

```text
Container memory is healthy
```

This is an important production troubleshooting point.

---

# 38. Spring Boot Metrics

Our Spring Boot services expose:

```text
/actuator/prometheus
```

Prometheus can collect JVM metrics such as:

```text
jvm_memory_used_bytes
jvm_memory_max_bytes
jvm_gc_pause_seconds
jvm_threads_live_threads
```

Example:

```promql
jvm_memory_used_bytes{
  application="order-service",
  area="heap"
}
```

---

# 39. Container Memory vs JVM Heap Dashboard

Our Grafana dashboard should ideally show:

```text
Container Memory
       |
       +---- JVM Heap
       |
       +---- JVM Non-Heap
       |
       +---- Threads
       |
       +---- GC
       |
       +---- CPU
```

This lets us answer:

> Is the memory problem inside the JVM heap or outside it?

---

# 40. Java Diagnostic Tools

Depending on the production diagnostic process:

```bash
jcmd
jmap
jstack
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

Use them carefully in production.

A heap dump can be:

```text
large
CPU/disk intensive
sensitive
```

and may contain application/customer information.

---

# 41. Node.js Memory Investigation

For Node.js, investigate:

```text
V8 heap
external memory
Buffers
event loop
worker processes
GC
```

Useful runtime metrics can be compared with:

```text
container_memory_working_set_bytes
```

to understand whether the process or container is approaching its limit.

---

# 42. Python Memory Investigation

Python memory problems may involve:

```text
large objects
unbounded collections
cache
worker processes
native libraries
data processing
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

Four workers can require substantially more memory than one process.

---

# 43. Go Memory Investigation

For Go:

```text
heap
goroutines
GC
allocations
native memory
```

should be investigated.

A goroutine leak can cause memory to increase over time.

---

# 44. .NET Memory Investigation

For .NET:

```text
managed heap
GC
large object allocations
thread pool
native memory
```

may contribute to memory saturation.

---

# 45. Memory Saturation at Node Level

Now consider the entire node.

```text
Node: 16Gi allocatable

Pod A → 4Gi
Pod B → 3Gi
Pod C → 3Gi
Pod D → 2Gi
System → 2Gi
```

Total:

```text
14Gi
```

Only:

```text
2Gi
```

remains.

A traffic increase could push the node toward memory pressure.

---

# 46. DaemonSet Memory Must Be Considered

Our EKS nodes run platform components such as:

```text
Fluent Bit
node-exporter
security agents
CNI components
other DaemonSets
```

These consume node resources.

Therefore:

```text
Application capacity
```

is not equal to:

```text
Entire node capacity
```

You must reserve capacity for platform services.

---

# 47. Observability Can Contribute to Memory Pressure

For example:

```text
Fluent Bit
   ↓
High log volume
   ↓
Buffering
   ↓
Memory ↑
```

or:

```text
Prometheus
   ↓
High cardinality
   ↓
Memory ↑
```

or:

```text
Logstash
   ↓
Large pipeline backlog
   ↓
Memory ↑
```

Therefore the observability stack itself must be monitored.

---

# 48. Redis and Application Memory

Our architecture uses Redis as an external cache.

This is generally preferable to putting a large cache inside every application process.

Instead of:

```text
order-service
   |
   +-- huge local cache
```

we can use:

```text
order-service
       |
      Redis
```

However, the application can still create large local objects while processing Redis results.

So Redis does not automatically eliminate application memory problems.

---

# 49. RDS and Memory

RDS itself has separate memory management.

If RDS becomes slow:

```text
RDS latency ↑
   ↓
Application requests remain active
   ↓
Concurrency ↑
   ↓
Application memory ↑
```

Therefore a memory incident can originate from:

```text
database performance
```

rather than a direct memory leak.

---

# 50. Kafka and Memory

A Kafka consumer can experience:

```text
Consumer processing slow
       ↓
Messages accumulate
       ↓
Application fetch/buffer behavior
       ↓
Memory pressure
```

Investigate:

```text
consumer lag
batch size
fetch settings
processing time
consumer concurrency
```

---

# 51. SQS and Memory

Similarly:

```text
SQS
 ↓
Consumer receives batch
 ↓
Application processing
 ↓
Large batch
 ↓
Memory spike
```

KEDA may scale consumers based on queue depth, but batch size and concurrency still need to be controlled.

---

# 52. Memory Saturation and KEDA

For asynchronous workloads:

```text
SQS/Kafka
   ↓
KEDA
   ↓
Consumer Pods
```

If consumers are too few:

```text
Backlog ↑
```

If consumers are too many:

```text
Downstream pressure ↑
DB connections ↑
Memory/CPU ↑
```

Therefore scaling must have:

```text
min replicas
max replicas
batch/concurrency controls
downstream capacity
```

---

# 53. Production Mitigation Options

Choose based on root cause.

### Traffic-driven memory

```text
Scale horizontally
```

### Memory leak

```text
Fix application
```

### Container undersized

```text
Right-size memory
```

### Large payload

```text
Limit / stream / paginate
```

### Large query

```text
Pagination / batching
```

### Unbounded cache

```text
TTL + maximum size + eviction
```

### Retry storm

```text
Timeout + backoff + jitter + circuit breaker
```

### Node pressure

```text
Scale nodes / redistribute workloads
```

### Bad deployment

```text
Rollback
```

---

# 54. Why Restarting the Pod Is Not a Permanent Fix

Suppose:

```text
Memory:
500Mi
600Mi
700Mi
800Mi
900Mi
```

Then restart:

```text
500Mi
```

The graph starts again:

```text
500
600
700
800
900
```

Restarting only resets the process.

It doesn't remove:

```text
memory leak
bad cache
large query
bad code
```

It may be an emergency mitigation, but not the root-cause solution.

---

# 55. Production Example — Memory Leak

### 10:00

```text
Memory = 450Mi
```

### 10:20

```text
Memory = 550Mi
```

### 10:40

```text
Memory = 650Mi
```

### 11:00

```text
Memory = 800Mi
```

### 11:20

```text
Memory = 950Mi
```

Traffic remains:

```text
200 RPS
```

Investigation shows:

```text
Unbounded cache
```

Correct action:

```text
Fix cache
```

not:

```text
Increase memory indefinitely
```

---

# 56. Production Example — Traffic

Another service:

```text
RPS = 100
Memory = 400Mi

RPS = 200
Memory = 500Mi

RPS = 400
Memory = 650Mi

RPS = 800
Memory = 900Mi
```

Testing confirms the behavior is expected.

Correct approach:

```text
HPA
+
appropriate resource sizing
```

---

# 57. Production Example — Node Memory Pressure

Suppose:

```text
node-a
Memory = 95%
MemoryPressure = True
```

Pods:

```text
order-service
payment-service
inventory-service
Fluent Bit
Prometheus
other workloads
```

The investigation should identify:

```text
Which workload increased?
```

Then:

```text
scale node capacity
+
move workloads
+
fix offending workload
```

as appropriate.

---

# 58. Production Example — Bad Deployment

At:

```text
10:00
```

version:

```text
1.8.0
```

Memory:

```text
500Mi
```

At:

```text
10:30
```

version:

```text
1.9.0
```

Memory:

```text
950Mi
```

Rollback:

```text
1.9.0 → 1.8.0
```

Memory:

```text
950Mi → 500Mi
```

Strong evidence of a deployment regression.

---

# 59. Prometheus Memory Alert

Example:

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

The actual production rule should:

* exclude infrastructure containers
* handle missing/zero limits
* identify namespace/service
* include ownership
* include a runbook link

---

# 60. Node Memory Alert

Conceptually:

```text
Node available memory
        ↓
Below safe threshold
        ↓
MemoryPressure risk
```

Alert on:

```text
node memory available
+
MemoryPressure
+
evictions
```

Don't rely on only one metric.

---

# 61. Memory Saturation Runbook

Create:

```text
runbooks/
└── memory-saturation.md
```

Recommended:

```markdown
# Memory Saturation Runbook

## Symptoms

- High memory usage
- Memory near container limit
- Node MemoryPressure
- Pod eviction
- OOMKilled
- Increasing restarts

## Immediate Checks

kubectl top pod -n ecommerce
kubectl top node
kubectl describe pod <pod>
kubectl describe node <node>

## Determine Scope

- container
- Pod
- node
- cluster

## Investigate

- memory trend
- traffic
- payload size
- database results
- cache
- queues
- retries
- runtime memory
- recent deployments
- platform DaemonSets

## Mitigation

- rollback
- scale horizontally
- increase node capacity
- right-size resources
- reduce workload

## Validation

- memory stable
- no eviction
- no OOM
- latency normal
- errors normal
- dependencies healthy

## RCA

Document actual root cause.
```

---

# 62. Memory Decision Tree

```text
                    High Memory
                         |
                         v
              Where is memory pressure?
                         |
        +----------------+----------------+
        |                |                |
     Container          Node            Cluster
        |                |                |
        v                v                v
   Check limit       Check all Pods   Check capacity
        |                |                |
        v                v                v
   Memory trend      MemoryPressure   Node scaling
        |
   +----+----+
   |         |
 Stable    Growing
   |         |
   v         v
Traffic?   Leak?
   |         |
   v      +--+--+
Scale?   |     |
         Yes   No
          |     |
       Fix app  Runtime/
                native/
                dependency
```

---

# 63. Production Checklist

## Container

* [ ] Memory usage checked
* [ ] Memory request checked
* [ ] Memory limit checked
* [ ] Memory trend checked
* [ ] OOMKilled checked
* [ ] Restart count checked

## Node

* [ ] Node memory checked
* [ ] MemoryPressure checked
* [ ] Allocatable memory checked
* [ ] All Pods on node checked
* [ ] DaemonSet overhead checked
* [ ] Evictions checked

## Application

* [ ] Traffic checked
* [ ] Payload size checked
* [ ] Database result size checked
* [ ] Cache checked
* [ ] In-memory queues checked
* [ ] Concurrency checked
* [ ] Retry behavior checked
* [ ] Recent deployment checked

## Runtime

* [ ] Java/JVM checked if applicable
* [ ] Node.js/V8 checked if applicable
* [ ] Python checked if applicable
* [ ] Go checked if applicable
* [ ] .NET checked if applicable

## Dependencies

* [ ] RDS
* [ ] Redis
* [ ] Kafka
* [ ] SQS
* [ ] External APIs

## Mitigation

* [ ] Rollback if required
* [ ] Scale safely
* [ ] Protect dependencies
* [ ] Increase node capacity if required
* [ ] Avoid blind memory increases

## Validation

* [ ] Memory stable
* [ ] No OOMKilled
* [ ] No evictions
* [ ] Latency normal
* [ ] Error rate normal
* [ ] Business transaction successful

## RCA

* [ ] Timeline
* [ ] Impact
* [ ] Root cause
* [ ] Mitigation
* [ ] Permanent fix
* [ ] Prevention

---

# 64. Complete Memory Architecture

```text
                         EKS Cluster
                              |
             +----------------+----------------+
             |                                 |
          Node A                            Node B
             |                                 |
      +------+------+                   +------+------+
      |             |                   |             |
    Pod A         Pod B               Pod C         Pod D
      |             |                   |             |
 Container      Container           Container      Container
      |             |                   |             |
 Application    Application         Application    Application
      |             |                   |             |
      +-------------+-------------------+-------------+
                            |
                     Dependencies
                            |
             +--------------+--------------+
             |              |              |
            RDS           Redis          Kafka/SQS
```

Observability:

```text
Applications
    |
    +---- Prometheus
    |
    +---- Grafana
    |
    +---- ELK
    |
    +---- OpenTelemetry
              |
            Jaeger
```

---

# 65. Relationship With Previous Incidents

Memory saturation connects directly to:

```text
143 CrashLoopBackOff
        ↓
144 OOMKilled
        ↓
145 CPU Saturation
        ↓
146 Memory Saturation
```

They can also create each other.

### Memory → CPU

```text
Memory pressure
    ↓
GC pressure
    ↓
CPU ↑
```

### CPU → Memory

```text
CPU saturation
    ↓
Requests process slowly
    ↓
Concurrency ↑
    ↓
Memory ↑
```

### Dependency → Memory

```text
RDS slow
    ↓
Requests wait
    ↓
Concurrency ↑
    ↓
Memory ↑
```

### Memory → dependency

```text
Pods restart
    ↓
Reconnect storm
    ↓
RDS/Redis connection spike
    ↓
Dependency pressure
```

This is why production troubleshooting must look at the **whole system**, not one metric.

---

# 66. Final Mental Model

When you receive:

```text
Memory Saturation Alert
```

think:

```text
                     MEMORY HIGH
                          |
                          v
              Container or Node?
                    /          \
              Container        Node
                  |              |
                  v              v
             Check limit    Check all Pods
                  |              |
                  v              v
            Memory trend    MemoryPressure
                  |
            +-----+-----+
            |           |
          Stable      Growing
            |           |
            v           v
         Traffic?     Leak?
            |           |
            v           v
        Scale safely  Fix application
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

# 67. Key Rules

### Rule 1

> **Memory saturation is not the same as OOMKilled.**

### Rule 2

> **Always determine whether the problem is container-level or node-level.**

### Rule 3

> **Check memory trends, not just current memory.**

### Rule 4

> **A memory leak should be fixed, not hidden by continuously increasing limits.**

### Rule 5

> **HPA can help traffic-driven memory pressure but does not fix a memory leak.**

### Rule 6

> **Resource requests are critical for Kubernetes capacity planning.**

### Rule 7

> **Java heap is only one part of Java process memory.**

### Rule 8

> **Node memory includes application Pods plus Kubernetes/platform overhead.**

### Rule 9

> **Always check dependencies such as RDS, Redis, Kafka and SQS.**

### Rule 10

> **Mitigate quickly when customers are impacted, but still find the actual root cause.**

---

## Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ✅
147 — Disk Full                    ⏭️
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

**Next: `147 — Disk Full Production Incident`.**
