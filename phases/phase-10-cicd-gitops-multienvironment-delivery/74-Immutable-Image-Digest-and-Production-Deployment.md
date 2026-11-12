# 74 — Immutable Image Digest & Production Deployment

Now we connect the pieces we already built:

```text
GitHub
   ↓
GitHub Actions
   ↓
Docker
   ↓
ECR
   ↓
Image Digest
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

The main goal of this chapter is:

> **Production should run the exact container image that CI built and tested.**

---

# 1. Why Do We Need Image Digests?

Until now we've used:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

This is useful because humans can understand:

```text
8f31a2c
```

is related to the Git commit.

But a Docker tag is still a **mutable reference** unless the registry prevents overwriting.

For production, we want to identify the exact image.

That's what the digest does.

Example:

```text
order-service@sha256:abc123...
```

The digest identifies the exact image manifest.

---

# 2. Tag vs Digest

Think of it this way:

```text
Tag
 ↓
"Give me the image currently associated with this name."
```

Digest:

```text
Digest
 ↓
"Give me exactly this image."
```

Example:

```text
Tag:

order-service:8f31a2c
```

Exact image:

```text
order-service@sha256:9a7b...
```

---

# 3. Our ECR Repository

We already created:

```text
ECR
└── order-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

CI pushes:

```text
order-service:8f31a2c
```

ECR stores the image and its digest.

---

# 4. Build the Image

From the `order-service` repository:

```bash
docker build \
  -t order-service:8f31a2c \
  .
```

Check:

```bash
docker images
```

You should see something like:

```text
REPOSITORY       TAG
order-service    8f31a2c
```

---

# 5. Authenticate to ECR

```bash
aws ecr get-login-password \
  --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

---

# 6. Tag for ECR

```bash
docker tag \
  order-service:8f31a2c \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

---

# 7. Push

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

You'll get output containing a digest similar to:

```text
digest: sha256:9a7b123...
```

That digest is important.

---

# 8. Get the Digest Directly from ECR

Run:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=8f31a2c \
  --region ap-south-1
```

The response contains information similar to:

```json
{
  "imageDetails": [
    {
      "imageDigest": "sha256:9a7b123...",
      "imageTags": [
        "8f31a2c"
      ]
    }
  ]
}
```

Therefore:

```text
Tag:
8f31a2c

Digest:
sha256:9a7b123...
```

---

# 9. Why Should CI Capture This?

Because CI knows exactly what it built.

The flow is:

```text
GitHub commit
     ↓
CI
     ↓
Docker build
     ↓
Docker push
     ↓
ECR digest
```

CI should then pass that exact digest to GitOps.

---

# 10. GitHub Actions

Our previous CI workflow already had roughly:

```text
Checkout
   ↓
Java setup
   ↓
Maven test
   ↓
SonarQube
   ↓
Trivy
   ↓
AWS OIDC
   ↓
ECR login
   ↓
Docker build
   ↓
Docker push
```

Now we improve it.

---

# 11. Example Production-Oriented Workflow

```yaml
name: Order Service CI

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: order-service

jobs:

  build-test:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Run tests
        run: ./mvnw test

  build-push:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    needs:
      - build-test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image variables
        id: image
        run: |
          REGISTRY="${{ steps.ecr.outputs.registry }}"
          IMAGE="${REGISTRY}/${ECR_REPOSITORY}"
          TAG="${GITHUB_SHA}"

          echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"
          echo "tag=${TAG}" >> "$GITHUB_OUTPUT"

      - name: Build image
        run: |
          docker build \
            -t "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}" \
            .

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"
          severity: CRITICAL,HIGH
          exit-code: "1"

      - name: Push image
        run: |
          docker push \
            "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"

      - name: Get image digest
        id: digest
        run: |
          DIGEST=$(aws ecr describe-images \
            --repository-name "${ECR_REPOSITORY}" \
            --image-ids imageTag="${{ steps.image.outputs.tag }}" \
            --region "${AWS_REGION}" \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"

          echo "Image digest:"
          echo "${DIGEST}"

      - name: Show immutable image
        run: |
          echo "${{ steps.image.outputs.image }}@${{ steps.digest.outputs.digest }}"
```

For production, third-party GitHub Actions should be pinned to reviewed commit SHAs rather than moving references such as `@master`; the example above keeps the workflow readable, but that hardening should be applied before treating it as a production standard.

---

# 12. What Happens in This Workflow?

Let's follow it slowly.

Git commit:

```text
8f31a2c
```

GitHub Actions sets:

```text
TAG=8f31a2c
```

Then:

```text
Docker image:

order-service:8f31a2c
```

CI scans it.

Then pushes it to ECR.

ECR gives:

```text
sha256:9a7b123...
```

So CI now knows:

```text
Git commit:
8f31a2c

Docker tag:
8f31a2c

Docker digest:
sha256:9a7b123...
```

---

# 13. The Immutable Image Reference

The complete reference is:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

This is the exact artifact we want production to run.

---

# 14. Update GitOps

Our GitOps values can contain:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

But our Helm chart needs to know how to use the digest.

---

# 15. Modify Helm Values

In:

```text
helm/order-service/values.yaml
```

use:

```yaml
image:
  repository: ""
  tag: ""
  digest: ""
  pullPolicy: IfNotPresent
```

---

# 16. Modify Deployment Template

Instead of always generating:

```text
repository:tag
```

we want:

```text
repository@digest
```

when a digest exists.

For example:

```yaml
image: >-
  {{ .Values.image.repository }}{{- if .Values.image.digest -}}
  @{{ .Values.image.digest }}
  {{- else -}}
  :{{ .Values.image.tag }}
  {{- end }}
```

Conceptually:

```text
digest exists?
       |
   +---+---+
   |       |
  YES      NO
   |       |
   v       v
@digest   :tag
```

---

# 17. Example

Values:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

Helm generates:

```yaml
image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

That's what Kubernetes uses.

---

# 18. Why Keep the Tag Too?

Because the tag is useful for humans and release tracking.

We can see:

```text
8f31a2c
```

and understand:

> This came from Git commit 8f31a2c.

The digest tells us:

> This is the exact image content.

So:

```text
Tag = human traceability
Digest = exact artifact identity
```

---

# 19. GitOps Promotion

Now DEV might contain:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

After testing, QA gets the **same values**:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

STAGING:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

PROD:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

Therefore:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

all use the **same image digest**.

---

# 20. This Is the Important Part

We are not saying:

```text
"Deploy version 8f31a2c."
```

only.

We are saying:

```text
"Deploy the exact image identified by
sha256:9a7b123..."
```

The Git commit and tag are additional traceability information.

---

# 21. What Happens in ArgoCD?

Suppose GitOps changes:

```yaml
digest: "sha256:9a7b123..."
```

ArgoCD detects:

```text
Git desired state
        ↓
new digest
```

It renders Helm:

```text
Helm
 ↓
Deployment
 ↓
image: repository@sha256:...
```

Then ArgoCD applies the change.

Kubernetes performs the rollout.

---

# 22. Kubernetes Deployment

Suppose currently:

```text
order-service Pod
    ↓
digest A
```

GitOps changes to:

```text
digest B
```

Deployment sees:

```text
Pod template changed
```

and performs the configured rolling update.

Our previous Helm configuration had:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

So Kubernetes attempts to:

```text
Start new Pod
      ↓
Readiness probe
      ↓
New Pod becomes Ready
      ↓
Terminate old Pod
```

subject to scheduling capacity and the other deployment constraints.

---

# 23. Verify the Exact Image

After deployment:

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

You may get something like:

```text
docker-pullable://123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

This is extremely useful.

You can compare:

```text
GitOps digest
      =
Running Pod imageID
```

If they match, you have strong evidence that the intended image is running.

---

# 24. Check Deployment Too

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

---

# 25. Complete Verification

You can now verify:

### Git

```text
8f31a2c
```

### ECR

```text
sha256:9a7b123...
```

### GitOps

```yaml
tag: 8f31a2c
digest: sha256:9a7b123...
```

### ArgoCD

```text
Synced
Healthy
```

### Kubernetes

```text
image@sha256:9a7b123...
```

That's the traceability chain.

---

# 26. What If Trivy Finds a Critical Vulnerability?

Suppose:

```text
Docker build
     ↓
Trivy
     ↓
CRITICAL vulnerability
```

Our workflow has:

```yaml
exit-code: "1"
```

Therefore CI fails.

The image should not be promoted through the normal release process.

Flow:

```text
Build
 ↓
Trivy
 ↓
FAIL
 ↓
No promotion
```

The developer/security team fixes the issue.

Then:

```text
new commit
 ↓
new image
 ↓
new digest
```

---

# 27. What If ECR Push Succeeds but GitOps Update Fails?

This can happen.

For example:

```text
CI
 ↓
Build successful
 ↓
Trivy successful
 ↓
ECR push successful
 ↓
GitOps PR creation fails
```

The image exists in ECR.

But it hasn't been deployed.

That's okay.

We have separated:

```text
Artifact creation
```

from:

```text
Deployment
```

The image can remain in ECR until it is intentionally promoted.

---

# 28. What If ArgoCD Fails?

Suppose:

```text
GitOps
 ↓
ArgoCD
 ↓
Deployment fails
```

The image remains in ECR.

We can troubleshoot Kubernetes without rebuilding the image.

If necessary:

```text
Git revert
 ↓
previous digest
 ↓
ArgoCD
 ↓
previous release
```

---

# 29. Rollback with Digest

Current:

```yaml
image:
  tag: "8f31a2c"
  digest: "sha256:NEW..."
```

Previous:

```yaml
image:
  tag: "7a21c9e"
  digest: "sha256:OLD..."
```

Rollback:

```diff
- tag: "8f31a2c"
- digest: "sha256:NEW..."
+ tag: "7a21c9e"
+ digest: "sha256:OLD..."
```

GitOps PR/revert:

```text
Git
 ↓
ArgoCD
 ↓
Kubernetes
 ↓
OLD DIGEST
```

No rebuild.

---

# 30. Security Benefit

Using immutable image references reduces ambiguity.

Instead of:

```text
order-service:latest
```

we have:

```text
order-service@sha256:9a7b123...
```

This gives us stronger artifact traceability.

It also works well with:

```text
ECR scanning
Trivy
SBOM
image signing
admission policies
audit trails
```

We will build those pieces later.

---

# 31. One Important Improvement to Our CI

Our current workflow uses:

```yaml
uses: aquasecurity/trivy-action@master
```

For an actual production pipeline, don't rely on a moving branch.

Instead, pin third-party actions to a reviewed commit SHA.

Conceptually:

```yaml
uses: aquasecurity/trivy-action@<reviewed-commit-sha>
```

Likewise review/pin the other third-party actions according to your organization's supply-chain policy.

The goal is:

```text
CI today
    =
CI tomorrow
```

rather than allowing an external moving reference to unexpectedly change pipeline behavior.

---

# 32. GitHub Actions → GitOps

Now comes an important architectural decision.

Should CI directly run:

```bash
kubectl apply
```

?

**No.**

Our architecture remains:

```text
GitHub Actions
     |
     | build/push/update GitOps
     v
GitOps
     |
     v
ArgoCD
     |
     v
EKS
```

Not:

```text
GitHub Actions
     |
     | kubectl apply
     v
EKS
```

---

# 33. Why?

Because GitOps gives us:

```text
audit history
review
rollback
desired-state management
drift detection
environment promotion
```

If CI directly modifies Kubernetes:

```text
CI
 ↓
kubectl
 ↓
Kubernetes
```

the deployment state can exist outside Git.

That weakens our desired-state model.

---

# 34. Complete CI/CD Architecture

Now our architecture is:

```text
                     DEVELOPER
                         |
                         v
                    GitHub Repo
                         |
                         v
                  GitHub Actions
                         |
          +--------------+--------------+
          |              |              |
        Tests         SonarQube       Trivy
          |              |              |
          +--------------+--------------+
                         |
                         v
                    Docker Build
                         |
                         v
                        ECR
                         |
                  image + digest
                         |
                         v
                   GitOps PR
                         |
                         v
                    ApplicationSet
                         |
                         v
                       ArgoCD
                         |
                         v
                        EKS
                         |
                         v
                   order-service
```

---

# 35. The Four Environment Promotion Model

The same digest moves through:

```text
sha256:9a7b123...
        |
        v
      DEV
        |
      TEST
        |
        v
       QA
        |
      TEST
        |
        v
    STAGING
        |
    APPROVAL
        |
        v
      PROD
```

The image is not rebuilt at every stage.

---

# 36. What We Have Now

At this point, our project has:

```text
✅ GitHub repositories
✅ Branch protection
✅ Application code
✅ Docker
✅ ECR
✅ Terraform
✅ EKS
✅ IAM/OIDC
✅ GitHub Actions
✅ Helm
✅ GitOps
✅ ArgoCD
✅ ApplicationSet
✅ DEV/QA/STAGING/PROD
✅ Image tags
✅ Image digests
✅ Rollback model
```

We are getting very close to a complete production deployment pipeline.

---

# 37. The Mental Model

Remember this:

```text
Git commit
    ↓
CI builds image
    ↓
ECR stores image
    ↓
ECR gives digest
    ↓
GitOps records digest
    ↓
ArgoCD reads GitOps
    ↓
Helm renders Deployment
    ↓
EKS pulls exact digest
    ↓
Pod runs exact image
```

The key sentence is:

> **CI creates the artifact. GitOps promotes the artifact. ArgoCD deploys the artifact. EKS runs the artifact.**

---

# 38. Practical Exercise

Before moving forward, implement this only for `order-service`.

### Step 1

Build:

```bash
docker build \
  -t order-service:8f31a2c \
  .
```

### Step 2

Push to ECR.

### Step 3

Get digest:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=8f31a2c \
  --region ap-south-1
```

### Step 4

Add the digest to:

```text
environments/dev/order-service/values.yaml
```

### Step 5

Update the Helm Deployment to support:

```text
repository@digest
```

### Step 6

Commit GitOps.

### Step 7

Allow ArgoCD to synchronize.

### Step 8

Verify:

```bash
kubectl get pod -n ecommerce-dev
```

### Step 9

Get the actual running image:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-dev \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

### Step 10

Compare:

```text
GitOps digest
      =
Pod imageID digest
```

Once those match, you have successfully implemented **immutable image deployment**.

---

# Next Chapter — `75-CI-to-GitOps-Automatic-Image-Promotion.md`

Next we remove the remaining manual step.

Currently:

```text
CI
 ↓
ECR
 ↓
YOU manually update GitOps
```

We will automate:

```text
CI
 ↓
ECR
 ↓
Get digest
 ↓
Update GitOps
 ↓
Create GitOps PR
 ↓
Review/merge
 ↓
ArgoCD
 ↓
EKS
```

Then you'll have a genuinely connected **CI → ECR → GitOps → ArgoCD → EKS** pipeline for `order-service`.
