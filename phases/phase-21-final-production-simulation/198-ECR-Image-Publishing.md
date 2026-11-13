# PHASE 21 — FINAL PRODUCTION SIMULATION

# Chapter 198 — ECR Image Publishing



Production implementation

Chapter 198 of 212



## 1. Objective

In Chapter 197, we prepared the Docker image for the `order-service`.

In this chapter, we will publish that exact Docker image to Amazon Elastic Container Registry (ECR).

We will implement the complete production workflow:

1. Retrieve the Docker image created in Chapter 197.
2. Verify the image artifact and checksum.
3. Create an ECR repository using Terraform.
4. Configure encryption, image scanning, tag immutability and lifecycle policies.
5. Configure GitHub Actions to authenticate with AWS using OIDC.
6. Give GitHub Actions only the required ECR permissions.
7. Authenticate Docker with ECR.
8. Push the image using its immutable commit SHA tag.
9. Capture the ECR image digest.
10. Verify the image in ECR.
11. Record publishing evidence.
12. Prepare the image reference for GitOps promotion.

Important production rule:

We build the Docker image once and promote the same image across DEV, QA, STAGING and PROD.

We do not rebuild the Docker image separately for each environment.

The image identity is established using its digest.

## 2. End-to-end architecture

[Identity Federation for GitHub Actions on AWS | ScaleSec](https://images.openai.com/static-rsc-4/zrYWtFzPiB-Yc74MksFJHzTUyiDo_67kONcN2tPFVktwM2_5I7VFCbpEdL5UZN3RSVA4mBhb_-QEG44XSACJTwuxO00V4FXBGm3TOML964BUbtRtAtitZccai9DyuSfM-yXE-ANAn_bHZ8Gb9YsiuW326-DvxGwNysGKpwCap2Y?purpose=inline)

[scalesec.com](https://scalesec.com/blog/identity-federation-for-github-actions-on-aws/?utm_source=chatgpt.com)

Our workflow:

```
Developer
   |
   v
GitHub Pull Request
   |
   v
CI Pipeline
   |
   v
Security Scanning
   |
   v
Docker Image Build
   |
   v
Image Artifact
   |
   | Docker image archive
   | SHA256 checksum
   v
GitHub Actions Publish Job
   |
   v
GitHub OIDC Authentication
   |
   v
AWS IAM Role
   |
   v
Amazon ECR
   |
   +--> Image Tag
   |
   +--> Image Digest
   |
   +--> Vulnerability Scan
   |
   +--> Lifecycle Policy
   |
   v
GitOps Repository
   |
   v
ArgoCD Deployment
```

### 2.1 Components involved

| Component      | Responsibility                        |
| -------------- | ------------------------------------- |
| GitHub Actions | Executes image publishing workflow    |
| GitHub OIDC    | Provides trusted identity to AWS      |
| AWS IAM        | Controls ECR access                   |
| Terraform      | Creates ECR repositories and policies |
| Docker         | Authenticates and pushes the image    |
| Amazon ECR     | Stores Docker images                  |
| ECR scanning   | Scans images for vulnerabilities      |
| GitOps         | References the published image        |
| ArgoCD         | Deploys the approved image            |

### 2.2 Current simulation status

| Item           | Value                   |
| -------------- | ----------------------- |
| Jira           | `ECOM-4821`             |
| Service        | `order-service`         |
| Source commit  | `d4e5f6a...`            |
| Image tag      | `d4e5f6a...`            |
| AWS region     | `ap-south-1`            |
| Registry       | Amazon ECR              |
| Environment    | AWS production platform |
| Image digest   | Pending actual push     |
| ECR publishing | This chapter            |

The commit SHA shown above is illustrative. Replace it with the real full commit SHA from your GitHub workflow.

We have not executed an actual Docker build or ECR push in this conversation. The commands below are the implementation steps, not claims of successful execution.

## 3. ECR repository architecture for 21 microservices

We have 21 microservices in our e-commerce application.

We need a consistent repository naming convention.

### 3.1 Repository naming

We will use one ECR repository per microservice.

Example:

```
company-ecommerce/
│
├── user-service
├── product-service
├── cart-service
├── order-service
├── payment-service
├── inventory-service
├── notification-service
├── shipping-service
├── pricing-service
├── discount-service
├── review-service
├── recommendation-service
├── search-service
├── catalog-service
├── auth-service
├── api-gateway
├── checkout-service
├── invoice-service
├── warehouse-service
├── delivery-service
└── reporting-service
```

Actual ECR repository names:

```
company-ecommerce/user-service
company-ecommerce/product-service
company-ecommerce/cart-service
company-ecommerce/order-service
company-ecommerce/payment-service
company-ecommerce/inventory-service
company-ecommerce/notification-service
...
```

For this chapter, we will work with:

```
company-ecommerce/order-service
```

### 3.2 Why one repository per microservice?

Consider an e-commerce application with 21 microservices.

If all images are stored in one repository, managing access, lifecycle policies and image ownership becomes more complicated.

Separate repositories provide:

- Independent image lifecycle policies.
- Service-specific access control.
- Easier vulnerability management.
- Clear image ownership.
- Easier release tracking.
- Cleaner auditing.
- Independent retention policies.

For example, the payment team can own the payment-service repository while the order team owns the order-service repository.

## 4. Repository structure

We will manage ECR infrastructure through Terraform.

Our infrastructure repository:

```
infrastructure/
│
├── terraform/
│   │
│   ├── modules/
│   │   │
│   │   └── ecr/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── versions.tf
│   │
│   ├── environments/
│   │   │
│   │   ├── dev/
│   │   │   └── ecr.tf
│   │   │
│   │   ├── qa/
│   │   │   └── ecr.tf
│   │   │
│   │   ├── staging/
│   │   │   └── ecr.tf
│   │   │
│   │   └── prod/
│   │       └── ecr.tf
│   │
│   └── README.md
│
└── .github/
    └── workflows/
        └── terraform-ecr.yml
```

Why use a reusable Terraform module?

Instead of writing 21 separate ECR configurations, we create one reusable module and call it for each microservice.

This provides consistent configuration and reduces duplication.

## 5. Create the ECR Terraform module

### 5.1 versions.tf

Location:

```
infrastructure/terraform/modules/ecr/versions.tf
```

```
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

The provider version is an example constraint. Pin and validate the provider version approved by your organization, and commit the dependency lock file.

### 5.2 variables.tf

Location:

```
infrastructure/terraform/modules/ecr/variables.tf
```

```
variable "repository_name" {
  description = "ECR repository name"
  type        = string
}

variable "image_tag_mutability" {
  description = "Image tag mutability setting"
  type        = string
  default     = "IMMUTABLE"

  validation {
    condition = contains(
      ["IMMUTABLE", "MUTABLE"],
      var.image_tag_mutability
    )

    error_message = "Use IMMUTABLE or MUTABLE."
  }
}

variable "scan_on_push" {
  description = "Enable ECR scan on push"
  type        = bool
  default     = true
}

variable "kms_key_arn" {
  description = "Optional customer-managed KMS key ARN"
  type        = string
  default     = null
}

variable "max_image_count" {
  description = "Maximum number of tagged images to retain"
  type        = number
  default     = 50
}

variable "untagged_image_days" {
  description = "Days to retain untagged images"
  type        = number
  default     = 7
}

variable "tags" {
  description = "Tags applied to the ECR repository"
  type        = map(string)
  default     = {}
}
```

### 5.3 main.tf

Location:

```
infrastructure/terraform/modules/ecr/main.tf
```

```
resource "aws_ecr_repository" "this" {
  name                 = var.repository_name
  image_tag_mutability = var.image_tag_mutability

  image_scanning_configuration {
    scan_on_push = var.scan_on_push
  }

  encryption_configuration {
    encryption_type = var.kms_key_arn == null ? "AES256" : "KMS"

    kms_key = var.kms_key_arn
  }

  tags = var.tags
}

resource "aws_ecr_lifecycle_policy" "this" {
  repository = aws_ecr_repository.this.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1

        description = "Remove old untagged images after configured retention"

        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = var.untagged_image_days
        }

        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2

        description = "Retain the most recent tagged images"

        selection = {
          tagStatus     = "tagged"
          tagPatternList = ["*"]
          countType     = "imageCountMoreThan"
          countNumber   = var.max_image_count
        }

        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

Important lifecycle consideration: This example is a starting policy. The tagged-image retention rule must be reviewed against your release and rollback requirements. For example, if production needs the previous 10 release images available for emergency rollback, your retention policy must preserve them. Also verify lifecycle rule behavior and supported policy syntax against the AWS provider and ECR API version used in your environment.

### 5.4 outputs.tf

Location:

```
infrastructure/terraform/modules/ecr/outputs.tf
```

```
output "repository_name" {
  value = aws_ecr_repository.this.name
}

output "repository_url" {
  value = aws_ecr_repository.this.repository_url
}

output "registry_id" {
  value = aws_ecr_repository.this.registry_id
}

output "repository_arn" {
  value = aws_ecr_repository.this.arn
}
```

These outputs provide the repository URL and identifiers needed by publishing workflows.

## 6. Create the order-service ECR repository

Location:

```
infrastructure/terraform/environments/prod/ecr.tf
```

```
module "order_service_ecr" {
  source = "../../modules/ecr"

  repository_name = "company-ecommerce/order-service"

  image_tag_mutability = "IMMUTABLE"

  scan_on_push = true

  max_image_count = 50

  untagged_image_days = 7

  tags = {
    Project     = "company-ecommerce"
    Service     = "order-service"
    Environment = "shared"
    ManagedBy   = "terraform"
    Owner       = "platform-team"
  }
}
```

The repository can be shared across deployment environments because the same image artifact is promoted through DEV, QA, STAGING and PROD.

Environment-specific access controls and deployment configuration are managed separately.

### 6.1 Why immutable tags?

Suppose we publish:

```
order-service:d4e5f6a
```

Later, someone attempts to push another image using the same tag.

With immutable tags enabled, ECR rejects the overwrite.

This prevents an existing release tag from silently pointing to a different image.

Production recommendation: Use a full commit SHA as the tag, not a shortened SHA, and deploy using the image digest wherever possible.

### 6.2 Why scan on push?

When an image is pushed to ECR, ECR scanning can inspect it for known vulnerabilities.

However:

- ECR scanning does not replace Trivy.
- Trivy can scan the local image before publishing.
- ECR scanning provides registry-side vulnerability visibility.
- Security teams can monitor image findings after publication.

For production, choose the ECR scanning mode and coverage deliberately. Basic scanning and enhanced scanning have different capabilities and coverage. Do not assume `scan_on_push` configures every enhanced scanning feature.

## 7. Deploy the Terraform configuration

Move to the production infrastructure directory.

```
cd infrastructure/terraform/environments/prod
```

Initialize Terraform:

```
terraform init
```

Validate the configuration:

```
terraform validate
```

Review the proposed infrastructure changes:

```
terraform plan -out=ecr.tfplan
```

Review the plan carefully.

Expected resource:

```
aws_ecr_repository.this
aws_ecr_lifecycle_policy.this
```

Because these resources are created through a module, Terraform may display their full module paths.

Apply:

```
terraform apply ecr.tfplan
```

Retrieve the repository URL:

```
terraform output
```

Example output format:

```
repository_url =
"123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service"
```

This account ID is illustrative.

### 7.1 Verify repository creation

```
aws ecr describe-repositories \
  --repository-names company-ecommerce/order-service \
  --region ap-south-1
```

Check the repository properties:

```
aws ecr describe-repositories \
  --repository-names company-ecommerce/order-service \
  --region ap-south-1 \
  --query 'repositories[0].{
    Name:repositoryName,
    URI:repositoryUri,
    ARN:repositoryArn,
    Mutability:imageTagMutability,
    Encryption:encryptionConfiguration
  }'
```

Expected configuration:

```
Name: company-ecommerce/order-service

Mutability: IMMUTABLE

Encryption:
  AES256
```

If you configured a customer-managed KMS key, the encryption result should show KMS instead.

## 8. Configure ECR access using AWS IAM

Creating the repository and pushing an image are two separate operations.

We need to understand two types of permissions:

1. IAM identity policy.
2. ECR repository policy.

### 8.1 IAM identity policy

The GitHub Actions publishing role needs permissions to authenticate and push images.

Create:

```
infrastructure/terraform/iam/ecr-publish-policy.json
```

Example policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuthentication",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PushOrderServiceImage",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/company-ecommerce/order-service"
    }
  ]
}
```

Replace the account ID with the actual AWS account ID.

### 8.2 Understand the permissions

| Permission                        | Purpose                            |
| --------------------------------- | ---------------------------------- |
| `ecr:GetAuthorizationToken`       | Obtain an ECR authentication token |
| `ecr:BatchCheckLayerAvailability` | Check whether image layers exist   |
| `ecr:InitiateLayerUpload`         | Start uploading a layer            |
| `ecr:UploadLayerPart`             | Upload layer parts                 |
| `ecr:CompleteLayerUpload`         | Complete layer upload              |
| `ecr:PutImage`                    | Publish image manifest and tag     |
| `ecr:BatchGetImage`               | Retrieve image manifests           |

The authentication token action requires `Resource: "*"` in this policy.

The image publishing actions are restricted to the order-service repository.

Important: Do not give the publishing role broad `AdministratorAccess` or unrestricted ECR permissions.

### 8.3 Why do we need a repository policy?

An IAM identity policy grants permissions to an AWS identity.

An ECR repository policy grants or restricts access at the repository level, particularly useful for cross-account access.

For the same-account publishing workflow, an appropriate IAM identity policy is normally sufficient, subject to applicable explicit denies, permission boundaries and organization controls.

For cross-account publishing, the destination repository may need a repository policy that trusts the publishing role in the source account.

## 9. Configure GitHub OIDC authentication

We should not store long-lived AWS access keys in GitHub repository secrets.

Instead, GitHub Actions uses OpenID Connect to request temporary AWS credentials.

### 9.1 Why OIDC?

Traditional approach:

```
GitHub Secrets
    |
    v
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
    |
    v
AWS Authentication
```

Problem:

- Long-lived credentials.
- Rotation requirements.
- Risk of accidental exposure.
- Difficult credential lifecycle management.

OIDC approach:

```
GitHub Actions
    |
    v
GitHub OIDC Token
    |
    v
AWS STS AssumeRoleWithWebIdentity
    |
    v
Temporary AWS Credentials
    |
    v
Amazon ECR
```

Benefits:

- No long-lived AWS access keys.
- Temporary credentials.
- Trust restrictions based on repository and branch or environment.
- Better auditability.
- Smaller credential exposure window.

### 9.2 Create the GitHub OIDC provider

First, check whether the AWS account already has a GitHub OIDC provider.

```
aws iam list-open-id-connect-providers
```

The standard GitHub issuer is:

```
https://token.actions.githubusercontent.com
```

If it does not exist, create it using your approved Terraform IAM module or AWS IAM administration process.

The provider must use the correct audience:

```
sts.amazonaws.com
```

Avoid creating duplicate OIDC providers.

### 9.3 Create the trust policy

Create:

```
infrastructure/terraform/iam/github-ecr-trust-policy.json
```

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPOSITORY:environment:prod-publish"
        }
      }
    }
  ]
}
```

Replace:

```
123456789012
YOUR_ORG
YOUR_REPOSITORY
```

with your real values.

The GitHub repository and environment must match exactly.

This example trusts a specific GitHub Environment named:

```
prod-publish
```

You can use a branch-based subject instead if that is the chosen organizational trust model. Do not use a wildcard trust condition unnecessarily.

### 9.4 Create the IAM role

Example Terraform:

```
resource "aws_iam_role" "github_ecr_publish" {
  name = "github-order-service-ecr-publish"

  assume_role_policy = file(
    "${path.module}/github-ecr-trust-policy.json"
  )
}

resource "aws_iam_role_policy" "ecr_publish" {
  name = "order-service-ecr-publish"

  role = aws_iam_role.github_ecr_publish.id

  policy = file(
    "${path.module}/ecr-publish-policy.json"
  )
}
```

In a production Terraform repository, make sure the JSON policy file paths resolve correctly from the module and environment where this configuration is declared.

### 9.5 Verify the role

```
aws iam get-role \
  --role-name github-order-service-ecr-publish
```

Record the role ARN.

Example:

```
arn:aws:iam::123456789012:role/github-order-service-ecr-publish
```

## 10. Configure GitHub repository settings

Navigate to the GitHub repository:

```
order-service
```

Open:

```
Settings
   |
   v
Environments
   |
   v
New environment
```

Create:

```
prod-publish
```

Configure:

- Deployment protection rules as required.
- Required reviewers where appropriate.
- Allowed deployment branches.
- Environment-scoped variables.

Add an environment variable:

```
AWS_ROLE_ARN
```

Example:

```
arn:aws:iam::123456789012:role/github-order-service-ecr-publish
```

Other configuration:

```
AWS_REGION=ap-south-1
ECR_REPOSITORY=company-ecommerce/order-service
```

The role ARN is not a secret, but it should still be managed as a controlled configuration value.

Do not place AWS access keys in GitHub secrets.

Important: GitHub Environment protection is an additional deployment control. It is not a replacement for AWS IAM trust restrictions or the organization's production change approval process.

## 11. Critical step: transfer the image from Chapter 197

We must not rebuild the image in Chapter 198.

This is important.

Imagine the build job runs on one GitHub Actions runner, and the publishing job runs on another runner.

The second runner does not automatically have the Docker image built by the first runner.

We need an explicit artifact handoff.

### 11.1 Build job output

In Chapter 197, the build job produced:

```
order-service:<FULL_COMMIT_SHA>
```

The build job should save the image as an archive.

Example:

```
docker save \
  order-service:${GITHUB_SHA} \
  -o order-service-image.tar
```

Generate the checksum:

```
sha256sum order-service-image.tar \
  > order-service-image.tar.sha256
```

Now the build job has:

```
order-service-image.tar
order-service-image.tar.sha256
```

Upload these files as a GitHub Actions artifact.

Example:

```
- name: Upload Docker image artifact
  uses: actions/upload-artifact@v4
  with:
    name: order-service-image
    path: |
      order-service-image.tar
      order-service-image.tar.sha256
    if-no-files-found: error
    retention-days: 3
```

For production, pin this action to a verified immutable commit SHA.

### 11.2 Why save the Docker image?

The image archive allows the publishing job to load the exact image created by the build job.

Without this handoff, the publishing job may attempt to rebuild the image.

That would violate our build-once-promote-many principle.

### 11.3 Download the image artifact

In the publishing job:

```
- name: Download Docker image artifact
  uses: actions/download-artifact@v4
  with:
    name: order-service-image
    path: ./image-artifact
```

### 11.4 Verify the checksum

```
cd image-artifact

sha256sum -c order-service-image.tar.sha256
```

Expected result:

```
order-service-image.tar: OK
```

If the checksum verification fails, stop the workflow.

Never publish an image archive whose integrity cannot be verified.

Additional production requirement: A checksum detects accidental corruption, but it does not independently establish trusted provenance. The build artifact should also be associated with a trusted workflow run, source commit, repository and security evidence. Restrict which workflow runs can supply artifacts to the publishing job.

## 12. Load the image into Docker

After downloading and verifying the archive:

```
docker load \
  -i image-artifact/order-service-image.tar
```

List local images:

```
docker images
```

Expected image identity:

```
order-service:<FULL_COMMIT_SHA>
```

Verify the image:

```
docker image inspect \
  order-service:${GITHUB_SHA}
```

Check the image ID:

```
docker image inspect \
  order-service:${GITHUB_SHA} \
  --format '{{.Id}}'
```

Check the image labels:

```
docker image inspect \
  order-service:${GITHUB_SHA} \
  --format '{{json .Config.Labels}}'
```

Validate that the image corresponds to the expected source commit.

Do not rely only on the tag name; confirm its build metadata and artifact provenance.

## 13. Authenticate Docker with ECR

Now we need to authenticate Docker with Amazon ECR.

First, retrieve the AWS account ID:

```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Set the registry URL:

```
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com"
```

Authenticate:

```
aws ecr get-login-password \
  --region ap-south-1 \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"
```

Expected result:

```
Login Succeeded
```

### 13.1 Understand the command

`aws ecr get-login-password` retrieves an authentication password.

The password is passed directly to Docker through standard input.

This avoids placing the password directly in the shell command line.

The username for ECR Docker authentication is:

```
AWS
```

### 13.2 Verify the AWS identity

Before pushing, check the current identity:

```
aws sts get-caller-identity
```

Verify:

- AWS account ID.
- Assumed role identity.
- Expected AWS account.
- Expected publishing role.

If the identity is wrong, stop the publishing workflow.

## 14. Tag the image for ECR

Our local image:

```
order-service:<FULL_COMMIT_SHA>
```

Our ECR repository:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
```

Set variables:

```
IMAGE_TAG="${GITHUB_SHA}"

ECR_REPOSITORY="company-ecommerce/order-service"

ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
```

Tag the image:

```
docker tag \
  "order-service:${IMAGE_TAG}" \
  "${ECR_IMAGE}"
```

Verify:

```
docker images
```

Expected:

```
REPOSITORY                                                        TAG
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service   <FULL_COMMIT_SHA>
```

The account ID and commit SHA are placeholders.

### Why do we use the commit SHA?

Imagine three releases:

```
Commit A -> Image A
Commit B -> Image B
Commit C -> Image C
```

Each image has a different identity.

This provides:

- Release traceability.
- Easier rollback.
- Reliable promotion.
- Immutable release references.
- Clear source-to-image mapping.

Avoid:

```
order-service:latest
```

`latest` does not reliably identify which source revision is deployed.

## 15. Push the image to ECR

Now execute:

```
docker push "$ECR_IMAGE"
```

Docker uploads the image layers and manifest.

Example output:

```
The push refers to repository
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service

Layer already exists
Pushed
Pushed

d4e5f6a...: digest:
sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The digest shown here is illustrative.

The actual digest is returned by the real push operation.

### 15.1 What happens internally?

```
Docker Client
    |
    v
ECR Authentication
    |
    v
Check Existing Layers
    |
    v
Upload Missing Layers
    |
    v
Upload Image Manifest
    |
    v
ECR Stores Image
    |
    v
Image Tag + Digest
```

Docker can reuse layers that are already present in the registry.

This can reduce upload time when multiple images share common base layers.

## 16. Capture the image digest

This is one of the most important steps in the chapter.

A tag identifies an image by a human-readable name.

A digest identifies the exact image manifest content.

Example:

```
Tag:

d4e5f6a...
```

```
Digest:

sha256:1234567890abcdef...
```

We should use the digest for production deployment.

### 16.1 Retrieve the digest

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region ap-south-1
```

Extract only the digest:

```
IMAGE_DIGEST=$(aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text)
```

Validate:

```
echo "$IMAGE_DIGEST"
```

Example format:

```
sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 16.2 Build the immutable image reference

```
IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}@${IMAGE_DIGEST}"
```

Example:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service@sha256:xxxxxxxx
```

This is the image reference we will eventually promote through GitOps.

### 16.3 Why use digest instead of tag?

Suppose someone publishes a new image with another tag.

If your deployment uses a mutable tag, the image selected by that tag can change.

A digest is content-addressed and identifies a specific manifest.

For production:

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:xxxxxxxx
```

We will use this reference in Chapter 199.

## 17. Verify the image in ECR

Check the repository:

```
aws ecr describe-repositories \
  --repository-names company-ecommerce/order-service \
  --region ap-south-1
```

Check the published image:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --region ap-south-1
```

Check only the required tag:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region ap-south-1 \
  --query 'imageDetails[0].{
    Digest:imageDigest,
    Tags:imageTags,
    PushedAt:imagePushedAt,
    Size:imageSizeInBytes
  }'
```

Confirm:

- Repository name is correct.
- Image tag is the expected commit SHA.
- Digest is available.
- Push timestamp is recorded.
- Image size is reasonable.

### 17.1 Verify the remote image

You can pull the image by digest:

```
docker pull "$IMAGE_URI"
```

Inspect it:

```
docker image inspect "$IMAGE_URI"
```

This provides an additional check that the published image can be retrieved using the recorded digest.

## 18. ECR vulnerability scanning

We have two different security scanning stages.

### 18.1 Trivy scanning before publishing

Trivy scans the image before it is pushed.

Example:

```
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "order-service:${IMAGE_TAG}"
```

This checks the local image.

Depending on the configured policy, the workflow can fail if vulnerabilities exceed the organization's accepted threshold.

### 18.2 ECR scanning after publishing

ECR performs registry-side image scanning.

Check scan findings:

```
aws ecr describe-image-scan-findings \
  --repository-name company-ecommerce/order-service \
  --image-id imageTag="${IMAGE_TAG}" \
  --region ap-south-1
```

The command may return a scan-not-found or scan-in-progress response depending on the scanning mode and timing.

Do not assume a scan is complete immediately after push.

### 18.3 Understand the difference

| Feature             | Trivy                                  | ECR scanning                                                      |
| ------------------- | -------------------------------------- | ----------------------------------------------------------------- |
| Scan target         | Local image or other configured target | Registry image                                                    |
| Execution           | CI pipeline or other approved scanner  | AWS-managed scanning                                              |
| Timing              | Before publishing, if configured       | After registry ingestion or according to configured scanning mode |
| CI blocking         | Yes, when policy is configured         | Requires explicit workflow integration                            |
| Registry visibility | Separate reports                       | ECR findings                                                      |
| Purpose             | Pre-publication security gate          | Registry-side vulnerability visibility                            |

Production rule: A successful push does not automatically mean the image has passed every security policy.

The release pipeline must explicitly decide whether ECR findings must be checked before promotion.

## 19. Complete GitHub Actions ECR publishing workflow

Now we will connect the steps.

The publishing workflow must:

1. Download the exact image archive.
2. Verify its checksum.
3. Load the image.
4. Authenticate with AWS using OIDC.
5. Authenticate Docker with ECR.
6. Tag the existing image.
7. Push the image.
8. Retrieve the digest.
9. Record the image reference.
10. Publish the digest as a workflow output.

This example assumes the image artifact has already been uploaded by the trusted build workflow and is available to the publishing workflow.

### 19.1 Workflow file

Location:

```
order-service/.github/workflows/publish-ecr.yml
```

```
name: Publish Order Service Image to ECR

on:
  workflow_dispatch:

  workflow_run:
    workflows:
      - Order Service CI
    types:
      - completed

permissions:
  contents: read
  id-token: write
  actions: read

jobs:

  publish:

    name: Publish Image to ECR

    # Only publish after the trusted CI workflow succeeds.
    if: >
      github.event_name == 'workflow_dispatch' ||
      github.event.workflow_run.conclusion == 'success'

    runs-on: ubuntu-latest

    environment: prod-publish

    outputs:
      image_digest: ${{ steps.image.outputs.digest }}
      image_uri: ${{ steps.image.outputs.uri }}

    env:
      AWS_REGION: ap-south-1
      ECR_REPOSITORY: company-ecommerce/order-service

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Validate source commit
        shell: bash
        run: |
          set -euo pipefail

          echo "Publishing workflow started"

          if [ "${GITHUB_EVENT_NAME}" = "workflow_run" ]; then
            echo "Source commit: ${{ github.event.workflow_run.head_sha }}"
          else
            echo "Manual workflow execution"
          fi

      - name: Download Docker image artifact
        uses: actions/download-artifact@v4
        with:
          name: order-service-image
          path: ./image-artifact
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Verify Docker image artifact
        shell: bash
        run: |
          set -euo pipefail

          cd image-artifact

          sha256sum -c order-service-image.tar.sha256

      - name: Load Docker image
        shell: bash
        run: |
          set -euo pipefail

          docker load \
            -i image-artifact/order-service-image.tar

      - name: Set image tag
        id: metadata
        shell: bash
        run: |
          set -euo pipefail

          if [ "${GITHUB_EVENT_NAME}" = "workflow_run" ]; then
            IMAGE_TAG="${{ github.event.workflow_run.head_sha }}"
          else
            IMAGE_TAG="${GITHUB_SHA}"
          fi

          echo "image_tag=${IMAGE_TAG}" >> "$GITHUB_OUTPUT"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        shell: bash
        run: |
          set -euo pipefail

          aws sts get-caller-identity

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Tag and push image
        shell: bash
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ steps.metadata.outputs.image_tag }}
        run: |
          set -euo pipefail

          SOURCE_IMAGE="order-service:${IMAGE_TAG}"

          ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

          docker image inspect "$SOURCE_IMAGE" >/dev/null

          docker tag "$SOURCE_IMAGE" "$ECR_IMAGE"

          docker push "$ECR_IMAGE"

      - name: Capture image digest
        id: image
        shell: bash
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ steps.metadata.outputs.image_tag }}
        run: |
          set -euo pipefail

          IMAGE_DIGEST=$(aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="$IMAGE_TAG" \
            --region "$AWS_REGION" \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          if [ -z "$IMAGE_DIGEST" ] || [ "$IMAGE_DIGEST" = "None" ]; then
            echo "Unable to retrieve image digest"
            exit 1
          fi

          IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}@${IMAGE_DIGEST}"

          echo "digest=${IMAGE_DIGEST}" >> "$GITHUB_OUTPUT"

          echo "uri=${IMAGE_URI}" >> "$GITHUB_OUTPUT"

          echo "Published image: ${IMAGE_URI}"

      - name: Verify image publication
        shell: bash
        env:
          IMAGE_TAG: ${{ steps.metadata.outputs.image_tag }}
        run: |
          set -euo pipefail

          aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="$IMAGE_TAG" \
            --region "$AWS_REGION"

      - name: Publish image metadata
        shell: bash
        run: |
          set -euo pipefail

          {
            echo "### ECR Image Published"
            echo ""
            echo "- Service: order-service"
            echo "- Source commit: ${{ steps.metadata.outputs.image_tag }}"
            echo "- Image digest: ${{ steps.image.outputs.digest }}"
            echo "- Image URI: ${{ steps.image.outputs.uri }}"
          } >> "$GITHUB_STEP_SUMMARY"
```

### 19.2 Important production limitations in this example

This workflow illustrates the publishing stages, but it must be hardened before use.

First: action pinning.

The example uses readable action tags such as:

```
uses: actions/checkout@v4
```

Production workflows should pin each third-party action to a reviewed, immutable commit SHA and use Dependabot or an equivalent process to manage updates.

Second: artifact trust.

The `workflow_run` event can involve a different workflow run and security context. Do not download an arbitrary artifact merely because it has the expected name.

Validate:

- The triggering workflow ID or expected workflow identity.
- The source repository.
- The source branch.
- The source commit.
- The successful conclusion.
- The artifact run ID.
- The artifact provenance.

The publishing job must only consume artifacts produced by the trusted build workflow.

Third: manual publishing.

The example includes `workflow_dispatch` for demonstration. In a production implementation, constrain manual publishing to approved branches and verify that the artifact is from the exact approved source commit. A manual trigger must not bypass the security and release gates.

Fourth: GitHub token permissions.

The artifact download token must have only the permissions needed to retrieve the trusted run artifact. Do not grant broad write access.

Fifth: environment approval.

The `prod-publish` environment can enforce a reviewer gate. If your process requires publishing before production approval, use a separate controlled publishing environment rather than treating ECR publishing as production deployment approval.

Sixth: duplicate publication.

Because tags are immutable, retry behavior needs to handle the case where an earlier push succeeded but the workflow failed before recording the digest. On retry, check whether the tag already exists. If it exists, verify that its digest matches the expected artifact; do not attempt to overwrite it.

## 20. Handle immutable-tag publishing safely

An immutable repository rejects attempts to overwrite an existing tag.

Imagine this situation:

```
First workflow run:

order-service:d4e5f6a
       |
       v
Push successful
       |
       v
Workflow runner fails before digest recording
```

The workflow is restarted.

It tries to push the same tag again.

The correct behavior is:

1. Check whether the tag already exists.
2. Retrieve its digest.
3. Compare the existing digest with the expected image digest.
4. If they match, treat publication as already complete.
5. If they do not match, fail and investigate.

Do not disable immutability to make a retry work.

Example check:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

If the tag is not found, the command returns an error.

Your publishing script should handle that specific not-found case separately from authentication, permission and network failures.

## 21. ECR access for EKS workloads

Publishing and pulling images are different operations.

The GitHub Actions publishing role pushes images.

The EKS worker nodes or image-pulling identity need permission to pull images.

Architecture:

```
GitHub Actions
      |
      | Push
      v
     ECR
      |
      | Pull
      v
   Amazon EKS
      |
      v
   Kubernetes Pod
```

### 21.1 Required pull permissions

The identity used by EKS to pull the image needs the relevant ECR read permissions.

Typical permissions include:

```
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
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/company-ecommerce/order-service"
    }
  ]
}
```

In EKS, image pulls are normally performed by the node's container runtime using the node's AWS identity and ECR credential mechanisms. Do not assume that an application ServiceAccount or IRSA role automatically provides image-pull permissions.

For a standard EKS configuration, ensure the appropriate worker-node role has the required ECR pull access. If using cross-account repositories, configure the repository policy as well.

### 21.2 Verify node permissions

For an EKS managed node group, inspect its node IAM role and confirm that it has the required pull permissions.

A commonly used AWS managed policy is:

```
AmazonEC2ContainerRegistryPullOnly
```

Use the least-privilege policy appropriate to your EKS configuration.

Do not attach the publishing role to EKS nodes.

Publishing and pulling should use separate identities.

## 22. Cross-account ECR publishing

Some organizations separate AWS environments into different accounts.

Example:

```
Development Account
        |
        v
Build and Security
        |
        v
Production Account
        |
        v
Production ECR
```

In this architecture:

- GitHub Actions authenticates into the approved publishing account.
- The destination ECR repository trusts the approved source identity.
- The source role receives only the permissions required to publish.
- Production EKS receives read access.
- The destination account maintains ownership of production images.

A repository policy can grant access to an external AWS role.

Example concept:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApprovedPublisher",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/github-ecr-publisher"
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage"
      ]
    }
  ]
}
```

The account IDs and role names are illustrative.

The publisher still needs its own identity permissions, and `ecr:GetAuthorizationToken` must be granted through an identity policy.

Do not introduce cross-account complexity if your organization uses a single AWS account for the relevant workflow.

## 23. Image retention and cleanup

ECR storage costs increase as more images are published.

We should define a retention policy.

Example:

| Image category               | Retention strategy                        |
| ---------------------------- | ----------------------------------------- |
| Untagged images              | Remove after an approved retention period |
| Development builds           | Keep a limited number                     |
| QA images                    | Retain according to testing needs         |
| Staging releases             | Preserve release candidates               |
| Production releases          | Preserve rollback-required versions       |
| Compliance-required releases | Retain according to organizational policy |

### 23.1 Why are untagged images created?

When an image manifest is replaced or removed, some layers may no longer be referenced by a tagged image.

ECR can contain untagged images.

Lifecycle policies help clean these up.

### 23.2 Inspect images

```
aws ecr list-images \
  --repository-name company-ecommerce/order-service \
  --region ap-south-1
```

### 23.3 Inspect lifecycle policy

```
aws ecr get-lifecycle-policy \
  --repository-name company-ecommerce/order-service \
  --region ap-south-1
```

### 23.4 Test lifecycle policy

Before relying on a lifecycle policy, use the ECR lifecycle policy preview capability and review which images are eligible for expiration.

Never introduce a retention policy without considering:

- Active production releases.
- Previous known-good releases.
- Rollback requirements.
- Incident investigation.
- Audit retention.
- Release frequency.

## 24. ECR monitoring and auditing

ECR publishing should be auditable.

We should know:

- Who published the image.
- Which GitHub repository triggered publishing.
- Which source commit was used.
- Which IAM role was assumed.
- When the image was pushed.
- What image digest was published.
- Whether scanning reported vulnerabilities.
- Which GitOps revision later deployed the image.

### 24.1 CloudTrail

AWS CloudTrail can provide evidence of relevant AWS API activity.

Examples of operations to investigate include:

```
PutImage
InitiateLayerUpload
UploadLayerPart
CompleteLayerUpload
BatchGetImage
DescribeImages
```

CloudTrail event availability and details depend on the API and event configuration.

Use the organization's approved CloudTrail configuration and retention policy.

### 24.2 ECR findings

Security teams should review ECR image scan findings.

Do not treat a successful push as proof that the image is vulnerability-free.

### 24.3 CI publishing evidence

The GitHub Actions workflow summary should record:

```
Jira ID
Service
Source commit
Build workflow run ID
Publishing workflow run ID
Image tag
ECR repository
Image digest
Publishing time
Security scan status
```

The publishing job should not contain credentials or sensitive values in its logs.

## 25. Common failures and troubleshooting

This section is important because ECR publishing failures are common in real production pipelines.

### Scenario 1 — AccessDeniedException

Error:

```
AccessDeniedException:
User is not authorized to perform ecr:PutImage
```

Possible causes:

- Missing IAM permission.
- Wrong role assumed.
- Repository ARN mismatch.
- Explicit deny in an organization policy.
- Cross-account repository policy missing.

Check the identity:

```
aws sts get-caller-identity
```

Check the role policy:

```
aws iam list-attached-role-policies \
  --role-name github-order-service-ecr-publish
```

For inline policies:

```
aws iam list-role-policies \
  --role-name github-order-service-ecr-publish
```

Fix the actual permission issue. Do not attach administrator access as a workaround.

### Scenario 2 — Cannot assume IAM role

Error:

```
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Possible causes:

- Incorrect OIDC provider.
- Wrong audience.
- Incorrect GitHub subject condition.
- Environment name mismatch.
- Branch not permitted.
- IAM role trust policy mismatch.

Check:

```
aws iam get-role \
  --role-name github-order-service-ecr-publish
```

Review the trust policy and compare it with the actual GitHub OIDC subject.

### Scenario 3 — Repository does not exist

Error:

```
RepositoryNotFoundException
```

Check:

```
aws ecr describe-repositories \
  --region ap-south-1
```

Possible causes:

- Terraform was not applied.
- Wrong repository name.
- Wrong AWS account.
- Wrong AWS region.

### Scenario 4 — Docker login failed

Error:

```
Error response from daemon:
login attempt failed
```

Check:

```
aws sts get-caller-identity
```

Then:

```
aws ecr get-login-password \
  --region ap-south-1 \
  | docker login \
      --username AWS \
      --password-stdin \
      123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Check network connectivity, account ID, registry endpoint, region and IAM permissions.

### Scenario 5 — Image tag already exists

Error:

```
ImageTagAlreadyExistsException
```

Cause:

The repository has immutable tags and the tag already exists.

Correct response:

- Retrieve the existing digest.
- Compare it with the expected image.
- If it is the same artifact, reuse the existing publication.
- If it is different, stop and investigate.

Never change the repository to mutable merely to bypass the failure.

### Scenario 6 — No basic auth credentials

Error:

```
no basic auth credentials
```

Cause:

Docker is not authenticated to the correct ECR registry.

Fix:

```
aws ecr get-login-password \
  --region ap-south-1 \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"
```

### Scenario 7 — Image artifact not found

Error:

```
Unable to find artifact
```

Possible causes:

- Build workflow failed.
- Wrong workflow run ID.
- Artifact name mismatch.
- Artifact expired.
- Publishing workflow cannot access the artifact.
- Artifact was created in a different repository.

Verify the build workflow run and artifact metadata.

Do not rebuild an image automatically to bypass an artifact retrieval failure without recording and validating a new build.

### Scenario 8 — Checksum verification failed

Error:

```
order-service-image.tar: FAILED
```

Cause:

The downloaded archive does not match its recorded checksum.

Possible causes:

- Incomplete upload.
- Corrupted artifact.
- Incorrect checksum file.
- Artifact substitution.

Action:

Stop publishing.

Retrieve the original artifact from the trusted build run and investigate the integrity failure.

### Scenario 9 — ECR scan findings not available

Error:

```
ScanNotFoundException
```

Possible causes:

- Scan has not completed.
- Scanning configuration does not cover the image as expected.
- Wrong image identifier.
- Wrong scanning mode.

Check the repository's scanning configuration and image details.

Do not assume that scan-not-found means no vulnerabilities exist.

### Scenario 10 — Image pull failure in EKS

Kubernetes error:

```
ImagePullBackOff
```

Check:

```
kubectl describe pod <pod-name> -n <namespace>
```

Check:

```
kubectl get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

Common causes:

- Incorrect ECR image URI.
- Incorrect digest.
- Node IAM role lacks pull permissions.
- Cross-account repository policy is missing.
- Network connectivity problem.
- Image has been deleted.
- Incorrect AWS region.

This is a deployment-side problem, not necessarily a publishing-side problem.

## 26. Production security checklist

Before allowing the publishing pipeline to run:

### AWS security

- GitHub OIDC provider configured.
- IAM trust policy restricts repository and environment or approved branch.
- No long-lived AWS access keys stored in GitHub.
- Publishing role has least-privilege ECR permissions.
- ECR repository encryption enabled.
- Immutable tags enabled.
- Repository scanning configuration reviewed.
- Lifecycle policy reviewed against rollback needs.
- Cross-account access restricted where applicable.
- CloudTrail auditing enabled according to organizational policy.

### GitHub Actions security

- Build and publish workflows are separated by a controlled artifact handoff.
- Artifact source run is verified.
- Artifact checksum is verified.
- Actions are pinned to immutable commit SHAs in production.
- `id-token: write` is granted only where needed.
- Publishing environment is protected.
- Manual publishing is restricted.
- No secrets are exposed to untrusted pull-request code.
- Workflow logs do not expose credentials.

### Image security

- Image was built in Chapter 197.
- Image is not rebuilt during publishing.
- Image tag uses the full commit SHA.
- Trivy policy passed.
- Image provenance is recorded.
- ECR image digest captured.
- ECR scan status recorded.
- Image can be pulled by digest.
- Required rollback images are retained.

## 27. Complete publishing execution flow

Here is the complete sequence from the build artifact to ECR.

```
# 1. Downloaded artifact should be available in image-artifact/

# 2. Verify artifact integrity
cd image-artifact

sha256sum -c order-service-image.tar.sha256

# 3. Load exact image
docker load -i order-service-image.tar

# 4. Verify AWS identity
aws sts get-caller-identity

# 5. Set AWS account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

# 6. Configure ECR registry
AWS_REGION="ap-south-1"

ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

ECR_REPOSITORY="company-ecommerce/order-service"

IMAGE_TAG="<FULL_COMMIT_SHA>"

# 7. Authenticate Docker
aws ecr get-login-password \
  --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"

# 8. Tag the existing image
docker tag \
  "order-service:${IMAGE_TAG}" \
  "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

# 9. Push the image
docker push \
  "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

# 10. Retrieve the digest
IMAGE_DIGEST=$(aws ecr describe-images \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$IMAGE_TAG" \
  --region "$AWS_REGION" \
  --query 'imageDetails[0].imageDigest' \
  --output text)

# 11. Build immutable image reference
IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}@${IMAGE_DIGEST}"

# 12. Display published image
echo "$IMAGE_URI"

# 13. Verify image metadata
aws ecr describe-images \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$IMAGE_TAG" \
  --region "$AWS_REGION"
```

Replace `<FULL_COMMIT_SHA>` with the actual full commit SHA. This is a sequence of implementation commands; execute them in a trusted publishing environment with the correct AWS identity.

## 28. How this applies to all 21 microservices

We should not maintain 21 separate copies of the same publishing workflow.

Instead, use reusable workflows and service-specific configuration.

Example service configuration:

```
services:
  order-service:
    repository: company-ecommerce/order-service

  payment-service:
    repository: company-ecommerce/payment-service

  inventory-service:
    repository: company-ecommerce/inventory-service

  user-service:
    repository: company-ecommerce/user-service

  cart-service:
    repository: company-ecommerce/cart-service
```

The actual production configuration should include all 21 service entries.

The reusable publishing workflow receives the repository name and trusted image artifact metadata as inputs.

Example conceptual reusable workflow call:

```
jobs:
  publish:
    uses: YOUR_ORG/platform-golden-path/.github/workflows/ecr-publish.yml@<PINNED_COMMIT_SHA>

    with:
      service_name: order-service
      ecr_repository: company-ecommerce/order-service
      aws_region: ap-south-1

    secrets: inherit
```

For a production workflow, prefer explicitly passing only the required secrets or configuration. Do not use `secrets: inherit` if the called workflow does not need all caller secrets.

The reusable workflow should also validate the source artifact and commit identity before pushing.

Benefits:

- Standardized publishing.
- Centralized security controls.
- Easier maintenance.
- Consistent image tagging.
- Consistent audit evidence.
- Reduced duplication.
- Easier onboarding of new microservices.

## 29. What happens after ECR publishing?

Our image is now available in ECR.

The next question is:

How do we move this exact image through DEV, QA, STAGING and PROD without rebuilding it?

That is the responsibility of GitOps promotion.

The next chapter will use the ECR digest recorded here.

Our flow will be:

```
Source Commit
      |
      v
CI Build
      |
      v
Security Scanning
      |
      v
Docker Image
      |
      v
ECR Publishing
      |
      v
Image Digest
      |
      v
GitOps Repository
      |
      v
DEV
      |
      v
QA
      |
      v
STAGING
      |
      v
Production Approval
      |
      v
PROD
```

We will not rebuild the image between environments.

## 30. Interview questions and practical answers

Q1. What is Amazon ECR?

Amazon Elastic Container Registry is a managed container image registry. We use it to store and distribute Docker images for our EKS workloads.

Q2. How do you authenticate GitHub Actions with AWS?

We configure GitHub OIDC federation. GitHub Actions requests an OIDC token, AWS STS validates the trust relationship, and the workflow receives temporary AWS credentials by assuming an IAM role.

Q3. Why don't you store AWS access keys in GitHub Secrets?

Long-lived access keys create credential management and exposure risks. OIDC provides temporary credentials without requiring permanent AWS keys in the repository.

Q4. How do you secure ECR repositories?

We use encryption, immutable tags, least-privilege IAM permissions, repository policies where required, image scanning, lifecycle policies and auditing.

Q5. What is the difference between an IAM policy and an ECR repository policy?

An IAM policy grants permissions to an identity, such as a role. An ECR repository policy controls access to a particular repository, especially useful for cross-account access.

Q6. Why use immutable image tags?

They prevent an existing tag from being overwritten. This improves release traceability and helps prevent accidental replacement of an image.

Q7. What is the difference between an image tag and an image digest?

A tag is a human-readable image reference. A digest identifies the exact image manifest content. We use digests for reliable production deployment.

Q8. What is the difference between Trivy and ECR scanning?

Trivy can scan an image before publication as part of CI. ECR scanning provides vulnerability findings for images stored in the registry. Both can contribute to the security process.

Q9. How do you deploy the same image across multiple environments?

We build and scan the image once, publish it to ECR, capture its digest, and promote that exact digest through the GitOps environment configurations.

Q10. What happens if an ECR image tag already exists?

Because the repository is immutable, ECR rejects the overwrite. We retrieve the existing digest and compare it with the expected image. If it is the same artifact, we can treat the earlier push as successful; otherwise, we investigate.

Q11. How does EKS pull images from ECR?

The node's container runtime uses the appropriate AWS identity and ECR authentication mechanism to retrieve the image. The node identity must have the necessary pull permissions.

Q12. What happens if ECR publishing succeeds but the workflow fails before recording the digest?

We query ECR using the immutable tag, retrieve the existing digest, verify that it matches the expected artifact and then record the publication evidence. We do not rebuild or overwrite the image.

Q13. Why do you use Terraform for ECR?

Terraform gives us version-controlled, repeatable infrastructure. We can standardize repository creation, encryption, immutability, scanning settings, lifecycle policies and tags across all microservices.

Q14. How do you manage ECR for 21 microservices?

We create one repository per service and use reusable Terraform modules and CI/CD publishing workflows. Each service gets independent repository ownership and lifecycle management.

## 31. Chapter 198 completion checklist

Implementation progress

# 0/20

ECR architecture understood

ECR naming standard defined for 21 microservices

Reusable Terraform ECR module created

Order-service repository configuration created

Encryption and tag immutability configured

Lifecycle policy reviewed

IAM ECR publishing policy created

GitHub OIDC trust relationship configured

GitHub publishing environment configured

Image artifact handoff defined

Image archive checksum verification implemented

Docker authentication to ECR configured

Existing image tagged with full commit SHA

Image pushed to ECR

ECR digest captured

Image pull by digest verified

Trivy and ECR scanning responsibilities understood

EKS image-pull permissions understood

Publishing evidence and audit requirements defined

Troubleshooting and security checklist reviewed

Reset checklist

## 32. Final production state

After completing Chapter 198, our simulated release has the following state:

| Item                    | Status                           |
| ----------------------- | -------------------------------- |
| Jira issue `ECOM-4821`  | Existing                         |
| Source commit           | `d4e5f6a...` illustrative        |
| CI execution            | Covered in Chapter 195           |
| Security scanning       | Chapter 196                      |
| Docker image build      | Chapter 197                      |
| Image archive handoff   | Defined                          |
| ECR repository          | Terraform implementation defined |
| AWS OIDC                | Implementation defined           |
| ECR publishing          | Implementation defined           |
| Actual ECR image digest | Pending real execution           |
| GitOps promotion        | Not started                      |

Do not record the release as published until the actual workflow succeeds and the real ECR digest is captured.

# Next: Chapter 199 — GitOps Promotion

We will implement the next part of the same production release:

```
199-GitOps-Promotion.md
```

It will cover:

- ECR digest retrieval and verification.
- GitOps repository structure.
- DEV, QA, STAGING and PROD environment configuration.
- Helm values updates.
- Automated GitOps pull requests.
- Image promotion without rebuilding.
- GitOps PR review and merge controls.
- ArgoCD Application configuration.
- ArgoCD sync and reconciliation.
- Deployment revision tracking.
- Environment promotion evidence.
- Rollback through Git history.
- Troubleshooting and production scenarios.

The central rule continues: the exact image digest published in Chapter 198 must be the image promoted in every environment.