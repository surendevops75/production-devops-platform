# 26-Secrets-Management.md

# Secrets Management — AWS Secrets Manager, KMS & EKS

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Primary secret store:** AWS Secrets Manager
> **Encryption:** AWS KMS
> **Kubernetes integration:** External Secrets Operator / EKS Pod Identity
> **Application:** Spring Boot microservices
> **GitOps:** ArgoCD

---

# 1. Why Secrets Management Matters

Our platform contains sensitive values such as:

```text
Database passwords
Redis credentials
RabbitMQ credentials
Kafka credentials
API keys
Payment-provider credentials
TLS private keys
OAuth client secrets
Third-party service tokens
```

A production system must answer:

> Where is the secret stored, who can access it, how does it reach the application, and how do we rotate it?

---

# 2. The Worst Possible Design

Never do this:

```yaml
spring:
  datasource:
    username: postgres
    password: MyPassword123
```

and commit it to Git.

That creates:

```text
Git
 ↓
GitHub
 ↓
Anyone with repository access
 ↓
Secret exposed
```

Deleting the commit later does not necessarily eliminate the exposure because Git history can retain it.

---

# 3. Never Put Secrets in These Places

Avoid secrets in:

```text
Git repositories
Dockerfiles
Docker images
application source code
Helm values
Terraform variables committed to Git
CI logs
PR comments
Kubernetes manifests
ConfigMaps
shell history
README files
```

Also be careful with:

```text
Terraform state
```

because sensitive values can still exist in state depending on how resources and providers are configured.

---

# 4. Production Secret Architecture

Our preferred architecture:

```text
                 AWS
                  │
                  ▼
          Secrets Manager
                  │
                  ▼
                KMS
                  │
                  │ encryption
                  ▼
          Secret encrypted
                  │
                  ▼
             EKS Pod
                  │
                  ▼
           Application
```

The application should not need AWS administrator credentials.

---

# 5. Complete Secret Flow

For `order-service`:

```text
AWS Secrets Manager
        │
        │ secret
        ▼
order-service/prod/database
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
order-service Pod
        │
        ▼
Spring Boot
```

The AWS access itself is controlled through:

```text
EKS Pod Identity
        │
        ▼
IAM Role
        │
        ▼
Secrets Manager
```

---

# 6. Two Different Secret Delivery Models

There are two major approaches.

### Model A — External Secrets Operator

```text
Secrets Manager
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

### Model B — Direct runtime retrieval

The application itself retrieves the secret:

```text
Pod
 │
 ▼
AWS SDK
 │
 ▼
Secrets Manager
```

Both are valid architectures.

For our project, we'll primarily use the External Secrets pattern because it integrates cleanly with Kubernetes/GitOps.

---

# 7. What Is AWS Secrets Manager?

AWS Secrets Manager is AWS's managed service for storing and retrieving secrets.

Example:

```text
order-service/prod/database
```

could contain:

```json
{
  "username": "order_app",
  "password": "********",
  "host": "orders-prod.xxxxx.ap-south-1.rds.amazonaws.com",
  "port": 5432,
  "database": "orders"
}
```

The actual password is never committed to Git.

---

# 8. What Is KMS?

AWS KMS provides encryption-key management.

Conceptually:

```text
Secret
  │
  ▼
KMS
  │
  ▼
Encrypted secret
```

Think of:

```text
Secrets Manager
```

as:

> The secure storage/service for the secret.

And:

```text
KMS
```

as:

> The cryptographic key-management layer protecting data.

Secrets Manager can use AWS managed encryption or a customer managed KMS key depending on requirements.

---

# 9. Customer Managed KMS Key

For stronger organizational control:

```text
KMS
 │
 └── ecommerce-secrets-prod
```

Possible controls include:

```text
Key policy
IAM permissions
Key rotation
Audit logging
Cross-account permissions
```

Example:

```text
Secrets Manager
       │
       ▼
KMS Key
       │
       ▼
Encrypted secret
```

---

# 10. Secret Naming Convention

Use predictable names.

For example:

```text
/ecommerce/prod/order-service/database
/ecommerce/prod/payment-service/database
/ecommerce/prod/redis
/ecommerce/prod/rabbitmq
/ecommerce/prod/payment-provider
```

Or a flatter convention:

```text
ecommerce/prod/order-service/database
ecommerce/prod/payment-service/database
```

The important part is consistency.

---

# 11. Environment Separation

Don't use one secret for everything.

Bad:

```text
ecommerce/database
```

used by:

```text
dev
qa
prod
```

Better:

```text
ecommerce/dev/order-service/database
ecommerce/qa/order-service/database
ecommerce/prod/order-service/database
```

Production credentials should never be casually available to development workloads.

---

# 12. Service Separation

Don't give:

```text
order-service
```

the password for:

```text
payment-service
```

Use:

```text
order-service
   │
   └── order DB secret

payment-service
   │
   └── payment DB secret
```

This follows least privilege.

---

# 13. IAM Mental Model

Always ask:

```text
WHO
 ↓
CAN DO WHAT
 ↓
ON WHICH RESOURCE
```

Example:

```text
order-service Pod
      │
      ▼
IAM Role
      │
      ▼
secretsmanager:GetSecretValue
      │
      ▼
ecommerce/prod/order-service/database
```

Not:

```text
order-service
   ↓
secretsmanager:*
   ↓
*
```

---

# 14. EKS Pod Identity

Our preferred AWS access flow:

```text
Order Pod
    │
    ▼
ServiceAccount
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

No:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside the container.

This is an important production security principle.

---

# 15. Why Static AWS Keys Are Dangerous

Bad:

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    value: AKIA...
  - name: AWS_SECRET_ACCESS_KEY
    value: ...
```

Problems:

```text
credential leakage
rotation difficulty
Git exposure
CI exposure
developer exposure
long-lived credentials
```

Use workload identity instead.

---

# 16. External Secrets Operator

External Secrets Operator allows Kubernetes resources to reference external secret stores.

Conceptually:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
```

The Kubernetes Secret is generated/synchronized from the external secret.

This lets Git contain:

```text
which secret to retrieve
```

without containing:

```text
the actual password
```

---

# 17. Installation Placement

External Secrets Operator belongs in the platform layer.

Example:

```text
EKS
 │
 ├── argocd
 ├── ingress-system
 ├── monitoring
 ├── logging
 ├── tracing
 └── external-secrets
```

It should not be installed separately inside every application namespace.

---

# 18. Operator Architecture

Conceptually:

```text
             AWS Secrets Manager
                     │
                     ▼
          External Secrets Operator
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     production              qa
       Secret                 Secret
          │                     │
          ▼                     ▼
     order-service         order-service
```

The operator watches ExternalSecret resources and synchronizes them.

---

# 19. SecretStore

A `SecretStore` defines how Kubernetes connects to an external secret provider within a namespace.

Conceptually:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
```

The exact API version/schema should match the External Secrets Operator version deployed in your cluster.

---

# 20. ClusterSecretStore

For centralized platform configurations, a `ClusterSecretStore` can be used.

Conceptually:

```text
ClusterSecretStore
        │
        ├── production
        ├── qa
        └── dev
```

Use this carefully.

A cluster-wide secret-store configuration can increase the blast radius if permissions are too broad.

---

# 21. ExternalSecret

An `ExternalSecret` describes which external secret should populate a Kubernetes Secret.

Example:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: order-service-db
  namespace: production
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore

  target:
    name: order-service-db

  data:
    - secretKey: username
      remoteRef:
        key: ecommerce/prod/order-service/database
        property: username

    - secretKey: password
      remoteRef:
        key: ecommerce/prod/order-service/database
        property: password
```

Again, exact fields should be checked against the installed operator version.

---

# 22. What Goes Into Git?

This:

```yaml
remoteRef:
  key: ecommerce/prod/order-service/database
```

can be stored in Git.

This should **not**:

```yaml
password: SuperSecret123
```

The GitOps repository stores the desired reference, not the secret value.

---

# 23. ArgoCD Interaction

Our architecture becomes:

```text
GitHub
  │
  ▼
GitOps Repository
  │
  ▼
ArgoCD
  │
  ▼
ExternalSecret
  │
  ▼
External Secrets Operator
  │
  ▼
AWS Secrets Manager
  │
  ▼
Kubernetes Secret
```

This is an important distinction:

> ArgoCD manages the desired Kubernetes configuration; Secrets Manager remains the source of truth for secret values.

---

# 24. Secret Rotation

Suppose:

```text
database password
```

needs rotation.

Without centralized management:

```text
change password
 ↓
update Git
 ↓
build image
 ↓
redeploy
```

This is bad.

With Secrets Manager:

```text
Secrets Manager
       │
       ▼
Rotate secret
       │
       ▼
ExternalSecret refresh
       │
       ▼
Kubernetes Secret updated
```

Application behavior determines whether a restart is required.

---

# 25. Rotation Does Not Automatically Mean Application Recovery

This is important.

Suppose:

```text
DB password
     │
     ▼
Kubernetes Secret
     │
     ▼
Environment variable
```

The process may already have loaded:

```text
OLD_PASSWORD
```

into its environment.

Updating the Kubernetes Secret doesn't magically update the process environment.

Therefore you need a strategy such as:

```text
Secret rotation
   │
   ▼
restart/reload application
```

or use a runtime secret retrieval/reload mechanism.

---

# 26. Environment Variable vs Mounted Secret

Kubernetes secrets can reach an application in multiple ways.

### Environment variable

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: password
```

Spring Boot:

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}
```

Simple and common.

---

# 27. Secret Volume

Another option is mounting the secret as a file:

```text
Pod
 │
 ▼
/mnt/secrets/
 ├── username
 └── password
```

The application reads the files.

This can be useful when applications support dynamic file-based credential reload.

---

# 28. Important Difference

Environment variable:

```text
Secret
 ↓
Process environment
```

File mount:

```text
Secret
 ↓
Filesystem
```

The correct method depends on application behavior and rotation requirements.

---

# 29. Spring Boot Integration

Our production application configuration should look like:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Kubernetes provides:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

The values come from the Kubernetes Secret.

---

# 30. Example Deployment

Conceptually:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3

  template:
    spec:
      serviceAccountName: order-service

      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0

          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: order-service-db
                  key: username

            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-db
                  key: password
```

---

# 31. Database URL

The URL isn't always secret.

For example:

```text
jdbc:postgresql://orders-prod.xxxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

may be treated as configuration.

But credentials are sensitive.

Therefore:

```text
DB_URL        → ConfigMap/env configuration
DB_USERNAME   → Secret
DB_PASSWORD   → Secret
```

The exact classification depends on your organization's policy.

---

# 32. Redis Secret

Our Redis architecture:

```text
order-service
      │
      ▼
ElastiCache
```

Configuration:

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

`REDIS_PASSWORD` should come from the secret-management system.

---

# 33. RabbitMQ Secret

If we use RabbitMQ:

```text
RABBITMQ_USERNAME
RABBITMQ_PASSWORD
```

should not be committed to Git.

Architecture:

```text
Secrets Manager
       │
       ▼
ExternalSecret
       │
       ▼
Kubernetes Secret
       │
       ▼
RabbitMQ client
```

---

# 34. Kafka Credentials

For Kafka, depending on the deployment/security model:

```text
username
password
TLS certificate
private key
CA certificate
```

may require secret management.

For AWS MSK, the authentication model may differ depending on the chosen authentication mechanism.

Again:

> Don't blindly create username/password secrets when the platform supports identity-based authentication.

---

# 35. Payment Provider Secrets

Payment integration is highly sensitive.

Example:

```text
PAYMENT_API_KEY
PAYMENT_WEBHOOK_SECRET
PAYMENT_CLIENT_SECRET
```

Architecture:

```text
Secrets Manager
       │
       ▼
payment-service
```

Only:

```text
payment-service
```

should normally have access.

---

# 36. Secret Least Privilege

Suppose:

```text
payment-service
```

has:

```text
GetSecretValue
```

on:

```text
ecommerce/prod/payment-service/*
```

That might still be broader than necessary.

Prefer the exact secret:

```text
ecommerce/prod/payment-service/provider
```

where practical.

---

# 37. IAM Policy Example

Conceptually:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:ecommerce/prod/order-service/database-*"
}
```

If a customer-managed KMS key is used, the workload may also require the appropriate KMS permission according to the encryption configuration and AWS service behavior.

---

# 38. Don't Give Applications KMS Admin

Bad:

```text
order-service
  ↓
kms:*
```

The application usually doesn't need administrative key management operations.

It needs the minimum permissions required to retrieve/use the protected secret through the chosen architecture.

---

# 39. KMS Key Policy

KMS has its own authorization model.

Think:

```text
IAM
 │
 ├── who can call KMS
 │
 ▼
KMS Key Policy
 │
 └── who is trusted to use/manage key
```

Key management permissions should be tightly controlled.

---

# 40. Secret Access Logging

AWS CloudTrail can provide an audit trail for AWS API activity.

For secret management, you want to know:

```text
Who
When
Which secret
What API action
From which identity
```

This is important for:

```text
security investigations
compliance
incident response
auditing
```

---

# 41. Secret Access Monitoring

Watch for abnormal behavior:

```text
order-service
```

suddenly reading:

```text
50 different secrets
```

instead of:

```text
1 expected secret
```

This may indicate:

```text
misconfiguration
excessive permissions
compromised workload
```

---

# 42. Kubernetes Secret Is Not Your Ultimate Source of Truth

A common misunderstanding:

```text
Kubernetes Secret
```

is itself the permanent secret-management solution.

For our architecture:

```text
AWS Secrets Manager
        │
        ▼
source of truth
        │
        ▼
Kubernetes Secret
        │
        ▼
application
```

Kubernetes Secret is a delivery/runtime object.

---

# 43. Kubernetes Secret Encryption

Kubernetes Secret objects should be protected appropriately.

On EKS, encryption-at-rest configuration and AWS-managed integrations should be considered as part of the cluster security design.

Do not assume:

```text
Secret object
```

automatically means:

```text
encrypted everywhere
```

Review:

```text
etcd encryption
KMS integration
RBAC
API access
```

according to the EKS configuration.

---

# 44. Secret RBAC

Don't allow every developer/service account to read all secrets.

Bad:

```text
RoleBinding
  ↓
get secrets
  ↓
*
```

Better:

```text
order-service ServiceAccount
        │
        ▼
order-service Secret
```

Namespace-level isolation and RBAC should reinforce IAM boundaries.

---

# 45. Defense in Depth

Our security model:

```text
AWS IAM
   +
KMS
   +
Secrets Manager
   +
EKS Pod Identity
   +
Kubernetes RBAC
   +
NetworkPolicy
   +
Application authorization
```

No single control should be the only protection.

---

# 46. Secret in CI/CD

CI may need secrets for:

```text
SonarQube
Docker registry
AWS authentication
third-party APIs
deployment tooling
```

But avoid:

```text
echo "$SECRET"
```

or:

```text
printenv
```

in CI logs.

Use the CI platform's secret mechanism and mask sensitive values.

---

# 47. GitHub Actions

Example:

```yaml
env:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Don't:

```yaml
run: echo "${{ secrets.SONAR_TOKEN }}"
```

Also remember:

> GitHub Actions secrets are for CI/CD credentials, not necessarily for application runtime secrets.

Application runtime secrets should generally remain in the runtime secret-management system.

---

# 48. GitLab CI

GitLab protected/masked variables can hold CI credentials.

Example concept:

```text
SONAR_TOKEN
AWS_ROLE
```

But again:

```text
CI secret
≠
application runtime secret
```

Keep their purposes separate.

---

# 49. Jenkins

Jenkins Credentials can store:

```text
tokens
passwords
SSH credentials
certificates
```

Use:

```text
withCredentials
```

or appropriate Jenkins credential bindings.

Never hard-code:

```groovy
password = "..."
```

in a Jenkinsfile.

---

# 50. Terraform Secrets

Terraform introduces an important problem.

Suppose:

```hcl
resource "aws_secretsmanager_secret_version" "db" {
  secret_string = var.db_password
}
```

The secret may end up in:

```text
Terraform state
```

even if:

```hcl
sensitive = true
```

is set.

Important:

> `sensitive = true` hides output; it does not magically remove the value from Terraform state.

---

# 51. Terraform State Protection

Our state architecture:

```text
Terraform
    │
    ▼
S3 state
    │
    ├── encryption
    ├── versioning
    └── locking
```

Access should be limited to the Terraform execution identities.

State itself is sensitive infrastructure data.

---

# 52. Better Secret Creation Strategy

Where possible:

```text
Terraform
   │
   └── creates secret metadata/policy
```

while:

```text
controlled secret bootstrap/rotation process
   │
   ▼
actual secret value
```

is handled separately.

For database credentials, AWS-managed password features can sometimes avoid handling the plaintext password directly in Terraform configuration, depending on the AWS resource/service and provider capabilities.

---

# 53. Don't Put Secrets in Helm Values

Bad:

```yaml
database:
  username: order
  password: MyPassword
```

Better:

```yaml
database:
  secretName: order-service-db
```

The Helm chart should reference the Secret, not contain the value.

---

# 54. Don't Put Secrets in ConfigMaps

ConfigMaps are intended for non-sensitive configuration.

Bad:

```yaml
apiVersion: v1
kind: ConfigMap
data:
  DB_PASSWORD: "..."
```

Use:

```text
Secret
```

for sensitive values.

---

# 55. GitOps Repository

Our GitOps repository:

```text
gitops/
├── applications/
├── environments/
│   ├── dev/
│   ├── qa/
│   └── prod/
└── argocd/
```

can contain:

```text
ExternalSecret manifests
SecretStore configuration
ServiceAccount
Deployment references
```

but not:

```text
actual production passwords
```

---

# 56. Secret GitOps Flow

```text
Developer
   │
   ▼
GitOps PR
   │
   ▼
Review
   │
   ▼
ArgoCD
   │
   ▼
ExternalSecret
   │
   ▼
AWS Secrets Manager
   │
   ▼
Kubernetes Secret
```

This is much safer than storing encrypted/plaintext credentials directly in Git.

---

# 57. What About Sealed Secrets?

Another approach is:

```text
SealedSecret
```

where encrypted secret manifests are stored in Git and decrypted inside Kubernetes.

Architecture:

```text
Git
 │
 ▼
Encrypted Secret
 │
 ▼
Kubernetes controller
 │
 ▼
Secret
```

This can be useful.

But for our AWS project, we prefer:

```text
AWS Secrets Manager
+
External Secrets Operator
```

because it keeps the authoritative secret value in AWS.

---

# 58. Secrets Store CSI Driver

Another Kubernetes pattern is:

```text
Secrets Store CSI Driver
```

which can mount secrets from external providers into pods.

Conceptually:

```text
Secrets Manager
       │
       ▼
CSI Driver
       │
       ▼
Pod filesystem
```

This is another valid architecture.

The choice between:

```text
External Secrets Operator
Secrets Store CSI Driver
application SDK
```

depends on application behavior, rotation requirements and platform standards.

---

# 59. Our Preferred Project Pattern

For this project:

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
Spring Boot Pod
```

AWS access:

```text
EKS Pod Identity
        │
        ▼
IAM Role
```

Encryption:

```text
Secrets Manager
        │
        ▼
KMS
```

Audit:

```text
CloudTrail
```

---

# 60. Secret Rotation Architecture

For a database:

```text
                   Secrets Manager
                         │
                    rotation
                         │
                         ▼
                     New secret
                         │
                         ▼
                    RDS credential
                         │
                         ▼
                ExternalSecret refresh
                         │
                         ▼
                   Application
```

But successful rotation requires coordination between:

```text
credential source
database
secret store
application
connection pool
```

---

# 61. HikariCP and Password Rotation

Our Spring Boot application uses HikariCP.

Existing connections may already have authentication established.

Changing:

```text
DB_PASSWORD
```

does not necessarily invalidate/recreate all existing database connections immediately.

Therefore rotation testing must verify:

```text
new connections use new credential
existing connections behave correctly
failed connections recover
application reload/restart strategy works
```

---

# 62. Rotation Frequency

Don't say:

```text
rotate every 1 day
```

without a reason.

Choose based on:

```text
security policy
compliance
credential type
operational risk
provider capabilities
application support
```

Examples:

```text
database password
API credential
TLS certificate
service account secret
```

may all have different rotation requirements.

---

# 63. Secret Expiration

Some secrets should have an explicit lifecycle.

Example:

```text
Third-party API token
```

could have:

```text
creation date
expiration date
owner
rotation procedure
```

Track:

```text
secret
 │
 ├── owner
 ├── purpose
 ├── environment
 ├── rotation method
 └── expiry
```

---

# 64. Secret Inventory

Enterprise environments should maintain an inventory.

Example:

| Secret           | Owner        | Environment | Purpose       |
| ---------------- | ------------ | ----------- | ------------- |
| order DB         | Backend Team | Prod        | PostgreSQL    |
| payment provider | Payment Team | Prod        | Payment API   |
| Redis auth       | Platform     | Prod        | Cache         |
| Sonar token      | DevOps       | CI          | Code analysis |

This makes incident response and rotation much easier.

---

# 65. Secret Ownership

Every production secret should have:

```text
Owner
Purpose
Application
Environment
Rotation method
Recovery method
```

Avoid:

```text
"Nobody knows who owns this secret."
```

That becomes a major operational problem.

---

# 66. Secret Leakage in Logs

Bad:

```java
log.info("Connecting to DB with password {}", password);
```

Never log:

```text
password
token
API key
authorization header
private key
session secret
```

Even debug logging can expose sensitive data.

---

# 67. Secret Leakage in Exceptions

Be careful with:

```text
HTTP headers
connection strings
request objects
configuration dumps
```

An exception such as:

```text
jdbc:postgresql://...?...password=...
```

could expose credentials in centralized logging.

Our ELK pipeline should therefore be treated as a sensitive data path too.

---

# 68. Secret Leakage in Metrics

Don't create:

```text
http_request_total{api_key="..."}
```

or:

```text
payment_failure{token="..."}
```

Metrics labels become highly visible and can have long retention.

Never use secrets as labels.

---

# 69. Secret Leakage in Traces

OpenTelemetry spans can accidentally capture:

```text
Authorization headers
request payloads
database URLs
tokens
```

Tracing instrumentation must be configured to redact sensitive attributes.

This will matter when we reach:

```text
39-OpenTelemetry.md
```

---

# 70. Secret Leakage in Kubernetes

Be careful with:

```text
kubectl describe pod
kubectl get secret
kubectl logs
debug containers
```

A user with sufficient Kubernetes permissions may be able to retrieve secrets.

Therefore:

```text
Kubernetes RBAC
+
IAM
```

must be tightly controlled.

---

# 71. Secret Leakage in Docker

Never:

```dockerfile
ENV DB_PASSWORD=...
```

Never:

```dockerfile
COPY .env .
```

Never bake credentials into an image.

An image may be stored in:

```text
ECR
developer laptop
CI cache
registry mirror
backup
```

Once leaked, it can be extremely difficult to contain.

---

# 72. Secret Scanning

Our CI security pipeline should scan source code for accidental secrets.

We already use:

```text
Gitleaks
```

Example:

```text
Developer
   │
   ▼
Git commit
   │
   ▼
Gitleaks
   │
   ├── clean → continue
   │
   └── secret found → fail
```

This is a safety net, not a replacement for proper secret management.

---

# 73. Secret Rotation Incident

Suppose:

```text
Payment API key
```

is accidentally committed.

Correct response:

```text
1. Revoke/rotate credential
2. Identify affected systems
3. Remove exposure from active branches/history as appropriate
4. Audit usage
5. Investigate access
6. Deploy new credential
7. Document incident
```

Don't just delete the Git line and assume the problem is solved.

---

# 74. Production Troubleshooting

### Application says:

```text
AccessDeniedException
```

Check:

```text
Pod Identity
IAM role
IAM policy
secret ARN
region
KMS permissions
resource policy
```

---

# 75. Secret Not Found

Check:

```text
secret name
AWS region
AWS account
environment
ExternalSecret
SecretStore
operator status
IAM role
```

A very common mistake is:

```text
Secret exists in us-east-1
```

while the workload is configured for:

```text
ap-south-1
```

---

# 76. ExternalSecret Not Syncing

Check:

```bash
kubectl get externalsecret -n production
```

Then:

```bash
kubectl describe externalsecret order-service-db -n production
```

Check operator logs:

```bash
kubectl logs \
  -n external-secrets \
  deployment/external-secrets
```

The exact deployment name depends on how the operator was installed.

---

# 77. Kubernetes Secret Missing

Check:

```bash
kubectl get secret -n production
```

Then:

```bash
kubectl describe externalsecret order-service-db -n production
```

Look for:

```text
provider authentication
permission errors
secret name
property name
refresh errors
```

---

# 78. Pod Has Empty Environment Variable

Check:

```text
ExternalSecret
       │
       ▼
Kubernetes Secret
       │
       ▼
Deployment env reference
```

The failure can occur at any layer.

Don't immediately blame Spring Boot.

---

# 79. IAM Troubleshooting Chain

When AWS secret access fails:

```text
Pod
 │
 ▼
ServiceAccount
 │
 ▼
EKS Pod Identity
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
 │
 ▼
Secrets Manager
 │
 ▼
KMS
```

Check each layer.

---

# 80. Cross-Account Secrets

Enterprise AWS environments may use:

```text
Security Account
Production Account
```

or centralized security services.

Cross-account secret access requires careful configuration of:

```text
IAM
resource policies
KMS key policy
network
trust relationships
```

Don't automatically centralize every application secret across accounts.

Often environment-specific ownership is simpler and safer.

---

# 81. Multi-Region Secrets

For disaster recovery, some secrets may need replication.

Conceptually:

```text
Primary Region
ap-south-1
    │
    ▼
Secrets Manager
    │
    └────────► DR Region
                ap-southeast-1
```

But not every secret should be replicated automatically.

Consider:

```text
data residency
compliance
credential validity
application dependencies
DR architecture
```

---

# 82. Disaster Recovery

Our DR architecture should ensure that applications can obtain required secrets in the DR region.

Don't build:

```text
DR infrastructure
```

and then discover:

```text
DR application cannot authenticate to database/API
```

Secret recovery is part of disaster recovery.

---

# 83. Backup vs Secret Recovery

A secret is not usually something you "restore" like an RDS snapshot.

Instead, you need:

```text
secret definition
+
secret value/rotation process
+
permissions
+
KMS key availability
```

DR runbooks must account for all of these.

---

# 84. Secret Management Across Environments

Our structure:

```text
DEV
 │
 └── dev secrets

QA
 │
 └── qa secrets

PROD
 │
 └── prod secrets
```

Each workload receives only its environment's credentials.

---

# 85. Production Architecture

The complete architecture:

```text
                         AWS
                          │
                    Secrets Manager
                          │
                         KMS
                          │
                          ▼
                    Secret value
                          │
                          ▼
                 External Secrets
                    Operator
                          │
                          ▼
                     Kubernetes
                       Secret
                          │
                          ▼
                      EKS Pod
                          │
                          ▼
                    Spring Boot
```

Identity:

```text
EKS Pod
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

# 86. Complete Security Chain

Our production security path is:

```text
Git
 │
 ├── No plaintext secrets
 │
 ▼
CI
 │
 ├── Gitleaks
 │
 ├── secret masking
 │
 └── least privilege
 │
 ▼
ECR
 │
 └── no baked credentials
 │
 ▼
ArgoCD
 │
 └── ExternalSecret manifests
 │
 ▼
EKS
 │
 ├── RBAC
 │
 ├── Pod Identity
 │
 └── NetworkPolicy
 │
 ▼
Secrets Manager
 │
 └── KMS
```

---

# 87. Production Tool Placement

| Tool                      | Where it belongs | Purpose                   |
| ------------------------- | ---------------- | ------------------------- |
| AWS Secrets Manager       | AWS              | Secret source of truth    |
| KMS                       | AWS              | Encryption/key management |
| External Secrets Operator | EKS              | Sync external secrets     |
| Kubernetes Secret         | EKS              | Runtime secret object     |
| EKS Pod Identity          | EKS/AWS          | Workload identity         |
| ArgoCD                    | EKS              | GitOps deployment         |
| Gitleaks                  | CI               | Detect leaked secrets     |
| Secrets Manager rotation  | AWS              | Credential lifecycle      |
| CloudTrail                | AWS              | Audit                     |
| ELK                       | EKS/platform     | Log monitoring            |
| OpenTelemetry             | EKS/application  | Trace observability       |

---

# 88. What DevOps Owns

The DevOps/platform team typically owns:

```text
Secrets Manager architecture
KMS architecture
IAM roles
Pod Identity
External Secrets Operator
RBAC
GitOps integration
rotation automation
audit
monitoring
incident procedures
```

Application teams own:

```text
which credentials the application requires
safe usage
connection handling
secret redaction
application-level rotation support
```

Security teams define organizational requirements and review controls.

---

# 89. Production Checklist

### Storage

* [ ] Secrets stored in Secrets Manager
* [ ] No plaintext credentials in Git
* [ ] Appropriate KMS encryption
* [ ] Naming convention
* [ ] Environment separation

### IAM

* [ ] EKS Pod Identity
* [ ] Least privilege
* [ ] Exact secret access
* [ ] KMS permissions
* [ ] No long-lived AWS keys

### Kubernetes

* [ ] External Secrets Operator
* [ ] RBAC
* [ ] Secret references
* [ ] No secrets in ConfigMaps
* [ ] No secret values in manifests

### CI/CD

* [ ] Gitleaks
* [ ] Masked CI secrets
* [ ] No secret logging
* [ ] No secrets baked into images

### Application

* [ ] Environment/file-based injection
* [ ] No secret logging
* [ ] Redacted traces
* [ ] Redacted metrics
* [ ] Rotation support

### Operations

* [ ] Rotation procedure
* [ ] Ownership
* [ ] Audit
* [ ] Alerts
* [ ] Incident response
* [ ] DR procedure
* [ ] Periodic access review

---

# 90. Final Mental Model

The most important thing to remember:

```text
             SECRET MANAGEMENT

       Where is the secret?
               │
               ▼
       AWS Secrets Manager
               │
               ▼
            KMS
               │
               ▼
       Who can access it?
               │
               ▼
         IAM / Pod Identity
               │
               ▼
        How does it enter?
               │
               ▼
     External Secrets Operator
               │
               ▼
      Kubernetes Secret
               │
               ▼
            EKS Pod
               │
               ▼
         Spring Boot
```

And the security rule:

> **Git stores references/configuration. Secrets Manager stores secret values. IAM controls access. KMS protects encryption. EKS Pod Identity provides workload identity. Kubernetes delivers the secret to the workload.**

---

# 91. Our Project Now

The infrastructure picture is becoming:

```text
                         AWS
                          │
             ┌────────────┼────────────┐
             │            │            │
            VPC          ECR       Secrets Manager
             │                         │
             ▼                         ▼
            EKS                       KMS
             │
    ┌────────┼──────────┐
    │        │          │
   Apps    ArgoCD   Platform
    │
    ├── RDS
    ├── Redis
    ├── SQS/SNS
    ├── RabbitMQ*
    └── Kafka*
```

`*` = alternative architectures we learned; they don't all need to be deployed simultaneously.

---

## Completed sequence

```text
20-RDS.md                    ✅
21-EKS-Database-Management.md ✅
22-Redis.md                  ✅
23-SQS-SNS.md                ✅
24-RabbitMQ.md               ✅
25-Kafka.md                  ✅
26-Secrets-Management.md     ✅
```

### Next: `27-GitOps-Repository.md`

We'll now move from **infrastructure/services** into the actual **GitOps operating model**:

```text
Developer
    │
    ▼
Application Repo
    │
    ▼
CI
    │
    ▼
ECR
    │
    ▼
GitOps Repo
    │
    ▼
ArgoCD
    │
    ▼
EKS
```

and go deeply into **repo structure, environment separation, Helm values, image promotion, PR-based deployment, ArgoCD Application/ApplicationSet design, production approvals, rollback, drift detection, multi-cluster/multi-account GitOps, and how Terraform + GitOps coexist without fighting each other.**
