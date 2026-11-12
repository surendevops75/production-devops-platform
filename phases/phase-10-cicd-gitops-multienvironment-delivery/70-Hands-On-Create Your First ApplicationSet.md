# 70 — Hands-On: Create Your First ApplicationSet for `order-service`

Now we will **actually implement** what we discussed.

For now, forget all 21 microservices.

We will deploy **only `order-service` to DEV**.

Once this works, we will expand the exact same pattern to the remaining 20 services.

---

# 1. What Are We Building?

Our target is:

```text
GitHub
   |
   | order-service repository
   |
   v
Helm Chart
   |
   |
GitOps Repository
   |
   v
ApplicationSet
   |
   v
order-service-dev
   |
   v
ArgoCD
   |
   v
EKS
   |
   v
ecommerce-dev namespace
   |
   v
order-service Pod
```

The important part is:

```text
ApplicationSet
      ↓
automatically creates
      ↓
order-service-dev
```

---

# 2. Current Situation

From the previous chapters, we already have:

```text
order-service repository
        +
Helm chart
        +
ECR
        +
EKS
        +
ArgoCD
        +
GitOps repository
```

Our GitOps repository should currently look approximately like:

```text
gitops/
├── applications/
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
├── scripts/
└── README.md
```

---

# 3. First Create the DEV Directory

From your GitOps repository:

```bash
cd gitops
```

Create:

```bash
mkdir -p environments/dev/order-service
```

Now:

```text
gitops/
└── environments/
    └── dev/
        └── order-service/
```

---

# 4. Create `app.yaml`

Create:

```text
environments/dev/order-service/app.yaml
```

Put:

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: main

enabled: true

team: backend-team
```

Let's understand this **line by line**.

---

## `service`

```yaml
service: order-service
```

This is simply the service name.

ApplicationSet will eventually create:

```text
order-service-dev
```

---

## `repository`

```yaml
repository: https://github.com/company-ecommerce/order-service.git
```

This tells ArgoCD:

> The Helm chart comes from the `order-service` repository.

Remember, this is **not the GitOps repository**.

We have two repositories involved:

```text
order-service repo
       ↓
Helm chart

GitOps repo
       ↓
Environment configuration
```

---

## `chartPath`

```yaml
chartPath: helm/order-service
```

Inside the application repository:

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
helm/order-service
```

is the Helm chart location.

---

## `chartRevision`

```yaml
chartRevision: main
```

This tells ArgoCD:

> Use the `main` branch of the order-service repository.

For our first DEV implementation, that's fine.

Later we'll change production to something like:

```yaml
chartRevision: v1.4.0
```

or an immutable commit SHA.

---

## `enabled`

```yaml
enabled: true
```

This tells our configuration that the service should be deployed.

We will use this more effectively later when we build the complete service catalog.

---

## `team`

```yaml
team: backend-team
```

This is metadata.

It can later be used for:

```text
ownership
labels
RBAC
reporting
notifications
```

---

# 5. Create `values.yaml`

Now create:

```text
environments/dev/order-service/values.yaml
```

Example:

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  pullPolicy: IfNotPresent

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

### Important

Replace:

```text
123456789012
```

with your AWS account ID.

And replace:

```text
8f31a2c
```

with the image tag that actually exists in your ECR repository.

Check ECR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

---

# 6. Your GitOps Directory Now Looks Like

```text
gitops/
│
├── environments/
│   └── dev/
│       └── order-service/
│           ├── app.yaml
│           └── values.yaml
│
└── argocd/
    └── appsets/
```

This is important.

We are separating:

```text
app.yaml
    ↓
Application metadata

values.yaml
    ↓
Helm environment configuration
```

---

# 7. Create the ApplicationSet

Now create:

```text
argocd/appsets/ecommerce-dev.yaml
```

Use:

```yaml
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

        repoURL: https://github.com/company-ecommerce/gitops.git

        revision: main

        files:

          - path: environments/dev/*/app.yaml

  template:

    metadata:

      name: '{{.service}}-dev'

      labels:

        company-ecommerce.io/environment: dev

        company-ecommerce.io/service: '{{.service}}'

        company-ecommerce.io/team: '{{.team}}'

    spec:

      project: development

      sources:

        - repoURL: '{{.repository}}'

          targetRevision: '{{.chartRevision}}'

          path: '{{.chartPath}}'

          helm:

            valueFiles:

              - '$values/environments/dev/{{.service}}/values.yaml'

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

          - PruneLast=true

        retry:

          limit: 5

          backoff:

            duration: 5s

            factor: 2

            maxDuration: 3m
```

Now let's **not move on until you understand what this means**.

---

# 8. Understand the ApplicationSet YAML

The first part:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
```

says:

> This Kubernetes object is an ArgoCD ApplicationSet.

---

## Name

```yaml
metadata:
  name: ecommerce-dev
  namespace: argocd
```

We're creating:

```text
ApplicationSet
     |
     └── ecommerce-dev
```

It lives inside:

```text
argocd
```

namespace.

---

# 9. The Most Important Part — Generator

Look at:

```yaml
generators:

  - git:

      repoURL: https://github.com/company-ecommerce/gitops.git

      revision: main

      files:

        - path: environments/dev/*/app.yaml
```

This means:

> Go to the GitOps repository and look for `app.yaml` files underneath `environments/dev`.

The pattern:

```text
environments/dev/*/app.yaml
```

matches:

```text
environments/dev/order-service/app.yaml
```

Later it will also match:

```text
environments/dev/user-service/app.yaml
environments/dev/payment-service/app.yaml
environments/dev/cart-service/app.yaml
```

etc.

---

# 10. What Does `*` Mean?

This:

```text
dev/*/app.yaml
```

means:

```text
dev/order-service/app.yaml
dev/user-service/app.yaml
dev/cart-service/app.yaml
dev/payment-service/app.yaml
```

In other words:

```text
*
```

means:

> Any service directory.

That's how one ApplicationSet can discover all our services.

---

# 11. What Does `goTemplate` Do?

We have:

```yaml
goTemplate: true
```

This allows us to use values from `app.yaml`.

For example:

```yaml
service: order-service
```

can be used as:

```text
{{.service}}
```

So:

```yaml
name: '{{.service}}-dev'
```

becomes:

```text
order-service-dev
```

---

# 12. This Is the Magic

Our `app.yaml` says:

```yaml
service: order-service
```

ApplicationSet template says:

```yaml
name: '{{.service}}-dev'
```

Therefore:

```text
{{.service}}
```

becomes:

```text
order-service
```

and the final name becomes:

```text
order-service-dev
```

---

# 13. Another Example

Suppose later we have:

```yaml
service: payment-service
```

The same template:

```yaml
name: '{{.service}}-dev'
```

becomes:

```text
payment-service-dev
```

We don't need to write another Application YAML.

That's the whole purpose of ApplicationSet.

---

# 14. Understand `sources`

This part is extremely important:

```yaml
sources:

  - repoURL: '{{.repository}}'
    targetRevision: '{{.chartRevision}}'
    path: '{{.chartPath}}'

  - repoURL: https://github.com/company-ecommerce/gitops.git
    targetRevision: main
    ref: values
```

We are telling ArgoCD about **two Git sources**.

### Source 1

Application repository:

```text
order-service
```

contains:

```text
helm/order-service
```

### Source 2

GitOps repository:

```text
gitops
```

contains:

```text
environments/dev/order-service/values.yaml
```

So:

```text
Application repository
        +
GitOps repository
        ↓
Helm deployment
```

---

# 15. Why Two Repositories?

Because we don't want environment configuration mixed into application code.

Application repo:

```text
order-service
```

owns:

```text
Java
Dockerfile
Helm chart
```

GitOps repo owns:

```text
DEV configuration
QA configuration
STAGING configuration
PROD configuration
```

This is GitOps separation.

---

# 16. The `$values` Part

This line may look confusing:

```yaml
valueFiles:
  - '$values/environments/dev/{{.service}}/values.yaml'
```

We previously gave the GitOps source:

```yaml
ref: values
```

Therefore:

```text
$values
```

means:

> Start from the Git repository that has the reference name `values`.

So:

```text
$values/environments/dev/order-service/values.yaml
```

means:

```text
GitOps repository
    ↓
environments
    ↓
dev
    ↓
order-service
    ↓
values.yaml
```

---

# 17. What Does ArgoCD Finally Build?

ArgoCD effectively combines:

```text
order-service repo
        |
        └── helm/order-service
                  +
GitOps repo
        |
        └── environments/dev/order-service/values.yaml
```

into:

```text
Kubernetes manifests
```

Conceptually:

```text
Helm Chart
    +
DEV values
    ↓
Deployment
Service
HPA
PDB
ConfigMap
...
```

---

# 18. Destination

We have:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: ecommerce-dev
```

This means:

> Deploy this application into the Kubernetes cluster where ArgoCD is running, in the `ecommerce-dev` namespace.

Therefore:

```text
ArgoCD
   ↓
EKS
   ↓
ecommerce-dev
```

---

# 19. Why `CreateNamespace=true`?

We have:

```yaml
syncOptions:
  - CreateNamespace=true
```

This tells ArgoCD:

> If `ecommerce-dev` doesn't exist, create it.

Therefore we don't necessarily need to manually run:

```bash
kubectl create namespace ecommerce-dev
```

ArgoCD can create it.

---

# 20. Automated Sync

We have:

```yaml
automated:
  prune: true
  selfHeal: true
```

### `selfHeal`

Suppose Git says:

```text
replicas = 2
```

but somebody manually changes Kubernetes:

```bash
kubectl scale deployment order-service --replicas=5
```

Now Kubernetes says:

```text
5 replicas
```

but Git says:

```text
2 replicas
```

ArgoCD detects the difference.

Because:

```yaml
selfHeal: true
```

ArgoCD will reconcile it back toward Git's desired state.

---

# 21. `prune`

Suppose Git says:

```text
Deployment
Service
ConfigMap
```

and later you remove a resource from the Helm chart.

ArgoCD can remove the resource from Kubernetes when pruning is enabled.

For DEV this is useful.

For production, we use much more controlled deletion practices.

---

# 22. Now Commit Everything

Your GitOps repository should contain:

```text
gitops/
├── environments/
│   └── dev/
│       └── order-service/
│           ├── app.yaml
│           └── values.yaml
│
└── argocd/
    └── appsets/
        └── ecommerce-dev.yaml
```

Run:

```bash
git status
```

Then:

```bash
git add .
```

Commit:

```bash
git commit -m "feat: add order service dev applicationset"
```

Push:

```bash
git push origin main
```

---

# 23. Apply ApplicationSet to ArgoCD

Now connect to EKS:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-dev
```

Verify:

```bash
kubectl get nodes
```

Then verify ArgoCD:

```bash
kubectl get pods -n argocd
```

You should see components such as:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-redis
```

---

# 24. Apply the ApplicationSet

Run:

```bash
kubectl apply \
  -f argocd/appsets/ecommerce-dev.yaml
```

Expected:

```text
applicationset.argoproj.io/ecommerce-dev created
```

---

# 25. Check ApplicationSet

```bash
kubectl get applicationset -n argocd
```

You should see:

```text
NAME             AGE
ecommerce-dev    ...
```

---

# 26. Now the Important Test

Run:

```bash
kubectl get applications -n argocd
```

You should see:

```text
NAME                 SYNC STATUS
order-service-dev    ...
```

This is the moment we are looking for.

We manually created only:

```text
ecommerce-dev
```

ApplicationSet automatically created:

```text
order-service-dev
```

---

# 27. Check the Application

```bash
kubectl get application order-service-dev -n argocd
```

Then:

```bash
kubectl describe application order-service-dev -n argocd
```

You can also use:

```bash
argocd app get order-service-dev
```

---

# 28. What Should Happen?

The flow should now be:

```text
GitOps
  |
  | app.yaml
  ↓
ApplicationSet
  |
  | automatically creates
  ↓
order-service-dev
  |
  ↓
ArgoCD
  |
  | reads Helm chart
  |
  | reads values.yaml
  ↓
Kubernetes manifests
  |
  ↓
ecommerce-dev
  |
  ↓
order-service
  |
  ↓
Pods
```

---

# 29. Check Kubernetes

```bash
kubectl get all -n ecommerce-dev
```

You should eventually see:

```text
NAME                                 READY
pod/order-service-xxxxxxxxxx-xxxxx   1/1
pod/order-service-xxxxxxxxxx-yyyyy   1/1

NAME                    TYPE
service/order-service   ClusterIP

NAME                            READY
deployment.apps/order-service   2/2

NAME                                       REFERENCE
horizontalpodautoscaler/order-service      Deployment/order-service
```

The exact pod names will differ.

---

# 30. Check ArgoCD

Run:

```bash
argocd app get order-service-dev
```

You want to eventually see something conceptually like:

```text
Name:               order-service-dev
Project:            development
Server:             https://kubernetes.default.svc
Namespace:          ecommerce-dev

Sync Status:        Synced
Health Status:      Healthy
```

These two are different.

### Synced

Means:

```text
Git desired state
        =
Kubernetes desired/live state
```

### Healthy

Means:

```text
Kubernetes resources
        =
working normally
```

You want:

```text
Synced + Healthy
```

---

# 31. Very Important Test — Add Another Service

Don't immediately add all 20 services.

Let's prove ApplicationSet works.

Create:

```text
environments/dev/user-service/
```

with:

```text
app.yaml
values.yaml
```

Example:

```yaml
service: user-service

repository: https://github.com/company-ecommerce/user-service.git

chartPath: helm/user-service

chartRevision: main

enabled: true

team: backend-team
```

Commit:

```bash
git add .
git commit -m "feat: add user service dev configuration"
git push origin main
```

ApplicationSet detects:

```text
environments/dev/user-service/app.yaml
```

and automatically generates:

```text
user-service-dev
```

Now:

```bash
kubectl get applications -n argocd
```

should show:

```text
order-service-dev
user-service-dev
```

That proves the automation.

---

# 32. This Is the Key Difference

Before ApplicationSet:

```text
You create:

order-service-dev.yaml
order-service-qa.yaml
order-service-prod.yaml
user-service-dev.yaml
user-service-qa.yaml
...
```

After ApplicationSet:

```text
You create:

app.yaml
app.yaml
app.yaml
...
```

and ApplicationSet generates the ArgoCD Applications.

---

# 33. What Happens When CI Updates the Image?

Suppose developer changes code:

```text
Git commit
   ↓
GitHub Actions
   ↓
Docker build
   ↓
Trivy
   ↓
ECR
```

Image:

```text
order-service:abc1234
```

CI updates:

```text
environments/dev/order-service/values.yaml
```

from:

```yaml
image:
  tag: "8f31a2c"
```

to:

```yaml
image:
  tag: "abc1234"
```

GitOps PR is merged.

Then:

```text
GitOps change
      ↓
ArgoCD sees difference
      ↓
order-service-dev becomes OutOfSync
      ↓
ArgoCD sync
      ↓
Deployment updated
      ↓
Kubernetes rolling update
      ↓
New Pods
      ↓
order-service:abc1234
```

ApplicationSet doesn't need to change.

It continues managing:

```text
order-service-dev
```

---

# 34. What ApplicationSet Is NOT Doing

This is very important.

ApplicationSet does **not**:

```text
❌ build Java
❌ run Maven
❌ build Docker
❌ run Trivy
❌ push Docker to ECR
❌ create ECR images
❌ replace GitHub Actions
❌ directly build your application
```

Its job is primarily:

```text
Git configuration
       ↓
Generate/manage ArgoCD Applications
```

---

# 35. Final Mental Model

If you remember only this, remember:

```text
                    CI
                     |
                     v
                   ECR
                     |
                     v
                  GitOps
                     |
                     v
              ApplicationSet
                     |
            +--------+--------+
            |                 |
            v                 v
    order-service-dev   user-service-dev
            |                 |
            v                 v
          ArgoCD            ArgoCD
            |                 |
            +--------+--------+
                     |
                     v
                    EKS
```

And the responsibilities:

```text
GitHub Actions
    = BUILD

ECR
    = STORE IMAGE

GitOps
    = DECLARE WHAT VERSION SHOULD RUN

ApplicationSet
    = CREATE MANY ARGOCD APPLICATIONS

ArgoCD
    = DEPLOY / RECONCILE

EKS
    = RUN
```

### Our practical checkpoint

Before moving to the next chapter, the implementation target is simply:

```text
✅ ecommerce-dev ApplicationSet
        ↓
✅ order-service-dev Application generated
        ↓
✅ ArgoCD sees it
        ↓
✅ order-service deployed to ecommerce-dev
        ↓
✅ Synced
        ↓
✅ Healthy
```

Once this works, adding the remaining 20 services becomes a **repeatable pattern**, rather than trying to understand 84 Applications at once.
