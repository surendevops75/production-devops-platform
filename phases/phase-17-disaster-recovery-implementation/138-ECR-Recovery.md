# 138 — ECR RECOVERY

## 1. Objective

This chapter covers **production Amazon ECR recovery** for our 21-microservice e-commerce platform.

ECR is the source of immutable container artifacts used by EKS:

```text
Developer
   ↓
GitHub
   ↓
CI
   ↓
Docker Build
   ↓
ECR
   ↓
ArgoCD
   ↓
EKS
   ↓
21 Microservices
```

If ECR becomes unavailable, images are deleted, tags are corrupted, or the application cannot pull images, the Kubernetes workloads may fail even though the EKS cluster itself is healthy.

The goal is to recover:

```text
Container images
Image tags/digests
Repository configuration
Cross-region availability
IAM access
Encryption
Scanning configuration
Lifecycle policies
```

---

# 2. Our ECR Architecture

We use one repository per microservice.

```text
ECR
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

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

---

# 3. ECR Is an Artifact Store

ECR should contain the immutable application artifact.

Example:

```text
Git commit
    |
    v
Docker image
    |
    v
ECR
    |
    v
image digest
```

Example:

```text
order-service:1.8.4
```

may resolve to:

```text
sha256:abc123...
```

The digest identifies the exact image content.

---

# 4. Why Image Digests Matter

Suppose:

```text
order-service:1.8.4
```

was pushed today.

Later someone pushes another image using the same tag.

Now:

```text
1.8.4
```

could point to different content.

This is why production deployments should preferably use immutable tags and/or digests.

Our desired model:

```text
Git SHA
   ↓
Image
   ↓
Immutable ECR tag
   ↓
Digest
   ↓
GitOps
   ↓
EKS
```

---

# 5. ECR Failure Scenarios

We will cover:

```text
1. ECR repository unavailable
2. Repository accidentally deleted
3. Image accidentally deleted
4. Wrong image tag
5. Image digest mismatch
6. ECR IAM AccessDenied
7. EKS ImagePullBackOff
8. ECR region failure
9. ECR replication failure
10. KMS/encryption problem
11. Lifecycle policy mistake
12. Cross-account pull failure
13. Complete ECR recovery
```

---

# 6. First Recovery Rule

If Pods report:

```text
ImagePullBackOff
```

do not immediately conclude:

```text
ECR is down
```

Possible causes include:

```text
ECR repository missing
Image tag missing
Wrong image name
IAM problem
Network problem
ECR authentication problem
KMS problem
Region mismatch
Registry endpoint issue
Image architecture mismatch
```

---

# 7. ImagePullBackOff Investigation

Check:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl describe pod \
  <pod-name> \
  -n ecommerce-prod
```

Look at Events:

```text
Failed to pull image
pull access denied
repository does not exist
manifest unknown
no basic auth credentials
```

The exact error is critical.

---

# 8. Example: Repository Does Not Exist

Suppose:

```text
order-service
```

is configured as:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

but the repository was deleted.

Kubernetes may show:

```text
repository does not exist
```

Recovery:

```text
Confirm repository
        ↓
Recreate ECR repository
        ↓
Restore image
        ↓
Validate permissions
        ↓
Restart/reconcile workload
```

But recreating the repository alone does not restore the image.

---

# 9. Repository Configuration

Our ECR Terraform module:

```text
infrastructure/
└── terraform/
    └── modules/
        └── ecr/
            ├── main.tf
            ├── variables.tf
            └── outputs.tf
```

Example:

```hcl
resource "aws_ecr_repository" "this" {
  name                 = var.repository_name
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Service     = var.repository_name
    Environment = var.environment
  }
}
```

---

# 10. Terraform Can Recreate the Repository

Suppose:

```text
order-service ECR
```

was accidentally deleted.

Terraform can recreate:

```text
Repository
Encryption configuration
Tag mutability
Scan configuration
Tags
```

But Terraform cannot recreate:

```text
order-service image layers
```

unless those images exist somewhere else.

This distinction is critical.

---

# 11. Infrastructure vs Artifact

Think of ECR as two layers:

```text
ECR
│
├── Repository configuration
│
└── Container artifacts
```

Terraform manages primarily:

```text
Repository configuration
```

CI manages:

```text
Container artifacts
```

Therefore:

```text
Terraform Recovery
      +
Artifact Recovery
```

are both required.

---

# 12. ECR Repository Recovery

First inspect:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

If it exists:

```text
Repository available
```

If not:

```text
Repository missing
```

Then recover the infrastructure through the approved Terraform workflow.

---

# 13. List Images

Check available images:

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

For more detail:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

You should inspect:

```text
imageTags
imageDigest
imagePushedAt
imageSizeInBytes
```

---

# 14. Image Digest

Example:

```text
order-service:1.8.4
```

Digest:

```text
sha256:abc123...
```

Production GitOps can reference:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

This ensures the deployment refers to exact image content.

---

# 15. Image Tag vs Digest

### Tag

```text
order-service:1.8.4
```

Human-friendly.

### Digest

```text
order-service@sha256:abc123...
```

Content-specific.

A mature deployment model can use:

```text
Version tag
+
Digest pinning
```

---

# 16. Image Recovery Sources

If an image disappears from ECR, possible sources include:

```text
Primary ECR
     |
     +--> DR ECR
     |
     +--> CI rebuild from Git
     |
     +--> Artifact retention system
```

The preferred recovery source depends on whether the original image must be preserved byte-for-byte.

---

# 17. Rebuilding an Image From Git

Suppose:

```text
ECR image deleted
```

but Git still contains:

```text
Git commit abc123
```

and:

```text
Dockerfile
source code
dependencies
```

We can rebuild:

```text
Git commit
    ↓
CI
    ↓
Docker build
    ↓
ECR
```

However, rebuilding later may not always produce a byte-for-byte identical image unless the build is fully reproducible.

Therefore:

> For strict artifact preservation, retain and replicate the original image rather than relying solely on rebuilding from source.

---

# 18. Build Once, Promote Many

Our CI/CD principle:

```text
Build
 ↓
Scan
 ↓
Push to ECR
 ↓
Promote same image
```

Not:

```text
DEV build
 ↓
QA build
 ↓
STAGING build
 ↓
PROD build
```

Instead:

```text
One artifact
     |
     +--> DEV
     |
     +--> QA
     |
     +--> STAGING
     |
     +--> PROD
```

This makes ECR recovery and release traceability easier.

---

# 19. ECR Cross-Region Replication

For regional DR:

```text
Primary:
ap-south-1
     |
     | ECR replication
     v
DR:
ap-southeast-1
```

Repositories/images can be replicated according to configured ECR replication rules.

Architecture:

```text
ECR Primary
     |
     v
ECR DR
     |
     v
DR EKS
```

---

# 20. Why Cross-Region ECR Matters

Suppose:

```text
ap-south-1
```

has a major outage.

If the DR EKS cluster is in:

```text
ap-southeast-1
```

and its Pods reference only:

```text
ap-south-1 ECR
```

the DR cluster may not be able to pull the required images.

Therefore:

```text
DR EKS
   ↓
DR ECR
```

is preferable for regional independence.

---

# 21. ECR DR Architecture

```text
                GitHub
                   |
                  CI
                   |
          +--------+--------+
          |                 |
      ECR Primary       ECR DR
     ap-south-1      ap-southeast-1
          |                 |
         EKS               EKS
       Primary              DR
```

The GitOps configuration should select the appropriate regional registry for each environment.

---

# 22. ECR Replication Does Not Replace CI

ECR replication provides artifact availability.

CI remains responsible for:

```text
Build
Test
Security scan
Push
Metadata
Release
```

So:

```text
CI
 |
 v
Primary ECR
 |
 v
DR ECR
```

is the intended model.

---

# 23. ECR IAM

For CI:

```text
GitHub Actions
       |
       v
OIDC
       |
       v
IAM Role
       |
       v
ECR Push
```

The CI role should have only the required ECR permissions.

Typical permissions include:

```text
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
```

---

# 24. EKS Pull Permissions

Runtime architecture:

```text
EKS Node / Pod
       |
       v
ECR
       |
       v
Image
```

The exact IAM design depends on the EKS configuration and whether image pulls are authorized through the node role or another supported mechanism.

For standard EKS managed-node architectures, the node's AWS identity must be able to retrieve the required image layers.

---

# 25. ECR AccessDenied

Example:

```text
ImagePullBackOff
```

Events:

```text
pull access denied
```

Investigate:

```text
Node IAM role
Pod identity architecture
ECR repository policy
AWS account
Region
Repository name
Image tag
```

Do not immediately modify IAM to:

```text
AdministratorAccess
```

---

# 26. Cross-Account ECR

Enterprise architecture may have:

```text
Account A
Production EKS

Account B
Central ECR
```

Architecture:

```text
Production EKS
      |
      v
Central ECR
```

Then repository policies and IAM must explicitly permit the production workload environment to pull images.

This is common in centralized platform models.

---

# 27. Cross-Account Recovery

During recovery verify:

```text
Account ID
Repository ARN
Repository policy
IAM role
ECR registry
Region
KMS
```

A common mistake is:

```text
DR EKS
  |
  v
Wrong AWS account
  |
  v
AccessDenied
```

---

# 28. ECR Authentication

For manual diagnostics:

```bash
aws ecr get-login-password \
  --region ap-south-1
```

can be used with Docker authentication where appropriate.

Example:

```bash
aws ecr get-login-password \
  --region ap-south-1 | \
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

This is useful for troubleshooting.

It should not become the normal production deployment mechanism when GitOps and Kubernetes already handle image pulls.

---

# 29. ECR ImagePullBackOff Investigation

Use:

```bash
kubectl describe pod \
  <pod-name> \
  -n ecommerce-prod
```

Possible messages:

### Repository missing

```text
repository does not exist
```

### Tag missing

```text
manifest unknown
```

### Permission

```text
pull access denied
```

### Network

```text
i/o timeout
```

Each requires a different recovery path.

---

# 30. ECR Network Path

Private EKS nodes can access AWS services through appropriate VPC networking.

Depending on the architecture, verify:

```text
VPC
Private subnet
Route
NAT or relevant VPC endpoints
Security controls
DNS
AWS service connectivity
```

A private cluster does not automatically mean every AWS service endpoint is reachable without deliberate network configuration.

---

# 31. ECR and VPC Endpoints

For private environments, AWS PrivateLink interface endpoints can be used for relevant ECR API/Docker registry access patterns.

The exact endpoint set depends on the architecture and AWS service behavior.

Do not blindly create every possible VPC endpoint.

Instead:

```text
Workload requirement
      ↓
Required AWS service
      ↓
Required endpoint
```

---

# 32. ECR Encryption

Repositories can use:

```text
SSE-S3
```

or:

```text
SSE-KMS
```

For KMS-backed repositories:

```text
ECR
 |
 v
KMS
```

Recovery must verify:

```text
KMS key exists
Key enabled
IAM permissions
Key policy
Region
Cross-account access if applicable
```

---

# 33. KMS Failure

Suppose:

```text
ECR repository
```

exists but access fails due to KMS permissions.

Symptoms may look like:

```text
AccessDenied
```

Investigation:

```text
ECR
 ↓
KMS
 ↓
IAM
 ↓
Key policy
```

Do not rotate/delete encryption keys during an incident without understanding the impact on existing encrypted artifacts.

---

# 34. ECR Lifecycle Policies

Lifecycle policies can remove old images.

Example:

```text
Keep last 30 production images
Delete older untagged images
```

But lifecycle rules must account for rollback requirements.

Suppose:

```text
Current:
1.8.4

Previous:
1.8.3
1.8.2
1.8.1
```

If all older images are immediately removed:

```text
Rollback capability
        ↓
Lost
```

Therefore retention should be based on:

```text
Rollback window
Deployment frequency
Compliance
Storage cost
DR strategy
```

---

# 35. Untagged Images

Docker image layers can become untagged after tag deletion or manifest changes.

Example:

```text
Image
 |
 +--> tag: 1.8.4
 |
tag removed
 |
 v
Untagged
```

Lifecycle policies often clean these up.

But during incident recovery, don't delete untagged images blindly.

First determine whether they are required for:

```text
Rollback
Recovery
Replication
Forensics
```

---

# 36. ECR Repository Recovery Procedure

```text
Incident
   ↓
Identify repository
   ↓
Check repository existence
   ↓
Check image tags
   ↓
Check image digests
   ↓
Check replication
   ↓
Check CI/artifact source
   ↓
Restore repository configuration
   ↓
Restore image
   ↓
Validate IAM
   ↓
Validate EKS pull
```

---

# 37. Recover Repository With Terraform

Example:

```bash
cd infrastructure/terraform/environments/prod
```

Then:

```bash
terraform plan
```

Review:

```text
Repository creation
Encryption
Tag mutability
Scanning
Lifecycle
Replication
Tags
```

Then apply through the approved production change process.

---

# 38. Restore Image From DR ECR

Suppose:

```text
Primary:
ap-south-1

DR:
ap-southeast-1
```

Primary image:

```text
order-service:1.8.4
```

If the primary registry is unavailable but DR replication has completed:

```text
DR ECR
   |
   v
order-service:1.8.4
```

The DR GitOps configuration can point to the DR registry.

---

# 39. Image Digest Verification

Before using a recovered image:

```text
Primary digest:
sha256:AAA...
```

Compare:

```text
DR digest:
sha256:AAA...
```

If they match:

```text
Same image content
```

If they don't:

```text
Different image content
```

Investigate before deployment.

---

# 40. Image Provenance

Each production image should be traceable to:

```text
Git repository
Commit SHA
Build workflow
Build number
Image tag
Image digest
Security scan
Deployment
```

Example:

```text
order-service
Version: 1.8.4
Commit: abc123
Image digest: sha256:...
Build: GitHub Actions #842
Environment: production
```

This makes recovery much easier.

---

# 41. ECR and CI Recovery

Suppose ECR is lost but GitHub remains available.

Recovery:

```text
GitHub
   |
   v
CI workflow
   |
   +--> Build
   +--> Test
   +--> Scan
   +--> Push
   |
   v
New ECR
```

The CI pipeline should be capable of recreating the artifact repository workflow.

---

# 42. CI Must Not Depend on Production EKS

Our architecture:

```text
CI
 |
 +--> Build
 +--> Test
 +--> Scan
 +--> ECR
```

Production EKS is not required to build the application.

This is important during disaster recovery.

If:

```text
EKS = unavailable
```

CI can still:

```text
Build image
Push image
Prepare release
```

---

# 43. ECR and GitOps Recovery

Suppose:

```text
ECR = recovered
```

but:

```text
GitOps = points to old digest
```

Then ArgoCD may continue deploying the wrong image.

Therefore:

```text
ECR Recovery
+
GitOps Verification
```

are both required.

---

# 44. GitOps Image Configuration

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.8.4"
```

More strictly:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:abc123..."
```

The exact Helm schema depends on our chart implementation.

---

# 45. ECR Recovery After EKS Recovery

Complete flow:

```text
EKS
 ↓
ArgoCD
 ↓
Application Deployment
 ↓
Image Pull
 ↓
ECR
```

If:

```text
ECR missing
```

then:

```text
Pods
 ↓
ImagePullBackOff
```

Therefore ECR should be recovered before restoring application workloads.

---

# 46. Recommended Recovery Order

```text
Terraform
   ↓
VPC
   ↓
EKS
   ↓
IAM
   ↓
ECR
   ↓
EKS Add-ons
   ↓
ArgoCD
   ↓
Secrets
   ↓
Applications
```

Why ECR before applications?

Because:

```text
Deployment
   ↓
Image pull
   ↓
ECR
```

must work before Pods can become healthy.

---

# 47. ECR Recovery With 21 Services

We must verify:

```text
21 repositories
```

Example:

```text
[ ] auth-service
[ ] user-service
[ ] customer-service
[ ] catalog-service
[ ] product-service
[ ] inventory-service
[ ] pricing-service
[ ] promotion-service
[ ] cart-service
[ ] order-service
[ ] payment-service
[ ] fraud-service
[ ] shipping-service
[ ] delivery-service
[ ] notification-service
[ ] search-service
[ ] recommendation-service
[ ] review-service
[ ] tax-service
[ ] invoice-service
[ ] reporting-service
```

---

# 48. Bulk ECR Verification

A controlled script can verify repository existence.

Example:

```bash
SERVICES=(
  auth-service
  user-service
  customer-service
  catalog-service
  product-service
  inventory-service
  pricing-service
  promotion-service
  cart-service
  order-service
  payment-service
  fraud-service
  shipping-service
  delivery-service
  notification-service
  search-service
  recommendation-service
  review-service
  tax-service
  invoice-service
  reporting-service
)

for SERVICE in "${SERVICES[@]}"; do
  aws ecr describe-repositories \
    --repository-names "$SERVICE" \
    --region ap-south-1 \
    >/dev/null 2>&1

  if [ $? -eq 0 ]; then
    echo "OK: $SERVICE"
  else
    echo "MISSING: $SERVICE"
  fi
done
```

This is a verification script, not a replacement for Terraform.

---

# 49. Verify Image Availability

Example:

```bash
for SERVICE in "${SERVICES[@]}"; do
  echo "===== $SERVICE ====="

  aws ecr describe-images \
    --repository-name "$SERVICE" \
    --region ap-south-1 \
    --query 'imageDetails[?imageTags!=null].[imageTags,imageDigest,imagePushedAt]' \
    --output table
done
```

This helps identify:

```text
Repository exists
Image exists
Tags exist
Digest exists
```

---

# 50. ECR Recovery and Security Scanning

Our pipeline includes:

```text
Source
 ↓
Build
 ↓
Test
 ↓
SonarQube
 ↓
Dependency scan
 ↓
Trivy
 ↓
Docker image
 ↓
ECR
```

ECR can also perform supported image scanning.

Therefore:

```text
Trivy
+
ECR scanning
```

can provide layered vulnerability visibility.

Do not treat scanning as a substitute for patching.

---

# 51. Security Recovery

If an image was compromised:

```text
order-service:1.8.4
```

should not simply be restored and redeployed.

Process:

```text
Compromised image
      ↓
Quarantine
      ↓
Identify affected digest
      ↓
Identify deployments
      ↓
Build trusted version
      ↓
Scan
      ↓
Push
      ↓
Deploy
      ↓
Validate
```

---

# 52. Compromised ECR Credentials

If CI credentials are compromised:

```text
GitHub OIDC
      |
      v
CI IAM Role
```

actions:

```text
Review CloudTrail
Restrict trust policy
Review repository access
Check pushed images
Check unexpected tags
Check image digests
Rotate other affected credentials
```

With OIDC, there should generally be no long-lived AWS access keys stored in GitHub secrets for normal AWS authentication.

---

# 53. ECR Audit Trail

Investigate:

```text
Who pushed image?
Who deleted image?
Who modified repository?
When?
Which account?
Which IAM role?
```

Use:

```text
CloudTrail
ECR metadata
GitHub Actions history
Git history
ArgoCD history
```

Correlate:

```text
Git commit
→ CI build
→ ECR push
→ GitOps change
→ ArgoCD deployment
```

---

# 54. ECR Recovery Timeline

Example:

```text
10:00
Normal deployment

10:20
Repository accidentally deleted

10:23
Pods begin ImagePullBackOff

10:25
Incident detected

10:28
Repository deletion confirmed

10:32
DR ECR verified

10:36
Repository recreated

10:40
Image digest verified

10:43
GitOps updated/verified

10:47
Pods pull image

10:52
Application healthy

10:55
Business transaction successful
```

Measured:

```text
Recovery time = 32 minutes
```

Compare against:

```text
ECR RTO target
```

---

# 55. ECR Recovery Game Day

Use a dedicated DR/test account or environment.

Scenario:

> `order-service` ECR repository is unavailable.

Exercise:

```text
1. Deploy known image
2. Record digest
3. Simulate repository loss
4. Observe ImagePullBackOff
5. Recover repository
6. Recover image
7. Verify digest
8. Verify IAM
9. Restart/reconcile deployment
10. Validate service
11. Validate business flow
12. Measure RTO
```

Do not delete production repositories merely to practice recovery.

---

# 56. ECR Region-Failure Game Day

Scenario:

```text
Primary ECR:
ap-south-1
```

becomes unavailable.

DR:

```text
ap-southeast-1
```

Exercise:

```text
1. Activate DR EKS
2. Select DR ECR
3. Pull known image
4. Verify digest
5. Deploy order-service
6. Deploy remaining services
7. Validate all images
8. Validate business transaction
```

---

# 57. ECR Recovery Checklist

## Repository

```text
[ ] Repository exists
[ ] Correct region
[ ] Correct account
[ ] Immutable tags
[ ] Encryption enabled
[ ] Scanning configured
[ ] Lifecycle policy verified
```

## Images

```text
[ ] Required image exists
[ ] Correct tag
[ ] Correct digest
[ ] Image layers available
[ ] Image architecture correct
```

## Security

```text
[ ] CI IAM role works
[ ] EKS pull IAM works
[ ] Repository policy correct
[ ] KMS permissions correct
[ ] CloudTrail reviewed
```

## DR

```text
[ ] DR ECR exists
[ ] Replication configured
[ ] Replicated images available
[ ] Digest verified
[ ] DR IAM works
```

## Kubernetes

```text
[ ] ImagePullBackOff resolved
[ ] Pods Running
[ ] Readiness probes healthy
[ ] Services healthy
[ ] ArgoCD synced
```

## Business

```text
[ ] Login works
[ ] Product browsing works
[ ] Cart works
[ ] Order creation works
[ ] Payment workflow works
[ ] Notification works
```

---

# 58. Final ECR Recovery Architecture

```text
                         GitHub
                           |
                           v
                          CI
                           |
                 +---------+---------+
                 |                   |
                 v                   v
            ECR Primary         ECR DR
            ap-south-1       ap-southeast-1
                 |                   |
                 | replication       |
                 +---------+---------+
                           |
                    Image Digest
                           |
                           v
                         EKS
                           |
                         ArgoCD
                           |
                    21 Microservices
```

---

# 59. Complete Disaster-Recovery Chain

We now have:

```text
135 EKS Recovery
        ↓
136 RDS Recovery
        ↓
137 S3 Recovery
        ↓
138 ECR Recovery
```

The platform recovery chain is becoming:

```text
Terraform
    ↓
EKS
    ↓
ECR
    ↓
ArgoCD
    ↓
Applications
    ↓
RDS
    ↓
S3
    ↓
Redis
    ↓
SQS/Kafka
    ↓
Business Validation
```

The critical lesson is:

> **A Kubernetes cluster can be recreated quickly only if the infrastructure, container artifacts, configuration, secrets, and persistent data have all been designed for independent recovery.**

---

## Next Chapter — 139 Secrets/KMS Recovery

The next chapter will cover:

```text
139 — SECRETS/KMS RECOVERY

AWS Secrets Manager recovery
KMS key recovery
Secret deletion/recovery
Secret rotation failures
External Secrets Operator recovery
EKS Pod Identity
IAM/KMS dependency
Application startup failures
Database credential recovery
Cross-region secrets
KMS key policies
Key disable/delete incidents
Secret version recovery
Security incident response
RTO/RPO
Controlled Secrets/KMS recovery game day
```
