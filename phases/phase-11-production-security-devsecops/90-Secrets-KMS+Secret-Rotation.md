# 90 — Secrets, KMS and Secret Rotation

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS
> **Secrets Source of Truth:** AWS Secrets Manager
> **Encryption:** AWS KMS
> **Kubernetes Integration:** External Secrets Operator
> **Workload Identity:** EKS Pod Identity
> **GitOps:** ArgoCD
> **Previous:** 89 — Kubernetes RBAC + AWS IAM Deep Dive
> **Next:** 91 — AWS Security Monitoring and Detection

---

# 1. Objective

Secrets are one of the most sensitive parts of our production platform.

Our applications need credentials such as:

```text
PostgreSQL username/password
Redis credentials
Payment-provider API keys
SMTP credentials
Third-party API tokens
TLS certificates
Internal service credentials
```

We do **not** want these values inside:

```text
GitHub repositories
Docker images
Helm values
ConfigMaps
Terraform source files
GitOps repositories
Application logs
```

Our production architecture is:

```text
                    AWS Secrets Manager
                            |
                         KMS
                            |
                            v
                  External Secrets Operator
                            |
                            v
                     Kubernetes Secret
                            |
                            v
                     Application Pod
                            |
                       Environment
                         / File
```

The application should never need to know how the secret is stored in AWS.

---

# 2. The Secret Lifecycle

A production secret has a lifecycle:

```text
Create
  ↓
Encrypt
  ↓
Store
  ↓
Access
  ↓
Use
  ↓
Rotate
  ↓
Distribute
  ↓
Audit
  ↓
Revoke
  ↓
Delete
```

The important point is:

> **Secret management is a lifecycle, not simply "put the password in Secrets Manager."**

---

# 3. What Counts as a Secret?

For our project:

```text
Database passwords
API keys
OAuth client secrets
Payment credentials
SMTP passwords
Private keys
TLS private keys
Signing credentials
Webhook secrets
Third-party service tokens
```

Not everything is a secret.

For example:

```text
APPLICATION_NAME=order-service
SERVER_PORT=8080
LOG_LEVEL=INFO
```

are configuration values.

Those can normally be stored in:

```text
ConfigMap
Helm values
environment configuration
```

---

# 4. Secret vs Configuration

Example:

```yaml
env:

  - name: SPRING_PROFILES_ACTIVE
    value: prod

  - name: LOG_LEVEL
    value: INFO
```

These are configuration values.

But:

```yaml
env:

  - name: DB_PASSWORD
    value: "mypassword"
```

is a secret.

Never commit the second pattern into Git.

---

# 5. Our Secret Architecture

For the Order Service:

```text
AWS Secrets Manager
        |
        | encrypted with KMS
        |
        v
External Secrets Operator
        |
        | Pod Identity
        |
        v
AWS IAM Role
        |
        v
Kubernetes Secret
        |
        v
Order Service
```

The application's permission model is:

```text
Order Service IAM Role
        |
        +--> GetSecretValue
        |
        +--> ONLY required secret
```

---

# 6. Why AWS Secrets Manager?

We selected AWS Secrets Manager because it provides:

```text
Centralized storage
Encryption
Access control
Audit integration
Secret versions
Rotation capabilities
AWS integration
```

Instead of:

```text
21 applications
21 .env files
21 password locations
```

we have:

```text
Centralized secret management
```

---

# 7. Example Secret Structure

For Order Service:

```text
prod/ecommerce/order-service/database
```

The secret could contain:

```json
{
  "username": "order_app",
  "password": "REDACTED",
  "host": "orders-prod.xxxxx.ap-south-1.rds.amazonaws.com",
  "port": 5432,
  "database": "orders"
}
```

The actual password should only exist in the secret store.

---

# 8. Naming Convention

Use predictable secret names.

For example:

```text
dev/ecommerce/order-service/database
qa/ecommerce/order-service/database
staging/ecommerce/order-service/database
prod/ecommerce/order-service/database
```

For another service:

```text
prod/ecommerce/payment-service/database
```

API:

```text
prod/ecommerce/payment-service/payment-provider
```

Redis:

```text
prod/ecommerce/order-service/redis
```

This gives us:

```text
environment
    /
application
    /
purpose
```

---

# 9. Why Environment Separation Matters

Don't use:

```text
ecommerce/order-service/database
```

for every environment.

Instead:

```text
dev/ecommerce/order-service/database
qa/ecommerce/order-service/database
staging/ecommerce/order-service/database
prod/ecommerce/order-service/database
```

This prevents:

```text
DEV → production credentials
```

and makes IAM policies easier to scope.

---

# 10. KMS Encryption

Secrets Manager encrypts secrets.

We can use:

```text
AWS managed encryption
```

or:

```text
customer-managed KMS key
```

For organizations with stronger governance requirements, a customer-managed key may be useful.

Architecture:

```text
Secret
  |
  v
Secrets Manager
  |
  v
KMS
  |
  v
Encrypted data
```

---

# 11. KMS Key Architecture

For production we can have a dedicated KMS key:

```text
alias/ecommerce-secrets-prod
```

Then:

```text
Secrets Manager
       |
       v
KMS Key
       |
       v
Encrypted secret
```

The KMS key itself needs strict access control.

---

# 12. KMS Is Not the Secret Store

Important distinction:

```text
Secrets Manager
    =
stores/manages secret

KMS
    =
encrypts/decrypts data
```

Don't try to use KMS as a database for secrets.

---

# 13. KMS Key Policy

A KMS key policy determines who can use/manage the key.

We should distinguish:

```text
Key administrators
Key users
AWS service principals
Security/audit users
```

Avoid:

```text
everyone
```

having:

```text
kms:*
```

---

# 14. Secret Access Flow

Suppose Order Service starts.

```text
Order Pod
   |
   v
ServiceAccount
   |
   v
EKS Pod Identity
   |
   v
Order Service IAM Role
   |
   v
Secrets Manager
   |
   v
Secret
```

If a customer-managed KMS key is involved:

```text
Secrets Manager
   |
   v
KMS
```

The application does not directly need to manage encryption.

---

# 15. External Secrets Operator

Kubernetes itself has:

```text
Secret
```

but we don't want developers manually storing secret values in Git.

External Secrets Operator solves this.

Architecture:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
```

The GitOps repository stores:

```text
which secret to fetch
```

but not:

```text
the secret value
```

---

# 16. Install External Secrets Operator

Create namespace:

```bash
kubectl create namespace external-secrets
```

Add the Helm repository:

```bash
helm repo add external-secrets \
  https://charts.external-secrets.io
```

Update:

```bash
helm repo update
```

Install using a pinned chart version in production:

```bash
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

For our actual production platform:

```text
Helm
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

should manage this installation.

---

# 17. Verify ESO

```bash
kubectl get pods -n external-secrets
```

Then:

```bash
kubectl get crds | grep external-secrets
```

You should see resources such as:

```text
externalsecrets.external-secrets.io
secretstores.external-secrets.io
clustersecretstores.external-secrets.io
```

---

# 18. SecretStore

A `SecretStore` defines how ESO accesses the external secret provider.

For our case:

```text
AWS Secrets Manager
```

Conceptually:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore

metadata:
  name: aws-secrets
  namespace: ecommerce-prod

spec:

  provider:

    aws:

      service: SecretsManager
      region: ap-south-1
```

The exact API version and provider fields should be pinned to the External Secrets Operator version deployed in the cluster.

---

# 19. ClusterSecretStore

We could also use:

```text
ClusterSecretStore
```

which is cluster-wide.

But be careful.

If every namespace can reference a broad secret store:

```text
all namespaces
       |
       v
AWS Secrets Manager
```

the blast radius becomes larger.

For our application secrets, namespace-scoped `SecretStore` objects can provide stronger isolation.

---

# 20. ExternalSecret

Now define which AWS secret should be synchronized.

Example:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:
  name: order-service-database
  namespace: ecommerce-prod

spec:

  refreshInterval: 1h

  secretStoreRef:
    name: aws-secrets
    kind: SecretStore

  target:
    name: order-service-database
    creationPolicy: Owner

  data:
    - secretKey: username
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: username

    - secretKey: password
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: password

    - secretKey: host
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: host
```

Again, use the exact API version/schema supported by the pinned ESO release.

---

# 21. What Gets Stored in Git?

The GitOps repository can contain:

```yaml
remoteRef:
  key: prod/ecommerce/order-service/database
```

But NOT:

```yaml
password: MyRealPassword
```

Therefore:

```text
GitOps
   |
   +--> secret reference
   |
   X
   +--> secret value
```

---

# 22. Resulting Kubernetes Secret

ESO creates:

```text
Secret
order-service-database
```

Conceptually:

```text
order-service-database
   |
   +--> username
   +--> password
   +--> host
```

The application can consume this Secret.

---

# 23. Environment Variable Injection

For Spring Boot:

```yaml
env:

  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: order-service-database
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-database
        key: password

  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: order-service-database
        key: host
```

Then:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/orders
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

---

# 24. Secret as a Mounted File

Another option is mounting the secret:

```yaml
volumeMounts:
  - name: database-secret
    mountPath: /etc/secrets/database
    readOnly: true
```

and:

```yaml
volumes:
  - name: database-secret
    secret:
      secretName: order-service-database
```

Then the container gets:

```text
/etc/secrets/database/
    username
    password
    host
```

Choose environment variables or files based on application requirements.

---

# 25. Important Secret Rotation Problem

Suppose:

```text
AWS Secrets Manager
password = OLD
```

Then we rotate:

```text
password = NEW
```

ESO updates the Kubernetes Secret.

But what about the running application?

If the application read the environment variable at startup:

```text
Pod starts
   |
   v
DB_PASSWORD=OLD
```

then changing the Kubernetes Secret doesn't necessarily change the already-running process environment.

This is an important production issue.

---

# 26. Secret Rotation Flow

The complete process is:

```text
Old Secret
    |
    v
Rotation
    |
    v
AWS Secrets Manager
    |
    v
ESO detects update
    |
    v
Kubernetes Secret updated
    |
    v
Application must reload/restart
```

The final step depends on how the application consumes the secret.

---

# 27. Environment Variables and Rotation

If secrets are injected through environment variables:

```text
Kubernetes Secret
      |
      v
Pod environment
```

the existing process generally retains the old value.

Therefore:

```text
Secret changed
      |
      v
Restart/redeploy required
```

We can automate this through GitOps/controller mechanisms, but it should be deliberately designed and tested.

---

# 28. Mounted Secret Files

If the application reads a mounted Secret volume, Kubernetes can update the mounted content after the Secret changes.

But the application must actually re-read the file.

If the application loads the value only once at startup:

```text
startup
  ↓
read file
  ↓
memory
```

the application still needs a reload mechanism.

---

# 29. Production Rotation Strategy

For Order Service database credentials:

```text
Rotation
   ↓
New credential created
   ↓
RDS/database accepts new credential
   ↓
Secrets Manager updated
   ↓
ESO updates Kubernetes Secret
   ↓
Application reload/restart
   ↓
Health check
   ↓
Old credential revoked
```

The exact order matters.

Don't revoke the old credential first.

---

# 30. Database Rotation Risk

Imagine:

```text
Old password = A
New password = B
```

If we immediately revoke A:

```text
Existing pods
   |
   +--> still using A
   |
   v
FAIL
```

Therefore a safe rotation strategy needs coordination.

---

# 31. Rotation Patterns

Common approaches include:

### Dual credentials

```text
Credential A = current
Credential B = new
```

Move workloads to B.

Then revoke A.

Or use a managed rotation mechanism where supported.

---

# 32. RDS Secrets

Our previous RDS chapter discussed managed credentials.

The architecture can be:

```text
RDS
 |
 v
Secrets Manager
 |
 v
External Secrets
 |
 v
Kubernetes
```

This avoids putting database passwords in:

```text
Terraform variables
Helm values
Git
```

---

# 33. Terraform State Warning

This is extremely important.

Terraform can store sensitive values in state depending on the resources/data involved.

Therefore:

```text
Terraform state
```

must be treated as sensitive.

Our state architecture already uses:

```text
S3
+
encryption
+
versioning
+
locking
+
restricted IAM
```

But:

> `sensitive = true` does not magically remove a value from Terraform state.

It mainly prevents casual display in CLI output.

---

# 34. Don't Do This

Avoid:

```hcl
variable "db_password" {
  default = "MyPassword123"
}
```

Never commit:

```text
terraform.tfvars
```

containing production credentials.

Don't commit:

```text
secret.auto.tfvars
```

either.

---

# 35. Terraform Secret Creation

Where possible, use:

```text
AWS-managed secret generation
```

rather than hardcoding the value into Terraform source.

For example, a password can be generated and stored through an appropriate AWS-managed mechanism.

But remember:

```text
Terraform
    |
    v
State
```

may still contain sensitive data depending on the resource and implementation.

So state access must remain highly restricted.

---

# 36. Kubernetes Secrets Are Not Plaintext in Normal API Responses

Kubernetes stores Secret data encoded rather than simply showing plaintext in standard manifests.

But:

```text
Base64 != encryption
```

Base64 is encoding.

For example:

```text
password
```

becomes something like:

```text
cGFzc3dvcmQ=
```

That is not security.

---

# 37. Encrypt Kubernetes Secrets at Rest

EKS should use encryption for Kubernetes Secrets at rest.

Architecture:

```text
Kubernetes Secret
       |
       v
Kubernetes API / etcd
       |
       v
Encryption at Rest
       |
       v
KMS
```

This protects stored Secret data.

But it does not solve:

```text
who can read the Secret
```

RBAC still matters.

---

# 38. Two Separate Controls

```text
Encryption
   |
   +--> protects stored data

RBAC
   |
   +--> controls who can retrieve it
```

You need both.

---

# 39. Never Put Secrets in ConfigMaps

Bad:

```yaml
apiVersion: v1
kind: ConfigMap

data:
  DB_PASSWORD: "secret"
```

ConfigMap is for non-sensitive configuration.

Use:

```text
Secret
```

for sensitive values.

---

# 40. Never Put Secrets in Helm Values

Bad:

```yaml
database:
  username: order_user
  password: MyPassword
```

GitOps should contain:

```yaml
externalSecret:
  name: order-service-database
```

not the actual password.

---

# 41. Never Put Secrets in Docker Images

Never:

```dockerfile
COPY .env /app/.env
```

Never:

```dockerfile
ENV PAYMENT_API_KEY=...
```

Images may be stored for years.

A secret accidentally embedded in an image can remain accessible even after the application is fixed.

---

# 42. Never Log Secrets

Bad:

```java
log.info("Database password: {}", password);
```

Bad:

```text
Authorization: Bearer eyJ...
```

Bad:

```text
payment-api-key=...
```

Logging systems are designed to retain data.

A secret in:

```text
ELK
```

can become much harder to remove.

---

# 43. Never Put Secrets in Metrics

Avoid labels such as:

```text
http_request{api_key="..."}
```

or:

```text
payment_request{token="..."}
```

Prometheus labels are especially dangerous because high-cardinality/sensitive values can spread through monitoring infrastructure.

---

# 44. Never Put Secrets in Traces

OpenTelemetry spans may include:

```text
HTTP headers
request bodies
attributes
database statements
```

Do not blindly capture:

```text
Authorization
Cookie
API-Key
password
```

Our observability architecture needs explicit sensitive-data filtering.

---

# 45. Secret Detection in Git

We already introduced:

```text
Gitleaks
```

Pipeline:

```text
Developer
   |
   v
Git commit
   |
   v
Pull Request
   |
   v
Gitleaks
   |
   +--> secret found
           |
           v
          FAIL
```

But remember:

> Secret scanning is prevention, not a guarantee.

Developers can still accidentally leak secrets through:

```text
logs
tickets
screenshots
chat
artifacts
container images
```

---

# 46. What If a Secret Is Accidentally Committed?

Don't simply delete the line and assume everything is fixed.

Suppose:

```text
Git commit A
    |
    +--> API_KEY=SECRET
```

Even if commit B removes it:

```text
Git commit B
    |
    X
```

the secret may remain in Git history.

The response should be:

```text
1. Revoke/rotate the exposed credential
2. Investigate usage
3. Remove secret from repository/history where appropriate
4. Scan for further exposure
5. Issue replacement credential
6. Document incident
```

Rotation is the critical step.

---

# 47. Secret Access Auditing

AWS CloudTrail can help us investigate access to AWS services.

For example:

```text
Who accessed Secrets Manager?
When?
Which secret?
From which identity?
```

This becomes part of our Chapter 91 security monitoring.

---

# 48. IAM Policy for ESO

External Secrets Operator needs permission to retrieve secrets.

For example:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": [
    "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/ecommerce/order-service/*"
  ]
}
```

This is much better than:

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:*",
  "Resource": "*"
}
```

---

# 49. ESO Identity Architecture

We need to decide which AWS identity ESO uses.

A production architecture can use:

```text
External Secrets Operator
        |
        v
Dedicated ServiceAccount
        |
        v
EKS Pod Identity
        |
        v
ESO IAM Role
        |
        v
Secrets Manager
```

This means:

```text
ESO
```

gets only the secrets it needs.

---

# 50. Application vs ESO Permissions

This is a subtle but important design.

Option A:

```text
Order Service
    |
    +--> Secrets Manager
```

Option B:

```text
ESO
 |
 +--> Secrets Manager
 |
 v
Kubernetes Secret
 |
 v
Order Service
```

Our preferred architecture is generally **B** for standard application configuration secrets.

Then:

```text
Order Service IAM Role
```

doesn't necessarily need:

```text
secretsmanager:GetSecretValue
```

because ESO handles retrieval.

This reduces application AWS permissions.

---

# 51. Why This Is Better

Instead of:

```text
21 services
   |
   +--> Secrets Manager
```

we can have:

```text
ESO
   |
   +--> Secrets Manager
```

with carefully scoped secret access.

Then applications receive only the secrets they need.

---

# 52. But Kubernetes Secrets Still Need Protection

Yes.

Once ESO creates:

```text
Secret
```

we must protect it with:

```text
RBAC
Encryption at rest
Namespace isolation
Pod security
```

So the chain is:

```text
AWS Secrets Manager
       |
      KMS
       |
       v
ESO
       |
       v
Kubernetes Secret
       |
   +---+---+
   |       |
 RBAC   Encryption
   |       |
   +---+---+
       |
       v
      Pod
```

---

# 53. Secret Refresh

Example:

```yaml
spec:
  refreshInterval: 1h
```

means ESO periodically checks the external secret.

Depending on the chosen configuration, you may use a shorter or longer interval.

Don't automatically use:

```text
1 minute
```

for every secret.

That increases API calls and isn't necessarily useful.

---

# 54. Different Secret Frequencies

Example:

```text
Database credentials
    ↓
hours/days depending on rotation

Payment API credentials
    ↓
organization policy

TLS certificates
    ↓
certificate lifecycle

Emergency credentials
    ↓
immediate rotation
```

The refresh strategy should reflect the secret's lifecycle.

---

# 55. Secret Rotation vs Secret Refresh

These are different.

### Rotation

The secret value itself changes.

```text
password A
    ↓
password B
```

### Refresh

ESO checks whether the external value changed.

```text
ESO
 ↓
AWS Secrets Manager
 ↓
same value
```

No actual rotation occurred.

---

# 56. Failure Scenario — AWS Secrets Manager Unavailable

Suppose:

```text
ESO
  X
Secrets Manager
```

What happens?

Existing Kubernetes Secret may remain available.

The application may continue using its existing credentials.

But:

```text
new pods
```

or:

```text
secret refresh
```

could fail.

Therefore monitor:

```text
ExternalSecret status
ESO logs
AWS API errors
```

---

# 57. Failure Scenario — Secret Does Not Exist

Suppose GitOps says:

```text
prod/ecommerce/order-service/database
```

but the secret was never created.

ESO reports an error.

Check:

```bash
kubectl get externalsecret \
  order-service-database \
  -n ecommerce-prod
```

Then:

```bash
kubectl describe externalsecret \
  order-service-database \
  -n ecommerce-prod
```

---

# 58. Failure Scenario — AccessDenied

If ESO receives:

```text
AccessDeniedException
```

check:

```text
ServiceAccount
      ↓
Pod Identity
      ↓
IAM Role
      ↓
IAM Policy
      ↓
Secret ARN
```

A common mistake:

```text
Policy allows:
prod/ecommerce/order-service/*
```

but the actual secret is:

```text
prod/ecommerce/orderservice/*
```

Exact resource naming matters.

---

# 59. Failure Scenario — Application Starts With Old Password

Suppose:

```text
Secrets Manager
OLD → NEW
```

ESO updates Kubernetes Secret.

But:

```text
Pod
DB_PASSWORD=OLD
```

because it came from an environment variable.

The solution is not:

```text
disable rotation
```

Instead implement:

```text
Secret update
   ↓
controlled workload restart/reload
   ↓
new credential
```

This can be automated where appropriate.

---

# 60. Secret Rotation and Zero Downtime

For production:

```text
Old credential
     |
     +--> existing pods
     |
New credential
     |
     +--> new pods
```

We want overlap during transition.

Example:

```text
Pod 1 → old credential
Pod 2 → old credential
Pod 3 → old credential

       rotation

Pod 4 → new credential
Pod 5 → new credential
Pod 6 → new credential
```

Only revoke the old credential after all required workloads have moved successfully.

This is particularly important for databases.

---

# 61. Secret Rotation With Rolling Deployment

A safe pattern:

```text
Update secret
    ↓
Validate new credential
    ↓
Restart/roll workload
    ↓
Readiness checks
    ↓
New pods healthy
    ↓
Old pods terminated
    ↓
Verify
    ↓
Revoke old credential
```

This connects directly to Chapter 81's rolling deployment strategy.

---

# 62. Secret Rotation and GitOps

GitOps should contain:

```yaml
externalSecret:
  name: order-service-database
```

It should not contain:

```yaml
password: NewPassword
```

Secret rotation therefore happens outside normal GitOps value changes.

The GitOps repository defines:

> **Which secret should be consumed.**

AWS Secrets Manager defines:

> **What the secret value is.**

---

# 63. Secret Ownership

Define ownership clearly.

For example:

```text
Database credentials
    ↓
Platform/DB team

Payment credentials
    ↓
Payment/platform owners

Application-specific API keys
    ↓
Application team

KMS keys
    ↓
Security/platform team
```

Without ownership:

```text
secret expired
   ↓
nobody knows who fixes it
```

---

# 64. Production Secret Naming Standard

Recommended:

```text
<environment>/ecommerce/<service>/<purpose>
```

Examples:

```text
dev/ecommerce/order-service/database
qa/ecommerce/order-service/database
staging/ecommerce/order-service/database
prod/ecommerce/order-service/database
```

Payment:

```text
prod/ecommerce/payment-service/provider
```

Redis:

```text
prod/ecommerce/order-service/redis
```

---

# 65. Secret Metadata

Secrets should have useful metadata/tags.

For example:

```text
Environment = prod
Application  = order-service
Owner        = backend-team
ManagedBy    = terraform
Purpose      = database
```

This helps:

```text
inventory
cost/governance
ownership
automation
auditing
```

---

# 66. Secret Deletion

Don't immediately delete production secrets.

A safer lifecycle:

```text
Active
  ↓
Deprecated
  ↓
Unused
  ↓
Recovery period
  ↓
Delete
```

Keep recovery windows according to organizational requirements.

---

# 67. Disaster Recovery

Secrets are part of DR.

Suppose:

```text
ap-south-1
```

fails.

Can the recovery environment access:

```text
Secrets Manager
KMS
```

?

We need to plan:

```text
Secret replication
KMS strategy
IAM roles
region availability
ESO configuration
```

---

# 68. Multi-Region Secret Strategy

Conceptually:

```text
Primary Region
ap-south-1
      |
      v
Secrets Manager
      |
      v
Recovery Region
ap-south-2/example
```

The exact AWS regional architecture should be designed according to the application's DR model.

Don't simply copy every secret everywhere.

---

# 69. Secret Access During DR

When EKS is recreated:

```text
New EKS
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
Secrets Manager
```

All these dependencies must exist.

DR is therefore not just:

```text
restore EKS
```

It is:

```text
restore identity
+
restore encryption
+
restore secret availability
+
restore application
```

---

# 70. Security Checklist

## Storage

```text
[✓] AWS Secrets Manager
[✓] KMS encryption
[✓] No secrets in Git
[✓] No secrets in Docker
[✓] No secrets in ConfigMaps
```

## Access

```text
[✓] Least-privilege IAM
[✓] EKS Pod Identity
[✓] Dedicated ServiceAccounts
[✓] Restricted Kubernetes RBAC
[✓] Namespace isolation
```

## Kubernetes

```text
[✓] External Secrets Operator
[✓] SecretStore
[✓] ExternalSecret
[✓] Encryption at rest
[✓] Secret access restricted
```

## Application

```text
[✓] No secret logging
[✓] No secret metrics
[✓] No secret traces
[✓] No secret command-line arguments
[✓] Controlled reload/restart
```

## Operations

```text
[✓] Rotation strategy
[✓] Rotation monitoring
[✓] Audit trail
[✓] Ownership
[✓] Expiration/review
[✓] DR strategy
```

---

# 71. Complete Production Flow

Our Order Service secret architecture is now:

```text
                    AWS KMS
                       |
                       v
              AWS Secrets Manager
                       |
             prod/ecommerce/
             order-service/
             database
                       |
                       v
              External Secrets
                  Operator
                       |
                  Pod Identity
                       |
                  IAM Role
                       |
                       v
              Kubernetes Secret
                       |
               +-------+-------+
               |               |
          RBAC protects    Encryption
               |             at rest
               +-------+-------+
                       |
                       v
                 Order Service
                       |
                 Spring Boot
                       |
              DB_USERNAME
              DB_PASSWORD
              DB_HOST
                       |
                       v
                 PostgreSQL
```

---

# 72. Complete Security Architecture

Our security chain has now become:

```text
GitHub
   |
   +--> Gitleaks
   +--> SonarQube
   +--> Dependency Scan
   |
   v
Docker
   |
   +--> Trivy
   +--> SBOM
   |
   v
ECR
   |
   +--> Immutable
   +--> Scan
   +--> Sign
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
Admission
   |
   +--> Pod Security
   +--> Kyverno
   +--> Image Verification
   |
   v
EKS
   |
   +--> SecurityContext
   +--> NetworkPolicy
   +--> RBAC
   |
   v
Pod Identity
   |
   v
IAM
   |
   +--> Secrets Manager
   |        |
   |        v
   |       KMS
   |
   +--> SQS
   +--> S3
   +--> other AWS services
```

---

# 73. Important Production Principle

Never think:

> "The password is in Kubernetes Secret, so it's secure."

Instead ask:

```text
Where is it created?
        ↓
How is it encrypted?
        ↓
Who can read it?
        ↓
How does it reach the pod?
        ↓
Can it appear in logs?
        ↓
How is it rotated?
        ↓
What happens when it expires?
        ↓
How is it recovered during DR?
```

That is production-grade secret management.

---

# 74. What We Have Completed

Security phase:

```text
85  Production Networking
86  Kubernetes Security Hardening
87  Container Image Security / SBOM / Signing
88  Admission Control / Image Verification
89  Kubernetes RBAC + AWS IAM
90  Secrets + KMS + Rotation
```

So our security architecture is now:

```text
CODE
 ↓
IMAGE
 ↓
IDENTITY
 ↓
ADMISSION
 ↓
RUNTIME
 ↓
SECRETS
 ↓
AWS
```

---

# 75. Next Chapter

## 91 — AWS Security Monitoring and Detection

We'll now move from **preventing security problems** to **detecting them**.

We'll build the architecture around:

```text
CloudTrail
     |
     +--> IAM activity
     +--> API calls
     +--> resource changes
     |
     v
GuardDuty
     |
     +--> threat detection
     |
     v
Security Hub
     |
     +--> findings
     |
     v
CloudWatch / EventBridge
     |
     v
Alerting
     |
     v
Incident Response
```

We'll also connect this with our existing:

```text
EKS Audit Logs
VPC Flow Logs
Prometheus
Grafana
ELK
Alertmanager
```

so that AWS-level security events and Kubernetes-level security events become part of **one production security monitoring and incident-response model**.
