# 147 — Disk Full Production Incident

> **Phase 18 — Incident Management**
>
> This chapter covers a production-grade **Disk Full incident** in Kubernetes/EKS. The investigation is language-independent first, followed by Java/Spring Boot-specific examples because the project uses Java 21 + Spring Boot.

---

## 1. What Is a Disk Full Incident?

A Disk Full incident occurs when a filesystem, volume, or node's available storage becomes critically low or reaches 100%.

In Kubernetes, this can affect:

* Application containers
* Pods
* Nodes
* `emptyDir` volumes
* Container writable layers
* Container images
* Application logs
* Kubernetes runtime storage
* Persistent Volumes
* Prometheus
* Elasticsearch
* Logstash
* Jaeger
* Temporary files

A disk problem can eventually become a **cluster-level reliability problem**.

Example:

```text
Application generates large files
        ↓
Node filesystem fills
        ↓
DiskPressure = True
        ↓
Kubernetes starts evicting Pods
        ↓
Pods become unstable
        ↓
New images cannot be pulled
        ↓
Applications fail to start
        ↓
Customer impact
```

---

# 2. Disk Full Is a Symptom, Not the Root Cause

Do not stop investigation at:

```text
Disk is 100% full
```

The real question is:

> **What consumed the disk, where did it consume it, and why was it allowed to grow?**

Possible root causes:

```text
Large application logs
Large temporary files
Heap dump
JFR recording
Container images
Container writable layer
emptyDir
Log buffers
Elasticsearch data
Prometheus WAL
Large application-generated reports
Core dumps
Old files
Deleted files still held open
Filesystem inode exhaustion
PVC capacity exhausted
EBS volume capacity exhausted
Bad retention policy
Runaway application
```

---

# 3. Different Types of Disk Problems

This distinction is extremely important.

## 3.1 Node Root Disk Full

Example:

```text
EKS Node
   |
   +-- /
       |
       +-- /var/log
       +-- /var/lib/kubelet
       +-- container runtime
       +-- system files
       +-- images
```

If `/` reaches 100%, the node can become unhealthy.

---

# 3.2 Container Writable Layer Full

A container can generate temporary files inside its writable filesystem.

Example:

```text
order-service container
        |
        +-- /tmp
        +-- /app/tmp
        +-- generated reports
        +-- temporary files
```

If the application continuously writes files, ephemeral storage can be exhausted.

---

# 3.3 `emptyDir` Full

Example:

```yaml
volumes:
  - name: temp-data
    emptyDir: {}
```

Application:

```text
/tmp
```

may be backed by an `emptyDir` volume.

If the application writes:

```text
/tmp/report1
/tmp/report2
/tmp/report3
...
```

storage can eventually be exhausted.

---

# 3.4 PVC Full

A PersistentVolumeClaim can also reach capacity.

Example:

```text
Pod
 |
 +--> PVC
       |
       +--> EBS
             |
             +--> filesystem
```

Example:

```text
PVC size: 100Gi
Used:     100Gi
Free:       0Gi
```

Kubernetes may still show the PVC as:

```text
Bound
```

even though the filesystem inside the volume is full.

---

# 3.5 Inode Exhaustion

This is often missed.

You can have:

```text
Disk space:
20% used
```

but:

```text
Inodes:
100% used
```

because the system contains millions of tiny files.

Example:

```text
/tmp/file-000001
/tmp/file-000002
/tmp/file-000003
...
```

So always check both:

```bash
df -h
```

and:

```bash
df -i
```

---

# 4. Kubernetes DiskPressure

Kubernetes monitors node filesystem pressure.

Check:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Conditions:

DiskPressure   True
```

Example:

```text
Conditions:
  Type             Status
  ----             ------
  MemoryPressure   False
  DiskPressure     True
  PIDPressure      False
  Ready            True
```

This means the node is experiencing filesystem pressure.

---

# 5. Why DiskPressure Is Dangerous

When disk pressure becomes severe, Kubernetes may:

* Evict Pods
* Prevent new Pods from scheduling
* Affect image pulls
* Affect container creation
* Affect kubelet operations
* Affect application writes
* Cause cascading failures

Example:

```text
Node disk fills
      ↓
DiskPressure
      ↓
Pod eviction
      ↓
Pods restart elsewhere
      ↓
More image pulls
      ↓
Additional disk/network pressure
```

---

# 6. Production Incident Scenario

Let's use a realistic example.

### Incident

`order-service` starts generating large PDF reports locally.

The application writes:

```text
/tmp/order-report-*.pdf
```

Each report:

```text
500 MB
```

Traffic increases.

After several hours:

```text
Node filesystem:
85%
   ↓
92%
   ↓
97%
   ↓
100%
```

Kubernetes reports:

```text
DiskPressure=True
```

Then unrelated Pods start getting evicted.

Some new Pods fail with:

```text
ImagePullBackOff
```

The incident initially looks like multiple application failures.

But the common root cause is:

```text
Node filesystem exhaustion
```

---

# 7. Production Incident Flow

Use this sequence:

```text
Alert
  ↓
Confirm customer impact
  ↓
Determine node/PVC/container scope
  ↓
Check DiskPressure
  ↓
Check filesystem usage
  ↓
Check inode usage
  ↓
Identify largest consumers
  ↓
Check recent deployments
  ↓
Check application logs/files/temp data
  ↓
Check container runtime/image storage
  ↓
Check observability storage
  ↓
Mitigate
  ↓
Restore capacity
  ↓
Validate workloads
  ↓
Identify root cause
  ↓
Prevent recurrence
  ↓
RCA
```

---

# 8. Step 1 — Check Pods

Start with:

```bash
kubectl get pods -A -o wide
```

Look for:

```text
Evicted
CrashLoopBackOff
ContainerCreating
ImagePullBackOff
Pending
```

Example:

```text
order-service-7d6f8c8f7b-x2abc     Evicted
payment-service-6c7d8f9f5d-k91xy    Running
catalog-service-7f8d9c6d4b-q2abc    ImagePullBackOff
```

The failures may initially appear unrelated.

---

# 9. Step 2 — Check Nodes

```bash
kubectl get nodes
```

Then:

```bash
kubectl describe node <node-name>
```

Look for:

```text
DiskPressure
```

Also inspect:

```text
Conditions
Allocated resources
Events
```

---

# 10. Step 3 — Check Node Events

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

Look for messages such as:

```text
DiskPressure
Evicted
ephemeral-storage
image filesystem
node has disk pressure
```

Example:

```text
Node has disk pressure
Pod was evicted
```

---

# 11. Step 4 — SSH Into the Node

If your operational access model permits SSH/SSM access:

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       80G   79G  1G    99% /
```

This immediately confirms filesystem exhaustion.

---

# 12. Step 5 — Check Inodes

Run:

```bash
df -i
```

Example:

```text
Filesystem      Inodes   IUsed   IFree IUse% Mounted on
/dev/root       5.0M     5.0M       0  100% /
```

This is a different failure mode.

You might have:

```text
df -h → 30% used
df -i → 100% used
```

In this situation, the problem is too many files, not too much total storage.

---

# 13. Step 6 — Identify Large Directories

Use:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Then inspect suspicious directories.

For example:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Then:

```bash
sudo du -xhd1 /var/log 2>/dev/null | sort -h
```

Example:

```text
/var/log              25G
/var/lib              45G
/home                  2G
/opt                   1G
```

Now investigate:

```text
/var/lib
```

---

# 14. Check `/var/log`

```bash
sudo du -sh /var/log/* 2>/dev/null
```

Potential result:

```text
2G   /var/log/journal
18G  /var/log/application.log
3G   /var/log/messages
```

A runaway log file may be the root cause.

---

# 15. Check Systemd Journal

On systems using systemd:

```bash
journalctl --disk-usage
```

Example:

```text
Archived and active journals take up 12.4G in the file system.
```

This is an important clue.

---

# 16. Check Large Files

A useful investigation command:

```bash
sudo find / -xdev -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

Example:

```text
/var/log/order-service.log     8G
/tmp/report.zip                4G
/tmp/heapdump.hprof            6G
```

Now we have a likely root cause.

---

# 17. Check Deleted Files Still Held Open

This is an advanced production troubleshooting case.

A process can keep a deleted file open.

Example:

```text
application.log = 10 GB
```

Someone deletes it:

```bash
rm application.log
```

But the process still has the file open.

The disk space may not be released until the process closes the file descriptor.

Check:

```bash
sudo lsof +L1
```

Example:

```text
java    1234
/order-service.log (deleted)
```

This can explain:

```text
df -h → 95%
```

while:

```text
du → much lower
```

That mismatch is an important diagnostic signal.

---

# 18. Check Container Runtime Storage

EKS commonly uses containerd.

Investigate runtime storage carefully.

Possible locations include:

```text
/var/lib/containerd
/var/lib/kubelet
```

Check sizes:

```bash
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
```

Then:

```bash
sudo du -xhd1 /var/lib/containerd 2>/dev/null | sort -h
```

And:

```bash
sudo du -xhd1 /var/lib/kubelet 2>/dev/null | sort -h
```

### Important Production Rule

**Do not manually delete files from:**

```text
/var/lib/containerd
/var/lib/kubelet
```

or other container-runtime internals.

Do not run random:

```bash
rm -rf /var/lib/containerd/*
```

This can make the node worse.

Use supported container runtime/image garbage-collection mechanisms and your node lifecycle process.

---

# 19. Check Kubernetes Ephemeral Storage

Kubernetes supports resource requests and limits for ephemeral storage.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
    ephemeral-storage: "1Gi"

  limits:
    cpu: "1"
    memory: "1Gi"
    ephemeral-storage: "2Gi"
```

This communicates storage expectations to Kubernetes.

---

# 20. Why `ephemeral-storage` Matters

A Pod may consume storage through:

```text
Container writable layer
emptyDir
container logs
temporary files
```

Example:

```text
order-service
     |
     +-- /tmp
     |
     +-- /app/tmp
     |
     +-- generated files
```

Without appropriate controls, a workload can consume disproportionate node storage.

---

# 21. `emptyDir` Storage

Example:

```yaml
volumes:
  - name: workdir
    emptyDir: {}
```

Container:

```yaml
volumeMounts:
  - name: workdir
    mountPath: /work
```

Application writes:

```text
/work/file1
/work/file2
/work/file3
```

These files consume node ephemeral storage unless using a memory-backed `emptyDir`.

---

# 22. `emptyDir` With Size Limit

Example:

```yaml
volumes:
  - name: workdir
    emptyDir:
      sizeLimit: 2Gi
```

This provides an explicit bound for the workload.

Still, application design must ensure the workload can operate within that limit.

---

# 23. Check Pod Ephemeral Storage

Inspect the workload:

```bash
kubectl describe pod <pod-name> -n ecommerce
```

Check:

```text
Requests
Limits
Volumes
Events
```

For supported metrics environments, also inspect your metrics system for container filesystem/ephemeral-storage metrics.

Do not assume `kubectl top` exposes every storage metric on every cluster/version.

---

# 24. PVC Disk Full

Now consider a different incident.

```text
order-service
     |
     +-- PVC
           |
           +-- EBS
```

Check:

```bash
kubectl get pvc -n ecommerce
```

Example:

```text
NAME              STATUS   VOLUME       CAPACITY
postgres-data     Bound    pvc-123       100Gi
```

The PVC can be:

```text
Bound
```

while the filesystem inside it is completely full.

---

# 25. Check PVC Details

```bash
kubectl describe pvc postgres-data -n ecommerce
```

Then inspect the filesystem from the Pod where appropriate:

```bash
kubectl exec -it <pod-name> -n ecommerce -- df -h
```

Example:

```text
Filesystem      Size  Used Avail Use%
/dev/nvme1n1    100G  100G  0G   100%
```

Now the problem is storage capacity on the mounted filesystem.

---

# 26. EBS vs Filesystem

Important distinction:

```text
EBS volume
    ↓
Filesystem
    ↓
Application
```

Increasing the EBS volume is not always the entire operation.

Depending on the filesystem and setup, the filesystem may also need to be expanded.

Kubernetes/EBS CSI volume expansion should be performed through the supported StorageClass/PVC workflow.

Do not manually resize production storage without understanding the storage layer.

---

# 27. Observability Can Cause Disk Problems

Your platform itself can consume significant storage.

Architecture:

```text
Applications
    |
    +--> Fluent Bit
           |
           +--> Logstash
                   |
                   +--> Elasticsearch
```

And:

```text
Applications
    |
    +--> Prometheus
```

And:

```text
Applications
    |
    +--> OpenTelemetry Collector
            |
            +--> Jaeger
```

Each component has different storage requirements.

---

# 28. Elasticsearch Disk Full

Elasticsearch/OpenSearch nodes can run out of disk.

Possible causes:

```text
Too many logs
Long retention
Too many shards
Large documents
No lifecycle policy
High replica count
Unexpected traffic
```

This can lead to:

```text
cluster health problems
index write failures
log ingestion failures
```

Therefore:

```text
ELK disk capacity
```

must be monitored separately from application node disk.

---

# 29. Prometheus Disk Full

Prometheus stores:

* Time-series data
* WAL
* Blocks
* Metadata

A Prometheus PVC can fill due to:

```text
High retention
High cardinality
Unexpected metric volume
Large scrape targets
Bad labels
```

Do not simply increase retention forever.

Investigate:

```text
retention
cardinality
scrape volume
metric labels
storage sizing
```

---

# 30. Jaeger Storage

Jaeger storage can also grow based on:

```text
Trace volume
Sampling rate
Retention
Span size
Number of services
```

This is one reason production tracing requires controlled sampling and storage lifecycle policies.

---

# 31. Java/Spring Boot Specific Disk Problems

Our project uses:

```text
Java 21
Spring Boot
Maven
Docker
Kubernetes
```

Java applications can consume disk in several ways.

---

## 31.1 Log Files

If the application writes file-based logs:

```text
/app/logs/order-service.log
```

the file can grow indefinitely if rotation is misconfigured.

Prefer centralized logging through:

```text
stdout/stderr
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
```

rather than relying on large local application log files.

---

# 32. Heap Dumps

Java can create a large heap dump.

Example:

```text
-XX:+HeapDumpOnOutOfMemoryError
```

A heap dump may be several GB.

Example:

```text
/tmp/java_pid1234.hprof
```

If multiple dumps accumulate:

```text
heapdump1.hprof
heapdump2.hprof
heapdump3.hprof
```

disk consumption can become significant.

Heap dumps are valuable for diagnosis, but their storage destination and retention must be controlled.

---

# 33. JFR Recordings

Java Flight Recorder can create recordings.

Example:

```text
order-service.jfr
```

Long or repeated recordings can consume significant storage.

Therefore production diagnostics should include:

```text
duration
file size
destination
retention
cleanup
```

---

# 34. Java Temporary Files

Java uses temporary storage.

Check:

```bash
java -XshowSettings:properties -version 2>&1 | grep java.io.tmpdir
```

Typically:

```text
java.io.tmpdir = /tmp
```

Applications may write:

```text
/tmp/uploads
/tmp/reports
/tmp/export
```

If cleanup is missing, storage grows continuously.

---

# 35. Large File Processing

Bad pattern:

```text
Download 10 GB file
       ↓
Store locally
       ↓
Process
       ↓
Delete later
```

Better architecture:

```text
S3
 |
 +--> streaming processing
 |
 +--> temporary bounded workspace
 |
 +--> delete temporary data
```

For large files, object storage such as S3 is usually more appropriate than indefinite node-local storage.

---

# 36. Other Runtime Examples

Disk problems are not Java-specific.

### Node.js

Possible sources:

```text
npm cache
temporary files
uploaded files
logs
generated reports
core dumps
```

### Python

Possible sources:

```text
/tmp
ML/model artifacts
CSV exports
temporary datasets
logs
cache
```

### Go

Possible sources:

```text
temporary files
profiling output
application-generated files
logs
```

### .NET

Possible sources:

```text
temporary files
dump files
logs
generated reports
diagnostic traces
```

The Kubernetes investigation remains largely the same.

---

# 37. Check Recent Deployment

Disk incidents can start immediately after deployment.

Check:

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

Also inspect:

```bash
kubectl get deployment order-service -n ecommerce -o yaml
```

Look for changes involving:

```text
volumeMounts
emptyDir
logging
ephemeral-storage
application arguments
environment variables
sidecars
diagnostic configuration
```

---

# 38. Correlate With GitOps

Check ArgoCD:

```bash
argocd app get order-service
```

Look for:

```text
sync
deployment version
health
recent changes
```

Also inspect Git:

```text
Git commit
   ↓
GitOps change
   ↓
ArgoCD sync
   ↓
Deployment
   ↓
Disk growth
```

This establishes whether the incident correlates with a recent change.

---

# 39. Check Prometheus

Useful metrics include filesystem and container storage metrics where available.

Conceptually investigate:

```text
node filesystem available
node filesystem usage
container filesystem usage
container ephemeral storage
Pod restarts
Pod evictions
DiskPressure
```

Example conceptual PromQL:

```promql
100 *
(
  1 -
  node_filesystem_avail_bytes{mountpoint="/"}
  /
  node_filesystem_size_bytes{mountpoint="/"}
)
```

Always adapt labels and metric availability to your installed exporters.

---

# 40. Check Grafana

Create a disk dashboard containing:

```text
Node filesystem usage
Node filesystem available
Inode usage
Ephemeral storage
Pod eviction count
DiskPressure
Container filesystem usage
EBS volume capacity
PVC usage
Prometheus storage
Elasticsearch storage
```

Use:

```text
Current value
Trend
Growth rate
Forecast
```

The trend is often more useful than a single 95% value.

---

# 41. Example Incident Timeline

```text
10:00
order-service version 2.1 deployed

10:15
temporary report generation increases

10:45
node disk reaches 80%

11:10
node disk reaches 92%

11:20
DiskPressure alert fires

11:22
Pods begin eviction

11:25
catalog-service Pod becomes Pending

11:27
new order-service Pod cannot pull image

11:30
Incident declared

11:35
Investigation identifies /tmp/report files

11:40
Runaway workload stopped

11:45
Approved temporary files removed

11:50
Node disk returns to healthy level

11:55
Pods recover

12:05
Customer transactions validated

12:30
RCA started
```

---

# 42. Immediate Mitigation

Mitigation depends on the root cause.

## Case 1 — Runaway temporary files

Stop or scale down the responsible workload if necessary.

Then remove only **known-safe application-generated files**.

Example:

```bash
sudo du -sh /approved/application/path
```

Do not blindly delete system directories.

---

## Case 2 — Large Logs

Use the approved logging/rotation process.

Do not simply run:

```bash
rm -rf /var/log/*
```

Production logs may be needed for:

* incident investigation
* security investigations
* compliance
* RCA

Preserve evidence when required.

---

## Case 3 — PVC Full

Possible actions:

```text
clean unnecessary application data
increase PVC capacity
expand underlying volume
archive old data
fix retention
```

---

## Case 4 — Container Images

Use supported container-runtime garbage collection and node lifecycle mechanisms.

Do not manually delete runtime metadata.

---

## Case 5 — Node Unhealthy

If the node is disposable and the workload is healthy elsewhere:

```text
cordon
   ↓
drain
   ↓
replace node
```

Example:

```bash
kubectl cordon <node-name>
```

Drain must be performed carefully with:

* PDBs
* workload capacity
* DaemonSets
* local storage
* critical workloads

A node replacement is **not** a substitute for finding why disk filled.

---

# 43. GitOps Mitigation

Suppose deployment `2.1.0` introduced runaway local file generation.

Do not only manually modify the live Pod.

Instead:

```text
Identify bad version
      ↓
Revert GitOps change
      ↓
Commit
      ↓
PR/review
      ↓
ArgoCD sync
      ↓
Known-good version
```

This preserves the desired-state model.

---

# 44. Important Rule: Do Not Treat Disk Cleanup as the Fix

Example:

```text
Disk = 100%
```

You delete 20 GB.

Now:

```text
Disk = 75%
```

But the application continues generating:

```text
500 MB/hour
```

The incident will return.

The real fix is:

```text
Stop uncontrolled growth
```

For example:

```text
local files
     ↓
S3
```

or:

```text
unbounded logs
     ↓
structured stdout
     ↓
centralized logging
     ↓
retention policy
```

---

# 45. Preventive Architecture

For our project:

```text
Application
    |
    +--> stdout/stderr
    |       |
    |       +--> Fluent Bit
    |               |
    |               +--> Logstash
    |                       |
    |                       +--> Elasticsearch
    |
    +--> large objects
    |       |
    |       +--> S3
    |
    +--> cache
    |       |
    |       +--> Redis
    |
    +--> database
    |       |
    |       +--> RDS
    |
    +--> temporary workspace
            |
            +--> bounded emptyDir
```

This reduces uncontrolled node-local storage.

---

# 46. Ephemeral Storage Guardrails

For workloads that use temporary storage:

```yaml
resources:
  requests:
    ephemeral-storage: "500Mi"

  limits:
    ephemeral-storage: "2Gi"
```

Then monitor actual usage.

Do not choose:

```text
500Mi
2Gi
```

as universal production values.

Measure the workload.

---

# 47. Application Design Rules

Applications should:

* Avoid writing large permanent files to local disk
* Use S3 for durable objects
* Clean temporary files
* Bound temporary workspace
* Rotate logs
* Avoid unlimited local caches
* Avoid unlimited diagnostic recordings
* Avoid repeated heap dumps
* Use streaming for large files where appropriate
* Define retention
* Monitor storage growth

---

# 48. Node Capacity Planning

Disk capacity is part of Kubernetes capacity planning.

Consider:

```text
OS
+
container images
+
container writable layers
+
logs
+
emptyDir
+
kubelet
+
runtime
+
observability agents
+
failure headroom
```

Do not size a node's disk based only on:

```text
Application data
```

---

# 49. Multi-AZ Consideration

Our EKS architecture uses multiple AZs.

Example:

```text
AZ-A
  Node 1
  Node 2

AZ-B
  Node 3
  Node 4

AZ-C
  Node 5
  Node 6
```

If one node reaches:

```text
DiskPressure=True
```

workloads should have enough capacity to move elsewhere.

This requires:

* sufficient node capacity
* topology spread
* PDBs
* correct requests
* autoscaling
* healthy nodes in other AZs

---

# 50. Disk Full + Autoscaling

A common mistake is thinking:

> "Karpenter/HPA will solve it."

Not necessarily.

HPA scales:

```text
Pods
```

Karpenter scales:

```text
Nodes
```

Neither automatically fixes:

```text
Application writes unlimited files
```

If every new Pod writes another 10 GB:

```text
more Pods
   ↓
more disk consumption
```

The incident becomes worse.

---

# 51. Disk Full + Observability

Observability must monitor itself.

Architecture:

```text
Applications
     |
     +--> Metrics
     +--> Logs
     +--> Traces
            |
            ↓
      Observability Platform
            |
            +--> Storage
```

Monitor:

```text
Prometheus disk
Elasticsearch disk
Logstash queue disk
Jaeger storage
Fluent Bit buffering
```

Otherwise:

```text
Application incident
      ↓
Logging platform fills
      ↓
Logs disappear
      ↓
Troubleshooting becomes harder
```

---

# 52. Disk Full + Database

If PostgreSQL is running on RDS:

```text
EKS node disk full
```

is not the same problem as:

```text
RDS storage full
```

For RDS:

```text
RDS
 |
 +--> allocated storage
 |
 +--> PostgreSQL filesystem
```

RDS storage needs its own monitoring and recovery procedure.

Do not attempt to fix an RDS storage incident by cleaning an EKS node.

---

# 53. Incident Decision Tree

```text
Disk Alert
    |
    v
Is customer impact occurring?
    |
    +-- No --> Investigate before escalation
    |
    +-- Yes
          |
          v
      Where is disk full?
          |
          +--> Node
          |
          +--> Pod/container
          |
          +--> emptyDir
          |
          +--> PVC
          |
          +--> Observability platform
          |
          v
      Space or inodes?
          |
          +--> Space
          |
          +--> Inodes
          |
          v
      What is consuming it?
          |
          +--> Logs
          +--> Temp files
          +--> Heap dump
          +--> Images
          +--> Runtime storage
          +--> Application data
          +--> Observability data
          |
          v
      Mitigate
          |
          v
      Restore capacity
          |
          v
      Fix root cause
          |
          v
      Validate
          |
          v
      RCA + prevention
```

---

# 54. Production Commands Cheat Sheet

### Kubernetes

```bash
kubectl get nodes
```

```bash
kubectl describe node <node>
```

```bash
kubectl get pods -A -o wide
```

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

```bash
kubectl describe pod <pod> -n ecommerce
```

```bash
kubectl get pvc -n ecommerce
```

```bash
kubectl describe pvc <pvc> -n ecommerce
```

---

### Node

```bash
df -h
```

```bash
df -i
```

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

```bash
sudo du -sh /var/log/* 2>/dev/null
```

```bash
journalctl --disk-usage
```

```bash
sudo find / -xdev -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
sudo lsof +L1
```

---

### Kubernetes Pod

```bash
kubectl exec -it <pod> -n ecommerce -- df -h
```

```bash
kubectl exec -it <pod> -n ecommerce -- df -i
```

```bash
kubectl exec -it <pod> -n ecommerce -- sh
```

Then:

```bash
du -sh /tmp/*
```

---

# 55. What NOT to Do During Production Incident

Never blindly execute:

```bash
rm -rf /var/lib/containerd/*
```

or:

```bash
rm -rf /var/lib/kubelet/*
```

Do not blindly execute:

```bash
rm -rf /var/log/*
```

Do not delete:

```text
application evidence
database files
container runtime metadata
Kubernetes internal files
```

without understanding the impact.

Do not immediately reboot every node.

Do not immediately increase disk without identifying the growth source.

Do not assume:

```text
DiskPressure = application bug
```

or:

```text
DiskPressure = Kubernetes bug
```

Investigate first.

---

# 56. Example Root Cause

Suppose investigation finds:

```text
Node disk: 98%

Largest directory:
/tmp/order-reports

Size:
45 GB
```

Then:

```text
Application
    ↓
Generated PDF reports
    ↓
Stored locally
    ↓
No cleanup
    ↓
Disk growth
    ↓
DiskPressure
    ↓
Pod eviction
```

### Root Cause

> `order-service` stored generated reports on node-local ephemeral storage without a bounded retention/cleanup strategy.

Not:

> "Kubernetes DiskPressure."

DiskPressure is the symptom.

---

# 57. Permanent Fix

Change:

```text
order-service
     |
     +--> local filesystem
```

to:

```text
order-service
     |
     +--> S3
```

For temporary processing:

```text
S3
 |
 +--> bounded temporary workspace
 |
 +--> process
 |
 +--> cleanup
```

Add:

```text
ephemeral-storage requests
ephemeral-storage limits
storage monitoring
S3 lifecycle
application cleanup
```

---

# 58. Alerting Strategy

Create alerts such as:

```text
NodeFilesystemUsageHigh
NodeInodeUsageHigh
NodeDiskPressure
PodEphemeralStorageHigh
PVCUsageHigh
PrometheusStorageHigh
ElasticsearchStorageHigh
```

Example conceptual alert:

```yaml
- alert: NodeFilesystemUsageHigh
  expr: |
    100 *
    (
      1 -
      node_filesystem_avail_bytes{mountpoint="/"}
      /
      node_filesystem_size_bytes{mountpoint="/"}
    ) > 85
  for: 10m
  labels:
    severity: warning
    team: platform
  annotations:
    summary: "Node filesystem usage is high"
```

A higher threshold can trigger critical action.

Exact thresholds should be based on workload growth rate and available recovery time.

---

# 59. Runbook

Create:

```text
runbooks/disk-full.md
```

Recommended structure:

```markdown
# Disk Full Runbook

## Symptoms

## Customer Impact

## Initial Checks

## Check DiskPressure

## Check Filesystem Usage

## Check Inodes

## Identify Large Directories

## Check Logs

## Check Temporary Files

## Check Container Runtime

## Check PVC

## Check Observability Storage

## Immediate Mitigation

## Validation

## Root Cause

## Rollback

## Recovery

## Prevention

## RCA
```

---

# 60. RCA Template

```markdown
# Disk Full Incident RCA

## Incident

Disk Full / DiskPressure

## Date

YYYY-MM-DD

## Severity

SEV-2

## Impact

Describe affected services and customers.

## Detection

How was the incident detected?

## Timeline

- 10:00 deployment
- 10:15 disk growth
- 10:30 alert
- 10:35 mitigation
- 10:50 recovery

## Root Cause

Describe the actual technical cause.

## Contributing Factors

- Missing ephemeral-storage limits
- Missing cleanup
- Insufficient alerting
- No storage growth dashboard

## Mitigation

Describe temporary mitigation.

## Permanent Fix

Describe engineering fix.

## Customer Impact

Describe affected requests/orders.

## RTO

Measured recovery time.

## Data Loss

Yes/No

## Corrective Actions

- [ ] Add disk alerts
- [ ] Add inode alerts
- [ ] Add ephemeral-storage limits
- [ ] Move durable files to S3
- [ ] Configure retention
- [ ] Add integration test
- [ ] Update runbook
```

---

# 61. Production Prevention Checklist

### Kubernetes

* [ ] Monitor node filesystem
* [ ] Monitor inode usage
* [ ] Monitor `DiskPressure`
* [ ] Define ephemeral-storage requests
* [ ] Define ephemeral-storage limits
* [ ] Monitor Pod evictions
* [ ] Maintain node disk headroom
* [ ] Test node replacement

### Application

* [ ] Avoid unlimited local files
* [ ] Clean temporary files
* [ ] Bound local cache
* [ ] Configure log rotation
* [ ] Control heap dumps
* [ ] Control JFR recordings
* [ ] Stream large files where appropriate
* [ ] Store durable objects in S3

### PVC

* [ ] Monitor PVC utilization
* [ ] Define storage capacity correctly
* [ ] Test volume expansion
* [ ] Configure retention
* [ ] Monitor EBS capacity

### Observability

* [ ] Prometheus storage monitored
* [ ] Elasticsearch storage monitored
* [ ] Logstash queues monitored
* [ ] Jaeger storage monitored
* [ ] Fluent Bit buffers monitored
* [ ] Retention policies configured

### Incident Management

* [ ] Disk runbook exists
* [ ] On-call knows commands
* [ ] Alerts route correctly
* [ ] RCA template exists
* [ ] Recovery tested

---

# 62. Interview/Production Mental Model

When someone says:

> "Production Pods are failing because the disk is full."

Don't immediately answer:

> "Increase the disk."

Think:

```text
WHERE?
 |
 +-- Node?
 +-- Container?
 +-- emptyDir?
 +-- PVC?
 +-- Observability?
 |
WHAT?
 |
 +-- Space?
 +-- Inodes?
 |
WHY?
 |
 +-- Logs?
 +-- Temp?
 +-- Heap dump?
 +-- Images?
 +-- Runtime?
 +-- Application files?
 +-- Retention?
 |
HOW TO MITIGATE?
 |
 +-- Clean safe data
 +-- Stop runaway workload
 +-- Replace node
 +-- Expand volume
 +-- Restore service
 |
HOW TO PREVENT?
 |
 +-- Limits
 +-- Monitoring
 +-- Retention
 +-- S3
 +-- Capacity planning
 +-- Application fix
```

---

# 63. Final Production Architecture

```text
                    INTERNET
                       |
                       v
                CloudFront / WAF
                       |
                       v
                     ALB
                       |
                       v
                    EKS
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
   Application      Platform       Observability
      Pods           Pods              |
        |              |               |
        |              |        +------+------+
        |              |        |      |      |
        |              |        v      v      v
        |              |    Prometheus ELK  Jaeger
        |              |
        |              +--> bounded ephemeral storage
        |
        +--> RDS
        +--> Redis
        +--> SQS
        +--> Kafka
        +--> S3
```

Storage principle:

```text
Ephemeral data
      ↓
bounded local storage

Durable application data
      ↓
RDS / Redis / S3 / messaging

Logs
      ↓
centralized logging

Metrics
      ↓
Prometheus

Traces
      ↓
OpenTelemetry → Jaeger
```

---

# 64. Final Incident Flow

```text
Disk Full Alert
      ↓
Customer Impact?
      ↓
Identify affected node/PVC/platform
      ↓
Check DiskPressure
      ↓
df -h
      ↓
df -i
      ↓
du
      ↓
Find largest consumer
      ↓
Logs / tmp / heap dump / images / PVC /
observability / application files
      ↓
Mitigate safely
      ↓
Restore capacity
      ↓
Recover Pods
      ↓
Validate application
      ↓
Validate business transaction
      ↓
Find root cause
      ↓
Permanent fix
      ↓
Alert + dashboard + guardrail
      ↓
RCA
      ↓
Game-day test
```

---

# 65. Key Rules to Remember

1. **Disk Full is a symptom, not the root cause.**
2. Always check both **disk space and inodes**.
3. Distinguish **node disk, ephemeral storage, `emptyDir`, and PVC**.
4. `DiskPressure=True` can cause Pod eviction.
5. Do not manually delete container runtime internals.
6. Do not blindly delete production logs.
7. Java heap dumps and JFR files can consume large amounts of disk.
8. Durable files should generally go to **S3**, not node-local storage.
9. HPA/Karpenter cannot fix an application that continuously generates files.
10. Observability platforms need their own storage monitoring.
11. Increasing disk is mitigation, not necessarily the permanent fix.
12. In a GitOps environment, fix the desired state in Git.
13. Measure storage **growth rate**, not only current utilization.
14. Validate customer transactions after recovery.
15. Every production disk incident should result in a prevention action.

---

## Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ✅
147 — Disk Full                    ✅
148 — ImagePullBackOff             ⏭️
149 — ALB 5xx                      ⏭️
150 — Service Connectivity Failure ⏭️
151 — DNS Failure                  ⏭️
152 — RDS Connection Exhaustion    ⏭️
153 — Redis Failure                ⏭️
154 — SQS Backlog                  ⏭️
155 — Kafka Consumer Lag            ⏭️
156 — Node Failure                 ⏭️
157 — AZ Failure                   ⏭️
158 — ArgoCD Drift                 ⏭️
159 — Bad Production Deployment    ⏭️
160 — Security Incident            ⏭️
```

**Next: `148 — ImagePullBackOff Production Incident`**.
