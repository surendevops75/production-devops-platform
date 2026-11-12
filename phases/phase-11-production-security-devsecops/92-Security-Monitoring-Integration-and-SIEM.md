# 92 — Security Monitoring Integration and SIEM

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Kubernetes:** Amazon EKS
> **Purpose:** Integrate AWS security signals with our existing Prometheus, Grafana, ELK, OpenTelemetry, Alertmanager, EventBridge, and Jira ecosystem.

---

## 1. Objective

In Chapter 91 we built the AWS security detection layer:

```text
CloudTrail
GuardDuty
Security Hub
AWS Config
IAM Access Analyzer
VPC Flow Logs
EKS Audit Logs
        │
        ▼
Security Events
```

Now we need to answer a practical production question:

> **Where do these security events go, how are they searched, how are they alerted on, and how does an engineer investigate them?**

Our goal is to create a centralized security-observability architecture without mixing every type of telemetry into one system.

---

# 2. Important: SIEM vs Observability

Before building the architecture, understand the difference.

### Observability platform

Our existing platform answers:

> **Is the system working?**

```text
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
```

It handles:

* CPU
* memory
* latency
* errors
* application logs
* traces
* Kubernetes metrics
* infrastructure metrics

### Security monitoring / SIEM

Security monitoring answers:

> **Is something potentially unauthorized, suspicious, or malicious happening?**

Examples:

```text
Unexpected IAM activity
Security group modification
Suspicious API calls
GuardDuty finding
Unauthorized Kubernetes operation
Unexpected secret access
Unusual network connection
```

A SIEM generally brings security-relevant events together for investigation, correlation, detection, and response.

---

# 3. Our Target Architecture

We will use a **hybrid architecture**.

AWS-native security services remain the primary detection sources.

Our existing ELK platform becomes an investigation/search destination where appropriate.

```text
                         AWS
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
  CloudTrail          GuardDuty          AWS Config
       │                  │                  │
       │             Security Hub          │
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                    Security Events
                          │
                     EventBridge
                          │
              ┌───────────┼───────────┐
              │           │           │
             SQS         SNS        Lambda
              │
              ▼
       Security Event Pipeline
              │
              ▼
       Normalization / Routing
              │
       ┌──────┴───────────┐
       │                  │
       ▼                  ▼
 Elasticsearch        S3 Archive
       │
       ▼
     Kibana
       │
       ├── Security dashboards
       ├── Investigation
       ├── Search
       └── Correlation
```

---

# 4. Don't Send Everything Directly to Elasticsearch

A common beginner architecture is:

```text
CloudTrail → Elasticsearch
GuardDuty → Elasticsearch
Config → Elasticsearch
EKS → Elasticsearch
Everything → Elasticsearch
```

This can become expensive and difficult to operate.

Instead:

```text
Source
  │
  ▼
Security Event
  │
  ├── Alert immediately?
  │       └── EventBridge → SNS/SQS
  │
  ├── Need investigation?
  │       └── Elasticsearch
  │
  └── Long-term retention?
          └── S3
```

Every event should have a purpose.

---

# 5. Security Event Pipeline

Our production flow:

```text
AWS Security Services
        │
        ▼
 EventBridge / AWS APIs
        │
        ▼
 Security Event Queue
        │
        ▼
 Security Processor
        │
        ├── Normalize
        ├── Enrich
        ├── Classify
        └── Route
             │
      ┌──────┼────────┐
      │      │        │
      ▼      ▼        ▼
     S3     ELK      Alert
```

---

# 6. Why SQS?

Security events should not depend directly on Elasticsearch availability.

Bad design:

```text
EventBridge
     │
     ▼
Elasticsearch
```

If Elasticsearch is unavailable:

```text
Event
 │
 X
```

Potentially problematic.

Better:

```text
EventBridge
     │
     ▼
SQS
     │
     ▼
Processor
     │
     ▼
Elasticsearch
```

Now:

```text
Elasticsearch DOWN
        │
        ▼
SQS stores events
        │
        ▼
Elasticsearch RECOVERS
        │
        ▼
Processor continues
```

This gives us buffering.

---

# 7. Security Queue

Example:

```text
security-events-prod
```

Architecture:

```text
EventBridge
     │
     ▼
security-events-prod
     │
     ├── Visibility timeout
     ├── Retention
     ├── Encryption
     └── DLQ
```

---

# 8. Dead-Letter Queue

If an event repeatedly fails processing:

```text
SQS
 │
 ├── Attempt 1 → fail
 ├── Attempt 2 → fail
 ├── Attempt 3 → fail
 │
 ▼
DLQ
```

Example:

```text
security-events-prod-dlq
```

Never silently discard security events.

---

# 9. Security Event Processor

The processor can be implemented as:

* Lambda
* ECS task
* Kubernetes service
* another approved event-processing platform

For our architecture, the exact implementation depends on event volume and operational requirements.

A conceptual processor:

```text
SQS
 │
 ▼
Security Event Processor
 │
 ├── Parse
 ├── Validate
 ├── Normalize
 ├── Enrich
 ├── Classify severity
 └── Route
```

---

# 10. Example Raw Event

CloudTrail events have structured JSON.

Conceptually:

```json
{
  "eventSource": "ec2.amazonaws.com",
  "eventName": "AuthorizeSecurityGroupIngress",
  "awsRegion": "ap-south-1",
  "sourceIPAddress": "203.0.113.10",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:iam::123456789012:role/example"
  }
}
```

The exact event structure varies by AWS service and event type.

---

# 11. Normalize the Event

Instead of storing completely different structures for every source, create a common security-event representation.

Example:

```json
{
  "timestamp": "2026-09-26T10:30:00Z",
  "source": "aws.cloudtrail",
  "event_type": "security_group_change",
  "severity": "high",
  "account_id": "123456789012",
  "region": "ap-south-1",
  "environment": "prod",
  "service": "ec2",
  "action": "AuthorizeSecurityGroupIngress",
  "principal": "arn:aws:iam::123456789012:role/example",
  "resource": "sg-0123456789",
  "source_ip": "203.0.113.10"
}
```

This makes searching easier.

---

# 12. Event Enrichment

Raw security events often don't contain all the context an investigator wants.

We can enrich them.

Example:

```text
Raw event
    │
    ▼
IAM role
    │
    ▼
Role metadata
    │
    ▼
Environment
    │
    ▼
Owner
    │
    ▼
Service
    │
    ▼
Final security event
```

Possible enrichment:

```text
account
environment
application
team
resource owner
cluster
namespace
service
Jira change ID
deployment version
```

Be careful not to put sensitive secrets into enrichment fields.

---

# 13. Security Event Schema

A useful schema might include:

```text
@timestamp
event.id
event.type
event.action
event.category
event.severity
event.outcome

cloud.account.id
cloud.region
cloud.provider

aws.service
aws.resource
aws.event_name

user.id
user.arn
source.ip

kubernetes.cluster
kubernetes.namespace
kubernetes.pod
kubernetes.serviceaccount

service.name
environment

jira.change_id
git.commit
image.digest
```

Not every event will populate every field.

---

# 14. Security Event Categories

We can classify events into:

```text
IAM
NETWORK
KUBERNETES
CONTAINER
DATA
CONFIGURATION
THREAT
AUTHENTICATION
ACCESS
DEPLOYMENT
```

Example:

```text
Security Group Change
→ NETWORK

IAM Policy Change
→ IAM

GuardDuty Finding
→ THREAT

Kubernetes Deployment Delete
→ KUBERNETES

Secret Access
→ DATA / ACCESS
```

---

# 15. Security Severity

Example normalized severity:

```text
critical
high
medium
low
informational
```

But severity must not be assigned purely based on the event name.

Context matters.

For example:

```text
Security Group modification
```

could be:

```text
Expected approved change
```

or:

```text
Unauthorized production modification
```

The latter deserves more attention.

---

# 16. Correlating With Jira

This is where our previous chapters become useful.

Suppose:

```text
Security Group changed
```

Our event processor can look for an associated change reference when such correlation is available.

Architecture:

```text
Security Event
      │
      ▼
CloudTrail
      │
      ▼
Event Processing
      │
      ▼
Change Correlation
      │
      ▼
Jira CHG-10452
```

Then Kibana could show:

```text
Change ID: CHG-10452
Service: order-service
Environment: prod
Deployment: approved
```

or:

```text
Change ID: NONE
```

That should trigger investigation according to policy.

---

# 17. Correlating With GitHub

Our deployment chain is:

```text
GitHub Commit
      │
      ▼
CI
      │
      ▼
ECR Image Digest
      │
      ▼
GitOps Commit
      │
      ▼
ArgoCD
      │
      ▼
EKS
```

We can correlate:

```text
Security Event
      │
      ▼
Kubernetes resource
      │
      ▼
Deployment
      │
      ▼
Image digest
      │
      ▼
GitOps commit
      │
      ▼
GitHub commit
```

This gives investigators much better context.

---

# 18. Example

Security event:

```text
Deployment changed in production
```

Investigator can potentially answer:

```text
Who initiated it?
What image?
Which digest?
Which Git commit?
Which GitOps commit?
Was there a Jira change?
Did ArgoCD perform it?
Was it manually changed?
```

That is a production-grade audit chain.

---

# 19. ELK Integration

Our existing logging pipeline is:

```text
Application
    │
    ▼
stdout/stderr
    │
    ▼
Fluent Bit
    │
    ▼
Logstash
    │
    ▼
Elasticsearch
    │
    ▼
Kibana
```

Security events should remain distinguishable.

For example:

```text
Elasticsearch

logs-application-*
logs-security-*
logs-audit-*
```

or an equivalent data-stream/index strategy.

---

# 20. Separate Security Data

Don't mix:

```text
application logs
```

and:

```text
security events
```

without a clear schema.

Better:

```text
Application logs
     │
     ▼
application data stream

Security events
     │
     ▼
security data stream

Kubernetes audit
     │
     ▼
audit data stream
```

This improves:

* access control
* retention
* search
* lifecycle management
* investigation

---

# 21. Security Index/Data Stream

Example conceptual naming:

```text
security-events-prod
security-events-nonprod
kubernetes-audit-prod
aws-cloudtrail-prod
```

In a modern Elasticsearch deployment, data streams and lifecycle policies may be preferable to manually creating many daily indices.

---

# 22. Security Data Retention

Security data can have different retention requirements.

Example architecture:

```text
Security Events
      │
      ├── Hot → Elasticsearch
      │
      ├── Warm → lower-cost searchable storage
      │
      └── Archive → S3
```

Retention must follow:

* company policy
* compliance
* legal requirements
* investigation needs
* cost constraints

---

# 23. Kibana Security Dashboard

Create a dedicated dashboard:

```text
┌──────────────────────────────────────────┐
│         Production Security              │
├──────────────────────────────────────────┤
│ Critical Findings      High Findings     │
│       2                      8            │
├──────────────────────────────────────────┤
│ IAM Changes             SG Changes       │
│      15                       3           │
├──────────────────────────────────────────┤
│ Kubernetes Audit Events                  │
│              1,238                       │
├──────────────────────────────────────────┤
│ GuardDuty Findings                       │
│              4                           │
├──────────────────────────────────────────┤
│ Unauthorized Activity                   │
│              1                           │
└──────────────────────────────────────────┘
```

---

# 24. Kibana Investigation Workflow

An engineer receives:

```text
HIGH — Unexpected IAM activity
```

Open Kibana.

Filter:

```text
environment: prod
AND
severity: high
```

Then:

```text
event.type
event.action
user.arn
source.ip
resource
timestamp
```

---

# 25. Search by IAM Principal

Example conceptual KQL:

```text
user.arn : "arn:aws:iam::*:role/order-service-*"
```

This lets us investigate activity generated by an application role.

---

# 26. Search Security Group Changes

Conceptual KQL:

```text
event.action : (
  "AuthorizeSecurityGroupIngress" OR
  "AuthorizeSecurityGroupEgress" OR
  "RevokeSecurityGroupIngress" OR
  "RevokeSecurityGroupEgress"
)
```

Then filter:

```text
environment : "prod"
```

---

# 27. Search Unauthorized Kubernetes Operations

Conceptually:

```text
event.category : "kubernetes"
AND
event.action : "delete"
AND
environment : "prod"
```

Then investigate:

```text
user
namespace
resource
resource_name
source_ip
```

---

# 28. Search Secret Access

Conceptual query:

```text
aws.service : "secretsmanager"
AND
event.action : "GetSecretValue"
AND
environment : "prod"
```

Then compare the principal with the expected workload identity.

---

# 29. Search S3 Access

Conceptually:

```text
aws.service : "s3"
AND
event.action : *
AND
environment : "prod"
```

Then narrow by:

```text
bucket
principal
source
object
action
```

Data-event logging can generate significant volume, so enable it selectively based on requirements.

---

# 30. Security Alerts

We already have Alertmanager for Prometheus.

Do not force all security events through Prometheus.

Instead:

```text
Security event
      │
      ▼
EventBridge
      │
      ├── SNS
      ├── SQS
      └── Incident system
```

Prometheus/Alertmanager remains primarily for metric-based alerts.

---

# 31. Two Alerting Worlds

### Operational alert

```text
Prometheus
    │
    ▼
Alertmanager
    │
    ▼
On-call
```

Example:

```text
API error rate > SLO
```

### Security alert

```text
GuardDuty / CloudTrail / Security Hub
    │
    ▼
EventBridge
    │
    ▼
Security notification/incident workflow
```

Example:

```text
Potential credential compromise
```

---

# 32. Combined Incident View

An actual incident can involve both.

Example:

```text
09:10 — IAM anomaly
09:12 — GuardDuty finding
09:14 — API latency increases
09:15 — 5xx increases
09:17 — unusual outbound network traffic
```

Now we correlate:

```text
Security telemetry
       +
Application telemetry
       +
Infrastructure telemetry
```

This is where centralized observability becomes powerful.

---

# 33. Prometheus Security Metrics

Prometheus can still expose **security-related operational metrics**.

For example:

```text
security_event_processor_events_total
security_event_processor_failures_total
security_event_queue_depth
security_event_processing_latency_seconds
```

These are metrics about the security-monitoring pipeline itself.

They are not replacements for the security event data.

---

# 34. Monitor the Security Pipeline

Our security monitoring system itself must be monitored.

```text
EventBridge
     │
     ▼
SQS
     │
     ▼
Processor
     │
     ▼
Elasticsearch
```

Monitor:

```text
Queue depth
Processing errors
DLQ messages
Processing latency
Elasticsearch errors
Event ingestion rate
```

---

# 35. Example Security Pipeline Alerts

### Queue backlog

```text
security_queue_depth > threshold
```

### DLQ messages

```text
security_dlq_messages > 0
```

### Processor failure

```text
security_processor_errors increasing
```

### Elasticsearch failure

```text
security_events_indexing_errors increasing
```

These should be operational alerts.

---

# 36. Security Event Deduplication

Security services may generate related events.

Without deduplication:

```text
1 incident
   │
   ├── Event 1
   ├── Event 2
   ├── Event 3
   ├── Event 4
   └── Event 5
```

could become:

```text
5 separate pages
```

We need grouping/correlation.

Possible event identity:

```text
event.id
finding.id
resource
principal
time window
```

---

# 37. Correlation Example

Suppose:

```text
GuardDuty finding
+
same IAM role
+
same EC2 instance
+
same 10-minute window
```

The system can group these for investigation.

Do not automatically collapse unrelated events merely because they occurred close together.

---

# 38. GuardDuty → Security Hub → EventBridge

A common AWS-native pattern is:

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
Automation / Notification
```

This keeps AWS security findings centralized before downstream processing.

---

# 39. Security Hub → EventBridge

EventBridge can be configured to react to security findings.

Conceptual flow:

```text
Security Hub finding
        │
        ▼
EventBridge rule
        │
        ├── severity filter
        ├── account filter
        └── environment filter
        │
        ▼
SQS / SNS / automation
```

---

# 40. Example Filtering

We don't necessarily want:

```text
EVERY informational finding
```

to page the security team.

Instead:

```text
Critical
   │
   └── Immediate incident

High
   │
   └── Security queue / on-call according to policy

Medium
   │
   └── Investigation queue

Low
   │
   └── Dashboard/report
```

Actual thresholds should be defined by the organization's incident policy.

---

# 41. Multi-Account Security Events

In enterprise AWS:

```text
Dev Account
QA Account
Staging Account
Production Account
```

all generate security findings.

Central architecture:

```text
                 AWS Organization
                       │
       ┌───────────────┼────────────────┐
       │               │                │
      Dev              QA              Prod
       │               │                │
       └───────────────┼────────────────┘
                       ▼
              Central Security Account
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
         GuardDuty  Sec Hub   Config
                       │
                       ▼
                  EventBridge
                       │
                       ▼
                 Security Team
```

This reduces the need for security responders to inspect every account independently.

---

# 42. Log Archive Account

A dedicated log archive account can provide stronger separation.

```text
Production
    │
    ├── CloudTrail
    └── Security logs
          │
          ▼
     Log Archive
          │
          ▼
          S3
```

Application teams shouldn't normally have delete permissions over the central security archive.

---

# 43. Encryption

Security event storage should use encryption.

```text
Security Events
      │
      ▼
SQS / S3 / Elasticsearch
      │
      ▼
Encryption
```

For AWS resources, use AWS-managed encryption or customer-managed KMS keys according to security requirements.

---

# 44. Access Control

Security data can reveal:

* user identities
* source IPs
* infrastructure details
* API actions
* Kubernetes identities
* resource names
* security findings

Therefore:

```text
Developer
   │
   └── Limited operational visibility

DevOps
   │
   └── Platform/security operational access

Security Team
   │
   └── Security investigation access

Security Admin
   │
   └── Controlled administrative access
```

Use least privilege.

---

# 45. Don't Put Secrets in Security Logs

A major rule:

```text
NO
```

to:

```text
Passwords
API keys
AWS credentials
Private keys
Database credentials
Secret values
```

inside:

* logs
* traces
* security events
* Kibana documents
* CloudWatch messages
* Jira tickets

Only record metadata required for investigation.

---

# 46. Sensitive Kubernetes Audit Data

Kubernetes audit logs can contain sensitive information depending on what is logged and how the API request is configured.

Therefore:

```text
Audit logging
      │
      ▼
Review sensitivity
      │
      ▼
Access control
      │
      ▼
Retention policy
```

Don't treat audit logs as ordinary application logs.

---

# 47. SIEM Integration

Our ELK platform can function as a security investigation/search platform.

A dedicated enterprise SIEM may also be used.

Examples of SIEM platforms include:

* Splunk
* Microsoft Sentinel
* IBM QRadar
* Elastic Security

The architecture principle remains:

```text
AWS Security Signals
        │
        ▼
Central Security Event Pipeline
        │
        ▼
SIEM
        │
        ▼
Security Operations
```

The exact SIEM depends on company architecture.

---

# 48. Don't Duplicate Detection Everywhere

Avoid:

```text
GuardDuty detection
+
SIEM detection
+
Lambda detection
+
Prometheus detection
+
custom script detection
```

for the same exact condition unless there is a reason.

Prefer:

```text
AWS-native detection
        │
        ▼
Central event processing
        │
        ▼
SIEM investigation
        │
        ▼
Incident response
```

Then add custom detections where native coverage isn't sufficient.

---

# 49. Custom Detection Example

Suppose our company policy is:

> Production security group changes must have an approved change reference.

We can create a custom workflow:

```text
CloudTrail
   │
   ▼
Security Group Change
   │
   ▼
Check change metadata
   │
   ├── Approved → record
   │
   └── Not approved
           │
           ▼
        Security Alert
```

This is a business-specific security control.

---

# 50. Detection Rule Lifecycle

Every custom detection should follow:

```text
Define
  ↓
Test
  ↓
Tune
  ↓
Deploy
  ↓
Monitor false positives
  ↓
Review
```

Don't immediately enforce a rule that has never been tested.

---

# 51. False Positives

Security monitoring produces false positives.

Example:

```text
Alert:
Production security group modified
```

But:

```text
Jira CHG-10452
```

shows an approved deployment.

Therefore:

```text
Alert ≠ Confirmed Incident
```

Instead:

```text
Alert
  ↓
Triage
  ↓
Context
  ↓
Investigation
  ↓
Incident or expected activity
```

---

# 52. Security Investigation Runbook

For a high-severity security event:

### Step 1

Identify:

```text
What happened?
```

### Step 2

Identify:

```text
When?
```

### Step 3

Identify:

```text
Which account?
```

### Step 4

Identify:

```text
Which resource?
```

### Step 5

Identify:

```text
Which identity?
```

### Step 6

Check:

```text
CloudTrail
```

### Step 7

Check:

```text
GuardDuty / Security Hub
```

### Step 8

Check:

```text
VPC Flow Logs
```

### Step 9

Check:

```text
EKS audit logs
```

### Step 10

Check:

```text
Application logs
```

### Step 11

Check:

```text
GitHub / GitOps / ArgoCD
```

### Step 12

Check:

```text
Jira change
```

---

# 53. Complete Investigation Example

Imagine an unexpected production deployment.

### Security event

```text
Deployment changed
```

### Kubernetes

```text
Deployment
   │
   ▼
Pod
   │
   ▼
Image digest
```

### GitOps

```text
GitOps commit
```

### GitHub

```text
PR
```

### CI

```text
Source commit
```

### ECR

```text
Image digest
```

### Jira

```text
Change request
```

Now the investigator has:

```text
Who
 ↓
What
 ↓
When
 ↓
Why
 ↓
Which artifact
 ↓
Which deployment
```

That is the level of traceability we want.

---

# 54. Security Monitoring Terraform

Our infrastructure repository can eventually contain:

```text
terraform/
├── modules/
│   ├── cloudtrail/
│   ├── guardduty/
│   ├── securityhub/
│   ├── config/
│   ├── flow-logs/
│   ├── eventbridge/
│   ├── security-s3/
│   └── security-sqs/
│
└── environments/
    └── prod/
        └── security/
```

---

# 55. Security GitOps

Kubernetes security components belong in:

```text
platform/
└── security/
    ├── kyverno/
    ├── policies/
    ├── security-monitoring/
    └── README.md
```

For example:

```text
platform/security/
├── kyverno/
├── network-policies/
├── security-context/
└── monitoring/
```

---

# 56. Production Deployment Flow

Security changes themselves follow controlled deployment.

```text
Developer
    │
    ▼
GitHub PR
    │
    ▼
CI
    │
    ├── Tests
    ├── Lint
    └── Security checks
    │
    ▼
Review
    │
    ▼
GitOps
    │
    ▼
ArgoCD
    │
    ▼
EKS
```

For AWS security infrastructure:

```text
Terraform PR
    │
    ▼
terraform plan
    │
    ▼
Security review
    │
    ▼
Approval
    │
    ▼
Terraform apply
```

---

# 57. Security Monitoring Testing

We need to test that alerts actually work.

Do not wait for a real attack.

Create controlled test scenarios.

Examples:

```text
Test 1
Security group test modification

Test 2
Controlled IAM policy modification

Test 3
Test Kubernetes audit event

Test 4
Test EventBridge rule

Test 5
Send test message to security queue

Test 6
Force processor failure

Test 7
Verify DLQ

Test 8
Verify security dashboard
```

Only perform these tests in an authorized test environment unless an approved production exercise exists.

---

# 58. Test Event Flow

For a controlled test:

```text
Test Security Event
       │
       ▼
EventBridge
       │
       ▼
SQS
       │
       ▼
Processor
       │
       ▼
Elasticsearch
       │
       ▼
Kibana
       │
       ▼
Alert
```

Verify every stage.

---

# 59. Security Pipeline Failure Scenarios

### Scenario 1

```text
EventBridge DOWN / misconfigured
```

Investigate:

```text
AWS event delivery
```

### Scenario 2

```text
SQS backlog increases
```

Investigate:

```text
Processor
```

### Scenario 3

```text
DLQ contains messages
```

Investigate:

```text
Parser / processor
```

### Scenario 4

```text
Elasticsearch unavailable
```

Investigate:

```text
Elasticsearch cluster
```

while SQS buffers events.

---

# 60. Security Monitoring Architecture — Final

```text
                           AWS
                            │
     ┌──────────────────────┼────────────────────────┐
     │                      │                        │
 CloudTrail              GuardDuty              AWS Config
     │                      │                        │
     │                 Security Hub                  │
     │                      │                        │
     └──────────────────────┼────────────────────────┘
                            │
                       EventBridge
                            │
                            ▼
                       Security SQS
                            │
                            ▼
                  Security Event Processor
                            │
                 ┌──────────┼──────────┐
                 │          │          │
                 ▼          ▼          ▼
                S3         ELK       Alerting
                 │          │          │
                 │          ▼          ▼
                 │       Kibana      SNS/Security
                 │                    Team
                 │
                 ▼
             Long-term
              Archive
```

---

# 61. Integration With the Complete Platform

Our final platform now looks like:

```text
                         USERS
                           │
                           ▼
                  Route53 / CloudFront
                           │
                       WAF / ALB
                           │
                           ▼
                         EKS
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  21 Microservices      Redis             Messaging
        │                  │             SQS/Rabbit/Kafka
        │                  │                  │
        ▼                  ▼                  ▼
       RDS                Cache             Events


                  OBSERVABILITY
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
    Prometheus         Fluent Bit         OTel
        │                │                 │
      Grafana          Logstash            │
                         │                 │
                    Elasticsearch          │
                         │                 │
                       Kibana              Jaeger


                  SECURITY
                         │
        ┌────────────────┼─────────────────────┐
        │                │                     │
    CloudTrail        GuardDuty             Config
        │                │                     │
        ├──────────── Security Hub ────────────┤
        │                                      │
    Flow Logs                           EKS Audit Logs
        │                                      │
        └────────────── EventBridge ───────────┘
                              │
                             SQS
                              │
                     Security Processor
                              │
                  ┌───────────┼───────────┐
                  │           │           │
                 S3          ELK        Alerts
                  │           │           │
              Archive       Kibana      Jira/SOC
```

---

# 62. What We Have Achieved

At this point, the project has three distinct layers.

### Layer 1 — Prevent

```text
IAM
RBAC
Security Groups
NetworkPolicy
Kyverno
Pod Security
Image Verification
```

### Layer 2 — Detect

```text
CloudTrail
GuardDuty
Security Hub
AWS Config
IAM Access Analyzer
VPC Flow Logs
EKS Audit Logs
```

### Layer 3 — Investigate and Respond

```text
SQS
EventBridge
ELK
Kibana
Alerting
Jira
Incident Response
```

Together:

```text
PREVENT
   ↓
DETECT
   ↓
INVESTIGATE
   ↓
RESPOND
   ↓
RECOVER
   ↓
IMPROVE
```

---

# 63. Production Checklist

### AWS Security

* [ ] CloudTrail configured
* [ ] Central logging strategy
* [ ] GuardDuty enabled
* [ ] Security Hub enabled
* [ ] Config enabled where required
* [ ] IAM Access Analyzer enabled
* [ ] VPC Flow Logs configured
* [ ] EKS control-plane logs enabled as required
* [ ] EventBridge security rules configured

### Event Pipeline

* [ ] Security SQS queue
* [ ] DLQ
* [ ] Encryption
* [ ] Event processor
* [ ] Event normalization
* [ ] Event enrichment
* [ ] Deduplication/correlation
* [ ] Failure monitoring

### ELK

* [ ] Security data separated
* [ ] Security index/data stream strategy
* [ ] Kibana security dashboards
* [ ] Investigation queries
* [ ] Lifecycle/retention
* [ ] RBAC
* [ ] Encryption

### Incident Response

* [ ] Security severity model
* [ ] Alert routing
* [ ] Security runbooks
* [ ] Jira process
* [ ] Change correlation
* [ ] Incident ownership
* [ ] Security exercises

---

# 64. Final Mental Model

The most important thing to remember is:

```text
              AWS / EKS
                  │
                  ▼
            Security Signals
                  │
                  ▼
       ┌──────────────────────┐
       │ Detection Layer      │
       │                      │
       │ GuardDuty            │
       │ CloudTrail           │
       │ Config               │
       │ Security Hub         │
       │ EKS Audit            │
       │ Flow Logs            │
       └──────────┬───────────┘
                  │
                  ▼
             EventBridge
                  │
                  ▼
                SQS
                  │
                  ▼
        Security Event Pipeline
                  │
          ┌───────┴────────┐
          ▼                ▼
        ELK               S3
          │
          ▼
       Kibana
          │
          ▼
    Investigation
          │
          ▼
       Response
          │
          ▼
        Jira/SOC
```

The key production principle is:

> **AWS-native services perform security detection; the event pipeline transports and enriches findings; ELK/SIEM provides investigation; EventBridge/SNS/SQS provides routing; Jira/incident management tracks the response.**

This keeps each platform responsible for the job it is designed to perform rather than turning one tool into the entire security stack.
