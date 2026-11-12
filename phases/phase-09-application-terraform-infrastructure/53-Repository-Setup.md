# 53 — Repository Setup

> **Project:** Enterprise E-Commerce Platform
> **Organization:** `company-ecommerce`
> **GitHub:** Private repositories
> **Primary branch:** `main`
> **Deployment model:** GitOps with ArgoCD
> **Infrastructure:** Terraform
> **Cloud:** AWS
> **Runtime:** Amazon EKS

---

## 1. Objective

Before writing application code or Terraform, we need a clean enterprise repository structure.

The repository layer will separate:

```text
Application Code
      │
      ▼
Application Repositories
      │
      ├── CI
      ├── Tests
      ├── Security Scans
      └── Docker Image
                │
                ▼
               ECR
                │
                ▼
          GitOps Repository
                │
                ▼
              ArgoCD
                │
                ▼
               EKS
```

Infrastructure and platform configuration remain separate:

```text
infrastructure/
        │
        └── Terraform
             ├── VPC
             ├── EKS
             ├── IAM
             ├── RDS
             ├── Redis
             ├── ECR
             ├── SQS/SNS
             └── KMS

platform/
        │
        ├── Ingress Controller
        ├── External Secrets
        ├── Prometheus
        ├── Grafana
        ├── ELK
        ├── OpenTelemetry
        └── Jaeger

gitops/
        │
        ├── Dev
        ├── QA
        ├── Staging
        └── Prod
```

---

# 2. GitHub Organization

Create:

```text
company-ecommerce
```

Keep the organization private/internal according to your company's GitHub setup.

The organization will eventually contain:

```text
company-ecommerce/
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
├── reporting-service
│
├── infrastructure
├── gitops
└── platform
```

Total:

```text
21 application repositories
+
3 platform repositories
=
24 repositories
```

---

# 3. Do Not Create Environment Branches

We are **not** going to use:

```text
main
develop
qa
staging
production
```

for every application repository.

Instead:

```text
main
 │
 ├── feature/*
 ├── bugfix/*
 └── hotfix/*
```

Environment promotion happens through GitOps.

Example:

```text
order-service
       │
       ▼
     main
       │
       ▼
CI builds image
       │
       ▼
ECR
       │
       ▼
GitOps DEV
       │
       ▼
GitOps QA
       │
       ▼
GitOps STAGING
       │
       ▼
GitOps PROD
       │
       ▼
ArgoCD
       │
       ▼
EKS
```

This prevents application source branches from becoming deployment environments.

---

# 4. Teams

Create these GitHub teams:

```text
backend-team
platform-team
devops-team
security-team
sre-team
architecture-team
```

### Responsibility

| Team              | Responsibility                         |
| ----------------- | -------------------------------------- |
| backend-team      | Microservices/application code         |
| platform-team     | Kubernetes platform/GitOps/shared Helm |
| devops-team       | AWS/Terraform/infrastructure           |
| security-team     | Security reviews/policies              |
| sre-team          | Reliability/monitoring/operations      |
| architecture-team | Architecture standards/reviews         |

---

# 5. Repository Ownership

| Repository        | Primary owner |
| ----------------- | ------------- |
| Application repos | backend-team  |
| `infrastructure`  | devops-team   |
| `gitops`          | platform-team |
| `platform`        | platform-team |

Security and architecture teams participate through reviews where required.

---

# 6. Repository Naming Convention

Use lowercase kebab-case:

```text
order-service
payment-service
catalog-service
```

Avoid:

```text
OrderService
order_service
Order-Service
orderservice
```

For this project:

```text
company-ecommerce/order-service
company-ecommerce/infrastructure
company-ecommerce/gitops
company-ecommerce/platform
```

---

# 7. First Four Repositories

Do **not** immediately create all 24 repositories.

Start with:

```text
order-service
infrastructure
gitops
platform
```

Why?

Because these four establish the complete deployment path:

```text
Developer
   ↓
order-service
   ↓
GitHub Actions
   ↓
Docker
   ↓
ECR
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
   ↓
RDS
```

Once this works, replicate the pattern for the remaining 20 services.

---

# 8. Create `order-service`

Repository:

```text
company-ecommerce/order-service
```

Initial structure:

```text
order-service/
│
├── .github/
│   ├── CODEOWNERS
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/
│
├── src/
│   ├── main/
│   └── test/
│
├── deploy/
│
├── helm/
│
├── Dockerfile
├── .dockerignore
├── .gitignore
├── .editorconfig
├── CONTRIBUTING.md
├── README.md
└── SECURITY.md
```

We will populate the Java application in the next implementation chapter.

---

# 9. Create `infrastructure`

Repository:

```text
company-ecommerce/infrastructure
```

Initial structure:

```text
infrastructure/
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│
├── terraform/
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
├── README.md
├── CONTRIBUTING.md
└── SECURITY.md
```

Terraform will eventually manage:

```text
AWS
│
├── VPC
├── Subnets
├── NAT
├── VPC Endpoints
├── EKS
├── IAM
├── ECR
├── RDS
├── ElastiCache
├── S3
├── SQS
├── SNS
├── KMS
└── supporting resources
```

---

# 10. Create `gitops`

Repository:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
├── applications/
│   ├── order-service/
│   ├── payment-service/
│   ├── catalog-service/
│   └── ...
│
├── README.md
├── CONTRIBUTING.md
└── SECURITY.md
```

This repository represents:

> **What should be running in Kubernetes?**

It does **not** contain:

```text
AWS infrastructure
application source code
database passwords
AWS access keys
```

---

# 11. Create `platform`

Repository:

```text
company-ecommerce/platform
```

Structure:

```text
platform/
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│
├── helm/
│   ├── aws-load-balancer-controller/
│   ├── external-secrets/
│   ├── kube-prometheus-stack/
│   ├── fluent-bit/
│   ├── logstash/
│   ├── elasticsearch/
│   ├── kibana/
│   ├── opentelemetry/
│   └── jaeger/
│
├── manifests/
│
├── policies/
│
├── README.md
├── CONTRIBUTING.md
└── SECURITY.md
```

This repository contains shared platform components.

For example:

```text
EKS
 │
 ├── AWS Load Balancer Controller
 ├── External Secrets Operator
 ├── Prometheus
 ├── Grafana
 ├── Alertmanager
 ├── Fluent Bit
 ├── Logstash
 ├── Elasticsearch
 ├── Kibana
 ├── OpenTelemetry Collector
 └── Jaeger
```

---

# 12. Standard `.gitignore`

For Java application repositories:

```gitignore
# Maven
target/
!.mvn/wrapper/maven-wrapper.jar

# Java
*.class
*.log

# IDE
.idea/
*.iml
.vscode/
.project
.classpath
.settings/

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.*
!.env.example

# Temporary
tmp/
temp/

# Coverage
coverage/
*.exec

# Local Docker
docker-data/

# Secrets
*.pem
*.key
*.crt
```

Never use `.gitignore` as your only secret protection.

A secret that was committed once should be considered compromised.

---

# 13. Infrastructure `.gitignore`

For Terraform:

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
crash.*.log

# Terraform variables containing secrets
*.tfvars
*.tfvars.json

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI config
.terraformrc
terraform.rc

# OS / IDE
.DS_Store
.idea/
.vscode/
```

Important:

```text
terraform.tfstate
```

must **not** be committed.

Our state will eventually be stored remotely in S3.

---

# 14. `.dockerignore`

For application repositories:

```text
.git
.github
.gitignore
.idea
.vscode
target
*.log
.env
.env.*
README.md
docker-compose.yaml
```

The goal is to prevent unnecessary files from entering the Docker build context.

---

# 15. CODEOWNERS

Create:

```text
.github/CODEOWNERS
```

For `order-service`:

```text
# Application source
/src/       @company-ecommerce/backend-team

# Deployment configuration
/deploy/    @company-ecommerce/platform-team

# Helm
/helm/      @company-ecommerce/platform-team

# CI/CD
/.github/   @company-ecommerce/platform-team

# Security
/security/  @company-ecommerce/security-team
```

This means changes to different areas automatically request reviews from the responsible teams.

For example:

```text
Developer modifies:

src/main/java/...
        │
        ▼
backend-team review
```

while:

```text
Developer modifies:

helm/order-service/...
        │
        ▼
platform-team review
```

---

# 16. Infrastructure CODEOWNERS

```text
/terraform/  @company-ecommerce/devops-team
/scripts/    @company-ecommerce/devops-team
/.github/    @company-ecommerce/devops-team
```

For security-sensitive infrastructure, you can additionally require security review through repository rules.

---

# 17. GitOps CODEOWNERS

```text
/environments/dev/       @company-ecommerce/platform-team
/environments/qa/        @company-ecommerce/platform-team
/environments/staging/   @company-ecommerce/platform-team
/environments/prod/      @company-ecommerce/platform-team

/argocd/                  @company-ecommerce/platform-team
/applications/            @company-ecommerce/platform-team
/.github/                 @company-ecommerce/platform-team
```

Production changes should receive stronger review than development changes.

---

# 18. Pull Request Template

Create:

```text
.github/PULL_REQUEST_TEMPLATE.md
```

```markdown
## Summary

Describe what changed.

## Change Type

- [ ] Feature
- [ ] Bug fix
- [ ] Refactoring
- [ ] Infrastructure
- [ ] CI/CD
- [ ] Security
- [ ] Documentation

## Related Issue

Closes #

## Testing

Describe how the change was tested.

- [ ] Unit tests
- [ ] Integration tests
- [ ] Local validation
- [ ] Docker validation
- [ ] Kubernetes validation

## Security

- [ ] No secrets committed
- [ ] Security impact reviewed
- [ ] Dependency changes reviewed
- [ ] Container changes reviewed

## Deployment

Describe deployment impact.

## Rollback

Describe how this change can be rolled back.

## Database Changes

- [ ] No database changes
- [ ] Backward-compatible migration
- [ ] Migration tested
- [ ] Rollback plan available

## Checklist

- [ ] Code reviewed
- [ ] Tests passing
- [ ] Documentation updated
- [ ] Logs/metrics considered
- [ ] Alerts considered
```

---

# 19. CONTRIBUTING.md

Use the same basic policy across repositories:

```markdown
# Contributing

## Branches

Use short-lived branches.

Examples:

feature/order-validation
bugfix/order-status
hotfix/payment-timeout

## Pull Requests

All changes must go through a pull request.

Direct pushes to `main` are not allowed.

## Commits

Use conventional commits.

Examples:

feat: add order creation API
fix: handle duplicate order request
test: add order service tests
refactor: simplify order validation
ci: add container security scan
docs: update deployment documentation

## Reviews

Changes must be reviewed by the appropriate CODEOWNERS.

## Testing

Run tests locally before opening a pull request.

## Secrets

Never commit:

- passwords
- API keys
- AWS credentials
- tokens
- private keys
- production configuration containing secrets
```

---

# 20. SECURITY.md

```markdown
# Security Policy

## Reporting a Vulnerability

Do not create a public issue for security vulnerabilities.

Report security issues through the organization's approved security reporting process.

## Secrets

Never commit credentials or secrets.

If a secret is accidentally committed:

1. Revoke the secret immediately.
2. Rotate the credential.
3. Remove it from the repository history where appropriate.
4. Investigate usage.
5. Document the incident.

## Dependencies

Security vulnerabilities in dependencies should be reviewed and remediated according to organizational severity and SLA requirements.
```

---

# 21. `.editorconfig`

Create:

```text
.editorconfig
```

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4

[*.yaml]
indent_size = 2

[*.yml]
indent_size = 2

[*.json]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

This prevents unnecessary formatting differences between developers and CI.

---

# 22. README Template

Every repository should have a useful README.

For `order-service`:

```markdown
# Order Service

Order management microservice for the Enterprise E-Commerce Platform.

## Ownership

Backend Team

## Technology

- Java 21
- Spring Boot
- PostgreSQL
- Docker
- Kubernetes
- Helm

## Responsibilities

The service manages:

- Order creation
- Order retrieval
- Order updates
- Order cancellation

## APIs

POST /api/v1/orders

GET /api/v1/orders/{id}

GET /api/v1/orders

PUT /api/v1/orders/{id}

DELETE /api/v1/orders/{id}

## Local Development

See the local development documentation.

## Build

./mvnw clean verify

## Docker

docker build -t order-service:local .

## Testing

Run unit and integration tests before creating a pull request.

## Deployment

Application deployment is managed through:

GitHub Actions
→ ECR
→ GitOps
→ ArgoCD
→ EKS

## Observability

Metrics:
Prometheus

Logs:
ELK

Traces:
OpenTelemetry + Jaeger

## Security

Security scanning is performed during CI.

## Ownership

Backend Team
```

---

# 23. Dependabot

Create:

```text
.github/dependabot.yml
```

Example:

```yaml
version: 2

updates:

  - package-ecosystem: maven
    directory: "/"
    schedule:
      interval: weekly

  - package-ecosystem: github-actions
    directory: "/.github/workflows"
    schedule:
      interval: weekly
```

This helps detect dependency and workflow-action updates.

In the production CI chapter we will also discuss pinning third-party GitHub Actions to approved commit SHAs.

---

# 24. Initialize the Repository Locally

For example:

```bash
mkdir order-service
cd order-service

git init -b main
```

Create the files:

```text
README.md
.gitignore
.editorconfig
CONTRIBUTING.md
SECURITY.md
Dockerfile
.dockerignore
```

Then:

```bash
mkdir -p .github/workflows
mkdir -p src/main
mkdir -p src/test
mkdir -p deploy
mkdir -p helm
```

Add CODEOWNERS:

```bash
touch .github/CODEOWNERS
```

Add PR template:

```bash
touch .github/PULL_REQUEST_TEMPLATE.md
```

---

# 25. First Commit

Check:

```bash
git status
```

Then:

```bash
git add .
```

Commit:

```bash
git commit -m "chore: initialize order service repository"
```

Check:

```bash
git log --oneline
```

Expected:

```text
abc1234 chore: initialize order service repository
```

---

# 26. Create GitHub Repository

Using GitHub CLI, after authenticating:

```bash
gh auth login
```

Verify:

```bash
gh auth status
```

Create the repository:

```bash
gh repo create company-ecommerce/order-service \
  --private \
  --source=. \
  --remote=origin \
  --push
```

Verify:

```bash
git remote -v
```

Expected:

```text
origin  git@github.com:company-ecommerce/order-service.git
```

Then:

```bash
git branch
```

Expected:

```text
* main
```

---

# 27. Alternative: GitHub UI

If GitHub CLI isn't available:

1. Create repository in the `company-ecommerce` organization.
2. Repository name:

```text
order-service
```

3. Visibility:

```text
Private
```

4. Do **not** initialize with another README if you already created one locally.
5. Do **not** add a second `.gitignore`.
6. Create repository.
7. Add remote locally:

```bash
git remote add origin git@github.com:company-ecommerce/order-service.git
```

8. Push:

```bash
git push -u origin main
```

---

# 28. Create the Other Three Repositories

Repeat for:

```text
infrastructure
gitops
platform
```

The final organization should initially look like:

```text
company-ecommerce
│
├── order-service
│
├── infrastructure
│
├── gitops
│
└── platform
```

Do not create the remaining 20 application repositories until the first application path is working.

---

# 29. Git Branch Workflow

Developer starts work:

```bash
git checkout main
git pull origin main
```

Create feature branch:

```bash
git checkout -b feature/order-validation
```

Work:

```bash
git add .
git commit -m "feat: add order validation"
```

Push:

```bash
git push -u origin feature/order-validation
```

Then:

```text
feature/order-validation
          │
          ▼
        Pull Request
          │
          ▼
       CI Pipeline
          │
          ├── Build
          ├── Unit Test
          ├── SonarQube
          ├── Secret Scan
          ├── Dependency Scan
          ├── Docker Build
          └── Trivy
                  │
                  ▼
             Review
                  │
                  ▼
             Approval
                  │
                  ▼
          Squash + Merge
                  │
                  ▼
                main
```

---

# 30. Main Branch Protection

`main` must be protected.

The important controls are:

```text
Pull request required
        +
Required approvals
        +
Required CI checks
        +
Conversation resolution
        +
No force push
        +
No branch deletion
```

Initially, before CI exists, don't configure required status checks that don't exist yet.

After GitHub Actions is implemented, add the actual required checks.

Typical production application checks might eventually include:

```text
build
unit-test
sonarqube
secret-scan
dependency-scan
trivy-filesystem
docker-build
trivy-image
```

---

# 31. Merge Strategy

Use:

```text
Squash and merge
```

Example feature branch:

```text
feature/order-validation

├── fix typo
├── add validation
├── fix test
├── debug logging
└── final fix
```

After squash:

```text
main

└── feat: add order validation
```

This keeps `main` clean.

---

# 32. Hotfix Workflow

Production incident:

```text
main
 │
 └── hotfix/payment-timeout
          │
          ▼
        PR
          │
          ▼
      Fast review
          │
          ▼
        CI
          │
          ▼
       main
          │
          ▼
       GitOps
          │
          ▼
        ArgoCD
```

We do not create:

```text
production branch
```

just for hotfixes.

---

# 33. Repository Responsibility Boundary

This distinction is extremely important.

### Application repository

```text
order-service
```

Contains:

```text
Java
Tests
Dockerfile
Application configuration
Helm chart
CI
```

---

### Infrastructure repository

```text
infrastructure
```

Contains:

```text
Terraform
AWS infrastructure
VPC
EKS
RDS
IAM
ECR
SQS
SNS
Redis
KMS
```

---

### Platform repository

```text
platform
```

Contains:

```text
Prometheus
Grafana
Alertmanager
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
Jaeger
External Secrets
AWS Load Balancer Controller
```

---

### GitOps repository

```text
gitops
```

Contains:

```text
Desired Kubernetes state
Environment configuration
ArgoCD Applications
ApplicationSets
Helm values
Image versions/digests
```

---

# 34. What Must Never Be Committed

Never commit:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY

DB_PASSWORD
REDIS_PASSWORD

JWT_PRIVATE_KEY
TLS_PRIVATE_KEY

production .env files

Terraform state

Kubernetes secret values

API tokens

SonarQube tokens

GitHub tokens
```

Instead:

```text
Secrets Manager
        │
        ▼
External Secrets
        │
        ▼
Kubernetes Secret
        │
        ▼
Application
```

This is why we created the Secrets Management architecture earlier.

---

# 35. Final Repository Architecture

After this chapter:

```text
                    GitHub Organization
                   company-ecommerce
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    Application       Infrastructure     Platform
     Repositories       Repository       Repository
          │                │                │
          │                ▼                ▼
          │             Terraform       Shared Platform
          │                │                │
          │                ▼                │
          │                AWS              │
          │                                 │
          ▼                                 │
        CI                                  │
          │                                 │
          ▼                                 │
        ECR                                 │
          │                                 │
          └──────────────┐                  │
                         ▼                  ▼
                       GitOps Repository
                              │
                              ▼
                           ArgoCD
                              │
                              ▼
                             EKS
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
         Microservices      Platform        Observability
             │                │                │
             ▼                ▼                ▼
            RDS            Redis          Prometheus
             │                │             Grafana
             ├───────────────┤             ELK
             │                             OTel
             ▼                             Jaeger
           SQS/SNS
```

---

# 36. Validation Checklist

Before moving forward, verify:

```text
[ ] GitHub organization exists
[ ] backend-team exists
[ ] platform-team exists
[ ] devops-team exists
[ ] security-team exists
[ ] sre-team exists
[ ] architecture-team exists

[ ] order-service created
[ ] infrastructure created
[ ] gitops created
[ ] platform created

[ ] All repositories are private
[ ] main branch exists
[ ] CODEOWNERS exists
[ ] README exists
[ ] CONTRIBUTING.md exists
[ ] SECURITY.md exists
[ ] .gitignore exists
[ ] .editorconfig exists

[ ] main branch protection configured
[ ] Direct pushes to main restricted
[ ] Squash merge configured
[ ] Feature branch workflow understood
```

---

# 37. Most Important Concept

At the end of this chapter, remember this separation:

```text
CODE
  ↓
Application Repository
  ↓
CI
  ↓
ECR
  ↓
GitOps
  ↓
ArgoCD
  ↓
EKS
```

while:

```text
INFRASTRUCTURE
  ↓
Terraform
  ↓
AWS
```

and:

```text
PLATFORM
  ↓
Helm/GitOps
  ↓
EKS Platform Components
```

This separation is what allows the project to scale from **1 service to 21 services** without turning the repository and deployment process into a mess.

---

## Next implementation chapter

**54 — Order Service Implementation**

We will now stop discussing repository structure and actually build the first microservice:

```text
Spring Boot
     ↓
Maven
     ↓
PostgreSQL
     ↓
REST APIs
     ↓
Unit Tests
     ↓
Actuator
     ↓
Prometheus metrics
     ↓
Docker image
     ↓
Local Docker Compose
```

Then we will validate the service locally before touching EKS.
