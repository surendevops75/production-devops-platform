# 33 — Node Exporter

> Production-grade EC2/EKS worker-node monitoring with Prometheus, Grafana, and Alertmanager

---

## 1. Purpose

In our EKS platform, application metrics tell us what is happening **inside the applications**.

Node Exporter tells us what is happening **at the Linux host level**.

Our monitoring chain becomes:

```text
EC2 Worker Node
      │
      ▼
Node Exporter
      │
      ▼
Prometheus
      │
      ├──────────► Grafana
      │
      └──────────► Alertmanager
```

Node Exporter primarily exposes hardware and OS-level metrics such as:

* CPU
* memory
* filesystem
* disk I/O
* network
* load
* system statistics

For EKS, this is important because Kubernetes workloads ultimately run on worker nodes.

---

# 2. Where Node Exporter Fits

Our EKS architecture:

```text
                         AWS
                          │
                          ▼
                         EKS
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       EC2 Node        EC2 Node         EC2 Node
          │               │                │
          ▼               ▼                ▼
    Node Exporter    Node Exporter    Node Exporter
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                     Prometheus
                          │
                    ┌─────┴─────┐
                    ▼           ▼
                 Grafana    Alertmanager
```

There should normally be a Node Exporter instance on every node that we want to monitor.

---

# 3. Why Node Exporter Is Needed

Suppose Kubernetes reports:

```text
Pod:
order-service-abc123

CPU:
80%
```

That tells us about the container.

But we may also need to know:

```text
Node CPU
Node memory
Disk usage
Disk I/O
Network traffic
Filesystem capacity
System load
```

Example:

```text
Pod CPU high
      │
      ▼
Is the application busy?
      │
      ├── Yes → application traffic
      │
      └── No
           │
           ▼
      Is the node overloaded?
```

Node Exporter helps answer that question.

---

# 4. Node Exporter vs kube-state-metrics

These are frequently confused.

### Node Exporter

Measures the **Linux host**.

```text
CPU
Memory
Disk
Filesystem
Network
Load
```

### kube-state-metrics

Measures the **state of Kubernetes objects**.

```text
Deployments
Pods
DaemonSets
StatefulSets
Nodes
Jobs
HPA
ReplicaSets
```

Mental model:

```text
Node Exporter
     ↓
"What is happening on the machine?"

kube-state-metrics
     ↓
"What does Kubernetes believe is happening?"
```

We will cover kube-state-metrics in chapter 34.

---

# 5. Node Exporter vs CloudWatch

AWS already provides EC2 monitoring through CloudWatch.

So why Node Exporter?

Because they provide different perspectives.

```text
CloudWatch
    ↓
AWS / EC2 perspective

Node Exporter
    ↓
Linux operating-system perspective
```

For example, Node Exporter can expose detailed filesystem and Linux metrics that are useful for Kubernetes troubleshooting.

A mature platform can use both.

---

# 6. Deployment Model

Node Exporter should normally run as a:

```text
DaemonSet
```

Why?

Because we want:

```text
1 Node
   ↓
1 Node Exporter Pod
```

If there are:

```text
10 worker nodes
```

we normally expect approximately:

```text
10 Node Exporter Pods
```

subject to scheduling exclusions.

---

# 7. DaemonSet Architecture

```text
EKS
│
├── node-1
│   └── node-exporter
│
├── node-2
│   └── node-exporter
│
├── node-3
│   └── node-exporter
│
└── node-4
    └── node-exporter
```

When a new node joins:

```text
New EC2 node
     │
     ▼
DaemonSet controller
     │
     ▼
Node Exporter automatically scheduled
```

When a node disappears:

```text
Node disappears
     │
     ▼
Node Exporter disappears with it
```

---

# 8. Production Installation

For Kubernetes, the preferred production approach is generally:

```text
Helm
+
Prometheus Operator / kube-prometheus-stack
+
GitOps
```

Rather than:

```text
SSH into EC2
↓
Install binary manually
```

Manual installation is useful for understanding the software, but not for managing an EKS fleet.

---

# 9. kube-prometheus-stack

Our monitoring platform can deploy Node Exporter through the Prometheus ecosystem.

Conceptually:

```text
kube-prometheus-stack
│
├── Prometheus
├── Alertmanager
├── Grafana
├── Node Exporter
├── kube-state-metrics
└── Prometheus Operator
```

This is convenient because the monitoring components can be integrated through the same Kubernetes monitoring model.

---

# 10. Verify Node Exporter

After installation:

```bash
kubectl get daemonset -n monitoring
```

Example:

```text
NAME             DESIRED   CURRENT   READY
node-exporter    6         6         6
```

Then:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter
```

You should see one Pod per eligible node.

---

# 11. Verify Node Coverage

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get pods -n monitoring -o wide
```

You should be able to correlate:

```text
Node
 ↓
Node Exporter Pod
```

Example:

```text
ip-10-0-11-25
   ↓
node-exporter-x82kd

ip-10-0-12-31
   ↓
node-exporter-p91lm
```

---

# 12. Prometheus Scraping

Node Exporter exposes metrics over HTTP.

Typically:

```text
/metrics
```

Prometheus discovers the Node Exporter targets through the Kubernetes monitoring configuration.

Architecture:

```text
Node Exporter
     │
     │ HTTP /metrics
     ▼
Prometheus
```

Prometheus periodically scrapes:

```text
GET /metrics
```

---

# 13. Example Metrics

You will see metrics similar to:

```text
node_cpu_seconds_total
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
node_filesystem_size_bytes
node_disk_read_bytes_total
node_disk_written_bytes_total
node_network_receive_bytes_total
node_network_transmit_bytes_total
```

The exact available collectors and metric set depend on the Node Exporter version/configuration.

---

# 14. CPU Monitoring

One of the most common metrics:

```text
node_cpu_seconds_total
```

This is a counter.

We generally calculate CPU utilization from the rate of CPU time consumption.

Example:

```promql
100 *
(
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{
      mode="idle"
    }[5m])
  )
)
```

This gives approximate CPU utilization.

---

# 15. Why `rate()` Is Needed

`node_cpu_seconds_total` is a counter.

It continually increases.

Example:

```text
10
20
30
40
50
```

The raw value doesn't directly mean:

```text
50% CPU
```

We care about the **rate of change**.

Therefore:

```promql
rate(node_cpu_seconds_total[5m])
```

calculates the average increase over the window.

---

# 16. CPU by Mode

Linux CPU time can be divided into modes such as:

```text
user
system
idle
iowait
steal
nice
irq
softirq
```

This can help troubleshoot.

For example:

```text
High user CPU
```

may indicate application/process workload.

While:

```text
High iowait
```

may suggest storage I/O pressure.

---

# 17. Memory

Important metrics include:

```text
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes
```

A useful utilization query:

```promql
100 *
(
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

This gives approximate host memory utilization.

---

# 18. Memory Is Different From Kubernetes Memory

This distinction is extremely important.

Node Exporter:

```text
EC2 node memory
```

Container metrics:

```text
Pod/container memory
```

Kubernetes scheduling:

```text
Pod requests/limits
```

These are different layers.

Example:

```text
Node has 32 GiB RAM
       │
       ├── Kubernetes/system processes
       ├── kubelet
       ├── container runtime
       ├── DaemonSets
       └── Application Pods
```

Therefore:

```text
Pod memory
≠
Node memory
```

---

# 19. Filesystem Monitoring

Disk capacity problems are common production incidents.

Useful metrics:

```text
node_filesystem_size_bytes
node_filesystem_avail_bytes
node_filesystem_free_bytes
```

Example:

```promql
100 *
(
  1 -
  node_filesystem_avail_bytes{
    mountpoint="/"
  }
  /
  node_filesystem_size_bytes{
    mountpoint="/"
  }
)
```

---

# 20. Disk Full Incident

Suppose:

```text
/ filesystem = 98%
```

Possible consequences:

```text
Logs cannot be written
Container runtime problems
Application failures
Kubelet problems
Image pulls fail
Temporary files fail
```

This is why filesystem alerts are important.

---

# 21. Inode Exhaustion

Disk capacity is not the only problem.

Linux filesystems also have:

```text
inodes
```

You can have:

```text
Disk:
40% used
```

but:

```text
Inodes:
100% used
```

Then new files cannot be created.

Monitor:

```text
node_filesystem_files
node_filesystem_files_free
```

Example:

```promql
100 *
(
  1 -
  node_filesystem_files_free
  /
  node_filesystem_files
)
```

This is a classic production troubleshooting detail.

---

# 22. Container Logs and Disk Pressure

Consider:

```text
Application generates huge logs
        ↓
Container logs
        ↓
Node filesystem
        ↓
Disk usage increases
        ↓
DiskPressure
```

Therefore:

```text
Application logging
        +
Node filesystem monitoring
```

are related.

Our Fluent Bit logging architecture must also prevent uncontrolled log growth.

---

# 23. Disk I/O

Node Exporter can provide disk I/O metrics.

Useful concepts:

```text
read bytes
write bytes
read operations
write operations
I/O time
```

High I/O can affect:

```text
Database workloads
Logging
Container runtime
Image extraction
Application temporary files
```

---

# 24. I/O Wait

Suppose CPU utilization appears moderate:

```text
CPU = 40%
```

but:

```text
iowait = very high
```

Applications may still experience latency.

This is why simply watching:

```text
CPU %
```

is insufficient.

---

# 25. Network Metrics

Node Exporter exposes network metrics such as:

```text
node_network_receive_bytes_total
node_network_transmit_bytes_total
node_network_receive_packets_total
node_network_transmit_packets_total
```

Use these to investigate:

```text
Network throughput
Packet rates
Errors
Drops
Interface behavior
```

---

# 26. Network Errors

Monitor:

```text
receive errors
transmit errors
receive drops
transmit drops
```

Potential causes can include:

```text
Interface issues
Network saturation
Driver problems
Underlying infrastructure issues
Configuration problems
```

In AWS, correlate these metrics with:

```text
CloudWatch
VPC
ENI
Security Groups
Load Balancers
```

---

# 27. EKS VPC CNI Context

EKS uses the AWS VPC CNI for Pod networking.

Therefore, node-level monitoring must be combined with CNI metrics.

Example incident:

```text
CPU healthy
Memory healthy
Disk healthy
```

but:

```text
New Pods cannot start
```

One possibility:

```text
Pod IP exhaustion
```

This is not something Node Exporter alone can diagnose.

So:

```text
Node Exporter
+
VPC CNI metrics
+
Kubernetes metrics
```

provide a better picture.

---

# 28. Load Average

Linux load average indicates runnable/uninterruptible work.

Typical metrics include:

```text
node_load1
node_load5
node_load15
```

Example:

```promql
node_load1
```

But don't interpret load without considering:

```text
number of CPU cores
CPU utilization
I/O wait
workload type
```

A load average of:

```text
8
```

means something very different on:

```text
2-core node
```

versus:

```text
32-core node
```

---

# 29. Node Exporter Collectors

Node Exporter uses collectors to gather different categories of metrics.

Conceptually:

```text
Node Exporter
│
├── CPU
├── Memory
├── Filesystem
├── Disk
├── Network
├── Load
├── Processes
└── Other Linux collectors
```

Do not enable every possible collector blindly.

More collectors can mean:

```text
More metrics
+
More scrape cost
+
More cardinality
```

Enable what you actually need.

---

# 30. Node Exporter Resource Usage

Node Exporter itself consumes resources.

In production, set resource requests/limits appropriately.

Example:

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi

  limits:
    cpu: 200m
    memory: 256Mi
```

These are illustrative values, not universal production defaults.

Measure the actual workload.

---

# 31. Priority

Node Exporter is part of the observability infrastructure.

If application workloads consume all node resources, monitoring Pods could also become unhealthy.

Therefore:

```text
Observability workloads
+
System workloads
```

should have appropriate scheduling priority and capacity planning.

A dedicated monitoring node pool can be useful for larger clusters.

---

# 32. Dedicated Monitoring Nodes

For a larger EKS cluster:

```text
EKS
│
├── system node pool
│
├── application node pool
│
└── monitoring node pool
      ├── Prometheus
      ├── Grafana
      ├── Alertmanager
      └── other monitoring workloads
```

But Node Exporter is different.

Because it monitors nodes, we usually want it on the nodes being monitored.

If a node is intentionally excluded from Node Exporter, make sure the monitoring coverage is understood.

---

# 33. Taints and Tolerations

Suppose monitoring nodes have:

```text
taint:
dedicated=monitoring:NoSchedule
```

Then monitoring workloads need a matching toleration.

Example:

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: monitoring
    effect: NoSchedule
```

Again, the exact strategy depends on cluster design.

---

# 34. Node Exporter Security

Node Exporter needs visibility into host-level information.

Therefore, its DaemonSet commonly requires host access such as:

```text
host filesystem
host network namespace
```

depending on the deployment configuration.

This makes security important.

Do not treat Node Exporter as an ordinary application Pod.

---

# 35. Host Filesystem Access

Node Exporter may need to read host filesystem information.

Conceptually:

```text
Node Exporter Pod
       │
       ▼
Host filesystem
       │
       ▼
Linux metrics
```

Any hostPath or privileged-like capability should be reviewed carefully.

Use the minimum access required by the selected version/configuration.

---

# 36. Service Exposure

Node Exporter normally does not need to be exposed publicly.

Architecture:

```text
Node Exporter
      │
      ▼
Cluster/internal Service
      │
      ▼
Prometheus
```

Do not create:

```text
Internet
   ↓
Node Exporter
```

There is normally no reason for that.

---

# 37. NetworkPolicy

A useful security model:

```text
Prometheus
    │
    │ allowed
    ▼
Node Exporter
```

Other application namespaces:

```text
Application
    X
    │
    ▼
Node Exporter
```

Only the monitoring components that need access should be allowed.

---

# 38. Prometheus Target Discovery

If using Prometheus Operator, Node Exporter is typically discovered through Kubernetes monitoring resources/configuration.

Conceptually:

```text
DaemonSet
   ↓
Service
   ↓
ServiceMonitor
   ↓
Prometheus
```

The exact resource names depend on the Helm chart.

Check:

```bash
kubectl get servicemonitors -n monitoring
```

and:

```bash
kubectl get endpoints -n monitoring
```

---

# 39. Troubleshooting — Target Missing

Suppose:

```text
Node Exporter Pod = Running
```

but:

```text
Prometheus = no target
```

Check in order:

```text
1. Pod
2. Service
3. Endpoints
4. ServiceMonitor
5. Prometheus selector
6. Network connectivity
```

Commands:

```bash
kubectl get pods -n monitoring
```

```bash
kubectl get svc -n monitoring
```

```bash
kubectl get endpoints -n monitoring
```

```bash
kubectl get servicemonitor -n monitoring
```

---

# 40. Troubleshooting — Node Exporter Pod Missing

Check:

```bash
kubectl get daemonset -n monitoring
```

Then:

```bash
kubectl describe daemonset <name> -n monitoring
```

Possible causes:

```text
Node selector
Taints
Tolerations
Affinity
Resource constraints
Scheduling restrictions
```

---

# 41. Troubleshooting — Pod Running but No Metrics

Check the Pod:

```bash
kubectl logs -n monitoring <node-exporter-pod>
```

Then test the metrics endpoint from inside the cluster if appropriate:

```bash
kubectl exec -n monitoring <debug-pod> -- \
  curl http://<node-exporter-service>:9100/metrics
```

If metrics are available:

```text
Node Exporter
     ↓
working
```

Then investigate:

```text
Prometheus discovery
ServiceMonitor
network policy
```

---

# 42. Troubleshooting — Filesystem Metrics Missing

Check:

```text
mount configuration
filesystem type
collector configuration
container mounts
exclude/include filters
```

Some filesystems should intentionally be excluded.

For example:

```text
temporary virtual filesystems
container-specific mounts
```

may create noise.

---

# 43. Troubleshooting — CPU Looks Wrong

If CPU values look strange:

```text
Check instance
Check CPU count
Check mode
Check rate interval
Check dashboard query
```

Don't immediately assume Node Exporter is broken.

A dashboard query can easily calculate the wrong percentage.

---

# 44. Troubleshooting — Node Exporter High CPU

Node Exporter should normally be lightweight.

If CPU becomes unexpectedly high:

```text
Check enabled collectors
Check scrape interval
Check metric volume
Check number of CPUs
Check Prometheus scrape behavior
```

Also inspect:

```bash
kubectl top pod -n monitoring
```

---

# 45. Alert — Node Not Ready

Node readiness itself is generally better detected using Kubernetes state metrics.

But the investigation can use Node Exporter.

Example:

```text
NodeNotReady
     │
     ▼
Grafana
     │
     ├── Node CPU
     ├── Memory
     ├── Disk
     ├── Network
     └── System load
```

This demonstrates why:

```text
kube-state-metrics
+
Node Exporter
```

work together.

---

# 46. Alert — Filesystem Nearly Full

Example conceptual alert:

```yaml
- alert: NodeFilesystemAlmostFull
  expr: |
    (
      node_filesystem_avail_bytes{
        mountpoint="/",
        fstype!~"tmpfs|overlay"
      }
      /
      node_filesystem_size_bytes{
        mountpoint="/",
        fstype!~"tmpfs|overlay"
      }
    ) < 0.10

  for: 10m

  labels:
    severity: warning

  annotations:
    summary: "Node filesystem is almost full"
    description: >
      Less than 10% filesystem capacity remains.
```

The filesystem filters should be adapted to the actual environment.

---

# 47. Alert — Inodes Nearly Exhausted

Conceptually:

```yaml
- alert: NodeInodesAlmostFull
  expr: |
    (
      node_filesystem_files_free
      /
      node_filesystem_files
    ) < 0.10

  for: 10m

  labels:
    severity: warning
```

This catches a problem that disk-capacity alerts can miss.

---

# 48. Alert — High Node CPU

Example:

```yaml
- alert: NodeHighCPU
  expr: |
    100 *
    (
      1 -
      avg by (instance) (
        rate(node_cpu_seconds_total{
          mode="idle"
        }[5m])
      )
    ) > 90

  for: 10m

  labels:
    severity: warning
```

But remember:

```text
High CPU
```

is not automatically:

```text
Page the on-call engineer
```

The right severity depends on impact.

---

# 49. Alert — High Memory

Example:

```yaml
- alert: NodeHighMemory
  expr: |
    100 *
    (
      1 -
      node_memory_MemAvailable_bytes
      /
      node_memory_MemTotal_bytes
    ) > 90

  for: 10m

  labels:
    severity: warning
```

Correlate this with:

```text
Kubernetes memory pressure
Pod OOMKilled
Evictions
Scheduling
```

---

# 50. Real Production Scenario — Memory Pressure

Suppose:

```text
Node memory
92%
```

Grafana shows:

```text
Memory ↑
```

Then Kubernetes:

```text
MemoryPressure = true
```

Then:

```text
Pods begin eviction
```

Then:

```text
Applications restart
```

The troubleshooting chain becomes:

```text
Node Exporter
      ↓
Memory pressure
      ↓
kube-state-metrics
      ↓
Pod eviction
      ↓
Application metrics
      ↓
Errors/latency
```

---

# 51. Real Production Scenario — Disk Pressure

```text
Node filesystem
       ↓
95%
       ↓
Logs increase
       ↓
Fluent Bit processing
       ↓
Node disk fills
       ↓
Kubernetes DiskPressure
       ↓
Pod eviction
       ↓
Application outage
```

This is why observability components must be considered together.

---

# 52. Real Production Scenario — Network Problem

Suppose:

```text
Application latency ↑
```

but:

```text
CPU normal
Memory normal
```

Node Exporter:

```text
Network transmit errors ↑
```

Then investigate:

```text
Node network
VPC
ENI
Security groups
CNI
Load balancer
AWS networking
```

Metrics narrow the investigation.

---

# 53. Grafana Node Dashboard

A useful dashboard:

```text
Node Overview
```

Panels:

```text
CPU utilization
Memory utilization
Load average
Filesystem utilization
Inode utilization
Disk read throughput
Disk write throughput
Network receive
Network transmit
Network errors
Network drops
```

---

# 54. Node Dashboard Variables

Use variables:

```text
cluster
environment
node
instance
```

Example:

```text
Environment: production
Node:        ip-10-0-11-25
```

Then all panels update for that node.

---

# 55. Node Capacity Dashboard

Create a separate dashboard:

```text
EKS Node Capacity
```

Include:

```text
Total nodes
Ready nodes
CPU allocatable
CPU requested
CPU utilization
Memory allocatable
Memory requested
Memory utilization
Pod count
Pod capacity
```

Combine:

```text
Node Exporter
+
kube-state-metrics
```

because capacity and actual OS usage are different measurements.

---

# 56. Node Exporter and HPA

HPA generally scales Pods based on metrics associated with workloads.

Node Exporter does not directly mean:

```text
Node CPU → HPA
```

Instead:

```text
Application metrics
       ↓
HPA
       ↓
Pod replicas
```

Separately:

```text
Node CPU
       ↓
Cluster capacity monitoring
       ↓
Node scaling / Karpenter / node group
```

So:

```text
HPA
=
Pod scaling

Node autoscaling
=
Node capacity scaling
```

---

# 57. Node Exporter and Karpenter

Our production EKS architecture may use:

```text
HPA
+
Karpenter
```

Flow:

```text
Traffic ↑
   ↓
HPA
   ↓
More Pods
   ↓
Insufficient node capacity
   ↓
Pending Pods
   ↓
Karpenter
   ↓
New EC2 node
   ↓
Node Exporter automatically appears
```

This creates a useful observability chain.

---

# 58. Node Exporter and Cluster Autoscaler

If using Cluster Autoscaler instead:

```text
Pending Pods
   ↓
Cluster Autoscaler
   ↓
Node group scales
   ↓
New node
   ↓
DaemonSet
   ↓
Node Exporter
```

The monitoring model remains the same.

---

# 59. Node Exporter Does Not Monitor Everything

Node Exporter does **not** replace:

```text
Prometheus
kube-state-metrics
AWS CloudWatch
Application metrics
Fluent Bit
OpenTelemetry
```

Each operates at a different layer.

```text
Application
   ↓
Micrometer

Container/Kubernetes
   ↓
kube-state-metrics

Host
   ↓
Node Exporter

AWS
   ↓
CloudWatch

Logs
   ↓
Fluent Bit / Logstash

Traces
   ↓
OpenTelemetry / Jaeger
```

---

# 60. Production Tool Placement

| Tool               | Where it runs            | Purpose                    |
| ------------------ | ------------------------ | -------------------------- |
| Node Exporter      | EKS nodes as DaemonSet   | Linux host metrics         |
| kube-state-metrics | EKS monitoring namespace | Kubernetes object metrics  |
| Prometheus         | EKS monitoring           | Metrics collection/storage |
| Grafana            | EKS monitoring           | Visualization              |
| Alertmanager       | EKS monitoring           | Alert routing              |
| Fluent Bit         | EKS nodes as DaemonSet   | Log collection             |
| Logstash           | EKS logging namespace    | Log processing             |
| Elasticsearch      | Logging platform         | Log storage/search         |
| Kibana             | Logging platform         | Log visualization          |
| OpenTelemetry      | EKS/apps                 | Telemetry                  |
| Jaeger             | Tracing platform         | Trace storage/query        |

---

# 61. Production Checklist

### Deployment

* [ ] DaemonSet deployed
* [ ] One exporter per monitored node
* [ ] Version pinned
* [ ] Helm/GitOps managed
* [ ] Resource requests configured
* [ ] Scheduling reviewed

### Metrics

* [ ] CPU
* [ ] Memory
* [ ] Filesystem
* [ ] Inodes
* [ ] Disk I/O
* [ ] Network
* [ ] Load
* [ ] Required collectors only

### Prometheus

* [ ] Targets discovered
* [ ] Scrapes successful
* [ ] ServiceMonitor/configuration correct
* [ ] Recording rules where useful
* [ ] Cardinality reviewed

### Security

* [ ] No public exposure
* [ ] NetworkPolicy
* [ ] Host access reviewed
* [ ] Minimal privileges
* [ ] Approved image/version

### Alerting

* [ ] High CPU
* [ ] High memory
* [ ] Filesystem exhaustion
* [ ] Inode exhaustion
* [ ] Network errors
* [ ] Node-level capacity alerts

---

# 62. Final Architecture

Our node monitoring path is:

```text
                       AWS EKS
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
           Node 1       Node 2       Node 3
             │            │            │
             ▼            ▼            ▼
        Node Exporter Node Exporter Node Exporter
             │            │            │
             └────────────┼────────────┘
                          ▼
                     Prometheus
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
           Grafana               Alertmanager
              │                       │
              ▼                       ▼
       Investigation              Notification
```

And the complete observability model is now:

```text
                    EKS PLATFORM
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
 Applications          Nodes           Kubernetes
       │                 │                  │
       ▼                 ▼                  ▼
 Micrometer        Node Exporter     kube-state-metrics
       │                 │                  │
       └─────────────────┼──────────────────┘
                         ▼
                    Prometheus
                         │
                  ┌──────┴──────┐
                  ▼             ▼
               Grafana     Alertmanager
                  │             │
                  ▼             ▼
             Investigation   Notification
```

The key idea:

> **Node Exporter tells us what is happening on the worker machine; Kubernetes metrics tell us what Kubernetes thinks is happening; application metrics tell us what the application is experiencing.**

You need all three perspectives when troubleshooting a production EKS incident.

---

## Next

**`34-kube-state-metrics.md`**

That chapter will cover the Kubernetes-control-plane/object perspective:

```text
Kubernetes API
      ↓
kube-state-metrics
      ↓
Prometheus
      ↓
Grafana
      ↓
Alertmanager
```

We'll go through **Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, HPA, PDB, node conditions, desired vs available replicas, scheduling failures, CrashLoopBackOff detection, Kubernetes capacity dashboards, production alerts, and real EKS troubleshooting scenarios**.
