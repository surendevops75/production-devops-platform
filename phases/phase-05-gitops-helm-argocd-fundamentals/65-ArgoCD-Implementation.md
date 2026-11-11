# 65 — ArgoCD Implementation

## 1. Objective

In the previous chapters we created:

```text
Application Repository
        │
        ▼
GitHub Actions
        │
        ▼
Amazon ECR
        │
        ▼
GitOps Repository
```

Now we implement:

```text
GitOps Repository
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
 order-service
```

The goal is to deploy `order-service` into the DEV EKS environment using **ArgoCD**, with Git as the desired-state source.

---

# 2. What ArgoCD Does

ArgoCD continuously compares:

```text
Git desired state
        VS
Kubernetes actual state
```

For example, Git says:

```yaml
replicaCount: 3
```

but Kubernetes currently has:

```text
2 Pods
```

ArgoCD detects the difference.

Conceptually:

```text
                 Git
                  │
                  │ desired state
                  ▼
               ArgoCD
                  │
           comparison engine
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
       Desired          Live State
        State           in EKS
```

If synchronization is enabled, ArgoCD reconciles the difference.

---

# 3. Why We Use ArgoCD

Without GitOps:

```text
Developer
   │
   ▼
CI
   │
   ▼
kubectl
   │
   ▼
EKS
```

With GitOps:

```text
Developer
   │
   ▼
CI
   │
   ▼
ECR
   │
   ▼
GitOps
   │
   ▼
ArgoCD
   │
   ▼
EKS
```

This gives us:

* Git-based audit history
* controlled promotion
* declarative deployment
* automatic drift detection
* rollback through Git
* separation between CI and CD
* reduced Kubernetes credentials in CI

---

# 4. ArgoCD Architecture

Our target architecture:

```text
                         GitHub
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
      order-service                    gitops
             │                           │
             │ Helm chart                │ values
             │                           │
             └──────────────┬────────────┘
                            │
                            ▼
                         ArgoCD
                            │
                ┌───────────┼───────────┐
                │           │           │
                ▼           ▼           ▼
           API Server   Repo Server   Controller
                │                       │
                │                       ▼
                │                 Kubernetes API
                │                       │
                │                       ▼
                │                      EKS
                │
                ▼
             ArgoCD UI
```

---

# 5. ArgoCD Components

The important components are:

## API Server

Handles:

```text
CLI
UI
API
authentication
authorization
```

---

## Repository Server

Responsible for obtaining and rendering application sources.

For our application:

```text
order-service Git repo
        +
gitops Git repo
        ↓
Helm rendering
```

---

## Application Controller

This is the important reconciliation component.

It compares:

```text
desired state
vs
live Kubernetes state
```

and performs synchronization when configured to do so.

---

## Redis

ArgoCD uses Redis internally for caching and application-related operations.

It is an ArgoCD platform component, not our application Redis/ElastiCache.

Do not confuse:

```text
ArgoCD Redis
```

with:

```text
Application Redis / ElastiCache
```

---

# 6. Where Should ArgoCD Run?

ArgoCD runs inside the EKS cluster.

Example:

```text
EKS
│
├── kube-system
│
├── argocd
│   ├── argocd-server
│   ├── argocd-repo-server
│   ├── argocd-application-controller
│   ├── argocd-redis
│   └── ...
│
├── ecommerce-dev
│   └── order-service
│
└── monitoring
```

Create a dedicated namespace:

```text
argocd
```

Do not install ArgoCD into:

```text
default
```

---

# 7. Prerequisites

Before installing ArgoCD, verify:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-dev
```

Verify:

```bash
kubectl config current-context
```

Expected context will correspond to the EKS cluster.

Check nodes:

```bash
kubectl get nodes
```

Example:

```text
NAME                          STATUS   ROLES    AGE
ip-10-0-11-25.ec2.internal    Ready    <none>   20m
ip-10-0-12-37.ec2.internal    Ready    <none>   20m
ip-10-0-13-41.ec2.internal    Ready    <none>   20m
```

Check namespaces:

```bash
kubectl get namespaces
```

---

# 8. Install ArgoCD Namespace

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get namespace argocd
```

Expected:

```text
NAME      STATUS   AGE
argocd    Active   ...
```

---

# 9. Install ArgoCD

For the initial implementation, install ArgoCD using its Helm chart.

Add the repository:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

Update:

```bash
helm repo update
```

Check:

```bash
helm search repo argo/argo-cd
```

Before production, pin a version that has been tested with your EKS/Kubernetes version rather than allowing an uncontrolled chart upgrade.

---

# 10. Create ArgoCD Values

Create:

```text
platform/
└── argocd/
    └── values-dev.yaml
```

Example baseline:

```yaml
global:
  domain: argocd-dev.example.com

server:
  replicas: 2

repoServer:
  replicas: 2

controller:
  replicas: 1

redis:
  replicas: 1

configs:
  params:
    server.insecure: "true"
```

For production, we will improve this significantly with:

```text
HA
TLS
SSO
RBAC
PDB
resource requests
anti-affinity
secret management
network policies
```

The initial installation is intentionally simple.

---

# 11. Install ArgoCD

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values platform/argocd/values-dev.yaml
```

Check:

```bash
kubectl get pods -n argocd
```

You should eventually see components such as:

```text
argocd-application-controller-...
argocd-repo-server-...
argocd-server-...
argocd-redis-...
```

Check services:

```bash
kubectl get svc -n argocd
```

---

# 12. Check ArgoCD Deployment

```bash
kubectl get deployments -n argocd
```

Check:

```bash
kubectl get statefulsets -n argocd
```

Check:

```bash
kubectl get pods -n argocd -o wide
```

Check events:

```bash
kubectl get events -n argocd --sort-by=.lastTimestamp
```

---

# 13. Troubleshooting Installation

If a Pod is stuck:

```bash
kubectl get pods -n argocd
```

Then:

```bash
kubectl describe pod <pod-name> -n argocd
```

Logs:

```bash
kubectl logs <pod-name> -n argocd
```

For example:

```bash
kubectl logs deploy/argocd-server -n argocd
```

Repository server:

```bash
kubectl logs deploy/argocd-repo-server -n argocd
```

Controller:

```bash
kubectl logs \
  statefulset/argocd-application-controller \
  -n argocd
```

---

# 14. Access ArgoCD Locally First

Before exposing ArgoCD publicly, use port forwarding.

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443
```

Then access:

```text
https://localhost:8080
```

This is useful because:

```text
Internet
   X
   │
   │ no public exposure
   ▼
ArgoCD
```

We can validate the installation without exposing the management UI.

---

# 15. Get Initial Admin Password

For the initial installation:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Store the password temporarily and log in.

Do **not** put it into:

```text
Git
GitHub Actions
Helm values
ConfigMap
README
```

After proper authentication is configured, the initial admin account should be handled according to the organization's ArgoCD security policy.

---

# 16. Install ArgoCD CLI

Verify:

```bash
argocd version --client
```

Then connect through the forwarded port:

```bash
argocd login localhost:8080 \
  --username admin \
  --password '<PASSWORD>' \
  --insecure
```

Check:

```bash
argocd account get-user-info
```

---

# 17. Verify ArgoCD

```bash
argocd version
```

Then:

```bash
argocd cluster list
```

You should see the Kubernetes cluster registered with ArgoCD.

For a single-cluster installation, the in-cluster Kubernetes API is commonly used.

---

# 18. Connect ArgoCD to GitHub

There are two broad ways to access private repositories.

## HTTPS

Using Git credentials/token.

## SSH

Using a deploy key or SSH credential.

For our implementation, repository credentials should be stored as an ArgoCD-managed Secret/credential configuration, not committed to Git.

---

# 19. Why ArgoCD Needs Git Credentials

Our repositories are private:

```text
company-ecommerce/order-service
company-ecommerce/gitops
```

ArgoCD needs to read:

```text
order-service
```

and:

```text
gitops
```

Therefore:

```text
ArgoCD
   │
   ├── read order-service
   │
   └── read gitops
```

It does **not** need:

```text
write application source
write GitOps
merge PRs
```

ArgoCD should normally have **read-only repository access**.

---

# 20. Repository Credential Principle

Use the minimum permission required.

ArgoCD needs:

```text
Git read
```

It does not need:

```text
Git write
```

CI may need:

```text
GitOps write/PR
```

but CI does not need:

```text
Kubernetes production admin
```

This produces:

```text
CI
 │
 ├── ECR write
 └── GitOps PR/write
       X
       │
       └── no direct production kubectl

ArgoCD
 │
 ├── Git read
 └── Kubernetes reconcile
```

---

# 21. Add GitHub Repository

With ArgoCD CLI, repository credentials can be registered.

For HTTPS-based private repository access, use an appropriately scoped GitHub credential.

Conceptually:

```bash
argocd repo add https://github.com/company-ecommerce/order-service.git \
  --username <GITHUB_USERNAME> \
  --password <TOKEN>
```

And:

```bash
argocd repo add https://github.com/company-ecommerce/gitops.git \
  --username <GITHUB_USERNAME> \
  --password <TOKEN>
```

For production, use an organization-approved credential mechanism such as a GitHub App or dedicated repository credential rather than a broad personal token.

Verify:

```bash
argocd repo list
```

The repositories should show as connected.

---

# 22. GitOps Repository Layout

ArgoCD will consume:

```text
gitops/
│
├── environments/
│   └── dev/
│       └── order-service/
│           └── values.yaml
│
└── argocd/
    └── applications/
        └── order-service-dev.yaml
```

The Application definition tells ArgoCD:

```text
Use this Helm chart
+
Use these environment values
+
Deploy to this namespace
```

---

# 23. ArgoCD Application

Our Application definition:

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

This is a **multi-source Application**.

---

# 24. Why Multi-Source?

We intentionally separated:

```text
Application repo
        │
        └── Helm chart

GitOps repo
        │
        └── Environment values
```

ArgoCD combines them:

```text
order-service repo
       │
       │ chart
       ▼
     Helm
       ▲
       │ values
       │
gitops repo
```

Then:

```text
Helm rendered manifests
        │
        ▼
      ArgoCD
        │
        ▼
       EKS
```

---

# 25. Create Application Namespace

Although ArgoCD can create it using:

```yaml
syncOptions:
  - CreateNamespace=true
```

we can understand the target explicitly:

```bash
kubectl create namespace ecommerce-dev
```

Verify:

```bash
kubectl get namespace ecommerce-dev
```

For the first deployment, either approach is fine. The GitOps/Application definition remains the source of desired state.

---

# 26. Create the Application

Once the Application YAML is committed:

```bash
git add argocd/applications/order-service-dev.yaml
git commit -m "feat(argocd): add order-service dev application"
git push
```

Then apply the ArgoCD Application definition:

```bash
kubectl apply \
  -f argocd/applications/order-service-dev.yaml
```

Important distinction:

This `kubectl apply` is being used to **bootstrap ArgoCD's Application object**, not as the application's normal deployment mechanism.

Later, we can manage the Application object itself through an ArgoCD "App of Apps" or ApplicationSet pattern.

---

# 27. Check the Application

```bash
kubectl get applications -n argocd
```

Or:

```bash
argocd app list
```

Expected conceptually:

```text
NAME               STATUS      HEALTH
order-service-dev  Synced      Healthy
```

Initially you may see:

```text
OutOfSync
```

while ArgoCD is calculating/rendering the desired state.

---

# 28. Inspect Application

```bash
argocd app get order-service-dev
```

This gives information about:

```text
Repository
Revision
Path
Destination
Sync status
Health status
Resources
```

---

# 29. Sync Manually First

For the first deployment, I recommend understanding manual synchronization before enabling automated reconciliation.

Run:

```bash
argocd app sync order-service-dev
```

ArgoCD will:

```text
Git
 │
 ▼
Read chart
 │
 ▼
Read values
 │
 ▼
Render Helm
 │
 ▼
Compare resources
 │
 ▼
Apply resources
 │
 ▼
EKS
```

---

# 30. Check Kubernetes

After synchronization:

```bash
kubectl get all -n ecommerce-dev
```

Expected resources include:

```text
deployment.apps/order-service
service/order-service
pod/order-service-...
```

Check:

```bash
kubectl get pods -n ecommerce-dev
```

Expected:

```text
NAME                            READY   STATUS
order-service-xxxxx             1/1     Running
order-service-yyyyy             1/1     Running
```

---

# 31. Check Deployment

```bash
kubectl get deployment order-service \
  -n ecommerce-dev
```

Then:

```bash
kubectl describe deployment order-service \
  -n ecommerce-dev
```

Verify:

```text
Replicas
Image
Resources
Service Account
Probes
Environment
RollingUpdate strategy
```

---

# 32. Check Service

```bash
kubectl get service order-service \
  -n ecommerce-dev
```

Expected:

```text
NAME            TYPE        CLUSTER-IP      PORT
order-service   ClusterIP   10.x.x.x        8080
```

The application is intentionally:

```text
ClusterIP
```

because the external traffic path is:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Kubernetes Service
   │
   ▼
Order Pods
```

---

# 33. Check Helm Rendering

If something is wrong, render the chart yourself.

From the application repository:

```bash
helm template order-service \
  helm/order-service \
  -f <dev-values-file>
```

This helps distinguish:

```text
Helm problem
```

from:

```text
ArgoCD problem
```

and:

```text
Kubernetes problem
```

---

# 34. ArgoCD Health

ArgoCD doesn't only care about whether manifests were applied.

It also tracks resource health.

Conceptually:

```text
Synced
```

means:

```text
Git desired state == Kubernetes configuration
```

while:

```text
Healthy
```

means:

```text
The resources are operating as expected
```

Therefore:

```text
Synced + Healthy
```

is the normal target.

---

# 35. Synced vs Healthy

This distinction is extremely important for interviews and real production troubleshooting.

### Synced

Git:

```text
replicas = 3
```

Kubernetes:

```text
replicas = 3
```

Therefore:

```text
Synced
```

But Pods may be:

```text
CrashLoopBackOff
```

So:

```text
Synced
Unhealthy
```

is possible.

---

### OutOfSync

Git:

```text
replicas = 3
```

Kubernetes:

```text
replicas = 2
```

Then:

```text
OutOfSync
```

---

# 36. Enable Automated Sync

Once manual deployment is understood, enable:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### `selfHeal`

Suppose someone manually changes:

```bash
kubectl scale deployment order-service \
  -n ecommerce-dev \
  --replicas=1
```

But Git says:

```text
replicas=2
```

ArgoCD detects the drift.

With:

```yaml
selfHeal: true
```

ArgoCD reconciles the deployment back to:

```text
2 replicas
```

---

# 37. Why Manual `kubectl` Changes Are Dangerous

Imagine:

```text
Git
replicas = 3
```

Engineer:

```bash
kubectl scale ... --replicas=7
```

Now:

```text
Git          Kubernetes
  3      !=      7
```

That's configuration drift.

GitOps says:

```text
Git is the desired state.
```

Therefore the engineer should normally modify:

```text
Git
```

rather than manually modifying:

```text
cluster
```

---

# 38. Enable Pruning

Suppose Git contains:

```text
deployment
service
configmap
```

and we remove:

```text
configmap
```

from the desired manifests.

Without pruning:

```text
ConfigMap
    │
    └── may remain in cluster
```

With:

```yaml
prune: true
```

ArgoCD can remove resources that are no longer part of the desired application state.

Use pruning carefully, especially for shared resources.

---

# 39. Production Sync Strategy

DEV can use:

```yaml
automated:
  prune: true
  selfHeal: true
```

For production, a more controlled approach can be:

```text
Git PR
   │
   ▼
Review
   │
   ▼
Git merge
   │
   ▼
ArgoCD detects change
   │
   ▼
Production sync approval
   │
   ▼
Deployment
```

Whether production sync is automated or manually approved depends on the organization's release controls.

The important point is:

> **The production desired state still comes from Git.**

---

# 40. ArgoCD UI

After port forwarding:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443
```

Open:

```text
https://localhost:8080
```

You should see:

```text
Applications
```

and:

```text
order-service-dev
```

Clicking the application lets you inspect:

```text
Application
 ├── Deployment
 ├── ReplicaSet
 ├── Pods
 ├── Service
 ├── ConfigMap
 ├── HPA
 └── PDB
```

This resource graph is extremely useful for troubleshooting.

---

# 41. ArgoCD Resource Tree

Conceptually:

```text
order-service-dev
│
├── Deployment
│   │
│   └── ReplicaSet
│       ├── Pod
│       └── Pod
│
├── Service
│
├── ConfigMap
│
├── ServiceAccount
│
├── HPA
│
└── PDB
```

If the deployment is unhealthy, you can follow the dependency chain.

---

# 42. Image Update Test

Suppose ECR contains:

```text
order-service:8f31a2c
```

Update GitOps:

```yaml
image:
  tag: "8f31a2c"
```

Commit:

```bash
git commit -m "deploy(order-service): promote 8f31a2c to dev"
```

Push:

```bash
git push
```

ArgoCD sees:

```text
Git changed
```

and:

```text
Desired image changed
```

Then:

```text
ArgoCD
   │
   ▼
Deployment
   │
   ▼
New ReplicaSet
   │
   ▼
New Pods
```

---

# 43. Observe Rolling Deployment

Run:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-dev
```

Also:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -w
```

You can watch:

```text
old Pod
   ↓
new Pod starts
   ↓
readiness passes
   ↓
traffic becomes available
   ↓
old Pod terminates
```

This connects directly to our zero-downtime deployment chapter.

---

# 44. Test Self-Healing

First check:

```bash
kubectl get deployment order-service \
  -n ecommerce-dev
```

Suppose desired replicas:

```text
2
```

Manually scale:

```bash
kubectl scale deployment order-service \
  -n ecommerce-dev \
  --replicas=1
```

Immediately inspect:

```bash
kubectl get deployment order-service \
  -n ecommerce-dev
```

ArgoCD should detect drift.

Depending on reconciliation timing, it should return the workload toward the Git-defined replica count.

Check:

```bash
argocd app get order-service-dev
```

---

# 45. Test Git Rollback

Suppose:

```text
Current:
8f31a2c
```

and previous:

```text
72c1abc
```

Revert the GitOps commit.

Then:

```bash
git push
```

ArgoCD sees:

```text
Desired:
72c1abc
```

and deploys the previous version.

Verify:

```bash
kubectl get deployment order-service \
  -n ecommerce-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

# 46. ArgoCD Application Status

Useful CLI commands:

```bash
argocd app list
```

```bash
argocd app get order-service-dev
```

```bash
argocd app history order-service-dev
```

```bash
argocd app diff order-service-dev
```

```bash
argocd app sync order-service-dev
```

```bash
argocd app wait order-service-dev
```

---

# 47. ArgoCD Diff

Before syncing:

```bash
argocd app diff order-service-dev
```

You may see something conceptually like:

```diff
- image: order-service:72c1abc
+ image: order-service:8f31a2c
```

This is very useful because you can inspect what ArgoCD believes will change before synchronization.

---

# 48. Application History

Run:

```bash
argocd app history order-service-dev
```

This provides deployment history associated with the ArgoCD application.

Combined with Git history:

```bash
git log
```

we get:

```text
Git commit
   ↓
GitOps change
   ↓
ArgoCD sync
   ↓
Kubernetes rollout
```

---

# 49. Troubleshooting: Application OutOfSync

Run:

```bash
argocd app get order-service-dev
```

Then:

```bash
argocd app diff order-service-dev
```

Check:

```bash
kubectl get deployment \
  -n ecommerce-dev
```

Check Git:

```bash
git log --oneline
```

Possible causes:

```text
Wrong values
Wrong revision
Manual Kubernetes change
Helm rendering difference
Application controller issue
Ignored differences
Resource generated outside ArgoCD
```

---

# 50. Troubleshooting: Application Unhealthy

Start with:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Logs:

```bash
kubectl logs <pod> \
  -n ecommerce-dev
```

Previous container:

```bash
kubectl logs <pod> \
  -n ecommerce-dev \
  --previous
```

Then inspect:

```bash
kubectl get events \
  -n ecommerce-dev \
  --sort-by=.lastTimestamp
```

---

# 51. Troubleshooting: ArgoCD Cannot Read Git

Check:

```bash
argocd repo list
```

Then inspect repository server:

```bash
kubectl logs \
  deploy/argocd-repo-server \
  -n argocd
```

Potential causes:

```text
Invalid credentials
Repository URL incorrect
Private repository
GitHub permission issue
Network/DNS problem
SSH host key problem
TLS issue
Repository revision doesn't exist
```

---

# 52. Troubleshooting: Helm Rendering Failure

If ArgoCD says it cannot generate manifests:

Check:

```bash
argocd app get order-service-dev
```

Then inspect repo-server logs:

```bash
kubectl logs \
  deploy/argocd-repo-server \
  -n argocd
```

Locally reproduce:

```bash
helm lint helm/order-service
```

and:

```bash
helm template order-service \
  helm/order-service \
  -f values.yaml
```

This is an important troubleshooting technique:

```text
ArgoCD error
    ↓
Reproduce Helm rendering locally
    ↓
Identify chart/value problem
```

---

# 53. Troubleshooting: ImagePullBackOff

ArgoCD may report:

```text
Synced
Unhealthy
```

while Kubernetes reports:

```text
ImagePullBackOff
```

Check:

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Look at:

```text
Events:
Failed to pull image
```

Possible causes:

```text
Wrong ECR repository
Wrong image tag
ECR permissions
Node IAM permissions
Network connectivity
Image doesn't exist
```

Verify ECR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

---

# 54. Troubleshooting: Pod Running but Application Unhealthy

Check:

```bash
kubectl get pods -n ecommerce-dev
```

If:

```text
Running
```

but:

```text
0/1 Ready
```

inspect:

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Check readiness:

```text
/actuator/health/readiness
```

Test from inside the Pod:

```bash
kubectl exec -it <pod> \
  -n ecommerce-dev \
  -- curl localhost:8080/actuator/health
```

This connects:

```text
Spring Boot
    ↓
Actuator
    ↓
Kubernetes readiness
    ↓
ArgoCD health
```

---

# 55. ArgoCD and AWS IAM

A critical distinction:

ArgoCD does not need AWS permissions simply to deploy normal Kubernetes resources into the cluster.

The Kubernetes API handles:

```text
Deployment
Service
ConfigMap
HPA
PDB
Ingress
ServiceAccount
```

AWS integrations are handled by Kubernetes/AWS components such as:

```text
AWS Load Balancer Controller
EBS CSI Driver
EKS Pod Identity
VPC CNI
```

For example:

```text
ArgoCD
   │
   ▼
Kubernetes Ingress
   │
   ▼
AWS Load Balancer Controller
   │
   ▼
AWS ALB
```

The AWS Load Balancer Controller has its own AWS permissions.

---

# 56. ArgoCD Does Not Need Application AWS Credentials

Our order-service Pod might need:

```text
SQS
Secrets Manager
S3
```

Its AWS permissions come from:

```text
ServiceAccount
      │
      ▼
EKS Pod Identity
      │
      ▼
IAM Role
```

Not from ArgoCD.

This separation is important:

```text
ArgoCD
    │
    └── deploys Kubernetes resources

Order Service
    │
    └── accesses AWS services using workload IAM
```

---

# 57. App of Apps

As we move from one service to 21 services, manually creating:

```text
21 Application YAML files
```

is possible, but repetitive.

A common ArgoCD pattern is:

```text
Root Application
      │
      ├── order-service
      ├── payment-service
      ├── inventory-service
      ├── cart-service
      └── ...
```

This is often called:

```text
App of Apps
```

Our repository already has:

```text
argocd/applications/
```

because we will build toward this architecture.

---

# 58. ApplicationSet

Another approach is:

```text
ApplicationSet
      │
      ├── auth-service
      ├── user-service
      ├── catalog-service
      ├── product-service
      ├── order-service
      └── ...
```

Instead of manually defining every Application.

For 21 microservices, ApplicationSet becomes very useful.

We will implement it after the first service is working end-to-end.

---

# 59. Production ArgoCD Architecture

The eventual architecture is:

```text
                         GitHub
                           │
             ┌─────────────┴──────────────┐
             │                            │
      Application repos              GitOps repo
             │                            │
             │                            │
             └──────────────┬─────────────┘
                            │
                            ▼
                         ArgoCD
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
            DEV            QA          STAGING
              │             │             │
              └─────────────┼─────────────┘
                            │
                            ▼
                           PROD
                            │
                            ▼
                           EKS
```

For multi-cluster production:

```text
                    ArgoCD
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
   DEV Cluster     QA Cluster      PROD Cluster
```

---

# 60. What We Have Now

The complete implementation chain is now:

```text
53 Repository Setup
        ↓
54 Order Service
        ↓
55 Local Docker
        ↓
56 Terraform State
        ↓
57 VPC
        ↓
58 IAM
        ↓
59 ECR
        ↓
60 EKS
        ↓
61 EKS Add-ons
        ↓
62 GitHub Actions CI
        ↓
63 Helm
        ↓
64 GitOps
        ↓
65 ArgoCD
```

The first actual deployment path is now ready:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
EKS
    ↓
Order Service
```

---

# 61. Important Production Rule

From this point onward, keep this mental model:

```text
                    ┌─────────────┐
                    │ Application │
                    │    Code     │
                    └──────┬──────┘
                           │
                           ▼
                          CI
                           │
                    "Can we build it?"
                           │
                           ▼
                          ECR
                           │
                           ▼
                       GitOps
                           │
                  "Should we deploy it?"
                           │
                           ▼
                        ArgoCD
                           │
                  "Make cluster match Git"
                           │
                           ▼
                          EKS
                           │
                           ▼
                      Application
```

That separation is the foundation of the production platform.

---

# 62. Next Chapter

### `66-Order-Service-EKS-Deployment.md`

Now we will perform the **first complete application deployment** rather than discussing individual components.

We will connect everything:

```text
Order Service source
        ↓
Maven build
        ↓
Docker image
        ↓
Trivy scan
        ↓
ECR
        ↓
Helm
        ↓
GitOps values
        ↓
ArgoCD
        ↓
EKS
        ↓
ALB
        ↓
Order Service
        ↓
PostgreSQL RDS
```

Then we will test the complete request path:

```text
Client
  ↓
Route 53
  ↓
ALB
  ↓
EKS Service
  ↓
Order Pod
  ↓
RDS PostgreSQL
```

including **real deployment commands, health checks, logs, ALB verification, database connectivity, rollback, and troubleshooting**.
