# 126. Production Observability Runbooks

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Kubernetes:** Amazon EKS
> **Application:** Java + Spring Boot
> **Services:** 21 microservices
> **Metrics:** Prometheus
> **Dashboards:** Grafana
> **Logs:** ELK
> **Traces:** OpenTelemetry + Jaeger v2
> **Alerting:** Alertmanager
> **Deployment:** ArgoCD + GitOps

---

# 1. Objective

We now have:

```text
Prometheus → Metrics
ELK        → Logs
Jaeger     → Traces
SLO        → Reliability targets
Alertmanager → Alerts
```

But production engineers need more than dashboards.

When an alert fires at:

```text
02:15 AM
```

the engineer needs to know:

```text
What happened?
How do I confirm it?
What should I check first?
What commands should I run?
How do I identify the root cause?
How do I mitigate it safely?
How do I verify recovery?
What should go into the RCA?
```

That is the purpose of a **production runbook**.

---

# 2. What Is a Runbook?

A runbook is a documented operational procedure for handling a known problem.

Example:

```text
Alert:
OrderServiceHigh5xx

Runbook:
1. Confirm alert
2. Check Grafana
3. Check pods
4. Check application logs
5. Check Jaeger
6. Check dependencies
7. Identify cause
8. Mitigate
9. Verify
10. Record incident
```

---

# 3. Runbook vs Troubleshooting Guide

A troubleshooting guide can be broad:

```text
"Kubernetes pod troubleshooting"
```

A runbook is more operational:

```text
"Production order-service HTTP 5xx incident"
```

It should tell an engineer exactly what to do.

---

# 4. Standard Incident Flow

Every runbook in our platform follows:

```text
ALERT
  ↓
CONFIRM
  ↓
SCOPE
  ↓
INVESTIGATE
  ↓
CORRELATE
  ↓
MITIGATE
  ↓
VERIFY
  ↓
RECOVER
  ↓
RCA
  ↓
PREVENT RECURRENCE
```

---

# 5. Golden Investigation Workflow

For application incidents:

```text
Alertmanager
     |
     v
Grafana
     |
     +---- Metrics
     |
     v
Affected Service
     |
     +---- Kubernetes
     |
     +---- Logs / ELK
     |
     +---- Jaeger Trace
     |
     +---- RDS
     |
     +---- Redis
     |
     +---- SQS
     |
     +---- Kafka
     |
     +---- ALB
```

---

# 6. Incident Safety Rules

Before running commands:

```text
1. Confirm the environment.
2. Confirm the namespace.
3. Confirm the affected service.
4. Avoid destructive commands.
5. Capture evidence before changing anything.
6. Prefer reversible mitigation.
7. Record every production change.
```

Always verify:

```bash
kubectl config current-context
```

and:

```bash
kubectl get nodes
```

before making production changes.

---

# 7. Production Evidence Collection

Before changing anything, capture:

```bash
kubectl get pods -n ecommerce -o wide
```

```bash
kubectl get events -n ecommerce --sort-by=.lastTimestamp
```

```bash
kubectl get deploy -n ecommerce
```

```bash
kubectl get svc -n ecommerce
```

```bash
kubectl get ingress -n ecommerce
```

And application logs:

```bash
kubectl logs -n ecommerce deploy/order-service --since=30m
```

This gives us a baseline.

---

# 8. RUNBOOK 01 — High API Latency

## Alert

Example:

```text
OrderServiceHighLatency
```

Condition:

```text
P95/P99 latency above SLO threshold
```

---

## Step 1 — Confirm

Open Grafana.

Check:

```text
Request rate
P50
P95
P99
5xx
CPU
Memory
Pod restarts
```

Determine whether the problem is:

```text
single service
multiple services
entire cluster
```

---

## Step 2 — Check Kubernetes

```bash
kubectl get pods -n ecommerce -l app=order-service
```

Check resources:

```bash
kubectl top pods -n ecommerce -l app=order-service
```

Check deployment:

```bash
kubectl describe deployment order-service -n ecommerce
```

---

## Step 3 — Check Logs

```bash
kubectl logs -n ecommerce deploy/order-service --since=15m
```

Search for:

```text
timeout
connection
exception
database
Redis
Kafka
HTTP 5xx
```

---

## Step 4 — Check Jaeger

Find a slow trace.

Example:

```text
POST /api/v1/orders
       |
       +-- customer-service   40ms
       +-- pricing-service    50ms
       +-- payment-service    100ms
       +-- inventory-service  5.2s
```

The trace identifies where the latency is actually occurring.

---

## Step 5 — Check Dependencies

If inventory is slow:

```text
inventory-service
      |
      +-- PostgreSQL
```

Check:

```text
RDS latency
connections
locks
slow queries
```

If Redis:

```text
Redis latency
connections
memory
evictions
```

---

## Step 6 — Mitigation

Possible mitigations:

```text
scale service
rollback recent deployment
reduce traffic
disable non-critical feature
increase capacity
fix dependency
```

Do not blindly increase timeouts.

---

## Step 7 — Verify

Confirm:

```text
P95 ↓
P99 ↓
5xx normal
SLO recovering
Jaeger traces normal
```

---

# 9. RUNBOOK 02 — High HTTP 5xx

## Alert

```text
OrderServiceHigh5xx
```

---

## Step 1

Grafana:

```text
5xx rate
request rate
latency
```

Determine:

```text
5xx from application?
5xx from ALB?
5xx from dependency?
```

---

## Step 2 — Kubernetes

```bash
kubectl get pods -n ecommerce -l app=order-service
```

```bash
kubectl get events -n ecommerce --sort-by=.lastTimestamp
```

---

## Step 3 — Logs

```bash
kubectl logs -n ecommerce deploy/order-service --since=10m
```

Look for:

```text
NullPointerException
SQLException
TimeoutException
Connection refused
OutOfMemoryError
```

---

## Step 4 — Jaeger

Find failed traces.

```text
order-service
      |
      +-- payment-service
             |
             X
          HTTP 500
```

---

## Step 5 — Check Recent Deployment

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

If the incident started immediately after a release, compare:

```text
deployment timestamp
incident timestamp
```

---

## Step 6 — Mitigation

If deployment is confirmed as the cause:

```bash
kubectl rollout undo deployment/order-service -n ecommerce
```

**Controlled production action:** only perform rollback according to your organization's change/incident procedure and after confirming the target revision.

---

## Step 7 — Verify

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

Then verify:

```text
5xx
latency
SLO
Jaeger
business transaction
```

---

# 10. RUNBOOK 03 — CrashLoopBackOff

## Alert

```text
KubernetesPodCrashLooping
```

---

## Step 1

```bash
kubectl get pods -n ecommerce
```

Find:

```text
order-service-xxxxx
CrashLoopBackOff
```

---

## Step 2

```bash
kubectl describe pod <pod> -n ecommerce
```

Look at:

```text
Last State
Reason
Exit Code
Events
```

---

## Step 3 — Previous Logs

Very important:

```bash
kubectl logs <pod> -n ecommerce --previous
```

Because the current container may have already restarted.

---

## Step 4 — Common Causes

```text
bad configuration
missing secret
database unavailable
application startup exception
wrong command
image issue
failed health check
```

---

## Step 5 — Check Config

```bash
kubectl get configmap -n ecommerce
```

```bash
kubectl get secret -n ecommerce
```

Do not print secret contents into an incident channel.

---

## Step 6 — Check Recent Deployment

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

---

## Step 7 — Mitigate

Possible:

```text
rollback
restore configuration
fix Secret reference
restore dependency
```

---

# 11. RUNBOOK 04 — OOMKilled

## Alert

```text
ContainerMemoryLimitExceeded
```

---

## Step 1

```bash
kubectl get pods -n ecommerce
```

Then:

```bash
kubectl describe pod <pod> -n ecommerce
```

Look for:

```text
Reason: OOMKilled
```

---

## Step 2 — Check Memory

```bash
kubectl top pod <pod> -n ecommerce
```

Check Deployment:

```bash
kubectl get deployment order-service -n ecommerce -o yaml
```

Look for:

```yaml
resources:
  requests:
    memory: ...
  limits:
    memory: ...
```

---

## Step 3 — Determine Cause

Possible:

```text
memory leak
traffic increase
large payload
too-low memory limit
unbounded cache
JVM heap configuration
```

---

## Step 4 — JVM Investigation

For Java/Spring Boot:

```text
heap
GC
threads
heap dump
JVM metrics
```

should be investigated.

Do not immediately increase memory without understanding whether the application is leaking memory.

---

## Step 5 — Mitigation

Depending on evidence:

```text
rollback
scale horizontally
temporarily increase memory
reduce traffic
fix application
```

---

# 12. RUNBOOK 05 — CPU Saturation

## Alert

```text
ContainerHighCPU
```

---

## Step 1

```bash
kubectl top pods -n ecommerce
```

---

## Step 2

Check HPA:

```bash
kubectl get hpa -n ecommerce
```

```bash
kubectl describe hpa order-service -n ecommerce
```

---

## Step 3

Check replicas:

```bash
kubectl get deployment order-service -n ecommerce
```

---

## Step 4

Check application latency.

High CPU + high latency can indicate:

```text
CPU saturation
```

But CPU alone doesn't establish root cause.

---

## Step 5

Check Jaeger for expensive operations:

```text
large DB query
serialization
external API
business computation
```

---

## Step 6

Mitigation:

```text
scale horizontally
optimize code
optimize database query
increase CPU capacity
```

Prefer fixing the underlying bottleneck rather than permanently overprovisioning.

---

# 13. RUNBOOK 06 — Memory Saturation

Check:

```bash
kubectl top pods -n ecommerce
```

and:

```bash
kubectl top nodes
```

Determine whether memory pressure is:

```text
Pod
Node
Entire cluster
```

Node pressure:

```bash
kubectl describe node <node>
```

Look for:

```text
MemoryPressure
```

---

# 14. RUNBOOK 07 — Disk Full

## Check Node

```bash
df -h
```

For Kubernetes nodes, investigate node-level filesystem pressure through:

```bash
kubectl describe node <node>
```

Look for:

```text
DiskPressure
```

---

## Application Logs

Large logs can consume disk.

Check:

```bash
du -sh /var/log/*
```

on systems where you have appropriate node access.

Also inspect:

```text
container logs
container runtime storage
temporary files
image layers
```

---

## Mitigation

Depending on the cause:

```text
rotate/retire logs
remove unused images according to node-management policy
increase node disk
replace node
fix excessive application logging
```

Do not manually delete arbitrary Kubernetes/container runtime files on a production node without following the platform runbook.

---

# 15. RUNBOOK 08 — ImagePullBackOff

## Step 1

```bash
kubectl get pods -n ecommerce
```

---

## Step 2

```bash
kubectl describe pod <pod> -n ecommerce
```

Look for:

```text
Failed to pull image
unauthorized
manifest unknown
image not found
```

---

## Step 3 — Check ECR

Confirm:

```text
repository exists
image tag exists
image digest exists
```

Example:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

---

## Step 4 — IAM

Check the node/pod identity model used by EKS.

Potential causes:

```text
missing ECR permissions
incorrect image
wrong AWS account
wrong region
expired/misconfigured authentication
```

---

## Step 5 — GitOps

Check whether ArgoCD deployed the intended image:

```text
GitOps
  ↓
Helm values
  ↓
Deployment
  ↓
ECR
```

---

# 16. RUNBOOK 09 — ALB 5xx

ALB 5xx does not automatically mean the application generated the error.

First distinguish:

```text
ALB-generated error
```

from:

```text
target-generated 5xx
```

---

## Check

Grafana:

```text
ALB request count
HTTPCode_ELB_5XX_Count
HTTPCode_Target_5XX_Count
TargetResponseTime
HealthyHostCount
```

---

## Kubernetes

```bash
kubectl get ingress -n ecommerce
```

Check:

```text
AWS Load Balancer Controller
target groups
service
endpoints
```

---

## Service

```bash
kubectl get svc order-service -n ecommerce
```

```bash
kubectl get endpoints order-service -n ecommerce
```

---

# 17. ALB Troubleshooting Flow

```text
Client
  ↓
ALB
  ↓
Target Group
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Application
```

Investigate from left to right.

---

# 18. RUNBOOK 10 — Service Connectivity Failure

Suppose:

```text
order-service
      X
payment-service
```

---

## Step 1

Check service:

```bash
kubectl get svc payment-service -n ecommerce
```

---

## Step 2

Check endpoints:

```bash
kubectl get endpoints payment-service -n ecommerce
```

If no endpoints:

```text
selector mismatch
pods unavailable
readiness failure
```

---

## Step 3

Check DNS

From a diagnostic pod:

```bash
nslookup payment-service.ecommerce.svc.cluster.local
```

---

## Step 4

Check NetworkPolicy

```bash
kubectl get networkpolicy -n ecommerce
```

---

## Step 5

Check application logs and Jaeger.

Look for:

```text
connection refused
timeout
DNS failure
HTTP 503
```

---

# 19. RUNBOOK 11 — DNS Failure

Check CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Depending on the EKS configuration and labels, inspect the actual CoreDNS deployment/service.

```bash
kubectl get svc -n kube-system
```

---

## Test DNS

From a diagnostic pod:

```bash
nslookup kubernetes.default.svc.cluster.local
```

Then:

```bash
nslookup payment-service.ecommerce.svc.cluster.local
```

---

## Check CoreDNS Logs

```bash
kubectl logs -n kube-system deploy/coredns
```

Look for:

```text
timeouts
upstream errors
resource exhaustion
```

---

# 20. RUNBOOK 12 — RDS Connection Exhaustion

## Alert

```text
DatabaseConnectionPoolExhausted
```

---

## Step 1

Check application:

```text
Hikari active connections
Hikari idle connections
Hikari pending threads
```

---

## Step 2

Check RDS:

```text
DatabaseConnections
CPU
FreeableMemory
ReadLatency
WriteLatency
```

---

## Step 3

PostgreSQL:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Check connection states:

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

---

# 21. Common Causes

```text
too many pods
pool too large
connection leak
slow queries
long transactions
idle-in-transaction sessions
deployment surge
retry storm
```

Remember:

```text
Total DB connections
≈
number of pods × pool size
```

This was covered in our RDS connection-pooling chapter.

---

# 22. Mitigation

Possible:

```text
reduce HPA max replicas
reduce connection pool
terminate problematic sessions according to DB procedure
rollback bad deployment
optimize slow queries
reduce traffic
```

Do not terminate database sessions blindly.

---

# 23. RUNBOOK 13 — Redis Failure

## Alert

```text
RedisUnavailable
```

---

## Step 1

Check application logs:

```text
RedisConnectionException
timeout
connection refused
```

---

## Step 2

Check Redis metrics:

```text
CPU
memory
connections
latency
evictions
replication
failover
```

---

## Step 3

Determine:

```text
cache failure
```

or:

```text
application depends critically on Redis
```

Our preferred application pattern is:

```text
Application
    |
    v
Redis
    |
    X
    |
    v
Fallback → RDS
```

where appropriate.

---

# 24. Redis Failure Protection

Use:

```text
short timeout
circuit breaker
fallback
connection limits
request rate limiting
cache stampede protection
```

Avoid:

```text
Redis failure
    ↓
all requests hit RDS
    ↓
RDS overloaded
    ↓
complete outage
```

This was covered in Chapter 111.

---

# 25. RUNBOOK 14 — SQS Backlog

## Alert

```text
SQSBacklogHigh
```

Check:

```text
ApproximateNumberOfMessagesVisible
ApproximateAgeOfOldestMessage
```

---

## Step 1

Determine whether:

```text
producer spike
```

or:

```text
consumer capacity problem
```

---

## Step 2

Check consumers:

```bash
kubectl get pods -n ecommerce -l app=notification-service
```

Check HPA/KEDA:

```bash
kubectl get hpa -n ecommerce
```

---

## Step 3

Check application logs.

Possible:

```text
database slow
external API slow
poison messages
consumer crash
permission failure
```

---

# 26. SQS Mitigation

Possible:

```text
scale consumers
fix failed dependency
increase consumer capacity
pause producer if necessary
move poison messages to DLQ
controlled replay
```

Do not blindly replay a DLQ into production.

---

# 27. RUNBOOK 15 — Kafka Consumer Lag

## Alert

```text
KafkaConsumerLagHigh
```

Check:

```text
consumer lag
consumer group
partition distribution
consumer errors
broker health
```

---

## Step 1

Determine whether:

```text
producer rate increased
```

or:

```text
consumer processing slowed
```

---

## Step 2

Check Kubernetes:

```bash
kubectl get pods -n ecommerce -l app=inventory-service
```

---

## Step 3

Check Jaeger

Find slow consumer processing.

---

## Step 4

Check downstream:

```text
RDS
Redis
external APIs
```

---

# 28. Kafka Mitigation

Possible:

```text
scale consumers
increase partition-aware consumer capacity
fix slow processing
fix downstream dependency
reduce producer pressure
```

Avoid blindly increasing consumers if partitions are insufficient or the downstream database is already saturated.

---

# 29. RUNBOOK 16 — Node Failure

## Alert

```text
KubernetesNodeNotReady
```

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl describe node <node>
```

---

## Determine

```text
NotReady
MemoryPressure
DiskPressure
NetworkUnavailable
```

---

## Check Workloads

```bash
kubectl get pods -A -o wide
```

Determine whether workloads were rescheduled.

---

# 30. Node Failure Mitigation

If the cluster has sufficient capacity:

```text
Kubernetes reschedules workloads
```

Then verify:

```text
replicas
readiness
availability
SLO
```

If the node requires replacement, follow the EKS/node-group or Karpenter operational procedure rather than manually terminating infrastructure without checking workload disruption.

---

# 31. RUNBOOK 17 — AZ Failure

Our production design targets:

```text
3 Availability Zones
```

Workloads should be distributed across AZs.

---

## Check

```bash
kubectl get pods -n ecommerce -o wide
```

Determine:

```text
pod distribution
node distribution
```

---

## Check Topology

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

---

## Expected Architecture

```text
AZ-A
 ├── Nodes
 └── Pods

AZ-B
 ├── Nodes
 └── Pods

AZ-C
 ├── Nodes
 └── Pods
```

---

# 32. AZ Failure Mitigation

The architecture should allow:

```text
failed AZ
    ↓
remaining AZs
    ↓
continue serving traffic
```

provided:

```text
capacity
replicas
PDBs
load balancing
database HA
```

are correctly designed.

---

# 33. RUNBOOK 18 — ArgoCD Drift

## Alert

```text
ArgoCDApplicationOutOfSync
```

Check:

```text
ArgoCD UI
Application
Git revision
Live state
Desired state
```

---

## CLI

```bash
argocd app get order-service
```

Then:

```bash
argocd app diff order-service
```

---

# 34. Common Causes

```text
manual kubectl change
incorrect Git commit
Helm value change
controller mutation
configuration drift
resource generated by another controller
```

---

# 35. Correct Approach

If the live change is legitimate:

```text
update Git
```

Then:

```text
ArgoCD sync
```

If the live change is unauthorized:

```text
revert/restore desired Git state
```

Do not make repeated manual Kubernetes changes to fight GitOps.

---

# 36. RUNBOOK 19 — Bad Production Deployment

Symptoms:

```text
5xx ↑
latency ↑
pods unhealthy
SLO burn ↑
```

---

## Step 1

Check:

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

---

## Step 2

Check history:

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

---

## Step 3

Compare:

```text
current image
previous image
Git commit
deployment timestamp
incident timestamp
```

---

## Step 4

Check Jaeger

Look for:

```text
new errors
new latency
dependency behavior
```

---

## Step 5

Rollback if confirmed.

```bash
kubectl rollout undo deployment/order-service -n ecommerce
```

Then update the GitOps source as required by the organization's GitOps rollback procedure so that the desired state and live state remain consistent.

---

# 37. RUNBOOK 20 — Security Incident

Security incidents require a different approach.

Examples:

```text
credential exposure
suspicious pod
unexpected IAM activity
container vulnerability
unexpected outbound traffic
```

---

## First Rule

Do not destroy evidence.

Before changing things:

```text
capture logs
capture timestamps
capture affected resources
capture relevant CloudTrail events
capture Kubernetes audit/security telemetry where available
```

---

## Containment

Depending on the incident:

```text
isolate workload
revoke compromised credentials
disable affected identity
block network path
quarantine workload
```

Actions should follow the organization's security incident response process.

---

# 38. RUNBOOK 21 — Prometheus Failure

Symptoms:

```text
dashboards empty
alerts missing
metrics unavailable
```

Check:

```bash
kubectl get pods -n observability
```

---

## Check

```text
Prometheus Pod
PVC
Service
Targets
resource usage
configuration
```

If Prometheus itself is down:

```text
Alertmanager may not receive new Prometheus alerts
Grafana may have stale/no metrics
```

This is why observability infrastructure must be monitored separately.

---

# 39. RUNBOOK 22 — ELK Failure

Symptoms:

```text
logs missing
Kibana unavailable
```

Check:

```text
Fluent Bit
Logstash
Elasticsearch
Kibana
```

Architecture:

```text
Application
    ↓
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

Identify the broken layer before restarting everything.

---

# 40. RUNBOOK 23 — OpenTelemetry Collector Failure

Symptoms:

```text
No new Jaeger traces
```

Check:

```bash
kubectl get pods -n observability
```

Then:

```bash
kubectl logs -n observability deploy/otel-collector
```

Check:

```text
receiver
processor
exporter
memory
queue
network
Jaeger
```

---

# 41. RUNBOOK 24 — Jaeger Failure

Check:

```text
Jaeger Collector
Jaeger Query
storage
network
resource usage
```

Architecture:

```text
Application
    ↓
OTel Collector
    ↓
Jaeger
    ↓
Storage
```

Determine whether:

```text
ingestion failure
```

or:

```text
query failure
```

or:

```text
storage failure
```

---

# 42. Observability Dependency Rule

A critical principle:

```text
Business request
      |
      +---- Metrics
      +---- Logs
      +---- Traces
```

The business request should not normally become:

```text
Business request
      |
      v
Tracing backend
      |
      X
      |
Business request fails
```

Observability should be designed to degrade safely.

---

# 43. Runbook Command Reference

### Kubernetes

```bash
kubectl get pods -A
```

```bash
kubectl describe pod <pod> -n <namespace>
```

```bash
kubectl logs <pod> -n <namespace>
```

```bash
kubectl logs <pod> -n <namespace> --previous
```

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

```bash
kubectl get nodes
```

```bash
kubectl describe node <node>
```

```bash
kubectl top pods -n <namespace>
```

```bash
kubectl top nodes
```

---

# 44. Deployment Commands

```bash
kubectl get deployment -n ecommerce
```

```bash
kubectl describe deployment order-service -n ecommerce
```

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

Controlled rollback:

```bash
kubectl rollout undo deployment/order-service -n ecommerce
```

---

# 45. Service Commands

```bash
kubectl get svc -n ecommerce
```

```bash
kubectl describe svc order-service -n ecommerce
```

```bash
kubectl get endpoints order-service -n ecommerce
```

For newer Kubernetes environments, EndpointSlices may also be useful:

```bash
kubectl get endpointslice -n ecommerce
```

---

# 46. HPA Commands

```bash
kubectl get hpa -n ecommerce
```

```bash
kubectl describe hpa order-service -n ecommerce
```

Check:

```text
current replicas
desired replicas
CPU
memory
custom metrics
events
```

---

# 47. ArgoCD Commands

```bash
argocd app list
```

```bash
argocd app get order-service
```

```bash
argocd app diff order-service
```

```bash
argocd app history order-service
```

Production sync/rollback actions should follow your organization's GitOps approval process.

---

# 48. AWS Commands

Check EKS:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --region ap-south-1
```

Check ECR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

Check RDS:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-instance> \
  --region ap-south-1
```

These commands are read-only inspection examples.

---

# 49. Incident Severity

We can define:

```text
SEV-1
SEV-2
SEV-3
SEV-4
```

Example definitions:

### SEV-1

Major customer/business impact or critical production outage.

### SEV-2

Significant degradation affecting an important portion of traffic.

### SEV-3

Limited impact with workaround available.

### SEV-4

Minor issue or non-urgent defect.

Exact definitions should be agreed by the organization.

---

# 50. Incident Roles

For a serious incident:

```text
Incident Commander
        |
        +-- Technical Lead
        |
        +-- Communications
        |
        +-- SRE/Platform
        |
        +-- Application Team
        |
        +-- Security if required
```

The Incident Commander coordinates rather than personally performing every technical task.

---

# 51. Incident Timeline

Always maintain:

```text
10:00 — Deployment started
10:03 — First 5xx increase
10:05 — Alert fired
10:07 — Incident declared
10:10 — Root cause suspected
10:12 — Rollback started
10:15 — Rollback completed
10:18 — Error rate normalized
10:25 — Customer validation completed
```

This is essential for the RCA.

---

# 52. Evidence Collection

Capture:

```text
Grafana screenshots/queries
Prometheus metrics
Jaeger trace IDs
ELK log searches
Kubernetes events
ArgoCD revision
Git commit
Docker image tag/digest
AWS events
deployment timestamps
```

Avoid storing secrets in incident documents.

---

# 53. Communication During Incident

Good incident communication is factual:

```text
"order-service is returning elevated 5xx responses."

"Impact began at 10:03 IST."

"Approximately 12% of eligible requests are failing."

"Investigation points to the latest deployment."

"Rollback is in progress."
```

Avoid unsupported statements like:

```text
"Everything is definitely fixed."
```

until validation is complete.

---

# 54. Mitigation vs Root Cause

These are different.

Example:

```text
Root cause:
database connection pool exhaustion
```

Mitigation:

```text
rollback deployment
```

The rollback restores service but does not necessarily fix:

```text
connection pool configuration
```

Therefore:

```text
Mitigation
≠
Root Cause Fix
```

---

# 55. Verification After Mitigation

Never stop after:

```text
kubectl rollout status
```

Verify customer behavior.

Check:

```text
5xx
latency
availability
SLO
Jaeger
logs
business transaction
```

For order-service:

```text
create order
payment
inventory reservation
order status
```

should be validated according to the incident.

---

# 56. Post-Incident RCA

Every significant incident should document:

```text
Incident ID
Date
Duration
Severity
Services affected
Customer impact
Detection
Timeline
Root cause
Contributing factors
Mitigation
Recovery
Error budget consumed
Corrective actions
Owners
Due dates
```

---

# 57. Example RCA

```text
Incident:
Order API latency

Severity:
SEV-2

Impact:
Order requests experienced elevated latency.

Start:
10:03 IST

End:
10:18 IST

Detection:
SLO burn-rate alert

Root cause:
Slow inventory database query

Mitigation:
Traffic reduced and affected workload stabilized

Permanent fix:
Database query optimization and index improvement

Follow-up:
Add query latency alert and regression test
```

---

# 58. Corrective Action Categories

Actions can be:

```text
Code
Infrastructure
Monitoring
Testing
Security
Capacity
Process
Documentation
```

Example:

```text
Code:
optimize SQL query

Monitoring:
add DB latency alert

Testing:
add load-test scenario

Capacity:
recalculate connection budget

Process:
require DB performance review
```

---

# 59. Runbook Repository

We should maintain runbooks in Git.

```text
platform/
└── runbooks/
    ├── README.md
    ├── kubernetes/
    │   ├── crashloopbackoff.md
    │   ├── oomkilled.md
    │   ├── cpu-saturation.md
    │   ├── memory-saturation.md
    │   └── node-failure.md
    │
    ├── application/
    │   ├── high-5xx.md
    │   ├── high-latency.md
    │   └── bad-deployment.md
    │
    ├── database/
    │   ├── rds-connection-exhaustion.md
    │   └── rds-failure.md
    │
    ├── messaging/
    │   ├── sqs-backlog.md
    │   └── kafka-consumer-lag.md
    │
    ├── observability/
    │   ├── prometheus-failure.md
    │   ├── elk-failure.md
    │   ├── otel-failure.md
    │   └── jaeger-failure.md
    │
    └── security/
        └── security-incident.md
```

---

# 60. Alert → Runbook Mapping

Every important alert should have a runbook.

Example:

```text
Alert                         Runbook
------------------------------------------------
OrderServiceHigh5xx           high-5xx.md
OrderServiceHighLatency      high-latency.md
PodCrashLooping               crashloopbackoff.md
PodOOMKilled                  oomkilled.md
NodeNotReady                  node-failure.md
RDSConnectionsHigh            rds-connection-exhaustion.md
RedisUnavailable              redis-failure.md
SQSBacklogHigh                sqs-backlog.md
KafkaConsumerLagHigh          kafka-consumer-lag.md
ArgoCDOutOfSync               argocd-drift.md
JaegerUnavailable             jaeger-failure.md
```

This is one of the most important operational practices in the project.

---

# 61. Alert Annotation

Prometheus alert rules should include a runbook reference.

Conceptually:

```yaml
annotations:
  summary: "Order service has elevated 5xx"
  runbook_url: "internal-runbook-reference"
```

The exact URL should point to the organization's actual documentation system or Git-based runbook.

---

# 62. Runbook Quality Standard

A good runbook should answer:

```text
What does this alert mean?
What is the customer impact?
What should I check first?
What commands should I run?
What evidence should I capture?
What are the likely causes?
What are safe mitigations?
How do I verify recovery?
When should I escalate?
What should I document?
```

If it doesn't answer these questions, it isn't production-ready.

---

# 63. Runbook Decision Tree

Example high-5xx decision tree:

```text
High 5xx
   |
   v
Is ALB generating it?
   |
  Yes ----------------> Check ALB/Target
   |
   No
   |
   v
Application generating it?
   |
  Yes
   |
   +---- Recent deployment?
   |          |
   |         Yes → Investigate/Rollback
   |
   +---- Dependency failure?
   |          |
   |         Yes → Investigate dependency
   |
   +---- Resource exhaustion?
              |
             Yes → Scale/Mitigate
```

---

# 64. Production Troubleshooting Principle

Always move from:

```text
Symptom
```

to:

```text
Evidence
```

then:

```text
Hypothesis
```

then:

```text
Test
```

then:

```text
Root cause
```

Example:

```text
Symptom:
API slow

Evidence:
P99 = 7 seconds

Hypothesis:
Database slow

Test:
Jaeger database span

Evidence:
DB query = 6.5 seconds

Root cause:
missing index
```

This is much stronger than guessing.

---

# 65. Don't Restart Everything

A common beginner response is:

```text
kubectl delete pod ...
```

or:

```text
restart everything
```

That can destroy evidence and make the problem harder to diagnose.

Prefer:

```text
observe
→ identify
→ mitigate
→ verify
```

Restart only when there is a justified operational reason.

---

# 66. Production Troubleshooting Hierarchy

Use this sequence:

```text
1. Customer impact
2. Scope
3. Recent changes
4. Metrics
5. Logs
6. Traces
7. Dependencies
8. Infrastructure
9. Mitigation
10. Verification
```

This keeps investigations structured.

---

# 67. Observability Runbook Architecture

Our complete system now becomes:

```text
                    Alertmanager
                         |
                         v
                    Alert fires
                         |
                         v
                      Runbook
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Metrics           Logs            Traces
   Prometheus           ELK            Jaeger
        |                |                |
        +----------------+----------------+
                         |
                         v
                    Root Cause
                         |
                         v
                     Mitigate
                         |
                         v
                      Verify
                         |
                         v
                       RCA
```

---

# 68. Production Incident Lifecycle

```text
Detection
    ↓
Triage
    ↓
Investigation
    ↓
Mitigation
    ↓
Recovery
    ↓
Validation
    ↓
Communication
    ↓
RCA
    ↓
Corrective Actions
    ↓
Runbook Improvement
```

Notice the final step:

> **Every major incident should improve the platform.**

---

# 69. What We Have Achieved

We now have:

```text
Metrics
    ↓
Prometheus

Logs
    ↓
ELK

Traces
    ↓
OpenTelemetry → Jaeger

Reliability
    ↓
SLO / Error Budget / Burn Rate

Operations
    ↓
Production Runbooks
```

So the platform is no longer just:

```text
"monitoring"
```

It has become:

```text
Observability
+
Reliability
+
Incident Response
```

---

# 70. Production Runbook Checklist

```text
[ ] Alert has owner
[ ] Alert has runbook
[ ] Customer impact defined
[ ] Severity defined
[ ] Scope procedure defined
[ ] Metrics investigation documented
[ ] Log investigation documented
[ ] Jaeger investigation documented
[ ] Kubernetes commands documented
[ ] AWS checks documented
[ ] Dependency checks documented
[ ] Safe mitigation documented
[ ] Rollback procedure documented
[ ] Recovery verification documented
[ ] Escalation path documented
[ ] Evidence collection documented
[ ] RCA template documented
[ ] Corrective actions tracked
```

---

# 71. Final Operational Model

Our production platform now follows:

```text
                        CUSTOMER
                           |
                           v
                      APPLICATION
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
     Metrics             Logs              Traces
        |                  |                  |
   Prometheus              ELK          OpenTelemetry
        |                                   |
        |                                   v
        |                                 Jaeger
        |                                   |
        +------------------+----------------+
                           |
                           v
                          SLO
                           |
                           v
                    Error Budget
                           |
                           v
                      Alertmanager
                           |
                           v
                       RUNBOOK
                           |
                           v
                    INCIDENT RESPONSE
                           |
                           v
                          RCA
                           |
                           v
                  PLATFORM IMPROVEMENT
```

---

# 72. Chapter Completion Checklist

```text
[ ] Understand production runbooks
[ ] Understand incident lifecycle
[ ] Understand high-latency investigation
[ ] Understand high-5xx investigation
[ ] Understand CrashLoopBackOff investigation
[ ] Understand OOMKilled investigation
[ ] Understand CPU saturation
[ ] Understand memory saturation
[ ] Understand disk pressure
[ ] Understand ImagePullBackOff
[ ] Understand ALB 5xx
[ ] Understand service connectivity failure
[ ] Understand DNS failure
[ ] Understand RDS connection exhaustion
[ ] Understand Redis failure
[ ] Understand SQS backlog
[ ] Understand Kafka consumer lag
[ ] Understand node failure
[ ] Understand AZ failure
[ ] Understand ArgoCD drift
[ ] Understand bad deployment
[ ] Understand security incident handling
[ ] Understand Prometheus failure
[ ] Understand ELK failure
[ ] Understand OpenTelemetry failure
[ ] Understand Jaeger failure
[ ] Understand incident evidence collection
[ ] Understand mitigation vs root cause
[ ] Understand RCA
[ ] Understand corrective actions
[ ] Understand runbook ownership
```

---

# 73. Current Project Position

We have now completed the observability/reliability foundation:

```text
118 → Prometheus Production Implementation
119 → Grafana Production Dashboards
120 → Alertmanager Production Alerts
121 → ELK Production Implementation
122 → Log Correlation
123 → Distributed Tracing
124 → OpenTelemetry + Jaeger Production Implementation
125 → SLO / SLA / SLI Implementation
126 → Production Observability Runbooks
```

The next chapter is:

# **127 — Load Testing**

There we will take the **21-microservice EKS platform** and generate controlled traffic against it.

We will cover:

```text
Load testing architecture
        ↓
k6 / JMeter / Gatling
        ↓
Baseline traffic
        ↓
Low / medium / high traffic
        ↓
RPS
        ↓
P50 / P95 / P99
        ↓
CPU / Memory
        ↓
HPA
        ↓
KEDA
        ↓
Karpenter
        ↓
RDS connections
        ↓
Redis
        ↓
SQS
        ↓
Kafka
        ↓
Prometheus
        ↓
Grafana
        ↓
Jaeger
        ↓
SLO impact
```

The important goal will be **not merely generating requests**, but learning how to determine where the platform breaks as traffic increases and how to prove the bottleneck using metrics, logs, traces, and infrastructure data.
