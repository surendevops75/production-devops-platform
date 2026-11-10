# 08 — Amazon ECR & Container Registry

Now we connect our **Dockerized `order-service`** to AWS.

Our pipeline is becoming:

```text
Developer
    |
    v
GitHub
    |
    v
order-service
    |
    v
Docker Build
    |
    v
Security Scan
    |
    v
Amazon ECR
```

ECR is where AWS will store the container images that EKS eventually pulls.

---

# 1. What Is ECR?

Amazon Elastic Container Registry is AWS's container image registry.

Instead of:

```text
Developer Laptop
      |
      v
Docker Image
```

we'll have:

```text
Developer / CI
      |
      v
Docker Image
      |
      v
Amazon ECR
      |
      v
Amazon EKS
```

For our 21 services:

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

---

# 2. One Repository Per Microservice

We are **not** going to put all images into one ECR repository.

Bad:

```text
ecommerce
   |
   +-- order
   +-- payment
   +-- inventory
```

Instead:

```text
ECR
 |
 +-- order-service
 |
 +-- payment-service
 |
 +-- inventory-service
```

This makes:

* permissions easier
* lifecycle policies easier
* image management easier
* vulnerability management easier
* deployment tracking easier

---

# 3. Image Naming

Our AWS account:

```text
123456789012
```

Region:

```text
ap-south-1
```

ECR registry:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Order repository:

```text
order-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0
```

Or Git SHA:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f72a91
```

---

# 4. Why We Don't Use `latest`

Avoid production:

```text
order-service:latest
```

Imagine:

```text
10:00 AM
latest -> version 1.5.1
```

Production runs it.

Then:

```text
11:00 AM
latest -> version 1.5.2
```

Now the meaning of:

```text
latest
```

changed.

That makes troubleshooting harder.

Instead:

```text
order-service:1.5.1
order-service:1.5.2
```

or:

```text
order-service:<git-sha>
```

The tag should identify a specific build.

---

# 5. Even Better: Image Digest

A Docker tag can be moved unless you enforce immutability.

An image digest identifies the exact image content:

```text
sha256:abcdef123456...
```

Conceptually:

```text
order-service:1.5.2
        |
        v
sha256:abc123...
```

For highly controlled deployments, Kubernetes can reference the image by digest.

For our learning project we'll use immutable version/Git-SHA tags and later discuss digest pinning.

---

# 6. Create ECR Repository With Terraform

We don't want to manually create 21 repositories from the AWS console.

Terraform:

```text
infrastructure
      |
      v
Terraform
      |
      v
ECR
```

Create:

```text
terraform/modules/ecr/
```

Structure:

```text
ecr/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

# 7. ECR Resource

`main.tf`:

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

  image_tag_mutability_exclusion_filter {
    filter_type = "WILDCARD"
    filter      = "dev-*"
  }

  tags = {
    Name = var.repository_name
  }
}
```

The exclusion filter is optional and depends on your tagging strategy. If you want **all tags immutable**, omit the exclusion filter entirely.

For our production repositories, we want immutable release tags.

---

# 8. Why Immutable Tags?

Suppose:

```text
order-service:1.5.2
```

has already been deployed to production.

Someone shouldn't be able to overwrite it with another image.

Immutable:

```text
1.5.2
   |
   X
cannot be replaced
```

Then:

```text
1.5.3
```

is a new image.

This provides much better release traceability.

---

# 9. Image Scanning

We enable:

```hcl
image_scanning_configuration {
  scan_on_push = true
}
```

This means when an image is pushed:

```text
Docker Image
     |
     v
ECR
     |
     v
Vulnerability scanning
```

ECR scanning and our CI scanner serve different purposes.

We'll also use:

```text
Trivy
```

in CI.

---

# 10. Encryption

ECR supports encryption at rest.

Our example:

```hcl
encryption_configuration {
  encryption_type = "AES256"
}
```

For environments with stronger centralized key-management requirements, we can use a customer-managed KMS key.

Then:

```text
ECR
 |
 v
KMS
 |
 v
Encrypted image data
```

---

# 11. Variables

`variables.tf`:

```hcl
variable "repository_name" {

  description = "ECR repository name"

  type = string
}
```

---

# 12. Output

`outputs.tf`:

```hcl
output "repository_url" {

  description = "ECR repository URL"

  value = aws_ecr_repository.this.repository_url
}
```

Terraform will output something like:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

---

# 13. Call the ECR Module

In:

```text
terraform/environments/dev/main.tf
```

add:

```hcl
module "order_service_ecr" {

  source = "../../modules/ecr"

  repository_name = "order-service"
}
```

Now:

```text
Dev Environment
       |
       v
ECR Module
       |
       v
order-service ECR
```

---

# 14. Terraform Plan

Run:

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

You should see something conceptually similar to:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform intends to create:

```text
aws_ecr_repository.this
```

---

# 15. Apply

```bash
terraform apply
```

After approval:

```text
Terraform
   |
   v
AWS ECR
   |
   v
order-service
```

---

# 16. Verify With AWS CLI

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

You should receive repository information.

---

# 17. Login to ECR

Now use your local Docker client.

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

# 18. Build Our Application

From:

```text
order-service/
```

run:

```bash
./mvnw clean package -DskipTests
```

Then:

```bash
docker build \
  -t order-service:1.0.0 \
  .
```

Check:

```bash
docker images
```

---

# 19. Tag for ECR

Local:

```text
order-service:1.0.0
```

ECR:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0
```

Command:

```bash
docker tag \
  order-service:1.0.0 \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0
```

---

# 20. Push

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0
```

Docker uploads:

```text
Image
 |
 +-- Layer 1
 +-- Layer 2
 +-- Layer 3
 +-- Layer 4
 |
 v
ECR
```

---

# 21. Verify Image

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

You should see:

```text
1.0.0
```

---

# 22. Inspect Image

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

This provides:

```text
imageDigest
imageTags
imageSizeInBytes
imagePushedAt
```

This information becomes useful during production troubleshooting.

---

# 23. Image Digest

Suppose ECR reports:

```text
imageTag:
1.0.0

imageDigest:
sha256:abc123...
```

Then we know exactly which content:

```text
order-service:1.0.0
        |
        v
sha256:abc123...
```

was pushed.

---

# 24. CI Will Eventually Build This Automatically

Right now:

```text
Developer
   |
   v
docker build
   |
   v
docker push
```

But developers shouldn't manually push production images.

Eventually:

```text
Developer
   |
   v
GitHub PR
   |
   v
Merge
   |
   v
GitHub Actions
   |
   +--> Build
   +--> Test
   +--> Security
   +--> Docker
   +--> Trivy
   |
   v
ECR
```

That is the next major pipeline we're going to build.

---

# 25. Git SHA Tagging

Suppose commit:

```text
8f72a91
```

caused the build.

CI can produce:

```text
order-service:8f72a91
```

Then:

```text
Git commit
    |
    v
8f72a91
    |
    v
Docker image
    |
    v
ECR
```

Now production can answer:

> Which source code produced this container?

Answer:

```text
commit 8f72a91
```

---

# 26. Better Release Metadata

We can also add OCI labels to the image.

Dockerfile example:

```dockerfile
ARG APP_VERSION
ARG GIT_COMMIT

LABEL org.opencontainers.image.version="${APP_VERSION}"
LABEL org.opencontainers.image.revision="${GIT_COMMIT}"
LABEL org.opencontainers.image.source="https://github.com/company-ecommerce/order-service"
```

Then CI can build:

```bash
docker build \
  --build-arg APP_VERSION=1.0.0 \
  --build-arg GIT_COMMIT=8f72a91 \
  -t order-service:1.0.0 .
```

This makes image metadata much more useful.

---

# 27. ECR Lifecycle Policy

If we continuously build:

```text
order-service:commit1
order-service:commit2
order-service:commit3
...
```

we'll eventually have thousands of images.

We need cleanup.

Example lifecycle policy:

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

This is only a starting example.

Production policy should reflect:

```text
release retention
rollback requirements
compliance
development image volume
storage cost
```

---

# 28. Production Retention Example

A possible strategy:

```text
Production tags
    |
    +-- Keep recent 30 releases

Development tags
    |
    +-- Keep recent 20

Untagged
    |
    +-- Delete after 7 days
```

The exact numbers are organization-specific.

---

# 29. ECR Permissions

Our CI role should be able to:

```text
GetAuthorizationToken
BatchCheckLayerAvailability
InitiateLayerUpload
UploadLayerPart
CompleteLayerUpload
PutImage
```

But it shouldn't automatically be able to:

```text
Delete every repository
Modify IAM
Delete VPC
Create EKS
```

Again:

```text
Least privilege
```

---

# 30. EKS Pull Permission

Later EKS worker infrastructure/workload identity needs to pull the image.

Architecture:

```text
EKS
 |
 v
Container Runtime
 |
 v
ECR
 |
 v
Image
```

The appropriate EKS node/workload permissions will allow image retrieval.

We will configure that as part of the EKS module.

---

# 31. ECR and CI Separation

Notice our roles:

```text
Terraform
   |
   +--> Create ECR repository
   +--> Configure lifecycle
   +--> Configure encryption
```

CI:

```text
GitHub Actions
   |
   +--> Login
   +--> Push image
```

EKS:

```text
EKS
   |
   +--> Pull image
```

Three different responsibilities.

---

# 32. Final ECR Architecture

```text
                     GitHub
                        |
                        v
                  GitHub Actions
                        |
                        | OIDC
                        v
                     AWS IAM
                        |
                        | ECR Push
                        v
              +----------------------+
              |        ECR           |
              |                      |
              | order-service        |
              | payment-service      |
              | inventory-service    |
              | ...                  |
              +----------------------+
                        |
                        | Pull
                        v
                       EKS
                        |
                        v
                       Pod
```

---

# 33. Our Project Progress

We've now built:

```text
01 GitHub Organization             ✓
02 Repository Strategy            ✓
03 Branch / PR Strategy           ✓
04 Order Service Foundation       ✓
05 Dockerization                  ✓
06 AWS Infrastructure Foundation  ✓
07 IAM Foundation                 ✓
08 ECR                            ✓
```

Current actual flow:

```text
Developer
   |
   v
GitHub
   |
   v
order-service
   |
   v
Docker
   |
   v
ECR
```

Infrastructure:

```text
infrastructure
   |
   v
Terraform
   |
   v
AWS VPC
   |
   v
ECR
```

---

# 34. What We Have NOT Done Yet

We haven't yet created:

```text
EKS
```

That's intentional.

Before EKS, we need to make our **CI pipeline** properly build and publish the image.

Our next pipeline will become:

```text
Pull Request
     |
     +--> Checkout
     |
     +--> Java Setup
     |
     +--> Dependency Cache
     |
     +--> Compile
     |
     +--> Unit Tests
     |
     +--> SonarQube
     |
     +--> Dependency Scan
     |
     +--> Secret Scan
     |
     +--> Docker Build
     |
     +--> Trivy
     |
     v
   Merge
     |
     v
GitHub Actions
     |
     +--> AWS OIDC
     |
     +--> ECR Login
     |
     +--> Push Image
     |
     v
    ECR
```

And we'll make this pipeline work with the actual repository we've already created.

---

# 35. Next: Complete CI Pipeline

**Next file: `09-CI-Pipeline-Foundation.md`**

We'll build the actual `.github/workflows/ci.yaml` and not just show a conceptual diagram.

We'll cover:

```text
GitHub Actions
├── workflow triggers
├── permissions
├── jobs
├── runners
├── Maven caching
├── unit tests
├── artifact handling
├── SonarQube
├── dependency scanning
├── secret scanning
├── Docker BuildKit
├── Trivy
├── AWS OIDC
├── ECR login
├── image tagging
├── image push
├── failure handling
├── PR vs main behavior
└── Git SHA traceability
```

The key transition will be:

```text
                     BEFORE

Developer → Docker → ECR


                     AFTER

Developer
    ↓
GitHub PR
    ↓
CI
    ↓
Tests
    ↓
Security
    ↓
Docker
    ↓
Trivy
    ↓
Merge
    ↓
OIDC
    ↓
ECR
```

That will be our **first real production-style DevOps pipeline**.
