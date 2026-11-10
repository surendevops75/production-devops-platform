# 02 — Repository Strategy for 20+ Microservices

Now we move from **creating one repository** to designing how the entire engineering organization manages **21 microservices + infrastructure + GitOps + platform repositories**.

This is important because managing one repository is easy. Managing 20+ services without creating chaos is the real DevOps problem.

---

## 1. Final GitHub Organization

Our organization will be:

```text
company-ecommerce
```

Repositories:

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

So:

```text
21 Application Repositories
        +
1 Infrastructure Repository
        +
1 GitOps Repository
        +
1 Platform Repository
        =
24 Repositories
```

---

# 2. Why Not One Monorepo?

You could technically create:

```text
ecommerce/
├── services/
│   ├── auth/
│   ├── order/
│   ├── payment/
│   └── ...
├── infrastructure/
├── kubernetes/
└── ...
```

But we'll intentionally use separate repositories.

Why?

Because services have independent:

* ownership
* releases
* permissions
* CI pipelines
* deployment versions
* rollback history
* development teams

For example:

```text
order-service
```

may release:

```text
1.5.2
```

while:

```text
payment-service
```

remains:

```text
2.1.7
```

There is no reason to rebuild the entire platform because Order Service changed.

---

# 3. Service Ownership

Let's assign ownership.

Example:

```text
backend-team
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
└── reporting-service
```

Platform:

```text
platform-team
│
├── gitops
└── platform
```

Infrastructure:

```text
devops-team
│
└── infrastructure
```

Security:

```text
security-team
```

can review sensitive changes across repositories.

---

# 4. Team-Based Access

We don't want to give every engineer:

```text
Admin access
```

to every repository.

Instead:

```text
Repository              Team                Access

order-service           backend-team        Write
payment-service         backend-team        Write
inventory-service       backend-team        Write

infrastructure          devops-team         Write
gitops                   platform-team      Write
platform                platform-team      Write
```

Security team might have:

```text
security-team
    |
    +-- Read access to application repos
    +-- Security review permissions
```

---

# 5. Why Infrastructure Must Be Separate

Consider this change:

```text
feature/order-validation
```

A developer should modify:

```text
order-service/src/
```

They should not casually modify:

```text
infrastructure/terraform/modules/eks/
```

Because an EKS change can affect:

```text
100+ pods
```

or even the entire cluster.

Therefore:

```text
Application repository
        X
Infrastructure repository
```

are intentionally separated.

---

# 6. Why GitOps Must Be Separate

This is another very important design decision.

Application repository:

```text
order-service
```

contains:

```text
application code
Dockerfile
tests
CI
```

GitOps repository:

```text
gitops
```

contains:

```text
deployment version
environment configuration
Helm values
ArgoCD
```

Example:

```text
order-service

Git SHA:
8f72a91

Image:
order-service:1.5.2
```

CI builds it and pushes to ECR.

Then GitOps changes:

```yaml
image:
  tag: "1.5.2"
```

ArgoCD detects:

```text
GitOps changed
```

and deploys it.

---

# 7. The Complete Code-to-Production Chain

Keep this architecture in mind throughout the entire project:

```text
Developer
   |
   v
Feature Branch
   |
   v
Pull Request
   |
   v
GitHub
   |
   v
CI
   |
   +--> Build
   +--> Unit Tests
   +--> SonarQube
   +--> Dependency Scan
   +--> Secret Scan
   +--> Docker Build
   +--> Trivy
   |
   v
Amazon ECR
   |
   v
GitOps Repository
   |
   v
ArgoCD
   |
   v
AWS EKS
   |
   v
Kubernetes
   |
   v
Production
```

This is the backbone of our project.

---

# 8. Repository Template

Now here's the important part.

We have:

```text
21 microservices
```

We do **not** want to manually design 21 different repositories.

Instead, we'll create a standard template.

For example:

```text
order-service
```

becomes our reference implementation.

The standard structure:

```text
order-service/
│
├── src/
│
├── tests/
│
├── Dockerfile
│
├── .dockerignore
│
├── pom.xml
│
├── README.md
│
├── .gitignore
│
├── deploy/
│
├── helm/
│
└── .github/
    ├── workflows/
    │   ├── ci.yaml
    │   └── security.yaml
    │
    └── CODEOWNERS
```

Then:

```text
order-service
     |
     | template
     |
     +------> inventory-service
     |
     +------> payment-service
     |
     +------> shipping-service
     |
     +------> notification-service
     |
     +------> ...
```

---

# 9. What Should Be Standardized?

Across all services we want standardized:

```text
Git structure
Docker build
CI pipeline
Security scanning
Image naming
Logging
Metrics
Health endpoints
Kubernetes deployment
Helm structure
Resource configuration
Labels
Annotations
Service accounts
```

For example, every service should expose:

```text
/actuator/health
```

and ideally:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Every service should produce structured logs.

Every service should expose metrics.

Every service should have:

```text
readinessProbe
livenessProbe
```

---

# 10. What Should NOT Be Identical?

Not everything should be copied blindly.

For example:

```text
payment-service
```

may require:

```text
2 CPU
4 GB memory
```

while:

```text
notification-service
```

may need:

```text
250m CPU
512 MB memory
```

Similarly:

```text
payment-service
```

might require access to:

```text
payment database
Secrets Manager
payment gateway
```

while:

```text
catalog-service
```

may require:

```text
catalog database
Elasticsearch
S3
```

Therefore:

```text
Standard platform
        +
Service-specific configuration
```

is our approach.

---

# 11. Standard Repository Labels

Every repository should use consistent GitHub topics/metadata.

For example:

```text
order-service
```

topics:

```text
microservice
java
spring-boot
ecommerce
kubernetes
aws
eks
```

This makes repositories easier to discover.

---

# 12. CODEOWNERS Design

For our application repository:

```text
.github/CODEOWNERS
```

we can use:

```text
# Application source
/src/ @company-ecommerce/backend-team

# Kubernetes manifests
/deploy/ @company-ecommerce/platform-team

# Helm
/helm/ @company-ecommerce/platform-team

# GitHub Actions
/.github/workflows/ @company-ecommerce/platform-team

# Security configuration
/security/ @company-ecommerce/security-team
```

Now different changes automatically route to the correct reviewers.

---

# 13. More Granular Ownership

Suppose later our service becomes larger:

```text
order-service/
├── src/
│   ├── order/
│   ├── payment/
│   └── fulfillment/
```

We could define:

```text
/src/order/        @company-ecommerce/order-team
/src/payment/      @company-ecommerce/payment-team
/src/fulfillment/  @company-ecommerce/fulfillment-team
```

This is especially useful in larger organizations.

---

# 14. Branch Strategy

We will use:

```text
main
```

plus short-lived branches.

Example:

```text
main
 |
 +-- feature/order-validation
 |
 +-- feature/order-events
 |
 +-- bugfix/order-timeout
 |
 +-- hotfix/payment-timeout
```

We don't need:

```text
development
staging
release
production
```

branches just because environments exist.

Environments are better represented through:

```text
GitOps
+
ArgoCD
+
environment-specific configuration
```

rather than creating a complicated application branch model.

---

# 15. Feature Branch Lifecycle

Developer receives requirement:

> Add order cancellation API.

They create:

```bash
git checkout main
git pull origin main

git checkout -b feature/order-cancellation
```

Work:

```text
src/
tests/
```

Then:

```bash
git add .
git commit -m "feat: add order cancellation API"
```

Push:

```bash
git push origin feature/order-cancellation
```

GitHub:

```text
PR #101
feature/order-cancellation
             |
             v
           main
```

---

# 16. Pull Request Checks

Our PR should automatically execute:

```text
PR
 |
 +--> Checkout
 |
 +--> Compile
 |
 +--> Unit Tests
 |
 +--> Code Coverage
 |
 +--> SonarQube
 |
 +--> Dependency Scan
 |
 +--> Secret Detection
 |
 +--> Docker Build
 |
 +--> Trivy
 |
 +--> Kubernetes/Helm validation
 |
 v
PASS
```

Then reviewers approve.

---

# 17. Required Approvals

Example:

```text
Source code change
    |
    v
1 backend approval
```

Kubernetes:

```text
Helm / Kubernetes change
    |
    v
backend approval
    +
platform approval
```

Security-sensitive:

```text
Security configuration
    |
    v
security-team approval
```

This gives us a realistic governance model.

---

# 18. GitHub Actions Permissions

One of the most important security principles:

Don't give GitHub Actions unnecessary permissions.

Example:

```yaml
permissions:
  contents: read
```

For AWS authentication later we'll need:

```yaml
permissions:
  contents: read
  id-token: write
```

Why?

Because we'll use:

```text
GitHub Actions
      |
      | OIDC
      v
AWS IAM Role
```

rather than storing:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside GitHub.

---

# 19. GitHub → AWS Authentication

Eventually:

```text
GitHub Actions
      |
      | OIDC Token
      v
AWS IAM
      |
      | AssumeRole
      v
CI IAM Role
      |
      v
ECR
```

The CI role might only have permission to:

```text
Authenticate to ECR
Push images
Read required metadata
```

It should **not** have:

```text
AdministratorAccess
```

---

# 20. Separate IAM Roles

Later we will create different AWS IAM roles.

For example:

```text
GitHub Actions CI Role
        |
        +--> ECR push

Terraform Role
        |
        +--> Infrastructure provisioning

ArgoCD Role
        |
        +--> AWS resources it actually needs

Application IAM Role
        |
        +--> Service-specific AWS permissions
```

This follows least privilege.

---

# 21. Application Repository Doesn't Own Production

This is subtle but very important.

`order-service` repository can say:

```text
Build version = 1.5.2
```

But production deployment is controlled through:

```text
gitops
```

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
    └── prod/
        └── order-service/
            └── values.yaml
```

Production could contain:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.5.2"
```

---

# 22. Why This Is Powerful

Suppose production is running:

```text
order-service:1.5.1
```

Developer releases:

```text
1.5.2
```

CI builds:

```text
1.5.2
```

and pushes:

```text
ECR
 |
 +-- 1.5.1
 +-- 1.5.2
```

QA receives:

```text
1.5.2
```

After validation, GitOps production changes:

```diff
- tag: "1.5.1"
+ tag: "1.5.2"
```

ArgoCD deploys.

If something goes wrong:

```diff
- tag: "1.5.2"
+ tag: "1.5.1"
```

ArgoCD reconciles.

That's our rollback mechanism.

---

# 23. Repository Naming Matrix

We'll eventually have:

| Repository               | Purpose               | Primary Owner |
| ------------------------ | --------------------- | ------------- |
| `auth-service`           | Authentication        | Backend       |
| `user-service`           | Users                 | Backend       |
| `customer-service`       | Customers             | Backend       |
| `catalog-service`        | Catalog               | Backend       |
| `product-service`        | Products              | Backend       |
| `inventory-service`      | Inventory             | Backend       |
| `pricing-service`        | Pricing               | Backend       |
| `promotion-service`      | Promotions            | Backend       |
| `cart-service`           | Cart                  | Backend       |
| `order-service`          | Orders                | Backend       |
| `payment-service`        | Payments              | Backend       |
| `fraud-service`          | Fraud                 | Backend       |
| `shipping-service`       | Shipping              | Backend       |
| `delivery-service`       | Delivery              | Backend       |
| `notification-service`   | Notifications         | Backend       |
| `search-service`         | Search                | Backend       |
| `recommendation-service` | Recommendations       | Backend       |
| `review-service`         | Reviews               | Backend       |
| `tax-service`            | Tax                   | Backend       |
| `invoice-service`        | Invoices              | Backend       |
| `reporting-service`      | Reporting             | Backend       |
| `infrastructure`         | AWS/Terraform         | DevOps        |
| `gitops`                 | Kubernetes/ArgoCD     | Platform      |
| `platform`               | Shared platform tools | Platform      |

---

# 24. The Important Part: Don't Build All 21 Immediately

This is how we'll do it practically.

### Phase 1

Build:

```text
order-service
```

completely.

From:

```text
Java code
```

to:

```text
production EKS
```

including:

```text
GitHub
Docker
Terraform
AWS
ECR
Kubernetes
Helm
ArgoCD
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
```

### Phase 2

Take the proven pattern and introduce:

```text
catalog-service
inventory-service
payment-service
```

### Phase 3

Expand to all 21 services.

This is much more realistic than trying to write 21 applications simultaneously.

---

# 25. Our Reference Microservice

The first service:

```text
order-service
```

will eventually have:

```text
order-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/order/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── model/
│   │   │       ├── dto/
│   │   │       └── config/
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       └── application-prod.yaml
│   │
│   └── test/
│
├── Dockerfile
├── pom.xml
│
├── helm/
│   └── order-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-prod.yaml
│       └── templates/
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yaml
│   │   └── security.yaml
│   └── CODEOWNERS
│
└── README.md
```

This will become our **golden service pattern**.

---

# 26. What Happens When We Have 21 Services?

Eventually:

```text
21 services
      |
      v
same engineering standards
      |
      +--> same CI principles
      +--> same Docker standards
      +--> same security scanning
      +--> same Kubernetes standards
      +--> same observability standards
      +--> same deployment principles
```

But:

```text
business logic
database
dependencies
resources
scaling
configuration
```

remain service-specific.

That's the balance we want.

---

# 27. Real Production Ownership Model

The final structure will look like:

```text
                         CTO
                          |
                     Engineering
                          |
        +-----------------+------------------+
        |                 |                  |
     Backend           Platform           Security
        |                 |                  |
   21 services      GitOps/ArgoCD       Security controls
        |                 |
        |              DevOps/SRE
        |                 |
        |          Infrastructure
        |                 |
        |              AWS/EKS
        |                 |
        +-------- Production Platform
```

Your DevOps/SRE responsibility isn't simply:

> "I deploy applications."

It becomes:

> "I provide and operate the platform through which engineering teams safely build, release, deploy, observe, and troubleshoot applications."

That's the mindset we're building with this project.

---

# 28. Current State vs Final State

### Right now

```text
GitHub
  |
  └── order-service
```

### After this phase

```text
GitHub Organization
│
├── 21 microservices
├── infrastructure
├── gitops
└── platform
```

### Final production architecture

```text
GitHub
   |
   +-------------------+
   |                   |
Application          Infrastructure
Repos                  Repo
   |                     |
   | CI                  | Terraform
   v                     v
  ECR                  AWS
   |                     |
   |                     v
   +-----> GitOps ----> EKS
              |
            ArgoCD
              |
              v
        21 Microservices
              |
       +------+------+------+
       |      |      |      |
      RDS   Redis   SQS   DynamoDB
              |
        Observability
              |
    +---------+----------+
    |         |          |
 Prometheus   ELK     OpenTelemetry
    |         |          |
 Grafana    Kibana     Jaeger
```

---

## Next: `03-Branching-and-PR-Strategy.md`

Next we'll actually configure the **development workflow**, not just discuss it.

We'll create the exact:

```text
branch rules
PR rules
CODEOWNERS
GitHub environments
approval model
status checks
merge strategy
commit conventions
hotfix process
release process
```

and then we'll start writing the **actual `order-service` application code**, followed by its Dockerfile and first complete CI pipeline.
