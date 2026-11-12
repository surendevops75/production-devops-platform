# 76 — GitOps Promotion: DEV → QA

> **Goal:** Take the **exact image already deployed and validated in DEV** and promote that same immutable image to QA.
>
> We will **not rebuild the application** for QA.

---

## 1. What We Have Now

From Chapter 75:

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Docker Build
   ↓
Trivy
   ↓
ECR
   ↓
Image Digest
   ↓
GitOps DEV PR
   ↓
Merge
   ↓
ArgoCD
   ↓
EKS DEV
```

For example, DEV is running:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

Now we want:

```text
DEV
 │
 │ validate
 ▼
QA
```

with the **same**:

```text
sha256:9a7b123...
```

---

# 2. The Most Important Rule

The promotion process is:

```text
Build once
    ↓
Test once
    ↓
Scan once
    ↓
Push once
    ↓
Promote same artifact
```

Not:

```text
DEV
 ↓
rebuild
 ↓
QA
 ↓
rebuild
 ↓
STAGING
 ↓
rebuild
 ↓
PROD
```

Why?

Because rebuilding can produce a different artifact.

Our desired model is:

```text
                ECR
                 │
        sha256:ABC123
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
       DEV       QA      PROD
```

---

# 3. Environment Configuration vs Application Artifact

This distinction is extremely important.

The **image** remains the same:

```text
order-service
sha256:9a7b123...
```

But environment configuration can differ.

For example:

### DEV

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
```

### QA

```yaml
replicaCount: 2

resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

### Production

```yaml
replicaCount: 4

resources:
  requests:
    cpu: 500m
    memory: 1Gi
```

The application image is still:

```text
sha256:9a7b123...
```

---

# 4. Our GitOps Structure

We already have:

```text
gitops/
└── environments/
    ├── dev/
    │   └── order-service/
    │       ├── app.yaml
    │       └── values.yaml
    │
    ├── qa/
    │   └── order-service/
    │       ├── app.yaml
    │       └── values.yaml
    │
    ├── staging/
    │
    └── prod/
```

So:

```text
DEV
→ environments/dev/order-service/values.yaml

QA
→ environments/qa/order-service/values.yaml
```

---

# 5. DEV Values

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."

replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 250m
    memory: 512Mi
```

---

# 6. QA Values

Initially:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "previous"
  digest: "sha256:previous..."

replicaCount: 2

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

Promotion changes only:

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

The QA-specific settings remain unchanged.

---

# 7. What Actually Performs the Promotion?

We have two possible approaches.

### Approach 1 — Developer manually creates PR

```text
DEV validated
     ↓
Developer creates QA PR
     ↓
Review
     ↓
Merge
```

### Approach 2 — Promotion automation creates PR

```text
DEV validation
     ↓
Promotion automation
     ↓
QA GitOps PR
     ↓
Review
     ↓
Merge
```

For our project, we'll use **GitOps PR-based promotion**.

That gives us:

```text
Automation
   +
Git review
   +
Audit history
```

---

# 8. Do Not Use the DEV Tag Alone

Suppose someone says:

```text
Promote order-service:latest
```

That is dangerous.

A tag can potentially move.

Instead, our promotion request should contain:

```text
Tag:
8f31a2c

Digest:
sha256:9a7b123...
```

The digest is the important identity.

---

# 9. Verify the Image Exists

Before creating the QA promotion PR, verify the digest exists in ECR.

Example:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageDigest=sha256:9a7b123... \
  --region ap-south-1
```

If ECR returns the image:

```text
image exists
```

continue.

If not:

```text
STOP
```

Do not create the promotion.

---

# 10. Verify DEV Is Actually Running That Digest

Before promotion, we can verify the DEV deployment.

First:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -l app.kubernetes.io/name=order-service
```

Find the Pod:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-dev \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

This confirms:

```text
GitOps says:
sha256:9a7b123...

Running Pod:
sha256:9a7b123...
```

---

# 11. Why This Verification Matters

Imagine:

```text
GitOps DEV:
sha256:ABC
```

but the Pod is still running:

```text
sha256:XYZ
```

Maybe:

```text
ArgoCD hasn't synced yet
```

or:

```text
Deployment rollout failed
```

or:

```text
old Pods are still running
```

We should not blindly promote based only on the Git file.

The promotion gate should establish:

```text
DEV GitOps
      +
DEV actual runtime
      +
ECR artifact
```

are consistent.

---

# 12. QA Promotion Script

Let's create:

```text
scripts/promote-image.sh
```

Example:

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="$1"
SOURCE_ENV="$2"
TARGET_ENV="$3"
IMAGE_TAG="$4"
IMAGE_DIGEST="$5"

SOURCE_FILE="environments/${SOURCE_ENV}/${SERVICE}/values.yaml"
TARGET_FILE="environments/${TARGET_ENV}/${SERVICE}/values.yaml"

echo "======================================"
echo "GitOps Promotion"
echo "======================================"

echo "Service       : ${SERVICE}"
echo "Source        : ${SOURCE_ENV}"
echo "Target        : ${TARGET_ENV}"
echo "Image Tag     : ${IMAGE_TAG}"
echo "Image Digest  : ${IMAGE_DIGEST}"
echo

if [[ ! -f "$SOURCE_FILE" ]]; then
    echo "ERROR: Source file does not exist:"
    echo "$SOURCE_FILE"
    exit 1
fi

if [[ ! -f "$TARGET_FILE" ]]; then
    echo "ERROR: Target file does not exist:"
    echo "$TARGET_FILE"
    exit 1
fi

export IMAGE_TAG
export IMAGE_DIGEST

yq -i \
  '.image.tag = strenv(IMAGE_TAG) |
   .image.digest = strenv(IMAGE_DIGEST)' \
  "$TARGET_FILE"

echo
echo "Updated target:"
cat "$TARGET_FILE"
```

---

# 13. Test the Promotion Script

Suppose:

```text
Service:
order-service

Source:
dev

Target:
qa

Tag:
8f31a2c

Digest:
sha256:9a7b123...
```

Run:

```bash
./scripts/promote-image.sh \
  order-service \
  dev \
  qa \
  8f31a2c \
  sha256:9a7b123...
```

The result:

```text
DEV
 │
 │ same image
 ▼
QA
```

---

# 14. But There Is an Important Improvement

The script above accepts:

```text
IMAGE_TAG
IMAGE_DIGEST
```

from the caller.

A safer promotion workflow should first verify:

```text
Does DEV actually contain this image?
```

For example:

```bash
DEV_DIGEST=$(yq -r '.image.digest' \
  environments/dev/order-service/values.yaml)
```

Then:

```bash
if [[ "$DEV_DIGEST" != "$IMAGE_DIGEST" ]]; then
    echo "ERROR: Image digest does not match DEV"
    exit 1
fi
```

Now we cannot accidentally promote a different image.

---

# 15. Safer Promotion Script

A better version:

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="$1"
SOURCE_ENV="$2"
TARGET_ENV="$3"

SOURCE_FILE="environments/${SOURCE_ENV}/${SERVICE}/values.yaml"
TARGET_FILE="environments/${TARGET_ENV}/${SERVICE}/values.yaml"

if [[ ! -f "$SOURCE_FILE" ]]; then
    echo "ERROR: Source file not found: $SOURCE_FILE"
    exit 1
fi

if [[ ! -f "$TARGET_FILE" ]]; then
    echo "ERROR: Target file not found: $TARGET_FILE"
    exit 1
fi

SOURCE_TAG=$(yq -r '.image.tag' "$SOURCE_FILE")
SOURCE_DIGEST=$(yq -r '.image.digest' "$SOURCE_FILE")

if [[ -z "$SOURCE_TAG" || "$SOURCE_TAG" == "null" ]]; then
    echo "ERROR: Source image tag is missing"
    exit 1
fi

if [[ -z "$SOURCE_DIGEST" || "$SOURCE_DIGEST" == "null" ]]; then
    echo "ERROR: Source image digest is missing"
    exit 1
fi

echo "Promoting:"
echo "Service : $SERVICE"
echo "Source  : $SOURCE_ENV"
echo "Target  : $TARGET_ENV"
echo "Tag     : $SOURCE_TAG"
echo "Digest  : $SOURCE_DIGEST"

export SOURCE_TAG
export SOURCE_DIGEST

yq -i \
  '.image.tag = strenv(SOURCE_TAG) |
   .image.digest = strenv(SOURCE_DIGEST)' \
  "$TARGET_FILE"

echo
echo "Promotion completed."
```

Now:

```text
DEV values
    │
    ├── tag
    └── digest
          │
          ▼
QA values
```

The target gets exactly what the source contains.

---

# 16. QA Promotion PR

After running the script:

```text
gitops/
└── environments/
    └── qa/
        └── order-service/
            └── values.yaml
```

changes.

Before:

```yaml
image:
  tag: "7ab123"
  digest: "sha256:old..."
```

After:

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

Then:

```bash
git checkout -b promotion/order-service-dev-to-qa
```

Commit:

```bash
git add environments/qa/order-service/values.yaml
```

```bash
git commit \
  -m "promote: order-service from dev to qa"
```

Push:

```bash
git push origin promotion/order-service-dev-to-qa
```

Create:

```text
DEV → QA promotion PR
```

---

# 17. What the QA PR Should Show

Example:

```text
Title:

promote: order-service 8f31a2c from dev to qa
```

Body:

```text
Service:
order-service

Source:
DEV

Target:
QA

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Tag:
8f31a2c

Digest:
sha256:9a7b123...

DEV validation:
PASS

DEV runtime:
PASS

ECR image:
EXISTS

Requested action:
Deploy the same immutable image to QA.
```

---

# 18. QA GitOps Validation

The PR should trigger GitOps CI.

For example:

```text
GitOps PR
   │
   ├── YAML validation
   ├── Helm lint
   ├── Helm template
   ├── Kubernetes schema validation
   └── Policy checks
```

If everything passes:

```text
CI
 ↓
PASS
```

---

# 19. QA Reviewer

The reviewer sees:

```text
DEV:
order-service@sha256:ABC

QA:
order-service@sha256:ABC
```

So the reviewer can verify:

```text
Same artifact?
YES

Different configuration?
Possibly

New application build?
NO
```

This is a clean promotion model.

---

# 20. Merge the QA PR

After approval:

```text
QA PR
  ↓
Merge
```

GitOps becomes:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

ArgoCD sees the change.

---

# 21. ArgoCD QA Application

Our ApplicationSet creates:

```text
order-service-qa
```

and points it to:

```text
environments/qa/order-service/values.yaml
```

ArgoCD:

```text
GitOps
  ↓
ApplicationSet
  ↓
Application
  ↓
Helm
  ↓
Kubernetes
```

---

# 22. QA Namespace

For example:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

So:

```text
DEV
└── ecommerce-dev
    └── order-service

QA
└── ecommerce-qa
    └── order-service
```

For stronger isolation, these environments should normally also be separated by AWS accounts/clusters according to the organization's security and operational requirements.

---

# 23. QA Deployment

ArgoCD syncs:

```text
order-service-qa
```

Then:

```bash
kubectl get pods -n ecommerce-qa
```

Example:

```text
NAME                              READY   STATUS
order-service-7d9f8c7f6d-x8abc    1/1     Running
order-service-7d9f8c7f6d-k2xyz    1/1     Running
```

---

# 24. Verify the Exact Digest in QA

Again:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-qa \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
.../order-service@sha256:9a7b123...
```

Now we have:

```text
ECR:
sha256:9a7b123...

DEV:
sha256:9a7b123...

QA:
sha256:9a7b123...
```

Excellent.

---

# 25. QA Testing

Now QA testing happens against the deployed application.

Typical tests:

```text
Health
API
Authentication
Database
Redis
Messaging
External integrations
Performance
Security
```

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

Expected:

```json
{
  "status": "UP"
}
```

---

# 26. Application-Level QA

We should not only test:

```text
Pod = Running
```

A Pod being `Running` does not prove the application works.

We should test:

```text
Pod
 ↓
Service
 ↓
Application
 ↓
Database
 ↓
Dependencies
```

For order-service:

```text
POST /api/v1/orders
GET /api/v1/orders/{id}
```

should work.

---

# 27. QA Promotion Gate

A realistic promotion gate can be:

```text
                    DEV
                     │
                     ▼
              Deployment successful
                     │
                     ▼
                Smoke tests
                     │
                     ▼
                Integration tests
                     │
                     ▼
               QA Promotion PR
                     │
                     ▼
                  Approval
                     │
                     ▼
                    QA
                     │
                     ▼
               QA test suite
```

Only then do we consider:

```text
QA → STAGING
```

---

# 28. What Changes Between DEV and QA?

This is an important interview and real-world question.

### Same

```text
Application image
Image digest
Helm chart
Application code
```

### Different

```text
Replicas
CPU/memory
Database endpoint
Redis endpoint
Queue names
Secrets
External API endpoints
Ingress hostname
Autoscaling thresholds
Resource policies
```

For example:

```text
DEV:
order-service-dev.example.internal

QA:
order-service-qa.example.internal
```

But both run:

```text
sha256:9a7b123...
```

---

# 29. What Must NOT Change During Promotion?

We should not change:

```text
Application source code
Docker image
Image digest
```

during:

```text
DEV → QA
```

If the image changes:

```text
DEV:
sha256:AAA

QA:
sha256:BBB
```

then QA is testing something different.

That defeats the purpose of artifact promotion.

---

# 30. Environment Variables

Suppose the application needs:

```text
DB_URL
REDIS_HOST
PAYMENT_API_URL
```

DEV might use:

```text
DB_URL=dev-db...
REDIS_HOST=dev-redis...
PAYMENT_API_URL=dev-payment...
```

QA:

```text
DB_URL=qa-db...
REDIS_HOST=qa-redis...
PAYMENT_API_URL=qa-payment...
```

The container image doesn't change.

The environment configuration changes.

---

# 31. Secrets

Secrets should not be copied between environments.

For example:

```text
DEV Secret
   ↓
DEV namespace

QA Secret
   ↓
QA namespace
```

The application gets the correct secret based on the environment.

Our architecture remains:

```text
AWS Secrets Manager
       │
       ▼
External Secrets Operator
       │
       ▼
Kubernetes Secret
       │
       ▼
Pod
```

---

# 32. Database Safety

This is where promotion becomes more complicated.

Suppose application version:

```text
8f31a2c
```

requires a new database column.

Do not assume:

```text
DEV database
   ↓
QA database
```

will automatically be identical.

Each environment has its own database.

A proper migration should be compatible with the deployment strategy.

For example:

```text
Migration 1:
ADD nullable column

Deploy new application

Later:
make column required
```

This is the **expand-and-contract** approach we discussed earlier.

---

# 33. What Happens If QA Deployment Fails?

Suppose GitOps contains:

```text
sha256:NEW
```

but QA Pods fail.

ArgoCD might show:

```text
Progressing
```

or:

```text
Degraded
```

Check:

```bash
kubectl get pods -n ecommerce-qa
```

Then:

```bash
kubectl describe pod <POD> -n ecommerce-qa
```

Then:

```bash
kubectl logs <POD> -n ecommerce-qa
```

And:

```bash
kubectl get events \
  -n ecommerce-qa \
  --sort-by=.lastTimestamp
```

---

# 34. Failed QA Promotion

If the new image is broken:

```text
QA
 │
 ▼
Deployment failure
 │
 ▼
Stop promotion
 │
 ▼
Do NOT promote to STAGING
```

This is critical.

The pipeline must not blindly continue.

---

# 35. Rollback QA

If the previous image was:

```text
sha256:OLD
```

and new image:

```text
sha256:NEW
```

we revert the GitOps change.

```text
QA
 ↓
Git revert
 ↓
ArgoCD
 ↓
previous image
```

No rebuild.

---

# 36. Promotion Chain

Our complete release flow now looks like:

```text
Developer
   │
   ▼
GitHub
   │
   ▼
CI
   │
   ▼
ECR
   │
   ▼
Image Digest
   │
   ▼
DEV GitOps
   │
   ▼
DEV
   │
   ▼
DEV validation
   │
   ▼
QA GitOps PR
   │
   ▼
QA
   │
   ▼
QA validation
   │
   ▼
STAGING GitOps PR
   │
   ▼
STAGING
   │
   ▼
Production approval
   │
   ▼
PROD GitOps PR
   │
   ▼
PROD
```

---

# 37. The Key Principle

Think of the pipeline as moving **one artifact** through environments.

```text
                  ┌─────────┐
                  │   ECR   │
                  └────┬────┘
                       │
                sha256:ABC
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
         DEV          QA         STAGING
          │            │            │
          └────────────┼────────────┘
                       │
                       ▼
                      PROD
```

Not:

```text
DEV → rebuild → QA → rebuild → PROD
```

---

# 38. Why GitOps Is Powerful Here

Without GitOps:

```text
CI
 ↓
kubectl
 ↓
EKS
```

You have less visibility into:

```text
Who changed it?
Why?
Which version?
When?
What was previous?
How do we rollback?
```

With GitOps:

```text
Git commit
   ↓
ArgoCD
   ↓
Kubernetes
```

Git provides the desired-state history.

---

# 39. Production Promotion Model

Our eventual production architecture will be:

```text
                    ┌──────────────┐
                    │   CI Build   │
                    └──────┬───────┘
                           │
                           ▼
                          ECR
                           │
                           ▼
                    Immutable Digest
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
                    approval/testing
                           │
                           ▼
                          PROD
```

The same digest travels through all stages.

---

# 40. Practical Exercise

For our actual project, implement this now for:

```text
order-service
```

### Step 1

Verify DEV:

```bash
kubectl get pods -n ecommerce-dev
```

### Step 2

Get DEV digest:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-dev \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

### Step 3

Compare it with:

```bash
yq '.image.digest' \
  environments/dev/order-service/values.yaml
```

### Step 4

Verify ECR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

### Step 5

Run promotion:

```bash
./scripts/promote-image.sh \
  order-service \
  dev \
  qa
```

### Step 6

Inspect:

```bash
git diff
```

### Step 7

Create:

```text
promotion/order-service-dev-to-qa
```

### Step 8

Create the PR.

### Step 9

Merge after validation.

### Step 10

Verify QA:

```bash
kubectl get pods -n ecommerce-qa
```

### Step 11

Verify the digest:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-qa \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

You should see the **same digest** as DEV.

---

# 41. Final Result

At the end of this chapter:

```text
                    ORDER-SERVICE
                         │
                         ▼
                     CI Build
                         │
                         ▼
                   ECR Image
                         │
                  sha256:ABC
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
            DEV                     QA
             │                       │
        sha256:ABC              sha256:ABC
             │                       │
             ▼                       ▼
        Validation              Validation
```

The key achievement is:

> **QA is running the exact container artifact that was already deployed to DEV.**

That is the foundation for safely continuing the promotion chain to **STAGING → PROD**.

### Next chapter

**`77-GitOps-Promotion-QA-to-STAGING.md`**

We will add the next promotion stage and introduce **release approval, promotion gates, automated validation, and how staging should differ from QA without changing the application artifact.**
