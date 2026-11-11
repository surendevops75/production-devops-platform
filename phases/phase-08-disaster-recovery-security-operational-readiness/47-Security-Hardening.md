# 47 — Security Hardening

> **Enterprise E-Commerce Platform | AWS EKS | Kubernetes | Terraform | GitHub Actions | GitLab CI/CD | Jenkins | ECR | Trivy | SonarQube | Secrets Manager | KMS | ArgoCD**

Security hardening means taking the platform we have designed so far and asking:

> **What happens if someone makes a mistake, a credential leaks, a container is compromised, an application is attacked, or an attacker gains access to one component?**

Production security is not one tool.

It is a layered system:

```text
Developer
   │
   ▼
Source Code
   │
   ▼
CI/CD
   │
   ▼
Container
   │
   ▼
ECR
   │
   ▼
EKS
   │
   ├── IAM
   ├── RBAC
   ├── NetworkPolicy
   ├── Security Groups
   ├── Secrets
   └── Runtime Security
   │
   ▼
Application
   │
   ▼
RDS / Redis / SQS / S3
```

---

# 1. Security Principles

Our platform should follow these principles:

```text
Least privilege
Defense in depth
Zero trust
Immutable artifacts
Secure defaults
Separation of duties
Encryption
Strong authentication
Continuous scanning
Auditing
Fast revocation
```

The most important principle:

> **Assume every layer can eventually fail or be compromised, and limit the blast radius.**

---

# 2. Security Is Not Just Trivy

A common beginner approach is:

```text
Install Trivy
     ↓
Scan image
     ↓
"We are secure"
```

That's not enough.

Trivy helps identify vulnerabilities.

But security also requires:

```text
IAM
RBAC
NetworkPolicy
Secrets
KMS
container configuration
dependency security
image provenance
runtime controls
logging
auditing
```

---

# 3. Security Layers

For our platform:

```text
                    SECURITY
                       │
      ┌────────────────┼────────────────┐
      ▼                ▼                ▼
     AWS            Kubernetes        CI/CD
      │                │                │
     IAM              RBAC           GitHub
     KMS              Network        GitLab
     VPC              Security       Jenkins
     SG               Secrets
      │                │                │
      └────────────────┼────────────────┘
                       ▼
                    Runtime
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
            App       Data     Network
```

---

# 4. AWS Account Security

Before EKS:

```text
AWS Account
   │
   ├── IAM
   ├── CloudTrail
   ├── Guardrails
   ├── KMS
   └── Organizations/SCP where applicable
```

Production should not be operated using the root user.

Use:

```text
Human
  ↓
SSO / federated identity
  ↓
IAM role
```

rather than:

```text
Human
  ↓
permanent AWS access key
```

---

# 5. Root User

The AWS root account should be treated as highly restricted.

Use it only for account-level tasks that specifically require root.

Security controls should include:

```text
MFA
strong recovery controls
no routine access keys
restricted operational use
```

---

# 6. Human Access

Developers should not automatically have:

```text
AdministratorAccess
```

Production access should be role-based.

Example:

```text
Developer
   ↓
Read-only production

DevOps
   ↓
Infrastructure role

SRE
   ↓
Operational role

Security
   ↓
Security/audit role
```

Permissions should be based on actual responsibilities.

---

# 7. IAM Least Privilege

Always ask:

```text
WHO
  ↓
CAN DO WHAT
  ↓
ON WHICH RESOURCE
```

Bad:

```text
Action: *
Resource: *
```

Better:

```text
sqs:SendMessage
Resource:
arn:aws:sqs:ap-south-1:123456789012:order-events
```

---

# 8. EKS Identity Architecture

Our applications should not contain AWS access keys.

Bad:

```text
Pod
 │
 ├── AWS_ACCESS_KEY_ID
 └── AWS_SECRET_ACCESS_KEY
```

Better:

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
AWS API
```

This is the preferred direction for workloads where EKS Pod Identity is supported and appropriate.

---

# 9. Order Service IAM

Suppose `order-service` needs to publish to:

```text
order-events
```

Its IAM role should allow:

```text
sqs:SendMessage
```

only to:

```text
order-events
```

It should not automatically have:

```text
s3:*
ec2:*
iam:*
rds:*
```

---

# 10. Pod Identity

Conceptually:

```text
order-service Pod
       │
       ▼
EKS Pod Identity Association
       │
       ▼
IAM Role
       │
       ▼
SQS
```

For Secrets Manager:

```text
order-service Pod
       │
       ▼
IAM Role
       │
       ▼
Secrets Manager
```

No static AWS credentials are required inside the container.

---

# 11. Kubernetes RBAC

AWS IAM controls AWS resources.

Kubernetes RBAC controls Kubernetes resources.

These are different.

```text
AWS IAM
   ↓
AWS APIs

Kubernetes RBAC
   ↓
Kubernetes API
```

---

# 12. Kubernetes Roles

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-reader
  namespace: production

rules:
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
```

Then bind it to a ServiceAccount.

---

# 13. Don't Give Pods Cluster-Admin

Avoid:

```text
ClusterRole:
  "*"
```

with:

```text
verbs:
  "*"
```

unless there is a very specific platform requirement.

Application Pods generally don't need broad Kubernetes API access.

---

# 14. Service Accounts

Every workload should have an appropriate ServiceAccount.

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
```

Then:

```yaml
spec:
  serviceAccountName: order-service
```

This allows identity to be explicit rather than relying on the namespace default account.

---

# 15. Default Service Account

Don't assume every application should run as:

```text
default
```

Use dedicated identities.

```text
order-service
payment-service
inventory-service
```

can have separate ServiceAccounts and AWS permissions.

---

# 16. Network Security

Our architecture:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Ingress
   │
   ▼
EKS
```

We should not expose every Pod directly to the Internet.

Bad:

```text
Internet
 ├── order-service
 ├── database
 ├── Redis
 ├── Elasticsearch
 └── Kafka
```

Better:

```text
Internet
   │
   ▼
WAF / ALB
   │
   ▼
Ingress
   │
   ▼
Application
   │
   ├── RDS
   ├── Redis
   └── SQS
```

---

# 17. Private EKS Nodes

Our production architecture uses:

```text
Public Subnets
   ↓
ALB

Private Subnets
   ↓
EKS nodes
```

This prevents direct Internet access to worker nodes.

---

# 18. Security Groups

AWS Security Groups provide network-level controls.

Example:

```text
ALB SG
  ↓
EKS workload/node SG
```

RDS:

```text
EKS workload SG
       ↓
RDS SG
       ↓
5432
```

Redis:

```text
EKS workload SG
       ↓
Redis SG
       ↓
6379
```

Only required communication should be allowed.

---

# 19. RDS Security Group

Avoid:

```text
0.0.0.0/0
```

on PostgreSQL port 5432.

Instead:

```text
Source:
approved EKS workload security group

Port:
5432
```

Traffic:

```text
order-service
     │
     ▼
EKS networking
     │
     ▼
RDS SG
     │
     ▼
5432
```

---

# 20. Redis Security Group

Similarly:

```text
EKS
 │
 ▼
Redis SG
 │
 ▼
6379
```

Do not expose Redis directly to the Internet.

---

# 21. Kubernetes NetworkPolicy

Security Groups operate at the AWS network layer.

NetworkPolicy provides Kubernetes-level workload isolation.

Example:

```text
order-service
     │
     ├── allowed → PostgreSQL
     ├── allowed → Redis
     └── allowed → SQS via AWS API
```

But:

```text
order-service
     ✕
payment-service
```

unless communication is required.

---

# 22. Default Deny

A strong Kubernetes security pattern is:

```text
deny by default
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production

spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This means we then explicitly allow required traffic.

---

# 23. Why Default Deny?

Without isolation:

```text
Compromised Pod
      │
      ├── Pod A
      ├── Pod B
      ├── Pod C
      └── Database
```

With appropriate NetworkPolicies:

```text
Compromised Pod
      │
      ├── allowed dependency
      │
      └── blocked unrelated workloads
```

This reduces lateral movement.

---

# 24. NetworkPolicy Example

Suppose only `order-service` should receive traffic from an ingress controller.

Conceptually:

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            name: ingress-system
```

And then explicitly allow required application-to-application traffic.

The exact policy must match the actual CNI/network-policy implementation deployed in EKS.

---

# 25. EKS VPC CNI Security

EKS networking involves:

```text
Pod
 ↓
VPC CNI
 ↓
AWS networking
```

Therefore:

```text
subnet IP capacity
security groups
routing
network policies
```

all matter.

A network failure can be both:

```text
availability problem
+
security problem
```

---

# 26. Container Security

Our Dockerfile uses:

```dockerfile
USER 10001
```

This is important.

Avoid:

```text
USER root
```

unless absolutely required.

---

# 27. Why Non-Root?

If an application is compromised:

```text
Attacker
   ↓
Application process
```

Running as root inside the container can increase the impact of a container breakout or misconfiguration.

Non-root reduces privileges inside the container.

It is not a complete security boundary by itself.

---

# 28. Read-Only Filesystem

Where the application supports it:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Then provide writable temporary storage only where necessary.

For example:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

with:

```yaml
emptyDir: {}
```

This reduces unnecessary filesystem write capability.

---

# 29. Drop Linux Capabilities

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

Only add back capabilities that the application genuinely requires.

---

# 30. Security Context

A production Pod can use controls such as:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

The exact settings must be tested against the application.

---

# 31. Privileged Containers

Avoid:

```yaml
securityContext:
  privileged: true
```

for normal application workloads.

A privileged application Pod has substantially more access to the underlying node.

---

# 32. HostPath

Avoid unnecessary:

```yaml
hostPath:
```

mounts.

A host filesystem mount can expose sensitive node data to a workload.

Our observability components such as Fluent Bit and Node Exporter may legitimately need host access, but those are specialized platform workloads and should be isolated and reviewed carefully.

---

# 33. Secrets

Our secret architecture is:

```text
AWS Secrets Manager
       │
       ▼
EKS Pod Identity
       │
       ▼
External Secrets Operator
       │
       ▼
Kubernetes Secret
       │
       ▼
Application
```

This keeps secret values out of:

```text
Git
Dockerfile
Helm values
ConfigMap
logs
```

---

# 34. Secrets in Environment Variables

Environment variables are convenient:

```text
DB_PASSWORD
REDIS_PASSWORD
```

but can potentially appear through debugging or process/environment inspection.

For highly sensitive workloads, mounted files or another controlled secret-delivery mechanism may sometimes be preferable.

The application should never log secret values.

---

# 35. Secrets Rotation

Suppose:

```text
DB_PASSWORD
```

is rotated.

The flow is:

```text
Secrets Manager
      ↓
new version
      ↓
External Secrets
      ↓
Kubernetes Secret
      ↓
Application
```

But the application may need a restart or connection refresh depending on how the secret is consumed.

For example, an existing HikariCP connection pool won't necessarily magically replace every existing connection credential simply because a Kubernetes Secret changed.

---

# 36. KMS

Use encryption for sensitive data:

```text
RDS
S3
Secrets Manager
EBS
```

where appropriate.

Architecture:

```text
Application
    │
    ▼
AWS service
    │
    ▼
KMS
```

IAM must permit only the required principals to use the key.

---

# 37. ECR Security

Our ECR repositories use:

```text
IMMUTABLE
```

tags.

Example:

```text
order-service:1.6.0
```

should not be silently replaced by another image.

Better still, deployment can use:

```text
order-service@sha256:...
```

to identify the exact image.

---

# 38. ECR Scanning

We already use:

```text
ECR scan on push
```

and:

```text
Trivy
```

in CI.

These provide different control points.

```text
CI
 │
 └── Trivy

ECR
 │
 └── AWS scanning capabilities
```

---

# 39. Trivy's Role

Trivy can scan:

```text
filesystem
dependencies
container images
configuration
Kubernetes/IaC in supported workflows
```

Our pipeline:

```text
Source
  ↓
Trivy filesystem/dependencies
  ↓
Docker build
  ↓
Trivy image
  ↓
ECR
```

---

# 40. Trivy Is a CI Tool

In our architecture:

```text
GitHub/GitLab/Jenkins runner
        │
        ▼
      Trivy
```

Not:

```text
Production Pod
      │
      ▼
Trivy
```

We don't normally install Trivy inside every application container.

---

# 41. SonarQube

SonarQube addresses code quality and certain code security issues.

Our flow:

```text
Source
   ↓
Build/Test
   ↓
SonarQube
   ↓
Quality Gate
```

SonarQube is a centralized platform service.

The scanner runs in CI.

```text
CI runner
   │
   └── Sonar Scanner
             │
             ▼
        SonarQube
```

---

# 42. Secret Scanning

Use:

```text
Gitleaks
```

in CI.

Example:

```text
Developer
   ↓
Git
   ↓
CI
   ↓
Gitleaks
```

Search for:

```text
AWS keys
API tokens
passwords
private keys
connection strings
```

---

# 43. What If a Secret Is Committed?

This is critical.

Do not simply:

```text
delete line
commit again
```

because Git history may still contain the secret.

Immediate response:

```text
1. Revoke/rotate secret
2. Determine exposure
3. Remove from repository history if required
4. Check logs/artifacts
5. Audit usage
6. Replace credential
```

Rotation comes first because the credential may already be compromised.

---

# 44. GitHub/GitLab/Jenkins Security

CI systems have powerful permissions.

Therefore protect:

```text
repository
workflow files
secrets
runners
credentials
deployment permissions
```

A compromised CI pipeline can become a production compromise.

---

# 45. GitHub Actions

Use:

```yaml
permissions:
  contents: read
  id-token: write
```

rather than broad permissions.

The AWS role trust policy should restrict:

```text
repository
branch/environment
organization
```

as appropriate.

---

# 46. Pin Third-Party Actions

Avoid blindly using:

```yaml
uses: some-action@v4
```

for high-security production pipelines when your organization's policy requires immutable references.

Prefer pinning to an approved commit SHA.

Why?

Because a mutable action reference can change over time.

---

# 47. GitHub OIDC

Our CI authentication:

```text
GitHub Actions
      │
      ▼
OIDC token
      │
      ▼
AWS STS
      │
      ▼
IAM Role
```

No permanent:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside GitHub Secrets are needed for this authentication model.

---

# 48. GitLab CI

Similarly:

```text
GitLab Runner
     │
     ▼
OIDC / federation where configured
     │
     ▼
AWS IAM role
```

Avoid long-lived AWS keys when federation can satisfy the workflow.

---

# 49. Jenkins

Jenkins should use:

```text
Jenkins credentials / federation
```

rather than embedding secrets in:

```text
Jenkinsfile
```

Example:

```groovy
withCredentials(...) {
    sh '...'
}
```

Secrets should never be printed.

---

# 50. Jenkins Controller Security

If Jenkins remains in an organization:

```text
Jenkins Controller
       │
       ├── Authentication
       ├── Authorization
       ├── TLS
       ├── Plugin control
       └── Agent isolation
```

The Jenkins controller should not be unnecessarily exposed to the public Internet.

---

# 51. CI Runner Security

CI runners execute arbitrary repository code.

Therefore:

```text
Runner
   ↓
untrusted build commands
```

must be treated carefully.

Avoid allowing untrusted Pull Requests to access:

```text
production credentials
AWS production role
deployment secrets
```

This is one reason protected environments and branch restrictions matter.

---

# 52. Build Environment Separation

A useful model:

```text
PR CI
 │
 └── low privilege

Main build
 │
 └── publish to ECR

Production deployment
 │
 └── separate controlled permissions
```

Don't give every Pull Request:

```text
AdministratorAccess
```

---

# 53. EKS API Security

The EKS Kubernetes API endpoint should be configured intentionally.

Avoid unnecessarily exposing it broadly to the Internet.

Possible architecture:

```text
Admin
  │
  ▼
VPN / corporate network / controlled access
  │
  ▼
EKS API
```

The exact endpoint configuration depends on the organization's network architecture.

---

# 54. Kubernetes API Authentication

Access should be:

```text
Identity
   ↓
AWS / EKS authentication
   ↓
Kubernetes RBAC
```

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

Both are required.

---

# 55. Namespace Isolation

Our namespaces include:

```text
production
monitoring
logging
tracing
argocd
ingress-system
```

Don't give developers unrestricted access across all namespaces.

For example:

```text
developer
  ↓
development namespace
```

while:

```text
production
```

requires stronger access controls.

---

# 56. Production Namespace

Critical workloads:

```text
order-service
payment-service
auth-service
```

should be isolated from unrelated workloads using:

```text
RBAC
NetworkPolicy
resource controls
security contexts
```

---

# 57. Resource Limits Are Also Security Controls

A compromised or buggy application can consume:

```text
CPU
memory
network
```

and affect other workloads.

Use:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

for workloads where appropriate.

---

# 58. ResourceQuota

At namespace level:

```text
production
   ↓
ResourceQuota
```

can prevent uncontrolled resource consumption.

This is particularly useful in shared clusters.

---

# 59. LimitRange

A `LimitRange` can establish defaults or constraints for Pods/containers in a namespace.

This helps prevent workloads from being deployed with:

```text
no resource requests
```

which can negatively affect:

```text
scheduling
HPA
Karpenter
cluster stability
```

---

# 60. Pod Security

Kubernetes supports Pod Security Standards.

A production namespace can be configured toward a restrictive security posture where compatible with the workloads.

The goal is to prevent things such as:

```text
privileged containers
host networking
unsafe host mounts
root execution
```

unless explicitly required.

---

# 61. Admission Control

We can enforce policies before workloads enter the cluster.

Possible policy engines include:

```text
Kyverno
OPA Gatekeeper
```

Examples:

```text
Reject image: latest
Reject privileged Pods
Require resource requests
Require non-root
Require approved registries
Require labels
```

---

# 62. Example Policy

We can enforce:

```text
image repository must be:

123456789012.dkr.ecr.ap-south-1.amazonaws.com/*
```

Then someone cannot accidentally deploy:

```text
docker.io/random-user/malicious-image
```

into production.

---

# 63. Image Tag Policy

Reject:

```text
latest
```

in production.

Require:

```text
1.6.0
```

or preferably:

```text
sha256 digest
```

This improves:

```text
traceability
rollback
reproducibility
security
```

---

# 64. Image Signing

A more advanced supply-chain control is:

```text
Build
  ↓
Sign image
  ↓
ECR
  ↓
Verify signature
  ↓
EKS
```

Tools/ecosystems can include:

```text
Sigstore Cosign
```

The exact signing and admission architecture should be selected and tested during implementation.

---

# 65. SBOM

SBOM = Software Bill of Materials.

It answers:

> What software components are inside this application?

Example:

```text
order-service
 ├── Spring Boot
 ├── PostgreSQL driver
 ├── Jackson
 ├── Netty
 └── other dependencies
```

Generate an SBOM during CI.

---

# 66. Supply Chain

Our secure artifact chain becomes:

```text
Source
  │
  ▼
Build
  │
  ▼
Test
  │
  ▼
SonarQube
  │
  ▼
Dependency Scan
  │
  ▼
Gitleaks
  │
  ▼
Docker
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
  │
  ▼
Verify
  │
  ▼
EKS
```

This is much stronger than:

```text
docker build
docker push
kubectl apply
```

---

# 67. Vulnerability Severity

Not every vulnerability should automatically block every build.

A mature policy considers:

```text
severity
exploitability
runtime exposure
package usage
available fix
business criticality
exception policy
```

For example:

```text
Critical + exploitable + reachable
        ↓
likely block
```

while:

```text
Low + not exploitable + no fix
        ↓
document/monitor
```

The organization's security policy determines the actual gates.

---

# 68. Security Exceptions

Sometimes a vulnerability cannot immediately be fixed.

Don't simply ignore it.

Document:

```text
Vulnerability
Owner
Reason
Risk
Compensating control
Expiry date
Remediation plan
```

Example:

```text
CVE:
XYZ

Owner:
Platform Team

Reason:
No vendor fix yet

Compensating control:
Network isolation

Review:
30 days
```

Exceptions should expire.

---

# 69. Network Segmentation

Our AWS architecture:

```text
Internet
   │
   ▼
Public ALB
   │
   ▼
Private EKS
   │
   ├── RDS
   ├── Redis
   └── AWS services
```

RDS and Redis should not be public.

---

# 70. Egress Control

Security isn't only inbound.

Applications can also attack external systems if compromised.

Example:

```text
Compromised Pod
     │
     ▼
Internet
     │
     ▼
attacker infrastructure
```

Where appropriate, control egress through:

```text
NetworkPolicy
security controls
proxy/firewall architecture
VPC routing
AWS network controls
```

Don't blindly allow unrestricted outbound traffic for every workload when stronger controls are required.

---

# 71. S3 Security

S3 buckets should use:

```text
Block Public Access
Encryption
Least-privilege IAM
Versioning where required
Access logging/auditing
Lifecycle policies
```

Application:

```text
order-service
    │
    ▼
specific S3 bucket
    │
    ▼
specific actions
```

Not:

```text
s3:*
*
```

---

# 72. SQS Security

Application IAM:

```text
order-service
    │
    ▼
sqs:SendMessage
    │
    ▼
order-events
```

Consumer:

```text
notification-service
    │
    ▼
sqs:ReceiveMessage
sqs:DeleteMessage
    │
    ▼
notification queue
```

Different roles can have different permissions.

---

# 73. KMS for Messaging

Where customer/business requirements justify it:

```text
SQS/SNS
   ↓
KMS
```

Use IAM to restrict who can:

```text
encrypt
decrypt
generate data keys
```

---

# 74. RDS Encryption

Our RDS architecture:

```text
EKS
 │
 ▼
RDS PostgreSQL
 │
 └── encrypted storage
       │
       ▼
      KMS
```

Also ensure:

```text
TLS
```

for database connections where supported/configured.

---

# 75. Redis Encryption

For ElastiCache:

```text
Application
    │
    ▼
TLS
    │
    ▼
Redis/Valkey
```

Use authentication/access controls appropriate to the selected service configuration.

---

# 76. Application Security

DevOps cannot solve all application vulnerabilities.

The application team must address:

```text
SQL injection
XSS
authentication
authorization
input validation
CSRF
business logic
dependency vulnerabilities
secure headers
```

DevOps provides security controls around the application.

---

# 77. SQL Injection

Our application uses:

```text
Spring Data JPA
```

which can reduce some SQL injection risks when used correctly.

Still:

```text
raw SQL
dynamic queries
unvalidated input
```

must be reviewed.

Security scanning doesn't replace secure application development.

---

# 78. API Authentication

Example:

```text
Client
   │
   ▼
ALB
   │
   ▼
Ingress
   │
   ▼
auth-service
   │
   ▼
application
```

Services should validate authentication and authorization appropriately.

Do not assume:

```text
inside EKS = trusted
```

---

# 79. Service-to-Service Trust

Bad model:

```text
order-service
    ↓
"internal network, so trusted"
```

Better:

```text
order-service
    ↓
authenticated request
    ↓
payment-service
```

Combined with:

```text
NetworkPolicy
TLS where appropriate
authorization
```

---

# 80. WAF

Our public path:

```text
Internet
   ↓
CloudFront
   ↓
WAF
   ↓
ALB
   ↓
Ingress
```

WAF can help protect against common web attack patterns and abusive traffic.

But:

> WAF is not a replacement for application security.

---

# 81. Rate Limiting

Protect high-risk APIs:

```text
/login
/payment
/order
/search
```

against abuse.

Rate limiting can exist at multiple layers:

```text
CloudFront/WAF
Ingress
Application
Redis-backed limiter
```

The correct layer depends on the requirement.

---

# 82. DDoS Consideration

AWS infrastructure provides some baseline protections, but applications should still be designed to handle abusive traffic.

For public systems consider:

```text
CloudFront
WAF
rate limiting
autoscaling
caching
connection limits
application timeouts
```

---

# 83. Logging Security

Never log:

```text
password
access token
refresh token
AWS secret
credit-card data
session secret
```

Bad:

```java
log.info("DB password={}", password);
```

Better:

```text
log:
order_id
request_id
trace_id
service
status
duration
```

without sensitive values.

---

# 84. Trace Security

OpenTelemetry attributes can accidentally contain sensitive information.

Avoid putting:

```text
password
authorization header
payment data
personal secrets
```

into spans.

Use:

```text
trace.id
span.id
service.name
service.version
request metadata
```

carefully.

---

# 85. Log Retention

Don't retain sensitive logs forever.

Use:

```text
retention
access control
redaction
lifecycle
```

for Elasticsearch/S3 logging systems.

---

# 86. Audit Logging

Security incidents require evidence.

AWS:

```text
CloudTrail
```

can provide API activity auditing.

Kubernetes:

```text
Kubernetes audit logging
```

where enabled/configured, can help investigate API activity.

Application logs provide:

```text
business/application behavior
```

These three layers complement each other.

---

# 87. Security Investigation

Suppose someone changed an IAM policy.

Investigate:

```text
CloudTrail
   │
   ▼
who made change?
   │
   ▼
what changed?
   │
   ▼
when?
   │
   ▼
from where/context?
```

Then correlate with:

```text
Kubernetes
CI/CD
Git
application logs
```

---

# 88. GitOps Security

GitOps repositories are security-sensitive.

Protect:

```text
main
production manifests
ArgoCD configuration
RBAC
NetworkPolicies
secrets references
```

Require:

```text
PR
review
CODEOWNERS
status checks
```

---

# 89. ArgoCD Security

ArgoCD should have:

```text
SSO
RBAC
TLS
least privilege
repository credentials protection
project restrictions
```

Avoid allowing every ArgoCD user to:

```text
deploy every application
every namespace
every cluster
```

---

# 90. ArgoCD Project Restrictions

For example:

```text
production project
```

should only allow:

```text
approved repositories
approved clusters
approved namespaces
```

This prevents accidental deployment to unauthorized destinations.

---

# 91. Production Deployment Permission

A useful model:

```text
Developer
   ↓
PR

Reviewer
   ↓
Approve

ArgoCD
   ↓
Deploy

Production
```

For highly sensitive services:

```text
manual promotion
```

can provide an additional control.

---

# 92. Security Scanning Pipeline

Our complete CI security pipeline:

```text
Checkout
   │
   ▼
Build
   │
   ▼
Unit Tests
   │
   ▼
SonarQube
   │
   ▼
Dependency Scan
   │
   ▼
Gitleaks
   │
   ▼
Docker Build
   │
   ▼
Trivy Image Scan
   │
   ▼
SBOM
   │
   ▼
Image Signing
   │
   ▼
ECR
```

---

# 93. Runtime Security Pipeline

After deployment:

```text
EKS
 │
 ├── Pod Security
 ├── RBAC
 ├── NetworkPolicy
 ├── IAM/Pod Identity
 ├── Secrets
 ├── Security Groups
 └── Admission Policies
       │
       ▼
Monitoring
       │
       ▼
Alerting
```

---

# 94. Security Monitoring

Security events should become observable.

Examples:

```text
unexpected Pod creation
privileged Pod
failed authentication
IAM change
secret access anomaly
unusual network traffic
unexpected image
container restart storm
```

Alerting should focus on meaningful signals rather than generating thousands of low-value alerts.

---

# 95. Compromised Pod Scenario

Imagine:

```text
order-service
   ↓
vulnerability exploited
   ↓
attacker gains shell
```

Defense layers:

```text
non-root
+
read-only filesystem
+
dropped capabilities
+
NetworkPolicy
+
limited IAM
+
limited RBAC
+
resource limits
+
logging
```

The goal is to prevent:

```text
compromised Pod
    ↓
entire AWS account
```

---

# 96. Lateral Movement

Suppose attacker compromises:

```text
order-service
```

They try:

```text
→ payment-service
→ RDS
→ S3
→ AWS IAM
→ Kubernetes API
```

Our security model should make each step independently difficult.

```text
Pod
 │
 ├── NetworkPolicy → restrict
 ├── IAM → restrict
 ├── RBAC → restrict
 ├── SG → restrict
 └── secrets → restrict
```

This is defense in depth.

---

# 97. Node Security

Remember that:

```text
Pod
   ↓
Node
```

The node is another security boundary.

Use:

```text
managed node groups
hardened AMIs
regular patching
minimal host access
restricted SSH
controlled SSM access where appropriate
```

Avoid exposing SSH broadly to the Internet.

---

# 98. EKS Node Access

Prefer controlled management mechanisms such as:

```text
AWS Systems Manager
```

where appropriate rather than opening:

```text
22/tcp
0.0.0.0/0
```

to worker nodes.

---

# 99. EKS Add-On Security

Review:

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI
AWS Load Balancer Controller
External Secrets
ArgoCD
monitoring components
```

Every platform component has:

```text
image
ServiceAccount
RBAC
network access
AWS permissions
```

Review those permissions.

---

# 100. Don't Forget Third-Party Helm Charts

Using Helm does not automatically mean secure.

Before production:

```text
Review chart
Review images
Review RBAC
Review ServiceAccounts
Review NetworkPolicy
Review host mounts
Review privileges
Review exposed Services
Review versions
```

Pin approved chart/image versions.

---

# 101. Security Hardening for Helm

Our Helm values should explicitly control:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false

resources:
  requests:
    cpu: ...
    memory: ...
```

and avoid accidental defaults such as:

```text
privileged: true
hostNetwork: true
hostPID: true
```

unless required.

---

# 102. Security Hardening for Dockerfile

Our order-service Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw
RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Important properties:

```text
multi-stage build
JRE runtime image
non-root user
small runtime image
```

---

# 103. Improve the Docker Supply Chain

Production can further add:

```text
base image pinning
image scanning
SBOM
signing
provenance
approved registry
regular rebuilds
```

Don't blindly assume:

```text
base image = secure forever
```

Dependencies change over time.

---

# 104. Vulnerability Management

Suppose Trivy finds:

```text
Critical CVE
```

The process should be:

```text
Detect
  ↓
Assess
  ↓
Identify fix
  ↓
Update dependency/base image
  ↓
Test
  ↓
Scan
  ↓
Release
```

Not:

```text
Scan
 ↓
Ignore
```

---

# 105. Security Patch Management

Track:

```text
OS packages
Java dependencies
Docker base images
Helm charts
Kubernetes versions
EKS add-ons
Terraform providers
GitHub Actions
Jenkins plugins
```

Security hardening is continuous.

---

# 106. Security Checklist — AWS

```text
☐ Root account protected
☐ MFA enabled
☐ SSO/federation
☐ Least-privilege IAM
☐ No long-lived workload keys
☐ EKS Pod Identity
☐ CloudTrail
☐ KMS
☐ Private RDS
☐ Private Redis
☐ S3 Block Public Access
☐ VPC segmentation
☐ Restricted security groups
```

---

# 107. Security Checklist — EKS

```text
☐ Private worker nodes
☐ Controlled EKS API access
☐ RBAC
☐ Dedicated ServiceAccounts
☐ Pod Security
☐ NetworkPolicy
☐ Non-root containers
☐ No unnecessary privileges
☐ Resource requests/limits
☐ ResourceQuota where appropriate
☐ Admission policies
☐ Secure add-ons
```

---

# 108. Security Checklist — CI/CD

```text
☐ Branch protection
☐ CODEOWNERS
☐ Required reviews
☐ Minimal workflow permissions
☐ OIDC
☐ No permanent AWS keys
☐ Gitleaks
☐ SonarQube
☐ Dependency scanning
☐ Trivy
☐ SBOM
☐ Image signing
☐ Immutable images
☐ Protected production deployment
```

---

# 109. Security Checklist — Application

```text
☐ Authentication
☐ Authorization
☐ Input validation
☐ Secure database access
☐ Dependency updates
☐ No secrets in logs
☐ Secure headers
☐ TLS
☐ Timeouts
☐ Rate limiting
☐ Idempotency
☐ Audit logging
```

---

# 110. Security Checklist — Data

```text
☐ RDS encryption
☐ Redis encryption
☐ S3 encryption
☐ SQS/SNS encryption where required
☐ KMS controls
☐ Backup protection
☐ Backup access controls
☐ Secret rotation
☐ Private networking
☐ Database least privilege
```

---

# 111. Security Incident Flow

Suppose an application credential is compromised.

```text
Alert
  │
  ▼
Confirm
  │
  ▼
Contain
  │
  ├── revoke credential
  ├── isolate workload
  └── restrict access
  │
  ▼
Investigate
  │
  ├── CloudTrail
  ├── Kubernetes
  ├── Git
  ├── CI
  ├── logs
  └── traces
  │
  ▼
Eradicate
  │
  ▼
Recover
  │
  ▼
Monitor
  │
  ▼
Post-Incident Review
```

---

# 112. Security Is Connected to DR

Security incidents can become DR events.

Example:

```text
Credential compromise
       ↓
Attacker deletes data
       ↓
RDS corruption
       ↓
Restore
```

Therefore:

```text
Security
+
Backup
+
DR
```

must work together.

---

# 113. Security Is Connected to Observability

Without logs:

```text
security incident
   ↓
no evidence
```

With:

```text
CloudTrail
Kubernetes audit
application logs
metrics
traces
```

we can reconstruct events more effectively.

---

# 114. Security Is Connected to GitOps

Suppose someone changes:

```yaml
replicas: 3
```

to:

```yaml
privileged: true
```

A secure GitOps workflow should catch this through:

```text
PR review
policy checks
admission control
```

before it reaches production.

---

# 115. Security Is Connected to CI

Suppose a malicious dependency enters:

```text
pom.xml
```

CI should detect it through:

```text
dependency scanning
SonarQube
Trivy
SBOM
```

before deployment.

---

# 116. Security Is Connected to ECR

Suppose an image is vulnerable after it was already published.

We need:

```text
continuous vulnerability awareness
+
image retention
+
rebuild process
+
deployment controls
```

not simply:

```text
scan once
```

---

# 117. Complete Security Architecture

```text
                         DEVELOPER
                             │
                             ▼
                           GitHub
                             │
                    Branch Protection
                             │
                             ▼
                            CI
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
  SonarQube              Gitleaks               Trivy
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             ▼
                           SBOM
                             │
                             ▼
                         Image Sign
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
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
      IAM                   RBAC             NetworkPolicy
       │                     │                     │
       ▼                     ▼                     ▼
 Pod Identity         ServiceAccount       Workload isolation
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             ▼
                         Applications
                             │
               ┌─────────────┼─────────────┐
               ▼             ▼             ▼
              RDS          Redis          SQS
               │             │             │
               └─────────────┼─────────────┘
                             ▼
                            KMS
```

---

# 118. The Security Mental Model

Think in terms of **blast radius**.

Without controls:

```text
Compromised Pod
      ↓
Node
      ↓
Cluster
      ↓
AWS
      ↓
Everything
```

With layered controls:

```text
Compromised Pod
      │
      ├── Non-root
      ├── No privilege escalation
      ├── NetworkPolicy
      ├── Restricted IAM
      ├── Restricted RBAC
      ├── Limited secrets
      ├── Security Groups
      └── Admission policies
             │
             ▼
       Smaller blast radius
```

That is the real objective.

---

# 119. Our Production Security Chain

The final security model for the project is:

```text
SOURCE
  ↓
Branch Protection
  ↓
PR Review
  ↓
CI
  ↓
SonarQube
  ↓
Dependency Scan
  ↓
Gitleaks
  ↓
Trivy
  ↓
SBOM
  ↓
Image Signing
  ↓
ECR
  ↓
GitOps
  ↓
ArgoCD
  ↓
Admission Policy
  ↓
EKS
  ↓
Non-Root Pod
  ↓
NetworkPolicy
  ↓
Pod Identity
  ↓
Least-Privilege IAM
  ↓
Encrypted Data
  ↓
Monitoring + Audit
```

---

# 120. Final Production Checklist

Before declaring our platform production-ready:

```text
SOURCE
☐ Protected main
☐ CODEOWNERS
☐ Required reviews
☐ Secret scanning

CI/CD
☐ OIDC
☐ Least privilege
☐ SonarQube
☐ Dependency scanning
☐ Trivy
☐ SBOM
☐ Image signing
☐ Immutable images

AWS
☐ IAM least privilege
☐ KMS
☐ CloudTrail
☐ Private networking
☐ Restricted SGs

EKS
☐ RBAC
☐ Pod Identity
☐ NetworkPolicy
☐ Non-root
☐ No unnecessary privileges
☐ Pod Security
☐ Admission policies

DATA
☐ RDS encryption
☐ Redis security
☐ S3 security
☐ Messaging security
☐ Backup protection

RUNTIME
☐ Secrets Manager
☐ External Secrets
☐ TLS
☐ Rate limiting
☐ Resource limits
☐ Security monitoring

OBSERVABILITY
☐ Logs
☐ Metrics
☐ Traces
☐ Audit logs
☐ Security alerts

RECOVERY
☐ Backups
☐ Restore tests
☐ DR runbook
☐ Credential rotation procedure
☐ Incident response procedure
```

---

# 121. Final Architecture

Our complete platform now has security built into every layer:

```text
                         INTERNET
                            │
                            ▼
                       CloudFront
                            │
                            ▼
                           WAF
                            │
                            ▼
                           ALB
                            │
                            ▼
                         Ingress
                            │
                            ▼
                          EKS
                            │
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
  NetworkPolicy          RBAC               Pod Identity
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                      21 Microservices
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
            RDS           Redis           SQS
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                           KMS

Security Pipeline:
Git → CI → SonarQube → Gitleaks → Trivy → SBOM → Sign → ECR

Observability:
Prometheus → Grafana → Alertmanager
Fluent Bit → Logstash → Elasticsearch → Kibana
OpenTelemetry → Jaeger
```

> **The goal of security hardening is not to make the system impossible to attack. The goal is to make unauthorized access harder, reduce blast radius, protect sensitive data, detect abnormal behavior, and recover quickly when something goes wrong.**

We have now covered the major **deployment + resilience + security preparation** layers.

**Next: `48-Cost-Optimization.md`** — we'll analyze where the money actually goes in this 21-microservice EKS platform: **EKS nodes, Karpenter, NAT Gateways, ALB, RDS, ElastiCache, EBS, ECR, CloudWatch, Elasticsearch, Logstash, data transfer, observability, Spot instances, right-sizing, dev/QA cost controls, and production cost-optimization without destroying reliability.**
