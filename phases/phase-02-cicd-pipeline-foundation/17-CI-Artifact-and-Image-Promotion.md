# 17-CI-Artifact-and-Image-Promotion.md

# CI Artifact & Image Promotion — DEV → QA → STAGING → PROD

Now we connect the CI, security, ECR, and GitOps concepts into a **real production release process**.

The most important principle in this chapter is:

> **Build the application once, create one immutable image, validate that image, and promote the exact same artifact through every environment.**

We should never rebuild the application separately for QA or production just because the target environment changed.

---

# 1. The Production Problem

Suppose a developer commits:

```text
8f3a21c9
```

CI builds:

```text
order-service:8f3a21c9
```

Then someone says:

> "Build it again for QA."

And later:

> "Build it again for production."

That creates:

```text
Git
 │
 ├── Build #1 → DEV image
 │
 ├── Build #2 → QA image
 │
 └── Build #3 → PROD image
```

Even when the source commit is the same, these builds are not necessarily guaranteed to be byte-for-byte identical.

That is not the model we want.

---

# 2. Correct Production Model

Instead:

```text
Git commit
    │
    ▼
Build ONCE
    │
    ▼
order-service image
    │
    ▼
Security validation
    │
    ▼
Immutable artifact
    │
    ├───────────────┐
    ▼               ▼
   DEV             QA
                    │
                    ▼
                 STAGING
                    │
                    ▼
                  PROD
```

The artifact stays the same.

Only the **environment configuration and deployment state** change.

---

# 3. What Actually Changes Between Environments?

The application image should generally remain unchanged.

These can change:

```text
DEV
├── replicas
├── database endpoint
├── Redis endpoint
├── SQS configuration
├── resource sizing
└── environment-specific configuration

QA
├── replicas
├── database endpoint
├── Redis endpoint
└── configuration

PROD
├── replicas
├── database endpoint
├── Redis endpoint
├── resource sizing
├── HA configuration
└── production secrets
```

The application artifact:

```text
order-service:8f3a21c9
```

doesn't need to change.

---

# 4. Complete Production Architecture

Our target architecture:

```text
                         GitHub
                           │
                           ▼
                    ┌──────────────┐
                    │ CI Pipeline  │
                    └──────┬───────┘
                           │
                           ▼
                      Build Image
                           │
                           ▼
                      Security Gates
                           │
                           ▼
                          ECR
                           │
                           ▼
                    Immutable Digest
                           │
                           ▼
                    GitOps Repository
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
              DEV          QA        STAGING
               │           │           │
               └───────────┴───────────┘
                           │
                       Approval
                           │
                           ▼
                          PROD
                           │
                           ▼
                         ArgoCD
                           │
                           ▼
                          EKS
```

---

# 5. Separate CI and CD

This distinction is extremely important.

## CI

Continuous Integration:

```text
Source
 ↓
Build
 ↓
Test
 ↓
Scan
 ↓
Package
 ↓
Image
 ↓
ECR
```

## CD

Continuous Delivery/Deployment:

```text
ECR
 ↓
GitOps
 ↓
ArgoCD
 ↓
EKS
```

Therefore:

```text
CI ≠ CD
```

They work together but have different responsibilities.

---

# 6. CI Produces the Artifact

For `order-service`:

```text
Developer
    ↓
Git commit
    ↓
CI
    ↓
Maven build
    ↓
Unit tests
    ↓
SonarQube
    ↓
Gitleaks
    ↓
Trivy
    ↓
Docker build
    ↓
SBOM
    ↓
Image signing
    ↓
ECR
```

CI's job ends around the artifact publication/promotion handoff.

---

# 7. CD Deploys the Artifact

Then:

```text
ECR
 │
 ▼
GitOps repository
 │
 ▼
ArgoCD
 │
 ▼
EKS
```

ArgoCD does not compile Java.

ArgoCD does not run Maven.

ArgoCD does not build Docker images.

ArgoCD deploys the artifact described by Git.

---

# 8. Artifact Metadata

When CI finishes, we should know:

```text
Application:
order-service

Git repository:
order-service

Commit:
8f3a21c9

Image:
order-service:8f3a21c9

Digest:
sha256:abc123...

Build:
CI #1234

SBOM:
Generated

Security:
Passed
```

This information should be available to the release process.

---

# 9. Why the Digest Is Important

Imagine:

```text
order-service:8f3a21c9
```

points to:

```text
sha256:ABC
```

That digest represents the exact image.

Production should ideally deploy:

```text
order-service@sha256:ABC
```

rather than relying only on:

```text
order-service:8f3a21c9
```

This eliminates ambiguity.

---

# 10. Environment Promotion

Let's take our image:

```text
order-service@sha256:ABC
```

Promotion:

```text
                   IMAGE
                     │
             sha256:ABC
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         DEV        QA       STAGING
                                │
                                ▼
                              PROD
```

The digest doesn't change.

---

# 11. DEV Deployment

GitOps:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:ABC
```

ArgoCD syncs.

EKS starts:

```text
order-service
      │
      ├── Pod 1
      ├── Pod 2
      └── Pod 3
```

---

# 12. DEV Validation

After deployment, CI/CD or testers validate:

```text
Health check
API functionality
Database connectivity
Redis connectivity
SQS integration
Authentication
Logging
Metrics
Tracing
```

For example:

```bash
curl https://dev-api.example.com/api/v1/orders/123
```

If successful:

```text
DEV = PASS
```

Then promotion can continue.

---

# 13. QA Promotion

QA GitOps changes:

```yaml
image:
  digest: sha256:ABC
```

ArgoCD deploys the **same image**.

QA may execute:

```text
Integration tests
Contract tests
Regression tests
Performance tests
Security tests
End-to-end tests
```

---

# 14. Staging

Staging should resemble production as closely as practical.

For example:

```text
STAGING
├── same EKS architecture
├── same ingress architecture
├── same observability
├── same security controls
├── same deployment mechanism
└── similar infrastructure topology
```

The main difference is generally:

```text
Scale
Data
Traffic
Cost
```

---

# 15. Production Promotion

Production promotion should be controlled.

Example:

```text
QA PASS
   ↓
STAGING PASS
   ↓
Production approval
   ↓
GitOps PR
   ↓
Review
   ↓
Merge
   ↓
ArgoCD
   ↓
EKS
```

This is much safer than:

```text
CI
 ↓
kubectl apply
 ↓
PROD
```

---

# 16. Who Approves Production?

A mature organization may have:

```text
Developer
    │
    ▼
QA approval
    │
    ▼
SRE / Platform review
    │
    ▼
Change approval
    │
    ▼
Production
```

The exact approval model depends on the company's change-management process.

For our project, we'll model:

```text
DEV → automatic
QA → controlled
STAGING → controlled
PROD → explicit approval
```

---

# 17. GitOps Repository

Our GitOps repository:

```text
gitops/
│
├── environments/
│   ├── dev/
│   │   └── order-service/
│   ├── qa/
│   │   └── order-service/
│   ├── staging/
│   │   └── order-service/
│   └── prod/
│       └── order-service/
│
├── applications/
└── argocd/
```

Each environment describes the desired deployment state.

---

# 18. DEV Values

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f3a21c9"

replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 250m
    memory: 512Mi
```

---

# 19. PROD Values

Production may have:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:ABC

replicaCount: 3

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

Notice:

```text
Image = same
```

but:

```text
replicas = different
resources = different
```

---

# 20. Configuration vs Artifact

This is a fundamental deployment principle.

### Artifact

```text
Java code
Spring Boot
dependencies
runtime
Docker image
```

### Configuration

```text
Database URL
Queue names
Redis endpoint
feature flags
replicas
resource limits
environment variables
```

The same artifact should be capable of running in multiple environments using environment-specific configuration.

---

# 21. Twelve-Factor Principle

A good application separates:

```text
Application code
```

from:

```text
Environment configuration
```

For example:

```yaml
spring:
  datasource:
    url: ${DB_URL}
```

The image doesn't contain:

```text
prod-database.internal
```

hardcoded into the application.

Instead Kubernetes supplies it.

---

# 22. Secrets

Never put:

```yaml
DB_PASSWORD: production-password
```

in the GitOps repository.

Instead:

```text
GitOps
  │
  └── reference to secret
             │
             ▼
       Secrets Manager
             │
             ▼
           EKS Pod
```

We'll cover this properly in the secrets-management chapter.

---

# 23. Image Promotion vs Rebuilding

### Wrong

```text
DEV:
build A

QA:
build B

PROD:
build C
```

### Correct

```text
Build A
   │
   ▼
ECR
   │
   ├── DEV
   ├── QA
   ├── STAGING
   └── PROD
```

This is the core principle of artifact promotion.

---

# 24. What If Production Needs a Different Configuration?

Suppose production needs:

```text
DB_URL = prod-db
```

We do **not** rebuild.

Instead:

```text
Same image
     │
     ├── DEV config
     ├── QA config
     ├── STAGING config
     └── PROD config
```

---

# 25. What If Production Needs a Different JVM Setting?

Same principle.

For example:

```text
JAVA_OPTS
```

can be environment-specific.

The image remains:

```text
order-service@sha256:ABC
```

while runtime configuration differs.

---

# 26. Promotion by Git Commit

Suppose DEV currently has:

```yaml
digest: sha256:OLD
```

CI updates:

```yaml
digest: sha256:ABC
```

Git commit:

```text
promote order-service to 8f3a21c9
```

ArgoCD sees the Git change.

```text
Git change
    ↓
ArgoCD detects difference
    ↓
Sync
    ↓
EKS
```

---

# 27. GitOps PR Promotion

For QA:

```text
CI / Release automation
       │
       ▼
GitOps branch
       │
       ▼
PR
       │
       ▼
Review
       │
       ▼
Merge
       │
       ▼
ArgoCD
```

This provides an audit trail.

---

# 28. Production GitOps PR

For production:

```text
Release candidate
      │
      ▼
PROD GitOps PR
      │
      ├── Image digest
      ├── Release version
      ├── Change summary
      ├── Test evidence
      └── Security status
      │
      ▼
Approvers
      │
      ▼
Merge
      │
      ▼
ArgoCD
```

This is a realistic production control.

---

# 29. Example Production PR

Title:

```text
Promote order-service to 1.4.2
```

Description:

```text
Application: order-service

Version: 1.4.2

Git commit:
8f3a21c9

Image digest:
sha256:ABC...

DEV:
PASS

QA:
PASS

STAGING:
PASS

Trivy:
PASS

SonarQube:
PASS

SBOM:
Available

Rollback image:
sha256:OLD...
```

This makes the release review much easier.

---

# 30. Rollback

Suppose:

```text
Current:
sha256:ABC
```

Previous:

```text
sha256:OLD
```

Production incident occurs.

Revert GitOps:

```text
sha256:ABC
      ↓
sha256:OLD
```

ArgoCD:

```text
Git
 ↓
ArgoCD
 ↓
EKS
```

The previous image is restored.

---

# 31. Rollback Without Rebuild

This is one of the biggest advantages.

Bad:

```text
Incident
 ↓
checkout old code
 ↓
build
 ↓
test
 ↓
scan
 ↓
push
 ↓
deploy
```

That takes time and introduces another build.

Better:

```text
Incident
 ↓
Known-good image already exists
 ↓
GitOps rollback
 ↓
ArgoCD
 ↓
EKS
```

The exact rollback process still needs appropriate operational checks, but the artifact is already available.

---

# 32. Emergency Deployment

Sometimes production has a critical incident requiring an emergency fix.

Example:

```text
Critical security vulnerability
```

Process:

```text
Fix code
 ↓
CI
 ↓
Tests
 ↓
Security scans
 ↓
Image
 ↓
ECR
 ↓
Emergency approval
 ↓
GitOps
 ↓
Production
```

Even emergency releases should retain the security and traceability controls as much as possible.

---

# 33. Promotion Metadata

A mature platform can maintain a release record:

```yaml
release:
  application: order-service
  version: 1.4.2
  gitCommit: 8f3a21c9
  imageDigest: sha256:ABC...
  sbom: available
  signature: verified

environments:
  dev:
    status: passed
  qa:
    status: passed
  staging:
    status: passed
  production:
    status: approved
```

This can be stored in release systems or generated automatically.

---

# 34. Image Tag Should Not Be Environment-Specific

Avoid:

```text
order-service:dev
order-service:qa
order-service:prod
```

because these tags can move.

Instead:

```text
order-service:8f3a21c9
```

and promote the same immutable artifact.

Environment state belongs in GitOps.

---

# 35. Bad Architecture

```text
                    Git
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        DEV CI      QA CI      PROD CI
          │          │          │
          ▼          ▼          ▼
       Build A    Build B     Build C
          │          │          │
          ▼          ▼          ▼
        ECR        ECR        ECR
```

Problems:

* repeated builds
* harder traceability
* potential differences
* slower releases
* more attack surface
* difficult rollback

---

# 36. Better Architecture

```text
                       Git
                        │
                        ▼
                       CI
                        │
                    Build ONCE
                        │
                        ▼
                    Security
                        │
                        ▼
                       ECR
                        │
                        ▼
                Immutable Digest
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
            DEV         QA       STAGING
                                  │
                                  ▼
                                PROD
```

---

# 37. Multi-Account Promotion

In a mature enterprise:

```text
DEV Account
    │
    ▼
DEV ECR
    │
    │ promotion
    ▼
PROD Account
    │
    ▼
PROD ECR
    │
    ▼
Production EKS
```

The promotion process should preserve the artifact identity.

For example:

```text
DEV ECR digest:
sha256:ABC

PROD ECR digest:
sha256:ABC
```

The digest should represent the same image contents.

---

# 38. Why Separate Production AWS Account?

Production account isolation gives us a stronger security boundary.

```text
Developer / CI
      │
      ▼
DEV account
```

Production:

```text
PROD account
      │
      ├── EKS
      ├── ECR
      ├── RDS
      └── Redis
```

Access is explicitly granted between environments.

---

# 39. CI Should Not Have Permanent Production Access

A mature model avoids:

```text
GitHub Actions
     ↓
AdministratorAccess
     ↓
Production
```

Instead:

```text
CI
 │
 ├── build
 ├── scan
 └── publish
       │
       ▼
approved artifact
```

Production deployment is controlled through GitOps and appropriate deployment identities.

---

# 40. Artifact Promotion Security

Before promotion, verify:

```text
[ ] Image exists
[ ] Image digest recorded
[ ] CI passed
[ ] Unit tests passed
[ ] SonarQube passed
[ ] Trivy passed
[ ] SBOM exists
[ ] Signature valid
[ ] DEV tests passed
[ ] QA tests passed
[ ] Required approval obtained
```

Then promote.

---

# 41. Deployment Strategies

Once we reach EKS, promotion can use different deployment strategies.

### Rolling

```text
Old pods
   ↓
New pods gradually
   ↓
Old pods removed
```

### Blue/Green

```text
Blue = current
Green = new

Traffic
   ↓
Green

Rollback
   ↓
Blue
```

### Canary

```text
100% old
   ↓
95% old / 5% new
   ↓
90% old / 10% new
   ↓
50% / 50%
   ↓
100% new
```

We'll cover these in later chapters.

---

# 42. Why Promotion Is Separate From Deployment Strategy

Promotion answers:

> **Should this artifact move to the next environment?**

Deployment strategy answers:

> **How should this artifact be introduced into that environment?**

For example:

```text
Promotion:
QA → PROD

Deployment:
Canary
```

These are different decisions.

---

# 43. Production Release Flow

Let's follow a real release.

Developer commits:

```text
8f3a21c9
```

CI:

```text
Build
 ↓
Test
 ↓
SonarQube
 ↓
Gitleaks
 ↓
Trivy
 ↓
Docker
 ↓
SBOM
 ↓
Sign
```

ECR:

```text
order-service@sha256:ABC
```

DEV:

```text
Deploy
 ↓
Smoke test
 ↓
PASS
```

QA:

```text
Integration
 ↓
Regression
 ↓
PASS
```

STAGING:

```text
Production-like validation
 ↓
PASS
```

PROD:

```text
Approval
 ↓
GitOps merge
 ↓
ArgoCD
 ↓
Canary/Rolling
 ↓
Monitoring
 ↓
SUCCESS
```

---

# 44. What If QA Fails?

Suppose:

```text
DEV = PASS
QA = FAIL
```

Do not promote.

```text
QA FAIL
   ↓
Stop promotion
   ↓
Investigate
   ↓
Fix
   ↓
New commit
   ↓
New image
```

You don't rebuild the same source separately for production.

---

# 45. What If Production Fails?

Suppose:

```text
DEV = PASS
QA = PASS
STAGING = PASS
PROD = FAIL
```

Then:

```text
Production incident
       ↓
Observe metrics/logs/traces
       ↓
Determine rollback
       ↓
GitOps rollback
       ↓
Previous digest
       ↓
ArgoCD
       ↓
EKS
```

Then investigate the failed release.

---

# 46. Release Identity

Every production release should be traceable to:

```text
Git commit
+
CI build
+
Image digest
+
SBOM
+
Security scan
+
GitOps commit
+
ArgoCD deployment
```

This is the complete software supply-chain chain.

---

# 47. Production Audit Question

Imagine an auditor asks:

> "Which version of order-service was running in production at 14:30 yesterday?"

We should be able to answer:

```text
Application:
order-service

Git commit:
8f3a21c9

Image:
order-service:1.4.2

Digest:
sha256:ABC...

GitOps commit:
7ab123...

ArgoCD application:
order-service-prod

Deployment:
2026-09-07 14:10

Security scan:
Passed
```

That is what proper traceability gives us.

---

# 48. Production Troubleshooting Question

Suppose the application is failing.

First determine:

```bash
kubectl get pods -n production
```

Then:

```bash
kubectl describe pod <pod-name> -n production
```

Then:

```bash
kubectl get pod <pod-name> \
  -n production \
  -o jsonpath='{.spec.containers[0].image}'
```

You should identify:

```text
Which image?
Which tag?
Which digest?
```

Then trace:

```text
Pod
 ↓
Image digest
 ↓
ECR
 ↓
GitOps
 ↓
Git commit
 ↓
CI build
```

This is extremely useful in production incidents.

---

# 49. Important Difference: Tag vs Digest vs Release

Keep these separate:

```text
Git SHA
```

identifies source.

```text
Image tag
```

identifies a convenient artifact label.

```text
Image digest
```

identifies exact image contents.

```text
Release version
```

communicates the business/software release.

Example:

```text
Git:
8f3a21c9

Release:
1.4.2

Image:
order-service:8f3a21c9

Digest:
sha256:ABC...
```

---

# 50. Recommended Model for Our Project

For our e-commerce platform:

```text
                    Git
                     │
                     ▼
              CI Build Once
                     │
                     ▼
             Security Gates
                     │
                     ▼
                    ECR
                     │
             ┌───────┴────────┐
             │                │
             ▼                ▼
       Image Tag          Image Digest
             │                │
             └───────┬────────┘
                     ▼
                  GitOps
                     │
             ┌───────┼────────┐
             ▼       ▼        ▼
            DEV      QA     STAGING
                              │
                         Approval
                              │
                              ▼
                             PROD
                              │
                              ▼
                            ArgoCD
                              │
                              ▼
                             EKS
```

---

# 51. Production Checklist

Before production promotion:

```text
SOURCE
[ ] PR approved
[ ] Git commit identified

CI
[ ] Build passed
[ ] Unit tests passed
[ ] Integration tests passed where applicable

QUALITY
[ ] SonarQube passed
[ ] Coverage policy passed

SECURITY
[ ] Gitleaks passed
[ ] Trivy filesystem passed
[ ] Trivy image passed
[ ] SBOM generated
[ ] Image signing/attestation passed

ARTIFACT
[ ] Image pushed to ECR
[ ] Immutable tag
[ ] Digest recorded
[ ] ECR scanning enabled

ENVIRONMENTS
[ ] DEV passed
[ ] QA passed
[ ] STAGING passed

PRODUCTION
[ ] Change approved
[ ] GitOps PR approved
[ ] Rollback image identified
[ ] Monitoring ready
[ ] Deployment executed
[ ] Post-deployment validation passed
```

---

# 52. The Core Production Principle

If you remember only one thing from this chapter:

```text
              BUILD ONCE
                  │
                  ▼
              TEST ONCE
                  │
                  ▼
             SCAN/SECURE
                  │
                  ▼
              STORE ONCE
                  │
                  ▼
           PROMOTE SAME IMAGE
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
      DEV         QA       STAGING
                              │
                              ▼
                             PROD
```

**Do not rebuild for every environment.**

The environment changes configuration.

The artifact should remain immutable.

---

# 53. Where We Are Now

We have established the complete path from source to production artifact:

```text
GitHub
   ↓
CI
   ↓
Maven
   ↓
Tests
   ↓
SonarQube
   ↓
Gitleaks
   ↓
Trivy
   ↓
Docker
   ↓
SBOM
   ↓
Image signing
   ↓
ECR
   ↓
Artifact promotion
```

The next major step is finally moving into **Kubernetes itself**.

# Next Chapter

**`18-Kubernetes-Fundamentals.md`**

There we'll build the Kubernetes mental model from a **production DevOps perspective**:

```text
Kubernetes Cluster
│
├── Control Plane
│
├── Worker Nodes
│
├── Namespaces
│
├── Pods
│
├── Deployments
│
├── ReplicaSets
│
├── Services
│
├── ConfigMaps
│
├── Secrets
│
├── ServiceAccounts
│
├── Ingress
│
├── HPA
│
├── PDB
│
└── NetworkPolicies
```

And, importantly, we'll connect each Kubernetes object to **what actually happens inside production EKS**, including scheduling, node selection, pod networking, service discovery, health probes, resource requests/limits, autoscaling, failures, and troubleshooting.
