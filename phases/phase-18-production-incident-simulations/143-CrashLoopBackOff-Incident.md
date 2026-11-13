# PHASE 18 — INCIDENT MANAGEMENT

# 143 — CrashLoopBackOff Production Incident

## 1. Objective

Now we move from architecture and disaster recovery into **real production incident troubleshooting**.

In this chapter, we will simulate a `CrashLoopBackOff` affecting one of our 21 Spring Boot microservices running on EKS.

We will not stop at:

```bash
kubectl get pods
kubectl logs
```

We will follow the complete production incident lifecycle:

```text
Deployment
    ↓
Pod starts
    ↓
Application crashes
    ↓
Kubernetes restarts container
    ↓
Repeated failures
    ↓
CrashLoopBackOff
    ↓
Monitoring alert
    ↓
Incident investigation
    ↓
Root cause
    ↓
Mitigation
    ↓
Fix / Rollback
    ↓
Business validation
    ↓
RCA
    ↓
Prevent recurrence
```

---

# 2. Incident Scenario

Suppose a new version of `order-service` is deployed:

```text
order-service

Old:
1.8.4

New:
1.9.0
```

GitOps flow:

```text
Developer
   ↓
Pull Request
   ↓
CI
   ↓
Build + Test + Scan
   ↓
Docker Image
   ↓
ECR
   ↓
GitOps Repo
   ↓
ArgoCD
   ↓
EKS
```

ArgoCD deploys:

```text
order-service:1.9.0
```

A few seconds later:

```text
order-service-7c8d6f8d9f-x8k2m
CrashLoopBackOff
```

---

# 3. What Does CrashLoopBackOff Actually Mean?

`CrashLoopBackOff` is not the original application error.

It describes Kubernetes behavior after a container repeatedly:

```text
Starts
  ↓
Crashes
  ↓
Restarts
  ↓
Crashes
  ↓
Restarts
```

Kubernetes begins delaying subsequent restart attempts.

Conceptually:

```text
Container starts
      ↓
Application exits
      ↓
restartPolicy applies
      ↓
Container restarted
      ↓
Application exits again
      ↓
Repeated failure
      ↓
Backoff delay increases
      ↓
CrashLoopBackOff
```

So when you see:

```text
CrashLoopBackOff
```

your first question should be:

> **Why is the container terminating?**

Not:

> "How do I fix CrashLoopBackOff?"

---

# 4. Example Production Impact

Suppose `order-service` normally has:

```text
6 replicas
```

During deployment:

```text
Old Pods:
6 healthy

New Pods:
6 attempted
```

But the new version crashes.

Depending on the Deployment strategy and readiness behavior, Kubernetes may retain some healthy old replicas while the new rollout stalls.

If configuration is poor or someone performs an unsafe deployment, customer impact could become:

```text
POST /api/v1/orders
        ↓
5xx
```

Business impact:

```text
Customers cannot place orders.
```

---

# 5. Alert

Prometheus may detect:

```text
Container restart count increasing
```

or:

```text
Available replicas below desired replicas
```

or application symptoms:

```text
HTTP 5xx increasing
```

Alertmanager sends:

```text
CRITICAL

Service:
order-service

Namespace:
ecommerce

Issue:
Container restart rate elevated

Customer impact:
Order API degraded
```

---

# 6. Incident Begins

The engineer should not immediately:

```bash
kubectl delete pod ...
```

Deleting the Pod may only produce:

```text
Broken Pod
   ↓
Delete
   ↓
New Broken Pod
   ↓
CrashLoopBackOff
```

Instead:

```text
Confirm
   ↓
Scope
   ↓
Collect evidence
   ↓
Identify cause
   ↓
Mitigate
```

---

# 7. Step 1 — Check Pods

Start with:

```bash
kubectl get pods -n ecommerce
```

Example:

```text
NAME                              READY   STATUS             RESTARTS
order-service-7c8d6f8d9f-x8k2m   0/1     CrashLoopBackOff   6
order-service-7c8d6f8d9f-b7m4p   0/1     CrashLoopBackOff   5
inventory-service-68d9f7b5c4-x2   1/1     Running            0
payment-service-77b9d4f58c-k9     1/1     Running            0
```

Now we know:

```text
order-service
```

is affected.

---

# 8. Check Deployment

```bash
kubectl get deployment order-service -n ecommerce
```

Example:

```text
NAME            READY   UP-TO-DATE   AVAILABLE
order-service   0/3     3            0
```

Now check rollout:

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

Potential result:

```text
Waiting for deployment "order-service" rollout to finish...
```

---

# 9. Step 2 — Describe the Pod

This is one of the most important commands:

```bash
kubectl describe pod \
  order-service-7c8d6f8d9f-x8k2m \
  -n ecommerce
```

Look at:

```text
State
Last State
Reason
Exit Code
Restart Count
Environment
Volumes
Mounts
Events
Probes
```

---

# 10. Example Container State

You might see:

```text
State:
  Waiting

Reason:
  CrashLoopBackOff

Last State:
  Terminated

Reason:
  Error

Exit Code:
  1
```

This tells us:

```text
Application started
      ↓
Application exited
      ↓
Exit code 1
```

But it still doesn't tell us the actual application error.

For that, we need logs.

---

# 11. Step 3 — Check Current Logs

```bash
kubectl logs \
  order-service-7c8d6f8d9f-x8k2m \
  -n ecommerce
```

Suppose:

```text
Spring Boot starting...

ERROR
Failed to configure DataSource

org.postgresql.util.PSQLException:
Connection refused
```

Now we have our first real clue.

---

# 12. Very Important — Previous Container Logs

When a container restarts, the most useful logs may belong to the **previous container instance**.

Use:

```bash
kubectl logs \
  order-service-7c8d6f8d9f-x8k2m \
  -n ecommerce \
  --previous
```

This command is extremely useful for `CrashLoopBackOff`.

Why?

Because:

```text
Container #1
   ↓
crashes
   ↓
Container #2 starts
```

Normal:

```bash
kubectl logs
```

may show Container #2.

But:

```bash
kubectl logs --previous
```

can show the logs from the previous terminated container instance retained by kubelet.

---

# 13. First Troubleshooting Pattern

Remember this sequence:

```bash
kubectl get pods -n ecommerce

kubectl describe pod <pod> -n ecommerce

kubectl logs <pod> -n ecommerce

kubectl logs <pod> -n ecommerce --previous
```

This alone resolves a large percentage of CrashLoop investigations.

But production troubleshooting should go deeper.

---

# 14. Step 4 — Check Events

```bash
kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

Possible events:

```text
BackOff
Back-off restarting failed container

Unhealthy
Startup probe failed

FailedMount
Unable to attach or mount volumes
```

Events help distinguish Kubernetes-level failures from application failures.

---

# 15. Common CrashLoopBackOff Causes

For our Spring Boot services, common causes include:

| Cause                 | Example                        |
| --------------------- | ------------------------------ |
| Application exception | Spring Boot startup failure    |
| Database              | RDS unavailable                |
| Secret                | Missing DB password            |
| Configuration         | Wrong environment variable     |
| Port                  | Application binds incorrectly  |
| Probe                 | Startup/liveness misconfigured |
| Memory                | OOMKilled                      |
| Filesystem            | Permission failure             |
| Dependency            | Redis/Kafka unavailable        |
| Migration             | Flyway failure                 |
| IAM                   | AWS access denied              |
| Certificate           | TLS/certificate problem        |
| Entrypoint            | Wrong Docker command           |

We diagnose the evidence rather than assuming one of these.

---

# 16. Scenario A — Missing Secret

Suppose logs show:

```text
Could not resolve placeholder 'DB_PASSWORD'
```

Architecture:

```text
Secrets Manager
      ↓
External Secrets Operator
      ↓
Kubernetes Secret
      ↓
order-service
```

Check:

```bash
kubectl get externalsecret -n ecommerce
```

Then:

```bash
kubectl describe externalsecret \
  order-service-secret \
  -n ecommerce
```

Check Kubernetes Secret metadata:

```bash
kubectl get secret \
  order-service-secret \
  -n ecommerce
```

Do **not** casually print production secret values.

---

# 17. Check Deployment Secret Reference

```bash
kubectl get deployment \
  order-service \
  -n ecommerce \
  -o yaml
```

You might see:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-secret
        key: db-password
```

Suppose the Secret contains:

```text
password
```

but Deployment expects:

```text
db-password
```

Then application startup fails.

Root cause:

```text
Secret key mismatch
```

---

# 18. Scenario B — RDS Connectivity

Suppose logs show:

```text
Connection to database failed
```

Do not immediately blame RDS.

Check the complete path:

```text
order-service
     ↓
DNS
     ↓
RDS endpoint
     ↓
Network
     ↓
Security Group
     ↓
PostgreSQL
     ↓
Authentication
```

---

# 19. Check DNS

From an appropriate diagnostic Pod:

```bash
nslookup <rds-endpoint>
```

If DNS fails:

```text
Application
    ↓
Cannot resolve RDS
```

investigate CoreDNS/VPC DNS/networking rather than changing database credentials.

---

# 20. Check Network Connectivity

From a controlled diagnostic environment:

```bash
nc -vz <rds-endpoint> 5432
```

Possible:

```text
Connection succeeded
```

Then networking is probably working.

If:

```text
Connection timed out
```

investigate:

```text
Security Groups
Routes
NetworkPolicy
NACLs
VPC connectivity
```

---

# 21. RDS Security Group

Expected flow:

```text
EKS application workload
        |
        | TCP 5432
        v
RDS Security Group
        |
        v
PostgreSQL
```

Do not "fix" the incident by opening:

```text
0.0.0.0/0 → 5432
```

That creates a security problem instead of solving the root cause.

---

# 22. Scenario C — Wrong Database Password

Logs:

```text
password authentication failed
```

Now network connectivity probably exists.

Investigate:

```text
Secrets Manager
      ↓
ESO
      ↓
Kubernetes Secret
      ↓
Environment variable
      ↓
Spring Boot
```

Possible root cause:

```text
Database password rotated
      ↓
Secret updated
      ↓
Pod still using old environment variable
```

Remember:

> Updating a Kubernetes Secret does not automatically change an environment variable already loaded inside a running process.

A controlled rollout/restart may be necessary after validating the new secret.

---

# 23. Scenario D — Flyway Migration Failure

Suppose:

```text
FlywayValidateException

Migration checksum mismatch
```

Then:

```text
Spring Boot
   ↓
Flyway
   ↓
Database migration validation
   ↓
Failure
   ↓
Application exits
   ↓
CrashLoopBackOff
```

Do not:

```text
delete flyway_schema_history
```

or:

```text
run flyway clean
```

in production.

Investigate which migration changed.

Recall our rule:

```text
Never modify an already-applied migration.
```

Create a forward migration when appropriate.

---

# 24. Scenario E — Liveness Probe Failure

Suppose application actually starts successfully.

But:

```text
Liveness probe failed
```

Kubernetes repeatedly kills it.

Example bad configuration:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080

  initialDelaySeconds: 5
  periodSeconds: 5
```

But Spring Boot takes:

```text
40 seconds
```

to start.

Sequence:

```text
Container starts
      ↓
Spring Boot initializing
      ↓
5 seconds
      ↓
Liveness probe fails
      ↓
Kubernetes kills container
      ↓
restart
      ↓
same thing again
```

Result:

```text
CrashLoopBackOff
```

---

# 25. Startup Probe

For slow-starting applications, a startup probe can prevent premature liveness failures.

Example:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 5
  failureThreshold: 12
```

Conceptually this allows up to:

```text
5 × 12 = 60 seconds
```

for startup before Kubernetes considers startup failed.

Exact values must be measured from real application startup behavior.

---

# 26. Three Probe Roles

Understand this clearly.

## Startup Probe

Question:

```text
Has the application finished starting?
```

---

## Readiness Probe

Question:

```text
Should this Pod receive traffic?
```

If readiness fails:

```text
Pod stays running
but
Service stops sending traffic to it
```

---

## Liveness Probe

Question:

```text
Is this process unhealthy enough that restarting it is appropriate?
```

If liveness repeatedly fails:

```text
Kubelet restarts container
```

---

# 27. Probe Mental Model

```text
                 Container
                     |
                     v
               Startup Probe
                     |
                 successful
                     |
                     v
          +----------+----------+
          |                     |
          v                     v
     Readiness               Liveness
          |                     |
          v                     v
Receive traffic?        Restart needed?
```

Do not make liveness depend blindly on every external dependency.

If RDS briefly fails and every application's liveness probe also fails:

```text
RDS incident
    ↓
21 services restart
    ↓
connection storm
    ↓
larger incident
```

---

# 28. Scenario F — OOMKilled

Check:

```bash
kubectl describe pod <pod> -n ecommerce
```

You may see:

```text
Last State:
  Terminated

Reason:
  OOMKilled

Exit Code:
  137
```

Then the real issue is memory.

```text
Memory usage
      ↓
Container limit
      ↓
Exceeded
      ↓
Killed
      ↓
Restart
      ↓
CrashLoopBackOff
```

This will be covered deeply in:

```text
144 — OOMKilled
```

---

# 29. Scenario G — Bad Docker Entrypoint

Suppose Dockerfile contains:

```dockerfile
ENTRYPOINT ["java", "-jar", "wrong.jar"]
```

Container starts:

```text
java -jar wrong.jar
```

Then:

```text
Error: Unable to access jarfile wrong.jar
```

Container exits immediately.

Kubernetes restarts it.

Result:

```text
CrashLoopBackOff
```

This is why the exact application image should be tested in CI before deployment.

---

# 30. Scenario H — File Permission

Our production image uses a non-root user:

```dockerfile
USER 10001
```

Suppose the application tries:

```text
/app/logs/application.log
```

but user `10001` cannot write there.

Logs:

```text
Permission denied
```

Then:

```text
Application exits
      ↓
CrashLoopBackOff
```

Correct solution:

```text
Fix filesystem/application configuration
```

not:

```text
Run container as root
```

unless there is a justified design reason.

---

# 31. Scenario I — Redis Dependency

Suppose application startup requires Redis.

Logs:

```text
Unable to connect to Redis
```

Check:

```text
DNS
Network
TLS
Credentials
Redis availability
```

But also ask:

> Should Redis failure prevent `order-service` from starting at all?

For a cache:

```text
Redis unavailable
      ↓
Application may degrade
      ↓
RDS fallback
```

may be preferable to:

```text
Redis unavailable
      ↓
Application crashes
```

depending on the service design.

---

# 32. Scenario J — Kafka Dependency

Suppose:

```text
Kafka unavailable
```

Application exits at startup.

Again ask:

```text
Does order-service really need Kafka available to start?
```

Maybe:

```text
Application starts
      ↓
Kafka connection retries
      ↓
Readiness reflects critical capability
```

is safer than crashing the process indefinitely.

The answer depends on business requirements.

---

# 33. Determine Whether This Started After a Deployment

One of the highest-value incident questions is:

```text
What changed?
```

Check:

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

Example:

```text
REVISION
12
13
14
```

Then inspect the deployment/image.

```bash
kubectl get deployment order-service \
  -n ecommerce \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Example:

```text
.../order-service:1.9.0
```

---

# 34. Check ArgoCD

Because our environment uses GitOps:

```text
GitOps Repo
     ↓
ArgoCD
     ↓
EKS
```

Check application state:

```bash
argocd app get order-service
```

Look for:

```text
Sync Status
Health Status
Revision
Image
Resources
```

Important:

```text
Synced
```

does not necessarily mean:

```text
Application works.
```

ArgoCD can successfully deploy a bad application configuration.

---

# 35. Check Git Change

Suppose previous version:

```text
1.8.4
```

worked.

New:

```text
1.9.0
```

fails.

Investigate:

```text
Git commit
PR
configuration changes
Dockerfile changes
Helm changes
Secret references
database migrations
```

This is where deployment metadata in Grafana is extremely useful.

---

# 36. Metrics Correlation

Grafana timeline:

```text
09:55
Healthy

10:00
ArgoCD deployment

10:01
Pod restarts increase

10:02
Available replicas decrease

10:03
5xx increases
```

That strongly suggests:

```text
Recent deployment
```

is related to the incident.

It is evidence, not automatically proof of the exact root cause.

---

# 37. Logs Correlation

Search ELK:

```text
service = order-service
environment = prod
version = 1.9.0
```

Compare with:

```text
version = 1.8.4
```

This makes version-specific failures much easier to identify.

---

# 38. Trace Impact

If some requests still reach healthy old Pods, Jaeger might show:

```text
Client
  ↓
order-service 1.8.4
  ↓
success
```

while traffic to the new version may fail before meaningful request traces are generated.

That tells us the problem may occur during:

```text
Application startup
```

rather than request processing.

---

# 39. Immediate Mitigation vs Permanent Fix

These are different.

Suppose:

```text
1.8.4 = healthy
1.9.0 = CrashLoopBackOff
```

The immediate priority may be:

```text
Restore customer service
```

rather than:

```text
Debug 1.9.0 for 45 minutes while customers cannot order
```

Therefore mitigation can be:

```text
Rollback
```

Then investigate safely.

---

# 40. Rollback in a GitOps Environment

This is important.

Because ArgoCD manages the Deployment, avoid making uncontrolled manual changes that Git immediately overwrites.

Preferred flow:

```text
Identify last known-good Git revision/image
        ↓
Revert GitOps change
        ↓
Merge/approve emergency rollback
        ↓
ArgoCD sync
        ↓
Kubernetes rollout
```

For example:

```text
order-service:1.9.0
        ↓
revert
        ↓
order-service:1.8.4
```

---

# 41. Emergency `kubectl rollout undo`

Kubernetes provides:

```bash
kubectl rollout undo deployment/order-service -n ecommerce
```

This can be useful operationally, but in our GitOps architecture there is an important issue:

```text
kubectl rollback
       ↓
Cluster = 1.8.4

Git = 1.9.0
       ↓
ArgoCD detects drift
       ↓
may restore 1.9.0
```

Therefore emergency manual rollback must be coordinated with ArgoCD/GitOps.

The desired state in Git must also be corrected.

---

# 42. Correct GitOps Incident Flow

```text
Production failure
      ↓
Incident declared
      ↓
Determine bad release
      ↓
Revert GitOps commit
      ↓
ArgoCD reconciles
      ↓
Known-good image
      ↓
Pods become Ready
      ↓
Traffic restored
```

This preserves:

```text
Git = desired state
```

and:

```text
Cluster = desired state
```

---

# 43. Watch Rollout

After rollback:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce
```

Watch:

```bash
kubectl get pods \
  -n ecommerce \
  -w
```

Expected:

```text
New healthy Pods
       ↓
Ready
       ↓
Service endpoints updated
       ↓
Traffic restored
```

---

# 44. Check Endpoints

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

This helps verify that healthy Pods are actually available behind the Service.

Remember our CoreDNS discussion:

```text
order-service
      ↓
CoreDNS
      ↓
Service ClusterIP
      ↓
Service endpoints
      ↓
Healthy Pods
```

---

# 45. Validate Service DNS

From another application Pod:

```bash
nslookup order-service
```

Expected:

```text
order-service.ecommerce.svc.cluster.local
```

resolving to the Service address for a normal ClusterIP Service.

Then test:

```bash
curl http://order-service:8080/actuator/health
```

from an appropriate diagnostic location.

---

# 46. Business Validation After Recovery

Do not stop when:

```text
kubectl get pods
```

shows:

```text
Running
```

We need business validation.

Test:

```text
Login
  ↓
Product
  ↓
Cart
  ↓
Create Order
  ↓
Inventory
  ↓
Payment
```

Create:

```text
ORD-INCIDENT-001
```

Verify:

```text
Order created
Payment correct
Inventory correct
Events correct
```

---

# 47. Verify Metrics

Grafana should show recovery:

```text
Pod restart rate
      ↓

Available replicas
      ↑

HTTP 5xx
      ↓

Successful orders
      ↑
```

---

# 48. Verify Logs

ELK:

```text
service = order-service
version = 1.8.4
```

should show normal startup.

Example:

```text
Started OrderServiceApplication
```

No continuous:

```text
ERROR
FATAL
Exception
```

startup loop.

---

# 49. Verify Traces

Jaeger:

```text
POST /api/v1/orders
```

should show:

```text
order-service
   |
   +--> customer-service
   +--> product-service
   +--> inventory-service
   +--> payment-service
```

with normal latency and successful status.

---

# 50. Incident Timeline Example

```text
10:00 — order-service 1.9.0 deployed

10:01 — new Pods begin restarting

10:02 — Prometheus restart alert fires

10:03 — order API 5xx increases

10:04 — incident declared

10:05 — kubectl describe shows exit code 1

10:06 — previous logs show DB configuration error

10:08 — Git change identified

10:10 — rollback approved

10:12 — GitOps commit reverted

10:13 — ArgoCD reconciles

10:15 — order-service 1.8.4 healthy

10:17 — order workflow validated

10:20 — incident stabilized
```

---

# 51. Example Root Cause

Suppose the actual issue was:

```text
order-service 1.9.0
```

introduced:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-secret
        key: database-password
```

but the actual Secret key was:

```text
db-password
```

Application startup:

```text
Spring Boot
    ↓
DB_PASSWORD unavailable
    ↓
DataSource initialization fails
    ↓
Application exits
    ↓
Kubernetes restarts
    ↓
CrashLoopBackOff
```

---

# 52. Root Cause vs Symptom

Do not write RCA:

```text
Root cause:
CrashLoopBackOff
```

That's incorrect.

`CrashLoopBackOff` is the symptom/state.

A proper RCA would be:

```text
Root Cause:

The order-service 1.9.0 deployment referenced an
incorrect Kubernetes Secret key.

The DB_PASSWORD environment variable could not be
populated correctly, causing Spring Boot DataSource
initialization to fail and the process to exit.

Kubernetes repeatedly restarted the failed container,
eventually reporting CrashLoopBackOff.
```

---

# 53. Why Did CI Not Catch It?

This is the next important question.

CI tested:

```text
Java
Unit tests
Docker build
Trivy
SonarQube
```

but didn't validate:

```text
Helm Secret reference
```

Therefore the incident exposes a pipeline gap.

---

# 54. Prevention

Add Helm validation:

```bash
helm lint ./helm/order-service
```

Render templates:

```bash
helm template order-service ./helm/order-service
```

Add schema/policy tests where appropriate.

But simple rendering still may not prove the referenced Secret key exists in a live environment.

Therefore we also need:

```text
Pre-production deployment
      ↓
Integration test
      ↓
Smoke test
      ↓
Promotion
```

---

# 55. Better Promotion Pipeline

```text
PR
 ↓
Build
 ↓
Unit Tests
 ↓
SonarQube
 ↓
Security Scan
 ↓
Docker Build
 ↓
ECR
 ↓
DEV
 ↓
Integration Test
 ↓
QA
 ↓
Smoke Test
 ↓
STAGING
 ↓
Production Approval
 ↓
Canary
 ↓
Production
```

This significantly reduces the chance of configuration errors reaching all production replicas.

---

# 56. Canary Would Reduce Blast Radius

Instead of immediately deploying the bad version everywhere:

```text
100% → 1.9.0
```

we could initially send limited traffic/capacity to it.

Conceptually:

```text
1.8.4
95%

1.9.0
5%
```

If:

```text
1.9.0
CrashLoopBackOff
```

then promotion stops.

The healthy version continues serving most traffic.

---

# 57. Deployment Strategy

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

This can help preserve available replicas while new ones become ready.

However, the exact settings depend on:

```text
capacity
startup time
resource availability
traffic
PDB
HPA
```

---

# 58. Readiness Protects Traffic

If a new Pod starts but isn't ready:

```text
Pod
 |
 | readiness = false
 |
 X
Service traffic
```

Only after readiness succeeds:

```text
Pod
 |
 | readiness = true
 |
 v
Service traffic
```

This prevents partially initialized Pods from receiving requests.

---

# 59. Alert Rules

We should monitor restart behavior.

Conceptually:

```promql
increase(
  kube_pod_container_status_restarts_total{
    namespace="ecommerce"
  }[10m]
) > 3
```

This is illustrative.

Production alert thresholds should be based on normal workload behavior and service criticality.

---

# 60. Additional Alert

Monitor unavailable replicas:

```text
Desired replicas
      >
Available replicas
```

Also monitor customer symptoms:

```text
HTTP 5xx
latency
order success rate
```

A restart alert alone doesn't necessarily mean customer impact.

---

# 61. CrashLoopBackOff Troubleshooting Decision Tree

```text
CrashLoopBackOff
      |
      v
kubectl describe pod
      |
      +-----------------------------+
      |                             |
      v                             v
Container terminated?        Probe killing it?
      |                             |
      v                             v
Check reason/exit code       Check startup/liveness
      |
      v
kubectl logs --previous
      |
      +---------------------------------------+
      |        |        |       |             |
      v        v        v       v             v
   Config     DB      Secret   OOM        Application
      |        |        |       |          exception
      v        v        v       v             v
Investigate root dependency / configuration
      |
      v
Recent deployment?
      |
    +---+---+
    |       |
   Yes      No
    |       |
    v       v
Compare   deeper
release   investigation
    |
    v
Mitigate
    |
    +---- rollback
    |
    +---- config fix
    |
    +---- dependency recovery
    |
    v
Validate
    |
    v
RCA
```

---

# 62. Commands to Remember

```bash
kubectl get pods -n ecommerce
```

```bash
kubectl describe pod <pod> -n ecommerce
```

```bash
kubectl logs <pod> -n ecommerce
```

```bash
kubectl logs <pod> -n ecommerce --previous
```

```bash
kubectl get events -n ecommerce --sort-by=.lastTimestamp
```

```bash
kubectl get deployment order-service -n ecommerce
```

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

These commands should become second nature when troubleshooting Kubernetes.

---

# 63. Evidence Collection

Before changing things, capture:

```text
Pod name
Pod status
Restart count
Exit code
Last termination reason
Events
Current logs
Previous logs
Image digest/version
Deployment revision
ArgoCD revision
Recent Git change
CPU/memory
Probe state
Secret/config references
Incident timestamp
```

This makes RCA much easier.

---

# 64. Incident Runbook

```text
ALERT:
CrashLoopBackOff / restart rate
        ↓
Confirm customer impact
        ↓
kubectl get pods
        ↓
kubectl describe pod
        ↓
Check termination reason
        ↓
kubectl logs --previous
        ↓
Check events
        ↓
Check recent deployment
        ↓
Check config/secrets/dependencies
        ↓
Determine root cause
        ↓
Mitigate
        ↓
Rollback/fix
        ↓
ArgoCD reconciliation
        ↓
Validate Pods
        ↓
Validate Service
        ↓
Validate business transaction
        ↓
Monitor
        ↓
RCA
```

---

# 65. RCA Template

```markdown
# Incident RCA — order-service CrashLoopBackOff

## Incident Date

## Severity

## Duration

## Affected Service

order-service

## Customer Impact

Customers experienced failures while creating orders.

## Detection

Prometheus / Alertmanager

## Timeline

## Symptoms

Pods repeatedly restarted and entered CrashLoopBackOff.

## Root Cause

Incorrect Kubernetes Secret key reference introduced
in the order-service deployment configuration.

## Contributing Factors

The configuration mismatch was not detected before
production deployment.

## Mitigation

Reverted the GitOps release to the previous known-good version.

## Permanent Fix

Corrected Secret reference and added deployment validation.

## Preventive Actions

- Helm validation
- Integration tests
- Staging smoke tests
- Canary deployment
- Restart-rate alert
- Configuration contract validation

## Owner

Platform / Application Team

## Follow-up Date
```

---

# 66. Production Lesson

The most important troubleshooting principle is:

```text
CrashLoopBackOff
        ≠
Root cause
```

Instead:

```text
CrashLoopBackOff
        ↓
Container repeatedly failing
        ↓
Find termination reason
        ↓
Find previous logs
        ↓
Check events
        ↓
Check recent changes
        ↓
Check dependencies/configuration
        ↓
Identify actual root cause
```

And in our production platform:

```text
Prometheus
     ↓
Alertmanager
     ↓
Incident
     ↓
kubectl
     ↓
Logs + Events
     ↓
ELK + Grafana + Jaeger
     ↓
Git/ArgoCD correlation
     ↓
Root Cause
     ↓
GitOps Rollback
     ↓
Business Validation
     ↓
RCA
```

---

# 67. Phase 18 Progress

```text
PHASE 18 — INCIDENT MANAGEMENT

143 — CrashLoopBackOff                 ✅
144 — OOMKilled                        NEXT
145 — CPU Saturation
146 — Memory Saturation
147 — Disk Full
148 — ImagePullBackOff
149 — ALB 5xx
150 — Service Connectivity Failure
151 — DNS Failure
152 — RDS Connection Exhaustion
153 — Redis Failure
154 — SQS Backlog
155 — Kafka Consumer Lag
156 — Node Failure
157 — AZ Failure
158 — ArgoCD Drift
159 — Bad Production Deployment
160 — Security Incident
```

## Next: **144 — OOMKilled Production Incident**

The next incident will go deeper into:

```text
Spring Boot JVM memory
      ↓
Container memory
      ↓
Kubernetes requests/limits
      ↓
Heap + non-heap + native memory
      ↓
Memory limit exceeded
      ↓
OOMKilled
      ↓
Exit code 137
      ↓
CrashLoopBackOff
```

and, importantly, how to determine whether the correct fix is **more memory, JVM tuning, fixing a memory leak, changing requests/limits, reducing workload, or scaling horizontally**.
