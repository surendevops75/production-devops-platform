# 148 — ImagePullBackOff Production Incident

> **Phase 18 — Incident Management**
>
> This chapter covers a production-grade **ImagePullBackOff / ErrImagePull incident** in Kubernetes/EKS, starting with language-independent Kubernetes/container fundamentals and then connecting the failure to our CI → ECR → GitOps → ArgoCD → EKS architecture.

---

## 1. What Is ImagePullBackOff?

`ImagePullBackOff` means Kubernetes attempted to pull a container image, the pull failed, and Kubernetes is backing off before retrying.

Typical flow:

```text
Pod scheduled
    ↓
Kubelet needs container image
    ↓
Container runtime requests image
    ↓
ECR / registry
    ↓
Image pull fails
    ↓
Retry
    ↓
Backoff
    ↓
ImagePullBackOff
```

Important:

> **ImagePullBackOff is a symptom. It does not tell you why the image pull failed.**

Possible causes include:

```text
Wrong image repository
Wrong image tag
Image doesn't exist
Image deleted
ECR repository missing
ECR authorization failure
IAM problem
Node cannot reach ECR
DNS failure
Network/NAT/VPC endpoint problem
Registry outage
Image architecture mismatch
Registry rate limit
KMS/encryption-related access issue
Cross-account ECR policy issue
GitOps deployed wrong image
```

---

# 2. Our Production Image Flow

For our e-commerce platform:

```text
Developer
    |
    v
GitHub
    |
    v
CI
    |
    +--> Maven
    +--> Tests
    +--> SonarQube
    +--> Dependency Scan
    +--> Trivy
    |
    v
Docker Build
    |
    v
ECR
    |
    v
GitOps Repository
    |
    v
ArgoCD
    |
    v
EKS
    |
    v
Kubernetes Deployment
    |
    v
Kubelet
    |
    v
Container Runtime
    |
    v
ECR
```

Therefore an ImagePullBackOff incident can originate from **multiple layers**.

---

# 3. First Principle

When a Pod shows:

```text
ImagePullBackOff
```

don't immediately restart the Pod.

Instead ask:

```text
What image is Kubernetes trying to pull?
        ↓
Does that image exist?
        ↓
Can this node authenticate?
        ↓
Can this node reach ECR?
        ↓
Is the repository correct?
        ↓
Is the tag/digest correct?
        ↓
Is the image architecture compatible?
        ↓
Did GitOps deploy the intended artifact?
```

---

# 4. Production Incident Scenario

Suppose we deploy:

```text
order-service:2.0.0
```

CI successfully builds the image.

But GitOps contains:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: 2.0.1
```

while ECR contains only:

```text
2.0.0
```

ArgoCD deploys:

```text
2.0.1
```

Kubernetes tries:

```text
ECR → order-service:2.0.1
```

ECR returns:

```text
manifest unknown
```

Pod enters:

```text
ImagePullBackOff
```

This is a **deployment/artifact consistency problem**, not a Kubernetes scheduling problem.

---

# 5. Production Incident Flow

Use this sequence:

```text
Alert
  ↓
Confirm customer impact
  ↓
Get affected Pod
  ↓
Describe Pod
  ↓
Read Events
  ↓
Identify exact image
  ↓
Classify error
  ↓
Check ECR repository
  ↓
Check tag/digest
  ↓
Check IAM
  ↓
Check node network/DNS
  ↓
Check architecture
  ↓
Check GitOps/ArgoCD
  ↓
Mitigate
  ↓
Validate
  ↓
RCA
  ↓
Prevent recurrence
```

---

# 6. Step 1 — Check Pods

```bash
kubectl get pods -n ecommerce
```

Example:

```text
NAME                             READY   STATUS             RESTARTS
order-service-6d8f7c9d8-x2abc   0/1     ImagePullBackOff   0
```

Another common state:

```text
ErrImagePull
```

The progression may look like:

```text
Pending
  ↓
ContainerCreating
  ↓
ErrImagePull
  ↓
ImagePullBackOff
```

---

# 7. ImagePullBackOff vs ErrImagePull

### `ErrImagePull`

The image pull attempt failed.

### `ImagePullBackOff`

Kubernetes continues retrying but introduces increasing delays between attempts.

So:

```text
ErrImagePull
```

is generally the immediate pull failure, while:

```text
ImagePullBackOff
```

indicates Kubernetes is backing off retries.

---

# 8. Step 2 — Describe the Pod

This is usually the **most important command**.

```bash
kubectl describe pod <pod-name> -n ecommerce
```

Go to:

```text
Events:
```

You might see:

```text
Failed to pull image
```

or:

```text
manifest unknown
```

or:

```text
pull access denied
```

or:

```text
no basic auth credentials
```

or:

```text
i/o timeout
```

or:

```text
context deadline exceeded
```

The exact event usually determines the next troubleshooting branch.

---

# 9. Example — Tag Does Not Exist

Event:

```text
Failed to pull image ".../order-service:2.0.1":
manifest unknown
```

Meaning:

```text
Kubernetes requested:
order-service:2.0.1

ECR:
2.0.0 exists
2.0.1 does not exist
```

This is an artifact/tag problem.

---

# 10. Check the Deployment

```bash
kubectl get deployment order-service -n ecommerce -o yaml
```

Find:

```yaml
containers:
  - name: order-service
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:2.0.1
```

Confirm:

```text
repository
tag
digest
container name
```

---

# 11. Check the Actual Pod Image

You can inspect:

```bash
kubectl get pod <pod-name> -n ecommerce \
  -o jsonpath='{.spec.containers[*].image}'
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:2.0.1
```

This is important because your local Git repository may contain a different value from what is currently running in the cluster.

---

# 12. Check ECR

List repositories:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

If the repository exists, inspect images:

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

You may see:

```text
1.8.0
1.9.0
2.0.0
```

but Kubernetes is requesting:

```text
2.0.1
```

Root cause identified.

---

# 13. Check Specific Image

You can inspect image details:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=2.0.0 \
  --region ap-south-1
```

This can show:

```text
imageDigest
imageTags
imageSizeInBytes
imagePushedAt
```

For production, the **digest** is the strongest artifact identity.

---

# 14. Tag vs Digest

Tag:

```text
order-service:2.0.0
```

Digest:

```text
order-service@sha256:abcdef...
```

A tag is a human-friendly reference.

A digest identifies the exact image manifest.

Our production principle is:

```text
Build once
    ↓
ECR
    ↓
Immutable artifact
    ↓
Promote same artifact
```

---

# 15. Why We Prefer Immutable ECR Tags

Our ECR repository uses:

```hcl
image_tag_mutability = "IMMUTABLE"
```

Therefore:

```text
2.0.0
```

cannot silently be overwritten with another image.

This gives us stronger artifact traceability.

Bad practice:

```text
order-service:latest
```

Better:

```text
order-service:2.0.0
```

Even better for strict promotion:

```text
order-service@sha256:<digest>
```

---

# 16. Common Cause #1 — Wrong Repository

Example GitOps:

```yaml
image:
  repository: .../orders-service
```

Actual ECR:

```text
.../order-service
```

Difference:

```text
orders-service
order-service
```

Kubernetes cannot find the intended repository.

---

# 17. Common Cause #2 — Wrong AWS Region

Suppose image exists in:

```text
ap-south-1
```

but Deployment references:

```text
ap-southeast-1
```

Example:

```text
123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/order-service
```

The image may not exist in that region.

This becomes especially important during DR.

---

# 18. Common Cause #3 — Image Deleted

ECR lifecycle policies may delete old images.

Suppose GitOps still references:

```text
order-service:1.2.0
```

but lifecycle cleanup deleted it.

Then:

```text
Deployment
   ↓
old image
   ↓
ECR
   ↓
image missing
   ↓
ImagePullBackOff
```

This is why ECR lifecycle policies must preserve the required rollback window.

---

# 19. Common Cause #4 — ECR Repository Deleted

Terraform may have recreated:

```text
order-service
```

but the old image artifacts are gone.

Remember:

> Terraform recreating an ECR repository does not recreate the image contents.

Infrastructure and artifacts are separate recovery concerns.

---

# 20. Common Cause #5 — IAM Failure

Suppose the image exists.

But the node/runtime identity cannot pull it.

The event may contain:

```text
pull access denied
```

or:

```text
403 Forbidden
```

or:

```text
AccessDeniedException
```

Now investigate identity.

---

# 21. Who Pulls the Image?

This distinction is important.

Typical EKS architecture:

```text
Pod
  |
  v
Node
  |
  v
Container Runtime
  |
  v
ECR
```

The image pull is performed by the node/container runtime path, using the EKS node's AWS permissions or the applicable EKS/ECR integration.

This is different from:

```text
Application Pod
    |
    v
AWS API
```

where we use:

```text
Pod Identity
    |
    v
IAM Role
```

for application-level AWS API access.

---

# 22. Node IAM Permissions

The EKS node identity needs appropriate ECR pull permissions.

Modern EKS designs may use the AWS-managed:

```text
AmazonEC2ContainerRegistryPullOnly
```

or an equivalent least-privilege policy appropriate to the chosen node architecture.

Do not give:

```text
AdministratorAccess
```

just to solve an image pull problem.

---

# 23. Cross-Account ECR

Suppose:

```text
AWS Account A
    |
    +--> ECR
```

and:

```text
AWS Account B
    |
    +--> EKS
```

Then EKS in Account B needs access to the ECR repository in Account A.

You may need:

```text
EKS/node IAM permissions
+
ECR repository policy
```

Both sides matter.

Architecture:

```text
EKS Account
    |
    | IAM
    v
ECR Account
    |
    v
Repository Policy
    |
    v
Image
```

---

# 24. Common Cause #6 — Network Failure

Suppose:

```text
Image exists
IAM is correct
```

but the node cannot reach ECR.

Then investigate:

```text
DNS
routing
NAT Gateway
VPC endpoints
security controls
proxy
AWS regional connectivity
```

---

# 25. EKS Private Nodes

Our production nodes are normally private.

Architecture:

```text
Private EKS Node
       |
       +--> ECR
       |
       +--> S3
       |
       +--> AWS APIs
```

Depending on architecture, connectivity can use:

```text
NAT Gateway
```

and/or appropriate:

```text
VPC Endpoints
```

---

# 26. ECR VPC Connectivity

ECR image pulls involve AWS services such as:

```text
ECR API
ECR Docker registry
S3
```

because ECR image layers are stored/retrieved through AWS's underlying service architecture.

For private EKS environments, network design must account for the required endpoints/routes.

Do not assume:

> "We created an ECR endpoint, therefore every ECR pull will work."

Validate the actual required AWS endpoints and DNS/routing design for your environment.

---

# 27. Network Troubleshooting

Check node networking.

For example:

```bash
kubectl get nodes -o wide
```

Then inspect:

```bash
kubectl describe node <node-name>
```

Review:

```text
node subnet
node conditions
events
```

From an approved troubleshooting environment, verify DNS and connectivity according to your organization's access model.

---

# 28. DNS Failure

If the node cannot resolve:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

the image pull can fail.

This can connect directly to:

```text
151 — DNS Failure
```

which we will investigate separately.

For this incident:

```text
ImagePullBackOff
     ↓
DNS resolution failure
```

is the dependency chain.

---

# 29. Common Cause #7 — Architecture Mismatch

Suppose the CI runner builds:

```text
linux/arm64
```

but EKS nodes run:

```text
linux/amd64
```

The image may exist, but the node cannot run the required image architecture.

You may see errors indicating incompatible platform/architecture.

---

# 30. Multi-Architecture Images

If your cluster contains:

```text
amd64 nodes
arm64 nodes
```

your image strategy must support them.

For example:

```text
image
 |
 +--> linux/amd64
 |
 +--> linux/arm64
```

This can be represented through a multi-platform manifest.

But don't introduce multi-architecture complexity unless your infrastructure actually needs it.

---

# 31. Check Image Architecture

From a trusted environment:

```bash
docker buildx imagetools inspect \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:2.0.0
```

Check:

```text
linux/amd64
linux/arm64
```

Compare with:

```bash
kubectl get nodes -L kubernetes.io/arch
```

---

# 32. Common Cause #8 — CI Push Failed

Sometimes the deployment is updated even though the image was never successfully pushed.

Bad CI flow:

```text
Build
  ↓
Docker image
  ↓
ECR push fails
  ↓
GitOps updated anyway
  ↓
ArgoCD deploys
  ↓
ImagePullBackOff
```

This is a pipeline design problem.

---

# 33. Correct CI Flow

The pipeline should be:

```text
Checkout
   ↓
Build
   ↓
Test
   ↓
Security scans
   ↓
Docker build
   ↓
ECR authentication
   ↓
Push image
   ↓
Verify push
   ↓
Capture digest
   ↓
Update GitOps
```

**Only after the artifact exists should deployment promotion occur.**

---

# 34. GitHub Actions Example

Conceptually:

```yaml
- name: Build image
  run: |
    docker build -t "$IMAGE_URI:$GITHUB_SHA" .

- name: Push image
  run: |
    docker push "$IMAGE_URI:$GITHUB_SHA"

- name: Capture digest
  run: |
    docker inspect --format='{{index .RepoDigests 0}}' \
      "$IMAGE_URI:$GITHUB_SHA"
```

The actual workflow should also verify the push succeeded before modifying GitOps.

---

# 35. GitLab CI

Same principle:

```text
GitLab Runner
    ↓
Build
    ↓
Test
    ↓
Scan
    ↓
Docker build
    ↓
ECR login
    ↓
Push
    ↓
Verify
    ↓
GitOps update
```

The implementation differs; the production control remains the same.

---

# 36. Jenkins

Same production sequence:

```text
Jenkins Agent
     ↓
Build
     ↓
Test
     ↓
Scan
     ↓
Docker Build
     ↓
ECR Push
     ↓
Verify
     ↓
GitOps
```

Jenkins should not update the deployment before confirming artifact availability.

---

# 37. Common Cause #9 — Wrong GitOps Value

Example:

CI successfully pushes:

```text
order-service:2.0.0
```

GitOps PR contains:

```text
order-service:2.0.1
```

ArgoCD correctly does what Git says.

The problem is:

```text
GitOps artifact reference != actual artifact
```

This is why GitOps does not eliminate deployment mistakes. It makes desired state explicit and auditable.

---

# 38. GitOps Investigation

Check ArgoCD:

```bash
argocd app get order-service
```

Then inspect GitOps repository.

Find:

```yaml
image:
  repository: ...
  tag: ...
```

or:

```yaml
image:
  digest: ...
```

Compare:

```text
Git
↓
ArgoCD
↓
Kubernetes
↓
ECR
```

All four should point to the same artifact.

---

# 39. Artifact Chain

Our production chain should be:

```text
Git commit
    |
    v
CI run
    |
    v
Image
    |
    v
ECR digest
    |
    v
GitOps
    |
    v
ArgoCD
    |
    v
Deployment
    |
    v
Pod
```

Example:

```text
Git:
abc123

Image:
order-service:abc123

Digest:
sha256:123abc...

GitOps:
order-service@sha256:123abc...

ArgoCD:
same digest

Pod:
same digest
```

This gives strong traceability.

---

# 40. Common Cause #10 — Wrong Secret Is Usually Not the First Suspect

An important distinction:

If the Pod is showing:

```text
ImagePullBackOff
```

don't immediately investigate:

```text
application DB password
Redis password
Kafka password
```

The container hasn't necessarily started yet.

First determine whether:

```text
image pull
```

is failing.

Only after the image successfully starts should you investigate application startup failures such as:

```text
CrashLoopBackOff
```

---

# 41. ImagePullBackOff vs CrashLoopBackOff

### ImagePullBackOff

```text
Image cannot be obtained
```

### CrashLoopBackOff

```text
Image obtained
      ↓
Container started
      ↓
Application/process failed
      ↓
Container restarted
```

Mental model:

```text
ImagePullBackOff
        ↓
Can Kubernetes get the image?

CrashLoopBackOff
        ↓
Can the application stay alive?
```

This distinction is extremely important in production troubleshooting.

---

# 42. ImagePullBackOff vs Pending

`Pending` can mean:

```text
No suitable node
Insufficient CPU
Insufficient memory
Affinity issue
Taint/toleration issue
PVC issue
```

`ImagePullBackOff` means the Pod got far enough for Kubernetes to attempt pulling the image.

---

# 43. Check Events Chronologically

```bash
kubectl get events -n ecommerce --sort-by=.lastTimestamp
```

Events may reveal:

```text
Scheduled
Pulling
Failed
BackOff
```

Example:

```text
Successfully assigned order-service to ip-10-0-12-45
Pulling image
Failed to pull image
Back-off pulling image
```

This tells us:

```text
Scheduling succeeded.
Image retrieval failed.
```

---

# 44. Production Incident Example — Wrong Tag

### T0

Developer merges:

```text
feat: improve order processing
```

### T1

CI builds:

```text
order-service:2.0.0
```

### T2

CI pushes:

```text
ECR
```

### T3

GitOps automation accidentally writes:

```text
2.0.1
```

### T4

ArgoCD syncs.

### T5

Pods:

```text
ImagePullBackOff
```

### T6

Alert fires.

### T7

Engineer runs:

```bash
kubectl describe pod ...
```

and finds:

```text
manifest unknown
```

### T8

ECR confirms:

```text
2.0.0 exists
2.0.1 missing
```

### Root Cause

Incorrect GitOps image reference.

---

# 45. Immediate Mitigation

If a known-good version exists:

```text
2.0.0
```

restore the GitOps reference.

Example:

```yaml
image:
  repository: ...
  tag: 2.0.0
```

Then:

```text
Git commit
   ↓
PR/review
   ↓
Merge
   ↓
ArgoCD sync
   ↓
Pods pull 2.0.0
```

---

# 46. Emergency Rollback

If your deployment process supports a controlled emergency rollback:

```text
Bad image reference
       ↓
Restore known-good GitOps state
       ↓
ArgoCD
       ↓
Known-good image
```

Avoid making an untracked manual cluster change and leaving GitOps pointing at the broken image.

Otherwise:

```text
kubectl change
      ↓
ArgoCD reconciliation
      ↓
Git desired state wins
      ↓
Bad image returns
```

---

# 47. What If the Image Was Deleted?

Suppose GitOps references:

```text
order-service:1.5.0
```

but ECR lifecycle policy deleted it.

Possible recovery:

```text
Check image replication
      ↓
Check retained artifact/archive
      ↓
Restore/push trusted original image
      ↓
Verify digest
      ↓
Deploy
```

If the original immutable artifact is unavailable, rebuilding from source may be necessary, but recognize that:

```text
rebuild from source
```

is not automatically identical to:

```text
original production artifact
```

This matters for strict artifact reproducibility.

---

# 48. ECR Lifecycle Policy Design

Lifecycle policies should consider:

```text
rollback window
compliance
release frequency
DR
incident recovery
storage cost
```

Don't simply say:

```text
Delete everything older than 7 days
```

if production may need a 30-day rollback window.

---

# 49. ECR Cross-Region Recovery

For our DR architecture:

```text
Primary:
ap-south-1

DR:
ap-southeast-1
```

Production artifact strategy can include ECR replication:

```text
Primary ECR
     |
     +----> DR ECR
```

Then DR EKS can pull the same trusted artifact from the DR region.

This reduces dependence on rebuilding images during a regional incident.

---

# 50. ImagePullBackOff During Region Failure

Imagine:

```text
ap-south-1
```

has failed.

DR EKS starts in:

```text
ap-southeast-1
```

But GitOps still references:

```text
ap-south-1 ECR
```

Then DR Pods may fail:

```text
ImagePullBackOff
```

This demonstrates why DR requires:

```text
EKS
+
ECR
+
GitOps
+
Secrets
+
KMS
+
DNS
+
data
```

not only EKS.

---

# 51. Image Pull During Node Scaling

Suppose Karpenter launches:

```text
10 new nodes
```

and all nodes immediately need:

```text
order-service
payment-service
catalog-service
...
```

Each node pulls images.

Potential issues:

```text
ECR traffic
network bandwidth
startup latency
large image size
node disk usage
registry limits
```

Therefore image design affects autoscaling performance.

---

# 52. Keep Images Small

Bad:

```text
1.5 GB application image
```

Better:

```text
250 MB
```

for the same workload where practical.

Our multi-stage Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

...

FROM eclipse-temurin:21-jre

...
```

keeps build tooling out of the final runtime image.

---

# 53. Production Image Principles

Use:

```text
Multi-stage builds
Minimal runtime image
Pinned base image
Non-root user
No unnecessary tools
No secrets
No source repository
No build cache
No credentials
```

Our Dockerfile includes:

```dockerfile
USER 10001
```

which prevents running the application as root.

---

# 54. Image Security

Our CI pipeline includes:

```text
SonarQube
Dependency Scan
Trivy
```

Trivy can scan:

```text
OS packages
language dependencies
container image vulnerabilities
configuration
```

The image should be scanned before production promotion.

---

# 55. Don't Solve ImagePullBackOff by Disabling Security

Bad response:

```text
Give node AdministratorAccess
```

or:

```text
Allow everything in ECR
```

or:

```text
Disable image scanning
```

or:

```text
Use public ECR access
```

Instead:

```text
Identify exact permission
       ↓
Grant least privilege
       ↓
Verify
```

---

# 56. ImagePullBackOff + Security Incident

Suppose a vulnerable image is identified.

Security team quarantines:

```text
sha256:badimage
```

Then GitOps accidentally still references it.

Deployment attempts to pull a blocked/deleted artifact.

Pods enter:

```text
ImagePullBackOff
```

This is an example where:

```text
Security control
+
deployment process
```

must be coordinated.

---

# 57. Image Digest Verification

For stronger supply-chain controls:

```text
CI
 |
 +--> image
 |
 +--> digest
 |
 v
GitOps
 |
 +--> digest
 |
 v
ArgoCD
 |
 v
EKS
```

You can additionally introduce:

```text
image signing
SBOM
provenance
admission policy
```

later in the platform-engineering/security chapters.

---

# 58. Node-Level Diagnosis

When only Pods on one node fail:

```text
node-A → ImagePullBackOff
node-B → Running
node-C → Running
```

suspect node-specific issues:

```text
IAM identity
network
DNS
runtime
disk
configuration
architecture
```

When Pods fail across every node:

```text
node-A → ImagePullBackOff
node-B → ImagePullBackOff
node-C → ImagePullBackOff
```

suspect shared dependencies:

```text
ECR
IAM
GitOps image reference
region
network architecture
image availability
```

This is a powerful scope-based diagnostic technique.

---

# 59. Scope the Incident

### One Pod

Potential:

```text
Pod-specific image reference
```

### One node

Potential:

```text
node-specific problem
```

### One service

Potential:

```text
wrong image
wrong repository
wrong tag
```

### All services

Potential:

```text
ECR
network
IAM
regional issue
```

Scope first.

---

# 60. Validation After Recovery

Do not stop at:

```text
Pod = Running
```

Validate:

```bash
kubectl get pods -n ecommerce
```

Then:

```bash
kubectl get endpointslice -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Check:

```text
Ready endpoints
```

Then:

```text
ALB
 ↓
order-service
 ↓
PostgreSQL
 ↓
business transaction
```

Test an actual order workflow.

---

# 61. Observability Validation

After recovery check:

### Prometheus

```text
Pod availability
Restart count
HTTP errors
Latency
```

### ELK

```text
Image startup errors
Application startup logs
```

### Jaeger

```text
Successful request traces
```

### ArgoCD

```text
Synced
Healthy
```

---

# 62. Business Validation

For the e-commerce platform:

```text
Login
  ↓
Browse product
  ↓
Add to cart
  ↓
Create order
  ↓
Inventory reservation
  ↓
Payment authorization
  ↓
Order confirmation
  ↓
Notification
```

Infrastructure recovery is not complete until the business path works.

---

# 63. Production Prevention

### CI

- [ ] Build succeeds
- [ ] Tests pass
- [ ] Security scans pass
- [ ] Image push succeeds
- [ ] Image digest captured
- [ ] ECR availability verified
- [ ] GitOps updated only after successful push

### ECR

- [ ] Immutable tags
- [ ] Lifecycle policy
- [ ] Rollback retention
- [ ] Image scanning
- [ ] Cross-region replication if required
- [ ] Repository policy
- [ ] Monitoring

### GitOps

- [ ] Valid image reference
- [ ] Digest verification where appropriate
- [ ] PR review
- [ ] ArgoCD health checks
- [ ] Rollback procedure

### EKS

- [ ] Node IAM correct
- [ ] Private networking correct
- [ ] DNS working
- [ ] ECR connectivity tested
- [ ] Node architecture compatible

---

# 64. Alerting

Create alerts around:

```text
ImagePullBackOff
ErrImagePull
Pod unavailable
Deployment unavailable
ArgoCD degraded
```

Example conceptual alert:

```yaml
- alert: KubernetesImagePullFailure
  expr: |
    kube_pod_container_status_waiting_reason{
      reason=~"ImagePullBackOff|ErrImagePull"
    } > 0
  for: 5m
  labels:
    severity: critical
    team: platform
  annotations:
    summary: "Kubernetes image pull failure"
```

The exact metric and labels depend on your `kube-state-metrics` version/configuration.

---

# 65. Runbook

Create:

```text
runbooks/image-pull-backoff.md
```

Recommended structure:

```markdown
# ImagePullBackOff Runbook

## Symptoms

## Customer Impact

## Initial Checks

## Check Pod Events

## Identify Image

## Check ECR Repository

## Check Image Tag

## Check Image Digest

## Check IAM

## Check Node Network

## Check DNS

## Check Architecture

## Check GitOps

## Check ArgoCD

## Immediate Mitigation

## Rollback

## Validation

## Root Cause

## Prevention

## RCA
```

---

# 66. RCA Example

```markdown
# ImagePullBackOff Incident RCA

## Incident

order-service Pods entered ImagePullBackOff.

## Severity

SEV-2

## Impact

New order-service Pods could not start.

## Detection

Prometheus alert detected ImagePullBackOff.

## Root Cause

GitOps referenced image tag 2.0.1,
but CI had successfully published only 2.0.0.

## Contributing Factors

- GitOps update was not validated against ECR.
- Deployment promotion was not digest-based.
- No pre-deployment image existence check.

## Mitigation

Reverted GitOps image reference to 2.0.0.

## Permanent Fix

CI now verifies the pushed image and digest
before updating GitOps.

## Prevention

- Immutable ECR tags
- Digest promotion
- CI artifact verification
- Deployment validation
- Rollback procedure
```

---

# 67. Advanced Production Improvement

The mature architecture becomes:

```text
Developer
    |
    v
Git
    |
    v
CI
    |
    +--> Build
    +--> Test
    +--> Security
    +--> Trivy
    |
    v
ECR
    |
    +--> image digest
    |
    v
GitOps
    |
    +--> immutable digest
    |
    v
ArgoCD
    |
    v
Admission / Policy
    |
    v
EKS
```

Now there is much less ambiguity.

---

# 68. Example Immutable Deployment

Instead of:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: latest
```

prefer:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:..."
```

The exact Helm implementation can vary.

The principle is:

> **Deploy the exact artifact that CI built and verified.**

---

# 69. Troubleshooting Decision Tree

```text
ImagePullBackOff
       |
       v
kubectl describe pod
       |
       v
What does Events say?
       |
       +-----------------------------+
       |                             |
       v                             v
manifest unknown              access denied
       |                             |
       v                             v
Check tag/repo              Check IAM/ECR policy
       |
       +-----------------------------+
       |
       v
timeout / connection error
       |
       v
Check DNS/network/NAT/endpoints
       |
       +-----------------------------+
       |
       v
platform mismatch
       |
       v
Check amd64/arm64
       |
       v
Is GitOps correct?
       |
       v
Is ECR image present?
       |
       v
Fix desired state
       |
       v
ArgoCD
       |
       v
Pod Running
       |
       v
Business validation
```

---

# 70. Production Mental Model

When you see:

```text
ImagePullBackOff
```

think:

```text
Kubernetes cannot obtain the image
```

Then:

```text
IMAGE
 |
 +--> Repository correct?
 |
 +--> Tag correct?
 |
 +--> Digest correct?
 |
 +--> Image exists?
 |
 +--> Image architecture correct?
 |
 +--> Image deleted?
 |
 v
ACCESS
 |
 +--> IAM?
 |
 +--> ECR policy?
 |
 v
NETWORK
 |
 +--> DNS?
 |
 +--> Route?
 |
 +--> NAT?
 |
 +--> VPC endpoint?
 |
 v
DEPLOYMENT
 |
 +--> GitOps?
 |
 +--> ArgoCD?
 |
 v
RECOVERY
 |
 +--> Fix
 +--> Rollback
 +--> Validate
 +--> RCA
```

---

# 71. Key Rules

1. **ImagePullBackOff is a symptom, not the root cause.**
2. Always run `kubectl describe pod` first.
3. Read the **Events** section carefully.
4. Verify the exact image repository.
5. Verify the exact tag or digest.
6. Confirm the image actually exists in ECR.
7. Check ECR IAM permissions.
8. Check private-node network connectivity.
9. Check DNS if the error indicates connectivity failure.
10. Check `amd64` vs `arm64` when relevant.
11. Never use `latest` for production deployments.
12. Prefer immutable tags and/or image digests.
13. CI should verify the image before updating GitOps.
14. ECR lifecycle policies must preserve rollback artifacts.
15. DR requires ECR artifact recovery/replication too.
16. Fix GitOps rather than leaving manual Kubernetes drift.
17. `ImagePullBackOff` happens before application startup; don't confuse it with `CrashLoopBackOff`.
18. Validate the business transaction after recovery.
19. Record the exact image digest involved in the incident.
20. The ideal production chain is:

```text
Git Commit
   ↓
CI
   ↓
Verified Image
   ↓
ECR Digest
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

---

# 72. Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ✅
147 — Disk Full                    ✅
148 — ImagePullBackOff             ✅
149 — ALB 5xx                      ⏭️
150 — Service Connectivity Failure ⏭️
151 — DNS Failure                  ⏭️
152 — RDS Connection Exhaustion    ⏭️
153 — Redis Failure                ⏭️
154 — SQS Backlog                  ⏭️
155 — Kafka Consumer Lag            ⏭️
156 — Node Failure                 ⏭️
157 — AZ Failure                   ⏭️
158 — ArgoCD Drift                 ⏭️
159 — Bad Production Deployment    ⏭️
160 — Security Incident            ⏭️
```

**Next: `149 — ALB 5xx Production Incident`**.