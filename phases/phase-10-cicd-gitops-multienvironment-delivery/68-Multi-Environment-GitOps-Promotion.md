# 68 — Multi-Environment GitOps Promotion

## 1. Objective

In the previous chapters we automated:

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Docker
   ↓
Trivy
   ↓
ECR
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

Now we implement controlled promotion across:

```text
DEV → QA → STAGING → PROD
```

The most important rule is:

> **We promote the same immutable artifact between environments. We do not rebuild the application for each environment.**

---

# 2. Target Architecture

```text
                    GitHub
                       │
                       ▼
                 Application PR
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
                       image + digest
                              │
                              ▼
                       GitOps DEV
                              │
                              ▼
                           ArgoCD
                              │
                              ▼
                             EKS
                              │
                         DEV testing
                              │
                              ▼
                       GitOps QA PR
                              │
                              ▼
                           QA EKS
                              │
                              ▼
                    Integration testing
                              │
                              ▼
                    GitOps STAGING PR
                              │
                              ▼
                        STAGING EKS
                              │
                              ▼
                    Production approval
                              │
                              ▼
                      GitOps PROD PR
                              │
                              ▼
                         PROD EKS
```

---

# 3. Four Environments

We now formally define the environments.

| Environment | Purpose                            |
| ----------- | ---------------------------------- |
| DEV         | Developer/integration validation   |
| QA          | Functional and integration testing |
| STAGING     | Production-like validation         |
| PROD        | Customer-facing workload           |

Each environment can have different:

```text
replicas
resources
hostnames
database
Redis
queues
feature configuration
autoscaling
network configuration
```

But the **application image should remain the same** during promotion.

---

# 4. Repository Structure

Our GitOps repository becomes:

```text
gitops/
│
├── environments/
│   │
│   ├── dev/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   ├── qa/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   ├── staging/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   └── prod/
│       └── order-service/
│           └── values.yaml
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
├── scripts/
│   ├── promote-image.sh
│   └── verify-image.sh
│
└── .github/
    └── workflows/
        └── validate.yaml
```

---

# 5. The Same Image

Suppose CI creates:

```text
order-service:8f31a2c
```

ECR gives:

```text
sha256:abc123...
```

We want:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 8f31a2c
PROD      → 8f31a2c
```

Not:

```text
DEV       → build A
QA        → build B
STAGING   → build C
PROD      → build D
```

---

# 6. Why Rebuilding for Production Is Wrong

Imagine:

```text
DEV
image: 8f31a2c
```

Then someone runs a new build for PROD:

```text
PROD
image: 91abc77
```

Even if both came from "the same source branch", they may differ because of:

```text
Dependency changes
Base image changes
Build environment differences
Generated artifacts
Build-time configuration
Toolchain differences
```

Therefore:

```text
Build once
Promote many times
```

is the safer deployment model.

---

# 7. Tags vs Digests

Our initial implementation uses:

```yaml
image:
  tag: "8f31a2c"
```

For stronger production control, we should eventually use:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:abc123..."
```

The important distinction:

```text
Tag:
8f31a2c

Digest:
sha256:abc123...
```

The digest identifies the exact image content.

---

# 8. Improve Helm to Support Digest

Our current chart uses:

```yaml
image:
  repository: ...
  tag: ...
```

We can extend it.

In:

```text
order-service/helm/order-service/values.yaml
```

use:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"
  digest: ""
  pullPolicy: IfNotPresent
```

---

# 9. Update Deployment Template

The current image template may look like:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Change it to support either tag or digest.

Example:

```yaml
image: >-
  {{ .Values.image.repository }}:{{ .Values.image.tag }}
  {{- if .Values.image.digest }}
  @{{ .Values.image.digest }}
  {{- end }}
```

However, this creates:

```text
repository:tag@digest
```

which is valid as an image reference but requires careful formatting.

A cleaner Helm approach is:

```yaml
image: "{{ .Values.image.repository }}{{ if .Values.image.digest }}@{{ .Values.image.digest }}{{ else }}:{{ .Values.image.tag }}{{ end }}"
```

Now:

If digest exists:

```text
repository@sha256:abc123
```

If digest is empty:

```text
repository:8f31a2c
```

---

# 10. Development Values

DEV can initially use the tag:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: ""
```

This keeps development simple.

---

# 11. Production Values

Production can eventually use:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:abc123..."
```

The deployment actually uses:

```text
repository@sha256:abc123...
```

The tag remains useful for human traceability.

---

# 12. Promotion Data

A release should carry:

```text
SERVICE
IMAGE TAG
IMAGE DIGEST
SOURCE COMMIT
```

Example:

```text
Service:
order-service

Tag:
8f31a2c

Digest:
sha256:abc123...

Source:
8f31a2c
```

This information should follow the artifact through all environments.

---

# 13. DEV Promotion

After CI succeeds:

```text
GitHub Actions
      │
      ▼
ECR
      │
      ▼
image: 8f31a2c
      │
      ▼
GitOps DEV
```

GitOps:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

ArgoCD detects:

```text
OutOfSync
```

and deploys:

```text
order-service:8f31a2c
```

---

# 14. DEV Validation

After deployment:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-dev
```

Health:

```bash
curl https://order-dev.example.com/actuator/health
```

Application API:

```bash
curl \
  https://order-dev.example.com/api/v1/orders
```

Check logs:

```bash
kubectl logs \
  deployment/order-service \
  -n ecommerce-dev
```

---

# 15. What Does DEV Approval Mean?

DEV validation should establish that:

```text
Application starts
Database connectivity works
Health probes work
API responds
Basic business flow works
No critical deployment errors
```

DEV does not necessarily mean:

```text
Production ready
```

It means the artifact is ready to progress to the next validation environment.

---

# 16. QA Promotion

After DEV validation, create:

```text
promote/order-service-8f31a2c-to-qa
```

Change:

```text
environments/qa/order-service/values.yaml
```

from:

```yaml
image:
  tag: "previous"
```

to:

```yaml
image:
  tag: "8f31a2c"
```

Commit:

```bash
git commit \
  -m "promote(order-service): 8f31a2c to qa"
```

Open PR.

---

# 17. QA PR Should Show

The PR should clearly show:

```diff
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
-  tag: "72c1abc"
+  tag: "8f31a2c"
```

Only the intended environment changes.

---

# 18. QA Environment

ArgoCD application:

```text
order-service-qa
```

Destination:

```text
ecommerce-qa
```

Values:

```text
environments/qa/order-service/values.yaml
```

Example:

```yaml
replicaCount: 2

config:
  springProfile: qa

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
```

The image remains:

```text
8f31a2c
```

---

# 19. QA Testing

QA can perform:

```text
Unit/integration validation
API testing
Database testing
Authentication testing
Order creation
Order retrieval
Negative test cases
Dependency failure testing
Performance checks
```

The important point is:

```text
Same application artifact
Different environment configuration
```

---

# 20. STAGING Promotion

After QA approval:

```text
QA
 │
 │ approved
 ▼
STAGING PR
```

Update:

```text
environments/staging/order-service/values.yaml
```

to:

```yaml
image:
  tag: "8f31a2c"
```

ArgoCD deploys to:

```text
ecommerce-staging
```

---

# 21. Why STAGING Exists

STAGING should resemble production as much as practical.

For example:

```text
Production:
3 replicas
large resources
production-like ALB
production-like database topology
production-like monitoring
```

Staging should use similar:

```text
deployment configuration
networking
observability
scaling
security controls
```

but should remain isolated from production data and production credentials.

---

# 22. Production Promotion

Production should require a controlled PR.

Example:

```text
promote/order-service-8f31a2c-to-prod
```

Change:

```text
environments/prod/order-service/values.yaml
```

from:

```yaml
image:
  tag: "72c1abc"
```

to:

```yaml
image:
  tag: "8f31a2c"
```

For digest-based deployment:

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:abc123..."
```

---

# 23. Production PR Checklist

Use:

```markdown
## Production Promotion

Service: order-service

Release:
8f31a2c

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Digest:
sha256:abc123...

Previous version:
72c1abc

Validation:

- [ ] CI passed
- [ ] Unit tests passed
- [ ] Security scan passed
- [ ] DEV validated
- [ ] QA validated
- [ ] STAGING validated
- [ ] Database compatibility checked
- [ ] Monitoring verified
- [ ] Rollback version confirmed
- [ ] Production approval obtained
```

---

# 24. Production Approval

The intended workflow is:

```text
Production PR
      │
      ▼
Automated validation
      │
      ▼
Required reviewers
      │
      ▼
Approval
      │
      ▼
Merge
      │
      ▼
ArgoCD
      │
      ▼
Production EKS
```

The CI system should not bypass this process.

---

# 25. ArgoCD Production Sync

Once GitOps changes:

```text
prod/order-service/values.yaml
```

ArgoCD sees:

```text
Git:
8f31a2c

Cluster:
72c1abc
```

Therefore:

```text
OutOfSync
```

The production release process then synchronizes the application according to the configured production sync policy.

---

# 26. Production Deployment

The deployment proceeds using the normal rolling strategy:

```text
Old:
72c1abc
```

New:

```text
8f31a2c
```

Kubernetes:

```text
Old Pods
   │
   ├── old
   ├── old
   └── old
         │
         ▼
New ReplicaSet
   │
   ├── new
   ├── new
   └── new
```

Readiness probes determine when new Pods can receive traffic.

---

# 27. Production Verification

Immediately after deployment:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-prod
```

Check:

```bash
kubectl get endpoints \
  order-service \
  -n ecommerce-prod
```

Check ALB:

```bash
kubectl get ingress \
  -n ecommerce-prod
```

Then application health:

```bash
curl \
  https://order.example.com/actuator/health
```

---

# 28. Verify the Running Image

Always verify the actual production image.

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

For digest-based deployment:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:abc123...
```

---

# 29. Verify All Pods

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app.kubernetes.io/name=order-service \
  -o wide
```

Check:

```text
READY
STATUS
RESTARTS
NODE
```

Target:

```text
1/1 Running
```

with no unexpected restarts.

---

# 30. Verify ArgoCD

```bash
argocd app get order-service-prod
```

Target:

```text
Sync Status: Synced
Health Status: Healthy
```

Also:

```bash
argocd app history order-service-prod
```

This provides the ArgoCD deployment history.

---

# 31. Release Traceability

For release `8f31a2c` we should be able to establish:

```text
GitHub PR #250
       │
       ▼
Source commit 8f31a2c
       │
       ▼
GitHub Actions run #812
       │
       ▼
ECR image
       │
       ▼
sha256:abc123
       │
       ▼
DEV GitOps PR #400
       │
       ▼
QA GitOps PR #405
       │
       ▼
STAGING GitOps PR #410
       │
       ▼
PROD GitOps PR #415
       │
       ▼
ArgoCD sync
       │
       ▼
Production
```

This is the release chain we want.

---

# 32. Rollback Scenario

Suppose production version:

```text
8f31a2c
```

has an application problem.

Previous production version:

```text
72c1abc
```

We do not rebuild.

We simply revert the GitOps change.

```text
GitOps
   │
   ▼
Revert production promotion
   │
   ▼
image: 72c1abc
   │
   ▼
ArgoCD
   │
   ▼
EKS
```

---

# 33. Rollback Command

Find the production GitOps commit:

```bash
git log --oneline \
  -- environments/prod/order-service/values.yaml
```

Then:

```bash
git revert <promotion-commit>
```

Push:

```bash
git push origin main
```

ArgoCD sees:

```text
desired:
72c1abc
```

and reconciles.

---

# 34. Emergency Rollback

During a serious incident, the organization may have an emergency procedure.

The important principle remains:

```text
Emergency
   ↓
Restore known-good desired state
   ↓
Git
   ↓
ArgoCD
   ↓
EKS
```

Avoid leaving production in a manually modified state such as:

```bash
kubectl set image ...
```

without subsequently recording that state in Git.

---

# 35. Rollback Verification

After rollback:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-prod
```

Verify image:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Then test:

```bash
curl \
  https://order.example.com/actuator/health
```

And business API:

```bash
curl \
  https://order.example.com/api/v1/orders
```

---

# 36. What If QA Finds a Bug?

Suppose:

```text
DEV → 8f31a2c
QA  → 8f31a2c
```

QA discovers a bug.

We do **not** promote to STAGING or PROD.

State remains:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 72c1abc
PROD      → 72c1abc
```

Developer creates:

```text
9c21abc
```

After fixing:

```text
DEV → 9c21abc
```

Then QA tests:

```text
9c21abc
```

This is a controlled promotion process.

---

# 37. What If DEV Is Updated Again?

Suppose:

```text
DEV → 9c21abc
```

while QA is still testing:

```text
QA → 8f31a2c
```

That's completely possible.

The environments are independent desired states.

Therefore:

```text
DEV
9c21abc

QA
8f31a2c

STAGING
72c1abc

PROD
72c1abc
```

This is one of the main benefits of GitOps.

---

# 38. Environment Configuration vs Application Artifact

This distinction is critical.

The **artifact**:

```text
order-service
```

contains:

```text
Java application
dependencies
compiled code
Docker filesystem
```

The **environment configuration** contains:

```text
replicas
resources
hostname
environment variables
scaling
deployment policy
```

Therefore:

```text
Same artifact
+
Different environment configuration
```

is the normal model.

---

# 39. Example

DEV:

```yaml
replicaCount: 2

resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

PROD:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 500m
    memory: 768Mi
```

But:

```yaml
image:
  tag: "8f31a2c"
```

is the same.

---

# 40. Database Separation

Never point DEV and PROD at the same database simply because the image is the same.

Example:

```text
DEV
   ↓
dev RDS

QA
   ↓
qa RDS

STAGING
   ↓
staging RDS

PROD
   ↓
production RDS
```

The application image remains:

```text
8f31a2c
```

but database endpoints differ.

---

# 41. Secrets Separation

Likewise:

```text
DEV
 ↓
dev secrets

QA
 ↓
qa secrets

STAGING
 ↓
staging secrets

PROD
 ↓
prod secrets
```

The GitOps repository should contain references, not credentials.

---

# 42. AWS Account Separation

As the platform becomes more enterprise-grade, environments can be separated by AWS account:

```text
AWS Organization
│
├── Management
├── Security
├── Logging
├── Development
├── QA
├── Staging
└── Production
```

Then:

```text
DEV ArgoCD → DEV account/EKS
QA ArgoCD → QA account/EKS
STAGING ArgoCD → STAGING account/EKS
PROD ArgoCD → PROD account/EKS
```

For a learning implementation, one AWS account with separate EKS namespaces/clusters can be used initially.

---

# 43. Namespace Separation

If using a single cluster initially:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

Example:

```bash
kubectl get namespaces
```

Expected:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

Production isolation is stronger with separate clusters/accounts, but namespaces are useful for learning and smaller environments.

---

# 44. ArgoCD Applications

We now have:

```text
order-service-dev
order-service-qa
order-service-staging
order-service-prod
```

Each Application points to:

```text
same Helm chart
+
different GitOps values
+
different Kubernetes namespace/cluster
```

Conceptually:

```text
                  order-service
                       │
                    Helm chart
                       │
        ┌──────────────┼──────────────┐
        │              │              │
       DEV            QA          STAGING
        │              │              │
        ▼              ▼              ▼
      values         values         values
        │              │              │
        ▼              ▼              ▼
       EKS            EKS            EKS
                       │
                       ▼
                      PROD
```

---

# 45. ApplicationSet Direction

Creating four Applications for one service is manageable.

For 21 services:

```text
21 × 4 = 84 Applications
```

Manually maintaining 84 Application definitions becomes repetitive.

Therefore we eventually use:

```text
ApplicationSet
```

to generate applications.

Conceptually:

```text
ApplicationSet
       │
       ├── auth-dev
       ├── auth-qa
       ├── auth-staging
       ├── auth-prod
       │
       ├── order-dev
       ├── order-qa
       ├── order-staging
       ├── order-prod
       │
       └── ...
```

This will be covered after the promotion model is established.

---

# 46. Release Metadata

It is useful to store release metadata in GitOps.

For example:

```text
releases/
└── order-service/
    └── 8f31a2c.yaml
```

Example:

```yaml
service: order-service

sourceCommit: 8f31a2c

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: 8f31a2c
  digest: sha256:abc123...

createdAt: "2026-09-21T10:00:00Z"
```

This is optional.

The Git history itself may provide enough traceability, so don't create duplicate metadata unless it serves a real operational purpose.

---

# 47. Git Tags

For releases, application Git tags can be useful:

```text
order-service-v1.4.0
```

However:

```text
Git release tag
```

and:

```text
Docker image tag
```

serve different purposes.

Example:

```text
Git:
v1.4.0

Docker:
8f31a2c

Digest:
sha256:abc123
```

The digest remains the immutable artifact identifier.

---

# 48. Release Naming

A practical model:

```text
Application release:
v1.4.0

Source commit:
8f31a2c

Container:
order-service:8f31a2c

Digest:
sha256:abc123
```

This gives humans a release name while preserving technical traceability.

---

# 49. Promotion Rules

A useful rule set:

### DEV

```text
CI passes
→ promote automatically or with lightweight approval
```

### QA

```text
DEV validation
→ promotion PR
```

### STAGING

```text
QA validation
→ promotion PR
```

### PROD

```text
STAGING validation
→ production approval
→ promotion PR
```

The exact approval policy should match the organization's release controls.

---

# 50. Prevent Skipping Environments

If the desired policy is:

```text
DEV → QA → STAGING → PROD
```

the promotion workflow should verify that the candidate image was previously promoted.

For example, before PROD:

```text
Is 8f31a2c present in STAGING?
```

If:

```text
NO
```

then:

```text
PROD promotion = FAIL
```

---

# 51. Promotion Verification Script

Create:

```text
scripts/verify-promotion.sh
```

Example:

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="${1:?Service required}"
IMAGE_TAG="${2:?Image tag required}"
ENVIRONMENT="${3:?Environment required}"

VALUES_FILE="environments/${ENVIRONMENT}/${SERVICE}/values.yaml"

if [[ ! -f "$VALUES_FILE" ]]; then
  echo "ERROR: ${VALUES_FILE} does not exist"
  exit 1
fi

CURRENT_TAG=$(yq -r '.image.tag' "$VALUES_FILE")

if [[ "$CURRENT_TAG" != "$IMAGE_TAG" ]]; then
  echo "ERROR: ${SERVICE} in ${ENVIRONMENT} is not running ${IMAGE_TAG}"
  echo "Current: ${CURRENT_TAG}"
  exit 1
fi

echo "Verified ${SERVICE} ${IMAGE_TAG} in ${ENVIRONMENT}"
```

---

# 52. Example Promotion Check

Before PROD:

```bash
./scripts/verify-promotion.sh \
  order-service \
  8f31a2c \
  staging
```

If successful:

```text
Verified order-service 8f31a2c in staging
```

Then production promotion can proceed.

---

# 53. Important Limitation

Checking Git alone tells us:

```text
STAGING desired state = 8f31a2c
```

It doesn't necessarily prove:

```text
STAGING cluster successfully ran 8f31a2c
```

For stronger promotion gates, CI/CD can query ArgoCD:

```text
Application
    ↓
Synced?
    ↓
Healthy?
```

before allowing the next promotion.

---

# 54. Stronger Promotion Gate

The desired model becomes:

```text
DEV GitOps
   ↓
ArgoCD
   ↓
DEV Healthy
   ↓
QA Promotion
   ↓
ArgoCD
   ↓
QA Healthy
   ↓
STAGING Promotion
   ↓
ArgoCD
   ↓
STAGING Healthy
   ↓
PROD Approval
```

This is much stronger than simply copying image tags.

---

# 55. Production Deployment Gate

For PROD, verify:

```text
Image exists
       ↓
Image digest matches
       ↓
STAGING desired image matches
       ↓
STAGING ArgoCD Healthy
       ↓
Automated tests passed
       ↓
Approval
       ↓
Production GitOps update
```

---

# 56. Failed Promotion

If STAGING is unhealthy:

```text
STAGING
   │
   ▼
ArgoCD
   │
   ▼
Unhealthy
   │
   X
PROD promotion
```

Do not promote.

Investigate:

```bash
argocd app get order-service-staging
```

Then:

```bash
kubectl get pods -n ecommerce-staging
```

and:

```bash
kubectl logs deployment/order-service \
  -n ecommerce-staging
```

---

# 57. Promotion Dashboard

Eventually Grafana can display:

```text
Order Service
──────────────

DEV        8f31a2c   Healthy
QA         8f31a2c   Healthy
STAGING    8f31a2c   Healthy
PROD       72c1abc   Healthy
```

This immediately tells the platform team:

```text
Current production:
72c1abc

Candidate:
8f31a2c
```

Observability will be implemented in later chapters.

---

# 58. Complete Promotion Example

Suppose CI produces:

```text
8f31a2c
```

### Step 1

ECR:

```text
order-service:8f31a2c
```

### Step 2

DEV:

```text
8f31a2c
```

### Step 3

DEV passes.

### Step 4

QA:

```text
8f31a2c
```

### Step 5

QA passes.

### Step 6

STAGING:

```text
8f31a2c
```

### Step 7

STAGING passes.

### Step 8

Production approval.

### Step 9

PROD:

```text
8f31a2c
```

Final state:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 8f31a2c
PROD      → 8f31a2c
```

---

# 59. What Happens During a Hotfix?

Suppose production has:

```text
72c1abc
```

A critical fix is developed:

```text
99a7bcd
```

The same pipeline applies:

```text
99a7bcd
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
PROD
```

If the organization has an emergency-release procedure, some gates may be accelerated, but the artifact should still remain traceable and immutable.

---

# 60. What We Have Achieved

We now have:

```text
Code
 ↓
CI
 ↓
Immutable ECR artifact
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

with:

```text
Git
 ↓
ArgoCD
 ↓
EKS
```

controlling every deployment.

---

# 61. Final Architecture

```text
                           GitHub
                              │
                              ▼
                         Source PR
                              │
                              ▼
                       GitHub Actions
                              │
             ┌────────────────┼────────────────┐
             │                │                │
           Build             Test           Security
             │                │                │
             └────────────────┼────────────────┘
                              │
                              ▼
                         Docker Image
                              │
                              ▼
                           Trivy
                              │
                              ▼
                             ECR
                              │
                              ▼
                       Immutable Digest
                              │
                              ▼
                        GitOps DEV
                              │
                              ▼
                           ArgoCD
                              │
                              ▼
                          EKS DEV
                              │
                        validation
                              │
                              ▼
                        GitOps QA
                              │
                              ▼
                          EKS QA
                              │
                        validation
                              │
                              ▼
                      GitOps STAGING
                              │
                              ▼
                       EKS STAGING
                              │
                        approval
                              │
                              ▼
                       GitOps PROD
                              │
                              ▼
                        ArgoCD PROD
                              │
                              ▼
                          EKS PROD
```

---

# 62. Key Rules to Remember

### Rule 1

```text
Never rebuild an image just because it is going to PROD.
```

### Rule 2

```text
Promote the same immutable artifact.
```

### Rule 3

```text
Environment configuration belongs in GitOps.
```

### Rule 4

```text
Secrets do not belong in GitOps.
```

### Rule 5

```text
CI should not directly deploy to production.
```

### Rule 6

```text
ArgoCD is the deployment reconciler.
```

### Rule 7

```text
Git is the desired-state source of truth.
```

### Rule 8

```text
Rollback should restore a known-good Git state.
```

---

# Next Chapter

## `69-ArgoCD-ApplicationSet-and-21-Microservices.md`

Now that `order-service` has a complete:

```text
CI → ECR → GitOps → ArgoCD → EKS
```

pipeline, the next major problem appears:

**How do we manage 21 microservices without manually creating and maintaining dozens of ArgoCD Applications?**

We will implement:

```text
ApplicationSet
      │
      ├── auth-service
      ├── user-service
      ├── customer-service
      ├── catalog-service
      ├── product-service
      ├── inventory-service
      ├── pricing-service
      ├── promotion-service
      ├── cart-service
      ├── order-service
      ├── payment-service
      ├── fraud-service
      ├── shipping-service
      ├── delivery-service
      ├── notification-service
      ├── search-service
      ├── recommendation-service
      ├── review-service
      ├── tax-service
      ├── invoice-service
      └── reporting-service
```

and build the **scalable GitOps structure for all 21 services across DEV, QA, STAGING, and PROD**.
