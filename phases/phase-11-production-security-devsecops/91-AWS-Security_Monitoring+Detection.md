# 91 — AWS Security Monitoring and Detection

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Purpose:** Build a production-grade AWS security monitoring and detection architecture for the 21-microservice platform.

---

## 1. Objective

In the previous chapters we implemented:

* Network security
* Security Groups
* Kubernetes NetworkPolicies
* Kubernetes RBAC
* AWS IAM
* EKS Pod Identity
* Secrets Manager
* KMS
* Container security
* SBOM
* Image signing
* Admission control
* Trivy
* Kyverno

Now we need to answer:

> **How do we know when someone or something is actually doing something suspicious in our AWS/EKS environment?**

Security controls prevent or restrict actions.

Security monitoring detects what happened.

Our production architecture therefore needs both.

```text
                    SECURITY PREVENTION
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
       IAM              SG/NACL        Kubernetes Policy
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                    AWS / EKS Platform
                           │
                           ▼
                    SECURITY DETECTION
                           │
        ┌──────────────────┼─────────────────────┐
        │                  │                     │
    CloudTrail         GuardDuty            AWS Config
        │                  │                     │
    VPC Flow Logs     Security Hub       EKS Audit Logs
        │                  │                     │
        └──────────────────┼─────────────────────┘
                           │
                           ▼
                     EventBridge
                           │
              ┌────────────┼────────────┐
              │            │            │
             SNS          SQS       Incident Tool
              │                         │
              ▼                         ▼
          Alerting                 Jira / SIEM
```

---

# 2. Production Security Monitoring Architecture

For our project, security monitoring is **not another set of containers that we install into every application namespace**.

A large part of security monitoring is provided by AWS managed services.

```text
                         AWS ORGANIZATION
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
     Security Account     Log Archive Account    Production
          │                    │                    │
          │                    │                    │
     Security Hub          S3 Logs              EKS
     GuardDuty             CloudTrail           RDS
     Config                                      S3
     IAM Analyzer                                IAM
          │                                      │
          └──────────────────┬───────────────────┘
                             │
                             ▼
                       EventBridge
                             │
             ┌───────────────┼────────────────┐
             │               │                │
            SNS             SQS           Automation
             │               │                │
             ▼               ▼                ▼
          Security        Incident        Remediation
           Team            Queue
                             │
                             ▼
                       Jira / SIEM
```

For the learning environment we can initially operate everything in one AWS account.

For enterprise production, separate accounts are preferred.

---

# 3. Security Monitoring Components

Our primary security monitoring stack will contain:

| Service                | Primary purpose                      |
| ---------------------- | ------------------------------------ |
| CloudTrail             | Record AWS API activity              |
| GuardDuty              | Threat detection                     |
| Security Hub           | Central security findings            |
| AWS Config             | Configuration/compliance monitoring  |
| IAM Access Analyzer    | Detect unintended resource access    |
| VPC Flow Logs          | Network traffic visibility           |
| EKS Control Plane Logs | Kubernetes API/audit visibility      |
| CloudWatch             | Log storage, metrics and alarms      |
| EventBridge            | Security event routing               |
| S3                     | Long-term log archive                |
| KMS                    | Encryption                           |
| SNS/SQS                | Notification/event delivery          |
| ELK                    | Search and operational investigation |
| Grafana                | Visualization where appropriate      |
| Jira                   | Incident/change tracking             |

---

# 4. CloudTrail

## 4.1 What is CloudTrail?

AWS CloudTrail records AWS API activity.

For example:

```text
User / Role
     │
     ▼
AWS API
     │
     ▼
CloudTrail
     │
     ▼
Event
```

Suppose someone changes a security group.

CloudTrail can record information such as:

```text
Who?
Which IAM identity?

What?
AuthorizeSecurityGroupIngress

Which resource?
sg-xxxxxxxx

When?
2026-09-26T...

From where?
Source IP / AWS context

Result?
Success / failure
```

---

# 5. Why CloudTrail Is Important

Imagine this production incident:

> Someone opened port `22` to the internet on the production security group.

You need to answer:

```text
Who changed it?
When?
Which API call?
Which security group?
From which identity?
Was it successful?
```

Without API auditing, investigation becomes difficult.

With CloudTrail:

```text
Security Group Change
        │
        ▼
CloudTrail Event
        │
        ▼
EventBridge Rule
        │
        ▼
Security Alert
        │
        ▼
Security Team
```

---

# 6. CloudTrail Production Architecture

Do not rely only on logs sitting locally in a developer environment.

A production architecture commonly uses a centralized log archive.

```text
AWS Accounts
     │
     ├── Production
     ├── QA
     ├── Dev
     ├── Security
     └── Shared Services
             │
             ▼
         CloudTrail
             │
             ▼
      Central S3 Bucket
             │
       ┌─────┴─────┐
       │           │
     KMS          Retention
       │           │
       └─────┬─────┘
             ▼
       Investigation
```

---

# 7. What CloudTrail Should Capture

At minimum, production monitoring should consider:

* Management events
* Important data events where justified
* Read/write API activity
* IAM activity
* EKS API-related AWS activity
* S3 access where required
* Lambda/API activity where applicable

Do not blindly enable every possible high-volume event.

Security logging also has a cost.

---

# 8. CloudTrail Log Storage

A common architecture:

```text
CloudTrail
    │
    ▼
S3
    │
    ├── Encryption
    ├── Versioning
    ├── Lifecycle
    └── Restricted access
```

The log archive bucket should be tightly controlled.

Application developers generally should not have permission to delete security logs.

---

# 9. CloudTrail Encryption

Use KMS where organizational requirements call for customer-managed encryption.

Conceptually:

```text
CloudTrail
     │
     ▼
S3
     │
     ▼
KMS
```

Keep KMS permissions tightly controlled.

---

# 10. CloudTrail Log File Validation

Where required, CloudTrail log file validation can help detect modification of delivered logs.

The objective is:

```text
Generated log
     │
     ▼
Stored log
     │
     ▼
Integrity verification
```

This is particularly important for forensic/audit environments.

---

# 11. CloudTrail Example Investigation

Suppose:

```text
Production RDS security group
        │
        ▼
Port 5432 opened to 0.0.0.0/0
```

Investigation:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --region ap-south-1
```

Then inspect:

```text
Username
EventTime
EventName
SourceIPAddress
Resources
CloudTrailEvent
```

---

# 12. GuardDuty

CloudTrail tells us:

> **What AWS API activity occurred?**

GuardDuty is designed to help answer:

> **Does observed activity indicate a potential threat?**

Conceptually:

```text
AWS Signals
     │
     ├── CloudTrail
     ├── VPC Flow Logs
     ├── DNS-related signals
     ├── S3 activity
     └── Other AWS threat-detection signals
             │
             ▼
         GuardDuty
             │
             ▼
          Finding
```

---

# 13. GuardDuty Findings

A GuardDuty finding can indicate suspicious activity.

Examples of categories include:

* suspicious API activity
* compromised credentials
* unusual network behavior
* potentially malicious activity involving AWS resources
* suspicious EC2 activity
* suspicious S3 activity
* EKS-related security findings where the relevant GuardDuty capabilities are enabled

The exact finding types and available protections can change, so production implementation should follow the current AWS service documentation.

---

# 14. GuardDuty Is Not an Antivirus

Important distinction:

```text
Trivy
   │
   └── Container vulnerability scanning

GuardDuty
   │
   └── AWS threat detection

CloudTrail
   │
   └── AWS API audit trail

Security Hub
   │
   └── Centralized security findings

AWS Config
   │
   └── Configuration/compliance evaluation
```

These services solve different problems.

---

# 15. GuardDuty Production Placement

GuardDuty is an AWS managed service.

We do **not** install a GuardDuty container into:

```text
ecommerce-prod
```

Instead:

```text
AWS Account
     │
     ▼
GuardDuty
     │
     ▼
Findings
```

For an organization:

```text
Management / delegated security administration
                 │
        ┌────────┼─────────┐
        ▼        ▼         ▼
      Dev       QA        Prod
        │        │         │
        └────────┼─────────┘
                 ▼
           Central findings
```

---

# 16. GuardDuty EKS Monitoring

Our project runs EKS.

Therefore we should evaluate the GuardDuty EKS-related capabilities appropriate for the current AWS service model.

The important architectural concept is:

```text
EKS
 │
 ├── Kubernetes activity
 ├── Runtime activity
 └── AWS activity
       │
       ▼
   GuardDuty
       │
       ▼
 Security Finding
```

Do not assume that every GuardDuty EKS feature is automatically enabled just because the EKS cluster exists.

Verify the protection features enabled for the account/region.

---

# 17. Security Hub

GuardDuty generates findings.

Other security products can generate findings too.

Security Hub provides a central security findings view.

```text
GuardDuty ────────┐
                  │
AWS Config ───────┤
                  │
Other integrations┤
                  ▼
             Security Hub
                  │
                  ▼
          Central Findings
```

This becomes useful when an organization has many AWS accounts.

---

# 18. Example Security Hub Finding

Suppose:

```text
Production EC2
     │
     ▼
Security finding
     │
     ▼
Security Hub
```

Security team can investigate:

```text
Account
Region
Resource
Finding type
Severity
Status
First observed
Last observed
Workflow status
```

---

# 19. Security Hub vs GuardDuty

```text
GuardDuty
    ↓
Detect threats

Security Hub
    ↓
Centralize security findings
```

Think:

```text
GuardDuty = detection engine

Security Hub = security findings hub
```

---

# 20. AWS Config

AWS Config monitors resource configuration and evaluates it against rules/policies.

Example:

```text
Security Group
     │
     ▼
AWS Config
     │
     ▼
Evaluation
     │
     ├── COMPLIANT
     └── NON_COMPLIANT
```

Example policy:

> Production security groups must not allow unrestricted SSH access.

---

# 21. Config Example

Bad:

```text
TCP
Port: 22
Source: 0.0.0.0/0
```

Potential result:

```text
NON_COMPLIANT
```

Desired:

```text
TCP
Port: 22
Source: approved administration network
```

Potential result:

```text
COMPLIANT
```

---

# 22. Config Is Not the Same as GuardDuty

This distinction is important in interviews and production.

### AWS Config

Asks:

> Is this resource configured according to our rules?

### GuardDuty

Asks:

> Is there evidence of potentially malicious or suspicious activity?

### CloudTrail

Asks:

> What AWS API activity occurred?

### Security Hub

Asks:

> What security findings do we need to manage centrally?

---

# 23. IAM Access Analyzer

We already implemented IAM least privilege.

Now we need to monitor for unintended access.

Example:

```text
S3 Bucket
    │
    ▼
IAM Access Analyzer
    │
    ▼
Potential external access
```

This can help identify resource policies that grant access outside the intended trust boundary.

Resources commonly considered include:

* S3
* IAM roles
* KMS keys
* SQS
* Secrets Manager
* other supported resource types

---

# 24. Example Problem

Suppose someone accidentally changes an S3 bucket policy:

```json
{
  "Principal": "*"
}
```

The bucket may now be accessible outside the intended account.

IAM Access Analyzer can help surface unintended external access.

---

# 25. VPC Flow Logs

Now move from AWS API activity to network activity.

```text
EKS
 │
 ▼
VPC
 │
 ▼
Flow Logs
```

Flow Logs provide network-flow visibility for supported VPC resources.

They can help answer questions such as:

```text
Source
Destination
Port
Protocol
Traffic action
```

---

# 26. Example Network Investigation

Order service cannot connect to RDS.

Application logs:

```text
connection timeout
```

We investigate:

```text
Pod
 │
 ▼
VPC routing
 │
 ▼
Security Group
 │
 ▼
Network ACL
 │
 ▼
RDS
```

Flow Logs can provide additional evidence about whether traffic was accepted or rejected.

---

# 27. Flow Logs Are Not Packet Capture

Important distinction:

```text
VPC Flow Logs
    ≠
Full packet capture
```

They provide flow metadata, not arbitrary full packet contents.

This makes them useful for network visibility without treating them as a packet-sniffing system.

---

# 28. Flow Log Destination

Possible architecture:

```text
VPC Flow Logs
       │
       ├────────► CloudWatch Logs
       │
       └────────► S3
```

Choose the destination based on:

* investigation requirements
* retention
* cost
* query requirements
* centralized logging architecture

---

# 29. EKS Control Plane Logs

EKS can provide control-plane logs to CloudWatch Logs.

For security investigations, important categories can include:

```text
API
Audit
Authenticator
Controller Manager
Scheduler
```

The exact enabled categories should be chosen based on operational/security requirements and current EKS support.

---

# 30. Kubernetes Audit Logging

Suppose someone attempts:

```bash
kubectl delete deployment order-service
```

Kubernetes audit logging can help answer:

```text
Who?
What resource?
Which namespace?
What action?
When?
```

Architecture:

```text
kubectl / API Client
        │
        ▼
EKS API Server
        │
        ▼
Kubernetes Audit Event
        │
        ▼
CloudWatch Logs
```

---

# 31. Why EKS Audit Logs Matter

Imagine:

```text
Production deployment unexpectedly deleted
```

CloudTrail may show AWS-level activity.

But you also want:

```text
Which Kubernetes identity
performed the delete operation?
```

That's where Kubernetes audit information becomes important.

---

# 32. CloudWatch

CloudWatch is the operational telemetry layer for AWS resources and AWS-generated logs/metrics.

Our architecture can look like:

```text
CloudTrail ────────┐
                   │
EKS Logs ──────────┤
                   │
VPC Flow Logs ─────┤
                   ▼
              CloudWatch
                   │
             ┌─────┴─────┐
             │           │
           Logs        Metrics
             │           │
             ▼           ▼
         Investigation  Alarms
```

---

# 33. CloudWatch vs Prometheus

We already built Prometheus.

They have different primary responsibilities.

### Prometheus

Excellent for:

```text
Kubernetes
Application metrics
Service metrics
SLOs
Custom metrics
```

### CloudWatch

Excellent for:

```text
AWS managed services
AWS logs
AWS metrics
CloudTrail
EKS control-plane logs
RDS
ALB
SQS
Lambda
```

Therefore:

```text
AWS telemetry → CloudWatch

Kubernetes/application telemetry → Prometheus
```

There can be integration between them.

---

# 34. EventBridge

Now we need to automate security event routing.

```text
Security Event
      │
      ▼
EventBridge
      │
      ├── SNS
      ├── SQS
      ├── Lambda
      ├── Incident tooling
      └── Other supported targets
```

---

# 35. Example: Security Group Modification

Imagine:

```text
AuthorizeSecurityGroupIngress
```

occurs in production.

Flow:

```text
CloudTrail
    │
    ▼
EventBridge
    │
    ▼
Rule
    │
    ▼
SNS
    │
    ▼
Security Team
```

---

# 36. EventBridge Security Rule Concept

Example conceptual event pattern:

```json
{
  "source": [
    "aws.ec2"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "ec2.amazonaws.com"
    ],
    "eventName": [
      "AuthorizeSecurityGroupIngress",
      "AuthorizeSecurityGroupEgress",
      "RevokeSecurityGroupIngress",
      "RevokeSecurityGroupEgress"
    ]
  }
}
```

This is a starting pattern.

Production rules should be tuned to reduce unnecessary alerts.

---

# 37. EventBridge → SQS

For important security events, an SQS queue can provide durable asynchronous processing.

```text
EventBridge
     │
     ▼
Security Events Queue
     │
     ▼
Security Automation
```

Advantages:

* buffering
* retry handling
* decoupling
* asynchronous processing

---

# 38. EventBridge → SNS

SNS is useful for notification fan-out.

```text
EventBridge
     │
     ▼
SNS
 ┌───┼────┐
 ▼   ▼    ▼
Email Chat Incident Tool
```

---

# 39. EventBridge → Jira

Do not make Jira the primary security-event transport.

Instead:

```text
Security Event
     │
     ▼
EventBridge
     │
     ▼
Security workflow
     │
     ▼
Incident / Finding
     │
     ▼
Jira
```

Jira becomes the tracking system.

---

# 40. Security Monitoring and Existing ELK

We already have:

```text
Fluent Bit
Logstash
Elasticsearch
Kibana
```

These primarily handle application/platform logs.

Security telemetry can also be centralized when there is a justified integration.

Example:

```text
AWS
 │
 ├── CloudTrail
 ├── VPC Flow Logs
 ├── EKS Logs
 └── Security Events
          │
          ▼
       AWS Log Layer
          │
          ▼
       Integration
          │
          ▼
     Elasticsearch
          │
          ▼
        Kibana
```

---

# 41. Do Not Put Everything Into Elasticsearch

This is important.

You don't automatically send every AWS security signal into ELK.

Consider:

```text
CloudTrail
Security Hub
GuardDuty
VPC Flow Logs
EKS audit logs
```

Then decide:

```text
Which events require
long-term search?

Which require alerting?

Which require compliance retention?

Which require SIEM ingestion?
```

Architecture should be driven by requirements.

---

# 42. Security Monitoring + Grafana

Grafana already monitors:

```text
CPU
Memory
Pods
Services
RDS
Redis
SQS
Application metrics
SLOs
```

Security dashboards can complement this.

For example:

```text
Security Dashboard

GuardDuty findings
CloudTrail suspicious events
Security group changes
IAM failures
EKS authentication events
VPC rejected traffic
```

But security findings should not be forced into Prometheus simply because Grafana exists.

---

# 43. Alert Severity

We need consistent severity.

Example:

```text
P1
│
├── Confirmed major compromise
└── Critical production security incident

P2
│
├── High-confidence suspicious activity
└── Significant security control violation

P3
│
├── Configuration/security violation
└── Investigation required

P4
│
└── Informational
```

The exact definitions should be agreed with the organization's incident-response policy.

---

# 44. Example Security Alerts

### Alert 1 — Production IAM Change

```text
IAM policy modified
        │
        ▼
CloudTrail
        │
        ▼
EventBridge
        │
        ▼
Security Alert
```

---

### Alert 2 — Security Group Opened

```text
Port 22 → 0.0.0.0/0
        │
        ▼
CloudTrail / Config
        │
        ▼
Security finding
```

---

### Alert 3 — GuardDuty Finding

```text
Suspicious AWS activity
        │
        ▼
GuardDuty
        │
        ▼
Security Hub
        │
        ▼
EventBridge
        │
        ▼
Incident
```

---

### Alert 4 — Unexpected Secrets Access

```text
Unexpected IAM role
       │
       ▼
Secrets Manager API
       │
       ▼
CloudTrail
       │
       ▼
Investigation
```

---

# 45. Order-Service Security Monitoring

Let's apply this to our project.

Order Service architecture:

```text
Client
  │
  ▼
ALB
  │
  ▼
order-service
  │
  ├── RDS
  ├── Redis
  ├── SQS
  ├── S3
  └── Secrets Manager
```

Its IAM role may have:

```text
SQS
Secrets Manager
S3
```

permissions.

---

# 46. Scenario — Unexpected Secrets Access

Suppose an application role suddenly accesses a secret it normally doesn't use.

Investigation:

```text
Secrets Manager
       │
       ▼
CloudTrail
       │
       ▼
Who accessed secret?
       │
       ▼
IAM Role
       │
       ▼
Pod Identity Association
       │
       ▼
Order Service Pod
```

Then inspect:

```bash
kubectl get pods -n ecommerce-prod
```

and:

```bash
kubectl describe pod <pod-name> -n ecommerce-prod
```

---

# 47. Scenario — Unexpected S3 Access

Order Service normally accesses:

```text
s3://company-ecommerce-invoices/orders/
```

But an unexpected identity accesses another bucket.

Investigation:

```text
CloudTrail
    │
    ▼
S3 Data Event
    │
    ▼
IAM Principal
    │
    ▼
Pod Identity / Role
    │
    ▼
Workload
```

This is why least-privilege IAM and auditing work together.

---

# 48. Scenario — Security Group Changed

```text
Production ALB SG
        │
        ▼
Unexpected inbound rule
        │
        ▼
CloudTrail
        │
        ▼
Identify IAM principal
        │
        ▼
Check Jira change
        │
        ├── Expected change?
        │       │
        │       └── Yes → document
        │
        └── No
             │
             ▼
         Security incident
```

This is an important production connection:

> **Security monitoring should be correlated with authorized change management.**

---

# 49. Scenario — Kubernetes Deployment Deleted

```text
Production
    │
    ▼
Deployment disappears
    │
    ▼
EKS audit logs
    │
    ▼
Identify Kubernetes actor
    │
    ▼
Check GitOps
    │
    ▼
Check Jira
```

Possible conclusions:

```text
Expected ArgoCD action
```

or:

```text
Unexpected human/API action
```

---

# 50. Scenario — Unauthorized Production Deployment

Our intended deployment path is:

```text
Developer
   │
   ▼
GitHub PR
   │
   ▼
CI
   │
   ▼
ECR
   │
   ▼
GitOps PR
   │
   ▼
Jira Change
   │
   ▼
Approval
   │
   ▼
ArgoCD
   │
   ▼
EKS
```

If somebody bypasses this and directly modifies Kubernetes:

```text
kubectl
   │
   ▼
EKS API
```

the audit trail becomes important.

---

# 51. Security Monitoring Does Not Replace Prevention

This distinction is critical.

```text
IAM
     ↓
Prevent unauthorized AWS access

RBAC
     ↓
Prevent unauthorized Kubernetes access

NetworkPolicy
     ↓
Restrict pod communication

Kyverno
     ↓
Prevent unsafe workloads

Trivy
     ↓
Detect vulnerable images before deployment

GuardDuty
     ↓
Detect suspicious activity

CloudTrail
     ↓
Record AWS activity
```

We need all of them.

---

# 52. Security Monitoring Account Architecture

For a mature enterprise:

```text
                 AWS Organization
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
   Management        Security          Log Archive
       │              Account             Account
       │                │                 │
       │          GuardDuty             S3
       │          Security Hub        CloudTrail
       │          Config
       │          IAM Analyzer
       │
       ├───────────────┬─────────────────┐
       │               │                 │
      Dev              QA               Prod
       │               │                 │
      EKS             EKS               EKS
```

This provides stronger isolation than placing all environments into one account.

---

# 53. Central Security Team

A centralized security team can monitor:

```text
Dev
QA
Staging
Production
```

without requiring developers to have broad security administration permissions.

Example separation:

```text
Platform Team
    │
    └── EKS infrastructure

DevOps Team
    │
    └── CI/CD / Terraform

Security Team
    │
    ├── GuardDuty
    ├── Security Hub
    ├── Config
    └── security investigations

Application Team
    │
    └── Microservices
```

---

# 54. Security Log Retention

Retention requirements depend on:

* company policy
* compliance requirements
* investigation needs
* storage cost
* regulatory requirements

Typical architecture:

```text
Hot
 │
 └── CloudWatch / Elasticsearch

Warm
 │
 └── searchable archive

Cold
 │
 └── S3 archival
```

Don't choose a retention number merely because it is common elsewhere.

Use the organization's actual requirements.

---

# 55. Protect Security Logs

Security logs themselves are sensitive.

We should restrict:

```text
Delete
Modify
Read
Export
```

permissions.

Example:

```text
Application Role
     │
     └── NO access to CloudTrail archive

Security Role
     │
     └── Controlled read access

Log Archive Administration
     │
     └── Restricted administration
```

---

# 56. Security Monitoring IAM

Security monitoring itself needs least privilege.

Don't give:

```text
Security monitoring service
       │
       ▼
AdministratorAccess
```

unless the service's documented architecture specifically requires a permission.

Use the AWS-managed service roles/policies and documented least-privilege configuration appropriate to the feature.

---

# 57. Example CloudTrail Investigation Commands

List recent events:

```bash
aws cloudtrail lookup-events \
  --region ap-south-1 \
  --max-results 20
```

Filter by event:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=DeleteRole \
  --region ap-south-1
```

Filter by username:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=Username,AttributeValue=some-user \
  --region ap-south-1
```

---

# 58. Check GuardDuty

The exact APIs/features depend on the current GuardDuty configuration.

Start by checking detector state:

```bash
aws guardduty list-detectors \
  --region ap-south-1
```

Then retrieve findings for the detector according to the currently supported GuardDuty API.

---

# 59. Check Security Hub

Check whether Security Hub is enabled/configured in the region:

```bash
aws securityhub describe-hub \
  --region ap-south-1
```

Findings can then be queried using the Security Hub APIs/CLI.

---

# 60. Check CloudTrail

```bash
aws cloudtrail describe-trails \
  --region ap-south-1
```

Then:

```bash
aws cloudtrail get-trail-status \
  --name <trail-name> \
  --region ap-south-1
```

This helps verify the trail configuration/status.

---

# 61. Check EKS Logging

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1 \
  --query 'cluster.logging'
```

This helps verify the configured EKS control-plane log types.

---

# 62. Check Kubernetes Audit Activity

Once EKS control-plane logs are delivered to CloudWatch, investigate the corresponding EKS log groups and streams.

The exact log-group naming and filtering should be verified against the current EKS logging configuration.

Conceptually:

```text
CloudWatch
   │
   ▼
EKS control-plane logs
   │
   ▼
Audit records
```

---

# 63. VPC Flow Log Investigation

If flow logs are delivered to CloudWatch, investigate the relevant log group.

Example architecture:

```text
VPC
 │
 ▼
Flow Logs
 │
 ▼
CloudWatch
 │
 ▼
Logs Insights
```

You can then query:

```text
srcaddr
dstaddr
srcport
dstport
action
bytes
packets
```

using the appropriate VPC Flow Logs format.

---

# 64. Security Detection Flow

Our complete detection flow is:

```text
                     AWS / EKS
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
   API Activity       Network          Kubernetes
       │                 │                 │
       ▼                 ▼                 ▼
  CloudTrail        VPC Flow Logs      EKS Logs
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
               ┌─────────┴─────────┐
               │                   │
          GuardDuty             Config
               │                   │
               └─────────┬─────────┘
                         ▼
                   Security Hub
                         │
                         ▼
                    EventBridge
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
            SNS         SQS        Lambda
             │           │           │
             └───────────┼───────────┘
                         ▼
                 Incident Management
                         │
                         ▼
                  Jira / Security Team
```

---

# 65. Integration With Our DevSecOps Pipeline

Now connect everything we have built.

```text
Developer
    │
    ▼
GitHub
    │
    ▼
CI
 ┌──┼───────────────┐
 │  │               │
Sonar Trivy      Gitleaks
 │  │               │
 └──┼───────────────┘
    │
    ▼
Docker Image
    │
    ▼
SBOM
    │
    ▼
Image Signing
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
Kyverno
    │
    ▼
EKS
    │
    ▼
Runtime
    │
    ├── Prometheus
    ├── Grafana
    ├── Fluent Bit
    ├── ELK
    ├── OpenTelemetry
    └── Jaeger

AWS Security Layer
    │
    ├── CloudTrail
    ├── GuardDuty
    ├── Security Hub
    ├── Config
    ├── IAM Access Analyzer
    ├── VPC Flow Logs
    └── EKS Audit Logs
```

This is now becoming a real enterprise DevSecOps architecture.

---

# 66. Security Monitoring vs Observability

Do not confuse these.

### Observability

Answers:

> Is the application working?

Examples:

```text
CPU
Memory
Latency
Errors
Throughput
Logs
Traces
```

### Security monitoring

Answers:

> Is someone or something behaving in a potentially unauthorized or malicious way?

Examples:

```text
Unexpected IAM activity
Suspicious network behavior
Security-group modification
Unauthorized Kubernetes API action
Unexpected secret access
Threat detection finding
```

Both are required.

---

# 67. Production Security Incident Flow

Suppose GuardDuty produces a high-priority finding.

```text
GuardDuty
    │
    ▼
Security Hub
    │
    ▼
EventBridge
    │
    ▼
Incident Queue
    │
    ▼
Security Team
    │
    ▼
Incident Response
```

The team then correlates:

```text
GuardDuty
     +
CloudTrail
     +
VPC Flow Logs
     +
EKS Audit Logs
     +
Application Logs
     +
Prometheus
     +
Jira
```

This is much more powerful than investigating each system independently.

---

# 68. Example Investigation — Compromised Pod

Imagine:

```text
order-service pod
       │
       ▼
Suspicious outbound traffic
```

Investigation:

### Step 1

Check Kubernetes:

```bash
kubectl get pods -n ecommerce-prod
```

### Step 2

Check pod details:

```bash
kubectl describe pod <pod> -n ecommerce-prod
```

### Step 3

Check application logs:

```bash
kubectl logs <pod> -n ecommerce-prod
```

### Step 4

Check VPC Flow Logs.

### Step 5

Check GuardDuty.

### Step 6

Identify the pod's ServiceAccount.

### Step 7

Identify its EKS Pod Identity association.

### Step 8

Identify the IAM role.

### Step 9

Review CloudTrail activity for that role.

### Step 10

Check recent deployment/change records.

```text
Pod
 │
 ├── Logs
 ├── Network
 ├── IAM
 ├── Deployment
 └── AWS activity
```

---

# 69. Detection + Response

Detection alone is not enough.

We need:

```text
Detect
  ↓
Investigate
  ↓
Contain
  ↓
Eradicate
  ↓
Recover
  ↓
Learn
```

This connects directly with our previously created:

```text
50-Incident-Response.md
```

---

# 70. Example Automated Response

Some events may support controlled automation.

Example:

```text
Security Finding
      │
      ▼
EventBridge
      │
      ▼
Lambda / Automation
      │
      ▼
Containment
```

Potential actions depend heavily on the event and organization's incident-response policy.

Examples might include:

* disabling a compromised credential
* isolating a resource
* adding a resource to an investigation workflow
* creating an incident
* notifying security responders

**Do not automatically perform destructive actions for every finding.**

False positives exist.

---

# 71. Security Automation Principle

Use:

```text
High confidence + high impact
        │
        ▼
Controlled automation
```

For uncertain findings:

```text
Detection
   │
   ▼
Human investigation
```

This avoids an automated security control accidentally taking down production.

---

# 72. Production Change Correlation

This is particularly important for our project.

Suppose an alert says:

```text
Production security group changed
```

Before declaring an incident:

```text
Check Event
    │
    ▼
CloudTrail
    │
    ▼
IAM identity
    │
    ▼
GitHub
    │
    ▼
Jira CHG
```

If there is:

```text
CHG-10452
```

with an approved implementation window, the activity may be expected.

If there is no approved change:

```text
Potential unauthorized activity
```

This is why our:

```text
79-Jira-Change-Management-and-Production-Release.md
```

is part of the security story too.

---

# 73. What Gets Installed Where?

This is one of the most important production questions.

| Component              | Where                   |
| ---------------------- | ----------------------- |
| CloudTrail             | AWS managed             |
| GuardDuty              | AWS managed             |
| Security Hub           | AWS managed             |
| AWS Config             | AWS managed             |
| IAM Access Analyzer    | AWS managed             |
| VPC Flow Logs          | AWS/VPC configuration   |
| EKS control-plane logs | EKS → CloudWatch        |
| CloudWatch             | AWS managed             |
| EventBridge            | AWS managed             |
| SNS                    | AWS managed             |
| SQS                    | AWS managed             |
| KMS                    | AWS managed             |
| Prometheus             | EKS/platform            |
| Grafana                | EKS/platform or managed |
| Fluent Bit             | EKS DaemonSet           |
| Logstash               | EKS/platform            |
| Elasticsearch          | EKS/platform or managed |
| Kibana                 | EKS/platform or managed |
| ArgoCD                 | EKS/platform            |
| Kyverno                | EKS/platform            |
| Trivy                  | CI/security pipeline    |
| SonarQube              | Central platform/SaaS   |
| Gitleaks               | CI                      |
| Cosign                 | CI/release pipeline     |

This is the production-placement model we want to maintain throughout the project.

---

# 74. Terraform Strategy

Security infrastructure should also be managed as code wherever practical.

Example:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── cloudtrail/
    │   ├── guardduty/
    │   ├── security-hub/
    │   ├── config/
    │   ├── flow-logs/
    │   └── eventbridge/
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

---

# 75. GitOps vs Terraform

Use the right tool for the right layer.

### Terraform

Good for:

```text
AWS infrastructure
IAM
CloudTrail
GuardDuty configuration
Security Hub
VPC Flow Logs
S3
EventBridge
SNS
SQS
```

### GitOps/ArgoCD

Good for:

```text
Kubernetes resources
Kyverno
Prometheus
Grafana
Fluent Bit
Logstash
ArgoCD
Application workloads
```

Architecture:

```text
Terraform
   │
   └── AWS layer

GitOps
   │
   └── Kubernetes layer
```

---

# 76. Security Monitoring Repository Structure

We can extend our infrastructure repository:

```text
infrastructure/
│
├── terraform/
│   ├── modules/
│   │   ├── cloudtrail/
│   │   ├── guardduty/
│   │   ├── security-hub/
│   │   ├── config/
│   │   ├── flow-logs/
│   │   └── eventbridge/
│   │
│   └── environments/
│       ├── dev/
│       ├── qa/
│       ├── staging/
│       └── prod/
│
└── README.md
```

Platform Kubernetes components remain in:

```text
platform/
├── monitoring/
├── logging/
├── security/
│   ├── kyverno/
│   └── policies/
└── README.md
```

---

# 77. Production Security Monitoring Checklist

## AWS

* [ ] CloudTrail enabled
* [ ] Organization-level strategy defined
* [ ] Central log archive designed
* [ ] S3 log protection configured
* [ ] KMS strategy defined
* [ ] GuardDuty enabled
* [ ] Required GuardDuty protections evaluated
* [ ] Security Hub enabled
* [ ] AWS Config enabled where required
* [ ] IAM Access Analyzer enabled
* [ ] VPC Flow Logs enabled where justified
* [ ] EKS control-plane logs configured
* [ ] EventBridge security rules configured

## EKS

* [ ] EKS audit logging reviewed
* [ ] Kubernetes RBAC implemented
* [ ] Pod Identity implemented
* [ ] NetworkPolicies implemented
* [ ] Kyverno implemented
* [ ] SecurityContext enforced
* [ ] Non-root containers
* [ ] Image signing
* [ ] Image verification
* [ ] Security events correlated with deployments

## Operations

* [ ] Security alerts routed
* [ ] Incident ownership defined
* [ ] Jira integration/process defined
* [ ] Runbooks available
* [ ] Security logs protected
* [ ] Retention requirements defined
* [ ] Security monitoring tested
* [ ] Incident response exercised

---

# 78. Final Architecture

Our complete security architecture is now:

```text
                         AWS ORGANIZATION
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
          CloudTrail         GuardDuty          Config
             │                  │                  │
             └──────────────────┼──────────────────┘
                                │
                         Security Hub
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
        IAM Analyzer       VPC Flow Logs       EKS Logs
             │                  │                  │
             └──────────────────┼──────────────────┘
                                │
                         CloudWatch / S3
                                │
                                ▼
                           EventBridge
                                │
                  ┌─────────────┼─────────────┐
                  │             │             │
                 SNS           SQS          Lambda
                  │             │             │
                  └─────────────┼─────────────┘
                                │
                         Security Team
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                   SIEM        Jira       Alerts
                    │
                    ▼
                Investigation
                    │
                    ▼
              Incident Response
```

And our application platform remains:

```text
Developer
   │
   ▼
GitHub
   │
   ▼
CI
   ├── SonarQube
   ├── Gitleaks
   ├── Trivy
   ├── SBOM
   └── Cosign
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
Kyverno
   │
   ▼
EKS
   │
   ├── Prometheus
   ├── Grafana
   ├── Fluent Bit
   ├── Logstash
   ├── Elasticsearch
   ├── Kibana
   ├── OpenTelemetry
   └── Jaeger
```

---

# 79. The Key Production Mental Model

Remember this separation:

```text
                  PREVENTION
                     │
       ┌─────────────┼──────────────┐
       │             │              │
      IAM       NetworkPolicy    Kyverno
       │             │              │
       └─────────────┼──────────────┘
                     │
                     ▼
                  DETECTION
                     │
       ┌─────────────┼──────────────┐
       │             │              │
  CloudTrail      GuardDuty      Config
       │             │              │
       └─────────────┼──────────────┘
                     │
                     ▼
               INVESTIGATION
                     │
       ┌─────────────┼──────────────┐
       │             │              │
   EKS Logs      Flow Logs        ELK
       │             │              │
       └─────────────┼──────────────┘
                     │
                     ▼
                  RESPONSE
                     │
              Incident Response
                     │
                     ▼
                  RECOVERY
```

The goal is **not simply to install security tools**.

The goal is to create a chain where:

> **An action happens → it is recorded → suspicious activity is detected → the right team is alerted → the activity is investigated → the incident is contained → the system is recovered → the evidence is retained.**

That is the production security-monitoring model for our EKS platform.

---

## Next chapter

### `92-Security-Monitoring-Integration-and-SIEM.md`

We will connect this security layer to the monitoring/logging platform we already built:

```text
CloudTrail
GuardDuty
Security Hub
AWS Config
VPC Flow Logs
EKS Audit Logs
        │
        ▼
Central Security Events
        │
        ├── Elasticsearch
        ├── Kibana
        ├── Grafana
        ├── Alertmanager
        ├── EventBridge
        └── Jira
```

The next chapter will focus specifically on **how these security signals are integrated, normalized, searched, alerted on, and investigated alongside the existing Prometheus + Grafana + ELK + OpenTelemetry platform**.
