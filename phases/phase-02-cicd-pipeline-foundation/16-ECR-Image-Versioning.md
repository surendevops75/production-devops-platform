# 16-ECR-Image-Versioning.md

# ECR Image Versioning & Production Container Lifecycle

In the previous chapters, we built the CI security chain:

```text
Source
  ↓
Build
  ↓
Test
  ↓
SonarQube
  ↓
Gitleaks
  ↓
Trivy
  ↓
Docker Image
  ↓
ECR
```

Now we need to answer a critical production question:

> **Once CI builds a container image, how do we identify, store, promote, deploy, and roll back that exact image safely?**

For our enterprise e-commerce platform, Amazon ECR is the central private container registry.

---

# 1. Production Architecture

Our production flow is:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
CI Pipeline
    │
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Gitleaks
    ├── Trivy
    ├── SBOM
    └── Image signing
    │
    ▼
Amazon ECR
    │
    │ immutable image
    ▼
GitOps Repository
    │
    ▼
ArgoCD
    │
    ▼
Amazon EKS
    │
    └── order-service
```

ECR is **not the deployment system**.

It is the **artifact registry**.

---

# 2. Why Do We Need ECR?

Without a container registry:

```text
Developer laptop
      ↓
Docker image
      ↓
???
```

Kubernetes nodes need somewhere from which they can pull the image.

With ECR:

```text
Docker Build
     ↓
ECR
     ↓
EKS nodes
     ↓
Container runtime
     ↓
Pod
```

For AWS workloads, ECR integrates naturally with EKS and IAM.

---

# 3. One Repository Per Microservice

Our platform has 21 microservices.

Therefore we create:

```text
ECR
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

This gives us clear ownership and lifecycle management.

---

# 4. Production Naming

We should establish a consistent naming convention.

For example:

```text
company-ecommerce/order-service
company-ecommerce/payment-service
company-ecommerce/catalog-service
```

or, depending on the organization's AWS account/ECR naming standard:

```text
order-service
payment-service
catalog-service
```

The important thing is consistency.

For our project:

```text
ECR repository = service name
```

Example:

```text
order-service
```

---

# 5. Image Naming

An ECR image reference looks like:

```text
<account>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:abc123
```

The pieces are:

```text
123456789012
       │
       └── AWS Account

ap-south-1
       │
       └── AWS Region

order-service
       │
       └── ECR repository

abc123
       │
       └── image tag
```

---

# 6. Never Use `latest` for Production

Avoid:

```text
order-service:latest
```

Why?

Suppose today:

```text
latest → version A
```

Tomorrow:

```text
latest → version B
```

Then the meaning of:

```text
latest
```

has changed.

This creates problems with:

* rollback
* auditing
* debugging
* reproducibility
* deployment tracking

Instead:

```text
order-service:<immutable-version>
```

---

# 7. Git SHA as Image Tag

A strong production approach is:

```text
Git commit
     ↓
SHA
     ↓
Docker image tag
```

Example:

```text
Git commit:
8f3a21c9...
```

Image:

```text
order-service:8f3a21c9
```

Now we can immediately determine:

> Which source code produced this image?

---

# 8. Why Git SHA Is Powerful

Imagine production is running:

```text
order-service:7c92ab4
```

We can search Git:

```bash
git show 7c92ab4
```

and identify the exact source revision.

The chain becomes:

```text
Git commit
     ↓
CI build
     ↓
Docker image
     ↓
ECR
     ↓
GitOps
     ↓
EKS
```

This is excellent for auditing.

---

# 9. Semantic Versioning

We can also use semantic versions:

```text
1.0.0
1.1.0
1.1.1
2.0.0
```

For example:

```text
order-service:1.4.2
```

Semantic versioning communicates release meaning.

Generally:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
1.4.2
│ │ │
│ │ └── patch
│ └──── minor
└────── major
```

---

# 10. Git SHA vs Semantic Version

They solve slightly different problems.

| Tag              | Purpose                          |
| ---------------- | -------------------------------- |
| Git SHA          | Exact source/build identity      |
| Semantic version | Human-friendly release identity  |
| Digest           | Immutable image content identity |

A mature system can use both:

```text
order-service:1.4.2
order-service:8f3a21c9
```

Both point to the same image digest.

---

# 11. Image Digest

The strongest identity is:

```text
sha256:<digest>
```

For example:

```text
order-service@sha256:abcdef123456...
```

A digest identifies the exact image contents.

Think of:

```text
Tag
 ↓
Human-friendly label
```

while:

```text
Digest
 ↓
Exact immutable content
```

---

# 12. Recommended Production Model

For our project:

```text
Git commit
     │
     ▼
Image
     │
     ├── Git SHA tag
     ├── Release tag
     └── Digest
```

Example:

```text
order-service:8f3a21c9
order-service:1.4.2
```

both resolve to:

```text
sha256:abcdef...
```

The GitOps deployment can eventually pin the digest:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abcdef...
```

---

# 13. Why Digest-Based Deployment Is Powerful

Imagine someone accidentally moves a tag.

If GitOps says:

```text
order-service:1.4.2
```

the tag could theoretically point somewhere different later.

If GitOps says:

```text
order-service@sha256:abcdef...
```

the content is explicitly identified.

Therefore:

```text
Production
    ↓
Exact digest
    ↓
Exact image
```

This gives stronger reproducibility.

---

# 14. ECR Tag Immutability

For production repositories, enable:

```text
IMMUTABLE
```

Conceptually:

```text
order-service:8f3a21c9
        │
        ▼
Image A
```

A second push attempting to overwrite:

```text
order-service:8f3a21c9
```

should fail.

This protects the relationship:

```text
tag → image
```

---

# 15. Why Mutable Tags Are Dangerous

Without immutability:

```text
10:00 AM
order-service:1.4.2
        ↓
Image A
```

Someone later pushes:

```text
order-service:1.4.2
        ↓
Image B
```

Now:

```text
1.4.2
```

doesn't reliably identify the original artifact.

This is unacceptable for controlled production releases.

---

# 16. Terraform ECR Repository

Our infrastructure repository can provision ECR.

Example:

```hcl
resource "aws_ecr_repository" "this" {

  name = var.repository_name

  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Name = var.repository_name
  }
}
```

For our service:

```hcl
module "order_service_ecr" {

  source = "../../modules/ecr"

  repository_name = "order-service"
}
```

---

# 17. ECR Encryption

Container images should be encrypted at rest.

For example:

```hcl
encryption_configuration {
  encryption_type = "AES256"
}
```

Organizations with stronger key-management requirements may use:

```text
AWS KMS
```

with a customer-managed KMS key.

Architecture:

```text
ECR
 │
 └── KMS
      │
      └── Encryption key
```

This becomes especially important for regulated or compliance-heavy environments.

---

# 18. ECR Access

We should never make ECR publicly writable.

Access should be controlled through IAM.

Typical identities:

```text
CI
 │
 └── Push image

EKS node/workload identity
 │
 └── Pull image

Developers
 │
 └── Read access as required
```

---

# 19. CI Push Permissions

The CI role needs only the permissions required to push.

Typical permissions include:

```text
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
```

The important principle is:

> **The CI role should not have unrestricted AWS administrator permissions merely because it needs to push an image.**

---

# 20. EKS Pull Permissions

When EKS needs an image:

```text
EKS
 │
 ▼
ECR
 │
 ▼
Image
```

The relevant AWS identity must have appropriate ECR pull permissions.

The exact identity mechanism depends on the EKS architecture and whether we're using node IAM, EKS Pod Identity, or another supported model.

The principle remains:

```text
WHO
 ↓
CAN PULL
 ↓
FROM WHICH ECR REPOSITORY
```

---

# 21. Image Lifecycle

Images accumulate quickly.

Imagine:

```text
10 deployments/day
×
21 microservices
×
365 days
```

This can produce thousands of images.

Therefore ECR lifecycle policies are necessary.

---

# 22. Untagged Images

Suppose:

```text
Image A
   ↓
tag removed
   ↓
untagged image
```

Untagged images usually don't need indefinite retention.

Example lifecycle rule:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

# 23. Production Retention Is Different

Do not blindly say:

```text
Delete everything older than 7 days.
```

Production retention depends on:

* rollback requirements
* release frequency
* compliance
* incident investigation
* disaster recovery
* audit requirements
* storage cost

For example:

```text
DEV
  ↓
short retention

QA
  ↓
moderate retention

PROD
  ↓
longer controlled retention
```

---

# 24. Rollback Requirement

Imagine production currently runs:

```text
1.5.2
```

A deployment of:

```text
1.5.3
```

causes an outage.

We need:

```text
1.5.2
```

to still exist.

Therefore:

```text
ECR
 ├── 1.5.1
 ├── 1.5.2  ← rollback target
 └── 1.5.3  ← failed release
```

Never design retention without considering rollback.

---

# 25. Build Once, Promote Many

This is one of the most important concepts in our architecture.

Bad:

```text
DEV
 ↓
build image A

QA
 ↓
build image B

PROD
 ↓
build image C
```

Even if source is supposedly the same, the artifacts can differ.

Better:

```text
Source
  ↓
Build ONCE
  ↓
Image A
  ↓
ECR
  │
  ├── DEV
  ├── QA
  ├── STAGING
  └── PROD
```

The exact same artifact is promoted.

---

# 26. Environment Promotion

Our production flow:

```text
                   ECR
                    │
             Image: abc123
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
         DEV       QA        PROD
```

Not:

```text
DEV → rebuild
QA  → rebuild
PROD → rebuild
```

---

# 27. GitOps Controls Promotion

Our CI pipeline should not directly run:

```bash
kubectl apply
```

against production.

Instead:

```text
CI
 │
 ├── Build
 ├── Test
 ├── Scan
 └── Push ECR
       │
       ▼
   GitOps update
       │
       ▼
   Pull Request
       │
       ▼
     Review
       │
       ▼
     ArgoCD
       │
       ▼
      EKS
```

This provides separation between:

```text
Build
```

and:

```text
Deployment
```

---

# 28. Example Promotion

Suppose CI produces:

```text
order-service:8f3a21c9
```

DEV GitOps:

```yaml
image:
  tag: "8f3a21c9"
```

After DEV validation:

```text
DEV
 ↓
Testing
 ↓
Approval
 ↓
QA
```

QA points to the **same image**:

```yaml
image:
  tag: "8f3a21c9"
```

Then production:

```yaml
image:
  tag: "8f3a21c9"
```

or preferably the corresponding digest.

---

# 29. What CI Should Record

After pushing the image, CI should record:

```text
Application:
order-service

Git commit:
8f3a21c9

Image:
order-service:8f3a21c9

Registry:
ECR

Digest:
sha256:...

Build:
GitHub Actions #1234

SBOM:
sbom.json

Security scan:
PASSED
```

This creates traceability.

---

# 30. Artifact Traceability

Our ideal chain:

```text
GitHub PR #245
      │
      ▼
Commit 8f3a21c9
      │
      ▼
CI Build #1234
      │
      ▼
Image order-service:8f3a21c9
      │
      ▼
Digest sha256:abc...
      │
      ▼
ECR
      │
      ▼
GitOps commit
      │
      ▼
ArgoCD deployment
      │
      ▼
EKS
```

Now production operations can answer:

> Where did this running container come from?

---

# 31. Kubernetes Image Reference

Eventually our Helm values might contain:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f3a21c9"
```

or:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:abc..."
```

The second approach gives stronger immutability.

---

# 32. Don't Change Images Manually in Production

Bad operational process:

```text
Engineer
   ↓
kubectl edit deployment
   ↓
change image
```

Now Kubernetes differs from Git.

Instead:

```text
GitOps
   ↓
Change image reference
   ↓
PR
   ↓
Review
   ↓
Merge
   ↓
ArgoCD
   ↓
EKS
```

Git remains the source of truth.

---

# 33. Rollback

Suppose:

```text
Production
order-service:8f3a21c9
```

New deployment:

```text
order-service:92bd4421
```

causes errors.

With GitOps:

```text
GitOps
  │
  └── revert deployment commit
          │
          ▼
       old image
          │
          ▼
        ArgoCD
          │
          ▼
         EKS
```

This is a clean rollback mechanism.

---

# 34. Emergency Rollback

During a severe incident, operational procedures may allow an emergency rollback.

But even then, the long-term desired state should be restored through Git.

For example:

```text
Emergency action
      ↓
Restore service
      ↓
Document incident
      ↓
Update GitOps
      ↓
Git = actual desired state
```

Otherwise drift develops.

---

# 35. Image Vulnerability After Deployment

Imagine:

```text
Day 1
Image 8f3a21c9
       ↓
Trivy PASS
       ↓
Production
```

Day 30:

```text
New CVE published
       ↓
Component inside image affected
       ↓
Existing production image now vulnerable
```

We cannot say:

> "The image passed CI 30 days ago, so it is permanently safe."

Security is continuous.

Possible response:

```text
CVE detected
   ↓
Identify affected images
   ↓
Identify affected services
   ↓
Patch dependency/base image
   ↓
Build new image
   ↓
Scan
   ↓
Deploy
```

---

# 36. Base Image Updates

Our Dockerfile uses:

```dockerfile
FROM eclipse-temurin:21-jre
```

Base images also receive security updates.

Therefore:

```text
Application dependency
+
Base image
+
OS packages
```

must all be considered.

A new secure base image may require rebuilding the application image.

---

# 37. Don't Assume Rebuilding Is Always Safe

Base image update:

```text
Old
21-jre
   ↓
New
21-jre
```

Still needs:

```text
Build
 ↓
Tests
 ↓
Security scans
 ↓
Integration tests
 ↓
DEV
 ↓
QA
 ↓
PROD
```

Security updates must still pass the delivery process.

---

# 38. Multi-Environment ECR Strategy

There are two common organizational approaches.

### Approach A — One ECR account/repository

```text
ECR
└── order-service
```

Same image promoted across environments.

### Approach B — Separate AWS accounts

```text
DEV Account
└── ECR

QA Account
└── ECR

PROD Account
└── ECR
```

This is common in mature multi-account environments.

The image can be replicated/promoted between accounts.

---

# 39. Recommended Enterprise Account Model

Our target architecture eventually looks like:

```text
AWS Organization
│
├── Management Account
│
├── Security Account
│
├── Logging Account
│
├── Dev Account
│    └── ECR
│
├── QA Account
│    └── ECR
│
└── Production Account
     └── ECR
```

Production is isolated from lower environments.

This is more realistic for enterprise architecture than placing everything in one AWS account.

---

# 40. ECR Cross-Account Promotion

Conceptually:

```text
DEV ECR
   │
   │ approved image
   ▼
PROD ECR
   │
   ▼
Production EKS
```

The key principle is:

> **Promote the existing artifact rather than rebuilding it for production.**

---

# 41. Why Production Account Isolation Matters

If CI only has access to DEV:

```text
CI
 │
 └── DEV ECR
```

and production is separate:

```text
PROD
 │
 └── PROD ECR
```

then compromising a development pipeline does not automatically provide unrestricted production access.

This is a major security boundary.

---

# 42. ECR Repository Policy

ECR repository policies can control access between AWS principals/accounts.

Conceptually:

```text
Production ECR
     │
     ├── Production EKS → Pull
     │
     ├── Approved promotion role → Push
     │
     └── Developers → Limited/No write
```

This is preferable to:

```text
Everyone
   ↓
ecr:*
```

---

# 43. Production Image Promotion Model

A mature design:

```text
Developer
    ↓
CI
    ↓
Build
    ↓
Scan
    ↓
Sign
    ↓
DEV ECR
    ↓
DEV deployment
    ↓
Validation
    ↓
Promotion approval
    ↓
PROD ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
PROD EKS
```

Notice that:

```text
PROD ECR
```

receives an **approved artifact**, not an arbitrary new build.

---

# 44. Image Metadata

We should maintain useful metadata.

Examples:

```text
org.opencontainers.image.source
org.opencontainers.image.revision
org.opencontainers.image.version
org.opencontainers.image.created
```

Docker build can use OCI labels.

Conceptually:

```dockerfile
LABEL org.opencontainers.image.source="..."
LABEL org.opencontainers.image.revision="..."
LABEL org.opencontainers.image.version="..."
```

This improves artifact traceability.

---

# 45. Image Tagging Convention

For our project, a practical convention is:

```text
<git-sha>
```

for every CI build.

Release builds may additionally receive:

```text
<semantic-version>
```

Example:

```text
order-service:8f3a21c9
order-service:1.4.2
```

Avoid tags such as:

```text
latest
final
prod
stable
new
test
```

for immutable production identification.

---

# 46. Production ECR Checklist

For each production ECR repository:

```text
[ ] Private repository
[ ] Immutable tags
[ ] Encryption enabled
[ ] Scan-on-push enabled where required
[ ] Lifecycle policy
[ ] Repository access policy
[ ] Least-privilege IAM
[ ] CloudTrail/audit visibility
[ ] Image retention policy
[ ] Backup/replication strategy where required
[ ] Vulnerability monitoring
[ ] Image signing/verification strategy
```

---

# 47. What Happens During a Normal Release?

Let's follow `order-service` through a real release.

Developer changes:

```text
Order validation
```

Commit:

```text
8f3a21c9
```

CI starts:

```text
Build
 ↓
Test
 ↓
SonarQube
 ↓
Gitleaks
 ↓
Trivy
```

Everything passes.

Docker:

```text
order-service:8f3a21c9
```

Image scan:

```text
PASS
```

SBOM:

```text
Generated
```

Image:

```text
ECR
```

Then GitOps:

```text
order-service
image → 8f3a21c9
```

ArgoCD detects the Git change.

Then:

```text
EKS
 ↓
Rolling deployment
 ↓
New Pods
 ↓
Readiness checks
 ↓
Traffic
```

---

# 48. What Happens During a Failed Release?

Suppose:

```text
order-service:92bd4421
```

is deployed.

Then:

```text
HTTP 500 increases
```

Monitoring detects:

```text
Error rate ↑
```

SRE investigates:

```text
Current image = 92bd4421
Previous image = 8f3a21c9
```

Rollback:

```text
GitOps
 ↓
revert
 ↓
8f3a21c9
 ↓
ArgoCD
 ↓
EKS
```

Now production returns to the previous known-good artifact.

---

# 49. The Most Important Rule

Never think:

```text
"Deploy version 1.4.2."
```

Think:

```text
"Deploy the exact artifact represented by this immutable digest,
which was produced from this Git commit and passed these security gates."
```

That is the difference between basic container deployment and mature software supply-chain management.

---

# 50. Production Mental Model

Keep this architecture in mind:

```text
                SOURCE
                   │
                   ▼
                GitHub
                   │
                   ▼
                  CI
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Tests     Security     Build
                   │          │
                   └────┬─────┘
                        ▼
                     IMAGE
                        │
                 ┌──────┴──────┐
                 ▼             ▼
               SBOM         Signature
                 │             │
                 └──────┬──────┘
                        ▼
                       ECR
                        │
                        ▼
                 Image Digest
                        │
                        ▼
                    GitOps
                        │
                        ▼
                     ArgoCD
                        │
                        ▼
                      EKS
                        │
                        ▼
                  Production
```

---

# 51. Key Production Principles

Remember these:

### Principle 1

**Never use `latest` for controlled production deployments.**

### Principle 2

**Use immutable image tags.**

### Principle 3

**Prefer digest-based deployment for strong immutability.**

### Principle 4

**Build once, promote the same artifact.**

### Principle 5

**Do not rebuild separately for PROD.**

### Principle 6

**ECR stores artifacts; ArgoCD deploys them.**

### Principle 7

**GitOps remains the deployment source of truth.**

### Principle 8

**CI should have only the AWS permissions it needs.**

### Principle 9

**Keep enough image history for rollback and compliance.**

### Principle 10

**A previously scanned image can become vulnerable when new CVEs are discovered.**

---

# 52. Final Architecture for Our Project

```text
                         GITHUB
                            │
                            ▼
                    ┌───────────────┐
                    │ CI             │
                    │                │
                    │ Build          │
                    │ Test           │
                    │ SonarQube      │
                    │ Gitleaks       │
                    │ Trivy          │
                    │ SBOM           │
                    │ Sign           │
                    └───────┬────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   DEV ECR     │
                    │               │
                    │ order-service │
                    └───────┬───────┘
                            │
                       Promotion
                            │
                            ▼
                    ┌───────────────┐
                    │  PROD ECR     │
                    │               │
                    │ Immutable     │
                    │ Images        │
                    └───────┬───────┘
                            │
                            ▼
                       GitOps Repo
                            │
                            ▼
                         ArgoCD
                            │
                            ▼
                     Production EKS
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
       order-service   payment-service   catalog-service
```

This is the foundation we'll use when we later build the **actual EKS deployment and GitOps promotion process**.

---

## Next Chapter

**`17-CI-Artifact-and-Image-Promotion.md`**

This is where we'll connect everything we've built so far:

```text
GitHub / GitLab / Jenkins
          ↓
CI
          ↓
Build ONCE
          ↓
Security gates
          ↓
ECR
          ↓
DEV
          ↓
QA
          ↓
STAGING
          ↓
PRODUCTION
```

We'll cover **how a real enterprise promotes the exact same image between environments, who approves each promotion, how GitOps updates happen, how digest promotion works, and how rollback works without rebuilding the application.**
