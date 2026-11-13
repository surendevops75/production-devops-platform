# Chapter 191 — GitOps Generators

Phase 20 — Platform Engineering

Complete GitHub-ready Markdown chapter · AWS EKS · ArgoCD · ApplicationSets · Helm · GitOps automation · 21 microservices

File: `191-gitops-generators.md`

## 191.1. Objective

In this chapter, we will build a reusable GitOps generation system for our enterprise e-commerce platform.

We have 21 microservices. Managing 21 ArgoCD Applications manually across DEV, QA, STAGING and PRODUCTION creates unnecessary operational work.

For example, without generators, we might maintain separate Application YAML files:

```
auth-service-dev.yaml
auth-service-qa.yaml
auth-service-staging.yaml
auth-service-production.yaml

user-service-dev.yaml
user-service-qa.yaml
user-service-staging.yaml
user-service-production.yaml

order-service-dev.yaml
order-service-qa.yaml
order-service-staging.yaml
order-service-production.yaml
```

As the number of microservices grows, this becomes difficult to maintain.

We want to define application metadata once and generate the required ArgoCD Applications consistently.

Our objectives:

1. Understand GitOps generators.
2. Understand ArgoCD Application and ApplicationSet.
3. Generate Applications for all 21 microservices.
4. Generate environment-specific Applications.
5. Manage Helm chart references.
6. Manage environment-specific values.
7. Support Git-based discovery.
8. Implement pull-request-based promotion.
9. Configure safe production deployment.
10. Validate generated resources in CI.
11. Integrate with the existing Terraform, Helm and platform repositories.
12. Implement troubleshooting and operational runbooks.

## 191.2. What is a GitOps generator?

A GitOps generator creates Kubernetes deployment configuration from structured source information.

Instead of manually writing 84 ArgoCD Application resources for 21 services across four environments, we maintain a controlled definition and generate the Applications.

The basic idea:

```
Microservice Inventory
        |
        v
GitOps Generator
        |
        v
ArgoCD ApplicationSet
        |
        v
Generated Applications
        |
        v
ArgoCD
        |
        v
Kubernetes Cluster
        |
        v
21 Microservices
```

A generator does not deploy an application by itself.

It generates Application resources that ArgoCD uses to reconcile the desired state.

## 191.3. Our production architecture

We will use the following repository responsibilities.

Enterprise e-commerce GitOps architecture

Application repositories

Auth · User · Product · Order · Payment · Other services

Source code, tests, Dockerfiles and CI pipelines

Build, scan and publish immutable image

Amazon ECR

Container images identified by immutable tags and digests

GitOps repository

Service inventory · Environment configuration · ApplicationSet definitions

ApplicationSet generators

ArgoCD

Generated Applications · Sync · Health · Drift reconciliation

DEV EKS

Development namespace

QA EKS

Testing namespace

STAGING EKS

Release validation

PRODUCTION EKS

Approved releases

The actual cluster arrangement may use one EKS cluster per environment or multiple environments in a shared cluster. This chapter uses separate environment destinations as the example design.

## 191.4. Understand ArgoCD Application

An ArgoCD Application defines:

- The source Git repository.
- The branch, tag or commit to use.
- The source path or Helm chart.
- The destination cluster.
- The destination namespace.
- The synchronization behavior.

Example:

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-dev
  namespace: argocd

spec:
  project: ecommerce

  source:
    repoURL: https://github.com/company-ecommerce/order-service.git

    targetRevision: main

    path: helm/order-service

  destination:
    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This Application deploys the Order Service chart from its Git repository.

However, this is only one application in one environment.

We need a reusable solution.

## 191.5. Understand ArgoCD ApplicationSet

An ApplicationSet creates multiple ArgoCD Applications from generator input.

For example:

```
ApplicationSet
      |
      +-- auth-service
      |
      +-- user-service
      |
      +-- customer-service
      |
      +-- catalog-service
      |
      +-- product-service
      |
      +-- ...
      |
      +-- reporting-service
```

The generated resources are normal ArgoCD Applications.

ApplicationSet is useful when many applications share a deployment pattern but differ in names, repositories, paths, clusters or values.

### 191.5.1. Application versus ApplicationSet

| Feature       | Application                     | ApplicationSet                  |
| ------------- | ------------------------------- | ------------------------------- |
| Purpose       | Defines one deployment          | Generates multiple Applications |
| Typical use   | One service/environment         | Many services/environments      |
| Configuration | Explicit source and destination | Generator plus template         |
| Scaling       | Manual duplication              | Centralized generation          |
| Ownership     | ArgoCD                          | ApplicationSet controller       |

## 191.6. Understand available generator types

ArgoCD ApplicationSet supports multiple generator types.

| Generator     | Purpose                                                        |
| ------------- | -------------------------------------------------------------- |
| List          | Generates Applications from explicit entries                   |
| Git directory | Discovers directories in a Git repository                      |
| Git file      | Reads structured files from Git                                |
| Cluster       | Generates Applications for registered clusters                 |
| Matrix        | Combines two generators                                        |
| Merge         | Combines generator outputs                                     |
| SCM Provider  | Discovers repositories from supported source-control providers |
| Pull Request  | Generates Applications for matching pull requests              |
| Plugin        | Uses an external generator plugin                              |

We will focus on the generators useful for our platform:

1. List generator.
2. Git directory generator.
3. Git file generator.
4. Matrix generator.
5. Cluster generator.
6. Merge generator.

We will also discuss Pull Request generators as an optional preview environment workflow.

## 191.7. Repository architecture

Create or update the GitOps repository:

```
company-ecommerce-gitops/
|
+-- README.md
|
+-- bootstrap/
|   |
|   +-- argocd/
|       |
|       +-- project.yaml
|       +-- root-application.yaml
|
+-- applicationsets/
|   |
|   +-- ecommerce-services.yaml
|   +-- ecommerce-environments.yaml
|   +-- ecommerce-dev.yaml
|   +-- ecommerce-qa.yaml
|   +-- ecommerce-staging.yaml
|   +-- ecommerce-production.yaml
|
+-- inventory/
|   |
|   +-- services/
|   |   |
|   |   +-- auth-service.yaml
|   |   +-- user-service.yaml
|   |   +-- customer-service.yaml
|   |   +-- catalog-service.yaml
|   |   +-- product-service.yaml
|   |   +-- inventory-service.yaml
|   |   +-- pricing-service.yaml
|   |   +-- promotion-service.yaml
|   |   +-- cart-service.yaml
|   |   +-- order-service.yaml
|   |   +-- payment-service.yaml
|   |   +-- fraud-service.yaml
|   |   +-- shipping-service.yaml
|   |   +-- delivery-service.yaml
|   |   +-- notification-service.yaml
|   |   +-- search-service.yaml
|   |   +-- recommendation-service.yaml
|   |   +-- review-service.yaml
|   |   +-- tax-service.yaml
|   |   +-- invoice-service.yaml
|   |   +-- reporting-service.yaml
|   |
|   +-- environments/
|       |
|       +-- dev.yaml
|       +-- qa.yaml
|       +-- staging.yaml
|       +-- production.yaml
|
+-- environments/
|   |
|   +-- dev/
|   |   +-- values/
|   |   +-- applications/
|   |
|   +-- qa/
|   |   +-- values/
|   |   +-- applications/
|   |
|   +-- staging/
|   |   +-- values/
|   |   +-- applications/
|   |
|   +-- production/
|       +-- values/
|       +-- applications/
|
+-- platform/
|   |
|   +-- namespaces/
|   +-- network-policies/
|   +-- service-accounts/
|   +-- monitoring/
|
+-- policies/
|   |
|   +-- production-promotion.md
|   +-- application-ownership.md
|   +-- repository-access.md
|
+-- scripts/
|   |
|   +-- validate-inventory.sh
|   +-- validate-applicationsets.sh
|   +-- check-service-count.sh
|
+-- .github/
    |
    +-- workflows/
        +-- gitops-validation.yaml
```

## 191.8. Define the 21-microservice inventory

We will define each microservice once.

Each service entry contains:

- Service name.
- Repository URL.
- Helm chart path.
- Application namespace.
- Deployment owner.
- Whether it is enabled.
- Image repository.
- Environment deployment configuration.

Example:

`inventory/services/order-service.yaml`

```
name: order-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/order-service.git

  revision: main

  chartPath: helm/order-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service

  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform

  applicationOwner: order-team

deployment:
  replicas: 2

  port: 8080
```

The inventory is the source of metadata for generation.

It does not replace the application's actual Helm values or image release configuration.

### 191.8.1. Auth Service

`inventory/services/auth-service.yaml`

```
name: auth-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/auth-service.git
  revision: main
  chartPath: helm/auth-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/auth-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: identity-team
```

### 191.8.2. User Service

`inventory/services/user-service.yaml`

```
name: user-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/user-service.git
  revision: main
  chartPath: helm/user-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/user-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: customer-team
```

### 191.8.3. Customer Service

`inventory/services/customer-service.yaml`

```
name: customer-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/customer-service.git
  revision: main
  chartPath: helm/customer-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/customer-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: customer-team
```

### 191.8.4. Catalog Service

`inventory/services/catalog-service.yaml`

```
name: catalog-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/catalog-service.git
  revision: main
  chartPath: helm/catalog-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/catalog-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: catalog-team
```

### 191.8.5. Product Service

`inventory/services/product-service.yaml`

```
name: product-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/product-service.git
  revision: main
  chartPath: helm/product-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/product-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: catalog-team
```

### 191.8.6. Inventory Service

`inventory/services/inventory-service.yaml`

```
name: inventory-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/inventory-service.git
  revision: main
  chartPath: helm/inventory-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/inventory-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: inventory-team
```

### 191.8.7. Pricing Service

`inventory/services/pricing-service.yaml`

```
name: pricing-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/pricing-service.git
  revision: main
  chartPath: helm/pricing-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/pricing-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: commerce-team
```

### 191.8.8. Promotion Service

`inventory/services/promotion-service.yaml`

```
name: promotion-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/promotion-service.git
  revision: main
  chartPath: helm/promotion-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/promotion-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: commerce-team
```

### 191.8.9. Cart Service

`inventory/services/cart-service.yaml`

```
name: cart-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/cart-service.git
  revision: main
  chartPath: helm/cart-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/cart-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: checkout-team
```

### 191.8.10. Order Service

`inventory/services/order-service.yaml`

```
name: order-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/order-service.git
  revision: main
  chartPath: helm/order-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: checkout-team
```

### 191.8.11. Payment Service

`inventory/services/payment-service.yaml`

```
name: payment-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/payment-service.git
  revision: main
  chartPath: helm/payment-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/payment-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: payments-team
```

### 191.8.12. Fraud Service

`inventory/services/fraud-service.yaml`

```
name: fraud-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/fraud-service.git
  revision: main
  chartPath: helm/fraud-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/fraud-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: risk-team
```

### 191.8.13. Shipping Service

`inventory/services/shipping-service.yaml`

```
name: shipping-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/shipping-service.git
  revision: main
  chartPath: helm/shipping-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/shipping-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: logistics-team
```

### 191.8.14. Delivery Service

`inventory/services/delivery-service.yaml`

```
name: delivery-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/delivery-service.git
  revision: main
  chartPath: helm/delivery-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/delivery-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: logistics-team
```

### 191.8.15. Notification Service

`inventory/services/notification-service.yaml`

```
name: notification-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/notification-service.git
  revision: main
  chartPath: helm/notification-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/notification-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: communications-team
```

### 191.8.16. Search Service

`inventory/services/search-service.yaml`

```
name: search-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/search-service.git
  revision: main
  chartPath: helm/search-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/search-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: discovery-team
```

### 191.8.17. Recommendation Service

`inventory/services/recommendation-service.yaml`

```
name: recommendation-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/recommendation-service.git
  revision: main
  chartPath: helm/recommendation-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/recommendation-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: discovery-team
```

### 191.8.18. Review Service

`inventory/services/review-service.yaml`

```
name: review-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/review-service.git
  revision: main
  chartPath: helm/review-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/review-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: customer-team
```

### 191.8.19. Tax Service

`inventory/services/tax-service.yaml`

```
name: tax-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/tax-service.git
  revision: main
  chartPath: helm/tax-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/tax-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: finance-team
```

### 191.8.20. Invoice Service

`inventory/services/invoice-service.yaml`

```
name: invoice-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/invoice-service.git
  revision: main
  chartPath: helm/invoice-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/invoice-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: finance-team
```

### 191.8.21. Reporting Service

`inventory/services/reporting-service.yaml`

```
name: reporting-service

enabled: true

namespace: ecommerce

repository:
  url: https://github.com/company-ecommerce/reporting-service.git
  revision: main
  chartPath: helm/reporting-service

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/reporting-service
  pullPolicy: IfNotPresent

ownership:
  team: commerce-platform
  applicationOwner: analytics-team
```

Important: The inventory files describe the generator's metadata. Their fields are not automatically available to a Git file generator unless the generator reads the corresponding files and their structure is compatible with the configured template. We will implement that explicitly below.

## 191.9. Environment configuration

We need four environments:

```
DEV
QA
STAGING
PRODUCTION
```

Each environment has its own:

- Cluster destination.
- Namespace policy.
- Values configuration.
- Image promotion state.
- Sync policy.
- Approval process.

Example:

`inventory/environments/dev.yaml`

```
environment: dev

cluster:
  name: ecommerce-dev

  server: https://dev-api.example.internal

namespace: ecommerce

autoSync: true

prune: true

selfHeal: true
```

`inventory/environments/qa.yaml`

```
environment: qa

cluster:
  name: ecommerce-qa

  server: https://qa-api.example.internal

namespace: ecommerce

autoSync: true

prune: true

selfHeal: true
```

`inventory/environments/staging.yaml`

```
environment: staging

cluster:
  name: ecommerce-staging

  server: https://staging-api.example.internal

namespace: ecommerce

autoSync: false

prune: false

selfHeal: true
```

`inventory/environments/production.yaml`

```
environment: production

cluster:
  name: ecommerce-production

  server: https://production-api.example.internal

namespace: ecommerce

autoSync: false

prune: false

selfHeal: false
```

These destination URLs are placeholders. Use the actual registered ArgoCD cluster destinations.

The sync settings are example policies, not universal production defaults. In particular, disabling self-heal in production changes how drift is handled and should be an intentional operational decision.

## 191.10. Generator 1 — List generator

The List generator is the simplest way to generate Applications from a known list.

Create:

`applicationsets/ecommerce-services.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-services-dev
  namespace: argocd

spec:
  generators:
    - list:
        elements:
          - service: auth-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/auth-service.git
            path: helm/auth-service

          - service: user-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/user-service.git
            path: helm/user-service

          - service: customer-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/customer-service.git
            path: helm/customer-service

          - service: catalog-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/catalog-service.git
            path: helm/catalog-service

          - service: product-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/product-service.git
            path: helm/product-service

          - service: inventory-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/inventory-service.git
            path: helm/inventory-service

          - service: pricing-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/pricing-service.git
            path: helm/pricing-service

          - service: promotion-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/promotion-service.git
            path: helm/promotion-service

          - service: cart-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/cart-service.git
            path: helm/cart-service

          - service: order-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/order-service.git
            path: helm/order-service

          - service: payment-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/payment-service.git
            path: helm/payment-service

          - service: fraud-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/fraud-service.git
            path: helm/fraud-service

          - service: shipping-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/shipping-service.git
            path: helm/shipping-service

          - service: delivery-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/delivery-service.git
            path: helm/delivery-service

          - service: notification-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/notification-service.git
            path: helm/notification-service

          - service: search-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/search-service.git
            path: helm/search-service

          - service: recommendation-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/recommendation-service.git
            path: helm/recommendation-service

          - service: review-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/review-service.git
            path: helm/review-service

          - service: tax-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/tax-service.git
            path: helm/tax-service

          - service: invoice-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/invoice-service.git
            path: helm/invoice-service

          - service: reporting-service
            namespace: ecommerce
            repo: https://github.com/company-ecommerce/reporting-service.git
            path: helm/reporting-service

  template:
    metadata:
      name: '{{service}}-dev'

    spec:
      project: ecommerce

      source:
        repoURL: '{{repo}}'
        targetRevision: main
        path: '{{path}}'

      destination:
        server: https://dev-api.example.internal
        namespace: '{{namespace}}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This generates 21 Applications for DEV.

### 191.10.1. How it works

For the Order Service entry:

```
- service: order-service
  namespace: ecommerce
  repo: https://github.com/company-ecommerce/order-service.git
  path: helm/order-service
```

The template substitutes:

```
metadata:
  name: order-service-dev
```

And:

```
source:
  repoURL: https://github.com/company-ecommerce/order-service.git
  targetRevision: main
  path: helm/order-service
```

### 191.10.2. Advantages

- Simple configuration.
- Easy to understand.
- Suitable for a fixed list of applications.
- No external generator required.
- Easy to validate.

### 191.10.3. Disadvantages

- Service information is duplicated.
- Every new service requires updating the list.
- It becomes lengthy with hundreds of applications.
- The inventory and ApplicationSet can become inconsistent.

For our 21-service platform, List is useful for learning and smaller deployments. For a growing platform, Git generators provide better discovery and centralized metadata.

## 191.11. Generator 2 — Git directory generator

The Git directory generator discovers directories from a Git repository.

For example:

```
company-ecommerce-gitops/
|
+-- services/
    |
    +-- auth-service/
    |   +-- Chart.yaml
    |
    +-- user-service/
    |   +-- Chart.yaml
    |
    +-- order-service/
    |   +-- Chart.yaml
    |
    +-- payment-service/
        +-- Chart.yaml
```

Create:

`applicationsets/ecommerce-git-directory.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-git-directory
  namespace: argocd

spec:
  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: main

        directories:
          - path: services/*

  template:
    metadata:
      name: '{{path.basename}}-dev'

    spec:
      project: ecommerce

      source:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        targetRevision: main

        path: '{{path}}'

      destination:
        server: https://dev-api.example.internal

        namespace: ecommerce

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 191.11.1. How the directory generator works

Suppose the Git repository contains:

```
services/auth-service
services/user-service
services/order-service
```

The generator discovers those directories and creates one Application per matching directory.

The Application name is derived from the directory basename.

### 191.11.2. Exclude a directory

You can use an exclusion pattern.

Example:

```
directories:
  - path: services/*

  - path: services/experimental-*
    exclude: true
```

This excludes directories matching the experimental pattern.

### 191.11.3. When to use it

Use a directory generator when:

- All applications follow a consistent directory structure.
- Every discovered directory represents a deployable application.
- The source repository is the GitOps repository.
- The team wants automatic discovery of new applications.

Do not point the generator at a broad repository path containing documentation, scripts or non-deployable directories.

## 191.12. Generator 3 — Git file generator

The Git file generator reads structured files from Git and exposes their fields to the ApplicationSet template.

This is useful when we maintain a service inventory in YAML or JSON.

Create a separate directory:

```
inventory/services/
|
+-- auth-service.json
+-- user-service.json
+-- customer-service.json
+-- catalog-service.json
+-- product-service.json
+-- inventory-service.json
+-- pricing-service.json
+-- promotion-service.json
+-- cart-service.json
+-- order-service.json
+-- payment-service.json
+-- fraud-service.json
+-- shipping-service.json
+-- delivery-service.json
+-- notification-service.json
+-- search-service.json
+-- recommendation-service.json
+-- review-service.json
+-- tax-service.json
+-- invoice-service.json
+-- reporting-service.json
```

Example:

`inventory/services/order-service.json`

```
{
  "name": "order-service",
  "enabled": true,
  "namespace": "ecommerce",
  "repository": {
    "url": "https://github.com/company-ecommerce/order-service.git",
    "revision": "main",
    "chartPath": "helm/order-service"
  },
  "ownership": {
    "team": "commerce-platform",
    "applicationOwner": "checkout-team"
  }
}
```

The JSON file is intentionally used here because its field structure is straightforward to demonstrate.

Create:

`applicationsets/ecommerce-git-file.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-git-file-dev
  namespace: argocd

spec:
  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: main

        files:
          - path: inventory/services/*.json

  template:
    metadata:
      name: '{{name}}-dev'

      labels:
        app.kubernetes.io/part-of: ecommerce
        environment: dev

    spec:
      project: ecommerce

      source:
        repoURL: '{{repository.url}}'

        targetRevision: '{{repository.revision}}'

        path: '{{repository.chartPath}}'

      destination:
        server: https://dev-api.example.internal

        namespace: '{{namespace}}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 191.12.1. Understand field substitution

For the JSON:

```
{
  "name": "order-service",
  "namespace": "ecommerce"
}
```

The template:

```
metadata:
  name: '{{name}}-dev'
```

generates:

```
metadata:
  name: order-service-dev
```

And:

```
destination:
  namespace: '{{namespace}}'
```

generates:

```
destination:
  namespace: ecommerce
```

### 191.12.2. Enable Go templates

For more advanced logic, enable Go templates.

```
spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error
```

Then use Go template expressions.

For example:

```
metadata:
  name: '{{ .name }}-dev'
```

And:

```
source:
  repoURL: '{{ .repository.url }}'
  targetRevision: '{{ .repository.revision }}'
  path: '{{ .repository.chartPath }}'
```

When `goTemplate: true` is enabled, use the Go template syntax consistently. Do not mix it with the legacy fast-template syntax.

### 191.12.3. Filter disabled applications

We can add a selector to the Git generator to select only inventory entries with the enabled label if the repository structure and generator configuration provide that label.

For example, if generator values expose a suitable `enabled` value, use an appropriate `selector` or template-level filtering strategy supported by the installed ApplicationSet version.

Do not assume that a YAML or JSON boolean automatically becomes a Kubernetes label. Labels are strings, and filtering must match the actual generated parameter structure.

For this chapter, the production implementation should validate that only approved inventory entries generate Applications. A simple, reliable approach is to keep only enabled service files under the generator's discovery path and move disabled entries to a separate directory.

Example:

```
inventory/
|
+-- services-enabled/
|   +-- auth-service.json
|   +-- order-service.json
|
+-- services-disabled/
    +-- experimental-service.json
```

Then configure:

```
files:
  - path: inventory/services-enabled/*.json
```

This avoids depending on ambiguous boolean conversion.

## 191.13. Generator 4 — Matrix generator

The Matrix generator combines the output of two generators.

For our use case:

- Generator 1 provides services.
- Generator 2 provides environments.

The combination generates service/environment Applications.

If we have:

- 21 services.
- 4 environments.

The theoretical result is:

\\[ 21 \times 4 = 84 \\]

Applications.

### 191.13.1. Matrix generator example

Create:

`applicationsets/ecommerce-matrix.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-matrix
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

              revision: main

              files:
                - path: inventory/services/*.json

          - list:
              elements:
                - environment: dev
                  clusterServer: https://dev-api.example.internal
                  namespace: ecommerce

                - environment: qa
                  clusterServer: https://qa-api.example.internal
                  namespace: ecommerce

                - environment: staging
                  clusterServer: https://staging-api.example.internal
                  namespace: ecommerce

                - environment: production
                  clusterServer: https://production-api.example.internal
                  namespace: ecommerce

  template:
    metadata:
      name: '{{ .name }}-{{ .environment }}'

      labels:
        app.kubernetes.io/part-of: ecommerce
        environment: '{{ .environment }}'

    spec:
      project: ecommerce

      source:
        repoURL: '{{ .repository.url }}'

        targetRevision: '{{ .repository.revision }}'

        path: '{{ .repository.chartPath }}'

      destination:
        server: '{{ .clusterServer }}'

        namespace: '{{ .namespace }}'
```

This example demonstrates the matrix combination. Before using it in production, confirm that all generated parameters are compatible and that the installed ApplicationSet controller supports the exact template and generator configuration.

### 191.13.2. Expected generated Application names

```
auth-service-dev
auth-service-qa
auth-service-staging
auth-service-production

user-service-dev
user-service-qa
user-service-staging
user-service-production

order-service-dev
order-service-qa
order-service-staging
order-service-production

payment-service-dev
payment-service-qa
payment-service-staging
payment-service-production
```

The full result contains 84 Applications if all 21 services are enabled in all four environments.

### 191.13.3. Why Matrix is useful

Without Matrix:

```
DEV ApplicationSet
QA ApplicationSet
STAGING ApplicationSet
PRODUCTION ApplicationSet
```

We maintain four separate configurations.

With Matrix:

```
Service generator
       +
Environment generator
       |
       v
Matrix generator
       |
       v
All service/environment combinations
```

However, generating 84 Applications does not mean we should automatically deploy all 84 at the same time. Promotion and approval must remain controlled.

## 191.14. Production caution — do not automatically promote every image

The Matrix generator creates Application resources.

It does not understand whether an image has passed:

- Functional testing.
- Integration testing.
- Security scanning.
- Performance testing.
- Release approval.
- Production change approval.

Do not configure production to consume an uncontrolled `latest` tag.

Preferred release configuration:

```
image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service

  tag: "1.4.2"

  digest: "sha256:REPLACE_WITH_APPROVED_DIGEST"
```

The actual application chart must use the selected image reference consistently.

For stronger release integrity, deploy using an immutable image digest:

```
image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service

  digest: sha256:REPLACE_WITH_APPROVED_DIGEST
```

The application chart must render the digest form correctly, for example:

```
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

Use either a tag or a digest according to the chart's validated interface. Do not append both in an invalid image reference.

## 191.15. Generator 5 — Cluster generator

The Cluster generator creates Applications for clusters registered with ArgoCD.

This is useful when we manage multiple clusters.

Example:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-cluster-generator
  namespace: argocd

spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: dev

  template:
    metadata:
      name: 'ecommerce-{{name}}'

    spec:
      project: ecommerce

      source:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        targetRevision: main

        path: applications/ecommerce

      destination:
        server: '{{server}}'

        namespace: ecommerce
```

### 191.15.1. Register cluster labels

In ArgoCD, clusters can have labels.

Example intended labels:

```
environment=dev
environment=qa
environment=staging
environment=production
```

The Cluster generator selects clusters based on those labels.

### 191.15.2. Inspect registered clusters

```
argocd cluster list
```

Review:

- Cluster name.
- API server.
- Connection status.
- Namespace configuration.
- Environment labels.

The exact cluster-label management workflow depends on how the cluster was registered and which ArgoCD version is installed.

### 191.15.3. Production protection

Never allow a general cluster generator to deploy an application into every registered cluster without an explicit selector.

Use separate ApplicationSets or strict selectors for production.

For example:

```
selector:
  matchLabels:
    environment: dev
```

Production must have a separate, approved release path.

## 191.16. Generator 6 — Merge generator

The Merge generator combines parameters from multiple generators using a merge key.

It is useful when we need to override selected service settings.

For example:

- A base service list defines the normal deployment.
- An override list provides additional configuration for specific services.

Illustrative configuration:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-merge-example
  namespace: argocd

spec:
  generators:
    - merge:
        mergeKeys:
          - service

        generators:
          - list:
              elements:
                - service: order-service
                  namespace: ecommerce
                  replicas: "2"

                - service: payment-service
                  namespace: ecommerce
                  replicas: "2"

          - list:
              elements:
                - service: payment-service
                  replicas: "3"

  template:
    metadata:
      name: '{{service}}-dev'

    spec:
      project: ecommerce

      source:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        targetRevision: main

        path: applications/{{service}}

      destination:
        server: https://dev-api.example.internal

        namespace: '{{namespace}}'
```

The merged output can provide the overridden value for Payment Service.

Important: Merge behavior depends on the configured merge keys and generator parameter structure. Do not assume arbitrary nested fields will merge recursively. Test the rendered parameters using the installed ApplicationSet controller before relying on the behavior.

## 191.17. Use Helm values for environment differences

The generator decides which Application to create.

Helm values decide how that application is configured.

For example:

```
DEV
|
+-- replicas: 1
+-- logLevel: debug
+-- resource limits: smaller

QA
|
+-- replicas: 2
+-- logLevel: info
+-- test configuration

STAGING
|
+-- replicas: 2
+-- logLevel: info
+-- production-like configuration

PRODUCTION
|
+-- replicas: 3
+-- logLevel: warn
+-- production resource limits
```

Create:

```
environments/
|
+-- dev/
|   +-- values/
|       +-- order-service.yaml
|
+-- qa/
|   +-- values/
|       +-- order-service.yaml
|
+-- staging/
|   +-- values/
|       +-- order-service.yaml
|
+-- production/
    +-- values/
        +-- order-service.yaml
```

### 191.17.1. DEV values

`environments/dev/values/order-service.yaml`

```
replicaCount: 1

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.4.2"
  pullPolicy: IfNotPresent

service:
  port: 8080

resources:
  requests:
    cpu: 100m
    memory: 256Mi

  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

environment:
  LOG_LEVEL: debug
```

### 191.17.2. QA values

`environments/qa/values/order-service.yaml`

```
replicaCount: 2

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.4.2"
  pullPolicy: IfNotPresent

service:
  port: 8080

resources:
  requests:
    cpu: 200m
    memory: 384Mi

  limits:
    cpu: 750m
    memory: 768Mi

autoscaling:
  enabled: false

environment:
  LOG_LEVEL: info
```

### 191.17.3. STAGING values

`environments/staging/values/order-service.yaml`

```
replicaCount: 2

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.4.2"
  pullPolicy: IfNotPresent

service:
  port: 8080

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true

  minReplicas: 2

  maxReplicas: 5

environment:
  LOG_LEVEL: info
```

### 191.17.4. PRODUCTION values

`environments/production/values/order-service.yaml`

```
replicaCount: 3

image:
  repository: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:REPLACE_WITH_APPROVED_DIGEST"
  pullPolicy: IfNotPresent

service:
  port: 8080

resources:
  requests:
    cpu: 500m
    memory: 1Gi

  limits:
    cpu: 1500m
    memory: 2Gi

autoscaling:
  enabled: true

  minReplicas: 3

  maxReplicas: 10

environment:
  LOG_LEVEL: warn
```

These resource values are illustrative starting points. Production sizing must be based on load testing, observed consumption, latency objectives and capacity planning.

## 191.18. Connect ApplicationSet to environment values

A Helm-based ArgoCD Application can reference values files.

Example:

```
source:
  repoURL: https://github.com/company-ecommerce/order-service.git

  targetRevision: main

  path: helm/order-service

  helm:
    valueFiles:
      - values.yaml
      - values-dev.yaml
```

When environment values are stored in a separate GitOps repository, use ArgoCD multi-source Applications where supported by the installed version.

Illustrative multi-source Application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-dev
  namespace: argocd

spec:
  project: ecommerce

  sources:
    - repoURL: https://github.com/company-ecommerce/order-service.git

      targetRevision: main

      path: helm/order-service

      helm:
        valueFiles:
          - $values/environments/dev/values/order-service.yaml

    - repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

      targetRevision: main

      ref: values

  destination:
    server: https://dev-api.example.internal

    namespace: ecommerce
```

Here:

- The first source provides the Helm chart.
- The second source provides environment values.
- `$values` references the source identified by `ref: values`.

The values file path is relative to the root of the referenced Git repository.

This approach avoids duplicating Helm charts for every environment.

## 191.19. Production ApplicationSet design

For a real platform, it is often safer to separate production from lower environments.

Why?

Production has different controls:

- Change approval.
- Protected Git branches.
- Immutable release versions.
- Manual or controlled synchronization.
- More restrictive repository access.
- Release audit requirements.

Recommended structure:

```
applicationsets/
|
+-- ecommerce-dev.yaml
+-- ecommerce-qa.yaml
+-- ecommerce-staging.yaml
+-- ecommerce-production.yaml
```

### 191.19.1. DEV ApplicationSet

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-dev
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: main

        files:
          - path: inventory/services/*.json

  template:
    metadata:
      name: '{{ .name }}-dev'

      labels:
        environment: dev
        app.kubernetes.io/part-of: ecommerce

    spec:
      project: ecommerce

      source:
        repoURL: '{{ .repository.url }}'

        targetRevision: '{{ .repository.revision }}'

        path: '{{ .repository.chartPath }}'

        helm:
          valueFiles:
            - values.yaml

      destination:
        server: https://dev-api.example.internal

        namespace: '{{ .namespace }}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This is a simplified example. It assumes each application's chart has a suitable values file. For values stored in a separate GitOps repository, use the multi-source approach described earlier.

### 191.19.2. QA ApplicationSet

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-qa
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: main

        files:
          - path: inventory/services/*.json

  template:
    metadata:
      name: '{{ .name }}-qa'

      labels:
        environment: qa
        app.kubernetes.io/part-of: ecommerce

    spec:
      project: ecommerce

      source:
        repoURL: '{{ .repository.url }}'

        targetRevision: '{{ .repository.revision }}'

        path: '{{ .repository.chartPath }}'

      destination:
        server: https://qa-api.example.internal

        namespace: '{{ .namespace }}'
```

### 191.19.3. STAGING ApplicationSet

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-staging
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: main

        files:
          - path: inventory/services/*.json

  template:
    metadata:
      name: '{{ .name }}-staging'

      labels:
        environment: staging
        app.kubernetes.io/part-of: ecommerce

    spec:
      project: ecommerce

      source:
        repoURL: '{{ .repository.url }}'

        targetRevision: '{{ .repository.revision }}'

        path: '{{ .repository.chartPath }}'

      destination:
        server: https://staging-api.example.internal

        namespace: '{{ .namespace }}'
```

### 191.19.4. PRODUCTION ApplicationSet

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-production
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

        revision: production

        files:
          - path: inventory/services/*.json

  template:
    metadata:
      name: '{{ .name }}-production'

      labels:
        environment: production
        app.kubernetes.io/part-of: ecommerce

    spec:
      project: ecommerce

      source:
        repoURL: '{{ .repository.url }}'

        targetRevision: '{{ .repository.revision }}'

        path: '{{ .repository.chartPath }}'

      destination:
        server: https://production-api.example.internal

        namespace: '{{ .namespace }}'

      syncPolicy:
        syncOptions:
          - CreateNamespace=false
```

This production example intentionally omits automated synchronization.

It still needs a complete production release process: reviewed Git changes, protected promotion, appropriate sync approval and an audited deployment. Also ensure that the generated Application source references the approved immutable release rather than an uncontrolled branch head.

## 191.20. ArgoCD AppProject

ApplicationSets should not bypass project restrictions.

Create:

`bootstrap/argocd/project.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject

metadata:
  name: ecommerce

  namespace: argocd

spec:
  description: E-commerce application project

  sourceRepos:
    - https://github.com/company-ecommerce/*

  destinations:
    - namespace: ecommerce
      server: https://dev-api.example.internal

    - namespace: ecommerce
      server: https://qa-api.example.internal

    - namespace: ecommerce
      server: https://staging-api.example.internal

    - namespace: ecommerce
      server: https://production-api.example.internal

  clusterResourceWhitelist: []

  namespaceResourceWhitelist:
    - group: ""
      kind: Service

    - group: ""
      kind: ConfigMap

    - group: ""
      kind: Secret

    - group: apps
      kind: Deployment

    - group: apps
      kind: StatefulSet

    - group: networking.k8s.io
      kind: NetworkPolicy

    - group: autoscaling
      kind: HorizontalPodAutoscaler
```

This is a starting example, not a complete universal whitelist.

Add only the resource types required by the actual application charts. If applications need custom resources, explicitly review and allow the necessary API groups and kinds.

Do not grant applications unrestricted cluster-scoped resource creation.

### 191.20.1. Project security considerations

Review:

- Which Git repositories are trusted.
- Which clusters are allowed.
- Which namespaces are allowed.
- Which resource kinds are allowed.
- Which roles can synchronize Applications.
- Whether project roles have excessive permissions.

A project that allows every repository and every destination provides weak isolation.

## 191.21. Bootstrap ArgoCD using a root Application

A common GitOps pattern is the app-of-apps pattern.

A root Application manages the child ApplicationSets and other platform resources.

Create:

`bootstrap/argocd/root-application.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: ecommerce-root

  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/company-ecommerce/company-ecommerce-gitops.git

    targetRevision: main

    path: applicationsets

    directory:
      recurse: true

  destination:
    server: https://kubernetes.default.svc

    namespace: argocd

  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

Apply the root Application during the controlled ArgoCD bootstrap process:

```
kubectl apply -f bootstrap/argocd/root-application.yaml
```

This is a bootstrap operation. Once the root Application is installed and connected, the normal management of its child resources should proceed through GitOps.

The root Application must be protected carefully because a mistake in its repository path or pruning configuration can affect many child resources.

## 191.22. ApplicationSet lifecycle and deletion protection

One of the most important production concerns is what happens when a generator stops producing an Application.

For example:

```
Before:
order-service-dev exists

Inventory entry removed:
order-service is no longer discovered

After:
ApplicationSet no longer generates order-service-dev
```

Depending on the ApplicationSet deletion policy and installed ArgoCD behavior, removing an Application from the generated set can cause the Application resource to be deleted.

If the Application is deleted, its managed resources may also be deleted depending on the Application's finalizers and deletion behavior.

### 191.22.1. Understand the lifecycle

```
Inventory entry exists
        |
        v
ApplicationSet generates Application
        |
        v
ArgoCD manages Application
        |
        v
Application manages Kubernetes resources
        |
        v
Inventory entry removed
        |
        v
Generated Application disappears
        |
        v
Deletion policy determines outcome
```

Never treat deleting an inventory file as a harmless cleanup operation.

### 191.22.2. Configure an ApplicationSet deletion policy

ApplicationSet supports policies that control deletion behavior.

For example:

```
spec:
  syncPolicy:
    applicationsSync: create-update
```

The `create-update` policy allows the ApplicationSet controller to create and update generated Applications without deleting them when they are no longer generated.

The precise policy options available depend on the installed ApplicationSet controller version and controller configuration.

For sensitive production ApplicationSets, evaluate:

- `applicationsSync`.
- `preserveResourcesOnDeletion`.
- Application finalizers.
- AppProject restrictions.
- Manual deletion approval.

### 191.22.3. Preserve resources where appropriate

Example:

```
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true
```

This setting controls whether resources managed by generated Applications are preserved when those Applications are deleted.

It is not a substitute for a backup and recovery strategy.

It can also leave orphaned resources that need an explicit cleanup process.

### 191.22.4. Production deletion policy

Recommended operational process:

1. Disable the service through an approved change.
2. Confirm the service is not receiving production traffic.
3. Check dependencies.
4. Review persistent data and storage.
5. Back up stateful data where applicable.
6. Review Application finalizers and resource deletion behavior.
7. Obtain the required approval.
8. Remove the service from the active generator inventory.
9. Verify that the expected Application lifecycle behavior occurred.
10. Clean up retained resources only through a separate approved process.

## 191.23. GitHub Actions validation for GitOps generators

We must validate the GitOps repository before merging changes.

The pipeline should check:

- YAML syntax.
- JSON syntax.
- Inventory structure.
- Duplicate service names.
- Required repository fields.
- ApplicationSet manifests.
- Generated Application naming.
- Environment configuration.
- Unexpected production destination changes.

Create:

`.github/workflows/gitops-validation.yaml`

```
name: GitOps Validation

on:
  pull_request:
    paths:
      - "applicationsets/**"
      - "inventory/**"
      - "environments/**"
      - "bootstrap/**"

  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  validate:
    name: Validate GitOps Configuration

    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install yq
        run: |
          sudo snap install yq

      - name: Install jq
        run: |
          sudo apt-get update
          sudo apt-get install -y jq

      - name: Validate JSON files
        run: |
          find inventory -name '*.json' -print0 |
          while IFS= read -r -d '' file; do
            jq empty "$file"
          done

      - name: Validate service count
        run: |
          count=$(find inventory/services -name '*.json' | wc -l)

          echo "Discovered service files: $count"

          if [ "$count" -ne 21 ]; then
            echo "Expected 21 service inventory files"
            exit 1
          fi

      - name: Validate required service fields
        run: |
          for file in inventory/services/*.json; do

            jq -e '
              .name and
              .namespace and
              .repository.url and
              .repository.revision and
              .repository.chartPath
            ' "$file" > /dev/null

          done

      - name: Check duplicate service names
        run: |
          names=$(find inventory/services -name '*.json' \
            -exec jq -r '.name' {} \;)

          duplicates=$(printf '%s\n' "$names" |
            sort |
            uniq -d)

          if [ -n "$duplicates" ]; then
            echo "Duplicate service names:"
            echo "$duplicates"
            exit 1
          fi

      - name: Validate YAML syntax
        run: |
          find applicationsets bootstrap environments \
            -name '*.yaml' -o -name '*.yml' |
          while read -r file; do
            yq eval '.' "$file" > /dev/null
          done
```

This is a starter validation workflow. For production, pin third-party actions to reviewed commit SHAs, pin tool versions, handle filenames safely, and use a schema-aware validator for Kubernetes and ArgoCD resources.

Also note that the `find` expression should be grouped when both extensions are included in a larger production script. A safer version is:

```
find applicationsets bootstrap environments \
  \( -name '*.yaml' -o -name '*.yml' \) \
  -type f
```

## 191.24. Validate ApplicationSet resources

Install the appropriate Kubernetes and ArgoCD schemas into your validation process.

A schema validator can check whether a manifest matches the installed API schema.

For example, with a pinned `kubeconform` binary:

```
kubeconform \
  -strict \
  -summary \
  applicationsets/
```

A standard Kubernetes schema bundle may not include every ArgoCD custom resource definition schema.

Therefore, configure the validator with the appropriate CRD schemas for:

- `Application`.
- `ApplicationSet`.
- `AppProject`.

Do not assume that a successful generic Kubernetes validation proves an ArgoCD custom resource is valid.

## 191.25. Test generator output before deployment

Before deploying an ApplicationSet to production, test its output.

A useful validation environment should include:

- The same ArgoCD version as the target environment.
- The same ApplicationSet controller behavior.
- The relevant CRDs.
- Representative inventory files.
- Representative environment configuration.
- The actual repository access configuration.

Check:

```
kubectl get applicationsets -n argocd
```

Check generated Applications:

```
kubectl get applications -n argocd
```

Check one Application:

```
kubectl describe application order-service-dev \
  -n argocd
```

Inspect ApplicationSet status:

```
kubectl describe applicationset ecommerce-dev \
  -n argocd
```

Review controller logs when generation fails:

```
kubectl logs deployment/argocd-applicationset-controller \
  -n argocd \
  --tail=200
```

The exact controller deployment name can differ by installation.

## 191.26. Practical deployment workflow

Now we will follow the real workflow for adding a new microservice.

Suppose we introduce a new service called:

```
returns-service
```

The service is not yet part of the 21-service inventory.

### Step 1 — Developer creates the application repository

```
company-ecommerce/returns-service
```

The repository contains:

```
returns-service/
|
+-- src/
+-- tests/
+-- Dockerfile
+-- pom.xml
+-- helm/
|   +-- returns-service/
|       +-- Chart.yaml
|       +-- values.yaml
|       +-- templates/
|
+-- .github/
    +-- workflows/
```

### Step 2 — Developer implements application functionality

The developer builds and tests the service.

### Step 3 — CI builds the Docker image

The CI pipeline:

1. Checks out the source.
2. Runs unit tests.
3. Runs code quality checks.
4. Runs security scanning.
5. Builds the Docker image.
6. Scans the image.
7. Publishes the approved image to ECR.

Example image:

```
111122223333.dkr.ecr.ap-south-1.amazonaws.com/returns-service:1.0.0
```

The release record should capture the immutable digest.

### Step 4 — Add the service inventory

Create:

`inventory/services/returns-service.json`

```
{
  "name": "returns-service",
  "enabled": true,
  "namespace": "ecommerce",
  "repository": {
    "url": "https://github.com/company-ecommerce/returns-service.git",
    "revision": "main",
    "chartPath": "helm/returns-service"
  },
  "ownership": {
    "team": "commerce-platform",
    "applicationOwner": "order-team"
  }
}
```

### Step 5 — Add environment values

Create:

```
environments/dev/values/returns-service.yaml
environments/qa/values/returns-service.yaml
environments/staging/values/returns-service.yaml
environments/production/values/returns-service.yaml
```

### Step 6 — Submit a pull request

The PR includes:

- Inventory file.
- Environment values.
- Application ownership.
- NetworkPolicy requirements.
- ServiceAccount requirements.
- Monitoring configuration.
- Deployment resource requirements.

### Step 7 — Run validation

GitHub Actions validates:

- Inventory syntax.
- Required fields.
- Duplicate names.
- ApplicationSet manifests.
- Environment values.
- Chart rendering.

### Step 8 — Review and merge

The application owner and platform team review the change.

Production changes require the appropriate additional approvals.

### Step 9 — ApplicationSet discovers the new service

The Git file generator detects:

```
inventory/services/returns-service.json
```

It generates the corresponding Applications for the environments included in its generator configuration.

### Step 10 — ArgoCD reconciles the application

ArgoCD reads the application repository and renders the Helm chart.

### Step 11 — Deployment validation

Check:

```
kubectl get deployments -n ecommerce
```

Check pods:

```
kubectl get pods -n ecommerce
```

Check ArgoCD:

```
argocd app get returns-service-dev
```

### Step 12 — Promote through environments

```
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

The same approved image digest should be promoted between environments unless a new build is explicitly approved.

## 191.27. Promotion workflow — do not rebuild for each environment

A major GitOps principle is to promote the same tested artifact.

Bad workflow:

```
Build DEV image
       |
       v
Build QA image
       |
       v
Build STAGING image
       |
       v
Build PRODUCTION image
```

This can result in different binaries being deployed to each environment.

Preferred:

```
Build application once
       |
       v
Publish image to ECR
       |
       v
Capture image digest
       |
       v
Deploy to DEV
       |
       v
Validate DEV
       |
       v
Promote same digest to QA
       |
       v
Validate QA
       |
       v
Promote same digest to STAGING
       |
       v
Validate STAGING
       |
       v
Production approval
       |
       v
Promote same digest to PRODUCTION
```

The GitOps change updates the environment's approved release reference.

### 191.27.1. Promotion PR example

For example, DEV has:

```
image:
  digest: sha256:abc123...
```

After testing, QA is updated to the same digest.

Then STAGING.

Finally, PRODUCTION is updated after approval.

The digest values above are illustrative. Use the actual ECR image digest from the build pipeline.

## 191.28. Rollback using GitOps

Suppose the current production image is:

```
sha256:NEW_DIGEST
```

The previous approved image was:

```
sha256:PREVIOUS_DIGEST
```

If the new release causes an incident:

1. Identify the previous known-good digest.
2. Confirm the rollback version is compatible with current data and APIs.
3. Open an emergency or normal rollback change according to the incident process.
4. Update the production values to the previous digest.
5. Review and approve the change.
6. Merge the GitOps change.
7. Allow ArgoCD to reconcile the desired state.
8. Validate application health.
9. Monitor errors and latency.

The desired state in Git becomes:

```
image:
  digest: sha256:PREVIOUS_DIGEST
```

Do not manually change the running Deployment and leave Git out of sync.

Also remember that a container rollback does not automatically reverse database migrations, messages already published or external side effects. Those require separate recovery plans.

## 191.29. Troubleshooting

### Scenario 1 — ApplicationSet creates no Applications

Check:

```
kubectl get applicationsets -n argocd
```

Inspect:

```
kubectl describe applicationset ecommerce-dev \
  -n argocd
```

Check the ApplicationSet controller logs.

Possible causes:

- Git repository authentication failure.
- Incorrect repository URL.
- Wrong revision.
- Incorrect file path.
- JSON syntax error.
- Template field does not exist.
- Generator selector excludes all files.
- ApplicationSet controller issue.

### Scenario 2 — Generated Application has an invalid name

Example:

```
Invalid value for metadata.name
```

Kubernetes resource names must follow the relevant DNS naming rules.

Check:

- Lowercase letters.
- Numbers.
- Hyphens.
- Maximum length.
- No invalid punctuation.
- No unexpected spaces.

Use stable service slugs such as:

```
order-service-dev
payment-service-production
```

### Scenario 3 — Application is generated but OutOfSync

Check:

```
argocd app get order-service-dev
```

Inspect the diff:

```
argocd app diff order-service-dev
```

Possible causes:

- Wrong Helm values.
- Wrong image digest.
- Manually changed Kubernetes resource.
- Different rendered chart version.
- Missing configuration.
- Resource defaults applied by Kubernetes.

### Scenario 4 — Application is generated but repository cannot be accessed

Check:

- Repository URL.
- Repository credentials.
- GitHub App or deploy key configuration.
- Branch or tag.
- Network access from ArgoCD repo-server.
- Repository permissions.

Do not place GitHub credentials in the ApplicationSet manifest.

Use ArgoCD's supported repository credential management.

### Scenario 5 — Application points to the wrong cluster

Check:

```
argocd cluster list
```

Check the generated Application:

```
kubectl get application order-service-dev \
  -n argocd \
  -o yaml
```

Inspect:

```
spec:
  destination:
    server: ...
```

A wrong destination can cause a service to be deployed to the wrong environment.

Production destinations must be protected through review and policy enforcement.

### Scenario 6 — New inventory file does not generate an Application

Check the file:

```
jq empty inventory/services/order-service.json
```

Check its name:

```
jq -r '.name' inventory/services/order-service.json
```

Check the generator file pattern:

```
files:
  - path: inventory/services/*.json
```

Check that the file is committed to the configured revision.

### Scenario 7 — ApplicationSet changes delete an Application

Investigate:

- The inventory entry.
- Generator filters.
- ApplicationSet deletion policy.
- Application finalizers.
- `preserveResourcesOnDeletion`.
- Whether the Application was intentionally removed.

Do not immediately delete remaining Kubernetes resources. First determine the intended lifecycle and the data impact.

### Scenario 8 — Production updated without approval

Treat this as a release-control incident.

Investigate:

- Who merged the Git change?
- Which branch was used?
- Which ApplicationSet generated the Application?
- Was automatic sync enabled?
- Were branch protection rules bypassed?
- Did a user manually synchronize the Application?
- Were ArgoCD permissions too broad?

Corrective controls may include:

- Protected production branch.
- Required reviewers.
- Restricted ArgoCD sync permissions.
- Immutable release references.
- Production-specific ApplicationSet.
- Change-management integration.
- Audit logging.

## 191.30. Operational commands

### List ApplicationSets

```
kubectl get applicationsets -n argocd
```

### Describe ApplicationSet

```
kubectl describe applicationset ecommerce-dev \
  -n argocd
```

### List Applications

```
kubectl get applications -n argocd
```

### Filter DEV applications

```
kubectl get applications -n argocd \
  -l environment=dev
```

### Inspect an Application

```
kubectl get application order-service-dev \
  -n argocd \
  -o yaml
```

### Get Application status

```
argocd app get order-service-dev
```

### View application diff

```
argocd app diff order-service-dev
```

### Sync an application

```
argocd app sync order-service-dev
```

Use manual sync only where authorized by the environment's deployment policy.

### View controller logs

```
kubectl logs deployment/argocd-applicationset-controller \
  -n argocd \
  --tail=200
```

### View repository server logs

```
kubectl logs deployment/argocd-repo-server \
  -n argocd \
  --tail=200
```

## 191.31. Security controls for GitOps generators

A generator can create a large number of deployments from a small configuration change.

That makes the generator itself a security-sensitive platform component.

Implement these controls.

| Control                     | Purpose                                                  |
| --------------------------- | -------------------------------------------------------- |
| Protected Git branches      | Prevent unauthorized changes                             |
| Required PR reviews         | Review deployment changes                                |
| CODEOWNERS                  | Require platform team review                             |
| AppProjects                 | Restrict repositories and destinations                   |
| Namespace restrictions      | Prevent uncontrolled namespace access                    |
| Generator path restrictions | Avoid discovering unintended directories                 |
| Immutable image digests     | Prevent image drift                                      |
| Production approval         | Control production releases                              |
| Audit logs                  | Track changes and synchronization                        |
| Deletion policies           | Prevent accidental mass deletion                         |
| RBAC                        | Restrict who can modify Applications and ApplicationSets |
| Schema validation           | Catch invalid resources before merge                     |

### 191.31.1. CODEOWNERS example

Create:

`.github/CODEOWNERS`

```
/applicationsets/ @company-ecommerce/platform-team

/inventory/ @company-ecommerce/platform-team

/environments/production/ @company-ecommerce/platform-team

/bootstrap/ @company-ecommerce/platform-team

/policies/ @company-ecommerce/security-team
```

This is an example ownership configuration. Use the actual GitHub team names and repository permissions.

### 191.31.2. Protect production changes

Recommended production pull request checks:

- At least one platform review.
- Application owner review.
- Security review when permissions or exposure change.
- Immutable image digest.
- Validated Helm rendering.
- Successful environment test.
- Approved production change.
- Rollback version identified.

## 191.32. GitOps generator design decisions

Before implementation, agree on these standards.

| Design question                                | Platform decision                         |
| ---------------------------------------------- | ----------------------------------------- |
| What creates Applications?                     | ApplicationSet                            |
| Where is service metadata stored?              | GitOps inventory                          |
| How are environments represented?              | Separate environment configuration        |
| How are deployment charts selected?            | Repository and chart path metadata        |
| How are images promoted?                       | Immutable image digest                    |
| Who approves production?                       | Authorized release approvers              |
| How are Applications deleted?                  | Controlled lifecycle policy               |
| Who owns generator definitions?                | Platform Engineering                      |
| Who owns application deployment configuration? | Application owner with platform standards |
| How are changes audited?                       | Git PR history and ArgoCD audit trail     |

## 191.33. Recommended production generator strategy

For our 21-microservice platform, use this approach:

| Requirement               | Recommended implementation             |
| ------------------------- | -------------------------------------- |
| Service discovery         | Git file generator                     |
| Environment combinations  | Matrix generator where appropriate     |
| Fixed platform components | List generator                         |
| Multi-cluster targeting   | Cluster generator                      |
| Environment values        | Separate GitOps values repository/path |
| Production control        | Dedicated production ApplicationSet    |
| Application deployment    | ArgoCD Application                     |
| Infrastructure            | Terraform                              |
| Application packaging     | Helm                                   |
| Image publishing          | CI pipeline and ECR                    |
| Release promotion         | Reviewed GitOps PR                     |
| Drift management          | ArgoCD reconciliation                  |

Do not use every generator simply because it exists. Choose the smallest set that solves the actual platform requirement.

## 191.34. Practical lab

We will implement the workflow in DEV before enabling it in the other environments.

### Lab objective

Generate an ArgoCD Application for Order Service from a Git inventory file.

### Prerequisites

- EKS DEV cluster.
- ArgoCD installed.
- ApplicationSet controller installed.
- GitOps repository accessible to ArgoCD.
- Order Service Helm chart available.
- `ecommerce` namespace available.
- Required repository credentials configured.

### Step 1 — Confirm ArgoCD

```
kubectl get pods -n argocd
```

### Step 2 — Confirm ApplicationSet CRD

```
kubectl get crd applicationsets.argoproj.io
```

### Step 3 — Create inventory file

```
inventory/services/order-service.json
```

Use the Order Service JSON example from this chapter.

### Step 4 — Create the DEV ApplicationSet

Use the Git file generator configuration.

### Step 5 — Validate the manifest

```
kubectl apply --dry-run=server \
  -f applicationsets/ecommerce-dev.yaml
```

Server-side dry run validates the submitted resource against the API server, but does not prove that the ApplicationSet controller can successfully generate and reconcile every resulting Application.

### Step 6 — Commit and push

```
git add inventory applicationsets

git commit -m "Add Order Service DEV GitOps generator"

git push
```

### Step 7 — Deploy through the bootstrap GitOps process

The ApplicationSet should be managed through the approved root Application or platform bootstrap process.

### Step 8 — Verify generated Application

```
kubectl get applications -n argocd
```

Expected:

```
order-service-dev
```

### Step 9 — Check sync status

```
argocd app get order-service-dev
```

### Step 10 — Validate application deployment

```
kubectl get deployment order-service \
  -n ecommerce
```

```
kubectl get pods \
  -n ecommerce \
  -l app.kubernetes.io/name=order-service
```

### Step 11 — Validate the application endpoint

Use the approved internal test endpoint or service route.

Check:

- Application responds.
- Health endpoint succeeds.
- Dependencies are reachable.
- Logs are available.
- Prometheus metrics are collected.
- NetworkPolicies permit only required communication.

### Step 12 — Test a GitOps update

Change the DEV replica count in the Helm values.

Submit a PR and merge after validation.

Confirm that ArgoCD reconciles the change.

### Step 13 — Test a rejected change

Introduce an invalid chart value or invalid ApplicationSet template in a test branch.

Confirm that the validation pipeline rejects the PR.

### Step 14 — Test inventory removal safely

Do not remove a production service to test deletion behavior.

In DEV, use a disposable test ApplicationSet and test service.

Verify the configured ApplicationSet deletion policy before removing the inventory entry.

## 191.35. Definition of Done

Chapter acceptance criteria

### Repository

- GitOps repository structure created.
- Service inventory created.
- All 21 service entries documented.
- Environment configuration created.
- Helm values organized by environment.
- ApplicationSet definitions committed.

### Generators

- List generator tested.
- Git directory generator tested.
- Git file generator tested.
- Matrix generator tested.
- Cluster generator tested.
- Merge generator understood and validated if used.

### ArgoCD

- AppProject configured.
- Root Application configured.
- ApplicationSets deployed.
- Generated Applications verified.
- Repository access verified.
- Destination clusters verified.
- Deletion policy reviewed.

### CI/CD

- GitHub Actions validation configured.
- Inventory validation passes.
- Duplicate service check passes.
- YAML and JSON validation passes.
- ApplicationSet schema validation passes.
- PR protection enabled.
- Production reviewers configured.

### Release management

- DEV deployment tested.
- QA deployment tested.
- STAGING deployment tested.
- PRODUCTION approval path documented.
- Immutable image promotion documented.
- Rollback process documented.
- Application lifecycle process documented.

### Operations

- Troubleshooting commands documented.
- Controller logs accessible.
- Runbooks created.
- Ownership established.
- Security review completed.

## 191.36. Final architecture

```
                  21 APPLICATION REPOSITORIES
                             |
                             |
                             v
                    CI/CD PIPELINES
                             |
                  +----------+----------+
                  |                     |
                  v                     v
             SECURITY SCAN         DOCKER BUILD
                                        |
                                        v
                                  AMAZON ECR
                                        |
                                        v
                                IMAGE DIGEST
                                        |
                                        v
                         GITOPS RELEASE PROMOTION
                                        |
                                        v
                           SERVICE INVENTORY
                                        |
                                        v
                              APPLICATIONSET
                                        |
                   +--------------------+--------------------+
                   |                    |                    |
                   v                    v                    v
              GIT FILE              MATRIX              CLUSTER
              GENERATOR            GENERATOR            GENERATOR
                   |                    |                    |
                   +--------------------+--------------------+
                                        |
                                        v
                              GENERATED APPLICATIONS
                                        |
                                        v
                                     ARGOCD
                                        |
                    +-------------------+-------------------+
                    |                   |                   |
                    v                   v                   v
                   DEV                  QA                STAGING
                    |                   |                   |
                    +-------------------+-------------------+
                                        |
                                        v
                               PRODUCTION APPROVAL
                                        |
                                        v
                                   PRODUCTION
                                        |
                                        v
                               HEALTH VALIDATION
                                        |
                                        v
                                RELEASE COMPLETE
```

## 191.37. Final takeaway

A GitOps generator helps us manage many microservices without maintaining repetitive Application YAML files.

For this e-commerce platform:

- Service inventory defines what applications exist.
- ApplicationSet generates ArgoCD Applications.
- Helm defines how applications are deployed.
- Environment values define environment-specific configuration.
- ECR stores immutable application images.
- GitOps pull requests control promotion.
- ArgoCD reconciles the desired state.
- Platform Engineering governs the deployment standards.

The important distinction is that automation should reduce repetitive work without removing deployment controls. Generating an Application is not approval to deploy a release to production.

## 191.38. Connection to the next chapter

| Chapter                                 | Platform capability                                     |
| --------------------------------------- | ------------------------------------------------------- |
| 182 — Reusable Helm Templates           | Reusable Kubernetes packaging                           |
| 183 — Reusable GitHub Actions           | Reusable CI actions                                     |
| 184 — Terraform Modules                 | Reusable infrastructure                                 |
| 185 — Reusable GitHub Actions Workflows | Shared pipeline workflows                               |
| 186 — Security Policy Templates         | Standardized security rules                             |
| 187 — Monitoring Templates              | Standardized monitoring                                 |
| 188 — Alert Rule Templates              | Standardized alerts                                     |
| 189 — NetworkPolicy Templates           | Workload network isolation                              |
| 190 — ServiceAccount Templates          | Workload identity and permissions                       |
| 191 — GitOps Generators                 | Automated Application generation                        |
| 192 — Microservice Golden Path          | Complete developer onboarding and deployment experience |

Next chapter — Chapter 192: Microservice Golden Path

We will combine the reusable Helm charts, GitHub Actions, Terraform modules, security policies, monitoring templates, NetworkPolicies, ServiceAccounts and GitOps generators into one standardized developer workflow.

The goal is that a developer can create a new microservice using an approved template, submit a pull request, build and scan the image, publish it to ECR, and deploy it through ArgoCD without manually configuring every platform component.