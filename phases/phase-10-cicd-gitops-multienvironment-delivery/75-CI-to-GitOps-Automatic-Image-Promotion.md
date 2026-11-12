# 75 — CI to GitOps Automatic Image Promotion

> **Goal:** Remove the remaining manual step from our deployment process.
>
> After this chapter, the flow will be:
>
> **Developer → GitHub → CI → ECR → GitOps PR → ArgoCD → EKS**
>
> We will implement this for **`order-service` first**, then reuse the same pattern for all 21 microservices.

---

## 1. What We Have Before This Chapter

We already have:

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
   ├── Trivy Scan
   ├── Docker Build
   │
   ▼
Amazon ECR
   │
   └── order-service:<git-sha>
```

We also have:

```text
GitOps Repository
        │
        ▼
environments/
└── dev/
    └── order-service/
        └── values.yaml
```

For example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

And ArgoCD watches the GitOps repository.

### The remaining manual step

Previously, someone had to manually change:

```yaml
tag: "old-sha"
digest: "old-digest"
```

to:

```yaml
tag: "new-sha"
digest: "new-digest"
```

Then create a Git commit.

We are going to automate that.

---

# 2. Final Architecture

After this chapter:

```text
Developer
    │
    │ git push
    ▼
GitHub
order-service
    │
    ▼
GitHub Actions
    │
    ├── Checkout
    ├── Build
    ├── Unit Tests
    ├── SonarQube
    ├── Secret Scan
    ├── Docker Build
    ├── Trivy
    │
    ▼
Amazon ECR
    │
    ├── Image tag
    └── Image digest
    │
    ▼
GitHub App
    │
    │ write GitOps repository
    ▼
GitOps Repository
    │
    ├── update image tag
    ├── update image digest
    ├── create branch
    └── create Pull Request
    │
    ▼
GitOps PR Review
    │
    ▼
Merge to main
    │
    ▼
ArgoCD
    │
    ▼
EKS
    │
    ▼
order-service Pods
```

The important principle is:

> **CI builds the artifact. GitOps controls deployment.**

---

# 3. Why CI Should Not Directly Deploy to EKS

A common beginner implementation is:

```text
GitHub Actions
      │
      ▼
kubectl apply
      │
      ▼
EKS
```

We are **not** doing that.

Instead:

```text
GitHub Actions
      │
      ▼
GitOps Repository
      │
      ▼
ArgoCD
      │
      ▼
EKS
```

Why?

Because Git becomes the deployment record.

For example:

```text
Git commit:
abc1234

Image:
order-service@sha256:123abc...

Environment:
DEV

Deployment:
2026-09-23 18:30
```

You can inspect Git history and answer:

> "Why is this image running in production?"

without searching through CI logs.

---

# 4. Very Important Security Boundary

GitHub Actions should **not** receive:

```text
EKS admin credentials
Kubernetes admin token
kubectl credentials
```

The CI job only needs:

```text
AWS OIDC
   │
   ▼
AWS IAM Role
   │
   ▼
ECR permissions
```

And separately:

```text
GitHub Actions
   │
   ▼
GitHub App
   │
   ▼
GitOps repository
```

So:

```text
CI
 ├── AWS access → ECR
 │
 └── GitHub App → GitOps repository
```

There is no:

```text
CI → EKS admin
```

---

# 5. How the Automatic Promotion Works

Suppose the developer pushes:

```text
commit: 8f31a2c
```

CI builds:

```text
order-service:8f31a2c
```

Trivy scans it.

Then CI pushes it to ECR.

ECR returns:

```text
sha256:9a7b123456789...
```

CI now knows:

```text
Image tag:
8f31a2c

Image digest:
sha256:9a7b123456789...
```

CI updates:

```text
gitops/environments/dev/order-service/values.yaml
```

from:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "old"
  digest: "old-digest"
```

to:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123456789..."
```

Then CI creates:

```text
automation/order-service/8f31a2c
```

and opens:

```text
PR:
ci: promote order-service 8f31a2c to dev
```

---

# 6. GitHub App vs Personal Access Token

There are two ways to allow CI to modify the GitOps repository.

### Learning setup

You could use:

```text
Fine-grained Personal Access Token
```

But we don't want to make that our production design.

### Production-oriented setup

Use:

```text
GitHub App
```

Architecture:

```text
GitHub Actions
      │
      ▼
GitHub App
      │
      ▼
GitOps repository
```

The App gets only the permissions it needs.

---

# 7. Create the GitHub App

At organization level, create a GitHub App.

Conceptually:

```text
Organization Settings
       │
       ▼
Developer Settings
       │
       ▼
GitHub Apps
       │
       ▼
New GitHub App
```

Example name:

```text
company-ecommerce-gitops-bot
```

---

# 8. GitHub App Permissions

Give the App only the permissions required to update the GitOps repository.

Repository permissions:

```text
Contents:
Read and write

Pull requests:
Read and write

Metadata:
Read-only
```

We do **not** need:

```text
Administration
Actions administration
Organization administration
Secrets administration
```

Install the App only on:

```text
company-ecommerce/gitops
```

Do not install it on every repository unless there is a genuine requirement.

---

# 9. Generate the GitHub App Private Key

GitHub provides a private key for the App.

You will receive something similar to:

```text
company-ecommerce-gitops-bot.private-key.pem
```

Keep this private.

Do not put it in:

```text
Git
GitOps repository
Docker image
Helm values
README
Kubernetes Secret committed to Git
```

---

# 10. Add GitHub Actions Secrets

Inside the `order-service` repository:

```text
Settings
   ↓
Secrets and variables
   ↓
Actions
```

Add:

```text
GITOPS_APP_ID
GITOPS_APP_PRIVATE_KEY
```

Example:

```text
GITOPS_APP_ID = 1234567
```

The private key contains multiple lines.

GitHub Actions Secrets handles that value as a secret.

---

# 11. GitOps Repository Structure

Our GitOps repository now looks like:

```text
gitops/
│
├── environments/
│   ├── dev/
│   │   └── order-service/
│   │       ├── app.yaml
│   │       └── values.yaml
│   │
│   ├── qa/
│   ├── staging/
│   └── prod/
│
├── applications/
├── argocd/
│
├── scripts/
│   └── update-image.sh
│
└── .github/
    └── workflows/
```

---

# 12. Create Image Update Script

Create:

```text
scripts/update-image.sh
```

Content:

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="$1"
ENVIRONMENT="$2"
IMAGE_TAG="$3"
IMAGE_DIGEST="$4"

FILE="environments/${ENVIRONMENT}/${SERVICE}/values.yaml"

if [[ ! -f "$FILE" ]]; then
    echo "ERROR: File not found: $FILE"
    exit 1
fi

echo "Updating:"
echo "Service : $SERVICE"
echo "Environment : $ENVIRONMENT"
echo "Tag : $IMAGE_TAG"
echo "Digest : $IMAGE_DIGEST"
echo "File : $FILE"

export IMAGE_TAG
export IMAGE_DIGEST

yq -i \
  '.image.tag = strenv(IMAGE_TAG) |
   .image.digest = strenv(IMAGE_DIGEST)' \
  "$FILE"

echo
echo "Updated values:"
cat "$FILE"
```

Make it executable:

```bash
chmod +x scripts/update-image.sh
```

---

# 13. Test the Script Locally

Suppose:

```text
SERVICE=order-service
ENVIRONMENT=dev
TAG=8f31a2c
DIGEST=sha256:9a7b123456
```

Run:

```bash
./scripts/update-image.sh \
  order-service \
  dev \
  8f31a2c \
  sha256:9a7b123456
```

Before:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "old"
  digest: "sha256:old"
```

After:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123456"
```

---

# 14. GitOps Values File

Our file remains:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123456"

replicaCount: 2

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

The CI system should modify only the image fields.

It should **not** rewrite:

```yaml
replicaCount
resources
service configuration
database configuration
secrets
HPA
PDB
```

unless those changes are intentionally part of another workflow.

---

# 15. GitHub Actions Workflow

Now we connect everything.

Create/update:

```text
.github/workflows/ci.yaml
```

A simplified production-oriented structure:

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
  GITOPS_REPOSITORY: company-ecommerce/gitops

concurrency:
  group: order-service-dev-promotion
  cancel-in-progress: false

jobs:

  build-test:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Run Tests
        run: |
          ./mvnw test
```

This is the first job.

It answers:

> "Does the application code work?"

---

# 16. Build and Push Job

Now create:

```yaml
  build-push:
    name: Build and Push Image
    if: github.event_name == 'push' &&
        github.ref == 'refs/heads/main'

    needs:
      - build-test

    runs-on: ubuntu-latest

    outputs:
      image: ${{ steps.image.outputs.image }}
      tag: ${{ steps.image.outputs.tag }}
      digest: ${{ steps.digest.outputs.digest }}

    steps:

      - name: Checkout
        uses: actions/checkout@v4
```

This job only runs after:

```text
build-test
```

succeeds.

---

# 17. Authenticate to AWS

Use the OIDC configuration from our IAM chapter:

```yaml
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
```

Then verify:

```yaml
      - name: Verify AWS Identity
        run: |
          aws sts get-caller-identity
```

Expected output identifies the IAM role assumed by GitHub Actions.

---

# 18. Login to ECR

```yaml
      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2
```

Now obtain the registry:

```yaml
      - name: Set Image Information
        id: image
        run: |
          REGISTRY="${{ steps.ecr.outputs.registry }}"
          IMAGE="${REGISTRY}/${ECR_REPOSITORY}"
          TAG="${GITHUB_SHA}"

          echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"
          echo "tag=${TAG}" >> "$GITHUB_OUTPUT"

          echo "Image: ${IMAGE}"
          echo "Tag: ${TAG}"
```

For example:

```text
Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Tag:
8f31a2c...
```

---

# 19. Build Docker Image

```yaml
      - name: Build Docker Image
        run: |
          docker build \
            -t "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}" \
            .
```

Now:

```text
Docker image
      │
      ▼
order-service:8f31a2c
```

exists on the CI runner.

---

# 20. Scan the Same Image

Run Trivy:

```yaml
      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"
          severity: CRITICAL,HIGH
          exit-code: "1"
```

Important:

```text
Build
  ↓
Trivy
  ↓
Push
```

If Trivy fails:

```text
Trivy
   ↓
FAIL
   ↓
No ECR push
   ↓
No GitOps update
   ↓
No deployment
```

That is exactly what we want.

For production, pin third-party Actions to reviewed commit SHAs rather than relying on moving branch references such as `@master`.

---

# 21. Push to ECR

```yaml
      - name: Push Image
        run: |
          docker push \
            "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"
```

Now:

```text
CI
 │
 ▼
ECR
 │
 └── order-service:8f31a2c
```

---

# 22. Get the ECR Digest

After pushing:

```yaml
      - name: Get Image Digest
        id: digest
        run: |

          DIGEST=$(aws ecr describe-images \
            --repository-name "${ECR_REPOSITORY}" \
            --image-ids imageTag="${{ steps.image.outputs.tag }}" \
            --region "${AWS_REGION}" \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          if [[ -z "$DIGEST" || "$DIGEST" == "None" ]]; then
            echo "ERROR: Could not determine image digest"
            exit 1
          fi

          echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"

          echo "Image digest:"
          echo "${DIGEST}"
```

Example:

```text
sha256:9a7b123456789abcdef...
```

Now CI knows the **exact image identity**.

---

# 23. Optional: Scan the Registry Image by Digest

For stronger verification, you can also scan the image after pushing it:

```bash
trivy image \
  "${IMAGE}@${DIGEST}"
```

Conceptually:

```text
Local image
    ↓
Trivy
    ↓
ECR push
    ↓
ECR digest
    ↓
Optional registry scan
```

This gives additional confidence that the artifact referenced by the deployment is the artifact that passed security checks.

---

# 24. Create GitHub App Token

Now comes the GitOps part.

The CI job needs permission to write to:

```text
company-ecommerce/gitops
```

Use the GitHub App.

Example:

```yaml
      - name: Create GitOps App Token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.GITOPS_APP_ID }}
          private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
          owner: company-ecommerce
          repositories: gitops
```

Again, pin the action to a reviewed commit SHA in a hardened production implementation.

The token is temporary.

That is preferable to putting a long-lived PAT into the workflow.

---

# 25. Checkout GitOps Repository

Now checkout the GitOps repository separately:

```yaml
      - name: Checkout GitOps Repository
        uses: actions/checkout@v4
        with:
          repository: company-ecommerce/gitops
          token: ${{ steps.app-token.outputs.token }}
          path: gitops
```

Workspace now looks approximately like:

```text
workspace/
│
├── order-service/
│   ├── src/
│   ├── pom.xml
│   ├── Dockerfile
│   └── helm/
│
└── gitops/
    ├── environments/
    ├── argocd/
    └── scripts/
```

---

# 26. Install yq

The CI job needs `yq` to safely modify YAML.

For example:

```yaml
      - name: Install yq
        uses: mikefarah/yq@v4
```

Again, in a hardened environment, pin the action/tool version or reviewed commit according to your organization's supply-chain policy.

---

# 27. Update GitOps Values

Now:

```yaml
      - name: Update GitOps Image
        env:
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
          IMAGE_DIGEST: ${{ steps.digest.outputs.digest }}
        run: |

          cd gitops

          ./scripts/update-image.sh \
            order-service \
            dev \
            "$IMAGE_TAG" \
            "$IMAGE_DIGEST"
```

The GitOps file changes automatically.

---

# 28. Example Result

Before:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "123abc"
  digest: "sha256:old..."
```

CI changes it to:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

---

# 29. Validate the GitOps Change

Before creating the PR, validate the resulting Helm deployment.

From the workflow:

```yaml
      - name: Validate Helm
        run: |

          helm lint \
            helm/order-service

          helm template order-service \
            helm/order-service \
            -f gitops/environments/dev/order-service/values.yaml \
            > /tmp/rendered.yaml

          echo "Helm rendering successful"
```

This catches problems such as:

```text
Invalid YAML
Invalid Helm values
Missing required values
Template rendering errors
```

before the GitOps PR is created.

---

# 30. Create Automation Branch

Now:

```yaml
      - name: Create GitOps Branch
        env:
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
        run: |

          cd gitops

          BRANCH="automation/order-service/${IMAGE_TAG}"

          git checkout -b "$BRANCH"

          echo "branch=${BRANCH}" >> "$GITHUB_ENV"
```

Example:

```text
automation/order-service/8f31a2c
```

---

# 31. Commit the Change

```yaml
      - name: Commit GitOps Change
        env:
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
        run: |

          cd gitops

          git config user.name "company-ecommerce-gitops-bot"
          git config user.email "gitops-bot@company-ecommerce.example"

          git add \
            environments/dev/order-service/values.yaml

          git commit \
            -m "ci: promote order-service ${IMAGE_TAG} to dev"
```

Now Git history contains:

```text
ci: promote order-service 8f31a2c to dev
```

---

# 32. Push the Branch

```yaml
      - name: Push GitOps Branch
        run: |

          cd gitops

          git push \
            origin \
            "$BRANCH"
```

Now GitHub contains:

```text
main
 │
 ├── ...
 │
 └── automation/order-service/8f31a2c
```

---

# 33. Create Pull Request

Using GitHub CLI:

```yaml
      - name: Create GitOps Pull Request
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
          IMAGE_DIGEST: ${{ steps.digest.outputs.digest }}
        run: |

          cd gitops

          gh pr create \
            --base main \
            --head "$BRANCH" \
            --title "ci: promote order-service ${IMAGE_TAG} to dev" \
            --body "$(cat <<EOF
          ## Order Service DEV Promotion

          ### Image

          Repository:
          \`123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service\`

          Tag:
          \`${IMAGE_TAG}\`

          Digest:
          \`${IMAGE_DIGEST}\`

          ### Source

          Commit:
          \`${GITHUB_SHA}\`

          ### Validation

          - Unit tests passed
          - Docker image built
          - Trivy scan passed
          - ECR push completed
          - Helm validation passed

          ### Deployment

          Target environment:
          \`dev\`

          ArgoCD will deploy this image after the GitOps change is merged.
          EOF
          )"
```

---

# 34. Complete Flow

Now our pipeline becomes:

```text
                    APPLICATION REPO
                         │
                         ▼
                   GitHub Actions
                         │
              ┌──────────┴──────────┐
              │                     │
           Test                  Security
              │                     │
              └──────────┬──────────┘
                         ▼
                    Docker Build
                         │
                         ▼
                      Trivy
                         │
                         ▼
                        ECR
                         │
                         ▼
                    Image Digest
                         │
                         ▼
                  GitHub App Token
                         │
                         ▼
                   GitOps Repository
                         │
                         ▼
                  Update values.yaml
                         │
                         ▼
                     Git Branch
                         │
                         ▼
                    GitOps PR
                         │
                         ▼
                      Review
                         │
                         ▼
                       Merge
                         │
                         ▼
                      ArgoCD
                         │
                         ▼
                        EKS
```

---

# 35. What Happens After the PR Is Merged?

Suppose the GitOps PR is merged.

GitOps now contains:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

ArgoCD detects the Git change.

Then:

```text
GitOps changed
      ↓
ArgoCD detects change
      ↓
Application becomes OutOfSync
      ↓
ArgoCD sync
      ↓
Deployment updated
      ↓
Kubernetes rolling update
      ↓
New Pod
      ↓
New image
```

Because we configured DEV for automated synchronization:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

the deployment proceeds after the GitOps merge.

---

# 36. Production Is Different

We should **not** automatically do:

```text
Developer merge
    ↓
Production
```

Instead:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

Each promotion should have its own controls.

For example:

```text
CI
 ↓
DEV GitOps PR
 ↓
DEV validation
 ↓
QA promotion
 ↓
QA validation
 ↓
STAGING promotion
 ↓
STAGING validation
 ↓
PROD approval
 ↓
PROD GitOps change
 ↓
ArgoCD
```

This is why we separated environments earlier.

---

# 37. CI Should Not Update All Environments

A common mistake would be:

```text
CI
 │
 ├── update DEV
 ├── update QA
 ├── update STAGING
 └── update PROD
```

We don't want that.

Instead, CI normally produces the artifact and may automatically promote it into the first environment according to the organization's release policy.

Later promotion is controlled:

```text
Same image
    │
    ├── DEV
    ├── QA
    ├── STAGING
    └── PROD
```

The artifact remains the same.

---

# 38. Why the Same Image Matters

Suppose CI builds:

```text
order-service:8f31a2c
```

and digest:

```text
sha256:9a7b123...
```

DEV uses:

```text
sha256:9a7b123...
```

QA uses:

```text
sha256:9a7b123...
```

STAGING uses:

```text
sha256:9a7b123...
```

PROD uses:

```text
sha256:9a7b123...
```

We don't rebuild.

Therefore:

```text
DEV = QA = STAGING = PROD
```

at the **container artifact level**.

Only environment configuration changes.

---

# 39. Race Condition Problem

Imagine two developers merge almost simultaneously:

```text
Developer A
commit A
   ↓
CI
   ↓
GitOps PR A
```

and:

```text
Developer B
commit B
   ↓
CI
   ↓
GitOps PR B
```

Both modify:

```text
environments/dev/order-service/values.yaml
```

This can create competing promotions.

That's why we added:

```yaml
concurrency:
  group: order-service-dev-promotion
  cancel-in-progress: false
```

This makes promotion jobs for this service/environment execute in an orderly fashion rather than running uncontrolled concurrent updates.

For a larger platform, promotion automation should also handle stale PRs and merge conflicts explicitly.

---

# 40. GitOps PR Is an Important Audit Point

The PR contains:

```text
Application:
order-service

Environment:
dev

Source commit:
8f31a2c

Image:
order-service:8f31a2c

Digest:
sha256:9a7b123...

CI:
passed

Trivy:
passed

Helm validation:
passed
```

A reviewer can inspect exactly what will change.

This is much safer than:

```bash
kubectl set image ...
```

executed manually from someone's laptop.

---

# 41. Rollback

Suppose:

```text
8f31a2c
```

causes a production problem.

GitOps history might be:

```text
Commit A
order-service:123abc

Commit B
order-service:456def

Commit C
order-service:8f31a2c
```

Rollback means reverting the GitOps change.

For example:

```text
8f31a2c
   ↓
previous known-good digest
```

ArgoCD sees the Git change and reconciles EKS.

No new build is required.

---

# 42. Important Security Rules

Never put these into GitOps:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
database passwords
API keys
GitHub tokens
GitHub App private keys
```

GitOps contains:

```text
image repository
image tag
image digest
replicas
resources
configuration references
Helm settings
```

Secrets come from our previously designed secrets architecture:

```text
AWS Secrets Manager
        ↓
External Secrets
        ↓
Kubernetes Secret
        ↓
Pod
```

---

# 43. Final `order-service` Deployment Chain

We now have a complete chain:

```text
1. Developer writes code
          ↓
2. Pull Request
          ↓
3. CI tests code
          ↓
4. Security scanning
          ↓
5. Docker image
          ↓
6. Trivy
          ↓
7. ECR
          ↓
8. ECR digest
          ↓
9. GitHub App
          ↓
10. GitOps values update
          ↓
11. GitOps Pull Request
          ↓
12. Review
          ↓
13. Merge
          ↓
14. ArgoCD detects Git change
          ↓
15. ArgoCD sync
          ↓
16. EKS Deployment
          ↓
17. Kubernetes rolling update
          ↓
18. New Pods
          ↓
19. Readiness checks
          ↓
20. Traffic reaches new Pods
```

---

# 44. The Most Important Mental Model

Remember these three repositories:

```text
order-service
     │
     │ owns
     ▼
Application code + Dockerfile + Helm chart
```

```text
infrastructure
     │
     │ owns
     ▼
AWS infrastructure + IAM + EKS + ECR + RDS
```

```text
gitops
     │
     │ owns
     ▼
Which version runs in each environment
```

And these three systems:

```text
GitHub Actions
     │
     └── Builds artifacts

ECR
     │
     └── Stores artifacts

ArgoCD
     │
     └── Deploys artifacts
```

That separation is the core of our production architecture.

---

# 45. Final Architecture After Chapter 75

```text
                         INTERNET
                            │
                            ▼
                    Route 53 / CloudFront
                            │
                            ▼
                           ALB
                            │
                            ▼
                         EKS
                            │
             ┌──────────────┴──────────────┐
             │                             │
       21 Microservices              Platform Services
             │                             │
             ▼                             ▼
           RDS                         ArgoCD
          Redis                       Prometheus
       SQS / SNS                       Grafana
        Secrets                       Fluent Bit
                                      Logstash
                                    Elasticsearch
                                       Kibana
                                     OTel/Jaeger


DEVELOPER
   │
   ▼
GitHub
   │
   ▼
GitHub Actions
   │
   ├── Test
   ├── SonarQube
   ├── Security scans
   ├── Docker build
   ├── Trivy
   │
   ▼
ECR
   │
   ▼
Image Digest
   │
   ▼
GitHub App
   │
   ▼
GitOps PR
   │
   ▼
GitOps main
   │
   ▼
ArgoCD
   │
   ▼
EKS
```

## Chapter 75 outcome

At this point, **`order-service` has a realistic end-to-end CI → ECR → GitOps → ArgoCD → EKS deployment path**, including immutable image digests and a reviewable GitOps promotion step.

**Next: `76-GitOps-Promotion-DEV-to-QA.md`** — we will take the exact same immutable image from DEV and build the practical **DEV → QA promotion workflow**, including what changes between environments, how QA approval works, and how we prevent a different image from being accidentally deployed.
