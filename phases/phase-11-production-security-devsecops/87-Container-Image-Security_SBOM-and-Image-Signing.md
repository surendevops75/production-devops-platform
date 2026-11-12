# 87 — Container Image Security, SBOM and Image Signing

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Registry:** Amazon ECR
> **CI:** GitHub Actions
> **GitOps:** ArgoCD
> **Previous:** 86 — Production Kubernetes Security Hardening
> **Next:** 88 — Kubernetes Admission Control and Image Verification

---

# 1. Objective

In Chapter 86, we secured the Kubernetes runtime.

Now we secure the **container image supply chain**.

Our objective is:

```text
Source Code
    ↓
Dockerfile
    ↓
Base Image
    ↓
Dependencies
    ↓
Docker Build
    ↓
Vulnerability Scan
    ↓
SBOM
    ↓
ECR
    ↓
Image Digest
    ↓
Image Signature
    ↓
GitOps
    ↓
ArgoCD
    ↓
Admission Verification
    ↓
EKS
```

The production requirement is:

> **Only an approved, scanned, signed, immutable image should reach production.**

---

# 2. Why Container Image Security Matters

A Docker image contains much more than our application code.

For `order-service`:

```text
order-service
│
├── Java runtime
├── Spring Boot
├── Spring dependencies
├── PostgreSQL driver
├── Jackson
├── Linux packages
├── CA certificates
├── system libraries
└── application JAR
```

A vulnerability could exist in:

```text
Application code
Dependency
Base image
OS package
Runtime
```

For example:

```text
Spring Boot
    ↓
dependency
    ↓
vulnerable library
```

Our application may work perfectly while still containing a security vulnerability.

---

# 3. Image Security Layers

We will use multiple controls.

```text
                    Container Image
                          |
        +-----------------+------------------+
        |                 |                  |
   Base Image         Dependencies       App Code
        |                 |                  |
        +-----------------+------------------+
                          |
                     Vulnerability
                        Scanning
                          |
              +-----------+-----------+
              |                       |
            Trivy                   ECR
              |                    Scanning
              +-----------+-----------+
                          |
                         SBOM
                          |
                    Image Digest
                          |
                    Image Signing
                          |
                  Admission Policy
```

---

# 4. Start With the Dockerfile

Our current Dockerfile is:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

This already demonstrates an important security pattern:

```text
Build image
    ↓
Compile application
    ↓
Runtime image
    ↓
Only required JAR copied
```

---

# 5. Why Multi-Stage Builds?

Bad approach:

```dockerfile
FROM eclipse-temurin:21-jdk

COPY .
RUN ./mvnw package

CMD ["java", "-jar", "target/app.jar"]
```

The final image may contain:

```text
Maven
source code
build tools
compiler
temporary files
development dependencies
```

We don't need those at runtime.

Multi-stage:

```text
Builder
  |
  +-- JDK
  +-- Maven
  +-- source
  +-- build dependencies
  |
  v
JAR
  |
  v
Runtime
  |
  +-- JRE
  +-- JAR
```

Smaller runtime image generally means a smaller attack surface.

---

# 6. Build Context Security

Our `.dockerignore` should include:

```text
.git
.github
.gitignore
.idea
.vscode

target
*.log

.env
.env.*

README.md

node_modules
```

Especially:

```text
.env
.env.*
```

We never want:

```text
.env
```

accidentally copied into an image.

---

# 7. Never Put Secrets in Dockerfile

Never do this:

```dockerfile
ENV DB_PASSWORD="mypassword"
```

Never:

```dockerfile
ARG AWS_SECRET_ACCESS_KEY="..."
```

Never:

```dockerfile
COPY secrets.json /app/
```

Even if later removed in another Docker layer, secrets can potentially remain in image history/layers.

Use:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
Pod
```

instead.

---

# 8. Base Image Selection

Our application depends on:

```text
eclipse-temurin:21
```

But we should ask:

```text
Is this image:
    trusted?
    maintained?
    patched?
    minimal enough?
    compatible with our application?
```

Possible runtime choices depend on application requirements.

For Java:

```text
JDK
JRE
JRE-like runtime
distroless Java
```

Don't blindly choose the smallest image.

First verify:

```text
Application compatibility
TLS/CA requirements
Debugging requirements
Native libraries
Security scanning
Operational support
```

---

# 9. Avoid `latest`

Never:

```dockerfile
FROM eclipse-temurin:latest
```

Prefer a controlled version.

For example:

```dockerfile
FROM eclipse-temurin:21-jdk
```

and establish an organizational process for updating the base image.

For higher reproducibility, production build systems can also pin a base image by digest:

```dockerfile
FROM eclipse-temurin:21-jre@sha256:<digest>
```

The exact digest should come from the approved image release being used.

---

# 10. Image Tags vs Digests

Suppose:

```text
order-service:8f31a2c
```

currently points to:

```text
sha256:AAA
```

The tag identifies the image conveniently.

The digest identifies the exact image content.

Therefore:

```text
Tag:
order-service:8f31a2c

Digest:
order-service@sha256:AAA
```

For production deployment, our preferred identity is:

```text
repository@digest
```

---

# 11. Build Once

This is critical.

We do **not** want:

```text
DEV
 ↓
build image

QA
 ↓
build image again

STAGING
 ↓
build image again

PROD
 ↓
build image again
```

Instead:

```text
Source Commit
      ↓
CI Build
      ↓
ONE IMAGE
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

The same digest travels through all environments.

---

# 12. Example Artifact

Suppose Git commit:

```text
8f31a2c
```

produces:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

ECR gives:

```text
sha256:9a7b123...
```

The production artifact becomes:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

---

# 13. Trivy

We use Trivy in CI.

Its role is to detect security issues before promotion.

Example:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$IMAGE"
```

Meaning:

```text
HIGH or CRITICAL found
        ↓
exit code 1
        ↓
CI fails
```

---

# 14. Don't Treat Every Vulnerability Identically

Suppose Trivy reports:

```text
CRITICAL  0
HIGH      2
MEDIUM    7
LOW       15
```

Your organization needs a defined policy.

Example:

```text
CRITICAL:
    block

HIGH:
    block unless formally accepted

MEDIUM:
    track/remediate according to SLA

LOW:
    track according to policy
```

The exact threshold is an organizational security decision.

Do not simply use:

```text
ignore everything
```

to make CI green.

---

# 15. Trivy CI Example

A simplified GitHub Actions step:

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@<PINNED-COMMIT-SHA>
  with:
    image-ref: ${{ steps.image.outputs.image }}:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: "1"
    ignore-unfixed: true
```

Notice:

```text
<PINNED-COMMIT-SHA>
```

rather than:

```text
@master
```

For production CI, third-party actions should be pinned and periodically reviewed.

---

# 16. `ignore-unfixed`

This option deserves attention.

Suppose:

```text
Vulnerability exists
but
no upstream fix is available
```

You might choose:

```yaml
ignore-unfixed: true
```

But that doesn't mean:

> Vulnerability is safe.

It means:

> The pipeline is not automatically failing on vulnerabilities for which no fix is currently available.

Your security process still needs to track them.

---

# 17. ECR Scanning

We also enabled:

```hcl
image_scanning_configuration {
  scan_on_push = true
}
```

Therefore:

```text
CI
 ↓
Push image
 ↓
ECR
 ↓
ECR vulnerability scanning
```

This gives us another security control.

Trivy and ECR scanning should not be treated as identical controls.

---

# 18. Why Scan in CI if ECR Also Scans?

Because they provide different operational points.

CI:

```text
Developer
    ↓
Build
    ↓
Trivy
    ↓
FAIL BEFORE PUSH
```

ECR:

```text
Image
    ↓
ECR
    ↓
Registry-level security visibility
```

Using both can provide defense in depth.

---

# 19. Scan the Same Image You Deploy

This is extremely important.

Bad:

```text
Build image A
 ↓
Scan image A

Build image B
 ↓
Push image B

Deploy image B
```

We scanned one artifact and deployed another.

Correct:

```text
Build image
    ↓
Scan image
    ↓
Push SAME image
    ↓
Get digest
    ↓
Deploy SAME digest
```

---

# 20. Docker Image Digest

After pushing:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag="$GITHUB_SHA" \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Example:

```text
sha256:9a7b123...
```

Store this value as the artifact identity.

---

# 21. SBOM

SBOM:

> Software Bill of Materials

Think of it as:

```text
"What's inside this image?"
```

Example:

```text
order-service
│
├── Spring Boot
├── Spring Framework
├── Jackson
├── Hibernate
├── PostgreSQL JDBC
├── Logback
├── Java runtime
└── OS packages
```

---

# 22. Why SBOM Matters

Imagine a vulnerability is announced:

```text
CVE-XXXX-YYYY
```

Security team asks:

> Which of our production services use this dependency?

Without SBOM:

```text
Search hundreds of repositories
```

With SBOM:

```text
CVE
 ↓
dependency
 ↓
SBOM
 ↓
affected images
 ↓
affected services
```

This dramatically improves vulnerability response.

---

# 23. Generate SBOM With Trivy

Example:

```bash
trivy image \
  --format cyclonedx \
  --output sbom.json \
  "$IMAGE"
```

The output:

```text
sbom.json
```

can be uploaded as a CI artifact.

---

# 24. SBOM Should Be Associated With the Digest

Don't only store:

```text
order-service-sbom.json
```

because that doesn't tell us which image it belongs to.

Prefer metadata such as:

```text
Service:
order-service

Git commit:
8f31a2c

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Digest:
sha256:9a7b123...

SBOM:
sbom.json
```

Now we know exactly what was analyzed.

---

# 25. SBOM Lifecycle

A practical model:

```text
Build
 ↓
Generate SBOM
 ↓
Associate with digest
 ↓
Store artifact
 ↓
Retain according to policy
 ↓
Query during vulnerability response
```

For regulated environments, retention requirements may be longer than ordinary CI artifact retention.

---

# 26. Image Signing

Now we answer another question:

> How do we know the image came from our trusted build process?

This is where signing helps.

Conceptually:

```text
CI
 |
 | builds
 v
Image
 |
 | digest
 v
sha256:AAA
 |
 | sign
 v
Signature
```

---

# 27. Cosign

A commonly used tool for container signing is:

```text
Cosign
```

from the Sigstore ecosystem.

Conceptual flow:

```text
GitHub Actions
       |
       v
Build image
       |
       v
Push ECR
       |
       v
Get digest
       |
       v
Cosign sign
       |
       v
ECR
```

---

# 28. Why Sign the Digest?

Suppose:

```text
order-service:1.0.0
```

points to:

```text
sha256:AAA
```

The signature is associated with the artifact identity.

Conceptually:

```text
Image Digest
     +
Signature
     +
Identity
```

Now an admission control system can require:

```text
Image must have a valid trusted signature.
```

---

# 29. Keyless Signing

Modern Sigstore workflows can use identity-based/keyless signing models rather than distributing a long-lived private signing key to every CI runner.

Conceptually:

```text
GitHub Actions
      |
      v
OIDC identity
      |
      v
Signing workflow
      |
      v
Signature
```

This reduces the need to store a permanent private signing key as a conventional CI secret.

The exact production implementation should be pinned to the chosen Sigstore/Cosign workflow and organizational trust model.

---

# 30. Important: AWS OIDC vs Sigstore Identity

Don't confuse these.

We already have:

```text
GitHub
   |
   v
GitHub OIDC
   |
   v
AWS IAM Role
   |
   v
ECR
```

This is AWS authentication.

Image signing introduces another identity/trust flow:

```text
GitHub Actions
   |
   v
Signing identity
   |
   v
Image signature
```

They solve different problems.

---

# 31. Production Artifact Chain

Our artifact now contains:

```text
Source Commit
      |
      v
Docker Image
      |
      +--> Trivy result
      |
      +--> SBOM
      |
      +--> ECR digest
      |
      +--> Signature
      |
      v
GitOps
```

Example:

```text
Git:
8f31a2c

Image:
order-service:8f31a2c

Digest:
sha256:9a7b123...

SBOM:
sbom-9a7b123.json

Signature:
valid

Environment:
production
```

---

# 32. ECR Repository Policy

Our Terraform repository already uses:

```hcl
resource "aws_ecr_repository" "this" {
  name                 = var.repository_name
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }
}
```

The critical setting:

```text
IMMUTABLE
```

means a tag cannot simply be overwritten with another image.

---

# 33. Why Immutable Tags?

Suppose:

```text
order-service:8f31a2c
```

was originally:

```text
sha256:AAA
```

An attacker or mistake attempts:

```text
order-service:8f31a2c
        ↓
sha256:BBB
```

Immutable tags prevent that tag from being overwritten.

But remember:

> Digest pinning remains stronger for exact deployment identity.

---

# 34. ECR Permissions

CI needs permission to push.

The CI role should have only the required ECR actions.

For example:

```text
ecr:GetAuthorizationToken

ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
```

The ECR repository permissions should be scoped to the required repository where practical.

Not:

```text
AdministratorAccess
```

---

# 35. Production CI Pipeline

Our Order Service pipeline now becomes:

```text
Pull Request
     |
     +--> Checkout
     |
     +--> Maven Test
     |
     +--> SonarQube
     |
     +--> Dependency Scan
     |
     +--> Gitleaks
     |
     v
Merge
     |
     v
Build Docker Image
     |
     +--> Trivy
     |
     +--> Generate SBOM
     |
     v
Push ECR
     |
     v
Get Digest
     |
     v
Sign Image
     |
     v
GitOps PR
```

---

# 36. Practical GitHub Actions Pipeline

A simplified production-oriented workflow:

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

  test:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@<PINNED-COMMIT-SHA>

      - name: Setup Java
        uses: actions/setup-java@<PINNED-COMMIT-SHA>
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Run tests
        run: ./mvnw test

      - name: Gitleaks
        uses: gitleaks/gitleaks-action@<PINNED-COMMIT-SHA>


  build:
    if: github.event_name == 'push' &&
        github.ref == 'refs/heads/main'

    needs:
      - test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@<PINNED-COMMIT-SHA>

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@<PINNED-COMMIT-SHA>
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@<PINNED-COMMIT-SHA>

      - name: Set image
        id: image
        run: |
          REGISTRY="${{ steps.ecr.outputs.registry }}"
          IMAGE="${REGISTRY}/${ECR_REPOSITORY}"

          echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"
          echo "tag=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - name: Build image
        run: |
          docker build \
            -t "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}" \
            .

      - name: Scan image
        uses: aquasecurity/trivy-action@<PINNED-COMMIT-SHA>
        with:
          image-ref: "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"
          severity: CRITICAL,HIGH
          exit-code: "1"
          ignore-unfixed: true

      - name: Generate SBOM
        run: |
          trivy image \
            --format cyclonedx \
            --output sbom.json \
            "${{ steps.image.outputs.image }}:${{ steps.image.outputs.tag }}"

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

      - name: Upload SBOM
        uses: actions/upload-artifact@<PINNED-COMMIT-SHA>
        with:
          name: order-service-sbom-${{ github.sha }}
          path: sbom.json
```

This is the foundation.

---

# 37. Signing Stage

The signing stage should occur **after the image has been pushed and its immutable digest is known**.

Conceptually:

```text
Build
 ↓
Scan
 ↓
SBOM
 ↓
Push
 ↓
Digest
 ↓
Sign digest
```

Not:

```text
Build
 ↓
Sign tag
 ↓
Push
```

The first model gives us a stronger artifact identity.

---

# 38. Verify the Image

Before deployment, we want:

```text
Image exists
        +
Digest matches
        +
Signature valid
        +
Signer trusted
```

Conceptually:

```bash
cosign verify \
  <IMAGE>@<DIGEST>
```

The exact production verification arguments depend on whether your organization uses:

```text
key-based signing
```

or:

```text
keyless identity-based signing
```

and what identity/issuer is trusted.

---

# 39. Why Verification Happens at Admission

CI verification alone is not enough.

Imagine:

```text
CI
 ↓
signed image
 ↓
ECR
```

Later someone changes the Kubernetes manifest manually.

We want Kubernetes admission to say:

```text
Is this image signed?
        |
       YES
        |
        v
Who signed it?
        |
        v
Is that signer trusted?
        |
       YES
        |
        v
ALLOW
```

Otherwise:

```text
unsigned image
     ↓
production
```

could still happen through another deployment path.

---

# 40. Admission Security

The final architecture becomes:

```text
GitOps
   |
   v
ArgoCD
   |
   v
Kubernetes API
   |
   v
Admission Policy
   |
   +--> approved registry?
   |
   +--> immutable image?
   |
   +--> signed?
   |
   +--> trusted signer?
   |
   +--> non-root?
   |
   +--> privileged=false?
   |
   v
Pod accepted
```

This is the subject of the next chapter in detail.

---

# 41. Image Promotion

Remember our environment flow:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

We don't rebuild.

Example:

```text
Digest:
sha256:9a7b123...
```

DEV:

```text
order-service@sha256:9a7b123
```

QA:

```text
order-service@sha256:9a7b123
```

STAGING:

```text
order-service@sha256:9a7b123
```

PROD:

```text
order-service@sha256:9a7b123
```

Same artifact.

---

# 42. What If a Vulnerability Is Discovered Later?

Suppose production has:

```text
order-service@sha256:AAA
```

A new vulnerability is announced.

SBOM tells us:

```text
sha256:AAA
   |
   +--> vulnerable dependency
```

Security team can identify:

```text
Affected image
Affected service
Affected environment
Affected dependency
```

Then:

```text
Update dependency/base image
        ↓
New Git commit
        ↓
New image
        ↓
New digest
        ↓
New SBOM
        ↓
New signature
        ↓
Normal promotion
```

We don't modify the existing image.

---

# 43. Never Mutate Production Images

If:

```text
sha256:AAA
```

is already deployed, don't attempt to "fix" that image in place.

Instead:

```text
Old:
sha256:AAA

New:
sha256:BBB
```

Then:

```text
Git
 ↓
CI
 ↓
Scan
 ↓
SBOM
 ↓
Sign
 ↓
ECR
 ↓
GitOps
 ↓
ArgoCD
```

This preserves auditability.

---

# 44. Emergency Vulnerability Response

Suppose a critical vulnerability affects the current production image.

The emergency flow is:

```text
CVE discovered
      |
      v
Identify affected images using SBOM
      |
      v
Assess exploitability
      |
      v
Build patched image
      |
      v
Trivy/security checks
      |
      v
New digest
      |
      v
Sign
      |
      v
Emergency Change Request
      |
      v
GitOps production update
      |
      v
ArgoCD
      |
      v
Production
```

This connects directly to our Jira Change Management chapter.

---

# 45. Security Gates

For our project, the image pipeline should have explicit gates.

```text
Source
  |
  v
Tests
  |
  +-- FAIL → STOP
  |
  v
SonarQube
  |
  +-- FAIL → STOP/REVIEW
  |
  v
Gitleaks
  |
  +-- SECRET → STOP
  |
  v
Docker Build
  |
  +-- FAIL → STOP
  |
  v
Trivy
  |
  +-- Policy violation → STOP
  |
  v
SBOM
  |
  +-- FAIL → STOP
  |
  v
ECR
  |
  v
Sign
  |
  +-- FAIL → STOP
  |
  v
GitOps
```

---

# 46. What Happens If Trivy Fails?

Example:

```text
CRITICAL: 1
HIGH: 3
```

Pipeline:

```text
Trivy
  |
  v
exit code 1
  |
  v
GitHub Actions FAILED
```

The image should not proceed to the production promotion path.

The developer/security team investigates:

```text
Which package?
Which CVE?
Is a fixed version available?
Can base image be updated?
Can dependency be upgraded?
Is there a documented exception?
```

---

# 47. Vulnerability Exception Process

Sometimes an organization cannot immediately fix a vulnerability.

Don't solve this with:

```bash
trivy --ignore-all
```

Instead have an explicit exception process.

Example:

```text
Vulnerability:
CVE-XXXX-YYYY

Severity:
HIGH

Affected service:
order-service

Reason:
No upstream fix available

Risk assessment:
Documented

Compensating controls:
Network restriction
No affected feature exposed

Owner:
Security Team

Expiry:
2026-10-15
```

The exception should expire and be reviewed.

---

# 48. Production Image Security Checklist

## Dockerfile

```text
[ ] Multi-stage build
[ ] Minimal runtime image
[ ] Approved base image
[ ] Supported runtime
[ ] No secrets
[ ] Non-root user
[ ] .dockerignore
[ ] No unnecessary packages
```

## Scanning

```text
[ ] Trivy
[ ] ECR scanning
[ ] Dependency scanning
[ ] OS package scanning
[ ] SBOM
[ ] Defined severity policy
[ ] Vulnerability exceptions documented
```

## Artifact

```text
[ ] Immutable tag
[ ] Digest captured
[ ] SBOM associated with digest
[ ] Image signed
[ ] Signature verified
```

## Deployment

```text
[ ] Approved ECR repository
[ ] Digest pinned
[ ] Admission verification
[ ] GitOps only
[ ] No manual production image changes
```

---

# 49. Complete Supply-Chain Architecture

Our production architecture now looks like this:

```text
                    DEVELOPER
                        |
                        v
                    GitHub PR
                        |
          +-------------+-------------+
          |             |             |
        Tests       SonarQube      Gitleaks
          |             |             |
          +-------------+-------------+
                        |
                     MERGE
                        |
                        v
                 Docker Build
                        |
              +---------+---------+
              |                   |
            Trivy               SBOM
              |                   |
              +---------+---------+
                        |
                        v
                      ECR
                        |
                  Scan on Push
                        |
                  Immutable Tag
                        |
                     Digest
                        |
                     Sign
                        |
                        v
                    GitOps PR
                        |
                        v
                     ArgoCD
                        |
                        v
              Admission Controller
                        |
              +---------+---------+
              |                   |
         Signature valid?    Security policy?
              |                   |
              +---------+---------+
                        |
                       YES
                        |
                        v
                      EKS
                        |
              +---------+---------+
              |                   |
        SecurityContext     NetworkPolicy
              |                   |
              +---------+---------+
                        |
                   Pod Identity
                        |
                     IAM Role
                        |
                AWS Resources
```

---

# 50. Final Production Rule

The deployment should never be:

```text
Developer
   ↓
docker build
   ↓
docker push
   ↓
kubectl apply
```

Instead:

```text
Developer
   ↓
GitHub
   ↓
CI security gates
   ↓
Trivy
   ↓
SBOM
   ↓
ECR
   ↓
Digest
   ↓
Signature
   ↓
GitOps
   ↓
ArgoCD
   ↓
Admission verification
   ↓
EKS
```

That gives us **traceability + integrity + vulnerability visibility + runtime enforcement**.

---

# 51. Where We Are Now

Our project security architecture has now reached:

```text
85  Production Networking
86  Kubernetes Security Hardening
87  Container Image Security
```

The next missing enforcement layer is important.

We have learned how to **sign** an image.

Now we need to make Kubernetes actually **reject an unsigned or untrusted image**.

---

# 52. Next Chapter

## 88 — Kubernetes Admission Control and Image Verification

We will implement the runtime enforcement layer:

```text
Developer
    ↓
CI
    ↓
Trivy
    ↓
SBOM
    ↓
ECR
    ↓
Cosign Signature
    ↓
GitOps
    ↓
ArgoCD
    ↓
Kubernetes API
    ↓
Admission Controller
    ↓
┌───────────────────────────────┐
│ Is registry approved?         │
│ Is image signed?              │
│ Is signer trusted?            │
│ Is image digest pinned?       │
│ Is privileged=false?          │
│ Is runAsNonRoot=true?         │
│ Are resources defined?        │
└───────────────────────────────┘
    ↓
 ALLOW / REJECT
    ↓
   EKS
```

We will make this practical for the **Order Service first**, then show how the same policy automatically protects all **21 microservices**.
