# 160 — Security Incident Production Incident

> **Phase 18 — Incident Management**  
> **Incident #160**  
> **Environment:** Production  
> **Platform:** Amazon EKS + AWS + ArgoCD + GitHub Actions  
> **Example affected service:** `order-service`  
> **Severity example:** SEV-1

---

# 1. Objective

This chapter explains how to respond to a **production security incident** in our 21-microservice ecommerce platform.

A security incident can involve:

- compromised credentials
- leaked AWS credentials
- compromised Kubernetes ServiceAccount
- excessive IAM permissions
- malicious container/image
- vulnerable dependency actively exploited
- exposed Kubernetes service
- compromised application
- suspicious API activity
- secret leakage
- unauthorized production changes
- container escape
- data access anomaly
- suspicious ECR image
- compromised CI/CD pipeline
- ransomware/destructive activity

The goal is:

```text
Detect
  ↓
Contain
  ↓
Preserve Evidence
  ↓
Investigate
  ↓
Eradicate
  ↓
Recover
  ↓
Validate
  ↓
RCA
  ↓
Prevent Recurrence
```

---

# 2. Critical Security Principle

During a security incident:

> **Do not destroy evidence while trying to fix the problem.**

For example, don't immediately:

```bash
kubectl delete pod
```

if that Pod is suspected to be compromised, unless containment requires it.

Instead:

```text
Alert
 ↓
Preserve evidence
 ↓
Capture relevant metadata/logs
 ↓
Contain
 ↓
Investigate
```

The exact evidence-preservation process should follow the organization's security/forensics procedures.

---

# 3. Production Security Architecture

Our security architecture:

```text
                         Internet
                            │
                            ▼
                     CloudFront / WAF
                            │
                            ▼
                           ALB
                            │
                            ▼
                         AWS EKS
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       Pods              Services          Ingress
          │
    ┌─────┼──────────────────────────────┐
    │     │              │               │
    ▼     ▼              ▼               ▼
 IAM    Secrets       NetworkPolicy    Runtime
Pod     Manager          │             Security
Identity     │           │
    │        ▼           ▼
    │        ESO       Kubernetes
    │                    │
    ▼                    ▼
 AWS APIs             Databases
```

Security controls exist at multiple layers.

---

# 4. Security Layers

## Layer 1 — Identity

```text
IAM
EKS Pod Identity
RBAC
GitHub OIDC
```

## Layer 2 — Network

```text
VPC
Security Groups
NetworkPolicy
Private Subnets
WAF
```

## Layer 3 — Application

```text
Authentication
Authorization
Input Validation
Rate Limiting
Secure Coding
```

## Layer 4 — Supply Chain

```text
GitHub
CI
SonarQube
Dependency Scan
Trivy
ECR
Image Signing/Provenance
```

## Layer 5 — Runtime

```text
Pod Security
Non-root containers
Read-only filesystem where practical
Resource limits
NetworkPolicy
Runtime monitoring
```

## Layer 6 — Data

```text
KMS
Secrets Manager
Encryption
RDS security
S3 security
Audit logs
```

---

# 5. Example Security Incident

Suppose an attacker obtains credentials associated with:

```text
order-service
```

The attacker attempts to access:

```text
AWS Secrets Manager
S3
SQS
```

The expected application role should only have:

```text
SQS:
SendMessage

Secrets Manager:
GetSecretValue
specific secret

S3:
specific bucket/prefix
```

Instead, due to an IAM policy mistake, the role has:

```text
secretsmanager:*
s3:*
sqs:*
```

This creates a major blast-radius problem.

---

# 6. Detection

Possible detection sources:

```text
AWS GuardDuty
AWS CloudTrail
AWS Security Hub
AWS WAF
EKS audit logs
Application logs
ELK
Prometheus
Falco/runtime security
Trivy
GitHub security alerts
ECR scanning
```

The exact security tooling depends on the organization's platform.

---

# 7. Example Alert

Security monitoring detects:

```text
Unusual Secrets Manager API activity
```

Example:

```text
order-service IAM role
```

accesses:

```text
payment-production-secret
```

which it normally should never access.

This is suspicious.

---

# 8. First Incident Questions

Immediately determine:

```text
What happened?
When?
Which identity?
Which resource?
From where?
Which account?
Which region?
What permissions were used?
Was data accessed?
Is the attacker still active?
```

---

# 9. Incident Severity

Example:

### SEV-1

Confirmed compromise or significant unauthorized production/data access.

### SEV-2

Credible security incident with limited scope.

### SEV-3

Security weakness or suspicious activity without confirmed compromise.

### SEV-4

Low-risk security finding.

Actual severity classification should follow the organization's security policy.

---

# 10. Security Incident Roles

Security incidents should not be handled by one engineer alone.

Typical roles:

```text
Incident Commander
Security Team
DevOps/Platform
Application Team
Cloud/IAM Owner
Database Team
SRE
Legal/Compliance
Management/Communications
```

Depending on the incident.

---

# 11. Incident Commander

The Incident Commander coordinates:

```text
Scope
Priorities
Communication
Actions
Approvals
Timeline
Escalation
```

The IC should prevent:

```text
10 engineers
+
10 simultaneous changes
=
larger outage
```

---

# 12. First Priority — Containment

Security response priorities:

```text
1. Protect customers
2. Stop attacker access
3. Preserve evidence
4. Prevent lateral movement
5. Protect data
6. Recover safely
```

---

# 13. Example Containment

If a compromised IAM role is confirmed:

```text
Compromised Role
       ↓
Restrict permissions
       ↓
Revoke/replace credentials where applicable
       ↓
Block further access
       ↓
Investigate usage
```

Do not simply delete everything before understanding the scope.

---

# 14. IAM Role Investigation

Identify the role:

```text
arn:aws:iam::123456789012:role/order-service-role
```

Check its policies.

Conceptually:

```bash
aws iam list-attached-role-policies \
  --role-name order-service-role
```

And inline policies:

```bash
aws iam list-role-policies \
  --role-name order-service-role
```

Do not expose sensitive credentials in incident notes.

---

# 15. Check CloudTrail

CloudTrail can help answer:

```text
Who called AWS?
What API?
When?
From which source?
Which resource?
Was it successful?
```

Examples:

```text
GetSecretValue
PutObject
GetObject
SendMessage
DeleteObject
CreateAccessKey
AssumeRole
```

Look for:

```text
unexpected API
unexpected principal
unexpected source
unexpected time
unexpected resource
```

---

# 16. Example Suspicious Activity

Normal:

```text
order-service-role
→ GetSecretValue
→ order-db-secret
```

Suspicious:

```text
order-service-role
→ GetSecretValue
→ payment-provider-secret
```

Even if the API call succeeds, this should trigger investigation.

---

# 17. Do Not Assume Compromise From One Event

A single unusual API call does not always prove an attacker.

Possible causes:

- deployment
- automation
- backup
- security scanner
- administrator
- Terraform
- rotation
- troubleshooting

Therefore correlate:

```text
CloudTrail
+
Git
+
CI/CD
+
Kubernetes audit
+
Application logs
+
Security tooling
```

---

# 18. EKS Investigation

Check:

```bash
kubectl get pods -A -o wide
```

Identify:

```text
Unknown Pods
Unexpected Pods
Unexpected images
Unexpected namespaces
Unexpected ServiceAccounts
```

---

# 19. Check Recent Kubernetes Changes

```bash
kubectl get events \
  -A \
  --sort-by=.lastTimestamp
```

Look for:

```text
Created
Updated
Scaled
Deleted
```

For sensitive resources:

```bash
kubectl get serviceaccounts -A
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterrolebindings
```

---

# 20. Suspicious ServiceAccount

Example:

```text
order-service
```

normally uses:

```text
order-service-sa
```

But a suspicious Pod uses:

```text
admin-sa
```

This is a high-risk finding.

Investigate:

```bash
kubectl get pod <pod> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

---

# 21. Check Pod Identity

For AWS access, determine:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
AWS API
```

Identify:

```text
Which Pod?
Which ServiceAccount?
Which IAM role?
Which AWS APIs?
```

The role should follow least privilege.

---

# 22. Least Privilege Example

Bad:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Better:

```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage"
  ],
  "Resource": "arn:aws:sqs:ap-south-1:123456789012:order-events"
}
```

The exact ARN and permissions should match the application's real requirements.

---

# 23. NetworkPolicy During Security Incident

Suppose:

```text
order-service
```

is compromised.

Without NetworkPolicy:

```text
Compromised Pod
   ↓
Potentially broad cluster connectivity
```

With restrictive NetworkPolicy:

```text
Compromised Pod
   ↓
Only approved destinations
```

For example:

```text
order-service
 ├── PostgreSQL
 ├── Redis
 ├── payment-service
 └── Kafka
```

and nothing else.

---

# 24. Network Isolation

During containment, security/platform teams may isolate a workload.

Possible mechanisms:

```text
NetworkPolicy
Security Group
WAF rule
Ingress restriction
IAM restriction
ServiceAccount restriction
```

The safest mechanism depends on the incident.

---

# 25. Compromised Container

Suppose a Pod runs:

```text
order-service:1.9.0
```

and runtime security detects suspicious behavior.

Do not immediately assume the entire EKS cluster is compromised.

Determine:

```text
Container
Pod
Node
ServiceAccount
IAM role
Namespace
Image
Registry
Cluster
```

scope systematically.

---

# 26. Image Investigation

Check ECR:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageDigest=sha256:...
```

Verify:

```text
Digest
Tags
Push time
Scan findings
Image provenance
CI build
Git commit
```

---

# 27. Build Provenance

Trace:

```text
Git Commit
     ↓
GitHub Actions
     ↓
Docker Build
     ↓
ECR
     ↓
Image Digest
     ↓
GitOps
     ↓
ArgoCD
     ↓
EKS
```

If the image was not built by the expected CI pipeline, treat that as suspicious.

---

# 28. ECR Image Replacement

Never simply overwrite a production image tag.

If compromise is confirmed:

```text
Bad digest
    ↓
Quarantine
    ↓
Stop deployment
    ↓
Build trusted version
    ↓
Security scan
    ↓
Push new immutable digest
    ↓
GitOps update
    ↓
Deploy
```

Use immutable tags/digests.

---

# 29. Secret Compromise

Suppose:

```text
payment-provider-secret
```

may have been exposed.

Do not only update the Kubernetes Secret.

You need:

```text
Revoke old credential
      ↓
Generate new credential
      ↓
Store in Secrets Manager
      ↓
ESO sync
      ↓
Restart/reload application
      ↓
Validate
```

The old credential must no longer be usable.

---

# 30. Secret Rotation

Architecture:

```text
Security Incident
       ↓
Revoke old credential
       ↓
Create new credential
       ↓
AWS Secrets Manager
       ↓
External Secrets Operator
       ↓
Kubernetes Secret
       ↓
Application
```

Monitor:

```text
ESO sync
Secret version
Application restart
Authentication success
Authentication failure
```

---

# 31. KMS Considerations

If encryption keys are involved:

```text
Secret
 ↓
KMS
```

or:

```text
S3
 ↓
SSE-KMS
 ↓
KMS
```

Check:

```text
Key policy
IAM permissions
Key state
CloudTrail
Decrypt events
```

Do not disable a production KMS key blindly.

That can create a major outage.

---

# 32. Database Security Incident

If database access is suspected:

```text
Application
    ↓
RDS
```

Check:

- IAM where applicable
- database credentials
- PostgreSQL users
- active sessions
- database audit/logging
- unusual queries
- source addresses
- access patterns

The DBA/database security team should lead database forensic investigation where appropriate.

---

# 33. S3 Security Incident

If S3 access is suspected:

Check:

```text
Bucket policy
IAM
CloudTrail
Object access logs/data events if configured
KMS
Versioning
Object Lock
Replication
```

Important:

```text
S3 Versioning
```

can help recover overwritten/deleted objects.

Do not delete suspicious objects before evidence requirements are understood.

---

# 34. RDS Security Incident

If unauthorized database access is suspected:

```text
1. Identify account
2. Identify user
3. Identify source
4. Identify time
5. Identify queries
6. Identify affected tables
7. Identify credentials
8. Contain access
9. Rotate credentials
10. Preserve evidence
```

Do not immediately reboot RDS unless there is a clear operational/security reason and the incident commander approves.

---

# 35. CI/CD Security Incident

The GitHub Actions pipeline is part of the production supply chain.

Potential attack:

```text
GitHub account compromised
        ↓
Workflow modified
        ↓
Malicious code builds
        ↓
Docker image
        ↓
ECR
        ↓
Production
```

Therefore inspect:

```text
Git commits
Pull requests
Workflow files
GitHub Actions runs
OIDC role assumption
ECR pushes
GitOps changes
```

---

# 36. GitHub OIDC Security

Architecture:

```text
GitHub
   ↓
OIDC token
   ↓
AWS STS
   ↓
IAM role
   ↓
ECR
```

Trust policy should restrict:

```text
Repository
Branch/tag/environment
Audience
```

Avoid:

```text
Any GitHub repository
```

being allowed to assume the production role.

---

# 37. Malicious Workflow Example

Suppose:

```yaml
permissions:
  id-token: write
```

and a compromised workflow can assume:

```text
production-deployment-role
```

The attacker could potentially access production resources.

Therefore:

```text
GitHub OIDC
+
least privilege
+
branch/environment restrictions
+
protected environments
```

are critical.

---

# 38. ArgoCD Security Incident

Suppose an attacker modifies:

```text
gitops/prod/order-service/values.yaml
```

and changes:

```text
image:
  digest: malicious
```

ArgoCD sees:

```text
Git changed
```

and deploys it.

This demonstrates:

> **GitOps does not automatically mean secure.**

Git itself becomes part of the production security boundary.

---

# 39. GitOps Security Controls

Use:

```text
Protected branches
CODEOWNERS
Required reviews
Signed commits where appropriate
GitHub environment protection
Least privilege
Audit logs
Branch restrictions
Secret scanning
Dependency scanning
```

Production GitOps changes should require appropriate approvals.

---

# 40. Kubernetes RBAC Security

Avoid:

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

for application ServiceAccounts.

Instead:

```text
order-service
 ↓
Only required Kubernetes permissions
```

Many application Pods may need:

```text
No Kubernetes API access at all.
```

That is even better.

---

# 41. Container Security

Production containers should preferably:

```text
Run as non-root
Use minimal base images
Avoid unnecessary packages
Use read-only filesystem where practical
Drop Linux capabilities
Use seccomp
Use resource limits
Use NetworkPolicy
Use immutable images
```

Example:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

The exact configuration must be compatible with the application.

---

# 42. Trivy's Role

Trivy can detect:

```text
OS vulnerabilities
Dependency vulnerabilities
Misconfigurations
Secrets
Container image findings
```

But:

> Trivy is not a complete runtime security system.

It should be combined with:

```text
CI security
ECR scanning
Runtime detection
IAM
NetworkPolicy
CloudTrail
GuardDuty
```

---

# 43. Runtime Detection

A runtime security tool may detect suspicious behavior such as:

```text
Unexpected shell
Unexpected binary
Privilege escalation
Suspicious process
Unexpected network connection
```

For Kubernetes, tools such as Falco or equivalent runtime-security platforms may be used.

The exact production tool depends on organizational architecture.

---

# 44. Security Incident Containment Matrix

| Problem | Possible Containment |
|---|---|
| Compromised Pod | Isolate workload |
| Compromised IAM role | Restrict/revoke permissions |
| Leaked secret | Revoke + rotate |
| Malicious image | Quarantine digest |
| Compromised GitHub workflow | Disable/restrict workflow |
| Malicious GitOps commit | Revert Git change |
| Exposed endpoint | WAF/network restriction |
| Suspicious database access | Revoke credential/session |
| S3 exposure | Restrict policy/access |
| Compromised node | Isolate/replace node after evidence preservation |

---

# 45. Do Not Destroy Evidence

Potential evidence:

```text
CloudTrail
EKS audit logs
Application logs
ELK
WAF logs
ALB logs
Git history
GitHub Actions logs
ECR metadata
Pod metadata
Security alerts
Network telemetry
RDS logs
S3 access evidence
```

Retention requirements should be defined by security/compliance policy.

---

# 46. Evidence Timeline

Create:

```text
security-incident-timeline.md
```

Example:

```text
10:00 Normal
10:05 Suspicious API request
10:07 IAM anomaly detected
10:09 Security alert
10:11 Incident declared
10:13 Role access restricted
10:15 Secret rotation started
10:20 Compromised Pod isolated
10:25 Image investigated
10:35 No additional compromised workloads identified
10:45 Recovery
11:00 Validation
```

---

# 47. Security Incident Decision Tree

```text
                Security Alert
                     │
                     ▼
             Is compromise confirmed?
                /           \
              No             Yes
              │               │
       Investigate        Declare Incident
                              │
                              ▼
                         Preserve Evidence
                              │
                              ▼
                           Contain
                              │
                ┌─────────────┼─────────────┐
                │             │             │
               IAM          Pod/Image      Git
                │             │             │
                ▼             ▼             ▼
             Restrict      Isolate       Revert
                │             │             │
                └─────────────┼─────────────┘
                              ▼
                         Investigate
                              │
                              ▼
                         Eradicate
                              │
                              ▼
                          Recover
                              │
                              ▼
                          Validate
                              │
                              ▼
                            RCA
```

---

# 48. Example Security Incident

Suppose GuardDuty/security monitoring detects unusual AWS API activity from:

```text
order-service-role
```

The role accesses:

```text
payment-production-secret
```

The application should only access:

```text
order-db-secret
```

Security declares:

```text
SEV-1
```

---

# 49. Investigation

CloudTrail shows:

```text
Principal:
order-service-role

API:
secretsmanager:GetSecretValue

Resource:
payment-production-secret

Time:
10:07 IST
```

Kubernetes investigation identifies:

```text
order-service
Pod:
order-service-7fd...
```

ServiceAccount:

```text
order-service-sa
```

Pod Identity:

```text
order-service-role
```

---

# 50. Scope Analysis

Investigate:

```text
Was payment secret successfully read?
Was secret value exposed?
Was it used?
Were other Secrets accessed?
Were S3 objects accessed?
Were SQS messages accessed?
Were other IAM roles assumed?
Was the container modified?
Was the image compromised?
```

Do not assume the scope from one event.

---

# 51. Immediate Containment

Security team decides to:

```text
1. Restrict compromised IAM role
2. Stop suspicious workload
3. Rotate potentially exposed secrets
4. Block unnecessary network access
5. Preserve evidence
```

The order-service workload is moved to a trusted image after evidence requirements are satisfied.

---

# 52. Recovery

Trusted image:

```text
order-service:1.8.1
```

is deployed.

Secrets are rotated.

IAM policy is corrected:

```text
order-service
→ only order-db-secret
```

ArgoCD reconciles the trusted GitOps state.

---

# 53. Business Validation

Validate:

```text
Login
 ↓
Product
 ↓
Cart
 ↓
Order
 ↓
Payment
 ↓
Inventory
 ↓
Notification
```

Also validate:

```text
No unauthorized credentials remain active
No suspicious Pods remain
No unexpected IAM permissions remain
No malicious Git commits remain
```

---

# 54. Example RCA — Security Incident

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `SEC-2026-160` |
| Severity | SEV-1 |
| Date | 2026-10-04 |
| Duration | 10:07–11:02 IST |
| Affected Service | `order-service` |
| Security Issue | Excessive IAM permission |
| Potentially Exposed Resource | Payment secret |
| Confirmed Data Exfiltration | No evidence found |
| Customer Data Loss | No |

---

# 55. Customer Impact

A compromised `order-service` workload accessed a Secrets Manager secret that was outside its intended permission boundary.

Security investigation found:

```text
Unauthorized secret access attempt
```

The payment secret was protected by additional controls and no evidence of successful external exfiltration was found.

The service was temporarily isolated during investigation.

Some order requests experienced elevated latency.

---

# 56. Timeline

```text
10:00  Production normal

10:07  Suspicious GetSecretValue event detected

10:09  Security alert generated

10:11  Security team validates unexpected resource access

10:13  SEV-1 declared

10:15  Order-service IAM role restricted

10:17  Suspicious Pod isolated

10:20  CloudTrail investigation begins

10:24  Git and CI history reviewed

10:28  No unauthorized Git change identified

10:31  ECR image provenance validated

10:35  Secret access scope confirmed

10:39  Potentially exposed credentials rotated

10:44  IAM policy corrected

10:48  Trusted image deployed

10:52  Order-service health restored

10:56  Business transaction validated

11:02  Security containment completed
```

---

# 57. Root Cause

The root cause was an overly broad IAM policy attached to the `order-service` workload role.

Instead of:

```text
Specific secret
```

the policy allowed access to:

```text
Multiple production secrets
```

The excessive permission increased the potential blast radius of a compromised workload.

---

# 58. Amplification Mechanism

The security issue was amplified by:

```text
Compromised workload
       ↓
Broad IAM permissions
       ↓
Access to unrelated secrets
       ↓
Potential lateral movement/data exposure
```

The application did not require those permissions.

Therefore:

> **The vulnerability was not only the workload compromise; excessive IAM permissions increased what the compromised workload could do.**

---

# 59. Contributing Factors

1. IAM policy was too broad.
2. Resource-level permissions were not enforced.
3. No automated policy validation blocked the excessive permission.
4. Security monitoring detected the suspicious access only after the event.
5. The workload had a larger potential blast radius than necessary.

---

# 60. What Went Well

- CloudTrail provided API evidence.
- Security monitoring detected unusual access.
- EKS Pod Identity allowed the affected workload identity to be identified.
- Secrets Manager supported credential rotation.
- GitOps allowed trusted workload recovery.
- ECR image provenance was available.
- No evidence of permanent customer-data loss was found.

---

# 61. What Went Wrong

- IAM policy violated least privilege.
- The application role could access unrelated resources.
- Security policy validation did not catch the issue earlier.
- The workload's potential blast radius was too large.

---

# 62. Immediate Mitigation

```text
1. Restrict IAM role
2. Isolate workload
3. Preserve evidence
4. Rotate potentially exposed secrets
5. Validate image provenance
6. Review CloudTrail
7. Deploy trusted version
8. Restore minimum required permissions
9. Validate business functionality
```

---

# 63. Corrective Actions

| Action | Owner |
|---|---|
| Replace wildcard IAM permissions | Platform/Security |
| Scope resources to exact ARNs | Platform |
| Enable IAM Access Analyzer | Security |
| Add IAM policy checks to CI | Security |
| Review all 21 workload roles | Platform |
| Rotate sensitive credentials | Security |
| Add runtime detection | Security/SRE |
| Review GitHub OIDC policies | Platform |
| Improve CloudTrail alerting | Security |
| Conduct security game day | Security/SRE |

---

# 64. Five Whys

### Why could `order-service` access the payment secret?

Its IAM role allowed access beyond its required resource.

### Why was the IAM role too broad?

The policy used overly broad resource permissions.

### Why wasn't this detected earlier?

Automated least-privilege policy validation was insufficient.

### Why was the blast radius significant?

The application identity had access to multiple production secrets.

### Why is this dangerous?

A compromised Pod can only do as much as its identity permits, so excessive permissions turn a workload compromise into a larger security incident.

---

# 65. Final Root Cause Statement

> The security incident was caused by an overly permissive IAM policy assigned to the `order-service` workload identity. The role allowed access to production secrets beyond the service's actual requirements, increasing the potential blast radius after suspicious workload activity. The incident was contained through IAM restriction, secret rotation, workload isolation and deployment of a trusted image.

---

# 66. Security Prevention Architecture

```text
                       Developer
                           │
                           ▼
                         Git PR
                           │
                           ▼
                          CI
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
   Unit Tests          Security Scan        IAM Policy
       │                   │                 Validation
       │                   │                    │
       └───────────────────┼────────────────────┘
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
                  ┌────────┴────────┐
                  │                 │
             NetworkPolicy      Pod Identity
                  │                 │
                  ▼                 ▼
              Limited Network    Least Privilege
                                    │
                                    ▼
                              AWS Resources
                                    │
                   ┌────────────────┼──────────────┐
                   │                │              │
                Secrets           S3             SQS
                   │
                   ▼
                KMS
```

---

# 67. Security Monitoring Architecture

```text
                  Production
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
    EKS Logs      AWS APIs        Network
       │              │              │
       ▼              ▼              ▼
      ELK         CloudTrail       WAF
       │              │
       └──────┬───────┘
              ▼
        Security Platform
              │
       ┌──────┼──────┐
       │      │      │
       ▼      ▼      ▼
   GuardDuty  SIEM  Alerts
       │      │      │
       └──────┼──────┘
              ▼
       Incident Response
```

---

# 68. Security Incident Runbook

Create:

```text
runbooks/security-incident.md
```

Recommended flow:

```text
1. Detect alert
2. Validate security event
3. Assign Incident Commander
4. Declare severity
5. Preserve evidence
6. Identify affected identity
7. Identify affected workload
8. Identify affected AWS resources
9. Contain
10. Rotate credentials if required
11. Isolate compromised workload
12. Review CloudTrail
13. Review EKS audit logs
14. Review Git/GitHub
15. Review ECR provenance
16. Determine scope
17. Eradicate
18. Deploy trusted configuration/image
19. Restore least privilege
20. Validate business functionality
21. Monitor
22. RCA
23. Corrective actions
```

---

# 69. Production Security Checklist

### Identity

- [ ] IAM least privilege
- [ ] Pod Identity/IRSA scoped
- [ ] GitHub OIDC restricted
- [ ] No long-lived AWS keys
- [ ] RBAC reviewed

### Containers

- [ ] Non-root
- [ ] Minimal image
- [ ] Immutable digest
- [ ] Trivy scanning
- [ ] ECR scanning
- [ ] No secrets in image

### Kubernetes

- [ ] NetworkPolicy
- [ ] Restricted ServiceAccounts
- [ ] Pod Security controls
- [ ] Resource limits
- [ ] No unnecessary host access

### AWS

- [ ] CloudTrail
- [ ] GuardDuty
- [ ] Security monitoring
- [ ] KMS
- [ ] Secrets Manager
- [ ] S3 protection
- [ ] IAM Access Analyzer

### CI/CD

- [ ] Protected branches
- [ ] CODEOWNERS
- [ ] Required reviews
- [ ] OIDC restrictions
- [ ] Security scanning
- [ ] Immutable ECR images
- [ ] GitOps approval

### Incident Response

- [ ] Evidence preserved
- [ ] Containment completed
- [ ] Credentials rotated
- [ ] Scope determined
- [ ] Trusted image deployed
- [ ] Business validated
- [ ] RCA completed

---

# 70. Important Commands

### EKS Pods

```bash
kubectl get pods -A -o wide
```

### ServiceAccount

```bash
kubectl get serviceaccount \
  -n ecommerce
```

### Pod ServiceAccount

```bash
kubectl get pod <pod> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

### RBAC

```bash
kubectl get roles -A
```

```bash
kubectl get rolebindings -A
```

```bash
kubectl get clusterrolebindings
```

### Events

```bash
kubectl get events \
  -A \
  --sort-by=.lastTimestamp
```

### IAM

```bash
aws iam list-attached-role-policies \
  --role-name <role>
```

### CloudTrail

Use CloudTrail/Event history or your organization's centralized CloudTrail/SIEM workflow to investigate API activity.

Avoid putting sensitive secret values into commands, tickets or chat.

---

# 71. What DevOps Does During a Security Incident

DevOps/Platform commonly owns:

```text
EKS
IAM
Pod Identity
NetworkPolicy
ECR
ArgoCD
CI/CD
Infrastructure
Secrets integration
Monitoring
Logging
Containment support
Recovery
```

Security team commonly owns:

```text
Incident classification
Forensics
Threat intelligence
Threat hunting
Evidence requirements
Security investigation
Compliance
Communication requirements
```

Application team owns:

```text
Application vulnerability
Business logic
Credential usage
Application remediation
Secure coding
```

DBA/database security team owns:

```text
Database access
Database users
Query investigation
Database audit
Data impact
```

The exact ownership model depends on the organization.

---

# 72. Security Incident vs Normal Incident

A normal production incident:

```text
Service failure
   ↓
Restore availability
```

A security incident:

```text
Security event
   ↓
Preserve evidence
   ↓
Contain attacker
   ↓
Determine scope
   ↓
Eradicate
   ↓
Recover
```

You cannot treat them exactly the same.

---

# 73. Security Incident Interview Answer

If asked:

> **"What would you do if you discovered a compromised Pod in production?"**

Answer:

> "First I would declare the appropriate security incident and involve the security team and Incident Commander. I would avoid destroying evidence unnecessarily and identify the Pod, ServiceAccount, IAM role, image, node and namespace. I would investigate CloudTrail, Kubernetes audit logs, application logs and image provenance to determine what the workload accessed and whether there was lateral movement. For containment, I would restrict the compromised identity, isolate the workload and rotate any potentially exposed credentials. I would then deploy a trusted immutable image, restore only the minimum required permissions and validate the application. Finally, I would perform an RCA covering the initial compromise and the controls that allowed the blast radius to increase."

---

# 74. Security Incident Interview — IAM

If asked:

> **"How do you prevent a compromised Pod from accessing all AWS resources?"**

Answer:

> "We use workload identity with least-privilege IAM roles. Each service gets its own identity and permissions are scoped to the exact AWS actions and resources required. For example, order-service might have `sqs:SendMessage` only on the order-events queue and `secretsmanager:GetSecretValue` only on its own database secret. We avoid wildcard permissions, do not put AWS credentials in Pods, and use controls such as IAM Access Analyzer and policy validation to detect excessive permissions."

---

# 75. Security Incident Interview — Secret Leak

If asked:

> **"What do you do if a production secret is leaked?"**

Answer:

> "I treat the secret as compromised. I identify where it was exposed and determine whether it was accessed. Then I revoke or disable the old credential, generate a new credential, store it in Secrets Manager, allow External Secrets Operator to synchronize it, restart or reload the affected application as required, and validate authentication. I also investigate CloudTrail and application logs to determine whether the old credential was used and perform an RCA to prevent recurrence."

---

# 76. Key Production Lessons

### Lesson 1

> **Security incidents require containment and evidence preservation, not just service restoration.**

### Lesson 2

> **Least privilege limits the blast radius of a compromised workload.**

### Lesson 3

> **A Pod compromise does not automatically mean the entire EKS cluster is compromised. Determine scope.**

### Lesson 4

> **CloudTrail is critical for AWS API investigation.**

### Lesson 5

> **Git and CI/CD are part of the production security boundary.**

### Lesson 6

> **Secrets must be revoked and rotated, not merely replaced in Kubernetes.**

### Lesson 7

> **Immutable images improve security investigation and recovery.**

### Lesson 8

> **NetworkPolicy limits lateral movement.**

### Lesson 9

> **Pod Identity should be scoped to the exact workload.**

### Lesson 10

> **The RCA must identify both the initial security weakness and the control failure that allowed the incident to expand.**

---

# 77. Phase 18 Complete

```text id="w0a4k9"
143 CrashLoopBackOff              ✅
144 OOMKilled                     ✅
145 CPU Saturation                ✅
146 Memory Saturation             ✅
147 Disk Full                     ✅
148 ImagePullBackOff              ✅
149 ALB 5xx                       ✅
150 Service Connectivity Failure  ✅
151 DNS Failure                   ✅
152 RDS Connection Exhaustion     ✅
153 Redis Failure                 ✅
154 SQS Backlog                   ✅
155 Kafka Consumer Lag            ✅
156 Node Failure                  ✅
157 AZ Failure                    ✅
158 ArgoCD Drift                  ✅
159 Bad Production Deployment     ✅
160 Security Incident             ✅
```

# Phase 18 — COMPLETE

```text
143 ──────────────────────────────── 160
          INCIDENT MANAGEMENT
                 ✅
              COMPLETE
```

The next phase is **Phase 19 — 21 Microservices Production Expansion**, beginning with **Chapter 161**, where the same production-grade patterns are systematically applied across all 21 microservices rather than manually rebuilding every service from scratch.