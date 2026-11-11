# 29-ArgoCD.md

````markdown
# 29 — ArgoCD

## 1. Overview

ArgoCD is the GitOps continuous delivery controller used to continuously reconcile Kubernetes clusters with the desired state stored in Git.

In our Enterprise E-Commerce Platform:

```text
Developer
    |
    v
Application Repository
    |
    v
CI Pipeline
    |
    v
Docker Image
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
Amazon EKS
    |
    v
Kubernetes
    |
    +--> 21 Microservices
    +--> Platform Components
    +--> Observability
````

The fundamental responsibility is:

```text
Git = Desired State

ArgoCD = Reconciliation

EKS = Runtime
```

ArgoCD does not build application images.

ArgoCD does not replace Terraform.

ArgoCD does not replace Kubernetes.

ArgoCD continuously ensures that the Kubernetes environment matches the configuration defined in Git.

---

# 2. Why ArgoCD?

Without GitOps:

```text
CI
 |
 v
kubectl apply
 |
 v
EKS
```

The CI system needs direct deployment access to production.

This creates concerns around:

```text
Credentials
Security
Auditability
Drift
Rollback
Environment separation
Manual intervention
```

With ArgoCD:

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
ArgoCD
 |
 v
EKS
```

CI does not need to directly modify production Kubernetes resources.

---

# 3. CI/CD Separation

Our architecture separates CI and CD.

## CI

CI performs:

```text
Checkout
Build
Unit Tests
SonarQube
Dependency Scan
Secret Scan
Docker Build
Trivy
SBOM
Image Push
```

Result:

```text
Immutable Image
```

Example:

```text
order-service@sha256:abc123...
```

## CD

CD performs:

```text
GitOps Change
        |
        v
ArgoCD
        |
        v
EKS
```

This separation is extremely important.

---

# 4. ArgoCD Mental Model

Think of ArgoCD as:

```text
                    Git
                     |
                     | Desired State
                     v
                  ArgoCD
                     |
             Compare / Reconcile
                     |
                     v
                   EKS
                     |
                     | Live State
                     |
                     +-------> Kubernetes
```

ArgoCD repeatedly asks:

```text
What does Git say?

What does the cluster currently have?

Are they the same?
```

If not:

```text
OutOfSync
```

If configured for automated reconciliation:

```text
ArgoCD
   |
   v
Apply desired state
```

---

# 5. ArgoCD Architecture

A typical ArgoCD installation contains several components:

```text
argocd namespace
│
├── argocd-server
├── argocd-repo-server
├── argocd-application-controller
├── argocd-dex-server
├── argocd-redis
├── argocd-applicationset-controller
└── argocd-notifications-controller
```

The exact component set depends on the ArgoCD version and enabled features.

---

# 6. ArgoCD Server

The API/UI server provides:

```text
Web UI
CLI/API
Authentication
Application management
Repository configuration
Cluster configuration
```

Users interact with:

```text
Browser
   |
   v
ArgoCD Server
```

Example:

```text
https://argocd.example.com
```

Production access should be protected with:

```text
TLS
SSO
RBAC
Network controls
```

---

# 7. Repo Server

The repository server is responsible for retrieving application source/configuration and generating manifests.

For Helm:

```text
Git
 |
 v
Repo Server
 |
 v
Helm rendering
 |
 v
Kubernetes manifests
```

For example:

```text
values.yaml
+
Chart
+
environment configuration
```

becomes:

```yaml
Deployment
Service
ConfigMap
HPA
PDB
```

---

# 8. Application Controller

The Application Controller is the reconciliation engine.

Conceptually:

```text
Git desired state
        |
        v
Application Controller
        |
        v
Kubernetes API
        |
        v
Live state
```

It determines:

```text
Synced?
OutOfSync?
Healthy?
Degraded?
Missing?
```

and performs reconciliation according to the application's sync policy.

---

# 9. Redis

ArgoCD uses Redis for internal caching/state-related operations.

Conceptually:

```text
ArgoCD Components
       |
       v
     Redis
```

Do not confuse this with our application Redis/ElastiCache.

We have:

```text
ArgoCD Redis
```

and:

```text
Application Redis
```

They serve completely different purposes.

---

# 10. Dex / SSO

Organizations may integrate ArgoCD with an identity provider.

For example:

```text
User
 |
 v
Corporate Identity Provider
 |
 v
ArgoCD
```

Possible identity providers include:

```text
OIDC
SAML through an identity platform
GitHub/GitLab identity integrations
Enterprise SSO providers
```

The exact integration depends on organizational identity infrastructure.

---

# 11. ApplicationSet Controller

With 21 microservices and multiple environments, creating every ArgoCD Application manually becomes repetitive.

ApplicationSet can generate Applications.

Conceptually:

```text
ApplicationSet
      |
      +--> order-service-dev
      +--> payment-service-dev
      +--> catalog-service-dev
      |
      +--> order-service-prod
      +--> payment-service-prod
      +--> catalog-service-prod
```

This becomes particularly useful for large application fleets.

---

# 12. Notifications Controller

ArgoCD can integrate with notification systems.

For example:

```text
Deployment failed
       |
       v
ArgoCD
       |
       +--> Slack
       +--> Email
       +--> Webhook
```

This is useful for:

```text
Sync failure
Health degradation
Deployment success
Production rollback
```

---

# 13. Installing ArgoCD on EKS

ArgoCD itself runs inside Kubernetes.

Example:

```text
EKS
│
├── argocd namespace
│   ├── argocd-server
│   ├── argocd-repo-server
│   ├── argocd-application-controller
│   ├── argocd-redis
│   └── ...
│
├── production
│   ├── order-service
│   ├── payment-service
│   └── ...
│
└── monitoring
```

ArgoCD belongs to the platform layer.

It should not run inside an individual application namespace.

---

# 14. Namespace

Create:

```bash
kubectl create namespace argocd
```

Then install ArgoCD using the organization's approved installation method.

A common learning command is:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

For production, do not blindly consume a floating `stable` URL.

Instead:

```text
Pin a tested ArgoCD version
Review release notes
Test in non-production
Promote the exact version
```

Production platform installation should itself be version controlled.

---

# 15. Production Installation Model

A production platform team should preferably manage ArgoCD through Git/Helm/Terraform/bootstrap automation.

Conceptually:

```text
Infrastructure Repository
        |
        v
EKS
        |
        v
Bootstrap
        |
        v
ArgoCD
        |
        v
GitOps Repository
```

The bootstrap mechanism should be deterministic and reproducible.

Avoid:

```text
Engineer laptop
    |
    v
kubectl apply
    |
    v
Production
```

as the long-term operating model.

---

# 16. ArgoCD Installation Responsibility

Terraform:

```text
VPC
EKS
IAM
Security Groups
AWS infrastructure
```

ArgoCD:

```text
Kubernetes application delivery
```

Platform bootstrap:

```text
ArgoCD installation
Platform controllers
CRDs
```

A mature platform may eventually use ArgoCD itself to manage many platform components after the initial bootstrap.

This is sometimes called:

```text
App-of-Apps
```

or a similar bootstrap pattern.

---

# 17. ArgoCD Application

The central ArgoCD object is:

```text
Application
```

An Application defines:

```text
Source
Destination
Project
Sync policy
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-prod
  namespace: argocd

spec:
  project: production

  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    path: environments/prod/order-service
    targetRevision: main

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

# 18. Source

The source tells ArgoCD:

```text
Where is the Git repository?

Which revision?

Which directory?

Which rendering method?
```

Example:

```yaml
source:
  repoURL: https://github.com/company-ecommerce/gitops.git
  targetRevision: main
  path: environments/prod/order-service
```

ArgoCD retrieves that configuration.

---

# 19. Destination

Destination defines:

```text
Which cluster?

Which namespace?
```

Example:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: production
```

For external clusters:

```text
ArgoCD
   |
   +--> DEV EKS
   |
   +--> QA EKS
   |
   +--> STAGING EKS
   |
   +--> PROD EKS
```

---

# 20. ArgoCD Project

Projects provide logical and security boundaries.

Example:

```text
production project
```

can restrict:

```text
Allowed repositories
Allowed clusters
Allowed namespaces
Allowed resource types
```

Conceptually:

```text
production project
 |
 +--> GitOps repository
 |
 +--> production cluster
 |
 +--> production namespace
```

This is especially important in multi-team environments.

---

# 21. Project Example

Conceptual:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject

metadata:
  name: production
  namespace: argocd

spec:
  sourceRepos:
    - https://github.com/company-ecommerce/gitops.git

  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
```

A real production project should contain carefully designed restrictions rather than broad wildcard permissions.

---

# 22. Application + Project

The relationship:

```text
AppProject
     |
     +--> Application
             |
             +--> Git
             |
             +--> Cluster
             |
             +--> Namespace
```

Example:

```text
production project
        |
        +--> order-service-prod
        +--> payment-service-prod
        +--> catalog-service-prod
        +--> inventory-service-prod
```

---

# 23. Helm Source

ArgoCD can use Helm.

Example:

```yaml
source:
  repoURL: https://github.com/company-ecommerce/gitops.git
  targetRevision: main
  path: applications/order-service

  helm:
    valueFiles:
      - ../../environments/prod/order-service/values.yaml
```

The exact path structure should be designed so that ArgoCD can resolve the files reliably.

A common alternative is to make each environment path self-contained.

---

# 24. Recommended Environment Structure

For simplicity:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            ├── Chart.yaml
            ├── values.yaml
            └── templates/
```

Then ArgoCD can point directly to:

```text
environments/prod/order-service
```

This reduces cross-directory path complexity.

Another architecture can separate charts and values.

Both are valid if consistently implemented.

---

# 25. GitOps Repository

Our project:

```text
gitops/
├── applications/
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
└── argocd/
    ├── projects/
    ├── applications/
    └── appsets/
```

ArgoCD configuration itself should also be version controlled.

---

# 26. Sync Status

ArgoCD commonly shows:

```text
Synced
OutOfSync
Unknown
```

### Synced

Git and cluster desired state match.

```text
Git = Cluster
```

### OutOfSync

They differ.

```text
Git != Cluster
```

### Unknown

ArgoCD cannot reliably determine the current state.

Possible reasons:

```text
Cluster connectivity
Permission problem
Resource error
Controller issue
```

---

# 27. Health Status

Sync and health are different.

A service can be:

```text
Synced
```

but:

```text
Degraded
```

Example:

```text
Git configuration:
valid

ArgoCD sync:
successful

Pod:
CrashLoopBackOff
```

Therefore:

```text
Sync != Application health
```

Always check both.

---

# 28. Automated Sync

Example:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

This tells ArgoCD to automatically reconcile changes.

Useful for:

```text
DEV
QA
non-critical environments
```

depending on organizational policy.

---

# 29. Manual Production Sync

Production can use:

```text
Git PR
   |
   v
Review
   |
   v
Merge
   |
   v
ArgoCD
   |
   v
Manual sync
   |
   v
Production
```

This provides an additional operational control.

The right model depends on release governance.

---

# 30. Self-Heal

Suppose Git says:

```text
replicas: 3
```

Someone changes the live Deployment:

```text
replicas: 1
```

ArgoCD detects:

```text
Desired = 3
Live = 1
```

If self-heal is enabled:

```text
ArgoCD
   |
   v
restore desired state
   |
   v
replicas = 3
```

This prevents persistent manual drift.

---

# 31. Prune

Suppose a resource is removed from Git.

Example:

```text
Git:
Deployment
Service
```

Then Deployment is removed from Git.

If pruning is enabled:

```text
Git
 |
 X Deployment
 |
 v
ArgoCD
 |
 v
remove Deployment
```

Pruning is powerful.

Incorrect Git deletion can therefore cause actual production resource deletion.

Protect GitOps `main` accordingly.

---

# 32. Sync Waves

Some resources need to be created before others.

Example:

```text
Secret
   |
   v
Deployment
```

or:

```text
CRD
   |
   v
Custom Resource
```

ArgoCD supports synchronization ordering mechanisms such as sync waves.

Conceptually:

```text
Wave 0
CRDs
 |
 v
Wave 1
Platform resources
 |
 v
Wave 2
Applications
```

This is useful when application startup depends on infrastructure/controllers already existing.

---

# 33. Hooks

ArgoCD supports lifecycle hooks for certain deployment workflows.

Conceptually:

```text
PreSync
   |
   v
Sync
   |
   v
PostSync
```

Potential use cases:

```text
Migration
Validation
Smoke tests
Post-deployment checks
```

Hooks should be used carefully.

A migration hook that runs automatically against production databases can be dangerous if not designed for idempotency and rollback compatibility.

---

# 34. Database Migration Warning

Never assume:

```text
ArgoCD sync
=
database migration safe
```

Example:

```text
Application deployment
       |
       v
Database migration
       |
       X
Breaking schema
       |
       v
Old Pods fail
```

Production migrations must be backward compatible and carefully sequenced.

Use tools such as:

```text
Flyway
Liquibase
```

according to application standards.

---

# 35. ArgoCD Diff

Before deployment:

```text
Git
   |
   v
ArgoCD
   |
   v
Diff
```

Example:

```diff
replicas:
- 3
+ 5
```

or:

```diff
image:
- sha256:old123
+ sha256:new456
```

Reviewing this diff is one of the most important GitOps operational practices.

---

# 36. Image Promotion Example

Current:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:old123
```

New release:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:new456
```

GitOps PR:

```text
Promote order-service 1.6.0 to production
```

After approval:

```text
Git
 |
 v
ArgoCD
 |
 v
Helm
 |
 v
Deployment
 |
 v
EKS
```

The image itself is not rebuilt.

---

# 37. Build Once, Deploy Many

The artifact chain:

```text
Application commit
      |
      v
CI build
      |
      v
Docker image
      |
      v
ECR digest
      |
      +------> DEV
      |
      +------> QA
      |
      +------> STAGING
      |
      +------> PROD
```

Example:

```text
sha256:abc123
```

must remain the same throughout promotion.

Only environment configuration changes.

---

# 38. ArgoCD Rollback

Suppose:

```text
Version 1.5.2
     |
     v
Version 1.6.0
```

causes an incident.

Preferred GitOps rollback:

```text
Git commit
   |
   v
Revert promotion
   |
   v
ArgoCD
   |
   v
1.5.2
```

This keeps:

```text
Git = desired state
```

rather than creating a hidden cluster-only rollback.

---

# 39. ArgoCD History

ArgoCD provides application history.

Conceptually:

```text
Revision 10
order-service 1.5.2

Revision 11
order-service 1.6.0

Revision 12
rollback to 1.5.2
```

This helps with:

```text
Auditing
Incident investigation
Release history
Rollback analysis
```

Git history remains the primary source of configuration history.

---

# 40. ArgoCD CLI

Common commands include:

```bash
argocd login <argocd-server>
```

List applications:

```bash
argocd app list
```

Inspect:

```bash
argocd app get order-service-prod
```

Sync:

```bash
argocd app sync order-service-prod
```

History:

```bash
argocd app history order-service-prod
```

Diff:

```bash
argocd app diff order-service-prod
```

These commands should be used according to production access policies.

---

# 41. Kubernetes Inspection

ArgoCD does not replace `kubectl`.

After deployment:

```bash
kubectl get applications -n argocd
```

Then:

```bash
kubectl get deployment -n production
```

Then:

```bash
kubectl get pods -n production
```

Then:

```bash
kubectl logs deployment/order-service -n production
```

ArgoCD tells us about desired-state reconciliation.

Kubernetes commands tell us what is happening inside the cluster.

---

# 42. ArgoCD Troubleshooting Layer

Use this order:

```text
1. Git
2. ArgoCD Application
3. Helm rendering
4. Sync
5. Kubernetes resource
6. Pod
7. Container
8. Application
9. Dependencies
```

Example:

```text
Git
 |
 v
ArgoCD
 |
 v
Helm
 |
 v
Deployment
 |
 v
ReplicaSet
 |
 v
Pod
 |
 v
Spring Boot
 |
 +--> RDS
 +--> Redis
 +--> SQS
```

Do not jump directly to the application when the actual problem is a GitOps rendering failure.

---

# 43. Scenario: Git Change Not Detected

Check:

```bash
argocd app get order-service-prod
```

Potential causes:

```text
Wrong repository
Wrong branch
Wrong path
Git authentication problem
Repository webhook/detection delay
Application points to another revision
```

Verify:

```text
repoURL
targetRevision
path
```

---

# 44. Scenario: OutOfSync

Check:

```text
ArgoCD Application
 |
 v
Diff
```

Possible causes:

```text
Git changed
Manual cluster change
Controller-generated fields
Ignored fields configuration
Resource mutation
```

Determine whether the difference is:

```text
Expected
```

or:

```text
Drift
```

Do not blindly press Sync without understanding the diff.

---

# 45. Scenario: Sync Failed

Possible causes:

```text
Invalid YAML
Helm template failure
Kubernetes validation failure
RBAC denial
CRD missing
Namespace missing
Admission policy denial
Resource quota
Invalid image reference
```

Inspect:

```bash
argocd app get order-service-prod
```

and:

```bash
kubectl get events -n production
```

---

# 46. Scenario: Pod ImagePullBackOff

ArgoCD may be:

```text
Synced
```

while Kubernetes is:

```text
ImagePullBackOff
```

Check:

```bash
kubectl describe pod <pod> -n production
```

Verify:

```text
ECR repository
Image digest
Node permissions
Network connectivity
Image existence
```

The problem is now in the runtime layer, not necessarily ArgoCD.

---

# 47. Scenario: CrashLoopBackOff

Check:

```bash
kubectl logs <pod> -n production
```

If previous container crashed:

```bash
kubectl logs <pod> -n production --previous
```

Then:

```bash
kubectl describe pod <pod> -n production
```

Potential causes:

```text
Bad environment variable
Missing Secret
Database unavailable
Redis unavailable
Application exception
Incorrect command
Port/configuration problem
OOMKilled
```

---

# 48. Scenario: Readiness Probe Failure

ArgoCD:

```text
Synced
```

Kubernetes:

```text
Pod Running
Ready: False
```

This means the container may be running but Kubernetes does not consider it ready.

Check:

```bash
kubectl describe pod <pod> -n production
```

Possible causes:

```text
Wrong health endpoint
Wrong port
Application startup too slow
Database dependency
Incorrect probe path
TLS mismatch
```

Because the Pod is not ready, Service traffic may not reach it.

---

# 49. Scenario: ArgoCD Cannot Access Cluster

Potential causes:

```text
Expired/invalid cluster credentials
RBAC
Network connectivity
API endpoint restrictions
Security Groups
EKS endpoint configuration
```

Architecture:

```text
ArgoCD Pod
    |
    v
Kubernetes API
    |
    v
EKS
```

For a private EKS API endpoint, ensure ArgoCD has appropriate network connectivity.

---

# 50. Scenario: ArgoCD Cannot Access Git

Potential causes:

```text
SSH key
HTTPS token
Repository permissions
Network
DNS
Proxy
Certificate
Credential expiration
```

The repo server must be able to retrieve:

```text
Git repository
```

A healthy cluster does not guarantee healthy Git connectivity.

---

# 51. GitHub Repository Authentication

ArgoCD can access repositories using:

```text
HTTPS credentials
SSH deploy keys
GitHub App integration
```

Production organizations should prefer short-lived or tightly scoped credentials where supported.

Do not use a developer's personal GitHub password.

---

# 52. GitOps Repository Permissions

ArgoCD should have only the repository access required.

Example:

```text
ArgoCD
 |
 +--> company-ecommerce/gitops
```

It should not automatically receive:

```text
Write access to every GitHub repository
```

If ArgoCD only needs read access to Git:

```text
Read-only Git access
```

is preferable.

---

# 53. ArgoCD RBAC

ArgoCD RBAC can control:

```text
Who can:
view applications
sync applications
delete applications
manage repositories
manage clusters
modify projects
```

Example organizational model:

```text
Developer
 |
 +--> View
 +--> Sync DEV

SRE
 |
 +--> View
 +--> Sync QA/STAGING
 +--> Emergency operations

Release Manager
 |
 +--> Production approval

Platform Admin
 |
 +--> ArgoCD administration
```

Exact roles depend on company policy.

---

# 54. Production RBAC Principle

Do not give every developer:

```text
ArgoCD admin
```

Production access should follow least privilege.

Example:

```text
Developer
   |
   v
Git PR

ArgoCD
   |
   v
automated/restricted deployment
```

Instead of:

```text
Developer
   |
   v
cluster-admin
```

---

# 55. SSO Architecture

Example:

```text
Engineer
   |
   v
Corporate SSO
   |
   v
Identity Provider
   |
   v
ArgoCD
   |
   v
RBAC
   |
   +--> Developer
   +--> SRE
   +--> Platform
   +--> Admin
```

This provides centralized identity management.

---

# 56. TLS

Production ArgoCD UI/API should use TLS.

Architecture:

```text
Engineer
   |
 HTTPS
   |
   v
ALB / Ingress
   |
   v
argocd-server
```

Do not expose ArgoCD's management interface to the public internet without strong authentication and access controls.

---

# 57. ArgoCD Behind ALB

A production AWS design could be:

```text
Route 53
   |
   v
ALB
   |
   v
Ingress
   |
   v
argocd-server
```

The ALB can provide:

```text
TLS termination
WAF where appropriate
Access controls
DNS integration
```

ArgoCD itself still needs authentication/RBAC.

---

# 58. Private ArgoCD

An enterprise may instead keep ArgoCD internal:

```text
Corporate Network
       |
       v
Internal ALB
       |
       v
ArgoCD
```

This is often preferable for management-plane interfaces.

External exposure should be justified by a real requirement.

---

# 59. ArgoCD High Availability

Production ArgoCD should be designed for availability.

Consider:

```text
Multiple replicas
Pod anti-affinity/topology spread
PodDisruptionBudget
Adequate resources
Highly available cluster
```

For critical platform components:

```text
ArgoCD
Prometheus
Grafana
Logging
Tracing
```

should not depend on a single Pod.

---

# 60. ArgoCD Resource Sizing

Do not assume:

```text
CPU: 100m
Memory: 128Mi
```

will always be enough.

ArgoCD workload depends on:

```text
Number of applications
Number of resources
Number of clusters
Sync frequency
Manifest complexity
Helm rendering
Repository size
ApplicationSet scale
```

A small lab environment and a 500-application enterprise installation have very different resource requirements.

---

# 61. ArgoCD Monitoring

Monitor:

```text
Application sync status
Application health
Reconciliation failures
Controller CPU
Controller memory
Repo server latency
Git fetch failures
API server errors
Redis health
Kubernetes API errors
```

Prometheus can collect ArgoCD metrics.

Grafana dashboards can visualize:

```text
Healthy applications
OutOfSync applications
Sync failures
Deployment frequency
Reconciliation latency
```

---

# 62. ArgoCD Alerts

Useful alerts:

```text
Production application OutOfSync
Production application Degraded
Repeated sync failure
Repository unavailable
Cluster unavailable
ArgoCD controller unhealthy
High reconciliation latency
```

Example:

```text
order-service-prod
   |
   v
Health = Degraded
   |
   v
Alertmanager
   |
   v
SRE notification
```

---

# 63. ArgoCD and Observability

The overall platform becomes:

```text
                         ArgoCD
                           |
                           v
                         EKS
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 Applications          Kubernetes          Platform
       |                                       |
       +-------------------+-------------------+
                           |
                           v
                    Observability
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
    Prometheus           ELK             OpenTelemetry
        |                  |                  |
        v                  v                  v
     Grafana             Kibana             Jaeger
```

This allows both deployment and runtime health to be observed.

---

# 64. Multi-Cluster Architecture

Suppose we have:

```text
DEV EKS
QA EKS
STAGING EKS
PROD EKS
```

ArgoCD can manage multiple clusters.

Conceptually:

```text
                         ArgoCD
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
         DEV EKS          QA EKS        STAGING EKS
                                            |
                                            v
                                         PROD EKS
```

Access to each cluster should be tightly controlled.

---

# 65. Multi-Account Architecture

Our enterprise target may be:

```text
AWS Organization
│
├── Dev Account
│    └── EKS
│
├── QA Account
│    └── EKS
│
├── Staging Account
│    └── EKS
│
└── Production Account
     └── EKS
```

ArgoCD may run in a dedicated management/platform cluster or in one environment, depending on architecture.

Cross-account access must be explicitly designed.

---

# 66. ArgoCD in Management Cluster

One model:

```text
Management EKS
     |
     v
   ArgoCD
     |
     +--------> Dev EKS
     |
     +--------> QA EKS
     |
     +--------> Staging EKS
     |
     +--------> Prod EKS
```

Advantages:

```text
Centralized GitOps
Centralized visibility
Centralized RBAC
```

Tradeoff:

```text
Management cluster becomes critical infrastructure
```

Therefore it needs strong HA and DR.

---

# 67. ArgoCD Per Cluster

Another model:

```text
DEV EKS
 |
 +--> ArgoCD

QA EKS
 |
 +--> ArgoCD

PROD EKS
 |
 +--> ArgoCD
```

Advantages:

```text
Strong isolation
Reduced blast radius
Independent operation
```

Tradeoff:

```text
More ArgoCD installations
More platform maintenance
```

The right design depends on organizational scale.

---

# 68. App-of-Apps

A common pattern is:

```text
Root Application
       |
       +--> order-service
       +--> payment-service
       +--> catalog-service
       +--> inventory-service
       +--> platform components
```

Conceptually:

```text
root-app
   |
   +--> applications
   |
   +--> projects
   |
   +--> observability
   |
   +--> ingress
```

This allows a small set of root definitions to bootstrap many applications.

However, it must be carefully controlled because a root application can have a very large blast radius.

---

# 69. ApplicationSet vs App-of-Apps

They solve related but different problems.

### ApplicationSet

Useful for generating many Applications from a template.

```text
ApplicationSet
   |
   +--> 21 services
```

### App-of-Apps

Useful for hierarchical application bootstrapping.

```text
Root Application
   |
   +--> child Applications
```

They can also be used together when the architecture justifies it.

---

# 70. Example ApplicationSet Concept

Conceptually:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: ecommerce-services
  namespace: argocd

spec:
  generators:
    - list:
        elements:
          - service: order-service
            environment: prod
          - service: payment-service
            environment: prod
          - service: catalog-service
            environment: prod
```

The template then generates Applications based on these values.

A production implementation should validate generated names, paths, destinations and permissions carefully.

---

# 71. Sync Ordering Across Microservices

Consider:

```text
auth-service
order-service
payment-service
notification-service
```

Order service may depend on:

```text
auth
payment
notification
```

But Kubernetes services may start independently.

Do not use ArgoCD sync ordering as a substitute for proper distributed-system resilience.

Applications should tolerate temporary dependency failures through:

```text
Retries
Timeouts
Circuit breakers
Readiness
Backoff
Message queues
Idempotency
```

ArgoCD should handle deployment ordering only where there is a real deployment dependency.

---

# 72. Health Checks

ArgoCD can assess resource health.

Example:

```text
Deployment:
Healthy

Pod:
Ready

Service:
Healthy
```

But application-specific health may require additional logic.

For example:

```text
Pod Ready
```

does not necessarily mean:

```text
Business workflow healthy
```

Runtime monitoring remains necessary.

---

# 73. ArgoCD and Kubernetes Readiness

Deployment flow:

```text
ArgoCD
   |
   v
Deployment
   |
   v
Pod
   |
   v
Startup
   |
   v
Readiness
   |
   v
Service endpoints
```

If readiness fails:

```text
Pod Running
Pod NotReady
```

ArgoCD may report degraded application health depending on resource health assessment.

This is why health status must be monitored after synchronization.

---

# 74. Zero-Downtime Deployment

ArgoCD deploys desired state.

Kubernetes performs the rollout.

For example:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Combined with:

```text
Readiness probe
PDB
Multiple replicas
Topology distribution
```

we can achieve controlled rolling deployments.

Zero-downtime deployment will be covered more deeply later.

---

# 75. Production Deployment Flow

Our complete production process:

```text
1. Developer changes code
        |
        v
2. Application PR
        |
        v
3. CI
        |
        +--> Test
        +--> SonarQube
        +--> Security
        +--> Docker
        +--> Trivy
        +--> SBOM
        |
        v
4. ECR
        |
        v
5. Immutable image digest
        |
        v
6. GitOps PR
        |
        v
7. DEV
        |
        v
8. QA
        |
        v
9. STAGING
        |
        v
10. Production approval
        |
        v
11. Git merge
        |
        v
12. ArgoCD detects change
        |
        v
13. Helm renders manifests
        |
        v
14. ArgoCD applies resources
        |
        v
15. Kubernetes rollout
        |
        v
16. Readiness passes
        |
        v
17. Production traffic
```

---

# 76. Complete Order-Service Example

GitOps contains:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

ArgoCD reads:

```text
environments/prod/order-service
```

Helm renders:

```yaml
containers:
  - name: order-service
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:abc123...
```

ArgoCD compares:

```text
Git desired state
        |
        v
Current cluster state
```

If different:

```text
OutOfSync
```

After synchronization:

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
```

Once readiness succeeds:

```text
Service
   |
   v
Pods
```

and traffic reaches the application.

---

# 77. GitOps Rollback Scenario

Production:

```text
order-service 1.6.0
```

Incident:

```text
HTTP 500 rate increased
```

SRE investigates:

```text
Grafana
ELK
Jaeger
Kubernetes
```

Decision:

```text
Rollback
```

GitOps:

```text
revert commit
```

ArgoCD:

```text
detect change
      |
      v
sync
```

Kubernetes:

```text
1.6.0
   |
   v
1.5.2
```

Observability confirms recovery.

---

# 78. What ArgoCD Does Not Solve

ArgoCD does not automatically solve:

```text
Application bugs
Database failures
Bad migrations
RDS capacity
Redis failures
Kafka/SQS problems
Network failures
Bad Docker images
Security vulnerabilities
Insufficient node capacity
Bad application design
```

It only manages desired-state delivery/reconciliation.

For example:

```text
Git:
correct

ArgoCD:
healthy

Pod:
Running

Application:
returns HTTP 500
```

ArgoCD can be completely healthy while the business application is broken.

---

# 79. ArgoCD + Prometheus

Prometheus can collect:

```text
ArgoCD metrics
Kubernetes metrics
Application metrics
Node metrics
```

Grafana can then show:

```text
ArgoCD
 |
 +--> Applications Healthy
 +--> Applications OutOfSync
 +--> Sync failures
 +--> Reconciliation latency
```

This makes GitOps itself observable.

---

# 80. ArgoCD + Alertmanager

Example:

```text
order-service-prod
Health = Degraded
       |
       v
Prometheus
       |
       v
Alertmanager
       |
       v
SRE
```

Possible notification:

```text
PRODUCTION ALERT

Application:
order-service-prod

Status:
Degraded

Reason:
Readiness probe failing
```

This creates a feedback loop between deployment and operations.

---

# 81. Production Security Checklist

```text
[ ] ArgoCD version pinned
[ ] TLS enabled
[ ] SSO configured
[ ] RBAC configured
[ ] Git repository access restricted
[ ] Cluster access restricted
[ ] Production Project restricted
[ ] No unnecessary cluster-admin
[ ] ArgoCD namespace protected
[ ] Secrets not stored in Git
[ ] Production sync controlled
[ ] GitOps main branch protected
[ ] CODEOWNERS configured
[ ] Audit logging enabled
[ ] ArgoCD metrics monitored
[ ] Backup/DR plan defined
```

---

# 82. Production Reliability Checklist

```text
[ ] Multiple ArgoCD replicas where appropriate
[ ] Pod disruption protection
[ ] Topology distribution
[ ] Adequate resources
[ ] Git repository availability
[ ] Cluster API availability
[ ] Network connectivity
[ ] Monitoring
[ ] Alerts
[ ] Incident procedure
[ ] Rollback procedure
[ ] DR procedure
```

---

# 83. Production GitOps Checklist

```text
[ ] Application source is reviewed
[ ] CI passed
[ ] Image immutable
[ ] Image digest recorded
[ ] GitOps PR reviewed
[ ] Helm rendering validated
[ ] Security policy passed
[ ] Environment configuration reviewed
[ ] Secrets referenced securely
[ ] Production approval obtained
[ ] ArgoCD sync monitored
[ ] Pod rollout monitored
[ ] Application metrics monitored
[ ] Logs monitored
[ ] Traces monitored
[ ] Rollback plan available
```

---

# 84. Troubleshooting Decision Tree

When production deployment fails:

```text
                Deployment Problem
                       |
                       v
              Is Git correct?
                 /        \
               NO          YES
               |            |
               v            v
          Fix Git       ArgoCD healthy?
                            /      \
                          NO        YES
                          |          |
                          v          v
                    Fix ArgoCD    K8s healthy?
                                    /    \
                                  NO      YES
                                  |        |
                                  v        v
                             Fix Pods   App healthy?
                                          /    \
                                        NO      YES
                                        |        |
                                        v        v
                                  App/dependency  Done
                                  troubleshooting
```

This prevents random troubleshooting.

---

# 85. Layer-by-Layer Troubleshooting

## Layer 1 — Git

```text
Correct branch?
Correct path?
Correct values?
Correct image digest?
```

## Layer 2 — ArgoCD

```text
Application exists?
Repository accessible?
Comparison successful?
Sync successful?
```

## Layer 3 — Kubernetes

```text
Deployment?
ReplicaSet?
Pods?
Events?
```

## Layer 4 — Application

```text
Startup?
Readiness?
Logs?
Metrics?
```

## Layer 5 — Dependencies

```text
RDS?
Redis?
SQS?
SNS?
External APIs?
```

---

# 86. Final Architecture

```text
                         GitHub
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
 Application Repo    Infrastructure Repo   GitOps Repo
          |                 |                  |
          v                 v                  v
         CI             Terraform            Helm
          |                 |                  |
          v                 v                  v
         ECR                AWS              ArgoCD
                                                |
                                                v
                                               EKS
                                                |
                    +---------------------------+------------------------+
                    |                           |                        |
                    v                           v                        v
             Microservices                 Platform              Observability
                    |                           |                        |
                    v                           v                        v
               RDS/Redis                  Ingress/Secrets        Prometheus/ELK
                    |                                                    |
                    v                                                    v
                 SQS/SNS                                             OTel/Jaeger
```

---

# 87. Most Important Mental Model

Remember:

```text
Terraform
    |
    v
AWS Infrastructure

CI
    |
    v
Container Artifact

ECR
    |
    v
Immutable Image

GitOps
    |
    v
Desired Kubernetes State

Helm
    |
    v
Manifest Rendering

ArgoCD
    |
    v
Reconciliation

Kubernetes
    |
    v
Runtime
```

And:

```text
Prometheus
ELK
OpenTelemetry
Jaeger
```

observe the runtime.

---

# 88. Final Production Flow

The production architecture is:

```text
Developer
    |
    v
GitHub Application Repo
    |
    v
Pull Request
    |
    v
CI
    |
    +--> Build
    +--> Test
    +--> SonarQube
    +--> Dependency Scan
    +--> Gitleaks
    +--> Trivy
    +--> SBOM
    |
    v
ECR
    |
    v
Immutable Image Digest
    |
    v
GitOps Pull Request
    |
    v
Approval
    |
    v
GitOps main
    |
    v
ArgoCD
    |
    v
Helm
    |
    v
Kubernetes API
    |
    v
EKS
    |
    v
Deployment
    |
    v
Pods
    |
    +----------+----------+----------+
    |          |          |          |
    v          v          v          v
   RDS       Redis       SQS        SNS
```

This gives us:

```text
Reproducibility
Auditability
Security
Controlled promotion
Immutable artifacts
Drift detection
Rollback
Environment separation
Declarative infrastructure
```

---

# 89. What We Have Now Completed

At this stage, our project has covered:

```text
20  RDS
21  EKS Database Management
22  Redis
23  SQS/SNS
24  RabbitMQ
25  Kafka
26  Secrets Management
27  GitOps Repository
28  Helm
29  ArgoCD
```

The next major section is observability.

---

# 90. Next Chapter

## 30 — Prometheus

The next chapter will build the metrics layer:

```text
Applications
     |
     v
Micrometer
     |
     v
Prometheus
     |
     +--> Kubernetes metrics
     +--> Application metrics
     +--> Node metrics
     +--> ArgoCD metrics
     |
     v
Grafana
     |
     v
Alertmanager
```

We will cover Prometheus in production depth:

```text
Prometheus architecture
Prometheus server
Prometheus Operator
kube-prometheus-stack
ServiceMonitor
PodMonitor
scrape configuration
labels
relabeling
metric types
Counter
Gauge
Histogram
Summary
PromQL
application metrics
Spring Boot metrics
Kubernetes metrics
kube-state-metrics integration
Node Exporter
EKS monitoring
ArgoCD monitoring
SLO metrics
RED metrics
USE metrics
recording rules
alerting rules
retention
storage
high availability
Thanos considerations
remote write
cardinality
memory usage
Prometheus troubleshooting
production sizing
production security
production architecture
```

The target architecture will become:

```text
                         EKS
                          |
       +------------------+------------------+
       |                  |                  |
       v                  v                  v
  Microservices       Kubernetes Nodes     ArgoCD
       |                  |                  |
       v                  v                  v
  Micrometer          Node Exporter      ArgoCD metrics
       |                  |                  |
       +------------------+------------------+
                          |
                          v
                     Prometheus
                          |
                          v
                       Grafana
                          |
                          v
                    Alertmanager
```

This is the metrics foundation for the remaining observability stack.

```
```
