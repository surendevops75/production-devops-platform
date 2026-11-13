# 158 — ArgoCD Drift Production Incident

> **Phase 18 — Incident Management**  
> **Incident #158**  
> **Environment:** Production  
> **GitOps Platform:** ArgoCD  
> **Kubernetes Platform:** Amazon EKS  
> **Primary example:** `order-service`

---

# 1. Objective

This chapter explains how to troubleshoot a **GitOps/ArgoCD drift incident** in a production EKS environment.

The most important principle is:

> **In a GitOps environment, Git is the desired-state source of truth.**

The architecture is:

```text
Developer
    ↓
Git PR
    ↓
GitOps Repository
    ↓
ArgoCD
    ↓
Kubernetes API
    ↓
EKS
    ↓
Workloads
```

Drift occurs when:

```text
Git desired state
        ≠
Kubernetes live state
```

For example:

```text
Git:
replicas = 3

Kubernetes:
replicas = 8
```

ArgoCD detects:

```text
OUT OF SYNC
```

---

# 2. What Is Configuration Drift?

Configuration drift means the actual Kubernetes state differs from the state declared in Git.

Example:

```yaml
replicas: 3
```

Git says:

```text
3 replicas
```

But someone executes:

```bash
kubectl scale deployment order-service \
  --replicas=8 \
  -n ecommerce
```

Now:

```text
Git desired state = 3
Live state         = 8
```

ArgoCD detects the difference.

---

# 3. Production GitOps Architecture

Our production deployment flow:

```text
                    ┌─────────────────┐
                    │ Application Repo│
                    └────────┬────────┘
                             │
                             ▼
                           CI/CD
                             │
                             ▼
                           ECR
                             │
                             ▼
                    ┌─────────────────┐
                    │ GitOps Repository│
                    └────────┬────────┘
                             │
                             ▼
                          ArgoCD
                             │
                             ▼
                     Kubernetes API
                             │
                             ▼
                           EKS
```

ArgoCD continuously compares:

```text
Git desired state
        ↕
Kubernetes live state
```

---

# 4. Why Drift Matters

Drift is not always immediately customer-impacting.

However, it creates serious operational risk.

Example:

```text
Git:
HPA maxReplicas = 20

Live:
HPA maxReplicas = 50
```

Someone manually changed production.

Later:

```text
ArgoCD sync
```

may restore:

```text
50 → 20
```

This could suddenly reduce application capacity.

Another example:

```text
Git:
NetworkPolicy allows only required traffic

Live:
NetworkPolicy manually modified
```

A future synchronization could remove the manual security change.

---

# 5. Types of Drift

## 5.1 Manual Drift

Someone runs:

```bash
kubectl edit deployment
```

or:

```bash
kubectl scale
```

---

## 5.2 Emergency Drift

During an incident:

```text
Production issue
    ↓
Engineer changes replica count manually
    ↓
Incident resolved
```

But Git is not updated.

This creates emergency drift.

---

## 5.3 Controller-Generated Differences

Some Kubernetes resources are continuously modified by controllers.

Examples:

- status fields
- generated fields
- HPA-managed replica count
- Service fields
- admission webhook mutations

These need to be handled carefully.

Not every live-state difference should be treated as dangerous configuration drift.

---

## 5.4 Helm Rendering Drift

Git may contain:

```text
Helm values
```

while rendered Kubernetes manifests differ due to:

- chart version
- values
- defaults
- template changes

---

## 5.5 Environment Drift

Example:

```text
DEV:
replicas = 2

QA:
replicas = 3

PROD:
replicas = 5
```

Environment-specific values are expected.

The problem occurs when production has an undocumented manual change.

---

# 6. ArgoCD Application States

Common ArgoCD states include:

```text
SYNC STATUS
Synced
OutOfSync
Unknown
```

and:

```text
HEALTH
Healthy
Progressing
Degraded
Missing
Suspended
Unknown
```

Example:

```text
Application: order-service
Sync: OutOfSync
Health: Healthy
```

This means:

```text
Application is running
BUT
Live state differs from Git
```

This is different from:

```text
Sync: OutOfSync
Health: Degraded
```

which may indicate actual application impact.

---

# 7. First Incident Question

Do not immediately click:

```text
SYNC
```

Ask:

> **Why is ArgoCD reporting drift?**

The difference may be:

- intentional emergency change
- HPA-generated value
- controller-managed field
- unauthorized manual change
- Git configuration change
- Helm rendering difference
- Kubernetes default
- admission mutation
- actual configuration error

Blindly synchronizing can cause an outage.

---

# 8. Incident Detection

Example alert:

```text
ArgoCD Application OutOfSync
```

Possible alert:

```text
ArgoCD app:
order-service

Sync Status:
OutOfSync

Health:
Healthy
```

A useful production alert should include:

```text
Application
Environment
Namespace
Team
Severity
Runbook
```

---

# 9. First Investigation

Using ArgoCD CLI:

```bash
argocd app get order-service
```

Look for:

```text
Sync Status
Health Status
Repository
Revision
Namespace
Resources
```

Example:

```text
Name: order-service
Project: ecommerce
Server: https://kubernetes.default.svc
Namespace: ecommerce
Sync Status: OutOfSync
Health Status: Healthy
```

---

# 10. Check Application Details

```bash
argocd app get order-service \
  --show-operation
```

Depending on ArgoCD version, inspect the application history and current operation.

Also:

```bash
argocd app history order-service
```

This helps determine:

```text
Last deployment
Revision
Sync operation
Previous versions
```

---

# 11. Kubernetes Side

Check:

```bash
kubectl get deployment \
  order-service \
  -n ecommerce \
  -o yaml
```

Compare with the GitOps source.

Important fields:

```text
replicas
image
resources
environment
ports
probes
annotations
labels
securityContext
serviceAccount
```

---

# 12. Example Drift

Git:

```yaml
spec:
  replicas: 3
```

Live:

```yaml
spec:
  replicas: 8
```

ArgoCD reports:

```text
OUT OF SYNC
```

The immediate question is:

> Who changed replicas from 3 to 8 and why?

---

# 13. Check Recent Git Changes

Inspect:

```text
Git commit history
PR
author
timestamp
Helm values
Application manifest
```

Example:

```bash
git log --oneline --decorate -20
```

Look for:

```text
feat: increase order service capacity
fix: change order deployment
chore: update helm values
```

If Git says 3 replicas and live says 8, the change likely happened outside Git.

---

# 14. Check Kubernetes Audit Evidence

In a production EKS environment, Kubernetes API audit logging and AWS/EKS logging architecture should be configured according to organizational requirements.

The goal is to determine:

```text
Who?
What?
When?
From where?
```

For example:

```text
User:
devops-engineer

Action:
patch deployment/order-service

Time:
14:32 IST
```

This is much better than guessing.

---

# 15. AWS CloudTrail vs Kubernetes Audit Logs

These are different.

### CloudTrail

Useful for AWS API activity:

```text
EC2
EKS
IAM
RDS
ECR
S3
```

### Kubernetes audit logs

Useful for Kubernetes API activity:

```text
Deployment
Service
ConfigMap
Secret
HPA
Role
RoleBinding
```

A `kubectl scale` operation is primarily a Kubernetes API action.

---

# 16. Common Drift Scenario — Emergency Scaling

Suppose production experiences:

```text
Traffic spike
```

Engineer executes:

```bash
kubectl scale deployment order-service \
  --replicas=12 \
  -n ecommerce
```

Before:

```text
Git = 6
Live = 6
```

After:

```text
Git = 6
Live = 12
```

ArgoCD:

```text
OutOfSync
```

This is intentional emergency drift.

---

# 17. Correct Response

Do not immediately sync.

First:

```text
Incident
  ↓
Emergency scaling
  ↓
Customer impact resolved
  ↓
Create Git change
  ↓
PR
  ↓
Review
  ↓
GitOps updated
  ↓
ArgoCD becomes Synced
```

This converts:

```text
temporary manual change
```

into:

```text
controlled desired state
```

---

# 18. Dangerous Response

Bad process:

```text
Incident
  ↓
kubectl scale 12
  ↓
Problem fixed
  ↓
Forget about it
```

Later:

```text
ArgoCD sync
  ↓
12 → 6
```

Now the original incident may return.

---

# 19. Another Example — Manual Image Change

Git:

```yaml
image:
  repository: ...
  digest: sha256:ABC
```

Engineer changes live deployment:

```bash
kubectl set image deployment/order-service \
  order-service=...@sha256:XYZ \
  -n ecommerce
```

Now:

```text
Git digest = ABC
Live digest = XYZ
```

ArgoCD reports:

```text
OutOfSync
```

---

# 20. Security Concern

Manual production changes can bypass:

- code review
- security scanning
- approval
- change management
- audit process
- deployment testing

Therefore:

> Production Kubernetes should generally be changed through GitOps, except for controlled emergency response.

Emergency access should itself be:

- authenticated
- authorized
- audited
- time-limited where possible

---

# 21. ArgoCD Self-Heal

ArgoCD can automatically correct drift.

Conceptually:

```text
Git:
replicas = 3

Live:
replicas = 8

ArgoCD detects drift
        ↓
Self-heal
        ↓
Live → 3
```

This is powerful.

But it can also be dangerous if an emergency manual change is still needed.

Therefore teams must understand:

```text
automated reconciliation
```

before enabling it broadly.

---

# 22. Self-Heal Risk During Incidents

Imagine:

```text
RDS problem
```

Engineer temporarily reduces:

```text
order-service:
10 → 5 Pods
```

But Git says:

```text
10
```

ArgoCD self-heal may immediately restore:

```text
5 → 10
```

This could worsen the incident.

Therefore emergency changes need a controlled process.

Possible approaches include:

- change Git quickly
- controlled ArgoCD sync suspension where appropriate
- use an incident-approved Git change
- use temporary automation controls

Do not leave production permanently unsynchronized.

---

# 23. ArgoCD Sync vs Refresh

These are not the same.

Conceptually:

### Refresh

```text
ArgoCD checks:
Git
+
Live cluster
```

It updates ArgoCD's understanding.

### Sync

```text
ArgoCD applies desired Git state
to Kubernetes.
```

Therefore:

```text
OutOfSync
```

does not mean:

```text
Immediately Sync
```

Investigate first.

---

# 24. Compare Desired and Live State

ArgoCD UI/CLI can show differences.

Conceptually:

```text
Desired:

replicas: 3

Live:

replicas: 8
```

Other examples:

```text
image
resources
env
annotations
serviceAccount
NetworkPolicy
Ingress
ConfigMap
```

Use the diff to identify exactly what changed.

---

# 25. `argocd app diff`

Example:

```bash
argocd app diff order-service
```

Conceptual output:

```diff
spec:
  replicas:
-   3
+   8
```

This immediately tells us what drift exists.

---

# 26. Drift Due to HPA

A common mistake is to see:

```text
Deployment desired replicas:
3

Live:
8
```

and assume someone manually changed it.

But HPA may be responsible.

Example:

```text
Deployment:
spec.replicas = 3

HPA:
minReplicas = 3
maxReplicas = 10

Current:
8
```

The HPA is intentionally managing replica count.

Therefore ArgoCD should be configured to avoid treating HPA-controlled fields as undesirable drift.

---

# 27. Ignore Differences Carefully

ArgoCD supports mechanisms for ignoring specific differences.

Conceptually:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

This should be used carefully.

Do **not** ignore large portions of resources simply to make:

```text
OutOfSync
```

disappear.

Bad:

```text
ignore everything
```

Good:

```text
ignore only fields intentionally managed by another controller
```

---

# 28. Controller-Managed Fields

Examples that may legitimately change:

```text
HPA
Deployment controller
Service controller
Load Balancer Controller
cert-manager
External Secrets Operator
```

For example:

```text
ExternalSecret
    ↓
Secret generated
```

The Secret may be controller-managed.

GitOps should manage the source configuration, while the controller manages generated fields/resources as designed.

---

# 29. Drift in Secrets

Suppose:

```text
External Secrets Operator
```

manages:

```text
order-db-secret
```

ArgoCD may see differences in the generated Kubernetes Secret.

Do not blindly overwrite it.

Instead investigate:

```text
ExternalSecret
       ↓
Secrets Manager
       ↓
ESO
       ↓
Kubernetes Secret
```

The source of truth may be AWS Secrets Manager rather than the generated Secret.

---

# 30. Drift in ConfigMaps

Example:

```text
Git:
LOG_LEVEL=INFO
```

Live:

```text
LOG_LEVEL=DEBUG
```

Potential causes:

- manual edit
- Helm values
- environment overlay
- operator mutation

Determine the source before syncing.

---

# 31. Drift in NetworkPolicy

This is more serious.

Git:

```text
Allow:
order-service → payment-service
```

Live:

```text
Allow:
order-service → payment-service
order-service → reporting-service
```

Someone may have manually opened unnecessary traffic.

This should trigger:

```text
Security review
```

rather than simply ignoring the difference.

---

# 32. Drift in IAM / ServiceAccount

Example:

```text
Git:
order-service → ServiceAccount order-service
```

Live:

```text
ServiceAccount annotation changed
```

This can alter AWS permissions.

For example:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM role
```

or, depending on architecture, IRSA-related configuration.

A drifted identity configuration can create either:

```text
AccessDenied
```

or:

```text
Excessive permissions
```

---

# 33. Drift in Resource Limits

Git:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

Live:

```yaml
resources:
  limits:
    cpu: "4"
    memory: "4Gi"
```

This may allow a workload to consume much more cluster capacity.

During a traffic spike:

```text
Resource drift
   ↓
Unexpected resource consumption
   ↓
Node pressure
   ↓
Other workloads affected
```

---

# 34. Drift in Security Controls

Examples:

```text
NetworkPolicy
RBAC
ServiceAccount
PodSecurity
SecurityContext
Ingress
TLS
```

These should be treated as high-risk drift.

Security-related drift should generally trigger:

```text
Security investigation
+
change review
```

---

# 35. Production Drift Decision Tree

```text id="9l9p8w"
              ArgoCD OutOfSync
                     │
                     ▼
               Check diff
                     │
                     ▼
           Is difference expected?
              /             \
            Yes              No
             │                │
             ▼                ▼
      Controller-managed   Investigate
             │                │
             ▼                ▼
     Configure diff       Who changed it?
      handling                 │
                               ▼
                        Emergency change?
                          /          \
                        Yes           No
                         │             │
                         ▼             ▼
                  Update Git       Unauthorized
                  after approval      drift
                         │             │
                         ▼             ▼
                      Sync         Security/
                                   change review
```

---

# 36. Production Drift Recovery

The correct process:

```text
Detect
  ↓
Identify diff
  ↓
Classify drift
  ↓
Determine owner
  ↓
Determine whether intentional
  ↓
Assess customer/security impact
  ↓
Decide:
  ├── Keep change → update Git
  ├── Revert change → sync Git
  └── Controller-managed → configure appropriately
  ↓
Validate
  ↓
ArgoCD Synced
```

---

# 37. Emergency Change Procedure

During an active incident:

```text
Incident Commander
        ↓
Approve emergency change
        ↓
DevOps applies temporary mitigation
        ↓
Customer impact reduced
        ↓
Document exact change
        ↓
Create Git PR
        ↓
Review
        ↓
Merge
        ↓
ArgoCD sync
        ↓
Remove temporary manual state
```

This is much safer than leaving production drift.

---

# 38. What NOT to Do

Avoid:

```bash
kubectl edit deployment
```

as the normal production deployment process.

Avoid:

```bash
kubectl apply -f random-file.yaml
```

when GitOps owns that resource.

Avoid:

```bash
argocd app sync --prune
```

without understanding the diff.

Avoid:

```text
Ignore all differences
```

just to make ArgoCD green.

---

# 39. `--prune` Risk

ArgoCD pruning can delete resources that are no longer declared in Git.

For example:

```text
Git:
service-A
```

Live:

```text
service-A
service-B
```

If `service-B` is managed by another system but accidentally appears outside the intended ArgoCD ownership, aggressive pruning can delete it.

Therefore:

> **Understand application ownership before enabling automatic prune.**

---

# 40. Application Ownership

Our GitOps structure:

```text
gitops/
├── apps/
│   ├── order-service/
│   ├── payment-service/
│   └── inventory-service/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
└── platform/
    ├── prometheus/
    ├── grafana/
    ├── argocd/
    └── ingress/
```

ArgoCD Applications should have clear ownership.

---

# 41. ApplicationSet Consideration

For 21 microservices:

```text
ApplicationSet
     ↓
order-service
payment-service
inventory-service
...
```

If one template changes incorrectly:

```text
ApplicationSet
     ↓
21 Applications
     ↓
Potential widespread drift
```

Therefore platform changes require stronger review.

---

# 42. ArgoCD Drift Monitoring

Recommended metrics:

```text id="4r9s9d"
Application sync status
Application health
Sync failures
Reconciliation duration
Operation failures
Applications OutOfSync
Applications Degraded
```

Dashboard:

```text id="w2y1tr"
┌──────────────────────────────────────┐
│          ARGOCD HEALTH               │
├──────────────────────────────────────┤
│ Synced Applications                  │
│ OutOfSync Applications               │
│ Degraded Applications                │
│ Failed Syncs                         │
│ Recent Deployments                   │
│ Sync Duration                        │
└──────────────────────────────────────┘
```

---

# 43. Example Alert

Conceptually:

```text
Production application OutOfSync
for > 15 minutes
```

But avoid alerting on every temporary controller-generated difference.

The alert should be tuned for:

```text
Unexpected
+
Persistent
+
Production
```

drift.

---

# 44. DevOps Ownership

During an ArgoCD drift incident, Platform/DevOps usually investigates:

- ArgoCD
- GitOps repository
- Kubernetes state
- Helm
- Application manifests
- controllers
- RBAC
- audit logs
- recent deployments
- infrastructure changes

Application teams may own:

- application configuration
- image
- environment variables
- application-specific Helm values

Security may own:

- IAM
- RBAC
- NetworkPolicy
- security context
- secrets access

---

# 45. Example Production Scenario

Suppose:

```text
order-service
```

is experiencing high traffic.

An engineer runs:

```bash
kubectl scale deployment order-service \
  --replicas=15 \
  -n ecommerce
```

Git says:

```text
replicas = 6
```

ArgoCD detects:

```text
OutOfSync
```

The engineer says:

> "Don't sync. We need 15 replicas."

Correct response:

```text
1. Confirm incident need
2. Check RDS connection budget
3. Check cluster capacity
4. Check HPA configuration
5. Update Git if 15 should become desired state
6. Review
7. Merge
8. ArgoCD sync
```

---

# 46. Why Updating Git Matters

Suppose we leave:

```text
Git = 6
Live = 15
```

Tomorrow:

```text
ArgoCD sync
```

may restore:

```text
15 → 6
```

Now the production capacity suddenly drops.

Therefore:

> **Emergency manual changes must eventually become Git changes or be intentionally reverted.**

---

# 47. Example RCA — ArgoCD Drift

## Incident Summary

| Field | Details |
|---|---|
| Incident ID | `INC-2026-158` |
| Severity | SEV-2 |
| Date | 2026-10-04 |
| Duration | 15:20–16:05 IST |
| Affected Service | `order-service` |
| Root Cause | Emergency manual scaling was not reconciled into Git |
| Customer Impact | Temporary order API latency |
| Data Loss | No |

---

# 48. Customer Impact

During a traffic increase, the `order-service` deployment was manually scaled from:

```text
6 → 15 replicas
```

This initially reduced latency.

However, Git still declared:

```text
6 replicas
```

ArgoCD later reconciled the application.

The deployment changed:

```text
15 → 6
```

This caused:

```text
CPU ↑
Latency ↑
5xx ↑
```

for several minutes.

---

# 49. Timeline

```text
15:00  Traffic increases
15:05  order-service CPU rises
15:08  P95 latency increases
15:10  Incident declared
15:12  DevOps scales deployment 6 → 15
15:15  Latency improves
15:16  ArgoCD reports OutOfSync
15:18  Drift investigation starts
15:20  Engineer confirms manual emergency change
15:25  Git still contains replicas = 6
15:28  ArgoCD reconciliation occurs
15:29  Replicas drop 15 → 6
15:31  CPU/latency increase again
15:34  GitOps change prepared
15:38  PR approved
15:40  Git updated to desired capacity
15:42  ArgoCD sync
15:45  Replicas stabilize
15:50  Latency returns to baseline
16:05  Incident fully closed
```

---

# 50. Root Cause

The immediate operational failure was:

```text
Manual production change
        ↓
Git not updated
        ↓
ArgoCD detected drift
        ↓
Reconciliation restored Git state
        ↓
Replicas dropped
```

The underlying process failure was:

> Emergency production changes were not integrated into the GitOps workflow.

---

# 51. Amplification Mechanism

The original incident was:

```text
Traffic increase
```

Manual scaling temporarily fixed it.

But:

```text
Git remained at 6 replicas
```

ArgoCD later restored:

```text
6 replicas
```

Therefore:

```text
Traffic spike
    ↓
Manual scale to 15
    ↓
Customer impact improves
    ↓
Git remains at 6
    ↓
ArgoCD reconciliation
    ↓
15 → 6
    ↓
Customer impact returns
```

---

# 52. Contributing Factors

1. Emergency scaling was performed manually.
2. GitOps state was not updated immediately.
3. No emergency change procedure existed.
4. ArgoCD drift alert did not trigger an incident workflow.
5. HPA was not configured to handle the traffic pattern.
6. The team did not clearly document ownership of replica configuration.

---

# 53. What Went Well

- ArgoCD correctly detected drift.
- Manual scaling restored service temporarily.
- No data was lost.
- Git history allowed the configuration to be corrected.
- Application metrics clearly showed the replica-related impact.

---

# 54. What Went Wrong

- Emergency change was not reconciled into Git.
- ArgoCD reconciliation was not considered during mitigation.
- The team treated drift as an informational warning instead of part of the incident.
- Scaling policy was not aligned with the traffic pattern.

---

# 55. Immediate Mitigation

```text
1. Confirm desired vs live state
2. Confirm customer impact
3. Update Git desired state
4. Review PR
5. Merge
6. ArgoCD sync
7. Verify replica count
8. Verify HPA
9. Verify application latency
10. Confirm ArgoCD Synced
```

---

# 56. Corrective Actions

| Action | Owner |
|---|---|
| Create emergency GitOps change procedure | Platform |
| Require Git update after emergency `kubectl` changes | DevOps |
| Improve HPA configuration | Platform |
| Add OutOfSync production alert | SRE |
| Document controller-managed fields | Platform |
| Review ArgoCD self-heal policy | Platform |
| Add production change audit process | DevOps |
| Train incident responders | SRE |

---

# 57. Five Whys

### Why did order latency increase again?

Replicas dropped from 15 to 6.

### Why did replicas drop?

ArgoCD reconciled Kubernetes to Git.

### Why did Git say 6?

The emergency scaling change was never committed.

### Why wasn't it committed?

There was no documented emergency GitOps reconciliation procedure.

### Why did this create customer impact?

The scaling configuration and GitOps desired state were not treated as one operational system.

---

# 58. Final Root Cause Statement

> The ArgoCD drift incident occurred because an emergency manual scaling change increased `order-service` from 6 to 15 replicas without updating the GitOps desired state. ArgoCD later reconciled the application back to six replicas, reintroducing capacity pressure and customer-facing latency. The root process failure was incomplete reconciliation of emergency changes into Git.

---

# 59. Prevention Architecture

Improved process:

```text
                    Production Incident
                           │
                           ▼
                    Emergency Change
                           │
                           ▼
                  Temporary mitigation
                           │
                           ▼
                  Update Git immediately
                           │
                           ▼
                          PR
                           │
                           ▼
                       Review
                           │
                           ▼
                         Merge
                           │
                           ▼
                        ArgoCD
                           │
                           ▼
                   Desired = Live
```

---

# 60. ArgoCD Drift Runbook

Create:

```text
runbooks/argocd-drift.md
```

Recommended procedure:

```text
1. Check Application status
2. Check Sync status
3. Check Health
4. Inspect diff
5. Identify changed resource
6. Determine whether controller-managed
7. Check recent Git commits
8. Check Kubernetes audit evidence
9. Identify who changed live state
10. Determine customer/security impact
11. Decide whether to keep or revert
12. Update Git if change is intentional
13. Sync only after reviewing diff
14. Validate application
15. Confirm Synced
16. Confirm Healthy
17. RCA
```

---

# 61. Production Checklist

### ArgoCD

- [ ] Application identified
- [ ] Sync status checked
- [ ] Health checked
- [ ] Diff inspected
- [ ] Recent operation checked
- [ ] Git revision checked

### Kubernetes

- [ ] Live resource inspected
- [ ] Recent changes identified
- [ ] Controller ownership checked
- [ ] HPA checked
- [ ] Deployment checked
- [ ] ServiceAccount/RBAC checked

### Security

- [ ] IAM drift checked
- [ ] RBAC drift checked
- [ ] NetworkPolicy drift checked
- [ ] Secret configuration checked

### Recovery

- [ ] Intentional change identified
- [ ] Git updated
- [ ] PR approved
- [ ] ArgoCD synced
- [ ] Application validated
- [ ] No unexpected drift remains

---

# 62. Important Commands

### ArgoCD application

```bash
argocd app get order-service
```

### Application history

```bash
argocd app history order-service
```

### Diff

```bash
argocd app diff order-service
```

### Sync

```bash
argocd app sync order-service
```

**Only after reviewing the difference.**

### Kubernetes deployment

```bash
kubectl get deployment \
  order-service \
  -n ecommerce
```

### Full deployment

```bash
kubectl get deployment \
  order-service \
  -n ecommerce \
  -o yaml
```

### HPA

```bash
kubectl get hpa \
  -n ecommerce
```

### Application Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=order-service
```

---

# 63. Key Production Lessons

### Lesson 1

> **Git is the desired-state source of truth.**

### Lesson 2

> **OutOfSync does not automatically mean "Sync now."**

### Lesson 3

> **Always inspect the diff before reconciliation.**

### Lesson 4

> **Not every difference is bad drift; some fields are controller-managed.**

### Lesson 5

> **Emergency manual changes must be reconciled into Git.**

### Lesson 6

> **ArgoCD self-heal can be dangerous if emergency changes are not handled properly.**

### Lesson 7

> **Do not ignore differences just to make ArgoCD green.**

### Lesson 8

> **Security-related drift requires special attention.**

### Lesson 9

> **Kubernetes audit logs help answer who changed live state.**

### Lesson 10

> **A healthy GitOps platform means desired state and live state continuously converge.**

---

# 64. Interview Answer

If asked:

> **"What is ArgoCD drift and how would you troubleshoot it?"**

Answer:

> "ArgoCD drift means the Kubernetes live state differs from the desired state stored in Git. I first check the application's Sync and Health status and inspect the exact diff rather than immediately syncing. Then I determine whether the difference is intentional, caused by a controller such as HPA, or an unauthorized/manual change. I compare the Git history with the Kubernetes live resource and use audit evidence when necessary to identify who changed it. If the change is intentional, I update Git through a PR so Git and the cluster converge. If it is unauthorized or incorrect, I review the impact and then synchronize the known-good Git state. I also pay special attention to security resources such as IAM, RBAC and NetworkPolicies because drift there can create security risks."

---

# 65. Final GitOps Architecture

```text
                         GitHub
                            │
                            ▼
                     GitOps Repository
                            │
                    ┌───────┴───────┐
                    │               │
                 Helm Values     Manifests
                    │               │
                    └───────┬───────┘
                            ▼
                          ArgoCD
                            │
                 ┌──────────┴──────────┐
                 │                     │
            Desired State          Live State
                 │                     │
                 └──────────┬──────────┘
                            │
                       Diff Engine
                            │
                ┌───────────┴───────────┐
                │                       │
             Synced                  OutOfSync
                │                       │
                │                  Investigate
                │                       │
                │                ┌──────┴──────┐
                │                │             │
                │             Intentional   Unauthorized
                │                │             │
                │             Update Git    Revert/Review
                │                │             │
                └────────────────┴─────────────┘
                            │
                            ▼
                           EKS
                            │
                            ▼
                      Customer Traffic
```

---

## Phase 18 Progress

```text
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
```

**Next: `159 — Bad Production Deployment Production Incident`**