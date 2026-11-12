Absolutely. The previous chapter jumped too quickly into YAML and ApplicationSet. Let's slow down and understand **what problem ApplicationSet solves, what it actually does, and how one service moves through the system**.

# 69 — ArgoCD ApplicationSet & 21 Microservices — Simple Explanation

## 1. First, forget the 21 services

Let's take only **one service**:

```text
order-service
```

You already understand this flow from the previous chapters:

```text
order-service GitHub repo
        ↓
GitHub Actions
        ↓
Docker image
        ↓
ECR
        ↓
GitOps repo
        ↓
ArgoCD
        ↓
EKS
```

Suppose we have four environments:

```text
DEV
QA
STAGING
PROD
```

Therefore, the same `order-service` must be deployed four times:

```text
order-service → DEV
order-service → QA
order-service → STAGING
order-service → PROD
```

---

# 2. What does ArgoCD normally need?

Normally, ArgoCD uses an `Application`.

For example:

```yaml
kind: Application
metadata:
  name: order-service-dev
```

This tells ArgoCD:

> "Deploy order-service into the DEV environment."

Another Application:

```yaml
kind: Application
metadata:
  name: order-service-qa
```

means:

> "Deploy order-service into QA."

Another:

```yaml
kind: Application
metadata:
  name: order-service-prod
```

means:

> "Deploy order-service into PROD."

So for one service:

```text
order-service
     |
     +---- order-service-dev
     |
     +---- order-service-qa
     |
     +---- order-service-staging
     |
     +---- order-service-prod
```

That's already **4 ArgoCD Applications**.

---

# 3. Now imagine 21 services

We have:

```text
1. auth-service
2. user-service
3. customer-service
4. catalog-service
5. product-service
6. inventory-service
7. pricing-service
8. promotion-service
9. cart-service
10. order-service
11. payment-service
12. fraud-service
13. shipping-service
14. delivery-service
15. notification-service
16. search-service
17. recommendation-service
18. review-service
19. tax-service
20. invoice-service
21. reporting-service
```

Each service has:

```text
DEV
QA
STAGING
PROD
```

Therefore:

```text
21 services × 4 environments = 84 Applications
```

So we'd need:

```text
84 ArgoCD Application objects
```

If we manually create them:

```text
order-service-dev.yaml
order-service-qa.yaml
order-service-staging.yaml
order-service-prod.yaml

user-service-dev.yaml
user-service-qa.yaml
...
```

That becomes difficult to maintain.

---

# 4. This is why ApplicationSet exists

This is the **main idea of the chapter**.

### ArgoCD Application

Creates **one Application**.

### ArgoCD ApplicationSet

Creates **many Applications automatically**.

Think of it like this:

```text
Application
    ↓
Creates 1 application

ApplicationSet
    ↓
Creates many applications
```

For example:

```text
ApplicationSet
      |
      +---- order-service-dev
      +---- order-service-qa
      +---- order-service-staging
      +---- order-service-prod
      |
      +---- user-service-dev
      +---- user-service-qa
      +---- user-service-staging
      +---- user-service-prod
      |
      +---- ...
```

You write the **rule/template once**.

ApplicationSet creates the individual Applications.

---

# 5. Very simple real-world analogy

Imagine you are HR.

You have 21 employees.

You need to create an ID card for every employee.

Without automation:

```text
Create ID card manually
Create ID card manually
Create ID card manually
...
84 times
```

With a template:

```text
ID card template
      +
Employee list
      ↓
Automatically generate ID cards
```

ApplicationSet works similarly.

```text
ApplicationSet template
        +
Service information
        ↓
ArgoCD Applications
```

---

# 6. Where does ApplicationSet get the information?

This is where our GitOps repository comes in.

We have:

```text
gitops/
└── environments/
    ├── dev/
    │   ├── order-service/
    │   ├── user-service/
    │   ├── payment-service/
    │   └── ...
    │
    ├── qa/
    │   ├── order-service/
    │   ├── user-service/
    │   └── ...
    │
    ├── staging/
    │   └── ...
    │
    └── prod/
        └── ...
```

The directory structure itself tells us:

```text
environment
service
```

For example:

```text
environments/dev/order-service/
```

means:

```text
Environment = dev
Service     = order-service
```

---

# 7. Why do we have `app.yaml`?

Inside:

```text
environments/dev/order-service/
```

we have:

```text
app.yaml
values.yaml
```

`app.yaml` contains information about the application.

Example:

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: main

enabled: true

team: backend-team
```

Let's understand each line.

### `service`

```yaml
service: order-service
```

This tells ApplicationSet:

> The service is order-service.

---

### `repository`

```yaml
repository: https://github.com/company-ecommerce/order-service.git
```

This tells ArgoCD:

> Get the Helm chart from the order-service repository.

---

### `chartPath`

```yaml
chartPath: helm/order-service
```

Inside the application repository we have:

```text
order-service/
├── src/
├── Dockerfile
├── pom.xml
└── helm/
    └── order-service/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

Therefore:

```text
chartPath = helm/order-service
```

---

### `chartRevision`

```yaml
chartRevision: main
```

This tells ArgoCD which Git revision to use.

For DEV:

```text
main
```

is reasonable.

For production we eventually want:

```text
v1.4.0
```

or a specific commit SHA.

---

# 8. What is `values.yaml`?

Remember our Helm chapter.

The Helm chart is the **template**.

The values file contains **environment-specific settings**.

For example:

```text
helm/order-service/
    templates/deployment.yaml
```

is the common deployment template.

Then:

```text
environments/dev/order-service/values.yaml
```

might contain:

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

Production might contain:

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

So:

```text
Same Helm chart
       +
Different environment values
       ↓
Different deployment
```

---

# 9. Very important distinction

This is probably where the previous chapter became confusing.

We have **three different repositories/files playing different roles**.

### Repository 1 — Application

```text
order-service
```

Contains:

```text
Java code
Dockerfile
Helm chart
```

---

### Repository 2 — ECR

Contains:

```text
Docker images
```

For example:

```text
order-service:8f31a2c
```

---

### Repository 3 — GitOps

Contains:

```text
Which version should be deployed
Environment-specific configuration
ArgoCD configuration
```

For example:

```text
dev/order-service
qa/order-service
staging/order-service
prod/order-service
```

---

# 10. Complete picture

Now look at the complete flow:

```text
                 APPLICATION REPO
              order-service GitHub
                       |
                       | CI
                       ↓
                 Docker image
                       |
                       ↓
                      ECR
                       |
                       |
                 GITOPS REPO
                       |
          +------------+------------+
          |            |            |
         DEV          QA         STAGING
          |            |            |
          +------------+------------+
                       |
                      PROD
                       |
                       ↓
                    ArgoCD
                       |
                       ↓
                     EKS
```

ApplicationSet sits inside the ArgoCD/GitOps part:

```text
GitOps
   ↓
ApplicationSet
   ↓
ArgoCD Applications
   ↓
EKS
```

---

# 11. What exactly happens when we add order-service?

Suppose this exists:

```text
gitops/
└── environments/
    └── dev/
        └── order-service/
            ├── app.yaml
            └── values.yaml
```

ApplicationSet watches this directory:

```text
environments/dev/*/app.yaml
```

It discovers:

```text
order-service/app.yaml
```

Then it reads:

```yaml
service: order-service
repository: https://github.com/company-ecommerce/order-service.git
chartPath: helm/order-service
chartRevision: main
```

ApplicationSet then creates:

```text
order-service-dev
```

as an ArgoCD Application.

---

# 12. Then ArgoCD takes over

ApplicationSet creates:

```text
order-service-dev
```

ArgoCD sees:

```text
Application:
order-service-dev
```

It then looks at:

```text
order-service GitHub repo
        ↓
helm/order-service
```

and:

```text
GitOps repo
        ↓
environments/dev/order-service/values.yaml
```

Helm combines them:

```text
Helm chart
     +
DEV values
     ↓
Kubernetes manifests
```

ArgoCD applies those manifests to:

```text
ecommerce-dev
```

namespace.

---

# 13. Final result in Kubernetes

You eventually get:

```text
EKS
│
└── ecommerce-dev
    │
    └── order-service
        ├── Deployment
        ├── Pods
        ├── Service
        ├── HPA
        └── PDB
```

---

# 14. Now add the second service

Suppose we add:

```text
environments/dev/user-service/
├── app.yaml
└── values.yaml
```

ApplicationSet discovers it.

Automatically creates:

```text
user-service-dev
```

Now EKS contains:

```text
ecommerce-dev
├── order-service
└── user-service
```

No new ArgoCD Application YAML needs to be manually written.

---

# 15. Add all 21 services

ApplicationSet discovers:

```text
environments/dev/
├── auth-service/
├── user-service/
├── customer-service/
├── catalog-service/
├── product-service/
├── inventory-service/
├── pricing-service/
├── promotion-service/
├── cart-service/
├── order-service/
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
└── reporting-service/
```

ApplicationSet creates:

```text
auth-service-dev
user-service-dev
customer-service-dev
catalog-service-dev
...
order-service-dev
...
reporting-service-dev
```

---

# 16. Then QA

Same concept:

```text
environments/qa/
├── auth-service/
├── user-service/
├── ...
└── order-service/
```

ApplicationSet creates:

```text
auth-service-qa
user-service-qa
...
order-service-qa
```

---

# 17. Then STAGING

```text
environments/staging/
├── auth-service/
├── user-service/
├── ...
└── order-service/
```

Creates:

```text
auth-service-staging
user-service-staging
...
order-service-staging
```

---

# 18. Then PROD

```text
environments/prod/
├── auth-service/
├── user-service/
├── ...
└── order-service/
```

Creates:

```text
auth-service-prod
user-service-prod
...
order-service-prod
```

Therefore:

```text
21 services
      ×
4 environments
      =
84 ArgoCD Applications
```

ApplicationSet creates and maintains these for us.

---

# 19. Why did I create 4 ApplicationSets?

We can have:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

Each watches its own environment.

### DEV

```text
ecommerce-dev
        ↓
environments/dev/*
```

### QA

```text
ecommerce-qa
        ↓
environments/qa/*
```

### STAGING

```text
ecommerce-staging
        ↓
environments/staging/*
```

### PROD

```text
ecommerce-prod
        ↓
environments/prod/*
```

This makes the architecture easier to understand and gives us different policies for each environment.

---

# 20. Why is PROD different?

DEV can automatically deploy:

```text
GitOps change
      ↓
ArgoCD
      ↓
Automatic sync
```

Production is more controlled:

```text
GitOps PR
    ↓
Review
    ↓
Approval
    ↓
Merge
    ↓
ArgoCD detects change
    ↓
Production sync
```

We don't want somebody accidentally changing a production value and having everything automatically deployed without the intended approval process.

---

# 21. One complete example

Let's say CI creates:

```text
order-service image
```

with Git SHA:

```text
8f31a2c
```

ECR:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

CI then updates:

```text
gitops/environments/dev/order-service/values.yaml
```

to:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

GitOps PR is merged.

ApplicationSet doesn't build the image.

ApplicationSet doesn't push the image.

ApplicationSet simply ensures that the correct ArgoCD Application exists.

Then:

```text
ApplicationSet
      ↓
order-service-dev Application
      ↓
ArgoCD
      ↓
Helm
      ↓
Deployment
      ↓
EKS
      ↓
Pod
      ↓
order-service:8f31a2c
```

---

# 22. The most important thing to remember

Don't think:

> "ApplicationSet deploys my application."

Instead think:

> **ApplicationSet creates/manages ArgoCD Applications automatically.**

Then:

> **ArgoCD deploys the desired state to Kubernetes.**

And:

> **Helm generates the Kubernetes manifests.**

And:

> **GitOps stores what version/configuration should be deployed.**

So the responsibilities are:

| Component        | Responsibility                    |
| ---------------- | --------------------------------- |
| Application repo | Source code + Helm chart          |
| GitHub Actions   | Build/test/scan/image push        |
| ECR              | Store Docker image                |
| GitOps repo      | Desired environment configuration |
| ApplicationSet   | Create many ArgoCD Applications   |
| ArgoCD           | Reconcile desired state with EKS  |
| Helm             | Generate Kubernetes manifests     |
| EKS              | Run containers                    |

---

# 23. Your project in one diagram

```text
                         DEVELOPER
                             |
                             v
                    order-service repo
                             |
                             v
                     GitHub Actions
                             |
              +--------------+--------------+
              |              |              |
            Test         SonarQube        Trivy
              |              |              |
              +--------------+--------------+
                             |
                             v
                           ECR
                             |
                             v
                    GitOps Repository
                             |
              +--------------+--------------+
              |              |              |
             DEV            QA           STAGING
              |              |              |
              +--------------+--------------+
                             |
                            PROD
                             |
                             v
                      ApplicationSet
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
         ArgoCD Apps    ArgoCD Apps    ArgoCD Apps
              |              |              |
              +--------------+--------------+
                             |
                             v
                            EKS
                             |
        +--------------------+--------------------+
        |         |          |         |          |
       Auth      User      Product    Order     Payment
        |         |          |         |          |
        +--------------------+--------------------+
                             |
                  RDS / Redis / SQS / SNS
```

## 24. What you actually need to implement now

Don't try to create all 84 Applications manually.

For our practical project, do this in order:

```text
STEP 1
Create ONE ApplicationSet for DEV

        ↓

STEP 2
Add only order-service

        ↓

STEP 3
Verify ApplicationSet creates
order-service-dev

        ↓

STEP 4
Verify ArgoCD deploys order-service

        ↓

STEP 5
Add user-service

        ↓

STEP 6
Verify user-service-dev is automatically created

        ↓

STEP 7
Add remaining services

        ↓

STEP 8
Create QA ApplicationSet

        ↓

STEP 9
Create STAGING ApplicationSet

        ↓

STEP 10
Create controlled PROD ApplicationSet
```

**This is the practical approach I recommend for our project.** We should not jump directly from one service to 84 Applications.

The next implementation should therefore be **one small hands-on exercise: create `ecommerce-dev` ApplicationSet and make it automatically deploy `order-service`**, with every command explained line-by-line.
