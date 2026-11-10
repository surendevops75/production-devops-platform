# 06 — AWS Account & Infrastructure Repository

Now we start the **AWS side of the project**.

Up to now:

```text
Developer
   ↓
GitHub
   ↓
order-service
   ↓
Spring Boot
   ↓
Docker
   ↓
Local PostgreSQL
```

Now we're going to create the infrastructure that will eventually run this application in AWS.

The target is:

```text
                         AWS
                          |
                       Route 53
                          |
                      CloudFront
                          |
                         WAF
                          |
                         ALB
                          |
                    Private EKS
                          |
        +-----------------+-----------------+
        |                 |                 |
   Microservices     Microservices     Microservices
        |
        +---------+---------+---------+
        |         |         |         |
       RDS      Redis      SQS      DynamoDB
```

But **we will not create everything at once**.

We will build the infrastructure layer by layer.

---

# 1. Infrastructure Repository

Create:

```text
infrastructure
```

Our GitHub organization becomes:

```text
company-ecommerce/
│
├── order-service
├── payment-service
├── inventory-service
├── ...
│
├── infrastructure
├── gitops
└── platform
```

The infrastructure repository owns AWS infrastructure.

---

# 2. What Belongs in Infrastructure?

Terraform will eventually manage:

```text
Networking
    |
    +-- VPC
    +-- Subnets
    +-- Route Tables
    +-- Internet Gateway
    +-- NAT Gateway
    +-- Security Groups

Compute
    |
    +-- EKS
    +-- Node Groups

Security
    |
    +-- IAM
    +-- KMS
    +-- Security Groups

Data
    |
    +-- RDS
    +-- Redis
    +-- DynamoDB

Messaging
    |
    +-- SQS
    +-- SNS

Storage
    |
    +-- S3

Container
    |
    +-- ECR

DNS / Edge
    |
    +-- Route 53
    +-- CloudFront
    +-- WAF
```

Later we'll decide which components should be Terraform-managed versus managed by Kubernetes/Helm/ArgoCD.

---

# 3. Repository Structure

Create:

```text
infrastructure/
│
├── terraform/
│   │
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── iam/
│   │   ├── ecr/
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
├── scripts/
│
├── README.md
│
└── .github/
    └── workflows/
        └── terraform.yaml
```

---

# 4. Why Modules?

We don't want:

```text
prod/main.tf
```

with thousands of lines.

Instead:

```text
environment
     |
     +---- VPC module
     |
     +---- EKS module
     |
     +---- RDS module
     |
     +---- Redis module
     |
     +---- ECR module
```

For example:

```hcl
module "vpc" {
  source = "../../modules/vpc"
}
```

This lets us reuse the same infrastructure pattern.

---

# 5. Environment Separation

We'll have:

```text
dev
qa
prod
```

Each environment gets different sizing.

For example:

```text
DEV

EKS nodes:
2

RDS:
small

Redis:
small
```

QA:

```text
EKS nodes:
3

RDS:
medium

Redis:
medium
```

Production:

```text
EKS nodes:
multiple

RDS:
production sizing

Redis:
production sizing
```

The architecture remains similar, but capacity and resilience differ.

---

# 6. Important Rule

Don't do this:

```text
DEV
   |
   v
same production database
```

or:

```text
QA
 |
 v
production Redis
```

Environments should have appropriate isolation.

A simple model:

```text
AWS
│
├── Dev environment
│
├── QA environment
│
└── Production environment
```

In a mature organization these may be separate AWS accounts.

---

# 7. AWS Account Strategy

For learning, you can start with one AWS account and logically separate:

```text
dev
qa
prod
```

But the production-grade target architecture should be:

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
│
├── QA Account
│
└── Production Account
```

Why?

Because account boundaries provide strong isolation.

For this project we'll initially keep the Terraform structure compatible with multi-account AWS, even if you practice using fewer accounts.

---

# 8. AWS Region

We'll use:

```text
ap-south-1
```

for this project.

That is:

```text
Mumbai
```

You can change the region later.

Define it centrally rather than hardcoding it everywhere.

---

# 9. Terraform Version

Create:

```text
terraform/versions.tf
```

Example:

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

The exact provider version should be pinned according to the version you standardize on for the project.

---

# 10. AWS Provider

For development:

```hcl
provider "aws" {
  region = var.aws_region
}
```

Variables:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}
```

---

# 11. Don't Hardcode Credentials

Never do this:

```hcl
provider "aws" {
  access_key = "AKIA..."
  secret_key = "..."
}
```

Instead Terraform should obtain AWS credentials through the normal AWS credential chain.

For local development:

```text
AWS CLI
   |
   v
AWS credentials/profile
   |
   v
Terraform
```

For GitHub Actions later:

```text
GitHub Actions
   |
   v
OIDC
   |
   v
AWS IAM Role
   |
   v
Terraform
```

---

# 12. Local AWS Authentication

Install/configure AWS CLI.

Then:

```bash
aws configure
```

Provide your development credentials/profile as appropriate.

Verify:

```bash
aws sts get-caller-identity
```

Expected output resembles:

```json
{
  "UserId": "...",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/..."
}
```

Do **not** paste real credentials into GitHub, Terraform files, screenshots, or chat.

---

# 13. Terraform Initialization

Move into:

```bash
cd infrastructure/terraform/environments/dev
```

Eventually:

```bash
terraform init
```

Terraform will download:

```text
AWS provider
Terraform modules
provider dependencies
```

---

# 14. Terraform State

This is one of the most important concepts in Terraform.

Terraform maintains state:

```text
Desired configuration
        |
        v
Terraform
        |
        +---- State
        |
        v
AWS infrastructure
```

State allows Terraform to understand:

```text
What I declared
        vs
What already exists
```

---

# 15. Never Keep Production State Only Locally

Bad:

```text
developer laptop
   |
   └── terraform.tfstate
```

Imagine three engineers:

```text
Engineer A
Engineer B
Engineer C
```

All working with different local state files.

That can cause serious problems.

Instead:

```text
Terraform
   |
   v
Amazon S3
   |
   v
Remote state
```

---

# 16. Bootstrap Problem

But there is a circular problem.

We want Terraform to create:

```text
S3 state bucket
```

but Terraform itself needs the bucket for remote state.

Therefore we bootstrap the state infrastructure separately.

Conceptually:

```text
Bootstrap
   |
   +--> S3 state bucket
   |
   +--> state locking mechanism
   |
   +--> encryption
```

Then:

```text
Normal Terraform
       |
       v
Remote state
```

---

# 17. State Bucket

Production-style example:

```text
company-ecommerce-prod-terraform-state
```

with:

```text
S3
 |
 +-- Encryption
 +-- Versioning
 +-- Restricted access
 +-- Lifecycle controls
```

Terraform backend example:

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

`use_lockfile` is the modern S3-native Terraform state locking approach for supported Terraform versions.

---

# 18. Why State Locking?

Imagine:

```text
Engineer A
    |
    v
terraform apply
```

at the exact same time:

```text
Engineer B
    |
    v
terraform apply
```

Without locking, both could modify infrastructure concurrently.

With locking:

```text
Engineer A
    |
    v
STATE LOCKED
    |
    v
terraform apply
    |
    v
STATE UNLOCKED

Engineer B
    |
    v
terraform apply
```

This prevents concurrent state modification.

---

# 19. State Security

Terraform state can contain sensitive information.

Therefore:

```text
S3 bucket
│
├── encryption
├── versioning
├── restrictive IAM
└── no public access
```

Never make the state bucket publicly readable.

---

# 20. State Per Environment

Don't use:

```text
one giant state file
```

for everything.

Instead:

```text
S3
│
├── dev/
│   └── terraform.tfstate
│
├── qa/
│   └── terraform.tfstate
│
└── prod/
    └── terraform.tfstate
```

We can also separate state further by infrastructure domain if the organization requires it.

For example:

```text
prod/
├── network/
├── eks/
├── data/
└── security/
```

The correct granularity depends on operational and team boundaries.

---

# 21. Terraform Environment

Our Dev environment:

```text
terraform/environments/dev/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── backend.tf
└── terraform.tfvars
```

---

# 22. `providers.tf`

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "enterprise-ecommerce"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

This is useful because resources automatically receive common tags.

---

# 23. Variables

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}
```

---

# 24. `terraform.tfvars`

Dev:

```hcl
environment = "dev"
aws_region  = "ap-south-1"
```

QA:

```hcl
environment = "qa"
aws_region  = "ap-south-1"
```

Production:

```hcl
environment = "prod"
aws_region  = "ap-south-1"
```

Don't put passwords or secrets here.

---

# 25. First Terraform Resource

Before EKS, we'll build the **network**.

Why?

Because EKS needs networking.

Our dependency chain is:

```text
VPC
 |
 +-- Subnets
 |
 +-- Route Tables
 |
 +-- Internet Gateway
 |
 +-- NAT
 |
 v
EKS
```

So networking comes first.

---

# 26. VPC Architecture

Our target:

```text
                         VPC
                          |
             +------------+------------+
             |                         |
        Availability Zone A       Availability Zone B
             |                         |
       +-----+-----+             +-----+-----+
       |           |             |           |
    Public      Private       Public      Private
    Subnet      Subnet        Subnet      Subnet
       |           |             |           |
      ALB        EKS Nodes      ALB        EKS Nodes
                   |
                   v
                  Pods
```

We'll eventually use at least three AZs for production resilience.

For the first Dev environment, we'll keep the configuration understandable.

---

# 27. Why Public and Private Subnets?

This is fundamental.

Public subnet:

```text
Route
  |
  v
Internet Gateway
```

Private subnet:

```text
Private subnet
      |
      v
NAT Gateway
      |
      v
Internet Gateway
      |
      v
Internet
```

The EKS worker nodes should normally live in private subnets.

---

# 28. Why EKS Nodes in Private Subnets?

Suppose an EKS node has:

```text
Private IP:
10.0.11.20
```

It doesn't need a public IP.

Outbound traffic:

```text
Pod
 |
 v
Node
 |
 v
NAT Gateway
 |
 v
Internet
```

Inbound application traffic:

```text
Internet
 |
 v
ALB
 |
 v
Private EKS workload
```

This is much better than exposing every worker node directly to the Internet.

---

# 29. CIDR Plan

We'll use:

```text
VPC:
10.0.0.0/16
```

Example:

```text
Public-A:
10.0.1.0/24

Public-B:
10.0.2.0/24

Public-C:
10.0.3.0/24

Private-A:
10.0.11.0/24

Private-B:
10.0.12.0/24

Private-C:
10.0.13.0/24
```

Architecture:

```text
10.0.0.0/16
│
├── 10.0.1.0/24    Public-A
├── 10.0.2.0/24    Public-B
├── 10.0.3.0/24    Public-C
│
├── 10.0.11.0/24   Private-A
├── 10.0.12.0/24   Private-B
└── 10.0.13.0/24   Private-C
```

We'll use three AZs for the production target.

---

# 30. Terraform VPC Module

Create:

```text
terraform/modules/vpc/
│
├── main.tf
├── variables.tf
└── outputs.tf
```

For the project, we can use the AWS-maintained/community VPC module or implement our own module.

For learning Terraform deeply, we'll implement the important resources ourselves first so you understand what Terraform is actually creating.

---

# 31. VPC Resource

```hcl
resource "aws_vpc" "this" {

  cidr_block = var.vpc_cidr

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.name}-vpc"
  }
}
```

---

# 32. Why DNS Settings?

EKS and many AWS services depend on reliable DNS resolution.

Therefore:

```hcl
enable_dns_support   = true
enable_dns_hostnames = true
```

are important for our VPC.

---

# 33. Internet Gateway

Public subnets need Internet Gateway access.

```hcl
resource "aws_internet_gateway" "this" {

  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.name}-igw"
  }
}
```

Architecture:

```text
Public Subnet
      |
      v
Route Table
      |
      v
Internet Gateway
      |
      v
Internet
```

---

# 34. Public Subnet

Example:

```hcl
resource "aws_subnet" "public" {

  count = length(var.public_subnet_cidrs)

  vpc_id = aws_vpc.this.id

  cidr_block = var.public_subnet_cidrs[count.index]

  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "${var.name}-public-${count.index + 1}"
  }
}
```

---

# 35. Private Subnet

```hcl
resource "aws_subnet" "private" {

  count = length(var.private_subnet_cidrs)

  vpc_id = aws_vpc.this.id

  cidr_block = var.private_subnet_cidrs[count.index]

  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.name}-private-${count.index + 1}"
  }
}
```

Notice:

```text
public subnet
    |
    +-- public IP mapping

private subnet
    |
    +-- no public IP mapping
```

---

# 36. Route Tables

Public route table:

```hcl
resource "aws_route_table" "public" {

  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = {
    Name = "${var.name}-public-rt"
  }
}
```

So:

```text
0.0.0.0/0
   |
   v
Internet Gateway
```

---

# 37. Private Routing

Private subnets eventually use NAT.

```text
Private subnet
     |
     v
Private Route Table
     |
     v
NAT Gateway
     |
     v
Internet Gateway
```

This allows:

```text
EKS node
   |
   v
download package/image
```

without giving the node a public IP.

---

# 38. NAT Gateway

Production architecture:

```text
AZ-A
 |
 +-- NAT-A
 |
 +-- Private-A

AZ-B
 |
 +-- NAT-B
 |
 +-- Private-B

AZ-C
 |
 +-- NAT-C
 |
 +-- Private-C
```

This improves resilience.

However, NAT gateways cost money, so our Dev environment can use a smaller architecture if cost is a concern.

This is one of the areas where **production architecture and learning/lab architecture should be distinguished**.

---

# 39. Why Not One NAT Gateway for Production?

Suppose:

```text
AZ-A
 |
 NAT-A

AZ-B
 |
 NAT-A
```

and AZ-A fails.

Then workloads in AZ-B may lose their normal outbound path.

For production:

```text
AZ-A -> NAT-A
AZ-B -> NAT-B
AZ-C -> NAT-C
```

gives better AZ-level resilience.

It costs more, so cost vs resilience must be considered.

---

# 40. VPC Module Variables

```hcl
variable "name" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "availability_zones" {
  type = list(string)
}

variable "public_subnet_cidrs" {
  type = list(string)
}

variable "private_subnet_cidrs" {
  type = list(string)
}
```

---

# 41. Root Module Calling VPC

`environments/dev/main.tf`:

```hcl
module "vpc" {

  source = "../../modules/vpc"

  name = "ecommerce-dev"

  vpc_cidr = "10.0.0.0/16"

  availability_zones = [
    "ap-south-1a",
    "ap-south-1b",
    "ap-south-1c"
  ]

  public_subnet_cidrs = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]

  private_subnet_cidrs = [
    "10.0.11.0/24",
    "10.0.12.0/24",
    "10.0.13.0/24"
  ]
}
```

Now our environment doesn't need to know every implementation detail.

It simply says:

> Create me a VPC with these parameters.

---

# 42. Terraform Workflow

Now the workflow becomes:

```text
Terraform Code
      |
      v
terraform fmt
      |
      v
terraform validate
      |
      v
terraform plan
      |
      v
Review
      |
      v
terraform apply
      |
      v
AWS
```

---

# 43. Format

Run:

```bash
terraform fmt -recursive
```

This standardizes formatting.

---

# 44. Validate

```bash
terraform validate
```

Terraform checks whether the configuration is syntactically and structurally valid.

---

# 45. Plan

```bash
terraform plan
```

This is extremely important.

Terraform tells you:

```text
+ create
~ modify
- destroy
```

For example:

```text
Plan:

+ aws_vpc.this
+ aws_subnet.public[0]
+ aws_subnet.public[1]
+ aws_subnet.public[2]
```

Before applying, you review the plan.

---

# 46. Apply

Only after review:

```bash
terraform apply
```

Terraform asks for confirmation depending on how it is invoked.

Then:

```text
Terraform
   |
   v
AWS API
   |
   v
VPC created
```

---

# 47. Verify in AWS

Use AWS CLI:

```bash
aws ec2 describe-vpcs
```

Then:

```bash
aws ec2 describe-subnets
```

You should see our:

```text
10.0.0.0/16
```

VPC and the configured subnets.

---

# 48. The First Real Infrastructure Dependency Chain

We've now established:

```text
AWS
 |
 v
VPC
 |
 +-- Public Subnets
 |
 +-- Private Subnets
 |
 +-- Internet Gateway
 |
 +-- Route Tables
 |
 +-- NAT
 |
 v
Ready for EKS
```

And this is exactly why we didn't jump straight to:

```text
terraform resource "aws_eks_cluster"
```

without understanding the network underneath it.

---

# 49. Git Workflow for Infrastructure

This repository follows the same governance model.

Developer:

```bash
git checkout -b feature/dev-vpc
```

Changes:

```text
terraform/modules/vpc/
terraform/environments/dev/
```

Then:

```bash
terraform fmt -recursive
terraform validate
terraform plan
```

Commit:

```bash
git add .
git commit -m "feat: add reusable VPC module"
git push -u origin feature/dev-vpc
```

Create:

```text
PR
 |
 v
CI
 |
 +-- terraform fmt check
 +-- terraform validate
 +-- security scan
 +-- terraform plan
 |
 v
Review
 |
 v
Merge
```

---

# 50. Very Important: CI Should Not Automatically Destroy Production

Later our Terraform pipeline will have environment controls.

For example:

```text
DEV
 |
 +-- automatic plan
 +-- controlled apply

QA
 |
 +-- plan
 +-- approval
 +-- apply

PROD
 |
 +-- plan
 +-- mandatory review
 +-- approval
 +-- apply
```

We will never build a pipeline that blindly does:

```bash
terraform apply -auto-approve
```

against production without appropriate controls.

---

# 51. Infrastructure vs GitOps Boundary

At this point remember:

### Terraform

Creates:

```text
AWS infrastructure
```

such as:

```text
VPC
EKS
RDS
ECR
IAM
SQS
Redis
```

### ArgoCD/GitOps

Manages:

```text
Kubernetes workloads
```

such as:

```text
Deployment
Service
ConfigMap
HPA
Ingress
```

So:

```text
Terraform
    |
    v
AWS infrastructure
```

and:

```text
GitOps + ArgoCD
    |
    v
Kubernetes workloads
```

Don't mix these responsibilities unnecessarily.

---

# 52. Where We Are Now

Our complete project currently looks like:

```text
                    GitHub
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
order-service    infrastructure      gitops
       |               |
       v               v
Docker           Terraform
                       |
                       v
                      AWS
                       |
                       v
                      VPC
```

The application still isn't in EKS.

That's the next major milestone.

---

# 53. Next: IAM

Before creating EKS, we need to understand AWS permissions.

We'll create the IAM foundation:

```text
AWS
 |
 +-- Terraform execution role
 |
 +-- GitHub Actions CI role
 |
 +-- EKS cluster role
 |
 +-- EKS node role
 |
 +-- Application roles
 |
 +-- Service-specific IAM roles
```

And we'll implement:

```text
GitHub Actions
      |
      | OIDC
      v
AWS IAM
      |
      v
Role
      |
      v
ECR
```

We'll also cover:

* IAM users vs roles
* trust policies
* permission policies
* least privilege
* EKS cluster role
* EKS node role
* IRSA / EKS Pod Identity
* GitHub OIDC
* Terraform permissions
* service-specific AWS access
* why you should **not** give `AdministratorAccess` to CI
* exact Terraform YAML/HCL
* how the trust relationship works

Then we'll create **ECR**, so our `order-service` Docker image finally has a place in AWS.

**Next: `07-IAM-and-AWS-Identity.md` → IAM foundation → GitHub OIDC → ECR access → EKS identity model.**
