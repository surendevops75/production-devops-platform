# 86 — Production Kubernetes Security Hardening

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Scope:** Kubernetes workload and cluster security
> **Previous chapter:** 85 — Production Networking, Security Groups and Network Policies
> **Next chapter:** 87 — Container Image Security, SBOM and Image Signing

---

## 1. Objective

In the previous chapter, we secured the **network path**.

Now we secure the **workloads running inside Kubernetes**.

The goal is that even if:

* a container is compromised,
* an application has a vulnerability,
* a developer accidentally deploys an insecure manifest,
* a service account is misused,
* an attacker gains access to a pod,

the attacker should have as little ability as possible to:

* become root,
* access the host,
* access other pods,
* access Kubernetes APIs,
* access secrets,
* modify infrastructure,
* move laterally,
* escape the container.

Our security layers become:

```text
                    Internet
                       |
                  CloudFront/WAF
                       |
                      ALB
                       |
                Security Groups
                       |
                    AWS VPC
                       |
                     EKS
                       |
        +--------------+--------------+
        |                             |
   Kubernetes RBAC             NetworkPolicy
        |                             |
        +--------------+--------------+
                       |
                     Pod
                       |
        +--------------+--------------+
        |              |              |
 SecurityContext   ServiceAccount   Container
        |              |              |
   Non-root        Pod Identity    Read-only FS
   Seccomp         IAM Role       No capabilities
   Capabilities    Least privilege
```

---

# 2. Security Defense-in-Depth Model

For each microservice, security should exist at multiple layers.

```text
Source Code
    |
    +--> SonarQube
    +--> Dependency Scan
    +--> Gitleaks
    |
Docker Image
    |
    +--> Trivy
    +--> ECR Scan
    +--> SBOM
    +--> Image Signing
    |
Kubernetes
    |
    +--> Pod Security
    +--> SecurityContext
    +--> RBAC
    +--> NetworkPolicy
    +--> Admission Policy
    |
AWS
    |
    +--> IAM
    +--> Pod Identity
    +--> KMS
    +--> Security Groups
    +--> CloudTrail
```

No single security tool provides complete protection.

---

# 3. Pod Security Standards

Kubernetes provides three Pod Security Standards levels:

```text
Privileged
Baseline
Restricted
```

For application workloads, our target is:

```text
Restricted
```

The important idea is:

> Application pods should run with the minimum privileges necessary.

Our 21 application services should generally be designed to satisfy the Restricted profile.

---

# 4. Namespace Security

We don't want every workload to run in one namespace.

Example:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod

argocd
monitoring
logging
security
```

Application namespaces:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

Platform namespaces:

```text
argocd
monitoring
logging
security
```

---

# 5. Enable Pod Security Admission

For production application namespaces:

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

This gives us three levels of protection:

### enforce

Violating pods are rejected.

### audit

Violations are recorded for auditing.

### warn

Developers receive warnings.

---

# 6. Important Rollout Strategy

Don't immediately apply:

```text
enforce=restricted
```

to a large existing production namespace without testing.

First use:

```text
warn
audit
```

Fix workloads.

Then enable:

```text
enforce
```

A realistic rollout:

```text
Development
   |
   v
warn
   |
   v
audit
   |
   v
Fix workloads
   |
   v
QA
   |
   v
STAGING
   |
   v
PRODUCTION enforce
```

---

# 7. SecurityContext

This is one of the most important Kubernetes security configurations.

Example:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault
```

Container-level:

```yaml
containers:
  - name: order-service
    image: example/order-service:8f31a2c

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

This is much safer than:

```yaml
securityContext:
  privileged: true
```

---

# 8. What Does `runAsNonRoot` Do?

Without security restrictions, a container may run as:

```text
root
UID 0
```

If an attacker compromises the application, they initially have root privileges **inside the container**.

Instead:

```yaml
runAsNonRoot: true
runAsUser: 10001
```

means:

```text
Container
   |
   +-- UID 10001
       |
       +-- NOT root
```

---

# 9. Dockerfile Must Match Kubernetes Security

Our Dockerfile already uses:

```dockerfile
USER 10001
```

Example:

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

This is important.

Security should not exist only in Kubernetes.

We should secure the image itself.

---

# 10. Why Both Dockerfile and Kubernetes Security?

Because defense-in-depth.

```text
Dockerfile
    |
    +--> USER 10001
    |
    v
Container
    |
    +--> Kubernetes runAsNonRoot
    |
    +--> allowPrivilegeEscalation=false
    |
    +--> capabilities DROP ALL
    |
    +--> seccomp RuntimeDefault
```

If one layer is accidentally removed, another layer can still protect us.

---

# 11. `allowPrivilegeEscalation: false`

Use:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

This prevents a process from gaining additional privileges through mechanisms such as setuid/setgid binaries.

For normal application workloads:

```yaml
allowPrivilegeEscalation: false
```

should be the default.

---

# 12. Linux Capabilities

Linux capabilities divide root privileges into smaller permissions.

Instead of giving a process full root privileges, Linux can provide specific capabilities.

For normal Java/Node/Python application containers:

```yaml
capabilities:
  drop:
    - ALL
```

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

Only add a capability if the application genuinely requires it.

Do not blindly add:

```yaml
NET_ADMIN
SYS_ADMIN
SYS_PTRACE
```

These can significantly increase the attack surface.

---

# 13. `readOnlyRootFilesystem`

Use:

```yaml
readOnlyRootFilesystem: true
```

This means:

```text
/
├── application files       READ ONLY
├── system files            READ ONLY
└── writable directories    explicitly mounted
```

An attacker cannot simply modify arbitrary files in the container filesystem.

---

# 14. Temporary Writable Directory

Java applications may sometimes need temporary storage.

Instead of disabling:

```yaml
readOnlyRootFilesystem
```

we can provide:

```yaml
volumes:
  - name: tmp
    emptyDir: {}
```

Mount:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

Example:

```yaml
containers:
  - name: order-service

    securityContext:
      readOnlyRootFilesystem: true

    volumeMounts:
      - name: tmp
        mountPath: /tmp

volumes:
  - name: tmp
    emptyDir: {}
```

Now:

```text
Container filesystem
        |
        +-- /app       READ ONLY
        +-- /etc       READ ONLY
        +-- /tmp       WRITABLE
```

---

# 15. Seccomp

Linux provides a mechanism called:

```text
seccomp
```

It restricts system calls that a process can make.

For normal application containers:

```yaml
seccompProfile:
  type: RuntimeDefault
```

Example:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

Avoid:

```yaml
seccompProfile:
  type: Unconfined
```

unless there is a specific, documented requirement.

---

# 16. Privileged Containers

Never casually deploy:

```yaml
securityContext:
  privileged: true
```

A privileged container has significantly more access to the underlying host.

For application services:

```text
privileged: false
```

should be the expectation.

---

# 17. Host Network

Avoid:

```yaml
hostNetwork: true
```

for normal application workloads.

Normally:

```text
Pod network
   |
VPC CNI
   |
AWS networking
```

is sufficient.

`hostNetwork: true` makes the container use the node's network namespace and can increase security and operational complexity.

---

# 18. Host PID

Avoid:

```yaml
hostPID: true
```

Normal application containers do not need access to the host process namespace.

---

# 19. Host Path

Be extremely careful with:

```yaml
hostPath:
```

Example:

```yaml
volumes:
  - name: host
    hostPath:
      path: /
```

This can expose host filesystem data to the container.

For application workloads:

```text
Avoid hostPath
```

unless there is a documented infrastructure-level requirement.

---

# 20. Production Order-Service SecurityContext

Our Order Service can use:

```yaml
spec:
  template:
    spec:

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: order-service

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true

            capabilities:
              drop:
                - ALL

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}
```

This becomes our baseline application security configuration.

---

# 21. Service Accounts

Every application should have its own Kubernetes ServiceAccount.

Don't use:

```yaml
serviceAccountName: default
```

for production applications.

Instead:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: ecommerce-prod
```

Then:

```yaml
spec:
  template:
    spec:
      serviceAccountName: order-service
```

---

# 22. Why Separate Service Accounts?

Imagine:

```text
order-service
payment-service
notification-service
```

If all use:

```text
default
```

we lose clear workload identity.

Instead:

```text
order-service
     |
     +--> order-service ServiceAccount

payment-service
     |
     +--> payment-service ServiceAccount

notification-service
     |
     +--> notification-service ServiceAccount
```

This allows least-privilege IAM and Kubernetes RBAC.

---

# 23. AWS Pod Identity

Our architecture uses:

```text
Pod
 |
 v
ServiceAccount
 |
 v
EKS Pod Identity
 |
 v
IAM Role
 |
 v
AWS API
```

For Order Service:

```text
order-service
      |
      v
order-service ServiceAccount
      |
      v
EKS Pod Identity Association
      |
      v
order-service IAM Role
      |
      +--> SQS SendMessage
      +--> Secrets Manager GetSecretValue
      +--> S3 specific prefix
```

The pod does **not** contain:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

---

# 24. Kubernetes RBAC

There are two separate concepts:

```text
AWS IAM
   |
   +--> AWS resources

Kubernetes RBAC
   |
   +--> Kubernetes API resources
```

Don't confuse them.

For example:

```text
S3
SQS
Secrets Manager
RDS
```

are AWS resources.

Whereas:

```text
Pods
Deployments
Services
ConfigMaps
Secrets
```

are Kubernetes resources.

---

# 25. Don't Give Applications Kubernetes Admin

Never give an application:

```text
cluster-admin
```

unless it is an actual platform component that genuinely requires it.

A normal Order Service should not be able to execute:

```bash
kubectl delete namespace production
```

or:

```bash
kubectl get secrets --all-namespaces
```

---

# 26. Example Minimal RBAC

Suppose an internal controller needs to read ConfigMaps.

Create:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: ecommerce-prod

rules:
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch
```

Bind it:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: ecommerce-prod

subjects:
  - kind: ServiceAccount
    name: order-service
    namespace: ecommerce-prod

roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

Now:

```text
order-service
     |
     v
order-service ServiceAccount
     |
     v
RoleBinding
     |
     v
config-reader Role
```

---

# 27. Don't Give Unnecessary Secret Access

For example:

```yaml
resources:
  - secrets
verbs:
  - get
  - list
  - watch
```

should not automatically be added to every application.

If the application receives its database password through:

```text
External Secrets Operator
```

then the application may not need Kubernetes API access to Secrets at all.

That's preferable.

---

# 28. Secrets Architecture

Our architecture is:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Order Service
```

AWS access:

```text
ESO
 |
 +--> Pod Identity
 |
 +--> IAM
 |
 +--> Secrets Manager
```

The application itself doesn't need broad access to the Kubernetes API.

---

# 29. Admission Control

Security checks should happen **before** an insecure workload reaches production.

Architecture:

```text
Developer
   |
   v
GitHub PR
   |
   +--> CI
   |     |
   |     +--> Trivy
   |     +--> SonarQube
   |     +--> Gitleaks
   |
   v
GitOps PR
   |
   +--> Helm validation
   +--> Policy validation
   |
   v
ArgoCD
   |
   v
Kubernetes Admission
   |
   +--> Security policies
   |
   v
Pod
```

---

# 30. Policy Engines

Common Kubernetes policy approaches include:

```text
Kyverno
OPA Gatekeeper
ValidatingAdmissionPolicy
```

For our project, a policy engine such as **Kyverno** can be used to enforce organizational rules.

Examples:

```text
No privileged containers
No latest image tags
Must run as non-root
Must have resource requests
Must have approved registries
Must have securityContext
```

---

# 31. Example Security Policy Concept

Requirement:

```text
Containers must not run privileged.
```

A policy can reject:

```yaml
securityContext:
  privileged: true
```

This means the developer can't accidentally bypass the baseline just because the manifest passed Helm lint.

---

# 32. Image Registry Restriction

Our production containers should come from:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

We don't want arbitrary production images such as:

```text
random-user/random-image
```

A policy can require:

```text
approved registry only
```

Conceptually:

```text
Allowed:

123456789012.dkr.ecr.ap-south-1.amazonaws.com/*

Rejected:

docker.io/randomuser/*
```

---

# 33. Don't Use `latest`

Production:

```yaml
image:
  repository: ...
  tag: latest
```

should be rejected.

Instead:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:..."
```

Even better:

```text
repository@sha256:<digest>
```

This connects directly to our previous immutable-image chapter.

---

# 34. Required Resource Requests

A policy can require:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi
```

Why?

Without requests:

```text
Scheduler
   |
   +--> poor scheduling decisions
   +--> noisy neighbor risk
   +--> unstable capacity planning
```

Resource requests also support:

```text
HPA
Cluster Autoscaler
Karpenter
```

---

# 35. Security + CI

Our CI security pipeline becomes:

```text
Developer
   |
   v
Pull Request
   |
   +--> Unit Tests
   |
   +--> SonarQube
   |
   +--> Dependency Scan
   |
   +--> Gitleaks
   |
   +--> Docker Build
   |
   +--> Trivy
   |
   +--> SBOM
   |
   +--> Image Push
   |
   v
ECR
```

---

# 36. SonarQube

SonarQube focuses primarily on:

```text
Source code quality
Security issues
Code smells
Bugs
Duplicated code
Maintainability
```

For Java:

```text
Maven
 |
 v
Sonar Scanner
 |
 v
SonarQube
```

The scanner normally executes in the **CI environment**.

SonarQube itself is a centralized platform service.

It does not need to run inside every application pod.

---

# 37. Trivy

Trivy can scan:

```text
Container images
Filesystem
Dependencies
Kubernetes manifests
Infrastructure configuration
```

Example:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$IMAGE"
```

The scan happens in CI before the image is promoted.

---

# 38. Gitleaks

Gitleaks searches source repositories for potential secrets.

Example:

```bash
gitleaks detect \
  --source . \
  --redact \
  --exit-code 1
```

Potential examples:

```text
AWS access keys
private keys
tokens
passwords
API keys
```

If a developer accidentally commits a secret:

```text
PR
 |
 v
Gitleaks
 |
 v
FAIL
```

---

# 39. SBOM

SBOM means:

> Software Bill of Materials

It tells us what components exist inside our application/image.

Example:

```text
order-service
 |
 +-- Java 21
 +-- Spring Boot
 +-- PostgreSQL driver
 +-- Jackson
 +-- Hibernate
 +-- Logback
```

This becomes very important when a vulnerability is discovered in a dependency.

---

# 40. Example Trivy SBOM

Conceptually:

```bash
trivy image \
  --format cyclonedx \
  --output sbom.json \
  "$IMAGE"
```

The generated SBOM can be stored as a CI artifact or integrated into the organization's software supply-chain system.

For production, define:

```text
Who stores it?
How long?
Who can access it?
How is it associated with the image digest?
```

---

# 41. Image Signing

We want to know:

```text
Was this image actually produced by our CI pipeline?
```

Instead of trusting only:

```text
image tag
```

we can use:

```text
image signing
```

A common modern approach is:

```text
Sigstore / Cosign
```

Conceptual flow:

```text
GitHub Actions
      |
      v
Build image
      |
      v
Push ECR
      |
      v
Get digest
      |
      v
Sign digest
      |
      v
ECR
      |
      v
Admission verification
      |
      v
EKS
```

---

# 42. Why Sign the Digest?

Suppose:

```text
order-service:1.0.0
```

points to:

```text
sha256:AAA
```

Later someone changes the tag:

```text
order-service:1.0.0
        |
        v
sha256:BBB
```

A tag alone is not enough to establish artifact identity.

Digest:

```text
sha256:AAA
```

is immutable.

Signing the digest gives us:

```text
Exact artifact
+
Proof of signing identity
```

---

# 43. Production Supply Chain

Our final security supply chain becomes:

```text
GitHub
   |
   +--> PR review
   |
   +--> Unit tests
   |
   +--> SonarQube
   |
   +--> Dependency scanning
   |
   +--> Gitleaks
   |
   v
Docker Build
   |
   +--> Trivy
   +--> SBOM
   |
   v
ECR
   |
   +--> ECR scanning
   |
   +--> Image digest
   |
   +--> Signature
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
Admission Policy
   |
   v
EKS
```

---

# 44. Example Production Deployment SecurityContext

Our Helm Deployment should ultimately render something similar to:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce-prod

spec:
  replicas: 3

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service

  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service

    spec:

      serviceAccountName: order-service

      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001

        seccompProfile:
          type: RuntimeDefault

      containers:

        - name: order-service

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:EXAMPLE

          ports:
            - containerPort: 8080

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
              cpu: 1000m
              memory: 1Gi

          volumeMounts:

            - name: tmp
              mountPath: /tmp

      volumes:

        - name: tmp
          emptyDir: {}
```

---

# 45. Why `automountServiceAccountToken: false`?

By default, Kubernetes may make a ServiceAccount token available to a pod.

But if our application does not need to communicate with the Kubernetes API:

```yaml
automountServiceAccountToken: false
```

reduces unnecessary exposure.

Important:

If the application genuinely needs Kubernetes API access, don't disable it blindly.

Instead:

```text
Does application need Kubernetes API?
        |
       / \
     YES  NO
      |    |
 RBAC     Disable token
```

For our Order Service:

```text
Kubernetes API access?
NO
```

Therefore:

```yaml
automountServiceAccountToken: false
```

is appropriate unless another integration requires it.

---

# 46. Kubernetes NetworkPolicy + Pod Security

These controls solve different problems.

### SecurityContext

Controls:

```text
What can the container do?
```

### NetworkPolicy

Controls:

```text
Who can communicate with the pod?
```

### RBAC

Controls:

```text
What can the identity do through Kubernetes API?
```

### IAM

Controls:

```text
What can the workload do in AWS?
```

Together:

```text
                 Pod
                  |
      +-----------+-----------+
      |           |           |
 Security      Network       IAM
 Context       Policy       Identity
      |           |           |
 Container      Traffic      AWS APIs
```

---

# 47. Security Baseline for Every Microservice

Every application should target:

```text
[✓] Non-root
[✓] Fixed UID
[✓] No privilege escalation
[✓] Read-only root filesystem
[✓] Drop capabilities
[✓] RuntimeDefault seccomp
[✓] Dedicated ServiceAccount
[✓] No unnecessary ServiceAccount token
[✓] Resource requests
[✓] Resource limits
[✓] Readiness probe
[✓] Liveness probe
[✓] Startup probe where needed
[✓] NetworkPolicy
[✓] IAM least privilege
[✓] Secrets Manager
[✓] Immutable image
[✓] Trivy scan
[✓] SBOM
[✓] Image signing
```

---

# 48. What Should NOT Exist in Production?

Avoid:

```yaml
privileged: true
```

Avoid:

```yaml
hostNetwork: true
```

Avoid:

```yaml
hostPID: true
```

Avoid:

```yaml
hostPath: /
```

Avoid:

```yaml
runAsUser: 0
```

Avoid:

```yaml
allowPrivilegeEscalation: true
```

Avoid:

```yaml
image: application:latest
```

Avoid:

```text
AWS access keys inside Kubernetes Secrets
```

Avoid:

```text
cluster-admin for application ServiceAccounts
```

Avoid:

```text
0.0.0.0/0 -> PostgreSQL
```

Avoid:

```text
secrets in Git
```

---

# 49. Security Validation in CI

We can validate Kubernetes manifests before GitOps merge.

Example:

```bash
helm lint helm/order-service
```

Then:

```bash
helm template order-service helm/order-service \
  -f values-prod.yaml \
  > rendered.yaml
```

Then policy validation:

```text
rendered.yaml
      |
      +--> kubeconform
      |
      +--> Kyverno policy tests
      |
      +--> security checks
```

The important concept is:

> Don't wait until production to discover that a manifest violates security requirements.

---

# 50. Example Security Review

Suppose a developer submits:

```yaml
containers:
  - name: order-service
    image: order-service:latest

    securityContext:
      privileged: true
```

CI/policy should identify:

```text
FAIL

Reason 1:
latest tag is prohibited.

Reason 2:
privileged container is prohibited.

Reason 3:
approved ECR registry required.

Reason 4:
runAsNonRoot missing.

Reason 5:
resource requests missing.
```

The PR should not proceed.

---

# 51. Troubleshooting: Pod Rejected

Check:

```bash
kubectl get events -n ecommerce-prod \
  --sort-by=.lastTimestamp
```

Then:

```bash
kubectl describe pod <pod-name> -n ecommerce-prod
```

Look for:

```text
Forbidden
PodSecurity
admission webhook
policy violation
```

---

# 52. Troubleshooting: Container Cannot Start

Check:

```bash
kubectl describe pod <pod-name> -n ecommerce-prod
```

Then:

```bash
kubectl logs <pod-name> -n ecommerce-prod
```

Potential issue:

```text
readOnlyRootFilesystem=true
```

Application tries:

```text
write /app/file
```

and fails.

Correct solution:

```text
Don't disable security immediately.

Determine what must be writable.
Mount a specific emptyDir/PVC.
```

---

# 53. Troubleshooting: Permission Denied

Possible causes:

```text
runAsUser changed
file ownership incorrect
filesystem permissions incorrect
application expects root
```

Check:

```bash
kubectl exec -it <pod> -n ecommerce-prod -- id
```

Expected:

```text
uid=10001
gid=10001
```

Check filesystem:

```bash
kubectl exec -it <pod> -n ecommerce-prod -- ls -la /app
```

---

# 54. Troubleshooting: AWS Access Denied

If Order Service receives:

```text
AccessDeniedException
```

check:

```text
Pod
 |
 v
ServiceAccount
 |
 v
Pod Identity
 |
 v
IAM Role
 |
 v
IAM Policy
 |
 v
AWS Resource
```

First verify the identity and then inspect the policy.

Don't immediately give:

```text
AdministratorAccess
```

to make the error disappear.

---

# 55. Troubleshooting: Network Failure

Use the previous chapter's flow:

```text
DNS
 |
 v
Route
 |
 v
NACL
 |
 v
Security Group
 |
 v
NetworkPolicy
 |
 v
Service
 |
 v
EndpointSlice
 |
 v
Pod
 |
 v
Application
```

Security hardening should never become:

> “If something fails, disable security.”

Instead identify the exact control blocking the request.

---

# 56. Production Security Review Process

Before a new microservice reaches production:

```text
Developer
   |
   v
Code Review
   |
   v
CI Security
   |
   +--> SonarQube
   +--> Gitleaks
   +--> Dependency scan
   +--> Trivy
   +--> SBOM
   |
   v
ECR
   |
   +--> Scan
   +--> Sign
   |
   v
GitOps PR
   |
   +--> Manifest validation
   +--> Security policies
   |
   v
ArgoCD
   |
   v
Admission Controls
   |
   v
EKS
```

---

# 57. Production Security Checklist

## Kubernetes

```text
[ ] Pod Security Admission enabled
[ ] Restricted policy for application namespaces
[ ] No privileged application containers
[ ] Non-root containers
[ ] Fixed UID/GID
[ ] No privilege escalation
[ ] Read-only root filesystem
[ ] Capabilities dropped
[ ] RuntimeDefault seccomp
[ ] No unnecessary host namespaces
[ ] No unnecessary hostPath
```

## Identity

```text
[ ] Dedicated ServiceAccount
[ ] Kubernetes RBAC least privilege
[ ] AWS Pod Identity
[ ] IAM least privilege
[ ] No static AWS credentials
[ ] ServiceAccount token disabled where unnecessary
```

## Network

```text
[ ] NetworkPolicy
[ ] Default deny where appropriate
[ ] Restricted ingress
[ ] Restricted egress
[ ] Private nodes
[ ] Private RDS
[ ] Private Redis
[ ] Controlled ALB ingress
```

## Supply Chain

```text
[ ] Protected GitHub branches
[ ] Code review
[ ] SonarQube
[ ] Gitleaks
[ ] Dependency scanning
[ ] Trivy
[ ] ECR scanning
[ ] SBOM
[ ] Immutable image
[ ] Image signing
[ ] Approved registry
```

## Runtime

```text
[ ] Resource requests
[ ] Resource limits
[ ] Readiness probe
[ ] Liveness probe
[ ] Startup probe where required
[ ] HPA
[ ] PDB
[ ] Multi-AZ scheduling
```

---

# 58. Final Architecture

After this chapter, our production architecture is:

```text
                        GitHub
                           |
             +-------------+-------------+
             |             |             |
          SonarQube     Gitleaks       Tests
             |             |             |
             +-------------+-------------+
                           |
                      Docker Build
                           |
                    +------+------+
                    |             |
                  Trivy         SBOM
                    |             |
                    +------+------+
                           |
                          ECR
                           |
                    Image Signature
                           |
                           v
                        GitOps
                           |
                         ArgoCD
                           |
                   Admission Policies
                           |
                           v
                         EKS
                           |
          +----------------+----------------+
          |                |                |
      SecurityContext  NetworkPolicy       RBAC
          |                |                |
          +----------------+----------------+
                           |
                         Pod
                           |
                  Dedicated ServiceAccount
                           |
                      Pod Identity
                           |
                       IAM Role
                           |
          +----------------+----------------+
          |                |                |
         SQS             S3          Secrets Manager
```

---

# 59. The Key Production Principle

The most important lesson from this chapter is:

> **Security is not one tool. It is a chain of controls from source code to runtime.**

For our Order Service:

```text
Developer
   ↓
GitHub PR
   ↓
Code Review
   ↓
Tests
   ↓
SonarQube
   ↓
Gitleaks
   ↓
Dependency Scan
   ↓
Docker Build
   ↓
Trivy
   ↓
SBOM
   ↓
ECR
   ↓
Image Signing
   ↓
GitOps PR
   ↓
ArgoCD
   ↓
Admission Policy
   ↓
Restricted Pod
   ↓
NetworkPolicy
   ↓
Pod Identity
   ↓
Least-Privilege IAM
   ↓
AWS Resources
```

That is the security model we should replicate for **all 21 microservices**, rather than configuring security independently and inconsistently for each service.

---

## Next Chapter

**87 — Container Image Security, SBOM and Image Signing**

We will go deeper into the actual image supply chain:

```text
Dockerfile
   ↓
Minimal Base Image
   ↓
Multi-stage Build
   ↓
Trivy
   ↓
ECR Scan
   ↓
SBOM
   ↓
Cosign/Image Signing
   ↓
Signature Verification
   ↓
GitOps
   ↓
Admission Control
   ↓
EKS
```

including the **actual CI workflow, ECR configuration, Trivy thresholds, SBOM generation, Cosign signing, verification, and how to prevent an unsigned/untrusted image from being deployed to production**.
