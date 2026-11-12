# 103 — HPA / VPA / KEDA Production Implementation

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Region:** `ap-south-1`
> **Scope:** Horizontal Pod Autoscaler, Vertical Pod Autoscaler, and KEDA for production workloads

---

## 1. Purpose

In the previous chapter we established:

```text
Application
    ↓
Resource requests/limits
    ↓
Kubernetes Scheduler
    ↓
Node capacity
    ↓
Karpenter
```

Now we add **workload autoscaling**.

Our production architecture will use three different scaling mechanisms:

```text
HPA  → scale Pod replicas horizontally
VPA  → recommend/tune Pod resources vertically
KEDA → scale workloads from external/event metrics
```

They solve different problems.

---

# 2. The Three Autoscalers

| Technology | Scales              | Typical trigger                        |
| ---------- | ------------------- | -------------------------------------- |
| HPA        | Pod replicas        | CPU, memory, custom metrics            |
| VPA        | CPU/memory requests | observed workload usage                |
| KEDA       | Pod replicas        | SQS, Kafka, RabbitMQ, Prometheus, etc. |

Conceptually:

```text
                Autoscaling
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
      HPA          VPA          KEDA
       │            │            │
       ▼            ▼            ▼
   Replicas      Resources     Replicas
```

---

# 3. Why We Need All Three

Consider `order-service`.

During normal traffic:

```text
3 replicas
```

During high HTTP traffic:

```text
8 replicas
```

HPA handles this.

But suppose each replica gradually requires more memory because request payloads and workloads have changed.

VPA can identify:

```text
Current request: 512Mi
Observed requirement: 900Mi
```

and recommend a higher value.

Now consider `notification-service`.

It may not be CPU-heavy.

Instead:

```text
SQS backlog = 20,000 messages
```

KEDA can scale consumers based on the queue.

Therefore:

```text
HTTP workload       → HPA
Resource tuning     → VPA
Event-driven worker → KEDA
```

---

# 4. HPA Architecture

Our HPA flow is:

```text
User traffic
     ↓
ALB
     ↓
Service
     ↓
Pods
     ↓
CPU / Memory / Custom Metrics
     ↓
Metrics API
     ↓
HPA Controller
     ↓
Replica count
```

If additional Pods cannot fit:

```text
HPA
 ↓
Pending Pods
 ↓
Karpenter
 ↓
EC2
 ↓
Scheduler
```

This is the architecture we established in Chapter 102.

---

# 5. HPA Prerequisite — Resource Requests

For CPU utilization-based HPA, define CPU requests.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

If the container uses:

```text
500m CPU
```

and request is:

```text
500m
```

utilization is approximately:

```text
100%
```

If request is:

```text
1 CPU
```

the same 500m usage is approximately:

```text
50%
```

Therefore incorrect requests can produce unexpected HPA behavior.

---

# 6. HPA API

Modern Kubernetes clusters generally use:

```yaml
apiVersion: autoscaling/v2
```

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: ecommerce-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  minReplicas: 3
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

# 7. What This HPA Means

```text
Minimum:
3 Pods

Maximum:
10 Pods

Target:
70% average CPU utilization
```

Therefore:

```text
Low traffic
   ↓
3 Pods

Increasing traffic
   ↓
4
5
6
...

Very high traffic
   ↓
up to 10 Pods
```

---

# 8. Add Memory

We can also define:

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
```

HPA evaluates the configured metrics and calculates the desired replica count based on the metrics.

Don't blindly add many metrics. Each metric should represent a meaningful scaling signal.

---

# 9. Helm HPA

Our Helm chart should not hard-code production values.

`values.yaml`:

```yaml
autoscaling:
  enabled: true

  minReplicas: 3
  maxReplicas: 10

  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 75
```

---

# 10. Helm HPA Template

Example:

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "order-service.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "order-service.fullname" . }}

  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
{{- end }}
```

---

# 11. Production Values

For `order-service`:

```yaml
autoscaling:
  enabled: true

  minReplicas: 3
  maxReplicas: 15

  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 75
```

These numbers are examples.

The final values will come from:

```text
Load testing
+
production traffic
+
Prometheus
+
latency requirements
+
downstream capacity
```

---

# 12. Why We Don't Simply Set maxReplicas = 100

Suppose:

```yaml
maxReplicas: 100
```

The application may scale rapidly.

But downstream systems might not.

Example:

```text
Order Service
     ↓
Payment Service
     ↓
RDS
```

If Order Service scales:

```text
3 → 100 Pods
```

we may suddenly create:

```text
100 × database connections
```

or much more depending on connection pool configuration.

This can overload RDS.

Autoscaling must therefore consider **dependency capacity**.

---

# 13. HPA and RDS Connection Pooling

Suppose:

```text
HPA max replicas = 20
```

Each Pod has:

```text
maximum DB connections = 10
```

Potential connection demand:

```text
20 × 10 = 200
```

Now suppose RDS safely supports:

```text
150 application connections
```

Then unrestricted HPA can create a database bottleneck.

This is why our previous database design included:

```text
Pods × connection pool size
```

as a capacity-planning consideration.

---

# 14. HPA Behavior Configuration

Modern HPA supports behavior configuration.

Example:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60

  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
      - type: Percent
        value: 25
        periodSeconds: 60
```

Conceptually:

```text
Scale up:
fast

Scale down:
slow
```

This helps prevent unnecessary oscillation.

Exact behavior should be tuned from observed traffic.

---

# 15. Why Scale Down Should Often Be Slower

Imagine:

```text
Traffic:
1000 requests/sec
```

then briefly drops:

```text
1000 → 300
```

If we immediately scale down:

```text
10 → 3 Pods
```

and traffic returns:

```text
300 → 1000
```

we may immediately need to scale up again.

This creates:

```text
scale down
   ↓
scale up
   ↓
scale down
   ↓
scale up
```

A stabilization period can reduce this oscillation.

---

# 16. Check HPA

Run:

```bash
kubectl get hpa -n ecommerce-prod
```

Example:

```text
NAME            REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS
order-service   Deployment/order-service   45%/70%   3         15        5
```

---

# 17. Detailed HPA Troubleshooting

Run:

```bash
kubectl describe hpa order-service \
  -n ecommerce-prod
```

Check:

```text
Metrics
Min replicas
Max replicas
Current replicas
Desired replicas
Conditions
Events
```

---

# 18. HPA Shows `<unknown>`

Example:

```text
TARGETS
<unknown>/70%
```

Investigate:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod
```

Check resource requests.

Then:

```bash
kubectl get apiservice
```

and verify the resource metrics pipeline.

For basic CPU/memory metrics, the cluster needs a functioning Metrics API, commonly provided by Metrics Server.

---

# 19. Metrics Server

Metrics Server provides resource usage data for Kubernetes autoscaling and commands such as:

```bash
kubectl top pods
kubectl top nodes
```

Architecture:

```text
Kubelet
   ↓
Metrics Server
   ↓
Metrics API
   ↓
HPA
```

Metrics Server is **not** a replacement for Prometheus.

---

# 20. Metrics Server vs Prometheus

| Component      | Purpose                         |
| -------------- | ------------------------------- |
| Metrics Server | current resource metrics        |
| Prometheus     | detailed time-series monitoring |
| Grafana        | visualization                   |
| Alertmanager   | alert routing                   |

Think:

```text
Metrics Server
    ↓
HPA / kubectl top

Prometheus
    ↓
Monitoring / alerts / SLOs / analysis
```

---

# 21. HPA with Custom Metrics

CPU isn't always the best scaling signal.

For example:

```text
HTTP requests per second
```

may be more meaningful for an API.

Or:

```text
active requests
```

Or:

```text
queue depth
```

Kubernetes HPA can use other metric sources through supported metrics APIs.

For production, we should only use a custom metric after verifying:

```text
metric quality
metric availability
metric stability
cardinality
scaling behavior
```

---

# 22. Example — Requests Per Second

Suppose:

```text
1 Pod handles:
100 requests/sec
```

We want:

```text
approximately 100 requests/sec per Pod
```

Traffic:

```text
500 requests/sec
```

Desired replicas:

```text
5
```

This may represent application capacity more directly than CPU.

However, we must establish this relationship through load testing rather than assume it.

---

# 23. VPA

Now consider the second autoscaling mechanism:

```text
Vertical Pod Autoscaler
```

VPA adjusts or recommends:

```text
CPU requests
Memory requests
```

instead of changing replica count.

Conceptually:

```text
Current:
CPU request = 250m
Memory = 512Mi

Observed usage:
CPU = 400m
Memory = 800Mi

VPA recommendation:
CPU ≈ 500m
Memory ≈ 1Gi
```

---

# 24. VPA Modes

VPA supports different operating modes.

Conceptually:

```text
Off
Initial
Recreate
Auto
```

### Off

Only recommendations.

```text
VPA → recommendation
```

No automatic resource changes.

### Initial

Applies recommendations when Pods are initially created.

### Recreate

May evict/recreate Pods to apply new recommendations.

### Auto

Can automatically apply recommendations, subject to VPA behavior/version.

For production, start with **recommendation mode** and validate behavior before enabling automatic changes.

---

# 25. VPA Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service
  namespace: ecommerce-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  updatePolicy:
    updateMode: "Off"

  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        controlledResources:
          - cpu
          - memory
```

The important point:

```text
updateMode: Off
```

means:

> Give recommendations; don't automatically modify running Pods.

---

# 26. Why VPA Is Useful

Suppose developers initially configure:

```text
CPU request = 250m
Memory = 512Mi
```

After several days of realistic traffic, VPA may show:

```text
CPU recommendation = 450m
Memory recommendation = 900Mi
```

We can review this against:

```text
Prometheus
application metrics
load testing
latency
GC behavior
OOM events
```

Then update Helm values through GitOps.

---

# 27. VPA and HPA Caution

Using VPA and HPA on the **same resource metrics** can create competing decisions.

Example:

```text
HPA:
CPU utilization high
→ add Pods
```

while:

```text
VPA:
CPU request too low
→ increase CPU request
```

Changing the request changes the utilization calculation.

Therefore:

> Do not casually combine HPA and VPA on the same CPU/memory scaling signal.

A practical production approach is often:

```text
HPA → replica scaling
VPA → recommendations/resource tuning
```

and then explicitly validate any automatic VPA behavior.

---

# 28. KEDA

Now we move to event-driven workloads.

KEDA stands for:

```text
Kubernetes Event-driven Autoscaling
```

KEDA is particularly useful for our:

```text
SQS
Kafka
RabbitMQ
```

workloads.

Architecture:

```text
External event source
       ↓
     KEDA
       ↓
    HPA mechanism
       ↓
    Deployment
       ↓
      Pods
```

KEDA can create/manage the scaling mechanism based on the configured trigger.

---

# 29. Why KEDA Instead of CPU HPA?

Consider:

```text
notification-service
```

Suppose CPU usage is:

```text
20%
```

but:

```text
SQS queue:
50,000 messages
```

CPU doesn't necessarily tell us how much work remains.

Queue backlog does.

Therefore:

```text
CPU HPA
    ↓
may remain at 3 Pods

KEDA SQS
    ↓
sees backlog
    ↓
scales consumers
```

---

# 30. KEDA + SQS Architecture

Our architecture:

```text
Order Service
     │
     │ event
     ▼
   Amazon SQS
     │
     │ backlog
     ▼
    KEDA
     │
     ▼
notification-service
     │
     ▼
   Pods
```

If backlog increases:

```text
100
 ↓
1,000
 ↓
10,000
 ↓
50,000
```

KEDA can increase consumer replicas according to configured scaling behavior.

---

# 31. KEDA ScaledObject

Example structure:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-service
  namespace: ecommerce-prod
spec:
  scaleTargetRef:
    name: notification-service

  minReplicaCount: 2
  maxReplicaCount: 20

  pollingInterval: 15
  cooldownPeriod: 300

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123456789012/order-events
        queueLength: "50"
        awsRegion: ap-south-1
```

Exact KEDA scaler configuration depends on the installed KEDA version and authentication method.

Treat this as a template and validate against the installed version before production rollout.

---

# 32. What `queueLength` Represents

Conceptually:

```text
queueLength = 50
```

can be used as a scaling target.

For example:

```text
50 messages / consumer
```

is the intended scaling relationship.

Suppose:

```text
Queue = 500 messages
```

then roughly:

```text
500 / 50 = 10
```

consumer replicas may be targeted, subject to KEDA/HPA behavior and configured min/max limits.

This is not a promise of exact instantaneous replica count.

---

# 33. KEDA Authentication

Do not put AWS access keys into:

```text
Kubernetes Secret
Helm values
Git
Docker image
```

For EKS, use AWS-native workload identity mechanisms where supported.

Our architecture already uses:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
SQS
```

Therefore KEDA authentication should follow the supported workload identity/authentication mechanism for the installed KEDA version.

---

# 34. IAM for SQS Consumer

A consumer may need:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

The exact policy should be restricted to the required queue ARN.

Example conceptual policy:

```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage",
    "sqs:GetQueueAttributes"
  ],
  "Resource": "arn:aws:sqs:ap-south-1:123456789012:order-events"
}
```

Use the least privilege required by the actual application.

---

# 35. KEDA and Zero Replicas

Some event-driven workloads can scale down very aggressively.

For example:

```yaml
minReplicaCount: 0
```

This can be useful for:

```text
batch processors
background workers
non-latency-sensitive consumers
```

But don't automatically use zero for latency-sensitive services.

If an API needs immediate response:

```text
minReplicaCount: 2
```

may be more appropriate.

---

# 36. KEDA for Kafka

Our Kafka architecture can use:

```text
Kafka topic
    ↓
Consumer group
    ↓
Consumer lag
    ↓
KEDA
    ↓
Consumer replicas
```

Example:

```text
Consumer lag:
10,000 messages
```

KEDA can scale based on configured Kafka lag thresholds.

Important considerations:

```text
partition count
consumer group
max replicas
ordering
partition assignment
processing time
```

---

# 37. Kafka Partition Limitation

Suppose:

```text
Kafka topic:
10 partitions
```

and:

```text
consumer replicas:
20
```

Only up to the available partitions can have active consumers in the same consumer group at a time.

Therefore:

```text
20 replicas
```

does not automatically mean:

```text
20 active consumers
```

Partition design must therefore be considered during Kafka autoscaling.

---

# 38. KEDA for RabbitMQ

Architecture:

```text
RabbitMQ queue
      ↓
Queue depth
      ↓
KEDA
      ↓
Consumer replicas
```

Again, scaling depends on:

```text
queue depth
processing time
prefetch
acknowledgment
consumer concurrency
```

---

# 39. KEDA and Downstream Dependencies

Suppose:

```text
SQS backlog = 100,000
```

KEDA might want:

```text
100 Pods
```

But if every consumer writes to:

```text
RDS
```

this could overwhelm the database.

Therefore configure:

```text
maxReplicaCount
```

based on downstream capacity.

Example:

```yaml
maxReplicaCount: 20
```

not:

```yaml
maxReplicaCount: 200
```

just because backlog is high.

---

# 40. Autoscaling Chain

Our complete architecture now becomes:

```text
                  ┌──────────────┐
                  │    Traffic   │
                  └──────┬───────┘
                         │
                         ▼
                      HPA
                         │
                         ▼
                    Pod replicas
                         │
                         ▼
                 Pending Pods?
                         │
                         ▼
                    Karpenter
                         │
                         ▼
                      EC2
```

For event-driven services:

```text
              SQS / Kafka / RabbitMQ
                         │
                         ▼
                        KEDA
                         │
                         ▼
                    Pod replicas
                         │
                         ▼
                    Karpenter
                         │
                         ▼
                      EC2
```

For resource tuning:

```text
                    VPA
                     │
                     ▼
             Resource recommendations
                     │
                     ▼
              Helm / GitOps
                     │
                     ▼
                   Pods
```

---

# 41. Autoscaling Strategy for Our 21 Services

We should not give every service the same autoscaling model.

Example initial classification:

| Service                | Scaling model                       |
| ---------------------- | ----------------------------------- |
| auth-service           | HPA                                 |
| user-service           | HPA                                 |
| customer-service       | HPA                                 |
| catalog-service        | HPA                                 |
| product-service        | HPA                                 |
| inventory-service      | HPA                                 |
| pricing-service        | HPA                                 |
| promotion-service      | HPA                                 |
| cart-service           | HPA                                 |
| order-service          | HPA                                 |
| payment-service        | HPA + careful dependency limits     |
| fraud-service          | HPA / event-driven where applicable |
| shipping-service       | HPA                                 |
| delivery-service       | HPA/KEDA depending on events        |
| notification-service   | KEDA                                |
| search-service         | HPA                                 |
| recommendation-service | HPA / workload-specific             |
| review-service         | HPA                                 |
| tax-service            | HPA                                 |
| invoice-service        | KEDA/event-driven where applicable  |
| reporting-service      | KEDA/batch-oriented                 |

This is an **initial architecture classification**, not a final decision. Actual scaling signals should come from workload behavior.

---

# 42. API Services

For synchronous APIs:

```text
HTTP request
    ↓
Service
    ↓
Pod
```

HPA is generally the first scaling mechanism to evaluate.

Signals may include:

```text
CPU
memory
requests/sec
latency
concurrency
```

---

# 43. Worker Services

For asynchronous workers:

```text
Queue
   ↓
Worker
```

event-based metrics are often more meaningful:

```text
SQS backlog
Kafka lag
RabbitMQ queue depth
```

KEDA becomes particularly useful.

---

# 44. Batch Services

For:

```text
reporting-service
```

we may have:

```text
scheduled job
```

or:

```text
queue-driven job
```

Scaling might be based on:

```text
queue depth
job count
schedule
```

rather than HTTP CPU utilization.

---

# 45. Autoscaling and Graceful Shutdown

When scaling down:

```text
Pod termination
```

must not abruptly terminate active requests.

Our Spring Boot application should support graceful shutdown.

Conceptually:

```text
HPA reduces replicas
       ↓
Pod termination begins
       ↓
Readiness becomes false
       ↓
Service stops sending new traffic
       ↓
Existing requests complete
       ↓
Application shuts down
```

This connects directly to our zero-downtime deployment design.

---

# 46. Autoscaling and Readiness Probes

A Pod should not receive traffic before it is ready.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

This matters during:

```text
scale up
rolling deployment
node replacement
```

---

# 47. Autoscaling and Startup Time

Suppose:

```text
Spring Boot startup = 40 seconds
```

Traffic suddenly increases.

HPA creates:

```text
10 new Pods
```

But they need:

```text
40 seconds
```

to become ready.

Therefore scaling must account for:

```text
startup latency
image pull time
node provisioning time
application initialization
database connection establishment
```

This is why traffic-spike testing will be important in Phase 16.

---

# 48. Karpenter Interaction

Suppose:

```text
Current:
3 Pods
```

HPA decides:

```text
10 Pods
```

but existing nodes only have capacity for:

```text
6 Pods
```

Remaining:

```text
4 Pending Pods
```

Karpenter sees them and provisions capacity.

Therefore:

```text
HPA is not node autoscaling.
Karpenter is not Pod autoscaling.
```

They solve different layers.

---

# 49. HPA vs Karpenter

```text
HPA:
"How many Pods do I need?"

Karpenter:
"What EC2 capacity do I need to run those Pods?"
```

This distinction is fundamental.

---

# 50. HPA vs VPA

```text
HPA:
"How many replicas?"

VPA:
"How much CPU/memory should each Pod request?"
```

---

# 51. KEDA vs HPA

KEDA itself integrates with Kubernetes autoscaling mechanisms and can expose external event signals for scaling.

Think:

```text
HPA:
CPU/memory/custom metrics

KEDA:
external event sources
```

Examples:

```text
SQS
Kafka
RabbitMQ
Prometheus
Redis
```

depending on supported scalers.

---

# 52. Avoid Autoscaler Conflicts

A dangerous design is:

```text
HPA scales replicas
+
KEDA scales same workload independently
+
VPA automatically changes CPU requests
+
multiple external systems modify replicas
```

Without a clear design, this can produce unpredictable behavior.

For each workload define:

```text
Primary scaler
Scaling signals
Minimum replicas
Maximum replicas
Resource requests
Downstream limits
Scale-up speed
Scale-down behavior
```

---

# 53. Production Autoscaling Matrix

Example:

| Workload             | Primary scaler | Min | Max |
| -------------------- | -------------- | --: | --: |
| order-service        | HPA            |   3 |  15 |
| catalog-service      | HPA            |   3 |  12 |
| search-service       | HPA            |   3 |  20 |
| notification-service | KEDA           |   2 |  20 |
| reporting-service    | KEDA           | 0/1 |  10 |
| payment-service      | HPA            |   3 |  10 |

Final values come from performance testing.

---

# 54. Scaling Safety Limits

Every autoscaled workload should have:

```text
minimum replicas
maximum replicas
```

and dependency-aware limits.

For example:

```yaml
maxReplicas: 15
```

is a safety boundary.

Without maximum limits, an external traffic spike can potentially cause:

```text
massive Pod creation
massive EC2 provisioning
database overload
queue consumer overload
AWS cost spike
```

---

# 55. Flash Sale Example

Suppose normal traffic:

```text
1,000 req/s
```

Flash sale:

```text
20,000 req/s
```

HPA:

```text
3 → 15 Pods
```

Karpenter:

```text
existing capacity insufficient
```

Karpenter provisions:

```text
new nodes
```

Pods become:

```text
Ready
```

But if RDS cannot support the resulting connection load:

```text
application scales
↓
database saturates
↓
latency increases
↓
requests retry
↓
more load
```

This can become a cascading failure.

Therefore autoscaling is not simply:

> "Scale as high as possible."

It is:

> **Scale within the capacity of the complete dependency chain.**

---

# 56. Production Guardrails

Our autoscaling design should include:

```text
HPA maxReplicas
KEDA maxReplicaCount
PDB
ResourceQuota
RDS connection limits
Redis capacity
SQS throughput considerations
Kafka partition capacity
ALB limits
application concurrency
rate limiting
timeouts
circuit breakers
```

This becomes particularly important during:

```text
Traffic Spike / Flash Sale
```

in Phase 16.

---

# 57. Testing HPA

Controlled test:

```bash
kubectl get hpa -n ecommerce-dev
```

Generate load against the test environment.

Observe:

```bash
kubectl get pods -n ecommerce-dev -w
```

and:

```bash
kubectl get hpa -n ecommerce-dev -w
```

Then:

```bash
kubectl top pods -n ecommerce-dev
```

Expected:

```text
CPU increases
   ↓
HPA desired replicas increase
   ↓
Pods created
```

---

# 58. Testing KEDA

Controlled environment:

```text
1. Deploy consumer
2. Verify minimum replicas
3. Add messages to test queue
4. Observe queue depth
5. Observe KEDA
6. Observe replica count
7. Verify consumers process messages
8. Queue drains
9. Observe scale-down
```

Do not test destructive scaling scenarios directly against production queues.

---

# 59. HPA Failure Scenario

### Problem

```text
HPA stays at 3 replicas.
Traffic is high.
```

### Check

```bash
kubectl get hpa -n ecommerce-prod
```

Then:

```bash
kubectl describe hpa order-service \
  -n ecommerce-prod
```

Check:

```text
metrics
conditions
events
```

Then:

```bash
kubectl top pods -n ecommerce-prod
```

Then:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod
```

Possible causes:

```text
Metrics Server issue
incorrect resource requests
metric unavailable
HPA configuration
maxReplicas reached
```

---

# 60. KEDA Failure Scenario

### Problem

```text
SQS backlog:
50,000

Consumers:
2

Expected:
more replicas
```

Check:

```bash
kubectl get scaledobject -n ecommerce-prod
```

Then:

```bash
kubectl describe scaledobject notification-service \
  -n ecommerce-prod
```

Check:

```text
trigger
authentication
queue configuration
desired replicas
errors
```

Then inspect KEDA operator logs.

Exact resource names depend on the KEDA installation.

---

# 61. VPA Failure Scenario

### Problem

VPA recommendation appears unreasonable.

Do not immediately enable automatic updates.

Check:

```text
measurement window
traffic pattern
CPU spikes
memory spikes
startup behavior
batch processing
JVM behavior
```

A short-lived workload spike should not automatically dictate a permanent resource request.

---

# 62. Autoscaling Observability

Prometheus/Grafana should eventually show:

```text
HPA desired replicas
HPA current replicas
HPA metric values
Pod CPU
Pod memory
Pending Pods
Karpenter nodes
Node CPU
Node memory
SQS queue depth
Kafka consumer lag
RabbitMQ queue depth
```

Then an engineer can correlate:

```text
Traffic
 ↓
Metric
 ↓
HPA/KEDA
 ↓
Pods
 ↓
Karpenter
 ↓
Nodes
 ↓
Application latency
```

---

# 63. GitOps Ownership

Autoscaling configuration belongs in Git.

Example:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

Example:

```yaml
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 70
```

ArgoCD then reconciles:

```text
Git
 ↓
ArgoCD
 ↓
Kubernetes
```

We should not manually edit the production HPA and leave Git inconsistent.

---

# 64. Complete Autoscaling Flow

Our final production flow is:

```text
                         USER TRAFFIC
                              │
                              ▼
                             ALB
                              │
                              ▼
                          Kubernetes
                           Service
                              │
                              ▼
                           Pods
                              │
               ┌──────────────┼──────────────┐
               │              │              │
              CPU           Memory        App Metrics
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                             HPA
                              │
                              ▼
                        Replica Count
                              │
                              ▼
                     Pending Pods?
                              │
                              ▼
                         Karpenter
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                  Spot              On-Demand
                    │                   │
                    └─────────┬─────────┘
                              ▼
                         EC2 Nodes
```

Event-driven:

```text
SQS / Kafka / RabbitMQ
          │
          ▼
         KEDA
          │
          ▼
     Pod replicas
          │
          ▼
     Karpenter
          │
          ▼
       EC2 Nodes
```

Resource optimization:

```text
Workload
   │
   ▼
  VPA
   │
   ▼
Resource recommendation
   │
   ▼
Helm values
   │
   ▼
GitOps
   │
   ▼
ArgoCD
   │
   ▼
Deployment
```

---

# 65. Production Autoscaling Checklist

### HPA

```text
[ ] autoscaling/v2
[ ] resource requests configured
[ ] minReplicas defined
[ ] maxReplicas defined
[ ] target metric defined
[ ] scale behavior reviewed
[ ] downstream capacity reviewed
[ ] HPA tested
```

### VPA

```text
[ ] VPA installed
[ ] recommendation mode tested
[ ] recommendations reviewed
[ ] HPA/VPA interaction understood
[ ] automatic updates not enabled blindly
```

### KEDA

```text
[ ] KEDA installed
[ ] trigger selected
[ ] authentication configured
[ ] min replicas defined
[ ] max replicas defined
[ ] queue/lag metric validated
[ ] downstream capacity reviewed
[ ] scale-up tested
[ ] scale-down tested
```

### Karpenter

```text
[ ] Pending Pods trigger capacity
[ ] NodePool constraints validated
[ ] Spot strategy validated
[ ] On-Demand fallback available where required
[ ] AZ requirements validated
[ ] instance flexibility validated
```

---

# 66. Key Production Rules

### Rule 1

**HPA scales Pods, not EC2 nodes.**

### Rule 2

**Karpenter scales EC2 capacity, not application replicas.**

### Rule 3

**VPA changes/recommends Pod resources, not replica count.**

### Rule 4

**KEDA is particularly useful for event-driven workloads.**

### Rule 5

**Never configure maximum replicas without considering downstream systems.**

### Rule 6

**Resource requests must be realistic because they influence scheduling and autoscaling.**

### Rule 7

**Autoscaling must be tested under realistic load.**

### Rule 8

**GitOps remains the source of truth for production configuration.**

---

# 67. What We Have Now

Our EKS scaling architecture has progressed from:

```text
100 → Spot Instances
101 → Multi-AZ Workload Placement
102 → Kubernetes Resource Management
103 → HPA / VPA / KEDA
```

So we now have:

```text
Workload
   │
   ├── Resources
   │
   ├── HPA
   │
   ├── KEDA
   │
   ├── VPA recommendations
   │
   ├── Multi-AZ placement
   │
   ├── PDB
   │
   └── Karpenter
          │
          ├── Spot
          └── On-Demand
```

The next missing layer is the **cluster-level capacity controller**.

# 104 — Cluster Autoscaling

That chapter will connect:

```text
HPA
  ↓
Pending Pods
  ↓
Karpenter / Cluster Autoscaler
  ↓
Node provisioning
  ↓
EC2
  ↓
Scheduling
```

and explain when to use **Karpenter vs Managed Node Groups + Cluster Autoscaler**, system/application node pools, scale-up/scale-down, consolidation, Spot interruption, capacity limits, and production failure scenarios.
