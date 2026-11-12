# 67 — Order Service CI → GitOps Automation

## 1. Objective

Until Chapter 66, we proved the complete deployment manually:

```text
Developer
   ↓
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

Now we automate the handoff between CI and GitOps.

The target workflow is:

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
GitHub Actions
    │
    ├── Build
    ├── Unit Tests
    ├── SonarQube
    ├── Dependency Scan
    ├── Secret Scan
    ├── Docker Build
    ├── Trivy
    │
    └── Push ECR
            │
            ▼
       Image Digest
            │
            ▼
      GitOps Promotion
            │
            ▼
       Promotion PR
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

The most important rule remains:

> **CI builds and publishes the artifact. GitOps decides which environment receives it. ArgoCD performs the deployment.**

---

# 2. Why We Don't Let CI Run `kubectl`

A common implementation is:

```text
GitHub Actions
      │
      ▼
kubectl apply
      │
      ▼
Production EKS
```

We are deliberately avoiding this.

Instead:

```text
GitHub Actions
      │
      ▼
ECR
      │
      ▼
GitOps
      │
      ▼
ArgoCD
      │
      ▼
EKS
```

This gives us a clean separation:

| Component      | Responsibility           |
| -------------- | ------------------------ |
| GitHub         | Source code              |
| GitHub Actions | CI/build/security        |
| ECR            | Immutable images         |
| GitOps         | Desired deployment state |
| ArgoCD         | Reconciliation           |
| EKS            | Runtime                  |

---

# 3. Repository Structure

Our application repository:

```text
order-service/
├── src/
├── pom.xml
├── Dockerfile
├── helm/
│   └── order-service/
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yaml
│       └── security.yaml
└── README.md
```

GitOps:

```text
gitops/
├── environments/
│   ├── dev/
│   │   └── order-service/
│   │       └── values.yaml
│   ├── qa/
│   ├── staging/
│   └── prod/
├── argocd/
├── scripts/
└── .github/
```

---

# 4. CI Trigger Model

We want two different behaviors.

## Pull Request

Run:

```text
Build
Test
SonarQube
Dependency Scan
Secret Scan
Docker Build
Trivy
```

but **do not push the production image or deploy**.

---

## Main Branch

After merge:

```text
Build
Test
Security
Docker
Trivy
Push ECR
Create GitOps promotion
```

Therefore:

```text
PR
 │
 └── validation only

main
 │
 └── build + publish + promotion
```

---

# 5. GitHub Actions Permissions

At the top of the workflow:

```yaml
permissions:
  contents: read
  id-token: write
```

Why?

```text
contents: read
```

allows checkout.

```text
id-token: write
```

allows GitHub Actions to request an OIDC token for AWS authentication.

We do **not** need:

```yaml
permissions:
  contents: write
```

for the application repository merely to build an image.

---

# 6. AWS Authentication

The architecture is:

```text
GitHub Actions
      │
      │ OIDC
      ▼
GitHub OIDC Provider
      │
      ▼
AWS STS
      │
      ▼
IAM Role
      │
      ▼
ECR
```

No:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

is stored in GitHub Actions.

---

# 7. CI IAM Role

The GitHub Actions role should have only the ECR permissions required for this service.

For example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/order-service"
    }
  ]
}
```

Do not give:

```text
AdministratorAccess
```

to the CI role.

---

# 8. Image Naming

Use:

```text
<account>.dkr.ecr.<region>.amazonaws.com/order-service:<git-sha>
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

This makes the image traceable to Git.

Avoid:

```text
order-service:latest
```

because:

```text
latest
```

does not tell us exactly which source revision produced the image.

---

# 9. Create CI Workflow

Create:

```text
.github/workflows/ci.yaml
```

A practical baseline:

```yaml
name: Order Service CI

on:
  pull_request:
    branches:
      - main

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

  test:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Run tests
        run: ./mvnw clean test

      - name: Package application
        run: ./mvnw clean package -DskipTests


  docker:
    name: Build Docker Image
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build \
            -t order-service:${GITHUB_SHA} \
            .


  security:
    name: Container Security
    needs: docker
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build \
            -t order-service:${GITHUB_SHA} \
            .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: order-service:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true


  push:
    name: Push Image to ECR
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs:
      - test
      - security

    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker build \
            -t ${REGISTRY}/${ECR_REPOSITORY}:${GITHUB_SHA} \
            .

      - name: Push image
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker push \
            ${REGISTRY}/${ECR_REPOSITORY}:${GITHUB_SHA}
```

For production, pin third-party Actions to reviewed commit SHAs rather than relying indefinitely on moving tags.

---

# 10. Important Improvement

The workflow above demonstrates the concept, but it builds the Docker image twice:

```text
docker job
security job
push job
```

That is not ideal.

A production pipeline should generally:

```text
Build image once
      ↓
Scan same image
      ↓
Push same image
```

rather than:

```text
Build
 ↓
Build again
 ↓
Build again
```

We will therefore combine the build, scan, and push lifecycle into one controlled job.

---

# 11. Improved CI Architecture

```text
Checkout
   ↓
Maven Test
   ↓
Docker Build
   ↓
Trivy Scan
   ↓
AWS Authentication
   ↓
ECR Push
   ↓
Get Digest
   ↓
GitOps Promotion
```

The same local Docker image is used throughout that job.

---

# 12. Production-Style Build Job

Replace the earlier Docker/push jobs with:

```yaml
  build-and-push:
    name: Build Scan and Push
    needs: test

    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        env:
          REGISTRY: ${{ steps.ecr.outputs.registry }}
        run: |
          docker build \
            -t ${REGISTRY}/${ECR_REPOSITORY}:${GITHUB_SHA} \
            .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true

      - name: Push image
        env:
          REGISTRY: ${{ steps.ecr.outputs.registry }}
        run: |
          docker push \
            ${REGISTRY}/${ECR_REPOSITORY}:${GITHUB_SHA}

      - name: Get image digest
        id: image
        env:
          AWS_REGION: ${{ env.AWS_REGION }}
        run: |
          DIGEST=$(aws ecr describe-images \
            --repository-name "${ECR_REPOSITORY}" \
            --image-ids imageTag="${GITHUB_SHA}" \
            --region "${AWS_REGION}" \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"

      - name: Print image
        run: |
          echo "Image:"
          echo "${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${GITHUB_SHA}"

          echo "Digest:"
          echo "${{ steps.image.outputs.digest }}"
```

---

# 13. Build Once, Promote Same Artifact

This produces:

```text
Git SHA
   │
   ▼
Docker image
   │
   ▼
Trivy
   │
   ▼
ECR
   │
   ▼
Digest
```

Example:

```text
Git:
8f31a2c

Image:
order-service:8f31a2c

Digest:
sha256:abc...
```

Now the deployment process can refer to this exact artifact.

---

# 14. Why the Digest Matters

A tag such as:

```text
8f31a2c
```

is useful.

A digest:

```text
sha256:abc123...
```

identifies the exact image content.

Therefore the strongest deployment trace is:

```text
Git commit
     ↓
Image tag
     ↓
Image digest
     ↓
GitOps
     ↓
ArgoCD
     ↓
EKS
```

Later we can make the Helm chart deploy:

```text
repository@sha256:<digest>
```

rather than relying only on the tag.

---

# 15. GitOps Automation

After ECR push, CI needs to update:

```text
gitops/environments/dev/order-service/values.yaml
```

The desired flow is:

```text
ECR push
   ↓
Get image digest
   ↓
Clone GitOps
   ↓
Update image
   ↓
Validate
   ↓
Create branch
   ↓
Create PR
```

---

# 16. Don't Give CI Kubernetes Credentials

Notice what we are **not** doing.

We are not adding:

```text
AWS EKS admin credentials
```

to GitHub.

CI only needs:

```text
ECR push
GitOps repository update
```

ArgoCD has the Kubernetes deployment responsibility.

This is a major security boundary.

---

# 17. GitOps Authentication

For a learning implementation, we can use a narrowly scoped GitHub token.

For a production enterprise implementation, a GitHub App is preferable.

Architecture:

```text
GitHub Actions
      │
      ▼
GitHub App
      │
      ▼
gitops repository
```

Permissions should be limited to what is required for:

```text
Contents
Pull Requests
Metadata
```

on the GitOps repository.

---

# 18. GitHub Secret

In:

```text
order-service
```

configure an appropriate credential, for example:

```text
GITOPS_TOKEN
```

Do not put it in:

```text
values.yaml
```

or:

```text
.github/workflows/*.yaml
```

as plaintext.

---

# 19. GitOps Update Script

We already created:

```text
gitops/scripts/update-image.sh
```

Use:

```bash
./scripts/update-image.sh \
  order-service \
  dev \
  8f31a2c
```

This modifies:

```yaml
image:
  tag: "8f31a2c"
```

---

# 20. Better GitOps Promotion Script

Inside the GitOps repository, we can create:

```text
scripts/promote-image.sh
```

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="${1:?Service is required}"
ENVIRONMENT="${2:?Environment is required}"
IMAGE_TAG="${3:?Image tag is required}"

VALUES_FILE="environments/${ENVIRONMENT}/${SERVICE}/values.yaml"

if [[ ! -f "${VALUES_FILE}" ]]; then
  echo "ERROR: ${VALUES_FILE} does not exist"
  exit 1
fi

echo "Before update:"
grep -A3 '^image:' "${VALUES_FILE}" || true

yq -i \
  ".image.tag = \"${IMAGE_TAG}\"" \
  "${VALUES_FILE}"

echo
echo "After update:"
grep -A3 '^image:' "${VALUES_FILE}"
```

Make executable:

```bash
chmod +x scripts/promote-image.sh
```

---

# 21. GitOps PR Branch

CI creates:

```text
promote/order-service-8f31a2c
```

Then:

```bash
git checkout -b promote/order-service-8f31a2c
```

Update:

```bash
./scripts/promote-image.sh \
  order-service \
  dev \
  8f31a2c
```

Validate:

```bash
git diff
```

Commit:

```bash
git add environments/dev/order-service/values.yaml

git commit \
  -m "deploy(order-service): promote 8f31a2c to dev"
```

Push:

```bash
git push origin promote/order-service-8f31a2c
```

---

# 22. Create Promotion PR

The CI workflow can create a GitHub PR.

Conceptually:

```text
GitHub Actions
      │
      ▼
gitops branch
      │
      ▼
Pull Request
      │
      ├── GitOps validation
      ├── review
      └── approval
      │
      ▼
gitops/main
```

For DEV, organizations may choose automatic merge after validation.

For production:

```text
automatic production merge
```

should not be assumed.

---

# 23. GitOps Promotion Workflow

Create an additional workflow in the application repository:

```text
.github/workflows/promote-dev.yaml
```

A simplified example:

```yaml
name: Promote to DEV

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: "Image tag to promote"
        required: true
        type: string

permissions:
  contents: read

jobs:
  promote:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout GitOps repository
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
        run: |
          git clone \
            https://x-access-token:${GITOPS_TOKEN}@github.com/company-ecommerce/gitops.git

      - name: Configure Git
        working-directory: gitops
        run: |
          git config user.name "company-ecommerce-bot"
          git config user.email "devops-bot@company.example"

      - name: Install yq
        run: |
          sudo wget -qO /usr/local/bin/yq \
            https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          sudo chmod +x /usr/local/bin/yq

      - name: Update DEV image
        working-directory: gitops
        env:
          IMAGE_TAG: ${{ inputs.image_tag }}
        run: |
          ./scripts/promote-image.sh \
            order-service \
            dev \
            "${IMAGE_TAG}"

      - name: Create branch
        working-directory: gitops
        env:
          IMAGE_TAG: ${{ inputs.image_tag }}
        run: |
          git checkout -b "promote/order-service-${IMAGE_TAG}"

      - name: Commit
        working-directory: gitops
        env:
          IMAGE_TAG: ${{ inputs.image_tag }}
        run: |
          git add environments/dev/order-service/values.yaml

          git commit \
            -m "deploy(order-service): promote ${IMAGE_TAG} to dev"

      - name: Push branch
        working-directory: gitops
        env:
          IMAGE_TAG: ${{ inputs.image_tag }}
        run: |
          git push origin \
            "promote/order-service-${IMAGE_TAG}"
```

This demonstrates the mechanism.

For a complete production workflow, use a GitHub App credential and the GitHub API/CLI to create the PR rather than exposing a broad token.

---

# 24. Automatic Promotion After ECR

Once the manual promotion workflow works, we can trigger it automatically after:

```text
build
+
test
+
security
+
ECR push
```

The pipeline becomes:

```text
main
 │
 ▼
Test
 │
 ▼
Build image
 │
 ▼
Trivy
 │
 ▼
ECR
 │
 ▼
GitOps PR
```

---

# 25. Why We Don't Automatically Promote to PROD

Imagine:

```text
Developer merges code
       ↓
CI passes
       ↓
ECR
       ↓
Automatically deploy PROD
```

That means every merged change becomes a production deployment.

Our platform intentionally separates:

```text
Build
```

from:

```text
Promotion
```

Therefore:

```text
main merge
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

This gives us controlled promotion.

---

# 26. Environment Promotion

Suppose the image is:

```text
8f31a2c
```

DEV promotion changes:

```yaml
environments/dev/order-service/values.yaml
```

QA promotion changes:

```yaml
environments/qa/order-service/values.yaml
```

STAGING:

```yaml
environments/staging/order-service/values.yaml
```

Production:

```yaml
environments/prod/order-service/values.yaml
```

The image remains:

```text
8f31a2c
```

---

# 27. Same Artifact Across Environments

This is the desired state:

```text
             ECR
              │
          8f31a2c
              │
      ┌───────┼────────┐
      │       │        │
      ▼       ▼        ▼
     DEV      QA     STAGING
                       │
                       ▼
                      PROD
```

Not:

```text
DEV   → build A
QA    → build B
STAGING → build C
PROD  → build D
```

---

# 28. GitOps Promotion Matrix

For one release:

| Environment | Image     |
| ----------- | --------- |
| DEV         | `8f31a2c` |
| QA          | `8f31a2c` |
| STAGING     | `8f31a2c` |
| PROD        | `8f31a2c` |

If QA has not approved:

| Environment | Image     |
| ----------- | --------- |
| DEV         | `8f31a2c` |
| QA          | `8f31a2c` |
| STAGING     | previous  |
| PROD        | previous  |

This allows independent environment promotion without rebuilding.

---

# 29. GitOps Validation Before PR

The promotion workflow should validate:

```text
YAML
 ↓
Helm
 ↓
Kubernetes manifests
```

At minimum:

```bash
helm lint
```

and:

```bash
helm template
```

For our multi-source design, the validation job needs access to:

```text
order-service
```

and:

```text
gitops
```

at the revisions that ArgoCD will use.

---

# 30. Helm Validation

Example:

```bash
helm lint \
  ../order-service/helm/order-service
```

Then:

```bash
helm template order-service \
  ../order-service/helm/order-service \
  -f environments/dev/order-service/values.yaml
```

If rendering succeeds:

```text
GitOps values
        +
Helm chart
        ↓
Valid manifests
```

---

# 31. Kubernetes Schema Validation

The next layer can use a Kubernetes manifest validator such as:

```text
kubeconform
```

Conceptually:

```bash
helm template order-service \
  ../order-service/helm/order-service \
  -f environments/dev/order-service/values.yaml \
  | kubeconform
```

This catches things such as:

```text
Invalid API version
Malformed Kubernetes structure
Invalid fields
Schema mismatch
```

---

# 32. Security Validation

GitOps validation can additionally scan:

```text
Helm templates
Kubernetes manifests
Configuration
```

with tools such as:

```text
Trivy config
Conftest
OPA
Kyverno policies
```

Example policy checks:

```text
No privileged containers
Run as non-root
No hostNetwork
No hostPID
Resource limits required
Approved registries only
No plaintext secrets
```

---

# 33. Production Promotion PR

The production PR should contain information such as:

```markdown
# Production Promotion

Service:
order-service

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c

Digest:
sha256:...

Source commit:
8f31a2c

Previous production version:
72c1abc

Validation:
- CI passed
- Unit tests passed
- Security scan passed
- DEV validated
- QA validated
- STAGING validated

Rollback:
Revert this GitOps commit.
```

This creates a useful release record.

---

# 34. Deployment Traceability

At the end, we should be able to trace:

```text
Production
    │
    ▼
ArgoCD sync
    │
    ▼
GitOps commit abc123
    │
    ▼
Image digest sha256:...
    │
    ▼
Image tag 8f31a2c
    │
    ▼
Git commit 8f31a2c
    │
    ▼
Pull Request #123
```

This is exactly the type of chain you want during:

```text
Incident
Audit
Rollback
Security investigation
Release investigation
```

---

# 35. Failed Security Scan

Suppose Trivy detects:

```text
CRITICAL vulnerability
```

The pipeline must stop:

```text
Docker Build
    ↓
Trivy
    ↓
CRITICAL
    ↓
FAIL
    X
ECR Push
```

Therefore:

```text
No vulnerable image promotion
```

according to the configured policy.

The exact exception process for unfixed vulnerabilities should be explicitly governed rather than simply bypassing the scan.

---

# 36. Failed Unit Tests

```text
Maven Test
    │
    ▼
FAIL
    │
    X
Docker build/push
```

No image should be promoted.

---

# 37. Failed GitOps Validation

Suppose:

```text
GitOps PR
    │
    ▼
Helm validation
    │
    ▼
FAIL
```

Then:

```text
GitOps main
      X
```

is not updated.

Therefore ArgoCD never sees the bad configuration.

This is another important safety layer.

---

# 38. ArgoCD Becomes the Deployment Gate

The complete chain becomes:

```text
Code quality gate
       ↓
Security gate
       ↓
Image gate
       ↓
GitOps gate
       ↓
ArgoCD gate
       ↓
Kubernetes
```

Each stage answers a different question:

```text
CI:
Can we build it?

Security:
Is it acceptable under policy?

ECR:
Do we have the immutable artifact?

GitOps:
Should this environment run it?

ArgoCD:
Can the cluster reconcile to it?

Kubernetes:
Can the workload actually run?
```

---

# 39. What Happens During a Developer Change?

Suppose a developer changes:

```java
orderService.createOrder()
```

Workflow:

```text
Developer
   │
   ▼
feature/order-validation
   │
   ▼
Pull Request
   │
   ▼
GitHub Actions
   │
   ├── Compile
   ├── Test
   ├── SonarQube
   ├── Dependency Scan
   ├── Secret Scan
   └── Docker/Trivy validation
   │
   ▼
PR approval
   │
   ▼
main
   │
   ▼
Build image
   │
   ▼
ECR
   │
   ▼
GitOps DEV PR
   │
   ▼
ArgoCD
   │
   ▼
EKS
```

---

# 40. No Direct Production Credentials in CI

The GitHub Actions environment should not contain:

```text
EKS_ADMIN_ROLE
KUBECONFIG
PROD_KUBECONFIG
```

for normal application CI.

Instead:

```text
CI
 │
 ├── OIDC → ECR
 │
 └── GitHub App → GitOps
```

And:

```text
ArgoCD
 │
 └── Kubernetes API → EKS
```

This is a much cleaner trust model.

---

# 41. What If GitOps Is Down?

Suppose GitHub is temporarily unavailable.

Existing workloads continue running in EKS.

Why?

Because:

```text
ArgoCD
```

and:

```text
Kubernetes
```

already have the current desired/live state.

The limitation is:

```text
No new GitOps changes
```

until repository access is restored.

This is different from application runtime availability.

---

# 42. What If ArgoCD Is Down?

Existing application Pods generally continue running.

But:

```text
Git → cluster reconciliation
```

stops until ArgoCD is restored.

Therefore ArgoCD is a critical platform component and should eventually have:

```text
HA
PDB
multiple replicas
resource requests
monitoring
backups/configuration in Git
```

This will become part of the platform hardening work.

---

# 43. What If ECR Is Temporarily Unavailable?

Existing Pods continue running if their images are already pulled.

A new Pod may fail if the node cannot pull the required image.

Therefore:

```text
ECR
```

is part of the deployment dependency chain, while already-running containers are not continuously downloading their image.

---

# 44. What If RDS Is Down?

Then:

```text
Pod
 │
 ▼
Spring Boot
 │
 X
RDS
```

The application may remain running but database-dependent requests can fail.

This is why:

```text
Application health
```

and:

```text
Dependency health
```

must be designed carefully.

We will later implement:

```text
timeouts
connection pools
retry policies
circuit breakers
alerts
```

rather than allowing database failures to cause retry storms.

---

# 45. Final CI/CD Architecture

Our implementation now looks like:

```text
                         GitHub
                            │
                            ▼
                    Pull Request
                            │
                            ▼
                    GitHub Actions
                            │
             ┌──────────────┼──────────────┐
             │              │              │
           Build          Test          Security
             │              │              │
             └──────────────┼──────────────┘
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
                       Image Digest
                            │
                            ▼
                       GitOps PR
                            │
                            ▼
                       GitOps Main
                            │
                            ▼
                         ArgoCD
                            │
                            ▼
                           EKS
                            │
                    ┌───────┴────────┐
                    │                │
                    ▼                ▼
                   ALB             Pods
                                     │
                                     ▼
                                    RDS
```

---

# 46. Current Implementation Status

| Area                         | Status          |
| ---------------------------- | --------------- |
| Source repository            | ✅               |
| Java application             | ✅               |
| Local Docker                 | ✅               |
| Terraform state              | ✅               |
| VPC                          | ✅               |
| IAM                          | ✅               |
| ECR                          | ✅               |
| EKS                          | ✅               |
| EKS add-ons                  | ✅               |
| GitHub Actions               | ✅               |
| Helm                         | ✅               |
| GitOps                       | ✅               |
| ArgoCD                       | ✅               |
| First EKS deployment         | ✅               |
| ALB path                     | ✅               |
| RDS integration path         | ✅               |
| GitOps automation design     | ✅               |
| Automatic promotion workflow | 🔄 Implementing |
| Multi-environment promotion  | Next            |
| 21-service rollout           | Later           |

---

# 47. The Production Pattern We Will Reuse

Once `order-service` is stable, every other service follows the same pattern:

```text
service repository
       ↓
CI
       ↓
ECR
       ↓
GitOps
       ↓
ArgoCD
       ↓
EKS
```

For example:

```text
payment-service
       ↓
payment-service ECR
       ↓
GitOps
       ↓
ArgoCD
       ↓
EKS
```

and:

```text
inventory-service
       ↓
inventory-service ECR
       ↓
GitOps
       ↓
ArgoCD
       ↓
EKS
```

The platform architecture is reused rather than independently reinvented for every microservice.

---

# 48. Next Chapter

## `68-Multi-Environment-GitOps-Promotion.md`

Next we will implement the complete:

```text
DEV → QA → STAGING → PROD
```

promotion model.

We will cover:

```text
DEV automatic promotion
        ↓
QA promotion PR
        ↓
QA validation
        ↓
STAGING promotion
        ↓
Integration testing
        ↓
Production approval
        ↓
PROD GitOps
        ↓
ArgoCD
```

including **how the same image digest moves through all four environments, environment-specific Helm values, approval gates, rollback, release tagging, and how to avoid accidentally deploying a different build to production**.
