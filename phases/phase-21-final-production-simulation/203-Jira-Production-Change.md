# CHAPTER 203 — JIRA PRODUCTION CHANGE

**Filename:** `203-Jira-Production-Change.md`

---

# 1. Chapter Objective

In a production environment, deploying an application is not just a technical activity.

Every Production change must be documented, tracked, assessed, and approved through the organization's change-management process.

Jira provides a centralized way to manage:

- Production change requests.
- Application releases.
- Infrastructure modifications.
- Kubernetes configuration changes.
- Database changes.
- Security changes.
- Deployment schedules.
- Implementation plans.
- Testing evidence.
- Risk assessments.
- Rollback procedures.
- Change history.
- Communication and ownership.

In this chapter, we will implement a practical Jira Production Change Management workflow for the RoboShop microservices platform.

**Important distinction:**

- Chapter 203: Create and manage the Jira Production change request.
- Chapter 204: Obtain Production deployment approval.
- Chapter 205: Execute the Production deployment.

Do not combine these three activities into one chapter.

# 2. Production Change Management Architecture

The change-management workflow connects development, CI/CD, GitOps, Jira, and Production operations.

```text
Developer
    |
    v
Git Repository
    |
    v
CI Pipeline
    |
    +--> Unit Tests
    +--> SonarQube
    +--> Trivy
    +--> Docker Build
    +--> ECR Publishing
    |
    v
QA Deployment
    |
    v
Staging Deployment
    |
    v
Staging Validation
    |
    v
Create Jira Production Change
    |
    +--> Change Description
    +--> Release Information
    +--> Affected Services
    +--> Risk Assessment
    +--> Implementation Plan
    +--> Validation Plan
    +--> Rollback Plan
    +--> Deployment Window
    |
    v
Jira Change Record
    |
    v
Chapter 204
Production Approval
    |
    v
Chapter 205
Production Deployment
```

# 3. Understand Jira for DevOps

Jira is an issue and workflow management platform.

DevOps teams commonly use Jira to manage:

- Stories.
- Tasks.
- Bugs.
- Change requests.
- Incidents.
- Problems.
- Release activities.

The exact issue types and workflows depend on the Jira product and the organization's configuration.

For this implementation, we will use a dedicated Production Change issue type.

Example:

```text
Project: ROBOSHOP

Issue Type: Production Change

Issue Key: ROBOSHOP-450

Summary:
Production Release - RoboShop 2026.11.03

Environment:
Production

Release:
roboshop-2026.11.03

Status:
Draft
```

The issue key is an illustrative example. Jira generates the actual key when the issue is created.

# 4. Jira Project Structure

Create or identify the Jira project used for RoboShop production changes.

Example project:

```text
Project Name:
RoboShop Platform

Project Key:
ROBOSHOP

Project Type:
Software or Service Management
```

The Jira project should provide access to:

- Production Change issue type.
- Release tracking.
- Change workflow.
- Change history.
- Attachments.
- Comments.
- Activity history.
- Deployment evidence.

If the organization uses Jira Service Management, use its change-management capabilities where available.

If the organization uses Jira Software, implement the required issue types and workflow using its available project configuration.

# 5. Create a Production Change Issue Type

The Production Change issue type represents a planned modification to a live environment.

Examples:

| Change | Example |
|---|---|
| Application release | Deploy new User Service version |
| Infrastructure | Update EKS node group |
| Database | Apply approved schema migration |
| Networking | Update ALB ingress rules |
| Security | Rotate application credentials |
| Configuration | Change production Helm values |
| Platform | Upgrade Kubernetes add-on |

Create an issue type named:

```text
Production Change
```

If the Jira project does not allow custom issue types, use the organization's approved change issue type.

# 6. Define the Production Change Workflow

A controlled workflow ensures that every change moves through the required lifecycle.

Example workflow:

```text
Draft
  |
  v
Submitted
  |
  v
Under Review
  |
  +----> Rejected
  |
  v
Assessment Complete
  |
  v
Pending Approval
  |
  +----> Changes Requested
  |             |
  |             v
  |          Submitted
  |
  v
Approved
  |
  v
Scheduled
  |
  v
Implementation
  |
  +----> Failed
  |
  v
Implemented
  |
  v
Validation
  |
  +----> Unsuccessful
  |
  v
Completed
```

This is an example workflow. Configure the actual Jira statuses and transitions to match the organization's change-management policy.

Chapter 204 will cover the approval process in detail.

# 7. Define Jira Change Request Fields

A Production Change issue should capture enough information for reviewers and operators to understand the planned change.

Recommended fields:

| Field | Purpose |
|---|---|
| Summary | Short change description |
| Description | Detailed change information |
| Environment | Target environment |
| Change Type | Standard, normal, or emergency |
| Change Risk | Assessed risk |
| Business Reason | Why the change is required |
| Release ID | Release identifier |
| Git Commit | Source revision |
| Image Digest | Immutable artifact reference |
| Affected Services | Microservices involved |
| Implementation Plan | Deployment instructions |
| Validation Plan | Post-deployment checks |
| Rollback Plan | Recovery procedure |
| Planned Start | Deployment start |
| Planned End | Expected completion |
| Change Owner | Responsible engineer |
| Technical Owner | Technical contact |
| Dependencies | Related changes |
| Evidence | Test and deployment artifacts |

Not every Jira edition exposes these fields by default. Custom fields may need to be created by the Jira administrator.

# 8. Create a Production Change Request

Let's create an example Production Change for RoboShop.

## 8.1 Change Request Details

```text
Project:
ROBOSHOP

Issue Type:
Production Change

Summary:
Production Release - RoboShop 2026.11.03

Environment:
Production

Change Type:
Normal Change

Change Owner:
DevOps Engineer

Affected Application:
RoboShop Microservices

Release ID:
roboshop-2026.11.03
```

## 8.2 Business Reason

Example:

```text
This change is required to deploy the approved
RoboShop application release into the Production
environment.

The release contains approved application changes,
bug fixes, configuration updates, and associated
deployment artifacts.

The same immutable container image digests that
passed QA and Staging validation will be promoted
to Production.
```

## 8.3 Technical Description

Example:

```text
The release will be deployed to the Production
Amazon EKS cluster using Helm and Argo CD.

The production configuration will be updated through
the approved GitOps repository.

The deployment will use the approved release image
digests stored in Amazon ECR.

The rollout will be performed in controlled batches.

Application readiness, service health, ingress
availability, and business workflows will be validated
after deployment.
```

# 9. Record the Release Information

The Jira issue must identify the exact application release.

Example:

```yaml
release:
  id: roboshop-2026.11.03

  source:
    repository: roboshop-microservices
    commit: APPROVED_COMMIT_SHA

  gitops:
    repository: roboshop-gitops
    commit: APPROVED_GITOPS_COMMIT

  environment: production

  deployment:
    platform: aws-eks
    namespace: production
    method: argocd
    strategy: rolling-update
```

The actual release record must contain the real repository URLs, commit SHAs, and release identifiers.

Do not put passwords, access keys, tokens, or other credentials in Jira.

# 10. Record Immutable ECR Image Information

Every affected microservice should have its approved image reference.

Example:

```yaml
services:
  user:
    repository: roboshop/user
    image_digest: sha256:REAL_USER_DIGEST

  catalog:
    repository: roboshop/catalog
    image_digest: sha256:REAL_CATALOG_DIGEST

  cart:
    repository: roboshop/cart
    image_digest: sha256:REAL_CART_DIGEST

  orders:
    repository: roboshop/orders
    image_digest: sha256:REAL_ORDERS_DIGEST

  payment:
    repository: roboshop/payment
    image_digest: sha256:REAL_PAYMENT_DIGEST
```

For the complete 20+ microservices application, record every service included in the release.

Avoid using the mutable `latest` tag as the only release reference.

The Jira change record should make it possible to identify exactly which artifacts were deployed.

# 11. Record Affected Services

Not every release modifies every microservice.

Record the actual impact.

Example:

| Service | Change | Impact |
|---|---|---|
| User | New image | Application behavior |
| Catalog | Configuration update | Product APIs |
| Cart | No change | Not affected |
| Orders | New image | Order processing |
| Payment | Configuration update | Payment integration |
| Inventory | No change | Not affected |
| Notification | New image | Event processing |

The table is illustrative. Populate it using the actual Git changes and release manifest.

For each affected service, document:

- Previous version.
- New version.
- Image digest.
- Configuration changes.
- Dependencies.
- Validation steps.
- Rollback reference.

# 12. Production Change Classification

Organizations commonly classify changes into different categories.

## 12.1 Standard Change

A repeatable, pre-authorized, low-risk change.

Example:

- Routine configuration update.
- Approved certificate renewal.
- Repeated maintenance task.

Standard changes still require the controls defined by the organization's policy.

## 12.2 Normal Change

A planned change that requires assessment and approval.

Example:

- New microservice release.
- EKS configuration update.
- Database schema migration.
- Application networking change.

## 12.3 Emergency Change

A change required to restore or protect Production when normal scheduling is not appropriate.

Example:

- Critical security vulnerability.
- Production service outage.
- Urgent data integrity issue.

Emergency changes must follow the organization's emergency change process, including required authorization and retrospective documentation where applicable.

Do not classify every urgent deployment as an emergency change simply to bypass review.

# 13. Production Change Risk Assessment

Risk assessment identifies what could go wrong and how the team will reduce the risk.

Evaluate:

- Business impact.
- Application impact.
- Infrastructure impact.
- Security impact.
- Database impact.
- Network impact.
- Availability.
- Recoverability.
- Change complexity.

Example:

```yaml
risk_assessment:
  change: roboshop-production-release

  impact:
    application: medium
    infrastructure: low
    database: low
    security: medium
    availability: medium

  risk_factors:
    - Multiple microservices are included.
    - Application configuration changes are included.
    - Existing production traffic must remain available.

  mitigations:
    - Use immutable image digests.
    - Deploy in controlled batches.
    - Validate readiness before continuing.
    - Monitor HTTP error rates.
    - Keep a rollback plan available.

  overall_risk: medium
```

The risk classification must be based on the organization's approved risk methodology, not simply copied from this example.

# 14. Document the Implementation Plan

The implementation plan must explain exactly what the deployment team will do.

Example:

```text
IMPLEMENTATION PLAN

1. Verify the Production AWS account.
2. Verify the Production EKS cluster.
3. Confirm the approved deployment window.
4. Verify the approved release manifest.
5. Verify all ECR image digests.
6. Review the production GitOps PR.
7. Confirm the database migration plan.
8. Confirm Redis and messaging availability.
9. Start the approved Argo CD deployment.
10. Deploy the approved microservices in batches.
11. Monitor rollout status.
12. Validate Kubernetes resources.
13. Validate ingress and application endpoints.
14. Execute approved smoke tests.
15. Verify critical business workflows.
16. Capture deployment evidence.
17. Update the Jira change record.
```

The commands and detailed execution steps belong in the deployment runbook and Chapter 205.

# 15. Document the Validation Plan

The validation plan explains how the team will determine whether the change succeeded.

Example:

```text
VALIDATION PLAN

Infrastructure:
- Verify EKS node health.
- Verify required system components.
- Verify database connectivity.
- Verify messaging availability.

Kubernetes:
- Verify Deployment availability.
- Verify pod readiness.
- Verify Service endpoints.
- Verify HPA and PDB.

Application:
- Verify frontend access.
- Verify authentication.
- Verify product APIs.
- Verify cart operations.
- Verify order workflow.
- Verify payment integration using the approved test mode.

Networking:
- Verify ALB target health.
- Verify ingress routing.
- Verify DNS resolution.
- Verify TLS certificate.

Observability:
- Verify Prometheus targets.
- Verify Grafana dashboards.
- Verify ELK log ingestion.
- Verify production alerts.
```

Validation must include business-level checks, not only Kubernetes resource checks.

# 16. Document the Rollback Plan

Every Production change must have a recovery strategy.

Example:

```text
ROLLBACK PLAN

1. Stop further rollout.
2. Identify the affected services.
3. Identify the last known-good release.
4. Verify the previous image digests.
5. Revert the Production GitOps promotion commit.
6. Obtain required emergency authorization.
7. Synchronize Argo CD with the restored configuration.
8. Verify Deployment availability.
9. Verify service endpoints.
10. Verify ingress and application health.
11. Run smoke tests.
12. Confirm monitoring recovery.
13. Record rollback evidence in Jira.
```

Important:

Application rollback does not automatically reverse a database migration.

Database recovery must follow the separately approved database recovery procedure.

# 17. Define the Deployment Window

The Jira change record should contain the planned deployment window.

Example:

```text
Planned Start:
2026-11-03 22:00 IST

Planned End:
2026-11-03 23:30 IST

Time Zone:
Asia/Kolkata

Expected Duration:
90 minutes
```

This is an illustrative schedule.

The actual deployment window should be agreed with application owners, business stakeholders, operations, and the change-management team.

Also document:

- Maintenance window.
- Business blackout periods.
- On-call coverage.
- Communication schedule.
- Expected customer impact.
- Dependencies on other changes.

# 18. Identify Dependencies

A Production change may depend on other changes.

Example:

```text
RoboShop Production Release
|
+-- Approved ECR Images
|
+-- Production Helm Values
|
+-- Production Secrets
|
+-- Database Readiness
|
+-- Redis Availability
|
+-- Messaging Availability
|
+-- EKS Capacity
|
+-- ALB Configuration
|
+-- DNS and TLS
|
+-- Monitoring Readiness
```

Record dependent Jira issues using the organization's issue-linking conventions.

Examples:

- Blocks.
- Is blocked by.
- Relates to.
- Depends on.

Do not proceed if a required dependency is incomplete or unverified.

# 19. Attach Supporting Evidence

A Production change should include evidence that the release has been tested.

Recommended attachments:

```text
ROBOSHOP-450/
|
+-- release-manifest.yaml
+-- approved-image-digests.txt
+-- git-commit.txt
+-- qa-test-report.pdf
+-- staging-test-report.pdf
+-- sonarqube-report.pdf
+-- trivy-report.json
+-- performance-test-report.pdf
+-- database-migration-plan.md
+-- deployment-runbook.md
+-- rollback-runbook.md
+-- monitoring-dashboard-reference.txt
```

Use approved storage locations and attach links where reports are too large.

Do not upload sensitive credentials, private keys, or unredacted customer data.

# 20. Create a Production Change in Jira UI

The following steps describe a typical Jira workflow. Exact menu names depend on the Jira product and project configuration.

Step 1: Open the Jira project.

```text
Jira
  |
  v
Projects
  |
  v
RoboShop Platform
```

Step 2: Select Create.

Step 3: Select the issue type.

```text
Production Change
```

Step 4: Enter the summary.

```text
Production Release - RoboShop 2026.11.03
```

Step 5: Complete the description.

Include:

- Business reason.
- Technical description.
- Affected services.
- Release information.
- Implementation plan.
- Validation plan.
- Rollback plan.

Step 6: Complete the custom fields.

Enter:

- Environment: Production.
- Change Type: Normal.
- Risk: Assessed value.
- Planned Start.
- Planned End.
- Change Owner.

Step 7: Attach the supporting evidence.

Step 8: Submit the issue.

Example resulting issue:

```text
ROBOSHOP-450

Production Release - RoboShop 2026.11.03

Status: Draft
Environment: Production
Change Type: Normal
Release: roboshop-2026.11.03
```

# 21. Jira Change Request Template

Use the following template as the standard description for Production Change issues.

```markdown
# Production Change Request

## 1. Change Summary

Change Title:
Release ID:
Environment:
Change Type:
Change Owner:
Technical Owner:

## 2. Business Reason

Why is this change required?

## 3. Technical Description

What is changing?

## 4. Release Information

Application:
Source Repository:
Source Commit:
GitOps Repository:
GitOps Commit:
ECR Image Digests:

## 5. Affected Services

List every affected microservice.

## 6. Risk Assessment

Business Impact:
Application Impact:
Infrastructure Impact:
Security Impact:
Database Impact:
Overall Risk:

## 7. Dependencies

List all dependencies and linked Jira issues.

## 8. Implementation Plan

List the deployment steps in execution order.

## 9. Validation Plan

Infrastructure:
Kubernetes:
Application:
Networking:
Monitoring:
Business Workflows:

## 10. Rollback Plan

Describe the rollback steps and recovery conditions.

## 11. Deployment Window

Planned Start:
Planned End:
Time Zone:

## 12. Customer Impact

Expected Downtime:
Expected User Impact:
Communication Required:

## 13. Evidence

QA Results:
Staging Results:
Security Results:
Performance Results:
Release Manifest:

## 14. Approval Tracking

Required Reviewers:
Approval Status:
Related Approval Issue:

## 15. Completion Record

Actual Start:
Actual End:
Implementation Result:
Validation Result:
Final Status:
```

# 22. Jira Automation

Jira automation can reduce manual work and improve change-record consistency.

Possible automation rules:

| Trigger | Action |
|---|---|
| Change issue created | Assign change owner |
| Status becomes Submitted | Notify reviewers |
| Required fields missing | Prevent or flag submission |
| Approval completed | Notify deployment owner |
| Status becomes Scheduled | Notify stakeholders |
| Status becomes Implemented | Request validation update |
| Status becomes Completed | Notify change stakeholders |

Configure these rules only after confirming the available Jira automation features and permissions.

## 22.1 Example Automation: Notify Reviewers

Conceptual workflow:

```text
Trigger:
Issue transitioned to Submitted

Condition:
Issue Type = Production Change

Action:
Send notification to the configured review group
```

## 22.2 Example Automation: Required Field Validation

Conceptual workflow:

```text
Trigger:
Issue transitioned to Submitted

Conditions:
- Release ID is not empty.
- Environment is Production.
- Implementation Plan is not empty.
- Validation Plan is not empty.
- Rollback Plan is not empty.

If missing:
- Notify issue owner.
- Return issue for completion.
```

For strict enforcement, use Jira workflow validators or an equivalent supported configuration. A notification rule alone does not prevent an invalid transition.

# 23. Integrate CI/CD with Jira

Jira can be linked with source repositories and CI/CD systems.

The goal is to connect:

```text
Jira Change
    |
    v
Git Commit
    |
    v
CI Pipeline
    |
    v
ECR Artifact
    |
    v
GitOps Commit
    |
    v
Argo CD Deployment
    |
    v
Production Evidence
```

The integration should make the release traceable.

## 23.1 Link Jira Issue to Git

Include the Jira issue key in the branch name or commit message according to the organization's convention.

Example branch:

```bash
git checkout -b feature/ROBOSHOP-450-production-release
```

Example commit:

```bash
git commit -m "ROBOSHOP-450 update production release manifest"
```

This helps Jira integrations associate development activity with the change issue.

## 23.2 Link Jira Issue to CI Pipeline

The pipeline can publish:

- Build number.
- Source commit.
- Test status.
- Security scan status.
- Artifact reference.
- Release identifier.

Use the organization's configured Jira and CI integration.

## 23.3 Link Jira Issue to GitOps

The production GitOps promotion commit should reference the Jira issue.

Example:

```bash
git commit -m "ROBOSHOP-450 promote release to production"
```

The GitOps repository should retain the exact image digest and environment configuration.

## 23.4 Link Jira Issue to Argo CD

Capture:

- Argo CD application name.
- Git revision.
- Sync status.
- Health status.
- Deployment timestamp.
- Relevant application events.

Example:

```text
Argo CD Application:
roboshop-user-production

Git Revision:
APPROVED_GITOPS_COMMIT

Sync Status:
Synced

Health Status:
Healthy
```

These values must be recorded from the actual deployment.

# 24. Jira REST API Integration

Jira REST APIs can be used to automate change creation and updates.

The exact API version, endpoints, authentication method, and field identifiers depend on the Jira Cloud or Data Center deployment.

Do not assume every Jira instance uses the same API contract.

## 24.1 Store Credentials Securely

For a CI/CD integration, use a secret manager or the CI platform's protected secret store.

Example environment variables:

```bash
export JIRA_BASE_URL="https://YOUR-DOMAIN.atlassian.net"
export JIRA_EMAIL="YOUR_SERVICE_ACCOUNT_EMAIL"
export JIRA_API_TOKEN="LOAD_FROM_SECRET_MANAGER"
```

Do not hardcode the token into scripts or commit it to Git.

## 24.2 Create a Jira Issue

Example request structure for a Jira Cloud instance using an issue-creation endpoint:

```bash
curl --fail-with-body \
  --request POST \
  --url "${JIRA_BASE_URL}/rest/api/3/issue" \
  --user "${JIRA_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --data '{
    "fields": {
      "project": {
        "key": "ROBOSHOP"
      },
      "issuetype": {
        "name": "Production Change"
      },
      "summary": "Production Release - RoboShop",
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "paragraph",
            "content": [
              {
                "type": "text",
                "text": "Production change request for the approved RoboShop release."
              }
            ]
          }
        ]
      }
    }
  }'
```

The example uses a Jira Cloud-style rich-text description.

The issue type must exist in the project, and required custom fields must be supplied where configured.

## 24.3 Capture the Issue Key

A successful issue-creation response includes the generated issue key.

Example:

```json
{
  "id": "100450",
  "key": "ROBOSHOP-450",
  "self": "https://YOUR-DOMAIN.atlassian.net/rest/api/3/issue/100450"
}
```

This is an illustrative response.

Capture the returned key in the CI/CD system rather than assuming a fixed issue number.

## 24.4 Read the Issue

```bash
curl --fail-with-body \
  --request GET \
  --url "${JIRA_BASE_URL}/rest/api/3/issue/ROBOSHOP-450" \
  --user "${JIRA_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Accept: application/json"
```

## 24.5 Add a Comment

```bash
curl --fail-with-body \
  --request POST \
  --url "${JIRA_BASE_URL}/rest/api/3/issue/ROBOSHOP-450/comment" \
  --user "${JIRA_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --data '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            {
              "type": "text",
              "text": "Staging validation completed. Evidence is attached to the change record."
            }
          ]
        }
      ]
    }
  }'
```

## 24.6 API Integration Considerations

- Use a dedicated service account.
- Apply least-privilege permissions.
- Store tokens securely.
- Handle HTTP errors.
- Handle API rate limits.
- Validate required custom fields.
- Avoid duplicate change creation on pipeline retries.
- Log Jira issue keys.
- Do not log credentials.
- Record API failures as pipeline failures where the integration is a mandatory release control.

# 25. CI/CD Change Request Gate

A Production pipeline should verify that a valid change request exists before proceeding with a controlled deployment.

Example conceptual workflow:

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
Validate Issue Exists
          |
          v
Validate Environment = Production
          |
          v
Validate Release ID
          |
          v
Validate Required Fields
          |
          v
Chapter 204 Approval Gate
          |
          v
Production Deployment
```

The Jira change record is not itself proof that the change has been approved.

Approval validation must be implemented using the organization's approved Jira workflow and Chapter 204's approval controls.

# 26. Example Jenkins Jira Validation Stage

This example assumes the Jenkins environment already has a securely configured Jira credential.

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'JIRA_CHANGE_KEY',
            defaultValue: '',
            description: 'Approved Production Change issue key'
        )
    }

    environment {
        JIRA_URL = 'https://YOUR-DOMAIN.atlassian.net'
    }

    stages {
        stage('Validate Jira Change') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jira-service-account',
                        usernameVariable: 'JIRA_EMAIL',
                        passwordVariable: 'JIRA_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        if [ -z "$JIRA_CHANGE_KEY" ]; then
                            echo "ERROR: Jira change key is required"
                            exit 1
                        fi

                        HTTP_CODE=$(curl \
                          --silent \
                          --output jira-response.json \
                          --write-out "%{http_code}" \
                          --user "$JIRA_EMAIL:$JIRA_TOKEN" \
                          --header "Accept: application/json" \
                          "$JIRA_URL/rest/api/3/issue/$JIRA_CHANGE_KEY")

                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "ERROR: Jira issue lookup failed"
                            exit 1
                        fi

                        echo "Jira change issue exists."
                    '''
                }
            }
        }
    }
}
```

This stage only checks whether the Jira issue exists and can be retrieved.

A complete production gate must also validate:

- Issue type.
- Project.
- Environment.
- Release ID.
- Change status.
- Approval status.
- Deployment window.
- Required evidence.

The exact validation logic depends on the Jira workflow and custom-field identifiers.

# 27. Example GitHub Actions Jira Validation

The following example retrieves a Jira issue using GitHub Actions secrets.

```yaml
name: Production Change Validation

on:
  workflow_dispatch:
    inputs:
      jira_change_key:
        description: Jira Production Change issue key
        required: true
        type: string

jobs:
  validate-change:
    runs-on: ubuntu-latest

    steps:
      - name: Validate Jira Change
        env:
          JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
          JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
          JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
          JIRA_CHANGE_KEY: ${{ inputs.jira_change_key }}

        run: |
          set -euo pipefail

          if [ -z "$JIRA_CHANGE_KEY" ]; then
            echo "Jira change key is required"
            exit 1
          fi

          HTTP_CODE=$(curl \
            --silent \
            --output jira-response.json \
            --write-out "%{http_code}" \
            --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
            --header "Accept: application/json" \
            "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_CHANGE_KEY")

          if [ "$HTTP_CODE" != "200" ]; then
            echo "Jira validation failed"
            exit 1
          fi

          echo "Jira issue retrieved successfully"
```

This is an initial integration example, not a complete production approval gate.

# 28. Change Request Traceability

A production change should provide end-to-end traceability.

Example:

```text
Jira:
ROBOSHOP-450
    |
    v
Git Commit:
abc123...
    |
    v
CI Pipeline:
build-1250
    |
    v
ECR:
roboshop/user@sha256:...
    |
    v
GitOps Commit:
def456...
    |
    v
Argo CD:
roboshop-user-production
    |
    v
EKS:
production namespace
    |
    v
Validation:
Smoke tests and monitoring
    |
    v
Jira:
Implementation evidence
```

A reviewer should be able to follow the complete chain without guessing which build or image was deployed.

# 29. Change Request Communication

Production changes may affect several teams.

Identify the required communication groups:

- Application team.
- DevOps team.
- QA team.
- Security team.
- Database team.
- Platform team.
- Network team.
- Operations team.
- Business stakeholders.

Example communication record:

```text
Change:
ROBOSHOP-450

Release:
roboshop-2026.11.03

Environment:
Production

Planned Window:
Approved deployment window

Impact:
As documented in the change assessment.

Owner:
Assigned DevOps Engineer

Communication:
Pre-deployment notification
Deployment-start notification
Completion notification
Incident notification if required
```

Do not announce a successful deployment before validation has completed.

# 30. Change Request Failure Scenarios

## Scenario 1: Missing Release ID

Problem:

The change request does not identify the release.

Action:

1. Return the issue for correction.
2. Identify the approved release.
3. Add the release identifier.
4. Attach the release manifest.
5. Continue the workflow.

## Scenario 2: Missing Rollback Plan

Problem:

The implementation team has documented deployment steps but no rollback procedure.

Action:

1. Do not submit an incomplete change.
2. Identify the last known-good version.
3. Verify the previous image digest.
4. Review database compatibility.
5. Document the rollback steps.
6. Attach the recovery runbook.

## Scenario 3: Staging Evidence Missing

Problem:

The Production change has been created, but Staging validation evidence is missing.

Action:

1. Identify the missing tests.
2. Obtain the Staging results.
3. Attach the relevant reports.
4. Document exceptions.
5. Continue only through the approved workflow.

## Scenario 4: Deployment Window Conflict

Problem:

The requested deployment overlaps with another critical Production change.

Action:

1. Identify the conflicting change.
2. Assess shared dependencies.
3. Coordinate with the change owners.
4. Reschedule if required.
5. Update the Jira record.

## Scenario 5: Jira Unavailable

Problem:

Jira is unavailable when the team needs to record a Production change.

Action:

1. Follow the organization's documented change-management outage procedure.
2. Use only an authorized alternative record if permitted.
3. Capture the required change details and approvals.
4. Restore the Jira record when the service returns.
5. Preserve the audit trail.

Never assume that Jira being unavailable automatically authorizes a Production deployment.

# 31. Change Management Metrics

Track change-management effectiveness.

| Metric | Meaning |
|---|---|
| Change Volume | Number of changes over a period |
| Change Approval Time | Time required to reach an approval decision |
| Change Failure Rate | Percentage of changes resulting in failures |
| Emergency Change Rate | Proportion of emergency changes |
| Change Success Rate | Percentage of changes completed successfully |
| Rollback Rate | Percentage of changes requiring rollback |
| Change Lead Time | Time from change initiation to implementation |

These metrics help identify process bottlenecks and release risks.

Do not use approval speed alone as a measure of change-management quality.

# 32. Production Change Security Controls

Protect the change-management process.

Required controls may include:

- Role-based Jira access.
- Restricted change transitions.
- Separation of duties.
- Auditable approval records.
- Secure evidence storage.
- Credential protection.
- Restricted production deployment permissions.
- Controlled emergency changes.
- Change history retention.

A person who creates a change should not automatically receive permission to approve and deploy it.

Follow organizational separation-of-duties policies.

# 33. Production Change Completion

After implementation, the change owner should update the Jira record with actual results.

Example:

```text
Change:
ROBOSHOP-450

Release:
roboshop-2026.11.03

Implementation:
Completed

Validation:
Passed

Application Health:
Healthy

Monitoring:
Validated

Rollback:
Not required

Evidence:
Attached

Actual Start:
Recorded

Actual End:
Recorded

Final Result:
Successful
```

If implementation fails, record:

- Failure time.
- Affected services.
- Customer impact.
- Incident reference.
- Rollback decision.
- Recovery status.
- Supporting evidence.

Do not mark a change Completed merely because the deployment command finished.

# 34. Chapter Completion Checklist

## Jira Configuration

- [ ] Jira project identified.
- [ ] Production Change issue type configured.
- [ ] Change workflow reviewed.
- [ ] Required fields configured.
- [ ] Project permissions reviewed.
- [ ] Change owner assigned.

## Change Request

- [ ] Business reason documented.
- [ ] Technical description completed.
- [ ] Environment identified.
- [ ] Release ID recorded.
- [ ] Source commit recorded.
- [ ] GitOps commit recorded.
- [ ] ECR digests recorded.
- [ ] Affected services listed.
- [ ] Dependencies identified.

## Risk and Execution

- [ ] Risk assessment completed.
- [ ] Implementation plan documented.
- [ ] Validation plan documented.
- [ ] Rollback plan documented.
- [ ] Deployment window entered.
- [ ] Customer impact assessed.
- [ ] Communication plan documented.

## Evidence and Integration

- [ ] QA evidence attached.
- [ ] Staging evidence attached.
- [ ] Security evidence attached.
- [ ] Performance evidence attached.
- [ ] Release manifest attached.
- [ ] Git integration verified.
- [ ] CI/CD integration reviewed.
- [ ] Jira issue key traceable.

## Governance

- [ ] Required reviewers identified.
- [ ] Change submitted through the workflow.
- [ ] Approval status tracked.
- [ ] Change history preserved.

---

# 35. Chapter Summary

This chapter implemented the Production Change Management process for the RoboShop microservices platform.

Covered areas:

1. Jira project structure.
2. Production Change issue type.
3. Change-management workflow.
4. Production change request fields.
5. Release identification.
6. ECR image digest tracking.
7. Affected microservice inventory.
8. Change classification.
9. Risk assessment.
10. Implementation planning.
11. Validation planning.
12. Rollback planning.
13. Deployment window management.
14. Dependency tracking.
15. Evidence collection.
16. Jira UI workflow.
17. Standard change request template.
18. Jira automation.
19. Jira REST API integration.
20. Jenkins integration.
21. GitHub Actions integration.
22. Git and GitOps traceability.
23. Change communication.
24. Failure scenarios.
25. Change-management metrics.
26. Security and governance.
27. Change completion and evidence.

**Key principle:** Every Production change must be traceable from the original request through the source commit, CI pipeline, immutable artifact, GitOps promotion, deployment, and final validation.

---

# 36. Next Chapter

**CHAPTER 204 — PRODUCTION APPROVAL**

Filename:

`204-Production-Approval.md`

The next chapter will cover:

- Production change approval workflow.
- Change Advisory Board (CAB).
- Technical approval.
- Security approval.
- QA approval.
- Application owner approval.
- Business approval.
- Risk-based approval.
- Approval separation of duties.
- Jira approval transitions.
- Automated approval gates.
- Jenkins and GitHub Actions approval gates.
- Argo CD deployment authorization.
- Emergency approval workflow.
- Rejection and resubmission.
- Approval audit trail.
- Final Production go/no-go decision.