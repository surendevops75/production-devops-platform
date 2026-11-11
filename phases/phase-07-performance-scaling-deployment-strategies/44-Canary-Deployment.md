# 44 — Canary Deployment

> **Enterprise E-Commerce Platform | AWS EKS | Kubernetes | Helm | ArgoCD | Argo Rollouts | Prometheus | Grafana**

Canary Deployment is a production deployment strategy where a **small percentage of production traffic is sent to the new application version first**.

Instead of immediately sending:

```text
100% → v2
```

we gradually move traffic:

```text
1% → v2
5% → v2
10% → v2
25% → v2
50% → v2
100% → v2
```

At every stage we evaluate the application's health.

If something goes wrong:

```text
v2 → rollback
```

The majority of users can continue using the stable version.

---

# 1. Why Canary Deployment?

Imagine our current production version is:

```text
order-service v1.5.0
```

We want to release:

```text
order-service v1.6.0
```

A normal deployment could send everyone to v1.6.0.

```text
100% users
      │
      ▼
    v1.6.0
```

If v1.6.0 has a serious bug:

```text
100% users affected
```

With Canary:

```text
100% users
     │
     ├── 95% → v1.5.0
     │
     └── 5%  → v1.6.0
```

If v1.6.0 is broken:

```text
Only a small percentage
of production traffic
was exposed.
```

---

# 2. Canary vs Blue-Green

Blue-Green:

```text
                Traffic
                   │
             ┌─────┴─────┐
             ▼           ▼
           BLUE        GREEN
          100%           0%

                ↓

           BLUE          GREEN
             0%          100%
```

Canary:

```text
                Traffic
                   │
             ┌─────┴─────┐
             ▼           ▼
          STABLE       CANARY
           95%            5%

                ↓

          STABLE       CANARY
            75%          25%

                ↓

          STABLE       CANARY
             0%         100%
```

### Key difference

**Blue-Green**

> Switch traffic between two complete versions.

**Canary**

> Gradually increase traffic to the new version.

---

# 3. Canary Architecture in Our EKS Project

Our architecture:

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
WAF
   │
   ▼
ALB
   │
   ▼
Ingress
   │
   ▼
Traffic Management
      │
      ├──────────────┐
      ▼              ▼
   Stable          Canary
    v1.5             v1.6
```

For example:

```text
Stable:
order-service v1.5.0

Canary:
order-service v1.6.0
```

---

# 4. Canary Is Not Simply Two Deployments

This is a common mistake.

Suppose:

```text
stable Deployment = 9 Pods
canary Deployment = 1 Pod
```

That **does not automatically guarantee exactly 10% traffic**.

Why?

Because Kubernetes Services normally distribute connections across endpoints, not according to a simple Pod percentage guarantee.

If:

```text
9 stable Pods
1 canary Pod
```

you might approximately influence traffic distribution, but it is not a robust traffic-management mechanism.

Production Canary requires deliberate traffic control.

---

# 5. Traffic Splitting Options

There are several ways to implement Canary.

### Option 1 — Replica ratio

```text
9 stable
1 canary
```

Simple but limited.

### Option 2 — Ingress/controller traffic splitting

For example:

```text
ALB / Ingress
      │
      ├── 95% Stable
      └── 5% Canary
```

### Option 3 — Service mesh

Examples include:

```text
Istio
Linkerd
```

Traffic policies can explicitly control percentages.

### Option 4 — Argo Rollouts

Argo Rollouts can integrate with supported traffic-management providers and implement controlled Canary strategies.

For our project, we will primarily learn:

```text
Argo Rollouts
+
EKS
+
Ingress/ALB or traffic-management integration
+
Prometheus analysis
```

---

# 6. Argo Rollouts

Kubernetes Deployment supports:

```text
RollingUpdate
Recreate
```

For advanced progressive delivery, we can use:

```text
Argo Rollouts
```

Conceptually:

```text
Deployment
   ↓
Basic rollout

Argo Rollout
   ↓
Progressive delivery
   ├── Blue-Green
   └── Canary
```

---

# 7. Basic Argo Rollout

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  namespace: production

spec:
  replicas: 10

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

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
```

The exact API fields should be aligned with the Argo Rollouts version selected for our implementation.

---

# 8. Canary Strategy

Conceptually:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 5
      - pause: {}
      - setWeight: 25
      - pause: {}
      - setWeight: 50
      - pause: {}
```

This means:

```text
Deploy Canary
     │
     ▼
5%
     │
     ▼
Pause
     │
     ▼
25%
     │
     ▼
Pause
     │
     ▼
50%
     │
     ▼
Pause
     │
     ▼
100%
```

The actual traffic percentage behavior depends on the traffic-management integration.

---

# 9. Why Pause?

The pause gives us time to observe.

Example:

```text
5% Canary
     │
     ▼
Wait 10 minutes
     │
     ▼
Check metrics
```

Metrics:

```text
HTTP 5xx
p95 latency
p99 latency
CPU
memory
restarts
database errors
business failures
```

If healthy:

```text
5% → 25%
```

If unhealthy:

```text
Rollback
```

---

# 10. Automated Canary Analysis

Manual validation is useful.

But production systems can automatically evaluate metrics.

Example:

```text
Canary = 5%
       │
       ▼
Prometheus
       │
       ├── Error rate
       ├── Latency
       ├── Request rate
       └── Availability
       │
       ▼
Analysis
       │
   ┌───┴────┐
   ▼        ▼
Success   Failure
   │        │
   ▼        ▼
Continue  Rollback
```

---

# 11. Prometheus Analysis

Suppose:

```text
Stable 5xx = 0.2%
Canary 5xx = 0.3%
```

Probably acceptable depending on our SLO.

But:

```text
Stable 5xx = 0.2%
Canary 5xx = 7%
```

is clearly suspicious.

The rollout should stop.

---

# 12. AnalysisTemplate

Argo Rollouts supports analysis templates.

Conceptually:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: order-service-success-rate
  namespace: production

spec:
  metrics:
    - name: success-rate

      interval: 1m

      successCondition: result[0] >= 0.99

      failureLimit: 3

      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            ...
```

The PromQL query must be designed carefully for the actual metric labels produced by our application.

---

# 13. Don't Copy PromQL Blindly

For our Spring Boot applications we might have metrics such as:

```text
http_server_requests_seconds_count
```

or version-dependent Micrometer metric names.

Before writing a production AnalysisTemplate:

```text
1. Inspect actual metrics
2. Confirm labels
3. Confirm service/version labels
4. Test PromQL in Grafana/Prometheus
5. Verify stable vs canary separation
6. Only then automate promotion
```

This is important.

A wrong PromQL query can cause an unsafe rollout.

---

# 14. Canary Metrics

We want to compare:

```text
Canary
vs
Stable
```

not simply:

```text
entire cluster
```

Example dimensions:

```text
service="order-service"
version="canary"
```

versus:

```text
service="order-service"
version="stable"
```

Labels need to be designed consistently.

---

# 15. Error Rate

Conceptually:

```text
error_rate =
5xx_requests
--------------
total_requests
```

Example:

```text
Canary:

5xx = 50
total = 10,000

error rate = 0.5%
```

---

# 16. Latency

We should monitor:

```text
p50
p90
p95
p99
```

For production APIs, p95 and p99 are often particularly useful.

Example:

```text
Stable p95 = 180 ms
Canary p95 = 210 ms
```

Potentially acceptable.

But:

```text
Stable p95 = 180 ms
Canary p95 = 1,800 ms
```

should stop the rollout.

---

# 17. Availability

If our SLO is:

```text
99.9%
```

we should ensure Canary doesn't materially degrade availability.

Canary analysis should reflect the actual service SLO rather than arbitrary generic thresholds.

---

# 18. Business Metrics

Technical metrics are not enough.

For `order-service`, consider:

```text
orders created
orders failed
payment initiation failures
inventory reservation failures
```

For `payment-service`:

```text
authorization failures
timeouts
duplicate requests
gateway failures
```

A deployment can have:

```text
HTTP 200
low CPU
low memory
```

while business functionality is broken.

---

# 19. Canary for Order Service

Suppose:

```text
v1.5.0 = stable
v1.6.0 = canary
```

Traffic:

```text
95% → v1.5.0
5%  → v1.6.0
```

We monitor:

```text
order creation success
database latency
SQS publishing
Redis latency
HTTP errors
application exceptions
```

If healthy:

```text
5%
 ↓
25%
 ↓
50%
 ↓
100%
```

---

# 20. Canary and Database Changes

The same database rule from Blue-Green applies here.

During:

```text
95% v1
5%  v2
```

both application versions may be active simultaneously.

Therefore:

```text
Database schema
      ↓
must support
      ↓
v1 + v2
```

Use:

```text
expand
  ↓
deploy
  ↓
migrate
  ↓
contract
```

not:

```text
break schema
  ↓
hope Canary protects us
```

Canary cannot protect an incompatible database migration.

---

# 21. Canary and Redis

Same concern.

Suppose v1 uses:

```text
cart:v1:{customerId}
```

and v2 expects:

```text
cart:v2:{customerId}
```

During Canary:

```text
v1 + v2
```

operate simultaneously.

Therefore cache compatibility needs to be designed.

---

# 22. Canary and SQS

Suppose:

```text
Stable consumers → SQS
Canary consumers  → SQS
```

Both may consume real messages.

This can be dangerous.

For consumers, we need to distinguish:

```text
request-serving services
```

from:

```text
message-processing workers
```

Traffic-based Canary works naturally for HTTP traffic, but asynchronous consumers require additional rollout design.

---

# 23. Kafka Canary

Kafka introduces another important dimension:

```text
consumer groups
```

Suppose:

```text
order-events
```

is consumed by:

```text
order-worker-v1
```

A new version could use:

```text
order-worker-v2
```

But simply creating another consumer group means both groups may process every event.

That may cause duplicate business processing.

Therefore Kafka Canary requires careful consideration of:

```text
consumer groups
offsets
partition ownership
idempotency
processing semantics
```

---

# 24. Canary + Idempotency

Suppose:

```text
Canary
   ↓
payment request
```

and the request is retried.

If the application is not idempotent:

```text
Payment
Payment
```

could happen.

Therefore critical operations need idempotency keys.

For example:

```text
Idempotency-Key: order-123-payment
```

The exact implementation belongs in application design, but DevOps must understand the dependency.

---

# 25. Canary Traffic Strategies

There are several possible progression models.

### Conservative

```text
1%
5%
10%
25%
50%
100%
```

### Faster

```text
10%
25%
50%
100%
```

### Very cautious

```text
1%
2%
5%
10%
25%
50%
100%
```

There is no universally correct percentage.

It depends on:

```text
traffic volume
service criticality
risk
SLO
deployment frequency
capacity
observability
```

---

# 26. Low-Traffic Service Problem

Suppose a service receives:

```text
20 requests/hour
```

A 1% Canary gives:

```text
0.2 requests/hour
```

You cannot meaningfully evaluate the new version from such a small sample.

Therefore Canary requires enough traffic to generate useful statistical evidence.

For low-volume services:

```text
Blue-Green
or
Rolling + strong validation
```

may be more practical.

---

# 27. High-Traffic Service

Suppose:

```text
1,000,000 requests/hour
```

Even:

```text
1%
```

means:

```text
10,000 requests/hour
```

That's a very meaningful Canary.

High-traffic systems can use very small initial percentages.

---

# 28. Canary and HPA

Suppose:

```text
Stable:
10 Pods

Canary:
2 Pods
```

Canary receives 10% traffic.

Canary Pods may experience:

```text
high CPU
```

while stable Pods are fine.

HPA can scale Canary independently if metrics and workload structure support it.

Example:

```text
Canary
2 Pods
  ↓
CPU 85%
  ↓
HPA
  ↓
4 Pods
```

---

# 29. Canary and Karpenter

If Canary needs more capacity:

```text
Canary Pods
     │
     ▼
Pending
     │
     ▼
Karpenter
     │
     ▼
New Node
     │
     ▼
Canary Pods scheduled
```

This is why rollout capacity must be considered before deployment.

---

# 30. Resource Requests Matter

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

If requests are wrong:

```text
HPA behavior
+
Karpenter provisioning
+
scheduling
```

can all behave incorrectly.

---

# 31. Canary and Pod Topology

Don't accidentally place all Canary Pods on one node.

Example:

```text
AZ-A
 ├── stable
 └── canary

AZ-B
 ├── stable
 └── canary

AZ-C
 ├── stable
 └── canary
```

Use appropriate:

```text
topologySpreadConstraints
podAntiAffinity
```

for critical services.

---

# 32. Canary and PDB

PDB protects availability during voluntary disruptions.

Example:

```text
Canary
  │
  ▼
Node drain
  │
  ▼
PDB
```

But PDB does not determine:

```text
5% traffic
```

or:

```text
25% traffic
```

Traffic control and disruption protection are separate concepts.

---

# 33. Canary Rollback

Suppose:

```text
5% → Canary
```

and analysis fails.

Argo Rollouts can abort the rollout.

Conceptually:

```text
Canary
  │
  ▼
Analysis failed
  │
  ▼
Abort
  │
  ▼
Stable remains active
```

This is safer than:

```text
deploy v2
↓
wait for customers to complain
```

---

# 34. Rollback at 50%

Suppose:

```text
Stable = 50%
Canary = 50%
```

Then errors appear.

We immediately move traffic back:

```text
Stable = 100%
Canary = 0%
```

The failed version remains available for investigation until cleaned up.

---

# 35. Automatic vs Manual Rollback

### Automatic

Useful when:

```text
clear measurable failure
+
trusted analysis
```

Example:

```text
5xx > 5%
```

### Manual

Useful when:

```text
business metric
+
ambiguous signal
+
high-impact action
```

Production systems often combine both.

---

# 36. Alertmanager During Canary

Prometheus:

```text
Canary error rate ↑
```

Alertmanager:

```text
CRITICAL
```

Routes alert:

```text
service=order-service
severity=critical
team=backend
```

Then:

```text
On-call
   ↓
Grafana
   ↓
Kibana
   ↓
Jaeger
```

This gives the engineer evidence before deciding whether to continue or abort.

---

# 37. Canary + Grafana

A useful dashboard should show Stable and Canary side by side.

Example:

```text
ORDER-SERVICE CANARY

                Stable       Canary

Traffic          95%          5%
RPS              950          50
5xx              0.2%         0.4%
p95              180ms        210ms
p99              350ms        420ms
CPU              55%          62%
Memory           60%          65%
Restarts         0            0
```

This is much more useful than looking at Canary alone.

---

# 38. Canary + Kibana

Search:

```text
service.name:"order-service"
```

Then compare:

```text
service.version:"1.5.0"
```

versus:

```text
service.version:"1.6.0"
```

Look for:

```text
ERROR
Exception
timeout
database error
Redis error
SQS failure
```

---

# 39. Canary + Jaeger

Suppose users report:

```text
Order API is slow.
```

Find Canary traces:

```text
trace
 ↓
POST /api/v1/orders
 ↓
order-service
 ↓
PostgreSQL
 ↓
SQS
```

If the new version introduces:

```text
DB query = 2 seconds
```

Jaeger can expose the slow span.

---

# 40. Canary + OpenTelemetry

Ensure telemetry includes useful resource attributes such as:

```text
service.name
service.version
deployment.environment
```

For example:

```text
service.name = order-service
service.version = 1.6.0
deployment.environment = production
```

Then we can distinguish:

```text
Stable
vs
Canary
```

through observability.

---

# 41. Canary Traffic by User

An advanced approach is to send specific users to Canary.

Example:

```text
internal employees → Canary
```

or:

```text
test customers → Canary
```

This is sometimes called:

```text
header-based routing
cookie-based routing
user-based routing
```

It can be useful for controlled validation.

---

# 42. Canary Traffic by Header

Example:

```text
X-Canary: true
```

Traffic:

```text
Normal users
     ↓
Stable

X-Canary: true
     ↓
Canary
```

This is useful for:

* internal testing
* QA
* support engineers
* controlled validation

But don't expose an unsafe bypass to arbitrary internet users.

---

# 43. Canary and Security

Canary must use the same security controls as production.

Don't create:

```text
Canary
 ↓
security bypass
```

For example, Green/Canary should still have:

```text
IAM
NetworkPolicy
TLS
authentication
authorization
WAF
logging
secrets
RBAC
```

---

# 44. Canary and Secrets

Both versions may use:

```text
Secrets Manager
      ↓
External Secrets Operator
      ↓
Kubernetes Secret
      ↓
Stable + Canary
```

If v2 needs a new secret:

```text
Add secret first
      ↓
validate
      ↓
deploy Canary
```

Avoid:

```text
deploy Canary
      ↓
secret missing
      ↓
CrashLoopBackOff
```

---

# 45. Canary and Configuration

Configuration changes should also be backward-compatible.

Suppose v2 requires:

```text
NEW_PAYMENT_MODE=true
```

Don't immediately remove the old configuration.

Prefer:

```text
old config
+
new optional config
```

Then gradually migrate.

---

# 46. Canary and Feature Flags

Canary and feature flags solve different problems.

Canary:

```text
Which application version receives traffic?
```

Feature flag:

```text
Which feature is enabled?
```

They can work together.

Example:

```text
v1.6.0 Canary
     │
     └── new checkout feature = OFF
```

After validation:

```text
Canary healthy
     ↓
feature flag = 5%
     ↓
25%
     ↓
100%
```

This gives another control layer.

---

# 47. Canary Deployment Pipeline

Our complete deployment becomes:

```text
Developer
   │
   ▼
GitHub
   │
   ▼
CI
   │
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
   Argo Rollouts
         │
         ▼
      Canary 5%
         │
         ▼
      Analysis
         │
      ┌──┴──┐
      ▼     ▼
   Success Failure
      │       │
      ▼       ▼
   25%      Abort
      │
      ▼
    50%
      │
      ▼
   100%
```

---

# 48. Example Production Progression

For `order-service`:

```text
Current:

v1.5.0 = 100%
```

Deploy:

```text
v1.6.0 = Canary
```

### Stage 1

```text
v1.5 = 99%
v1.6 = 1%
```

Observe.

### Stage 2

```text
v1.5 = 95%
v1.6 = 5%
```

Observe.

### Stage 3

```text
v1.5 = 90%
v1.6 = 10%
```

Observe.

### Stage 4

```text
v1.5 = 75%
v1.6 = 25%
```

Observe.

### Stage 5

```text
v1.5 = 50%
v1.6 = 50%
```

Observe.

### Final

```text
v1.5 = 0%
v1.6 = 100%
```

---

# 49. What Does "Healthy" Actually Mean?

A Canary should not simply pass:

```text
Pod Ready = true
```

Healthy means multiple layers are acceptable:

```text
Infrastructure
      ↓
Pod healthy
      ↓
Application healthy
      ↓
HTTP healthy
      ↓
Dependency healthy
      ↓
Business transaction healthy
      ↓
SLO healthy
```

---

# 50. Example Failure

Imagine:

```text
Canary = 5%
```

Everything initially looks fine.

After 5 minutes:

```text
5xx ↑
```

Grafana shows:

```text
Stable = 0.2%
Canary = 6.8%
```

Jaeger shows:

```text
order-service
   ↓
PostgreSQL
   ↓
slow query
```

Kibana shows:

```text
timeout acquiring database connection
```

Root cause:

```text
v1.6 increased DB connection usage
```

The correct action:

```text
Abort Canary
↓
100% Stable
↓
Investigate connection pool
```

---

# 51. Canary Can Reveal Capacity Problems

Suppose v1.6 changes:

```text
DB pool size
```

At 5% traffic:

```text
everything okay
```

At 50%:

```text
RDS connections ↑
```

At 100%:

```text
too many connections
```

This demonstrates why gradual traffic is useful.

It lets us observe scaling behavior before exposing everyone.

---

# 52. Canary Does Not Eliminate Load Testing

Before production:

```text
Load Test
   ↓
Canary
   ↓
Production
```

Load testing helps answer:

```text
Can the new version handle expected traffic?
```

Canary answers:

```text
Does the new version behave correctly under real production conditions?
```

They complement each other.

---

# 53. Canary Does Not Eliminate Testing

Still required:

```text
Unit tests
Integration tests
Contract tests
Security scans
Performance tests
Smoke tests
```

Canary is another production safety layer.

It is not a replacement for CI testing.

---

# 54. Canary and Contracts

For microservices:

```text
order-service
      │
      ▼
payment-service
```

Suppose payment-service v2 changes an API unexpectedly.

Canary order-service may fail only when it calls payment-service.

Therefore:

```text
Consumer-driven contract testing
```

can reduce this risk.

---

# 55. Canary in a 21-Microservice Architecture

Don't deploy all 21 services as Canary simultaneously.

That creates:

```text
21 variables changing at once
```

Troubleshooting becomes difficult.

Prefer:

```text
order-service
   ↓
Canary
   ↓
observe
   ↓
stable
```

Then:

```text
payment-service
   ↓
Canary
```

This makes causality clearer.

---

# 56. Deployment Waves

For larger releases:

```text
Wave 1
order-service

Wave 2
payment-service

Wave 3
inventory-service

Wave 4
other services
```

This reduces blast radius.

---

# 57. Canary Cost

Canary generally needs additional capacity.

Example:

```text
Stable:
10 Pods

Canary:
2 Pods
```

Total:

```text
12 Pods
```

If Canary becomes:

```text
50%
```

we may need significantly more resources.

Karpenter can provision nodes dynamically, but AWS cost still increases.

---

# 58. Production Trade-Off

Canary provides:

```text
lower blast radius
+
gradual rollout
+
real production validation
+
automated rollback
```

But introduces:

```text
traffic management complexity
+
observability requirements
+
more deployment state
+
more capacity
+
more complicated troubleshooting
```

Therefore:

> Don't use Canary just because it sounds more advanced.

Use it when the risk justifies the complexity.

---

# 59. Blue-Green vs Canary — Practical Decision

Use **Blue-Green** when:

```text
fast full rollback is important
+
traffic switching is straightforward
+
extra capacity is acceptable
```

Use **Canary** when:

```text
you want gradual exposure
+
you have strong metrics
+
you want automated analysis
+
the service has enough traffic
```

Use **Rolling** when:

```text
risk is lower
+
standard Kubernetes rollout is sufficient
```

---

# 60. Recommended Strategy for Our Platform

For our project:

```text
                    Deployment Strategy
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
       Rolling          Blue-Green        Canary
          │                │                │
   normal services     high-risk        high-risk /
                       fast rollback    high traffic
```

Example:

```text
order-service
    → Blue-Green / Canary

payment-service
    → Canary

catalog-service
    → Rolling

search-service
    → Rolling

inventory-service
    → Canary
```

The final choice should be based on actual service behavior and business risk.

---

# 61. Production Canary Checklist

### Before Canary

```text
☐ CI passed
☐ Tests passed
☐ SonarQube passed
☐ Dependency scan passed
☐ Secret scan passed
☐ Trivy passed
☐ SBOM generated
☐ Image immutable
☐ GitOps PR approved
☐ Database compatibility verified
☐ Redis compatibility verified
☐ Messaging behavior verified
☐ Rollback plan verified
```

### During Canary

```text
☐ Canary Pods healthy
☐ Readiness healthy
☐ Error rate acceptable
☐ p95 latency acceptable
☐ p99 latency acceptable
☐ CPU acceptable
☐ Memory acceptable
☐ DB connections acceptable
☐ Redis healthy
☐ SQS/Kafka healthy
☐ Business metrics healthy
☐ No critical alerts
```

### Promotion

```text
☐ 1% healthy
☐ 5% healthy
☐ 10% healthy
☐ 25% healthy
☐ 50% healthy
☐ 100% healthy
```

The exact stages can vary.

---

# 62. Final Production Architecture

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
                    Traffic Management
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
          STABLE v1.5                  CANARY v1.6
            95%                            5%
                 │                         │
                 └────────────┬────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
               RDS          Redis           SQS
                │             │             │
                └─────────────┼─────────────┘
                              │
                       Observability
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
   Prometheus                ELK                   OTel
       │                      │                      │
       ▼                      ▼                      ▼
    Grafana                Kibana                  Jaeger
       │
       ▼
  Alertmanager
       │
       ▼
   On-Call / SRE
```

---

# 63. Final Mental Model

The complete progressive-delivery model is:

```text
                 BUILD
                   │
                   ▼
                 TEST
                   │
                   ▼
                SECURITY
                   │
                   ▼
                  ECR
                   │
                   ▼
                GitOps
                   │
                   ▼
                ArgoCD
                   │
                   ▼
             Argo Rollouts
                   │
                   ▼
             Canary 1–5%
                   │
                   ▼
              Observe
          ┌────────┼────────┐
          ▼        ▼        ▼
       Metrics    Logs    Traces
          │        │        │
          └────────┼────────┘
                   ▼
                Analysis
              ┌────┴────┐
              ▼         ▼
           Healthy    Failure
              │         │
              ▼         ▼
          Increase    Abort
           traffic      │
              │         ▼
              ▼       Stable
          100%          100%
              │
              ▼
          New Version
```

The most important principle is:

> **Canary is not just deploying a few Pods. It is a controlled production experiment where traffic exposure, observability, automated analysis, and rollback work together.**

For our project, this gives us the three major deployment strategies:

```text
Rolling
   ↓
Standard deployments

Blue-Green
   ↓
Fast environment switch

Canary
   ↓
Gradual production exposure
```

**Next: `45-Disaster-Recovery.md`** — we move from deployment safety to **what happens when the production environment itself fails**: AWS AZ failure, EKS failure, RDS failure, Redis failure, SQS problems, region failure, RPO/RTO, multi-AZ vs multi-region, backup strategy, recovery architecture, and real disaster scenarios.
