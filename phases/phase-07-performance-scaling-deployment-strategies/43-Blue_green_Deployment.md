# 43 — Blue-Green Deployment

> **Enterprise E-Commerce Platform | AWS EKS | Kubernetes | Helm | ArgoCD | GitOps**

Blue-Green Deployment is a production deployment strategy where **two complete application environments exist at the same time**:

* **Blue** → currently serving production traffic
* **Green** → new version deployed and validated, but not yet serving production traffic

Traffic is switched from Blue to Green only after Green passes validation.

If Green fails, traffic can be switched back to Blue quickly.

---

# 1. Why Blue-Green Deployment?

A normal rolling deployment gradually replaces old Pods with new Pods.

Example:

```text
v1
v1
v1
v1

       ↓ deployment

v1
v1
v2
v2

       ↓

v2
v2
v2
v2
```

This works well, but during the rollout:

* old and new versions coexist
* some users may hit v1
* other users may hit v2
* application compatibility becomes important
* rollback may require another rollout

Blue-Green is different.

```text
                 ┌──────────────┐
                 │ Production   │
                 │ Traffic      │
                 └──────┬───────┘
                        │
                        ▼
                  ┌───────────┐
                  │  Service  │
                  └─────┬─────┘
                        │
                ┌───────┴────────┐
                │                │
                ▼                ▼
          ┌──────────┐     ┌──────────┐
          │   BLUE   │     │  GREEN   │
          │   v1.5   │     │   v1.6   │
          │          │     │          │
          │ Serving  │     │ Testing  │
          │ Traffic  │     │          │
          └──────────┘     └──────────┘
```

After validation:

```text
Production
    │
    ▼
 Service
    │
    └──────────────► GREEN v1.6
```

Blue remains available for rollback.

---

# 2. Blue-Green in Our E-Commerce Project

We have:

```text
21 Microservices
        │
        ▼
      EKS
        │
        ▼
      Helm
        │
        ▼
     ArgoCD
        │
        ▼
      GitOps
```

For an individual service such as:

```text
order-service
```

we could have:

```text
order-service-blue
order-service-green
```

For example:

```text
Blue  = order-service:1.5.0
Green = order-service:1.6.0
```

Initially:

```text
                    Users
                      │
                      ▼
                 ALB / Ingress
                      │
                      ▼
             order-service-active
                      │
                      ▼
              ┌──────────────┐
              │ BLUE v1.5.0  │
              └──────────────┘

              ┌──────────────┐
              │ GREEN v1.6.0 │
              │              │
              │ No traffic   │
              └──────────────┘
```

After successful validation:

```text
                    Users
                      │
                      ▼
                 ALB / Ingress
                      │
                      ▼
             order-service-active
                      │
                      ▼
              ┌──────────────┐
              │ GREEN v1.6.0 │
              └──────────────┘

              ┌──────────────┐
              │ BLUE v1.5.0  │
              │              │
              │ Standby      │
              └──────────────┘
```

---

# 3. Important Concept — Blue and Green Are Not Namespaces

A common misunderstanding is:

> Blue must be one namespace and Green another namespace.

Not necessarily.

Blue-Green can be implemented using:

### Option 1 — Same namespace

```text
production

order-service-blue
order-service-green
order-service-active
```

This is simple and often appropriate.

### Option 2 — Separate namespaces

```text
production-blue
production-green
```

This gives stronger isolation but introduces additional complexity.

For our project, we can primarily demonstrate:

```text
production namespace

blue Deployment
green Deployment
active Service
```

---

# 4. Kubernetes Architecture

The important part is the Service selector.

Blue:

```yaml
metadata:
  name: order-service-blue
spec:
  selector:
    app: order-service
    version: blue
```

Green:

```yaml
metadata:
  name: order-service-green
spec:
  selector:
    app: order-service
    version: green
```

Active Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-active
spec:
  selector:
    app: order-service
    version: blue
  ports:
    - port: 80
      targetPort: 8080
```

The Active Service decides which environment receives traffic.

---

# 5. How Traffic Switching Works

Initially:

```text
Service selector:

version: blue
```

Traffic:

```text
Client
  │
  ▼
ALB
  │
  ▼
Service
  │
  ▼
Blue Pods
```

To switch:

```text
version: blue
```

to:

```text
version: green
```

Traffic becomes:

```text
Client
  │
  ▼
ALB
  │
  ▼
Service
  │
  ▼
Green Pods
```

This is one of the simplest forms of Blue-Green switching.

---

# 6. Blue Deployment

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  namespace: production
spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service
      version: blue

  template:
    metadata:
      labels:
        app: order-service
        version: blue

    spec:
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.5.0

          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

---

# 7. Green Deployment

Green uses the new image.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  namespace: production
spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service
      version: green

  template:
    metadata:
      labels:
        app: order-service
        version: green

    spec:
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.6.0

          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

Now both environments exist:

```text
Blue
3 Pods
v1.5.0

Green
3 Pods
v1.6.0
```

But only Blue receives production traffic.

---

# 8. Active Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-active
  namespace: production
spec:
  selector:
    app: order-service
    version: blue

  ports:
    - port: 80
      targetPort: 8080

  type: ClusterIP
```

The ALB/Ingress points to:

```text
order-service-active
```

The ALB does **not** need to know whether Blue or Green is active.

That decision happens behind the Service.

---

# 9. Complete Traffic Flow

Our production architecture becomes:

```text
Internet
   │
   ▼
Route 53
   │
   ▼
CloudFront
   │
   ▼
AWS WAF
   │
   ▼
ALB
   │
   ▼
Ingress
   │
   ▼
order-service-active
   │
   ├──────────────┐
   │              │
   ▼              ▼
 BLUE           GREEN
 v1.5.0         v1.6.0
```

Only one is active.

---

# 10. Deployment Lifecycle

A realistic production deployment looks like this.

## Step 1 — Current production

```text
Blue = v1.5.0
Green = empty
```

Production:

```text
Traffic → Blue
```

---

## Step 2 — CI builds new version

Developer changes code.

```text
Git commit
     │
     ▼
CI
     │
     ├── Build
     ├── Unit Test
     ├── SonarQube
     ├── Gitleaks
     ├── Dependency Scan
     ├── Trivy
     ├── SBOM
     └── Docker
          │
          ▼
         ECR
```

Image:

```text
order-service:1.6.0
```

Preferably we ultimately deploy by immutable digest.

---

# 11. GitOps Update

CI should not directly modify production.

Instead:

```text
CI
 │
 ▼
ECR
 │
 ▼
GitOps PR
 │
 ▼
Review
 │
 ▼
Merge
 │
 ▼
ArgoCD
```

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: 1.6.0
```

---

# 12. ArgoCD Deploys Green

ArgoCD reconciles the GitOps state.

```text
GitOps
   │
   ▼
ArgoCD
   │
   ▼
EKS
   │
   ▼
Green Deployment
```

Now:

```text
Blue = v1.5.0 → serving traffic
Green = v1.6.0 → no production traffic
```

---

# 13. Green Validation

Before switching traffic, validate Green.

We should check:

### Kubernetes health

```bash
kubectl get pods -n production
```

### Deployment

```bash
kubectl get deployment -n production
```

### Endpoints

```bash
kubectl get endpoints -n production
```

### Events

```bash
kubectl get events -n production --sort-by=.lastTimestamp
```

### Logs

```bash
kubectl logs deployment/order-service-green -n production
```

---

# 14. Application Smoke Testing

Green should be tested independently.

For example:

```text
GET /actuator/health
```

Then application-specific tests:

```text
POST /api/v1/orders
GET /api/v1/orders/{id}
```

We should verify:

```text
HTTP status
response structure
database connectivity
Redis connectivity
SQS connectivity
authentication
authorization
external dependencies
```

---

# 15. Important — Don't Test Green Through the Production Service

If:

```text
order-service-active
```

points to Blue, sending:

```bash
curl https://api.example.com/api/v1/orders
```

tests Blue.

It does **not** test Green.

We need a separate validation route.

For example:

```text
order-service-green-preview
```

which points specifically to:

```text
version: green
```

Architecture:

```text
Production URL
     │
     ▼
Active Service
     │
     ▼
Blue


Validation URL
     │
     ▼
Green Preview Service
     │
     ▼
Green
```

---

# 16. Preview Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-green-preview
  namespace: production

spec:
  selector:
    app: order-service
    version: green

  ports:
    - port: 80
      targetPort: 8080
```

Now:

```text
order-service-active
        │
        ▼
      BLUE


order-service-green-preview
        │
        ▼
      GREEN
```

---

# 17. Validate Green

Example:

```bash
kubectl run test-client \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl http://order-service-green-preview/api/v1/orders
```

This ensures we are testing:

```text
Green
```

rather than accidentally testing Blue.

---

# 18. Database Compatibility Is Critical

This is one of the biggest Blue-Green deployment challenges.

Suppose:

```text
Blue = application v1
Green = application v2
```

And v2 expects:

```text
new_column
```

If we change the database immediately:

```sql
ALTER TABLE orders ADD COLUMN new_column ...
```

we must ensure v1 can still operate.

Otherwise:

```text
Green works
Blue breaks
```

Then rollback becomes dangerous.

---

# 19. Expand-and-Contract Migration

Use backward-compatible database migrations.

### Phase 1 — Expand

Add new structure without breaking old application.

```text
DB
├── old_column
└── new_column
```

Blue still works.

---

### Phase 2 — Deploy Green

Green understands the new structure.

```text
Blue → old behavior
Green → new behavior
```

Both remain compatible.

---

### Phase 3 — Migrate Data

Backfill if required.

---

### Phase 4 — Contract

Only after old application is no longer needed:

```text
remove old_column
```

This is much safer than:

```text
change DB
→ immediately deploy application
→ hope rollback works
```

---

# 20. Blue-Green Does Not Mean Two Databases

Normally:

```text
Blue application ──┐
                   ├──► RDS PostgreSQL
Green application ─┘
```

Both versions may use the same database.

Therefore:

> The application versions must be database-compatible.

Creating a second production database just for Blue-Green introduces:

* data synchronization
* replication complexity
* consistency problems
* higher cost
* additional failure modes

Database strategy must be designed separately.

---

# 21. Switching Traffic

Once Green passes validation:

```text
Before:

Service
  │
  ▼
Blue
```

Change selector:

```yaml
selector:
  app: order-service
  version: green
```

Now:

```text
Service
  │
  ▼
Green
```

Traffic switches.

---

# 22. What Happens to Existing Connections?

Traffic switching does not magically terminate every connection instantly.

Existing connections may:

* complete
* drain
* timeout
* be reused depending on protocol/load balancer/client behavior

Therefore we still need:

```text
graceful shutdown
readiness
terminationGracePeriodSeconds
connection draining
```

This is why Blue-Green is not simply:

> Change selector and forget everything else.

---

# 23. Keep Blue During Observation

After switching:

```text
Green → production
Blue  → standby
```

Do **not** immediately delete Blue.

Monitor:

```text
HTTP 5xx
HTTP latency
CPU
Memory
Pod restarts
DB errors
Redis errors
SQS errors
business errors
```

For our platform:

```text
Grafana
   │
   ├── Prometheus
   └── CloudWatch

Kibana
   │
   └── Elasticsearch

Jaeger
   │
   └── OpenTelemetry
```

---

# 24. Production Validation

Example:

```text
Green v1.6.0
     │
     ├── Error rate
     ├── p95 latency
     ├── p99 latency
     ├── DB connections
     ├── Redis latency
     ├── SQS failures
     ├── application exceptions
     └── business transactions
```

For an order-service deployment, don't only check:

```text
HTTP 200
```

Also verify:

```text
Order created
      ↓
Database record exists
      ↓
Event published
      ↓
Downstream consumer receives event
```

---

# 25. Rollback

Suppose:

```text
Green v1.6.0
```

has a serious problem.

Switch the Service back:

```text
version: blue
```

Traffic:

```text
Users
  │
  ▼
Service
  │
  ▼
Blue v1.5.0
```

This is extremely fast compared with rebuilding the previous version.

---

# 26. But Rollback Has a Major Limitation

Application rollback:

```text
v2 → v1
```

is easy.

Database rollback:

```text
DB schema v2 → DB schema v1
```

may be difficult or dangerous.

Therefore:

> **Blue-Green provides fast application rollback, not automatic database rollback.**

This is why backward-compatible database migrations are so important.

---

# 27. Helm Structure

Our chart can support Blue-Green configuration.

Example:

```text
helm/order-service/

├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-qa.yaml
├── values-prod.yaml
│
└── templates/
    ├── deployment-blue.yaml
    ├── deployment-green.yaml
    ├── service-active.yaml
    ├── service-preview.yaml
    ├── serviceaccount.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── pdb.yaml
```

However, don't blindly duplicate large manifests.

A better Helm design can parameterize:

```yaml
deployment:
  color: blue
```

or use an appropriate rollout controller.

---

# 28. Argo Rollouts

For production, we should understand **Argo Rollouts**.

It extends Kubernetes deployment capabilities for advanced strategies:

```text
Rolling
Blue-Green
Canary
```

Instead of manually managing:

```text
blue Deployment
green Deployment
Service selector
```

we can use a Rollout resource.

Conceptually:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  strategy:
    blueGreen:
      activeService: order-service-active
      previewService: order-service-preview
      autoPromotionEnabled: false
```

The exact fields and behavior should be validated against the Argo Rollouts version selected for the project.

---

# 29. Why `autoPromotionEnabled: false`?

Production deployments should not necessarily switch traffic immediately after Green becomes healthy.

We may want:

```text
Green deployed
       ↓
Smoke tests
       ↓
Automated health checks
       ↓
Engineer approval
       ↓
Promotion
       ↓
Green becomes active
```

This gives us a controlled production gate.

---

# 30. GitOps Architecture with Argo Rollouts

```text
Developer
    │
    ▼
Application Repository
    │
    ▼
GitHub Actions
    │
    ├── Test
    ├── Security
    ├── Build
    └── Push
          │
          ▼
         ECR
          │
          ▼
      GitOps PR
          │
          ▼
       Approval
          │
          ▼
        ArgoCD
          │
          ▼
    Argo Rollouts
          │
       ┌──┴──┐
       ▼     ▼
     Blue   Green
       │     │
       └──┬──┘
          ▼
       Service
          │
          ▼
       Users
```

---

# 31. Blue-Green + ArgoCD Responsibilities

Keep responsibilities separate.

### GitHub Actions

```text
Build
Test
Scan
Publish image
```

### GitOps

```text
Desired application version
deployment configuration
Rollout strategy
```

### ArgoCD

```text
Reconcile Git
Apply desired state
```

### Argo Rollouts

```text
Deployment strategy
Promotion
Rollback
Traffic management
```

### Kubernetes

```text
Run workloads
```

---

# 32. Blue-Green + ALB

Blue-Green doesn't necessarily require changing the AWS ALB.

A common design is:

```text
ALB
 │
 ▼
Ingress
 │
 ▼
Stable Kubernetes Service
 │
 ▼
Blue OR Green
```

This is attractive because the external traffic entry point remains stable.

---

# 33. Alternative — ALB-Level Switching

Another architecture could be:

```text
ALB
 │
 ├── Target Group Blue
 │
 └── Target Group Green
```

Then traffic can be controlled at the ALB level.

This can be useful for more sophisticated traffic management, but adds complexity.

For our learning project, Kubernetes Service-level switching is easier to understand first.

---

# 34. Capacity Planning

Blue-Green requires additional capacity.

Suppose:

```text
Blue:
3 Pods

Green:
3 Pods
```

During deployment:

```text
6 Pods
```

instead of:

```text
3 Pods
```

Therefore:

```text
Blue-Green
    ↓
temporary capacity increase
    ↓
additional node utilization
    ↓
additional cost
```

This becomes important for large services.

---

# 35. EKS Autoscaling Interaction

Suppose Green requires:

```text
3 Pods
```

but the cluster has no capacity.

Then:

```text
Green Pods
   ↓
Pending
   ↓
Karpenter
   ↓
new node
   ↓
Pods scheduled
```

So Blue-Green interacts directly with:

```text
HPA
Karpenter
Node Pools
Pod requests
Topology constraints
```

---

# 36. Don't Forget System Workloads

When scaling EKS nodes, remember nodes also run:

```text
kube-proxy
CoreDNS
VPC CNI
Fluent Bit
Node Exporter
other DaemonSets
```

Therefore:

```text
Node capacity
≠
100% application capacity
```

This must be included in capacity planning.

---

# 37. PDB

We can use a PodDisruptionBudget:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

PDB protects against voluntary disruptions such as:

```text
node drain
cluster maintenance
node replacement
```

But:

> PDB is not a Blue-Green mechanism.

It protects availability during disruptions.

---

# 38. Multi-AZ

Our production EKS cluster should distribute Pods across AZs.

Example:

```text
AZ-A              AZ-B              AZ-C

Blue Pod          Blue Pod          Blue Pod
Green Pod         Green Pod         Green Pod
```

Use:

```text
topologySpreadConstraints
```

and/or:

```text
podAntiAffinity
```

depending on the workload.

---

# 39. Blue-Green for All 21 Services?

Not necessarily.

This is an important production decision.

We have:

```text
21 microservices
```

We shouldn't automatically create:

```text
42 permanent production deployments
```

just because Blue-Green exists.

Instead, use the deployment strategy appropriate to each service.

For example:

```text
order-service       → Blue-Green
payment-service     → Canary
catalog-service     → Rolling
notification-service → Rolling
```

Strategy depends on:

* risk
* traffic
* deployment frequency
* compatibility
* infrastructure cost
* business criticality

---

# 40. High-Risk Services

Blue-Green can be particularly useful for services where rollback speed is valuable.

For example:

```text
payment-service
order-service
auth-service
```

But payment systems also have external side effects.

A deployment rollback does not undo:

```text
payment captured
email sent
order submitted
shipment created
```

Therefore business idempotency remains essential.

---

# 41. Blue-Green and Messaging

Suppose:

```text
Blue → SQS consumer v1
Green → SQS consumer v2
```

If both consume the same queue during validation, Green may process real messages.

That may be unacceptable.

Therefore for message consumers, Blue-Green requires additional design.

Possible approaches:

```text
Production queue
       │
       ▼
Blue consumer
```

Green can be tested using:

```text
test queue
```

or controlled test messages.

For Kafka:

```text
production topic
```

requires careful consumer-group and offset planning.

---

# 42. Blue-Green and External APIs

Suppose Green changes:

```text
Payment API contract
```

but external consumers still expect:

```text
v1
```

A traffic switch does not solve API compatibility.

Use:

```text
backward-compatible API changes
```

For example:

```text
v1 response
+
new optional field
```

rather than immediately removing:

```text
old required field
```

---

# 43. Blue-Green and Cache

Suppose:

```text
Blue → Redis
Green → Redis
```

Both versions may use the same cache.

Potential problems:

```text
old key format
new key format
serialization changes
cache invalidation
```

Use versioned key strategies where appropriate:

```text
order:v1:{id}
order:v2:{id}
```

and plan invalidation carefully.

---

# 44. Blue-Green and Secrets

Both environments may need:

```text
DB credentials
Redis credentials
AWS permissions
API credentials
```

Our architecture:

```text
Secrets Manager
      │
      ▼
External Secrets Operator
      │
      ▼
Kubernetes Secret
      │
 ┌────┴─────┐
 ▼          ▼
Blue       Green
```

Never put secret values directly into:

```text
GitOps
Helm values
Docker image
GitHub repository
```

---

# 45. Observability During Promotion

A production promotion should look like:

```text
Green deployed
     │
     ▼
Metrics
     │
     ├── Error rate
     ├── Latency
     ├── Saturation
     └── Availability
     │
     ▼
Logs
     │
     ▼
Traces
     │
     ▼
Business validation
     │
     ▼
Promotion
```

Our tools:

```text
Prometheus
Grafana
Alertmanager
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
Jaeger
```

all contribute.

---

# 46. Example Promotion Decision

Suppose Green has:

```text
HTTP 5xx       0.1%
p95 latency    180 ms
p99 latency    350 ms
Pod restarts   0
DB errors      0
SQS failures   0
```

Blue currently:

```text
HTTP 5xx       0.1%
p95 latency    175 ms
```

Green looks healthy.

Promotion can proceed.

But if:

```text
Green HTTP 5xx = 8%
```

we stop.

```text
DO NOT PROMOTE
```

---

# 47. Automated Promotion

A more advanced system can automatically evaluate:

```text
Prometheus metrics
```

before promotion.

Example:

```text
Green
 │
 ▼
Analysis
 │
 ├── error rate < threshold
 ├── latency < threshold
 ├── availability > threshold
 └── no critical alerts
       │
       ▼
    Promote
```

Argo Rollouts can support automated analysis depending on the chosen configuration.

---

# 48. What If Green Is Broken?

Example:

```text
Green v1.6.0
```

starts producing:

```text
NullPointerException
```

Kibana:

```text
ERROR order creation failed
```

Jaeger:

```text
POST /orders
   ↓
500
```

Prometheus:

```text
5xx ↑
latency ↑
```

Alertmanager:

```text
CRITICAL
```

We do not promote Green.

Blue continues serving users.

This is one of the biggest advantages of Blue-Green.

---

# 49. What If Green Is Promoted and Then Breaks?

Example:

```text
14:00 → Green deployed
14:05 → validation passed
14:10 → Green promoted
14:15 → production errors detected
```

Rollback:

```text
Green
  ↓
deactivate
  ↓
Blue
  ↓
production
```

Then investigate Green.

---

# 50. Blue-Green Rollback Is Not a Substitute for Root-Cause Analysis

Rollback restores service.

It doesn't fix the defect.

Incident flow:

```text
Production failure
       │
       ▼
Rollback
       │
       ▼
Service restored
       │
       ▼
Investigate
       │
       ├── Logs
       ├── Metrics
       ├── Traces
       ├── Git diff
       └── Database changes
       │
       ▼
Fix
       │
       ▼
New release
```

---

# 51. Production Troubleshooting

## Green Pods Pending

```bash
kubectl get pods -n production
```

Check:

```bash
kubectl describe pod <pod> -n production
```

Possible reasons:

```text
insufficient CPU
insufficient memory
node affinity
taints
AZ constraints
PVC issues
```

---

# 52. Green Pods CrashLoopBackOff

```bash
kubectl logs <pod> -n production --previous
```

Check:

```text
environment variables
secrets
database connection
Redis
application configuration
startup failures
```

---

# 53. Green Healthy but Preview Doesn't Work

Check:

```bash
kubectl get svc -n production
kubectl get endpoints -n production
```

Common cause:

```text
Service selector
        ≠
Pod labels
```

Example:

Service:

```yaml
version: green
```

Pod:

```yaml
version: blue
```

Result:

```text
No endpoints
```

---

# 54. Production Still Goes to Blue

Check:

```bash
kubectl get svc order-service-active -n production -o yaml
```

Verify:

```yaml
selector:
  app: order-service
  version: green
```

Also verify endpoints:

```bash
kubectl get endpoints order-service-active -n production
```

---

# 55. Green Works Internally but Production Fails

Possible layers:

```text
Application
   ↓
Service
   ↓
Ingress
   ↓
ALB
   ↓
WAF
   ↓
CloudFront
   ↓
DNS
```

Don't immediately blame Kubernetes.

Check the entire traffic path.

---

# 56. Blue-Green Deployment Checklist

## Before Deployment

```text
☐ CI passed
☐ Unit tests passed
☐ SonarQube passed
☐ Secret scan passed
☐ Dependency scan passed
☐ Trivy passed
☐ SBOM generated
☐ Image pushed to ECR
☐ Image immutable
☐ GitOps PR reviewed
☐ Database migration reviewed
```

---

## Green Deployment

```text
☐ Green Pods scheduled
☐ Readiness successful
☐ Liveness successful
☐ No CrashLoopBackOff
☐ No ImagePullBackOff
☐ Preview Service works
☐ Smoke tests pass
☐ Logs clean
☐ Metrics healthy
☐ Traces healthy
☐ Dependencies healthy
```

---

## Promotion

```text
☐ Production health baseline known
☐ Green metrics acceptable
☐ Error rate acceptable
☐ Latency acceptable
☐ Business transaction validated
☐ Approval received
☐ Traffic switched
```

---

## After Promotion

```text
☐ Monitor Green
☐ Keep Blue available
☐ Check alerts
☐ Check logs
☐ Check traces
☐ Check business metrics
☐ Confirm stability
```

---

# 57. Blue-Green vs Rolling vs Canary

| Strategy   | Old + New         | Traffic Control | Rollback  | Cost   |
| ---------- | ----------------- | --------------- | --------- | ------ |
| Rolling    | Gradual           | Limited         | Moderate  | Low    |
| Blue-Green | Full environments | Fast switch     | Very fast | High   |
| Canary     | Gradual versions  | Fine-grained    | Fast      | Medium |
| Recreate   | No coexistence    | Full cutover    | Slow      | Low    |

---

# 58. When Should We Use Blue-Green?

Good candidate:

```text
High-risk deployment
+
fast rollback required
+
enough cluster capacity
+
compatible database changes
```

Less attractive when:

```text
very large workloads
+
doubling capacity is expensive
+
deployment changes stateful infrastructure
```

---

# 59. Recommended Strategy for Our Project

We should not force one deployment strategy on all 21 services.

A realistic approach:

```text
Standard services
      │
      ▼
Rolling Deployment


High-risk services
      │
      ▼
Blue-Green


Very high traffic / controlled rollout
      │
      ▼
Canary
```

Example:

```text
auth-service          → Rolling / Canary
user-service          → Rolling
catalog-service       → Rolling
product-service       → Rolling
inventory-service     → Canary
pricing-service       → Canary
promotion-service     → Rolling
cart-service           → Rolling
order-service          → Blue-Green
payment-service       → Blue-Green / Canary
fraud-service         → Canary
shipping-service      → Rolling
delivery-service      → Rolling
notification-service  → Rolling
search-service        → Rolling
recommendation-service → Canary
review-service        → Rolling
tax-service            → Rolling
invoice-service        → Rolling
reporting-service      → Rolling
```

This is an **example strategy**, not a fixed requirement.

The actual strategy should be decided based on service criticality and traffic.

---

# 60. Production Architecture

```text
                         INTERNET
                            │
                            ▼
                         Route 53
                            │
                            ▼
                       CloudFront/WAF
                            │
                            ▼
                           ALB
                            │
                            ▼
                         Ingress
                            │
                            ▼
                ┌─────────────────────┐
                │ Active Service      │
                │ order-service       │
                └──────────┬──────────┘
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
               BLUE v1.5      GREEN v1.6
                 │               │
                 │               │
                 └───────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
             RDS       Redis       SQS
              │          │          │
              └──────────┼──────────┘
                         │
                    Observability
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
   Prometheus           ELK               OTel
       │                 │                 │
       ▼                 ▼                 ▼
    Grafana            Kibana            Jaeger
       │
       ▼
  Alertmanager
```

---

# 61. The Most Important Mental Model

Don't think:

> Blue-Green means two Deployments.

Think:

```text
Two application versions
        +
One controlled production entry point
        +
Independent validation path
        +
Fast traffic switch
        +
Fast application rollback
        +
Backward-compatible database changes
        +
Observability
        +
GitOps-controlled promotion
```

That is production-grade Blue-Green.

---

# 62. Final Flow in Our Project

The complete release path now becomes:

```text
Developer
    │
    ▼
Feature Branch
    │
    ▼
Pull Request
    │
    ▼
CI
    │
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Gitleaks
    ├── Trivy
    ├── SBOM
    └── Docker
         │
         ▼
        ECR
         │
         ▼
      GitOps PR
         │
         ▼
      Approval
         │
         ▼
       ArgoCD
         │
         ▼
    Green Deployment
         │
         ▼
   Preview Validation
         │
         ├── Metrics
         ├── Logs
         ├── Traces
         └── Smoke Tests
         │
         ▼
      Promotion
         │
         ▼
       Green
         │
         ▼
     Production
```

If something goes wrong:

```text
Green
  │
  ▼
Rollback
  │
  ▼
Blue
  │
  ▼
Production restored
```

---

# 63. Production Takeaway

The real value of Blue-Green is **not simply having two Pods sets**.

The real value is:

> **Deploy the new version completely, validate it without exposing users to it, switch production traffic deliberately, and retain the previous version long enough to recover quickly.**

For our project, this also demonstrates an important DevOps principle:

```text
CI builds the artifact
        ↓
GitOps defines the desired state
        ↓
ArgoCD reconciles it
        ↓
Kubernetes runs it
        ↓
Argo Rollouts controls promotion
        ↓
Observability proves whether it is healthy
        ↓
Rollback restores the previous application version
```

**Next chapter: `44-Canary-Deployment.md`** — we will build on this and cover gradual traffic shifting, 1% → 5% → 25% → 50% → 100% releases, Argo Rollouts, ALB traffic management, automated Prometheus analysis, rollback, and how Canary differs from Blue-Green in a real EKS production environment.
