# 72 — Multi-Environment ApplicationSets: DEV → QA → STAGING → PROD

Now we will take the **one service we already understand — `order-service`** and deploy it through four environments:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

The goal is to understand **exactly what changes and what does not change** during promotion.

---

# 1. First Understand the Main Idea

We have one application:

```text
order-service
```

We have four environments:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

We do **not** create four different application repositories.

We have:

```text
order-service GitHub repository
```

and four sets of environment configuration in GitOps.

```text
order-service
       |
       +---- DEV configuration
       |
       +---- QA configuration
       |
       +---- STAGING configuration
       |
       +---- PROD configuration
```

---

# 2. GitOps Repository Structure

Our GitOps repository becomes:

```text
gitops/
│
├── environments/
│
│   ├── dev/
│   │   └── order-service/
│   │       ├── app.yaml
│   │       └── values.yaml
│   │
│   ├── qa/
│   │   └── order-service/
│   │       ├── app.yaml
│   │       └── values.yaml
│   │
│   ├── staging/
│   │   └── order-service/
│   │       ├── app.yaml
│   │       └── values.yaml
│   │
│   └── prod/
│       └── order-service/
│           ├── app.yaml
│           └── values.yaml
│
└── argocd/
    ├── projects/
    └── appsets/
```

This is the key structure for this chapter.

---

# 3. Why Four `values.yaml` Files?

Because environments have different requirements.

For example:

### DEV

```yaml
replicaCount: 2
```

### QA

```yaml
replicaCount: 2
```

### STAGING

```yaml
replicaCount: 3
```

### PROD

```yaml
replicaCount: 3
```

Resources can also differ.

DEV:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

PROD:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 768Mi
```

Same application.

Different environment requirements.

---

# 4. What Should NOT Change?

The application image should normally be the **same immutable artifact** as it moves through environments.

For example:

```text
order-service:8f31a2c
```

DEV:

```text
order-service:8f31a2c
```

QA:

```text
order-service:8f31a2c
```

STAGING:

```text
order-service:8f31a2c
```

PROD:

```text
order-service:8f31a2c
```

This is:

> **Build once, promote the same artifact.**

---

# 5. DEV Configuration

Create:

```text
environments/dev/order-service/app.yaml
```

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: main

enabled: true

team: backend-team
```

And:

```text
environments/dev/order-service/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1
```

---

# 6. QA Configuration

Create:

```text
environments/qa/order-service/app.yaml
```

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: v1.0.0

enabled: true

team: backend-team
```

Notice something important.

DEV:

```yaml
chartRevision: main
```

QA:

```yaml
chartRevision: v1.0.0
```

We are beginning to introduce **release control**.

QA should ideally consume a tested/released chart version rather than blindly following application `main`.

---

# 7. QA Values

```text
environments/qa/order-service/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"

resources:
  requests:
    cpu: 300m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1
```

Notice:

```text
Image = same
Resources = different
```

---

# 8. STAGING

Create:

```text
environments/staging/order-service/app.yaml
```

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: v1.0.0

enabled: true

team: backend-team
```

Values:

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"

resources:
  requests:
    cpu: 500m
    memory: 768Mi

  limits:
    cpu: 1500m
    memory: 1536Mi

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 65

pdb:
  enabled: true
  minAvailable: 2
```

---

# 9. PROD

Create:

```text
environments/prod/order-service/app.yaml
```

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: v1.0.0

enabled: true

team: backend-team
```

Values:

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"

resources:
  requests:
    cpu: 500m
    memory: 768Mi

  limits:
    cpu: 1500m
    memory: 1536Mi

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 65

pdb:
  enabled: true
  minAvailable: 2
```

---

# 10. Compare the Four Environments

| Setting        |       DEV |        QA |   STAGING |      PROD |
| -------------- | --------: | --------: | --------: | --------: |
| Replicas       |         2 |         2 |         3 |         3 |
| HPA Max        |         5 |         8 |        10 |        15 |
| CPU Request    |      250m |      300m |      500m |      500m |
| Memory Request |     512Mi |     512Mi |     768Mi |     768Mi |
| Image          | `8f31a2c` | `8f31a2c` | `8f31a2c` | `8f31a2c` |
| Chart          |    `main` |  `v1.0.0` |  `v1.0.0` |  `v1.0.0` |

The exact values are examples; in a real environment they come from capacity testing and operational requirements.

---

# 11. Now Create Four ApplicationSets

We already created:

```text
ecommerce-dev
```

Now create:

```text
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

The four ApplicationSets perform the same basic job.

The major differences are:

```text
Environment
Namespace
AppProject
Sync policy
```

---

# 12. QA ApplicationSet

Create:

```text
argocd/appsets/ecommerce-qa.yaml
```

```yaml
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
        repoURL: https://github.com/company-ecommerce/gitops.git
        revision: main
        files:
          - path: environments/qa/*/app.yaml

  template:
    metadata:
      name: '{{.service}}-qa'

      labels:
        company-ecommerce.io/environment: qa
        company-ecommerce.io/service: '{{.service}}'
        company-ecommerce.io/team: '{{.team}}'

    spec:
      project: qa

      sources:
        - repoURL: '{{.repository}}'
          targetRevision: '{{.chartRevision}}'
          path: '{{.chartPath}}'

          helm:
            valueFiles:
              - '$values/environments/qa/{{.service}}/values.yaml'

        - repoURL: https://github.com/company-ecommerce/gitops.git
          targetRevision: main
          ref: values

      destination:
        server: https://kubernetes.default.svc
        namespace: ecommerce-qa

      syncPolicy:
        automated:
          prune: true
          selfHeal: true

        syncOptions:
          - CreateNamespace=true
          - PruneLast=true

        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

---

# 13. What Will This Generate?

Our file:

```text
environments/qa/order-service/app.yaml
```

contains:

```yaml
service: order-service
```

The ApplicationSet template contains:

```yaml
name: '{{.service}}-qa'
```

Therefore:

```text
order-service-qa
```

is generated.

So now we have:

```text
order-service-dev
order-service-qa
```

---

# 14. STAGING ApplicationSet

Create:

```text
argocd/appsets/ecommerce-staging.yaml
```

```yaml
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
        repoURL: https://github.com/company-ecommerce/gitops.git
        revision: main
        files:
          - path: environments/staging/*/app.yaml

  template:
    metadata:
      name: '{{.service}}-staging'

      labels:
        company-ecommerce.io/environment: staging
        company-ecommerce.io/service: '{{.service}}'
        company-ecommerce.io/team: '{{.team}}'

    spec:
      project: staging

      sources:
        - repoURL: '{{.repository}}'
          targetRevision: '{{.chartRevision}}'
          path: '{{.chartPath}}'

          helm:
            valueFiles:
              - '$values/environments/staging/{{.service}}/values.yaml'

        - repoURL: https://github.com/company-ecommerce/gitops.git
          targetRevision: main
          ref: values

      destination:
        server: https://kubernetes.default.svc
        namespace: ecommerce-staging

      syncPolicy:
        automated:
          prune: true
          selfHeal: true

        syncOptions:
          - CreateNamespace=true
          - PruneLast=true

        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

---

# 15. PROD ApplicationSet

Production is different.

Create:

```text
argocd/appsets/ecommerce-prod.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-prod
  namespace: argocd

spec:
  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    - git:
        repoURL: https://github.com/company-ecommerce/gitops.git
        revision: main
        files:
          - path: environments/prod/*/app.yaml

  template:
    metadata:
      name: '{{.service}}-prod'

      labels:
        company-ecommerce.io/environment: prod
        company-ecommerce.io/service: '{{.service}}'
        company-ecommerce.io/team: '{{.team}}'

    spec:
      project: production

      sources:
        - repoURL: '{{.repository}}'
          targetRevision: '{{.chartRevision}}'
          path: '{{.chartPath}}'

          helm:
            valueFiles:
              - '$values/environments/prod/{{.service}}/values.yaml'

        - repoURL: https://github.com/company-ecommerce/gitops.git
          targetRevision: main
          ref: values

      destination:
        server: https://kubernetes.default.svc
        namespace: ecommerce-prod

      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true

        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

Notice:

```yaml
automated:
```

is intentionally absent.

---

# 16. Why No Automatic Production Sync?

Imagine this happens:

```text
Developer
   ↓
GitOps PR
   ↓
PR merged
   ↓
ArgoCD sees change
```

In DEV:

```text
ArgoCD
   ↓
automatic deployment
```

In PROD:

```text
ArgoCD
   ↓
Application becomes OutOfSync
   ↓
Production approval
   ↓
Manual sync
```

This provides an explicit production deployment control point.

---

# 17. AppProjects

We need four AppProjects:

```text
development
qa
staging
production
```

You already saw the concept in the previous chapter.

The important relationship is:

```text
ApplicationSet
      ↓
Application
      ↓
AppProject
      ↓
Allowed repository + namespace
```

For example:

```text
order-service-prod
        ↓
production AppProject
        ↓
ecommerce-prod
```

---

# 18. Apply the ApplicationSets

First check:

```bash
kubectl get applicationsets -n argocd
```

Then:

```bash
kubectl apply -f argocd/appsets/ecommerce-qa.yaml
```

```bash
kubectl apply -f argocd/appsets/ecommerce-staging.yaml
```

```bash
kubectl apply -f argocd/appsets/ecommerce-prod.yaml
```

Check:

```bash
kubectl get applicationsets -n argocd
```

Expected:

```text
NAME
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

---

# 19. Check Generated Applications

Run:

```bash
kubectl get applications -n argocd
```

You should eventually see:

```text
order-service-dev
order-service-qa
order-service-staging
order-service-prod
```

This is our first complete multi-environment deployment model.

---

# 20. Understand the Four Applications

They are separate ArgoCD Applications:

```text
order-service-dev
        ↓
ecommerce-dev
```

```text
order-service-qa
        ↓
ecommerce-qa
```

```text
order-service-staging
        ↓
ecommerce-staging
```

```text
order-service-prod
        ↓
ecommerce-prod
```

But all four use the same application repository:

```text
company-ecommerce/order-service
```

---

# 21. The Image Is Promoted

Suppose CI produces:

```text
order-service:8f31a2c
```

DEV:

```yaml
image:
  tag: "8f31a2c"
```

We test it.

Then promote:

```text
DEV
 ↓
QA
```

QA:

```yaml
image:
  tag: "8f31a2c"
```

Then:

```text
QA
 ↓
STAGING
```

STAGING:

```yaml
image:
  tag: "8f31a2c"
```

Then:

```text
STAGING
 ↓
PROD
```

PROD:

```yaml
image:
  tag: "8f31a2c"
```

---

# 22. What Actually Changes During Promotion?

Usually this:

```text
QA values.yaml
```

gets the tested image:

```yaml
image:
  tag: "8f31a2c"
```

Then:

```text
STAGING values.yaml
```

gets:

```yaml
image:
  tag: "8f31a2c"
```

Then:

```text
PROD values.yaml
```

gets:

```yaml
image:
  tag: "8f31a2c"
```

The application image itself isn't rebuilt.

---

# 23. Example Promotion

Suppose DEV currently runs:

```text
order-service:7a21c9e
```

CI builds:

```text
order-service:8f31a2c
```

DEV GitOps changes:

```diff
- tag: "7a21c9e"
+ tag: "8f31a2c"
```

ArgoCD deploys it.

Testing happens.

After successful testing, QA GitOps changes:

```diff
- tag: "7a21c9e"
+ tag: "8f31a2c"
```

Then staging:

```diff
- tag: "7a21c9e"
+ tag: "8f31a2c"
```

Then production:

```diff
- tag: "7a21c9e"
+ tag: "8f31a2c"
```

---

# 24. This Is a Very Important Production Principle

We don't want:

```text
DEV
  ↓
Build
  ↓
QA
  ↓
Build again
  ↓
STAGING
  ↓
Build again
  ↓
PROD
```

Instead:

```text
SOURCE
  ↓
BUILD ONCE
  ↓
IMAGE 8f31a2c
  ↓
SCAN
  ↓
TEST
  ↓
DEV
  ↓
QA
  ↓
STAGING
  ↓
PROD
```

The artifact is immutable.

---

# 25. What If Production Fails?

Suppose:

```text
PROD
```

is running:

```text
8f31a2c
```

and we discover a problem.

Previous version:

```text
7a21c9e
```

GitOps currently:

```yaml
image:
  tag: "8f31a2c"
```

Rollback:

```diff
- tag: "8f31a2c"
+ tag: "7a21c9e"
```

Create a Git revert/rollback PR.

After approval:

```text
GitOps
   ↓
ArgoCD
   ↓
Production
   ↓
7a21c9e
```

No rebuilding required.

---

# 26. Why Git Revert Is Powerful

Our deployment history is stored in Git.

For example:

```text
Git history:

A1 → version 7a21c9e
A2 → version 8f31a2c
A3 → production promotion
```

If A3 caused the problem:

```text
git revert A3
```

Now Git returns to the previous desired state.

ArgoCD sees:

```text
Git changed
```

and reconciles Kubernetes.

---

# 27. What About Environment-Specific Configuration?

Not everything should be identical.

For example:

### DEV

```yaml
replicaCount: 2
```

### PROD

```yaml
replicaCount: 3
```

Database endpoints are different:

```text
DEV
   ↓
dev RDS

PROD
   ↓
production RDS
```

Secrets are different:

```text
DEV
   ↓
DEV Secrets Manager

PROD
   ↓
PROD Secrets Manager
```

But those secrets should **not** be stored directly inside:

```text
values.yaml
```

Instead:

```text
AWS Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Pod
```

which we already covered in the Secrets Management chapter.

---

# 28. Environment Architecture

Our eventual architecture is:

```text
                         GitOps
                           |
          +----------------+----------------+
          |                |                |
         DEV              QA             STAGING
          |                |                |
     ecommerce-dev    ecommerce-qa   ecommerce-staging
          |                |                |
          +----------------+----------------+
                           |
                          PROD
                           |
                    ecommerce-prod
```

Each environment has:

```text
ApplicationSet
AppProject
values
Applications
```

---

# 29. Important Production Rule

Do not use production configuration like:

```yaml
image:
  tag: latest
```

Never make production depend on a moving tag.

Prefer:

```yaml
image:
  tag: "8f31a2c"
```

or, even stronger:

```yaml
image:
  digest: "sha256:..."
```

We will implement digest-based deployment in a later chapter.

---

# 30. What About the 21 Services?

Once this works for:

```text
order-service
```

the same structure becomes:

```text
environments/
├── dev/
│   ├── auth-service/
│   ├── user-service/
│   ├── customer-service/
│   ├── ...
│   └── reporting-service/
│
├── qa/
│   ├── auth-service/
│   ├── user-service/
│   ├── ...
│   └── reporting-service/
│
├── staging/
│   ├── auth-service/
│   ├── user-service/
│   ├── ...
│   └── reporting-service/
│
└── prod/
    ├── auth-service/
    ├── user-service/
    ├── ...
    └── reporting-service/
```

Then:

```text
21 × 4 = 84
```

Applications are generated automatically.

---

# 31. The Complete Flow

Now you should be able to visualize the entire system:

```text
Developer
    |
    v
order-service GitHub
    |
    v
GitHub Actions
    |
    +-- Maven Test
    +-- SonarQube
    +-- Dependency Scan
    +-- Trivy
    |
    v
ECR
    |
    | image: 8f31a2c
    v
GitOps Repository
    |
    +---- environments/dev
    |
    +---- environments/qa
    |
    +---- environments/staging
    |
    +---- environments/prod
    |
    v
ApplicationSet
    |
    +---- order-service-dev
    +---- order-service-qa
    +---- order-service-staging
    +---- order-service-prod
    |
    v
ArgoCD
    |
    +---- DEV
    +---- QA
    +---- STAGING
    +---- PROD
    |
    v
EKS
```

---

# 32. One Thing We Need to Fix Before Calling This Production-Ready

There is one subtle issue with our current design.

We currently use:

```text
DEV → chartRevision: main
QA → chartRevision: v1.0.0
STAGING → chartRevision: v1.0.0
PROD → chartRevision: v1.0.0
```

while the Docker image is separately versioned.

That means we have **two release dimensions**:

```text
Application image version
+
Helm chart version
```

For example:

```text
Image:
8f31a2c

Helm chart:
v1.0.0
```

That can be perfectly valid, but we need a deliberate release strategy.

Otherwise you can accidentally promote:

```text
Image A
+
Chart B
```

when you intended:

```text
Image A
+
Chart A
```

We will solve this properly in the next stage.

---

# 33. Chapter 72 Summary

You should now understand:

```text
ApplicationSet
    ↓
Creates multiple Applications

Four ApplicationSets
    ↓
DEV / QA / STAGING / PROD

Each environment
    ↓
Has its own values.yaml

Same image
    ↓
Promoted through environments

GitOps
    ↓
Controls desired state

ArgoCD
    ↓
Reconciles desired state

EKS
    ↓
Runs the application
```

The most important flow to remember is:

```text
BUILD ONCE
    ↓
ECR
    ↓
DEV
    ↓
TEST
    ↓
QA
    ↓
TEST
    ↓
STAGING
    ↓
APPROVAL
    ↓
PROD
```

### Next chapter: `73-Helm-Chart-Versioning-and-Release-Strategy.md`

We will solve the **image version + Helm chart version** problem properly and establish a production release model such as:

```text
Git commit
    ↓
Application version
    ↓
Docker image
    ↓
Helm chart version
    ↓
ECR
    ↓
GitOps promotion
    ↓
ArgoCD
    ↓
EKS
```

Then we can move into **immutable image digests, release tags, rollback, and complete traceability from Git commit → production Pod**.
