# 159 — Bad Production Deployment Production Incident

> **Phase 18 — Incident Management**  
> **Incident #159**  
> **Environment:** Production  
> **Platform:** Amazon EKS + ArgoCD + GitHub Actions + ECR  
> **Example affected service:** `order-service`

---

# 1. Objective

This chapter explains how to handle a **bad production deployment** where a newly released application version causes customer impact.

A bad deployment can be caused by:

- application bug
- incorrect environment variable
- bad database migration
- incompatible API change
- incorrect image
- wrong Helm values
- resource configuration
- readiness/liveness probe issue
- missing secret
- dependency incompatibility
- performance regression
- security configuration
- incorrect feature flag
- deployment configuration error

The key production principle is:

> **First stabilize the customer impact, then determine whether rollback or forward-fix is safer.**

---

# 2. Production Deployment Architecture

Our deployment path is:

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
    ├── Unit Tests
    ├── SonarQube
    ├── Dependency Scan
    ├── Trivy
    └── Docker Build
    │
    ▼
ECR
    │
    ▼
GitOps Repository
    │
    ▼
ArgoCD
    │
    ▼
EKS
    │
    ▼
Production
```

The deployment should use an immutable image:

```text
order-service@sha256:<digest>
```

rather than:

```text
order-service:latest
```

---

# 3. What Is a Bad Deployment?

A bad deployment occurs when a newly released version causes:

```text
Availability degradation
Latency degradation
5xx errors
Incorrect business behavior
Resource exhaustion
Security problems
Data integrity problems
```

Example:

```text
Version 1.8.0
     ↓
Healthy

Version 1.9.0
     ↓
Deployment
     ↓
5xx ↑
Latency ↑
     ↓
Customer impact
```

---

# 4. Typical Symptoms

You may see:

```text
HTTP 5xx ↑
P95/P99 latency ↑
CrashLoopBackOff
OOMKilled
CPU ↑
Memory ↑
RDS connections ↑
Redis errors ↑
Kafka lag ↑
SQS backlog ↑
ALB target failures
```

The important clue is often:

```text
The problem started immediately after deployment.
```

---

# 5. First Incident Question

Ask:

> **What changed immediately before the incident?**

Check:

```text
Deployment
Image
Helm values
ConfigMap
Secret
Database migration
Infrastructure
NetworkPolicy
IAM
Feature flag
Dependency
```

Recent changes are one of the highest-value signals during incident investigation.

---

# 6. Deployment Detection

Prometheus/Grafana might show:

```text
14:00
Normal

14:05
Deployment starts

14:07
5xx increases

14:08
P95 latency increases

14:09
Incident declared
```

This correlation is extremely important.

---

# 7. Check Deployment

```bash
kubectl get deployment \
  order-service \
  -n ecommerce
```

Example:

```text
NAME            READY   UP-TO-DATE   AVAILABLE
order-service   5/10    10           5
```

Only five Pods are currently available.

---

# 8. Check Rollout Status

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce
```

Possible result:

```text
Waiting for deployment "order-service" rollout to finish...
```

This indicates the deployment may not be progressing normally.

---

# 9. Check Rollout History

```bash
kubectl rollout history \
  deployment/order-service \
  -n ecommerce
```

Example:

```text
REVISION  CHANGE-CAUSE
10        order-service 1.7.0
11        order-service 1.8.0
12        order-service 1.9.0
```

We can identify the previous known-good revision.

---

# 10. Check ReplicaSets

```bash
kubectl get rs \
  -n ecommerce \
  -l app=order-service
```

Example:

```text
NAME                     DESIRED   CURRENT   READY
order-service-abc123     5         5         5
order-service-def456     5         5         0
```

This may indicate:

```text
Old version → healthy
New version → unhealthy
```

---

# 11. Check Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=order-service \
  -o wide
```

Look for:

```text
Running
Pending
CrashLoopBackOff
ImagePullBackOff
OOMKilled
NotReady
Terminating
```

---

# 12. Check Pod Logs

```bash
kubectl logs \
  <pod> \
  -n ecommerce
```

For previous container:

```bash
kubectl logs \
  <pod> \
  -n ecommerce \
  --previous
```

Look for:

```text
Exception
Connection failure
Authentication failure
SQL error
Migration error
Timeout
NullPointerException
OutOfMemoryError
Configuration error
```

---

# 13. Check Pod Events

```bash
kubectl describe pod \
  <pod> \
  -n ecommerce
```

Look at:

```text
Events
Readiness probe
Liveness probe
Startup probe
Image
Mounts
Secrets
ConfigMaps
Resources
```

---

# 14. Check Application Version

```bash
kubectl get deployment \
  order-service \
  -n ecommerce \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:ABC123
```

Now identify:

```text
Git commit
CI build
ECR image
GitOps commit
ArgoCD deployment
```

---

# 15. Deployment Traceability

We want:

```text
Git Commit
    ↓
CI Run
    ↓
Docker Image
    ↓
ECR Digest
    ↓
GitOps Commit
    ↓
ArgoCD Revision
    ↓
Kubernetes Deployment
    ↓
Pod
```

Example:

```text
Git:
8f3a2c1

CI:
#4821

ECR:
sha256:abc...

GitOps:
f1d91ab

ArgoCD:
synced revision f1d91ab

EKS:
order-service Pod
```

This gives us complete deployment traceability.

---

# 16. Golden Signals

During a bad deployment check:

```text
Latency
Traffic
Errors
Saturation
```

Example:

```text
Traffic:
1000 RPS → 1050 RPS

P95:
180ms → 2.4s

5xx:
0.2% → 8.5%

CPU:
55% → 92%
```

This strongly suggests deployment-related degradation.

---

# 17. Check Logs in ELK

Search:

```text
service.name:"order-service"
```

Then filter:

```text
environment:"prod"
```

and:

```text
version:"1.9.0"
```

Compare:

```text
1.8.0
vs
1.9.0
```

This is much more useful than looking at logs without version context.

---

# 18. Check Distributed Tracing

Use Jaeger/OpenTelemetry.

Compare:

```text
order-service 1.8.0
```

and:

```text
order-service 1.9.0
```

Look for:

```text
DB latency
Redis latency
HTTP dependency latency
Kafka latency
External API latency
Exception spans
```

Example:

```text
POST /api/v1/orders
        │
        ├── customer-service 120ms
        ├── product-service 100ms
        ├── pricing-service 90ms
        ├── PostgreSQL 2.1s  ← regression
        └── response
```

---

# 19. Deployment Could Be the Symptom, Not Root Cause

Do not automatically blame the application.

Example:

```text
Deployment
   ↓
New Pods
   ↓
RDS connections increase
   ↓
RDS connection exhaustion
   ↓
API 5xx
```

The application may have changed:

```text
Hikari maximumPoolSize:
10 → 50
```

The actual failure is:

```text
Database capacity exceeded
```

Therefore the investigation must continue into dependencies.

---

# 20. Database Migration Failure

A new version may require:

```text
V15__add_payment_reference.sql
```

but production migration may fail.

Flow:

```text
Deployment
    ↓
Application starts
    ↓
Database schema expected
    ↓
Column missing
    ↓
SQL exception
    ↓
Pod unhealthy
```

Never assume:

```text
Deployment failed = application code bug
```

It may be:

```text
Application + schema compatibility problem
```

---

# 21. Expand-and-Contract

For important database changes:

### Phase 1

Add new column/table:

```text
Old code works
New code works
```

### Phase 2

Deploy application using new schema.

### Phase 3

Migrate data.

### Phase 4

Remove old schema only after all old versions are gone.

This makes rollback safer.

---

# 22. Why Database Changes Make Rollback Difficult

Suppose version 1.9.0 performs:

```sql
ALTER TABLE orders
DROP COLUMN old_status;
```

Then we discover:

```text
1.9.0 is broken
```

Rolling application back to:

```text
1.8.0
```

may not work because:

```text
1.8.0 expects old_status
```

Therefore:

> **Application rollback is not automatically database rollback.**

Database migrations need their own compatibility strategy.

---

# 23. Deployment Strategies

Our standard strategy can be:

```text
Rolling Deployment
```

For higher-risk services:

```text
Canary
```

or:

```text
Blue/Green
```

---

# 24. Rolling Deployment

Example:

```text
Old:
10 Pods

Deployment
    ↓
Create new Pod
    ↓
New Pod healthy
    ↓
Terminate old Pod
    ↓
Repeat
```

Architecture:

```text
Old Old Old Old Old
 ↓
New
 ↓
Old Old Old New New
 ↓
...
 ↓
New New New New New
```

---

# 25. Canary Deployment

Canary:

```text
Old version = 90%
New version = 10%
```

Monitor:

```text
5xx
P95
business errors
CPU
RDS
Redis
```

If healthy:

```text
10%
 ↓
25%
 ↓
50%
 ↓
100%
```

If unhealthy:

```text
10%
 ↓
Rollback
```

---

# 26. Why Canary Is Safer

Suppose new code has a bug affecting:

```text
10% of requests
```

With a full deployment:

```text
100% traffic
```

may be affected.

With a 5–10% canary:

```text
5–10% traffic
```

is exposed initially.

This reduces blast radius.

---

# 27. Bad Production Deployment — Immediate Response

Use:

```text
Detect
 ↓
Confirm customer impact
 ↓
Identify recent deployment
 ↓
Stop further rollout
 ↓
Determine blast radius
 ↓
Check rollback safety
 ↓
Rollback OR forward-fix
 ↓
Validate
 ↓
Monitor
 ↓
RCA
```

---

# 28. Stop the Rollout

If the rollout is actively progressing:

```bash
kubectl rollout pause \
  deployment/order-service \
  -n ecommerce
```

But remember:

> In GitOps, the long-term desired state should be corrected in Git.

A Kubernetes-only pause is not a permanent GitOps solution.

---

# 29. GitOps Rollback

Suppose GitOps changed:

```text
1.8.0
```

to:

```text
1.9.0
```

If 1.8.0 is known-good:

```text
GitOps:
1.9.0 → 1.8.0
```

Create a revert PR.

Example:

```bash
git revert <bad-commit>
```

Then:

```text
PR
 ↓
Review
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

---

# 30. Why GitOps Rollback Is Preferred

If you only run:

```bash
kubectl rollout undo
```

Kubernetes may temporarily return to the previous ReplicaSet.

But Git still says:

```text
1.9.0
```

ArgoCD can later reconcile:

```text
1.8.0 → 1.9.0
```

and reintroduce the bad version.

Therefore:

> **When ArgoCD owns the deployment, rollback the Git desired state.**

---

# 31. When `kubectl rollout undo` Can Be Useful

During an emergency, if immediate customer protection is necessary:

```bash
kubectl rollout undo \
  deployment/order-service \
  -n ecommerce
```

may be used as a temporary mitigation.

But then:

```text
Immediately reconcile Git
```

so:

```text
Git = desired
Live = desired
```

again.

---

# 32. Rollback Decision

Use rollback when:

```text
Previous version is known-good
Application/schema are backward-compatible
No irreversible migration dependency
Rollback is faster than fixing forward
```

Forward-fix may be safer when:

```text
Database schema already changed
Data migration occurred
Previous version incompatible
Security vulnerability requires immediate fix
Rollback would create greater risk
```

---

# 33. Rollback Decision Tree

```text
Bad Deployment
      │
      ▼
Is customer impact occurring?
      │
      ├── No → Investigate normally
      │
      ▼
Is previous version known-good?
      │
      ├── No → Forward-fix
      │
      ▼
Is DB/schema backward-compatible?
      │
      ├── No → Forward-fix / compatibility fix
      │
      ▼
Can rollback safely reduce impact?
      │
      ├── Yes → GitOps rollback
      │
      └── No → Forward-fix
```

---

# 34. Resource Regression

A new version may change:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

from:

```text
250m / 256Mi
```

This can cause:

```text
Node pressure
Pending Pods
Karpenter scaling
Cost increase
```

Therefore deployment validation should include resource behavior.

---

# 35. Readiness Probe Regression

Bad deployment:

```text
Application starts
   ↓
/actuator/health/readiness
   ↓
Returns 503
```

Result:

```text
Pod Running
but
Pod NotReady
```

ALB/Service will not send traffic to it.

This can cause:

```text
Available replicas ↓
5xx ↑
```

---

# 36. Liveness Probe Regression

Bad liveness configuration:

```yaml
livenessProbe:
  httpGet:
    path: /health
```

Suppose `/health` depends on:

```text
RDS
Redis
Kafka
```

and RDS becomes temporarily slow.

Then:

```text
RDS slow
 ↓
Health endpoint slow
 ↓
Liveness fails
 ↓
Pod restarted
 ↓
More connections
 ↓
RDS pressure
```

This creates a restart storm.

Liveness should generally verify whether the process itself is alive, not indiscriminately depend on every downstream service.

---

# 37. Configuration Regression

Example:

```text
DB_URL
```

changed from:

```text
jdbc:postgresql://orders-db...
```

to:

```text
jdbc:postgresql://order-db...
```

Result:

```text
UnknownHostException
```

Pods may enter:

```text
CrashLoopBackOff
```

The deployment itself succeeded technically.

The configuration was wrong.

---

# 38. Secret Regression

A deployment may reference:

```text
DB_PASSWORD
```

but the Secret contains:

```text
POSTGRES_PASSWORD
```

Application starts:

```text
Missing required configuration
```

Pod crashes.

This is why CI/CD should validate:

- required environment variables
- Helm values
- Secret references
- configuration schema

before production.

---

# 39. Image Regression

CI could push:

```text
order-service:1.9.0
```

but the image might contain:

- wrong build
- wrong branch
- incorrect Dockerfile
- missing dependency
- incorrect Java version
- wrong architecture

Use:

```text
Git SHA
+
ECR digest
```

for traceability.

---

# 40. Security Scan Does Not Guarantee Correctness

Suppose:

```text
Trivy = PASS
SonarQube = PASS
```

The deployment can still be bad.

Why?

Security/static analysis may not detect:

```text
Business logic bug
Incorrect SQL
Wrong environment variable
Bad configuration
Latency regression
Incorrect API behavior
```

Therefore:

```text
Security scans
+
Tests
+
Integration tests
+
Staging
+
Canary
+
Production monitoring
```

are all necessary.

---

# 41. Production Deployment Gates

Recommended flow:

```text
Developer
   ↓
PR
   ↓
Unit Tests
   ↓
SonarQube
   ↓
Dependency Scan
   ↓
Trivy
   ↓
Build Image
   ↓
ECR
   ↓
DEV
   ↓
Integration Tests
   ↓
QA
   ↓
STAGING
   ↓
Performance/Smoke Tests
   ↓
Production Approval
   ↓
Canary/Rolling
   ↓
Production
```

---

# 42. Production Deployment Verification

After deployment:

```text
1. Pod health
2. Deployment health
3. Service endpoints
4. ALB target health
5. HTTP 5xx
6. Latency
7. CPU
8. Memory
9. RDS connections
10. Redis
11. Kafka/SQS
12. Logs
13. Traces
14. Business transaction
```

Do not stop at:

```text
kubectl rollout status = successful
```

A technically successful rollout can still be a business failure.

---

# 43. Business Validation

For `order-service`:

```text
Login
 ↓
Product
 ↓
Cart
 ↓
Create Order
 ↓
Payment
 ↓
Inventory
 ↓
Notification
```

Test a real transaction.

Example:

```text
POST /api/v1/orders
```

Validate:

```text
Order created
Payment processed
Inventory reserved
Event published
Notification generated
```

---

# 44. Example Bad Deployment

Version:

```text
order-service 1.9.0
```

contains a code change:

```java
order.setStatus(null);
```

This causes database constraint failure.

Production symptoms:

```text
HTTP 500 ↑
PostgreSQL errors ↑
Order creation failures ↑
```

The Pods are:

```text
Running
Ready
```

Therefore:

```text
Kubernetes health = Healthy
```

but:

```text
Business health = Broken
```

This is a critical lesson.

---

# 45. Example RCA — Bad Production Deployment

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-159` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 16:10–16:34 IST |
| Service | `order-service` |
| Bad Version | `1.9.0` |
| Previous Version | `1.8.0` |
| Root Cause | Application regression |
| Customer Impact | Order creation failures |
| Data Loss | No |

---

# 46. Customer Impact

After version `1.9.0` was deployed:

```text
Order API 5xx:
0.2% → 18%
```

P95 latency:

```text
180ms → 1.8s
```

Customers could browse products and add items to carts, but a significant percentage of new order requests failed.

No successful payment was recorded without a corresponding order.

---

# 47. Timeline

```text
16:00  Production healthy

16:05  order-service 1.9.0 deployment begins

16:07  10% of Pods running 1.9.0

16:08  5xx begins increasing

16:09  P95 latency increases

16:10  Production alert fires

16:11  SEV-1 declared

16:12  Deployment identified as recent change

16:14  ELK shows PostgreSQL constraint errors

16:16  Jaeger confirms failures during order creation

16:18  Application team identifies status-field regression

16:19  Rollback safety checked

16:21  GitOps revert PR created

16:23  PR approved

16:24  ArgoCD sync begins

16:27  1.8.0 Pods become healthy

16:29  5xx drops

16:31  Order transaction validated

16:34  Incident resolved
```

---

# 48. Investigation

The deployment itself was healthy:

```text
Pods:
Running

Readiness:
Passing

CPU:
Normal
```

But:

```text
HTTP 500:
High
```

Logs showed:

```text
DataIntegrityViolationException
```

and PostgreSQL constraint errors.

Tracing showed:

```text
POST /api/v1/orders
       ↓
OrderService
       ↓
PostgreSQL
       ↓
Constraint violation
```

This proved the issue was application/data logic rather than Kubernetes infrastructure.

---

# 49. Root Cause

A change in `order-service` version `1.9.0` incorrectly allowed an order status field to become `NULL`.

The database schema required the field to be non-null.

Therefore:

```text
Application regression
        ↓
Invalid entity
        ↓
PostgreSQL constraint violation
        ↓
Transaction failure
        ↓
HTTP 500
```

---

# 50. Amplification Mechanism

The deployment used a rolling strategy.

As more Pods moved to:

```text
1.9.0
```

more requests were handled by the broken code.

Therefore:

```text
10% new Pods
   ↓
small error increase

50% new Pods
   ↓
larger error increase

100% new Pods
   ↓
full blast radius
```

A canary deployment would have reduced the impact.

---

# 51. Contributing Factors

1. Business-level integration test did not cover the failing status path.
2. Canary was not enabled for this service.
3. Deployment was technically healthy from Kubernetes perspective.
4. Monitoring focused heavily on infrastructure health.
5. Business error-rate alert detected the issue only after customer impact.
6. The application change lacked sufficient regression coverage.

---

# 52. What Went Well

- Prometheus detected increased 5xx.
- ELK provided the PostgreSQL exception.
- Jaeger identified the failing dependency path.
- GitOps made rollback traceable.
- Previous image was immutable and available in ECR.
- Database was healthy.
- No permanent data loss occurred.
- Rollback restored service quickly.

---

# 53. What Went Wrong

- Business regression reached production.
- Canary was not used.
- Integration test coverage was incomplete.
- Deployment health did not represent business health.
- The failure occurred on a critical customer workflow.

---

# 54. Immediate Mitigation

```text
1. Stop rollout
2. Confirm customer impact
3. Identify bad version
4. Confirm previous version
5. Validate DB compatibility
6. Revert GitOps change
7. ArgoCD sync
8. Monitor 5xx
9. Validate order transaction
10. Confirm recovery
```

---

# 55. Corrective Actions

| Action | Owner |
|---|---|
| Add order creation integration test | Backend |
| Add database constraint test | Backend |
| Enable canary for critical services | Platform |
| Add business SLO alert | SRE |
| Add deployment health dashboard | SRE |
| Add production smoke test | DevOps |
| Add automated rollback criteria | Platform |
| Review release checklist | Engineering |

---

# 56. Five Whys

### Why did order creation fail?

The application attempted to persist an invalid order status.

### Why was the status invalid?

Version `1.9.0` contained a regression.

### Why wasn't the regression detected?

The affected business path was not covered by the required integration test.

### Why did it reach many customers?

The deployment did not use a canary rollout.

### Why wasn't business impact detected before full rollout?

Deployment validation focused more on Kubernetes health than business-level success metrics.

---

# 57. Final Root Cause Statement

> The production incident was caused by a regression introduced in `order-service` version `1.9.0`, which generated an invalid order status and triggered PostgreSQL constraint violations. The impact was amplified because the service was deployed without a canary stage and the affected business workflow was not sufficiently covered by integration testing.

---

# 58. Improved Deployment Architecture

```text
Developer
    ↓
PR
    ↓
CI
 ├── Build
 ├── Unit Test
 ├── SonarQube
 ├── Dependency Scan
 ├── Trivy
 └── Docker
    ↓
ECR
    ↓
GitOps
    ↓
DEV
    ↓
Integration Tests
    ↓
QA
    ↓
STAGING
    ↓
Performance + Business Smoke
    ↓
Production Approval
    ↓
5% Canary
    ↓
Observe
 ├── 5xx
 ├── P95
 ├── Business Success
 ├── RDS
 ├── Redis
 └── Kafka
    ↓
25%
    ↓
50%
    ↓
100%
```

---

# 59. Automated Canary Decision

Conceptually:

```text
Canary
  ↓
5xx < threshold?
  │
  ├── No → Rollback
  │
  ▼
Latency acceptable?
  │
  ├── No → Rollback
  │
  ▼
Business success rate acceptable?
  │
  ├── No → Rollback
  │
  ▼
Dependency health acceptable?
  │
  ├── No → Rollback
  │
  ▼
Increase traffic
```

The thresholds must come from the service's SLOs and tested behavior rather than arbitrary universal numbers.

---

# 60. Bad Deployment Runbook

Create:

```text
runbooks/bad-production-deployment.md
```

Recommended procedure:

```text
1. Confirm customer impact
2. Identify recent deployment
3. Stop rollout
4. Check deployment status
5. Check ReplicaSets
6. Check Pods
7. Check application logs
8. Check Prometheus
9. Check ELK
10. Check Jaeger
11. Check RDS
12. Check Redis
13. Check Kafka/SQS
14. Determine root cause
15. Evaluate rollback safety
16. Revert GitOps if appropriate
17. ArgoCD sync
18. Validate infrastructure
19. Validate business flow
20. Monitor
21. RCA
```

---

# 61. Rollback Checklist

Before rollback:

- [ ] Previous image exists
- [ ] Previous version known-good
- [ ] Database schema compatible
- [ ] No irreversible migration dependency
- [ ] Secrets compatible
- [ ] API compatibility checked
- [ ] Event schema compatibility checked
- [ ] Rollback owner identified

After rollback:

- [ ] Pods healthy
- [ ] Readiness passing
- [ ] ALB healthy
- [ ] 5xx normal
- [ ] Latency normal
- [ ] RDS healthy
- [ ] Redis healthy
- [ ] Kafka/SQS healthy
- [ ] Business transaction successful
- [ ] Git and live state synchronized

---

# 62. DevOps vs Application Team

### DevOps / Platform

Owns:

```text
CI/CD
ECR
ArgoCD
EKS
Helm
Deployment strategy
Rollout infrastructure
Canary platform
Monitoring
Rollback process
```

### Application Team

Owns:

```text
Application code
Unit tests
Integration tests
Business logic
API compatibility
Database compatibility
Runtime configuration
```

### SRE

Owns or collaborates on:

```text
SLO
Alerting
Incident response
Reliability
Error budgets
Observability
Capacity
```

### DBA

When relevant:

```text
Schema
Migration
Query behavior
Database constraints
Database performance
```

---

# 63. Production Deployment Safety Rules

### Rule 1

Never deploy using:

```text
latest
```

Prefer:

```text
Git SHA
semantic version
image digest
```

### Rule 2

Build once.

```text
Build
 ↓
ECR
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

Do not rebuild a different image for each environment.

### Rule 3

Use GitOps.

```text
Git
 ↓
ArgoCD
 ↓
EKS
```

### Rule 4

Use canary for high-risk services.

### Rule 5

Monitor business metrics, not only Pod health.

---

# 64. Production Deployment Metrics

Monitor:

```text
Deployment
 ├── Available replicas
 ├── Unavailable replicas
 ├── Restart count
 ├── Deployment duration
 └── Rollout failures

Application
 ├── RPS
 ├── 4xx
 ├── 5xx
 ├── P50
 ├── P95
 └── P99

Dependencies
 ├── RDS connections
 ├── Redis latency
 ├── Kafka lag
 └── SQS backlog

Business
 ├── Order success rate
 ├── Payment success rate
 ├── Inventory reservation success
 └── Checkout success rate
```

---

# 65. Interview Answer

If asked:

> **"A new production deployment is causing 5xx errors. What would you do?"**

Answer:

> "First I would confirm customer impact and correlate the error increase with the deployment timeline. I would stop the rollout and identify the exact image, Git commit, GitOps revision and ReplicaSet involved. Then I would inspect Pod logs, events, Prometheus metrics, ELK logs and Jaeger traces to determine whether the problem is application code, configuration, database, Redis, messaging or infrastructure. If the previous version is known-good and the database schema is backward-compatible, I would perform a GitOps rollback rather than only using `kubectl rollout undo`, because ArgoCD could otherwise reconcile the bad version again. After rollback I would validate Pod readiness, ALB health, 5xx, latency and an actual business transaction. Finally I would perform an RCA and add preventive controls such as better integration tests, canary deployment and business-level deployment monitoring."

---

# 66. Key Production Lessons

### Lesson 1

> **A Kubernetes-healthy deployment can still be a business-broken deployment.**

### Lesson 2

> **Always investigate the most recent change during an incident.**

### Lesson 3

> **Rollback the GitOps desired state when ArgoCD owns the deployment.**

### Lesson 4

> **Application rollback and database rollback are different problems.**

### Lesson 5

> **Canary reduces the blast radius of bad releases.**

### Lesson 6

> **Security scans do not replace functional testing.**

### Lesson 7

> **Monitor business success metrics, not only CPU and Pod status.**

### Lesson 8

> **Immutable ECR images make rollback safer.**

### Lesson 9

> **Deployment health must include downstream dependencies.**

### Lesson 10

> **The safest production deployment is one where failure is detected quickly, blast radius is limited and rollback is predictable.**

---

# 67. Final Production Deployment Architecture

```text
                         Developer
                             │
                             ▼
                           Git PR
                             │
                             ▼
                         CI Pipeline
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
      Tests              SonarQube              Trivy
        │                    │                    │
        └────────────────────┼────────────────────┘
                             ▼
                       Docker Image
                             │
                             ▼
                            ECR
                             │
                             ▼
                       GitOps Repository
                             │
                             ▼
                           ArgoCD
                             │
                             ▼
                            EKS
                             │
                       Canary / Rollout
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
       Metrics             Logs              Traces
     Prometheus             ELK               Jaeger
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                    Business Validation
                             │
                  ┌──────────┴──────────┐
                  │                     │
                Healthy              Failed
                  │                     │
                  ▼                     ▼
              Continue               Rollback
                  │                     │
                  ▼                     ▼
               100%                  Git Revert
                                        │
                                        ▼
                                      ArgoCD
```

---

## Phase 18 Progress

```text
143 CrashLoopBackOff              ✅
144 OOMKilled                     ✅
145 CPU Saturation                ✅
146 Memory Saturation             ✅
147 Disk Full                     ✅
148 ImagePullBackOff              ✅
149 ALB 5xx                       ✅
150 Service Connectivity Failure  ✅
151 DNS Failure                   ✅
152 RDS Connection Exhaustion     ✅
153 Redis Failure                 ✅
154 SQS Backlog                   ✅
155 Kafka Consumer Lag            ✅
156 Node Failure                  ✅
157 AZ Failure                    ✅
158 ArgoCD Drift                  ✅
159 Bad Production Deployment     ✅
```

**Next: `160 — Security Incident Production Incident`**