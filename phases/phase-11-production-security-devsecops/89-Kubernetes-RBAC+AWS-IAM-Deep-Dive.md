# 89 — Kubernetes RBAC + AWS IAM Deep Dive

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **CI:** GitHub Actions + OIDC
> **CD:** ArgoCD + GitOps
> **Workload Identity:** EKS Pod Identity
> **Previous:** 88 — Kubernetes Admission Control and Image Verification
> **Next:** 90 — Secrets, KMS and Secret Rotation

---

# 1. Objective

This chapter establishes the complete **identity and permission architecture** for our production EKS platform.

We have several different identities:

```text
Human Engineer
      |
      v
AWS IAM / Identity Center
      |
      v
EKS Access
```

```text
GitHub Actions
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

```text
ArgoCD
      |
      v
Kubernetes ServiceAccount
      |
      v
Kubernetes RBAC
      |
      v
Kubernetes API
```

```text
Application Pod
      |
      v
Kubernetes ServiceAccount
      |
      v
EKS Pod Identity
      |
      v
AWS IAM Role
      |
      +--> SQS
      +--> S3
      +--> Secrets Manager
```

The most important lesson:

> **Kubernetes RBAC and AWS IAM are different authorization systems.**

---

# 2. The Four Questions We Must Answer

For every identity in production, ask:

### Who are you?

Authentication.

### What are you allowed to do?

Authorization.

### Where can you do it?

Scope.

### Why do you have this permission?

Business/technical justification.

Example:

```text
order-service
    |
    | Who?
    v
ServiceAccount
    |
    | What?
    v
SendMessage
    |
    | Where?
    v
specific SQS queue
```

Not:

```text
order-service
    |
    +--> AdministratorAccess
```

---

# 3. AWS IAM vs Kubernetes RBAC

This distinction is critical.

## AWS IAM

Controls access to AWS services:

```text
S3
SQS
SNS
Secrets Manager
KMS
ECR
CloudWatch
DynamoDB
```

Example:

```text
order-service
    |
    v
SQS:SendMessage
```

---

## Kubernetes RBAC

Controls access to Kubernetes API resources:

```text
Pods
Deployments
Services
ConfigMaps
Secrets
Namespaces
Jobs
Nodes
```

Example:

```text
developer
    |
    v
get pods
```

These are completely different permissions.

---

# 4. Overall Identity Architecture

Our production model:

```text
                          AWS
                           |
          +----------------+----------------+
          |                |                |
       Human            CI/CD            EKS
          |                |                |
       IAM/SSO           OIDC             Workloads
          |                |                |
          v                v                v
       EKS Access      IAM Role       Pod Identity
          |                |                |
          v                v                v
    Kubernetes RBAC      ECR         AWS IAM Role
                                           |
                                  +--------+--------+
                                  |        |        |
                                 SQS      S3      Secrets
```

---

# 5. Human Access to EKS

A developer should not share:

```text
admin
password
```

or a common kubeconfig.

Instead:

```text
Engineer
   |
   v
Corporate Identity
   |
   v
AWS IAM Identity Center / IAM
   |
   v
EKS Access
   |
   v
Kubernetes RBAC
```

This gives us individual identity and auditability.

---

# 6. Human Roles

A realistic organization may have:

```text
Platform Admin
Platform Engineer
DevOps Engineer
Developer
SRE
Security Engineer
Read-Only Auditor
```

They should not all receive the same access.

Example:

```text
Developer
    |
    +--> dev namespace
    +--> read logs
    +--> inspect pods
    +--> restart own workloads if approved
```

while:

```text
Platform Engineer
    |
    +--> cluster administration
    +--> node management
    +--> platform namespaces
```

---

# 7. Least Privilege

The basic rule:

> Give the minimum permission necessary to perform the job.

Bad:

```text
Developer
    |
    +--> cluster-admin
```

Better:

```text
Developer
    |
    +--> ecommerce-dev
          |
          +--> get pods
          +--> get logs
          +--> describe deployments
```

---

# 8. Kubernetes RBAC Objects

Kubernetes RBAC primarily uses:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

Think of them like this:

```text
Role
   |
   +--> permissions inside one namespace

ClusterRole
   |
   +--> reusable/cluster-scoped permissions

RoleBinding
   |
   +--> connects identity to Role/ClusterRole in namespace

ClusterRoleBinding
   |
   +--> connects identity to ClusterRole cluster-wide
```

---

# 9. Role

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: order-service-reader
  namespace: ecommerce-dev

rules:

  - apiGroups:
      - ""

    resources:
      - pods

    verbs:
      - get
      - list
      - watch
```

This means:

```text
order-service-reader
        |
        +--> pods
        |
        +--> get
        +--> list
        +--> watch
```

Only inside:

```text
ecommerce-dev
```

---

# 10. RoleBinding

A Role doesn't automatically grant access to anyone.

We need a binding.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: developer-order-reader
  namespace: ecommerce-dev

subjects:

  - kind: Group
    name: backend-team
    apiGroup: rbac.authorization.k8s.io

roleRef:

  kind: Role
  name: order-service-reader
  apiGroup: rbac.authorization.k8s.io
```

Now:

```text
backend-team
     |
     v
RoleBinding
     |
     v
order-service-reader
     |
     v
pods
```

---

# 11. Why Groups?

Don't create individual RBAC rules everywhere.

Bad:

```text
Surendra
Developer A
Developer B
Developer C
```

Instead:

```text
backend-team
platform-team
sre-team
security-team
```

Then manage membership through the organization's identity system.

---

# 12. ClusterRole

Some permissions are cluster-wide.

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: node-reader

rules:

  - apiGroups:
      - ""

    resources:
      - nodes

    verbs:
      - get
      - list
      - watch
```

This can be bound to an appropriate operational group.

---

# 13. Avoid ClusterRoleBinding Unless Necessary

This is dangerous:

```yaml
kind: ClusterRoleBinding

roleRef:
  kind: ClusterRole
  name: cluster-admin
```

because it can grant broad cluster access.

For most application teams:

```text
Role + RoleBinding
```

is safer than:

```text
ClusterRole + ClusterRoleBinding
```

when namespace-level access is sufficient.

---

# 14. Kubernetes Verbs

Common verbs:

```text
get
list
watch
create
update
patch
delete
deletecollection
```

Think about permissions carefully.

For read-only access:

```text
get
list
watch
```

For deployment management:

```text
get
list
watch
create
update
patch
delete
```

Don't give:

```text
*
```

unless there is a strong reason.

---

# 15. Resource-Level Permissions

Example:

```yaml
rules:

  - apiGroups:
      - ""

    resources:
      - pods

    verbs:
      - get
      - list
```

This does **not** automatically mean:

```text
secrets
deployments
nodes
services
```

are accessible.

That's the advantage of explicit permissions.

---

# 16. Kubernetes Secrets Are Special

Be careful with:

```yaml
resources:
  - secrets

verbs:
  - get
```

Anyone who can read Kubernetes Secrets may obtain sensitive information.

Therefore:

```text
Developer
   |
   +--> get pods
   +--> get logs
   X
   +--> get secrets
```

should be the default unless there is a specific requirement.

Our application architecture already prefers:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
```

and the application itself doesn't need Kubernetes API access to retrieve it.

---

# 17. ServiceAccounts

Every application gets a dedicated ServiceAccount.

For Order Service:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: order-service
  namespace: ecommerce-prod
```

Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: order-service
```

---

# 18. ServiceAccount Does Not Automatically Mean AWS Access

This is important.

Creating:

```text
order-service ServiceAccount
```

doesn't automatically grant:

```text
S3
SQS
Secrets Manager
```

AWS access is provided through:

```text
EKS Pod Identity
```

which maps:

```text
ServiceAccount
      ↓
IAM Role
```

---

# 19. EKS Pod Identity

Our architecture:

```text
Pod
 |
 v
ServiceAccount
 |
 v
EKS Pod Identity Association
 |
 v
IAM Role
 |
 v
AWS API
```

For Order Service:

```text
order-service
       |
       v
order-service ServiceAccount
       |
       v
order-service IAM Role
       |
       +--> SQS
       +--> S3
       +--> Secrets Manager
```

No static AWS keys.

---

# 20. Why Not AWS Keys?

Never put this inside a pod:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

because credentials can potentially leak through:

```text
logs
debugging
environment inspection
container compromise
application errors
```

Instead:

```text
Pod Identity
```

allows AWS to provide temporary credentials to the workload.

---

# 21. Order Service AWS Permissions

Let's assume Order Service needs:

```text
SQS:
    send order events

S3:
    upload invoice documents

Secrets Manager:
    retrieve database secret
```

Then its IAM policy should contain only those permissions.

Conceptually:

```text
order-service IAM Role
        |
        +--> SQS SendMessage
        |
        +--> S3 PutObject
        |
        +--> SecretsManager GetSecretValue
```

Not:

```text
*
```

---

# 22. Example SQS Permission

Suppose the queue is:

```text
arn:aws:sqs:ap-south-1:123456789012:order-events
```

Permission:

```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage"
  ],
  "Resource": "arn:aws:sqs:ap-south-1:123456789012:order-events"
}
```

This means:

```text
order-service
    |
    +--> SendMessage
    |
    +--> order-events
```

It does **not** automatically allow:

```text
ReceiveMessage
DeleteMessage
PurgeQueue
```

---

# 23. Example S3 Permission

Suppose:

```text
company-ecommerce-prod-documents
```

Order Service only needs:

```text
orders/*
```

We can scope access to the required prefix.

Conceptually:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::company-ecommerce-prod-documents/orders/*"
}
```

The exact policy should reflect the application's actual access pattern.

---

# 24. Secrets Manager Permission

Suppose:

```text
prod/ecommerce/order-service/database
```

is the secret.

Then:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/ecommerce/order-service/database-*"
}
```

Only that secret should be accessible.

---

# 25. KMS Consideration

If the secret uses a customer-managed KMS key, the workload may also require appropriate KMS permissions depending on the AWS service/resource configuration.

Don't automatically add:

```text
kms:*
```

Instead determine exactly what the service integration requires.

This becomes especially important in Chapter 90.

---

# 26. Complete Order Service IAM Policy

Conceptually:

```json
{
  "Version": "2012-10-17",

  "Statement": [

    {
      "Sid": "SendOrderEvents",

      "Effect": "Allow",

      "Action": [
        "sqs:SendMessage"
      ],

      "Resource":
        "arn:aws:sqs:ap-south-1:123456789012:order-events"
    },

    {
      "Sid": "OrderDocuments",

      "Effect": "Allow",

      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],

      "Resource":
        "arn:aws:s3:::company-ecommerce-prod-documents/orders/*"
    },

    {
      "Sid": "ReadDatabaseSecret",

      "Effect": "Allow",

      "Action": [
        "secretsmanager:GetSecretValue"
      ],

      "Resource":
        "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/ecommerce/order-service/database-*"
    }

  ]
}
```

This is much safer than:

```text
AdministratorAccess
```

---

# 27. What About RDS?

Important distinction:

Order Service does **not** normally use IAM permissions to connect to PostgreSQL just because it has an IAM role.

Database connectivity involves:

```text
Order Service
     |
     v
VPC networking
     |
     v
Security Group
     |
     v
RDS PostgreSQL
```

Authentication can use:

```text
database credentials
```

or, where supported and intentionally configured:

```text
IAM database authentication
```

Our project currently uses Secrets Manager for database credentials.

Therefore:

```text
IAM
    |
    +--> retrieve secret

Network
    |
    +--> reach RDS

PostgreSQL
    |
    +--> authenticate database user
```

These are separate layers.

---

# 28. IAM Trust Policy

An IAM permission policy says:

> What can this role do?

A trust policy says:

> Who can assume this role?

These are different.

```text
Trust Policy
     |
     +--> WHO can assume?

Permissions Policy
     |
     +--> WHAT can they do?
```

---

# 29. GitHub Actions OIDC

Our CI architecture:

```text
GitHub Actions
      |
      v
GitHub OIDC Token
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

No long-lived AWS access keys.

---

# 30. GitHub OIDC Trust

The trust policy should restrict:

```text
GitHub organization
repository
branch/environment
```

For example:

```text
repo:company-ecommerce/order-service
```

and preferably the intended branch/environment context.

Conceptually:

```json
{
  "Effect": "Allow",

  "Principal": {
    "Federated":
      "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },

  "Action":
    "sts:AssumeRoleWithWebIdentity",

  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud":
        "sts.amazonaws.com"
    },

    "StringLike": {
      "token.actions.githubusercontent.com:sub":
        "repo:company-ecommerce/order-service:ref:refs/heads/main"
    }
  }
}
```

The exact trust conditions should match the workflow design.

---

# 31. GitHub CI Permissions

CI should only have what it needs.

For the Order Service build/push workflow:

```text
ECR:
    GetAuthorizationToken
    BatchCheckLayerAvailability
    InitiateLayerUpload
    UploadLayerPart
    CompleteLayerUpload
    PutImage
```

It should **not** have:

```text
EKS cluster-admin
```

or:

```text
AdministratorAccess
```

---

# 32. Why CI Shouldn't Have EKS Admin

Our architecture is:

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

If GitHub Actions has EKS admin:

```text
GitHub compromise
      |
      v
AWS credentials/token
      |
      v
EKS admin
      |
      v
Production compromise
```

Removing that permission significantly reduces blast radius.

---

# 33. GitHub → GitOps Authentication

Our CI also needs to update the GitOps repository.

We previously selected:

```text
GitHub App
```

as the preferred mechanism.

Architecture:

```text
GitHub Actions
      |
      v
GitHub App
      |
      v
GitOps Repository
```

The App should only have permissions required to:

```text
read repository
create branch
create/update PR
```

depending on the exact automation.

---

# 34. ArgoCD Identity

ArgoCD needs Kubernetes permissions to deploy applications.

Conceptually:

```text
ArgoCD
   |
   v
Kubernetes ServiceAccount
   |
   v
Kubernetes RBAC
   |
   v
Applications
```

ArgoCD should not automatically have unrestricted access to everything.

Use:

```text
AppProject
+
RBAC
+
destination restrictions
+
source repository restrictions
```

---

# 35. ArgoCD Project Isolation

We already introduced:

```text
development
qa
staging
production
```

projects.

Conceptually:

```text
AppProject: production

Allowed source:
    company-ecommerce/gitops

Allowed destination:
    production EKS cluster

Allowed namespace:
    ecommerce-prod

Allowed resource types:
    approved resources
```

This reduces accidental deployment outside the intended boundary.

---

# 36. ArgoCD Production Permission Model

```text
Developer
    |
    X
    | direct production deployment
    |
GitOps PR
    |
    v
Approval
    |
    v
ArgoCD
    |
    v
Production AppProject
    |
    v
ecommerce-prod
```

This combines:

```text
GitHub permissions
+
Jira approval
+
GitOps
+
ArgoCD RBAC
+
Kubernetes RBAC
```

---

# 37. Humans vs Workloads

A useful mental model:

```text
HUMANS

Engineer
   ↓
AWS Identity
   ↓
EKS Access
   ↓
Kubernetes RBAC
```

```text
WORKLOADS

Pod
   ↓
ServiceAccount
   ↓
Pod Identity
   ↓
AWS IAM
```

Don't use a human's AWS credentials inside an application.

---

# 38. Service-to-Service Authentication

IAM does not automatically solve:

```text
order-service → payment-service
```

That is application/service communication.

We need:

```text
NetworkPolicy
+
TLS
+
application authentication/authorization
```

For example:

```text
order-service
    |
    v
payment-service
```

can use:

```text
internal TLS
service identity
JWT/mTLS depending on architecture
```

IAM is for AWS APIs.

---

# 39. 21-Service IAM Model

Each service should have its own identity.

```text
auth-service
    ↓
auth-service IAM Role

user-service
    ↓
user-service IAM Role

catalog-service
    ↓
catalog-service IAM Role

...

order-service
    ↓
order-service IAM Role

payment-service
    ↓
payment-service IAM Role
```

Not:

```text
all-services
    ↓
one giant IAM Role
```

---

# 40. Why One IAM Role Is Dangerous

Suppose all services use:

```text
ecommerce-application-role
```

and it has:

```text
S3
SQS
Secrets Manager
DynamoDB
```

Then:

```text
payment-service compromise
        |
        v
shared role
        |
        v
access to order data
        |
        v
access to unrelated resources
```

Separate roles reduce blast radius.

---

# 41. Service Permission Matrix

Our design should eventually look like:

| Service              | SQS                      | S3           | Secrets         | DynamoDB          | Other            |
| -------------------- | ------------------------ | ------------ | --------------- | ----------------- | ---------------- |
| order-service        | Send                     | Orders       | DB secret       | —                 | —                |
| payment-service      | Send/Receive as required | Receipts     | Payment secret  | —                 | Payment provider |
| inventory-service    | Send/Receive             | —            | DB secret       | Inventory if used | —                |
| notification-service | Receive                  | Templates    | SMTP/API secret | —                 | —                |
| reporting-service    | Receive                  | Read reports | DB secret       | Read if required  | —                |

The exact permissions will come from each service's actual architecture.

---

# 42. IAM Access Analyzer

We should periodically review IAM policies.

Useful questions:

```text
Does this role have unused permissions?

Does it grant access outside the intended account?

Can this role be assumed by unintended identities?

Does this resource policy expose data externally?
```

AWS IAM Access Analyzer can help identify unintended access.

---

# 43. Permission Validation

Before granting a permission:

```text
Requirement
    ↓
Identify resource
    ↓
Identify action
    ↓
Scope resource ARN
    ↓
Create policy
    ↓
Test
    ↓
Monitor
```

Example:

```text
Requirement:
Order Service sends events.

Action:
sqs:SendMessage

Resource:
order-events queue

Policy:
only SendMessage on that ARN
```

---

# 44. Don't Start With `*`

Avoid:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Instead:

```json
{
  "Effect": "Allow",
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:..."
}
```

Least privilege starts from the actual application requirement.

---

# 45. Terraform Structure

Our infrastructure repo can manage IAM like:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── iam/
    │       ├── main.tf
    │       ├── variables.tf
    │       └── outputs.tf
    │
    └── environments/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Application IAM roles can be created through reusable modules.

---

# 46. Example Terraform Policy

Conceptually:

```hcl
data "aws_iam_policy_document" "order_service" {

  statement {
    sid    = "SendOrderEvents"
    effect = "Allow"

    actions = [
      "sqs:SendMessage"
    ]

    resources = [
      aws_sqs_queue.order_events.arn
    ]
  }

  statement {
    sid    = "ReadDatabaseSecret"
    effect = "Allow"

    actions = [
      "secretsmanager:GetSecretValue"
    ]

    resources = [
      aws_secretsmanager_secret.order_database.arn
    ]
  }
}
```

Then attach the policy to the Order Service IAM role.

---

# 47. Pod Identity Association

Conceptually:

```text
order-service ServiceAccount
            |
            v
EKS Pod Identity Association
            |
            v
order-service IAM Role
```

The exact Terraform resource/API configuration should follow the EKS Pod Identity implementation supported by the pinned AWS provider/EKS version.

We should not mix older IRSA-only patterns into this design unless there is a specific compatibility requirement.

---

# 48. Pod Identity vs IRSA

You may encounter:

```text
IRSA
```

in older EKS implementations.

IRSA:

```text
ServiceAccount
   ↓
OIDC
   ↓
IAM Role
```

Our current project prefers:

```text
EKS Pod Identity
```

where supported.

The key point is:

> Both provide workload-level AWS identity without embedding static AWS keys, but they are different implementation mechanisms.

---

# 49. Node Role vs Pod Role

Another critical distinction.

## Node IAM Role

The EC2 worker node may need permissions for:

```text
ECR image pull
EKS node operations
VPC networking components depending on configuration
```

## Pod IAM Role

The application receives:

```text
SQS
S3
Secrets Manager
```

permissions.

Therefore:

```text
Node
  |
  +--> infrastructure permissions

Pod
  |
  +--> application permissions
```

Don't put application permissions into the node role.

---

# 50. Why Node Role Must Not Have Application Access

Bad:

```text
Node IAM Role
   |
   +--> S3 entire company bucket
   +--> Secrets Manager *
   +--> SQS *
```

Then:

```text
Any compromised pod on node
        |
        v
Potentially exploit node credential path
        |
        v
Huge AWS access
```

Instead:

```text
Node Role
   |
   +--> node-level requirements

Pod Role
   |
   +--> exact application requirements
```

---

# 51. Kubernetes API Access From Applications

Our normal application:

```text
order-service
```

doesn't need:

```text
GET /api/v1/namespaces
```

or:

```text
list secrets
```

Therefore:

```yaml
automountServiceAccountToken: false
```

is appropriate for the application unless a specific dependency requires Kubernetes API access.

This is another defense layer.

---

# 52. Permission Review Example

Suppose Order Service originally has:

```text
SQS SendMessage
S3 GetObject
S3 PutObject
SecretsManager GetSecretValue
```

Six months later, the application no longer uploads documents.

We should remove:

```text
s3:PutObject
```

This is:

```text
Permission lifecycle management
```

not a one-time Terraform exercise.

---

# 53. Production IAM Review

We should periodically review:

```text
[ ] unused permissions
[ ] wildcard resources
[ ] wildcard actions
[ ] unused roles
[ ] cross-account access
[ ] trust policies
[ ] public resource policies
[ ] service account mappings
[ ] CI roles
[ ] ArgoCD permissions
```

---

# 54. Complete Identity Flow

For a production deployment:

```text
                    HUMAN
                      |
                 AWS Identity
                      |
                 EKS Access
                      |
               Kubernetes RBAC
                      |
                     EKS
                      |
       +--------------+--------------+
       |                             |
    ArgoCD                      Application
       |                             |
 Kubernetes RBAC              ServiceAccount
                                     |
                                Pod Identity
                                     |
                                  IAM Role
                                     |
                   +-----------------+----------------+
                   |                 |                |
                  SQS               S3          Secrets Manager
```

For CI:

```text
GitHub Actions
      |
   OIDC
      |
 AWS STS
      |
 IAM Role
      |
     ECR
```

---

# 55. Production Access Matrix

A useful final model:

| Identity          | Kubernetes          | AWS                      | Scope                 |
| ----------------- | ------------------- | ------------------------ | --------------------- |
| Developer         | RBAC                | Limited/none             | Dev namespace         |
| SRE               | RBAC                | Operational AWS          | Approved environments |
| Platform Engineer | Broad RBAC          | Infrastructure AWS       | Platform              |
| Security Team     | Audit/RBAC          | Security services        | Security              |
| GitHub Actions    | No production admin | ECR                      | CI                    |
| ArgoCD            | Deployment RBAC     | Only required AWS access | GitOps                |
| Order Service     | Minimal/no K8s API  | SQS/S3/Secrets           | Own resources         |
| Payment Service   | Minimal/no K8s API  | Payment resources        | Own resources         |

---

# 56. Troubleshooting: `Forbidden`

Suppose:

```bash
kubectl get pods -n ecommerce-prod
```

returns:

```text
Error from server (Forbidden)
```

First ask:

```text
Who am I?
```

Then:

```bash
kubectl auth whoami
```

Then test permission:

```bash
kubectl auth can-i get pods \
  -n ecommerce-prod
```

If false:

```text
Identity
   ↓
RoleBinding?
   ↓
Role?
   ↓
Permission?
```

---

# 57. Troubleshooting: Can I Access Secrets?

Test:

```bash
kubectl auth can-i get secrets \
  -n ecommerce-prod
```

If:

```text
no
```

that may be correct.

Don't automatically add:

```text
get secrets
```

because someone reports a permissions error.

Determine why the user needs that access.

---

# 58. Troubleshooting: Pod Gets AWS AccessDenied

Check:

```text
Pod
 ↓
ServiceAccount
 ↓
Pod Identity Association
 ↓
IAM Role
 ↓
IAM Policy
 ↓
AWS Resource
```

For example:

```text
order-service
   |
   v
order-service ServiceAccount
   |
   v
order-service IAM Role
   |
   v
sqs:SendMessage
   |
   v
order-events
```

If any link is wrong:

```text
AccessDenied
```

---

# 59. Troubleshooting: Wrong IAM Role

A common production mistake is:

```text
order-service
      |
      v
default ServiceAccount
      |
      v
no Pod Identity association
```

or:

```text
order-service
      |
      v
payment-service IAM role
```

This is why labels, ServiceAccounts and IAM mappings must be explicit.

---

# 60. Production Security Rules

For our project:

```text
RULE 1
No static AWS credentials in pods.

RULE 2
Each application gets its own ServiceAccount.

RULE 3
Each AWS-enabled application gets its own IAM role.

RULE 4
IAM permissions are resource-scoped.

RULE 5
Kubernetes RBAC is namespace-scoped whenever possible.

RULE 6
Avoid cluster-admin for application identities.

RULE 7
CI does not get EKS production admin.

RULE 8
ArgoCD production access is controlled by AppProjects/RBAC.

RULE 9
Node IAM roles do not contain application permissions.

RULE 10
Review permissions continuously.
```

---

# 61. Complete Order Service Identity Architecture

Our Order Service now looks like:

```text
                     GitHub
                        |
                  Pull Request
                        |
                        v
                  GitHub Actions
                        |
                    GitHub OIDC
                        |
                        v
                 AWS IAM CI Role
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
                 Kubernetes RBAC
                        |
                        v
               ecommerce-prod
                        |
                        v
                order-service Pod
                        |
                ServiceAccount
                        |
                 Pod Identity
                        |
                        v
              Order Service IAM Role
                        |
        +---------------+---------------+
        |               |               |
       SQS             S3        Secrets Manager
```

Meanwhile:

```text
Order Service
      |
      +--> RDS
      |
      +--> Redis
      |
      +--> other microservices
```

are controlled through the appropriate:

```text
NetworkPolicy
Security Groups
Database authentication
Application authentication
```

not simply by granting broad IAM permissions.

---

# 62. 21-Microservice Permission Architecture

Eventually:

```text
                        EKS
                         |
       +-----------------+------------------+
       |                 |                  |
 order-service      payment-service    inventory-service
       |                 |                  |
 ServiceAccount     ServiceAccount     ServiceAccount
       |                 |                  |
 Pod Identity       Pod Identity       Pod Identity
       |                 |                  |
 IAM Role           IAM Role           IAM Role
       |                 |                  |
       v                 v                  v
 SQS/S3/Secrets     Payment APIs       SQS/DynamoDB/etc.
```

Every service gets:

```text
unique identity
unique role
unique permissions
unique resources
```

where required.

---

# 63. Final Identity Model

The entire project now follows:

```text
AUTHENTICATION
      ↓
Who are you?
      ↓
AUTHORIZATION
      ↓
What can you do?
      ↓
SCOPE
      ↓
Where can you do it?
      ↓
AUDIT
      ↓
What did you actually do?
```

For AWS:

```text
IAM
```

For Kubernetes:

```text
RBAC
```

For application workloads:

```text
ServiceAccount + Pod Identity
```

For CI:

```text
OIDC
```

For GitOps:

```text
ArgoCD RBAC + AppProjects
```

---

# 64. Production Checklist

## Human Access

```text
[✓] Individual identities
[✓] No shared admin accounts
[✓] Group-based access
[✓] Namespace-level access where possible
[✓] Production access controlled
[✓] Auditability
```

## Kubernetes

```text
[✓] Dedicated ServiceAccounts
[✓] Role/RoleBinding
[✓] Minimal permissions
[✓] No unnecessary secrets access
[✓] No application cluster-admin
[✓] automountServiceAccountToken disabled where unnecessary
```

## AWS

```text
[✓] Pod Identity
[✓] Dedicated IAM role
[✓] Least privilege
[✓] Resource-specific ARNs
[✓] No static credentials
[✓] Node role separated from workload role
```

## CI/CD

```text
[✓] GitHub OIDC
[✓] No long-lived AWS keys
[✓] ECR-only CI permissions
[✓] No EKS admin from CI
[✓] GitHub App for GitOps automation
```

## ArgoCD

```text
[✓] AppProjects
[✓] Repository restrictions
[✓] Namespace restrictions
[✓] Production RBAC
[✓] Controlled production sync
```

---

# 65. Final Architecture

```text
                         COMPANY IDENTITY
                                |
                    +-----------+-----------+
                    |                       |
                 Humans                  GitHub
                    |                       |
              AWS Identity              OIDC
                    |                       |
                    v                       v
              EKS Access               AWS IAM
                    |                       |
              Kubernetes RBAC              ECR
                    |                       |
                    +-----------+-----------+
                                |
                               EKS
                                |
              +-----------------+-----------------+
              |                                   |
           ArgoCD                            Applications
              |                                   |
       Kubernetes RBAC                      ServiceAccount
                                                  |
                                             Pod Identity
                                                  |
                                             IAM Role
                                                  |
                        +-------------------------+------------------+
                        |                         |                  |
                       SQS                       S3            Secrets Manager
```

This gives us **separate identities, least privilege, controlled trust relationships, and clear audit boundaries** across the entire platform.

---

# 66. Where We Are Now

Security phase:

```text
85  Production Networking                    ✅
86  Kubernetes Security Hardening            ✅
87  Container Image Security / SBOM / Signing ✅
88  Admission Control                         ✅
89  Kubernetes RBAC + AWS IAM                 ✅
```

Next:

# 90 — Secrets, KMS and Secret Rotation

We'll implement the complete production secret lifecycle:

```text
Application Secret
       ↓
AWS Secrets Manager
       ↓
KMS Encryption
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Order Service
       ↓
Secret Rotation
       ↓
Pod/Application Reload
```

We'll cover **RDS credentials, Redis credentials, API keys, TLS secrets, GitHub/CI secrets, Terraform state exposure, KMS policies, rotation, failure scenarios, and how to prevent secrets from appearing in Git, Docker images, Helm values, logs, metrics, or traces.**
