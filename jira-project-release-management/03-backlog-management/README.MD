# 03 — Jira Backlog Management

## 1. Objective

This chapter explains how to manage a software development backlog using Jira.

We will use the Production DevOps Platform project, which manages 21 microservices running on AWS EKS.

The objective is to understand how a Product Owner, Project Manager, Scrum Master, Developers, DevOps Engineers and QA Engineers organize upcoming work before selecting it for a sprint.

By the end of this chapter, you will understand:

- Product backlog creation.
- Backlog refinement and grooming.
- Story prioritization.
- Story point estimation.
- Technical dependencies.
- Definition of Ready.
- Sprint readiness.
- Backlog management in Jira.

## 2. What Is a Jira Backlog?

A backlog is a prioritized list of work that the engineering team may implement in upcoming sprints.

It contains:

- Epics.
- Stories.
- Tasks.
- Bugs.
- Technical debt.
- Infrastructure improvements.
- Security improvements.

Example backlog for the Order Management capability:

| Jira Issue | Description | Priority |
|---|---|---|
| PLAT-101 | Create Order API | Highest |
| PLAT-102 | Track Order Status | High |
| PLAT-103 | Payment Service Integration | High |
| PLAT-104 | Inventory Integration | High |
| PLAT-105 | Order Service EKS Deployment | High |
| PLAT-106 | Configure Trivy Image Scanning | Medium |
| PLAT-107 | Add Application Health Checks | Medium |

The issue keys are illustrative examples. Jira generates actual keys when issues are created.

## 3. Backlog Management Workflow

The backlog management process follows these steps:

1. Capture requirements.
2. Create Stories, Tasks and Bugs.
3. Refine descriptions and acceptance criteria.
4. Identify dependencies.
5. Prioritize work.
6. Estimate effort using story points.
7. Review Definition of Ready.
8. Select suitable items during sprint planning.

## 4. Create Backlog Items in Jira

### Step 1: Open the Jira Project

1. Sign in to Jira.
2. Open Projects.
3. Select Production DevOps Platform.
4. Navigate to Backlog.
5. Select the option to create a work item.

### Step 2: Create a Story

Example:

| Field | Value |
|---|---|
| Issue Type | Story |
| Summary | Create Order API |
| Epic | Order Management |
| Priority | Highest |
| Assignee | Backend Developer |
| Description | Implement the order creation endpoint |

### Step 3: Add Acceptance Criteria

For the Create Order API Story:

1. The API accepts valid customer and product details.
2. The application validates incoming requests.
3. The application generates a unique order ID.
4. Order details are stored successfully.
5. The API returns an appropriate HTTP response.
6. Invalid requests return meaningful validation errors.
7. Unit tests cover expected scenarios.

### Step 4: Save and Validate

Verify that:

- The Story appears in the backlog.
- It is linked to the correct Epic.
- The description is clear.
- Acceptance criteria are available.
- Priority is configured.
- An owner is assigned when appropriate.

## 5. Backlog Refinement

Backlog refinement is the process of reviewing and improving upcoming work items.

The Product Owner and engineering team discuss requirements to ensure that everyone understands what needs to be delivered.

### Example: Initial Story

**PLAT-101 — Create Order API**

Initial description:

Implement an API to create orders.

This description is too broad.

### Refined Story

The team improves the description:

- Accept customer ID and product details.
- Validate product availability.
- Generate a unique order ID.
- Save order information.
- Return the order creation response.
- Handle invalid requests.

### Acceptance Criteria

| ID | Acceptance Criteria |
|---|---|
| AC-01 | Valid order request is accepted |
| AC-02 | Invalid customer ID is rejected |
| AC-03 | Invalid product ID is rejected |
| AC-04 | Successful order receives a unique ID |
| AC-05 | Order details are stored correctly |
| AC-06 | Unit tests pass |

If the Story is too large, split it into smaller Stories or engineering Tasks.

## 6. Backlog Prioritization

The Product Owner works with stakeholders and the engineering team to determine which work should be delivered first.

Prioritization considers:

- Business value.
- Customer impact.
- Technical risk.
- Security impact.
- Operational urgency.
- Dependencies.
- Delivery deadlines.

### Example Prioritized Backlog

| Priority | Work Item | Reason |
|---|---|---|
| Highest | Order creation API | Core business functionality |
| High | Payment integration | Required for order processing |
| High | Inventory integration | Prevent unavailable products being ordered |
| High | Order status API | Customer visibility |
| Medium | EKS deployment improvements | Operational improvement |
| Medium | Additional dashboards | Better operational visibility |
| Low | Documentation enhancement | Can be scheduled later |

Do not prioritize solely based on technical complexity.

### 6.1 MoSCoW Prioritization

MoSCoW is one approach to categorizing requirements.

| Category | Meaning | Example |
|---|---|---|
| Must Have | Essential requirement | Order creation |
| Should Have | Important requirement | Order tracking |
| Could Have | Useful improvement | Additional dashboard |
| Won't Have Now | Deferred requirement | Nonessential enhancement |

This method helps stakeholders understand which requirements are essential for a release.

### 6.2 Business Value and Risk

Consider both business impact and technical risk.

For example, a security vulnerability affecting production may need immediate attention even if it was not part of the original sprint plan.

Jira priority fields and backlog ranking can support prioritization. The Product Owner remains responsible for backlog ordering.

## 7. Story Point Estimation

Story points estimate the relative effort, complexity and uncertainty of a work item.

A common estimation approach uses the Fibonacci sequence.

| Story Points | Relative Size |
|---:|---|
| 1 | Very small |
| 2 | Small |
| 3 | Moderate |
| 5 | Medium |
| 8 | Large |
| 13 | Very large |

### Example Story Estimation

| Story | Estimated Points |
|---|---:|
| Create Order API | 8 |
| Track Order Status | 3 |
| Payment Integration | 8 |
| Inventory Integration | 5 |
| Configure Health Checks | 2 |

These are example estimates. The engineering team should estimate work collaboratively.

### Important Points

- Story points are not hours.
- Five story points do not mean five hours or five days.
- Estimation should consider complexity and uncertainty.
- Team members should discuss estimates before agreeing on them.
- Do not compare story points between different teams as if they represent identical effort.

## 8. Manage Dependencies

Some work cannot begin until another activity is completed.

Example: Order Processing Dependencies.

1. Order Service receives an order request.
2. Inventory Service validates stock.
3. Payment Service processes payment.
4. Order Service updates the order status.

### Example Dependency Table

| Jira Issue | Dependency |
|---|---|
| Order creation | Product Catalog API |
| Payment integration | Payment Service API |
| Inventory integration | Inventory Service API |
| Order status update | Order and payment processing events |
| Production deployment | EKS infrastructure and container image |

### Jira Issue Linking

Open the relevant Jira issue and use the issue linking option.

Common link types include:

- Blocks.
- Is blocked by.
- Relates to.
- Duplicates.
- Is duplicated by.

Example:

`PLAT-103 is blocked by PLAT-101`

This means the Payment Integration work depends on the Order API work.

Creating an issue link does not automatically enforce execution order. The team must manage dependencies during refinement and sprint planning.

## 9. Definition of Ready

Definition of Ready is a team agreement describing when a Story is sufficiently prepared for sprint planning.

Example checklist:

- [ ] Description is clear.
- [ ] Acceptance criteria are documented.
- [ ] Dependencies are identified.
- [ ] Priority is agreed.
- [ ] Story points are estimated.
- [ ] Required technical information is available.
- [ ] The Story is small enough for a sprint.
- [ ] The team understands the expected outcome.

If a Story is not ready, return it for clarification rather than forcing it into the sprint.

Definition of Ready is a team working agreement, not a mandatory Jira feature.

## 10. Backlog for the 21-Microservice Platform

Example work items across the platform:

| Service or Area | Backlog Work |
|---|---|
| User Service | Registration and authentication |
| Product Catalog | Product API enhancements |
| Cart Service | Cart management |
| Order Service | Order creation and tracking |
| Payment Service | Payment processing |
| Inventory Service | Stock validation |
| Notification Service | Email and event notifications |
| Platform Engineering | EKS deployment and infrastructure |
| DevSecOps | SonarQube and Trivy integration |
| Observability | Prometheus and Grafana dashboards |
| Database | Database connectivity and performance |
| Messaging | RabbitMQ and Kafka integration |

These are example backlog items. Actual work should reflect the application requirements and deployed components.

## 11. Backlog Refinement Meeting

A typical refinement meeting involves:

| Participant | Responsibility |
|---|---|
| Product Owner | Explains requirements and priorities |
| Scrum Master | Facilitates the meeting |
| Developers | Discuss implementation |
| DevOps Engineer | Identifies infrastructure and deployment dependencies |
| QA Engineer | Identifies testing scenarios |
| Technical Lead | Reviews architecture and technical risks |

### Example Meeting Agenda

1. Review upcoming Stories.
2. Clarify requirements.
3. Discuss acceptance criteria.
4. Identify technical dependencies.
5. Split oversized Stories.
6. Estimate effort.
7. Review priority.
8. Identify items ready for sprint planning.

The goal is to make upcoming work understandable and actionable.

## 12. Sprint Readiness Review

Before sprint planning, the Product Owner and engineering team review each candidate Story.

Questions to ask:

1. Is the business requirement understood?
2. Are acceptance criteria available?
3. Are dependencies known?
4. Is priority agreed?
5. Is the estimate available?
6. Does the team have the required technical knowledge?
7. Can the work be completed within the sprint?

Only sufficiently prepared work should be considered for sprint planning.

## 13. Backlog Management Best Practices

- Write meaningful issue summaries.
- Include acceptance criteria in Stories.
- Assign clear ownership.
- Avoid duplicate issues.
- Prioritize consistently.
- Link related Bugs and Stories.
- Break large engineering activities into manageable work items.
- Use Sub-tasks only when they improve visibility.
- Track dependencies.
- Keep issue details updated.
- Review the backlog regularly.
- Avoid putting every possible task into the next sprint.
- Do not commit to more work than the team can reasonably deliver.

## 14. Validation Checklist

- [ ] Created backlog items.
- [ ] Linked Stories to the correct Epic.
- [ ] Added descriptions and acceptance criteria.
- [ ] Set priorities.
- [ ] Estimated Stories using story points.
- [ ] Identified dependencies.
- [ ] Reviewed oversized Stories.
- [ ] Applied the Definition of Ready.
- [ ] Reviewed backlog ordering.
- [ ] Identified candidate Stories for the next sprint.

## 15. Expected Outcome

At the end of this chapter, the Product Owner and engineering team should have an organized, prioritized and refined backlog.

The team should understand:

- What needs to be delivered.
- Why the work is important.
- Who is responsible.
- How much effort the work involves.
- Which dependencies must be resolved.
- Whether the work is ready for sprint planning.

The backlog is now ready to support sprint planning.

## 16. Next Chapter

**Chapter 04 — Sprint Planning**

Topics:

1. Creating a Scrum sprint.
2. Defining a sprint goal.
3. Selecting backlog items.
4. Understanding team capacity.
5. Sprint commitment and forecasting.
6. Assigning work and tracking sprint progress.
7. Managing scope changes during a sprint.