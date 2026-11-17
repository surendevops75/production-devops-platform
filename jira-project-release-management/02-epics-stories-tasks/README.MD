# 02 — Jira Epics, Stories, Tasks, Bugs & Sub-tasks

## 1. Objective

This chapter explains how to organize engineering work in Jira using Epics, Stories, Tasks, Bugs and Sub-tasks.

We will use a practical Production DevOps Platform project containing 21 microservices deployed on AWS EKS.

The objective is to understand how a project manager, product owner, developer, DevOps engineer and QA engineer manage and track work throughout the software development lifecycle.

By the end of this chapter, we will create a realistic Jira work hierarchy for an Order Management microservice.

## 2. Project Scenario

Our engineering team is developing a production microservices platform.

The platform contains services such as:

- User Service
- Product Catalog Service
- Cart Service
- Order Service
- Payment Service
- Inventory Service
- Notification Service
- Other supporting microservices

The team needs to manage application development, infrastructure changes, CI/CD implementation, testing and production deployment.

For this purpose, we use Jira to organize project requirements and engineering activities.

### Example Project Details

| Field | Value |
|---|---|
| Jira Project | Production DevOps Platform |
| Project Key | PLAT |
| Project Type | Scrum |
| Application | Microservices Platform |
| Cloud | AWS |
| Container Platform | Amazon EKS |
| Source Code | GitHub |
| CI/CD | GitHub Actions |
| Container Registry | Amazon ECR |
| Deployment | Helm and Argo CD |

*PLAT is an illustrative Jira project key.*

## 3. Understand Jira Issue Types

Jira provides different work item types to organize development activities.

| Issue Type | Purpose | Example |
|---|---|---|
| Epic | Large feature or initiative | Order Management |
| Story | Functional requirement | Create Order API |
| Task | Engineering activity | Configure EKS deployment |
| Bug | Defect in the application | Order API returns HTTP 500 |
| Sub-task | Smaller part of an issue | Implement REST endpoint |

### 3.1 Epic

An Epic represents a large initiative that contains multiple related work items.

Example:

**Epic Name:** Order Management

Description:

Implement order creation, order processing, payment integration, inventory validation and order status tracking.

The Epic can contain multiple Stories and Tasks.

### 3.2 Story

A Story describes a requirement from the perspective of a user or stakeholder.

Example:

**Story:** Create Order API

Description:

As a customer, I want to create an order so that I can purchase products from the application.

Acceptance Criteria:

1. The API accepts valid customer and product details.
2. The API validates incoming requests.
3. The application generates a unique order ID.
4. The order is stored successfully.
5. The API returns the appropriate HTTP response.
6. Invalid requests return meaningful validation errors.
7. Unit tests cover expected scenarios.

### 3.3 Task

A Task represents a specific engineering activity.

Examples:

- Create Dockerfile.
- Configure GitHub Actions pipeline.
- Configure Trivy image scanning.
- Create Kubernetes Deployment YAML.
- Configure Helm chart.
- Configure Argo CD application.
- Validate EKS deployment.

### 3.4 Bug

A Bug represents an issue or defect that needs investigation and correction.

Example:

**Bug:** Order API returns HTTP 500 for an invalid product ID.

Description:

When a customer submits an order with an invalid product ID, the Order Service returns HTTP 500 instead of a validation error.

Expected Result:

The application should return an appropriate validation response.

Actual Result:

The application returns HTTP 500.

Acceptance Criteria:

- The issue is reproduced.
- Root cause is identified.
- Application validation is corrected.
- Regression tests pass.
- QA verifies the fix.

### 3.5 Sub-task

A Sub-task breaks a larger Story or Task into smaller activities.

Example:

Parent Story: Create Order API

Sub-tasks:

1. Implement REST endpoint.
2. Add request validation.
3. Configure database connectivity.
4. Implement business logic.
5. Write unit tests.
6. Update API documentation.

## 4. Understand the Jira Work Hierarchy

Our example hierarchy:

**Epic — Order Management**

- Story — Create Order API
  - Sub-task — Implement REST endpoint
  - Sub-task — Add request validation
  - Sub-task — Write unit tests
- Story — Track Order Status
  - Sub-task — Implement status endpoint
  - Sub-task — Integrate order database
- Story — Integrate Payment Service
  - Sub-task — Configure payment API integration
  - Sub-task — Handle payment failure
- Task — Configure Order Service deployment
- Bug — Fix order validation failure

An Epic provides a higher-level view of related work. Individual work items track implementation and progress.

The exact hierarchy and supported parent-child relationships depend on your Jira project configuration and issue type scheme.

## 5. Create an Epic in Jira

### Step 1: Open the Jira Project

1. Sign in to Jira.
2. Open Projects.
3. Select Production DevOps Platform.
4. Navigate to Backlog or the available project work item creation area.
5. Locate the Epic creation option.

### Step 2: Create the Epic

Enter the following example details:

| Field | Value |
|---|---|
| Epic Name | Order Management |
| Summary | Implement Order Management |
| Description | Develop and deploy order processing capabilities |
| Priority | High |
| Assignee | Product Owner or responsible lead |

Select Create or Save.

### Step 3: Validate the Epic

Verify that:

- The Epic is visible in the project.
- The Epic has a unique Jira issue key.
- The description is correct.
- The correct person is assigned.
- The Epic is available for linking related work.

Example key: `PLAT-100`.

## 6. Create Stories Under the Epic

Create the following Stories and associate them with the Order Management Epic.

| Example Key | Story Summary | Description |
|---|---|---|
| PLAT-101 | Create Order API | Implement order creation endpoint |
| PLAT-102 | Track Order Status | Implement order status retrieval |
| PLAT-103 | Integrate Payment Service | Connect order processing with payment |
| PLAT-104 | Integrate Inventory Service | Validate product availability |
| PLAT-105 | Deploy Order Service | Deploy the service to AWS EKS |

The keys above are illustrative. Jira generates the actual issue keys when you create the issues.

### Story Creation Steps

1. Open the Jira project.
2. Select Create.
3. Choose Story as the issue type.
4. Enter the Summary.
5. Add a detailed Description.
6. Select the parent Epic.
7. Set Priority.
8. Assign the Story to the appropriate team member.
9. Add Acceptance Criteria.
10. Save the Story.

### Important

Always write clear acceptance criteria.

A developer should understand what needs to be implemented, and QA should understand how to validate it.

## 7. Create Engineering Tasks

Engineering Tasks represent technical activities required to deliver the Stories.

Example: PLAT-105 — Deploy Order Service.

| Task | Responsibility |
|---|---|
| Create Dockerfile | Developer |
| Configure GitHub Actions | DevOps Engineer |
| Configure SonarQube | DevOps Engineer |
| Configure Trivy | DevSecOps Engineer |
| Push Docker image to ECR | DevOps Engineer |
| Create Helm chart | DevOps Engineer |
| Configure Argo CD | DevOps Engineer |
| Deploy to EKS | DevOps Engineer |
| Validate application health | QA and DevOps |

### Task Creation Steps

1. Select Create.
2. Choose Task.
3. Enter the engineering activity.
4. Add technical implementation details.
5. Link it to the appropriate Epic or parent item according to the project's hierarchy.
6. Assign the responsible engineer.
7. Set priority.
8. Save the Task.

## 8. Create Sub-tasks

Sub-tasks are useful when a Story or Task contains multiple smaller activities.

Example:

Parent Story: PLAT-101 — Create Order API.

| Sub-task | Description |
|---|---|
| Implement REST endpoint | Create the order creation API |
| Add request validation | Validate customer and product details |
| Configure database connection | Connect the service to the database |
| Implement business logic | Process order creation |
| Write unit tests | Validate application behavior |
| Update API documentation | Document endpoint and responses |

### Sub-task Creation Steps

1. Open the parent Story or Task.
2. Locate the Sub-tasks section or More actions menu.
3. Select Create sub-task.
4. Enter the Summary.
5. Add a Description.
6. Assign the work.
7. Set the priority.
8. Save the Sub-task.

The parent issue provides an overall view of progress across its Sub-tasks.

## 9. Create and Manage Bugs

Bugs should be tracked separately from planned development tasks.

Example:

**Bug Summary:** Order API returns HTTP 500 for invalid product ID.

| Field | Example |
|---|---|
| Issue Type | Bug |
| Component | Order Service |
| Environment | QA |
| Priority | High |
| Assignee | Backend Developer |
| Reporter | QA Engineer |
| Description | Invalid product ID causes HTTP 500 |
| Expected Result | Validation error response |
| Actual Result | HTTP 500 response |

### Bug Lifecycle

1. QA identifies the issue.
2. QA creates a Bug in Jira.
3. QA adds reproduction steps and evidence.
4. The Bug is assigned to a developer.
5. Developer investigates the issue.
6. Developer implements the fix.
7. Code review is completed.
8. CI pipeline executes tests.
9. QA retests the application.
10. The Bug is closed after successful verification.

A failed retest should send the Bug back for investigation rather than closing it.

## 10. Link Jira Issues with GitHub Development

Jira issue keys can be included in branch names, commit messages and pull request titles.

Example:

### Branch Creation

```bash
git checkout -b feature/PLAT-101-create-order-api
```

### Commit Message

```bash
git add .
git commit -m "PLAT-101 implement order creation API"
```

### Push Branch

```bash
git push origin feature/PLAT-101-create-order-api
```

### Pull Request Title

```text
PLAT-101: Implement Order Creation API
```

When the Jira-GitHub integration is configured correctly, Jira can display associated development information such as branches, commits and pull requests.

Simply adding an issue key to a branch or commit does not guarantee integration. The relevant GitHub and Jira integration must be enabled and authorized.

## 11. Example Issue Assignment

A practical project team can divide work like this:

| Team Member | Role | Assigned Work |
|---|---|---|
| Member 1 | Product Owner | Epic and requirements |
| Member 2 | Backend Developer | Order API |
| Member 3 | Backend Developer | Payment integration |
| Member 4 | DevOps Engineer | Docker and CI/CD |
| Member 5 | DevOps Engineer | EKS and Helm |
| Member 6 | QA Engineer | Functional testing |
| Member 7 | Release Manager | Release coordination |

These are illustrative team roles and assignments.

## 12. Recommended Issue Management Practices

- Write meaningful issue summaries.
- Include acceptance criteria in Stories.
- Assign clear ownership.
- Avoid creating duplicate issues.
- Use priorities consistently.
- Link Bugs to affected Stories where appropriate.
- Break large engineering activities into manageable Tasks.
- Use Sub-tasks only when they improve visibility.
- Link Jira issues with GitHub development activities.
- Keep issue statuses updated.
- Avoid closing work before its acceptance criteria are satisfied.

## 13. Validation Checklist

- [ ] Created Order Management Epic.
- [ ] Created Stories under the Epic.
- [ ] Added acceptance criteria.
- [ ] Created engineering Tasks.
- [ ] Created Sub-tasks.
- [ ] Created a sample Bug.
- [ ] Assigned work to team members.
- [ ] Set priorities.
- [ ] Linked related issues.
- [ ] Verified the Epic and issue hierarchy.
- [ ] Added Jira issue keys to sample GitHub branches and commits.
- [ ] Verified Jira development integration if configured.

## 14. Expected Outcome

At the end of this chapter, the Jira project will contain a structured set of work items for the Order Management capability.

The project team can now identify:

- What needs to be delivered.
- Which team member owns each activity.
- How a large requirement is broken down.
- Which Bugs are affecting development.
- How implementation work relates to the larger Epic.

This structure becomes the foundation for backlog refinement and sprint planning.

## 15. Next Chapter

**Chapter 03 — Backlog Management**

The next chapter covers:

1. Creating and maintaining the product backlog.
2. Backlog grooming and refinement.
3. Prioritizing Stories and Tasks.
4. Story point estimation.
5. Managing dependencies.
6. Sprint readiness.
7. Preparing work for sprint planning.