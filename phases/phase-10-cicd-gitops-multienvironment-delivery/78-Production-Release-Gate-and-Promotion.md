# 78 — Production Release Gate and Promotion

> **Goal:** Promote the **exact image validated in STAGING** into PROD using a controlled production approval process.
>
> This chapter connects everything we have built so far:
>
> **CI → ECR → DEV → QA → STAGING → PROD**
>
> The most important rule remains:
>
> **PROD must run the exact immutable artifact that passed STAGING.**

---

# 1. Where We Are

Our release currently looks like:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Build
    ├── Test
    ├── Security Scan
    ├── Docker Build
    └── Trivy
    │
    ▼
ECR
    │
    ▼
Immutable Image
    │
    ▼
DEV
    │
    ▼
QA
    │
    ▼
STAGING
```

Now we need:

```text
STAGING
   │
   ▼
Production Release Gate
   │
   ▼
PROD GitOps PR
   │
   ▼
Approval
   │
   ▼
ArgoCD
   │
   ▼
PROD EKS
```

---

# 2. The Artifact We Are Promoting

Suppose our release candidate is:

```text
Image:
order-service:8f31a2c
```

Digest:

```text
sha256:9a7b123456789...
```

STAGING is running:

```text
sha256:9a7b123456789...
```

Therefore PROD should run:

```text
sha256:9a7b123456789...
```

Not:

```text
latest
```

Not:

```text
8f31a2c rebuilt again
```

Not:

```text
some-new-build
```

---

# 3. Why Production Needs an Additional Gate

DEV and QA are primarily testing environments.

STAGING is where we perform production-like validation.

Production is different because a deployment can affect:

```text
Customers
Orders
Payments
Inventory
Revenue
External integrations
Business operations
```

Therefore we want a deliberate release gate.

Conceptually:

```text
STAGING
   │
   ├── Functional validation
   ├── Performance validation
   ├── Security validation
   ├── Observability validation
   ├── Rollback validation
   │
   ▼
Production Release Gate
   │
   ├── Artifact verification
   ├── Change review
   ├── Deployment window
   ├── Database compatibility
   ├── Monitoring readiness
   │
   ▼
Production
```

---

# 4. Production GitOps File

Our production file is:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            ├── app.yaml
            └── values.yaml
```

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "previous-version"
  digest: "sha256:previous..."
```

We want:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

---

# 5. First Rule: Read From STAGING

Don't manually type:

```text
8f31a2c
```

Instead:

```text
STAGING GitOps
       │
       ├── image.tag
       └── image.digest
              │
              ▼
          PROD GitOps
```

This creates a direct promotion relationship.

---

# 6. Verify STAGING GitOps

Run:

```bash
yq '.image' \
  environments/staging/order-service/values.yaml
```

Example:

```yaml
repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
tag: 8f31a2c
digest: sha256:9a7b123...
```

Record:

```text
STAGING TAG:
8f31a2c

STAGING DIGEST:
sha256:9a7b123...
```

---

# 7. Verify STAGING Runtime

Now check the actual Kubernetes workload.

```bash
kubectl get deployment \
  order-service \
  -n ecommerce-staging
```

Check rollout:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-staging
```

Expected:

```text
deployment "order-service" successfully rolled out
```

Now get a Pod:

```bash
kubectl get pods \
  -n ecommerce-staging \
  -l app.kubernetes.io/name=order-service
```

Then:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-staging \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
.../order-service@sha256:9a7b123...
```

---

# 8. Three-Way Artifact Verification

Before production promotion, verify three things.

### 1. GitOps

```text
STAGING GitOps
sha256:9a7b123...
```

### 2. Kubernetes

```text
STAGING Pod
sha256:9a7b123...
```

### 3. ECR

```text
ECR
sha256:9a7b123...
```

All three must match.

```text
              sha256:9a7b123...
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        ECR       STAGING      STAGING
                  GitOps       Pod
```

---

# 9. Production Promotion Script

We can reuse our previous script:

```bash
./scripts/promote-image.sh \
  order-service \
  staging \
  prod
```

This reads:

```text
environments/staging/order-service/values.yaml
```

and updates:

```text
environments/prod/order-service/values.yaml
```

---

# 10. But Production Needs an Extra Safety Check

Before changing PROD, verify that the source environment is actually STAGING.

Our script should reject:

```text
QA → PROD
```

if our production policy requires:

```text
QA → STAGING → PROD
```

We can create a simple promotion rule.

For example:

```bash
ALLOWED_SOURCE="staging"

if [[ "$SOURCE_ENV" != "$ALLOWED_SOURCE" ]]; then
    echo "ERROR: Production can only be promoted from staging"
    exit 1
fi
```

For production, explicit workflow logic is preferable to relying only on developer discipline.

---

# 11. Production Promotion Branch

Create:

```bash
git checkout -b \
  promotion/order-service-staging-to-prod
```

Then:

```bash
./scripts/promote-image.sh \
  order-service \
  staging \
  prod
```

Check:

```bash
git diff
```

You should see:

```diff
-image:
-  tag: "old"
-  digest: "sha256:old..."
+image:
+  tag: "8f31a2c"
+  digest: "sha256:9a7b123..."
```

---

# 12. Important: Review the Entire Diff

Do not blindly merge.

Run:

```bash
git diff \
  -- environments/prod/order-service/values.yaml
```

Verify that only intended changes are present.

You don't want:

```text
image change
+
replica change
+
database change
+
resource change
+
Ingress change
```

in the same release unless those changes were deliberately included.

A clean application promotion should ideally look like:

```diff
 image:
-  tag: "old"
-  digest: "sha256:old..."
+  tag: "8f31a2c"
+  digest: "sha256:9a7b123..."
```

---

# 13. Commit

```bash
git add \
  environments/prod/order-service/values.yaml
```

Then:

```bash
git commit \
  -m "release: promote order-service from staging to prod"
```

Push:

```bash
git push \
  origin \
  promotion/order-service-staging-to-prod
```

---

# 14. Create Production PR

The PR should clearly state:

```text
Title:

release: promote order-service 8f31a2c to prod
```

Body:

```text
## Production Release

Service:
order-service

Source:
STAGING

Target:
PROD

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Tag:
8f31a2c

Digest:
sha256:9a7b123...

STAGING runtime digest:
sha256:9a7b123...

ECR digest:
sha256:9a7b123...

Validation:
- Unit tests: PASS
- Security scan: PASS
- QA: PASS
- STAGING: PASS
- Helm validation: PASS

Rollback:
Revert this GitOps commit to restore the previous image.
```

---

# 15. Production PR Is the Release Record

This PR is now a very useful audit record.

Someone can answer:

> What exactly are we deploying?

```text
order-service
```

> Which artifact?

```text
sha256:9a7b123...
```

> Where was it tested?

```text
DEV
QA
STAGING
```

> Who approved the production change?

GitHub PR approval records.

> What changed?

Git diff.

> How do we rollback?

Git revert.

---

# 16. Production Approval

Production should have protected controls.

Conceptually:

```text
Production PR
      │
      ├── GitOps CI PASS
      ├── Required reviewers
      ├── Security checks
      ├── Change review
      └── Release approval
              │
              ▼
            MERGE
```

The exact number/type of reviewers is organization-specific.

For our learning architecture, we'll use at least:

```text
Platform/DevOps review
+
Application owner review
```

---

# 17. Production ArgoCD

Our production Application is:

```text
order-service-prod
```

ArgoCD watches:

```text
environments/prod/order-service
```

Important difference:

### DEV

Can use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### PROD

We deliberately omit automatic synchronization:

```yaml
syncPolicy:
  # no automated sync
```

So merging the GitOps PR changes the desired state, but production deployment still requires the intended production synchronization step.

---

# 18. Why Manual Production Sync?

Imagine:

```text
GitOps merge
     ↓
ArgoCD automatic sync
     ↓
PROD deployment
```

That can be appropriate in some organizations, but our learning architecture intentionally introduces a production release gate.

So:

```text
GitOps merge
     ↓
ArgoCD OutOfSync
     ↓
Production release operator
     ↓
Sync
```

This gives us another deliberate checkpoint.

---

# 19. Check ArgoCD

Run:

```bash
argocd app get order-service-prod
```

You may see:

```text
Sync Status: OutOfSync
Health Status: Healthy
```

Why?

Because:

```text
Git desired state
      ≠
current Kubernetes state
```

The GitOps change has been merged, but the new image has not yet been applied.

---

# 20. Review the ArgoCD Diff

Before syncing:

```bash
argocd app diff order-service-prod
```

You should see the image change.

Conceptually:

```diff
-image: ...@sha256:OLD
+image: ...@sha256:9a7b123...
```

This is another important safety check.

---

# 21. Production Sync

Once approved:

```bash
argocd app sync order-service-prod
```

ArgoCD now reconciles:

```text
GitOps
  ↓
Helm
  ↓
Kubernetes
```

---

# 22. What Kubernetes Does

Suppose PROD currently has:

```text
Pod A → OLD
Pod B → OLD
Pod C → OLD
```

Deployment changes to:

```text
NEW
```

Kubernetes performs a rolling update.

Conceptually:

```text
OLD OLD OLD
    │
    ▼
NEW OLD OLD
    │
    ▼
NEW NEW OLD
    │
    ▼
NEW NEW NEW
```

The exact rollout behavior depends on:

```text
maxSurge
maxUnavailable
Pod readiness
resource capacity
PDB
```

which we configured in our Helm implementation.

---

# 23. Readiness Is Critical

A Pod should not receive production traffic simply because:

```text
container started
```

Our application has readiness checks.

For example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

Kubernetes waits for the Pod to become Ready.

Then the Service can route traffic to it.

---

# 24. Production Deployment Flow

The complete rollout:

```text
ArgoCD Sync
    ↓
Deployment updated
    ↓
New Pod created
    ↓
Container starts
    ↓
Startup probe
    ↓
Readiness probe
    ↓
Pod Ready
    ↓
Service endpoint added
    ↓
Traffic reaches new Pod
    ↓
Old Pod removed
```

This is how we achieve a controlled rolling deployment.

---

# 25. Watch the Rollout

Run:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app.kubernetes.io/name=order-service \
  -w
```

Watch the transition.

---

# 26. Verify Production Digest

After the rollout:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app.kubernetes.io/name=order-service
```

Then:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
.../order-service@sha256:9a7b123...
```

---

# 27. Final Artifact Verification

Now we can prove:

```text
ECR
sha256:9a7b123...
        │
        ├── DEV
        │
        ├── QA
        │
        ├── STAGING
        │
        └── PROD
             │
             └── sha256:9a7b123...
```

That is the result we wanted from the beginning.

---

# 28. Production Smoke Test

After deployment, run controlled smoke tests.

For example:

```bash
curl https://order-service.example.com/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

Then test an appropriate business API.

For example:

```bash
curl \
  https://order-service.example.com/api/v1/orders
```

Use authentication and appropriate test credentials in a real environment.

---

# 29. Monitor Immediately After Deployment

Don't deploy and walk away.

Watch:

```text
Request rate
Error rate
Latency
CPU
Memory
Pod restarts
Database connections
Redis latency
SQS backlog
ALB 4xx
ALB 5xx
```

Our observability architecture helps:

```text
Application
   │
   ├── Metrics → Prometheus
   ├── Logs → Fluent Bit → Logstash → Elasticsearch
   └── Traces → OTel → Jaeger
```

Grafana gives us the operational view.

---

# 30. Example Production Dashboard

Immediately after deployment, look at:

```text
order-service
─────────────────────────────

Requests/sec        150
Error rate          0.1%
P95 latency         180ms
CPU                 42%
Memory              61%
Pod restarts        0
RDS connections     120
SQS backlog         0
```

The exact thresholds are application/SLO dependent.

The important idea is:

> **Deployment validation continues after Kubernetes reports success.**

---

# 31. What If Error Rate Increases?

Suppose before deployment:

```text
Error rate:
0.1%
```

After deployment:

```text
Error rate:
8%
```

Stop the rollout investigation.

Don't wait for customers to discover the problem.

Possible investigation:

```text
Grafana
   ↓
Which metric changed?
   ↓
Prometheus
   ↓
Which Pod/version?
   ↓
Logs
   ↓
Exception?
   ↓
Trace
   ↓
Dependency?
```

---

# 32. Production Rollback

Suppose the new version is broken.

Current:

```text
NEW
sha256:9a7b123...
```

Previous:

```text
OLD
sha256:123abc...
```

Because GitOps controls the deployment, rollback can be:

```text
Revert production GitOps commit
        ↓
GitOps points to OLD digest
        ↓
ArgoCD detects change
        ↓
Sync
        ↓
Kubernetes rolling update
```

---

# 33. Git Revert

Suppose:

```bash
git log --oneline
```

shows:

```text
abc123 release: promote order-service to prod
def456 previous release
```

Rollback:

```bash
git revert abc123
```

Then:

```bash
git push
```

The GitOps repository now declares the previous state.

---

# 34. Why Not `kubectl rollout undo` as the Main Mechanism?

Kubernetes supports:

```bash
kubectl rollout undo deployment/order-service
```

This can be useful for emergency operational recovery.

But in GitOps, the desired state should eventually be corrected in Git.

Otherwise:

```text
Kubernetes:
OLD

Git:
NEW
```

ArgoCD may later reconcile Kubernetes back to:

```text
NEW
```

That is GitOps drift.

Therefore the normal rollback model is:

```text
Git revert
   ↓
ArgoCD
   ↓
Kubernetes
```

---

# 35. Emergency Rollback

In a severe incident, an operator may need immediate action.

For example:

```text
Production outage
       ↓
Immediate mitigation
       ↓
kubectl/ArgoCD emergency rollback
       ↓
Service recovered
       ↓
GitOps repository corrected
       ↓
Permanent desired state restored
```

Emergency procedures should be documented in the incident runbook.

---

# 36. Database Compatibility

One of the biggest production deployment risks is not Kubernetes.

It is the database.

Suppose version 2 expects:

```text
new_column
```

but the database does not have it.

Then:

```text
New application
       ↓
Database query
       ↓
ERROR
```

Therefore production release should verify:

```text
Database migration complete
        +
Application backward compatibility
```

---

# 37. Expand-and-Contract Example

Suppose we need:

```text
customer_phone
```

### Step 1

Add nullable column:

```sql
ALTER TABLE orders
ADD COLUMN customer_phone VARCHAR(30);
```

Deploy new application.

### Step 2

New application starts using the column.

### Step 3

After all old application versions are gone:

```text
Remove old compatibility
```

This is much safer than:

```text
Drop old column
      ↓
Deploy new application
```

---

# 38. Production Pre-Deployment Checklist

Before syncing PROD:

```text
[ ] STAGING tests passed
[ ] STAGING runtime digest verified
[ ] ECR digest exists
[ ] Production GitOps PR reviewed
[ ] Helm validation passed
[ ] Security checks passed
[ ] Database migration reviewed
[ ] Backward compatibility confirmed
[ ] Rollback plan confirmed
[ ] Monitoring dashboards ready
[ ] Alerts ready
[ ] Capacity available
[ ] Deployment window appropriate
```

---

# 39. During Deployment Checklist

Watch:

```text
[ ] ArgoCD sync
[ ] Deployment status
[ ] Pod startup
[ ] Readiness
[ ] Pod restarts
[ ] Application errors
[ ] ALB errors
[ ] Latency
[ ] Database connections
[ ] Queue backlog
```

---

# 40. After Deployment Checklist

```text
[ ] All Pods Ready
[ ] Rollout successful
[ ] Runtime digest verified
[ ] Health check passed
[ ] Smoke tests passed
[ ] Error rate normal
[ ] Latency normal
[ ] Logs normal
[ ] Metrics normal
[ ] Traces normal
[ ] No unexpected alerts
```

---

# 41. Complete Production Flow

We now have the full release lifecycle:

```text
                         SOURCE
                           │
                           ▼
                        GitHub
                           │
                           ▼
                    GitHub Actions
                           │
             ┌─────────────┼─────────────┐
             │             │             │
            Test        Security       Build
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                          ECR
                           │
                      sha256:ABC
                           │
                           ▼
                          DEV
                           │
                      validation
                           │
                           ▼
                           QA
                           │
                      validation
                           │
                           ▼
                        STAGING
                           │
             ┌─────────────┼─────────────┐
             │             │             │
           Tests       Performance   Resilience
             │             │             │
             └─────────────┼─────────────┘
                           │
                           ▼
                  PRODUCTION RELEASE
                         GATE
                           │
                      Approval
                           │
                           ▼
                         PROD
                           │
                    ArgoCD Sync
                           │
                           ▼
                   Rolling Deployment
                           │
                           ▼
                  Runtime Verification
```

---

# 42. The Complete Artifact Chain

This is the most important picture from Chapters 75–78:

```text
                    ECR
                     │
              sha256:ABC123
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
       DEV           QA         STAGING
        │            │            │
 sha256:ABC    sha256:ABC    sha256:ABC
        │            │            │
        └────────────┼────────────┘
                     │
                     ▼
                    PROD
                     │
                sha256:ABC
```

There is **one artifact**.

There are multiple environments.

---

# 43. What We Have Built So Far

Our project now has:

```text
01. GitHub repositories
02. Branch strategy
03. Application source
04. Docker
05. Local development
06. Terraform
07. Terraform state
08. VPC
09. IAM
10. ECR
11. EKS
12. EKS add-ons
13. GitHub Actions
14. Helm
15. GitOps
16. ArgoCD
17. ApplicationSet
18. 21-microservice GitOps structure
19. Helm versioning
20. Immutable images
21. CI → GitOps automation
22. DEV promotion
23. QA promotion
24. STAGING promotion
25. PROD release gate
26. Production rollback
```

This is now becoming a genuine platform architecture rather than just a collection of Kubernetes YAML files.

---

# 44. One Simple Mental Model

Whenever you get confused, remember:

### CI answers:

> **"Can we build and trust this artifact?"**

```text
Code
 ↓
Test
 ↓
Scan
 ↓
Image
 ↓
ECR
```

### GitOps answers:

> **"Which artifact should each environment run?"**

```text
DEV
QA
STAGING
PROD
```

### ArgoCD answers:

> **"Is Kubernetes actually running what Git says?"**

```text
Git
 ↓
ArgoCD
 ↓
EKS
```

### Kubernetes answers:

> **"Can I keep the desired workload running?"**

```text
Deployment
 ↓
ReplicaSet
 ↓
Pods
```

### Observability answers:

> **"Is the application actually working?"**

```text
Metrics
Logs
Traces
Alerts
```

---

# 45. Final Production Architecture

```text
                         USERS
                           │
                           ▼
                    Route 53 / CDN
                           │
                           ▼
                         ALB
                           │
                           ▼
                    EKS PRODUCTION
                           │
             ┌─────────────┼─────────────┐
             │             │             │
           Pods          Pods          Pods
             │             │             │
             └─────────────┼─────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
             RDS         Redis       SQS/SNS
              │            │            │
              └────────────┼────────────┘
                           │
                       Observability
                           │
            ┌──────────────┼──────────────┐
            │              │              │
         Metrics         Logs           Traces
            │              │              │
       Prometheus       ELK/            OTel/
        Grafana        Kibana            Jaeger


                  DEPLOYMENT CONTROL PLANE

Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ▼
ECR
    │
    ▼
Immutable Digest
    │
    ▼
DEV → QA → STAGING
              │
              ▼
       Production Gate
              │
              ▼
        PROD GitOps
              │
              ▼
            ArgoCD
              │
              ▼
        EKS Production
```

---

## Chapter 78 outcome

We now have a complete **immutable-artifact promotion lifecycle**:

```text
Code
 ↓
CI
 ↓
ECR
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
Production approval
 ↓
PROD
```

And the same image digest can be traced through every environment.

**Next: `79-Production-Deployment-Strategies.md`**

We will go deeper into **Rolling Update, Recreate, Blue-Green, Canary, Argo Rollouts, traffic splitting, rollback behavior, when each strategy is used, and then implement a practical strategy for our 21-microservice EKS platform.**
