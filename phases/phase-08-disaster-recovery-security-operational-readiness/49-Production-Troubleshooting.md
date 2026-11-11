# 49 — Production Troubleshooting

> **Enterprise E-Commerce Platform — AWS EKS Production Guide**

Production troubleshooting is not about memorizing `kubectl` commands.

The real skill is:

> **Start from the user-visible symptom, identify the failing layer, collect evidence, isolate the root cause, fix it safely, and verify recovery.**

---

# 1. Production Troubleshooting Philosophy

Never start by randomly restarting Pods.

Use:

```text
Symptom
   ↓
Scope
   ↓
Evidence
   ↓
Layer isolation
   ↓
Root cause
   ↓
Safe remediation
   ↓
Verification
   ↓
Post-incident follow-up
```

Example:

```text
Customer:
"Checkout is failing"

Don't immediately do:
kubectl delete pod

Instead:

Customer
   ↓
CloudFront/WAF
   ↓
ALB
   ↓
Ingress
   ↓
Service
   ↓
Pod
   ↓
Application
   ↓
Payment
   ↓
Database / Queue / External API
```

---

# 2. Our Production Architecture

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
                    Ingress Controller
                           |
                     Kubernetes Service
                           |
                          Pod
                           |
                    Spring Boot Service
                 /          |          \
              Redis        RDS        SQS/SNS
```

Observability:

```text
Application
   |
   +--> Prometheus --> Grafana --> Alertmanager
   |
   +--> stdout/stderr
   |        |
   |     Fluent Bit
   |        |
   |     Logstash
   |        |
   |   Elasticsearch
   |        |
   |      Kibana
   |
   +--> OpenTelemetry
            |
         Collector
            |
          Jaeger
```

---

# 3. First Question — What Is the Scope?

Before investigating, determine:

```text
Is it:
- one request?
- one user?
- one Pod?
- one service?
- multiple services?
- entire EKS cluster?
- one AZ?
- entire AWS region?
```

This dramatically reduces the search space.

---

# 4. Severity Classification

Example:

### P1

```text
Entire checkout unavailable
```

### P2

```text
Payment failures affecting a significant percentage of traffic
```

### P3

```text
One non-critical endpoint degraded
```

### P4

```text
Minor operational issue
```

Actual severity definitions should be established by the organization.

---

# 5. First Five Minutes

When an alert arrives:

```text
1. Confirm the alert
2. Identify affected service
3. Identify affected environment
4. Check whether the issue is widespread
5. Check recent deployments
6. Check dashboards
7. Check logs
8. Check dependencies
9. Start incident communication
```

Don't immediately modify production.

---

# 6. Check Recent Deployment

A huge percentage of application incidents can correlate with a recent deployment.

Check:

```bash
kubectl rollout history deployment/order-service \
  -n production
```

Current status:

```bash
kubectl rollout status deployment/order-service \
  -n production
```

Check Pods:

```bash
kubectl get pods -n production \
  -l app=order-service -o wide
```

---

# 7. Check Deployment

```bash
kubectl get deployment order-service \
  -n production
```

Detailed information:

```bash
kubectl describe deployment order-service \
  -n production
```

Look for:

```text
replicas
available replicas
updated replicas
unavailable replicas
events
```

---

# 8. Check ReplicaSet

Remember:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

Run:

```bash
kubectl get rs -n production \
  -l app=order-service
```

If a new ReplicaSet exists but Pods are unhealthy:

```text
Deployment
    ↓
New ReplicaSet
    ↓
Unhealthy Pods
```

the problem is likely deployment/application/container related.

---

# 9. Check Pods

```bash
kubectl get pods -n production \
  -l app=order-service -o wide
```

Example:

```text
NAME                           READY   STATUS
order-service-7c8d9f-abc       1/1     Running
order-service-7c8d9f-def       0/1     CrashLoopBackOff
order-service-7c8d9f-ghi       1/1     Running
```

One unhealthy Pod doesn't necessarily mean the entire service is down.

---

# 10. Check Pod Events

```bash
kubectl describe pod \
  order-service-7c8d9f-def \
  -n production
```

Look at:

```text
Events:
```

Common clues:

```text
FailedScheduling
FailedMount
Back-off restarting container
Unhealthy
Failed probe
ImagePullBackOff
OOMKilled
```

Events are often one of the fastest ways to identify Kubernetes-level problems.

---

# 11. Check Application Logs

```bash
kubectl logs \
  order-service-7c8d9f-def \
  -n production
```

Previous crashed container:

```bash
kubectl logs \
  order-service-7c8d9f-def \
  -n production \
  --previous
```

This is particularly important for:

```text
CrashLoopBackOff
```

---

# 12. Follow Logs

During an active incident:

```bash
kubectl logs -f \
  deployment/order-service \
  -n production
```

For multiple replicas, use centralized logging rather than depending exclusively on one `kubectl logs` stream.

---

# 13. CrashLoopBackOff

Architecture:

```text
Container starts
     ↓
Application crashes
     ↓
Kubernetes restarts it
     ↓
Application crashes
     ↓
Backoff
```

Check:

```bash
kubectl get pods -n production
```

Then:

```bash
kubectl logs POD_NAME \
  -n production \
  --previous
```

And:

```bash
kubectl describe pod POD_NAME \
  -n production
```

Possible causes:

```text
bad environment variable
missing secret
DB connection failure
application startup exception
bad configuration
dependency unavailable
invalid command
memory problem
```

---

# 14. OOMKilled

Check:

```bash
kubectl describe pod POD_NAME \
  -n production
```

Look for:

```text
Reason: OOMKilled
```

Check:

```bash
kubectl top pod POD_NAME \
  -n production
```

Then compare:

```text
memory request
memory limit
actual usage
JVM heap
```

For Java:

```text
Container memory
    |
    +-- Heap
    +-- Metaspace
    +-- Threads
    +-- Native memory
    +-- JVM overhead
```

Don't simply increase memory limits without determining why memory grew.

---

# 15. ImagePullBackOff

Check:

```bash
kubectl describe pod POD_NAME \
  -n production
```

Typical causes:

```text
wrong image name
wrong tag
image doesn't exist
ECR authorization failure
node cannot reach ECR
registry/network issue
```

Check image:

```bash
kubectl get deployment order-service \
  -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Then verify ECR.

---

# 16. ECR Authentication Problems

EKS nodes need appropriate permissions to pull images.

If image pulling fails:

```text
Pod
 ↓
Kubelet
 ↓
Container runtime
 ↓
ECR
```

Check:

```bash
kubectl describe pod POD_NAME -n production
```

Look for messages such as:

```text
pull access denied
unauthorized
failed to resolve reference
```

Then investigate:

```text
IAM
ECR repository
network
image tag
image architecture
```

---

# 17. Pending Pod

Run:

```bash
kubectl get pods -n production
```

If:

```text
STATUS:
Pending
```

run:

```bash
kubectl describe pod POD_NAME \
  -n production
```

Look for:

```text
Insufficient cpu
Insufficient memory
node selector mismatch
taint
affinity
topology constraint
PVC unavailable
```

---

# 18. Pending Due to CPU/Memory

Example:

```text
0/6 nodes are available:
6 Insufficient cpu
```

Possible solution:

```text
HPA needs more Pods
        ↓
More Pods become Pending
        ↓
Karpenter detects pending capacity
        ↓
New node provisioned
```

If Karpenter doesn't provision:

```text
NodePool constraints
IAM
subnets
security groups
instance availability
capacity type
NodeClass
```

must be investigated.

---

# 19. Service Has No Endpoints

Check:

```bash
kubectl get service order-service \
  -n production
```

Then:

```bash
kubectl get endpoints order-service \
  -n production
```

Or EndpointSlices:

```bash
kubectl get endpointslice \
  -n production \
  -l kubernetes.io/service-name=order-service
```

If there are no endpoints:

```text
Service
   |
   X
No ready Pods
```

Check:

```bash
kubectl get pods \
  -n production \
  --show-labels
```

Compare Service selector:

```bash
kubectl describe service order-service \
  -n production
```

---

# 20. Selector Mismatch

Service:

```yaml
selector:
  app: order-service
```

Pod:

```yaml
labels:
  app: orders
```

Result:

```text
Service selector
      |
      X
Pod labels don't match
```

Therefore:

```text
Service
   ↓
No endpoints
   ↓
Traffic fails
```

This is a classic Kubernetes troubleshooting issue.

---

# 21. Readiness Probe Failure

Check:

```bash
kubectl describe pod POD_NAME \
  -n production
```

Example:

```text
Readiness probe failed:
HTTP probe failed with statuscode: 503
```

Important distinction:

```text
Readiness failure
    ↓
Pod removed from Service endpoints
```

The container may still be running.

This is often intentional.

---

# 22. Liveness Probe Failure

Liveness failure can cause:

```text
Kubernetes
   ↓
restart container
```

If incorrectly configured:

```text
Application healthy
      ↓
Bad liveness probe
      ↓
Kubernetes restarts it
      ↓
Availability decreases
```

Therefore probes must reflect actual application behavior.

---

# 23. Startup Probe

Java applications may take time to start.

Without a startup probe:

```text
Spring Boot
   ↓
slow startup
   ↓
liveness probe starts too early
   ↓
restart
```

Startup probe allows the application to initialize before liveness checking becomes active.

---

# 24. Ingress Troubleshooting

Architecture:

```text
Internet
   ↓
ALB
   ↓
Ingress
   ↓
Service
   ↓
Pod
```

Check:

```bash
kubectl get ingress \
  -n production
```

Detailed:

```bash
kubectl describe ingress \
  order-service \
  -n production
```

Check:

```text
host
path
backend service
annotations
ALB address
TLS configuration
```

---

# 25. AWS Load Balancer Controller

Check controller:

```bash
kubectl get pods \
  -n ingress-system
```

Logs:

```bash
kubectl logs \
  -n ingress-system \
  deployment/aws-load-balancer-controller
```

Problems can involve:

```text
IAM
subnet discovery
security groups
Ingress configuration
TargetGroupBinding
AWS API permissions
```

---

# 26. ALB Target Health

If ALB exists but requests fail:

```text
ALB
 ↓
Target Group
 ↓
EKS targets
```

Check target health from AWS.

Typical problems:

```text
health check path wrong
health check port wrong
security group
readiness state
Service targetPort
application not listening
```

For example:

```text
ALB health check:
GET /health
```

but application only exposes:

```text
/actuator/health
```

The target may be marked unhealthy.

---

# 27. Network Troubleshooting

First determine:

```text
Who cannot reach whom?
```

Examples:

```text
Pod → RDS
Pod → Redis
Pod → SQS
Pod → another Pod
ALB → Pod
Node → ECR
```

Then inspect:

```text
Security Group
NetworkPolicy
Route table
Subnet
DNS
NAT
VPC endpoint
port
TLS
```

---

# 28. Pod → RDS Connection Failure

Symptoms:

```text
Connection timeout
Connection refused
authentication failure
```

Check application logs first.

Then:

```text
Pod
 ↓
DNS
 ↓
RDS endpoint
 ↓
Route
 ↓
Security Group
 ↓
RDS
```

Test DNS from an approved troubleshooting environment:

```bash
nslookup <rds-endpoint>
```

or:

```bash
dig <rds-endpoint>
```

Don't install debugging packages permanently into production application containers.

---

# 29. RDS Security Group

Expected model:

```text
EKS workload security group
          |
          | TCP 5432
          v
RDS security group
```

Avoid:

```text
0.0.0.0/0
```

for database access.

If connection fails, verify:

```text
source SG
destination SG
port 5432
route
NACL
DNS
```

---

# 30. Too Many Database Connections

A common scaling problem:

```text
3 Pods
×
Hikari pool 20
=
60 possible connections
```

Then:

```text
10 Pods
×
20
=
200 connections
```

If RDS supports fewer usable connections:

```text
Application scaling
       ↓
Connection exhaustion
       ↓
Requests fail
```

Therefore HPA must consider downstream database capacity.

---

# 31. Slow Database

Application:

```text
Latency ↑
```

Check:

```text
RDS CPU
DatabaseConnections
ReadIOPS
WriteIOPS
latency
locks
slow queries
```

Application logs may show:

```text
query timeout
connection timeout
```

Use database-level diagnostics rather than blindly scaling Pods.

More Pods can actually make the DB problem worse.

---

# 32. Redis Troubleshooting

Symptoms:

```text
cache timeout
connection refused
high latency
cache miss increase
```

Check:

```text
Redis CPU
Memory
Connections
Evictions
Hit ratio
Network
```

Application path:

```text
Application
   ↓
Redis
   ↓
Cache hit?
```

If Redis is unavailable, application behavior should be designed intentionally.

For cache-only data:

```text
Redis failure
   ↓
fallback to RDS
```

may be possible.

For session/coordination use cases, the impact can be different.

---

# 33. SQS Troubleshooting

If processing is delayed:

Check:

```text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
AgeOfOldestMessage
consumer count
consumer errors
```

Architecture:

```text
Producer
   ↓
SQS
   ↓
Consumer Pods
```

If queue depth rises:

```text
Incoming rate > processing rate
```

Potential solutions:

```text
increase consumers
increase concurrency
optimize processing
check downstream dependency
use KEDA
```

---

# 34. SQS Poison Message

One message repeatedly fails:

```text
Message
 ↓
Consumer
 ↓
Failure
 ↓
Visibility timeout
 ↓
Retry
 ↓
Failure
```

Eventually:

```text
DLQ
```

Check:

```text
DLQ message count
receive count
consumer logs
application error
message payload
```

Don't repeatedly delete/requeue the same poison message without understanding the failure.

---

# 35. Kafka Troubleshooting

If Kafka is used:

Check:

```text
consumer lag
partition health
ISR
broker health
producer errors
consumer errors
rebalancing
```

Architecture:

```text
Producer
   ↓
Kafka Topic
   ↓
Partitions
   ↓
Consumer Group
   ↓
Pods
```

If consumer lag grows:

```text
producer rate
>
consumer processing rate
```

Investigate consumer scaling and partition parallelism.

---

# 36. RabbitMQ Troubleshooting

Check:

```text
queue depth
consumer count
unacknowledged messages
publisher confirms
connection count
channel count
node health
disk
memory
```

Important metric:

```text
Unacked messages
```

If consumers receive messages but don't ACK:

```text
Unacked ↑
```

Potential causes:

```text
slow processing
consumer stuck
downstream dependency
application exception
prefetch configuration
```

---

# 37. Kubernetes Node Problem

Check:

```bash
kubectl get nodes
```

Example:

```text
NAME       STATUS
node-a     Ready
node-b     NotReady
node-c     Ready
```

Detailed:

```bash
kubectl describe node node-b
```

Look for:

```text
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
conditions
events
```

---

# 38. Node Disk Pressure

Check:

```bash
kubectl describe node NODE_NAME
```

Possible:

```text
DiskPressure=True
```

Investigate:

```text
container logs
image layers
ephemeral storage
application temp files
```

Useful:

```bash
kubectl get pods -A -o wide
```

Then inspect node/container runtime metrics using the approved node-level operational tooling.

Don't blindly delete files from `/var/lib/containerd` or Kubernetes-managed directories.

---

# 39. Node Memory Pressure

Architecture:

```text
Node memory
 |
 +-- kubelet
 +-- system
 +-- DaemonSets
 +-- application Pods
```

If pressure rises:

```text
Evictions
```

can occur.

Check:

```bash
kubectl describe node NODE_NAME
```

and:

```bash
kubectl top node NODE_NAME
```

Then review:

```text
Pod requests
Pod limits
DaemonSet overhead
node size
memory leaks
```

---

# 40. Node NotReady

Check:

```bash
kubectl describe node NODE_NAME
```

Potential causes:

```text
kubelet
container runtime
network
EC2 problem
disk
memory
AWS infrastructure
EKS issue
```

For managed node groups, also inspect the underlying EC2 instance and AWS/EKS events.

---

# 41. VPC CNI IP Exhaustion

A particularly important EKS issue.

Symptoms:

```text
Pods stuck Pending
```

Events may indicate:

```text
failed to assign an IP address
```

Architecture:

```text
EKS Node
   |
VPC CNI
   |
ENI
   |
Private subnet IPs
```

Check:

```text
subnet free IPs
ENI capacity
pod density
instance type
VPC CNI configuration
```

Increasing EC2 CPU won't necessarily fix an IP exhaustion problem.

---

# 42. DNS Troubleshooting

Kubernetes:

```text
Pod
 ↓
CoreDNS
 ↓
DNS resolution
```

Check:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Check CoreDNS logs:

```bash
kubectl logs \
  -n kube-system \
  -l k8s-app=kube-dns
```

Test from a temporary approved troubleshooting Pod:

```bash
nslookup kubernetes.default
```

Then test application dependencies.

---

# 43. NetworkPolicy Problem

Suppose:

```text
order-service
      |
      X
      |
RDS
```

and NetworkPolicy was recently changed.

Check:

```bash
kubectl get networkpolicy \
  -n production
```

Then:

```text
source namespace
source Pod labels
destination
port
egress policy
ingress policy
```

A default-deny policy is good security, but every required communication path must be explicitly allowed.

---

# 44. Prometheus Troubleshooting

If a dashboard says:

```text
No data
```

don't immediately blame Grafana.

Flow:

```text
Application
   ↓
/actuator/prometheus
   ↓
ServiceMonitor
   ↓
Prometheus
   ↓
Grafana
```

Check application endpoint:

```bash
curl http://SERVICE:8080/actuator/prometheus
```

Then check:

```text
ServiceMonitor
labels
target
Prometheus configuration
```

---

# 45. Grafana Troubleshooting

If Prometheus contains data but Grafana doesn't:

```text
Prometheus
    ↓
Data source
    ↓
Grafana
```

Check:

```text
datasource URL
authentication
TLS
time range
PromQL
dashboard variables
```

Test the query directly in Grafana Explore.

---

# 46. ELK Troubleshooting

Full flow:

```text
Application
 ↓
stdout
 ↓
container log
 ↓
Fluent Bit
 ↓
Logstash
 ↓
Elasticsearch
 ↓
Kibana
```

If logs disappear, identify the first broken link.

---

# 47. Fluent Bit Problem

Check:

```bash
kubectl get pods -n logging
```

Then:

```bash
kubectl logs \
  -n logging \
  daemonset/fluent-bit
```

Potential problems:

```text
file path
permissions
position DB
Kubernetes metadata
output connection
buffer overflow
TLS
Logstash unavailable
```

---

# 48. Logstash Problem

Check:

```bash
kubectl get pods -n logging
```

Then:

```bash
kubectl logs \
  -n logging \
  deployment/logstash
```

Investigate:

```text
pipeline errors
JVM memory
persistent queue
Elasticsearch connection
bulk failures
backpressure
```

---

# 49. Elasticsearch Problem

Check cluster health.

Typical states:

```text
green
yellow
red
```

Meaning broadly:

```text
Green:
healthy

Yellow:
primary shards available but replica issue

Red:
one or more primary shards unavailable
```

Investigate:

```text
disk
nodes
shards
heap
allocation
storage
network
```

Don't immediately delete indexes to recover from a red cluster.

---

# 50. Kibana Problem

If:

```text
Elasticsearch healthy
```

but:

```text
Kibana shows no logs
```

check:

```text
Kibana Pod
Data View
index/data stream
time range
permissions
Elasticsearch connectivity
```

---

# 51. OpenTelemetry Troubleshooting

Trace flow:

```text
Application
 ↓
OTel instrumentation
 ↓
OTel Collector
 ↓
Jaeger
```

Check:

```text
OTLP endpoint
4317 gRPC
4318 HTTP
```

Then:

```text
Collector logs
Collector metrics
Jaeger ingestion
```

If traces arrive at Collector but not Jaeger:

```text
Collector
   ↓
X
Jaeger/exporter/storage
```

Focus on that boundary.

---

# 52. Jaeger Troubleshooting

If Kibana has:

```text
trace.id
```

but Jaeger has no trace:

```text
Application
   ↓
OTel Collector
   ↓
Jaeger
```

Check each stage.

Don't assume:

```text
trace ID exists in log
```

means Jaeger storage is healthy.

---

# 53. ArgoCD Troubleshooting

If deployment is not reaching EKS:

```text
Git
 ↓
GitOps repo
 ↓
ArgoCD
 ↓
Kubernetes
```

Check:

```bash
kubectl get applications \
  -n argocd
```

Then inspect the application in ArgoCD.

Look for:

```text
Sync status
Health status
Repository error
manifest generation error
permission error
resource conflict
```

---

# 54. GitOps Drift

Example:

```text
Git:
replicas = 3

Cluster:
replicas = 5
```

ArgoCD may report:

```text
OutOfSync
```

If self-heal is enabled:

```text
manual cluster change
       ↓
ArgoCD detects drift
       ↓
desired Git state restored
```

This is why production changes should normally go through GitOps.

---

# 55. Terraform Problem

If infrastructure is failing:

```text
terraform plan
terraform apply
```

should not be your first incident response action.

First identify:

```text
Is this actually infrastructure?
```

For example:

```text
Application Pod failing
```

does not mean:

```text
terraform apply
```

Terraform manages infrastructure.

ArgoCD manages Kubernetes desired state.

Application CI builds artifacts.

Keep those boundaries clear.

---

# 56. Recent Deployment Correlation

A very useful investigation:

```text
Incident:
10:15

Deployment:
10:08

Error rate:
10:09 ↑
```

This is strong evidence of correlation.

Then compare:

```text
previous image
new image
configuration
secret version
database migration
dependency behavior
```

Do not declare root cause solely because a deployment occurred.

---

# 57. Safe Rollback

If deployment is confirmed as the likely cause and rollback is approved:

```bash
kubectl rollout undo \
  deployment/order-service \
  -n production
```

However, in our GitOps model, the preferred production workflow is:

```text
GitOps commit
      ↓
revert bad change
      ↓
PR/review according to policy
      ↓
ArgoCD
      ↓
cluster
```

This preserves auditability.

---

# 58. Database Migration Incident

Suppose:

```text
Version 2 application
      |
New DB column
```

Then rollback application to:

```text
Version 1
```

If the database migration removed something required by v1:

```text
Rollback
   ↓
Application incompatible with DB
```

Therefore production migrations should use:

```text
Expand
   ↓
Deploy compatible application
   ↓
Migrate data
   ↓
Contract later
```

This is the **expand-and-contract** approach.

---

# 59. High CPU

Symptoms:

```text
CPU ↑
Latency ↑
```

Check:

```bash
kubectl top pods -n production
```

Then:

```text
Which service?
Which Pod?
Which endpoint?
Which dependency?
```

Potential causes:

```text
traffic spike
CPU-intensive code
infinite loop
GC pressure
bad query
retry storm
logging storm
```

Don't automatically scale Pods.

Scaling can amplify downstream load.

---

# 60. High Memory

Investigate:

```text
Pod memory
JVM heap
GC
traffic
request size
cache
memory leak
```

For Java:

```text
heap ↑ continuously
      ↓
possible memory leak
```

Whereas:

```text
heap ↑
      ↓
GC
      ↓
heap ↓
```

may represent normal workload behavior.

---

# 61. High Latency

Use the RED model:

```text
Rate
Errors
Duration
```

Check:

```text
request rate
error rate
p50
p95
p99
```

Then trace:

```text
ALB
 ↓
service
 ↓
downstream service
 ↓
database
```

Distributed tracing is extremely useful here.

---

# 62. Example — Checkout Latency

Suppose:

```text
POST /checkout
p95 = 8 seconds
```

Jaeger trace:

```text
checkout
 |
 +-- order-service       500ms
 |
 +-- inventory-service   200ms
 |
 +-- payment-service     6.8s
                           |
                           +-- external payment API
```

The Kubernetes Pods may all be healthy.

The actual bottleneck is downstream.

This is why:

> **Healthy Pods do not necessarily mean a healthy application.**

---

# 63. Error Rate Spike

Prometheus:

```text
5xx rate ↑
```

Investigate:

```text
Which service?
Which endpoint?
Which status?
Which deployment?
Which dependency?
```

Kibana:

```text
service.name
http.route
status
error.message
trace.id
```

Jaeger:

```text
find failing trace
```

This gives a powerful:

```text
Metrics → Logs → Traces
```

workflow.

---

# 64. Production Troubleshooting Decision Tree

```text
          User reports failure
                  |
                  v
          Is it widespread?
             /        \
           YES        NO
            |          |
        Platform?    Service?
            |          |
       Infrastructure  Pod?
            |          |
       Network/AWS    App?
            |          |
          DB/Redis   Dependency?
```

Then:

```text
Metrics
   +
Logs
   +
Traces
   +
Kubernetes events
   +
AWS metrics/events
```

---

# 65. The Golden Troubleshooting Commands

### Cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

### Namespace

```bash
kubectl get all -n production
kubectl get events -n production --sort-by=.lastTimestamp
```

### Deployment

```bash
kubectl get deployment -n production
kubectl describe deployment order-service -n production
kubectl rollout status deployment/order-service -n production
```

### Pods

```bash
kubectl get pods -n production -o wide
kubectl describe pod POD -n production
kubectl logs POD -n production
kubectl logs POD -n production --previous
```

### Services

```bash
kubectl get svc -n production
kubectl get endpoints -n production
kubectl get endpointslice -n production
```

### Resources

```bash
kubectl top nodes
kubectl top pods -n production
```

### Network

```bash
kubectl get ingress -n production
kubectl get networkpolicy -n production
```

---

# 66. Don't Run Dangerous Commands During Panic

Avoid blindly doing:

```bash
kubectl delete pods --all
```

or:

```bash
kubectl delete deployment
```

or:

```bash
kubectl delete pvc
```

or:

```bash
terraform destroy
```

or:

```bash
kubectl delete namespace production
```

Production troubleshooting must be evidence-driven.

---

# 67. Production Debugging Pod

Sometimes application containers intentionally contain very few tools.

Instead of modifying the application image, use a temporary approved troubleshooting container/Pod with the required network utilities.

Conceptually:

```text
Production App
      |
      | same namespace/network context
      |
Debug Pod
      |
      +-- DNS test
      +-- TCP test
      +-- HTTP test
      +-- TLS test
```

Remove the temporary troubleshooting resource after use.

---

# 68. Incident Evidence

During troubleshooting record:

```text
Incident start
Affected services
Affected endpoints
Error rate
Latency
Recent deployments
Pod status
Node status
Database status
Queue depth
Relevant logs
Relevant trace IDs
Actions performed
Recovery time
```

This makes the incident reproducible.

---

# 69. Root Cause vs Symptom

Example:

```text
Symptom:
order-service Pods restarted

Immediate cause:
OOMKilled

Deeper cause:
memory usage increased

Root cause:
new version introduced unbounded response caching

Contributing factor:
memory limit was not tested under peak traffic
```

Good incident analysis goes deeper than:

> "Pod restarted."

---

# 70. Production Troubleshooting Example

### Incident

```text
Customers report checkout failures.
```

### Step 1

Check Grafana:

```text
checkout 5xx ↑
```

### Step 2

Check ArgoCD:

```text
payment-service recently deployed
```

### Step 3

Check Pods:

```bash
kubectl get pods -n production \
  -l app=payment-service
```

### Step 4

Check logs:

```bash
kubectl logs \
  deployment/payment-service \
  -n production
```

### Step 5

Logs:

```text
connection timeout to payment provider
```

### Step 6

Jaeger:

```text
payment-provider span = 6.5 seconds
```

### Step 7

Conclusion:

```text
Payment provider latency
      ↓
Payment service timeout
      ↓
Checkout failure
```

Notice:

```text
EKS itself may be healthy.
```

---

# 71. Another Example — Service Down

```text
Customer
   ↓
ALB
   ↓
503
```

Investigate:

```text
ALB target health
   ↓
Ingress
   ↓
Service
   ↓
Endpoints
   ↓
Pods
```

Suppose:

```text
Service:
no endpoints
```

Then:

```text
Pods:
0/3 Ready
```

Then:

```text
Readiness probe failed
```

Then:

```text
application can't connect to RDS
```

Root cause:

```text
RDS connectivity
```

Not:

```text
ALB
```

---

# 72. Another Example — Queue Backlog

```text
SQS queue depth ↑
```

Check:

```text
Consumer replicas
CPU
processing duration
application errors
RDS latency
external API latency
```

Suppose:

```text
Consumer Pods:
10

CPU:
30%

Queue:
still increasing
```

Don't assume CPU is the bottleneck.

Trace:

```text
Consumer
   ↓
Payment API
   ↓
slow
```

The consumers are waiting on I/O.

---

# 73. Production Troubleshooting Matrix

| Symptom                 | First checks                       |
| ----------------------- | ---------------------------------- |
| 5xx                     | deployment, logs, traces           |
| 503 from ALB            | target health, Service, endpoints  |
| Pod Pending             | events, requests, scheduling       |
| CrashLoopBackOff        | previous logs, events              |
| OOMKilled               | memory, JVM, limits                |
| ImagePullBackOff        | image/tag/ECR/IAM                  |
| High latency            | RED + traces                       |
| High CPU                | top + application behavior         |
| High memory             | JVM/heap/cache                     |
| DB timeout              | DNS/network/SG/RDS                 |
| Too many DB connections | Hikari/pod count/RDS               |
| Redis timeout           | Redis health/network               |
| SQS backlog             | queue age/consumer throughput      |
| Kafka lag               | consumer throughput/partitions     |
| RabbitMQ backlog        | consumers/unacked/queue            |
| No metrics              | endpoint/ServiceMonitor/Prometheus |
| No logs                 | Fluent Bit → Logstash → ES         |
| No traces               | OTel → Collector → Jaeger          |
| ArgoCD OutOfSync        | Git vs cluster                     |
| Node NotReady           | node conditions/kubelet/AWS        |
| Pod IP exhaustion       | VPC CNI/subnet/IPs                 |

---

# 74. Production Troubleshooting Checklist

```text
[ ] Confirm the alert
[ ] Identify affected service
[ ] Identify environment
[ ] Determine blast radius
[ ] Check recent deployment
[ ] Check Grafana
[ ] Check Prometheus
[ ] Check application logs
[ ] Check Kubernetes events
[ ] Check Pod health
[ ] Check Service endpoints
[ ] Check Ingress/ALB
[ ] Check RDS
[ ] Check Redis
[ ] Check SQS/Kafka/RabbitMQ
[ ] Check network policies
[ ] Check security groups
[ ] Check EKS nodes
[ ] Check VPC CNI
[ ] Check DNS
[ ] Check traces
[ ] Record evidence
[ ] Apply controlled remediation
[ ] Verify recovery
[ ] Document root cause
[ ] Create follow-up actions
```

---

# 75. Final Production Troubleshooting Mental Model

When something breaks, think **left to right**:

```text
USER
  |
  v
Route 53
  |
  v
CloudFront
  |
  v
WAF
  |
  v
ALB
  |
  v
Ingress
  |
  v
Service
  |
  v
Pod
  |
  v
Application
  |
  +--------+--------+--------+
  |        |        |        |
  v        v        v        v
 RDS     Redis     SQS     External API
```

And use the observability signals:

```text
                 INCIDENT
                    |
          +---------+---------+
          |         |         |
       Metrics     Logs     Traces
          |         |         |
      Prometheus   ELK     Jaeger
          \         |        /
           \        |       /
            +-------+------+
                    |
               Root Cause
```

The most important production skill is not knowing hundreds of commands.

It is knowing **which layer to investigate next and why**.

---

**Next: `50-Incident-Response.md`** — we'll move from *technical troubleshooting* to the full production incident process: **alert → incident declaration → roles → communication → mitigation → rollback/failover → recovery → RCA → postmortem → preventive actions**.
