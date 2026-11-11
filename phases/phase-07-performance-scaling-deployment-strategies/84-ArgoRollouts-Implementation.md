# 84 — Argo Rollouts Implementation

> Production implementation of progressive delivery for `order-service` using **Argo Rollouts + ArgoCD + EKS + Kubernetes Services + AWS ALB + Prometheus + GitOps + ECR + Jira Change Management**.

---

# 1. Objective

We have now covered:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
83 — Canary Deployment
```

Now we implement the controller that can automate progressive delivery:

```text
Argo Rollouts
```

Instead of relying only on a standard Kubernetes Deployment:

```text
Deployment
    ↓
RollingUpdate
```

we can use:

```text
Rollout
    ↓
Canary strategy
    ↓
Traffic management
    ↓
Prometheus analysis
    ↓
Promote OR Abort
```

Our target architecture is:

```text
GitHub
   ↓
GitHub Actions
   ↓
ECR
   ↓
GitOps
   ↓
ArgoCD
   ↓
Argo Rollouts
   ↓
Stable + Canary
   ↓
AWS ALB
   ↓
Production users
```

---

# 2. What Argo Rollouts Adds

Standard Kubernetes Deployment gives us:

```text
RollingUpdate
```

Argo Rollouts adds progressive delivery capabilities such as:

```text
Canary steps
Traffic weighting
Pause
Promotion
Abort
Analysis
Automated decisions
Rollback
```

Conceptually:

```text
                 Argo Rollouts
                      |
        +-------------+-------------+
        |                           |
      Stable                     Canary
       v1                           v2
        |                           |
        +-------------+-------------+
                      |
                Traffic Router
                      |
                     ALB
                      |
                   Users
```

---

# 3. ArgoCD vs Argo Rollouts

These two tools have different responsibilities.

### ArgoCD

Responsible for:

```text
Git
 ↓
Desired Kubernetes state
 ↓
Cluster
```

### Argo Rollouts

Responsible for:

```text
How an application release progresses
```

So:

```text
GitOps
   ↓
ArgoCD
   ↓
Rollout resource
   ↓
Argo Rollouts controller
   ↓
Progressive deployment
```

Don't confuse:

```text
ArgoCD
```

with:

```text
Argo Rollouts
```

---

# 4. Current Architecture

Our production environment:

```text
                       GitHub
                          |
                    GitHub Actions
                          |
                         ECR
                          |
                       GitOps
                          |
                        ArgoCD
                          |
                    Argo Rollouts
                          |
              +-----------+-----------+
              |                       |
           Stable                  Canary
             v1                       v2
              |                       |
              +-----------+-----------+
                          |
                    AWS ALB
                          |
                       Users
```

Monitoring:

```text
Stable ──────┐
             ├── Prometheus
Canary ──────┘
                 |
               Analysis
```

---

# 5. Install Argo Rollouts

We already have:

```text
argocd
```

installed in the cluster.

Now install:

```text
argo-rollouts
```

Create namespace:

```bash
kubectl create namespace argo-rollouts
```

For an initial hands-on environment, install the controller using the official Argo Rollouts installation method appropriate for the version you choose.

For example, the project provides a Kubernetes manifest installation.

After installation:

```bash
kubectl get pods -n argo-rollouts
```

You should see the controller Pod.

Example:

```text
NAME                              READY
argo-rollouts-xxxxxxxxxx          1/1
```

Before production installation, pin and review the exact Argo Rollouts version rather than using an unpinned moving manifest.

---

# 6. Verify Controller

Check:

```bash
kubectl get deployment \
  -n argo-rollouts
```

Then:

```bash
kubectl get pods \
  -n argo-rollouts
```

Check logs:

```bash
kubectl logs \
  -n argo-rollouts \
  deployment/argo-rollouts
```

You want the controller to be healthy before creating Rollouts.

---

# 7. Install the Argo Rollouts CLI

The CLI makes troubleshooting and operations easier.

Verify:

```bash
kubectl argo rollouts version
```

You should see client/server version information.

The exact installation method depends on your workstation operating system.

Keep the CLI version compatible with the controller version used by the platform team.

---

# 8. Rollout vs Deployment

Previously:

```yaml
kind: Deployment
```

Now:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
```

For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: order-service

spec:
  replicas: 4
```

The rest of the Pod template still looks familiar.

---

# 9. Rollout Structure

A Rollout contains:

```text
Rollout
├── replicas
├── selector
├── template
└── strategy
      └── canary
            ├── steps
            ├── traffic routing
            └── analysis
```

---

# 10. Stable and Canary Services

For our implementation, create two Services.

### Stable

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service-stable

spec:
  selector:
    app.kubernetes.io/name: order-service

  ports:
    - port: 80
      targetPort: 8080
```

### Canary

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service-canary

spec:
  selector:
    app.kubernetes.io/name: order-service

  ports:
    - port: 80
      targetPort: 8080
```

Notice something important:

Both Services select the same application labels.

Argo Rollouts manages which ReplicaSet receives the stable/canary role.

---

# 11. Basic Rollout

Start with:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: order-service

spec:

  replicas: 4

  revisionHistoryLimit: 5

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service

  template:

    metadata:
      labels:
        app.kubernetes.io/name: order-service

    spec:

      containers:

        - name: order-service

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:NEW_DIGEST

          ports:
            - name: http
              containerPort: 8080

          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"

            limits:
              cpu: "1000m"
              memory: "1Gi"

          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http

            periodSeconds: 5
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http

            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http

            periodSeconds: 10
```

---

# 12. Canary Strategy

Now add:

```yaml
strategy:
  canary:
```

Example:

```yaml
strategy:

  canary:

    stableService: order-service-stable

    canaryService: order-service-canary

    steps:

      - setWeight: 5

      - pause:
          duration: 5m

      - setWeight: 10

      - pause:
          duration: 10m

      - setWeight: 25

      - pause:
          duration: 10m

      - setWeight: 50

      - pause:
          duration: 15m

      - setWeight: 100
```

This means:

```text
5%
 ↓
10%
 ↓
25%
 ↓
50%
 ↓
100%
```

---

# 13. But Where Does the Traffic Split Happen?

This is critical.

The Rollout itself doesn't magically control Internet traffic.

We need a traffic router.

Our architecture:

```text
Argo Rollouts
      |
      ↓
Traffic Router
      |
      ↓
AWS ALB
      |
 +----+----+
 |         |
Stable   Canary
```

For AWS ALB, configure the Kubernetes/ALB integration supported by the AWS Load Balancer Controller version used by your cluster.

---

# 14. ALB Traffic Routing Concept

The desired model is:

```text
ALB
 |
 +------------------+
 |                  |
Stable TG        Canary TG
 |
v1 Pods          v2 Pods
```

Traffic weights:

```text
Stable = 95
Canary = 5
```

Then:

```text
Stable = 90
Canary = 10
```

and so on.

The exact ALB annotations/action structure is version-dependent, so use the AWS Load Balancer Controller documentation corresponding to the version deployed in your EKS cluster.

---

# 15. Rollout Traffic Routing

The Rollout needs to know:

```text
stableService
canaryService
```

and the traffic-management mechanism.

Conceptually:

```yaml
strategy:
  canary:

    stableService: order-service-stable

    canaryService: order-service-canary

    trafficRouting:
      alb:
        ingress: order-service
        servicePort: 80
```

The exact field names supported by the chosen Argo Rollouts version should be verified against that version's ALB integration.

The important architecture is:

```text
Rollout
   ↓
Argo Rollouts
   ↓
ALB traffic configuration
   ↓
Stable / Canary
```

---

# 16. Production Helm Structure

Our application chart can now contain:

```text
helm/order-service/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── rollout.yaml
    ├── service-stable.yaml
    ├── service-canary.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── pdb.yaml
```

This is different from the simple Deployment implementation.

---

# 17. `values.yaml`

Example:

```yaml
replicaCount: 4

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:222222..."

rollout:

  enabled: true

  strategy: canary

  steps:

    - setWeight: 5
    - pause:
        duration: 5m

    - setWeight: 10
    - pause:
        duration: 10m

    - setWeight: 25
    - pause:
        duration: 10m

    - setWeight: 50
    - pause:
        duration: 15m

    - setWeight: 100
```

---

# 18. Why Keep This in GitOps?

Production rollout behavior is configuration.

For example:

```text
Git commit A
5 → 10 → 25 → 50 → 100
```

Later:

```text
Git commit B
5 → 10 → 25 → 50 → 75 → 100
```

Now you can answer:

```text
Who changed the rollout?
Why?
When?
Which Jira change?
```

---

# 19. AnalysisTemplate

Now we introduce automated validation.

Create:

```text
analysis-template.yaml
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate

metadata:
  name: order-service-canary-analysis

spec:

  metrics:

    - name: canary-error-rate

      interval: 1m

      count: 5

      successCondition: result[0] < 0.01

      failureLimit: 2

      provider:

        prometheus:

          address: http://prometheus.monitoring.svc:9090

          query: |
            sum(
              rate(
                http_server_requests_seconds_count{
                  service="order-service",
                  rollout="canary",
                  status=~"5.."
                }[5m]
              )
            )
            /
            sum(
              rate(
                http_server_requests_seconds_count{
                  service="order-service",
                  rollout="canary"
                }[5m]
              )
            )
```

This is an example.

Your actual metric names and labels must match the metrics exposed by your application.

---

# 20. What This Query Does

The query conceptually calculates:

```text
5xx requests
-----------------
all requests
```

For example:

```text
10 errors
1000 requests
```

gives:

```text
0.01
```

or:

```text
1%
```

So:

```yaml
successCondition: result[0] < 0.01
```

means:

```text
Error rate must be below 1%
```

Again, production thresholds should be based on the service's actual SLO/error budget.

---

# 21. Why `count: 5`?

If we check only once:

```text
1 measurement
```

we could make a bad decision because of temporary noise.

Instead:

```text
5 measurements
```

over a period of time give us more confidence.

For example:

```text
12:00 → 0.2%
12:01 → 0.3%
12:02 → 0.2%
12:03 → 0.4%
12:04 → 0.2%
```

This looks stable.

But:

```text
12:00 → 5%
12:01 → 7%
12:02 → 6%
```

is a strong warning.

---

# 22. Failure Limit

We might define:

```yaml
failureLimit: 2
```

Meaning the analysis can tolerate the configured number of failed measurements before being considered failed.

This should be designed carefully because:

```text
failureLimit
+
measurement interval
+
traffic volume
```

determine how quickly a bad release is detected.

---

# 23. Attach Analysis to Canary

Now the Rollout can reference the AnalysisTemplate.

Conceptually:

```yaml
strategy:

  canary:

    steps:

      - setWeight: 5

      - pause:
          duration: 5m

      - analysis:
          templates:
            - templateName: order-service-canary-analysis

      - setWeight: 10

      - pause:
          duration: 10m

      - analysis:
          templates:
            - templateName: order-service-canary-analysis

      - setWeight: 25

      - pause:
          duration: 10m
```

Now the release becomes:

```text
5%
 ↓
Analyze
 ↓
PASS?
 ↓
10%
 ↓
Analyze
 ↓
PASS?
 ↓
25%
```

---

# 24. What Happens on Analysis Failure?

Suppose:

```text
Canary = 10%
```

Prometheus reports:

```text
5xx = 6%
```

Analysis:

```text
FAIL
```

Argo Rollouts can abort the rollout according to its configured strategy.

Conceptually:

```text
Canary
  |
  v
10%
  |
  v
Prometheus
  |
  v
FAIL
  |
  v
Abort
  |
  v
Stable = 100%
Canary = 0%
```

---

# 25. Manual Promotion

Production teams don't always want fully automatic deployment.

We can pause:

```yaml
- pause: {}
```

This means:

```text
Deploy Canary
      ↓
5%
      ↓
STOP
```

An engineer reviews:

```text
Grafana
Prometheus
Logs
Traces
Jira
Application dashboards
```

Then promotes.

---

# 26. Check Rollout

Use:

```bash
kubectl argo rollouts get rollout order-service \
  -n ecommerce-prod
```

You may see something conceptually like:

```text
Name:            order-service
Status:          Progressing
Strategy:        Canary
Stable RS:       order-service-xxxxx
Canary RS:       order-service-yyyyy

Weights:
5%
```

The exact CLI output depends on the Argo Rollouts version.

---

# 27. Watch Rollout

```bash
kubectl argo rollouts get rollout order-service \
  -n ecommerce-prod \
  --watch
```

This gives you a live view of the rollout.

---

# 28. Promote

If the rollout is paused:

```bash
kubectl argo rollouts promote order-service \
  -n ecommerce-prod
```

This tells the Rollout controller to continue to the next stage.

---

# 29. Abort

If the canary is unhealthy:

```bash
kubectl argo rollouts abort order-service \
  -n ecommerce-prod
```

Then check:

```bash
kubectl argo rollouts get rollout order-service \
  -n ecommerce-prod
```

Verify traffic returns to the stable version according to the configured traffic-routing behavior.

---

# 30. Restart Rollout

If you need to restart the rollout:

```bash
kubectl argo rollouts restart order-service \
  -n ecommerce-prod
```

Use operational commands carefully in GitOps-managed environments.

The desired state remains in Git.

---

# 31. GitOps vs Emergency Commands

Normal process:

```text
Git
 ↓
ArgoCD
 ↓
Argo Rollouts
```

Emergency operation:

```text
kubectl argo rollouts abort
```

can be appropriate during an incident.

But afterward:

```text
Live state
    ↓
Git desired state
```

must be reconciled.

Otherwise you can create confusing state between:

```text
Git
ArgoCD
Rollout
Kubernetes
```

---

# 32. ArgoCD Application

Our ArgoCD Application still points to the GitOps configuration.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-prod
  namespace: argocd

spec:

  project: production

  sources:

    - repoURL: https://github.com/company-ecommerce/order-service.git
      targetRevision: v1.4.0
      path: helm/order-service

      helm:
        valueFiles:
          - $values/environments/prod/order-service/values.yaml

    - repoURL: https://github.com/company-ecommerce/gitops.git
      targetRevision: main
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce-prod
```

ArgoCD manages:

```text
Rollout
Services
Ingress
ConfigMap
ServiceAccount
HPA
PDB
```

Argo Rollouts then manages the progressive behavior of the Rollout.

---

# 33. Controller Responsibilities

Think of the controllers as layers.

### ArgoCD

```text
Git → Kubernetes desired state
```

### Argo Rollouts

```text
Rollout resource → progressive release state
```

### AWS Load Balancer Controller

```text
Kubernetes Ingress/Service configuration
→ AWS ALB
```

### Prometheus

```text
Application
→ metrics
```

### Grafana

```text
Metrics
→ visualization
```

Together:

```text
Git
 ↓
ArgoCD
 ↓
Argo Rollouts
 ↓
AWS LB Controller
 ↓
ALB
 ↓
Users

Prometheus
 ↓
Analysis
 ↓
Argo Rollouts
```

---

# 34. Complete Canary Flow

```text
                   GitHub
                      |
                 GitHub Actions
                      |
                Build/Test/Scan
                      |
                     ECR
                      |
                Image Digest
                      |
                  GitOps PR
                      |
                 Jira Change
                      |
                  Approval
                      |
                   ArgoCD
                      |
                Argo Rollouts
                      |
               Deploy Canary
                      |
                     5%
                      |
                 Prometheus
                      |
                AnalysisTemplate
                  /         \
                PASS        FAIL
                 |            |
                10%         Abort
                 |            |
            Analyze        100% Stable
                 |
                25%
                 |
             Analyze
                 |
                50%
                 |
             Analyze
                 |
               100%
                 |
             Promotion
```

---

# 35. Production Deployment Example

Suppose:

```text
Current:
v1
```

New release:

```text
v2
Git SHA = 8f31a2c
Digest = sha256:222222
```

### Stage 1

```text
Stable = 95%
Canary = 5%
```

Metrics:

```text
5xx = 0.2%
P95 = 125ms
```

PASS.

---

### Stage 2

```text
Stable = 90%
Canary = 10%
```

Metrics:

```text
5xx = 0.3%
P95 = 128ms
```

PASS.

---

### Stage 3

```text
Stable = 75%
Canary = 25%
```

Metrics:

```text
5xx = 0.4%
P95 = 135ms
```

PASS.

---

### Stage 4

```text
Stable = 50%
Canary = 50%
```

Metrics:

```text
5xx = 0.3%
P95 = 132ms
```

PASS.

---

### Final

```text
Stable = 0%
Canary = 100%
```

Promotion complete.

---

# 36. Failed Canary Example

New version:

```text
v2
```

At:

```text
5%
```

metrics:

```text
5xx = 0.2%
```

PASS.

At:

```text
10%
```

metrics:

```text
5xx = 0.4%
```

PASS.

At:

```text
25%
```

metrics:

```text
5xx = 6.2%
```

FAIL.

Argo Rollouts:

```text
Abort
```

Traffic:

```text
Stable = 100%
Canary = 0%
```

The release does not proceed to:

```text
50%
100%
```

---

# 37. What If Prometheus Is Down?

This is an important production scenario.

Suppose:

```text
Canary
   ↓
Analysis
   ↓
Prometheus unavailable
```

You don't want the system blindly assuming:

```text
PASS
```

Depending on the configured analysis behavior, the analysis should fail/error rather than silently treat missing observability as healthy.

This gives an important production principle:

> A release controller should not promote an application when its release-validation signal is unavailable.

The exact behavior should be tested in your chosen Argo Rollouts version.

---

# 38. What If Canary Has Very Low Traffic?

Suppose:

```text
5% traffic
```

but the application receives:

```text
10 requests/minute
```

Then you might see:

```text
0 errors
```

but only because there wasn't enough traffic.

This is why analysis should consider:

```text
Traffic volume
Observation window
Minimum sample count
```

A zero-error result from five requests isn't strong evidence.

---

# 39. Business Metrics

For `order-service`, technical metrics aren't enough.

Suppose:

```text
HTTP 200 = 99.9%
```

but:

```text
Order creation success = 80%
```

Then something is wrong.

You could expose application/business metrics such as:

```text
orders_created_total
orders_failed_total
order_processing_duration_seconds
payment_failure_total
inventory_reservation_failure_total
```

Then analyze them during the canary.

---

# 40. Canary and Dependencies

Remember the request path:

```text
Client
 ↓
ALB
 ↓
Order Service
 ↓
Inventory
 ↓
Payment
 ↓
Database
```

If Canary increases calls to Payment:

```text
Payment load ↑
```

Even if:

```text
Order Service CPU = normal
```

the release may still be harmful.

Therefore monitor:

```text
Canary
+
Dependencies
```

not just the Canary Pod.

---

# 41. Database Migration Rules

Argo Rollouts does not solve incompatible database migrations.

Continue using:

```text
Expand
 ↓
Deploy
 ↓
Migrate
 ↓
Verify
 ↓
Contract
```

Avoid:

```text
Drop old schema
 ↓
Deploy canary
```

when stable Pods still depend on that schema.

---

# 42. Image Immutability

Our Rollout should use:

```text
image digest
```

rather than:

```text
latest
```

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:222222...
```

This guarantees:

```text
Canary
 ↓
exact image
```

and makes the release traceable to:

```text
Git commit
ECR image
digest
GitOps commit
Jira change
```

---

# 43. Release Traceability

Our complete chain is now:

```text
CHG-10454
    ↓
GitHub PR
    ↓
Git commit
    ↓
Docker image
    ↓
ECR digest
    ↓
GitOps commit
    ↓
ArgoCD
    ↓
Argo Rollout
    ↓
Canary
    ↓
Prometheus analysis
    ↓
Production
```

If there is an incident later, you can trace exactly which artifact was deployed.

---

# 44. Security

Argo Rollouts needs appropriate Kubernetes permissions.

Don't give every engineer:

```text
cluster-admin
```

Use:

```text
RBAC
```

for:

```text
Developer
Release Engineer
SRE
Platform Team
```

For example, developers may be allowed to:

```text
View Rollouts
View Pods
View Events
```

while release/SRE roles may additionally:

```text
Promote
Abort
```

Production permissions should be deliberately scoped.

---

# 45. High Availability

The Rollouts controller is part of the deployment platform.

Therefore treat it as production infrastructure.

Consider:

```text
Controller availability
Resource requests
Pod disruption
Monitoring
Version management
Upgrade strategy
```

If the controller becomes unavailable:

```text
Existing workloads
```

don't automatically disappear.

But progressive rollout control may stop or become unavailable.

Therefore the controller itself needs operational monitoring.

---

# 46. Monitor Argo Rollouts

Monitor:

```text
Rollout status
Controller health
Analysis failures
Aborted rollouts
Stalled rollouts
Promotion duration
Rollback frequency
```

Useful operational questions:

```text
How many releases are currently progressing?

How many are paused?

How many failed?

How long do rollouts normally take?

How often do releases abort?
```

---

# 47. Production Failure Scenarios

## Canary Pod won't start

```text
ImagePullBackOff
```

Check:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

---

## Canary readiness fails

```text
Ready: False
```

Check:

```bash
kubectl logs <pod> -n ecommerce-prod
```

and:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

---

## ALB doesn't route traffic

Investigate:

```text
Ingress
Target groups
Target health
AWS Load Balancer Controller
Security groups
Subnets
Service
Endpoints
```

---

## Prometheus analysis fails

Check:

```text
Prometheus availability
Metric name
Labels
Query
Time window
Traffic volume
Service discovery
```

---

## Rollout stuck

Check:

```bash
kubectl argo rollouts get rollout order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl describe rollout order-service \
  -n ecommerce-prod
```

And:

```bash
kubectl get events \
  -n ecommerce-prod \
  --sort-by=.lastTimestamp
```

---

# 48. Rollout Troubleshooting Flow

```text
Rollout stuck
     |
     v
Rollout status?
     |
     +---- Paused
     |       |
     |    Promotion?
     |
     +---- Degraded
     |       |
     |    Pod/health?
     |
     +---- Analysis failed
     |       |
     |    Prometheus?
     |
     +---- Traffic problem
             |
          ALB/Service
```

---

# 49. Production Checklist

## Platform

```text
[ ] Argo Rollouts installed
[ ] Controller healthy
[ ] CLI available
[ ] Version pinned
[ ] RBAC configured
[ ] Monitoring configured
```

## Application

```text
[ ] Rollout manifest
[ ] Stable Service
[ ] Canary Service
[ ] Readiness probe
[ ] Liveness probe
[ ] Startup probe
[ ] Resource requests
[ ] Resource limits
[ ] PDB
```

## Traffic

```text
[ ] ALB configured
[ ] Stable target healthy
[ ] Canary target healthy
[ ] Traffic routing tested
[ ] 5% routing tested
[ ] Rollback tested
```

## Analysis

```text
[ ] Prometheus available
[ ] Queries tested
[ ] Metrics labels correct
[ ] Thresholds based on SLO
[ ] Observation windows defined
[ ] Minimum traffic considered
[ ] Failure behavior tested
```

## Change Management

```text
[ ] Jira change approved
[ ] Release artifact recorded
[ ] Image digest recorded
[ ] Rollout strategy recorded
[ ] Rollback plan recorded
[ ] Monitoring plan recorded
```

---

# 50. Final Architecture

```text
                         GitHub
                            |
                     GitHub Actions
                            |
                +-----------+-----------+
                |                       |
             Testing                 Security
                |                 Sonar/Trivy
                +-----------+-----------+
                            |
                           ECR
                            |
                      Image Digest
                            |
                       GitOps PR
                            |
                      Jira Change
                            |
                        Approval
                            |
                          ArgoCD
                            |
                     Argo Rollouts
                            |
                +-----------+-----------+
                |                       |
             Stable                  Canary
               v1                      v2
                |                       |
                +-----------+-----------+
                            |
                       AWS ALB
                            |
                          Users
                            |
                      Application
                       Metrics
                            |
                       Prometheus
                            |
                    AnalysisTemplate
                            |
                  +---------+---------+
                  |                   |
                PASS                 FAIL
                  |                   |
             Increase             Abort
              traffic                |
                  |                   |
               100%                Stable
                  |
              Promotion
```

---

# 51. The Four Deployment Strategies We Now Understand

Our deployment chapter series is now:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
83 — Canary Deployment
84 — Argo Rollouts
```

The practical progression is:

```text
Rolling
   ↓
Replace Pods gradually

Blue-Green
   ↓
Run two versions
   ↓
Switch traffic

Canary
   ↓
Expose new version gradually

Argo Rollouts
   ↓
Automate progressive delivery
   ↓
Traffic management
   ↓
Prometheus analysis
   ↓
Promotion / Abort
```

The most important production architecture to remember is:

```text
                    GitOps
                       |
                     ArgoCD
                       |
                 Argo Rollouts
                       |
             +---------+---------+
             |                   |
          Stable              Canary
             |                   |
             +---------+---------+
                       |
                 Traffic Router
                       |
                      ALB
                       |
                    Users
                       |
                  Prometheus
                       |
                  Analysis
                       |
                +------+------+
                |             |
              PASS           FAIL
                |             |
             Promote        Abort
```

This completes the **production deployment strategy implementation series**. The next logical part of the project is to move back into the broader platform implementation—especially **production-grade networking/security/observability integration and then expanding the proven `order-service` pattern across all 21 microservices**, rather than creating 21 independent implementations.
