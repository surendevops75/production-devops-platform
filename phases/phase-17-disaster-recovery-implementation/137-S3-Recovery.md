# 137 — S3 RECOVERY

## 1. Objective

This chapter covers **production S3 recovery** for our 21-microservice e-commerce platform.

S3 is used for objects such as:

```text
Invoices
Product images
Documents
Reports
Exports
Application artifacts
Logs or archival data
Backup files
Data-processing outputs
```

The objective is to recover safely from:

* Accidental object deletion
* Accidental overwrite
* Application bugs modifying objects
* Incorrect lifecycle configuration
* Bucket policy mistakes
* KMS permission problems
* Regional failure
* Object corruption
* Failed replication
* Terraform configuration loss
* Security incidents

The recovery architecture must protect:

```text
Data
Integrity
Availability
Security
Version history
Encryption
RPO
RTO
```

---

# 2. Our S3 Architecture

Production architecture:

```text
                         EKS
                          |
          +---------------+---------------+
          |               |               |
     invoice-service  reporting-service  ...
          |               |
          +-------+-------+
                  |
                  v
             S3 Bucket
                  |
       +----------+----------+
       |                     |
   Versioning              KMS
       |                     |
       +----------+----------+
                  |
            Replication
                  |
                  v
        DR / Secondary Region
```

Example:

```text
Primary Region:
ap-south-1

DR Region:
ap-southeast-1
```

The exact DR region should be selected according to business, compliance, latency, and organizational requirements.

---

# 3. S3 Recovery Is Different From EKS Recovery

EKS is primarily infrastructure:

```text
Terraform
   ↓
EKS
   ↓
ArgoCD
   ↓
Applications
```

S3 contains persistent objects:

```text
Application
   ↓
S3
   ↓
Objects
```

Therefore:

> Recreating an EKS cluster does not recreate the objects stored in S3.

The S3 data lifecycle must be designed independently.

---

# 4. Example Buckets

We can separate data by purpose.

```text
company-ecommerce-prod-invoices
company-ecommerce-prod-product-images
company-ecommerce-prod-reports
company-ecommerce-prod-exports
company-ecommerce-prod-archive
```

Avoid using one giant bucket for unrelated application data when separate ownership, retention, access, or lifecycle policies are required.

---

# 5. Bucket Ownership

Example:

```text
invoice-service
      |
      +--> invoices bucket

product-service
      |
      +--> product-images bucket

reporting-service
      |
      +--> reports bucket
```

This supports least privilege.

For example:

```text
invoice-service
```

should not automatically have:

```text
s3:*
```

on every production bucket.

---

# 6. S3 Recovery Scenarios

We will handle:

```text
1. Accidental object deletion
2. Accidental overwrite
3. Wrong application update
4. Delete marker
5. Bucket policy problem
6. IAM problem
7. KMS problem
8. Lifecycle problem
9. Replication failure
10. Region failure
11. Security incident
12. Bucket configuration loss
13. Terraform drift
14. Complete DR recovery
```

---

# 7. First Recovery Rule

If an S3 incident occurs:

```text
STOP
  ↓
Determine scope
  ↓
Protect remaining data
  ↓
Identify affected objects
  ↓
Check versions
  ↓
Check replication
  ↓
Recover
  ↓
Validate
```

Do not immediately:

```text
aws s3 rm ...
```

or modify bucket configuration without understanding the incident.

---

# 8. S3 Versioning

Versioning is one of the most important recovery mechanisms.

Without versioning:

```text
Object
  |
  v
Overwrite
  |
  v
Old object may be unavailable
```

With versioning:

```text
Object
 |
 +--> Version 1
 |
 +--> Version 2
 |
 +--> Version 3
```

Example:

```text
invoice-1001.pdf
```

could have:

```text
Version A
Version B
Version C
```

This allows recovery from accidental overwrites and deletions in many scenarios.

---

# 9. Enable Versioning

Example AWS CLI:

```bash
aws s3api put-bucket-versioning \
  --bucket company-ecommerce-prod-invoices \
  --versioning-configuration Status=Enabled
```

Check:

```bash
aws s3api get-bucket-versioning \
  --bucket company-ecommerce-prod-invoices
```

Expected:

```text
Status: Enabled
```

For production, versioning should be managed through infrastructure-as-code where appropriate.

---

# 10. Terraform Example

Example:

```hcl
resource "aws_s3_bucket" "invoices" {
  bucket = "company-ecommerce-prod-invoices"

  tags = {
    Name        = "company-ecommerce-prod-invoices"
    Environment = "prod"
    Service     = "invoice-service"
  }
}
```

Versioning:

```hcl
resource "aws_s3_bucket_versioning" "invoices" {
  bucket = aws_s3_bucket.invoices.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

---

# 11. Encryption

Production objects should be encrypted.

Example:

```text
Application
    |
    v
S3
    |
    v
KMS encryption
```

Depending on requirements, use:

```text
SSE-S3
```

or:

```text
SSE-KMS
```

For controlled enterprise environments, SSE-KMS is often useful when you need explicit key policies, auditability, or separation of key administration.

---

# 12. KMS Architecture

```text
invoice-service
       |
       v
S3
       |
       v
KMS Key
```

The application may need permission to:

```text
s3:PutObject
s3:GetObject
```

and, depending on the encryption/access pattern:

```text
kms:Encrypt
kms:Decrypt
kms:GenerateDataKey
```

Permissions must be scoped to the required bucket/key.

---

# 13. Important KMS Recovery Problem

Imagine:

```text
S3 object exists
```

but:

```text
KMS key access = denied
```

The object has not necessarily been lost.

The application simply cannot decrypt it.

Therefore an S3 recovery incident may actually be:

```text
S3
 +
KMS
 +
IAM
```

rather than an object-loss incident.

---

# 14. S3 Object Structure

Use predictable prefixes.

Example:

```text
s3://company-ecommerce-prod-invoices/
    2026/
      10/
        02/
          order-10001/
            invoice.pdf
```

Product images:

```text
s3://company-ecommerce-prod-product-images/
    products/
      10001/
        main.jpg
        thumbnail.jpg
```

Reports:

```text
s3://company-ecommerce-prod-reports/
    daily/
    monthly/
```

This makes:

```text
Access control
Lifecycle policies
Recovery
Investigation
```

easier.

---

# 15. Accidental Object Deletion

Suppose:

```text
invoice-10001.pdf
```

is deleted.

First:

```bash
aws s3api head-object \
  --bucket company-ecommerce-prod-invoices \
  --key 2026/10/02/order-10001/invoice.pdf
```

If it appears missing, don't immediately conclude that the object is permanently gone.

Check versions.

---

# 16. List Object Versions

Example:

```bash
aws s3api list-object-versions \
  --bucket company-ecommerce-prod-invoices \
  --prefix 2026/10/02/order-10001/invoice.pdf
```

You may see:

```text
Versions:
  VersionId: abc123
  IsLatest: false

DeleteMarkers:
  VersionId: xyz789
  IsLatest: true
```

This is a critical recovery clue.

---

# 17. Delete Marker

With versioning enabled, a normal delete can create a delete marker.

Conceptually:

```text
Original object
     |
     v
DELETE
     |
     v
Delete marker
```

The previous object version may still exist.

Therefore:

```text
Object appears deleted
```

does not necessarily mean:

```text
Data permanently deleted
```

---

# 18. Recovering a Delete Marker

First identify:

```text
Bucket
Key
Delete marker VersionId
Previous object VersionId
```

Then remove the delete marker carefully.

Example:

```bash
aws s3api delete-object \
  --bucket company-ecommerce-prod-invoices \
  --key 2026/10/02/order-10001/invoice.pdf \
  --version-id DELETE_MARKER_VERSION_ID
```

This is a controlled recovery action.

Verify afterward:

```bash
aws s3api head-object \
  --bucket company-ecommerce-prod-invoices \
  --key 2026/10/02/order-10001/invoice.pdf
```

---

# 19. Accidental Overwrite

Suppose:

```text
invoice.pdf
```

was correct:

```text
Version 1 = correct
```

Application accidentally uploads:

```text
Version 2 = corrupted
```

Versioning gives:

```text
Version 1
Version 2
```

Identify the correct version:

```bash
aws s3api list-object-versions \
  --bucket company-ecommerce-prod-invoices \
  --prefix invoice.pdf
```

Then recover the correct version.

---

# 20. Copy a Previous Version

A common recovery pattern is to copy a known-good version back to the active object.

Conceptually:

```text
Version 1
   |
   v
Known-good object
   |
   v
Current object
```

Use a controlled `copy-object` operation with the appropriate `versionId` source specification.

After recovery:

```text
New version
```

will normally be created rather than silently destroying history.

---

# 21. Why We Should Not Permanently Delete Versions During Recovery

Suppose:

```text
Version 1 = good
Version 2 = bad
Version 3 = bad
```

If you permanently delete Version 1:

```text
Version 1
   X
```

you may eliminate the very data required for recovery.

Therefore:

> During an incident, preserve historical versions until the recovery team has confirmed the recovery point.

---

# 22. Object Lock

For data requiring stronger immutability, S3 Object Lock can be considered.

Architecture:

```text
Application
    |
    v
S3
    |
    v
Object Lock
    |
    v
Retention
```

Object Lock can support WORM-style retention.

It is particularly relevant for:

```text
Compliance records
Audit evidence
Certain backup artifacts
Regulated data
```

But it changes deletion and retention behavior, so it must be designed before production use.

---

# 23. Object Lock and Recovery

Object Lock can protect against:

```text
Accidental deletion
Malicious deletion
Ransomware-like modification attempts
```

But it can also make legitimate cleanup more complicated.

Therefore:

```text
Retention requirement
        ↓
Object Lock design
        ↓
Recovery testing
```

must be completed before enabling it for critical workloads.

---

# 24. S3 Lifecycle Rules

Lifecycle policies can automatically:

```text
Transition objects
Expire objects
Delete old versions
Delete incomplete multipart uploads
```

Example:

```text
Current objects
     |
     v
30 days
     |
     v
Archive tier
     |
     v
Retention period
     |
     v
Expiration
```

Lifecycle policies are powerful.

A bad lifecycle rule can become a data-loss incident.

---

# 25. Lifecycle Incident

Suppose someone configures:

```text
Expire objects after 1 day
```

for a production bucket.

Potential impact:

```text
Production reports disappear
```

Recovery:

```text
Stop/modify lifecycle rule
   ↓
Identify affected objects
   ↓
Check version history
   ↓
Check replication
   ↓
Recover objects
```

Do not assume changing the lifecycle rule automatically restores already-removed objects.

---

# 26. Terraform Lifecycle Protection

Bucket configuration should be version-controlled.

Example:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "invoices" {
  bucket = aws_s3_bucket.invoices.id

  rule {
    id     = "archive-old-invoices"
    status = "Enabled"

    filter {
      prefix = "archive/"
    }

    # Transition/expiration policies should be selected
    # according to actual retention requirements.
  }
}
```

Test lifecycle behavior in a non-production bucket before production rollout.

---

# 27. S3 Bucket Policy Failure

Suppose:

```text
invoice-service
```

starts receiving:

```text
AccessDenied
```

Check:

```text
IAM Role
Bucket Policy
KMS Policy
S3 Block Public Access
VPC endpoint policy
SCP
Permissions boundary
```

Do not immediately make the bucket public.

---

# 28. Least-Privilege S3 IAM

Example application permissions:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::company-ecommerce-prod-invoices/*"
}
```

The exact permissions depend on the application.

Avoid:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

for normal application workloads.

---

# 29. EKS Pod Identity

Our application architecture:

```text
invoice-service Pod
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
S3
```

No AWS access keys inside:

```text
Docker image
Kubernetes Secret
Git
Helm values
application.properties
```

---

# 30. S3 Endpoint Architecture

For private EKS workloads, S3 access can use an S3 VPC Gateway Endpoint.

Architecture:

```text
EKS Pod
   |
   v
VPC
   |
   v
S3 Gateway Endpoint
   |
   v
S3
```

This can avoid routing S3 traffic through NAT for supported access patterns and can improve network architecture and cost characteristics.

---

# 31. S3 VPC Endpoint Recovery

If applications suddenly cannot access S3:

```text
Check:
```

```text
VPC endpoint
Route tables
Endpoint policy
IAM
Bucket policy
DNS
KMS
```

Example:

```bash
aws ec2 describe-vpc-endpoints \
  --region ap-south-1
```

---

# 32. Application-Level S3 Failure

Suppose:

```text
invoice-service
```

returns:

```text
500 Internal Server Error
```

Trace:

```text
Client
 ↓
ALB
 ↓
invoice-service
 ↓
S3
```

Logs might show:

```text
AccessDenied
```

The S3 object may be completely healthy.

Therefore:

```text
Application failure
≠
Data loss
```

Always identify the failure layer first.

---

# 33. S3 Replication

For regional disaster recovery:

```text
Primary Region
ap-south-1
     |
     | Replication
     v
DR Region
ap-southeast-1
```

S3 Replication can replicate objects between buckets according to configured rules.

Possible strategy:

```text
Production bucket
      |
      v
DR bucket
```

Replication requirements depend on:

```text
Versioning
IAM
Replication role
KMS permissions
Replication configuration
Object ownership
```

---

# 34. Replication Is Not Automatically Backup

Important distinction:

```text
Replication
=
Copy data elsewhere
```

It does not automatically protect against every logical error.

For example:

```text
Bad application writes corrupted object
        |
        v
Replication
        |
        v
DR bucket receives corrupted object
```

Therefore use:

```text
Versioning
+
Replication
+
Appropriate retention/immutability
```

according to the recovery requirements.

---

# 35. Cross-Region Recovery

Architecture:

```text
                   Route 53
                      |
            +---------+---------+
            |                   |
       Primary Region       DR Region
        ap-south-1        ap-southeast-1
            |                   |
          EKS                 EKS
            |                   |
           S3  -------------> S3
```

For S3:

```text
Primary bucket
     |
     v
Replication
     |
     v
DR bucket
```

---

# 36. RPO for S3

Suppose the business requirement is:

```text
RPO = 15 minutes
```

That means the organization must understand how much recently written S3 data could potentially be unavailable after a disaster.

Replication characteristics and object-write patterns must be tested rather than assuming an exact recovery point.

---

# 37. RTO for S3

RTO includes more than:

```text
S3 bucket exists
```

You need:

```text
Bucket
+
Objects
+
KMS
+
IAM
+
Application configuration
+
Network
+
DNS
```

Example:

```text
S3 recovery target = 30 minutes
```

Measure the entire process.

---

# 38. Region Failure Scenario

Suppose:

```text
ap-south-1
```

has a major regional outage.

Primary:

```text
EKS
RDS
S3
```

becomes unavailable.

DR:

```text
ap-southeast-1
```

must provide:

```text
EKS
Application
Data
S3
Secrets
KMS
DNS
```

S3 is only one component of regional DR.

---

# 39. S3 DR Architecture

```text
                 Route 53
                    |
           +--------+--------+
           |                 |
       Primary             DR
    ap-south-1         ap-southeast-1
           |                 |
          EKS               EKS
           |                 |
          S3  ----------->  S3
           |                 |
          RDS              RDS
           |                 |
        Messaging        Messaging
```

S3 recovery must therefore integrate with the broader regional recovery plan.

---

# 40. Bucket Configuration Recovery

Objects are not the only thing that matters.

You must recover:

```text
Bucket
Versioning
Encryption
KMS
Lifecycle
Replication
Bucket policy
CORS
Notifications
Access logging where used
Event configuration
Tags
```

Terraform should manage most of these configuration elements where appropriate.

---

# 41. Terraform as S3 Configuration Source of Truth

Example repository:

```text
infrastructure/
└── terraform/
    └── modules/
        └── s3/
            ├── main.tf
            ├── variables.tf
            ├── outputs.tf
            └── lifecycle.tf
```

Environment:

```text
terraform/
└── environments/
    └── prod/
        └── s3.tf
```

---

# 42. Terraform State vs S3 Data

Very important:

```text
Terraform state
     ≠
S3 application objects
```

Terraform state contains infrastructure configuration/state.

S3 bucket contains:

```text
PDF
Images
Reports
Exports
```

Backing up Terraform state does not back up your application objects.

---

# 43. Accidental Bucket Deletion

Suppose:

```text
company-ecommerce-prod-invoices
```

is accidentally deleted.

Recovery depends on what still exists:

```text
Objects?
Versions?
Replication?
Backups?
Terraform configuration?
```

A deleted bucket and deleted objects are not automatically reconstructed simply by rerunning Terraform.

Terraform can recreate the infrastructure definition, but it cannot magically recreate lost application data.

---

# 44. Bucket Deletion Protection

For critical production buckets, organizational controls should reduce the possibility of accidental deletion.

Examples:

```text
Restricted IAM
SCP guardrails
Terraform review
Separate production access
Change management
Object Lock where appropriate
Versioning
```

Do not rely solely on:

```text
Engineer knows not to delete it
```

---

# 45. Security Incident

Suppose an attacker or compromised workload gains S3 access.

Symptoms:

```text
Unexpected object deletions
Unexpected writes
Unexpected downloads
Unexpected IAM activity
```

Immediate priorities:

```text
1. Contain compromised identity
2. Preserve evidence
3. Restrict access
4. Determine affected objects
5. Check CloudTrail
6. Check S3 access logs/data events where configured
7. Recover objects
8. Rotate/replace credentials where required
9. Validate application
```

Do not destroy evidence during containment.

---

# 46. CloudTrail for S3

For sensitive buckets, configure appropriate CloudTrail data events so object-level activity can be investigated.

Questions:

```text
Who accessed the object?
Who deleted it?
When?
From which identity?
What API action occurred?
```

This is particularly important for security investigations.

---

# 47. S3 Recovery Investigation

Example:

```text
invoice-10001.pdf missing
```

Investigation:

```text
1. Does object exist?
2. Is versioning enabled?
3. Is there a delete marker?
4. What versions exist?
5. Was object replicated?
6. Who deleted it?
7. Was lifecycle involved?
8. Was the bucket policy changed?
9. Was IAM changed?
10. Was KMS involved?
```

---

# 48. S3 Access Denied Investigation

Use:

```text
IAM Policy Simulator
CloudTrail
S3 bucket policy
KMS policy
VPC endpoint policy
SCP
Permissions boundary
```

Architecture:

```text
Pod
 |
 v
IAM
 |
 +--> SCP
 |
 +--> Permissions Boundary
 |
 +--> IAM Policy
 |
 +--> Bucket Policy
 |
 +--> KMS Policy
 |
 v
S3
```

Any layer can deny access.

---

# 49. Recovering a Product Image

Example:

```text
product-service
      |
      v
S3
      |
      v
products/10001/main.jpg
```

Suppose the image was overwritten.

Recovery:

```text
List versions
     ↓
Identify known-good version
     ↓
Copy known-good version
     ↓
Validate image
     ↓
Validate application URL
```

---

# 50. Recovering an Invoice

Example:

```text
invoice-service
      |
      v
S3
      |
      v
orders/10001/invoice.pdf
```

Validate:

```text
Object exists
Content type correct
Size expected
Checksum where appropriate
Encryption correct
Application can retrieve it
Customer-facing URL works
```

---

# 51. S3 Integrity Validation

For important objects, application-level integrity mechanisms can be useful.

Possible validation:

```text
Object size
Checksum
Content type
Metadata
Application record
```

Example database record:

```text
invoice_id
s3_key
content_type
checksum
created_at
```

Then:

```text
Database says:
invoice-10001.pdf
checksum = X

S3 object:
checksum = X
```

This helps detect unexpected replacement/corruption.

---

# 52. S3 and Database Consistency

Consider:

```text
Order DB
    |
    +--> invoice_s3_key
```

If the DB record exists but the S3 object is missing:

```text
Database = healthy
S3 = inconsistent
```

Our recovery validation must identify these mismatches.

Example:

```text
Order 1001
Invoice record exists
Invoice object missing
```

This becomes an application-level recovery issue.

---

# 53. Event-Driven S3 Architecture

Some S3 workflows may generate events:

```text
S3
 |
 v
EventBridge / SNS / SQS
 |
 v
Microservice
```

After recovery, verify event processing.

For example:

```text
Invoice uploaded
      |
      v
S3 event
      |
      v
SQS
      |
      v
notification-service
```

The object may be recovered successfully while the downstream event remains unprocessed.

---

# 54. S3 Recovery and Idempotency

Suppose an object event is delivered twice:

```text
invoice-created
invoice-created
```

Consumer must not generate:

```text
two invoices
two notifications
two reports
```

Use:

```text
eventId
+
idempotency
+
processed-event tracking
```

This connects S3 recovery with our messaging architecture.

---

# 55. DR Application Configuration

In the DR region:

```text
invoice-service
```

must know:

```text
DR bucket
DR KMS key
DR region
```

Configuration should come from controlled environment configuration rather than hard-coded application code.

Example:

```text
S3_BUCKET
S3_REGION
KMS_KEY_ID
```

---

# 56. EKS Pod Identity During DR

The DR cluster needs its own valid workload identity configuration.

```text
DR Pod
 |
 v
DR ServiceAccount
 |
 v
EKS Pod Identity
 |
 v
IAM Role
 |
 v
DR S3
```

Do not assume that recreating EKS automatically recreates every identity association unless those associations are managed by Terraform or another declarative system.

---

# 57. S3 Recovery After EKS Rebuild

Complete sequence:

```text
EKS recreated
     ↓
Nodes ready
     ↓
ArgoCD
     ↓
ServiceAccounts
     ↓
Pod Identity
     ↓
Applications
     ↓
S3 configuration
     ↓
S3 access
     ↓
Business validation
```

---

# 58. Example S3 Test From EKS

Use a controlled diagnostic Pod or application test.

Conceptually:

```bash
aws s3api head-object \
  --bucket company-ecommerce-prod-invoices \
  --key test/recovery-test.txt
```

Then:

```bash
aws s3 cp \
  s3://company-ecommerce-prod-invoices/test/recovery-test.txt \
  /tmp/recovery-test.txt
```

The test identity must have only the permissions required for the exercise.

---

# 59. Do Not Test Recovery With Production Data

For a DR game day:

```text
DR test bucket
```

is preferable.

Example:

```text
company-ecommerce-dr-test-invoices
```

Use synthetic:

```text
DR-ORDER-001
DR-INVOICE-001
```

Then:

```text
Upload
Delete
Recover
Validate
```

---

# 60. S3 Recovery Game Day

Scenario:

> An application accidentally overwrites 1,000 production invoice objects.

Simulation:

```text
1. Generate synthetic invoice objects
2. Enable versioning
3. Record known-good versions
4. Perform controlled overwrite
5. Detect incident
6. Identify affected objects
7. Recover versions
8. Validate checksums
9. Validate application
10. Measure recovery time
```

---

# 61. Second Game Day — Delete Marker

Scenario:

```text
Invoice object deleted
```

Exercise:

```text
1. Create object
2. Delete object
3. Identify delete marker
4. Locate previous version
5. Remove delete marker
6. Verify object
7. Validate application
```

---

# 62. Third Game Day — Region Recovery

Scenario:

```text
Primary S3 region unavailable
```

Exercise:

```text
1. Stop primary application traffic
2. Activate DR application
3. Use DR S3 bucket
4. Validate replicated objects
5. Validate KMS
6. Validate IAM
7. Validate application
8. Perform invoice retrieval
9. Measure RTO/RPO
```

This exercise should be integrated with the broader region-failure game day later.

---

# 63. S3 Monitoring

Monitor:

```text
Object operations
Errors
AccessDenied
4xx
5xx
Replication health
Storage growth
Lifecycle activity
Unexpected access
```

Application metrics:

```text
S3 request latency
S3 error rate
Upload failures
Download failures
Missing object errors
```

---

# 64. S3 Alert Examples

Potential alerts:

```text
S3 AccessDenied spike
S3 application error spike
Replication failure
Unexpected storage growth
Unexpected object deletion activity
KMS AccessDenied spike
```

Avoid alerting on every normal S3 request.

Focus on:

```text
Customer impact
Security impact
Data integrity
Replication health
```

---

# 65. S3 Recovery Runbook

```text
S3 INCIDENT
     |
     v
Identify affected bucket/object
     |
     v
Check object existence
     |
     v
Check versioning
     |
     +----> Version exists
     |          |
     |          v
     |      Recover version
     |
     +----> No version
                |
                v
          Check replication/
          backup/other source
                |
                v
          Determine data loss
```

Then:

```text
Check IAM
Check KMS
Check bucket policy
Check lifecycle
Check CloudTrail
Check replication
Validate recovered object
Validate application
```

---

# 66. Complete S3 Recovery Architecture

```text
                         Application
                              |
                       EKS Pod Identity
                              |
                              v
                         IAM Role
                              |
                              v
                         S3 Bucket
                              |
          +-------------------+-------------------+
          |                   |                   |
      Versioning           KMS              Lifecycle
          |                   |                   |
          +-------------------+-------------------+
                              |
                         Replication
                              |
                              v
                         DR Bucket
                              |
                              v
                       DR Application
```

---

# 67. Production S3 Recovery Checklist

## Bucket

```text
[ ] Bucket exists
[ ] Correct region
[ ] Versioning enabled
[ ] Encryption enabled
[ ] KMS configured
[ ] Block Public Access enabled
[ ] Bucket policy validated
[ ] Lifecycle reviewed
```

## Data

```text
[ ] Object exists
[ ] Correct version identified
[ ] Delete marker checked
[ ] Object integrity validated
[ ] Critical objects recovered
```

## Security

```text
[ ] IAM validated
[ ] Pod Identity validated
[ ] KMS permissions validated
[ ] Bucket policy validated
[ ] CloudTrail investigated
[ ] No public access introduced
```

## DR

```text
[ ] Replication configured
[ ] DR bucket available
[ ] DR KMS available
[ ] DR IAM available
[ ] DR application configuration available
```

## Application

```text
[ ] invoice-service works
[ ] product-service works
[ ] reporting-service works
[ ] S3 uploads work
[ ] S3 downloads work
[ ] S3-generated events work
```

## Recovery

```text
[ ] RPO measured
[ ] RTO measured
[ ] Business validation completed
[ ] Incident timeline documented
[ ] RCA completed
```

---

# 68. Key Production Lessons

### Lesson 1 — Versioning is fundamental

```text
S3 Versioning
    ↓
Historical object versions
    ↓
Recovery capability
```

### Lesson 2 — Replication is not the same as backup

A bad object can be replicated.

Therefore:

```text
Versioning
+
Replication
+
Appropriate retention/immutability
```

should be designed together.

### Lesson 3 — S3 recovery is more than restoring objects

You also need:

```text
IAM
KMS
Bucket policies
Lifecycle
Network
Application configuration
```

### Lesson 4 — Terraform does not recover application data

Terraform can recreate:

```text
Bucket
Policy
Versioning
Encryption
Replication
```

It does not automatically recreate deleted PDFs, images, or reports.

### Lesson 5 — Validate the business

The final test is not:

```text
Bucket exists
```

It is:

```text
Customer
   ↓
Application
   ↓
S3
   ↓
Object
   ↓
Business transaction succeeds
```

---

# 69. Final S3 Recovery Flow

```text
                S3 Incident
                     |
                     v
                Identify scope
                     |
                     v
             Protect remaining data
                     |
                     v
              Check versioning
                     |
          +----------+----------+
          |                     |
     Version exists        No version
          |                     |
          v                     v
     Recover version       Check replication/
                           backup/DR source
          |                     |
          +----------+----------+
                     |
                     v
              Validate KMS/IAM
                     |
                     v
             Validate application
                     |
                     v
             Validate business data
                     |
                     v
              Measure RTO/RPO
                     |
                     v
                 RCA
```

---

# 70. Disaster Recovery Principle

Our complete persistent-data recovery model is now:

```text
                 DATA RECOVERY
                      |
       +--------------+--------------+
       |              |              |
      RDS             S3           Messaging
       |              |              |
     PITR         Versioning       Retention
    Snapshot      Replication      Replay
   Multi-AZ       KMS             Offsets
       |              |              |
       +--------------+--------------+
                      |
                      v
                 EKS Platform
                      |
                      v
                  ArgoCD
                      |
                      v
                21 Services
                      |
                      v
             Business Validation
```

The important architectural idea is:

> **EKS can be rebuilt, but persistent state must have its own independently tested recovery strategy.**

---

## Next Chapter — 138 ECR Recovery

The next chapter will cover:

```text
138 — ECR RECOVERY

ECR repository deletion
Image recovery
Immutable tags
Image digests
Cross-region ECR replication
ECR lifecycle policies
ECR vulnerability scanning
KMS encryption
EKS ImagePullBackOff after ECR failure
ECR IAM recovery
Cross-account ECR access
Recovering images after cluster loss
CI/CD rebuild strategy
ECR DR architecture
RTO/RPO
Controlled ECR recovery game day
```
