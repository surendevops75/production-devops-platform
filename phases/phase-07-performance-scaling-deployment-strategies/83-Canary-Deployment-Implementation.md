# 83 — Canary Deployment Implementation

> Production implementation of Canary Deployment for `order-service` on AWS EKS, including traffic splitting, Prometheus-based validation, rollback, GitOps, ArgoCD, ALB considerations, and Jira Change Management.

---

## 1. Objective

In the previous chapters:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
```

we deployed the application using two different strategies.

Now we implement:

```text
83 — Canary Deployment
```

The main idea is:

> Don't send all production traffic to the new version immediately.

Instead:

```text
v1 = stable
v2 = canary
```

Start with a small percentage:

```text
95% → v1
5%  → v2
```

Observe the new version.

If healthy:

```text
90% → v1
10% → v2
```

Then:

```text
75% → v1
25% → v2
```

Then:

```text
50% → v1
50% → v2
```

Finally:

```text
0%  → v1
100% → v2
```

If the canary shows problems:

```text
v2 → 0%
v1 → 100%
```

---

# 2. Why Canary Is Different

Blue-Green:

```text
100% v1
    ↓
100% v2
```

Canary:

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

The new version is exposed to real production traffic gradually.

---

# 3. Very Important: Pod Percentage Is Not Traffic Percentage

This is a common DevOps mistake.

Suppose you have:

```text
10 Pods
```

and deploy:

```text
1 canary Pod
9 stable Pods
```

You might think:

```text
1 / 10 = 10%
```

therefore:

```text
10% traffic → canary
```

That is **not a guaranteed traffic split**.

Why?

Because Kubernetes Services normally distribute traffic across endpoints without providing a precise percentage-routing guarantee.

Traffic can be influenced by:

```text
Connection reuse
HTTP keep-alive
Client behavior
Load-balancer behavior
Connection distribution
Endpoint selection
Protocol
Traffic patterns
```

Therefore:

```text
1 canary Pod
```

does **not** automatically mean:

```text
10% production traffic
```

---

# 4. What We Actually Need

For controlled traffic percentages, we need a traffic-management layer.

Conceptually:

```text
                    ALB
                     |
              Traffic Router
                     |
             +-------+-------+
             |               |
          Stable          Canary
           v1               v2
             |               |
             +-------+-------+
                     |
                  Metrics
                     |
                 Prometheus
```

The router decides:

```text
95% → Stable
5%  → Canary
```

rather than relying on Pod counts.

---

# 5. Canary Architecture Options

There are several ways to implement canary traffic.

### Option 1 — AWS ALB weighted target groups

```text
ALB
 |
 +-- Stable Target Group → v1
 |
 +-- Canary Target Group → v2
```

ALB can perform weighted forwarding between target groups.

---

### Option 2 — Service Mesh

Examples include:

```text
Istio
Linkerd
```

The mesh can provide traffic policies and weighted routing.

---

### Option 3 — Argo Rollouts

Argo Rollouts can coordinate:

```text
Deployment
+
Traffic routing
+
Analysis
+
Pause
+
Promotion
+
Abort
```

This is the approach we will eventually use in the production-style implementation.

---

# 6. Our Learning Architecture

We'll first understand the traffic model:

```text
                  ALB
                   |
           Traffic Management
                   |
        +----------+----------+
        |                     |
     Stable                 Canary
       v1                      v2
        |                     |
      Pods                   Pods
        |                     |
        +----------+----------+
                   |
              Dependencies
          RDS / Redis / SQS
```

The release controller changes:

```text
5%
10%
25%
50%
100%
```

while Prometheus evaluates:

```text
error rate
latency
availability
```

---

# 7. Canary Release State

We will use:

```text
Stable:
order-service:v1

Canary:
order-service:v2
```

Initially:

```text
Stable = 100%
Canary = 0%
```

Then:

```text
Stable = 95%
Canary = 5%
```

---

# 8. Initial Deployment

Suppose CI produces:

```text
Git commit:
8f31a2c
```

ECR:

```text
order-service:8f31a2c
```

Digest:

```text
sha256:222222...
```

GitOps records:

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:222222..."
```

But instead of immediately making it 100% active, we deploy it as the canary.

---

# 9. Stable Deployment

Stable:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service-stable

spec:
  replicas: 3

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
      rollout: stable

  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        rollout: stable

    spec:
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:STABLE_DIGEST
          ports:
            - containerPort: 8080
```

---

# 10. Canary Deployment

Canary:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service-canary

spec:
  replicas: 1

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
      rollout: canary

  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        rollout: canary

    spec:
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:CANARY_DIGEST
          ports:
            - containerPort: 8080
```

At this point:

```text
Stable
3 Pods
v1

Canary
1 Pod
v2
```

Again:

> This does not mean 25% of traffic goes to Canary.

The traffic router determines the actual percentage.

---

# 11. Stable and Canary Services

Conceptually:

```text
order-service-stable
        |
        +-- stable Pods


order-service-canary
        |
        +-- canary Pods
```

Example Stable Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service-stable

spec:
  selector:
    app.kubernetes.io/name: order-service
    rollout: stable

  ports:
    - port: 80
      targetPort: 8080
```

Canary:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service-canary

spec:
  selector:
    app.kubernetes.io/name: order-service
    rollout: canary

  ports:
    - port: 80
      targetPort: 8080
```

---

# 12. Traffic Router

Now the important component:

```text
                    ALB
                     |
               Traffic Router
                /          \
               /            \
          Stable             Canary
            |                   |
          v1 Pods             v2 Pods
```

Desired:

```text
Stable:
95%

Canary:
5%
```

The router must enforce that traffic distribution.

---

# 13. AWS ALB Weighted Routing

With AWS Load Balancer Controller, ALB can be configured with multiple target groups and weighted forwarding.

Conceptually:

```text
ALB listener
      |
      v
Forward action
      |
      +---- Stable TG
      |       Weight 95
      |
      +---- Canary TG
              Weight 5
```

This gives us an actual traffic-management mechanism rather than assuming Pod ratios equal traffic ratios.

The exact Kubernetes Ingress annotations/configuration should be validated against the AWS Load Balancer Controller version used by the cluster.

---

# 14. Conceptual ALB Configuration

Conceptually:

```text
Listener
   |
   +-------------------------+
   |                         |
Stable Target Group    Canary Target Group
Weight = 95            Weight = 5
   |                         |
Stable Pods              Canary Pods
```

Traffic:

```text
1000 requests
```

would target the groups according to the configured weighting, subject to normal load-balancer behavior and measurement variation.

It should not be interpreted as an exact guarantee that every arbitrary sample of 100 requests will be exactly 95/5.

---

# 15. Why We Need Health Checks

ALB should only send traffic to healthy targets.

Canary:

```text
Pod
 ↓
Readiness
 ↓
Target health
```

If the canary isn't healthy:

```text
Canary target
     ↓
Unhealthy
```

Traffic should not be sent to it as a healthy target.

But target health alone isn't enough.

The application could return:

```text
HTTP 200
```

while still having:

```text
High latency
Business failures
Database errors
Incorrect responses
```

That's where Prometheus becomes important.

---

# 16. Prometheus as Release Validation

We want to ask:

> Is Canary behaving acceptably under real production traffic?

We monitor:

```text
Request rate
Error rate
Latency
Availability
Pod restarts
CPU
Memory
Database errors
```

For HTTP:

```text
5xx rate
```

is especially important.

---

# 17. Example Canary Error Rate

Suppose during canary:

```text
Stable:
5xx = 0.10%

Canary:
5xx = 0.15%
```

Potentially acceptable depending on your SLO and baseline.

Now suppose:

```text
Stable:
0.10%

Canary:
5.5%
```

That is a strong signal to investigate and potentially abort.

The actual threshold should come from the service's SLO/SLA and historical baseline rather than an arbitrary universal number.

---

# 18. PromQL Concept

A common error-rate calculation is based on HTTP request counters.

Conceptually:

```promql
sum(rate(http_server_requests_seconds_count{
  service="order-service",
  status=~"5..",
  rollout="canary"
}[5m]))
/
sum(rate(http_server_requests_seconds_count{
  service="order-service",
  rollout="canary"
}[5m]))
```

This gives an approximate:

```text
Canary 5xx / Canary total requests
```

The exact metric name depends on your Spring Boot/Micrometer configuration.

---

# 19. Canary Latency

For latency we can examine histogram metrics.

For example, P95 latency can be calculated from histogram buckets.

Conceptually:

```promql
histogram_quantile(
  0.95,
  sum(
    rate(http_server_requests_seconds_bucket{
      service="order-service",
      rollout="canary"
    }[5m])
  ) by (le)
)
```

You can compare:

```text
Canary P95
vs
Stable P95
```

---

# 20. Canary Analysis

The release controller can evaluate:

```text
Canary error rate
Canary latency
Canary availability
```

Example policy:

```text
Canary error rate < defined threshold
AND
Canary P95 latency within SLO
AND
Canary Pods healthy
```

Then:

```text
PASS
```

Otherwise:

```text
FAIL
```

---

# 21. Release Steps

Our canary rollout:

```text
Step 1
100% Stable

Step 2
95% Stable
5% Canary

Step 3
90% Stable
10% Canary

Step 4
75% Stable
25% Canary

Step 5
50% Stable
50% Canary

Step 6
100% Canary
```

At every stage:

```text
Deploy
 ↓
Observe
 ↓
Analyze
 ↓
Continue OR Abort
```

---

# 22. First Canary — 5%

Start:

```text
Stable = 100%
Canary = 0%
```

Deploy canary.

Then:

```text
Stable = 95%
Canary = 5%
```

Wait:

```text
5 minutes
```

Then analyze:

```text
5xx
P95
P99
Pod health
Database
Logs
```

If healthy:

```text
Continue
```

Otherwise:

```text
Abort
```

---

# 23. Second Stage — 10%

Traffic:

```text
Stable = 90%
Canary = 10%
```

Observe again.

For example:

```text
Duration = 10 minutes
```

Metrics:

```text
Canary 5xx
Canary P95
Canary restart count
```

If within the release policy:

```text
Continue
```

---

# 24. Third Stage — 25%

```text
Stable = 75%
Canary = 25%
```

At this point Canary receives considerably more real traffic.

Now check:

```text
Database connections
RDS CPU
Redis
SQS
Downstream APIs
JVM
Node capacity
```

This is important because a problem may not appear at 5% traffic.

---

# 25. Fourth Stage — 50%

```text
Stable = 50%
Canary = 50%
```

Now the new version receives half of production traffic.

This is a significant validation point.

Check:

```text
Error rate
Latency
Throughput
Resource utilization
Downstream dependencies
Business metrics
```

---

# 26. Final Promotion

If all analysis passes:

```text
Stable = 0%
Canary = 100%
```

Canary becomes the new stable version.

Then:

```text
Canary → Stable
```

Conceptually:

```text
Before:

Stable = v1
Canary = v2


After:

Stable = v2
```

The old v1 can then be removed according to the retention policy.

---

# 27. What If Canary Fails?

Suppose:

```text
5% Canary
```

and:

```text
Canary 5xx = 7%
```

The release should stop.

```text
95% Stable
5% Canary
       |
       v
Analysis FAIL
       |
       v
Canary traffic → 0%
       |
       v
Stable → 100%
```

This is an important property of canary deployment:

> The failure blast radius is deliberately limited during the early stages.

---

# 28. Manual Canary Rollback

If using a simple ALB weighted setup:

```text
Current:

Stable = 90
Canary = 10
```

Rollback:

```text
Stable = 100
Canary = 0
```

Then verify:

```text
5xx
Latency
Health
Logs
```

---

# 29. Automated Canary Rollback

For automated progressive delivery, use a controller such as:

```text
Argo Rollouts
```

Architecture:

```text
                 Argo Rollouts
                       |
        +--------------+--------------+
        |                             |
   Traffic Router                 Analysis
        |                             |
 Stable / Canary                Prometheus
```

The controller can:

```text
set weight
pause
analyze
promote
abort
```

This is why Argo Rollouts is useful for sophisticated canary releases.

---

# 30. Argo Rollouts

Instead of:

```yaml
kind: Deployment
```

Argo Rollouts uses:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: order-service

spec:
  replicas: 4

  strategy:
    canary:
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

This is a simplified example. Real production configuration also needs the appropriate stable/canary Services and traffic-routing configuration.

---

# 31. What `setWeight` Means

This:

```yaml
- setWeight: 5
```

means:

```text
Canary traffic = 5%
```

**provided the configured traffic router supports and enforces that weight.**

It does not mean:

```text
1 of 20 Pods
```

It means the traffic-routing layer should direct approximately the configured percentage toward Canary.

---

# 32. Pause

Example:

```yaml
- pause:
    duration: 5m
```

means:

```text
Deploy 5%
        ↓
Wait 5 minutes
        ↓
Analyze
        ↓
Continue
```

A manual pause can also be used:

```yaml
- pause: {}
```

Then an operator explicitly promotes the rollout.

This is useful for production change windows.

---

# 33. AnalysisTemplate

We can define a Prometheus-based analysis.

Conceptually:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate

metadata:
  name: order-service-success-rate

spec:
  metrics:

    - name: canary-error-rate

      interval: 1m

      successCondition: result[0] < 0.01

      failureLimit: 3

      provider:

        prometheus:
          address: http://prometheus.monitoring.svc:9090

          query: |
            ...
```

This is intentionally a template rather than a universal threshold.

Your actual query and threshold should match your metric names and SLO.

---

# 34. What Happens When Analysis Fails?

Conceptually:

```text
Canary
   |
5% traffic
   |
Prometheus
   |
Analysis
   |
FAIL
   |
Argo Rollouts
   |
Abort
   |
Canary traffic → 0%
```

The stable version continues receiving traffic.

---

# 35. Analysis Is Not Just Error Rate

A mature production canary can evaluate:

```text
HTTP error rate
P95 latency
P99 latency
Pod restart rate
JVM memory
Database error rate
Queue backlog
Business transaction success
```

For `order-service`, useful business-level signals could include:

```text
Order creation success rate
Payment initiation failures
Inventory reservation failures
Order processing latency
```

Technical health alone isn't always enough.

---

# 36. Canary + GitOps

The GitOps architecture remains:

```text
Developer
    ↓
GitHub
    ↓
CI
    ↓
ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
Argo Rollouts
    ↓
EKS
```

The CI pipeline still does not directly deploy to EKS.

---

# 37. Canary GitOps Values

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:222222..."

canary:
  enabled: true

  steps:
    - setWeight: 5
    - pause: 5m

    - setWeight: 10
    - pause: 10m

    - setWeight: 25
    - pause: 10m

    - setWeight: 50
    - pause: 15m

    - setWeight: 100
```

The chart converts this into the Rollout configuration.

---

# 38. Jira Change Management

Canary requires a production change record.

Example:

```text
CHG-10454
```

Change type:

```text
Production Application Release
```

Service:

```text
order-service
```

Release:

```text
8f31a2c
```

Image:

```text
sha256:222222...
```

Strategy:

```text
Canary
```

Traffic stages:

```text
5%
10%
25%
50%
100%
```

---

# 39. Jira Implementation Plan

Example:

```text
1. Confirm staging validation
2. Confirm ECR digest
3. Confirm rollback version
4. Confirm Prometheus dashboards
5. Deploy canary
6. Shift 5%
7. Observe metrics
8. Shift 10%
9. Observe metrics
10. Shift 25%
11. Observe metrics
12. Shift 50%
13. Observe metrics
14. Promote to 100%
15. Validate application
16. Close change
```

---

# 40. Jira Rollback Plan

```text
1. Abort canary
2. Set canary traffic to 0%
3. Restore stable to 100%
4. Verify application health
5. Verify error rate
6. Verify latency
7. Run smoke tests
8. Investigate failed release
9. Record rollback
10. Create follow-up problem/change if required
```

---

# 41. Canary Monitoring Dashboard

Create a dashboard with two groups.

### Stable

```text
Requests
5xx
P95
P99
CPU
Memory
Restarts
```

### Canary

```text
Requests
5xx
P95
P99
CPU
Memory
Restarts
```

And a comparison:

```text
Metric             Stable       Canary

5xx rate            0.1%         0.2%
P95                 120ms        130ms
P99                 250ms        290ms
CPU                   35%          37%
Memory                52%          55%
Restarts               0            0
```

This gives the release engineer a direct comparison.

---

# 42. Logging

Logs should contain enough information to identify the release.

Example structured log:

```json
{
  "service": "order-service",
  "environment": "prod",
  "version": "2.4.0",
  "commit": "8f31a2c",
  "rollout": "canary",
  "level": "INFO",
  "message": "Order created"
}
```

Then ELK can filter:

```text
rollout:canary
```

and compare Canary with Stable.

Do not put passwords, tokens, card data, or other sensitive information into logs.

---

# 43. Trace Correlation

With OpenTelemetry:

```text
Client
  ↓
ALB
  ↓
Order Service Canary
  ↓
Inventory Service
  ↓
Payment Service
  ↓
RDS
```

You can inspect whether latency/error increases are caused by:

```text
Canary application
```

or:

```text
Downstream dependency
```

This becomes especially important at 25% and 50% traffic.

---

# 44. Canary and Database Load

Suppose production currently generates:

```text
1000 requests/min
```

At 5%:

```text
~50 requests/min
```

At 50%:

```text
~500 requests/min
```

Database load may increase or change depending on application behavior.

Monitor:

```text
RDS CPU
Database connections
Read/write IOPS
Query latency
Lock contention
Connection pool utilization
```

A canary isn't isolated from shared dependencies.

---

# 45. Canary and Redis

Monitor:

```text
Redis memory
CPU
Hit rate
Evictions
Connections
Latency
```

A new version could introduce:

```text
Cache key explosion
Different serialization
Higher cache miss rate
Larger objects
```

Even if HTTP metrics initially look healthy.

---

# 46. Canary and SQS

For asynchronous workflows:

```text
Order Service
    |
    v
SQS
    |
    v
Consumer
```

Monitor:

```text
ApproximateNumberOfMessagesVisible
AgeOfOldestMessage
DLQ messages
Consumer errors
Processing latency
```

A new release might process messages differently and cause backlog.

---

# 47. Canary and Kafka

If Kafka is involved, monitor:

```text
Consumer lag
Partition errors
Rebalances
Processing latency
DLQ/retry topics
```

Don't assume a successful HTTP canary means asynchronous processing is healthy.

---

# 48. Canary Failure Example

Suppose:

```text
Stable:
5xx = 0.15%

Canary:
5xx = 0.20%
```

At 5%, acceptable under the defined policy.

Continue:

```text
10%
```

Now:

```text
Canary:
5xx = 0.3%
```

Continue if within policy.

At 25%:

```text
Canary:
5xx = 4.5%
```

Analysis fails.

```text
25% Canary
     |
     v
Analysis FAIL
     |
     v
Abort
     |
     v
0% Canary
100% Stable
```

---

# 49. Why We Don't Automatically Pick a Universal Threshold

For example, don't blindly define:

```text
5xx > 1%
```

for every service.

Instead consider:

```text
Service SLO
Historical baseline
Traffic volume
Error budget
Business criticality
Normal variance
Measurement window
```

A threshold should be meaningful for the particular workload.

---

# 50. Canary Release Timeline

A realistic release might look like:

```text
10:00
Deploy canary

10:02
Canary healthy

10:05
5% traffic

10:10
Metrics pass

10:15
10% traffic

10:25
Metrics pass

10:30
25% traffic

10:40
Metrics pass

10:45
50% traffic

11:00
Metrics pass

11:05
100% traffic

11:15
Post-deployment validation

11:30
Change closed
```

The exact timing should depend on traffic volume and how quickly meaningful metrics accumulate.

---

# 51. Canary Rollback Example

Suppose:

```text
50% Canary
```

Then:

```text
Database errors ↑
HTTP 5xx ↑
P95 latency ↑
```

Argo Rollouts:

```text
Analysis
   ↓
FAIL
   ↓
Abort
```

Traffic:

```text
Stable = 100%
Canary = 0%
```

Then:

```text
Prometheus
   ↓
Errors return to baseline
```

The incident/change record captures:

```text
Release failed
Rollback performed
Metrics observed
Root cause investigation required
```

---

# 52. Canary Security

The canary version receives real production traffic.

Therefore it needs exactly the same security controls as the stable version:

```text
IAM
Pod Identity
NetworkPolicy
Security Groups
Secrets
TLS
WAF
Container scanning
RBAC
```

Do not treat Canary as:

```text
"just a test Pod"
```

It is potentially handling real customer traffic.

---

# 53. Canary Capacity Planning

If stable needs:

```text
3 Pods
```

Canary may initially need:

```text
1 Pod
```

but at later stages:

```text
50% traffic
```

you may need more canary capacity.

Example:

```text
Stable:
3 Pods

Canary:
3 Pods
```

At 50/50.

At 100%:

```text
Canary:
3 Pods
Stable:
0 Pods
```

Capacity must be available before increasing traffic.

---

# 54. Canary Strategy With HPA

Be careful with HPA.

Stable:

```text
HPA
3 → 8
```

Canary:

```text
HPA
1 → 4
```

Traffic percentage and replica count are independent concerns.

Traffic:

```text
10%
```

doesn't mean:

```text
10% Pods
```

The traffic router controls traffic.

HPA controls replica capacity.

---

# 55. Canary vs Blue-Green

| Characteristic         | Blue-Green         | Canary                  |
| ---------------------- | ------------------ | ----------------------- |
| New version            | Full environment   | Small initial exposure  |
| Initial traffic        | 0%                 | Small percentage        |
| Traffic shift          | Usually one switch | Gradual                 |
| Blast radius           | Larger at switch   | Smaller initially       |
| Extra capacity         | High               | Variable                |
| Observability          | Important          | Critical                |
| Rollback               | Fast               | Fast                    |
| Traffic routing        | Service/ALB/etc.   | Traffic router required |
| Automation complexity  | Medium             | High                    |
| Progressive validation | Limited            | Strong                  |

---

# 56. Canary vs Rolling

| Characteristic           | Rolling            | Canary              |
| ------------------------ | ------------------ | ------------------- |
| New Pods                 | Gradually created  | Explicit canary     |
| Traffic control          | Kubernetes Service | Traffic router      |
| Exact traffic percentage | No                 | Yes, when supported |
| Progressive exposure     | Limited            | Yes                 |
| Automated analysis       | Basic              | Strong              |
| Complexity               | Lower              | Higher              |
| Extra tooling            | Low                | Often required      |

---

# 57. Complete Production Architecture

```text
                       GitHub
                          |
                    GitHub Actions
                          |
              +-----------+-----------+
              |                       |
           Tests                    Security
              |                 Sonar / Trivy
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
             +------------+------------+
             |                         |
          Stable                    Canary
            v1                        v2
             |                         |
             +------------+------------+
                          |
                    Traffic Router
                          |
                         ALB
                          |
                       Clients
                          |
                     Prometheus
                          |
                     Analysis
                          |
              +-----------+-----------+
              |                       |
            PASS                     FAIL
              |                       |
          Increase                   Abort
           traffic                     |
              |                       |
          100% v2                  100% v1
```

---

# 58. Practical Production Flow

The complete release becomes:

```text
Developer
    ↓
Pull Request
    ↓
CI
    ↓
Unit Tests
    ↓
SonarQube
    ↓
Dependency Scan
    ↓
Trivy
    ↓
Docker Image
    ↓
ECR
    ↓
Digest
    ↓
GitOps PR
    ↓
Jira Change Approval
    ↓
ArgoCD
    ↓
Canary 5%
    ↓
Prometheus Analysis
    |
    +---- FAIL → Abort
    |
    +---- PASS
             ↓
          10%
             ↓
          Analysis
             |
             +---- FAIL → Abort
             |
             +---- PASS
                     ↓
                    25%
                     ↓
                  Analysis
                     ↓
                    50%
                     ↓
                  Analysis
                     ↓
                   100%
                     ↓
                 Release Complete
```

---

# 59. What You Should Remember

The most important concepts from this chapter are:

### 1. Canary is gradual exposure

```text
v1 → small v2 → more v2 → 100% v2
```

### 2. Pod count is not traffic percentage

```text
1 canary Pod ≠ guaranteed 10% traffic
```

### 3. Traffic routing is required

```text
ALB / Service Mesh / Argo Rollouts
```

### 4. Observability is part of the deployment

```text
Prometheus
Grafana
ELK
OpenTelemetry
```

### 5. Rollback should be planned before deployment

```text
Canary fails
    ↓
Traffic → Stable
```

### 6. Shared dependencies matter

```text
RDS
Redis
SQS
Kafka
External APIs
```

### 7. Jira records the production change

```text
CHG-10454
```

### 8. Git remains the source of desired state

```text
GitOps → ArgoCD → EKS
```

---

# 60. Next Chapter

We now have:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
83 — Canary Deployment
```

The next step is to bring these ideas together with a production-grade progressive delivery controller:

# **84 — Argo Rollouts Implementation**

There we'll implement the actual Kubernetes `Rollout` resource, **Stable Service + Canary Service**, traffic routing, `AnalysisTemplate` with Prometheus, automated promotion/abort, manual approval pauses, ArgoCD integration, rollout commands, failure testing, and the complete production release flow.
