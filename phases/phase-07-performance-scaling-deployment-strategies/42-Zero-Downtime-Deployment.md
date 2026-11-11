# `42-Zero-Downtime-Deployment.md`

````markdown
# 42. Zero-Downtime Deployment — Production Deployment Strategy on AWS EKS

## 1. Why Zero-Downtime Deployment Matters

Our e-commerce platform is a production system with 21 microservices.

Customers may be:

- browsing products
- adding products to carts
- placing orders
- making payments
- checking shipments
- receiving notifications

We cannot normally stop the application simply because we are deploying a new version.

Bad deployment:

```text
Version 1
   |
   v
STOP APPLICATION
   |
   v
DEPLOY VERSION 2
   |
   v
START APPLICATION
````

During this period:

```text
Customer request
      |
      v
Application DOWN
      |
      v
HTTP 5xx / timeout
```

A zero-downtime deployment attempts to keep the service available while the new version is rolled out.

---

# 2. Production Deployment Principle

The basic strategy is:

```text
Old Pods
   |
   | continue serving traffic
   |
   +----------------------+
                          |
                     New Pods
                          |
                          v
                   Become Ready
                          |
                          v
                    Receive traffic
                          |
                          v
                    Old Pods removed
```

The critical rule is:

```text
DO NOT SEND PRODUCTION TRAFFIC
TO A POD BEFORE IT IS READY.
```

---

# 3. Kubernetes Rolling Deployment

Kubernetes Deployments provide rolling updates.

Example:

```text
Version 1

order-service
   |
   +-- Pod v1
   +-- Pod v1
   +-- Pod v1
```

We deploy:

```text
Version 2
```

Kubernetes gradually creates:

```text
Pod v2
```

while keeping enough:

```text
Pod v1
```

running.

Eventually:

```text
Pod v1 -> removed
Pod v2 -> running
```

---

# 4. Rolling Update Flow

Suppose:

```text
replicas = 3
```

Initial:

```text
v1
v1
v1
```

Deployment starts.

```text
v1
v1
v1
v2
```

v2 becomes Ready.

Then:

```text
v1
v1
v2
v2
```

Eventually:

```text
v1 removed
v2
v2
v2
```

The exact number of simultaneous Pods depends on:

```text
maxSurge
maxUnavailable
```

---

# 5. Deployment Strategy

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Meaning conceptually:

```text
maxUnavailable = 0
```

Try not to voluntarily reduce available replicas during the rollout.

```text
maxSurge = 1
```

Allow one additional Pod above the desired replica count during rollout.

This is only an example.

Actual values depend on:

* application startup time
* node capacity
* PodDisruptionBudget
* traffic
* resource requirements
* cost
* availability requirements

---

# 6. Why `maxUnavailable: 0` Is Useful

Suppose:

```text
3 replicas
```

With:

```yaml
maxUnavailable: 1
```

Kubernetes may temporarily have:

```text
2 available Pods
```

With:

```yaml
maxUnavailable: 0
```

we aim to maintain:

```text
3 available Pods
```

while creating replacement Pods.

However, this requires enough cluster capacity.

If no capacity exists:

```text
New Pod
   |
   v
Pending
```

The rollout can stall.

Therefore:

```text
Zero downtime
+
capacity planning
```

go together.

---

# 7. `maxSurge`

Suppose:

```text
replicas = 3
maxSurge = 1
```

During rollout Kubernetes can temporarily have:

```text
4 Pods
```

Example:

```text
v1
v1
v1
v2
```

Once v2 becomes Ready, an old Pod can be terminated.

This creates a controlled replacement process.

---

# 8. Readiness Probe

Zero downtime depends heavily on readiness.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

The application should report:

```text
READY
```

only when it can safely receive production traffic.

---

# 9. Readiness vs Liveness

These are different.

## Readiness

Answers:

```text
"Can this Pod receive traffic?"
```

## Liveness

Answers:

```text
"Is this application still alive?"
```

Example:

```text
Readiness = false
```

means:

```text
Remove Pod from service endpoints.
```

It does not necessarily mean:

```text
Restart Pod.
```

---

# 10. Startup Probe

Java applications can take significant time to start.

Example:

```text
JVM startup
   |
   v
Spring Boot initialization
   |
   v
Configuration
   |
   v
Database connection
   |
   v
Application ready
```

A startup probe can protect slow-starting applications from being killed prematurely.

Example:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 5
  failureThreshold: 30
```

This allows startup to take approximately:

```text
30 × 5 = 150 seconds
```

before the startup probe fails.

Values must be based on actual application startup behavior.

---

# 11. Graceful Shutdown

A Pod being terminated should not immediately drop active requests.

Desired flow:

```text
SIGTERM
   |
   v
Application stops accepting new work
   |
   v
Readiness becomes false
   |
   v
Traffic drains
   |
   v
Existing requests complete
   |
   v
Connections close
   |
   v
Process exits
```

Spring Boot should use graceful shutdown.

Example:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

The timeout must match actual request and dependency behavior.

---

# 12. Kubernetes Termination Grace Period

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

This gives the application time to shut down gracefully.

The value should be longer than the expected graceful shutdown duration.

Do not blindly use:

```text
terminationGracePeriodSeconds: 5
```

for applications that routinely need longer to drain.

---

# 13. Deployment Lifecycle

A simplified rollout looks like:

```text
Deployment v1
     |
     v
Create ReplicaSet v2
     |
     v
Create v2 Pod
     |
     v
Startup probe
     |
     v
Readiness probe
     |
     v
Pod Ready
     |
     v
Service sends traffic
     |
     v
Old v1 Pod terminated
     |
     v
Repeat
```

This continues until all desired replicas run v2.

---

# 14. Service and Endpoints

The Service selects Pods using labels.

Example:

```yaml
selector:
  app: order-service
```

Pods:

```text
order-service-v1
order-service-v1
order-service-v2
```

As readiness changes, Kubernetes updates the endpoints used for traffic routing.

Conceptually:

```text
Service
   |
   +-- Ready v1
   +-- Ready v1
   +-- Ready v2
```

Not-ready Pods should not receive normal Service traffic.

---

# 15. ALB and Kubernetes

Our external path is:

```text
Route 53
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
Ingress
   |
   v
Kubernetes Service
   |
   v
Ready Pods
```

During a rolling deployment:

```text
ALB
 |
 v
Service
 |
 +----> v1
 |
 +----> v2
```

Traffic gradually moves to the new version.

---

# 16. Rolling Deployment Example

Initial:

```text
order-service = 3 replicas

v1:
Pod-A
Pod-B
Pod-C
```

Deploy:

```text
order-service:2.0.0
```

Kubernetes creates:

```text
v2:
Pod-D
```

Wait for:

```text
startupProbe
readinessProbe
```

Then:

```text
v1:
Pod-A
Pod-B

v2:
Pod-D
Pod-E
```

Continue:

```text
v1:
Pod-A

v2:
Pod-D
Pod-E
Pod-F
```

Finally:

```text
v2:
Pod-D
Pod-E
Pod-F
```

Deployment complete.

---

# 17. What Happens If the New Version Is Broken?

This is one of the most important advantages of readiness and rolling deployment.

Suppose:

```text
v2
```

has a startup problem.

```text
v2 Pod
   |
   v
Readiness = false
```

Kubernetes should not send normal traffic to that Pod.

Existing v1 Pods continue serving traffic.

```text
v1
v1
v1

v2
Not Ready
```

The rollout can stop rather than immediately replacing all healthy Pods.

---

# 18. Deployment Progress Deadline

Configure a rollout deadline.

Example:

```yaml
progressDeadlineSeconds: 600
```

If the rollout doesn't make progress within the configured period, Kubernetes reports the Deployment as stalled.

Check:

```bash
kubectl -n production describe deployment order-service
```

Look for:

```text
ProgressDeadlineExceeded
```

This tells engineers the deployment did not progress normally.

---

# 19. Production Deployment Validation

Never assume:

```text
kubectl apply succeeded
```

means:

```text
application deployment succeeded
```

After deployment:

```bash
kubectl -n production rollout status deployment/order-service
```

Then:

```bash
kubectl -n production get pods -l app=order-service
```

Then inspect:

```bash
kubectl -n production describe deployment order-service
```

---

# 20. Application-Level Validation

Kubernetes health is not enough.

Check:

```text
Pod Ready
+
HTTP success
+
business functionality
```

For example:

```text
POST /api/v1/orders
```

should successfully create an order.

A Pod can be:

```text
Running
Ready
```

while the business functionality is broken.

---

# 21. Smoke Tests

After deployment:

```text
GET /health
```

Then:

```text
GET /api/v1/orders
```

Then:

```text
POST /api/v1/orders
```

depending on the safe testing strategy.

Smoke tests should verify critical paths without creating unwanted production data.

---

# 22. Automated Deployment Verification

The GitOps/CD pipeline can verify:

```text
ArgoCD sync successful
        |
        v
Deployment healthy
        |
        v
Pods Ready
        |
        v
Smoke test
        |
        v
Deployment accepted
```

If validation fails:

```text
Deployment
    |
    v
Rollback / Git revert
```

according to the organization's deployment policy.

---

# 23. GitOps and Zero Downtime

Our CD architecture is:

```text
CI
 |
 v
Build image
 |
 v
ECR
 |
 v
GitOps repository
 |
 v
ArgoCD
 |
 v
EKS
```

CI does not directly manipulate production Pods.

Instead:

```text
GitOps commit
      |
      v
ArgoCD detects change
      |
      v
Kubernetes Deployment
      |
      v
Rolling update
```

This gives us:

* auditability
* version control
* rollback
* review
* repeatability

---

# 24. Image Immutability

Suppose GitOps says:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: 2.0.0
```

Ideally that tag should refer to one immutable image.

Better still, deployment can be pinned to a digest:

```text
image@sha256:<digest>
```

This prevents ambiguity.

The production deployment should run the exact artifact that passed CI.

---

# 25. Database Changes Are Different

Application deployment may be zero downtime while database migration is not.

Example:

```text
v1 application
      |
      v
Database schema v1

Deploy v2
      |
      v
v2 expects schema v2
```

If we immediately remove a column required by v1:

```text
v1 -> broken
```

Therefore database changes must be backward compatible during rolling deployments.

---

# 26. Expand-and-Contract Pattern

Use:

```text
Expand
  |
  v
Migrate
  |
  v
Deploy
  |
  v
Contract
```

Example:

Old schema:

```text
customer_name
```

We need:

```text
first_name
last_name
```

Do not immediately remove:

```text
customer_name
```

Instead:

```text
1. Add first_name
2. Add last_name
3. Application supports both
4. Backfill data
5. Deploy new application
6. Verify
7. Stop using old field
8. Remove old field later
```

This allows old and new application versions to coexist.

---

# 27. Flyway/Liquibase

Our production RDS architecture should use controlled migrations.

Do not use:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

for production schema management.

Prefer:

```text
Flyway
```

or:

```text
Liquibase
```

with versioned migrations.

Example:

```text
db/migration/
├── V1__create_orders.sql
├── V2__add_order_status.sql
├── V3__add_payment_reference.sql
```

---

# 28. Why Database Migration Must Be Separated

Imagine 3 Pods:

```text
Pod v1
Pod v1
Pod v2
```

During a rolling deployment both versions can temporarily exist.

Therefore the database must support:

```text
v1 application
+
v2 application
```

simultaneously.

This is the key zero-downtime database principle.

---

# 29. ConfigMap Changes

Configuration changes can also cause deployment problems.

Example:

```text
v2 application
requires:

PAYMENT_TIMEOUT
```

but the ConfigMap doesn't contain it.

Result:

```text
Pod starts
   |
   v
Missing configuration
   |
   v
CrashLoopBackOff
```

Therefore configuration must be deployed before the application that depends on it.

GitOps sync ordering can help.

---

# 30. Secrets

Similarly:

```text
v2
requires:

PAYMENT_API_KEY
```

The Secret must exist before the Pod starts.

Our secret architecture is:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Application Pod
```

The application should never contain the secret in its image.

---

# 31. Rolling Update and HPA

HPA may scale the Deployment while a deployment is happening.

Example:

```text
HPA:
5 -> 10 replicas

Deployment:
v1 -> v2
```

Kubernetes must reconcile:

```text
desired replicas
+
rolling update state
```

This can temporarily increase resource requirements.

Therefore deployment capacity must account for:

```text
normal replicas
+
max surge
+
autoscaling
```

---

# 32. Example Capacity Calculation

Suppose:

```text
HPA max:
20 Pods

maxSurge:
1
```

During deployment:

```text
20 old Pods
+
1 new Pod
=
21 Pods
```

If multiple workloads deploy simultaneously, node capacity must handle the combined surge.

Example:

```text
order-service:
20 + 1

payment-service:
15 + 1

inventory-service:
20 + 1
```

Potential temporary extra Pods:

```text
3 additional Pods
```

This matters for Karpenter and cluster capacity planning.

---

# 33. PodDisruptionBudget During Deployment

A PDB primarily protects against voluntary disruptions.

It should not be treated as the mechanism that controls Deployment rolling updates.

Deployment behavior is controlled primarily by:

```text
maxUnavailable
maxSurge
```

while PDB protects availability during events such as:

```text
node drain
cluster maintenance
voluntary disruptions
```

Understand these separately.

---

# 34. Node Drain and Zero Downtime

Suppose an EKS node needs maintenance.

```text
Node-A
 |
 +-- order Pod
 +-- payment Pod
 +-- catalog Pod
```

Node drain attempts to move Pods elsewhere.

A PDB can prevent too many replicas from being voluntarily disrupted simultaneously.

Combined with:

```text
multiple replicas
+
multi-AZ placement
+
PDB
```

we improve availability.

---

# 35. Availability Across AZs

Production:

```text
AZ-A
  order-1

AZ-B
  order-2

AZ-C
  order-3
```

If:

```text
AZ-A
```

has an outage:

```text
order-1
```

is lost.

But:

```text
order-2
order-3
```

remain.

Autoscaling and rescheduling can restore capacity.

---

# 36. Topology Spread

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

This is an illustrative configuration.

The actual scheduling policy should account for:

* node availability
* AZ capacity
* workload priority
* Karpenter constraints

---

# 37. Anti-Affinity

Another option is Pod anti-affinity.

Conceptually:

```text
Don't place all replicas on the same node.
```

Example:

```text
Node-A:
order-1

Node-B:
order-2

Node-C:
order-3
```

This reduces the impact of a node failure.

Topology spread often gives more explicit control over distribution.

---

# 38. Deployment Strategies

Rolling deployment is not the only strategy.

Common approaches:

```text
Rolling
Blue-Green
Canary
Recreate
```

We will use:

```text
Rolling
```

as the standard Kubernetes deployment strategy.

Later we can implement:

```text
Blue-Green
Canary
```

for services requiring stronger release control.

---

# 39. Rolling vs Blue-Green

Rolling:

```text
v1 + v2
simultaneously
```

Gradually replace v1.

Blue-Green:

```text
Blue = v1

Green = v2
```

Both environments can exist simultaneously.

Traffic switches:

```text
Blue
 |
 v
Green
```

This provides fast rollback but can require significantly more capacity.

---

# 40. Rolling vs Canary

Canary sends a small percentage of traffic to the new version.

Example:

```text
v1:
95%

v2:
5%
```

If metrics are healthy:

```text
v1:
80%

v2:
20%
```

Then:

```text
v1:
50%

v2:
50%
```

Finally:

```text
v2:
100%
```

Canary is especially useful for high-risk releases.

---

# 41. Deployment Strategy Selection

Use Rolling for:

```text
normal application releases
```

Use Canary for:

```text
high-risk releases
major code changes
performance-sensitive changes
```

Use Blue-Green when:

```text
fast rollback
environment isolation
full-version validation
```

is more important than infrastructure cost.

---

# 42. Argo Rollouts

For advanced deployment strategies, Kubernetes-native tooling such as Argo Rollouts can be used.

Conceptually:

```text
GitOps
   |
   v
ArgoCD
   |
   v
Argo Rollouts
   |
   +----> Canary
   |
   +----> Blue-Green
```

This is different from standard Kubernetes Deployment rolling updates.

We should introduce it when the platform requires progressive delivery.

---

# 43. Automated Canary Analysis

A mature platform can evaluate:

```text
error rate
latency
HTTP 5xx
business metrics
CPU
memory
```

Example:

```text
Canary = 5%

Metrics healthy?
       |
      Yes
       |
       v
Canary = 25%
```

If:

```text
error rate increases
```

then:

```text
Abort rollout
```

This creates progressive delivery.

---

# 44. Zero Downtime Does Not Mean Zero Risk

Important distinction:

```text
Zero downtime
!=
Zero deployment risk
```

A new application can be:

```text
Running
Ready
```

and still contain a business bug.

For example:

```text
Payment amount calculated incorrectly.
```

Kubernetes cannot detect this from a readiness probe.

Therefore we need:

```text
CI testing
+
SonarQube
+
security scanning
+
automated tests
+
smoke tests
+
metrics
+
logs
+
traces
+
progressive delivery
```

---

# 45. Production Deployment Gates

A production deployment can use:

```text
PR
 |
 v
CI
 |
 +-- Unit tests
 +-- Integration tests
 +-- SonarQube
 +-- Security scans
 +-- Image scan
 |
 v
ECR
 |
 v
GitOps PR
 |
 v
Approval
 |
 v
ArgoCD
 |
 v
Rolling deployment
 |
 v
Health validation
 |
 v
Business smoke test
 |
 v
Monitor
```

---

# 46. Monitoring During Deployment

During a rollout, watch:

```text
HTTP request rate
HTTP 4xx
HTTP 5xx
p50 latency
p95 latency
p99 latency
Pod restarts
CPU
memory
HPA
deployment status
database connections
Redis
SQS backlog
```

For traces:

```text
Jaeger
```

can show whether requests involving the new version are slower.

For logs:

```text
Kibana
```

can show application errors.

---

# 47. Grafana During Deployment

Example:

```text
Before deployment:

p95 = 200ms
5xx = 0.1%
```

After v2 begins:

```text
p95 = 800ms
5xx = 3%
```

Stop the rollout.

The deployment should not continue simply because Pods are technically Ready.

---

# 48. Jaeger During Deployment

Search traces by:

```text
service.name
service.version
```

Example:

```text
order-service
version=2.0.0
```

Compare:

```text
v1:
250ms

v2:
1.5s
```

This can identify performance regressions that CPU metrics alone may not reveal.

---

# 49. Kibana During Deployment

Search:

```text
service=order-service
version=2.0.0
level=ERROR
```

Look for:

```text
database timeout
NullPointerException
connection pool exhaustion
external API failure
configuration errors
```

This provides fast deployment validation.

---

# 50. Deployment Rollback

If the rollout is unhealthy:

```bash
kubectl -n production rollout undo deployment/order-service
```

But in GitOps production, the preferred rollback mechanism should generally be:

```text
Revert GitOps commit
       |
       v
ArgoCD
       |
       v
Previous desired state
```

Why?

Because manually changing Kubernetes state creates drift.

---

# 51. GitOps Rollback

Suppose GitOps changed:

```text
order-service:
1.5.0 -> 2.0.0
```

The repository history shows:

```text
commit A:
1.5.0

commit B:
2.0.0
```

Rollback:

```text
revert B
```

Now:

```text
Git:
1.5.0
```

ArgoCD reconciles:

```text
EKS:
2.0.0 -> 1.5.0
```

This preserves the desired-state model.

---

# 52. Rollback Is Not Always Enough

Suppose v2 executed:

```text
database migration
```

and then we rollback application code.

The database may still be:

```text
schema v2
```

while application is:

```text
v1
```

Therefore:

```text
application rollback
```

and:

```text
database rollback
```

are separate problems.

This is why backward-compatible migrations are so important.

---

# 53. Immutable Infrastructure Principle

Avoid changing running Pods manually.

Bad:

```text
kubectl exec
edit application files
restart container
```

Good:

```text
Source code
   |
   v
CI
   |
   v
Immutable image
   |
   v
ECR
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
EKS
```

The deployment should be reproducible.

---

# 54. What If the Node Has No Capacity?

Rolling deployment requires surge capacity.

Suppose:

```text
3 replicas
maxSurge = 1
```

New Pod:

```text
Pending
```

because the nodes are full.

Karpenter should be able to provision capacity if configured appropriately.

```text
Deployment
   |
   v
New Pod
   |
   v
Pending
   |
   v
Karpenter
   |
   v
New EC2
   |
   v
Pod scheduled
```

Therefore:

```text
Zero-downtime deployment
```

depends partly on:

```text
autoscaling
```

---

# 55. Deployment and Karpenter Interaction

During deployment:

```text
old Pods
+
new Pods
```

temporarily consume more resources.

Karpenter sees:

```text
unschedulable new Pod
```

and provisions capacity.

After deployment:

```text
old Pods removed
```

The new node may become underutilized.

Karpenter consolidation can eventually reduce unnecessary capacity according to its policy.

This is an important cost implication of rolling deployments.

---

# 56. Deployment and Pod Startup

Suppose image size is:

```text
1.5 GB
```

and application startup is:

```text
60 seconds
```

A rollout can take longer.

Therefore optimize:

```text
image size
startup time
dependency initialization
readiness checks
```

A multi-stage Docker build helps reduce runtime image size.

---

# 57. Java Application Startup

For our Spring Boot service:

```text
Container starts
   |
   v
JVM starts
   |
   v
Spring context
   |
   v
Beans initialized
   |
   v
DB connection pool
   |
   v
OTel instrumentation
   |
   v
Actuator
   |
   v
Readiness
```

Only after readiness should production traffic reach the Pod.

---

# 58. Avoid Heavy Readiness Checks

Bad readiness endpoint:

```text
/ready
```

that performs:

```text
database query
Redis query
external payment API
SQS call
```

on every readiness check.

This can create additional load.

Readiness should verify that the application is capable of serving traffic without unnecessarily hammering dependencies.

Dependency health checks should be designed carefully.

---

# 59. Startup vs Readiness vs Liveness

A useful mental model:

```text
Startup:
"Have I finished starting?"

Readiness:
"Can I receive traffic?"

Liveness:
"Am I stuck/broken enough to require restart?"
```

Use each probe for its actual purpose.

---

# 60. Common Zero-Downtime Failures

## Failure 1 — No readiness probe

New Pod receives traffic too early.

Result:

```text
5xx
```

---

## Failure 2 — Bad readiness endpoint

Pod never becomes Ready.

Result:

```text
rollout stuck
```

---

## Failure 3 — Insufficient capacity

New Pods remain Pending.

Result:

```text
rollout stuck
```

---

## Failure 4 — Aggressive termination

Existing requests are interrupted.

Result:

```text
connection reset
5xx
```

---

## Failure 5 — Incompatible DB migration

Old Pods fail against new schema.

Result:

```text
application errors
```

---

## Failure 6 — New version logically broken

Pods are Ready but business functionality fails.

Result:

```text
successful Kubernetes rollout
+
failed application
```

---

# 61. Troubleshooting a Stuck Rollout

Check:

```bash
kubectl -n production rollout status deployment/order-service
```

Then:

```bash
kubectl -n production get pods
```

Then:

```bash
kubectl -n production describe deployment order-service
```

Then inspect ReplicaSets:

```bash
kubectl -n production get rs
```

Then:

```bash
kubectl -n production describe pod <new-pod>
```

Look for:

```text
ImagePullBackOff
CrashLoopBackOff
Pending
Readiness probe failed
Insufficient CPU
Insufficient memory
```

---

# 62. If Pods Are Pending

Check:

```bash
kubectl -n production describe pod <pod>
```

Potential output:

```text
0/10 nodes are available:
Insufficient cpu
```

Then investigate:

```text
node capacity
requests
Karpenter
NodePool
taints
affinity
topology constraints
```

---

# 63. If Pods Crash

Check:

```bash
kubectl -n production logs <pod>
```

Previous container:

```bash
kubectl -n production logs <pod> --previous
```

Then inspect:

```bash
kubectl -n production describe pod <pod>
```

Typical causes:

```text
missing secret
bad environment variable
database connection failure
incorrect configuration
application bug
OOMKilled
```

---

# 64. If Readiness Fails

Check:

```bash
kubectl -n production describe pod <pod>
```

Look for:

```text
Readiness probe failed
```

Then test inside the Pod if appropriate:

```bash
kubectl -n production exec <pod> -- \
  curl -f http://localhost:8080/actuator/health/readiness
```

Then determine whether the problem is:

```text
application
configuration
port
probe path
startup timing
dependency
```

---

# 65. If Deployment Succeeds but Users See Errors

Do not stop at Kubernetes.

Check:

```text
ALB
Ingress
Service
Endpoints
Pods
Application logs
Prometheus
Grafana
Jaeger
RDS
Redis
Messaging
```

A deployment can be Kubernetes-healthy but application-unhealthy.

---

# 66. Production Deployment Checklist

## Before Deployment

* [ ] CI passed
* [ ] Tests passed
* [ ] SonarQube quality gate passed
* [ ] Security scans passed
* [ ] Image exists in ECR
* [ ] Image digest verified
* [ ] GitOps PR reviewed
* [ ] Database migration reviewed
* [ ] Config changes reviewed
* [ ] Secrets available
* [ ] Capacity available
* [ ] Rollback plan defined

## During Deployment

* [ ] ArgoCD status healthy
* [ ] Deployment progressing
* [ ] New Pods starting
* [ ] Readiness passing
* [ ] No unexpected restarts
* [ ] Error rate normal
* [ ] Latency normal
* [ ] Database connections normal
* [ ] Redis healthy
* [ ] Queue backlog normal

## After Deployment

* [ ] All replicas Ready
* [ ] Smoke tests passed
* [ ] Error rate normal
* [ ] Latency normal
* [ ] Traces normal
* [ ] Logs normal
* [ ] HPA stable
* [ ] No unexpected node scaling
* [ ] Old ReplicaSet scaled down
* [ ] ArgoCD Synced/Healthy

---

# 67. Complete Zero-Downtime Architecture

```text
                        Developer
                            |
                            v
                          GitHub
                            |
                            v
                            CI
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
       Tests            SonarQube          Security
          |                 |                  |
          +-----------------+------------------+
                            |
                            v
                           ECR
                            |
                            v
                       GitOps Repo
                            |
                            v
                          ArgoCD
                            |
                            v
                         EKS
                            |
                 Rolling Deployment
                            |
          +-----------------+------------------+
          |                                    |
          v                                    v
      Existing Pods                       New Pods
      continue traffic                    start
                                               |
                                               v
                                         Startup Probe
                                               |
                                               v
                                        Readiness Probe
                                               |
                                               v
                                            READY
                                               |
                                               v
                                        Receive traffic
                                               |
                                               v
                                      Old Pods terminate
                                               |
                                               v
                                        Rollout complete
```

Observability during the rollout:

```text
                    Deployment
                        |
       +----------------+----------------+
       |                |                |
       v                v                v
   Prometheus        Jaeger           Kibana
       |                |                |
       v                v                v
    Metrics           Traces           Logs
       |                |                |
       +----------------+----------------+
                        |
                        v
                 Deployment decision
```

---

# 68. Final Mental Model

Zero-downtime deployment is not one Kubernetes setting.

It is a complete system:

```text
Immutable Image
      |
      v
Rolling Update
      |
      v
Readiness
      |
      v
Graceful Shutdown
      |
      v
Enough Capacity
      |
      v
Multi-AZ Scheduling
      |
      v
Backward-Compatible Database
      |
      v
Observability
      |
      v
Automated Validation
      |
      v
Rollback
```

For our EKS platform:

```text
GitHub
   |
   v
CI
   |
   v
ECR
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
EKS
   |
   +---- HPA
   |
   +---- Karpenter
   |
   +---- Rolling Deployment
   |
   +---- Readiness/Liveness/Startup
   |
   +---- PDB
   |
   +---- Multi-AZ
   |
   +---- Prometheus/Grafana
   |
   +---- OpenTelemetry/Jaeger
   |
   +---- Fluent Bit/Logstash/Elasticsearch/Kibana
   |
   v
Production
```

The most important production lesson is:

> **Zero downtime is achieved by keeping enough healthy capacity available while gradually introducing the new version, validating it before sending traffic, draining the old version gracefully, and maintaining backward compatibility with dependencies such as databases.**

The next level after this is **progressive delivery** — Blue-Green and Canary deployments — where we don't send 100% of production traffic to a new version immediately.

```
```
