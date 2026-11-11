# 27-GitOps-Repository.md

````markdown
# 27 — GitOps Repository

## 1. Overview

GitOps is the operating model where the desired state of the Kubernetes environment is stored in Git and continuously reconciled into the cluster.

Instead of:

Developer -> CI -> kubectl apply -> Production

we use:

Developer
   |
   v
Application Repository
   |
   v
CI Pipeline
   |
   +--> Build
   +--> Test
   +--> Security Scan
   +--> Docker Image
   |
   v
Amazon ECR
   |
   v
GitOps Repository
   |
   v
Pull Request / Approval
   |
   v
ArgoCD
   |
   v
EKS
   |
   v
Kubernetes Workloads


The important principle is:

> Git describes what the environment should look like. ArgoCD continuously makes the cluster match Git.


---

# 2. Why GitOps?

Traditional deployment can look like:

```text
Developer
   |
   v
CI
   |
   v
kubectl apply
   |
   v
EKS
````

This creates several operational problems.

For example:

```text
Who deployed this?
Which commit?
Which image?
Which configuration?
Why is production different from Git?
Who changed the deployment?
What happened during rollback?
```

If someone manually runs:

```bash
kubectl edit deployment order-service
```

the cluster changes without Git knowing about it.

This creates configuration drift.

GitOps changes the model:

```text
Git
 |
 | desired state
 v
ArgoCD
 |
 | reconciliation
 v
EKS
```

Git becomes the source of truth for Kubernetes application configuration.

---

# 3. Core GitOps Principles

A production GitOps implementation normally follows these principles.

## 3.1 Declarative

Describe the desired state.

Example:

```yaml
replicas: 3
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

Kubernetes determines how to reach that state.

---

## 3.2 Version Controlled

Deployment configuration is stored in Git.

Example:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

Every change creates a Git commit.

---

## 3.3 Auditable

A production deployment should be traceable through:

```text
Developer
   |
   v
Application commit
   |
   v
CI build
   |
   v
Image digest
   |
   v
GitOps commit
   |
   v
ArgoCD sync
   |
   v
EKS
```

This creates an audit chain.

---

## 3.4 Reconciled

ArgoCD continuously compares:

```text
Git desired state
        |
        v
      ArgoCD
        |
        v
Cluster live state
```

If they differ, ArgoCD reports drift and can reconcile the cluster depending on the configured sync policy.

---

# 4. Repository Architecture

Our enterprise project uses four repository categories.

```text
GitHub Organization
company-ecommerce
│
├── Application Repositories
│   ├── auth-service
│   ├── user-service
│   ├── customer-service
│   ├── catalog-service
│   ├── product-service
│   ├── inventory-service
│   ├── pricing-service
│   ├── promotion-service
│   ├── cart-service
│   ├── order-service
│   ├── payment-service
│   ├── fraud-service
│   ├── shipping-service
│   ├── delivery-service
│   ├── notification-service
│   ├── search-service
│   ├── recommendation-service
│   ├── review-service
│   ├── tax-service
│   ├── invoice-service
│   └── reporting-service
│
├── infrastructure
│
├── gitops
│
└── platform
```

Each repository has a different responsibility.

---

# 5. Application Repository

Example:

```text
order-service/
```

Responsible for application source code.

```text
order-service/
├── src/
├── pom.xml
├── Dockerfile
├── .dockerignore
├── README.md
├── deploy/
├── helm/
└── .github/
    └── workflows/
        └── ci.yaml
```

The application repository should primarily answer:

> How is this application built?

It should not be the primary source of truth for production environment deployment state in our GitOps architecture.

---

# 6. Infrastructure Repository

Example:

```text
infrastructure/
```

Responsible for AWS infrastructure.

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   ├── iam/
    │   ├── ecr/
    │   ├── rds/
    │   ├── redis/
    │   ├── sqs/
    │   ├── sns/
    │   ├── s3/
    │   └── kms/
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

Terraform manages infrastructure such as:

```text
VPC
EKS
RDS
ElastiCache
ECR
SQS
SNS
IAM
KMS
S3
Security Groups
Route 53
```

Terraform should not compete with ArgoCD for ownership of application Kubernetes resources.

---

# 7. GitOps Repository

The GitOps repository contains desired application deployment state.

Example:

```text
gitops/
├── applications/
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
└── README.md
```

This repository answers:

> What should be running in each Kubernetes environment?

---

# 8. Platform Repository

The platform repository contains shared Kubernetes platform components.

Examples:

```text
platform/
├── ingress/
├── cert-manager/
├── external-secrets/
├── prometheus/
├── grafana/
├── fluent-bit/
├── logstash/
├── elasticsearch/
├── kibana/
├── opentelemetry/
├── jaeger/
└── policies/
```

Platform ownership normally belongs to:

```text
platform-team
```

Application deployment ownership belongs primarily to application/platform teams depending on organizational design.

---

# 9. What Goes Where?

| Responsibility              | Repository                                            |
| --------------------------- | ----------------------------------------------------- |
| Java source code            | Application repo                                      |
| Unit tests                  | Application repo                                      |
| Dockerfile                  | Application repo                                      |
| CI pipeline                 | Application repo                                      |
| Terraform VPC               | Infrastructure repo                                   |
| Terraform EKS               | Infrastructure repo                                   |
| Terraform RDS               | Infrastructure repo                                   |
| Helm chart                  | GitOps/platform/app repo depending on ownership model |
| Environment values          | GitOps repo                                           |
| ArgoCD Application          | GitOps repo                                           |
| Prometheus                  | Platform repo                                         |
| Grafana                     | Platform repo                                         |
| Fluent Bit                  | Platform repo                                         |
| External Secrets            | Platform repo                                         |
| Production image version    | GitOps repo                                           |
| Kubernetes deployment state | GitOps repo                                           |

---

# 10. Recommended GitOps Repository Structure

For our project:

```text
gitops/
│
├── applications/
│   ├── order-service/
│   ├── payment-service/
│   ├── catalog-service/
│   └── ...
│
├── environments/
│   ├── dev/
│   │   ├── order-service/
│   │   │   └── values.yaml
│   │   ├── payment-service/
│   │   │   └── values.yaml
│   │   └── catalog-service/
│   │       └── values.yaml
│   │
│   ├── qa/
│   │
│   ├── staging/
│   │
│   └── prod/
│       ├── order-service/
│       │   └── values.yaml
│       ├── payment-service/
│       │   └── values.yaml
│       └── catalog-service/
│           └── values.yaml
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
└── README.md
```

---

# 11. Environment Separation

We have:

```text
DEV
 |
 v
QA
 |
 v
STAGING
 |
 v
PRODUCTION
```

The important point is that environments are not represented by separate application branches.

We do not use:

```text
develop
qa
staging
production
```

branches in every application repository.

Instead:

```text
Application main
       |
       v
      CI
       |
       v
Immutable image
       |
       v
GitOps promotion
```

Environment differences are represented in GitOps configuration.

---

# 12. Example Order Service Promotion

Suppose CI builds:

```text
order-service
commit: 8a73b91
```

The resulting image is:

```text
order-service:8a73b91
```

and digest:

```text
sha256:abc123...
```

CI publishes it to ECR.

DEV GitOps configuration becomes:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

After validation:

```text
DEV
 |
 v
QA
 |
 v
STAGING
 |
 v
PROD
```

The same image digest is promoted.

---

# 13. Build Once, Promote Many

This is one of the most important production principles.

Bad approach:

```text
DEV -> build image
QA  -> build image
PROD -> build image
```

This could result in:

```text
DEV  = image A
QA   = image B
PROD = image C
```

Even if the source commit is supposedly the same, the artifacts could differ.

Preferred approach:

```text
Source
  |
  v
CI build
  |
  v
ONE immutable artifact
  |
  +----> DEV
  |
  +----> QA
  |
  +----> STAGING
  |
  +----> PROD
```

Example:

```text
sha256:abc123...
```

must remain the same throughout promotion.

---

# 14. Image Tags vs Image Digests

Tags are convenient:

```yaml
image:
  tag: 1.5.2
```

But tags can potentially be moved.

Digest references are immutable:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

Production GitOps should preferably pin the image digest.

Conceptually:

```text
Tag
 |
 v
order-service:1.5.2
 |
 v
sha256:abc123
```

GitOps should deploy:

```text
sha256:abc123
```

when strict artifact immutability is required.

---

# 15. Helm and GitOps

Our GitOps deployment uses Helm.

A chart might look like:

```text
order-service/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── pdb.yaml
```

Environment configuration lives separately.

Example:

```text
environments/
├── dev/
│   └── order-service/
│       └── values.yaml
│
├── qa/
│   └── order-service/
│       └── values.yaml
│
└── prod/
    └── order-service/
        └── values.yaml
```

---

# 16. Development Values

Example:

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
```

Development should not automatically use production-sized infrastructure.

---

# 17. Production Values

Example:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: 2
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

Production configuration should be reviewed like application code.

---

# 18. Configuration vs Secrets

GitOps should store configuration references.

Example:

```yaml
env:
  - name: DB_URL
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: DB_URL
```

Git should NOT contain:

```yaml
password: SuperSecretPassword123
```

Instead:

```text
GitOps
   |
   v
ExternalSecret
   |
   v
AWS Secrets Manager
   |
   v
Kubernetes Secret
   |
   v
Pod
```

This is covered in detail in:

```text
26-Secrets-Management.md
```

---

# 19. Never Store Production Secrets in Git

Do not commit:

```text
password
API keys
AWS access keys
database credentials
private keys
OAuth secrets
JWT signing keys
```

Even a private Git repository should not be treated as a secrets manager.

Bad:

```yaml
env:
  - name: DB_PASSWORD
    value: "MyProductionPassword"
```

Good:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: DB_PASSWORD
```

---

# 20. GitOps Promotion

A common production promotion flow:

```text
Application Developer
        |
        v
Application PR
        |
        v
CI
        |
        v
ECR
        |
        v
GitOps DEV PR
        |
        v
DEV
        |
        v
Validation
        |
        v
GitOps QA PR
        |
        v
QA
        |
        v
Testing
        |
        v
STAGING
        |
        v
Production approval
        |
        v
PROD
```

The application build does not happen again during promotion.

---

# 21. Who Changes GitOps?

There are several valid models.

## Model 1 — CI opens GitOps PR

```text
Application CI
      |
      v
ECR
      |
      v
GitOps PR
```

For example, CI updates:

```yaml
digest: sha256:abc123
```

and creates a PR.

This is highly auditable.

---

## Model 2 — Release engineer changes GitOps

An engineer manually creates the PR:

```text
Update order-service image to sha256:abc123
```

This provides stronger human control but introduces manual work.

---

## Model 3 — Automated image updater

A controller can detect new approved images and update Git.

However, production environments should still enforce:

```text
security policy
approval
promotion rules
```

rather than blindly deploying every newly pushed image.

---

# 22. Recommended Model for This Project

We use:

```text
CI
 |
 +--> Build
 +--> Test
 +--> Scan
 +--> Push ECR
 |
 v
Create GitOps PR
 |
 v
DEV
 |
 v
Validation
 |
 v
QA/STAGING
 |
 v
Production approval
 |
 v
PROD
```

This provides:

```text
Automation
+
Auditability
+
Human approval
+
Immutable artifact promotion
```

---

# 23. Example GitOps Change

Before:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:old123
```

New release:

```text
order-service
version: 1.6.0
digest: sha256:new456
```

GitOps PR:

```diff
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
-  digest: sha256:old123
+  digest: sha256:new456
```

PR description:

```text
Service: order-service

Version: 1.6.0

Application commit:
8a73b91

Image:
order-service

Digest:
sha256:new456

CI:
PASS

Unit tests:
PASS

SonarQube:
PASS

Gitleaks:
PASS

Trivy:
PASS

SBOM:
Generated

DEV validation:
PASS
```

This is much easier to audit than:

```bash
kubectl set image deployment/order-service ...
```

---

# 24. GitOps Branch Strategy

For the GitOps repository, avoid unnecessary branch complexity.

Recommended:

```text
main
```

is the authoritative branch.

Changes happen through PRs.

Example:

```text
main
 |
 +-- PR: promote order-service to QA
 |
 +-- PR: promote payment-service to PROD
 |
 +-- PR: increase catalog-service replicas
 |
 +-- PR: update ingress configuration
```

Short-lived branches are preferred:

```text
promote/order-service-1.6.0
config/payment-service-timeout
```

After merge:

```text
branch -> PR -> main
```

---

# 25. Production Protection

GitOps `main` should be protected.

Recommended controls:

```text
Pull Request required
Required approvals
CODEOWNERS
Required CI checks
No force push
No direct production changes
Resolved conversations
Signed commits where required
```

Production changes should never depend on:

```bash
git push --force
```

or manual edits to the cluster.

---

# 26. CODEOWNERS

Example:

```text
/environments/prod/       @company-ecommerce/platform-team
/environments/staging/    @company-ecommerce/platform-team
/environments/qa/         @company-ecommerce/backend-team @company-ecommerce/platform-team
/argocd/                  @company-ecommerce/platform-team
/applications/            @company-ecommerce/platform-team
```

For sensitive platform configuration:

```text
/argocd/projects/         @company-ecommerce/platform-team @company-ecommerce/security-team
```

This prevents arbitrary users from changing critical deployment configuration.

---

# 27. GitOps Repository Access

Example access model:

```text
Backend Team
   |
   +--> Application repos: Write
   +--> GitOps: PR access

Platform Team
   |
   +--> GitOps: Admin/Maintainer
   +--> Platform repo: Admin/Maintainer

DevOps Team
   |
   +--> Infrastructure: Admin/Maintainer
   +--> GitOps: Maintainer

Security Team
   |
   +--> Security configuration/review
```

Use least privilege.

---

# 28. GitOps and Terraform Boundary

This is extremely important.

Terraform manages:

```text
AWS infrastructure
```

ArgoCD manages:

```text
Kubernetes application state
```

Example:

```text
Terraform
 |
 +--> VPC
 +--> EKS
 +--> RDS
 +--> ECR
 +--> IAM
 +--> SQS
 +--> SNS
 +--> ElastiCache
 |
 v
AWS
```

ArgoCD:

```text
Git
 |
 v
ArgoCD
 |
 +--> Deployment
 +--> Service
 +--> ConfigMap
 +--> HPA
 +--> PDB
 +--> NetworkPolicy
 |
 v
EKS
```

Do not have both Terraform and ArgoCD trying to continuously manage the same Deployment.

---

# 29. What Terraform Creates for EKS?

Terraform can create:

```text
VPC
Subnets
Route Tables
NAT Gateways
EKS Cluster
Node Groups
IAM
Security Groups
ECR
RDS
ElastiCache
SQS
SNS
KMS
```

It can also bootstrap platform components where appropriate.

But application deployment should transition to GitOps.

---

# 30. What ArgoCD Deploys?

ArgoCD can deploy:

```text
order-service
payment-service
catalog-service
inventory-service
auth-service
...
```

plus their Kubernetes resources:

```text
Deployment
Service
ConfigMap
ServiceAccount
HPA
PDB
NetworkPolicy
Ingress
ExternalSecret
```

depending on platform architecture.

---

# 31. GitOps Desired State

Suppose Git says:

```yaml
replicas: 3
```

Cluster currently has:

```text
3 Pods
```

Everything is synchronized.

Now someone manually runs:

```bash
kubectl scale deployment order-service --replicas=1
```

Cluster becomes:

```text
1 Pod
```

Git still says:

```text
3 Pods
```

Therefore:

```text
Desired state = 3
Live state    = 1
```

This is drift.

---

# 32. Drift Detection

ArgoCD detects:

```text
Git:
replicas = 3

Cluster:
replicas = 1
```

Status:

```text
OutOfSync
```

Depending on policy:

```text
Manual sync
```

or:

```text
Automated sync
```

can restore the desired state.

---

# 33. Why Drift Detection Matters

Imagine someone manually changes:

```text
replicas
CPU limits
image
environment variable
service configuration
```

without creating a Git change.

Later, another engineer asks:

```text
Why is production configured this way?
```

GitOps provides the answer:

```text
It should not be this way.
```

The Git history becomes the authoritative record.

---

# 34. Manual kubectl in Production

`kubectl` is still necessary for:

```text
Troubleshooting
Debugging
Read-only investigation
Emergency operations
```

Examples:

```bash
kubectl get pods -n production

kubectl describe pod order-service-xxxxx -n production

kubectl logs deployment/order-service -n production

kubectl get events -n production
```

But avoid:

```bash
kubectl apply -f production.yaml
```

as the normal deployment process.

---

# 35. Emergency Production Change

Sometimes an emergency requires direct intervention.

Example:

```text
Critical production incident
```

An SRE may temporarily change:

```text
replicas
```

or:

```text
resource limits
```

The change must then be reconciled back into Git.

Correct process:

```text
Emergency kubectl change
        |
        v
Stabilize production
        |
        v
Create Git change
        |
        v
PR/review
        |
        v
Git becomes correct again
```

Never leave emergency changes undocumented.

---

# 36. ArgoCD Application Concept

ArgoCD needs to know:

```text
Where is Git?
What path?
Which cluster?
Which namespace?
Which project?
How should it sync?
```

Conceptually:

```yaml
spec:
  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    path: environments/prod/order-service

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

The complete ArgoCD implementation is covered later in:

```text
29-ArgoCD.md
```

---

# 37. ApplicationSet Concept

With 21 microservices, manually creating 21 ArgoCD Applications can become repetitive.

ApplicationSet can generate Applications from a common pattern.

Conceptually:

```text
ApplicationSet
       |
       +--> order-service
       +--> payment-service
       +--> catalog-service
       +--> inventory-service
       +--> shipping-service
       +--> ...
```

This becomes particularly useful when managing:

```text
21 services
+
multiple environments
+
multiple clusters
```

---

# 38. Multi-Cluster GitOps

Enterprise environments may eventually have:

```text
EKS DEV
EKS QA
EKS STAGING
EKS PROD
```

or:

```text
AWS Account
 |
 +--> DEV Account
 |      |
 |      +--> EKS
 |
 +--> QA Account
 |      |
 |      +--> EKS
 |
 +--> PROD Account
        |
        +--> EKS
```

GitOps can manage all of them.

Example:

```text
GitOps
 |
 +--> DEV cluster
 |
 +--> QA cluster
 |
 +--> STAGING cluster
 |
 +--> PROD cluster
```

The deployment artifact remains the same.

---

# 39. Multi-Account Architecture

A stronger enterprise model:

```text
AWS Organization
│
├── Security Account
├── Logging Account
├── Management Account
├── Dev Account
├── QA Account
└── Production Account
```

Each environment can have:

```text
Separate AWS account
Separate EKS cluster
Separate IAM boundary
Separate networking
```

GitOps controls the Kubernetes layer.

Terraform controls AWS infrastructure.

---

# 40. Production Environment Separation

Example:

```text
DEV
AWS Account: Dev
EKS: ecommerce-dev

QA
AWS Account: QA
EKS: ecommerce-qa

STAGING
AWS Account: Staging
EKS: ecommerce-staging

PROD
AWS Account: Production
EKS: ecommerce-prod
```

The same application image can be deployed to all environments.

---

# 41. Promotion Across Accounts

Example:

```text
CI
 |
 v
ECR
 |
 v
Image digest
sha256:abc123
 |
 +----------------+
 |                |
 v                v
DEV Account     QA Account
 |                |
 +--------+-------+
          |
          v
     STAGING
          |
          v
      PROD
```

If using separate ECR repositories/accounts, the artifact promotion mechanism must preserve the same artifact identity, often by controlled ECR replication/copying and digest verification.

The exact implementation depends on the organization's AWS account and artifact strategy.

---

# 42. GitOps Audit Trail

Suppose production currently runs:

```text
order-service
version: 1.6.0
digest: sha256:abc123
```

We should be able to determine:

```text
Application commit:
8a73b91

CI build:
#1827

Image:
order-service@sha256:abc123

SBOM:
build-1827

Security scan:
PASS

GitOps commit:
4fd829a

ArgoCD:
Synced

Cluster:
ecommerce-prod

Namespace:
production
```

This is a strong production audit chain.

---

# 43. Rollback

Suppose version `1.6.0` has a production issue.

Git history:

```text
commit A -> order-service 1.5.2
commit B -> order-service 1.6.0
```

Current:

```text
1.6.0
```

Rollback:

```text
Git revert B
```

Result:

```text
1.5.2
```

ArgoCD detects the desired state and deploys the previous image.

---

# 44. Why Git Revert Is Better Than Manual Rollback

Manual:

```bash
kubectl rollout undo deployment/order-service
```

may restore Kubernetes state temporarily.

But Git still says:

```text
1.6.0
```

Now:

```text
Git != Cluster
```

With GitOps:

```text
Git
 |
 | revert
 v
desired state = 1.5.2
 |
 v
ArgoCD
 |
 v
EKS
```

Now:

```text
Git = Cluster
```

---

# 45. Rollback Does Not Mean Database Rollback

This is a critical production concept.

Application rollback:

```text
1.6.0 -> 1.5.2
```

does not automatically mean:

```text
database schema rollback
```

For example:

```text
1.6.0
 |
 +--> adds database column
```

If 1.5.2 cannot work with the changed schema, simply rolling back the image may fail.

Therefore database migrations must be designed for backward compatibility.

Typical strategy:

```text
Expand
 |
 v
Deploy compatible application
 |
 v
Migrate data
 |
 v
Contract
```

Database migration strategy is covered in the RDS/database chapters.

---

# 46. GitOps and Database Migrations

Do not blindly make application startup execute destructive migrations.

Avoid production:

```yaml
ddl-auto: update
```

Instead:

```text
Migration tool
 |
 +--> Flyway
 |
 or
 |
 +--> Liquibase
```

A controlled release can look like:

```text
Migration
   |
   v
Compatibility validation
   |
   v
Application deployment
```

The exact orchestration depends on the application architecture.

---

# 47. GitOps and Configuration Changes

GitOps is not only for image versions.

It can manage:

```text
replicas
CPU
memory
HPA
PDB
environment variables
ConfigMaps
Ingress
Service
NetworkPolicy
feature configuration
timeouts
resource limits
```

Example:

```diff
 resources:
   requests:
-    cpu: 250m
+    cpu: 500m
```

A simple PR can therefore represent an operational change.

---

# 48. Configuration Review

Suppose production latency increases.

An engineer wants to increase:

```yaml
resources:
  requests:
    cpu: 500m
```

Instead of directly editing Kubernetes:

```bash
kubectl edit deployment
```

create:

```text
PR:
Increase order-service CPU request
```

Reviewers can inspect:

```text
Why?
Expected impact?
Cost impact?
Capacity available?
HPA interaction?
```

Then merge.

---

# 49. GitOps and Autoscaling

GitOps manages the HPA configuration:

```yaml
autoscaling:
  minReplicas: 3
  maxReplicas: 10
```

But the HPA controller dynamically changes the actual replica count.

This is not necessarily drift in the same sense as manually changing the Deployment's desired configuration.

Conceptually:

```text
Git
 |
 | HPA policy
 v
ArgoCD
 |
 v
HPA
 |
 v
Deployment
 |
 v
Pods
```

HPA dynamically calculates replicas according to metrics.

---

# 50. GitOps and Kubernetes Controllers

Kubernetes contains many controllers.

For example:

```text
Deployment Controller
HPA Controller
Service Controller
AWS Load Balancer Controller
External Secrets Controller
ArgoCD
```

GitOps does not replace these controllers.

Instead:

```text
GitOps
   |
   v
Desired Kubernetes resources
   |
   v
Kubernetes controllers
   |
   v
Actual infrastructure/workloads
```

---

# 51. GitOps and External Secrets

Example:

```text
Git
 |
 +--> ExternalSecret definition
 |
 v
ArgoCD
 |
 v
External Secrets Operator
 |
 v
AWS Secrets Manager
 |
 v
Kubernetes Secret
 |
 v
Order Pod
```

Git contains:

```text
secret reference
```

not:

```text
secret value
```

---

# 52. GitOps and Observability

GitOps can deploy observability components.

Example:

```text
platform repo
 |
 +--> Prometheus
 +--> Grafana
 +--> Alertmanager
 +--> Fluent Bit
 +--> Logstash
 +--> Elasticsearch
 +--> Kibana
 +--> OpenTelemetry
 +--> Jaeger
```

This allows platform configuration to be reviewed and version controlled.

---

# 53. GitOps Repository Example

A realistic final structure:

```text
gitops/
│
├── applications/
│   ├── order-service/
│   │   └── Chart.yaml
│   │
│   ├── payment-service/
│   │   └── Chart.yaml
│   │
│   └── catalog-service/
│       └── Chart.yaml
│
├── environments/
│   │
│   ├── dev/
│   │   ├── order-service/
│   │   │   └── values.yaml
│   │   ├── payment-service/
│   │   │   └── values.yaml
│   │   └── catalog-service/
│   │       └── values.yaml
│   │
│   ├── qa/
│   │   └── ...
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── prod/
│       ├── order-service/
│       │   └── values.yaml
│       ├── payment-service/
│       │   └── values.yaml
│       └── catalog-service/
│           └── values.yaml
│
├── argocd/
│   ├── projects/
│   │   ├── ecommerce-dev.yaml
│   │   ├── ecommerce-qa.yaml
│   │   ├── ecommerce-staging.yaml
│   │   └── ecommerce-prod.yaml
│   │
│   ├── applications/
│   │   └── ...
│   │
│   └── appsets/
│       └── ecommerce-services.yaml
│
└── README.md
```

---

# 54. Order-Service Example

For our project:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...

replicaCount: 3

service:
  port: 8080

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

Secrets are referenced separately.

---

# 55. Complete Deployment Flow

For `order-service`:

```text
1. Developer changes Java code
        |
        v
2. Pull Request
        |
        v
3. CI
        |
        +--> Compile
        +--> Unit tests
        +--> SonarQube
        +--> Dependency scan
        +--> Gitleaks
        +--> Docker build
        +--> Trivy
        +--> SBOM
        |
        v
4. Push immutable image to ECR
        |
        v
5. Record image digest
        |
        v
6. Update GitOps repository
        |
        v
7. GitOps Pull Request
        |
        v
8. Review/approval
        |
        v
9. Merge
        |
        v
10. ArgoCD detects desired state
        |
        v
11. ArgoCD sync
        |
        v
12. EKS Deployment
        |
        v
13. Kubernetes creates/updates Pods
        |
        v
14. Readiness probe passes
        |
        v
15. Traffic reaches new Pods
```

This is the complete CI-to-production GitOps model.

---

# 56. What Happens During a Rolling Deployment?

Suppose:

```text
Current:
3 Pods running version 1.5.2
```

Git changes:

```text
1.5.2 -> 1.6.0
```

ArgoCD applies the Deployment change.

Kubernetes performs the rollout.

Conceptually:

```text
1.5.2
1.5.2
1.5.2

      |
      v

1.6.0
1.5.2
1.5.2

      |
      v

1.6.0
1.6.0
1.5.2

      |
      v

1.6.0
1.6.0
1.6.0
```

Readiness probes determine when new Pods are ready to receive traffic.

---

# 57. GitOps Does Not Replace Kubernetes Deployment Strategy

ArgoCD determines:

```text
what configuration should exist
```

Kubernetes Deployment determines:

```text
how Pods are rolled out
```

For example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Later we can introduce:

```text
Blue-Green
Canary
Progressive Delivery
```

using appropriate controllers and traffic-management mechanisms.

---

# 58. Production Sync Strategy

Development may use:

```yaml
automated:
  prune: true
  selfHeal: true
```

Production requires more careful governance.

Possible model:

```text
GitOps PR
   |
   v
Review
   |
   v
Approval
   |
   v
ArgoCD sync
   |
   v
Production
```

For critical organizations, production sync can require an explicit approval or release process.

Do not blindly enable every automation feature just because it is technically possible.

---

# 59. Prune

Suppose Git contains:

```text
deployment.yaml
service.yaml
```

and the Deployment is removed from Git.

With pruning enabled, ArgoCD can remove the corresponding resource from the cluster.

This is powerful but dangerous.

Therefore:

```text
prune = true
```

should be used with careful repository controls and resource ownership boundaries.

---

# 60. Self-Heal

Suppose someone changes:

```text
replicas: 3 -> 1
```

manually.

If self-healing is enabled and the resource is managed by ArgoCD:

```text
Git:
3

Cluster:
1

ArgoCD:
drift detected

       |
       v

Cluster:
3
```

This protects against accidental manual configuration changes.

---

# 61. Common GitOps Failure

### Git changed but deployment did not happen

Check:

```bash
argocd app get order-service-prod
```

Then:

```bash
kubectl get application -n argocd
```

Check:

```text
Application status
Sync status
Health
Repository connectivity
Target revision
Path
Destination
```

Potential causes:

```text
Wrong repository
Wrong branch
Wrong path
Invalid YAML
Helm rendering failure
RBAC issue
Cluster unreachable
Sync disabled
```

---

# 62. Helm Rendering Failure

Example:

```text
ComparisonError
```

Check locally:

```bash
helm template order-service ./helm/order-service \
  -f environments/prod/order-service/values.yaml
```

Look for:

```text
Invalid YAML
Missing value
Invalid template
Wrong API version
Incorrect indentation
```

Never blindly debug only from the UI.

---

# 63. Image Pull Failure

ArgoCD may report:

```text
Synced
```

but Pods may show:

```text
ImagePullBackOff
```

Important distinction:

```text
ArgoCD sync success
```

does not necessarily mean:

```text
Application healthy
```

Check:

```bash
kubectl get pods -n production

kubectl describe pod <pod> -n production
```

Possible causes:

```text
Wrong ECR repository
Wrong digest
ECR permissions
Node IAM issue
Network problem
Image does not exist
Registry authentication issue
```

This is why:

```text
Sync
+
Health
```

must both be considered.

---

# 64. Application Unhealthy After Successful Sync

Example:

```text
ArgoCD:
Synced

Health:
Degraded
```

Check:

```bash
kubectl get pods -n production
kubectl describe pod <pod> -n production
kubectl logs <pod> -n production
kubectl get events -n production
```

Common reasons:

```text
CrashLoopBackOff
Readiness failure
Liveness failure
Wrong environment variable
Database connection failure
Redis failure
Secret missing
Insufficient resources
NetworkPolicy issue
```

GitOps only delivers desired state.

The application still needs to become healthy.

---

# 65. GitOps Troubleshooting Model

Always separate:

```text
Git problem
```

from:

```text
ArgoCD problem
```

from:

```text
Kubernetes problem
```

from:

```text
Application problem
```

Example:

```text
Git
 |
 | correct?
 v
ArgoCD
 |
 | synced?
 v
Kubernetes
 |
 | Pods healthy?
 v
Application
 |
 | dependency healthy?
 v
RDS / Redis / SQS
```

---

# 66. Useful Commands

Check Git:

```bash
git log --oneline
git show <commit>
git diff
```

Check ArgoCD:

```bash
argocd app list

argocd app get order-service-prod

argocd app history order-service-prod
```

Check Kubernetes:

```bash
kubectl get applications -n argocd

kubectl get deployments -n production

kubectl get pods -n production

kubectl describe deployment order-service -n production

kubectl logs deployment/order-service -n production

kubectl get events -n production --sort-by=.lastTimestamp
```

---

# 67. GitOps Security

GitOps repository security is production security.

Protect:

```text
main
```

Require:

```text
PR
Review
CODEOWNERS
CI checks
Approval
```

Use least privilege for:

```text
GitHub users
GitHub Actions
ArgoCD
Kubernetes ServiceAccounts
AWS IAM
```

Never give ArgoCD unnecessary cluster-admin privileges if namespace-scoped permissions are sufficient.

---

# 68. ArgoCD Permissions

ArgoCD itself requires access to deploy resources.

Conceptually:

```text
ArgoCD
 |
 v
Kubernetes API
 |
 +--> Namespace production
 |
 +--> Deployment
 +--> Service
 +--> ConfigMap
 +--> HPA
 +--> PDB
```

Permissions should be designed according to the applications and namespaces ArgoCD is responsible for.

For a multi-team organization, ArgoCD Projects can restrict:

```text
source repositories
destination clusters
destination namespaces
resource types
```

Detailed implementation belongs in:

```text
29-ArgoCD.md
```

---

# 69. GitOps Repository CI

The GitOps repository should also have CI.

Example:

```text
PR
 |
 +--> YAML validation
 +--> Helm lint
 +--> Helm template
 +--> Policy validation
 +--> Security checks
 +--> Secret detection
```

Example commands:

```bash
helm lint ./applications/order-service

helm template order-service \
  ./applications/order-service \
  -f environments/prod/order-service/values.yaml
```

This catches many problems before ArgoCD sees them.

---

# 70. Policy Validation

Production GitOps can enforce policies such as:

```text
No privileged containers
No hostNetwork unless approved
Resource requests required
Resource limits required
Run as non-root
Read-only root filesystem where possible
Approved registries only
Required labels
Required probes
No plaintext secrets
```

Tools can include policy engines such as:

```text
Kyverno
OPA Gatekeeper
Conftest
```

The exact tool can be selected later according to platform requirements.

---

# 71. GitOps Labels

Use consistent Kubernetes labels.

Example:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: argocd
```

Useful labels help with:

```text
Monitoring
Logging
Troubleshooting
Cost allocation
Ownership
Policy
```

Do not rely only on arbitrary labels such as:

```text
app: order
```

---

# 72. Ownership Metadata

Production resources should make ownership obvious.

Example:

```yaml
metadata:
  labels:
    team: backend-team
    service: order-service
    environment: production
```

Additional metadata can include:

```text
cost-center
business-unit
criticality
on-call-team
```

This becomes useful during incidents and cost analysis.

---

# 73. GitOps and Disaster Recovery

GitOps provides an important DR advantage.

Suppose an EKS cluster is destroyed.

Infrastructure can be recreated with:

```text
Terraform
```

Then application state can be recreated from:

```text
GitOps
```

Conceptually:

```text
Terraform
   |
   v
New EKS
   |
   v
ArgoCD
   |
   v
GitOps repository
   |
   v
21 microservices
```

However, GitOps alone does not recover:

```text
RDS data
Redis state
S3 objects
external systems
```

Those require their own backup/restore strategy.

---

# 74. GitOps as Infrastructure Recovery Input

A disaster recovery flow can look like:

```text
AWS infrastructure
       |
       X
   destroyed
       |
       v
Terraform
       |
       v
VPC + EKS recreated
       |
       v
ArgoCD installed
       |
       v
GitOps repository
       |
       v
Applications restored
       |
       v
Secrets restored/reconnected
       |
       v
RDS restored separately
```

This is why GitOps repositories must be treated as critical infrastructure assets.

---

# 75. GitOps Repository Backup

Git itself provides history, but organizations should still protect repositories.

Consider:

```text
Repository backup
Organization backup
Branch protection
Audit logs
Repository mirroring
Access recovery
```

A GitOps repository is effectively a production control-plane input.

---

# 76. What Happens if GitHub Is Down?

This depends on architecture.

Existing workloads can continue running in EKS.

ArgoCD generally works with the desired state it already has locally/cached as appropriate, but new Git changes cannot be retrieved while the Git source is unavailable.

Therefore:

```text
GitHub unavailable
       |
       v
Existing application
       |
       v
Usually continues running
```

But:

```text
New deployment
   |
   X
Git source unavailable
```

This demonstrates why Git hosting availability matters to the deployment control plane.

---

# 77. GitOps and CI Outage

Suppose CI is down.

Existing production workloads continue running.

But:

```text
new source
   |
   X
CI
```

cannot produce a new artifact.

GitOps therefore separates:

```text
Build availability
```

from:

```text
Deployment availability
```

This separation is one of the major advantages of the architecture.

---

# 78. GitOps and ECR Outage

Existing Pods may continue running because they already have images locally.

But new Pods requiring an image pull could fail if ECR/network access is unavailable.

Therefore production resilience must consider:

```text
ECR
AWS networking
NAT/VPC endpoints
EKS nodes
container runtime
```

GitOps cannot eliminate infrastructure dependencies.

---

# 79. Production Change Example

Business request:

```text
Increase order-service capacity before a sale.
```

Engineer creates:

```text
GitOps PR
```

Change:

```yaml
autoscaling:
  minReplicas: 5
  maxReplicas: 20
```

Review:

```text
CPU capacity
Memory capacity
Node scaling
RDS connection capacity
Redis connection capacity
ALB
HPA
Karpenter
Cost
```

After approval:

```text
Git merge
   |
   v
ArgoCD
   |
   v
EKS
```

This is a real production DevOps workflow.

---

# 80. Important Interaction: HPA + Karpenter

GitOps may define:

```text
HPA maxReplicas = 20
```

But if nodes cannot provide capacity:

```text
Pods pending
```

Karpenter or another node scaling mechanism may provision capacity.

The complete flow can become:

```text
Traffic increases
      |
      v
CPU increases
      |
      v
Prometheus metrics
      |
      v
HPA
      |
      v
More Pods
      |
      v
Insufficient node capacity
      |
      v
Karpenter
      |
      v
New EC2 capacity
      |
      v
Pods scheduled
```

GitOps controls the policies.

Runtime controllers perform dynamic scaling.

---

# 81. GitOps Does Not Mean Everything Must Be in One Repository

A common misconception is:

```text
GitOps = one giant repository
```

Not necessarily.

Our architecture intentionally separates concerns:

```text
Application repositories
        |
        v
Application source/build

Infrastructure repository
        |
        v
AWS infrastructure

Platform repository
        |
        v
Shared Kubernetes platform

GitOps repository
        |
        v
Environment desired state
```

This provides clearer ownership and access control.

---

# 82. Alternative GitOps Repository Models

### Model A — Environment-centric

```text
gitops/
├── dev/
├── qa/
├── staging/
└── prod/
```

Simple and easy to understand.

---

### Model B — Application-centric

```text
gitops/
├── order-service/
│   ├── dev/
│   ├── qa/
│   └── prod/
│
└── payment-service/
    ├── dev/
    ├── qa/
    └── prod/
```

Useful when service ownership is strong.

---

### Model C — Hybrid

```text
gitops/
├── applications/
├── environments/
└── argocd/
```

Our project uses the hybrid approach.

---

# 83. Why Hybrid?

We need to manage:

```text
21 microservices
4 environments
potentially multiple EKS clusters
```

A hybrid structure gives:

```text
Application definitions
+
Environment configuration
+
ArgoCD configuration
```

without creating an extremely deep or duplicated structure.

---

# 84. GitOps Deployment Contract

Every application should provide enough information for deployment.

For example:

```text
Service name
Image repository
Image digest
Container port
Environment variables
Resources
Probes
Service configuration
HPA
PDB
ServiceAccount
Network requirements
Secret references
```

For `order-service`:

```text
order-service
   |
   +--> ECR image
   +--> port 8080
   +--> RDS
   +--> Redis
   +--> SQS
   +--> Secrets Manager
   +--> Prometheus metrics
   +--> OpenTelemetry
```

The deployment contract should be consistent across services where possible.

---

# 85. Standardization Across 21 Services

Instead of 21 completely different Helm charts, establish common patterns.

Every service should ideally support:

```text
Deployment
Service
ConfigMap
ServiceAccount
Probes
Resources
HPA
PDB
NetworkPolicy
ExternalSecret
```

with service-specific additions where required.

---

# 86. Golden Helm Pattern

Platform team can create a standard application chart.

Example:

```text
company-service-chart/
```

It can provide:

```text
Deployment
Service
HPA
PDB
ServiceAccount
ConfigMap
NetworkPolicy
```

Applications supply:

```yaml
serviceName: order-service
image:
  repository: ...
  digest: ...
```

This reduces duplication and configuration drift between services.

---

# 87. GitOps and Release Metadata

A GitOps commit should ideally make the release obvious.

Example commit:

```text
deploy(order-service): promote 1.6.0 to production
```

Commit body:

```text
Service: order-service
Environment: production
Version: 1.6.0
Image digest: sha256:abc123
Application commit: 8a73b91
CI build: 1827
```

This makes incident investigation much easier.

---

# 88. Production GitOps Checklist

Before production deployment:

```text
[ ] Application PR approved
[ ] CI passed
[ ] Unit tests passed
[ ] SonarQube passed
[ ] Dependency scan passed
[ ] Secret scan passed
[ ] Trivy passed
[ ] SBOM generated
[ ] Image pushed to ECR
[ ] Image digest recorded
[ ] DEV validated
[ ] QA validated
[ ] STAGING validated
[ ] GitOps PR reviewed
[ ] Production approval obtained
[ ] Rollback plan known
[ ] Database migration compatible
[ ] Capacity checked
[ ] Monitoring available
[ ] Alerts understood
```

Then:

```text
Merge
  |
  v
ArgoCD
  |
  v
Production
```

---

# 89. Production Troubleshooting Checklist

If deployment fails:

```text
1. Check Git commit
2. Check GitOps diff
3. Validate Helm
4. Check ArgoCD Application
5. Check Sync status
6. Check Health status
7. Check Deployment
8. Check ReplicaSet
9. Check Pods
10. Check Events
11. Check image pull
12. Check Secrets
13. Check ConfigMaps
14. Check Service
15. Check NetworkPolicy
16. Check database
17. Check Redis
18. Check messaging
19. Check application logs
20. Check metrics/traces
```

Always troubleshoot from the deployment layer downward.

---

# 90. Final Architecture

The final production deployment model for our project is:

```text
                         GitHub
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
   Application Repos   Infrastructure       Platform
          |               Repo                Repo
          |                 |                  |
          v                 v                  v
         CI              Terraform          GitOps
          |                 |                  |
          v                 v                  |
         ECR              AWS                  |
          |                                   |
          +-------------------+---------------+
                              |
                              v
                         GitOps Repo
                              |
                              v
                           ArgoCD
                              |
                              v
                         EKS Cluster
                              |
        +---------------------+----------------------+
        |                     |                      |
        v                     v                      v
   Microservices         Platform              Observability
        |                     |                      |
        v                     v                      v
   RDS / Redis          Ingress/Secrets       Prometheus/ELK/OTel
        |
        v
    SQS/SNS
```

---

# 91. The Most Important Mental Model

Remember this separation:

```text
Application Repository
        |
        | "How do we build it?"
        v
       CI
        |
        | "Is it safe and valid?"
        v
       ECR
        |
        | "Which immutable artifact?"
        v
   GitOps Repository
        |
        | "What should each environment run?"
        v
      ArgoCD
        |
        | "Make cluster match Git"
        v
       EKS
        |
        | "Actually run it"
        v
     Kubernetes
```

And separately:

```text
Terraform
   |
   v
AWS infrastructure
```

The boundaries are:

```text
Terraform -> AWS infrastructure

CI -> Build/test/security/artifact

ECR -> Immutable container artifacts

GitOps -> Kubernetes desired state

ArgoCD -> Reconciliation

EKS/Kubernetes -> Runtime

Secrets Manager -> Secrets

Prometheus/ELK/OpenTelemetry -> Observability
```

This separation is the foundation of the entire enterprise DevOps architecture.

---

# 92. What We Will Build Next

The next chapter is:

```text
28-Helm.md
```

There we will go deeper into Helm itself:

```text
Chart structure
Chart.yaml
values.yaml
templates
_helpers.tpl
named templates
Go templating
conditionals
loops
default values
environment values
Secrets references
ConfigMaps
Deployments
Services
Ingress
HPA
PDB
ServiceAccounts
NetworkPolicies
Helm lint
helm template
Helm dependency management
Helm packaging
Helm repositories
OCI charts
versioning
production chart design
common Helm mistakes
GitOps + Helm
ArgoCD + Helm
```

After that:

```text
29-ArgoCD.md
```

will cover the ArgoCD control plane and actual production implementation in detail.

```
```
