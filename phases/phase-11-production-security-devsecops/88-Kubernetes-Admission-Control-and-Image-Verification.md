# 88 — Kubernetes Admission Control and Image Verification

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Registry:** Amazon ECR
> **CI:** GitHub Actions
> **GitOps:** ArgoCD
> **Policy:** Kyverno + Pod Security Admission
> **Previous:** 87 — Container Image Security, SBOM and Image Signing
> **Next:** 89 — Kubernetes RBAC + AWS IAM Deep Dive

---

# 1. Objective

In Chapter 87, we built the image security chain:

```text
Source Code
    ↓
Docker Build
    ↓
Trivy
    ↓
SBOM
    ↓
ECR
    ↓
Image Digest
    ↓
Image Signing
```

But there is a problem.

What prevents someone from deploying:

```yaml
image: docker.io/random-user/test-image:latest
```

directly into the cluster?

Or:

```yaml
securityContext:
  privileged: true
```

Or:

```yaml
runAsUser: 0
```

CI can detect these problems, but production needs another layer.

That layer is:

```text
Kubernetes Admission Control
```

Our target architecture becomes:

```text
GitHub
   ↓
CI Security
   ↓
ECR
   ↓
Signed Image
   ↓
GitOps
   ↓
ArgoCD
   ↓
Kubernetes API
   ↓
Admission Control
   ↓
ALLOW / REJECT
   ↓
EKS
```

---

# 2. What Is Admission Control?

When Kubernetes receives a request to create or modify a resource:

```text
kubectl
   |
   v
Kubernetes API Server
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Admission Control
   |
   v
Persistence
```

Admission control can inspect the requested object before Kubernetes accepts it.

For example:

```text
Deployment
    |
    v
Admission Policy
    |
    +--> privileged?
    +--> trusted image?
    +--> latest tag?
    +--> resource limits?
    +--> required labels?
    |
    v
ALLOW / DENY
```

---

# 3. Why Do We Need It If We Already Have CI?

Because CI is not the only path to Kubernetes.

Consider:

```text
GitHub Actions
      |
      v
GitOps
      |
      v
ArgoCD
```

That's our normal path.

But an administrator could also theoretically run:

```bash
kubectl apply -f deployment.yaml
```

A compromised deployment tool might also attempt to deploy something.

Admission control creates a **cluster-side security boundary**.

---

# 4. Our Defense Layers

We now have:

```text
Layer 1
Source Code Security
        ↓
SonarQube / Gitleaks / Dependency Scan

Layer 2
Container Security
        ↓
Trivy / ECR Scan / SBOM

Layer 3
Artifact Integrity
        ↓
Digest / Image Signing

Layer 4
GitOps
        ↓
ArgoCD

Layer 5
Admission Control
        ↓
Kyverno / Pod Security Admission

Layer 6
Runtime Security
        ↓
SecurityContext / NetworkPolicy / IAM
```

This is defense in depth.

---

# 5. Two Important Kubernetes Security Mechanisms

We will use two mechanisms:

```text
Pod Security Admission
```

and:

```text
Kyverno
```

They solve different problems.

---

# 6. Pod Security Admission

Kubernetes provides built-in Pod Security Standards.

The levels are:

```text
Privileged
Baseline
Restricted
```

For our application namespaces:

```text
Restricted
```

is the target.

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce-prod

  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

This protects against many unsafe pod configurations.

---

# 7. What PSA Does Well

Pod Security Admission can enforce the Kubernetes Pod Security Standards.

It helps prevent configurations such as:

```text
Privileged containers
Host namespaces
Unsafe volume usage
Running as root in restricted contexts
Unsafe security configurations
```

But we also need organization-specific rules.

For example:

```text
Only images from our ECR registry
```

or:

```text
All production pods must have company-specific labels
```

That's where Kyverno becomes useful.

---

# 8. What Is Kyverno?

Kyverno is a Kubernetes-native policy engine.

Instead of writing complicated external policy code, we can define policies as Kubernetes resources.

Conceptually:

```text
Policy
   |
   v
Kubernetes Admission
   |
   +--> Validate
   +--> Mutate
   +--> Generate
```

For our project, we'll primarily use:

```text
Validate
```

and selectively:

```text
Mutate
```

---

# 9. Why Use Kyverno?

Suppose the organization says:

> Production images must come only from ECR.

We can express that as a policy.

Then:

```yaml
image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:...
```

passes.

But:

```yaml
image: nginx:latest
```

fails.

---

# 10. Production Policy Model

We'll divide policies into categories.

```text
Security Policies
    |
    +-- Image policies
    |
    +-- Container security
    |
    +-- Resource policies
    |
    +-- Metadata policies
    |
    +-- Namespace policies
    |
    +-- Supply-chain policies
```

---

# 11. Policy Repository

Create a dedicated platform/security area.

For example:

```text
platform/
├── policies/
│   ├── kyverno/
│   │   ├── image/
│   │   ├── security/
│   │   ├── resources/
│   │   └── metadata/
│   │
│   └── pod-security/
│
├── charts/
├── addons/
└── README.md
```

This is better than manually creating policies from a laptop.

The policies themselves should be managed through Git.

---

# 12. Install Kyverno

Create namespace:

```bash
kubectl create namespace kyverno
```

Install Kyverno using its official Helm chart.

The exact chart version should be pinned in the production implementation.

Conceptually:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

Then:

```bash
helm install kyverno \
  kyverno/kyverno \
  --namespace kyverno
```

For production, we would put the Helm release into our platform/GitOps management rather than relying on an operator's laptop.

---

# 13. Verify Kyverno

Check:

```bash
kubectl get pods -n kyverno
```

You should see the Kyverno components running.

Then:

```bash
kubectl get deployments -n kyverno
```

And:

```bash
kubectl get crds | grep kyverno
```

---

# 14. Production Installation Architecture

Eventually:

```text
platform GitOps
       |
       v
ArgoCD
       |
       v
Kyverno Helm Release
       |
       v
EKS
```

So:

```text
ArgoCD
  |
  +--> Prometheus
  +--> Grafana
  +--> AWS Load Balancer Controller
  +--> External Secrets
  +--> Kyverno
  +--> Argo Rollouts
```

Platform components are controlled centrally.

---

# 15. First Policy — Reject Privileged Containers

Create:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:

  validationFailureAction: Enforce

  background: true

  rules:

    - name: privileged-containers-not-allowed

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Privileged containers are not allowed."

        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
```

The exact Kyverno policy schema should be tested against the pinned Kyverno version before production rollout.

---

# 16. Apply the Policy

```bash
kubectl apply -f disallow-privileged.yaml
```

Check:

```bash
kubectl get clusterpolicy
```

You should see:

```text
disallow-privileged
```

---

# 17. Test the Policy

Create a deliberately insecure pod:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: security-test

spec:

  containers:

    - name: test
      image: nginx:1.27

      securityContext:
        privileged: true
```

Try:

```bash
kubectl apply -f security-test.yaml
```

Expected result:

```text
DENIED
```

This proves the policy is actually enforcing something.

---

# 18. Why We Test With an Intentionally Bad Manifest

Because:

```text
Policy exists
```

doesn't mean:

```text
Policy works
```

We need:

```text
Bad deployment
     ↓
Admission controller
     ↓
REJECTED
```

Then:

```text
Good deployment
     ↓
Admission controller
     ↓
ALLOWED
```

This becomes a security test.

---

# 19. Policy #2 — Require Non-Root

Our application standard is:

```yaml
runAsNonRoot: true
```

We can create a policy requiring it.

Conceptually:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:

  validationFailureAction: Enforce

  background: true

  rules:

    - name: require-run-as-non-root

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:

        message: "Containers must run as non-root."

        pattern:

          spec:
            securityContext:
              runAsNonRoot: true
```

Again, test the exact policy behavior against your pinned Kyverno release before enabling it globally.

---

# 20. Why ClusterPolicy?

We could create:

```text
Policy
```

for one namespace.

Or:

```text
ClusterPolicy
```

for cluster-wide enforcement.

For common organizational security requirements:

```text
ClusterPolicy
```

is useful.

But don't make every policy cluster-wide automatically.

---

# 21. Policy Scope

For example:

```text
Production application namespaces
        |
        +--> strict policies

Development
        |
        +--> warn/audit

Platform namespaces
        |
        +--> different requirements
```

We shouldn't accidentally break:

```text
aws-load-balancer-controller
argocd
fluent-bit
```

with application-specific policies.

---

# 22. Policy #3 — Approved Image Registry

Our production registry:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Policy requirement:

```text
All application images must originate from approved ECR.
```

Conceptually:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-approved-registry

spec:

  validationFailureAction: Enforce

  rules:

    - name: validate-image-registry

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:

        message: "Images must come from the approved ECR registry."

        pattern:

          spec:
            containers:
              - image: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/*"
```

In a real organization, use your actual account/region or approved-registry pattern.

---

# 23. Why Registry Enforcement Matters

Without registry restrictions:

```text
Developer
   |
   +--> Docker Hub
   +--> random private registry
   +--> personal registry
   +--> unknown image
```

With policy:

```text
Developer
   |
   v
Approved ECR
   |
   v
EKS
```

This gives the security team a controlled software supply chain.

---

# 24. Policy #4 — Reject `latest`

We already decided:

```text
latest
```

should not be used in production.

We can enforce this.

Conceptually:

```text
Allowed:

order-service:8f31a2c
order-service@sha256:...

Rejected:

order-service:latest
```

This policy is particularly useful because humans make mistakes.

---

# 25. Digest Policy

For stronger production enforcement:

```text
Production
    ↓
Image must be referenced by digest
```

Example:

```yaml
image: >-
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

instead of:

```yaml
image: order-service:latest
```

This connects our previous chapter directly to admission control.

---

# 26. Important Practical Detail

Don't enable a digest-only policy immediately across the entire cluster.

First verify that:

```text
Helm
GitOps
ArgoCD
Kubernetes
image updates
```

all correctly handle digests.

Our rollout:

```text
DEV
 ↓
Test digest deployment
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

Then enforce it.

---

# 27. Policy #5 — Require Resource Requests

A production pod should have:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

Why enforce this?

Because missing requests can cause:

```text
Poor scheduling
Capacity uncertainty
Noisy-neighbor problems
HPA problems
Cluster autoscaling problems
```

---

# 28. Required Resources Policy

Conceptually:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resources

spec:

  validationFailureAction: Enforce

  rules:

    - name: require-resources

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:

        message: "CPU and memory requests/limits are required."

        pattern:

          spec:
            containers:
              - resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

Test this carefully against your exact policy engine version.

---

# 29. Policy #6 — Required Labels

Our application labels should be standardized.

Example:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/part-of: ecommerce-platform
  app.kubernetes.io/managed-by: argocd
  company-ecommerce.io/team: backend-team
  company-ecommerce.io/environment: prod
```

Why?

Labels support:

```text
Monitoring
Cost allocation
Troubleshooting
RBAC
NetworkPolicy
GitOps
Inventory
Ownership
```

---

# 30. Required Ownership

We should know:

```text
Who owns this service?
```

Example:

```yaml
company-ecommerce.io/team: backend-team
```

Then during an incident:

```text
Pod
 ↓
Service
 ↓
Team
 ↓
Owner
```

This is operationally important.

---

# 31. Policy #7 — Require SecurityContext

We want every application container to have:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

At the pod level:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

The policy should ensure these baseline controls exist.

---

# 32. Production Policy Set

Our application baseline becomes:

```text
POLICY-001
No privileged containers

POLICY-002
Run as non-root

POLICY-003
No privilege escalation

POLICY-004
Approved registry only

POLICY-005
No latest image

POLICY-006
Production image must use digest

POLICY-007
Resources required

POLICY-008
Required labels

POLICY-009
Restricted Pod Security

POLICY-010
Trusted image signature
```

---

# 33. Image Signature Verification

This is the most advanced policy in this chapter.

Our desired flow:

```text
Image
   |
   v
ECR
   |
   v
Signature
   |
   v
Kyverno verification
   |
   +--> Valid?
   |
   +--> Trusted signer?
   |
   v
ALLOW
```

If:

```text
unsigned image
```

then:

```text
REJECT
```

---

# 34. Trusted Signer

We need to define:

```text
Who is allowed to sign production images?
```

For example:

```text
GitHub Actions
company-ecommerce organization
trusted workflow
```

Not:

```text
developer laptop
random CI system
unknown identity
```

The policy therefore establishes a trust boundary.

---

# 35. Conceptual VerifyImages Policy

Kyverno supports image verification policies.

The conceptual policy is:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-order-service-image

spec:

  validationFailureAction: Enforce

  rules:

    - name: verify-image-signature

      match:
        any:
          - resources:
              kinds:
                - Pod

              namespaces:
                - ecommerce-prod

      verifyImages:

        - imageReferences:
            - "123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service*"

          required: true

          # Production signer configuration goes here.
```

The exact `verifyImages` configuration depends on whether we're using:

```text
Cosign key-based signing
```

or:

```text
Sigstore keyless signing
```

and the exact Kyverno version.

We should pin those versions before deploying this as production policy.

---

# 36. Why We Don't Fake the Signature Configuration

Image signing is security-sensitive.

We should **not** invent:

```text
fake public keys
fake issuer values
fake GitHub identity patterns
```

The correct production configuration depends on our selected trust model.

So our implementation process is:

```text
Choose signing model
       ↓
Create trusted identity/key
       ↓
Sign real ECR image
       ↓
Verify manually
       ↓
Test Kyverno verification
       ↓
Enable Enforce
```

---

# 37. Start With Audit Mode

Security policies should normally be rolled out gradually.

First:

```text
Audit
```

Then:

```text
Warn
```

Then:

```text
Enforce
```

Example:

```text
DEV
 ↓
Audit
 ↓
Fix violations
 ↓
QA
 ↓
Audit/Enforce
 ↓
STAGING
 ↓
Enforce
 ↓
PROD
```

This prevents a policy deployment from unexpectedly breaking production.

---

# 38. Policy Exceptions

Real production environments sometimes require exceptions.

For example:

```text
Fluent Bit
AWS Load Balancer Controller
EBS CSI Driver
```

may have different requirements from application pods.

We should not solve this by disabling the entire policy.

Instead:

```text
Policy
   |
   +--> Application namespaces
   |       |
   |       +--> strict
   |
   +--> Platform namespaces
           |
           +--> explicitly defined exceptions
```

Exceptions should be:

```text
documented
reviewed
scoped
time-bound where appropriate
```

---

# 39. GitOps for Policies

Do not manually run:

```bash
kubectl apply -f policy.yaml
```

every time.

Instead:

```text
platform repo
      |
      v
Pull Request
      |
      v
Review
      |
      v
CI validation
      |
      v
ArgoCD
      |
      v
EKS
```

Now the security configuration is auditable.

---

# 40. Recommended Platform Repository

Our platform repo can become:

```text
platform/
├── addons/
│   ├── aws-load-balancer-controller/
│   ├── external-secrets/
│   ├── kyverno/
│   └── argo-rollouts/
│
├── policies/
│   ├── kyverno/
│   │   ├── image/
│   │   │   ├── approved-registry.yaml
│   │   │   ├── no-latest.yaml
│   │   │   ├── require-digest.yaml
│   │   │   └── verify-signature.yaml
│   │   │
│   │   ├── security/
│   │   │   ├── disallow-privileged.yaml
│   │   │   ├── require-non-root.yaml
│   │   │   ├── require-seccomp.yaml
│   │   │   └── disallow-host-namespaces.yaml
│   │   │
│   │   └── resources/
│   │       ├── require-resources.yaml
│   │       └── required-labels.yaml
│   │
│   └── pod-security/
│
└── README.md
```

---

# 41. Policy CI

Before merging a policy:

```text
Pull Request
     |
     v
YAML validation
     |
     v
Kyverno policy validation
     |
     v
Test policies against sample resources
     |
     v
Review
     |
     v
Merge
```

This is extremely important.

A security policy is code.

Treat it like code.

---

# 42. Policy Testing

We should have:

```text
tests/
├── insecure/
│   ├── privileged.yaml
│   ├── root.yaml
│   ├── latest.yaml
│   └── untrusted-registry.yaml
│
└── secure/
    └── order-service.yaml
```

Expected behavior:

```text
insecure/privileged.yaml
       ↓
       FAIL

secure/order-service.yaml
       ↓
       PASS
```

---

# 43. Testing the Complete Order Service

Our Order Service should now pass:

```text
[✓] ECR image
[✓] immutable tag
[✓] digest
[✓] signed image
[✓] non-root
[✓] no privilege escalation
[✓] capabilities dropped
[✓] seccomp
[✓] read-only filesystem
[✓] resource requests
[✓] resource limits
[✓] required labels
[✓] dedicated ServiceAccount
[✓] NetworkPolicy
```

Then:

```text
ArgoCD
   ↓
Kubernetes
   ↓
Admission
   ↓
ALLOW
```

---

# 44. Deliberately Break the Deployment

A good production engineer doesn't only test success.

We'll test:

### Test 1 — Privileged

```yaml
privileged: true
```

Expected:

```text
REJECT
```

### Test 2 — Root

```yaml
runAsUser: 0
```

Expected:

```text
REJECT
```

### Test 3 — Wrong registry

```yaml
image: docker.io/random/image:1.0
```

Expected:

```text
REJECT
```

### Test 4 — Latest

```yaml
image: .../order-service:latest
```

Expected:

```text
REJECT
```

### Test 5 — Missing resources

Expected:

```text
REJECT
```

### Test 6 — Unsigned image

Expected:

```text
REJECT
```

---

# 45. ArgoCD Interaction

One important point:

ArgoCD can say:

```text
Synced
```

but the workload can still fail admission.

For example:

```text
GitOps
   |
   v
ArgoCD
   |
   v
Apply Deployment
   |
   v
Admission Controller
   |
   X
REJECT
```

Therefore:

```text
ArgoCD Sync
```

does not automatically mean:

```text
Application Healthy
```

---

# 46. Troubleshooting Rejected Deployment

Start with ArgoCD:

```bash
argocd app get order-service-prod
```

Then:

```bash
kubectl get events \
  -n ecommerce-prod \
  --sort-by=.lastTimestamp
```

Look for:

```text
admission webhook
policy violation
denied
validation failed
```

---

# 47. Inspect Kyverno

Check:

```bash
kubectl get pods -n kyverno
```

Then:

```bash
kubectl get clusterpolicy
```

For a specific policy:

```bash
kubectl describe clusterpolicy disallow-privileged
```

Also inspect Kyverno logs:

```bash
kubectl logs \
  -n kyverno \
  deployment/kyverno
```

Exact deployment names can vary by Kyverno version/chart configuration.

---

# 48. Check Policy Reports

Kyverno can produce policy reporting resources.

Conceptually:

```bash
kubectl get policyreport -A
```

and:

```bash
kubectl get clusterpolicyreport
```

These help answer:

```text
Which resources violate our policies?
```

This is especially useful when running in:

```text
Audit
```

mode before enforcing policies.

---

# 49. Production Security Flow

Our complete flow is now:

```text
                Developer
                    |
                    v
                GitHub PR
                    |
        +-----------+-----------+
        |           |           |
      Tests      SonarQube    Gitleaks
        |           |           |
        +-----------+-----------+
                    |
                    v
              Docker Build
                    |
              +-----+-----+
              |           |
            Trivy       SBOM
              |           |
              +-----+-----+
                    |
                    v
                   ECR
                    |
                 Digest
                    |
                 Sign
                    |
                    v
                 GitOps
                    |
                  ArgoCD
                    |
                    v
            Kubernetes API
                    |
             Admission Layer
                    |
       +------------+------------+
       |            |            |
       PSA        Kyverno      RBAC
       |            |            |
       +------------+------------+
                    |
                    v
                   Pod
                    |
          +---------+---------+
          |                   |
    SecurityContext      NetworkPolicy
          |                   |
          +---------+---------+
                    |
                Pod Identity
                    |
                  IAM
                    |
              AWS Resources
```

---

# 50. The Four Security Questions

Whenever a production pod is deployed, we should be able to answer four questions.

### 1. What is this image?

```text
ECR repository
+
digest
+
SBOM
```

### 2. Who built it?

```text
CI identity
+
image signature
```

### 3. Is it allowed to run?

```text
Admission policies
```

### 4. What can it access?

```text
ServiceAccount
+
NetworkPolicy
+
Pod Identity
+
IAM
```

That's the security model we are building.

---

# 51. Order Service Final Security Example

Our final production Order Service should conceptually look like:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:

  replicas: 3

  template:

    metadata:

      labels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/part-of: ecommerce-platform
        company-ecommerce.io/team: backend-team
        company-ecommerce.io/environment: prod

    spec:

      serviceAccountName: order-service

      automountServiceAccountToken: false

      securityContext:

        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001

        seccompProfile:
          type: RuntimeDefault

      containers:

        - name: order-service

          image: >-
            123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:EXAMPLE

          securityContext:

            allowPrivilegeEscalation: false

            readOnlyRootFilesystem: true

            capabilities:
              drop:
                - ALL

          resources:

            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: "1"
              memory: 1Gi
```

Then admission control verifies that this workload satisfies our organization's security rules.

---

# 52. Applying This to 21 Microservices

The major advantage is that we don't create 21 independent security systems.

We create:

```text
One security baseline
        |
        v
Admission policies
        |
        +--> auth-service
        +--> user-service
        +--> customer-service
        +--> catalog-service
        +--> product-service
        +--> inventory-service
        +--> pricing-service
        +--> promotion-service
        +--> cart-service
        +--> order-service
        +--> payment-service
        +--> fraud-service
        +--> shipping-service
        +--> delivery-service
        +--> notification-service
        +--> search-service
        +--> recommendation-service
        +--> review-service
        +--> tax-service
        +--> invoice-service
        +--> reporting-service
```

The policy automatically protects the platform.

---

# 53. Security Policy Lifecycle

We should manage policies just like application code:

```text
Create policy
     ↓
Test
     ↓
PR
     ↓
Security review
     ↓
Audit
     ↓
DEV
     ↓
QA
     ↓
STAGING
     ↓
PROD
     ↓
Monitor
     ↓
Update
```

Never treat security policies as:

```text
one-time kubectl commands
```

---

# 54. Production Checklist

## Pod Security

```text
[✓] Restricted Pod Security
[✓] Non-root
[✓] No privileged containers
[✓] No privilege escalation
[✓] Capabilities dropped
[✓] Seccomp RuntimeDefault
[✓] Read-only root filesystem
[✓] No unnecessary host namespaces
```

## Image Security

```text
[✓] Approved ECR
[✓] Immutable image
[✓] Digest
[✓] Trivy
[✓] ECR scanning
[✓] SBOM
[✓] Image signing
[✓] Signature verification
```

## Admission

```text
[✓] Kyverno installed
[✓] Policies stored in Git
[✓] Policy CI
[✓] Audit phase
[✓] Enforcement phase
[✓] Exceptions documented
[✓] Policy reports monitored
```

## Deployment

```text
[✓] GitOps
[✓] ArgoCD
[✓] No direct production kubectl deployment
[✓] Production approval
[✓] Jira change
[✓] Rollback procedure
```

---

# 55. Final Architecture After Chapter 88

We now have a complete security chain:

```text
                 SOURCE
                   |
                   v
             GitHub Security
                   |
        +----------+----------+
        |          |          |
     Review     Sonar       Gitleaks
        |          |          |
        +----------+----------+
                   |
                   v
              Docker Build
                   |
             +-----+-----+
             |           |
           Trivy       SBOM
             |           |
             +-----+-----+
                   |
                   v
                  ECR
                   |
             Immutable Digest
                   |
              Image Signature
                   |
                   v
                 GitOps
                   |
                 ArgoCD
                   |
                   v
          Kubernetes Admission
                   |
       +-----------+-----------+
       |           |           |
      PSA       Kyverno      RBAC
       |           |           |
       +-----------+-----------+
                   |
                   v
                  Pod
                   |
       +-----------+-----------+
       |                       |
 SecurityContext         NetworkPolicy
       |                       |
       +-----------+-----------+
                   |
              Pod Identity
                   |
                 IAM
                   |
             AWS Resources
```

---

# 56. What We Have Achieved

At this point, our project has moved beyond:

> "I know Kubernetes security."

We now have a model for:

> **How an enterprise can prevent insecure workloads from reaching an AWS EKS production cluster.**

The important chain is:

```text
SCAN
  ↓
SIGN
  ↓
VERIFY
  ↓
ENFORCE
```

And that is exactly why Chapter 88 follows Chapter 87.

---

## Next: Chapter 89 — Kubernetes RBAC + AWS IAM Deep Dive

We'll now go deep into **identity and permissions**, including:

```text
Human
  ↓
AWS IAM / SSO
  ↓
EKS Access

GitHub Actions
  ↓
OIDC
  ↓
AWS IAM Role
  ↓
ECR

ArgoCD
  ↓
Kubernetes RBAC

Application Pod
  ↓
ServiceAccount
  ↓
EKS Pod Identity
  ↓
IAM Role
  ↓
SQS / S3 / Secrets Manager
```

We'll build the actual permission model for **Order Service**, then generalize it across all 21 services.
