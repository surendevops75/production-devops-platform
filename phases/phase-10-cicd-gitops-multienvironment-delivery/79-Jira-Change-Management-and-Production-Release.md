# 79 — Jira Change Management and Production Release

> **Goal:** Add an enterprise-style **Jira Change Request** to our production deployment process and connect it with GitHub, GitOps, ArgoCD, and EKS.
>
> After this chapter, a production deployment will have a traceable chain:
>
> **Jira Change → GitHub PR → Git Commit → ECR Image → GitOps → ArgoCD → EKS → Validation**

---

# 1. Why Jira Change Management Matters

In a real enterprise environment, production deployment is usually not simply:

```text
Developer
   ↓
GitHub
   ↓
Production
```

There is normally a change-management process around production changes.

For example:

```text
Developer
   ↓
CI/CD
   ↓
STAGING
   ↓
Change Request
   ↓
Approval
   ↓
Production Deployment
```

The Jira change record answers questions such as:

* Why are we making this change?
* What exactly are we changing?
* Who requested it?
* Who approved it?
* When will it happen?
* What is the risk?
* What is the implementation plan?
* What is the rollback plan?
* How will we verify success?
* What happened after deployment?

---

# 2. Jira Does Not Deploy the Application

This distinction is very important.

Jira is not replacing:

```text
GitHub
GitHub Actions
ECR
ArgoCD
Kubernetes
```

Instead:

```text
Jira
 │
 └── Change management
```

```text
GitHub
 │
 └── Source/change implementation
```

```text
GitHub Actions
 │
 └── Build/test/security
```

```text
ECR
 │
 └── Container artifact
```

```text
ArgoCD
 │
 └── Deployment reconciliation
```

```text
EKS
 │
 └── Runtime
```

---

# 3. Complete Enterprise Flow

Our architecture now becomes:

```text
Developer
    │
    ▼
GitHub PR
    │
    ▼
CI
    │
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Security Scan
    ├── Trivy
    │
    ▼
ECR
    │
    ▼
Immutable Image
    │
    ▼
DEV
    │
    ▼
QA
    │
    ▼
STAGING
    │
    ▼
Jira Change Request
    │
    ▼
Approval
    │
    ▼
Production GitOps PR
    │
    ▼
Production Approval
    │
    ▼
ArgoCD
    │
    ▼
EKS PROD
    │
    ▼
Validation
    │
    ▼
Jira Change Closure
```

---

# 4. Example Change Request

Let's use our actual project:

```text
Service:
order-service
```

Suppose the image is:

```text
order-service:8f31a2c
```

Digest:

```text
sha256:9a7b123456789...
```

We create a change:

```text
CHG-10452
```

Example:

```text
Change:
Deploy order-service version 8f31a2c to production
```

---

# 5. Change Request Fields

A realistic change request can contain:

```text
Change ID
Change type
Service
Application version
Image tag
Image digest
Environment
Requester
Implementation owner
Risk
Business impact
Implementation plan
Validation plan
Rollback plan
Planned start
Planned end
Approvers
Related incident/problem
Related GitHub PR
```

Not every organization uses exactly these fields, but this is a useful production model.

---

# 6. Change Type

We can model three important types.

## Standard Change

A routine, pre-approved procedure.

Example:

```text
Routine application deployment
```

where the process is already well understood.

---

## Normal Change

A planned production change requiring review.

Example:

```text
New order-service release
```

This is our main scenario.

---

## Emergency Change

Used when an urgent production issue requires immediate action.

Example:

```text
Production payment failure
```

requiring an urgent rollback or hotfix.

---

# 7. Our Normal Production Release

For the project, we'll use:

```text
Normal Change
```

Example:

```text
CHG-10452
```

---

# 8. Change Request Example

```text
==================================================
CHANGE REQUEST
==================================================

Change ID:
CHG-10452

Service:
order-service

Environment:
Production

Change Type:
Normal Change

Application Version:
8f31a2c

Image:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c

Image Digest:
sha256:9a7b123456789...

Source Environment:
STAGING

Target Environment:
PRODUCTION

Requester:
Application Team

Implementation Owner:
DevOps / Platform Team
```

---

# 9. Change Description

Keep the description clear.

Example:

```text
Deploy the validated order-service release
8f31a2c to the production EKS environment.

The release has completed CI, security scanning,
QA validation, and staging validation.

The production deployment will use the same
immutable ECR image digest validated in staging.
```

This is much better than:

```text
Deploy latest order service.
```

because the latter does not identify the artifact.

---

# 10. Business Reason

Explain why the change is needed.

Example:

```text
Business Reason:

Release approved order-service enhancements
and defect fixes to production.
```

You don't need a huge paragraph.

The important thing is that someone reviewing the change understands the purpose.

---

# 11. Risk Assessment

Example:

```text
Risk:
Medium
```

Then explain:

```text
Potential risks:

- Application startup failure
- Database compatibility issue
- Increased API latency
- Increased error rate
- Dependency integration failure
```

Then explain mitigations:

```text
Mitigations:

- Release validated in staging
- Rolling deployment
- Readiness probes
- Monitoring dashboards
- Alerting
- Previous image retained in ECR
- GitOps rollback available
```

---

# 12. Impact Assessment

Example:

```text
Expected customer impact:

No planned downtime.

The deployment uses Kubernetes rolling update
with multiple replicas and readiness probes.
```

But don't automatically assume zero impact.

A realistic assessment should consider:

```text
Database migrations
Connection draining
Pod startup
ALB behavior
Dependency compatibility
```

---

# 13. Implementation Plan

This is one of the most important parts.

Example:

```text
1. Confirm STAGING deployment is healthy.

2. Verify STAGING runtime image digest.

3. Verify ECR contains the approved image.

4. Confirm production prerequisites.

5. Update production GitOps values to the
   approved image digest.

6. Create/review production GitOps PR.

7. Obtain required approvals.

8. Merge production GitOps PR.

9. Verify ArgoCD detects the change.

10. Review ArgoCD diff.

11. Synchronize order-service production application.

12. Monitor Kubernetes rollout.

13. Verify readiness of all new Pods.

14. Run production smoke tests.

15. Monitor metrics, logs, traces, and alerts.

16. Confirm deployment success.
```

---

# 14. Rollback Plan

Never create a production change without a rollback plan.

Example:

```text
Rollback Plan:

If deployment causes application errors,
latency increase, failed health checks,
or other unacceptable behavior:

1. Stop further rollout.

2. Revert the production GitOps commit.

3. Verify ArgoCD detects the rollback.

4. Synchronize the previous desired state.

5. Verify the previous image is running.

6. Run smoke tests.

7. Monitor application health.

8. Document the rollback in the change record.
```

---

# 15. Important: Rollback Must Be Tested

Don't write:

```text
Rollback available.
```

and assume it works.

We should test rollback in STAGING.

For example:

```text
STAGING
   ↓
Deploy NEW
   ↓
Verify NEW
   ↓
Rollback
   ↓
Verify OLD
```

Then we know the production rollback procedure is real.

---

# 16. Validation Plan

After deployment:

```text
Health:
PASS

Deployment:
PASS

Pod readiness:
PASS

Application API:
PASS

Database connectivity:
PASS

Error rate:
NORMAL

Latency:
NORMAL

Logs:
NORMAL

Traces:
NORMAL

Alerts:
NO CRITICAL ALERTS
```

---

# 17. Pre-Change Checklist

Before the production change:

```text
[ ] CI passed
[ ] Unit tests passed
[ ] Security scans passed
[ ] Image pushed to ECR
[ ] Image digest verified
[ ] QA passed
[ ] STAGING passed
[ ] STAGING runtime digest verified
[ ] Production GitOps PR created
[ ] Database compatibility verified
[ ] Rollback plan confirmed
[ ] Monitoring available
[ ] Required approvals obtained
[ ] Change window confirmed
```

---

# 18. Change Window

Some organizations have defined deployment windows.

Example:

```text
Start:
10:00 IST

Expected completion:
10:30 IST
```

The exact timing depends on business requirements.

The important thing is that the deployment team knows:

```text
When are we changing?
Who is responsible?
Who needs to be available?
```

---

# 19. Production GitOps PR

The Jira change should be referenced in the GitHub PR.

Example:

```text
Title:

release: promote order-service 8f31a2c to prod
```

PR body:

```text
## Production Release

Jira Change:
CHG-10452

Service:
order-service

Source:
STAGING

Target:
PRODUCTION

Image Tag:
8f31a2c

Image Digest:
sha256:9a7b123...

STAGING Runtime Digest:
sha256:9a7b123...

Validation:
- CI: PASS
- QA: PASS
- STAGING: PASS
- Security: PASS

Rollback:
Revert this GitOps commit.
```

---

# 20. The Traceability Chain

This is one of the most important concepts in the project.

```text
Jira Change
CHG-10452
      │
      ▼
GitHub PR
#248
      │
      ▼
Git Commit
abc1234
      │
      ▼
Docker Image
order-service:8f31a2c
      │
      ▼
ECR Digest
sha256:9a7b123...
      │
      ▼
GitOps Commit
def5678
      │
      ▼
ArgoCD Application
order-service-prod
      │
      ▼
Kubernetes Deployment
order-service
      │
      ▼
Running Pods
```

This is the level of traceability you want to be able to explain in a real production environment.

---

# 21. Jira ↔ GitHub Integration

In an enterprise setup, Jira and GitHub can be integrated.

The exact configuration depends on the Jira/GitHub setup, but conceptually:

```text
Jira
  │
  ├── CHG-10452
  │
  ▼
GitHub PR
  │
  └── CHG-10452
```

The Jira change can reference the GitHub PR.

The GitHub PR references the Jira change.

Therefore:

```text
Jira ↔ GitHub
```

becomes traceable.

---

# 22. Commit Convention

You can include the Jira identifier in the commit.

For example:

```text
release: promote order-service to prod [CHG-10452]
```

or your organization's preferred convention.

Then Git history contains:

```text
abc1234 release: promote order-service to prod [CHG-10452]
```

Now searching for:

```text
CHG-10452
```

can connect the release records.

---

# 23. Jira Change and GitOps

Remember:

```text
Jira does NOT contain the Kubernetes manifest as the source of truth.
```

GitOps does.

Jira records:

```text
Why
Who
When
Approval
Risk
Plan
Result
```

GitOps records:

```text
Desired production state
```

---

# 24. Production Deployment Sequence

Let's put everything together.

### Phase 1 — Development

```text
Developer
 ↓
Feature branch
 ↓
PR
 ↓
CI
```

### Phase 2 — Build

```text
Test
 ↓
Security
 ↓
Docker
 ↓
Trivy
 ↓
ECR
```

### Phase 3 — Environment Promotion

```text
DEV
 ↓
QA
 ↓
STAGING
```

### Phase 4 — Change Management

```text
STAGING PASS
 ↓
Jira Change
 ↓
Review
 ↓
Approval
```

### Phase 5 — Production

```text
Production GitOps PR
 ↓
Review
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

### Phase 6 — Validation

```text
Health
 ↓
Smoke Test
 ↓
Metrics
 ↓
Logs
 ↓
Traces
 ↓
Alerts
```

### Phase 7 — Closure

```text
Deployment successful
 ↓
Update Jira
 ↓
Record result
 ↓
Close Change
```

---

# 25. Jira Change Closure

After successful deployment, update the change.

Example:

```text
Change:
CHG-10452

Status:
Implemented / Successful

Implementation Result:
order-service version 8f31a2c deployed successfully.

Production Digest:
sha256:9a7b123...

Deployment:
Successful

Smoke Tests:
Passed

Error Rate:
Normal

Latency:
Normal

Rollback:
Not required

Post-deployment monitoring:
Completed
```

Then close the change according to the organization's workflow.

---

# 26. Failed Change

Suppose deployment fails.

Don't simply close it as successful.

Example:

```text
CHG-10452

Status:
Failed / Rolled Back
```

Document:

```text
Deployment:
Failed

Issue:
New application version caused elevated 5xx responses.

Action:
Production GitOps reverted.

Rollback:
Successful

Previous Digest:
sha256:OLD...

New Digest:
sha256:9a7b123...

Customer Impact:
X minutes

Follow-up:
Problem/incident investigation required.
```

This connects naturally to our **Incident Response** and **RCA** chapters.

---

# 27. Emergency Change

Now consider a serious production incident.

Example:

```text
Production
    ↓
Payment service failing
    ↓
Customer transactions affected
```

You might need:

```text
Emergency Change
```

The flow becomes:

```text
Incident
   ↓
Emergency Change
   ↓
Approved emergency action
   ↓
Rollback / hotfix
   ↓
Production
   ↓
Validation
   ↓
Incident resolution
   ↓
Post-incident review
```

The exact approval process depends on the organization's emergency-change policy.

---

# 28. Standard Change

Later, once our deployment process becomes mature, a routine release could potentially become a **pre-approved standard change**.

For example:

```text
Standard Change:
Routine microservice deployment using approved
GitOps pipeline and rollback procedure.
```

The key is that the process itself has already been assessed and approved by the organization.

We should not label a change "standard" merely because it is automated.

---

# 29. Change vs Incident vs Problem

These terms are often confused.

### Change

```text
Something we intentionally want to modify.
```

Example:

```text
Deploy order-service 8f31a2c.
```

### Incident

```text
Something is currently broken or degraded.
```

Example:

```text
Order API returning 5xx.
```

### Problem

```text
Underlying cause of one or more incidents.
```

Example:

```text
Connection pool configuration causes
database exhaustion under load.
```

So:

```text
Change
  ↓
Deployment
  ↓
Incident
  ↓
Problem investigation
```

can happen in a real production lifecycle.

---

# 30. Change Request vs GitHub PR

They are not duplicates.

### Jira

```text
Business/operational change record
```

### GitHub

```text
Technical code/configuration change
```

For example:

**Jira:**

```text
CHG-10452
Deploy order-service release to production
```

**GitHub:**

```text
PR #248
Change production image digest
```

Both describe different aspects of the same release.

---

# 31. Change Request vs ArgoCD

Again, different responsibilities.

```text
Jira
 ↓
Approval and governance
```

```text
GitHub
 ↓
Desired state
```

```text
ArgoCD
 ↓
Reconciliation
```

```text
Kubernetes
 ↓
Runtime
```

This separation is important.

---

# 32. Production Deployment With Jira

Our final enterprise flow is:

```text
                         DEVELOPMENT
                              │
                              ▼
                           GitHub
                              │
                              ▼
                       GitHub Actions
                              │
                    Build / Test / Scan
                              │
                              ▼
                             ECR
                              │
                       Immutable Digest
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
            DEV              QA             STAGING
                                             │
                                        Validation
                                             │
                                             ▼
                                      Jira Change
                                      CHG-10452
                                             │
                                         Approval
                                             │
                                             ▼
                                      GitOps PROD PR
                                             │
                                         Approval
                                             │
                                             ▼
                                           ArgoCD
                                             │
                                             ▼
                                            EKS
                                             │
                                             ▼
                                      Rolling Update
                                             │
                                             ▼
                                      Smoke Testing
                                             │
                                             ▼
                                       Monitoring
                                             │
                                             ▼
                                      Jira Closure
```

---

# 33. What We Should Implement in Our Project

For our practical project, the production release checklist should eventually look like this:

```text
1. CI builds order-service
2. Trivy scans image
3. Image pushed to ECR
4. Digest recorded
5. DEV deployment
6. DEV validation
7. QA promotion
8. QA validation
9. STAGING promotion
10. STAGING validation
11. Create Jira Change Request
12. Attach release information
13. Link GitHub production PR
14. Obtain production approval
15. Merge GitOps PR
16. Review ArgoCD diff
17. Sync production
18. Monitor rollout
19. Verify production digest
20. Run smoke tests
21. Monitor metrics/logs/traces
22. Update Jira with result
23. Close change
```

---

# 34. One Important Production Principle

Don't think:

> "Jira approved it, so deployment is safe."

Instead:

```text
Jira approval
       +
GitOps review
       +
Artifact verification
       +
Deployment controls
       +
Runtime validation
```

together provide the production control system.

---

# 35. Final Architecture We Have Reached

```text
                         ┌───────────────┐
                         │    JIRA       │
                         │ Change Mgmt   │
                         └───────┬───────┘
                                 │
                            Approval
                                 │
                                 ▼
Developer → GitHub → CI → ECR → DEV
                              │
                              ▼
                             QA
                              │
                              ▼
                          STAGING
                              │
                              ▼
                       Jira Change Request
                              │
                              ▼
                     Production GitOps PR
                              │
                              ▼
                           ArgoCD
                              │
                              ▼
                         EKS PROD
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              Metrics       Logs        Traces
                 │            │            │
                 └────────────┼────────────┘
                              ▼
                         Validation
                              │
                              ▼
                        Jira Closure
```

---

## Chapter 79 outcome

We now have the **enterprise change-management layer** around our production deployment.

The production release is no longer simply:

```text
GitOps → ArgoCD → EKS
```

It is:

```text
STAGING
   ↓
Jira Change Request
   ↓
Approval
   ↓
Production GitOps PR
   ↓
Approval
   ↓
ArgoCD
   ↓
EKS
   ↓
Validation
   ↓
Jira Closure
```

And the complete traceability chain is:

```text
CHG-10452
    ↓
GitHub PR
    ↓
Git Commit
    ↓
ECR Image
    ↓
Image Digest
    ↓
GitOps Commit
    ↓
ArgoCD
    ↓
Kubernetes Deployment
    ↓
Running Pod
```

**Next: `80-Production-Deployment-Strategies.md`** — we can now cover **Rolling, Recreate, Blue-Green, Canary, Argo Rollouts, traffic shifting, automated analysis, rollback, and practical EKS implementation** with the Jira/change-management process we just established.
