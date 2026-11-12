# 58 — Terraform IAM Implementation

> **Project:** Enterprise E-Commerce Platform
> **AWS Region:** `ap-south-1`
> **Purpose:** Build the IAM foundation for GitHub Actions, EKS, workloads, ECR, SQS, S3, and Secrets Manager.

This chapter establishes one of the most important production principles:

> **No long-lived AWS access keys in GitHub Actions, Kubernetes Pods, Docker images, or source code.**

Our identity architecture will be:

```text
                    AWS IAM
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
 GitHub Actions     EKS Platform   EKS Workloads
      │                  │              │
      ▼                  ▼              ▼
 GitHub OIDC       EKS IAM Roles   EKS Pod Identity
      │                                 │
      ▼                                 ▼
 AWS IAM Role                     Least-Privilege Role
      │                                 │
      ▼                          ┌──────┼──────┐
     ECR                         SQS     S3   Secrets
```

---

# 1. Why IAM Comes Before EKS

EKS itself needs AWS permissions.

GitHub Actions needs AWS permissions.

Applications running inside EKS may need AWS permissions.

For example:

```text
GitHub Actions
      │
      └── Push Docker image → ECR

order-service Pod
      │
      ├── Read secret → Secrets Manager
      └── Send event → SQS
```

We therefore need separate identities.

---

# 2. IAM Identity Separation

We will have at least:

```text
1. Terraform execution role
2. GitHub Actions CI role
3. EKS cluster role
4. EKS node role
5. Application workload roles
```

Conceptually:

```text
Terraform
   │
   ▼
Terraform IAM Role
```

```text
GitHub Actions
   │
   ▼
GitHub OIDC
   │
   ▼
GitHub Actions IAM Role
   │
   ▼
ECR
```

```text
EKS
 ├── Cluster
 │      └── EKS Cluster IAM Role
 │
 └── Nodes
        └── EKS Node IAM Role
```

```text
Order Pod
    │
    ▼
EKS Pod Identity
    │
    ▼
Order Service IAM Role
    │
    ├── SQS
    ├── Secrets Manager
    └── S3
```

---

# 3. IAM Principle

For every role ask:

```text
WHO?
  ↓
CAN DO WHAT?
  ↓
ON WHICH RESOURCE?
```

Example:

```text
order-service
  ↓
sqs:SendMessage
  ↓
arn:aws:sqs:ap-south-1:123456789012:order-events
```

Not:

```text
order-service
  ↓
sqs:*
  ↓
*
```

unless there is a documented reason.

---

# 4. Repository Structure

Add IAM modules:

```text
terraform/
│
├── modules/
│   ├── vpc/
│   │
│   └── iam/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
│
└── environments/
    └── dev/
```

Later we can split IAM further:

```text
modules/
├── iam-github-actions/
├── iam-eks/
├── iam-pod-identity/
└── iam-terraform/
```

For now, keep the first implementation understandable.

---

# 5. AWS Account ID

You'll need your AWS account ID.

Run:

```bash
aws sts get-caller-identity
```

Example:

```json
{
  "UserId": "AIDAXXXXXXXXX",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/admin"
}
```

The important value is:

```text
123456789012
```

Don't hardcode it everywhere.

Terraform can retrieve it.

---

# 6. Terraform Account Data

In the IAM module:

```hcl
data "aws_caller_identity" "current" {}
```

Then:

```hcl
data.aws_caller_identity.current.account_id
```

returns the account ID.

---

# 7. GitHub Actions Authentication

The desired flow is:

```text
GitHub Actions
      │
      ▼
GitHub OIDC Token
      │
      ▼
AWS STS
      │
      ▼
IAM Role
      │
      ▼
AWS APIs
```

There is no:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

stored in GitHub.

---

# 8. Why OIDC Is Better Than Access Keys

Old model:

```text
GitHub
   │
   ├── AWS_ACCESS_KEY_ID
   └── AWS_SECRET_ACCESS_KEY
          │
          ▼
         AWS
```

Problems include:

```text
long-lived credential
credential rotation
secret leakage
credential storage
credential exposure
```

OIDC:

```text
GitHub Job
    │
    ▼
short-lived identity token
    │
    ▼
AWS STS
    │
    ▼
temporary credentials
```

The temporary credentials expire.

---

# 9. GitHub OIDC Provider

AWS needs to trust GitHub's OIDC issuer.

The provider uses:

```text
https://token.actions.githubusercontent.com
```

and the audience:

```text
sts.amazonaws.com
```

In Terraform, the OIDC provider can be created as an AWS IAM OpenID Connect provider.

---

# 10. IAM OIDC Provider

Create:

```hcl id="e7m85c"
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com"
  ]

  thumbprint_list = [
    var.github_oidc_thumbprint
  ]
}
```

### Important production note

Don't blindly copy a historical GitHub OIDC thumbprint from an old tutorial.

OIDC certificate-chain handling and AWS/GitHub recommendations can change. For a production implementation, use the current AWS/GitHub documented value/process for the selected account and verify it before applying.

---

# 11. GitHub OIDC Thumbprint Variable

```hcl id="qv0i7c"
variable "github_oidc_thumbprint" {
  description = "Approved GitHub Actions OIDC provider certificate thumbprint"
  type        = string
}
```

For our learning implementation, we'll explicitly supply the currently approved value rather than burying it inside the module.

---

# 12. GitHub Actions IAM Role

Create:

```hcl id="y1x8j2"
resource "aws_iam_role" "github_actions" {
  name = "ecommerce-github-actions"

  assume_role_policy = data.aws_iam_policy_document.github_actions_trust.json

  tags = {
    Name = "ecommerce-github-actions"
  }
}
```

Now we need the trust policy.

---

# 13. GitHub Trust Policy

The trust relationship is the most important part.

Create:

```hcl id="s9d8ro"
data "aws_iam_policy_document" "github_actions_trust" {

  statement {
    effect = "Allow"

    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]

    principals {
      type = "Federated"

      identifiers = [
        aws_iam_openid_connect_provider.github.arn
      ]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"

      values = [
        "sts.amazonaws.com"
      ]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"

      values = [
        "repo:company-ecommerce/order-service:ref:refs/heads/main"
      ]
    }
  }
}
```

This says:

```text
Only
company-ecommerce/order-service
main branch
```

can assume this role.

---

# 14. Why Restrict the `sub` Claim?

Suppose you allowed:

```text
repo:company-ecommerce/*:*
```

That could potentially allow many repositories/refs.

Instead:

```text
repo:company-ecommerce/order-service:ref:refs/heads/main
```

is much narrower.

Think:

```text
Trust boundary
      ↓
Specific organization
      ↓
Specific repository
      ↓
Specific branch
```

For pull requests and different deployment roles, you may create separate trust conditions.

---

# 15. CI vs Production Deployment Roles

Don't necessarily use one giant GitHub Actions role for everything.

Better:

```text
GitHub Actions
│
├── CI role
│    └── ECR push
│
└── infrastructure role
     └── Terraform AWS permissions
```

And potentially:

```text
Production deployment
       │
       ▼
separate restricted role
```

This limits blast radius.

---

# 16. ECR Permissions

Our application CI needs to push images.

The required actions are typically around:

```text
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
```

Create a policy document:

```hcl id="aq12zn"
data "aws_iam_policy_document" "github_ecr" {

  statement {
    effect = "Allow"

    actions = [
      "ecr:GetAuthorizationToken"
    ]

    resources = ["*"]
  }

  statement {
    effect = "Allow"

    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:CompleteLayerUpload",
      "ecr:InitiateLayerUpload",
      "ecr:PutImage",
      "ecr:UploadLayerPart"
    ]

    resources = [
      aws_ecr_repository.order_service.arn
    ]
  }
}
```

We'll create the ECR repository in the next chapter.

---

# 17. Temporary Placeholder

Because Chapter 59 creates ECR, the final policy will depend on:

```text
aws_ecr_repository.order_service.arn
```

So for now, the architecture is:

```text
GitHub Actions Role
        │
        ▼
ECR Repository
        │
        ▼
order-service image
```

We'll wire the modules together in Chapter 59.

---

# 18. Don't Give CI AdministratorAccess

Avoid:

```text id="0p46cr"
AdministratorAccess
```

for application CI.

That would mean:

```text
Compromised CI
     ↓
full AWS account
```

Instead:

```text
Compromised CI
     ↓
ECR permissions
```

with tightly controlled permissions.

---

# 19. Terraform Execution Role

Terraform itself needs permissions to create:

```text
VPC
IAM
ECR
EKS
RDS
Redis
SQS
SNS
KMS
S3
```

For learning, you might initially run Terraform using an authenticated administrative identity.

For a production enterprise implementation, use:

```text
GitHub Actions
      ↓
OIDC
      ↓
Terraform IAM Role
      ↓
AWS
```

and restrict that role according to the infrastructure it manages.

---

# 20. Terraform Role Architecture

```text
Developer
   │
   ▼
GitHub PR
   │
   ▼
Terraform Plan
   │
   ▼
Approval
   │
   ▼
Terraform Apply
   │
   ▼
OIDC
   │
   ▼
Terraform IAM Role
   │
   ▼
AWS
```

This removes the need for long-lived AWS keys in CI.

---

# 21. EKS Cluster IAM Role

EKS needs a cluster IAM role.

Conceptually:

```text
EKS Control Plane
        │
        ▼
EKS Cluster IAM Role
        │
        ▼
AWS APIs required by EKS
```

Create:

```hcl id="4swjhc"
resource "aws_iam_role" "eks_cluster" {
  name = "ecommerce-eks-cluster"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "eks.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

---

# 22. EKS Cluster Policy

Attach the AWS-managed EKS cluster policy appropriate for the selected EKS implementation.

For example:

```hcl id="zv4fye"
resource "aws_iam_role_policy_attachment" "eks_cluster" {
  role = aws_iam_role.eks_cluster.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```

The exact AWS-managed policies should be verified against the EKS version and current AWS architecture when we create the cluster.

---

# 23. EKS Node IAM Role

Worker nodes also require an IAM role.

```hcl id="7t2bxx"
resource "aws_iam_role" "eks_node" {
  name = "ecommerce-eks-node"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "ec2.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

---

# 24. Node Role vs Pod Role

This distinction is extremely important.

### Node role

```text id="0iyh2x"
EC2/EKS Node
     │
     ▼
Node IAM Role
```

Used for node-level AWS operations.

### Pod role

```text id="j1bujc"
Order Pod
     │
     ▼
EKS Pod Identity
     │
     ▼
Order IAM Role
```

Used for application-specific AWS access.

Don't give the node role every permission that applications need.

---

# 25. Node Permissions

The EKS node role commonly needs permissions for:

```text
EKS node operation
ECR image pulls
VPC CNI operation
```

AWS-managed policies commonly involved include:

```text
AmazonEKSWorkerNodePolicy
AmazonEC2ContainerRegistryPullOnly
AmazonEKS_CNI_Policy
```

However, the exact modern EKS configuration can vary depending on whether permissions are attached to the node role or moved to EKS add-on/Pod Identity roles.

We will make that decision explicitly in the EKS chapter rather than blindly attaching every possible policy.

---

# 26. ECR Pull vs ECR Push

This is an important distinction.

### CI

```text
GitHub Actions
      │
      ▼
ECR
      │
      └── PUSH
```

needs:

```text
ECR write permissions
```

### EKS

```text
EKS node
      │
      ▼
ECR
      │
      └── PULL
```

needs:

```text
ECR read/pull permissions
```

Do not give EKS nodes ECR push permissions.

---

# 27. Application Workload IAM

Suppose Order Service needs to publish:

```text
order-events
```

to SQS.

We create:

```text id="uj8y75"
order-service
     │
     ▼
OrderService IAM Role
     │
     └── sqs:SendMessage
             │
             ▼
       order-events queue
```

---

# 28. Order Service IAM Policy

Conceptual:

```hcl id="j6t7qn"
data "aws_iam_policy_document" "order_service" {

  statement {
    effect = "Allow"

    actions = [
      "sqs:SendMessage"
    ]

    resources = [
      var.order_events_queue_arn
    ]
  }
}
```

Notice:

```text id="4d9u9v"
actions = ["sqs:SendMessage"]
```

not:

```text
sqs:*
```

And:

```text id="4r1p4a"
resource = exact queue ARN
```

not:

```text
*
```

---

# 29. Secrets Manager Access

Suppose Order Service needs its database credentials.

It should receive permission to:

```text id="6x9b5f"
secretsmanager:GetSecretValue
```

on:

```text id="3kgvck"
specific order-service secret
```

Example:

```hcl id="fyhkn4"
statement {
  effect = "Allow"

  actions = [
    "secretsmanager:GetSecretValue"
  ]

  resources = [
    var.order_service_db_secret_arn
  ]
}
```

If the secret uses a customer-managed KMS key, the workload may additionally require appropriate KMS permissions.

---

# 30. S3 Access

Suppose Order Service stores generated invoice files.

It might need:

```text id="75l3do"
s3:PutObject
s3:GetObject
```

against:

```text id="my-company-invoices"
```

but not:

```text id="qg0rbi"
s3:*
*
```

Example:

```hcl id="b4sh1u"
statement {
  effect = "Allow"

  actions = [
    "s3:GetObject",
    "s3:PutObject"
  ]

  resources = [
    "${var.invoice_bucket_arn}/orders/*"
  ]
}
```

---

# 31. EKS Pod Identity

For our EKS workloads, we prefer:

```text id="pl1wlj"
EKS Pod Identity
```

where supported.

The architecture:

```text
Kubernetes Pod
      │
      ▼
EKS Pod Identity Association
      │
      ▼
IAM Role
      │
      ▼
AWS service
```

This avoids putting AWS credentials inside:

```text
Kubernetes Secret
```

---

# 32. Example

Order Service:

```text id="tq2j0s"
Namespace:
production

ServiceAccount:
order-service

IAM Role:
ecommerce-order-service

Permissions:
    sqs:SendMessage
    secretsmanager:GetSecretValue
```

Then:

```text id="ud3gsi"
order-service Pod
       │
       ▼
ServiceAccount
       │
       ▼
EKS Pod Identity
       │
       ▼
IAM Role
       │
       ├── SQS
       └── Secrets Manager
```

---

# 33. Do Not Put AWS Keys in Kubernetes

Never do:

```yaml id="6x1p8w"
env:
  - name: AWS_ACCESS_KEY_ID
    value: AKIA...
  - name: AWS_SECRET_ACCESS_KEY
    value: ...
```

Also don't store them in:

```text
Helm values
ConfigMaps
Dockerfile
GitHub repository
Git
Kubernetes Secrets
```

Use workload identity.

---

# 34. IAM Policy Boundaries

For larger organizations, you may additionally use:

```text id="3f5q5x"
Permissions boundaries
Service Control Policies
IAM Access Analyzer
```

These provide higher-level controls.

Think:

```text id="e8bq0d"
IAM Role
   │
   ▼
Allowed permissions
   │
   ▼
Permissions Boundary
   │
   ▼
Maximum allowed permissions
```

---

# 35. IAM Role Naming

Use consistent names.

Examples:

```text id="ck9cys"
ecommerce-github-actions-ci
ecommerce-terraform
ecommerce-eks-cluster
ecommerce-eks-node
ecommerce-order-service
ecommerce-payment-service
```

Avoid:

```text
test-role
new-role
temp-role
admin-role
```

Names should communicate purpose.

---

# 36. IAM Module Variables

Create:

```hcl id="1rj2ay"
variable "project_name" {
  type = string
}

variable "github_org" {
  type = string
}

variable "github_repository" {
  type = string
}

variable "github_branch" {
  type    = string
  default = "main"
}
```

For workload roles, later add:

```hcl id="wbbqiz"
variable "order_events_queue_arn" {
  type = string
}

variable "order_service_secret_arn" {
  type = string
}
```

---

# 37. Outputs

Create:

```hcl id="6f7e4v"
output "github_actions_role_arn" {
  value = aws_iam_role.github_actions.arn
}

output "eks_cluster_role_arn" {
  value = aws_iam_role.eks_cluster.arn
}

output "eks_node_role_arn" {
  value = aws_iam_role.eks_node.arn
}
```

Later:

```text id="qqm65j"
Terraform output
      │
      ▼
EKS module
```

---

# 38. IAM Module Boundary

Keep the responsibilities clear.

```text
VPC module
  └── networking

IAM module
  └── identities and permissions

ECR module
  └── container registry

EKS module
  └── Kubernetes cluster
```

Don't create one enormous:

```text
main.tf
```

containing the entire AWS environment.

---

# 39. Environment Configuration

In:

```text id="0t8x5f"
terraform/environments/dev/
```

call the IAM module:

```hcl id="u7l3ax"
module "iam" {
  source = "../../modules/iam"

  project_name = "ecommerce"

  github_org = "company-ecommerce"

  github_repository = "order-service"

  github_branch = "main"
}
```

The exact resource dependencies will be completed as ECR and EKS modules are added.

---

# 40. IAM Validation

Before applying:

```bash id="9wix5q"
terraform fmt -recursive
```

Then:

```bash id="m0ivak"
terraform validate
```

Then:

```bash id="4p4yce"
terraform plan
```

Review:

```text id="03yl0u"
IAM Role
IAM Trust Policy
IAM Policy
OIDC Provider
```

---

# 41. Inspect IAM

AWS CLI:

```bash id="q1b4c5"
aws iam get-role \
  --role-name ecommerce-eks-cluster
```

Get policy attachments:

```bash id="b9o1h5"
aws iam list-attached-role-policies \
  --role-name ecommerce-eks-cluster
```

For GitHub:

```bash id="l2mgks"
aws iam get-role \
  --role-name ecommerce-github-actions-ci
```

---

# 42. Test GitHub OIDC Later

We will not fully test the GitHub role until Chapter 62.

The eventual workflow will include:

```yaml id="f0fl4m"
permissions:
  contents: read
  id-token: write
```

Then:

```text id="2p49zr"
GitHub Runner
     │
     ▼
OIDC token
     │
     ▼
AWS STS
     │
     ▼
Assume IAM Role
     │
     ▼
Temporary AWS credentials
```

---

# 43. GitHub Actions Configuration

Later our CI will use something conceptually like:

```yaml id="p1b8g8"
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@approved-version
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ap-south-1
```

In production, third-party Actions should be pinned to approved commit SHAs according to your organization's supply-chain policy.

We will implement this in Chapter 62.

---

# 44. IAM Troubleshooting

### `AccessDenied`

Start with:

```text id="jzqvqe"
Who is calling?
        ↓
Which role?
        ↓
Which action?
        ↓
Which resource?
        ↓
Which trust policy?
        ↓
Which identity policy?
```

Example:

```text id="qwmqf5"
AccessDenied:
sqs:SendMessage
```

Check:

```text id="3hwmjz"
Pod IAM role
        ↓
Does policy allow sqs:SendMessage?
        ↓
Correct queue ARN?
        ↓
Correct AWS region?
        ↓
Correct account?
        ↓
SQS queue policy?
```

---

# 45. Trust Policy vs Permission Policy

This distinction is extremely important.

### Trust policy

Answers:

> **Who can assume this role?**

Example:

```text id="5w82ar"
GitHub OIDC
      ↓
Can assume GitHub Actions role
```

### Permission policy

Answers:

> **What can the role do?**

Example:

```text id="72agxq"
GitHub Actions role
      ↓
ecr:PutImage
```

So:

```text id="lq68pl"
Trust Policy
    =
Who may become this identity?

Permission Policy
    =
What may this identity do?
```

---

# 46. Example: GitHub AccessDenied

Suppose CI receives:

```text id="vhsy5a"
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

This is probably a **trust policy** problem.

Check:

```text id="7i7hwm"
OIDC provider
aud claim
sub claim
repository
branch
```

If CI successfully assumes the role but gets:

```text id="lrb0qx"
AccessDenied: ecr:PutImage
```

that's a **permission policy** problem.

This distinction saves a lot of troubleshooting time.

---

# 47. Example: Pod AccessDenied

Suppose Order Service gets:

```text id="nqf6zi"
AccessDeniedException:
sqs:SendMessage
```

Check:

```text id="i5p3n3"
Pod
 ↓
ServiceAccount
 ↓
Pod Identity association
 ↓
IAM role
 ↓
IAM policy
 ↓
SQS ARN
```

Do not immediately change the application code.

---

# 48. IAM Security Checklist

```text
[ ] No long-lived AWS keys in GitHub
[ ] No AWS keys in Kubernetes
[ ] GitHub uses OIDC
[ ] Trust policies restrict repository/branch
[ ] CI role has minimum required permissions
[ ] EKS node role has minimum required permissions
[ ] Workloads use Pod Identity
[ ] Workload policies use exact ARNs
[ ] No unnecessary AdministratorAccess
[ ] IAM policies reviewed
[ ] Production roles separated from dev where appropriate
[ ] IAM changes go through Terraform
[ ] CloudTrail/audit logging enabled
[ ] IAM Access Analyzer considered
```

---

# 49. Final IAM Architecture

```text
                         GitHub
                           │
                           ▼
                      GitHub OIDC
                           │
                           ▼
                 AWS IAM CI Role
                           │
                           ▼
                          ECR
                           │
                           │
                           ▼
                        EKS
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       EKS Cluster Role            EKS Nodes
                                        │
                                        ▼
                                  Node IAM Role
                                        │
                                        │
                               ┌────────┴────────┐
                               │                 │
                               ▼                 ▼
                          EKS Pod Identity   AWS APIs
                               │
                               ▼
                       Application IAM Role
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
                   SQS        S3      Secrets Manager
```

---

# 50. One Critical Production Rule

Never solve an IAM problem by doing this:

```text
AccessDenied
    ↓
attach AdministratorAccess
    ↓
works
```

Instead:

```text
AccessDenied
    ↓
identify principal
    ↓
identify action
    ↓
identify resource
    ↓
fix trust/policy
    ↓
test
```

That's the difference between **making AWS work** and **operating AWS securely**.

---

# 51. Current Progress

```text
[✓] Repository setup
[✓] Order Service
[✓] Local Docker
[✓] Terraform state
[✓] VPC
[✓] Public/private networking
[✓] NAT
[✓] VPC endpoint foundation
[✓] IAM architecture
[✓] GitHub OIDC design
[✓] EKS IAM design
[✓] Workload IAM design

[ ] ECR
[ ] EKS
[ ] EKS add-ons
[ ] GitHub Actions
[ ] Helm
[ ] GitOps
[ ] ArgoCD
[ ] RDS deployment
```

---

# 52. Definition of Done

Before moving forward:

```text
[ ] IAM module created
[ ] Terraform validates
[ ] GitHub OIDC provider designed/configured
[ ] GitHub Actions trust restricted to repository/branch
[ ] EKS cluster role defined
[ ] EKS node role defined
[ ] CI ECR permissions designed
[ ] Workload role pattern defined
[ ] No AWS credentials stored in Git
[ ] No AWS credentials stored in Kubernetes
[ ] Least-privilege model understood
[ ] Trust policy vs permission policy understood
```

---

# 53. Next: `59-Terraform-ECR.md`

Now we have:

```text
VPC
 │
 └── Networking
     
IAM
 │
 └── Identity
```

The next layer is the container registry:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
CI
    │
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
EKS
```

In **59 — Terraform ECR**, we'll create the actual ECR repositories, immutable image tags, encryption, scan-on-push, lifecycle policies, repository policies, CI push permissions, EKS pull permissions, image naming/versioning, and the **build-once → scan → push → promote by digest** workflow.
