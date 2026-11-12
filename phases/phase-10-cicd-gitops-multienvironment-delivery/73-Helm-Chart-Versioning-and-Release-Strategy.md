# 73 — Helm Chart Versioning and Release Strategy

Now we need to solve one important problem from the previous chapter.

We currently have **two versions**:

```text
Docker image
    ↓
8f31a2c
```

and:

```text
Helm chart
    ↓
v1.0.0
```

That is not necessarily wrong, but we need to understand exactly how these versions relate.

---

# 1. First: What Is a Helm Chart?

Our `order-service` repository contains:

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

The Helm chart is basically the **deployment template**.

It tells Kubernetes how to run our application.

For example:

```text
Deployment
Service
HPA
PDB
ConfigMap
ServiceAccount
Ingress
```

The Docker image contains the **application**.

The Helm chart contains the **deployment instructions**.

---

# 2. Docker Image vs Helm Chart

Think of it this way:

```text
Docker image
    =
Application package
```

and:

```text
Helm chart
    =
Kubernetes deployment package
```

For example:

```text
Docker:
order-service:8f31a2c
```

contains:

```text
Java application
dependencies
runtime
```

while:

```text
Helm:
order-service chart v1.0.0
```

contains:

```text
Deployment
Service
HPA
PDB
ConfigMap
etc.
```

---

# 3. Why Do We Need Chart Versions?

Suppose we change the Deployment template.

Today:

```yaml
resources:
  requests:
    cpu: 250m
```

Tomorrow we change it to:

```yaml
resources:
  requests:
    cpu: 500m
```

That's a **Helm chart change**.

The application code might not have changed at all.

Therefore:

```text
Application version
```

and:

```text
Deployment configuration version
```

are different things.

---

# 4. Chart.yaml

Open:

```text
helm/order-service/Chart.yaml
```

Example:

```yaml
apiVersion: v2

name: order-service

description: Order Service Helm Chart

type: application

version: 1.0.0

appVersion: "1.0.0"
```

There are two important fields:

```text
version
appVersion
```

---

# 5. `version`

This is the **Helm chart version**.

Example:

```yaml
version: 1.0.0
```

If we change the Helm templates:

```text
v1.0.0
     ↓
v1.1.0
```

The chart version changes.

---

# 6. `appVersion`

This describes the application version.

For example:

```yaml
appVersion: "1.0.0"
```

But there is an important point:

**Do not use `appVersion` as the mechanism that actually deploys your Docker image.**

Our actual deployment should explicitly specify:

```yaml
image:
  repository: ...
  tag: "8f31a2c"
```

or preferably an image digest.

---

# 7. Why Keep Them Separate?

Imagine:

```text
Application code
doesn't change
```

but we change:

```text
HPA configuration
```

Then:

```text
Docker image:
8f31a2c

Helm chart:
1.1.0
```

is perfectly reasonable.

Conversely, application code can change:

```text
Docker:
abc1234
```

while the Helm chart remains:

```text
1.0.0
```

if the deployment template hasn't changed.

---

# 8. Our Release Model

We will use:

```text
Application release
        +
Helm chart release
```

For example:

```text
Application:
order-service 1.4.0

Docker image:
order-service:8f31a2c

Helm chart:
order-service 1.2.0
```

These can be recorded together in GitOps.

---

# 9. But This Looks Complicated

Don't worry.

For the project, remember this simple rule:

### Docker image

Answers:

> **What application code am I running?**

Example:

```text
8f31a2c
```

### Helm chart

Answers:

> **How should Kubernetes run that application?**

Example:

```text
1.2.0
```

---

# 10. What Should CI Build?

When a developer pushes code:

```text
Git commit
     ↓
GitHub Actions
```

CI should:

```text
1. Compile
2. Test
3. Security scan
4. Build Docker image
5. Scan Docker image
6. Push image to ECR
```

Then, if the Helm chart changed, CI can package/release a new chart version.

---

# 11. Example CI Flow

Developer makes:

```text
commit:
8f31a2c
```

GitHub Actions:

```text
Checkout
   ↓
Maven build
   ↓
Unit tests
   ↓
SonarQube
   ↓
Dependency scan
   ↓
Docker build
   ↓
Trivy
   ↓
ECR push
```

Image:

```text
order-service:8f31a2c
```

---

# 12. What About the Helm Chart?

CI checks:

```bash
helm lint helm/order-service
```

Then:

```bash
helm package helm/order-service
```

This creates something like:

```text
order-service-1.0.0.tgz
```

if:

```yaml
version: 1.0.0
```

---

# 13. Where Should the Chart Go?

There are two common approaches.

### Approach A — Git repository

ArgoCD reads the chart directly from Git.

That's what our current project uses.

```text
GitHub
   ↓
helm/order-service
   ↓
ArgoCD
```

### Approach B — OCI Helm registry

The chart is packaged and pushed to a registry.

For example:

```text
OCI registry
   ↓
order-service:1.0.0
```

Then ArgoCD pulls the chart from the registry.

For our current implementation, **stay with Git-based Helm charts**.

We don't need to introduce OCI Helm repositories yet.

---

# 14. Why Did We Put `chartRevision` in `app.yaml`?

Remember:

```yaml
repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: v1.0.0
```

This means:

```text
Git repository
       ↓
checkout v1.0.0
       ↓
helm/order-service
```

But there's a subtle issue.

A Git repository tag like:

```text
v1.0.0
```

is a **Git tag**, not automatically a Helm chart version.

We need to make sure our release process creates and uses the correct Git tag.

---

# 15. Recommended Release Strategy

For this project, let's use:

```text
Git tag
    =
Application/Chart release reference
```

For example:

```text
v1.4.0
```

The repository contains:

```yaml
version: 1.4.0
```

and the Git repository has:

```text
tag:
v1.4.0
```

Now the relationship is easy to understand:

```text
Git tag v1.4.0
       |
       +---- Helm chart version 1.4.0
       |
       +---- Released application configuration
```

The Docker image remains independently identifiable by commit SHA.

---

# 16. Example Release

Developer commits:

```text
8f31a2c
```

Application version:

```text
1.4.0
```

Create Docker image:

```text
order-service:8f31a2c
```

Update Helm:

```yaml
version: 1.4.0
```

Create Git tag:

```text
v1.4.0
```

Now we have:

```text
Git:
v1.4.0
   |
   +---- commit 8f31a2c
   |
   +---- Helm chart 1.4.0
   |
   +---- Docker image 8f31a2c
```

This gives us traceability.

---

# 17. But What If the Chart Changes Without Application Code?

Suppose we only change:

```text
HPA
```

The application image remains:

```text
8f31a2c
```

but chart changes from:

```text
1.4.0
```

to:

```text
1.4.1
```

Then:

```text
Docker:
8f31a2c

Chart:
1.4.1
```

That's okay.

The two versions don't need to be numerically identical.

---

# 18. Our GitOps Release Information

We could have:

```yaml
service: order-service

repository: https://github.com/company-ecommerce/order-service.git

chartPath: helm/order-service

chartRevision: v1.4.1

enabled: true

team: backend-team
```

And:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

  tag: "8f31a2c"
```

This means:

```text
Helm chart:
v1.4.1

Application image:
8f31a2c
```

---

# 19. Production Promotion

Now imagine DEV has:

```text
Chart:
v1.4.1

Image:
8f31a2c
```

QA should receive the same release combination:

```text
Chart:
v1.4.1

Image:
8f31a2c
```

STAGING:

```text
Chart:
v1.4.1

Image:
8f31a2c
```

PROD:

```text
Chart:
v1.4.1

Image:
8f31a2c
```

Now we have a very clear release.

---

# 20. Why Not Just Use `main` Everywhere?

Suppose PROD says:

```yaml
chartRevision: main
```

At 10:00 AM:

```text
main = chart version A
```

ArgoCD deploys it.

At 11:00 AM somebody changes the Helm chart.

Now:

```text
main = chart version B
```

ArgoCD can see a new desired state even though you didn't intentionally promote that chart change to production.

That's undesirable.

Therefore:

### DEV

Can track:

```text
main
```

### QA/STAGING/PROD

Prefer:

```text
release tag
```

or an immutable commit SHA.

---

# 21. Environment Strategy

A practical model:

```text
DEV
 ↓
main
```

Then:

```text
QA
 ↓
release tag
```

STAGING:

```text
release tag
```

PROD:

```text
release tag
```

For stricter environments, pin to a commit SHA or immutable chart artifact.

---

# 22. Application Release vs Deployment Promotion

This distinction is extremely important.

### Release

Creating:

```text
v1.4.0
```

means:

> This version is available for deployment.

### Promotion

Moving:

```text
v1.4.0
```

from:

```text
DEV → QA → STAGING → PROD
```

means:

> We have chosen to deploy that already-created release into another environment.

---

# 23. Never Rebuild During Promotion

Bad:

```text
DEV
 ↓
build
 ↓
QA
 ↓
build
 ↓
STAGING
 ↓
build
 ↓
PROD
```

Good:

```text
Source
 ↓
Build once
 ↓
Image 8f31a2c
 ↓
Test
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

The same artifact is promoted.

---

# 24. Immutable Image

An image tag like:

```text
latest
```

is dangerous for controlled promotion because its meaning can change.

Instead:

```text
order-service:8f31a2c
```

is tied to the source commit.

Even better, Kubernetes can deploy:

```text
order-service@sha256:<digest>
```

The digest identifies the exact image content.

---

# 25. Tag vs Digest

Tag:

```text
order-service:8f31a2c
```

is a human-friendly reference.

Digest:

```text
order-service@sha256:abcdef...
```

identifies the exact image manifest/content.

Think:

```text
Tag
 =
label

Digest
 =
exact artifact identity
```

---

# 26. Why Digest Is Better for Production

Imagine someone accidentally pushes another image under a mutable tag.

If production uses:

```text
order-service:release-1
```

the tag could potentially point somewhere different later.

A digest:

```text
order-service@sha256:123abc...
```

doesn't have that ambiguity.

Therefore our eventual production model will be:

```text
DEV:
tag acceptable for convenience

QA:
tag + digest traceability

STAGING:
digest preferred

PROD:
digest preferred/required
```

The exact policy depends on the organization's release controls.

---

# 27. How CI Gets the Digest

After pushing:

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

AWS ECR can report the image digest.

Example:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag=8f31a2c \
  --region ap-south-1
```

You will get metadata including a digest similar to:

```text
sha256:abc123...
```

The exact value will depend on the actual image.

---

# 28. GitOps Can Store the Digest

Our values can eventually become:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service

  tag: "8f31a2c"

  digest: "sha256:abc123..."
```

Our Helm Deployment can then construct:

```text
repository@digest
```

instead of relying solely on the tag.

---

# 29. Why Keep Both Tag and Digest?

Because they serve different purposes.

Tag:

```text
8f31a2c
```

is easy for humans.

Digest:

```text
sha256:...
```

is exact.

Therefore:

```text
GitOps
 ├── tag
 └── digest
```

provides both readability and immutability.

---

# 30. What Does a Production Deployment Now Mean?

Suppose production says:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:abc123..."
```

and:

```yaml
chartRevision: v1.4.1
```

Then production is effectively saying:

```text
Deploy:

Helm chart v1.4.1

with:

Docker image
8f31a2c
sha256:abc123...
```

That is a much stronger release definition.

---

# 31. Complete Traceability

Now we can trace:

```text
Developer commit
       ↓
8f31a2c
       ↓
Docker build
       ↓
ECR
       ↓
sha256:abc123...
       ↓
GitOps
       ↓
Helm chart v1.4.1
       ↓
ArgoCD
       ↓
EKS
       ↓
order-service Pod
```

If someone asks:

> "What exact code is running in production?"

we can trace it.

---

# 32. How to Check What Is Running

First:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app.kubernetes.io/name=order-service
```

Then:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

You might see:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:abc123...
```

That gives the exact deployed image reference.

---

# 33. Check ArgoCD

```bash
argocd app get order-service-prod
```

You can see the Git revision and deployment state.

Then:

```bash
kubectl describe deployment order-service \
  -n ecommerce-prod
```

This helps connect:

```text
ArgoCD
 ↓
Deployment
 ↓
Pod
 ↓
Image
```

---

# 34. Release Example

Let's put everything together.

Developer creates:

```text
Commit:
8f31a2c
```

CI builds:

```text
order-service:8f31a2c
```

ECR stores:

```text
order-service@sha256:abc123...
```

Helm chart:

```text
v1.4.1
```

Release tag:

```text
v1.4.1
```

GitOps DEV:

```yaml
chartRevision: v1.4.1

image:
  tag: 8f31a2c
  digest: sha256:abc123...
```

After testing:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

All use:

```text
Chart:  v1.4.1
Image:  8f31a2c
Digest: sha256:abc123...
```

---

# 35. Rollback

Suppose:

```text
Current PROD:
Chart v1.4.1
Image 8f31a2c
```

Previous release:

```text
Chart v1.3.0
Image 7a21c9e
```

Rollback GitOps:

```diff
- chartRevision: v1.4.1
+ chartRevision: v1.3.0
```

and:

```diff
- tag: 8f31a2c
+ tag: 7a21c9e
```

plus the corresponding digest.

Then:

```text
Git
 ↓
ArgoCD
 ↓
Helm
 ↓
Kubernetes
 ↓
Previous release
```

No rebuild.

---

# 36. Important Database Warning

Application rollback is not always enough.

Suppose release `v1.4.1` performs:

```text
Database migration
```

and changes:

```text
old column
```

into:

```text
new column
```

Then simply rolling the application back may fail if the old application cannot understand the new database schema.

That's why our earlier deployment strategy uses:

```text
Expand
 ↓
Migrate
 ↓
Deploy
 ↓
Contract
```

rather than destructive database changes during a normal application deployment.

We will cover this deeply in the zero-downtime/database migration section.

---

# 37. What Should We Change Now?

For our current project, update `Chart.yaml`:

```yaml
apiVersion: v2

name: order-service

description: Order Service Helm Chart

type: application

version: 1.0.0

appVersion: "1.0.0"
```

Then create a Git tag:

```bash
git tag v1.0.0
```

Push:

```bash
git push origin v1.0.0
```

Now Git has:

```text
main
 |
 +-- v1.0.0
```

---

# 38. Verify the Chart

Run:

```bash
helm lint helm/order-service
```

Then:

```bash
helm package helm/order-service
```

Expected:

```text
Successfully packaged chart and saved it to:
order-service-1.0.0.tgz
```

The exact path may differ depending on your current directory.

---

# 39. Important Production Rule

Don't change:

```yaml
version: 1.0.0
```

for every Docker image build.

For example, if you only changed application Java code:

```text
Commit A
Commit B
Commit C
```

you might have:

```text
Chart:
1.0.0
```

while Docker images are:

```text
A
B
C
```

If the Helm templates haven't changed, there's no reason to release a new chart solely because application code changed.

---

# 40. When Should Chart Version Change?

Change the chart version when the chart itself changes.

Examples:

```text
Deployment template changed
Service changed
HPA changed
PDB changed
Ingress changed
securityContext changed
environment configuration structure changed
```

Then:

```text
1.0.0
 ↓
1.1.0
```

or:

```text
1.0.0
 ↓
1.0.1
```

depending on the nature of the chart change.

---

# 41. Simple Rule for Versioning

Remember:

```text
Docker image version
    =
Application code

Helm chart version
    =
Kubernetes deployment configuration

Git tag
    =
Release reference
```

And:

```text
Digest
    =
Exact container artifact
```

---

# 42. Our Final Release Architecture

```text
                         SOURCE
                           |
                           v
                     Git commit
                       8f31a2c
                           |
              +------------+------------+
              |                         |
              v                         v
          Docker build             Helm chart
              |                         |
              v                         v
         ECR image                  Chart 1.4.1
              |                         |
              |                    Git tag v1.4.1
              |
              v
       SHA256 digest
              |
              +------------+
                           |
                           v
                       GitOps
                           |
             +-------------+-------------+
             |             |             |
            DEV           QA          STAGING
             |             |             |
             +-------------+-------------+
                           |
                           v
                         PROD
                           |
                           v
                         ArgoCD
                           |
                           v
                          EKS
```

---

# 43. The One Sentence to Remember

> **The Docker image tells us what code to run; the Helm chart tells Kubernetes how to run it; GitOps decides where and when that release should run; ArgoCD makes EKS match that decision.**

---

# Next Chapter

### `74-Immutable-Image-Digest-and-Production-Deployment.md`

Now we'll implement the next important part practically:

```text
GitHub Actions
      ↓
Docker build
      ↓
ECR push
      ↓
Get SHA256 digest
      ↓
Trivy scans exact image
      ↓
GitOps updated with digest
      ↓
ArgoCD
      ↓
EKS
      ↓
Verify exact image running
```

That will connect our **CI chapter directly to GitOps and production deployment**, rather than leaving the release process theoretical.
