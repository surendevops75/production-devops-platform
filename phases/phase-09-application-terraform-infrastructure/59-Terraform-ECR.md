# 59-Terraform-ECR.md

# Terraform ECR — Production-Grade Container Registry

This chapter implements **Amazon Elastic Container Registry (ECR)** for the Enterprise E-Commerce Platform.

The goal is not simply to create an ECR repository. We will build the repository structure so that it works correctly with:

* GitHub Actions
* IAM/OIDC
* Docker
* Trivy
* EKS
* GitOps
* ArgoCD
* immutable image promotion
* production rollback
* image retention
* security scanning

---

# 1. What ECR Does

ECR is the private container registry where our built Docker images are stored.

Our deployment flow is:

```text
Developer
    |
    v
GitHub Repository
    |
    v
GitHub Actions
    |
    +--> Build
    +--> Unit Tests
    +--> SonarQube
    +--> Dependency Scan
    +--> Secret Scan
    +--> Docker Build
    +--> Trivy Scan
    |
    v
AWS ECR
    |
    | image
    v
GitOps Repository
    |
    v
ArgoCD
    |
    v
Amazon EKS
```

ECR is therefore the boundary between **CI** and **CD**.

---

# 2. Why We Use One ECR Repository Per Microservice

We have 21 microservices:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

Therefore:

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

This gives us independent:

* permissions
* lifecycle policies
* vulnerability visibility
* image history
* deployment ownership
* retention policies
* rollback artifacts

---

# 3. Repository Naming

We use the service name directly:

```text
order-service
payment-service
inventory-service
```

Avoid names such as:

```text
prod-order
order-prod-final
order-service-new
order-service-v2
my-order-image
```

Environment should generally **not** be embedded into the repository name.

Instead:

```text
order-service
```

contains immutable images that can be promoted across:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

The same artifact should be promoted.

---

# 4. ECR Registry

For AWS account:

```text
123456789012
```

and region:

```text
ap-south-1
```

the registry address is:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

An image becomes:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0
```

or preferably:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:<git-sha>
```

---

# 5. Important Principle — Never Use `latest`

Avoid:

```text
order-service:latest
```

Why?

Imagine:

```text
latest -> version A
```

then a new deployment pushes:

```text
latest -> version B
```

Now Kubernetes cannot reliably tell which artifact was originally deployed.

Instead use:

```text
order-service:8f42a91
```

or:

```text
order-service:1.4.2
```

Even better, deployment systems can pin the image digest:

```text
order-service@sha256:abc123...
```

---

# 6. Our Image Strategy

We will use:

```text
Git commit
    |
    v
Docker image
    |
    v
Git SHA tag
    |
    v
ECR
    |
    v
Image digest
    |
    v
GitOps
    |
    v
EKS
```

Example:

```text
Git SHA:

8f42a91c6d1e...
```

Image:

```text
order-service:8f42a91c6d1e
```

ECR stores the immutable artifact.

---

# 7. Terraform Directory

Our infrastructure repository now becomes:

```text
infrastructure/
│
├── terraform/
│   │
│   ├── modules/
│   │   │
│   │   ├── vpc/
│   │   ├── iam/
│   │   ├── ecr/
│   │   ├── eks/
│   │   ├── rds/
│   │   ├── redis/
│   │   ├── dynamodb/
│   │   ├── s3/
│   │   ├── sqs/
│   │   ├── sns/
│   │   └── kms/
│   │
│   └── environments/
│       ├── dev/
│       ├── qa/
│       └── prod/
│
└── README.md
```

---

# 8. Create ECR Module

Navigate to:

```bash
cd infrastructure/terraform/modules
```

Create:

```bash
mkdir -p ecr
cd ecr
```

Files:

```text
ecr/
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

---

# 9. `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}
```

We deliberately avoid hardcoding an arbitrary AWS provider version here.

The root environment should pin the provider version used by the project and commit:

```text
.terraform.lock.hcl
```

to Git.

---

# 10. `variables.tf`

```hcl
variable "repository_name" {
  description = "Name of the ECR repository"
  type        = string
}

variable "tags" {
  description = "Tags applied to ECR resources"
  type        = map(string)
  default     = {}
}

variable "force_delete" {
  description = "Whether Terraform may delete the repository even when it contains images"
  type        = bool
  default     = false
}
```

---

# 11. `main.tf`

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

  force_delete = var.force_delete

  tags = merge(
    var.tags,
    {
      Name = var.repository_name
    }
  )
}
```

---

# 12. Why `IMMUTABLE`?

This is extremely important.

With:

```hcl
image_tag_mutability = "IMMUTABLE"
```

once:

```text
order-service:8f42a91
```

has been pushed, another image cannot overwrite the same tag.

For example:

```text
8f42a91 -> image A
```

Someone cannot later push:

```text
8f42a91 -> image B
```

This protects our release artifact.

---

# 13. Why Scan on Push?

We enable:

```hcl
image_scanning_configuration {
  scan_on_push = true
}
```

This provides registry-side image vulnerability scanning.

However, this does **not** replace our CI security pipeline.

We still perform:

```text
Trivy
```

during CI.

So we have:

```text
Developer
    |
    v
GitHub Actions
    |
    +--> Trivy
    |
    v
ECR
    |
    +--> ECR scanning
```

Two different security layers.

---

# 14. ECR Encryption

Our initial configuration:

```hcl
encryption_type = "AES256"
```

means ECR image data is encrypted at rest using AWS-managed encryption.

For stricter enterprise requirements, we can use a customer-managed KMS key:

```hcl
encryption_configuration {
  encryption_type = "KMS"
  kms_key         = aws_kms_key.ecr.arn
}
```

We will cover centralized KMS design separately.

Do not create a new KMS key for every tiny resource without an actual security/operational reason.

---

# 15. `outputs.tf`

```hcl
output "repository_name" {
  description = "ECR repository name"
  value       = aws_ecr_repository.this.name
}

output "repository_arn" {
  description = "ECR repository ARN"
  value       = aws_ecr_repository.this.arn
}

output "repository_url" {
  description = "ECR repository URL"
  value       = aws_ecr_repository.this.repository_url
}

output "registry_id" {
  description = "AWS account registry ID"
  value       = aws_ecr_repository.this.registry_id
}
```

---

# 16. ECR Lifecycle Policy

Images can accumulate very quickly.

Suppose CI builds:

```text
20 images/day
```

After six months:

```text
20 × 180
= 3,600 images
```

Keeping every temporary image forever creates unnecessary storage cost and operational clutter.

We therefore need a lifecycle policy.

---

# 17. Lifecycle Policy

Create:

```text
lifecycle-policy.json
```

Example:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images older than 7 days",
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

Terraform:

```hcl
resource "aws_ecr_lifecycle_policy" "this" {
  repository = aws_ecr_repository.this.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1

        description = "Delete untagged images older than 7 days"

        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }

        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

---

# 18. Why Only Untagged Images Initially?

Be careful with production retention.

Suppose:

```text
1.0.0
1.1.0
1.2.0
1.3.0
```

are production releases.

If we aggressively delete old tagged images, we may destroy our rollback capability.

Therefore:

```text
Untagged temporary images
        ↓
delete after retention period
```

while:

```text
Production release images
        ↓
retain according to rollback/compliance requirements
```

A real retention policy should be based on:

* rollback window
* deployment frequency
* compliance
* incident investigation
* disaster recovery
* storage cost

---

# 19. Dev vs Production Retention

For DEV:

```text
Untagged > 7 days
    -> delete
```

For PROD:

```text
Production releases
    -> retain according to release/rollback policy
```

For example, an organization might choose:

```text
Recent releases -> retain
Older releases -> expire after approved retention period
```

Do not blindly copy a seven-day policy into production.

---

# 20. Environment Configuration

Now we connect the ECR module to:

```text
terraform/environments/dev/
```

Example:

```text
terraform/environments/dev/
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
└── backend.tf
```

---

# 21. `main.tf`

```hcl
module "order_service_ecr" {
  source = "../../modules/ecr"

  repository_name = "order-service"

  tags = {
    Environment = "dev"
    Service     = "order-service"
    ManagedBy   = "terraform"
    Project     = "enterprise-ecommerce"
  }
}
```

---

# 22. Add the Other Services

We eventually need 21 repositories.

Example:

```hcl
module "auth_service_ecr" {
  source = "../../modules/ecr"

  repository_name = "auth-service"

  tags = {
    Environment = "dev"
    Service     = "auth-service"
    ManagedBy   = "terraform"
  }
}

module "user_service_ecr" {
  source = "../../modules/ecr"

  repository_name = "user-service"

  tags = {
    Environment = "dev"
    Service     = "user-service"
    ManagedBy   = "terraform"
  }
}
```

And so on.

However, manually repeating 21 blocks is unnecessary.

---

# 23. Better Terraform Pattern

Use a list/set of services.

`variables.tf`:

```hcl
variable "microservices" {
  description = "Microservices requiring ECR repositories"
  type        = set(string)

  default = [
    "auth-service",
    "user-service",
    "customer-service",
    "catalog-service",
    "product-service",
    "inventory-service",
    "pricing-service",
    "promotion-service",
    "cart-service",
    "order-service",
    "payment-service",
    "fraud-service",
    "shipping-service",
    "delivery-service",
    "notification-service",
    "search-service",
    "recommendation-service",
    "review-service",
    "tax-service",
    "invoice-service",
    "reporting-service"
  ]
}
```

Then:

```hcl
module "ecr" {
  for_each = var.microservices

  source = "../../modules/ecr"

  repository_name = each.value

  tags = {
    Environment = "dev"
    Service     = each.value
    ManagedBy   = "terraform"
    Project     = "enterprise-ecommerce"
  }
}
```

This creates:

```text
21 ECR repositories
```

from one module call.

---

# 24. Production Environment

The same pattern can be used in:

```text
terraform/environments/prod/
```

but production should use production-specific configuration.

For example:

```hcl
module "ecr" {
  for_each = var.microservices

  source = "../../modules/ecr"

  repository_name = each.value

  tags = {
    Environment = "prod"
    Service     = each.value
    ManagedBy   = "terraform"
    Project     = "enterprise-ecommerce"
  }
}
```

---

# 25. Important Architecture Decision

We need to decide whether DEV and PROD use:

### Option A — separate AWS accounts

```text
DEV AWS Account
    |
    +--> ECR

PROD AWS Account
    |
    +--> ECR
```

This is preferred for stronger enterprise isolation.

### Option B — same account

```text
AWS Account
    |
    ├── dev ECR
    └── prod ECR
```

This is simpler for learning.

Our learning implementation can start with one account, but the production architecture should be designed so moving to separate accounts is straightforward.

---

# 26. Build the Terraform Configuration

From:

```bash
cd infrastructure/terraform/environments/dev
```

run:

```bash
terraform init
```

Then:

```bash
terraform fmt -recursive
```

Then:

```bash
terraform validate
```

Then:

```bash
terraform plan
```

You should see resources such as:

```text
aws_ecr_repository
aws_ecr_lifecycle_policy
```

for the services.

---

# 27. Apply

For DEV:

```bash
terraform apply
```

Review:

```text
Plan: 42 to add, 0 to change, 0 to destroy.
```

The exact number depends on the number of resources/modules currently defined.

Never assume the exact plan count.

---

# 28. Verify ECR

AWS CLI:

```bash
aws ecr describe-repositories \
  --region ap-south-1
```

For one repository:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

---

# 29. Get Repository URL

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1 \
  --query 'repositories[0].repositoryUri' \
  --output text
```

Expected:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

---

# 30. Docker Login to ECR

Before pushing an image:

```bash
aws ecr get-login-password \
  --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Expected:

```text
Login Succeeded
```

---

# 31. Build the Order Service

From the `order-service` repository:

```bash
docker build \
  -t order-service:8f42a91 \
  .
```

Check:

```bash
docker images
```

---

# 32. Tag the Image for ECR

```bash
docker tag \
  order-service:8f42a91 \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f42a91
```

Now:

```bash
docker images
```

will show the local image and ECR-tagged image.

---

# 33. Push

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f42a91
```

Docker uploads:

```text
layers
manifest
metadata
```

to ECR.

---

# 34. Verify the Image

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

Or:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

---

# 35. Get Image Digest

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=8f42a91 \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Example:

```text
sha256:abc123...
```

This digest identifies the exact image artifact.

---

# 36. Tag vs Digest

Tag:

```text
order-service:8f42a91
```

Digest:

```text
order-service@sha256:abc123...
```

The tag is convenient for humans and CI/CD.

The digest provides exact artifact identity.

For high-control production deployments, GitOps can ultimately pin:

```text
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

rather than relying only on a mutable reference.

---

# 37. Build Once, Promote Many Times

This is one of the most important principles in our architecture.

Do **not** do:

```text
DEV
  |
  +--> rebuild image
  |
QA
  |
  +--> rebuild image
  |
PROD
  |
  +--> rebuild image
```

Instead:

```text
Source
  |
  v
Build
  |
  v
Image
  |
  v
ECR
  |
  +------> DEV
  |
  +------> QA
  |
  +------> STAGING
  |
  +------> PROD
```

The exact same artifact moves through environments.

---

# 38. Why This Matters

Suppose:

```text
DEV image = A
QA image  = B
PROD image = C
```

You don't actually know whether the same source was deployed.

With build-once:

```text
Git SHA
   |
   v
Docker image
   |
   v
Digest
   |
   +--> DEV
   +--> QA
   +--> PROD
```

we can prove that production is running the tested artifact.

---

# 39. ECR + GitHub Actions

Our IAM chapter created the architecture:

```text
GitHub
   |
   | OIDC
   v
AWS STS
   |
   v
GitHub Actions IAM Role
   |
   v
ECR
```

There are no long-lived AWS access keys stored in GitHub.

GitHub Actions obtains short-lived credentials.

---

# 40. CI ECR Permissions

The CI role should have only the permissions required for pushing images.

Typical ECR push permissions include:

```text
ecr:GetAuthorizationToken

ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
```

The repository-level permissions should be restricted to the repositories that CI actually needs.

Do not give:

```text
AdministratorAccess
```

to the CI role.

---

# 41. EKS Pull Access

The other direction is:

```text
EKS
 |
 v
ECR
 |
 v
Pull image
```

The workload must be able to pull the image.

Depending on the EKS node/workload architecture, ECR permissions are provided through the appropriate AWS identity associated with the image-pulling mechanism.

This will be implemented carefully in the EKS chapter rather than blindly attaching excessive permissions.

---

# 42. Important Distinction

CI needs:

```text
PUSH
```

EKS needs:

```text
PULL
```

Therefore:

```text
GitHub Actions IAM
    |
    +--> ECR PUSH

EKS runtime identity
    |
    +--> ECR PULL
```

Do not use the same broad role for both.

---

# 43. ECR and Trivy

Our CI pipeline will eventually look like:

```text
Checkout
   |
Build
   |
Unit Tests
   |
SonarQube
   |
Dependency Scan
   |
Secret Scan
   |
Docker Build
   |
Trivy Image Scan
   |
AWS OIDC
   |
ECR Login
   |
ECR Push
```

Trivy runs in the CI environment.

It does **not** need to run inside:

```text
order-service pod
```

The production EKS workload should not contain unnecessary security tooling.

---

# 44. Example Trivy Command

After building:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  order-service:8f42a91
```

If critical vulnerabilities are found:

```text
exit code 1
```

and CI can fail before publishing the image.

The exact vulnerability policy should be defined by the organization's risk model and exception process.

---

# 45. ECR Scan vs Trivy

They serve different roles.

| Tool         | Where   | Purpose                               |
| ------------ | ------- | ------------------------------------- |
| Trivy        | CI      | Shift-left vulnerability scanning     |
| ECR scanning | AWS/ECR | Registry-side image scanning          |
| SonarQube    | CI      | Source-code quality/security analysis |
| Gitleaks     | CI      | Secret detection                      |
| IAM          | AWS     | Access control                        |
| KMS          | AWS     | Encryption                            |

So:

```text
Source security
       ↓
CI security
       ↓
Registry security
       ↓
Runtime security
```

---

# 46. What Happens If We Push the Same Immutable Tag?

Suppose:

```text
order-service:8f42a91
```

already exists.

Try:

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f42a91
```

with a different image.

Because:

```text
IMMUTABLE
```

ECR will reject the overwrite.

This is desirable.

It prevents accidental replacement of an artifact.

---

# 47. Production Release Example

Developer commits:

```text
feat: add order validation
```

Git SHA:

```text
a91c2f7
```

CI creates:

```text
order-service:a91c2f7
```

Trivy scans it.

CI pushes:

```text
ECR
└── order-service
    └── a91c2f7
```

GitOps updates:

```yaml
image:
  tag: a91c2f7
```

ArgoCD deploys:

```text
EKS
└── order-service
    └── a91c2f7
```

---

# 48. Rollback

Suppose:

```text
Current:

order-service:a91c2f7
```

has a production problem.

The previous version was:

```text
order-service:7bc921a
```

GitOps can be reverted to:

```yaml
image:
  tag: 7bc921a
```

ArgoCD reconciles:

```text
GitOps
   |
   v
ArgoCD
   |
   v
EKS
   |
   v
7bc921a
```

We do not rebuild the old application.

The old immutable artifact already exists in ECR.

---

# 49. Cross-Account ECR

In a mature enterprise architecture:

```text
DEV Account
    |
    +--> ECR

QA Account
    |
    +--> ECR

PROD Account
    |
    +--> ECR
```

There are different ways to implement image promotion.

One pattern is:

```text
Build Account
     |
     v
Central ECR
     |
     +--> DEV
     +--> QA
     +--> PROD
```

Another is:

```text
CI Account
   |
   v
ECR
   |
   +--> replicate/promote
          |
          +--> PROD ECR
```

Cross-account permissions and replication should be explicitly designed rather than granting broad ECR access.

---

# 50. ECR Replication

For larger environments, ECR supports registry replication capabilities that can help distribute images between AWS accounts/regions.

This can be useful for:

```text
Multi-account
Multi-region
DR
```

Example:

```text
Mumbai
ap-south-1
    |
    v
Primary ECR
    |
    +------------------+
    |                  |
    v                  v
Mumbai PROD        DR Region
                   ap-southeast-1
```

Our detailed DR implementation comes later.

---

# 51. ECR and Disaster Recovery

ECR should be part of our DR planning.

Imagine the primary AWS region becomes unavailable.

We need the application image available in the recovery region/account.

Possible strategy:

```text
Primary ECR
     |
     v
ECR replication
     |
     v
DR ECR
```

Then:

```text
DR EKS
   |
   v
DR ECR
   |
   v
same application image
```

This is especially important when rebuilding EKS from Terraform.

---

# 52. ECR Repository Security

Production ECR should have:

* private repositories
* immutable tags
* encryption
* vulnerability scanning
* least-privilege IAM
* lifecycle policies
* CloudTrail visibility
* controlled cross-account access
* controlled deletion
* retention appropriate for rollback
* image provenance strategy

Later we can add:

```text
SBOM
image signing
attestation
admission verification
```

---

# 53. ECR Doesn't Deploy Anything

A common misconception is:

> "I pushed the image to ECR, so ECR deploys it."

No.

ECR only stores the image.

The flow is:

```text
Docker
   |
   v
ECR
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
Kubernetes
   |
   v
EKS
```

ECR does not replace:

* Helm
* GitOps
* ArgoCD
* Kubernetes
* deployment strategy

---

# 54. ECR vs Docker Hub

For our AWS production platform:

```text
Application
     |
     v
Private ECR
```

instead of:

```text
Application
     |
     v
Public Docker Hub
```

Benefits include:

* AWS IAM integration
* private repositories
* AWS-native authentication
* controlled access
* EKS integration
* encryption
* scanning
* AWS audit capabilities

---

# 55. ECR Troubleshooting

## Problem 1 — Repository Does Not Exist

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

Possible causes:

```text
Wrong AWS account
Wrong region
Repository wasn't created
Wrong repository name
```

---

# 56. Problem 2 — Docker Login Failure

Check:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws configure get region
```

Login again:

```bash
aws ecr get-login-password \
  --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

---

# 57. Problem 3 — AccessDenied

Example:

```text
AccessDeniedException
```

Check the identity:

```bash
aws sts get-caller-identity
```

Then inspect the IAM role.

Typical CI mistake:

```text
Role can authenticate to AWS
BUT
Role doesn't have ECR push permissions
```

OIDC authentication and ECR authorization are separate concepts.

---

# 58. Problem 4 — Immutable Tag Error

If you see an error indicating the image tag already exists:

```text
IMMUTABLE
```

do not try to overwrite it.

Create a new tag:

```text
new Git SHA
```

For example:

```text
8f42a91
```

instead of reusing:

```text
7bc921a
```

---

# 59. Problem 5 — EKS ImagePullBackOff

If EKS reports:

```text
ImagePullBackOff
```

check:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Failed to pull image
```

Then verify:

```text
1. ECR repository exists
2. Correct region
3. Correct account
4. Image/tag exists
5. EKS node/workload identity can pull
6. Network path works
7. ECR API/Docker registry access is available
```

Do not immediately blame Kubernetes.

Trace:

```text
Pod
 ↓
Kubelet
 ↓
Container runtime
 ↓
ECR authentication
 ↓
ECR registry
```

---

# 60. Verify the Exact Image

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=8f42a91 \
  --region ap-south-1
```

If the image does not exist:

```text
ECR problem
```

If it exists:

```text
continue troubleshooting EKS authentication/network/runtime
```

---

# 61. ECR Validation Checklist

Run:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

Then:

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

Then:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

Then:

```bash
aws ecr get-repository-policy \
  --repository-name order-service \
  --region ap-south-1
```

If no repository policy exists, that is not necessarily an error. Same-account access can be handled through IAM identity policies.

---

# 62. Final ECR Architecture

Our current architecture is:

```text
                     GitHub
                        |
                        | OIDC
                        v
                GitHub Actions
                        |
          +-------------+-------------+
          |             |             |
       Build        Security       Trivy
          |             |             |
          +-------------+-------------+
                        |
                        v
                     AWS ECR
                        |
              +---------+---------+
              |         |         |
              v         v         v
           Dev ECR   QA ECR   Prod ECR
              |         |         |
              +---------+---------+
                        |
                        v
                    GitOps Repo
                        |
                        v
                      ArgoCD
                        |
                        v
                       EKS
                        |
                        v
                 Order Service
```

For the simplified single-account learning implementation:

```text
GitHub Actions
      |
      v
ECR
      |
      v
GitOps
      |
      v
ArgoCD
      |
      v
EKS
```

---

# 63. What We Have Completed

At this point:

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
```

We now have the foundations for:

```text
Source Code
     ↓
Docker
     ↓
ECR
```

and the IAM architecture for:

```text
GitHub
   ↓
OIDC
   ↓
AWS
```

---

# 64. Next Chapter

## `60-Terraform-EKS.md`

This is a major chapter.

We will actually build:

```text
AWS VPC
   |
   v
EKS Control Plane
   |
   +-----------------------------+
   |                             |
   v                             v
Managed Node Group          Future Karpenter
   |
   +-----------------------------+
   |
   +--> system nodes
   |
   +--> application nodes
```

And cover:

* EKS cluster Terraform module
* control plane
* cluster IAM role
* node IAM role
* managed node groups
* private nodes
* Kubernetes API endpoint
* public/private endpoint considerations
* EKS security groups
* cluster logging
* OIDC / Pod Identity relationship
* Kubernetes version strategy
* node instance types
* subnet placement
* capacity planning
* system vs application nodes
* taints/tolerations
* EKS access configuration
* `aws eks update-kubeconfig`
* `kubectl` connectivity
* cluster validation
* node validation
* EKS production architecture
* common EKS creation failures
* why we **do not deploy the 21 microservices yet**

The next milestone will be:

```text
Terraform
   ↓
VPC
   ↓
IAM
   ↓
ECR
   ↓
EKS
```

After that, we install the **real EKS platform add-ons** before deploying the Order Service.
