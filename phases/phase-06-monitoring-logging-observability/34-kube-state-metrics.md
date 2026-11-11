# 34 — kube-state-metrics

> Production-grade Kubernetes object-state monitoring for the Enterprise E-Commerce Platform on AWS EKS

---

## 1. Purpose

In the previous chapter, we covered **Node Exporter**.

Node Exporter answers:

> **“What is happening on the Linux worker node?”**

Now we need another perspective:

> **“What is Kubernetes reporting about its own objects?”**

That is the role of **kube-state-metrics (KSM)**.

The architecture becomes:

```text
Kubernetes API
      │
      ▼
kube-state-metrics
      │
      ▼
Prometheus
      │
      ├──────────────► Grafana
      │
      └──────────────► Alertmanager
```

KSM exposes metrics about Kubernetes objects such as:

* Pods
* Deployments
* ReplicaSets
* StatefulSets
* DaemonSets
* Jobs
* CronJobs
* Nodes
* Namespaces
* Services
* ConfigMaps
* Secrets metadata
* Resource requests/limits
* HPAs
* PDBs

---

# 2. Node Exporter vs kube-state-metrics

This distinction is extremely important.

### Node Exporter

```text
EC2 / Linux
    ↓
CPU
Memory
Disk
Network
Filesystem
```

### kube-state-metrics

```text
Kubernetes API
    ↓
Deployment state
Pod state
Replica state
HPA state
Node conditions
PDB state
Job state
```

Think:

```text
Node Exporter
    ↓
"What is the machine doing?"

KSM
    ↓
"What does Kubernetes say about its objects?"
```

---

# 3. KSM Does Not Collect Container CPU

A common misconception is:

> kube-state-metrics collects everything about Pods.

It does not.

For example:

```text
Pod CPU usage
```

comes from Kubernetes/container metrics infrastructure, not simply from KSM.

KSM gives you object-state information such as:

```text
Pod phase
Pod status
Pod labels
Pod owner
Deployment replicas
HPA desired replicas
Node conditions
```

So our observability architecture combines multiple sources.

---

# 4. Production Architecture

Our EKS monitoring architecture:

```text
                         EKS
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       Linux Nodes    Kubernetes API   Applications
          │               │                │
          ▼               ▼                ▼
    Node Exporter    kube-state-       Micrometer
                     metrics
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                     Prometheus
                          │
                 ┌────────┴────────┐
                 ▼                 ▼
              Grafana         Alertmanager
```

This is the foundation of our Kubernetes monitoring.

---

# 5. Where KSM Runs

KSM normally runs inside the cluster.

Example:

```text
namespace: monitoring
```

Architecture:

```text
EKS
└── monitoring
    ├── prometheus
    ├── grafana
    ├── alertmanager
    ├── node-exporter
    └── kube-state-metrics
```

Unlike Node Exporter, KSM does **not** need one Pod per node.

It is normally deployed as one or more centralized Pods.

---

# 6. Why KSM Needs Kubernetes API Access

KSM observes Kubernetes objects.

Therefore:

```text
Kubernetes API
       │
       ▼
KSM
```

KSM watches object state and converts that state into Prometheus metrics.

For example:

```text
Deployment:
desired replicas = 5
available replicas = 3
```

KSM exposes metrics representing those states.

Prometheus then scrapes KSM.

---

# 7. RBAC

KSM needs permission to watch/read supported Kubernetes resources.

Conceptually:

```text
ServiceAccount
      │
      ▼
ClusterRole
      │
      ▼
ClusterRoleBinding
      │
      ▼
Kubernetes API
```

The permissions should follow least privilege.

Do not give KSM:

```text
cluster-admin
```

just because it needs cluster-wide read access.

---

# 8. ServiceAccount

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
```

The actual Helm chart normally creates the required ServiceAccount and RBAC resources.

Production recommendation:

```text
Use the maintained Helm/chart configuration
+
review generated RBAC
```

rather than manually recreating every component.

---

# 9. Installation

KSM is commonly installed through the Prometheus ecosystem.

For example, `kube-prometheus-stack` can deploy kube-state-metrics.

Our preferred production model:

```text
Git
 ↓
Helm values
 ↓
ArgoCD
 ↓
EKS
 ↓
kube-state-metrics
```

Not:

```text
Engineer laptop
 ↓
kubectl apply
 ↓
Production
```

---

# 10. Verify Deployment

Check:

```bash
kubectl get deployment -n monitoring
```

Example:

```text
NAME                  READY
kube-state-metrics    1/1
```

Check Pods:

```bash
kubectl get pods -n monitoring
```

Then:

```bash
kubectl describe deployment kube-state-metrics -n monitoring
```

---

# 11. Verify Metrics

KSM exposes a Prometheus metrics endpoint.

You can inspect it:

```bash
kubectl port-forward \
  -n monitoring \
  deployment/kube-state-metrics \
  8080:8080
```

Then:

```bash
curl http://localhost:8080/metrics
```

You will find metrics with names similar to:

```text
kube_deployment_*
kube_pod_*
kube_node_*
kube_statefulset_*
kube_daemonset_*
kube_hpa_*
kube_persistentvolume_*
```

The exact metric set depends on the KSM version and enabled resources/collectors.

---

# 12. Prometheus Scraping

The flow is:

```text
Kubernetes API
      │
      ▼
KSM
      │
      │ /metrics
      ▼
Prometheus
```

With Prometheus Operator, discovery is typically configured through Kubernetes monitoring resources.

Check:

```bash
kubectl get servicemonitor -n monitoring
```

and:

```bash
kubectl get service -n monitoring
```

---

# 13. Deployment Metrics

One of the most useful areas is Deployment state.

For example:

```text
Desired:
5

Available:
5
```

Everything looks healthy.

But:

```text
Desired:
5

Available:
2
```

we have a problem.

The application may still be partially available, but capacity has dropped.

---

# 14. Desired vs Available Replicas

Conceptually:

```promql
kube_deployment_spec_replicas
```

versus:

```promql
kube_deployment_status_replicas_available
```

Then:

```text
Desired replicas
       ↓
5

Available replicas
       ↓
2
```

This tells us:

```text
3 replicas are missing from available capacity
```

---

# 15. Deployment Availability Alert

Example:

```yaml
- alert: DeploymentReplicasUnavailable
  expr: |
    kube_deployment_spec_replicas{
      namespace="production"
    }
    >
    kube_deployment_status_replicas_available{
      namespace="production"
    }

  for: 10m

  labels:
    severity: warning

  annotations:
    summary: "Deployment has unavailable replicas"
```

This is simplified.

In production, make the alert more intelligent by considering:

* desired replica count
* minimum acceptable availability
* rollout state
* service criticality

---

# 16. Deployment Failure Scenario

Suppose we deploy:

```text
order-service v1.5.0
```

Desired:

```text
5 replicas
```

But:

```text
Available = 0
```

KSM shows the Deployment is not achieving desired state.

Now investigate:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
   ↓
Pod events
   ↓
Container
```

Commands:

```bash
kubectl describe deployment order-service -n production
```

Then:

```bash
kubectl get rs -n production
```

Then:

```bash
kubectl get pods -n production
```

---

# 17. ReplicaSet Metrics

Deployment creates ReplicaSets.

Relationship:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ▼
Pods
```

KSM exposes ReplicaSet state.

This helps identify:

```text
Old ReplicaSet
New ReplicaSet
Desired replicas
Available replicas
Ready replicas
```

During rolling deployment:

```text
old RS
  ↓
new RS
```

we can observe the transition.

---

# 18. Rolling Deployment Monitoring

Suppose:

```text
v1:
5 replicas

v2:
0 replicas
```

Deployment begins.

Then:

```text
v1 → 4
v2 → 1

v1 → 3
v2 → 2

v1 → 2
v2 → 3

v1 → 0
v2 → 5
```

KSM allows us to observe Kubernetes object state throughout the rollout.

---

# 19. StatefulSet Metrics

For stateful workloads:

```text
StatefulSet
```

KSM can expose state related to:

```text
desired replicas
ready replicas
current replicas
updated replicas
```

Example:

```text
Elasticsearch
desired = 3
ready = 2
```

This may indicate:

```text
one node unavailable
```

For databases or distributed systems, this can be operationally important.

---

# 20. DaemonSet Metrics

DaemonSets are important in our platform because we use them for components such as:

```text
Node Exporter
Fluent Bit
```

KSM can expose:

```text
desired number
current number
ready number
available number
updated number
```

Example:

```text
Desired:
6

Ready:
5
```

One node is missing its DaemonSet Pod.

That can be important for:

```text
logging
monitoring
security agents
```

---

# 21. Fluent Bit Example

Suppose we have:

```text
6 EKS nodes
```

Fluent Bit DaemonSet should normally have:

```text
6 desired
6 ready
```

If:

```text
6 desired
5 ready
```

we may have:

```text
one node not sending logs
```

KSM can expose the DaemonSet state.

---

# 22. Pod Metrics

KSM exposes Kubernetes object state for Pods.

Examples include concepts such as:

```text
Pod phase
Pod labels
Pod owner
Container waiting state
Container terminated state
Container readiness
```

This allows us to identify problematic states.

---

# 23. Pod Phase

Typical phases include:

```text
Pending
Running
Succeeded
Failed
Unknown
```

For an application:

```text
order-service:
Running
```

is normal.

But:

```text
order-service:
Pending
```

for a long period may indicate:

```text
insufficient capacity
taint mismatch
affinity problem
resource request too large
IP exhaustion
volume issue
```

---

# 24. Pending Pod Alert

A conceptual alert:

```yaml
- alert: PodStuckPending
  expr: |
    kube_pod_status_phase{
      namespace="production",
      phase="Pending"
    } == 1

  for: 10m

  labels:
    severity: warning
```

Again, don't alert simply because a Pod is briefly Pending.

Scheduling can legitimately take a short time.

---

# 25. CrashLoopBackOff

KSM provides useful state information about containers.

However, don't think:

```text
kube-state-metrics
    =
CrashLoopBackOff detector
```

Instead, combine:

```text
KSM
+
container restart metrics
+
Pod status
+
kubelet/container runtime signals
```

For example:

```promql
increase(
  kube_pod_container_status_restarts_total{
    namespace="production"
  }[15m]
) > 5
```

This indicates repeated restarts.

---

# 26. Container Waiting State

You can inspect metrics representing container waiting reasons.

For example, reasons can include states such as:

```text
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
CreateContainerConfigError
```

These are extremely useful when building Kubernetes dashboards.

---

# 27. ImagePullBackOff

Example incident:

```text
Deployment
   ↓
Pod created
   ↓
ImagePullBackOff
```

KSM can expose the container waiting reason.

Grafana:

```text
order-service
ImagePullBackOff
```

Then investigate:

```bash
kubectl describe pod <pod> -n production
```

Look for:

```text
image name
registry
credentials
network
IAM
ECR
tag/digest
```

---

# 28. ECR Troubleshooting

Our architecture uses ECR:

```text
CI
 ↓
ECR
 ↓
ArgoCD
 ↓
EKS
```

Suppose:

```text
Pod = ImagePullBackOff
```

Potential causes:

```text
Wrong repository
Wrong tag
Wrong digest
Image deleted
Node cannot reach ECR
ECR authentication problem
IAM issue
Network issue
```

KSM detects the Kubernetes state.

The actual diagnosis requires deeper investigation.

---

# 29. Node Conditions

KSM also exposes Kubernetes node-state information.

Important conditions include:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Example:

```text
Node:
Ready = false
```

or:

```text
DiskPressure = true
```

This complements Node Exporter.

---

# 30. Node Exporter + KSM

Suppose:

```text
Node DiskPressure = true
```

KSM tells us:

```text
Kubernetes sees DiskPressure
```

Node Exporter tells us:

```text
Filesystem usage = 97%
```

Together:

```text
KSM
 ↓
DiskPressure

Node Exporter
 ↓
Filesystem nearly full
```

Now the investigation is much clearer.

---

# 31. HPA Metrics

KSM can expose HPA state.

Important concepts:

```text
desired replicas
current replicas
minimum replicas
maximum replicas
```

Example:

```text
HPA:

min = 3
max = 10

current = 10
desired = 10
```

This tells us:

```text
HPA is already at maximum capacity.
```

---

# 32. HPA Maximum Alert

Example:

```yaml
- alert: HPAAtMaximumReplicas
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas
    >=
    kube_horizontalpodautoscaler_spec_max_replicas

  for: 10m

  labels:
    severity: warning
```

This should be correlated with:

```text
request rate
latency
error rate
CPU
memory
pending Pods
```

Being at maximum replicas is not inherently a failure.

---

# 33. HPA Scaling Scenario

Traffic increases:

```text
100 req/s
   ↓
500 req/s
   ↓
1000 req/s
```

HPA:

```text
3 → 5 → 8 → 10
```

KSM:

```text
desired replicas = 10
current replicas = 10
max replicas = 10
```

Application:

```text
latency ↑
```

This is a strong signal:

```text
The workload may have reached its scaling ceiling.
```

Now investigate:

```text
HPA maxReplicas
node capacity
Karpenter
application bottleneck
RDS
Redis
external dependencies
```

---

# 34. PDB Metrics

PodDisruptionBudget protects availability during voluntary disruptions.

Example:

```yaml
minAvailable: 2
```

If:

```text
5 replicas
```

we require at least:

```text
2 available
```

during applicable voluntary disruption scenarios.

KSM can expose PDB state.

This is useful during:

```text
node drain
EKS upgrades
node replacement
maintenance
```

---

# 35. PDB Problem

Suppose:

```text
Deployment:
5 replicas

PDB:
minAvailable = 5
```

Now an engineer tries:

```bash
kubectl drain node-1
```

Kubernetes may be unable to evict a Pod because the PDB blocks voluntary disruption.

This can cause:

```text
node drain hangs
```

Monitoring PDB state helps identify the issue.

---

# 36. Namespace Metrics

Namespaces are important for our architecture:

```text
production
monitoring
logging
tracing
argocd
ingress-system
```

We can build dashboards showing:

```text
Pods by namespace
Deployments by namespace
Failed Pods
Pending Pods
Restarting containers
Resource requests
```

This provides a cluster-level operational view.

---

# 37. Resource Requests

KSM exposes Kubernetes resource request information.

For example:

```text
CPU requests
Memory requests
```

This is different from actual usage.

Example:

```text
Pod:
CPU request = 500m
Actual CPU = 100m
```

This tells us:

```text
Scheduler reserves 500m
but workload currently consumes ~100m
```

---

# 38. Requests vs Usage

This distinction is essential.

```text
KSM
 ↓
Requests / limits

Container metrics
 ↓
Actual usage

Node Exporter
 ↓
Host usage
```

Therefore:

```text
Requested CPU
≠
Actual CPU
≠
Node CPU
```

---

# 39. Capacity Dashboard

Combine these sources:

```text
                 EKS Node
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     KSM         Container    Node Exporter
                  metrics
        │           │           │
        ▼           ▼           ▼
   Requests      Usage      Host usage
```

Then Grafana can show:

```text
CPU allocatable
CPU requested
CPU actual
Memory allocatable
Memory requested
Memory actual
```

This is far more useful than a single CPU percentage.

---

# 40. Jobs

KSM can expose Job state.

Important information:

```text
active
succeeded
failed
completion time
start time
```

This is useful for:

```text
database migrations
batch jobs
reports
cleanup jobs
maintenance
```

---

# 41. Failed Job Alert

Example:

```yaml
- alert: KubernetesJobFailed
  expr: |
    kube_job_status_failed{
      namespace="production"
    } > 0

  for: 5m

  labels:
    severity: warning
```

But distinguish:

```text
expected retry
```

from:

```text
permanently failed production job
```

---

# 42. CronJobs

Our platform may use CronJobs for tasks such as:

```text
report generation
cleanup
reconciliation
```

Monitor:

```text
last schedule
active jobs
failed jobs
successful jobs
```

A CronJob that silently stops running can be a serious business problem.

---

# 43. Example Business Failure

Suppose:

```text
reporting-service
```

generates a daily report.

CronJob:

```text
02:00
```

But:

```text
Job failed
```

Application APIs remain healthy.

Grafana:

```text
APIs:
Healthy

Nodes:
Healthy

RDS:
Healthy
```

But:

```text
CronJob:
Failed
```

This demonstrates why object-state monitoring matters.

---

# 44. Service Objects

KSM can expose information about Kubernetes Services.

This helps correlate:

```text
Service
 ↓
Endpoints / EndpointSlices
 ↓
Pods
```

A Service may exist:

```text
order-service
```

while its backend endpoints are missing.

Then:

```text
Service exists
+
No healthy backend
=
traffic failure
```

---

# 45. Service Selector Problem

Example:

```yaml
selector:
  app: order-service
```

but Pods have:

```yaml
labels:
  app: orders-service
```

Notice:

```text
order-service
orders-service
```

The Service selects nothing.

Result:

```text
ALB
 ↓
Service
 ↓
No endpoints
```

Kubernetes object-state metrics help surface this type of condition.

---

# 46. Real Production Scenario — Deployment Failure

Imagine ArgoCD deploys:

```text
order-service v2.0
```

Grafana shows:

```text
Desired replicas: 5
Available: 0
```

KSM:

```text
Deployment unavailable
```

Then Pod dashboard:

```text
Pods:
CrashLoopBackOff
```

Application metrics:

```text
No traffic
```

Logs:

```text
Database migration failed
```

Now the incident chain is:

```text
ArgoCD
 ↓
Deployment
 ↓
KSM
 ↓
Pod state
 ↓
Logs
 ↓
Root cause
```

---

# 47. Real Production Scenario — Node Failure

Suppose:

```text
node-3
```

fails.

KSM:

```text
Ready = false
```

Node Exporter:

```text
No more metrics from node-3
```

Kubernetes:

```text
Pods rescheduled
```

KSM:

```text
Deployment available replicas temporarily decrease
```

Grafana:

```text
Node down
+
Replica reduction
```

Alertmanager:

```text
NodeNotReady
```

This is the observability chain we want.

---

# 48. Real Production Scenario — HPA Limit

Traffic:

```text
↑↑↑
```

Application:

```text
latency ↑
```

HPA:

```text
3 → 10
```

KSM:

```text
desired = 10
max = 10
```

Node capacity:

```text
available
```

Therefore:

```text
Kubernetes has capacity
but
HPA is capped
```

Potential action:

```text
increase maxReplicas
```

—but only after understanding:

```text
RDS capacity
Redis capacity
application bottleneck
cost
business load
```

---

# 49. Real Production Scenario — PDB Blocking Upgrade

EKS node upgrade starts.

```text
kubectl drain
```

gets stuck.

Investigation:

```text
Pod eviction blocked
       ↓
PDB
       ↓
minAvailable too restrictive
```

KSM:

```text
PDB allowed disruptions = 0
```

This gives the SRE a direct clue.

---

# 50. Grafana Dashboard — Kubernetes Overview

Recommended dashboard:

```text
Kubernetes Cluster Overview
```

Panels:

```text
Nodes Ready
Nodes NotReady
Pods Running
Pods Pending
Pods Failed
Deployments Available
Deployments Degraded
DaemonSets Ready
StatefulSets Ready
Jobs Failed
HPAs at Maximum
PDBs with Zero Disruptions Allowed
```

---

# 51. Namespace Dashboard

Example:

```text
Namespace: production
```

Show:

```text
Deployments
Pods
Replica availability
Restarts
CPU requests
Memory requests
CPU usage
Memory usage
```

Variables:

```text
namespace
workload
pod
container
```

---

# 52. Workload Dashboard

For:

```text
order-service
```

Show:

```text
Deployment desired replicas
Deployment available replicas
ReplicaSet status
Pod phase
Pod restarts
CPU
Memory
HPA current replicas
HPA desired replicas
HPA max replicas
PDB state
```

This creates a powerful workload-level dashboard.

---

# 53. KSM Cardinality

KSM can generate a large number of metrics.

Especially dangerous are labels involving:

```text
pod
container
namespace
node
service
deployment
custom labels
```

Our 21 microservices can produce thousands of series.

Therefore:

```text
More objects
+
More labels
=
More Prometheus series
```

Monitor Prometheus cardinality.

---

# 54. Don't Export Everything Blindly

A common mistake:

```text
"Let's enable every KSM metric."
```

Instead:

```text
What does SRE need?
       ↓
What dashboards need?
       ↓
What alerts need?
       ↓
Enable required metrics
```

This reduces:

```text
Prometheus storage
query load
memory
network
cardinality
```

---

# 55. KSM Scaling

For a small cluster:

```text
1 KSM replica
```

may be sufficient.

For a large production cluster:

```text
2+ replicas
```

may be considered for availability.

However, simply increasing replicas does not automatically improve everything.

You must consider:

```text
API server load
KSM watch load
Prometheus scrape load
duplicate metrics
resource consumption
```

Use the supported HA/deployment pattern for the KSM version and chart being deployed.

---

# 56. API Server Load

KSM watches Kubernetes resources.

Therefore excessive KSM configuration can increase:

```text
Kubernetes API usage
```

Monitor:

```text
API server latency
API server request rate
KSM resource consumption
```

This becomes increasingly important as the cluster grows.

---

# 57. KSM Security

KSM should:

```text
Read Kubernetes state
```

not:

```text
Modify Kubernetes state
```

Its permissions should therefore be primarily read/watch/list operations.

Avoid unnecessary:

```text
create
update
patch
delete
```

permissions.

---

# 58. NetworkPolicy

KSM needs to communicate with:

```text
Kubernetes API
Prometheus
```

Conceptually:

```text
KSM
 ├──► Kubernetes API
 └──► Prometheus
```

Other application namespaces should not need arbitrary access to KSM.

---

# 59. KSM and ArgoCD

ArgoCD tells us:

```text
Desired application state
```

KSM tells us:

```text
Actual Kubernetes object state
```

These complement each other.

Example:

```text
GitOps desired:
5 replicas

ArgoCD:
Synced

KSM:
2 replicas available
```

This is an important production insight:

> **GitOps synchronization does not necessarily mean the application is healthy.**

---

# 60. Desired State vs Actual State

This is one of the most important DevOps concepts in the whole project.

```text
Git
 ↓
Desired state
 ↓
ArgoCD
 ↓
Kubernetes
 ↓
Actual state
 ↓
KSM
 ↓
Prometheus
 ↓
Grafana
```

Example:

```text
Desired:
Deployment replicas = 5

Actual:
Available replicas = 2
```

ArgoCD may say:

```text
Synced
```

while the application is:

```text
Degraded
```

Both pieces of information are needed.

---

# 61. KSM + Grafana + Alertmanager

The complete flow:

```text
Kubernetes API
       │
       ▼
KSM
       │
       ▼
Prometheus
       │
   ┌───┴────┐
   ▼        ▼
Grafana  Alertmanager
   │        │
   ▼        ▼
Dashboard Notification
```

Example:

```text
Deployment available replicas < desired
             │
             ▼
         Prometheus
             │
      ┌──────┴──────┐
      ▼             ▼
   Grafana      Alertmanager
      │             │
      ▼             ▼
 Investigation   Slack/PagerDuty
```

---

# 62. Production Troubleshooting Checklist

### KSM Pod

```bash
kubectl get pods -n monitoring
kubectl describe pod <ksm-pod> -n monitoring
kubectl logs <ksm-pod> -n monitoring
```

### Service

```bash
kubectl get svc -n monitoring
```

### Endpoint

```bash
kubectl get endpoints -n monitoring
```

### ServiceMonitor

```bash
kubectl get servicemonitor -n monitoring
```

### Metrics

```bash
kubectl port-forward \
  -n monitoring \
  deployment/kube-state-metrics \
  8080:8080
```

Then:

```bash
curl http://localhost:8080/metrics
```

---

# 63. If KSM Has No Metrics

Check in this order:

```text
1. KSM Pod
      ↓
2. KSM logs
      ↓
3. Service
      ↓
4. /metrics endpoint
      ↓
5. ServiceMonitor
      ↓
6. Prometheus target
      ↓
7. RBAC
      ↓
8. Kubernetes API connectivity
```

Don't immediately reinstall the component.

Follow the dependency chain.

---

# 64. Production Alert Categories

For KSM-based alerts:

```text
Deployment unavailable
Pod stuck pending
Container restarting
DaemonSet unavailable
StatefulSet unavailable
Node not ready
Node pressure
HPA at maximum
PDB disruption blocked
Job failed
CronJob missed/failed
Replica mismatch
```

---

# 65. What KSM Should NOT Alert On

Avoid alerts for every tiny state change.

For example:

```text
Pod restarted once
```

may not require a page.

But:

```text
Pod restarted 20 times in 15 minutes
```

is much more actionable.

Likewise:

```text
Pod Pending for 3 seconds
```

is usually normal.

While:

```text
Pod Pending for 15 minutes
```

may indicate a real capacity/scheduling problem.

---

# 66. Production Tool Placement

| Component          | Location               | Primary responsibility     |
| ------------------ | ---------------------- | -------------------------- |
| Node Exporter      | EKS nodes / DaemonSet  | Linux host metrics         |
| kube-state-metrics | EKS monitoring         | Kubernetes object state    |
| Prometheus         | EKS monitoring         | Metrics collection/storage |
| Grafana            | EKS monitoring         | Visualization              |
| Alertmanager       | EKS monitoring         | Alert routing              |
| ArgoCD             | EKS platform namespace | Desired-state deployment   |
| CloudWatch         | AWS managed            | AWS service metrics        |

This layered architecture is important.

---

# 67. Final Mental Model

Remember these three layers:

```text
                 WHAT IS HAPPENING?
                        │
                        ▼
                 Node Exporter
                        │
              Linux / EC2 metrics


                 WHAT DOES K8S SAY?
                        │
                        ▼
              kube-state-metrics
                        │
            Kubernetes object state


                 WHAT IS APP DOING?
                        │
                        ▼
                  Micrometer
                        │
                Application metrics
```

All three feed:

```text
                 Prometheus
                      │
             ┌────────┴────────┐
             ▼                 ▼
          Grafana         Alertmanager
```

---

# 68. Complete Monitoring Architecture So Far

```text
                         EKS
                          │
        ┌─────────────────┼──────────────────┐
        │                 │                  │
        ▼                 ▼                  ▼
 Applications          EC2 Nodes       Kubernetes API
        │                 │                  │
        ▼                 ▼                  ▼
   Micrometer       Node Exporter     kube-state-metrics
        │                 │                  │
        └─────────────────┼──────────────────┘
                          ▼
                     Prometheus
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
          Grafana                Alertmanager
             │                         │
             ▼                         ▼
       Visualization             Notifications
```

And the troubleshooting flow is:

```text
Incident
   │
   ▼
Grafana
   │
   ├── Application metrics
   │
   ├── Node metrics
   │
   └── Kubernetes state
   │
   ▼
Alert
   │
   ▼
Logs
   │
   ▼
Traces
   │
   ▼
Root Cause
```

---

# 69. Production Checklist

### Deployment

* [ ] KSM installed through Helm/GitOps
* [ ] Version pinned
* [ ] RBAC reviewed
* [ ] Resource requests configured
* [ ] HA considered for cluster size
* [ ] Prometheus scraping verified

### Kubernetes visibility

* [ ] Pods
* [ ] Deployments
* [ ] ReplicaSets
* [ ] StatefulSets
* [ ] DaemonSets
* [ ] Nodes
* [ ] HPAs
* [ ] PDBs
* [ ] Jobs
* [ ] CronJobs
* [ ] Namespaces

### Dashboards

* [ ] Cluster overview
* [ ] Namespace dashboard
* [ ] Workload dashboard
* [ ] Deployment health
* [ ] HPA status
* [ ] PDB status
* [ ] Job status

### Alerts

* [ ] Deployment unavailable
* [ ] Pod stuck pending
* [ ] Excessive restarts
* [ ] Node not ready
* [ ] Node pressure
* [ ] HPA maxed out
* [ ] Failed Jobs
* [ ] DaemonSet unavailable

### Security

* [ ] Least-privilege RBAC
* [ ] Internal metrics endpoint
* [ ] NetworkPolicy
* [ ] No unnecessary write permissions
* [ ] Version/image controlled

---

## Next file

**`35-Fluent-Bit.md`**

Now we move from **metrics into logging**.

The production logging path we already defined will become concrete:

```text
21 Microservices
      │
      │ stdout / stderr
      ▼
Kubernetes Container Logs
      │
      ▼
Fluent Bit DaemonSet
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

The next chapter will go deeply into **Fluent Bit installation on every EKS node, DaemonSet architecture, `/var/log/containers`, container-runtime log format, Kubernetes metadata enrichment, multiline Java/Spring Boot exceptions, buffering, backpressure, memory limits, retries, Logstash output, TLS, secrets, log loss scenarios, node failures, and production troubleshooting**.
