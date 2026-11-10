# 07 — IAM & AWS Identity Foundation

Now we build one of the most important parts of the platform: **identity and permissions**.

A common mistake in AWS projects is to start with:

```text
AdministratorAccess
```

and give it to everything.

We will **not** do that.

Our target is:

```text id="r9x4sq"
Who am I?
    ↓
What role am I assuming?
    ↓
What AWS resources can I access?
    ↓
What exact actions can I perform?
```

---

# 1. Identity Architecture

Our eventual AWS identity model:

```text
                           AWS
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
         Terraform      GitHub CI      EKS
           Role           Role          Roles
              |             |             |
              v             v             v
          AWS APIs        ECR       Kubernetes/AWS
```

For applications:

```text
                         EKS
                          |
                    Kubernetes Pod
                          |
                          v
                 Pod Identity / IAM
                          |
                          v
                     AWS IAM Role
                          |
             +------------+------------+
             |            |            |
            S3           SQS       Secrets Manager
```

Each identity gets only what it needs.

---

# 2. Four Different Identities

We'll distinguish these carefully.

## Identity 1 — Terraform

```text
Terraform
    |
    v
AWS infrastructure
```

Terraform needs permissions to create infrastructure.

---

## Identity 2 — GitHub Actions

```text
GitHub Actions
       |
       v
AWS IAM Role
       |
       v
ECR
```

CI needs to build/push images.

It doesn't need unrestricted access to your AWS account.

---

## Identity 3 — EKS

EKS itself requires AWS IAM roles for the control plane and worker infrastructure.

---

## Identity 4 — Application

Example:

```text
order-service
```

might need:

```text
SQS
Secrets Manager
S3
```

It should **not** receive:

```text
AdministratorAccess
```

---

# 3. IAM Users vs IAM Roles

A common old-school design:

```text
Developer
   |
   v
IAM User
   |
   +-- Access Key
   +-- Secret Key
```

We want to minimize long-lived credentials.

Instead:

```text
Developer
   |
   v
AWS Identity Center / appropriate federation
   |
   v
Temporary credentials
```

For GitHub:

```text
GitHub
   |
   v
OIDC
   |
   v
IAM Role
```

For EKS workloads:

```text
Pod
 |
 v
EKS Pod Identity
 |
 v
IAM Role
```

Temporary credentials are preferable to storing permanent access keys.

---

# 4. GitHub Actions → AWS

This is especially important for our project.

We don't want:

```text
GitHub Secrets
    |
    +-- AWS_ACCESS_KEY_ID
    +-- AWS_SECRET_ACCESS_KEY
```

Instead:

```text
GitHub Actions
       |
       v
GitHub OIDC
       |
       v
AWS IAM
       |
       v
Assume Role
       |
       v
Temporary AWS Credentials
```

---

# 5. What Is OIDC?

OIDC allows GitHub to prove:

> This workflow came from this GitHub repository/organization.

AWS then evaluates the token against the IAM role's trust policy.

Conceptually:

```text
GitHub
  |
  | "I am company-ecommerce/order-service"
  |
  v
OIDC Token
  |
  v
AWS STS
  |
  v
IAM Trust Policy
  |
  +---- Allowed? ----> AssumeRole
  |
  +---- Denied? -----> AccessDenied
```

This is much better than putting permanent AWS credentials into GitHub.

---

# 6. GitHub OIDC Trust

We want to restrict the role.

For example:

```text
AWS IAM Role
     |
     v
Trust Policy
     |
     +-- GitHub OIDC provider
     |
     +-- organization restriction
     |
     +-- repository restriction
     |
     +-- branch/environment restriction
```

The exact claims depend on how we configure the GitHub workflow.

---

# 7. Terraform AWS Provider

Our infrastructure repo will create IAM resources.

Example:

```hcl
provider "aws" {
  region = var.aws_region
}
```

---

# 8. GitHub OIDC Provider

AWS needs to trust GitHub's OIDC issuer.

Terraform example:

```hcl
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

**Important:** OIDC provider configuration details can change over time, so in a real implementation use the current AWS/GitHub guidance rather than blindly copying an old thumbprint.

---

# 9. CI Role

Create:

```text
GitHubActionsECRPushRole
```

Conceptually:

```text
GitHub
   |
   v
OIDC
   |
   v
GitHubActionsECRPushRole
   |
   v
ECR
```

---

# 10. Trust Policy

Example:

```hcl
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

      test = "StringEquals"

      variable = "token.actions.githubusercontent.com:aud"

      values = [
        "sts.amazonaws.com"
      ]
    }

    condition {

      test = "StringLike"

      variable = "token.actions.githubusercontent.com:sub"

      values = [
        "repo:company-ecommerce/order-service:ref:refs/heads/main"
      ]
    }
  }
}
```

This says, conceptually:

> Only an approved GitHub identity matching this repository and branch can assume this role.

---

# 11. Why Restrict the Repository?

Suppose we have:

```text
order-service
payment-service
```

We don't want:

```text
payment-service
      |
      v
Order CI role
      |
      v
ECR permissions for everything
```

Instead:

```text
order-service
    |
    v
Order CI Role
    |
    v
order-service ECR

payment-service
    |
    v
Payment CI Role
    |
    v
payment-service ECR
```

This is least privilege.

---

# 12. IAM Role

```hcl
resource "aws_iam_role" "github_actions_ecr" {

  name = "GitHubActionsOrderServiceECR"

  assume_role_policy =
    data.aws_iam_policy_document.github_actions_trust.json
}
```

---

# 13. ECR Permissions

We don't need:

```text
Action: "*"
Resource: "*"
```

Instead we can scope permissions to the relevant repository.

Example policy:

```hcl
data "aws_iam_policy_document" "ecr_push" {

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

Notice:

```text
GetAuthorizationToken
```

has different resource-scope behavior from repository-specific ECR actions, so don't blindly use the same ARN for every action.

---

# 14. Attach Policy

```hcl
resource "aws_iam_role_policy" "github_actions_ecr" {

  role = aws_iam_role.github_actions_ecr.id

  policy = data.aws_iam_policy_document.ecr_push.json
}
```

Now:

```text
GitHub Actions
      |
      v
IAM Role
      |
      v
ECR push
```

---

# 15. GitHub Workflow

Later our workflow will contain:

```yaml
permissions:
  contents: read
  id-token: write
```

Then:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.AWS_ROLE_ARN }}
    aws-region: ap-south-1
```

Then:

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
```

Then:

```bash
docker push ...
```

The AWS credentials are obtained dynamically.

---

# 16. What Happens Internally?

The workflow:

```text
GitHub Runner
     |
     v
Request OIDC token
     |
     v
GitHub OIDC
     |
     v
AWS STS
     |
     v
Evaluate IAM trust policy
     |
     v
Assume IAM role
     |
     v
Temporary credentials
     |
     v
ECR
```

The credentials expire.

That's what we want.

---

# 17. Terraform Role

Terraform also needs AWS permissions.

But we should distinguish:

```text
Terraform execution identity
```

from:

```text
GitHub application CI identity
```

Terraform might need:

```text
VPC
EKS
IAM
RDS
ECR
S3
SQS
SNS
KMS
```

Whereas application CI might need only:

```text
ECR
```

Don't combine them.

---

# 18. Terraform Permissions

For learning, you may initially use a controlled administrative role to bootstrap infrastructure.

For a mature platform, the Terraform execution role should be scoped to the resources it manages.

Example:

```text
GitHub
 |
 v
Infrastructure PR
 |
 v
Terraform Plan
 |
 v
Approval
 |
 v
Terraform Apply
 |
 v
AWS
```

Later we'll build this pipeline.

---

# 19. EKS IAM Architecture

Now we have another identity layer.

EKS contains:

```text
EKS Cluster
    |
    +-- Control Plane
    |
    +-- Worker Nodes
    |
    +-- Pods
```

Each has different responsibilities.

---

# 20. EKS Cluster IAM Role

The EKS control plane needs an IAM role.

Conceptually:

```text
EKS
 |
 v
Cluster IAM Role
 |
 v
AWS APIs required by EKS
```

Terraform will create this role when we build the EKS module.

---

# 21. EKS Node IAM Role

Worker nodes also need AWS permissions.

For example, nodes may need permissions related to:

```text
EC2
ECR
CNI networking
EKS
```

Architecture:

```text
EKS Node
   |
   v
Node IAM Role
   |
   +-- ECR image pull
   +-- networking-related operations
   +-- node functionality
```

But don't give nodes broad application permissions.

---

# 22. Application IAM

This is where things become interesting.

Suppose:

```text
order-service
```

needs to publish to:

```text
order-events
```

SQS queue.

We create:

```text
OrderServiceIAMRole
```

with:

```text
sqs:SendMessage
```

only against the required queue.

Architecture:

```text
order-service Pod
       |
       v
EKS Pod Identity
       |
       v
OrderServiceIAMRole
       |
       v
SQS Order Events
```

No AWS access keys inside the container.

---

# 23. Never Do This

Don't put:

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    value: ...
  - name: AWS_SECRET_ACCESS_KEY
    value: ...
```

inside Kubernetes.

Also don't build these into the Docker image.

---

# 24. Modern EKS Workload Identity

For this project we'll prefer:

```text
EKS Pod Identity
```

for workload-to-AWS authentication where supported and appropriate.

Conceptually:

```text
Pod
 |
 v
EKS Pod Identity Association
 |
 v
IAM Role
 |
 +---- SQS
 +---- S3
 +---- Secrets Manager
```

This keeps AWS authorization tied to the workload rather than embedding credentials.

---

# 25. Different Service Permissions

This is where our 21-microservice architecture becomes useful.

Example:

### Order Service

```text
SQS:
SendMessage

Secrets Manager:
GetSecretValue

RDS:
database connectivity
```

### Notification Service

```text
SQS:
ReceiveMessage
DeleteMessage

SES/SNS:
notification publishing
```

### Invoice Service

```text
S3:
PutObject
GetObject

SQS:
ReceiveMessage
```

### Catalog Service

```text
S3:
GetObject

DynamoDB:
specific table access
```

Each gets only what it needs.

---

# 26. IAM Policy Design

Think in this format:

```text
WHO
 |
 v
CAN DO WHAT
 |
 v
ON WHICH RESOURCE
```

Example:

```text
OrderServiceRole
      |
      v
sqs:SendMessage
      |
      v
arn:aws:sqs:ap-south-1:123456789012:order-events
```

Not:

```text
OrderServiceRole
      |
      v
*
      |
      v
*
```

---

# 27. IAM Naming Convention

Use predictable names.

```text
EcommerceGitHubOrderServiceECRRole

EcommerceTerraformDevRole

EcommerceTerraformProdRole

EcommerceEKSClusterRole

EcommerceEKSNodeRole

EcommerceOrderServiceRole

EcommercePaymentServiceRole
```

Consistent names make troubleshooting easier.

---

# 28. Tagging

IAM resources can also have useful descriptions/tags where supported.

For AWS infrastructure generally:

```text
Project:
enterprise-ecommerce

Environment:
dev

ManagedBy:
terraform

Owner:
devops
```

This becomes useful for:

```text
Cost Explorer
AWS inventory
troubleshooting
governance
```

---

# 29. The Full Identity Picture

Our architecture now looks like:

```text
                         AWS
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
     Terraform        GitHub CI          EKS
        Role             Role             |
          |               |               |
          v               v               |
      AWS APIs           ECR               |
                                          |
                               +----------+----------+
                               |                     |
                               v                     v
                           Node Role            Pod Identity
                                                     |
                                                     v
                                              Service IAM Role
                                                     |
                              +----------------------+----------------+
                              |                      |                |
                              v                      v                v
                             SQS                    S3        Secrets Manager
```

This is the identity architecture we'll build toward.

---

# 30. Security Boundary

Notice something important.

If an attacker compromises:

```text
order-service
```

they should ideally receive permissions belonging only to:

```text
OrderServiceRole
```

They should not automatically receive:

```text
TerraformRole
GitHubActionsRole
AdministratorRole
```

This is **blast-radius reduction**.

---

# 31. Why This Matters During Incidents

Suppose:

```text
order-service
```

gets compromised.

You can inspect:

```text
kubectl
   |
   v
Pod
   |
   v
ServiceAccount / Pod Identity
   |
   v
IAM Role
   |
   v
IAM Policy
```

Then determine:

```text
What AWS resources could this workload access?
```

That's an actual production troubleshooting/security workflow.

---

# 32. IAM Testing

After creating a role, don't simply assume it works.

Validate:

```text
Can it do what it should?
```

and:

```text
Can it do what it should NOT do?
```

For example:

```text
Order Service
   |
   +--> Send order event      ✓
   |
   +--> Read order secret     ✓
   |
   +--> Delete entire S3      X
   |
   +--> Modify IAM            X
   |
   +--> Create EKS cluster    X
```

That's least privilege in practice.

---

# 33. One More Important Boundary

We now have three major repositories:

```text
order-service
       |
       v
Application

infrastructure
       |
       v
AWS infrastructure

gitops
       |
       v
Kubernetes workloads
```

And three major automation identities:

```text
Application CI Role
Infrastructure Terraform Role
ArgoCD/Kubernetes workload identities
```

Keeping these separate will make the rest of the project much easier to understand.

---

# 34. Next: ECR

Now we're ready for our first AWS service directly related to the application.

We need:

```text
order-service
      |
      v
Docker image
      |
      v
Amazon ECR
```

We'll create:

```text
ECR
 |
 +-- order-service
 +-- auth-service
 +-- user-service
 +-- catalog-service
 +-- ...
 +-- reporting-service
```

We'll cover:

```text
ECR repository design
repository creation with Terraform
image naming
image tags
immutable tags
lifecycle policies
encryption
repository permissions
GitHub OIDC
Docker login
push
pull
Trivy scanning
CI integration
```

Then we'll take the **actual Docker image we created in Part 05** and push it to ECR.

After that, we'll finally have:

```text
Developer
   |
   v
GitHub
   |
   v
CI
   |
   v
Docker
   |
   v
ECR
```

and then we'll build the **EKS cluster that pulls that exact image**.

**Next: `08-ECR-and-Container-Registry.md`.**
