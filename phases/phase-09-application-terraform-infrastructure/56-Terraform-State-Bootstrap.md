# 56 — Terraform State Bootstrap

> **Project:** Enterprise E-Commerce Platform
> **AWS Region:** `ap-south-1`
> **Purpose:** Create a secure remote Terraform state backend before provisioning the VPC, EKS, RDS, IAM, ECR, and other AWS resources.

---

# 1. Why Terraform State Comes First

Terraform needs to remember what it manages.

For example:

```text
Terraform Code
     │
     ├── VPC
     ├── EKS
     ├── RDS
     ├── ECR
     └── IAM
          │
          ▼
    Terraform State
```

The state tells Terraform things such as:

```text
This VPC already exists.
This subnet has this AWS ID.
This EKS cluster is this cluster.
This RDS instance is this database.
```

Without reliable state management, multiple engineers or CI jobs can accidentally make conflicting changes.

---

# 2. Do Not Store Production State Locally

Avoid:

```text
terraform.tfstate
```

inside Git.

Bad architecture:

```text
Developer Laptop
      │
      ▼
terraform.tfstate
      │
      ▼
GitHub
```

Instead:

```text
Developer / CI
      │
      ▼
Terraform
      │
      ▼
Amazon S3
      │
      ▼
Remote State
```

Our backend will use:

```text
S3
+
Encryption
+
Versioning
+
Native S3 state locking
+
Restricted IAM access
```

---

# 3. Important: Bootstrap Is Special

There is a chicken-and-egg problem:

```text
Terraform
   ↓
needs S3 backend
   ↓
but S3 backend itself needs to exist
```

Therefore, the initial state infrastructure is bootstrapped separately.

The sequence is:

```text
Bootstrap Terraform
       │
       ├── Create S3 bucket
       ├── Enable versioning
       ├── Enable encryption
       └── Configure locking
                │
                ▼
        Remote Terraform State
                │
                ▼
       Normal Terraform workflow
```

After this, normal infrastructure uses the remote backend.

---

# 4. Repository Structure

In:

```text
company-ecommerce/infrastructure
```

create:

```text
infrastructure/
│
├── terraform/
│   ├── bootstrap/
│   │   └── state/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── versions.tf
│   │
│   ├── modules/
│   └── environments/
│       ├── dev/
│       ├── qa/
│       └── prod/
│
├── scripts/
└── README.md
```

---

# 5. State Bucket Naming

Use a globally unique bucket name.

Example:

```text
company-ecommerce-prod-terraform-state
```

However, S3 bucket names are globally unique, so your real organization may need something more unique, such as:

```text
company-ecommerce-ap-south-1-prod-terraform-state-<unique-suffix>
```

Do not blindly reuse a bucket name that already exists.

---

# 6. State Key Structure

Our production backend can use:

```text
eks/terraform.tfstate
```

For example:

```text
s3://company-ecommerce-prod-terraform-state/eks/terraform.tfstate
```

Later, separate environments:

```text
dev/terraform.tfstate
qa/terraform.tfstate
prod/terraform.tfstate
```

or, preferably, separate backend/state boundaries depending on your account/environment architecture.

The important principle is:

> **Do not put unrelated environments into one giant state file.**

---

# 7. Backend Configuration

Our eventual production backend will look like:

```hcl
terraform {
  backend "s3" {
    bucket       = "company-ecommerce-prod-terraform-state"
    key          = "eks/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

`use_lockfile = true` uses the modern S3-native Terraform state locking mechanism.

This is preferable to building a new setup around the older DynamoDB locking approach.

---

# 8. Terraform Version

Create:

```text
terraform/bootstrap/state/versions.tf
```

Example:

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

The exact provider version should be pinned/approved for the project rather than blindly using whatever happens to be newest.

For a real project, record the selected version in the repository and commit the generated lock file.

---

# 9. AWS Provider

Create:

```text
terraform/bootstrap/state/main.tf
```

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "enterprise-ecommerce"
      Environment = "management"
      ManagedBy   = "terraform"
    }
  }
}
```

---

# 10. Variables

Create:

```text
terraform/bootstrap/state/variables.tf
```

```hcl
variable "aws_region" {
  description = "AWS region for Terraform state infrastructure"
  type        = string
  default     = "ap-south-1"
}

variable "state_bucket_name" {
  description = "Globally unique S3 bucket name for Terraform state"
  type        = string
}
```

---

# 11. S3 State Bucket

Add to `main.tf`:

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.state_bucket_name
}
```

This creates the bucket.

But this alone is **not enough** for production.

We need:

```text
Encryption
Versioning
Public access protection
Ownership controls
Locking
```

---

# 12. Block Public Access

Add:

```hcl
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

The Terraform state bucket should never be publicly accessible.

---

# 13. Bucket Ownership

Add:

```hcl
resource "aws_s3_bucket_ownership_controls" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}
```

This simplifies object ownership by disabling legacy ACL-based ownership behavior.

---

# 14. Versioning

Enable versioning:

```hcl
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Why?

Suppose someone accidentally modifies or deletes the current state object.

With versioning:

```text
State v1
State v2
State v3
State v4
```

previous versions can remain available.

This is an important recovery mechanism.

---

# 15. Encryption

Use server-side encryption.

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }

    bucket_key_enabled = true
  }
}
```

For this initial bootstrap we use:

```text
SSE-S3 / AES256
```

Later, organizations with centralized key-management requirements may use:

```text
AWS KMS
```

with a customer-managed KMS key.

---

# 16. Why Terraform State Needs Encryption

Terraform state may contain sensitive infrastructure information.

For example:

```text
resource IDs
network configuration
database configuration
IAM relationships
possibly sensitive resource attributes
```

And depending on how resources are configured, state can contain values that you would not want broadly exposed.

Therefore:

```text
Terraform state
       ↓
Sensitive infrastructure artifact
       ↓
Protect it like a secret
```

---

# 17. State Locking

Modern Terraform can use S3 native locking:

```hcl
use_lockfile = true
```

Example backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "company-ecommerce-prod-terraform-state"
    key          = "eks/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The goal is to prevent concurrent Terraform operations from modifying the same state simultaneously.

Example:

```text
Engineer A
terraform apply
      │
      ▼
   State Lock
      │
      X
Engineer B
terraform apply
```

Engineer B should wait/fail rather than corrupting the shared operation.

---

# 18. Outputs

Create:

```text
terraform/bootstrap/state/outputs.tf
```

```hcl
output "state_bucket_name" {
  description = "Terraform state S3 bucket"
  value       = aws_s3_bucket.terraform_state.bucket
}

output "state_bucket_arn" {
  description = "Terraform state S3 bucket ARN"
  value       = aws_s3_bucket.terraform_state.arn
}
```

---

# 19. Complete Bootstrap `main.tf`

At this point:

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "enterprise-ecommerce"
      Environment = "management"
      ManagedBy   = "terraform"
    }
  }
}


resource "aws_s3_bucket" "terraform_state" {
  bucket = var.state_bucket_name
}


resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}


resource "aws_s3_bucket_ownership_controls" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}


resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}


resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }

    bucket_key_enabled = true
  }
}
```

---

# 20. Bootstrap Variable File

For local bootstrap, create:

```text
terraform/bootstrap/state/terraform.tfvars
```

```hcl
aws_region       = "ap-south-1"
state_bucket_name = "REPLACE-WITH-YOUR-UNIQUE-BUCKET-NAME"
```

Because this file can contain environment-specific information, keep it out of Git if your repository policy requires that.

Your `.gitignore` already excludes:

```text
*.tfvars
```

---

# 21. Initialize Terraform

Move into the bootstrap directory:

```bash
cd terraform/bootstrap/state
```

Initialize:

```bash
terraform init
```

Expected:

```text
Terraform has been successfully initialized!
```

---

# 22. Format

Run:

```bash
terraform fmt -recursive
```

Then:

```bash
terraform fmt -check
```

Expected:

```text
no formatting changes required
```

---

# 23. Validate

Run:

```bash
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

---

# 24. Plan

Run:

```bash
terraform plan \
  -var-file=terraform.tfvars
```

Review the planned resources.

You should see resources similar to:

```text
aws_s3_bucket.terraform_state
aws_s3_bucket_public_access_block.terraform_state
aws_s3_bucket_ownership_controls.terraform_state
aws_s3_bucket_versioning.terraform_state
aws_s3_bucket_server_side_encryption_configuration.terraform_state
```

---

# 25. Apply

Only after reviewing the plan:

```bash
terraform apply \
  -var-file=terraform.tfvars
```

Terraform asks:

```text
Do you want to perform these actions?
```

Enter:

```text
yes
```

---

# 26. Verify the Bucket

AWS CLI:

```bash
aws s3api head-bucket \
  --bucket YOUR_BUCKET_NAME
```

Check versioning:

```bash
aws s3api get-bucket-versioning \
  --bucket YOUR_BUCKET_NAME
```

Expected:

```json
{
  "Status": "Enabled"
}
```

---

# 27. Check Encryption

```bash
aws s3api get-bucket-encryption \
  --bucket YOUR_BUCKET_NAME
```

You should see:

```text
AES256
```

---

# 28. Check Public Access Protection

```bash
aws s3api get-public-access-block \
  --bucket YOUR_BUCKET_NAME
```

Expected values should all be:

```text
true
```

for:

```text
BlockPublicAcls
BlockPublicPolicy
IgnorePublicAcls
RestrictPublicBuckets
```

---

# 29. Important: Don't Put Backend Config in Bootstrap

The bootstrap Terraform itself should **not** use the bucket it is creating as its backend.

Why?

Because:

```text
Bootstrap Terraform
        │
        ▼
creates backend bucket
```

You can't use the backend before creating it.

So:

```text
bootstrap/state/
```

is a special one-time layer.

After the bucket exists:

```text
normal infrastructure
        │
        ▼
remote S3 backend
```

---

# 30. Normal Terraform Environment

Now create:

```text
terraform/environments/dev/
```

For example:

```text
terraform/environments/dev/
│
├── backend.tf
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

---

# 31. `backend.tf`

Example:

```hcl
terraform {
  backend "s3" {
    bucket       = "YOUR_STATE_BUCKET"
    key          = "dev/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

For production:

```text
key = "prod/terraform.tfstate"
```

---

# 32. Why Different State Keys?

Imagine:

```text
dev
qa
prod
```

We don't want:

```text
all environments
       ↓
one state file
```

Instead:

```text
S3 Bucket
│
├── dev/terraform.tfstate
├── qa/terraform.tfstate
└── prod/terraform.tfstate
```

This provides isolation.

For larger organizations, separate AWS accounts and separate state buckets are even better:

```text
Dev Account
   └── Dev State Bucket

QA Account
   └── QA State Bucket

Prod Account
   └── Prod State Bucket
```

---

# 33. Production State Architecture

Our eventual enterprise architecture should look approximately like:

```text
                    AWS Organization
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
    Dev Account        QA Account        Prod Account
        │                  │                  │
        ▼                  ▼                  ▼
   State Bucket       State Bucket       State Bucket
        │                  │                  │
        ▼                  ▼                  ▼
    Terraform          Terraform          Terraform
        │                  │                  │
        ▼                  ▼                  ▼
      AWS                AWS                AWS
```

For the learning project, starting with one AWS account is acceptable.

---

# 34. Terraform State Access

Not every engineer should have unrestricted access to the state bucket.

Think in terms of:

```text
WHO
 ↓
CAN DO WHAT
 ↓
ON WHICH RESOURCE
```

For example:

```text
Terraform CI role
    │
    ├── read/write state
    ├── acquire/release lock
    └── manage infrastructure according to approved role
```

A developer who only needs application logs should not automatically get:

```text
s3:GetObject
s3:PutObject
s3:DeleteObject
```

on the Terraform state bucket.

---

# 35. Terraform State Security

Treat this bucket as highly sensitive.

Controls:

```text
[✓] Private bucket
[✓] Public access blocked
[✓] Encryption
[✓] Versioning
[✓] Locking
[✓] Restricted IAM
[✓] Audit logging
[✓] Separate environments
[✓] No Git commit
```

---

# 36. State Bucket Deletion Protection

This is a subtle but important production concern.

Do not casually allow someone to delete the state bucket.

You can use organizational controls and IAM policies to restrict destructive operations.

For production, consider requiring elevated permissions for:

```text
s3:DeleteBucket
```

and state-object destructive operations.

The state bucket is part of the platform's recovery foundation.

---

# 37. State Recovery Scenario

Imagine:

```text
terraform state
     ↓
accidentally changed
```

Because versioning is enabled:

```text
State version 1
State version 2
State version 3
State version 4
```

you have historical versions available.

This does **not** mean state recovery is automatically safe.

The recovery procedure should be controlled and validated.

---

# 38. State Is Not a Backup of Your Infrastructure

This distinction matters.

Terraform state tells Terraform:

```text
What infrastructure exists
and
how Terraform relates to it
```

It is not a complete backup of:

```text
RDS data
S3 application data
EBS database data
Elasticsearch data
```

For those:

```text
RDS → automated backups/PITR
S3 → versioning/replication
EBS → snapshots where appropriate
Elasticsearch → snapshots
```

We covered these concepts in the DR/backup chapters.

---

# 39. Terraform Workflow

Once remote state is available:

```text
Developer
    │
    ▼
Terraform code
    │
    ▼
terraform fmt
    │
    ▼
terraform validate
    │
    ▼
terraform plan
    │
    ▼
Pull Request
    │
    ▼
Review
    │
    ▼
Approved
    │
    ▼
terraform apply
    │
    ▼
AWS
```

Eventually CI will execute most of this automatically.

---

# 40. Never Blindly Run `terraform apply`

Bad workflow:

```bash
terraform apply -auto-approve
```

on production without reviewing the plan.

Suppose Terraform wants:

```text
- destroy RDS
- replace EKS node group
- delete security group
```

A blind approval could cause a major outage.

Production workflow should be:

```text
Plan
 ↓
Review
 ↓
Approval
 ↓
Apply
```

---

# 41. Plan Artifact

In controlled CI environments, we can create:

```bash
terraform plan \
  -out=tfplan
```

Then inspect:

```bash
terraform show tfplan
```

The approved plan can then be applied:

```bash
terraform apply tfplan
```

This provides stronger separation between:

```text
what was reviewed
```

and:

```text
what was applied
```

The exact CI design will be implemented later.

---

# 42. Terraform Lock File

After:

```bash
terraform init
```

Terraform creates:

```text
.terraform.lock.hcl
```

Commit this file.

Do **not** add it to `.gitignore`.

It records provider selections/checksums and helps make provider installation reproducible.

---

# 43. What Goes Into Git?

Commit:

```text
terraform/
├── modules/
├── environments/
├── bootstrap/
└── *.tf

.terraform.lock.hcl
README.md
```

Do not commit:

```text
.terraform/
terraform.tfstate
terraform.tfstate.*
*.tfvars
*.tfplan
```

unless your organization has a specific, reviewed reason for a particular artifact.

---

# 44. State Backend vs Terraform Modules

These are different concerns.

### Backend

```text
Where Terraform stores state
```

Example:

```text
S3
```

### Module

```text
Reusable infrastructure code
```

Example:

```text
modules/vpc
modules/eks
modules/rds
```

So:

```text
Terraform
│
├── Backend
│     └── S3
│
└── Code
      ├── VPC module
      ├── EKS module
      ├── RDS module
      └── IAM module
```

---

# 45. Why We Build VPC Next

Once state is safe, we can finally provision networking.

Our target VPC:

```text
10.0.0.0/16
```

Three AZs:

```text
ap-south-1a
ap-south-1b
ap-south-1c
```

Public:

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

Private:

```text
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

Architecture:

```text
                    Internet
                       │
                       ▼
                    IGW
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Public A     Public B     Public C
          │            │            │
          │            │            │
        NAT A        NAT B        NAT C
          │            │            │
          ▼            ▼            ▼
       Private A    Private B    Private C
          │            │            │
          ▼            ▼            ▼
        EKS nodes / workloads
```

The private subnets will eventually host the EKS worker capacity.

---

# 46. Why Three AZs?

Production resilience:

```text
AZ-A failure
      ↓
AZ-B + AZ-C
      ↓
service continues
```

Our target:

```text
3 Availability Zones
```

For inexpensive development environments, we can deliberately reduce the architecture later to control cost.

---

# 47. NAT Gateway Cost Warning

The production design may use:

```text
NAT Gateway A
NAT Gateway B
NAT Gateway C
```

for AZ-level resilience.

But NAT gateways cost money.

Therefore:

```text
Production
→ prioritize resilience

Development
→ optimize cost
```

We will explicitly configure the environment differences rather than accidentally deploying a production-sized architecture everywhere.

---

# 48. Current State

We now have:

```text
[✓] GitHub repositories
[✓] Order service
[✓] Local Docker
[✓] PostgreSQL
[✓] Application health
[✓] Local persistence
[✓] Docker networking
[✓] Terraform repository
[✓] Terraform bootstrap design
[✓] Remote S3 state design
```

The next major layer is:

```text
                         AWS
                          │
                          ▼
                    Terraform State
                          │
                          ▼
                         VPC
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
        Public Subnets          Private Subnets
                                      │
                                      ▼
                                     EKS
```

---

# 49. Definition of Done

Before moving to the VPC chapter:

```text
[ ] AWS CLI authenticated
[ ] Terraform installed
[ ] Bootstrap directory created
[ ] S3 state bucket created
[ ] Public access blocked
[ ] Bucket versioning enabled
[ ] Encryption enabled
[ ] S3 native state locking configured
[ ] .terraform.lock.hcl committed
[ ] terraform.tfstate NOT committed
[ ] *.tfvars NOT committed
[ ] Normal environment backend configured
[ ] terraform init works against remote backend
[ ] terraform validate succeeds
[ ] terraform plan succeeds
```

---

# 50. The Infrastructure Flow From Here

We are now moving into the actual AWS build:

```text
56  Terraform State Bootstrap
             │
             ▼
57  Terraform VPC
             │
             ▼
58  Terraform IAM
             │
             ▼
59  Terraform ECR
             │
             ▼
60  Terraform EKS
             │
             ▼
61  EKS Add-ons
             │
             ▼
62  GitHub Actions CI
             │
             ▼
63  Helm Implementation
             │
             ▼
64  GitOps Implementation
             │
             ▼
65  ArgoCD Implementation
             │
             ▼
66  Order Service EKS Deployment
```

**Next: `57-Terraform-VPC.md`** — we'll build the actual AWS VPC with 3 AZs, public/private subnets, route tables, Internet Gateway, NAT gateways, VPC endpoints, DNS settings, Terraform modules, environment variables, validation commands, and the networking decisions needed by EKS.
