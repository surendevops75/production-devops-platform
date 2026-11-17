# 01 — Jira Project Setup

## 1. Objective

Set up a Jira Software project to manage the complete development lifecycle of a production DevOps platform containing 21 microservices.

The project will be used by developers, DevOps engineers, QA engineers, project managers and release managers.

By the end of this chapter, you will have:
- A Jira Software project.
- A Scrum board.
- Project roles and access permissions.
- Configured issue types.
- A basic workflow.
- A team-ready project structure.

## 2. Project Architecture

![Jira Project Setup Workflow](../../diagrams/jira-project-setup-workflow.svg)

## 3. Project Scenario

We are implementing a production microservices platform on AWS EKS.

The engineering team needs to manage:
- New microservice development.
- Existing application enhancements.
- Infrastructure changes.
- CI/CD improvements.
- Security remediation.
- Production bugs.
- Sprint delivery.
- Production releases.

**Example project configuration**

| Field | Value |
|---|---|
| Project Name | Production DevOps Platform |
| Project Key | PLAT |
| Project Type | Jira Software |
| Project Template | Scrum |
| Project Management | Company-managed |
| Team | Platform Engineering |
| Development Platform | AWS EKS |
| Source Control | GitHub |
| CI/CD | GitHub Actions |

The project key `PLAT` is an example. Select an available key when creating your project.

## 4. Why Scrum?

Scrum is useful when the engineering team delivers work in planned iterations.

Example:

| Sprint | Duration | Main objective |
|---|---|---|
| Sprint 1 | 2 weeks | Project foundation |
| Sprint 2 | 2 weeks | User and product services |
| Sprint 3 | 2 weeks | Cart and order services |
| Sprint 4 | 2 weeks | Payment and inventory |
| Sprint 5 | 2 weeks | Security and deployment improvements |

The team can use a backlog to plan work and a sprint board to track progress.

Kanban is an alternative when work arrives continuously and the team does not need fixed sprint commitments.

## 5. Create a Jira Project

### Step 1: Open Jira

1. Open your Atlassian account.
2. Sign in to Jira.
3. Navigate to the Projects section.
4. Select Create project.

### Step 2: Select a project template

1. Select Software Development.
2. Choose Scrum.
3. Review the available project templates.
4. Select the company-managed project option if it is available.

**Why company-managed?**

For a shared engineering project, company-managed projects provide centralized control over workflows, permissions and schemes. This is useful when multiple teams follow common engineering processes.

Team-managed projects can be suitable for smaller teams that want more independent configuration.

### Step 3: Configure project details

Enter the following example details:

| Field | Example |
|---|---|
| Name | Production DevOps Platform |
| Key | PLAT |
| Project type | Software |
| Template | Scrum |
| Project description | Manage development and release activities for the production microservices platform |

Select Create project.

**Expected result:** Jira creates the project and provides access to its backlog and board.

## 6. Configure Project Roles

A production project normally involves multiple responsibilities.

Example team structure:

| Role | Responsibility |
|---|---|
| Project Administrator | Project configuration and access |
| Project Manager | Project planning and delivery tracking |
| Product Owner | Requirements and backlog priorities |
| Scrum Master | Sprint coordination |
| Developer | Application development |
| DevOps Engineer | Infrastructure and CI/CD |
| QA Engineer | Testing and defect validation |
| Release Manager | Release readiness and coordination |

### Permission considerations

Configure permissions according to responsibility.

- Administrators: project configuration and administration.
- Developers: create and update development work.
- QA: update testing tasks and defects.
- Project managers: manage project work and track delivery.
- Release managers: coordinate release activities.

Use Jira project roles and permission schemes rather than granting administrator access to every team member.

Do not assign permissions based only on job titles. Follow your organization's access policy and least-privilege requirements.

## 7. Configure Issue Types

Issue types represent the different kinds of work managed in Jira.

| Issue Type | Purpose | Example |
|---|---|---|
| Epic | Large feature or initiative | Order Management |
| Story | User-facing requirement | Implement order creation |
| Task | Engineering activity | Configure EKS deployment |
| Bug | Defect | Order API returns HTTP 500 |
| Sub-task | Smaller part of an issue | Create Kubernetes Deployment YAML |

### Example issue hierarchy

Epic: Order Management

- Story: Implement order creation API
  - Sub-task: Create Spring Boot endpoint
  - Sub-task: Configure database connection
  - Sub-task: Write unit tests
- Task: Configure Order Service deployment
- Bug: Fix order validation failure

This structure allows the team to track engineering work at different levels.

## 8. Configure the Scrum Board

Open the project's board.

Configure the board to show the team's work clearly.

Recommended initial columns:

| Column | Purpose |
|---|---|
| To Do | Work not started |
| In Progress | Development underway |
| Code Review | Pull request review |
| QA Testing | Testing in progress |
| Done | Work completed according to the team's definition of done |

Configure status-to-column mapping appropriately. A Jira workflow status and a board column are related, but they are not necessarily the same thing.

### Estimation

For Scrum, choose a consistent estimation method:

- Story points, or
- Time estimates.

Story points can help the team estimate relative effort and plan sprint capacity.

Do not treat story points as exact hours.

## 9. Example Jira Workflow

A simple initial workflow can be:

`TO DO → IN PROGRESS → CODE REVIEW → QA TESTING → DONE`

Additional statuses such as Blocked, UAT and Ready for Release can be introduced when the team needs them.

Define clear entry and exit criteria for each status.

Example:

| Status | Meaning |
|---|---|
| To Do | Work is ready to be picked up |
| In Progress | Developer is implementing the requirement |
| Code Review | Pull request is awaiting review |
| QA Testing | QA is validating the implementation |
| Done | Acceptance criteria are satisfied and work is complete |

For this initial setup, keep the workflow simple. Release approval and production deployment will be covered in later chapters.

## 10. Example Project Team

Example project team for the 21-microservice platform:

| Team Member | Role | Main Responsibility |
|---|---|---|
| Member 1 | Project Manager | Project planning |
| Member 2 | Product Owner | Requirements and priorities |
| Member 3 | Scrum Master | Sprint coordination |
| Member 4 | Backend Developer | Java microservices |
| Member 5 | Backend Developer | Node.js microservices |
| Member 6 | DevOps Engineer | AWS and EKS |
| Member 7 | DevOps Engineer | Terraform and CI/CD |
| Member 8 | QA Engineer | Functional and regression testing |

These are illustrative roles, not actual team assignments.

## 11. Validate the Setup

Use this checklist after creating the project:

- [ ] Jira Software project created.
- [ ] Project name and key configured.
- [ ] Scrum template selected.
- [ ] Project access reviewed.
- [ ] Project roles configured.
- [ ] Issue types reviewed.
- [ ] Scrum board created.
- [ ] Board columns configured.
- [ ] Workflow reviewed.
- [ ] Team members invited or assigned.
- [ ] Project description added.

## 12. Expected Result

At the end of this chapter, the Production DevOps Platform Jira project should be ready for backlog creation and sprint planning.

The next chapter will use this project to create Epics, Stories, Tasks, Bugs and Sub-tasks for the microservices platform.

## 13. Important Notes

- Jira project setup does not automatically configure GitHub or CI/CD integration.
- Jira itself does not deploy applications to EKS.
- GitHub integration and pipeline automation will be configured in later chapters.
- Jira menu names and available options can vary between Jira Cloud editions and project types.
- Use a test project first if you are learning Jira administration.