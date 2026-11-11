# 80 — Production Deployment Strategies

> **Goal:** Understand and practically implement the major Kubernetes/EKS deployment strategies used in production:
>
> **Rolling Update → Recreate → Blue-Green → Canary → Argo Rollouts**
>
> We will also connect them with:
>
> **Jira Change → GitOps → ArgoCD → EKS → Observability → Rollback**

---

# 1. Why Deployment Strategy Matters

Suppose production currently runs:

```text
order-service v1
```

and we want to deploy:

```text
order-service v2
```

The important question is:

> **How do we move traffic from v1 to v2?**

There are several approaches.

```text
v1
 │
 ├── Rolling
 │
 ├── Recreate
 │
 ├── Blue-Green
 │
 └── Canary
        │
        └── Argo Rollouts can automate it
```

The strategy affects:

* downtime
* rollback speed
* infrastructure cost
* risk
* traffic management
* testing
* operational complexity

---

# 2. The Five Things We Need to Understand

Before implementation, separate these concepts:

### Deployment

Putting the new version onto Kubernetes.

### Release

Making the new version available to users.

### Traffic shifting

Determining which version receives requests.

### Validation

Determining whether the new version is healthy.

### Rollback

Returning traffic/workload to the previous version.

A deployment can succeed while a release still fails.

For example:

```text
Kubernetes:
Deployment successful

Application:
5xx increased to 20%
```

Therefore:

> **Kubernetes rollout success ≠ application success.**

---

# 3. Our Current Production Architecture

We have:

```text
Internet
   │
   ▼
Route 53
   │
   ▼
CloudFront / WAF
   │
   ▼
ALB
   │
   ▼
AWS Load Balancer Controller
   │
   ▼
Kubernetes Service
   │
   ▼
order-service Pods
```

For deployment:

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
EKS
```

For monitoring:

```text
Application
 ├── Metrics → Prometheus → Grafana
 ├── Logs → Fluent Bit → Logstash → Elasticsearch → Kibana
 └── Traces → OpenTelemetry → Jaeger
```

And production governance:

```text
Jira Change
   ↓
Approval
   ↓
GitOps PR
```

---

# 4. Strategy 1 — Rolling Update

Rolling Update is the normal Kubernetes Deployment strategy.

Suppose we have:

```text
3 Pods
```

Current:

```text
v1
v1
v1
```

We deploy v2.

Kubernetes gradually replaces them:

```text
v2
v1
v1
```

then:

```text
v2
v2
v1
```

then:

```text
v2
v2
v2
```

Traffic continues during the rollout.

---

# 5. Rolling Update Architecture

```text
                 Service
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        v2 Pod     v1 Pod    v1 Pod
          │
          │
       new Pod
```

As new Pods become Ready, old Pods are removed.

---

# 6. Kubernetes Rolling Configuration

Our Helm Deployment can contain:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

This means:

### `maxSurge: 1`

Kubernetes can temporarily create one additional Pod.

### `maxUnavailable: 0`

Kubernetes should not intentionally reduce available replicas during the rollout.

For:

```yaml
replicaCount: 3
```

you can temporarily have:

```text
4 Pods
```

during the rollout.

---

# 7. Why Readiness Probes Matter

Suppose Kubernetes creates:

```text
order-service-v2
```

but the application hasn't finished starting.

Without a proper readiness probe, traffic could reach it too early.

Our application has:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

Therefore:

```text
Pod created
   ↓
Application starts
   ↓
Readiness check
   ↓
Ready
   ↓
Traffic
```

---

# 8. Rolling Update Failure

Suppose:

```text
v1 v1 v1
```

and v2 starts failing.

We may see:

```text
v2
v1
v1
```

but v2 never becomes Ready.

Kubernetes does not continue replacing healthy v1 Pods indefinitely.

This helps limit the blast radius.

---

# 9. Rolling Update Rollback

Check:

```bash
kubectl rollout history \
  deployment/order-service \
  -n ecommerce-prod
```

You can inspect:

```bash
kubectl rollout history \
  deployment/order-service \
  -n ecommerce-prod \
  --revision=5
```

Emergency Kubernetes rollback:

```bash
kubectl rollout undo \
  deployment/order-service \
  -n ecommerce-prod
```

But remember our GitOps rule:

> After emergency rollback, correct GitOps so Git and Kubernetes agree.

Normal GitOps rollback:

```text
Git revert
   ↓
ArgoCD
   ↓
Kubernetes
```

---

# 10. Advantages of Rolling Update

```text
Low downtime
Low additional infrastructure
Native Kubernetes support
Simple operational model
Works well for most stateless services
```

---

# 11. Limitations

Rolling updates can be difficult when:

```text
v1 and v2 are incompatible
Database schema changes are unsafe
Long startup times exist
Large stateful workloads are involved
You need instant traffic switching
```

For those situations, other strategies may be more suitable.

---

# 12. Strategy 2 — Recreate

Recreate is much simpler.

Kubernetes does:

```text
v1
v1
v1
```

then:

```text
STOP v1
```

then:

```text
v2
v2
v2
```

There is a period with no application Pods.

---

# 13. Recreate Configuration

```yaml
strategy:
  type: Recreate
```

Kubernetes removes the existing Pods before creating the new version.

---

# 14. Recreate Timeline

```text
v1 v1 v1
    │
    ▼
   STOP
    │
    ▼
(no Pods)
    │
    ▼
v2
v2
v2
```

This creates downtime.

---

# 15. When Recreate Can Make Sense

It can be appropriate when:

* only one version can safely run
* application versions cannot coexist
* shared local state prevents parallel versions
* downtime is explicitly acceptable
* the workload is non-critical

It is generally not the default choice for a customer-facing stateless API.

---

# 16. Strategy 3 — Blue-Green

Now we move to a more advanced model.

We have two environments:

```text
BLUE
GREEN
```

Suppose BLUE is currently live:

```text
BLUE
v1
v1
v1
```

GREEN contains:

```text
GREEN
v2
v2
v2
```

Users are currently sent to BLUE.

---

# 17. Blue-Green Architecture

```text
                         ALB
                          │
                          ▼
                     Production
                      Service
                          │
                     ┌────┴────┐
                     │         │
                   BLUE      GREEN
                    v1         v2
```

Initially:

```text
Traffic → BLUE
```

After validation:

```text
Traffic → GREEN
```

---

# 18. Why Blue-Green Is Powerful

You can deploy v2 completely before sending production traffic to it.

For example:

```text
BLUE
v1
 │
 │ 100% traffic
 ▼
Users


GREEN
v2
 │
 │ 0% traffic
 ▼
Testing
```

Test GREEN.

If everything is correct:

```text
BLUE → 0%
GREEN → 100%
```

---

# 19. Blue-Green Rollback

Suppose GREEN has a problem.

Simply switch traffic back:

```text
GREEN → 0%
BLUE → 100%
```

That can make rollback very fast.

---

# 20. Blue-Green With Kubernetes Services

One basic implementation is:

```text
service:
  order-service
```

with Pods labeled:

```text
version: blue
```

and:

```text
version: green
```

The Service selector determines which version receives traffic.

For example:

```yaml
selector:
  app: order-service
  version: blue
```

Switching to:

```yaml
selector:
  app: order-service
  version: green
```

changes the destination.

---

# 21. Blue-Green With ALB

Another architecture can use AWS Load Balancer Controller and routing rules.

Conceptually:

```text
ALB
 │
 ├── BLUE target
 │
 └── GREEN target
```

Traffic can be directed to the appropriate target group.

This gives us more sophisticated traffic management.

---

# 22. Blue-Green Disadvantage

You need extra capacity.

If production normally needs:

```text
10 Pods
```

you may temporarily need:

```text
10 BLUE
+
10 GREEN
```

That's:

```text
20 Pods
```

plus associated resources.

Therefore Blue-Green can cost more than Rolling Update.

---

# 23. Database Problem With Blue-Green

This is very important.

Suppose:

```text
BLUE = v1
GREEN = v2
```

Both may access the same database.

If v2 changes the schema incompatibly:

```text
GREEN → new schema
BLUE → old schema
```

then switching back to BLUE may fail.

Therefore Blue-Green requires:

> **Backward-compatible database changes.**

This is why we introduced expand-and-contract migrations earlier.

---

# 24. Strategy 4 — Canary Deployment

Canary means:

> Send a small percentage of traffic to the new version before sending all traffic.

For example:

```text
v1 = 100%
v2 = 0%
```

Then:

```text
v1 = 95%
v2 = 5%
```

Then:

```text
v1 = 90%
v2 = 10%
```

Then:

```text
v1 = 75%
v2 = 25%
```

Then:

```text
v1 = 50%
v2 = 50%
```

Finally:

```text
v1 = 0%
v2 = 100%
```

---

# 25. Canary Architecture

```text
                    ALB
                     │
                     ▼
              Traffic Router
               /           \
              /             \
             ▼               ▼
          Stable           Canary
            v1               v2
           90%              10%
```

---

# 26. Why Canary Is Useful

Suppose v2 has a bug that only appears under real traffic.

With Rolling Update:

```text
v2 gradually becomes the entire deployment.
```

With Canary:

```text
Only 5% of traffic initially reaches v2.
```

If error rate increases:

```text
STOP
```

and return traffic to v1.

This can reduce the blast radius.

---

# 27. Canary Isn't Automatically Safe

Canary is only useful if you have good validation.

You need metrics such as:

```text
Error rate
Latency
Request rate
HTTP 5xx
CPU
Memory
Dependency failures
Business metrics
```

For our platform:

```text
Prometheus
    ↓
Grafana
    ↓
Canary analysis
```

---

# 28. Example Canary Analysis

Suppose:

```text
Stable v1:
5xx = 0.2%

Canary v2:
5xx = 0.3%
```

Continue.

But:

```text
Stable v1:
5xx = 0.2%

Canary v2:
5xx = 8%
```

Stop the rollout.

Potential action:

```text
v2 traffic → 0%
```

and investigate.

---

# 29. Canary Steps

A production rollout might be:

```text
Step 1:
5%

Step 2:
10%

Step 3:
25%

Step 4:
50%

Step 5:
100%
```

At each stage:

```text
Deploy
 ↓
Wait
 ↓
Analyze metrics
 ↓
Promote or abort
```

---

# 30. Strategy 5 — Argo Rollouts

Now we introduce:

**Argo Rollouts**

This is not simply another traffic strategy.

It is a Kubernetes controller that provides advanced rollout capabilities such as:

* Canary
* Blue-Green
* Progressive delivery
* Automated analysis
* Traffic management
* Pause/resume
* Automated rollback

So think:

```text
Canary
    +
Automated analysis
    +
Traffic control
    +
Rollback
    ↓
Argo Rollouts
```

---

# 31. Kubernetes Deployment vs Argo Rollout

Normal Kubernetes:

```yaml
kind: Deployment
```

Argo Rollouts:

```yaml
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
        - setWeight: 10
        - pause:
            duration: 2m

        - setWeight: 25
        - pause:
            duration: 5m

        - setWeight: 50
        - pause:
            duration: 10m

        - setWeight: 100
```

This is a conceptual starting point; production traffic routing and analysis configuration must also be defined.

---

# 32. What `setWeight` Means

```yaml
- setWeight: 10
```

means:

```text
Approximately 10%
```

of managed traffic goes to the canary, assuming the selected traffic-routing integration is configured.

Then:

```yaml
- setWeight: 25
```

moves toward:

```text
25%
```

and so on.

---

# 33. Pause

Example:

```yaml
- pause:
    duration: 5m
```

means:

```text
Deploy
 ↓
10% traffic
 ↓
wait 5 minutes
 ↓
analyze
 ↓
continue or abort
```

This creates a controlled rollout.

---

# 34. Automated Analysis

This is where Argo Rollouts becomes particularly useful.

We can define an analysis template.

Conceptually:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: order-service-success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 2
```

The actual PromQL query and thresholds must be designed from the application's SLOs and traffic patterns.

---

# 35. Prometheus Integration

Conceptually:

```text
Canary
  │
  ▼
Prometheus
  │
  ├── error rate
  ├── latency
  └── request rate
  │
  ▼
Argo Rollouts
  │
  ├── PASS → continue
  │
  └── FAIL → abort
```

This creates:

> **metrics-driven deployment.**

---

# 36. Example

Suppose canary starts:

```text
10%
```

Argo Rollouts checks:

```text
5xx < 1%
P95 latency < 500ms
```

If:

```text
5xx = 0.4%
P95 = 220ms
```

then:

```text
PASS
```

Move to:

```text
25%
```

But if:

```text
5xx = 7%
```

then:

```text
FAIL
```

Abort.

---

# 37. Automated Rollback

Conceptually:

```text
Canary
  ↓
10%
  ↓
Analysis
  ↓
FAIL
  ↓
Abort
  ↓
Stable version remains serving traffic
```

This is much more powerful than waiting for someone to manually notice an alert.

---

# 38. Traffic Management

Canary becomes much more useful when traffic can actually be controlled.

Possible mechanisms include:

```text
AWS Load Balancer Controller
Istio
NGINX
Gateway API implementations
other supported traffic-routing integrations
```

For our project, we'll first focus on AWS/EKS-native routing where practical and introduce additional service-mesh complexity only when it adds value.

---

# 39. Simple Canary vs Advanced Canary

### Simple

```text
Kubernetes replicas
```

You might have:

```text
v1 = 9 Pods
v2 = 1 Pod
```

But that does **not necessarily mean exactly 10% of user traffic**.

Requests are distributed among available endpoints according to the routing behavior.

Therefore:

> **Pod percentage ≠ guaranteed traffic percentage.**

---

# 40. True Traffic-Based Canary

For actual traffic percentages:

```text
ALB / traffic router
       │
       ├── Stable → 90%
       │
       └── Canary → 10%
```

Now we can intentionally control traffic distribution.

This is a key distinction.

---

# 41. Rolling vs Blue-Green vs Canary

| Strategy             | Downtime     | Extra capacity | Rollback  | Complexity |
| -------------------- | ------------ | -------------- | --------- | ---------- |
| Rolling              | Usually none | Low            | Moderate  | Low        |
| Recreate             | Yes          | Low            | Slower    | Low        |
| Blue-Green           | Usually none | High           | Fast      | Medium     |
| Canary               | Usually none | Medium         | Fast      | High       |
| Argo Rollouts Canary | Usually none | Medium         | Automated | High       |

These are descriptive trade-offs, not universal rules; actual behavior depends on application architecture and traffic setup.

---

# 42. Rolling Update

Use when:

```text
Stateless service
Backward-compatible release
Normal application deployment
```

Our initial default:

```text
order-service
```

can use Rolling Update.

---

# 43. Recreate

Use when:

```text
Two versions cannot safely coexist
```

and:

```text
Downtime is acceptable
```

It should not be our default for customer-facing APIs.

---

# 44. Blue-Green

Useful when:

```text
Fast rollback is important
Full new version validation is required
Extra capacity is available
```

Example:

```text
Large application release
```

---

# 45. Canary

Useful when:

```text
Release risk is high
Real traffic validation matters
Strong observability exists
```

Example:

```text
Payment-related service
High-volume API
Major application change
```

---

# 46. Argo Rollouts

Useful when:

```text
You need:
progressive delivery
+
automated metric analysis
+
controlled traffic
+
automated abort/rollback
```

This is particularly useful for mature platform environments.

---

# 47. Jira Change Fits Above the Strategy

Regardless of the deployment strategy:

```text
Jira Change
    │
    ▼
GitOps
    │
    ▼
ArgoCD
    │
    ▼
Deployment Strategy
```

For example:

### Rolling

```text
Jira
 ↓
GitOps
 ↓
ArgoCD
 ↓
Rolling Deployment
```

### Blue-Green

```text
Jira
 ↓
GitOps
 ↓
ArgoCD
 ↓
Blue-Green
```

### Canary

```text
Jira
 ↓
GitOps
 ↓
ArgoCD
 ↓
Argo Rollouts
 ↓
Canary
```

---

# 48. Observability Is Required

For Rolling:

```text
Metrics
Logs
Alerts
```

are important.

For Canary:

```text
Metrics
Logs
Alerts
```

become even more important because the rollout decision may depend directly on them.

Our architecture:

```text
                    Application
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Metrics      Logs      Traces
             │          │          │
             ▼          ▼          ▼
        Prometheus      ELK       Jaeger
             │
             ▼
           Grafana
             │
             ▼
      Argo Rollouts
```

---

# 49. Production Release Example

Let's say we're deploying:

```text
order-service
8f31a2c
sha256:ABC
```

Jira:

```text
CHG-10452
```

GitOps:

```text
prod/order-service/values.yaml
```

ArgoCD:

```text
order-service-prod
```

Strategy:

```text
Canary
```

Flow:

```text
CHG-10452
     ↓
GitOps merge
     ↓
ArgoCD
     ↓
Argo Rollouts
     ↓
10%
     ↓
Prometheus analysis
     ↓
25%
     ↓
Prometheus analysis
     ↓
50%
     ↓
Prometheus analysis
     ↓
100%
```

---

# 50. Canary Failure Example

At 25%:

```text
Error rate:
8%
```

Argo Rollouts:

```text
Analysis:
FAILED
```

Then:

```text
Canary
   ↓
ABORT
   ↓
Traffic returns to stable
```

Then:

```text
Jira
   ↓
Update change
   ↓
Deployment failed
   ↓
Incident investigation
```

This connects deployment strategy with our incident-management architecture.

---

# 51. Important: Deployment Strategy Does Not Replace Testing

Canary does not mean:

```text
We don't need QA.
```

Instead:

```text
CI
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
Production Canary
```

Each layer catches different problems.

---

# 52. Important: Deployment Strategy Does Not Replace Rollback

Every strategy should have a rollback plan.

### Rolling

```text
Git revert
```

### Blue-Green

```text
Switch traffic to BLUE
```

### Canary

```text
Abort canary
```

### Recreate

```text
Deploy previous version
```

The exact mechanics differ.

---

# 53. Database Compatibility Applies to Every Strategy

This is one of the biggest production considerations.

Regardless of:

```text
Rolling
Blue-Green
Canary
```

you need compatible database changes.

For example:

```text
v1
 │
 ├── old schema
 │
 ▼
Migration
 │
 ▼
compatible schema
 │
 ▼
v2
```

Use expand-and-contract where appropriate.

---

# 54. Our Recommended Implementation Sequence

We shouldn't implement all strategies simultaneously.

We'll implement them progressively.

```text
80 — Production Deployment Strategies
       │
       ▼
81 — Rolling Deployment Implementation
       │
       ▼
82 — Blue-Green Deployment Implementation
       │
       ▼
83 — Canary Deployment Implementation
       │
       ▼
84 — Argo Rollouts Implementation
```

This lets us understand each strategy practically.

---

# 55. First Strategy for Order-Service

Our first implementation should be:

```text
Rolling Update
```

because it is already supported by Kubernetes Deployments and fits the current Helm/ArgoCD architecture.

We'll configure:

```text
replicas
maxSurge
maxUnavailable
readinessProbe
livenessProbe
startupProbe
PDB
graceful shutdown
```

Then we'll deliberately simulate:

```text
successful deployment
failed deployment
slow startup
Pod failure
rollback
```

---

# 56. Later: Blue-Green

We'll then build:

```text
BLUE
order-service-v1

GREEN
order-service-v2
```

and practice:

```text
deploy green
 ↓
validate green
 ↓
switch traffic
 ↓
monitor
 ↓
rollback
```

---

# 57. Later: Canary

Then:

```text
v1 = 100%
```

to:

```text
v1 = 90%
v2 = 10%
```

then:

```text
v1 = 75%
v2 = 25%
```

then:

```text
v1 = 50%
v2 = 50%
```

and finally:

```text
v2 = 100%
```

with Prometheus-based validation.

---

# 58. Later: Argo Rollouts

Finally we'll replace the standard Deployment rollout mechanism where appropriate with:

```text
Rollout
AnalysisTemplate
TrafficRouting
```

and connect:

```text
Argo Rollouts
     ↓
Prometheus
     ↓
Analysis
     ↓
Promotion / Abort
```

---

# 59. Final Comparison

```text
                 ROLLING
                    │
              simplest default
                    │
                    ▼
              BLUE-GREEN
                    │
             fast rollback
                    │
                    ▼
                 CANARY
                    │
             gradual traffic
                    │
                    ▼
             ARGO ROLLOUTS
                    │
        automated progressive delivery
```

Think of them as increasing levels of deployment control and operational complexity—not as a strict hierarchy where one is always preferable.

---

# 60. Final Architecture

Our production platform now looks like:

```text
                         JIRA
                   Change Request
                         │
                      Approval
                         │
                         ▼
                      GitHub
                         │
                         ▼
                  GitHub Actions
                         │
                  Test / Scan / Build
                         │
                         ▼
                        ECR
                         │
                    Image Digest
                         │
                         ▼
                       DEV
                         │
                         ▼
                        QA
                         │
                         ▼
                     STAGING
                         │
                    Validation
                         │
                         ▼
                    PROD GATE
                         │
                         ▼
                       GitOps
                         │
                         ▼
                      ArgoCD
                         │
               ┌─────────┴─────────┐
               │                   │
          Kubernetes            Argo Rollouts
          Deployment            (advanced)
               │                   │
               ▼                   ▼
            Rolling          Canary/Blue-Green
               │                   │
               └─────────┬─────────┘
                         ▼
                       EKS
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Metrics       Logs       Traces
        Prometheus       ELK        OTel
             │
             ▼
           Grafana
```

## Chapter 80 outcome

You now have the complete conceptual model for:

* **Rolling Update**
* **Recreate**
* **Blue-Green**
* **Canary**
* **Argo Rollouts**
* traffic shifting
* automated analysis
* rollback
* database compatibility
* Jira change management
* GitOps/ArgoCD integration
* observability during deployments

For our actual implementation, we'll **start with Rolling Update**, prove it end-to-end, deliberately break it, recover it, and then move to Blue-Green.

**Next: `81-Rolling-Deployment-Implementation.md`** — a hands-on implementation for `order-service` on our EKS cluster, including Helm configuration, `maxSurge`, `maxUnavailable`, probes, PDB, graceful shutdown, ArgoCD rollout, testing, failure simulation, monitoring, and GitOps rollback.
