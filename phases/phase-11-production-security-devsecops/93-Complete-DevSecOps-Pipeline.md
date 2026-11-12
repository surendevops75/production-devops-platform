# 93 — Complete DevSecOps Pipeline

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Services:** 21 microservices
> **Goal:** Build the complete production DevSecOps lifecycle from developer commit to a verified production deployment.

---

## 1. Objective

We have now built almost every individual component:

```text
GitHub
Terraform
AWS IAM
ECR
EKS
Helm
GitOps
ArgoCD
Jira Change Management
Trivy
SonarQube
Gitleaks
SBOM
Cosign
Kyverno
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
CloudTrail
GuardDuty
Security Hub
AWS Config
EventBridge
```

Now we connect them.

The objective is to understand the **entire production flow**, not just individual tools.

---

# 2. Complete DevSecOps Lifecycle

Our final lifecycle:

```text
Developer
    │
    ▼
Feature Branch
    │
    ▼
Pull Request
    │
    ▼
CI Validation
    │
    ├── Build
    ├── Unit Tests
    ├── SonarQube
    ├── Dependency Scan
    ├── Secret Scan
    └── IaC Scan
    │
    ▼
Code Review
    │
    ▼
Merge to Main
    │
    ▼
Build Immutable Image
    │
    ├── Trivy
    ├── SBOM
    └── Image Signing
    │
    ▼
ECR
    │
    ▼
GitOps Promotion
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
Jira Change
    │
    ▼
Production Approval
    │
    ▼
PROD
    │
    ▼
ArgoCD
    │
    ▼
EKS
    │
    ▼
Runtime Security + Observability
```

---

# 3. The Most Important Principle

We do **not** want:

```text
Developer
   │
   ▼
Build
   │
   ▼
kubectl apply production
```

Instead:

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

This provides separation between:

```text
BUILD
```

and:

```text
DEPLOY
```

---

# 4. Four Major Pipelines

Our project actually has four related pipelines.

### Pipeline 1 — Application CI

```text
Source Code
   ↓
Build
   ↓
Test
   ↓
Security Scan
   ↓
Container
   ↓
ECR
```

### Pipeline 2 — GitOps Deployment

```text
Image
   ↓
GitOps repository
   ↓
ArgoCD
   ↓
EKS
```

### Pipeline 3 — Infrastructure

```text
Terraform
   ↓
Plan
   ↓
Review
   ↓
Apply
   ↓
AWS
```

### Pipeline 4 — Security Monitoring

```text
AWS/EKS
   ↓
Security Signals
   ↓
Detection
   ↓
Alert
   ↓
Investigation
   ↓
Incident Response
```

---

# 5. Repository Architecture

We established 24 repositories:

```text
company-ecommerce/
│
├── 21 application repositories
│
├── infrastructure
│
├── gitops
│
└── platform
```

For example:

```text
company-ecommerce/
│
├── order-service
├── payment-service
├── inventory-service
├── catalog-service
├── user-service
├── ...
│
├── infrastructure
├── gitops
└── platform
```

---

# 6. Application Repository

Example:

```text
order-service/
│
├── src/
├── test/
├── helm/
│   └── order-service/
│
├── Dockerfile
├── pom.xml
│
├── .github/
│   └── workflows/
│       ├── ci.yaml
│       └── security.yaml
│
├── .gitignore
├── CODEOWNERS
├── SECURITY.md
└── README.md
```

The application repository owns:

```text
Application code
Tests
Dockerfile
Helm chart
Application CI
```

---

# 7. GitOps Repository

```text
gitops/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
├── applications/
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── appsets/
│
├── scripts/
└── .github/
```

It owns:

```text
Environment configuration
Image versions/digests
ArgoCD configuration
Promotion
```

---

# 8. Infrastructure Repository

```text
infrastructure/
│
├── terraform/
│   ├── modules/
│   └── environments/
│
└── .github/
    └── workflows/
        └── terraform.yaml
```

It owns:

```text
AWS infrastructure
IAM
VPC
EKS
ECR
RDS
Redis
SQS
SNS
KMS
Security services
```

---

# 9. Platform Repository

```text
platform/
│
├── monitoring/
├── logging/
├── security/
├── argocd/
├── ingress/
└── README.md
```

It owns shared platform components such as:

```text
Prometheus
Grafana
Fluent Bit
Logstash
Kyverno
Argo Rollouts
AWS Load Balancer Controller
```

---

# 10. Developer Workflow

A developer starts with:

```bash
git checkout main
git pull
git checkout -b feature/order-discount
```

They modify:

```text
src/
```

Then run:

```bash
./mvnw test
```

Then:

```bash
git status
git add .
git commit -m "feat: add order discount support"
git push origin feature/order-discount
```

---

# 11. Pull Request

Developer opens:

```text
feature/order-discount
        │
        ▼
Pull Request
        │
        ▼
main
```

The PR should trigger CI.

---

# 12. PR CI

The PR pipeline should validate the code but normally should **not publish a production image**.

Example:

```text
Pull Request
    │
    ├── Checkout
    ├── Build
    ├── Unit Test
    ├── SonarQube
    ├── Dependency Scan
    ├── Gitleaks
    ├── Docker Build
    └── Trivy
```

---

# 13. PR Quality Gate

We want:

```text
Build       → PASS
Tests       → PASS
Sonar       → PASS
Secrets     → PASS
Dependencies→ PASS
Trivy       → PASS
```

Only then should the PR become eligible for merge, subject to human review.

---

# 14. Why Scan Before Merge?

Imagine the developer accidentally commits:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

We want:

```text
GitHub PR
   │
   ▼
Gitleaks
   │
   ▼
BLOCK
```

rather than:

```text
GitHub
   │
   ▼
main
   │
   ▼
Production
```

---

# 15. SonarQube

SonarQube primarily evaluates source-code quality/security characteristics.

For our Java service:

```text
Java source
   │
   ▼
Sonar Scanner
   │
   ▼
SonarQube
   │
   ├── Bugs
   ├── Code smells
   ├── Vulnerabilities
   ├── Coverage
   └── Quality Gate
```

The scanner executes in the CI environment.

SonarQube itself is a centralized service, not something we install into every application pod.

---

# 16. Dependency Scanning

We also need to identify vulnerable dependencies.

For Java:

```text
pom.xml
   │
   ▼
Dependency Scanner
   │
   ▼
Known vulnerabilities
```

Examples include:

```text
Spring dependency
Jackson
Netty
Log4j
PostgreSQL driver
```

The exact tool can vary by organization.

---

# 17. Secret Scanning

Example:

```text
application.yaml
```

contains:

```yaml
password: SuperSecretPassword
```

Gitleaks should identify the issue.

Correct design:

```yaml
password: ${DB_PASSWORD}
```

with the runtime secret supplied through our Secrets Manager → External Secrets → Kubernetes Secret architecture.

---

# 18. Infrastructure Security Scanning

Terraform also needs security scanning.

Example:

```text
Terraform
   │
   ▼
IaC scanner
   │
   ├── Public S3 bucket
   ├── 0.0.0.0/0
   ├── Weak IAM
   └── Unencrypted resource
```

Possible tools include:

* Checkov
* tfsec-compatible tooling
* Trivy config scanning
* other approved IaC scanners

Use the organization's selected tool consistently.

---

# 19. Main Branch

After:

```text
CI PASS
+
Code Review
+
Approval
```

the PR is merged.

```text
feature/order-discount
          │
          ▼
         main
```

---

# 20. Main Build

Now we create the release artifact.

```text
main
 │
 ▼
Build
 │
 ▼
Test
 │
 ▼
Docker Build
 │
 ▼
Trivy
 │
 ▼
SBOM
 │
 ▼
Sign
 │
 ▼
ECR
```

---

# 21. Immutable Image Tag

Use:

```bash
IMAGE_TAG="${GITHUB_SHA}"
```

For example:

```text
8f31a2c...
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

Avoid:

```text
order-service:latest
```

---

# 22. Why Git SHA?

Because we want:

```text
Git commit
     ↓
Docker image
     ↓
ECR
     ↓
GitOps
     ↓
EKS
```

to be traceable.

If the image is:

```text
8f31a2c
```

we know which source revision produced it.

---

# 23. Docker Build

Example:

```bash
docker build \
  -t "$REGISTRY/order-service:$GITHUB_SHA" \
  .
```

Then:

```bash
docker images
```

---

# 24. Trivy Scan

Scan the exact image we will publish:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$REGISTRY/order-service:$GITHUB_SHA"
```

Important:

```text
Scan image A
     ≠
Deploy image B
```

We want:

```text
Build image
   ↓
Scan same image
   ↓
Push same image
   ↓
Deploy same image
```

---

# 25. SBOM

Generate:

```bash
trivy image \
  --format cyclonedx \
  --output sbom.json \
  "$REGISTRY/order-service:$GITHUB_SHA"
```

Conceptually:

```text
Image
 │
 ▼
SBOM
 │
 ├── OS packages
 ├── Java libraries
 ├── Application dependencies
 └── Versions
```

---

# 26. Image Signing

After successful scanning:

```text
Image
  │
  ▼
Cosign
  │
  ▼
Signature
```

Conceptually:

```text
Image digest
     │
     ▼
Signed artifact
```

Later Kyverno can enforce:

```text
Only trusted images
```

---

# 27. Push to ECR

```bash
docker push \
  "$REGISTRY/order-service:$GITHUB_SHA"
```

Then retrieve digest:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag="$GITHUB_SHA" \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Example:

```text
sha256:9a7b123...
```

---

# 28. The Real Artifact

The most important identifier is:

```text
order-service@sha256:9a7b123...
```

not merely:

```text
order-service:8f31a2c
```

The tag is useful for humans.

The digest identifies the exact image content.

---

# 29. CI Artifact Record

We should record:

```text
Application:
order-service

Git commit:
8f31a2c

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

Tag:
8f31a2c

Digest:
sha256:9a7b123...

SBOM:
sbom.json

Scan:
PASS

Signature:
VALID
```

---

# 30. GitOps Update

CI now updates GitOps.

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

CI does **not** execute:

```bash
kubectl apply
```

against production.

---

# 31. GitOps Pull Request

The CI process creates:

```text
automation/order-service/8f31a2c
```

Then:

```text
GitOps PR
```

Example:

```text
promote: order-service 8f31a2c to dev
```

PR contains:

```text
Application: order-service
Environment: dev
Image tag: 8f31a2c
Digest: sha256:9a7b123...
Source commit: 8f31a2c
Trivy: PASS
SBOM: generated
```

---

# 32. ArgoCD

After the GitOps PR is merged:

```text
GitOps
   │
   ▼
ArgoCD
   │
   ▼
ApplicationSet
   │
   ▼
Helm
   │
   ▼
Kubernetes
```

ArgoCD sees the desired state.

---

# 33. Kyverno

Before the workload becomes accepted:

```text
ArgoCD
   │
   ▼
Kubernetes API
   │
   ▼
Kyverno
```

Policies can verify:

```text
Non-root
Resources defined
Trusted registry
No latest
SecurityContext
Required labels
Image signature
```

---

# 34. DEV Deployment

```text
ECR
 │
 ▼
GitOps
 │
 ▼
ArgoCD
 │
 ▼
DEV EKS
```

Verify:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-dev
```

---

# 35. Runtime Verification

Check image:

```bash
kubectl get deployment order-service \
  -n ecommerce-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check actual image:

```bash
kubectl get pod <pod> \
  -n ecommerce-dev \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

The second command is particularly useful for verifying the actual image pulled by the node.

---

# 36. Health Validation

Check:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Application health:

```text
/actuator/health
```

Metrics:

```text
/actuator/prometheus
```

---

# 37. DEV Observability

After deployment:

```text
order-service
     │
 ┌───┼──────────────┐
 ▼   ▼              ▼
Logs Metrics       Traces
 │    │              │
 ▼    ▼              ▼
ELK Prometheus     Jaeger
      │
      ▼
    Grafana
```

---

# 38. DEV Security Monitoring

At the same time:

```text
EKS
 │
 ├── CloudTrail
 ├── EKS Audit
 ├── Flow Logs
 └── GuardDuty
```

Security monitoring continues independently of application deployment.

---

# 39. QA Promotion

Once DEV validation passes:

```text
DEV
 │
 ▼
QA Promotion PR
```

But we do **not rebuild**.

The exact same digest moves:

```text
DEV
sha256:9a7b123

   ↓

QA
sha256:9a7b123
```

---

# 40. QA Validation

QA may perform:

```text
Unit/integration tests
API tests
Database tests
Messaging tests
Security tests
Regression tests
```

The environment changes.

The artifact does not.

---

# 41. STAGING

After QA:

```text
QA
 │
 ▼
STAGING
```

Again:

```text
same digest
```

Staging should resemble production sufficiently to validate important behavior.

Examples:

```text
same Kubernetes architecture
similar resource configuration
same ingress architecture
same security policies
same deployment mechanism
```

---

# 42. Production Gate

Before production:

```text
STAGING PASS
      │
      ▼
Jira Change Request
      │
      ▼
Production Review
      │
      ▼
Approval
```

Example:

```text
CHG-10452
```

Contains:

```text
Service
Image tag
Image digest
Risk
Implementation plan
Validation
Rollback
Change window
Approvers
```

---

# 43. Production GitOps PR

Now:

```text
staging
   │
   ▼
prod promotion PR
```

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."
```

---

# 44. Production Approval

Production PR requires appropriate reviewers.

For example:

```text
Backend Team
      │
      ▼
Platform Team
      │
      ▼
DevOps / SRE
      │
      ▼
Production Approver
```

The exact approval model depends on organizational policy.

---

# 45. ArgoCD Production Deployment

After GitOps merge:

```text
GitOps
   │
   ▼
ArgoCD
   │
   ▼
Production Application
```

Production can remain manually synchronized.

```bash
argocd app sync order-service-prod
```

Before synchronization:

```bash
argocd app diff order-service-prod
```

Review the change.

---

# 46. Rolling Deployment

Our default strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Example:

```text
v1 v1 v1
 │  │  │
 ▼  ▼  ▼
v2 v1 v1
 │  │  │
 ▼  ▼  ▼
v2 v2 v1
 │  │  │
 ▼  ▼  ▼
v2 v2 v2
```

---

# 47. Readiness Protection

A new pod should not receive traffic until ready.

```text
New Pod
   │
   ▼
Startup Probe
   │
   ▼
Readiness Probe
   │
   ├── FAIL → no traffic
   │
   └── PASS → traffic
```

This is one of the foundations of zero-downtime deployment.

---

# 48. Production Verification

After deployment:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service
```

Then:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

# 49. Verify Actual Digest

```bash
kubectl get pod <pod-name> \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Expected:

```text
...@sha256:9a7b123...
```

Now we know:

```text
CI artifact
   =
ECR artifact
   =
GitOps artifact
   =
Running container
```

---

# 50. Production Smoke Test

Example:

```bash
curl -f \
  https://api.example.com/api/v1/orders/12345
```

Then verify:

```text
HTTP status
Response time
Application logs
Error rate
Database connectivity
Dependencies
```

---

# 51. Production Metrics

Immediately watch:

```text
Request rate
Error rate
Latency
CPU
Memory
Pod restarts
HPA
RDS
Redis
SQS
```

Grafana dashboards:

```text
Service Dashboard
      │
      ├── RED
      ├── Resources
      ├── Dependencies
      └── SLO
```

---

# 52. Production Logs

Kibana:

```text
service.name : "order-service"
AND
environment : "prod"
AND
@timestamp >= "deployment time"
```

Look for:

```text
ERROR
Exception
Timeout
Connection refused
Database errors
Authentication failures
```

---

# 53. Production Traces

OpenTelemetry/Jaeger helps investigate:

```text
Client
  ↓
ALB
  ↓
order-service
  ↓
inventory-service
  ↓
payment-service
  ↓
RDS
```

If latency increases, trace the dependency chain.

---

# 54. Security Verification

Also check:

```text
GuardDuty
Security Hub
CloudTrail
EKS Audit
VPC Flow Logs
```

Why?

Because deployment should not only be operationally healthy.

It should also remain within security expectations.

---

# 55. Deployment Complete

A production release is complete only when:

```text
Application
   ├── Healthy
   ├── Metrics normal
   ├── Logs normal
   ├── Traces normal
   ├── Security checks normal
   └── Business smoke tests pass
```

Then:

```text
Jira Change
   ↓
Successful
   ↓
Close
```

---

# 56. Rollback

If deployment fails:

```text
Production
    │
    ▼
Problem detected
    │
    ▼
Stop / assess
    │
    ▼
Rollback
```

Preferred normal GitOps rollback:

```text
GitOps
   │
   ▼
Revert image change
   │
   ▼
PR
   │
   ▼
Merge
   │
   ▼
ArgoCD
```

---

# 57. Emergency Rollback

During a severe incident, an emergency operational rollback may be required.

For example:

```bash
kubectl rollout undo \
  deployment/order-service \
  -n ecommerce-prod
```

But afterward:

```text
Kubernetes state
      ↓
GitOps reconciliation
      ↓
Git desired state
```

must be brought back into alignment.

A manual `kubectl` change should not become a permanent undocumented state.

---

# 58. Database Compatibility

This is one of the biggest DevSecOps production concerns.

Suppose version 2 requires:

```text
new database column
```

Do not immediately remove the old column.

Use:

```text
Expand
   ↓
Deploy compatible application
   ↓
Migrate data
   ↓
Switch application behavior
   ↓
Contract
```

This allows old and new application versions to coexist during deployment.

---

# 59. Messaging Compatibility

The same principle applies to:

```text
SQS
Kafka
RabbitMQ
```

Suppose v1 publishes:

```json
{
  "orderId": "123"
}
```

and v2 expects:

```json
{
  "orderId": "123",
  "customerId": "456"
}
```

A rolling deployment can temporarily have:

```text
v1 producers
+
v2 consumers
```

Therefore event schemas should be backward compatible during transition.

---

# 60. Secrets During Deployment

Production pods should not receive secrets from:

```text
GitHub
GitOps
Helm values
Docker image
```

Instead:

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
Pod
```

IAM:

```text
Pod
 │
 ▼
EKS Pod Identity
 │
 ▼
IAM Role
 │
 ▼
Secrets Manager
```

---

# 61. Complete Artifact Chain

This is one of the most important diagrams in the entire project.

```text
Developer
   │
   ▼
Git Commit
   │
   ▼
GitHub PR
   │
   ▼
CI
   │
   ├── Tests
   ├── SonarQube
   ├── Gitleaks
   ├── Dependency Scan
   ├── Trivy
   └── SBOM
   │
   ▼
Docker Image
   │
   ▼
Image Digest
   │
   ├── Signed
   │
   ▼
ECR
   │
   ▼
GitOps PR
   │
   ▼
Jira Change
   │
   ▼
ArgoCD
   │
   ▼
Kyverno
   │
   ▼
EKS
   │
   ▼
Running Pod
```

At any point we should be able to trace backward.

---

# 62. Traceability Example

Suppose production has:

```text
order-service
image digest:
sha256:9a7b123...
```

We should be able to determine:

```text
Which ECR image?
        ↓
Which Git SHA?
        ↓
Which GitHub PR?
        ↓
Which developer change?
        ↓
Which GitOps commit?
        ↓
Which Jira change?
        ↓
Which ArgoCD sync?
        ↓
Which Kubernetes ReplicaSet?
```

This is production traceability.

---

# 63. DevSecOps Pipeline — Full Diagram

```text
                         DEVELOPER
                             │
                             ▼
                     Feature Branch
                             │
                             ▼
                       Pull Request
                             │
                             ▼
                    ┌────────────────┐
                    │      CI        │
                    │                │
                    │ Build          │
                    │ Unit Tests     │
                    │ SonarQube      │
                    │ Gitleaks       │
                    │ Dependency     │
                    │ IaC Scan       │
                    │ Trivy          │
                    └───────┬────────┘
                            │
                         PASS
                            │
                            ▼
                     Code Review
                            │
                            ▼
                       main branch
                            │
                            ▼
                    Docker Build
                            │
             ┌──────────────┼─────────────┐
             │              │             │
           Trivy           SBOM         Cosign
             │              │             │
             └──────────────┼─────────────┘
                            │
                            ▼
                           ECR
                            │
                            ▼
                     Image Digest
                            │
                            ▼
                       GitOps PR
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
                       Jira Change
                            │
                            ▼
                    Production Approval
                            │
                            ▼
                          PROD
                            │
                            ▼
                         ArgoCD
                            │
                            ▼
                           EKS
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
    Prometheus             ELK                  OTel
        │                   │                    │
      Grafana             Kibana               Jaeger
        │                   │                    │
        └───────────────────┼────────────────────┘
                            │
                       SECURITY
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
    CloudTrail          GuardDuty             Config
        │                   │                    │
        └──────────── Security Hub ──────────────┘
                            │
                       EventBridge
                            │
                            ▼
                    Incident Response
```

---

# 64. What Happens When Something Fails?

A mature DevSecOps engineer doesn't simply know commands.

They know **which layer to investigate**.

---

## Code failure

```text
PR
 ↓
CI
 ↓
Test / Sonar / Gitleaks
```

Investigate source code.

---

## Build failure

```text
CI
 ↓
Maven
 ↓
Docker
```

Investigate dependencies/build environment/Dockerfile.

---

## Security scan failure

```text
Trivy
 ↓
Vulnerability
```

Investigate:

```text
package
version
CVSS
exploitability
base image
upgrade
exception process
```

---

## ECR failure

```text
CI
 ↓
AWS OIDC
 ↓
ECR
```

Investigate:

```text
IAM
OIDC
repository
region
permissions
```

---

## GitOps failure

```text
GitOps PR
 ↓
ArgoCD
```

Investigate:

```text
YAML
Helm
repository
ApplicationSet
permissions
```

---

## Kubernetes failure

```text
ArgoCD
 ↓
Kubernetes
 ↓
Pod
```

Investigate:

```text
Events
Logs
Probes
Resources
Image
Service
Network
Secrets
```

---

## Application failure

```text
Pod
 ↓
Application
 ↓
Database / Redis / Queue
```

Investigate dependencies.

---

## Security incident

```text
AWS / EKS
 ↓
CloudTrail / GuardDuty / Security Hub
 ↓
EventBridge
 ↓
Incident Response
```

Investigate security telemetry.

---

# 65. Production CI/CD Permissions

This is extremely important.

### GitHub Actions

Should generally have:

```text
OIDC
 ↓
CI IAM Role
 ↓
ECR Push
```

Not:

```text
GitHub
 ↓
AdministratorAccess
```

---

### ArgoCD

ArgoCD should have:

```text
GitOps
 ↓
ArgoCD
 ↓
Kubernetes permissions
```

not AWS administrator credentials for normal application deployment.

---

### Application Pod

```text
Pod
 ↓
Pod Identity
 ↓
IAM role
 ↓
Specific AWS API permissions
```

---

# 66. Separation of Duties

We have:

```text
Developer
    │
    └── Code

CI
    │
    └── Build + security validation

Platform
    │
    └── Kubernetes platform

DevOps
    │
    └── Infrastructure

Security
    │
    └── Security controls/monitoring

SRE
    │
    └── Reliability/operations

Change Approver
    │
    └── Production change authorization
```

The exact organizational responsibilities vary, but the principle is:

> No single pipeline identity should automatically control every layer.

---

# 67. Security Gates

Our pipeline now has multiple security gates.

```text
Gate 1
Source security
   ↓
Gitleaks

Gate 2
Code quality/security
   ↓
SonarQube

Gate 3
Dependencies
   ↓
Dependency scanning

Gate 4
Container
   ↓
Trivy

Gate 5
Artifact
   ↓
SBOM + signing

Gate 6
Kubernetes
   ↓
Kyverno

Gate 7
Runtime
   ↓
GuardDuty / Security Hub
```

This is why the term **DevSecOps** is appropriate.

Security isn't a single final scan.

---

# 68. Continuous Security

Security continues after deployment.

```text
Build-time
   ↓
Deploy-time
   ↓
Runtime
   ↓
Detection
   ↓
Incident
   ↓
Remediation
```

For example:

```text
New CVE discovered
       │
       ▼
Existing image becomes vulnerable
       │
       ▼
Security monitoring / vulnerability management
       │
       ▼
Create remediation work
       │
       ▼
Update dependency/base image
       │
       ▼
New image
       │
       ▼
Same DevSecOps pipeline
```

---

# 69. Production Readiness Gate

Before declaring a service production-ready:

```text
Application
[ ] Tests pass

Container
[ ] Non-root
[ ] Image scanned
[ ] SBOM
[ ] Signed

Infrastructure
[ ] Terraform
[ ] IAM least privilege
[ ] Network security

Kubernetes
[ ] Probes
[ ] Resources
[ ] PDB
[ ] NetworkPolicy
[ ] Kyverno

Secrets
[ ] Secrets Manager
[ ] Pod Identity
[ ] Rotation

Deployment
[ ] GitOps
[ ] ArgoCD
[ ] Rollback

Observability
[ ] Metrics
[ ] Logs
[ ] Traces
[ ] Alerts

Security
[ ] CloudTrail
[ ] GuardDuty
[ ] Security Hub
[ ] EKS audit

Operations
[ ] Runbook
[ ] Jira change
[ ] Incident procedure
[ ] DR plan
```

---

# 70. Final Production Model

At this stage, our platform is no longer:

```text
"21 containers running on EKS"
```

It is:

```text
                         ENGINEERING
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 GitHub              Jira
                    │                   │
                    ▼                   │
                   CI                    │
                    │                   │
          ┌─────────┼─────────┐         │
          ▼         ▼         ▼         │
       Quality   Security   Build       │
          │         │         │         │
          └─────────┼─────────┘         │
                    ▼                   │
                   ECR                  │
                    │                   │
                    ▼                   │
                 GitOps ◄───────────────┘
                    │
                    ▼
                 ArgoCD
                    │
                    ▼
                 Kyverno
                    │
                    ▼
                   EKS
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
   Application    Data       Messaging
        │
        ▼
   Observability
        │
 ┌──────┼──────────────┐
 ▼      ▼              ▼
Metrics Logs          Traces
 │      │              │
 ▼      ▼              ▼
Prom   ELK            OTel
 │      │              │
 ▼      ▼              ▼
Grafana Kibana       Jaeger
        │
        ▼
     SECURITY
        │
 ┌──────┼──────────────────────┐
 ▼      ▼          ▼           ▼
Trail GuardDuty   Config   Security Hub
        │
        ▼
   EventBridge
        │
        ▼
   Incident Response
```

---

# 71. The One Flow You Should Be Able to Explain in an Interview

If someone asks:

> **"Explain your production DevSecOps deployment process."**

You should be able to explain it like this:

> A developer creates a feature branch and raises a pull request. CI runs compilation, unit tests, SonarQube analysis, dependency scanning, secret scanning and IaC checks. After review and merge, we build the container once, scan the same image with Trivy, generate an SBOM, sign the image and push it to ECR using GitHub OIDC rather than static AWS credentials.
>
> We capture the ECR image digest and update the GitOps repository. ArgoCD watches the GitOps repository and deploys the immutable image to EKS. Kyverno enforces our Kubernetes security policies before the workload is admitted.
>
> We promote the same image digest through DEV, QA and STAGING. Before production, we create and approve the Jira change request and production GitOps PR. ArgoCD then performs the production deployment, normally using a controlled rolling strategy.
>
> After deployment, we verify the rollout, image digest, application health, metrics, logs, traces and business smoke tests. At runtime, CloudTrail, GuardDuty, Security Hub, Config, VPC Flow Logs and EKS audit logs provide security monitoring. Prometheus, Grafana, ELK and OpenTelemetry provide operational observability. If something goes wrong, we use GitOps rollback and our incident-response process.

That is the complete story.

---

# 72. Chapter 93 Completion

We have now connected:

```text
SOURCE
  ↓
CI
  ↓
SECURITY
  ↓
CONTAINER
  ↓
ECR
  ↓
GITOPS
  ↓
ARGOCD
  ↓
KYVERNO
  ↓
EKS
  ↓
OBSERVABILITY
  ↓
SECURITY MONITORING
  ↓
INCIDENT RESPONSE
```

The next major phase is **Kubernetes Cluster Operations**: how we actually operate the EKS cluster day-to-day—nodes, upgrades, node groups, Karpenter, Cluster Autoscaler, capacity planning, drain/cordon, PDB interactions, EKS upgrades, add-on upgrades, node failures, AZ failures, and production cluster troubleshooting.

### Next file

```text
94-EKS-Cluster-Operations.md
```

That chapter will move from **building the platform** to **operating the platform like a production DevOps/SRE team**.
