# 50 — Incident Response

> **Enterprise E-Commerce Platform — AWS EKS Production Guide**

Incident response is different from troubleshooting.

**Troubleshooting** asks:

> “What is broken and why?”

**Incident response** asks:

> “How do we coordinate people, reduce customer impact, restore service safely, and prevent recurrence?”

A strong production engineer needs both skills.

---

# 1. Incident Response Lifecycle

Our production incident lifecycle:

```text
Alert
  ↓
Detection
  ↓
Triage
  ↓
Incident Declaration
  ↓
Assign Roles
  ↓
Assess Blast Radius
  ↓
Mitigate
  ↓
Recover
  ↓
Verify
  ↓
Communicate Resolution
  ↓
Root Cause Analysis
  ↓
Postmortem
  ↓
Preventive Actions
```

The key distinction:

```text
Mitigation ≠ Root Cause Fix
```

During an outage, restoring service safely can come before fully understanding the underlying cause.

---

# 2. Example Incident

Suppose customers report:

```text
"Checkout is failing."
```

Monitoring shows:

```text
POST /checkout

5xx:
0.5% → 18%

p95:
800ms → 7s
```

Alertmanager sends:

```text
CRITICAL:
Checkout error rate > threshold
```

This is now an incident.

---

# 3. Incident Detection

Incidents can originate from:

```text
Prometheus
Alertmanager
Grafana
CloudWatch
AWS health events
Application monitoring
Customer support
Developer
SRE
Business team
```

Important:

> **Customers can detect incidents before monitoring does.**

Therefore monitoring should not be considered the only source of truth.

---

# 4. Incident Severity

A company should define formal severity levels.

Example:

| Severity | Example                        |
| -------- | ------------------------------ |
| SEV-1    | Major customer-facing outage   |
| SEV-2    | Significant degradation        |
| SEV-3    | Limited functionality affected |
| SEV-4    | Minor operational issue        |

The exact definitions, response times, and escalation rules should be organization-specific.

---

# 5. SEV-1 Example

Imagine:

```text
Checkout
Payment
Order creation
```

are all unavailable.

Impact:

```text
Customers cannot complete purchases.
```

Response should immediately prioritize:

```text
Customer impact
+
Service restoration
+
Clear ownership
+
Communication
```

Do not spend the first 30 minutes trying to write the perfect root-cause explanation.

---

# 6. Incident Roles

A mature incident response process separates responsibilities.

Typical roles:

```text
Incident Commander
Technical Lead
Communications Lead
Operations/Scribe
Subject Matter Experts
```

---

# 7. Incident Commander

The Incident Commander coordinates the incident.

Responsibilities:

```text
Assess situation
Assign owners
Set priorities
Coordinate teams
Prevent duplicate work
Approve major decisions
Maintain focus on recovery
```

The Incident Commander does **not** need to personally run every command.

---

# 8. Technical Lead

The Technical Lead coordinates technical investigation.

Example:

```text
Technical Lead
     |
     +-- EKS investigation
     +-- Application investigation
     +-- Database investigation
     +-- Network investigation
     +-- AWS investigation
```

Different engineers can own different branches.

---

# 9. Communications Lead

Someone should own communication.

Internal:

```text
Engineering
SRE
Management
Support
Product
Security
```

External, when required:

```text
Status page
Customer communication
Support teams
```

The technical team should not simultaneously investigate an incident and manage every communication channel.

---

# 10. Scribe

Record:

```text
Time
Observation
Command/result
Decision
Action
Owner
```

Example:

```text
10:05 Alert fired

10:07 Checkout 5xx confirmed

10:10 Payment-service deployment identified

10:13 Rollback approved

10:16 Error rate decreasing

10:20 Service recovered
```

This becomes extremely valuable during the postmortem.

---

# 11. Incident Channel

Create a dedicated incident communication channel.

Example:

```text
#incident-2026-09-18-checkout
```

The exact platform may be Slack, Teams, PagerDuty-integrated tooling, or another approved system.

Keep the incident channel focused.

Avoid unrelated conversations.

---

# 12. Incident Declaration

Don't wait for absolute certainty.

You can declare:

```text
Potential SEV-1
```

while investigation continues.

Example:

```text
Incident:
Checkout failures affecting production customers.

Start:
10:05 IST

Current impact:
Approximately 18% checkout requests returning 5xx.

Incident Commander:
<person>

Technical Lead:
<person>

Status:
Investigating.
```

---

# 13. Establish the Blast Radius

Ask:

```text
Which customers?
Which region?
Which environment?
Which service?
Which API?
Which AZ?
Which dependency?
Since when?
```

Example:

```text
Production:
YES

QA:
NO

Checkout:
YES

Catalog:
NO

Payment:
YES
```

This tells us:

```text
Likely checkout dependency path
```

rather than:

```text
Entire platform failure
```

---

# 14. Build a Timeline

Create a timeline immediately.

Example:

```text
09:55
Payment deployment started

10:00
Deployment completed

10:03
Latency begins increasing

10:05
Alert fires

10:06
Customer reports received

10:08
Incident declared

10:12
New deployment identified as likely contributor

10:15
Rollback approved

10:18
Error rate falling

10:22
Service stable

10:30
Incident resolved
```

This is much more useful than relying on memory later.

---

# 15. Check Recent Changes

During incidents, investigate:

```text
Application deployment
Configuration change
Secret rotation
Database migration
Terraform change
ArgoCD change
Ingress change
NetworkPolicy change
IAM change
AWS infrastructure change
```

A recent change is a clue, not automatically the root cause.

---

# 16. Change Correlation

Example:

```text
Deployment:
10:00

Error rate:
10:03 ↑
```

This is a strong correlation.

Now compare:

```text
Previous image
New image
Environment variables
Secrets
ConfigMaps
Database schema
Dependencies
```

---

# 17. Mitigation First

Suppose the latest deployment caused:

```text
18% checkout failures
```

Potential mitigation:

```text
Rollback
```

The immediate objective is:

```text
Customer impact ↓
```

not:

```text
Perfect root cause explanation
```

---

# 18. Rollback Through GitOps

Our platform uses GitOps.

Preferred flow:

```text
Bad GitOps commit
       ↓
Identify previous known-good version
       ↓
Revert GitOps change
       ↓
Review/approval according to policy
       ↓
ArgoCD sync
       ↓
EKS
```

Avoid making undocumented production changes directly with:

```bash
kubectl edit
```

unless the incident process explicitly authorizes emergency intervention.

If emergency manual intervention occurs:

```text
Document it
+
Reconcile Git
```

afterward.

---

# 19. Rollback Is Not Always Safe

Consider:

```text
Application v2
    ↓
Database migration
```

If migration is destructive:

```text
v2 → v1
```

may not work.

Therefore before rollback:

```text
Check DB compatibility
Check schema changes
Check configuration
Check external API compatibility
```

This is why expand-and-contract migrations matter.

---

# 20. Scale as Mitigation

Suppose:

```text
Traffic:
3x normal

Pods:
3
```

HPA may increase:

```text
3 → 10 Pods
```

But ask:

```text
Can RDS handle 10 Pods?
Can Redis handle them?
Can the external API handle them?
Can the node capacity handle them?
```

Scaling the application can overload its dependencies.

---

# 21. Example Retry Storm

External payment API becomes slow.

Payment service:

```text
request
 ↓
timeout
 ↓
retry immediately
 ↓
timeout
 ↓
retry
```

Now traffic becomes:

```text
Original:
1,000 requests

Retries:
3,000 additional requests
```

This makes the external service even worse.

Use:

```text
bounded retries
+
exponential backoff
+
jitter
+
timeouts
+
circuit breaking where appropriate
```

---

# 22. Fail Fast

A dependency timeout should not consume application threads indefinitely.

Example:

```text
Payment API timeout:
30 seconds
```

If 500 requests are waiting:

```text
500 × 30 seconds
```

can exhaust resources.

Use carefully designed:

```text
connect timeout
read timeout
overall request timeout
retry timeout budget
```

---

# 23. Dependency Failure

Our order flow may look like:

```text
Client
 ↓
Order Service
 ↓
Inventory Service
 ↓
Payment Service
 ↓
External Payment API
```

If the payment API fails:

```text
Order Service
```

may still be healthy.

But:

```text
Business transaction
```

is failing.

This is why service-level health and business-level health must both be monitored.

---

# 24. Business Metrics

Technical metrics:

```text
CPU
Memory
Latency
5xx
Pod count
```

Business metrics:

```text
orders created
payments completed
checkout success rate
cart conversion
failed transactions
refund failures
```

Example:

```text
CPU:
40%

Pods:
Healthy

5xx:
Normal

Checkout success:
↓ 35%
```

Something is wrong even though infrastructure appears healthy.

---

# 25. Incident Communication

Good communication is:

```text
What happened?
What is affected?
What are we doing?
What is the current status?
When is the next update?
```

Example:

```text
10:15 IST

Checkout failures are affecting production customers.

The team has identified elevated failures in the payment
processing path and is investigating the latest deployment.

Customer impact:
Checkout completion is degraded.

Next update:
10:30 IST.
```

Avoid speculation.

Don't write:

```text
AWS is broken.
```

until evidence supports it.

---

# 26. Status Updates

During an ongoing incident:

```text
Investigating
    ↓
Identified
    ↓
Mitigating
    ↓
Recovering
    ↓
Monitoring
    ↓
Resolved
```

Each update should contain facts currently known.

---

# 27. Avoid Noise

Bad incident channel:

```text
Engineer A:
Maybe database?

Engineer B:
Maybe Kubernetes?

Engineer C:
Could be AWS.

Engineer D:
I think someone changed something.
```

Better:

```text
10:12
RDS metrics normal.

10:13
EKS nodes healthy.

10:14
Payment-service 5xx increased after deployment 2026.09.18.3.

10:15
Technical Lead investigating deployment.
```

Evidence beats speculation.

---

# 28. Observability During Incidents

Use all three signals:

```text
Metrics
Logs
Traces
```

### Metrics

Tell you:

```text
WHAT is changing?
```

### Logs

Tell you:

```text
WHAT happened?
```

### Traces

Tell you:

```text
WHERE did the request spend time/fail?
```

Combined:

```text
Prometheus
    +
ELK
    +
Jaeger
```

gives much stronger diagnosis.

---

# 29. Example: Metrics → Logs → Traces

Prometheus:

```text
payment-service 5xx = 20%
```

Kibana:

```text
Timeout calling payment-provider
```

Jaeger:

```text
payment-provider span = 7.2 seconds
```

Conclusion:

```text
Customer request
      ↓
payment-service
      ↓
external payment provider
      ↓
latency/timeout
```

Now the team has an evidence-based direction.

---

# 30. AWS Incident Investigation

When the platform appears affected, inspect:

```text
EKS
EC2
RDS
ElastiCache
ALB
NAT
VPC
SQS/SNS
CloudFront
WAF
Route 53
```

Also check AWS service health/status information relevant to the affected services and region.

---

# 31. Multi-AZ Incident

Suppose:

```text
AZ-A
   ↓
network problems
```

Our workloads should ideally have:

```text
AZ-A
AZ-B
AZ-C
```

with topology-aware scheduling.

Check:

```bash
kubectl get pods -n production -o wide
```

Then determine where replicas are running.

Example:

```text
order-service:

AZ-A → 1
AZ-B → 1
AZ-C → 1
```

This provides better resilience than:

```text
AZ-A → 3
AZ-B → 0
AZ-C → 0
```

---

# 32. Node Failure During Incident

Suppose:

```text
Node-A
 ↓
NotReady
```

Pods should be rescheduled where capacity exists.

Check:

```bash
kubectl get nodes
kubectl get pods -n production -o wide
```

Then verify:

```text
replicas
availability
topology
PDB
node capacity
```

Don't manually recreate every Pod unless there is a reason.

Kubernetes controllers normally handle Pod replacement.

---

# 33. PDB During Incidents

PodDisruptionBudget protects against **voluntary disruptions**.

Example:

```yaml
minAvailable: 2
```

for:

```text
3 replicas
```

This helps during:

```text
node drain
maintenance
voluntary disruptions
```

PDB does not magically protect against every failure.

For example:

```text
Node suddenly disappears
```

can still remove its Pods.

---

# 34. Database Incident

If RDS is unhealthy:

```text
Don't immediately restart application Pods.
```

First determine:

```text
RDS status
CPU
memory
connections
storage
IOPS
latency
locks
failover status
```

Then decide whether the problem is:

```text
Database
Application
Connection pool
Network
Query
Capacity
```

---

# 35. Database Connection Storm

Suppose:

```text
3 Pods
×
20 connections
=
60
```

Then HPA:

```text
3 → 15 Pods
```

Potential:

```text
15 × 20
=
300 connections
```

RDS becomes saturated.

Now:

```text
RDS
 ↓
connection errors
 ↓
application failures
 ↓
retries
 ↓
more connections
```

This can become a feedback loop.

---

# 36. Break the Feedback Loop

Possible mitigations:

```text
Stop uncontrolled scaling
Reduce retry rate
Tune connection pools
Protect database
Fix expensive queries
Use caching where appropriate
Scale DB if justified
```

Do not simply keep increasing:

```text
maxReplicas
```

---

# 37. Queue-Based Incident

Suppose:

```text
SQS:
AgeOfOldestMessage ↑
```

Check:

```text
consumer health
consumer replicas
processing duration
application errors
downstream dependencies
DLQ
```

If consumers are healthy but processing is slow:

```text
Trace one message processing path.
```

---

# 38. Incident Mitigation Options

Depending on the incident:

```text
Rollback
Scale
Failover
Disable feature
Rate limit
Reduce traffic
Stop a problematic consumer
Pause non-critical workloads
Switch dependency
Enable maintenance mode
Restore database
Fail over region
```

Every action should consider:

```text
blast radius
reversibility
customer impact
data integrity
security
```

---

# 39. Feature Flags

Feature flags can be useful during incidents.

Example:

```text
Recommendation Engine
```

fails repeatedly.

Instead of taking down the entire application:

```text
Recommendation
     ↓
disable feature
     ↓
Checkout continues
```

This is a powerful resilience technique.

Feature flag implementation should be designed and governed carefully.

---

# 40. Rate Limiting

Suppose an API receives:

```text
100,000 requests/min
```

while normal capacity is:

```text
20,000 requests/min
```

Without protection:

```text
CPU ↑
Pods ↑
RDS ↑
Redis ↑
latency ↑
```

Rate limiting can protect the platform.

Possible locations:

```text
CloudFront/WAF
ALB/application layer
API Gateway where used
service-level controls
```

---

# 41. Protect Critical Dependencies

Priority:

```text
Critical
  ↓
Order
Payment
Inventory
```

Non-critical:

```text
Recommendation
Analytics
Reporting
```

During resource pressure, non-critical workloads may be scaled or deprioritized according to business policy.

Kubernetes tools that can help include:

```text
PriorityClass
ResourceQuota
LimitRange
separate node pools
```

---

# 42. Incident Recovery

Recovery is not:

```text
Error rate = 0
```

Immediately declare success.

Verify:

```text
5xx
latency
throughput
Pod health
database
queues
external dependencies
business metrics
customer reports
```

Example:

```text
Technical:
5xx = normal

Business:
checkout success = normal
```

Now recovery is much more credible.

---

# 43. Monitoring Period

After mitigation:

```text
Don't immediately close the incident.
```

Enter:

```text
Monitoring
```

Watch for:

```text
15–30 minutes
```

or an organization-defined stabilization period appropriate to the incident.

Check:

```text
error rate
latency
resource utilization
queue depth
database
logs
traces
```

---

# 44. Incident Resolution

Example:

```text
Incident resolved.

Checkout error rate returned to baseline.

Payment-service latency returned to normal.

No queue backlog remains.

No additional customer impact observed.

The incident is now closed.
```

Then:

```text
Postmortem
```

begins.

---

# 45. Root Cause Analysis

A good RCA answers:

```text
What happened?
Why did it happen?
Why wasn't it detected earlier?
Why did safeguards fail?
What reduced the impact?
How will we prevent recurrence?
```

---

# 46. Five Whys

Example:

### Why did checkout fail?

Payment service returned 5xx.

### Why?

Payment provider requests timed out.

### Why?

New timeout configuration was too aggressive.

### Why wasn't it detected?

No deployment validation tested realistic provider latency.

### Why?

Performance test environment didn't model the dependency correctly.

Now preventive action becomes:

```text
Improve dependency testing
+
deployment validation
+
timeout monitoring
```

rather than:

```text
"Fix payment service."
```

---

# 47. Root Cause vs Contributing Factors

Example:

```text
Root cause:
Bad timeout configuration.

Contributing factor:
Insufficient integration testing.

Contributing factor:
No alert on payment-provider latency.

Contributing factor:
Rollback process was manual.
```

This produces better corrective actions.

---

# 48. Blameless Postmortem

Focus on:

```text
System
Process
Controls
Architecture
Detection
Recovery
```

rather than:

```text
Who made the mistake?
```

Example:

Bad:

```text
Engineer changed the wrong configuration.
```

Better:

```text
The deployment process allowed an incompatible
timeout configuration to reach production without
the required validation.
```

The objective is to improve the system.

---

# 49. Postmortem Template

```markdown
# Incident: Checkout Failure

## Date

2026-09-18

## Severity

SEV-1

## Duration

10:05 - 10:30 IST

## Impact

Customers experienced checkout failures.

## Detection

Prometheus/Alertmanager alert.

## Timeline

10:05 - Alert fired
10:08 - Incident declared
10:12 - Cause narrowed to payment path
10:15 - Rollback started
10:20 - Error rate normalized
10:30 - Incident resolved

## Root Cause

<documented root cause>

## Contributing Factors

- ...
- ...

## Mitigation

<what restored service>

## Customer Impact

<measured impact>

## What Went Well

- ...
- ...

## What Went Poorly

- ...
- ...

## Action Items

- [ ] ...
- [ ] ...
- [ ] ...

## Owners

- ...
- ...

## Due Dates

- ...
```

---

# 50. Action Items Must Be Concrete

Bad:

```text
[ ] Improve monitoring
```

Good:

```text
[ ] Add alert for payment-provider p95 latency
    Owner: SRE
    Due: 2026-09-25
```

Better:

```text
[ ] Add automated deployment test that injects
    3-second payment-provider latency and verifies
    timeout/retry behavior.
```

---

# 51. Preventive Controls

After an incident, possible improvements include:

```text
Better tests
Better alerts
Better dashboards
Better timeout configuration
Circuit breakers
Retry controls
Capacity planning
PodDisruptionBudgets
Topology spreading
Canary deployments
Automated rollback
Database migration checks
Runbooks
DR exercises
Security controls
```

---

# 52. Incident Runbooks

Create runbooks for common failures.

Example:

```text
runbooks/
├── eks-node-notready.md
├── pod-crashloop.md
├── rds-connection-failure.md
├── redis-failure.md
├── sqs-backlog.md
├── high-api-latency.md
├── payment-provider-outage.md
├── elasticsearch-red.md
└── argocd-sync-failure.md
```

Each runbook should contain:

```text
Symptoms
Checks
Commands
Decision points
Safe actions
Rollback
Escalation
Verification
```

---

# 53. Example Runbook — Pod CrashLoopBackOff

```markdown
# Pod CrashLoopBackOff

## 1. Identify Pod

kubectl get pods -n production

## 2. Describe

kubectl describe pod POD -n production

## 3. Previous Logs

kubectl logs POD -n production --previous

## 4. Check

- Secret
- ConfigMap
- DB connectivity
- Redis
- image
- probes
- memory

## 5. Remediate

Apply approved fix.

## 6. Verify

kubectl rollout status deployment/SERVICE -n production

## 7. Monitor

Check metrics/logs/traces.
```

---

# 54. Game Days

Production teams should practice incidents before real incidents occur.

Example:

```text
Game Day
   |
   +-- Kill application Pod
   +-- Drain worker node
   +-- Simulate RDS failure
   +-- Increase queue backlog
   +-- Simulate Redis outage
   +-- Simulate dependency latency
   +-- Test ArgoCD rollback
```

Do this in a controlled environment and according to an approved operational plan.

---

# 55. Disaster vs Incident

An incident:

```text
Single service failure
```

A disaster:

```text
Region unavailable
Major data corruption
Complete production environment loss
```

Incident response:

```text
restore service
```

DR response:

```text
restore the platform/environment
```

Our DR process is covered in:

```text
45-Disaster-Recovery.md
```

---

# 56. Security Incident

If an incident involves:

```text
credential leak
compromised Pod
suspicious IAM activity
malicious image
data exposure
```

don't treat it as a normal application outage.

Immediately involve:

```text
Security Team
Incident Commander
Cloud/Platform Team
Application Team
```

Potential actions:

```text
isolate workload
revoke/rotate credentials
restrict network
preserve evidence
review CloudTrail
review Kubernetes audit logs
identify affected resources
```

Do not destroy evidence unnecessarily.

---

# 57. Incident Command Structure

A practical model:

```text
                 Incident Commander
                         |
          +--------------+--------------+
          |              |              |
     Technical       Communications    Scribe
        Lead              Lead
          |
   +------+------+------+
   |      |      |      |
  EKS    App    DB    Network
```

This prevents:

```text
10 engineers
+
10 people doing the same investigation
```

---

# 58. Production Incident Flow

Our complete model:

```text
Alert
  |
  v
Confirm
  |
  v
Determine Severity
  |
  v
Declare Incident
  |
  v
Assign Roles
  |
  v
Determine Blast Radius
  |
  v
Check Recent Changes
  |
  v
Metrics + Logs + Traces
  |
  v
Identify Likely Cause
  |
  v
Mitigate
  |
  v
Verify Recovery
  |
  v
Monitor
  |
  v
Resolve
  |
  v
RCA
  |
  v
Postmortem
  |
  v
Preventive Actions
```

---

# 59. Production Incident Checklist

```text
[ ] Confirm alert
[ ] Determine customer impact
[ ] Determine blast radius
[ ] Assign Incident Commander
[ ] Assign Technical Lead
[ ] Assign Communications Lead
[ ] Start incident channel
[ ] Start timeline
[ ] Check recent changes
[ ] Check Grafana
[ ] Check Prometheus
[ ] Check logs
[ ] Check traces
[ ] Check EKS
[ ] Check AWS dependencies
[ ] Check RDS
[ ] Check Redis
[ ] Check queues
[ ] Choose mitigation
[ ] Evaluate rollback safety
[ ] Apply controlled remediation
[ ] Verify technical recovery
[ ] Verify business recovery
[ ] Monitor stabilization
[ ] Communicate resolution
[ ] Complete RCA
[ ] Create action items
[ ] Assign owners
[ ] Track action items
```

---

# 60. Final Mental Model

When production breaks:

```text
             INCIDENT
                 |
       +---------+---------+
       |         |         |
     PEOPLE    SYSTEM    DATA
       |         |         |
   IC/Teams   EKS/AWS   RDS/S3/etc.
       |         |
       +----+----+
            |
       OBSERVABILITY
            |
    +-------+-------+
    |       |       |
 Metrics  Logs    Traces
    |       |       |
Prometheus ELK   Jaeger
            |
        Evidence
            |
         Mitigate
            |
         Recover
            |
        Understand
            |
          Prevent
```

The most important production mindset is:

> **During an incident, stabilize first, investigate systematically, communicate clearly, preserve evidence, and only then optimize the system based on what you learned.**

---

**Next: `51-Production-Readiness.md`** — we'll take the entire platform and perform a **production readiness review** across AWS, EKS, Kubernetes, CI/CD, GitOps, security, databases, messaging, observability, DR, scaling, and operations before we begin the actual implementation.
