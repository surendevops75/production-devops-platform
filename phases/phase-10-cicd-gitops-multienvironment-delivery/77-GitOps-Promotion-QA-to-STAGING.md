# 77 — GitOps Promotion: QA → STAGING

> **Goal:** Promote the exact image that successfully passed QA into STAGING, while introducing stronger validation and a controlled approval step.
>
> **Important:** We still do **not rebuild the image**.

---

## 1. Where We Are

Our release currently looks like:

```text
Developer
    ↓
GitHub
    ↓
CI
    ↓
ECR
    ↓
Immutable Image
    ↓
DEV
    ↓
DEV Validation
    ↓
QA
    ↓
QA Validation
```

Now we add:

```text
QA
 ↓
QA Validation
 ↓
STAGING Promotion PR
 ↓
Approval
 ↓
STAGING
```

---

# 2. The Artifact Must Stay Identical

Suppose ECR contains:

```text
order-service:8f31a2c
```

with:

```text
sha256:9a7b123...
```

DEV:

```text
sha256:9a7b123...
```

QA:

```text
sha256:9a7b123...
```

STAGING must also run:

```text
sha256:9a7b123...
```

Therefore:

```text
DEV     = sha256:9a7b123...
QA      = sha256:9a7b123...
STAGING = sha256:9a7b123...
```

The image doesn't change.

---

# 3. Why Have STAGING?

QA and STAGING have different purposes.

A simplified model:

| Environment | Primary purpose                |
| ----------- | ------------------------------ |
| DEV         | Developer integration          |
| QA          | Functional/integration testing |
| STAGING     | Production-like validation     |
| PROD        | Real customer traffic          |

STAGING should be as close to production as reasonably practical.

For example:

```text
QA
├── smaller infrastructure
├── test integrations
└── functional testing

STAGING
├── production-like topology
├── production-like configuration
├── production-like dependencies
├── performance validation
└── release candidate verification
```

---

# 4. What Stays the Same?

These remain identical:

```text
Application source
Docker image
Image digest
Container filesystem
Helm chart version
```

For example:

```text
order-service
    │
    └── sha256:9a7b123...
```

---

# 5. What Can Change?

STAGING can have different:

```text
replica count
CPU/memory
database endpoint
Redis endpoint
queue names
external service endpoints
secrets
autoscaling configuration
Ingress hostname
resource limits
```

For example:

### QA

```yaml
replicaCount: 2
```

### STAGING

```yaml
replicaCount: 3
```

Same image.

---

# 6. GitOps Structure

We already have:

```text
gitops/
└── environments/
    ├── dev/
    │   └── order-service/
    │
    ├── qa/
    │   └── order-service/
    │
    ├── staging/
    │   └── order-service/
    │
    └── prod/
        └── order-service/
```

The STAGING values file:

```text
environments/staging/order-service/values.yaml
```

might contain:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "previous"
  digest: "sha256:previous..."

replicaCount: 3

resources:
  requests:
    cpu: 500m
    memory: 1Gi

  limits:
    cpu: "1"
    memory: 2Gi
```

---

# 7. Promotion Should Start From QA

We should not manually enter:

```text
tag = ?
digest = ?
```

Instead:

```text
QA GitOps
    │
    ├── tag
    └── digest
         │
         ▼
STAGING GitOps
```

This prevents accidentally selecting an unrelated image.

---

# 8. Read the QA Artifact

Run:

```bash
yq '.image' \
  environments/qa/order-service/values.yaml
```

Example:

```yaml
repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
tag: 8f31a2c
digest: sha256:9a7b123...
```

That is our promotion candidate.

---

# 9. Verify QA Runtime

First:

```bash
kubectl get pods \
  -n ecommerce-qa
```

Find the order-service Pod:

```bash
kubectl get pods \
  -n ecommerce-qa \
  -l app.kubernetes.io/name=order-service
```

Then:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-qa \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
.../order-service@sha256:9a7b123...
```

Now we know:

```text
QA GitOps
    ↓
sha256:9a7b123...

QA Runtime
    ↓
sha256:9a7b123...
```

---

# 10. Verify QA Health

Check:

```bash
kubectl get deployment \
  order-service \
  -n ecommerce-qa
```

Expected:

```text
READY
2/2
```

or whatever replica count is configured.

Check:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-qa
```

Expected:

```text
deployment "order-service" successfully rolled out
```

---

# 11. Run QA Smoke Tests

For example:

```bash
kubectl port-forward \
  svc/order-service \
  8080:8080 \
  -n ecommerce-qa
```

Then:

```bash
curl http://localhost:8080/actuator/health
```

And:

```bash
curl http://localhost:8080/api/v1/orders
```

You can also run automated API tests.

The important point is:

```text
Pod Running
```

is not enough.

We want:

```text
Application Healthy
```

---

# 12. Promotion Script

We already created:

```text
scripts/promote-image.sh
```

Use:

```bash
./scripts/promote-image.sh \
  order-service \
  qa \
  staging
```

The script reads:

```text
QA values.yaml
```

and updates:

```text
STAGING values.yaml
```

with the exact same:

```text
tag
digest
```

---

# 13. Example

Before:

### QA

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

### STAGING

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "7abc123"
  digest: "sha256:old..."
```

After promotion:

### QA

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

### STAGING

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

---

# 14. Create Promotion Branch

```bash
git checkout -b \
  promotion/order-service-qa-to-staging
```

Check:

```bash
git status
```

Then:

```bash
git diff \
  environments/staging/order-service/values.yaml
```

You should see only the intended image change.

---

# 15. Commit

```bash
git add \
  environments/staging/order-service/values.yaml
```

Then:

```bash
git commit \
  -m "promote: order-service from qa to staging"
```

Push:

```bash
git push \
  origin \
  promotion/order-service-qa-to-staging
```

---

# 16. Create the STAGING PR

Example:

```text
Title:

promote: order-service 8f31a2c from qa to staging
```

PR body:

```text
## Order Service Promotion

Source:
QA

Target:
STAGING

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Tag:
8f31a2c

Digest:
sha256:9a7b123...

QA Status:
PASS

QA Runtime Digest:
sha256:9a7b123...

Requested:
Promote the exact QA artifact to STAGING.
```

---

# 17. Why Include the QA Runtime Digest?

This is an important improvement.

Don't simply say:

```text
QA passed.
```

Record:

```text
QA GitOps digest:
sha256:9a7b123...

QA runtime digest:
sha256:9a7b123...
```

Then the reviewer can establish:

```text
ECR
 ↓
QA GitOps
 ↓
QA Runtime
```

all point to the same artifact.

---

# 18. GitOps CI Validates the PR

The GitOps repository should run validation.

For example:

```text
GitOps PR
    │
    ├── YAML validation
    │
    ├── Helm lint
    │
    ├── Helm template
    │
    ├── Kubernetes schema validation
    │
    └── Policy checks
```

If validation fails:

```text
PR
 ↓
CI FAILED
 ↓
Do not merge
```

---

# 19. STAGING Approval

Unlike an automatic DEV deployment, STAGING can have a stronger review gate.

For example:

```text
QA validation
     ↓
STAGING PR
     ↓
Platform approval
     ↓
Application approval
     ↓
Merge
```

The exact approval requirements depend on your organization's release policy.

---

# 20. Merge

After approval:

```text
STAGING PR
     ↓
Merge
```

GitOps now contains:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

ArgoCD detects the Git change.

---

# 21. ArgoCD STAGING Application

Our ApplicationSet generates:

```text
order-service-staging
```

pointing to:

```text
environments/staging/order-service/values.yaml
```

ArgoCD sees:

```text
GitOps changed
      ↓
Application OutOfSync
      ↓
Sync
      ↓
Deployment
```

---

# 22. STAGING Namespace

Example:

```bash
kubectl get pods \
  -n ecommerce-staging
```

Expected:

```text
NAME                              READY   STATUS
order-service-xxx                 1/1     Running
order-service-yyy                 1/1     Running
order-service-zzz                 1/1     Running
```

---

# 23. Verify STAGING Digest

Run:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-staging \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
.../order-service@sha256:9a7b123...
```

Now:

```text
ECR:
sha256:9a7b123...

DEV:
sha256:9a7b123...

QA:
sha256:9a7b123...

STAGING:
sha256:9a7b123...
```

That's the artifact-promotion model we want.

---

# 24. Why STAGING Should Be Production-Like

STAGING should approximate important production characteristics.

For example:

```text
Production:
3 AZ
multiple replicas
ALB
RDS
Redis
SQS
autoscaling
monitoring
logging
secrets
```

STAGING might use:

```text
3 AZ
multiple replicas
ALB
staging RDS
staging Redis
staging SQS
autoscaling
monitoring
logging
staging secrets
```

The goal is not necessarily to duplicate production size exactly.

The goal is to reproduce **important production behaviors**.

---

# 25. STAGING Database

STAGING should not normally use the production database.

Instead:

```text
DEV
 ↓
dev database

QA
 ↓
qa database

STAGING
 ↓
staging database

PROD
 ↓
production database
```

This protects production data and allows testing without accidentally modifying customer data.

---

# 26. STAGING Data

Be careful with production data.

Avoid casually copying production customer information into lower environments.

If production-like data is required, organizations commonly use:

```text
synthetic data
masked data
anonymized data
sanitized datasets
```

The exact approach depends on data classification and compliance requirements.

---

# 27. STAGING Tests

This is where we can perform broader validation.

For `order-service`:

```text
API tests
Integration tests
Database tests
Redis tests
SQS tests
Authentication
Authorization
Error handling
Timeout handling
Retry behavior
```

And platform-level tests:

```text
Deployment
Scaling
Pod restart
Node failure
ALB
DNS
Secrets
Observability
Alerts
```

---

# 28. Performance Validation

STAGING is also a useful place for controlled performance testing.

For example:

```text
100 requests/sec
        ↓
order-service
        ↓
HPA
        ↓
additional Pods
```

Observe:

```text
CPU
Memory
Latency
Error rate
Pod count
RDS connections
Redis latency
SQS backlog
```

This connects directly with our previous autoscaling and observability chapters.

---

# 29. Deployment Test

We should verify that deployment itself behaves correctly.

For example:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-staging
```

Then:

```bash
kubectl get pods \
  -n ecommerce-staging \
  -o wide
```

We want to see the rolling update complete without unnecessary downtime.

---

# 30. Failure Test

A useful STAGING test is to deliberately delete a Pod:

```bash
kubectl delete pod <POD_NAME> \
  -n ecommerce-staging
```

Then:

```bash
kubectl get pods \
  -n ecommerce-staging \
  -w
```

Kubernetes should recreate the Pod.

This verifies:

```text
Deployment
 ↓
ReplicaSet
 ↓
Pod self-healing
```

---

# 31. Verify Monitoring

After deploying to STAGING, verify that metrics appear.

For example:

```text
order-service
   ↓
/actuator/prometheus
   ↓
Prometheus
   ↓
Grafana
```

Check:

```text
request rate
error rate
latency
CPU
memory
pod restarts
```

This is important because a release is not truly validated if the application becomes invisible to your monitoring system.

---

# 32. Verify Logs

Application:

```text
stdout/stderr
    ↓
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

Search for:

```text
service.name = order-service
environment = staging
```

Confirm the new version is generating logs correctly.

---

# 33. Verify Traces

If OpenTelemetry is enabled:

```text
order-service
     ↓
OTel
     ↓
Collector
     ↓
Jaeger
```

Generate a test request:

```bash
curl ...
```

Then verify the trace.

You should be able to correlate:

```text
Request
 ↓
Trace
 ↓
Logs
 ↓
Metrics
```

---

# 34. What If STAGING Fails?

Suppose:

```text
sha256:9a7b123...
```

fails performance testing.

Do **not** promote it to production.

Flow:

```text
STAGING FAILED
      ↓
Stop release
      ↓
Investigate
      ↓
Fix code
      ↓
New commit
      ↓
New image
      ↓
New digest
      ↓
DEV
      ↓
QA
      ↓
STAGING
```

Notice:

> We don't modify the existing image.

We create a new artifact.

---

# 35. Example

Broken version:

```text
8f31a2c
sha256:AAA
```

Fix:

```text
91b7de4
sha256:BBB
```

The new flow is:

```text
DEV
 ↓
91b7de4
 ↓
QA
 ↓
91b7de4
 ↓
STAGING
 ↓
91b7de4
```

The old artifact:

```text
sha256:AAA
```

remains available for investigation/rollback according to ECR retention policy.

---

# 36. Never "Fix" an Image in Place

Don't do:

```text
order-service:8f31a2c
```

and later replace its contents.

Use immutable tagging:

```text
order-service:8f31a2c
```

and digest:

```text
sha256:...
```

Each build gets a unique identity.

---

# 37. QA → STAGING Automation

Later we can automate the PR creation.

The architecture becomes:

```text
QA validation
      │
      ▼
Promotion Controller/Workflow
      │
      ├── Read QA digest
      ├── Verify QA runtime
      ├── Update STAGING values
      ├── Create branch
      └── Create PR
               │
               ▼
          Human approval
               │
               ▼
             Merge
               │
               ▼
             ArgoCD
```

Notice:

> Automation creates the change; Git review controls the change.

---

# 38. Why Not Automatically Merge STAGING?

We could technically automate:

```text
QA passed
 ↓
STAGING PR
 ↓
automatic merge
```

But that removes a useful control point.

For our learning project, we'll keep:

```text
QA
 ↓
PR
 ↓
Review
 ↓
STAGING
```

This makes the release flow easier to understand and provides a clear audit point.

---

# 39. Environment Promotion Matrix

Our project now looks like this:

| Stage   | Image       | Deployment                   |
| ------- | ----------- | ---------------------------- |
| DEV     | New digest  | Automated                    |
| QA      | Same digest | Controlled promotion         |
| STAGING | Same digest | Controlled promotion         |
| PROD    | Same digest | Explicit production approval |

The important thing is not the exact automation level of each environment; it's that **promotion does not rebuild the artifact**.

---

# 40. Complete Promotion Chain

```text
                     ┌──────────────┐
                     │ GitHub CI    │
                     └──────┬───────┘
                            │
                            ▼
                           ECR
                            │
                       sha256:ABC
                            │
                            ▼
                           DEV
                            │
                       DEV tests
                            │
                            ▼
                            QA
                            │
                       QA tests
                            │
                            ▼
                         STAGING
                            │
               ┌────────────┼────────────┐
               │            │            │
             API         Perf        Resilience
             Tests        Tests         Tests
               │            │            │
               └────────────┼────────────┘
                            │
                            ▼
                     Production Gate
```

---

# 41. One Critical Rule for the Entire Project

From now onward, whenever we say:

> **Promote `order-service`**

we mean:

```text
Take the already-built immutable image
and change which environment references it.
```

We do **not** mean:

```text
Build another Docker image.
```

---

# 42. Practical Checklist

Before merging QA → STAGING:

```text
[ ] QA GitOps contains candidate digest
[ ] QA runtime uses candidate digest
[ ] ECR contains candidate digest
[ ] QA health checks passed
[ ] QA functional tests passed
[ ] QA integration tests passed
[ ] GitOps values updated
[ ] Helm validation passed
[ ] Kubernetes schema validation passed
[ ] STAGING PR reviewed
[ ] No secret values introduced
[ ] Same image digest preserved
```

After merge:

```text
[ ] ArgoCD detects change
[ ] ArgoCD sync succeeds
[ ] Deployment rolls out
[ ] Pods become Ready
[ ] Runtime digest verified
[ ] Metrics visible
[ ] Logs visible
[ ] Traces visible
[ ] Smoke tests pass
[ ] Performance/resilience checks pass where applicable
```

---

# 43. Final Architecture

We now have:

```text
                   APPLICATION REPOSITORY
                            │
                            ▼
                     GitHub Actions
                            │
                    Build / Test / Scan
                            │
                            ▼
                           ECR
                            │
                     Immutable Digest
                            │
                            ▼
                           DEV
                            │
                    Automated validation
                            │
                            ▼
                           QA
                            │
                    QA validation
                            │
                            ▼
                  QA → STAGING PR
                            │
                         Approval
                            │
                            ▼
                         STAGING
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             API        Performance   Resilience
            Tests          Tests         Tests
              │             │             │
              └─────────────┼─────────────┘
                            │
                            ▼
                       PROD GATE
```

And the artifact remains:

```text
DEV
  │
  └── sha256:9a7b123...

QA
  │
  └── sha256:9a7b123...

STAGING
  │
  └── sha256:9a7b123...
```

**Next: `78-Production-Release-Gate-and-Promotion.md`** — we will build the most important promotion stage: **STAGING → PROD**, including production approval, pre-deployment checks, change control, ArgoCD production sync, zero-downtime rollout, rollback, and how to prove that PROD received the exact artifact validated in STAGING.
