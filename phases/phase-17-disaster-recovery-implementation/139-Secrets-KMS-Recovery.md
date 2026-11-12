# 139 — SECRETS / KMS RECOVERY

## 1. Objective

This chapter covers production recovery of **AWS Secrets Manager, AWS KMS, External Secrets Operator, and EKS workload identity** for our 21-microservice e-commerce platform.

Secrets and encryption keys are foundational dependencies.

A service may have:

```text id="a7v4q1"
Healthy Pod
Healthy Node
Healthy EKS
Healthy RDS
```

and still fail because:

```text id="2n7x8d"
Secret unavailable
      OR
KMS access denied
      OR
Pod Identity broken
      OR
External Secrets Operator unavailable
```

The recovery architecture therefore must treat:

```text id="k2z6a0"
Secrets
IAM
KMS
Pod Identity
External Secrets
Applications
```

as one security dependency chain.

---

# 2. Our Production Secrets Architecture

The intended architecture is:

```text id="w0q4m8"
                 AWS Secrets Manager
                         |
                         v
                  External Secrets
                      Operator
                         |
                         v
                 Kubernetes Secret
                         |
                         v
                    Application
```

Authentication:

```text id="h4n8x2"
Application / ESO
        |
        v
EKS Pod Identity
        |
        v
IAM Role
        |
        v
Secrets Manager
        |
        v
KMS
```

The application should not contain AWS access keys.

---

# 3. What Goes Into Secrets Manager?

Examples:

```text id="c8m4v1"
Database credentials
External API credentials
Payment provider secrets
SMTP credentials
Third-party tokens
Internal service credentials
Webhook secrets
Encryption-related application secrets
```

Do **not** store:

```text id="6y3n9p"
Passwords in Git
AWS access keys in Dockerfiles
Secrets in Helm values
Secrets in GitHub source
Secrets in application.properties
Secrets in container images
```

---

# 4. Example Production Secrets

For `order-service`:

```text id="8q5r2m"
company/ecommerce/prod/order-service/database
```

Possible content:

```json id="g2h7s1"
{
  "username": "order_app",
  "password": "********",
  "host": "ecommerce-prod.xxxxxx.ap-south-1.rds.amazonaws.com",
  "port": "5432",
  "database": "orders"
}
```

For payment:

```text id="v6m2q8"
company/ecommerce/prod/payment-service/provider
```

For notification:

```text id="k4p9t3"
company/ecommerce/prod/notification-service/provider
```

---

# 5. Secret Ownership

Use service-specific ownership.

```text id="s3d7f9"
order-service
   |
   +--> order-service secrets

payment-service
   |
   +--> payment-service secrets

notification-service
   |
   +--> notification-service secrets
```

Avoid:

```text id="j8x2c6"
one giant secret
containing credentials for all 21 services
```

because compromise of one workload would expose unrelated credentials.

---

# 6. Recovery Scenarios

We need to handle:

```text id="r2c7m5"
1. Secret unavailable
2. Secret deleted
3. Wrong secret version
4. Secret rotation failure
5. External Secrets failure
6. Pod Identity failure
7. IAM AccessDenied
8. KMS AccessDenied
9. KMS key disabled
10. KMS key deletion scheduling
11. Region failure
12. Secret replication failure
13. Compromised secret
14. Application startup failure
15. Complete secrets-platform recovery
```

---

# 7. First Recovery Rule

When an application reports:

```text id="u6q4p9"
Unable to load database password
```

do not immediately recreate the secret.

First determine:

```text id="p8m3x7"
Does the secret exist?
       |
       v
Can ESO access it?
       |
       v
Can IAM assume the required role?
       |
       v
Can KMS decrypt it?
       |
       v
Is the correct secret version selected?
       |
       v
Did Kubernetes receive the Secret?
```

---

# 8. Recovery Dependency Chain

The complete dependency path is:

```text id="z1w5n8"
Application
     |
     v
Kubernetes Secret
     |
     v
ExternalSecret
     |
     v
External Secrets Operator
     |
     v
EKS Pod Identity
     |
     v
IAM Role
     |
     v
Secrets Manager
     |
     v
KMS
```

A failure anywhere can appear as an application secret failure.

---

# 9. Check Application Pod

Start with:

```bash id="p7m4c2"
kubectl get pods \
  -n ecommerce-prod
```

Then:

```bash id="d9k2w6"
kubectl describe pod \
  <pod-name> \
  -n ecommerce-prod
```

Look for:

```text id="r4v8m1"
Secret not found
FailedMount
AccessDenied
ExternalSecret not ready
```

Do not print secret values.

---

# 10. Check Kubernetes Secret

Example:

```bash id="n3h7q5"
kubectl get secret \
  order-db \
  -n ecommerce-prod
```

You can inspect metadata:

```bash id="c6m1v8"
kubectl describe secret \
  order-db \
  -n ecommerce-prod
```

Do not casually run:

```bash id="e2x7p4"
kubectl get secret order-db -o yaml
```

in an incident channel because the output may contain encoded sensitive data.

Base64 is encoding, not encryption.

---

# 11. Check ExternalSecret

Example:

```bash id="v5k8q3"
kubectl get externalsecret \
  -n ecommerce-prod
```

Then:

```bash id="m2r7w9"
kubectl describe externalsecret \
  order-db \
  -n ecommerce-prod
```

Look for:

```text id="f4p1x8"
Ready
SecretSynced
Refresh interval
Provider errors
Authentication errors
```

---

# 12. External Secrets Operator

Architecture:

```text id="n9c3v6"
ExternalSecret
      |
      v
External Secrets Operator
      |
      v
AWS Secrets Manager
```

Check:

```bash id="q6t2m8"
kubectl get pods \
  -n external-secrets
```

Then:

```bash id="x4p9k1"
kubectl get deployment \
  -n external-secrets
```

If ESO itself is unavailable:

```text id="j8r3w5"
AWS Secret
   |
   X
ESO
   |
   X
Kubernetes Secret
```

Applications may continue working with an already-created Kubernetes Secret, but new/updated secrets will not synchronize.

---

# 13. ESO Recovery

If ESO is missing after EKS rebuild:

```text id="y2k6v4"
New EKS
   |
   v
ArgoCD
   |
   v
External Secrets Operator
   |
   v
ExternalSecret resources
   |
   v
Kubernetes Secrets
```

ESO should be restored through GitOps rather than manually recreated for every service.

---

# 14. Secrets Manager Verification

Check the secret metadata:

```bash id="h7m2q9"
aws secretsmanager describe-secret \
  --secret-id company/ecommerce/prod/order-service/database \
  --region ap-south-1
```

Inspect:

```text id="p4c8n1"
ARN
Name
Status
KmsKeyId
RotationEnabled
LastChangedDate
DeletedDate
VersionIdsToStages
```

Do not expose the actual secret value.

---

# 15. Secret Does Not Exist

If:

```text id="k1v7m4"
ResourceNotFoundException
```

investigate:

```text id="c8p2x5"
Was it deleted?
Wrong region?
Wrong account?
Wrong secret name?
Wrong environment?
Terraform resource missing?
```

A common mistake:

```text id="q9m5r3"
Production application
      |
      v
Secrets Manager
      |
      v
ap-southeast-1
```

while the actual secret exists in:

```text id="s6k2v8"
ap-south-1
```

---

# 16. Secret Deletion

AWS Secrets Manager supports recovery behavior for secrets that are scheduled for deletion, subject to the service's deletion and recovery rules.

First check:

```bash id="r7x3m9"
aws secretsmanager describe-secret \
  --secret-id company/ecommerce/prod/order-service/database \
  --region ap-south-1
```

Look for:

```text id="c4n8p1"
DeletedDate
```

If the secret is scheduled for deletion and recovery is appropriate, cancel the deletion using the approved recovery procedure.

---

# 17. Important Recovery Rule

Do not immediately create a new secret with the same name.

First determine:

```text id="v2m7q4"
Original secret
       |
       v
Scheduled deletion?
       |
       +--> YES
             |
             v
       Restore if appropriate
```

Creating a replacement too quickly can create:

```text id="g9c5x2"
Old secret
New secret
Different ARN/version
Application ambiguity
```

---

# 18. Secret Versions

Secrets Manager supports multiple versions.

Conceptually:

```text id="m4p8y1"
Secret
 |
 +--> Version A
 |
 +--> Version B
 |
 +--> Version C
```

Stages identify which version is currently active.

Common staging concepts include:

```text id="f8q2r6"
AWSCURRENT
AWSPREVIOUS
```

This is useful during controlled rotation.

---

# 19. Wrong Secret Version

Suppose:

```text id="z5k1m8"
AWSCURRENT
   |
   v
wrong password
```

Application:

```text id="n7q3p9"
Database authentication failed
```

Do not assume RDS is broken.

Check:

```text id="x6c2v4"
Current secret version
Previous secret version
Database password
Rotation process
Application synchronization
```

---

# 20. Secret Rotation

Normal rotation:

```text id="j4m8q2"
Current credential
       |
       v
Create new credential
       |
       v
Validate
       |
       v
Promote new secret
       |
       v
Application refresh
```

The application must be able to handle the transition.

---

# 21. Rotation Failure

Example:

```text id="b8x5m3"
RDS password changed
       |
       X
Secrets Manager update failed
```

Result:

```text id="c2p9v7"
Application
     |
     v
Old password
     |
     X
RDS
```

The database may be healthy.

The secret synchronization is broken.

---

# 22. Rotation Recovery

Investigate:

```text id="u7r3k9"
1. What changed?
2. Which credential is valid?
3. Which secret version is current?
4. Was the database password changed?
5. Did ESO synchronize?
6. Did application Pods reload?
```

Then:

```text id="a4m8x2"
Establish valid credential
        ↓
Update secret
        ↓
Synchronize Kubernetes Secret
        ↓
Restart/reload application if required
        ↓
Validate DB connection
```

---

# 23. Kubernetes Secret Refresh

An important design detail:

If an application consumes a Secret as environment variables:

```yaml id="x2c6v8"
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-db
        key: password
```

a running Pod does not automatically update its process environment when the Secret changes.

Therefore:

```text id="q5m9r3"
Secret changed
     |
     v
Kubernetes Secret updated
     |
     v
Application restart/reload required
```

unless the application uses another supported dynamic-secret mechanism.

---

# 24. Controlled Rollout After Secret Change

Example:

```bash id="r8v4m2"
kubectl rollout restart deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash id="p1x7c5"
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

In production, prefer the approved GitOps workflow when the secret/configuration change is represented declaratively.

---

# 25. Pod Identity

Our architecture:

```text id="w3k9q6"
ESO Pod
   |
   v
EKS Pod Identity
   |
   v
IAM Role
   |
   v
Secrets Manager
```

If Pod Identity breaks:

```text id="z7m4c1"
ESO
  |
  X
AWS API
```

The secret itself may be completely healthy.

---

# 26. IAM AccessDenied

Example:

```text id="q2v8m5"
AccessDeniedException
```

Check:

```text id="c9r3k7"
Pod Identity association
IAM role
IAM policy
Secret ARN
KMS key
Region
AWS account
```

The policy should follow least privilege.

Example conceptual permission:

```json id="n5p1x8"
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:company/ecommerce/prod/order-service/database-*"
}
```

The exact ARN and policy conditions should match the deployed resource.

---

# 27. KMS Dependency

Secrets Manager may use a KMS key for encryption.

Architecture:

```text id="v8m2q6"
ESO
 |
 v
Secrets Manager
 |
 v
KMS
```

If KMS access fails:

```text id="f4x9p3"
Secret exists
       |
       X
KMS decrypt
```

Application may report a generic secret retrieval failure.

---

# 28. KMS AccessDenied

Investigate:

```text id="c7m3v9"
IAM policy
Key policy
KMS key ARN
Role
Account
Region
Grant configuration where applicable
```

Do not solve:

```text id="p5r8x2"
KMS AccessDenied
```

by granting:

```text id="d1q6m4"
kms:*
Resource: *
```

unless there is an explicitly approved administrative reason.

---

# 29. KMS Key Disabled

A KMS key can be disabled.

Conceptually:

```text id="m8v2r5"
Secret
  |
  v
KMS key
  |
  X
Disabled
```

Existing encrypted data may become inaccessible to workloads that require the key.

First investigate:

```text id="x3q7k1"
Who disabled the key?
When?
Why?
Was it intentional?
Which services depend on it?
```

CloudTrail is important here.

---

# 30. KMS Key Deletion

KMS key deletion is especially dangerous.

A key scheduled for deletion should trigger an immediate security/infrastructure investigation for production keys.

Do not casually create a new key and assume:

```text id="j6p4x9"
New KMS key
=
old encrypted data recoverable
```

It does not.

Existing ciphertext depends on the appropriate encryption key material and configuration.

---

# 31. KMS Recovery Principle

Treat production KMS keys as long-lived security infrastructure.

Architecture:

```text id="q8m3v7"
KMS
 |
 +--> RDS
 |
 +--> S3
 |
 +--> Secrets Manager
 |
 +--> ECR
```

Therefore deleting or replacing a key can have a much larger blast radius than one application.

---

# 32. KMS Key Inventory

Maintain an inventory:

```text id="c4r7n1"
Key
Purpose
Environment
Region
Owner
Services
Rotation
Policy
Deletion protection process
```

Example:

| Key                            | Purpose         | Consumers               |
| ------------------------------ | --------------- | ----------------------- |
| `alias/ecommerce-prod-secrets` | Secrets Manager | ESO / applications      |
| `alias/ecommerce-prod-rds`     | RDS             | PostgreSQL              |
| `alias/ecommerce-prod-s3`      | S3              | Invoice/report services |
| `alias/ecommerce-prod-ecr`     | ECR             | Container registry      |

Actual key separation should follow security and operational requirements.

---

# 33. Secret-to-KMS Dependency Map

```text id="t7x2m5"
order-service
     |
     v
order-db secret
     |
     v
Secrets Manager
     |
     v
Secrets KMS Key
```

Another:

```text id="r4c8p1"
invoice-service
     |
     v
Invoice S3 object
     |
     v
S3
     |
     v
S3 KMS Key
```

These are different recovery dependencies.

---

# 34. Secret Recovery After EKS Rebuild

Suppose the EKS cluster is lost.

New cluster:

```text id="q9m3v7"
EKS
 |
 v
ESO
 |
 v
Pod Identity
 |
 v
IAM
 |
 v
Secrets Manager
 |
 v
KMS
```

Then:

```text id="x4p8c2"
ExternalSecret
 |
 v
Kubernetes Secret
 |
 v
Application
```

This should happen automatically through GitOps and infrastructure-as-code.

---

# 35. GitOps Secret Configuration

Do not put:

```yaml id="b6m2q9"
password: MyProductionPassword
```

in Git.

Instead, Git stores a reference:

```yaml id="z3r7k1"
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: order-db
  namespace: ecommerce-prod
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore

  target:
    name: order-db
```

The exact External Secrets API fields should be matched to the version deployed in the cluster.

---

# 36. Secret Store

Architecture:

```text id="m8q4v1"
ClusterSecretStore
       |
       v
AWS Secrets Manager
```

The store defines how ESO reaches AWS.

After cluster recovery:

```text id="p6x2c8"
ESO
 |
 +--> SecretStore
 |
 +--> ExternalSecret
```

must both be restored.

---

# 37. Secret Sync Failure

Check:

```bash id="v2m8q5"
kubectl get externalsecret \
  -A
```

Then:

```bash id="n4c7x1"
kubectl describe externalsecret \
  order-db \
  -n ecommerce-prod
```

Potential errors:

```text id="h6p3r9"
AccessDenied
SecretNotFound
Invalid KMS key
Authentication failure
Wrong region
Secret key missing
```

---

# 38. Secret Key Mapping

Suppose AWS secret contains:

```json id="w7m3q8"
{
  "username": "order_app",
  "password": "secret",
  "host": "rds-endpoint"
}
```

Kubernetes might require:

```text id="q2c6v9"
DB_USERNAME
DB_PASSWORD
DB_HOST
```

ESO must map the correct values.

A typo such as:

```text id="p8x4m1"
passwrod
```

can cause:

```text id="n6r2v7"
Application startup failure
```

---

# 39. Application Startup Failure

Spring Boot may report:

```text id="m3c8q5"
Failed to configure a DataSource
```

or:

```text id="v7x2n4"
password authentication failed
```

Do not immediately rebuild RDS.

Trace:

```text id="f4m9q2"
Application
   |
   v
Kubernetes Secret
   |
   v
ESO
   |
   v
Secrets Manager
   |
   v
KMS
```

---

# 40. Database Secret Recovery

For our order-service:

```text id="x8p3m7"
RDS
 |
 +--> Username
 +--> Password
 +--> Endpoint
 +--> Database
```

Store connection information through the approved secret/configuration pattern.

Example:

```text id="r6c2q9"
DB_URL
DB_USERNAME
DB_PASSWORD
```

After recovery:

```text id="k4m8v1"
Secret
 |
 v
order-service
 |
 v
RDS
```

---

# 41. External API Secret Recovery

Payment-service may use:

```text id="z7q3m5"
Payment provider API credential
```

Recovery must verify:

```text id="c9x4r8"
Credential exists
Credential not revoked
Correct environment
Correct provider endpoint
Correct KMS access
Application loaded latest version
```

A secret being present does not mean the external credential remains valid.

---

# 42. Compromised Secret

If a production secret is compromised:

```text id="m2v8q4"
DO NOT simply restore the old secret.
```

Recovery should be:

```text id="p6r3x9"
Contain
  ↓
Revoke/disable old credential
  ↓
Generate new credential
  ↓
Store new secret
  ↓
Synchronize
  ↓
Restart/reload applications
  ↓
Validate
  ↓
Audit old credential usage
```

---

# 43. Secret Rotation Architecture

```text id="w4c9m7"
Credential
    |
    v
Rotation mechanism
    |
    v
Secrets Manager
    |
    v
ESO
    |
    v
Kubernetes Secret
    |
    v
Application
```

The entire chain must be tested.

---

# 44. Rotation Without Downtime

A safe rotation strategy often requires overlap.

Example:

```text id="a8q3m1"
Old credential = valid
New credential = created
```

Then:

```text id="r7v2c5"
Application updated
       |
       v
New credential verified
       |
       v
Old credential revoked
```

The exact mechanism depends on the external system.

For databases, credential rotation must be designed around database users and connection pooling.

---

# 45. Database Credential Rotation

A practical pattern is:

```text id="c3m8q6"
DB user A = current
DB user B = next
```

Then:

```text id="w5x2p9"
Create/enable B
      ↓
Store B credentials
      ↓
Refresh application
      ↓
Verify B
      ↓
Retire A
```

This can reduce downtime during credential rotation.

---

# 46. Secrets and HPA

HPA scaling can create many Pods.

Example:

```text id="q7m4v2"
order-service
 |
 +--> Pod 1
 +--> Pod 2
 +--> Pod 3
 ...
 +--> Pod 20
```

Every Pod needs:

```text id="f9x3c8"
Database secret
```

Kubernetes Secrets avoid each Pod individually calling Secrets Manager for every request.

The synchronization layer should therefore remain reliable.

---

# 47. Secrets and Pod Startup

Suppose:

```text id="p2r7m4"
HPA
 ↓
20 new Pods
```

and:

```text id="n8c3x6"
ExternalSecret
 ↓
broken
```

New Pods may fail to start.

This is why secret availability should be part of platform observability.

---

# 48. Secret Alerts

Potential alerts:

```text id="z5m1q8"
ExternalSecret sync failures
SecretNotFound
KMS AccessDenied
IAM AccessDenied
Secret refresh failures
Unexpected secret deletion
Rotation failures
```

Alert ownership:

```text id="j3c7v9"
Platform Team
Security Team
Application Team
```

depending on the failure.

---

# 49. Secrets Manager Monitoring

Monitor:

```text id="r8x4m2"
API errors
AccessDenied
GetSecretValue failures
Rotation failures
Unexpected modifications
```

CloudTrail provides audit information for API activity.

---

# 50. KMS Monitoring

Monitor:

```text id="v6p2c8"
Decrypt failures
GenerateDataKey failures
Key state
Unauthorized access
Key policy changes
```

Security-sensitive KMS events should be correlated with:

```text id="m4q9x1"
IAM changes
Secrets Manager changes
Application failures
```

---

# 51. Cross-Region Secrets

For regional DR:

```text id="k7m3p5"
Primary Region
Secrets Manager
       |
       v
DR Region
Secrets Manager
```

The organization must decide whether secrets are:

```text id="x2r8c6"
Replicated
Recreated
Generated separately
```

Do not assume a secret automatically exists in another region.

---

# 52. DR Secret Architecture

```text id="p4v9m2"
Primary Region
ap-south-1
    |
    +--> Secrets Manager
    |
    +--> KMS
    |
    v
DR Region
ap-southeast-1
    |
    +--> Secrets Manager
    |
    +--> KMS
```

Cross-region recovery requires appropriate permissions and a deliberate key-management design.

---

# 53. KMS Keys Are Regional

This is an important DR consideration.

A KMS key in:

```text id="s6x3q8"
ap-south-1
```

should not simply be assumed to be available as the same regional key in:

```text id="h2m7v4"
ap-southeast-1
```

DR encryption architecture must explicitly account for the destination region's key resources and permissions.

---

# 54. Multi-Region KMS Considerations

AWS supports multi-Region KMS key designs for appropriate use cases.

But:

```text id="k8p3m1"
Multi-Region key
```

does not mean:

```text id="z4q7x9"
No DR design required
```

You still need:

```text id="v5c2m8"
Key policy
IAM
Region
Replication
Application configuration
Recovery testing
```

---

# 55. Secret Recovery After Region Failure

Scenario:

```text id="b6m9q3"
Primary region unavailable
```

DR sequence:

```text id="c4r7x1"
DR EKS
   ↓
ESO
   ↓
DR Secrets Manager
   ↓
DR KMS
   ↓
Kubernetes Secrets
   ↓
21 services
```

Validate:

```text id="q8m2v5"
Database credentials
Payment credentials
Messaging credentials
S3 credentials/configuration
External API credentials
```

---

# 56. KMS Recovery After Region Failure

DR KMS must support:

```text id="x3p7c9"
Secrets
RDS
S3
ECR
```

depending on the DR architecture.

Therefore our DR dependency graph becomes:

```text id="m5q8v2"
             DR KMS
                |
     +----------+----------+
     |          |          |
 Secrets       S3         ECR
     |
    RDS
```

The actual key separation should follow the organization's security architecture.

---

# 57. KMS Recovery Test

In a dedicated DR environment:

```text id="j4c9m7"
1. Create encrypted secret
2. Encrypt using KMS
3. Access from ESO
4. Disable test key
5. Observe failure
6. Re-enable key
7. Validate recovery
```

Do not perform key-disable or key-deletion exercises against production keys merely for training.

---

# 58. Secret Recovery Game Day

Scenario:

> `order-service` cannot authenticate to PostgreSQL after a secret rotation.

Exercise:

```text id="r8m3x6"
1. Generate test credential
2. Store in Secrets Manager
3. Sync through ESO
4. Rotate credential
5. Introduce controlled synchronization failure
6. Detect application failure
7. Investigate secret version
8. Repair synchronization
9. Restart/reload application
10. Validate database connection
11. Validate order creation
12. Measure RTO
```

---

# 59. KMS Incident Game Day

Scenario:

> A test KMS key becomes unavailable.

Exercise:

```text id="w2q7p4"
1. Create test secret
2. Encrypt with test KMS key
3. Disable test key
4. Observe application/ESO failure
5. Identify KMS dependency
6. Restore key availability
7. Verify Secrets Manager
8. Verify ESO
9. Verify application
```

Use a dedicated test key.

---

# 60. Complete Secrets Recovery Flow

```text id="v9m4c2"
             Application Failure
                     |
                     v
             Check Kubernetes Secret
                     |
                     v
               Check ESO
                     |
                     v
             Check Pod Identity
                     |
                     v
                Check IAM
                     |
                     v
          Check Secrets Manager
                     |
                     v
                 Check KMS
                     |
                     v
             Identify failure
                     |
          +----------+----------+
          |                     |
       Secret                 KMS/IAM
       problem                 problem
          |                     |
          v                     v
    Recover version        Repair access
    /secret state          /key state
          |                     |
          +----------+----------+
                     |
                     v
                ESO sync
                     |
                     v
             Kubernetes Secret
                     |
                     v
               Application
                     |
                     v
             Business validation
```

---

# 61. Production Recovery Checklist

## Secrets Manager

```text id="e7m3q8"
[ ] Secret exists
[ ] Correct region
[ ] Correct account
[ ] Correct ARN
[ ] Correct version
[ ] AWSCURRENT correct
[ ] Rotation state healthy
[ ] No unexpected deletion
```

## KMS

```text id="x5c9m2"
[ ] Key exists
[ ] Key enabled
[ ] Correct region
[ ] IAM permission correct
[ ] Key policy correct
[ ] No unauthorized changes
[ ] CloudTrail reviewed
```

## External Secrets

```text id="q8v3p6"
[ ] ESO running
[ ] SecretStore healthy
[ ] ExternalSecret healthy
[ ] Synchronization working
[ ] Correct secret mapping
```

## EKS Identity

```text id="m4r7x1"
[ ] ServiceAccount correct
[ ] Pod Identity association correct
[ ] IAM role correct
[ ] IAM policy correct
```

## Application

```text id="z6c2q9"
[ ] Kubernetes Secret exists
[ ] Pod receives configuration
[ ] Application starts
[ ] DB connectivity works
[ ] External API connectivity works
[ ] Messaging works
```

## DR

```text id="p3m8v5"
[ ] DR secret available
[ ] DR KMS available
[ ] DR IAM available
[ ] DR ESO configured
[ ] DR applications validated
```

---

# 62. Security Checklist

Never:

```text id="f1x7m4"
[ ] Commit production secrets
[ ] Print secret values in logs
[ ] Put secrets in Docker images
[ ] Put AWS keys in application Pods
[ ] Grant kms:* unnecessarily
[ ] Grant secretsmanager:* unnecessarily
[ ] Make KMS keys public
[ ] Make S3 public to solve an access issue
```

Always:

```text id="q9c4m7"
[ ] Use least privilege
[ ] Use Pod Identity
[ ] Use Secrets Manager
[ ] Use KMS appropriately
[ ] Audit changes
[ ] Test recovery
[ ] Rotate compromised credentials
```

---

# 63. Final Architecture

```text id="u4p8m2"
                    GitOps
                       |
                       v
                  ExternalSecret
                       |
                       v
                External Secrets
                    Operator
                       |
                       v
                  Pod Identity
                       |
                       v
                    IAM Role
                       |
                       v
              Secrets Manager
                       |
                       v
                     KMS
                       |
                       v
                Kubernetes Secret
                       |
                       v
                Spring Boot App
                       |
            +----------+----------+
            |          |          |
           RDS        Redis      SQS/Kafka
```

---

# 64. Complete DR Architecture

Our recovery architecture now looks like:

```text id="q2m7v5"
                         GitHub
                           |
                         CI/CD
                           |
              +------------+------------+
              |                         |
         Primary Region             DR Region
          ap-south-1              ap-southeast-1
              |                         |
        +-----+-----+             +-----+-----+
        |           |             |           |
       EKS         ECR           EKS         ECR
        |           |             |           |
       ESO         |             ESO         |
        |           |             |           |
   Secrets Manager |        Secrets Manager   |
        |           |             |           |
       KMS         |             KMS         |
        |           |             |           |
       RDS         S3            RDS         S3
        |           |             |           |
        +-----------+-------------+-----------+
```

The exact cross-region replication and failover mechanisms depend on the service and the organization's DR design.

---

# 65. Key Production Lessons

### Lesson 1

**Secrets are infrastructure dependencies.**

A healthy Pod does not mean a healthy secret chain.

### Lesson 2

**KMS is a dependency of encrypted services.**

```text
Secret
 ↓
KMS
```

must be included in recovery planning.

### Lesson 3

**ESO is part of the platform.**

If ESO is unavailable, secret synchronization can stop.

### Lesson 4

**Pod Identity eliminates long-lived AWS credentials.**

```text
Pod
 ↓
Pod Identity
 ↓
IAM
 ↓
AWS
```

is preferable to embedding access keys in workloads.

### Lesson 5

**Secret recovery and credential recovery are different.**

A secret can be restored while the credential itself has already been revoked.

### Lesson 6

**Never casually delete production KMS keys.**

KMS keys can protect:

```text
RDS
S3
ECR
Secrets
```

and therefore have a potentially large blast radius.

---

# 66. Recovery Dependency Matrix

| Component         | Depends On          | Recovery Mechanism       |
| ----------------- | ------------------- | ------------------------ |
| Application       | Kubernetes Secret   | ESO                      |
| Kubernetes Secret | ESO                 | GitOps                   |
| ESO               | Pod Identity        | EKS/IAM                  |
| Pod Identity      | IAM                 | Terraform                |
| Secrets Manager   | KMS                 | KMS recovery             |
| RDS credentials   | Secrets Manager     | Secret version/rotation  |
| S3 credentials    | Secrets Manager/KMS | Secret + key recovery    |
| ECR encryption    | KMS                 | Key recovery             |
| DR application    | DR secrets          | Regional secret strategy |

---

# 67. Final Recovery Sequence

```text id="x8m3q6"
                 Incident
                    |
                    v
              Identify failure
                    |
                    v
              Protect secrets
                    |
                    v
          Check Secrets Manager
                    |
                    v
                 Check KMS
                    |
                    v
              Check IAM
                    |
                    v
          Check Pod Identity
                    |
                    v
              Check ESO
                    |
                    v
        Recover secret/key state
                    |
                    v
             Sync Kubernetes
                    |
                    v
            Restart/reload app
                    |
                    v
           Validate dependencies
                    |
                    v
           Validate business flow
                    |
                    v
             Measure RTO/RPO
                    |
                    v
                   RCA
```

---

## Next Chapter — 140 DNS Failover

The next chapter will cover:

```text
140 — DNS FAILOVER

Route 53 architecture
Public vs private DNS
Health checks
Failover routing
Primary/secondary regions
ALB failover
EKS recovery integration
RDS endpoint considerations
CloudFront integration
TTL strategy
DNS propagation realities
Weighted vs failover routing
Application health checks
Regional outage
Manual vs automated failover
DNS rollback
RTO/RPO
Controlled DNS failover game day
```
