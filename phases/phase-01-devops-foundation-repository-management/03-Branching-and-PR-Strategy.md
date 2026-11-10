# 03 — Branching, Pull Requests & Repository Governance

Now we make the GitHub workflow **production-grade**.

The goal is that a developer can safely go from:

```text
Requirement
   ↓
Feature branch
   ↓
Code
   ↓
Pull Request
   ↓
Automated validation
   ↓
Code review
   ↓
Merge
   ↓
CI/CD
```

without anyone casually pushing directly to production.

---

# 1. Our Branch Model

We will keep the application repository simple:

```text
main
  |
  +---- feature/*
  |
  +---- bugfix/*
  |
  +---- hotfix/*
```

We are **not** going to create unnecessary long-lived branches such as:

```text
develop
qa
staging
production
```

for the application repository.

Environment promotion will be handled later through the **GitOps repository**.

---

# 2. Why `main` Is Special

`main` represents:

> Code that has passed the repository's required validation and is eligible to become a release.

Developers should never do:

```bash
git checkout main
# modify files
git commit
git push
```

Instead:

```text
main
  |
  └── feature branch
          |
          └── Pull Request
                  |
                  └── CI
                        |
                        └── Review
                              |
                              └── Merge
```

---

# 3. Feature Branch

Suppose the requirement is:

> Add order cancellation functionality.

Start from current `main`:

```bash
git checkout main
git pull origin main
```

Create branch:

```bash
git checkout -b feature/order-cancellation
```

Now:

```text
main
 |
 └── feature/order-cancellation
```

Developer works only here.

---

# 4. Make the Change

For example:

```text
src/main/java/com/company/order/
│
├── controller/
│   └── OrderController.java
│
├── service/
│   └── OrderService.java
│
└── repository/
    └── OrderRepository.java
```

Developer adds cancellation logic.

Then tests:

```text
src/test/
└── OrderCancellationTest.java
```

---

# 5. Local Validation

Before pushing:

```bash
./mvnw clean test
```

Then:

```bash
./mvnw verify
```

Check Git:

```bash
git status
```

Then:

```bash
git diff
```

This is important.

A good engineer doesn't blindly:

```bash
git add .
```

without checking what changed.

---

# 6. Commit

Use a meaningful commit:

```bash
git add src/main/java src/test
git commit -m "feat: add order cancellation"
```

If you need another change:

```bash
git commit -m "test: add cancellation validation"
```

---

# 7. Push Feature Branch

```bash
git push -u origin feature/order-cancellation
```

GitHub now has:

```text
main
feature/order-cancellation
```

---

# 8. Create Pull Request

Create:

```text
feature/order-cancellation
             ↓
           main
```

PR title:

```text
feat: add order cancellation
```

Example PR description:

```markdown
## Summary

Adds order cancellation functionality.

## Changes

- Added cancellation API
- Added cancellation business validation
- Added unit tests
- Added error handling

## Testing

- ./mvnw clean test
- ./mvnw verify

## Risk

Low

## Rollback

Revert this PR.
```

This makes the PR useful for reviewers and future incident investigation.

---

# 9. CI Starts Automatically

Our GitHub Actions workflow triggers:

```yaml
on:
  pull_request:
    branches:
      - main
```

So:

```text
PR opened
   ↓
GitHub Actions
   ↓
CI
```

Eventually:

```text
             Pull Request
                   |
       +-----------+-----------+
       |           |           |
      Build      Tests       Security
       |           |           |
       +-----------+-----------+
                   |
                   v
              All Passed
                   |
                   v
              Code Review
```

---

# 10. Required Status Checks

The important concept is:

**A developer shouldn't be able to merge simply because another developer approved the PR.**

We want:

```text
Code Review
     +
CI PASS
     +
Required checks PASS
     ↓
   Merge
```

For example:

```text
Required checks

✓ build
✓ unit-tests
✓ sonar
✓ dependency-scan
✓ secret-scan
✓ helm-validation
```

Later we'll add container scanning.

---

# 11. Branch Protection Concept

For:

```text
main
```

we want rules equivalent to:

```text
Require pull request before merging
Require approvals
Require status checks
Require conversation resolution
Restrict direct pushes
```

We can also require:

```text
No force pushes
No branch deletion
```

for important repositories.

---

# 12. Minimum Approval Model

For normal source-code changes:

```text
1 approved review
```

For infrastructure-related changes:

```text
1 DevOps/platform approval
```

For security-sensitive changes:

```text
1 security approval
```

For production GitOps changes:

```text
platform approval
+
appropriate application owner approval
```

The exact number can be adjusted according to organizational risk.

---

# 13. CODEOWNERS

Our repository:

```text
.github/CODEOWNERS
```

contains:

```text
/src/       @company-ecommerce/backend-team
/deploy/    @company-ecommerce/platform-team
/helm/      @company-ecommerce/platform-team
/.github/   @company-ecommerce/platform-team
```

Now consider:

```text
Developer
   |
   v
Changes deployment.yaml
   |
   v
Pull Request
   |
   v
platform-team automatically requested
```

This prevents application developers from being the only reviewers of deployment infrastructure.

---

# 14. CODEOWNERS Is Not a Security Boundary by Itself

Important distinction.

CODEOWNERS tells GitHub:

> These people/teams should review this path.

But your repository rules must also make the relevant CODEOWNER review **required** if you want it enforced.

Conceptually:

```text
CODEOWNERS
     +
Branch protection/rules
     =
Enforced ownership review
```

Don't assume merely creating `CODEOWNERS` prevents merging.

---

# 15. Pull Request Review

Reviewer opens:

```text
PR #101
feat: add order cancellation
```

They inspect:

```text
Business logic
Tests
Error handling
Logging
Performance
Security
API compatibility
```

They might comment:

```text
Why are we returning HTTP 500 here?
```

Developer changes it.

Then:

```bash
git add .
git commit -m "fix: return correct status for invalid cancellation"
git push
```

The PR automatically updates.

---

# 16. Conversation Resolution

Suppose reviewer asks:

```text
Please add a test for already-cancelled orders.
```

Developer adds it.

Reviewer says:

```text
Looks good.
```

Conversation is resolved.

Our repository rules can require conversations to be resolved before merging.

---

# 17. Merge Strategy

We'll use a controlled merge strategy.

For application repositories, a good default is:

```text
Squash and merge
```

Example:

Feature branch has:

```text
commit 1
commit 2
commit 3
commit 4
```

After squash:

```text
main
 |
 └── feat: add order cancellation
```

This keeps `main` history clean.

---

# 18. Why Squash Helps

Without squashing:

```text
main

commit:
test

commit:
fix

commit:
fix again

commit:
final

commit:
really final
```

With squash:

```text
main

feat: add order cancellation
```

The detailed development history still exists in the PR, while `main` remains easier to understand.

---

# 19. What About Hotfixes?

Production incident:

```text
Payment requests returning HTTP 500
```

We don't make random changes directly on `main`.

Create:

```bash
git checkout main
git pull origin main

git checkout -b hotfix/payment-timeout
```

Make minimal change.

Then:

```text
hotfix/payment-timeout
          |
          v
         PR
          |
          v
    Emergency CI
          |
          v
      Approval
          |
          v
         main
```

Later we'll connect this to the GitOps production rollback/hotfix workflow.

---

# 20. Feature vs Bugfix vs Hotfix

Use:

```text
feature/
```

for new functionality:

```text
feature/order-cancellation
feature/order-events
```

Use:

```text
bugfix/
```

for normal defects:

```text
bugfix/order-validation
bugfix/order-timeout
```

Use:

```text
hotfix/
```

for production-impacting urgent fixes:

```text
hotfix/payment-api-timeout
hotfix/order-memory-leak
```

---

# 21. Commit Naming

We'll follow a simple Conventional Commit style.

```text
feat:
fix:
docs:
test:
refactor:
perf:
ci:
build:
chore:
```

Examples:

```text
feat: add order creation endpoint
fix: handle duplicate order requests
test: add order repository tests
refactor: simplify order validation
perf: optimize order lookup
ci: add trivy image scanning
build: update java base image
chore: update dependencies
docs: document local setup
```

---

# 22. Never Put Secrets in Git

This deserves its own rule.

Never:

```yaml
password: MyPassword123
```

Never:

```text
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

Never:

```yaml
database:
  username: admin
  password: production-password
```

Instead, later we'll use:

```text
AWS Secrets Manager
        |
        v
External Secrets
        |
        v
Kubernetes Secret
        |
        v
Pod
```

And GitHub Actions will use:

```text
OIDC
```

instead of permanent AWS credentials.

---

# 23. GitHub Secret Scanning

We should enable GitHub's available repository security features appropriate to the account/organization.

The purpose is:

```text
Developer accidentally commits secret
              |
              v
        Secret detection
              |
              v
           Alert
```

But remember:

**Secret scanning is not a replacement for preventing secrets from entering Git.**

The developer should still use:

```text
.env
```

locally where appropriate, while ensuring it is ignored and does not contain production credentials.

---

# 24. Dependabot / Dependency Updates

Our Java service has dependencies:

```text
Spring Boot
Jackson
Netty
PostgreSQL driver
AWS SDK
etc.
```

These dependencies eventually contain vulnerabilities.

We want automated dependency-update awareness.

Conceptually:

```text
pom.xml
   |
   v
Dependency monitoring
   |
   v
New version available
   |
   v
Pull Request
   |
   v
CI
```

We will configure this later as part of the security phase.

---

# 25. Don't Copy 21 CI Files Manually Forever

This is where enterprise DevOps becomes interesting.

Imagine:

```text
21 repositories
```

If each contains completely independent pipelines:

```text
21 × maintenance
```

Then someone changes the Trivy command.

You have to modify:

```text
21 workflows
```

That's bad.

Instead we'll eventually create reusable workflows.

Architecture:

```text
                    Shared CI
                       |
       +---------------+---------------+
       |               |               |
 order-service   payment-service   inventory-service
       |               |               |
       +---------------+---------------+
                       |
                Same CI standards
```

GitHub reusable workflows can be used for this.

For example:

```text
platform
└── reusable-workflows
    ├── java-service-ci.yaml
    ├── docker-build.yaml
    ├── security-scan.yaml
    └── helm-validation.yaml
```

Then:

```text
order-service
     |
     └── calls shared workflow

payment-service
     |
     └── calls shared workflow

inventory-service
     |
     └── calls shared workflow
```

We'll build this after the first service pipeline works.

---

# 26. But Don't Abstract Too Early

This is another real-world lesson.

We first understand:

```text
order-service
```

end-to-end.

Build:

```text
order-service CI
```

manually.

Understand every command.

Then extract common functionality.

Otherwise you'll have:

```text
21 services
+
one giant reusable workflow
+
nobody understands what it does
```

That's abstraction without understanding.

Our approach:

```text
Understand
   ↓
Build
   ↓
Validate
   ↓
Standardize
   ↓
Automate
   ↓
Reuse
```

---

# 27. Repository Lifecycle

Every application repository should follow:

```text
Create
  ↓
Initialize
  ↓
Configure ownership
  ↓
Configure branch rules
  ↓
Configure CI
  ↓
Configure security
  ↓
Develop
  ↓
Review
  ↓
Merge
  ↓
Release
  ↓
Deploy
  ↓
Observe
  ↓
Maintain
```

This is what we're building.

---

# 28. Release Does Not Mean Production Yet

This distinction is important.

After merge:

```text
main
 |
 v
CI
 |
 v
Docker image
 |
 v
ECR
```

doesn't automatically mean:

```text
PRODUCTION
```

Instead:

```text
ECR
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
PRODUCTION
```

The promotion mechanism will live in GitOps.

---

# 29. The Four Repositories Working Together

Eventually:

```text
                 Developer
                     |
                     v
              order-service
                     |
                     | CI
                     v
                    ECR
                     |
                     v
                  gitops
                     |
                     | ArgoCD
                     v
                    EKS
                     |
                     v
                Production
```

Infrastructure:

```text
infrastructure
       |
       | Terraform
       v
      AWS
```

Platform:

```text
platform
   |
   v
Prometheus
Grafana
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
Jaeger
ArgoCD
etc.
```

So each repository has a very clear responsibility.

---

# 30. Our Current Repository

We're still focusing on:

```text
order-service
```

Current structure:

```text
order-service/
│
├── src/
├── Dockerfile
├── pom.xml
├── README.md
├── .gitignore
│
├── deploy/
│
├── helm/
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
        └── ci.yaml
```

---

# 31. The Next Major Step: Actual Application Code

Now we're ready to stop talking about GitHub structure and **write the application**.

We'll build a real `order-service`.

Initial API:

```text
POST   /api/v1/orders
GET    /api/v1/orders/{orderId}
GET    /api/v1/orders
PUT    /api/v1/orders/{orderId}
DELETE /api/v1/orders/{orderId}
```

Architecture:

```text
Client
  |
  v
OrderController
  |
  v
OrderService
  |
  v
OrderRepository
  |
  v
PostgreSQL
```

And later:

```text
Order Service
      |
      +---- PostgreSQL
      |
      +---- Redis
      |
      +---- SQS
      |
      +---- Secrets Manager
      |
      +---- OpenTelemetry
      |
      +---- Prometheus
```

We'll write the actual:

```text
pom.xml
application.yaml
Java classes
DTOs
entities
repository
service
controller
exception handling
tests
Dockerfile
```

Then we'll run it locally.

After that:

```text
Local application
       ↓
Docker
       ↓
ECR
       ↓
EKS
```

And only then we'll introduce the infrastructure repo and Terraform.

**Next file: `04-Microservice-Foundation.md` — where we create the actual `order-service` source code from scratch.**
