# 35 — Fluent Bit

> Production-grade Kubernetes log collection with Fluent Bit for the Enterprise E-Commerce Platform on AWS EKS

---

## 1. Purpose

We have now built the metrics side of observability:

```text
Application
     │
     ▼
Micrometer
     │
     ▼
Prometheus
     │
     ├──► Grafana
     └──► Alertmanager
```

Now we build the **logging pipeline**.

Our target architecture is:

```text
21 Microservices
      │
      │ stdout / stderr
      ▼
Kubernetes Container Logs
      │
      ▼
Fluent Bit
      │
      ▼
Logstash
      │
      ▼
Elasticsearch
      │
      ▼
Kibana
```

Fluent Bit is the **log collector/forwarder** at the edge of the Kubernetes logging pipeline.

---

# 2. What Fluent Bit Does

Fluent Bit runs on EKS nodes and:

1. reads container logs
2. parses the log records
3. enriches them with Kubernetes metadata
4. optionally transforms/filter logs
5. buffers records
6. forwards them to Logstash

Think:

```text
Fluent Bit
=
Collect
+
Parse
+
Enrich
+
Buffer
+
Forward
```

It is **not** our primary log-storage system.

---

# 3. Fluent Bit vs Logstash

This distinction is critical.

### Fluent Bit

Runs close to the application:

```text
EKS node
   ↓
Fluent Bit
```

Responsibilities:

```text
Collect
Parse
Enrich
Buffer
Forward
```

### Logstash

Runs centrally:

```text
EKS
 ↓
Logstash
```

Responsibilities:

```text
Receive
Parse
Transform
Enrich
Route
Forward
```

### Elasticsearch

Stores/searches logs.

### Kibana

Visualizes/searches logs.

Therefore:

```text
Fluent Bit
     ↓
"Get logs off the node"

Logstash
     ↓
"Process and route logs"

Elasticsearch
     ↓
"Store/search logs"

Kibana
     ↓
"Investigate logs"
```

---

# 4. Why Fluent Bit Runs as a DaemonSet

Every EKS worker node generates container logs.

Therefore we want:

```text
Node 1 → Fluent Bit
Node 2 → Fluent Bit
Node 3 → Fluent Bit
Node 4 → Fluent Bit
```

Kubernetes primitive:

```text
DaemonSet
```

Architecture:

```text
EKS
│
├── node-1
│    ├── order-service
│    ├── payment-service
│    └── Fluent Bit
│
├── node-2
│    ├── catalog-service
│    ├── inventory-service
│    └── Fluent Bit
│
└── node-3
     ├── cart-service
     └── Fluent Bit
```

---

# 5. Why Not One Fluent Bit Pod?

Suppose we deployed only:

```text
Fluent Bit
    ↓
one Pod
```

Then logs from:

```text
node-1
node-2
node-3
node-4
```

would need to reach that central collector.

This creates:

* additional networking
* a centralized collection bottleneck
* larger blast radius
* potentially more complicated access to node log files

A DaemonSet gives us local collection:

```text
Node
 ↓
local Fluent Bit
```

---

# 6. Log Flow

The complete path:

```text
Spring Boot
     │
     │ stdout
     ▼
container runtime
     │
     ▼
/var/log/containers/*.log
     │
     ▼
Fluent Bit
     │
     ▼
Logstash
     │
     ▼
Elasticsearch
     │
     ▼
Kibana
```

This is one of the most important flows to understand for production troubleshooting.

---

# 7. Why Applications Should Log to stdout/stderr

Our containers should generally write application logs to:

```text
stdout
stderr
```

instead of creating application-specific log files inside the container.

Example:

```text
Spring Boot
   ↓
ConsoleAppender
   ↓
stdout
```

Kubernetes/container runtime then captures the output.

This allows Fluent Bit to collect logs consistently.

---

# 8. What Happens to stdout?

Suppose `order-service` prints:

```text
2026-09-10 10:30:12 INFO Order created id=12345
```

The container runtime captures it.

Kubernetes exposes container log files through the node's logging paths.

Common paths include:

```text
/var/log/containers/
/var/log/pods/
```

The exact runtime/path details should be verified against the EKS/AMI/container-runtime version being used.

---

# 9. `/var/log/containers`

You will commonly see files similar to:

```text
/var/log/containers/
```

with filenames containing:

```text
pod
namespace
container
```

Example conceptually:

```text
order-service-7c9f8d_production_order-service-abc123.log
```

Fluent Bit tails these log files.

---

# 10. HostPath Mounts

Because Fluent Bit runs inside a Pod but needs to read node log files, the DaemonSet commonly mounts host directories.

Conceptually:

```yaml
volumes:
  - name: varlog
    hostPath:
      path: /var/log
```

Then:

```text
Host
/var/log
   │
   │ mounted
   ▼
Fluent Bit Pod
```

This is why Fluent Bit requires elevated host-level access compared with a normal application.

---

# 11. Fluent Bit Security

This is important.

Fluent Bit is a privileged platform component because it reads host log data.

Therefore:

* minimize hostPath access
* run with the minimum privileges required
* restrict network access
* use an approved image
* pin versions
* avoid unnecessary host mounts
* review security context
* protect destination credentials

Do not treat Fluent Bit as an ordinary application container.

---

# 12. Installation

Production approach:

```text
Git
 ↓
Helm values
 ↓
ArgoCD
 ↓
EKS
 ↓
Fluent Bit DaemonSet
```

For learning:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Then install using an approved chart/version.

Production should pin:

```text
Fluent Bit version
+
Helm chart version
```

---

# 13. Namespace

Use a dedicated logging namespace:

```text
logging
```

Example:

```bash
kubectl create namespace logging
```

Our platform becomes:

```text
argocd
monitoring
logging
tracing
production
```

---

# 14. GitOps Structure

Example:

```text
gitops/
└── environments/
    └── prod/
        └── logging/
            ├── fluent-bit/
            │   ├── values.yaml
            │   └── config/
            │
            └── logstash/
                ├── values.yaml
                └── pipelines/
```

ArgoCD manages the deployment.

---

# 15. DaemonSet Verification

After installation:

```bash
kubectl get daemonset -n logging
```

Example:

```text
NAME          DESIRED   CURRENT   READY
fluent-bit    6         6         6
```

Then:

```bash
kubectl get pods -n logging -o wide
```

We should be able to map:

```text
node-1 → Fluent Bit
node-2 → Fluent Bit
node-3 → Fluent Bit
```

---

# 16. Verify Every Node

Compare:

```bash
kubectl get nodes
```

with:

```bash
kubectl get pods -n logging -o wide
```

If:

```text
6 nodes
5 Fluent Bit Pods
```

there is a logging coverage gap.

Investigate:

```text
node selector
taints
tolerations
resource constraints
DaemonSet configuration
```

---

# 17. Fluent Bit Architecture

Internally, think of Fluent Bit as:

```text
INPUT
  ↓
PARSER
  ↓
FILTER
  ↓
BUFFER
  ↓
OUTPUT
```

For us:

```text
INPUT
  ↓
tail container logs

FILTER
  ↓
Kubernetes metadata

OUTPUT
  ↓
Logstash
```

---

# 18. Input

The `tail` input reads log files.

Conceptually:

```ini
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    Tag kube.*
```

The exact configuration depends on the Fluent Bit version/chart.

---

# 19. Tail Input

The tail input continuously watches files.

Example:

```text
order-service.log
payment-service.log
catalog-service.log
```

As new lines appear:

```text
Application writes
       ↓
log file grows
       ↓
Fluent Bit reads new data
```

It doesn't need to repeatedly reread the entire file.

---

# 20. Position Database

Fluent Bit needs to remember where it stopped reading.

Conceptually:

```text
log file
   │
   ├── line 1
   ├── line 2
   ├── line 3
   └── line 4
             ↑
          offset
```

If Fluent Bit restarts:

```text
Fluent Bit
   ↓
reads stored position
   ↓
continues from known location
```

This reduces duplicate collection and helps recovery.

---

# 21. Buffering

What if Logstash is unavailable?

```text
Fluent Bit
    │
    ▼
Logstash
    X
```

Without appropriate buffering, logs can be lost.

Therefore Fluent Bit supports buffering strategies.

Conceptually:

```text
Application
    ↓
Fluent Bit
    ↓
Buffer
    ↓
Logstash
```

If Logstash temporarily fails:

```text
Fluent Bit
    ↓
retain records
    ↓
retry
```

---

# 22. Memory vs Filesystem Buffering

There are two broad strategies:

```text
Memory buffering
Filesystem buffering
```

### Memory

Fast:

```text
RAM
```

But memory is lost if the process/node disappears.

### Filesystem

More resilient:

```text
Node storage
```

but requires:

* disk space
* filesystem monitoring
* cleanup
* capacity planning

For production, choose based on the expected failure model and log volume.

---

# 23. Backpressure

Suppose:

```text
Applications
    ↓
10,000 logs/sec
```

but:

```text
Logstash
    ↓
5,000 logs/sec
```

Then:

```text
incoming rate > outgoing rate
```

The collector starts accumulating data.

This is:

```text
backpressure
```

Eventually:

```text
buffer grows
    ↓
disk/memory pressure
    ↓
logs delayed
    ↓
possible data loss
```

Therefore logging infrastructure must be capacity-planned.

---

# 24. Fluent Bit Backpressure

Monitor:

```text
buffer size
memory usage
filesystem usage
retry count
output failures
records processed
records dropped
```

A healthy pipeline isn't merely:

```text
Fluent Bit Pod = Running
```

We need to know:

```text
Is it successfully forwarding logs?
```

---

# 25. Kubernetes Metadata

Raw log:

```text
Order created
```

is not very useful by itself.

We want metadata:

```text
namespace=production
pod=order-service-abc123
container=order-service
node=ip-10-0-11-25
labels=...
```

Fluent Bit's Kubernetes filtering/enrichment capabilities can add this context.

---

# 26. Why Kubernetes Metadata Matters

Suppose Kibana shows:

```text
NullPointerException
```

Without metadata:

```text
Which service?
Which Pod?
Which namespace?
Which node?
```

Unknown.

With metadata:

```text
service=order-service
namespace=production
pod=order-service-abc123
node=ip-10-0-11-25
```

Investigation becomes much faster.

---

# 27. Kubernetes Filter

Conceptually:

```ini
[FILTER]
    Name kubernetes
    Match kube.*
```

The actual configuration should be adapted to the chart/version and container-runtime format.

The filter can enrich records using Kubernetes API metadata.

---

# 28. API Server Access

If Fluent Bit needs Kubernetes metadata, it may communicate with the Kubernetes API.

Architecture:

```text
Fluent Bit
    │
    ▼
Kubernetes API
    │
    ▼
Pod metadata
```

This means RBAC must be configured correctly.

Again:

```text
Read
```

should generally be sufficient.

No need for Fluent Bit to modify Pods.

---

# 29. Fluent Bit ServiceAccount

Conceptually:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
```

Then appropriate RBAC:

```text
ServiceAccount
      ↓
Role/ClusterRole
      ↓
read metadata
```

Use the chart's supported RBAC configuration rather than blindly applying a custom cluster-admin role.

---

# 30. JSON Logs

Our applications should preferably emit structured logs.

Instead of:

```text
Order created successfully
```

prefer something like:

```json
{
  "timestamp": "2026-09-10T10:30:12Z",
  "level": "INFO",
  "service": "order-service",
  "event": "order_created",
  "order_id": "12345"
}
```

This makes downstream searching and parsing much easier.

---

# 31. Why Structured Logging Matters

With structured logs:

```text
event=order_created
service=order-service
```

Kibana can filter precisely.

Instead of searching:

```text
"order created"
```

we can query:

```text
service:"order-service"
AND event:"order_created"
```

This is far more reliable.

---

# 32. Logging Levels

Typical levels:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

Production generally shouldn't run everything at:

```text
DEBUG
```

because log volume can become enormous.

A reasonable baseline is often:

```text
INFO
```

with targeted debug logging during controlled troubleshooting.

---

# 33. Never Log Secrets

Never log:

```text
password
JWT
access token
refresh token
AWS secret key
database credentials
API keys
credit-card data
sensitive personal data
```

Bad:

```text
password=SuperSecret
```

Even if Elasticsearch is secure:

```text
secret
 ↓
log
 ↓
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
```

The secret has already escaped into the logging pipeline.

---

# 34. Log Redaction

Sensitive fields should be removed/redacted before storage when appropriate.

Example:

```text
Authorization: Bearer abc123
```

should become:

```text
Authorization: [REDACTED]
```

However, **prefer preventing sensitive data from entering application logs in the first place**.

Don't rely solely on Fluent Bit filters for security.

---

# 35. Multiline Logs

Java applications create stack traces.

Example:

```text
java.lang.NullPointerException
    at com.company.order.OrderService.create(OrderService.java:42)
    at ...
    at ...
```

This is multiple physical lines but represents:

```text
one logical error
```

Without multiline handling:

```text
ERROR
line 1
line 2
line 3
```

may become separate log records.

---

# 36. Why Multiline Matters

In Kibana, we want:

```text
ONE exception
```

instead of:

```text
record 1
record 2
record 3
record 4
```

Fluent Bit supports multiline parsing.

Use the multiline configuration appropriate to the Fluent Bit version and actual application log format.

---

# 37. Spring Boot Logging

Our Spring Boot applications can use structured JSON logging.

Conceptually:

```text
Spring Boot
 ↓
JSON console logging
 ↓
stdout
 ↓
Fluent Bit
```

Then Fluent Bit can forward the JSON record without needing to reconstruct application semantics.

---

# 38. Trace Correlation

This becomes extremely powerful later.

Application log:

```json
{
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "Payment failed"
}
```

Then:

```text
Grafana
   ↓
Trace ID
   ↓
Jaeger
```

and:

```text
Kibana
   ↓
trace_id=abc123
   ↓
related logs
```

This connects:

```text
Metrics
+
Logs
+
Traces
```

---

# 39. Logstash Output

Our Fluent Bit output:

```text
Fluent Bit
     │
     ▼
Logstash
```

Conceptually:

```ini
[OUTPUT]
    Name http
    Match kube.*
    Host logstash.logging.svc.cluster.local
    Port 8080
```

The exact output plugin and transport should be chosen based on the Logstash input configured.

For example, if Logstash uses a Beats-compatible input, Fluent Bit can use the appropriate compatible output/plugin instead.

**Do not copy an HTTP/Beats example blindly into production; verify the selected Fluent Bit output and Logstash input versions/configuration match.**

---

# 40. Why Use Logstash?

You might ask:

> Why not send Fluent Bit directly to Elasticsearch?

You can.

Architecture:

```text
Fluent Bit
    ↓
Elasticsearch
```

is perfectly valid.

But our project intentionally uses:

```text
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
```

because it teaches an important enterprise pattern:

```text
Edge collection
        +
Central processing
        +
Central storage
```

---

# 41. Logstash Processing

Logstash can perform:

```text
parse
transform
rename fields
drop unwanted records
route logs
enrich logs
normalize formats
```

For example:

```text
service=order-service
environment=prod
```

could be normalized into fields used consistently across the platform.

---

# 42. Logstash as a Bottleneck

Remember:

```text
Fluent Bit × number of nodes
           ↓
       Logstash
           ↓
     Elasticsearch
```

If Logstash cannot keep up:

```text
Fluent Bit buffers
       ↓
buffers grow
       ↓
node disk/memory pressure
```

Therefore Logstash needs:

* sufficient replicas
* resource requests
* resource limits
* persistent buffering where appropriate
* scaling strategy
* monitoring

---

# 43. Logstash High Availability

Instead of:

```text
Fluent Bit
   ↓
Logstash-1
```

production:

```text
             Fluent Bit
                  │
          ┌───────┴───────┐
          ▼               ▼
      Logstash-1       Logstash-2
          │               │
          └───────┬───────┘
                  ▼
            Elasticsearch
```

If one Logstash Pod fails:

```text
Logstash-1 → unavailable
Logstash-2 → continues
```

assuming the surrounding service/load-balancing and buffering architecture is correctly configured.

---

# 44. Logstash Service

Kubernetes:

```text
Service
   ↓
Logstash Pods
```

Example:

```text
logstash.logging.svc.cluster.local
```

Fluent Bit should communicate through the Kubernetes Service rather than directly targeting a Pod IP.

---

# 45. DNS Flow

Fluent Bit:

```text
logstash.logging.svc.cluster.local
```

Kubernetes DNS:

```text
Service
   ↓
Pod endpoints
```

Therefore Logstash Pods can change without requiring Fluent Bit configuration changes.

---

# 46. TLS Between Fluent Bit and Logstash

Production traffic should be protected where required.

```text
Fluent Bit
    │
    │ TLS
    ▼
Logstash
```

Depending on architecture:

```text
certificate
CA
client authentication
authentication credentials
```

may be required.

Never transmit sensitive production logs over an untrusted network without appropriate protection.

---

# 47. NetworkPolicy

Logging traffic should be explicitly controlled.

Conceptually:

```text
logging namespace
│
├── Fluent Bit
│       │
│       └──► Logstash
│
└── Logstash
        │
        └──► Elasticsearch
```

Application namespaces:

```text
production
   X
   │
   └──► arbitrary logging components
```

unless explicitly required.

---

# 48. Log Volume

Our platform has:

```text
21 microservices
```

Suppose each service generates:

```text
100 logs/sec
```

Total:

```text
21 × 100
=
2,100 logs/sec
```

Now consider average size:

```text
1 KB/log
```

Approximate raw volume:

```text
2,100 KB/sec
≈ 2.1 MB/sec
```

Daily:

```text
~181 GB/day
```

before compression/index overhead.

This demonstrates why logging needs capacity planning.

---

# 49. High-Traffic Scenario

Suppose traffic increases 10×.

Logs can increase:

```text
100 logs/sec
   ↓
1,000 logs/sec
```

across a service.

Then:

```text
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
```

all need to handle the increased volume.

Logging can become one of the largest infrastructure costs if uncontrolled.

---

# 50. Logging Strategy

Don't log every successful internal operation repeatedly.

Bad:

```text
Request received
Request processed
Database called
Database returned
Cache checked
Cache hit
Controller finished
```

for every request.

At high traffic this becomes huge.

Better:

```text
INFO:
Important lifecycle/business events

WARN:
Potential problem

ERROR:
Failure requiring investigation

DEBUG:
Temporary diagnostic detail
```

---

# 51. Log Sampling

For extremely high-volume services, sampling can be useful.

For example:

```text
100,000 successful requests
```

might not require storing detailed logs for every request.

But:

```text
all ERROR logs
```

should generally be retained.

Sampling must be designed carefully so important forensic information is not lost.

---

# 52. Log Retention

We should define retention.

Example:

```text
Hot:
7 days

Warm:
30 days

Archive:
90+ days
```

These are examples only.

Actual retention depends on:

* troubleshooting requirements
* compliance
* business needs
* storage cost
* security policy

---

# 53. Elasticsearch Storage

The logging pipeline eventually becomes:

```text
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
 ↓
Storage
```

If Elasticsearch storage fills:

```text
Elasticsearch
    ↓
disk pressure
    ↓
indexing problems
    ↓
Logstash backpressure
    ↓
Fluent Bit buffers grow
    ↓
Node pressure
```

Therefore the entire pipeline must be monitored.

---

# 54. Fluent Bit Monitoring

Monitor:

```text
records received
records processed
records sent
retry count
dropped records
buffer size
memory
CPU
output errors
```

Also monitor:

```text
input rate
output rate
```

A critical relationship:

```text
input rate
    vs
output rate
```

If:

```text
input > output
```

for a sustained period:

```text
backlog grows
```

---

# 55. Fluent Bit Logs

Fluent Bit itself logs errors.

Check:

```bash
kubectl logs -n logging <fluent-bit-pod>
```

Look for:

```text
connection refused
timeout
retry
buffer full
parse failure
permission denied
DNS failure
TLS error
```

---

# 56. Troubleshooting — No Logs in Kibana

Never jump directly to Elasticsearch.

Follow the pipeline:

```text
Application
   ↓
Container log
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
   ↓
Kibana
```

---

# 57. Step 1 — Application

Check:

```bash
kubectl logs -n production <pod>
```

If there are no logs:

```text
Application logging problem
```

Don't troubleshoot Fluent Bit yet.

---

# 58. Step 2 — Node Log File

If the application logs exist through:

```bash
kubectl logs
```

but Fluent Bit doesn't see them:

```text
Node logging path
Tail configuration
Permissions
Container runtime format
```

need investigation.

---

# 59. Step 3 — Fluent Bit

Check:

```bash
kubectl logs -n logging <fluent-bit-pod>
```

Look for:

```text
input errors
parser errors
filter errors
output errors
retry loops
buffer problems
```

---

# 60. Step 4 — Logstash

Check:

```bash
kubectl get pods -n logging
```

Then:

```bash
kubectl logs -n logging <logstash-pod>
```

Possible:

```text
connection errors
pipeline errors
parsing errors
Elasticsearch output errors
```

---

# 61. Step 5 — Elasticsearch

Check:

```text
index creation
cluster health
disk usage
ingest errors
authentication
```

If Logstash is receiving logs but Elasticsearch rejects them:

```text
Fluent Bit
   ↓
healthy

Logstash
   ↓
healthy

Elasticsearch
   ↓
problem
```

---

# 62. Step 6 — Kibana

If Elasticsearch has the logs but Kibana doesn't show them:

```text
Elasticsearch
   ↓
index/data view
   ↓
Kibana
```

Check:

```text
time range
index pattern/data view
field mapping
query
permissions
```

A common mistake is simply using the wrong time range.

---

# 63. Troubleshooting — Fluent Bit CrashLoopBackOff

Check:

```bash
kubectl describe pod -n logging <fluent-bit-pod>
```

and:

```bash
kubectl logs -n logging <fluent-bit-pod>
```

Potential causes:

```text
invalid configuration
bad parser
missing volume
permission issue
resource exhaustion
incorrect plugin
version/config incompatibility
```

---

# 64. Troubleshooting — Fluent Bit Memory Growth

Potential causes:

```text
Logstash unavailable
Elasticsearch unavailable
Output too slow
Huge multiline records
Excessive log volume
Buffers too large
Retry backlog
```

Do not simply increase memory.

First determine:

```text
Why is Fluent Bit unable to drain its queue?
```

---

# 65. Troubleshooting — Logs Delayed

Suppose:

```text
Application:
10:00:00

Kibana:
10:05:00
```

Five-minute delay.

Investigate:

```text
Fluent Bit buffers
Logstash processing
Elasticsearch indexing
network latency
resource saturation
```

Check:

```text
input rate
output rate
queue depth
```

---

# 66. Troubleshooting — Logs Duplicated

Possible causes:

```text
offset database issue
file rotation behavior
collector restart
retry behavior
downstream acknowledgment behavior
configuration
```

Don't assume duplicates always mean Fluent Bit is broken.

Trace one record through:

```text
application
→ Fluent Bit
→ Logstash
→ Elasticsearch
```

and identify where duplication occurs.

---

# 67. Log Rotation

Container logs rotate.

Fluent Bit must handle:

```text
file created
file appended
file rotated
old file removed
new file created
```

This is another reason to use the maintained Kubernetes/Fluent Bit configuration rather than writing an ad-hoc tail implementation.

---

# 68. Node Replacement

EKS node lifecycle:

```text
Old node
   ↓
drain
   ↓
terminate
   ↓
new EC2 node
   ↓
DaemonSets scheduled
   ↓
Fluent Bit starts
```

Because Fluent Bit is a DaemonSet, new nodes automatically receive a collector.

This is one of the strongest reasons to use the Kubernetes-native architecture.

---

# 69. Karpenter Scenario

Suppose Karpenter adds:

```text
node-7
```

Automatically:

```text
DaemonSet
   ↓
Fluent Bit
   ↓
Node-7
```

No engineer needs to manually install a logging agent.

Same for:

```text
Node Exporter
```

---

# 70. EKS Upgrade

During an EKS/node upgrade:

```text
node-1
   ↓
cordon
   ↓
drain
   ↓
terminate
```

Fluent Bit is also terminated.

But:

```text
node-2
node-3
node-4
```

continue collecting.

Then the replacement node receives a new Fluent Bit Pod.

This reduces the logging blind spot during node replacement.

---

# 71. Logging Failure Domain

Think carefully about failure domains.

```text
Node failure
    ↓
Fluent Bit on that node disappears
```

Logs that existed only locally may be lost depending on buffering and shutdown behavior.

Therefore:

```text
short local buffer
+
fast centralized forwarding
```

is generally better than treating the node filesystem as durable log storage.

---

# 72. Graceful Shutdown

Fluent Bit should be given enough time to flush pending records during termination where supported/configured.

During:

```text
kubectl drain
```

or:

```text
Pod termination
```

we want:

```text
flush pending logs
      ↓
exit
```

rather than:

```text
SIGTERM
 ↓
immediate termination
 ↓
buffer lost
```

The exact termination/graceful-shutdown configuration should be tested with the selected Fluent Bit version.

---

# 73. Resource Requests

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi

  limits:
    cpu: 500m
    memory: 512Mi
```

These are illustrative.

Real values depend on:

```text
logs/sec
record size
parsing complexity
multiline processing
output destination
buffering
```

---

# 74. Logging Node Capacity

Remember:

```text
Fluent Bit runs on every node
```

Therefore its resource consumption scales approximately with:

```text
number of nodes
```

If:

```text
100 nodes
```

and Fluent Bit uses:

```text
100m CPU
```

roughly:

```text
10 CPU cores
```

across the cluster.

Logging agents must therefore be included in node capacity planning.

---

# 75. Fluent Bit and Security

Fluent Bit sees application logs.

That can include sensitive information if developers make mistakes.

Therefore:

```text
Fluent Bit
   ↓
sensitive data exposure
```

must be treated as a security concern.

Controls:

* RBAC
* NetworkPolicy
* TLS
* secret management
* log redaction
* least privilege
* access-controlled Elasticsearch/Kibana
* retention policies

---

# 76. Production Architecture

Our final logging architecture:

```text
                         EKS
                          │
       ┌──────────────────┼───────────────────┐
       │                  │                   │
       ▼                  ▼                   ▼
    Node-1             Node-2              Node-3
       │                  │                   │
       ▼                  ▼                   ▼
 Fluent Bit           Fluent Bit          Fluent Bit
       │                  │                   │
       └──────────────────┼───────────────────┘
                          ▼
                     Logstash
                   ┌──────┴──────┐
                   ▼             ▼
              Logstash-1    Logstash-2
                   │             │
                   └──────┬──────┘
                          ▼
                    Elasticsearch
                          │
                          ▼
                        Kibana
```

---

# 77. Complete Observability Architecture

We now have:

```text
                       EKS
                        │
        ┌───────────────┼─────────────────┐
        │               │                 │
        ▼               ▼                 ▼
 Applications         Nodes          Kubernetes
        │               │                 │
        ▼               ▼                 ▼
   Micrometer      Node Exporter    kube-state-metrics
        │               │                 │
        └───────────────┼─────────────────┘
                        ▼
                   Prometheus
                        │
                 ┌──────┴──────┐
                 ▼             ▼
              Grafana      Alertmanager
                                  │
                                  ▼
                            Notifications


 Applications
       │
       ▼
    stdout
       │
       ▼
  Fluent Bit
       │
       ▼
   Logstash
       │
       ▼
 Elasticsearch
       │
       ▼
    Kibana
```

This gives us two major observability paths:

```text
Metrics:
Application / Node / Kubernetes
          ↓
      Prometheus
          ↓
   Grafana / Alertmanager


Logs:
Applications
     ↓
Fluent Bit
     ↓
Logstash
     ↓
Elasticsearch
     ↓
Kibana
```

---

# 78. Production Checklist

### Fluent Bit

* [ ] DaemonSet
* [ ] Runs on all required nodes
* [ ] Version pinned
* [ ] Helm/GitOps managed
* [ ] Resource requests configured
* [ ] Host mounts reviewed
* [ ] RBAC reviewed
* [ ] Kubernetes metadata enrichment
* [ ] Position database configured
* [ ] Buffering configured
* [ ] Retry behavior tested

### Logging

* [ ] Applications log to stdout/stderr
* [ ] Structured logging preferred
* [ ] JSON format standardized
* [ ] Multiline Java exceptions handled
* [ ] Sensitive information excluded
* [ ] Log levels controlled
* [ ] Trace correlation fields included where possible

### Logstash

* [ ] Multiple replicas where required
* [ ] Resource sizing
* [ ] Pipeline configuration
* [ ] TLS
* [ ] Backpressure handling
* [ ] Elasticsearch connectivity
* [ ] Monitoring

### Elasticsearch

* [ ] Storage capacity planned
* [ ] Retention configured
* [ ] Index lifecycle strategy
* [ ] Cluster health monitored
* [ ] Disk alerts configured

### Security

* [ ] NetworkPolicy
* [ ] RBAC
* [ ] TLS
* [ ] Secrets managed securely
* [ ] Access to logs controlled
* [ ] Retention aligned with policy

---

# 79. Most Important Troubleshooting Mental Model

When someone says:

> **“Logs are missing.”**

Do **not** immediately restart Fluent Bit.

Follow the pipeline:

```text
                 LOG MISSING
                      │
                      ▼
             Is application logging?
                      │
                      ▼
              Does kubectl logs work?
                      │
                      ▼
            Does Fluent Bit see it?
                      │
                      ▼
          Is Fluent Bit forwarding?
                      │
                      ▼
          Is Logstash receiving?
                      │
                      ▼
      Is Logstash forwarding successfully?
                      │
                      ▼
       Is Elasticsearch indexing it?
                      │
                      ▼
           Is Kibana querying it?
```

This is exactly how production troubleshooting should work.

---

# 80. Final Mental Model

Remember:

```text
Fluent Bit
    =
"COLLECT FROM EVERY NODE"

Logstash
    =
"PROCESS CENTRALLY"

Elasticsearch
    =
"STORE + SEARCH"

Kibana
    =
"INVESTIGATE"
```

And the complete logging path:

```text
Application
    ↓
stdout/stderr
    ↓
Kubernetes node logs
    ↓
Fluent Bit DaemonSet
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

The next major chapter is:

# `36-Logstash.md`

There we'll build the **central log-processing layer** in detail:

```text
Fluent Bit
     │
     ▼
Kubernetes Service
     │
     ▼
┌───────────────┐
│  Logstash-1   │
│  Logstash-2   │
└───────┬───────┘
        │
        ▼
 Elasticsearch
```

It will cover **Logstash architecture, inputs/filters/outputs, pipelines, persistent queues, workers, pipeline tuning, multiline handling, JSON parsing, field normalization, routing logs by service/environment, TLS, secrets, HA, Kubernetes deployment, resource sizing, backpressure, dead-letter handling, Elasticsearch output, and real production troubleshooting**.
