# 82 — Blue-Green Deployment Implementation

> Production implementation of **Blue-Green Deployment** for `order-service` on AWS EKS using Helm, GitOps, ArgoCD, ECR, ALB, observability, Jira Change Management, and controlled rollback.

---

## 1. Objective

In Chapter 81 we implemented:

```text
Rolling Deployment
```

Now we implement:

```text
Blue-Green Deployment
```

The basic idea is simple:

```text
BLUE   = currently serving production traffic
GREEN  = new version deployed separately
```

Instead of gradually replacing Pods, we keep two complete application versions available.

```text
                    ALB
                     |
                     v
                Kubernetes
                  Service
                     |
              +------+------+
              |             |
          BLUE v1        GREEN v2
          3 Pods          3 Pods
```

Before release:

```text
Traffic
   |
   v
BLUE v1
```

During testing:

```text
No production traffic
        |
        v
GREEN v2
```

After approval:

```text
Traffic
   |
   v
GREEN v2
```

If a serious problem occurs:

```text
Traffic
   |
   v
BLUE v1
```

---

# 2. Important Difference From Rolling Deployment

Rolling:

```text
v1 → v1 + v2 → v2
```

Blue-Green:

```text
v1 + v2
   ↓
switch traffic
   ↓
v2
```

For example:

### Rolling

```text
3 × v1

Deploy v2:

2 × v1
1 × v2

then:

1 × v1
2 × v2

then:

0 × v1
3 × v2
```

### Blue-Green

```text
3 × v1
+
3 × v2
```

Both environments exist simultaneously.

Then:

```text
Traffic → v2
```

---

# 3. Why Would We Use Blue-Green?

Blue-Green is useful when you want:

```text
Fast traffic switching
Fast rollback
Full new-version validation
Reduced exposure during the switch
```

But there is a cost:

```text
BLUE capacity
+
GREEN capacity
```

So if production normally needs:

```text
3 Pods
```

Blue-Green may temporarily require:

```text
6 Pods
```

plus additional node capacity.

This is one of the main trade-offs.

---

# 4. Our Production Architecture

Our existing architecture:

```text
Internet
   |
Route 53
   |
CloudFront / WAF
   |
ALB
   |
AWS Load Balancer Controller
   |
Kubernetes
```

For Blue-Green:

```text
                         ALB
                          |
                    Kubernetes Service
                          |
               +----------+----------+
               |                     |
          BLUE deployment       GREEN deployment
             v1                     v2
          3 Pods                  3 Pods
```

The key question is:

> How does the Service decide which version receives traffic?

We have multiple implementation choices.

For this chapter, we will first implement the simplest Kubernetes-native approach:

```text
Service selector
       ↓
Blue OR Green
```

Later we will discuss ALB-based traffic switching and Argo Rollouts.

---

# 5. The Two Deployments

Instead of one Deployment:

```text
order-service
```

we create:

```text
order-service-blue
order-service-green
```

Example:

```text
order-service-blue
    |
    +-- Pod v1
    +-- Pod v1
    +-- Pod v1


order-service-green
    |
    +-- Pod v2
    +-- Pod v2
    +-- Pod v2
```

---

# 6. Labels Are Extremely Important

Blue:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/component: application
  deployment-slot: blue
```

Green:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/component: application
  deployment-slot: green
```

The critical label is:

```text
deployment-slot
```

because the Service will use it.

---

# 7. Blue Deployment

Conceptually:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service-blue

spec:
  replicas: 3

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
      deployment-slot: blue

  template:

    metadata:
      labels:
        app.kubernetes.io/name: order-service
        deployment-slot: blue

    spec:

      containers:
        - name: order-service

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:BLUE_DIGEST

          ports:
            - containerPort: 8080
```

---

# 8. Green Deployment

Green is almost identical:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service-green

spec:
  replicas: 3

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
      deployment-slot: green

  template:

    metadata:
      labels:
        app.kubernetes.io/name: order-service
        deployment-slot: green

    spec:

      containers:
        - name: order-service

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:GREEN_DIGEST

          ports:
            - containerPort: 8080
```

Now Kubernetes has:

```text
BLUE
3 Pods
v1


GREEN
3 Pods
v2
```

---

# 9. The Service

Now create one Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service

spec:

  type: ClusterIP

  selector:
    app.kubernetes.io/name: order-service
    deployment-slot: blue

  ports:
    - name: http
      port: 80
      targetPort: 8080
```

Notice:

```yaml
deployment-slot: blue
```

Therefore:

```text
Service
   |
   +-- Blue Pods
```

Green Pods are running but aren't selected by this Service.

---

# 10. Initial State

Production:

```text
                    ALB
                     |
                     v
              order-service
                     |
              selector: blue
                     |
          +----------+----------+
          |          |           |
       Blue-1     Blue-2      Blue-3
         v1          v1          v1


Green:
       Green-1
       Green-2
       Green-3

       v2
```

Traffic only reaches:

```text
Blue
```

---

# 11. Deploy Green

Suppose CI builds:

```text
order-service
```

with:

```text
Git SHA:
8f31a2c
```

ECR image:

```text
order-service:8f31a2c
```

Digest:

```text
sha256:GREEN_DIGEST
```

GitOps changes the Green deployment:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:GREEN_DIGEST"
```

ArgoCD applies it.

Now:

```text
Blue
3 Pods
v1

Green
3 Pods
v2
```

Traffic remains:

```text
ALB
 ↓
Service
 ↓
Blue
```

---

# 12. Validate Green Before Switching Traffic

This is one of the major advantages of Blue-Green.

We can test Green independently.

For example:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l deployment-slot=green
```

Expected:

```text
order-service-green-xxx   Running   1/1
order-service-green-yyy   Running   1/1
order-service-green-zzz   Running   1/1
```

Check:

```bash
kubectl get deployment order-service-green \
  -n ecommerce-prod
```

Expected:

```text
READY
3/3
```

---

# 13. Test Green Through a Temporary Service

We don't want production traffic going to Green yet.

Create a temporary Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service-green-test

spec:

  type: ClusterIP

  selector:
    app.kubernetes.io/name: order-service
    deployment-slot: green

  ports:
    - port: 80
      targetPort: 8080
```

Now:

```text
order-service-green-test
          |
          v
       Green Pods
```

We can test it from inside the cluster.

For example:

```bash
kubectl run curl-test \
  --rm \
  -it \
  --image=curlimages/curl \
  -n ecommerce-prod \
  -- sh
```

Then:

```bash
curl http://order-service-green-test/actuator/health
```

And:

```bash
curl http://order-service-green-test/api/v1/orders
```

This allows functional validation of Green before production traffic moves.

---

# 14. Green Validation Checklist

Before switching traffic:

```text
[ ] Green Pods Running
[ ] Green Pods Ready
[ ] Startup probes successful
[ ] Readiness probes successful
[ ] Liveness probes successful
[ ] Application health successful
[ ] Database connectivity successful
[ ] Redis connectivity successful
[ ] Required AWS permissions working
[ ] API smoke tests successful
[ ] Logs normal
[ ] Error rate normal
[ ] CPU normal
[ ] Memory normal
```

---

# 15. The Traffic Switch

Now comes the important operation.

Current Service:

```yaml
selector:
  deployment-slot: blue
```

Change to:

```yaml
selector:
  deployment-slot: green
```

Now:

```text
Before:

Service
   ↓
Blue
```

After:

```text
Service
   ↓
Green
```

The ALB doesn't need to know about Blue and Green.

It continues sending traffic to the Kubernetes Service.

The Service changes which Pods are endpoints.

---

# 16. Before the Switch

Run:

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=order-service
```

You should see Blue Pod IPs.

Conceptually:

```text
Service
   |
   +-- 10.0.1.10 Blue
   +-- 10.0.2.20 Blue
   +-- 10.0.3.30 Blue
```

---

# 17. After the Switch

Service selector changes:

```yaml
deployment-slot: green
```

Endpoint selection becomes:

```text
Service
   |
   +-- 10.0.1.40 Green
   +-- 10.0.2.50 Green
   +-- 10.0.3.60 Green
```

Traffic now goes to Green.

---

# 18. GitOps Should Control the Switch

We don't want an engineer manually editing:

```bash
kubectl edit service order-service
```

in normal production operations.

Instead:

```text
GitOps repository
        |
        v
Service selector
        |
        v
ArgoCD
        |
        v
EKS
```

Example production values:

```yaml
activeSlot: blue
```

Then:

```text
activeSlot: green
```

A GitOps PR changes the active slot.

---

# 19. Helm Values

For example:

```yaml
activeSlot: blue

blue:
  replicaCount: 3

  image:
    repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
    digest: "sha256:BLUE_DIGEST"

green:
  replicaCount: 3

  image:
    repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
    digest: "sha256:GREEN_DIGEST"
```

The Service template:

```yaml
selector:
  app.kubernetes.io/name: order-service
  deployment-slot: {{ .Values.activeSlot }}
```

So:

```text
activeSlot: blue
```

means:

```text
Service → Blue
```

and:

```text
activeSlot: green
```

means:

```text
Service → Green
```

---

# 20. Helm Deployment Templates

We can make the slot configurable.

Example:

```yaml
{{- range $slot := list "blue" "green" }}

apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ include "order-service.fullname" $ }}-{{ $slot }}

spec:
  replicas: {{ index $.Values $slot "replicaCount" }}

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
      deployment-slot: {{ $slot }}

  template:

    metadata:
      labels:
        app.kubernetes.io/name: order-service
        deployment-slot: {{ $slot }}

    spec:

      containers:

        - name: order-service

          image: ...

---

{{- end }}
```

In a real chart, I would structure this more explicitly rather than relying on a complicated template loop if it makes the chart difficult for the team to maintain.

The principle matters:

```text
Blue Deployment
Green Deployment
One active Service
```

---

# 21. Blue-Green State Machine

Think of deployment as states.

### State 1

```text
BLUE = ACTIVE
GREEN = INACTIVE
```

### State 2

```text
BLUE = ACTIVE
GREEN = DEPLOYING
```

### State 3

```text
BLUE = ACTIVE
GREEN = READY
```

### State 4

```text
BLUE = OLD
GREEN = ACTIVE
```

### Rollback

```text
BLUE = READY
GREEN = ACTIVE
```

Then:

```text
Service → BLUE
```

---

# 22. Blue-Green Switch

Suppose:

```text
BLUE
image digest:
sha256:1111

GREEN
image digest:
sha256:2222
```

Current:

```text
activeSlot: blue
```

Change:

```text
activeSlot: green
```

ArgoCD:

```text
GitOps
  ↓
ArgoCD
  ↓
Service selector
  ↓
Green
```

Traffic:

```text
Client
  ↓
ALB
  ↓
Service
  ↓
GREEN
```

---

# 23. The Switch Is Fast

The actual Kubernetes selector change is small.

You don't need to:

```text
Delete Blue
Create Green
```

Green already exists.

You're essentially changing:

```text
Service → Blue
```

to:

```text
Service → Green
```

This is why rollback can also be fast:

```text
Service → Green
```

back to:

```text
Service → Blue
```

---

# 24. But Don't Call It Instantaneous

There are still real-world considerations:

```text
Endpoint update
ALB target propagation
Connection reuse
Existing keep-alive connections
DNS/cache behavior where applicable
Application-level state
```

Therefore:

> Blue-Green provides a fast traffic transition, but you should validate the actual behavior of your ALB and application before treating the switch as instantaneous.

---

# 25. Smoke Test Immediately After Switch

Immediately after switching:

```bash
curl -f https://api.example.com/actuator/health
```

Then:

```bash
curl -f \
  https://api.example.com/api/v1/orders
```

Run representative business tests:

```text
Create order
Get order
Update order
Payment interaction
Inventory interaction
Notification event
```

depending on which dependencies are part of the service.

---

# 26. Verify Which Version Is Receiving Traffic

One useful production technique is to expose a non-sensitive build/version identifier.

For example:

```text
application version:
2.3.0
git commit:
8f31a2c
```

Then an internal endpoint can report:

```json
{
  "service": "order-service",
  "version": "2.3.0",
  "commit": "8f31a2c"
}
```

Don't expose unnecessary build metadata publicly.

Alternatively, inspect logs and metrics containing:

```text
service.version
deployment.environment
git.commit
```

This allows you to establish:

```text
Traffic
   ↓
Green
   ↓
Commit 8f31a2c
   ↓
Image digest sha256:...
```

---

# 27. Prometheus During the Switch

Before switch:

```text
Blue:
Error rate = normal
Latency = normal
```

After switch:

```text
Green:
Error rate = ?
Latency = ?
```

Monitor:

```text
HTTP 4xx
HTTP 5xx
P95 latency
P99 latency
Request rate
JVM memory
JVM CPU
Pod restarts
Database errors
```

A successful Kubernetes switch does not automatically mean a successful application release.

---

# 28. Automatic Rollback Concept

Suppose:

```text
Green becomes active
```

and within five minutes:

```text
HTTP 5xx increases
```

For example:

```text
Before:
5xx = 0.1%

After:
5xx = 8%
```

Your incident/change procedure can trigger rollback.

```text
GREEN ACTIVE
     ↓
Metrics abnormal
     ↓
Rollback decision
     ↓
Service → BLUE
     ↓
Blue receives traffic
```

For the simple Kubernetes Service implementation, this decision can be manual.

Later, **Argo Rollouts** can automate progressive traffic analysis and abort behavior.

---

# 29. Rollback

Suppose:

```text
BLUE = v1
GREEN = v2
```

Current:

```text
Service → GREEN
```

Rollback:

```text
Service → BLUE
```

GitOps:

```text
activeSlot: blue
```

Commit/revert:

```text
GitOps
  ↓
ArgoCD
  ↓
Service selector
  ↓
Blue
```

Then verify:

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=order-service
```

and:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l deployment-slot=blue
```

---

# 30. What Happens to Green After Rollback?

Do not immediately delete Green.

Keep it temporarily available.

For example:

```text
BLUE
v1
ACTIVE

GREEN
v2
INACTIVE
```

Why?

Because you may need:

```text
Logs
Debugging
Comparison
Root-cause analysis
Additional testing
```

After the incident/change is stabilized, decide whether Green should be scaled down.

---

# 31. Scaling Green Down

After successful deployment:

```text
BLUE = old
GREEN = active
```

Eventually:

```text
BLUE replicas:
3 → 0
```

while:

```text
GREEN replicas:
3
```

This saves resources.

But there's a trade-off:

```text
Keep Blue running
    ↓
Fastest rollback
    ↓
Higher cost
```

versus:

```text
Scale Blue to zero
    ↓
Lower cost
    ↓
Rollback requires starting Blue
```

This is an operational decision.

---

# 32. Database Compatibility

This is one of the most important Blue-Green considerations.

Suppose v1 expects:

```text
orders.status
```

and v2 expects:

```text
orders.order_status
```

If Green and Blue run simultaneously:

```text
Blue v1
   ↓
Database

Green v2
   ↓
Database
```

Both versions may access the same database.

If the database schema changes incompatibly:

```text
Blue breaks
```

even before you switch traffic.

Therefore:

> Blue-Green application deployment does not remove database compatibility requirements.

---

# 33. Expand-and-Contract Migration

Use a compatible migration sequence.

Suppose we need:

```text
old column:
status

new column:
order_status
```

### Phase 1 — Expand

Add:

```text
order_status
```

while keeping:

```text
status
```

Both versions can work.

### Phase 2 — Application

Deploy application capable of using the new field.

### Phase 3 — Backfill

Populate new data.

### Phase 4 — Verify

Confirm all applications use the new field.

### Phase 5 — Contract

Only later remove:

```text
status
```

This is safer than:

```text
Drop old column
       ↓
Deploy new application
```

---

# 34. Shared Database Architecture

Our architecture is:

```text
Blue order-service
       |
       +------+
              |
              v
        RDS PostgreSQL
              ^
       +------+
       |
Green order-service
```

Therefore both versions may temporarily coexist.

The database migration strategy must support that coexistence.

---

# 35. Secrets During Blue-Green

Both deployments use the same runtime secret mechanism.

For example:

```text
Blue
  ↓
ServiceAccount
  ↓
EKS Pod Identity
  ↓
IAM
  ↓
Secrets Manager
```

Green:

```text
Green
  ↓
ServiceAccount
  ↓
EKS Pod Identity
  ↓
IAM
  ↓
Secrets Manager
```

Do not create duplicate production credentials merely because you have Blue and Green.

Use the appropriate shared or versioned secret strategy.

---

# 36. Redis Consideration

Suppose:

```text
Blue
   ↓
Redis

Green
   ↓
Redis
```

If the new version changes cache structure:

```text
v1 cache format
≠
v2 cache format
```

Blue-Green can expose compatibility problems.

Use:

```text
Versioned keys
```

where necessary.

Example:

```text
order:v1:123
order:v2:123
```

or design the cache format so both application versions can coexist safely.

---

# 37. Messaging Consideration

For SQS/Kafka/RabbitMQ:

```text
Blue
  ↓
Consumer

Green
  ↓
Consumer
```

If both versions consume the same queue simultaneously, you need to understand the consequences.

For example:

```text
Queue
 |
 +-- Blue consumer
 |
 +-- Green consumer
```

Messages may be consumed by either version.

That may be correct or incorrect depending on the deployment design.

For consumers, Blue-Green often needs additional coordination around:

```text
Consumer groups
Message schema
Idempotency
Duplicate processing
Visibility timeout
Offsets
```

This is especially important for Kafka.

---

# 38. Blue-Green for Stateless HTTP Services

Our first implementation is most suitable for:

```text
order-service
```

assuming it is designed as a stateless HTTP application with externalized state.

The application should not depend on:

```text
Local Pod filesystem
In-memory session state
Pod-specific identity
Local-only cache
```

because Blue and Green may coexist.

---

# 39. Kubernetes Service Configuration

Our active Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service

spec:

  selector:
    app.kubernetes.io/name: order-service
    deployment-slot: {{ .Values.activeSlot }}

  ports:
    - name: http
      port: 80
      targetPort: 8080

  type: ClusterIP
```

This becomes the traffic control point.

---

# 40. Ingress/ALB Doesn't Change

Our ALB Ingress continues pointing to:

```text
order-service
```

Example:

```yaml
backend:
  service:
    name: order-service
    port:
      number: 80
```

We don't need to change the ALB every time.

The flow remains:

```text
ALB
 ↓
order-service Service
 ↓
active slot
 ↓
Pods
```

---

# 41. Why This Is Better Than Editing the ALB Manually

If we manually modify ALB configuration outside Kubernetes:

```text
ALB
 ↓
Manual console change
```

we create configuration drift.

Our preferred architecture:

```text
Git
 ↓
ArgoCD
 ↓
Kubernetes Service
 ↓
ALB
```

The desired state remains version controlled.

---

# 42. GitOps Example

Production:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

Before:

```yaml
activeSlot: blue
```

Deployment:

```yaml
blue:
  image:
    digest: sha256:111111

green:
  image:
    digest: sha256:222222
```

After Green validation:

```yaml
activeSlot: green
```

The Git change represents the traffic switch.

---

# 43. Jira Change Request

Our example:

```text
CHG-10453
```

Change:

```text
Blue-Green production deployment
```

Implementation plan:

```text
1. Verify staging deployment
2. Verify GREEN image digest
3. Deploy GREEN
4. Validate GREEN health
5. Execute functional smoke tests
6. Confirm monitoring
7. Switch Service from BLUE to GREEN
8. Monitor production
9. Close change
```

Rollback:

```text
1. Change Service selector GREEN → BLUE
2. ArgoCD sync
3. Verify Blue endpoints
4. Run smoke tests
5. Monitor metrics
6. Record rollback in Jira
```

---

# 44. Production Change Timeline

Example:

```text
10:00
Change window starts

10:05
Green deployment begins

10:08
Green Pods Ready

10:10
Green functional tests complete

10:12
Production traffic switched

10:13
Smoke tests

10:15
Metrics checked

10:30
Change successfully completed
```

If a problem occurs:

```text
10:12
Traffic switched

10:14
5xx increases

10:15
Rollback initiated

10:16
Traffic → Blue

10:18
Blue healthy

10:25
Rollback documented
```

---

# 45. Blue-Green Failure Scenarios

## Scenario 1 — Green image doesn't exist

```text
Green
 ↓
ImagePullBackOff
```

Traffic remains:

```text
Blue
```

No production traffic switch should occur.

---

## Scenario 2 — Green health check fails

```text
Green
 ↓
NotReady
```

Don't switch traffic.

---

## Scenario 3 — Green functional tests fail

```text
Green
 ↓
Application works
 ↓
Business operation fails
```

Don't switch.

---

## Scenario 4 — Green works in testing but fails under production traffic

Switch:

```text
Green
```

then metrics degrade.

Rollback:

```text
Green → Blue
```

---

# 46. Important Difference From Rolling

With Rolling:

```text
v1 + v2
```

may coexist while production traffic is already being served by both versions.

With Blue-Green:

```text
v1
```

can remain the active production version while:

```text
v2
```

is validated separately.

This can simplify release validation, but it requires additional infrastructure capacity.

---

# 47. Blue-Green Resource Requirements

Suppose:

```text
Blue:
3 Pods
```

and:

```text
Green:
3 Pods
```

During deployment:

```text
6 application Pods
```

If each Pod requests:

```text
250m CPU
512Mi memory
```

then application requests become approximately:

```text
CPU:
6 × 250m = 1500m

Memory:
6 × 512Mi = 3072Mi
```

And that is before accounting for:

```text
DaemonSets
system Pods
other workloads
headroom
```

Therefore cluster capacity must be checked before the deployment.

---

# 48. Karpenter / Cluster Autoscaler

If there isn't enough capacity:

```text
Green Pod
   ↓
Pending
   ↓
Insufficient resources
```

A node autoscaler can provision additional capacity.

Our architecture may therefore be:

```text
Blue + Green
      ↓
EKS
      ↓
Karpenter
      ↓
Additional node
```

But don't assume autoscaling will solve everything.

Check:

```text
Instance availability
Subnet IP capacity
Pod limits
Node constraints
AZ capacity
Taints
Tolerations
NodePool requirements
```

---

# 49. Observability Architecture

During Blue-Green:

```text
Blue Pods ─┐
           |
Green Pods ├── Prometheus
           |
           ├── Fluent Bit
           |
           ├── Logstash
           |
           └── OpenTelemetry
```

Use labels such as:

```text
deployment_slot="blue"
```

or:

```text
deployment_slot="green"
```

where appropriate.

Then you can compare:

```text
Blue error rate
vs
Green error rate
```

This is extremely useful during release validation.

---

# 50. Grafana Dashboard

A production Blue-Green dashboard could show:

```text
                 order-service

             BLUE          GREEN
Requests      1,250         0
5xx             0.1%        0%
P95 latency    120ms        115ms
CPU             32%         29%
Memory          55%         51%
Restarts          0           0
Ready            3/3         3/3
```

After the switch:

```text
                 order-service

             BLUE          GREEN
Requests         0         1,300
5xx             0.1%        0.2%
P95 latency    120ms        125ms
```

This makes release behavior much easier to observe.

---

# 51. Cleanup

After successful deployment:

```text
GREEN = active
BLUE = old
```

Options:

### Option A — Keep Blue

```text
Blue = 3
Green = 3
```

Fastest rollback, higher cost.

### Option B — Scale Blue Down

```text
Blue = 0
Green = 3
```

Lower cost, slower rollback.

### Option C — Keep reduced Blue capacity

For example:

```text
Blue = 1
Green = 3
```

This can provide some standby capacity while reducing cost.

The actual choice should be based on:

```text
RTO
Cost
Service criticality
Rollback requirements
Cluster capacity
```

---

# 52. Important Limitation of This Simple Implementation

The Kubernetes Service selector approach gives us:

```text
100% Blue
OR
100% Green
```

It does **not** natively give:

```text
90% Blue
10% Green
```

or:

```text
50% Blue
50% Green
```

For controlled percentage traffic shifting, we need a traffic-management mechanism such as:

```text
AWS ALB weighted routing
Service mesh
Argo Rollouts traffic routing
Another supported ingress/traffic controller
```

That leads directly into Canary deployment.

---

# 53. Blue-Green vs Rolling

| Feature                       | Rolling             | Blue-Green      |
| ----------------------------- | ------------------- | --------------- |
| Old/new coexist               | Yes                 | Yes             |
| Full new environment          | No                  | Yes             |
| Extra capacity                | Lower               | Higher          |
| Traffic switch                | Gradual replacement | Explicit switch |
| Rollback                      | Git revert/rollout  | Switch back     |
| Pre-production validation     | Limited             | Strong          |
| Cost                          | Lower               | Higher          |
| Operational complexity        | Lower               | Medium          |
| Fast rollback                 | Moderate            | Fast            |
| Database compatibility needed | Yes                 | Yes             |

These are characteristics, not a universal rule for which strategy should be used.

---

# 54. Complete Blue-Green Flow

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    +-- Test
    +-- SonarQube
    +-- Trivy
    +-- Build
    |
    v
ECR
    |
    v
Immutable Digest
    |
    v
GitOps PR
    |
    v
Jira Change
    |
    v
Approval
    |
    v
ArgoCD
    |
    v
Deploy GREEN
    |
    v
Green Validation
    |
    +---- FAIL ----> Stop
    |
    v
Production Traffic Switch
    |
    v
GREEN ACTIVE
    |
    +---- Healthy ----> Complete
    |
    +---- Failure ----> BLUE ACTIVE
```

---

# 55. Practical Commands

### Check Blue

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l deployment-slot=blue
```

### Check Green

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l deployment-slot=green
```

### Check Deployments

```bash
kubectl get deployment \
  -n ecommerce-prod
```

### Check Service

```bash
kubectl get svc order-service \
  -n ecommerce-prod \
  -o yaml
```

### Check EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=order-service
```

### Check ArgoCD

```bash
argocd app get order-service-prod
```

### Check events

```bash
kubectl get events \
  -n ecommerce-prod \
  --sort-by=.lastTimestamp
```

---

# 56. Production Checklist

## Green Preparation

```text
[ ] Correct image tag
[ ] Correct image digest
[ ] ECR image exists
[ ] Trivy passed
[ ] GitOps PR approved
[ ] Jira change approved
[ ] Green deployment created
[ ] Green Pods Ready
```

## Green Validation

```text
[ ] Health check
[ ] Readiness
[ ] API tests
[ ] Database connectivity
[ ] Redis connectivity
[ ] Messaging connectivity
[ ] Logs
[ ] Metrics
[ ] Traces
[ ] Resource usage
```

## Traffic Switch

```text
[ ] Blue healthy
[ ] Green healthy
[ ] Monitoring active
[ ] Rollback confirmed
[ ] Service selector changed
[ ] EndpointSlices verified
[ ] Smoke tests passed
```

## After Switch

```text
[ ] Error rate normal
[ ] Latency normal
[ ] Application metrics normal
[ ] Logs normal
[ ] Database normal
[ ] No critical alerts
[ ] Jira updated
[ ] Old Blue retained/scaled according to policy
```

---

# 57. Final Architecture

```text
                         Internet
                            |
                        Route 53
                            |
                     CloudFront/WAF
                            |
                           ALB
                            |
                    Kubernetes Service
                            |
                 activeSlot = GREEN
                            |
              +-------------+-------------+
              |                           |
          BLUE v1                     GREEN v2
          3 Pods                      3 Pods
              |                           |
              |                           |
              +-------------+-------------+
                            |
                  External Dependencies
                  /       |        \
                RDS      Redis      SQS
```

The important concept is:

```text
BLUE
 ↓
Current production version

GREEN
 ↓
New production candidate

Service
 ↓
Traffic switch
```

And rollback:

```text
GREEN
 ↓
Problem
 ↓
Service selector → BLUE
 ↓
Traffic restored to previous version
```

---

# 58. What We Have Now

We have implemented two production deployment models:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
```

The next major model is:

```text
83 — Canary Deployment Implementation
```

Canary is different because instead of:

```text
100% Blue
      ↓
100% Green
```

we will implement the concept of:

```text
100% v1
      ↓
95% v1 + 5% v2
      ↓
90% v1 + 10% v2
      ↓
75% v1 + 25% v2
      ↓
50% v1 + 50% v2
      ↓
100% v2
```

and, importantly, we'll cover **why simply running 1 Pod of v2 does not mean exactly 10% of user traffic**, how ALB/traffic routing actually handles percentages, and how Prometheus metrics can be used as the release gate.
