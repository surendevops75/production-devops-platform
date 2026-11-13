# CHAPTER 204 — PRODUCTION APPROVAL

**Filename:** `204-Production-Approval.md`

---

# 1. Chapter Objective

Production approval is a critical control in a production-grade DevOps environment.

Creating a Jira Production Change request does not automatically authorize a deployment.

Before deploying a release to Production, the responsible teams must review the change, evaluate the risk, verify the supporting evidence, and provide the required approvals.

In this chapter, we will implement a practical Production approval workflow for the RoboShop microservices platform.

We will cover:

- Production approval architecture.
- Approval responsibilities.
- Change Advisory Board (CAB).
- Technical approval.
- QA approval.
- Security approval.
- Database approval.
- Infrastructure approval.
- Application owner approval.
- Business approval.
- Risk-based approval.
- Separation of duties.
- Jira approval workflow.
- Approval evidence.
- Jenkins approval gate.
- GitHub Actions approval gate.
- GitLab CI/CD approval controls.
- Argo CD deployment authorization.
- Emergency change approval.
- Rejection and resubmission.
- Approval expiration.
- Audit trail.
- Final Production go/no-go decision.

**Important chapter boundaries:**

- Chapter 203: Create and manage the Jira Production Change.
- Chapter 204: Review and authorize the Production change.
- Chapter 205: Execute the Production deployment.
- Chapter 206: Perform rolling and canary deployment strategies.

Approval and deployment are separate activities.

# 2. Production Approval Architecture

The approval process connects Jira, engineering teams, CI/CD, GitOps, and Production operations.

```text
                 CHAPTER 203
             Jira Production Change
                       |
                       v
              Change Submitted
                       |
                       v
              Initial Validation
                       |
                       v
              Technical Review
                       |
          +------------+------------+
          |            |            |
          v            v            v
       DevOps         QA         Security
          |            |            |
          +------------+------------+
                       |
                       v
              Database Review
                       |
                       v
              Application Owner
                       |
                       v
                Risk Assessment
                       |
                       v
               CAB Review
                       |
                       v
              Business Approval
                       |
                       v
              Final Approval
                       |
                       v
               Approved Status
                       |
                       v
             Production Go/No-Go
                       |
                       v
                 CHAPTER 205
              Production Deployment
```

The exact approval groups depend on the organization's governance model and the impact of the change.

A low-risk application release may require fewer approvers than a change involving database migrations, networking, security controls, or critical infrastructure.

# 3. Understand Production Approval

Production approval is a formal decision that a proposed change can proceed under defined conditions.

An approver should understand:

1. What is changing?
2. Why is the change required?
3. Which services are affected?
4. What is the business impact?
5. What is the technical risk?
6. Has QA validation passed?
7. Has Staging validation passed?
8. Have security checks passed?
9. Are database changes understood?
10. Is the rollback plan practical?
11. Is the deployment window suitable?
12. Are the responsible engineers available?
13. Are monitoring and alerting ready?
14. Are dependencies available?
15. Is the release artifact immutable and traceable?

An approval should be based on evidence, not merely on a request from the deployment team.

# 4. Approval Roles and Responsibilities

A production release may involve several teams.

## 4.1 DevOps Engineer

Responsibilities:

- Verify the deployment architecture.
- Verify EKS cluster readiness.
- Verify Kubernetes manifests.
- Verify Helm values.
- Verify Argo CD configuration.
- Verify ECR image digests.
- Verify GitOps promotion.
- Review resource capacity.
- Review deployment strategy.
- Verify rollback readiness.
- Provide deployment runbook.

## 4.2 QA Engineer

Responsibilities:

- Review QA test results.
- Review Staging test results.
- Verify regression testing.
- Verify API testing.
- Verify integration testing.
- Verify critical business workflows.
- Document known defects.
- Confirm release acceptance.

## 4.3 Security Engineer

Responsibilities:

- Review SonarQube results.
- Review Trivy results.
- Review dependency vulnerabilities.
- Review container image vulnerabilities.
- Review secrets management.
- Review IAM permissions.
- Review security exceptions.
- Verify that critical findings are handled according to policy.

## 4.4 Database Administrator

Required when database changes are involved.

Responsibilities:

- Review schema changes.
- Review migration scripts.
- Verify backups.
- Review database compatibility.
- Review rollback or forward-recovery strategy.
- Review connection pool impact.
- Verify migration execution permissions.

## 4.5 Application Owner

Responsibilities:

- Understand application changes.
- Verify business functionality.
- Review service dependencies.
- Confirm application readiness.
- Accept known limitations.
- Confirm application support availability.

## 4.6 Platform or Infrastructure Owner

Required when infrastructure changes are involved.

Responsibilities:

- Review EKS changes.
- Review networking changes.
- Review node group changes.
- Review storage changes.
- Review Terraform changes.
- Review platform capacity.
- Verify infrastructure rollback or recovery options.

## 4.7 Business Approver

Responsibilities:

- Understand customer impact.
- Review expected business benefits.
- Confirm the release window.
- Accept documented business risk.
- Approve the change where business authorization is required.

## 4.8 Change Manager

Responsibilities:

- Review the change record.
- Verify required assessments.
- Confirm required approvals.
- Check deployment scheduling.
- Coordinate the CAB where applicable.
- Ensure change governance is followed.

# 5. Approval Matrix

Create an approval matrix to define who must approve each category of change.

Example:

| Change category | Required reviewers |
|---|---|
| Application release | DevOps, QA, Application Owner |
| Container image update | DevOps, Security, Application Owner |
| Database migration | DBA, QA, Application Owner |
| EKS upgrade | Platform, DevOps, Security |
| Network change | Network, Platform, Security |
| IAM permission change | Security, Platform |
| Critical security fix | Security, Application Owner, DevOps |
| Major business release | Business Owner, QA, DevOps, Change Manager |

This is an illustrative matrix.

The actual organization must define its mandatory approvals and escalation rules.

# 6. Separation of Duties

Separation of duties prevents one individual from controlling every stage of a sensitive Production change.

Example:

```text
Developer
   |
   v
Creates Code Change
   |
   v
CI Pipeline
   |
   v
QA Engineer
   |
   v
Validates Release
   |
   v
Security Engineer
   |
   v
Reviews Security Evidence
   |
   v
Change Manager
   |
   v
Coordinates Approval
   |
   v
Authorized Approver
   |
   v
Approves Production Change
   |
   v
Deployment Engineer
   |
   v
Executes Production Deployment
```

Recommended controls:

- A developer should not automatically approve their own Production change.
- A pipeline identity should not approve a change merely because its build succeeded.
- Production deployment permissions should be restricted.
- Approval records should identify the actual approver.
- Emergency access should be separately controlled and audited.

The exact separation rules depend on company policy and regulatory requirements.

# 7. Review the Jira Production Change

Start with the Jira issue created in Chapter 203.

Illustrative issue:

```text
Issue:
ROBOSHOP-450

Summary:
Production Release - RoboShop 2026.11.03

Environment:
Production

Change Type:
Normal Change

Status:
Submitted

Release:
roboshop-2026.11.03
```

The reviewer should inspect the following.

## 7.1 Change Description

Confirm that the description explains:

- What is changing.
- Why it is changing.
- Which services are affected.
- Which environment is targeted.
- What the expected outcome is.

## 7.2 Release Information

Verify:

- Source repository.
- Source commit.
- CI pipeline execution.
- ECR repository.
- Image digest.
- GitOps repository.
- GitOps commit.

## 7.3 Testing Evidence

Review:

- Unit test results.
- Integration test results.
- QA test results.
- Staging test results.
- Regression test results.
- Performance test results.
- Security scan results.

## 7.4 Rollback Plan

Verify:

- Previous image references are available.
- Previous Helm values are identifiable.
- Previous GitOps revision is available.
- Database recovery considerations are documented.
- Recovery responsibilities are assigned.

If critical information is missing, return the change for correction rather than approving an incomplete request.

# 8. Technical Approval

Technical approval verifies that the proposed deployment is technically safe and executable.

## 8.1 Kubernetes Review

Verify the following:

```bash
kubectl config current-context

kubectl get nodes

kubectl get namespaces

kubectl get pods -n production

kubectl get deployments -n production

kubectl get services -n production

kubectl get hpa -n production

kubectl get pdb -n production
```

These are inspection commands. Confirm that the selected context is the intended Production cluster before running them.

Review:

- Cluster health.
- Node readiness.
- Pod availability.
- Deployment replicas.
- Resource requests.
- Resource limits.
- Readiness probes.
- Liveness probes.
- Startup probes.
- HPA configuration.
- Pod disruption budgets.

## 8.2 Helm Review

Verify the exact release configuration.

```bash
helm list -n production

helm history roboshop-user -n production

helm get values roboshop-user \
  -n production \
  --all
```

Review the candidate values before deployment:

```bash
helm lint ./charts/user

helm template roboshop-user ./charts/user \
  --namespace production \
  -f environments/production/user-values.yaml
```

Do not deploy merely because a chart passes linting.

Review the rendered resources, security configuration, image references, and environment-specific values.

## 8.3 ECR Image Review

Verify that the release references the intended immutable image.

Example:

```text
Repository:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user

Image:
sha256:APPROVED_IMAGE_DIGEST
```

Confirm:

- Repository exists.
- Image digest exists.
- Image belongs to the intended release.
- Image scan evidence is available.
- The digest matches the QA and Staging release evidence.

## 8.4 GitOps Review

Review the Production GitOps pull request.

Confirm:

- Correct environment folder.
- Correct image digest.
- Correct Helm values.
- Correct application configuration.
- Correct Argo CD application.
- No unrelated changes.
- Required reviewers approved the PR.
- CI checks passed.

Production GitOps changes must follow the repository's branch protection and merge policy.

# 9. QA Approval

QA approval confirms that the release meets the required testing criteria.

## 9.1 Functional Testing

For RoboShop, validate relevant workflows such as:

```text
User Registration
       |
       v
User Login
       |
       v
Browse Products
       |
       v
Add to Cart
       |
       v
Place Order
       |
       v
Payment Processing
       |
       v
Inventory Update
       |
       v
Order Notification
```

Use the actual application dependency map and test environment.

## 9.2 API Testing

Validate important endpoints.

Example:

```bash
curl -f https://staging.example.com/health

curl -f https://staging.example.com/api/catalog

curl -f https://staging.example.com/api/cart
```

The paths are examples; use the actual endpoints configured by the application.

## 9.3 Regression Testing

Confirm that existing functionality remains intact.

## 9.4 QA Acceptance Record

Example:

```text
QA APPROVAL

Jira Change:
ROBOSHOP-450

Release:
roboshop-2026.11.03

Unit Tests:
Passed

Integration Tests:
Passed

Regression Tests:
Passed

API Tests:
Passed

Business Workflows:
Passed

Known Defects:
Documented

QA Decision:
Approved

Evidence:
QA test report attached
```

Only the authorized QA reviewer can provide the actual approval.

# 10. Security Approval

Security approval verifies that the release satisfies the organization's security requirements.

## 10.1 SonarQube Review

Review:

- New bugs.
- Vulnerabilities.
- Code smells.
- Security hotspots.
- Quality gate.
- Coverage requirements where enforced.

Example:

```text
SonarQube Project:
roboshop-user

Quality Gate:
PASSED

New Vulnerabilities:
0

New Blocker Issues:
0

Decision:
Ready for security review
```

The exact acceptance thresholds must follow organizational policy.

## 10.2 Trivy Image Scan

Example:

```bash
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:APPROVED_IMAGE_DIGEST
```

This example fails when findings at the selected severities are detected.

A real release process must define:

- Accepted vulnerability thresholds.
- Exception handling.
- Vulnerability ownership.
- Fix deadlines.
- Scanner database freshness.
- Image scan evidence retention.

## 10.3 Secrets Review

Confirm:

- No secrets committed to Git.
- Kubernetes secrets are managed through the approved mechanism.
- External Secrets or the approved secret integration is functioning.
- IAM permissions follow least privilege.
- No credentials appear in pipeline logs.
- Application configuration does not expose sensitive values.

## 10.4 Security Approval Record

```text
SECURITY APPROVAL

Jira Change:
ROBOSHOP-450

SonarQube:
Reviewed

Trivy:
Reviewed

Secrets:
Reviewed

IAM:
Reviewed

Exceptions:
Documented

Security Decision:
Approved / Rejected

Evidence:
Security scan reports attached
```

A scan passing does not automatically replace security review when the organization's policy requires a human decision.

# 11. Database Approval

Database approval is required when the change modifies database schemas, data, database infrastructure, or database access.

Review:

- Migration scripts.
- Schema compatibility.
- Backward compatibility.
- Data integrity.
- Backup status.
- Database capacity.
- Connection pool settings.
- Locking behavior.
- Migration duration.
- Recovery approach.

Example review:

```text
DATABASE APPROVAL

Database:
RoboShop Production RDS

Change:
Orders table schema update

Migration:
V20261103__orders_update.sql

Backup:
Verified

Compatibility:
Reviewed

Application Compatibility:
Reviewed

Rollback / Recovery:
Documented

DBA Decision:
Approved
```

Important:

Some schema changes cannot safely be reversed after application traffic has written new data.

For these changes, use a forward-fix or restore strategy where appropriate rather than assuming that running a reverse SQL script is safe.

# 12. Infrastructure Approval

Infrastructure approval is required when the release changes AWS or Kubernetes infrastructure.

Review:

- Terraform plan.
- VPC changes.
- Security groups.
- IAM roles and policies.
- EKS cluster configuration.
- Node groups.
- Load balancers.
- Storage.
- DNS.
- Certificates.
- Network policies.

Example:

```bash
terraform fmt -check

terraform validate

terraform plan \
  -var-file=environments/production.tfvars
```

The approver must review the actual plan output.

Never run `terraform apply` merely because the plan command succeeded.

Example infrastructure approval record:

```text
INFRASTRUCTURE APPROVAL

Change:
ROBOSHOP-450

Terraform Plan:
Reviewed

IAM:
Reviewed

Network:
Reviewed

EKS:
Reviewed

Storage:
Reviewed

Decision:
Approved
```

# 13. Application Owner Approval

The application owner confirms that the release is ready from the application's operational and functional perspective.

Review:

- Feature changes.
- API compatibility.
- Service dependencies.
- Configuration changes.
- External integrations.
- Customer impact.
- Known limitations.
- Application support coverage.

Example:

```text
APPLICATION OWNER APPROVAL

Application:
RoboShop

Release:
roboshop-2026.11.03

Affected Services:
Listed in change request

Compatibility:
Verified

Business Functionality:
Accepted

Known Issues:
Documented

Decision:
Approved
```

# 14. Change Advisory Board (CAB)

A Change Advisory Board is a group that reviews and advises on changes according to the organization's change-management policy.

The CAB may include:

- Change Manager.
- DevOps Lead.
- Application Owner.
- Security Lead.
- QA Lead.
- Infrastructure Lead.
- Database Administrator.
- Business Representative.

Not every change requires a CAB meeting.

The organization may define risk-based review requirements.

## 14.1 CAB Responsibilities

The CAB evaluates:

1. Business justification.
2. Technical impact.
3. Risk.
4. Testing evidence.
5. Change dependencies.
6. Resource availability.
7. Deployment window.
8. Rollback readiness.
9. Communication.
10. Operational readiness.

## 14.2 Example CAB Review

```text
CAB REVIEW

Change ID:
ROBOSHOP-450

Change:
RoboShop Production Release

Technical Review:
Completed

QA Review:
Completed

Security Review:
Completed

Database Review:
Not Required

Infrastructure Review:
Completed

Business Impact:
Reviewed

Rollback Plan:
Reviewed

CAB Decision:
Approved with conditions
```

Any conditions must be documented and verified before deployment.

## 14.3 CAB Decision Types

- Approved.
- Approved with conditions.
- Rejected.
- Deferred.
- More information required.

A deferred or rejected change must not proceed as an approved deployment.

# 15. Risk-Based Approval

Not every change has the same operational risk.

Example risk categories:

| Risk | Example | Review approach |
|---|---|---|
| Low | Small configuration change | Standard policy review |
| Medium | Multiple microservice release | Technical and application review |
| High | Database or network change | Additional specialist review |
| Critical | Major platform or security change | Enhanced review and escalation |

The organization must define its own risk scoring model.

Example assessment:

```yaml
change_risk:
  application_impact: medium
  availability_impact: medium
  data_impact: low
  security_impact: medium
  infrastructure_impact: low

  risk_factors:
    - Multiple services affected
    - Production configuration changes
    - Customer-facing workflows affected

  mitigations:
    - QA and Staging validation
    - Immutable image digests
    - Controlled rollout
    - Monitoring
    - Tested recovery procedure

  overall_risk: medium
```

Risk assessment must reflect the actual change rather than using a fixed rating for every release.

# 16. Configure Jira Approval Workflow

The Jira workflow should enforce the organization's approval process.

Example:

```text
Draft
  |
  v
Submitted
  |
  v
Technical Review
  |
  v
QA Review
  |
  v
Security Review
  |
  v
Application Owner Review
  |
  v
CAB Review
  |
  v
Pending Approval
  |
  +------> Rejected
  |
  v
Approved
  |
  v
Scheduled
  |
  v
Ready for Deployment
```

Some reviews can occur in parallel.

For example, QA, Security, and Infrastructure may review the same release independently.

The workflow must not move to Approved until all mandatory reviews have been completed.

## 16.1 Configure Statuses

Possible statuses:

```text
Submitted
Technical Review
QA Review
Security Review
Business Review
CAB Review
Pending Approval
Changes Requested
Rejected
Approved
Scheduled
Ready for Deployment
```

Use only statuses that are needed by the organization's process.

## 16.2 Configure Transitions

Example:

| From | To | Required control |
|---|---|---|
| Submitted | Technical Review | Change record complete |
| Technical Review | QA Review | Technical review recorded |
| QA Review | Security Review | QA result recorded |
| Security Review | Business Review | Security decision recorded |
| Business Review | CAB Review | Business decision recorded |
| CAB Review | Pending Approval | CAB outcome recorded |
| Pending Approval | Approved | All mandatory approvals complete |
| Approved | Scheduled | Deployment window assigned |
| Scheduled | Ready for Deployment | Pre-deployment checks complete |

These are example transitions, not universal Jira defaults.

## 16.3 Configure Jira Permissions

Restrict who can:

- Submit changes.
- Perform technical review.
- Record QA approval.
- Record security approval.
- Approve changes.
- Schedule changes.
- Cancel changes.
- Close changes.

Use Jira project roles and permission schemes.

Where possible, enforce separation between the change creator, approver, and deployer.

# 17. Jira Approval Evidence

Every approval should be traceable.

Record:

- Approver identity.
- Approval decision.
- Date and time.
- Change issue key.
- Review category.
- Supporting evidence.
- Conditions.
- Rejection reason, if applicable.

Example:

```text
Approval Record

Change:
ROBOSHOP-450

Review:
Security

Approver:
Authorized Security Reviewer

Decision:
Approved

Timestamp:
Recorded by Jira

Comments:
Security checks reviewed. No unresolved blocking findings.
```

Do not use a generic comment such as "Looks good" as the only evidence when a formal approval decision is required.

# 18. Automated Approval Validation

The Production deployment pipeline should verify that the change has reached the correct authorized state.

A Jira issue existing is not enough.

The gate should verify:

- Correct Jira issue key.
- Correct issue type.
- Correct project.
- Correct environment.
- Correct release identifier.
- Required reviews completed.
- Final approval recorded.
- Approval has not been revoked.
- Deployment window is valid.
- Approved artifact matches the release being deployed.

The exact implementation depends on the Jira workflow, custom fields, and approval integration.

## 18.1 Approval Gate Architecture

```text
Production Deployment Trigger
          |
          v
Read Jira Change Key
          |
          v
Fetch Jira Change
          |
          v
Validate Issue Type
          |
          v
Validate Environment
          |
          v
Validate Release ID
          |
          v
Validate Review Results
          |
          v
Validate Final Approval
          |
          v
Validate Artifact Digest
          |
          v
Validate Deployment Window
          |
          v
Approval Gate Passed
          |
          v
Chapter 205 Deployment
```

If any mandatory check fails, the deployment must stop.

# 19. Jenkins Production Approval Gate

Jenkins supports interactive approval using the `input` step.

However, an interactive Jenkins prompt alone is not a complete change-management approval system.

The approval must be linked to the Jira record and restricted to authorized users.

Example:

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'JIRA_CHANGE_KEY',
            defaultValue: '',
            description: 'Jira Production Change key'
        )
    }

    stages {
        stage('Production Approval') {
            steps {
                script {
                    if (!params.JIRA_CHANGE_KEY?.trim()) {
                        error('Jira change key is required')
                    }

                    timeout(time: 30, unit: 'MINUTES') {
                        input(
                            message: """
                            Production deployment approval required.

                            Jira Change:
                            ${params.JIRA_CHANGE_KEY}

                            Release:
                            roboshop-2026.11.03

                            Confirm that the Jira approval
                            workflow has been completed.
                            """,
                            ok: 'Confirm approval',
                            submitter: 'production-approvers',
                            submitterParameter: 'APPROVED_BY'
                        )
                    }

                    echo "Approval recorded by ${env.APPROVED_BY}"
                }
            }
        }
    }
}
```

Important:

- Configure the `production-approvers` group in Jenkins.
- Restrict access to the approval job.
- Record the actual approver.
- Verify Jira approval state independently.
- Do not allow an arbitrary build user to approve the release.

The Jenkins input step is an additional deployment gate, not a substitute for the formal approval record.

# 20. GitHub Actions Production Approval Gate

GitHub Actions environments support deployment protection rules, including required reviewers where available for the repository and plan.

Create a protected environment:

```text
Environment:
production
```

Configure:

- Required reviewers.
- Deployment branch restrictions.
- Environment secrets.
- Deployment permissions.
- Protection rules.

Example workflow:

```yaml
name: Production Deployment Authorization

on:
  workflow_dispatch:
    inputs:
      jira_change_key:
        description: Jira Production Change
        required: true
        type: string

jobs:
  validate-change:
    runs-on: ubuntu-latest

    steps:
      - name: Validate Jira Change Key
        env:
          JIRA_CHANGE_KEY: ${{ inputs.jira_change_key }}
        run: |
          set -euo pipefail

          if [[ ! "$JIRA_CHANGE_KEY" =~ ^ROBOSHOP-[0-9]+$ ]]; then
            echo "Invalid Jira issue key"
            exit 1
          fi

          echo "Jira change key format validated"

  production-approval:
    needs: validate-change

    runs-on: ubuntu-latest

    environment:
      name: production

    steps:
      - name: Confirm Protected Environment Approval
        run: |
          echo "Protected Production environment approved."
          echo "Continue with the separately validated deployment process."
```

This example demonstrates the protected environment mechanism.

For a production implementation, add a Jira API validation step that checks the actual approval state and release information.

The Jira approval and GitHub environment approval should not be treated as interchangeable unless the organization has explicitly designed and audited that integration.

# 21. GitLab CI/CD Production Approval

GitLab provides protected environments and deployment approvals depending on the product tier and configuration.

Example:

```yaml
stages:
  - validate
  - deploy

validate_change:
  stage: validate
  image: alpine:latest
  script:
    - |
      if [ -z "$JIRA_CHANGE_KEY" ]; then
        echo "Jira change key is required"
        exit 1
      fi
    - echo "Jira change key provided"

production_deployment:
  stage: deploy
  script:
    - echo "Production approval requirements must be satisfied."
    - echo "Execute the approved deployment procedure."
  environment:
    name: production
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

This is a simplified manual deployment example.

For actual production use:

- Protect the Production environment.
- Configure eligible deployment approvers.
- Restrict deployment permissions.
- Validate Jira approval.
- Validate the approved GitOps revision.
- Ensure the deployment job uses protected variables.
- Keep the approval audit trail.

# 22. Argo CD Production Deployment Authorization

Argo CD manages application deployment through GitOps.

Production authorization must protect the GitOps change and the ability to synchronize the Production application.

Recommended controls:

1. Protect the Production GitOps branch.
2. Require pull request reviews.
3. Restrict merge permissions.
4. Restrict Argo CD Production project permissions.
5. Restrict sync permissions.
6. Use separate Production and non-Production access roles.
7. Audit sync operations.
8. Record the Jira change key in the Git commit or pull request.

Example GitOps commit:

```bash
git commit \
  -m "ROBOSHOP-450 promote approved release to production"
```

Example Argo CD application inspection:

```bash
argocd app get roboshop-user-production
```

Example sync command:

```bash
argocd app sync roboshop-user-production
```

The sync command is an execution action, not an approval action.

Do not execute it until the required approval controls have passed.

# 23. Approval Expiration and Revalidation

Approval should apply to the specific change that was reviewed.

If important details change after approval, the change may need to be reviewed again.

Examples:

- New image digest.
- New GitOps commit.
- Different deployment window.
- New affected service.
- New database migration.
- Changed rollback procedure.
- New critical vulnerability.
- Material change in business impact.

Example policy:

```text
If the approved artifact changes:

1. Stop the deployment.
2. Update the Jira change record.
3. Attach the new artifact evidence.
4. Repeat affected reviews.
5. Obtain renewed approval.
6. Record the new approval history.
```

An approval must not be reused for a materially different release.

The organization should also define how long approvals remain valid.

# 24. Approval Rejection and Resubmission

A change can be rejected or returned for correction.

## Scenario 1: QA Rejection

Reason:

Critical regression test failed.

Action:

1. Record the failure.
2. Notify the application owner.
3. Fix the issue.
4. Build a new artifact if required.
5. Repeat affected tests.
6. Update the release evidence.
7. Resubmit for review.

## Scenario 2: Security Rejection

Reason:

A critical vulnerability violates the release policy.

Action:

1. Stop Production authorization.
2. Identify the vulnerable component.
3. Remediate or obtain an explicitly permitted exception.
4. Rebuild the image if needed.
5. Repeat security scanning.
6. Update Jira evidence.
7. Request renewed security review.

## Scenario 3: DBA Rejection

Reason:

Migration may cause incompatible schema changes.

Action:

1. Stop the approval workflow.
2. Review the migration design.
3. Validate backward compatibility.
4. Test the migration in a representative environment.
5. Update the recovery plan.
6. Obtain renewed DBA approval.

## Scenario 4: Business Rejection

Reason:

The deployment window conflicts with a critical business event.

Action:

1. Document the rejection reason.
2. Identify another deployment window.
3. Update Jira.
4. Reassess time-sensitive dependencies.
5. Obtain any required renewed approvals.

## Scenario 5: CAB Deferral

Reason:

Required technical evidence is incomplete.

Action:

1. Record the missing evidence.
2. Assign owners.
3. Complete the required checks.
4. Schedule another review.
5. Resubmit the change.

# 25. Emergency Production Approval

Emergency changes require a controlled process for urgent risk reduction or service restoration.

Examples:

- Critical Production outage.
- Active security incident.
- Serious data integrity issue.
- Urgent vulnerability requiring immediate mitigation.

Emergency process:

```text
Production Incident
       |
       v
Emergency Change Identified
       |
       v
Assess Immediate Risk
       |
       v
Contact Authorized Approver
       |
       v
Emergency Approval
       |
       v
Document Implementation Plan
       |
       v
Execute Authorized Change
       |
       v
Validate Recovery
       |
       v
Update Jira Change Record
       |
       v
Post-Implementation Review
```

Emergency controls should include:

- Authorized emergency approver.
- Documented reason.
- Scope limitation.
- Risk assessment.
- Recovery plan.
- Implementation evidence.
- Post-change review.
- Audit trail.

Do not bypass all authorization because a change is urgent.

Follow the organization's emergency change policy.

# 26. Approval Audit Trail

Maintain a complete history of the approval process.

Example:

| Event | Actor | Result |
|---|---|---|
| Change created | DevOps Engineer | Created |
| Technical review | Platform Lead | Approved |
| QA review | QA Lead | Approved |
| Security review | Security Lead | Approved |
| Application review | Application Owner | Approved |
| CAB review | CAB | Approved |
| Final approval | Authorized Approver | Approved |
| Deployment authorization | Release Manager | Granted |

The names and timestamps in the actual audit trail must come from Jira or the configured approval system.

Preserve:

- Who approved.
- What was approved.
- When approval occurred.
- What evidence was reviewed.
- What conditions were imposed.
- Whether approval was withdrawn.
- Whether the release changed after approval.

# 27. Production Go/No-Go Decision

Before Chapter 205 begins, the deployment team should perform a final go/no-go assessment.

## 27.1 Go Criteria

Proceed only when all mandatory requirements are satisfied.

```text
GO CHECKLIST

[ ] Jira Production Change exists.
[ ] Change is in the authorized state.
[ ] Required technical reviews completed.
[ ] QA approval completed.
[ ] Security approval completed.
[ ] Application owner approval completed.
[ ] Database approval completed when applicable.
[ ] Infrastructure approval completed when applicable.
[ ] Required CAB decision recorded.
[ ] Release ID verified.
[ ] Image digests verified.
[ ] GitOps revision verified.
[ ] Deployment window confirmed.
[ ] Rollback plan verified.
[ ] Monitoring readiness confirmed.
[ ] Deployment team available.
[ ] No unresolved blocking conditions.
```

## 27.2 No-Go Criteria

Do not deploy if:

- Required approval is missing.
- Approval has been revoked.
- Release artifact differs from the approved artifact.
- Staging validation failed.
- Critical security findings remain unresolved.
- Database migration is not approved.
- Infrastructure prerequisites are not met.
- Rollback strategy is unavailable.
- Deployment window is invalid.
- A mandatory dependency is unavailable.
- A production incident makes deployment unsafe.

## 27.3 Record the Final Decision

Example:

```text
PRODUCTION GO / NO-GO

Jira Change:
ROBOSHOP-450

Release:
roboshop-2026.11.03

Final Decision:
GO

Approval State:
Verified

Artifact:
Verified

Deployment Window:
Confirmed

Rollback:
Ready

Monitoring:
Ready

Deployment Owner:
Assigned

Decision Recorded:
In Jira
```

A GO decision authorizes the deployment process to proceed under the approved conditions. It does not mean the application has already been deployed.

# 28. Production Approval Troubleshooting

## Problem 1: Approver Cannot Access Jira Issue

Possible causes:

- Missing project permissions.
- Incorrect project role.
- Issue security restrictions.
- Inactive account.

Resolution:

1. Verify the issue key.
2. Check project permissions.
3. Check issue security.
4. Verify the approver's role.
5. Contact the Jira administrator.

## Problem 2: Approval Transition Is Missing

Possible causes:

- Incorrect workflow state.
- Missing transition permission.
- Required validator failed.
- Approval condition not satisfied.

Resolution:

1. Inspect the current workflow status.
2. Verify transition conditions.
3. Review validators.
4. Confirm the correct approval group.
5. Ask the Jira administrator to investigate.

## Problem 3: CI/CD Approval Gate Fails

Possible causes:

- Jira API authentication failure.
- Incorrect issue key.
- Missing custom field.
- Wrong status.
- API rate limit.
- Network connectivity issue.

Resolution:

1. Check the HTTP response.
2. Verify the Jira service account.
3. Verify required permissions.
4. Verify the issue fields.
5. Check the Jira workflow.
6. Retry only after confirming the failure cause.

Do not bypass the gate just to make a pipeline green.

## Problem 4: Approval Is Completed but Release Changed

Possible causes:

- New image digest.
- New GitOps commit.
- Modified configuration.
- Updated database migration.

Resolution:

1. Compare the approved and current release manifests.
2. Identify the differences.
3. Determine which reviews are affected.
4. Update the Jira change.
5. Obtain renewed approval where required.

## Problem 5: Approver Is Unavailable

Resolution:

1. Use the organization's designated backup approver.
2. Verify their authority.
3. Record the substitution.
4. Preserve the approval audit trail.

Do not allow an unauthorized team member to approve the release.

# 29. Production Approval Security Best Practices

Implement the following controls:

- Use individual user accounts for approval.
- Enforce MFA where supported and required.
- Restrict production approver groups.
- Use least-privilege permissions.
- Separate deployment and approval privileges.
- Protect CI/CD credentials.
- Protect GitOps branches.
- Audit approval transitions.
- Restrict direct Production cluster access.
- Require renewed approval after material changes.
- Review emergency access periodically.
- Retain records according to organizational policy.

# 30. End-to-End Example

Let's connect Chapter 203 and Chapter 204 using one RoboShop release.

## Step 1: Jira Change Created

```text
ROBOSHOP-450

Production Release:
roboshop-2026.11.03

Status:
Submitted
```

## Step 2: Technical Review

DevOps reviews:

- EKS configuration.
- Helm values.
- ECR image digests.
- GitOps PR.
- Rollback readiness.

Result: Approved.

## Step 3: QA Review

QA reviews:

- Integration tests.
- Regression tests.
- Business workflows.
- Staging test evidence.

Result: Approved.

## Step 4: Security Review

Security reviews:

- SonarQube.
- Trivy.
- IAM permissions.
- Secret management.

Result: Approved.

## Step 5: Database Review

The DBA checks whether the release contains database changes.

If no database change exists, record that DBA approval is not required according to policy.

If a database change exists, obtain the required approval.

## Step 6: Application Owner Review

The application owner confirms application readiness and accepts the documented impact.

Result: Approved.

## Step 7: CAB Review

The CAB reviews the complete change record when required.

Result: Approved.

## Step 8: Final Approval

The authorized approver confirms that all mandatory requirements have been met.

Result: Approved.

## Step 9: Go/No-Go

The deployment team verifies:

- Correct release.
- Correct image digests.
- Correct GitOps commit.
- Approved deployment window.
- Available monitoring.
- Rollback readiness.

Result: GO.

## Step 10: Chapter 205

The approved Production deployment can now begin.

# 31. Chapter Completion Checklist

## Approval Process

- [ ] Production approval architecture understood.
- [ ] Approval responsibilities defined.
- [ ] Approval matrix created.
- [ ] Separation of duties established.
- [ ] Jira change reviewed.
- [ ] Technical approval completed.
- [ ] QA approval completed.
- [ ] Security approval completed.
- [ ] Database approval completed where required.
- [ ] Infrastructure approval completed where required.
- [ ] Application owner approval completed.
- [ ] Business approval completed where required.
- [ ] CAB review completed where required.

## Jira Workflow

- [ ] Approval statuses configured.
- [ ] Approval transitions configured.
- [ ] Jira permissions reviewed.
- [ ] Required approval evidence recorded.
- [ ] Rejection workflow tested.
- [ ] Resubmission workflow tested.
- [ ] Audit trail verified.

## CI/CD and GitOps

- [ ] Jenkins approval gate reviewed.
- [ ] GitHub Actions protected environment configured.
- [ ] GitLab protected environment reviewed.
- [ ] Jira approval validation designed.
- [ ] GitOps branch protection enabled.
- [ ] Argo CD access restricted.
- [ ] Production deployment authorization verified.

## Final Readiness

- [ ] Approval is valid for the exact release.
- [ ] Release artifact matches approved digest.
- [ ] Deployment window confirmed.
- [ ] Rollback plan ready.
- [ ] Monitoring ready.
- [ ] Final GO/NO-GO recorded.

---

# 32. Chapter Summary

In this chapter, we implemented the Production approval process for the RoboShop microservices platform.

We covered:

1. Production approval architecture.
2. Approval responsibilities.
3. Approval matrix.
4. Separation of duties.
5. Jira change review.
6. Technical approval.
7. QA approval.
8. Security approval.
9. Database approval.
10. Infrastructure approval.
11. Application owner approval.
12. CAB review.
13. Risk-based approval.
14. Jira approval workflow.
15. Jira approval evidence.
16. Automated approval validation.
17. Jenkins approval gate.
18. GitHub Actions approval gate.
19. GitLab CI/CD approval.
20. Argo CD deployment authorization.
21. Approval expiration.
22. Rejection and resubmission.
23. Emergency approval.
24. Approval audit trail.
25. Production go/no-go.
26. Troubleshooting.
27. Security best practices.
28. End-to-end approval implementation.

**Key principle:** A Production deployment must be authorized for the exact release, image digests, and GitOps revision that will be deployed. Approval of one release must not silently authorize a different release.

---

# 33. Next Chapter

## CHAPTER 205 — PRODUCTION DEPLOYMENT

**Filename:** `205-Production-Deployment.md`

The next chapter will cover the actual execution of the approved Production release.

Planned topics:

1. Production deployment architecture.
2. Production AWS account verification.
3. EKS cluster access and context validation.
4. Production namespace and resource readiness.
5. IAM and IRSA verification.
6. ECR image verification.
7. Production configuration and secrets.
8. Database readiness.
9. Redis and messaging readiness.
10. Helm chart and values validation.
11. GitOps production repository structure.
12. Argo CD Production application configuration.
13. Immutable image promotion.
14. Deployment execution.
15. Microservice deployment order and dependencies.
16. Kubernetes rollout verification.
17. Service and EndpointSlice verification.
18. ALB and ingress verification.
19. DNS and TLS verification.
20. Application smoke testing.
21. End-to-end business workflow testing.
22. Production monitoring validation using Prometheus, Grafana, and ELK.
23. Deployment evidence collection.
24. Deployment failure handling.
25. Production handover and completion.

**The next file must be exactly `205-Production-Deployment.md`.**