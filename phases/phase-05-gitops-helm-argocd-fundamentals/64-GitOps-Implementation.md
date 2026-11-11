# 64 — GitOps Implementation

## 1. Objective

This chapter implements the **GitOps repository** for the Enterprise E-Commerce Platform.

The target deployment flow is:

```text
Developer
   │
   ▼
Application Repository
   │
   │ Pull Request
   ▼
GitHub Actions
   │
   ├── Build
   ├── Test
   ├── SonarQube
   ├── Security Scan
   ├── Docker Build
   ├── Trivy
   └── Push Image
           │
           ▼
          ECR
           │
           │ immutable image
           ▼
    GitOps Repository
           │
           │ desired state
           ▼
         ArgoCD
           │
           ▼
          EKS
           │
           ▼
     Order Service
```

The important principle is:

> **CI builds the application. GitOps controls what should be deployed. ArgoCD makes the Kubernetes cluster match Git.**

CI does **not** directly deploy to production using `kubectl`.

---

# 2. What Is the GitOps Repository?

The GitOps repository contains the **desired deployment state** of the platform.

It answers questions such as:

```text
Which image should DEV run?
Which replicas should QA have?
Which hostname should STAGING use?
Which version is approved for PROD?
Which Helm chart version should be deployed?
```

It should **not** contain:

```text
Application source code
Database passwords
AWS access keys
API tokens
TLS private keys
Production secrets
```

Instead, secrets are referenced through the secrets-management architecture implemented earlier:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Application Pod
```

---

# 3. Application Repository vs GitOps Repository

We now have two different responsibilities.

## Application repository

Example:

```text
order-service/
```

Contains:

```text
src/
pom.xml
Dockerfile
helm/
.github/
README.md
```

The application team owns:

```text
Java code
Tests
Dockerfile
Application Helm chart
CI pipeline
Application dependencies
```

---

## GitOps repository

Example:

```text
gitops/
```

Contains:

```text
Environment configuration
Deployment versions
Environment-specific values
ArgoCD configuration
Promotion state
```

Platform/SRE teams own this repository.

---

# 4. Final Repository Architecture

Our repositories now look like:

```text
company-ecommerce/
│
├── order-service/
├── auth-service/
├── user-service/
├── customer-service/
├── catalog-service/
├── product-service/
├── inventory-service/
├── pricing-service/
├── promotion-service/
├── cart-service/
├── payment-service/
├── fraud-service/
├── shipping-service/
├── delivery-service/
├── notification-service/
├── search-service/
├── recommendation-service/
├── review-service/
├── tax-service/
├── invoice-service/
├── reporting-service/
│
├── infrastructure/
├── gitops/
└── platform/
```

---

# 5. GitOps Repository Structure

Create:

```text
gitops/
│
├── applications/
│
├── environments/
│   │
│   ├── dev/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   ├── qa/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   ├── staging/
│   │   └── order-service/
│   │       └── values.yaml
│   │
│   └── prod/
│       └── order-service/
│           └── values.yaml
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
├── scripts/
│   └── update-image.sh
│
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       └── validate.yaml
│
├── .gitignore
├── README.md
└── CONTRIBUTING.md
```

Later, when all 21 services are onboarded:

```text
environments/
├── dev/
│   ├── auth-service/
│   ├── user-service/
│   ├── catalog-service/
│   ├── order-service/
│   └── ...
│
├── qa/
├── staging/
└── prod/
```

---

# 6. Create GitOps Repository

Using GitHub CLI:

```bash
gh repo create company-ecommerce/gitops \
  --private \
  --description "GitOps deployment configuration for Enterprise E-Commerce Platform"
```

Clone it:

```bash
git clone git@github.com:company-ecommerce/gitops.git
cd gitops
```

Create directories:

```bash
mkdir -p applications

mkdir -p environments/dev/order-service
mkdir -p environments/qa/order-service
mkdir -p environments/staging/order-service
mkdir -p environments/prod/order-service

mkdir -p argocd/projects
mkdir -p argocd/applications
mkdir -p argocd/appsets

mkdir -p scripts

mkdir -p .github/workflows
```

---

# 7. Initialize Git

```bash
git init
git branch -M main
```

Create `.gitignore`:

```gitignore
.DS_Store
*.log

.env
.env.*

*.tmp
*.swp

.idea/
.vscode/

node_modules/

*.secret.yaml
*-secret.yaml

secrets/
```

Do not blindly ignore every Kubernetes Secret manifest forever. The important rule is that **actual secret values must not be committed**.

---

# 8. Environment Configuration

The application Helm chart already exists inside:

```text
order-service/helm/order-service/
```

The GitOps repository will provide environment-specific values.

For example:

```text
gitops/
└── environments/
    ├── dev/
    │   └── order-service/
    │       └── values.yaml
    │
    ├── qa/
    │   └── order-service/
    │       └── values.yaml
    │
    ├── staging/
    │   └── order-service/
    │       └── values.yaml
    │
    └── prod/
        └── order-service/
            └── values.yaml
```

---

# 9. DEV Values

Create:

```text
environments/dev/order-service/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

config:
  springProfile: dev

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

ingress:
  enabled: true
  className: alb
  host: order-dev.example.com
```

---

# 10. QA Values

Create:

```text
environments/qa/order-service/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 300m
    memory: 512Mi

  limits:
    cpu: 1200m
    memory: 1Gi

config:
  springProfile: qa

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

ingress:
  enabled: true
  className: alb
  host: order-qa.example.com
```

---

# 11. STAGING Values

```text
environments/staging/order-service/values.yaml
```

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 500m
    memory: 768Mi

  limits:
    cpu: 1500m
    memory: 1536Mi

config:
  springProfile: staging

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  className: alb
  host: order-staging.example.com
```

---

# 12. Production Values

Create:

```text
environments/prod/order-service/values.yaml
```

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 500m
    memory: 768Mi

  limits:
    cpu: 1500m
    memory: 1536Mi

config:
  springProfile: prod

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  className: alb
  host: order.example.com
```

---

# 13. Why Keep Values Outside the Application Repository?

Consider:

```text
order-service
```

The backend team changes:

```text
Java code
```

and produces:

```text
order-service:8f31a2c
```

The image is pushed to:

```text
ECR
```

But production should not automatically change.

Instead:

```text
Application CI
     │
     ▼
ECR image
     │
     ▼
GitOps PR
     │
     ▼
Approval
     │
     ▼
GitOps main
     │
     ▼
ArgoCD
     │
     ▼
Production
```

This gives us a deployment audit trail.

---

# 14. Image Promotion

Suppose CI creates:

```text
order-service:8f31a2c
```

DEV:

```yaml
image:
  tag: "8f31a2c"
```

After testing, QA uses the **same image**:

```yaml
image:
  tag: "8f31a2c"
```

STAGING:

```yaml
image:
  tag: "8f31a2c"
```

Production:

```yaml
image:
  tag: "8f31a2c"
```

We do **not** rebuild the application for every environment.

That is:

```text
BUILD ONCE
   ↓
PROMOTE SAME ARTIFACT
```

This prevents:

```text
DEV image != QA image != PROD image
```

---

# 15. Tag vs Digest

The initial implementation can use:

```yaml
image:
  tag: "8f31a2c"
```

For stronger production immutability, the chart can later support:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:..."
```

The strongest deployment reference is effectively:

```text
repository@sha256:<digest>
```

because the digest identifies the exact image artifact.

We will introduce digest-based promotion as the platform matures.

---

# 16. GitOps Promotion Model

The environment flow is:

```text
                ┌─────────────┐
                │   DEV       │
                └──────┬──────┘
                       │
                       │ validation
                       ▼
                ┌─────────────┐
                │    QA       │
                └──────┬──────┘
                       │
                       │ testing
                       ▼
                ┌─────────────┐
                │  STAGING    │
                └──────┬──────┘
                       │
                       │ approval
                       ▼
                ┌─────────────┐
                │    PROD     │
                └─────────────┘
```

The same image is promoted through each environment.

---

# 17. GitOps Branch Strategy

Use:

```text
main
```

as the only long-lived branch.

Short-lived branches:

```text
promote/order-service-dev
promote/order-service-qa
promote/order-service-staging
promote/order-service-prod
```

Example:

```text
main
 │
 ├── promote/order-service-dev
 │
 ├── promote/order-service-qa
 │
 ├── promote/order-service-staging
 │
 └── promote/order-service-prod
```

---

# 18. Production Promotion

Production should not be changed by an uncontrolled CI process.

Example:

```text
CI
 │
 ├── build
 ├── test
 ├── scan
 └── push ECR
        │
        ▼
   image: 8f31a2c
        │
        ▼
GitOps DEV PR
        │
        ▼
      DEV
        │
        ▼
      QA
        │
        ▼
   STAGING
        │
        ▼
Production promotion PR
        │
        ▼
 Approval
        │
        ▼
     PROD
```

This creates a clear approval point.

---

# 19. CODEOWNERS

Create:

```text
.github/CODEOWNERS
```

Example:

```text
/environments/dev/        @company-ecommerce/platform-team
/environments/qa/         @company-ecommerce/platform-team
/environments/staging/    @company-ecommerce/platform-team @company-ecommerce/sre-team
/environments/prod/       @company-ecommerce/platform-team @company-ecommerce/sre-team

/argocd/                  @company-ecommerce/platform-team
/applications/            @company-ecommerce/platform-team
/scripts/                 @company-ecommerce/platform-team
```

For production:

```text
/environments/prod/
```

can require additional approval from the SRE/platform team.

---

# 20. GitOps Pull Request Template

Create:

```text
.github/pull_request_template.md
```

````markdown
## Deployment Promotion

### Service

- [ ] order-service

### Environment

- [ ] DEV
- [ ] QA
- [ ] STAGING
- [ ] PROD

### Image

Image tag:

```text
REPLACE_ME
````

Image digest:

```text
REPLACE_ME
```

### Validation

* [ ] Image exists in ECR
* [ ] CI completed successfully
* [ ] Security scans passed
* [ ] Helm validation passed
* [ ] Previous environment validated
* [ ] Rollback plan understood

### Production Only

* [ ] Production approval obtained
* [ ] Database compatibility verified
* [ ] Monitoring available
* [ ] Rollback procedure confirmed

````

---

# 21. Updating the Image Version

Create:

```text
scripts/update-image.sh
````

```bash
#!/usr/bin/env bash

set -euo pipefail

SERVICE="${1:?Service is required}"
ENVIRONMENT="${2:?Environment is required}"
IMAGE_TAG="${3:?Image tag is required}"

VALUES_FILE="environments/${ENVIRONMENT}/${SERVICE}/values.yaml"

if [[ ! -f "$VALUES_FILE" ]]; then
    echo "ERROR: Values file not found: $VALUES_FILE"
    exit 1
fi

echo "Updating ${SERVICE}"
echo "Environment: ${ENVIRONMENT}"
echo "Image tag: ${IMAGE_TAG}"

yq -i ".image.tag = \"${IMAGE_TAG}\"" "$VALUES_FILE"

echo "Updated:"
grep -A3 '^image:' "$VALUES_FILE"
```

Make executable:

```bash
chmod +x scripts/update-image.sh
```

Example:

```bash
./scripts/update-image.sh order-service dev 8f31a2c
```

Result:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

---

# 22. Why `yq`?

We should not modify YAML using:

```bash
sed
```

for complex configuration.

Bad:

```bash
sed -i 's/tag:.*/tag: 8f31a2c/' values.yaml
```

This can accidentally modify the wrong field.

Use a YAML-aware tool:

```bash
yq
```

Example:

```bash
yq -i '.image.tag = "8f31a2c"' values.yaml
```

---

# 23. GitOps Commit

After changing DEV:

```bash
git status
```

Example:

```text
modified:
environments/dev/order-service/values.yaml
```

Check:

```bash
git diff
```

Expected:

```diff
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
-  tag: "old-version"
+  tag: "8f31a2c"
```

Commit:

```bash
git add environments/dev/order-service/values.yaml

git commit -m "deploy(order-service): promote 8f31a2c to dev"
```

Push:

```bash
git push origin HEAD
```

Then create a PR.

---

# 24. ArgoCD Source Design

We have an important architecture decision.

The Helm chart lives here:

```text
order-service/
└── helm/
    └── order-service/
```

Environment values live here:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

ArgoCD can combine these using **multiple sources**.

Conceptually:

```text
                 ArgoCD
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
 order-service repo      gitops repo
          │                 │
          │ Helm chart      │ values
          └────────┬────────┘
                   │
                   ▼
              Helm render
                   │
                   ▼
              Kubernetes
```

This preserves the repository boundaries.

---

# 25. DEV ArgoCD Application

Create:

```text
argocd/applications/order-service-dev.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-dev
  namespace: argocd

spec:
  project: default

  sources:
    - repoURL: https://github.com/company-ecommerce/order-service.git
      targetRevision: main
      path: helm/order-service

      helm:
        valueFiles:
          - $values/environments/dev/order-service/values.yaml

    - repoURL: https://github.com/company-ecommerce/gitops.git
      targetRevision: main
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

This means:

```text
Chart:
company-ecommerce/order-service
        │
        ▼
helm/order-service

Values:
company-ecommerce/gitops
        │
        ▼
environments/dev/order-service/values.yaml
```

---

# 26. Important Production Difference

For DEV, tracking:

```yaml
targetRevision: main
```

can be acceptable during rapid development.

For production, we want stronger control.

Instead of:

```yaml
targetRevision: main
```

we can pin the application chart to a release:

```yaml
targetRevision: v1.4.0
```

or another controlled immutable revision.

Then:

```text
Application source
      │
      ▼
Release v1.4.0
      │
      ▼
GitOps production
```

This prevents an unrelated chart change merged into `main` from silently changing production.

---

# 27. Production Application Example

Conceptually:

```yaml
sources:
  - repoURL: https://github.com/company-ecommerce/order-service.git
    targetRevision: v1.4.0
    path: helm/order-service

    helm:
      valueFiles:
        - $values/environments/prod/order-service/values.yaml

  - repoURL: https://github.com/company-ecommerce/gitops.git
    targetRevision: main
    ref: values
```

Now production changes require deliberate promotion.

---

# 28. GitOps Repository Validation

We should not allow malformed values to reach ArgoCD.

Create:

```text
.github/workflows/validate.yaml
```

Example:

```yaml
name: GitOps Validation

on:
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4

      - name: Install yq
        run: |
          sudo wget -qO /usr/local/bin/yq \
            https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          sudo chmod +x /usr/local/bin/yq

      - name: Validate YAML files
        run: |
          find environments -name "*.yaml" -print

      - name: Check required image fields
        run: |
          for file in $(find environments -name values.yaml); do
            echo "Checking $file"

            yq -e '.image.repository' "$file" >/dev/null
            yq -e '.image.tag' "$file" >/dev/null
          done
```

For production, pin action/tool versions rather than relying on moving `latest` downloads.

---

# 29. Better Validation

Eventually the GitOps CI should perform:

```text
YAML validation
       │
       ▼
Helm validation
       │
       ▼
Policy validation
       │
       ▼
Security validation
       │
       ▼
PR approval
```

Possible tools:

```text
Helm
kubeconform
Conftest/OPA
Kyverno policies
Trivy config scanning
```

The exact platform stack can be introduced later.

---

# 30. GitOps CI Should Not Deploy

A common mistake is:

```text
GitHub Actions
     │
     ├── build
     ├── push ECR
     └── kubectl apply
```

We are deliberately **not** doing that.

Our model is:

```text
GitHub Actions
     │
     ▼
ECR
     │
     ▼
GitOps commit/PR
     │
     ▼
ArgoCD
     │
     ▼
EKS
```

Therefore CI does not need production Kubernetes credentials.

This dramatically reduces the blast radius of a compromised CI runner.

---

# 31. Authentication for Updating GitOps

There are several approaches.

## Option A — GitHub App

Preferred enterprise pattern:

```text
GitHub Actions
      │
      ▼
GitHub App
      │
      ▼
GitOps repository
```

Permissions can be limited to:

```text
Contents: Read/Write
Pull requests: Read/Write
Metadata: Read
```

Only the GitOps repository is authorized.

---

## Option B — Fine-Grained PAT

For learning, a fine-grained token can be used.

Store it as:

```text
GITOPS_TOKEN
```

in the application repository's GitHub Actions secrets.

Limit it to:

```text
company-ecommerce/gitops
```

and grant only the required repository permissions.

Do not use:

```text
classic PAT with broad organization access
```

for this workflow.

---

# 32. GitOps Update from CI

After ECR push:

```text
IMAGE_TAG=8f31a2c
```

CI can clone the GitOps repository:

```bash
git clone https://x-access-token:${GITOPS_TOKEN}@github.com/company-ecommerce/gitops.git
cd gitops
```

Update:

```bash
./scripts/update-image.sh \
  order-service \
  dev \
  "${IMAGE_TAG}"
```

Then:

```bash
git config user.name "company-ecommerce-bot"
git config user.email "devops-bot@company.example"

git checkout -b "promote/order-service-${IMAGE_TAG}"

git add environments/dev/order-service/values.yaml

git commit -m "deploy(order-service): promote ${IMAGE_TAG} to dev"

git push origin "promote/order-service-${IMAGE_TAG}"
```

Then create a PR.

---

# 33. Why Use a PR Instead of Direct Push?

Direct push:

```text
CI
 │
 ▼
GitOps main
```

is less controlled.

PR:

```text
CI
 │
 ▼
GitOps branch
 │
 ▼
PR
 │
 ├── validation
 ├── review
 └── approval
 │
 ▼
main
```

gives us:

```text
Audit trail
Review
Validation
Approval
Rollback history
```

For DEV, organizations may choose an automated direct update depending on their controls. For QA/STAGING/PROD, PR-based promotion gives stronger governance.

---

# 34. Complete Deployment Lifecycle

Now combine everything we've implemented.

```text
Developer
    │
    │ git push
    ▼
Feature Branch
    │
    │ PR
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Maven build
    ├── Unit tests
    ├── SonarQube
    ├── Dependency scan
    ├── Secret scan
    ├── Docker build
    ├── Trivy
    ├── ECR login
    └── ECR push
             │
             ▼
       ECR Image
       8f31a2c
             │
             ▼
      GitOps PR
             │
             ▼
      GitOps main
             │
             ▼
          ArgoCD
             │
             ▼
       Kubernetes API
             │
             ▼
           EKS
             │
             ▼
      Order Service Pods
```

---

# 35. Promotion Between Environments

Suppose the image is:

```text
8f31a2c
```

Initial state:

```text
DEV       → 8f31a2c
QA        → 72c1abc
STAGING   → 72c1abc
PROD      → 72c1abc
```

After DEV validation:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 72c1abc
PROD      → 72c1abc
```

After QA:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 8f31a2c
PROD      → 72c1abc
```

After production approval:

```text
DEV       → 8f31a2c
QA        → 8f31a2c
STAGING   → 8f31a2c
PROD      → 8f31a2c
```

The artifact is unchanged.

---

# 36. Rollback

Suppose:

```text
Production
    │
    ▼
8f31a2c
```

has a production problem.

Previous version:

```text
72c1abc
```

GitOps contains the deployment history.

Revert:

```bash
git revert <bad-gitops-commit>
```

Push:

```bash
git push origin main
```

ArgoCD detects:

```text
Git desired state
       ↓
72c1abc
```

and reconciles Kubernetes.

Therefore:

```text
Git revert
    ↓
ArgoCD
    ↓
EKS
    ↓
Previous image
```

No manual:

```bash
kubectl rollout undo
```

is required as the primary GitOps rollback mechanism.

---

# 37. Why Git Revert Is the Rollback Mechanism

Kubernetes state:

```text
Current cluster
```

can be changed.

But Git provides:

```text
Desired state history
```

Therefore the source of truth remains:

```text
Git
```

Example:

```text
Commit A
   │
   ▼
image: 72c1abc

Commit B
   │
   ▼
image: 8f31a2c
```

Rollback:

```text
Revert Commit B
        │
        ▼
Desired state returns to 72c1abc
        │
        ▼
ArgoCD reconciles
```

This is one of the key benefits of GitOps.

---

# 38. Secrets — Never Do This

Never put:

```yaml
database:
  username: postgres
  password: SuperSecret123
```

inside:

```text
gitops/environments/prod/
```

Never commit:

```yaml
stringData:
  password: ...
```

with actual production credentials.

Instead:

```text
GitOps
   │
   │ reference only
   ▼
ExternalSecret
   │
   ▼
AWS Secrets Manager
   │
   ▼
Kubernetes Secret
   │
   ▼
Pod
```

This follows the Secrets Management chapter.

---

# 39. GitOps Repository Security

Protect:

```text
main
```

with:

```text
Pull request required
Required status checks
Required reviews
CODEOWNERS
No force push
No branch deletion
Resolved conversations
```

For production:

```text
/environments/prod/
```

should have stricter ownership and approval.

---

# 40. GitOps Audit Trail

Suppose production currently runs:

```text
order-service
image: 8f31a2c
```

We should be able to answer:

```text
Who promoted it?
When?
Which PR?
Which CI run?
Which commit?
Which image digest?
Which source commit?
Which Helm chart version?
```

The chain becomes:

```text
Source Commit
      │
      ▼
CI Run
      │
      ▼
Image Tag
      │
      ▼
ECR Digest
      │
      ▼
GitOps Commit
      │
      ▼
ArgoCD Sync
      │
      ▼
Kubernetes Deployment
```

This traceability becomes extremely valuable during incidents.

---

# 41. GitOps Repository README

Create:

```text
README.md
```

```markdown
# E-Commerce Platform GitOps

This repository contains the desired deployment state for the
Enterprise E-Commerce Platform.

## Repository Responsibilities

This repository manages:

- Environment-specific Helm values
- Deployment versions
- ArgoCD Applications
- ArgoCD Projects
- ApplicationSets
- Promotion configuration

## Environments

- DEV
- QA
- STAGING
- PROD

## Deployment Flow

Application source
    ↓
CI
    ↓
Docker image
    ↓
Amazon ECR
    ↓
GitOps PR
    ↓
ArgoCD
    ↓
Amazon EKS

## Important Rules

1. Do not store secrets in this repository.
2. Do not use `latest` image tags.
3. Promote immutable images.
4. Production changes require review.
5. Do not deploy directly using `kubectl`.
6. Rollbacks should normally be performed through Git.
7. Environment configuration must remain reviewable.
```

---

# 42. First Commit

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
git commit -m "feat(gitops): initialize deployment repository"
```

Push:

```bash
git push origin main
```

---

# 43. Verify Repository

Expected:

```text
gitops/
├── applications/
├── environments/
│   ├── dev/
│   │   └── order-service/
│   │       └── values.yaml
│   ├── qa/
│   │   └── order-service/
│   │       └── values.yaml
│   ├── staging/
│   │   └── order-service/
│   │       └── values.yaml
│   └── prod/
│       └── order-service/
│           └── values.yaml
│
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
├── scripts/
│   └── update-image.sh
│
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       └── validate.yaml
│
├── .gitignore
├── CONTRIBUTING.md
└── README.md
```

---

# 44. Current Architecture

At this point our implementation is:

```text
                    GitHub
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
order-service                    gitops
        │                           │
        │ source + Helm             │ desired state
        │                           │
        ▼                           │
 GitHub Actions                    │
        │                           │
        ├── Build                   │
        ├── Test                    │
        ├── Scan                    │
        ├── Docker                  │
        └── ECR                     │
                │                   │
                ▼                   │
               ECR                  │
                │                   │
                └───────┬───────────┘
                        │
                        ▼
                     ArgoCD
                        │
                        ▼
                       EKS
                        │
                        ▼
                 order-service
```

---

# 45. What We Have Implemented So Far

| Component                  | Status               |
| -------------------------- | -------------------- |
| GitHub repositories        | ✅                    |
| Order service              | ✅                    |
| Local Docker               | ✅                    |
| Terraform state            | ✅                    |
| VPC                        | ✅                    |
| IAM                        | ✅                    |
| ECR                        | ✅                    |
| EKS                        | ✅                    |
| EKS add-ons                | ✅                    |
| GitHub Actions CI          | ✅                    |
| Helm chart                 | ✅                    |
| GitOps repository          | ✅                    |
| Environment values         | ✅                    |
| Promotion structure        | ✅                    |
| Rollback model             | ✅                    |
| ArgoCD                     | Architecture defined |
| ArgoCD actual installation | Next                 |

---

# 46. Final Principle

The production deployment model we are building is:

```text
                    CODE
                     │
                     ▼
                 GitHub PR
                     │
                     ▼
                    CI
                     │
       ┌─────────────┼─────────────┐
       │             │             │
      Test        Security       Build
       │             │             │
       └─────────────┼─────────────┘
                     ▼
                    ECR
                     │
                     ▼
              Immutable Image
                     │
                     ▼
                GitOps PR
                     │
                     ▼
                GitOps Main
                     │
                     ▼
                  ArgoCD
                     │
                     ▼
                    EKS
                     │
                     ▼
               Production
```

The critical separation is:

```text
CI = "Can we build and verify this?"

GitOps = "What version should each environment run?"

ArgoCD = "Make the cluster match Git."

EKS = "Run the workload."
```

---

# Next Chapter

### `65-ArgoCD-Implementation.md`

We will now move from **GitOps configuration** to the actual **ArgoCD implementation inside EKS**, including:

```text
Install ArgoCD
       ↓
ArgoCD namespace
       ↓
Helm installation
       ↓
Ingress / ALB
       ↓
GitHub repository access
       ↓
ArgoCD Project
       ↓
Application
       ↓
DEV deployment
       ↓
Sync
       ↓
Health checks
       ↓
Self-heal
       ↓
Prune
       ↓
Rollback
       ↓
Production deployment
```

This is where the first real **Git → ArgoCD → EKS deployment** will happen.
