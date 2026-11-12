# 62-GitHub-Actions-CI.md

# GitHub Actions CI — Production-Grade CI Pipeline

This chapter implements the **real CI pipeline** for the `order-service`.

We now have:

```text
53 Repository Setup
54 Order Service
55 Local Docker
56 Terraform State
57 Terraform VPC
58 Terraform IAM
59 Terraform ECR
60 Terraform EKS
61 EKS Add-ons
```

Now we connect the application repository to the infrastructure.

---

# 1. CI/CD Architecture

Our architecture deliberately separates CI from CD.

```text
                         CI
              +----------------------+
              |    GitHub Actions    |
              +----------------------+
                        |
                        v
                  Source Code
                        |
                        v
                    Build/Test
                        |
                        v
                    SonarQube
                        |
                        v
                   Security Scan
                        |
                        v
                    Docker Build
                        |
                        v
                     Trivy
                        |
                        v
                       ECR
                        |
                        |
              =====================
                    CI Boundary
              =====================
                        |
                        v
                 GitOps Repository
                        |
                        v
                     ArgoCD
                        |
                        v
                       EKS
                        |
                        v
                 order-service Pods
```

The key rule is:

> **CI builds and publishes the artifact. CD deploys the artifact.**

---

# 2. What CI Does

Our GitHub Actions pipeline will perform:

```text
Checkout
   ↓
Java setup
   ↓
Maven dependency cache
   ↓
Compile
   ↓
Unit tests
   ↓
SonarQube
   ↓
Dependency/security scan
   ↓
Secret scan
   ↓
Docker build
   ↓
Trivy image scan
   ↓
AWS authentication via OIDC
   ↓
ECR login
   ↓
Push image
```

It does **not** deploy to EKS.

---

# 3. What CD Does

Later:

```text
ECR
 |
 v
GitOps Repository
 |
 v
ArgoCD
 |
 v
EKS
```

ArgoCD continuously reconciles the Kubernetes state.

This gives us:

```text
CI
=
Build artifact

CD
=
Deploy artifact
```

---

# 4. Repository Structure

Our `order-service` repository becomes:

```text
order-service/
│
├── src/
├── pom.xml
├── Dockerfile
├── .dockerignore
│
├── .github/
│   ├── CODEOWNERS
│   ├── dependabot.yml
│   │
│   └── workflows/
│       ├── ci.yaml
│       └── security.yaml
│
├── security/
│   └── ...
│
└── README.md
```

Create:

```bash
mkdir -p .github/workflows
```

---

# 5. GitHub Actions Runner

The pipeline executes on a runner.

For example:

```yaml
runs-on: ubuntu-latest
```

Architecture:

```text
GitHub
   |
   v
GitHub Actions
   |
   v
Runner
   |
   +--> Java
   +--> Maven
   +--> Docker
   +--> Trivy
   +--> AWS CLI
```

For this initial implementation we can use a GitHub-hosted runner.

Enterprise environments may use self-hosted or ephemeral runners where network isolation, private dependencies, custom tooling, or compliance requires them.

---

# 6. GitHub Actions Permissions

This is critical.

Our workflow should use:

```yaml
permissions:
  contents: read
  id-token: write
```

Why?

```text
contents: read
    |
    +--> Checkout source

id-token: write
    |
    +--> Request GitHub OIDC token
```

The OIDC token is then exchanged with AWS STS.

---

# 7. AWS Authentication

We do **not** use:

```yaml
AWS_ACCESS_KEY_ID: ...
AWS_SECRET_ACCESS_KEY: ...
```

Instead:

```text
GitHub Actions
       |
       v
OIDC Token
       |
       v
AWS STS
       |
       v
IAM Role
       |
       v
ECR
```

This was prepared in Chapter 58.

---

# 8. Required GitHub Secret?

For AWS OIDC:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

are **not required**.

The workflow needs the AWS role ARN:

```text
AWS_ROLE_ARN
```

It is not a secret by itself.

For example:

```yaml
role-to-assume: arn:aws:iam::123456789012:role/github-actions-order-service
```

For a reusable enterprise setup, the ARN can also be stored as a GitHub environment/repository variable.

---

# 9. Basic Workflow

Create:

```text
.github/workflows/ci.yaml
```

Start with:

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

jobs:

  build-test:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Build and test
        run: ./mvnw clean verify
```

This is the starting point.

---

# 10. Why `verify`?

Instead of only:

```bash
./mvnw test
```

we use:

```bash
./mvnw clean verify
```

Maven performs the lifecycle stages required to compile, test, package and verify the project according to the configured build.

---

# 11. Maven Wrapper

Our repository should contain:

```text
.mvn/
├── wrapper/
└── ...
```

and:

```text
mvnw
mvnw.cmd
```

Then CI executes:

```bash
./mvnw
```

instead of relying on whatever Maven version happens to be installed on the runner.

This improves build reproducibility.

---

# 12. Pull Request Flow

Developer creates:

```text
feature/order-validation
```

Then:

```text
feature/order-validation
          |
          v
       Pull Request
          |
          v
     GitHub Actions
          |
          +--> Build
          +--> Test
          +--> Security
          |
          v
       Review
          |
          v
     Merge to main
```

---

# 13. Why CI Runs on Pull Requests

We want to catch problems **before merge**.

Example:

```text
Developer
   |
   v
PR
   |
   +--> Compile failure
   |
   +--> Test failure
   |
   +--> Security failure
   |
   X
Merge blocked
```

If everything succeeds:

```text
CI PASS
   |
   v
Review
   |
   v
Merge
```

---

# 14. Main Branch CI

After merge:

```text
main
 |
 v
CI
 |
 +--> Test
 +--> Security
 +--> Docker
 +--> ECR
```

This creates the deployable artifact.

---

# 15. Image Tagging Strategy

We need a unique tag.

GitHub provides:

```text
GITHUB_SHA
```

Example:

```text
a91c2f7...
```

We can generate:

```text
order-service:a91c2f7
```

Workflow:

```yaml
- name: Set image tag
  id: image
  run: |
    echo "tag=${GITHUB_SHA::12}" >> "$GITHUB_OUTPUT"
```

Then:

```text
${{ steps.image.outputs.tag }}
```

contains:

```text
a91c2f7d1234
```

---

# 16. Why Use Git SHA?

Suppose:

```text
Commit A
a91c2f7
```

creates:

```text
order-service:a91c2f7
```

and:

```text
Commit B
f42a812
```

creates:

```text
order-service:f42a812
```

Now we can directly trace:

```text
Docker image
      ↓
Git SHA
      ↓
Git commit
      ↓
Pull Request
      ↓
Developer change
```

This is valuable during production incidents.

---

# 17. Docker Build Stage

Add:

```yaml
- name: Build Docker image
  run: |
    docker build \
      -t order-service:${{ steps.image.outputs.tag }} \
      .
```

The runner now has:

```text
order-service:<git-sha>
```

---

# 18. Trivy Installation

We can use the official Trivy action:

```yaml
- name: Run Trivy vulnerability scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: order-service:${{ steps.image.outputs.tag }}
    format: table
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: '1'
```

For production, action references should ideally be pinned to a specific release or immutable commit rather than blindly tracking a moving branch.

The same principle applies to third-party GitHub Actions generally.

---

# 19. Why Trivy Runs Before ECR Push

Our sequence is:

```text
Docker Build
     |
     v
Trivy
     |
     +---- FAIL ---> Stop
     |
     v
AWS Authentication
     |
     v
ECR Push
```

If the image contains unacceptable vulnerabilities:

```text
CI
 |
 X
ECR push doesn't happen
```

This is preferable to publishing known-bad artifacts.

---

# 20. Trivy Policy

Initial policy:

```text
HIGH
CRITICAL
```

with:

```text
exit-code: 1
```

Meaning:

```text
Vulnerability found
       |
       v
CI failure
```

However, a real enterprise vulnerability policy should define:

* severity
* exploitability
* fix availability
* exception process
* false positives
* application exposure
* compensating controls
* SLA for remediation

Don't simply suppress every vulnerability to make CI green.

---

# 21. SonarQube

SonarQube analyzes source code.

Architecture:

```text
GitHub Actions
      |
      v
Sonar Scanner
      |
      v
SonarQube
      |
      v
Quality Gate
```

SonarQube itself is a centralized platform service.

It is not normally installed inside:

```text
order-service Pod
```

---

# 22. Maven Sonar Configuration

Add the Sonar Maven plugin or use the current supported scanner integration for your SonarQube/SonarCloud setup.

Example:

```bash
./mvnw verify sonar:sonar
```

A typical command may look like:

```bash
./mvnw clean verify sonar:sonar \
  -Dsonar.projectKey=order-service \
  -Dsonar.host.url="${SONAR_HOST_URL}" \
  -Dsonar.token="${SONAR_TOKEN}"
```

The exact authentication/property names should match the SonarQube version and scanner configuration used by your organization.

---

# 23. SonarQube Secrets

Do not put:

```text
SONAR_TOKEN=actual-token
```

inside:

```text
application.yaml
```

or:

```text
pom.xml
```

Use GitHub Secrets/Environments.

Example:

```text
SONAR_HOST_URL
SONAR_TOKEN
```

The token should be stored as a secret.

---

# 24. SonarQube Quality Gate

Conceptually:

```text
Source
  |
  v
SonarQube
  |
  v
Quality Gate
  |
  +---- FAIL ---> PR blocked
  |
  +---- PASS ---> continue
```

Typical quality checks can include:

```text
bugs
vulnerabilities
code smells
coverage
duplication
security hotspots
```

The exact thresholds should be defined by the engineering/security team.

---

# 25. Dependency Scanning

Application dependencies can contain vulnerabilities.

For Java:

```text
pom.xml
   |
   +--> Spring Boot
   +--> PostgreSQL driver
   +--> libraries
```

We should scan them independently from the Docker image.

Possible tools include:

```text
OWASP Dependency-Check
Trivy filesystem/dependency scanning
GitHub Dependabot
Snyk
```

Our platform can use more than one layer, but avoid creating noisy duplicate gates without a reason.

---

# 26. Trivy Filesystem Scan

Example:

```yaml
- name: Trivy filesystem scan
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: fs
    scan-ref: .
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: '1'
```

Again, pin third-party actions in production.

---

# 27. Secret Scanning

We don't want this:

```text
application.yaml
    |
    +--> AWS_SECRET_ACCESS_KEY
```

or:

```text
config/
    |
    +--> production-password
```

in Git.

A secret scanner such as Gitleaks can detect likely secrets.

Example:

```yaml
- name: Secret scan
  uses: gitleaks/gitleaks-action@v2
```

Pin the action reference appropriately for production.

---

# 28. Security Pipeline

Conceptually:

```text
Source
 |
 +--> SonarQube
 |
 +--> Dependency Scan
 |
 +--> Secret Scan
 |
 +--> Docker Build
 |
 +--> Trivy Image Scan
 |
 v
ECR
```

Each stage catches a different class of problem.

---

# 29. AWS OIDC Configuration

Now we authenticate to AWS.

Use:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.AWS_ECR_ROLE_ARN }}
    aws-region: ap-south-1
```

The workflow receives temporary credentials.

There are no permanent AWS keys in the repository.

---

# 30. AWS Identity Test

Immediately after authentication:

```yaml
- name: Verify AWS identity
  run: |
    aws sts get-caller-identity
```

Example:

```text
Account: 123456789012
Arn: arn:aws:sts::123456789012:assumed-role/github-actions-order-service/...
```

This is an excellent troubleshooting step.

---

# 31. ECR Login

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

The action uses the temporary AWS credentials obtained through OIDC.

---

# 32. ECR Repository URI

We can construct:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

For maintainability, repository information can be stored as:

```text
AWS_ACCOUNT_ID
AWS_REGION
ECR_REPOSITORY
```

or retrieved from infrastructure outputs.

---

# 33. Push Image

```yaml
- name: Push Docker image
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    IMAGE_TAG: ${{ steps.image.outputs.tag }}

  run: |
    docker tag \
      order-service:${IMAGE_TAG} \
      ${REGISTRY}/order-service:${IMAGE_TAG}

    docker push \
      ${REGISTRY}/order-service:${IMAGE_TAG}
```

Now:

```text
GitHub Actions
      |
      v
ECR
      |
      v
order-service:<git-sha>
```

---

# 34. Complete Initial Workflow

A practical starting version:

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

  build-test-scan-push:
    name: Build, Test, Scan and Push
    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Build and test
        run: |
          ./mvnw clean verify

      - name: Set image tag
        id: image
        run: |
          echo "tag=${GITHUB_SHA::12}" >> "$GITHUB_OUTPUT"

      - name: Build Docker image
        run: |
          docker build \
            -t order-service:${{ steps.image.outputs.tag }} \
            .

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: order-service:${{ steps.image.outputs.tag }}
          format: table
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '1'

      - name: Configure AWS credentials
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ECR_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          aws sts get-caller-identity

      - name: Login to Amazon ECR
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Push image to ECR
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
        run: |
          docker tag \
            order-service:${IMAGE_TAG} \
            ${REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}

          docker push \
            ${REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
```

This is our baseline.

---

# 35. Important: PR Does Not Push to ECR

Notice:

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

is used for AWS authentication and ECR push.

Therefore:

```text
Pull Request
    |
    +--> Build
    +--> Test
    +--> Scan
    |
    X
    ECR Push
```

while:

```text
main
 |
 +--> Build
 +--> Test
 +--> Scan
 +--> ECR Push
```

This prevents every PR from becoming a published deployment artifact.

---

# 36. Why Not Build on PR and Rebuild on Main?

A more advanced pipeline can avoid unnecessary duplicate builds.

Our conceptual production principle is:

```text
Build once
   |
   v
Test
   |
   v
Scan
   |
   v
Publish immutable artifact
```

For highly controlled release workflows, the artifact itself can be promoted rather than rebuilt.

---

# 37. Image Digest

After push, retrieve the digest:

```yaml
- name: Get image digest
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  env:
    IMAGE_TAG: ${{ steps.image.outputs.tag }}
  run: |
    aws ecr describe-images \
      --repository-name "${ECR_REPOSITORY}" \
      --image-ids imageTag="${IMAGE_TAG}" \
      --region "${AWS_REGION}" \
      --query 'imageDetails[0].imageDigest' \
      --output text
```

Example:

```text
sha256:abc123...
```

This digest becomes important for GitOps.

---

# 38. Artifact Traceability

We want:

```text
Git Commit
    |
    v
Git SHA
    |
    v
Docker Tag
    |
    v
ECR Digest
    |
    v
GitOps Commit
    |
    v
ArgoCD Application
    |
    v
Kubernetes Pod
```

This creates an audit chain.

---

# 39. GitHub Environment Variables

A clean repository can use:

```text
Repository Variables
```

for non-secret configuration:

```text
AWS_REGION
ECR_REPOSITORY
AWS_ECR_ROLE_ARN
SONAR_HOST_URL
```

Secrets:

```text
SONAR_TOKEN
```

should be stored as secrets.

Never put actual credentials into:

```text
.github/workflows/ci.yaml
```

---

# 40. GitHub Environment Separation

Later we can use:

```text
Development
QA
Staging
Production
```

GitHub Environments can provide:

* environment variables
* secrets
* approval rules
* deployment protection

However, our production deployment itself is handled by ArgoCD/GitOps.

Therefore GitHub environments should not become a second competing CD system.

---

# 41. GitHub Action Security

Third-party actions are code.

This:

```yaml
uses: some-org/some-action@main
```

is risky because the branch can change.

Prefer:

```text
release tag
```

or, for stricter supply-chain control:

```text
immutable commit SHA
```

Example conceptually:

```yaml
uses: some-org/some-action@<immutable-commit>
```

The repository should periodically review and update pinned action versions.

---

# 42. Dependency Pinning

The same principle applies to:

```text
Maven
Docker base images
GitHub Actions
Terraform providers
Helm charts
Kubernetes add-ons
```

We want:

```text
Known version
      |
      v
Reviewed
      |
      v
Tested
      |
      v
Production
```

not:

```text
latest
```

everywhere.

---

# 43. Docker Base Image

Our Dockerfile currently uses:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
```

and:

```dockerfile
FROM eclipse-temurin:21-jre
```

For stricter supply-chain control, production can pin a specific base image version/digest after validating the chosen image.

Example concept:

```dockerfile
FROM eclipse-temurin:<approved-version>@sha256:<digest>
```

This makes the base artifact immutable.

---

# 44. Build Context

Our `.dockerignore` should exclude:

```text
.git
.github
target
.env
logs
.idea
.vscode
```

This reduces:

```text
build context
image leakage risk
build time
```

---

# 45. Docker Image Verification

After building:

```bash
docker images order-service
```

Then:

```bash
docker history order-service:<tag>
```

This can help inspect the resulting image.

Trivy then scans:

```text
OS packages
language dependencies
known vulnerabilities
```

depending on its configuration and database.

---

# 46. CI Failure Example

Suppose unit tests fail:

```text
[ERROR] Tests run: 15
[ERROR] Failures: 1
```

Then:

```text
Build
  |
  X
Test failure
  |
  X
Docker
  |
  X
ECR
```

No image is published.

---

# 47. Security Failure Example

Suppose Trivy detects a critical vulnerability:

```text
CRITICAL
```

Then:

```text
Docker Build
     |
     v
Trivy
     |
     X
CI FAILURE
```

No ECR push.

---

# 48. AWS Authentication Failure

Suppose:

```text
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Check:

```text
GitHub repository
      |
      v
OIDC token
      |
      v
IAM trust policy
```

Verify:

```text
aud
sub
repository
branch
IAM role
```

Our Chapter 58 trust policy restricts the repository/branch.

---

# 49. ECR Access Failure

Suppose AWS authentication succeeds:

```text
aws sts get-caller-identity
```

but push fails:

```text
AccessDeniedException
```

Then:

```text
OIDC
 |
 v
IAM role
 |
 v
AWS authentication SUCCESS
 |
 v
ECR authorization FAILURE
```

Check the role's ECR permissions.

Authentication and authorization are separate.

---

# 50. Wrong Repository

Error:

```text
repository does not exist
```

Check:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

Also verify:

```text
AWS account
AWS region
repository name
```

---

# 51. Wrong Region

Suppose ECR is:

```text
ap-south-1
```

but CI uses:

```text
us-east-1
```

Then login/push can fail or point to the wrong registry.

Always keep:

```yaml
env:
  AWS_REGION: ap-south-1
```

consistent with the environment.

---

# 52. Immutable Tag Failure

Suppose the tag already exists:

```text
order-service:a91c2f7
```

and someone attempts to push another image with the same tag.

Because our repository is:

```text
IMMUTABLE
```

the push is rejected.

Correct solution:

```text
new commit
   |
   v
new SHA
   |
   v
new image
```

Do not overwrite the old artifact.

---

# 53. CI Security Model

The final CI security chain is:

```text
                         GitHub
                            |
                            v
                       Pull Request
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
          Code Review    SonarQube    Secret Scan
                            |
                            v
                       Dependency Scan
                            |
                            v
                       Docker Build
                            |
                            v
                          Trivy
                            |
                            v
                       AWS OIDC
                            |
                            v
                           ECR
```

---

# 54. What We Have Achieved

We can now take:

```text
order-service
```

from:

```text
GitHub source
```

to:

```text
Docker image
```

to:

```text
ECR
```

without manually creating AWS access keys.

The flow is:

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    +--> Java/Maven
    +--> Tests
    +--> SonarQube
    +--> Security scans
    +--> Docker
    +--> Trivy
    |
    v
OIDC
    |
    v
AWS IAM
    |
    v
ECR
```

---

# 55. What We Have NOT Done Yet

The CI pipeline currently stops at:

```text
ECR
```

We intentionally have not implemented:

```text
ECR
 |
 X
kubectl apply
```

Instead:

```text
ECR
 |
 v
GitOps Repository
 |
 v
ArgoCD
 |
 v
EKS
```

will be implemented next.

---

# 56. Current Project Progress

```text
53 Repository Setup
        ↓
54 Order Service
        ↓
55 Local Docker
        ↓
56 Terraform State
        ↓
57 Terraform VPC
        ↓
58 Terraform IAM
        ↓
59 Terraform ECR
        ↓
60 Terraform EKS
        ↓
61 EKS Add-ons
        ↓
62 GitHub Actions CI
```

The application delivery path now reaches:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Tests
    ↓
Security
    ↓
Docker
    ↓
Trivy
    ↓
ECR
```

---

# 57. Next Chapter

## `63-Helm-Implementation.md`

Now we create the Kubernetes deployment package for `order-service`.

We will build:

```text
helm/order-service/
│
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-qa.yaml
├── values-prod.yaml
│
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    └── pdb.yaml
```

And we will implement the production Kubernetes concepts that matter:

```text
ECR image
     ↓
Deployment
     ↓
Pods
     ↓
Service
     ↓
ALB
     ↓
Health probes
     ↓
Resource requests/limits
     ↓
HPA
     ↓
PDB
     ↓
ServiceAccount
     ↓
Pod Identity
```

Then the next stage will connect that Helm chart to:

```text
GitOps Repository
       ↓
ArgoCD
       ↓
EKS
```

which finally completes the **CI → ECR → GitOps → ArgoCD → EKS** delivery chain.
