# 04 — Jira Sprint Planning

## 1. Objective

This chapter explains how to plan and organize a development sprint using Jira Scrum.

We will use the Production DevOps Platform project and the Order Management microservice as our practical example.

The objective is to understand how a Product Owner, Scrum Master and engineering team select backlog items, define a sprint goal, estimate team capacity and prepare work for execution.

By the end of this chapter, you will understand:

- Sprint creation in Jira.
- Sprint goals.
- Sprint duration.
- Team capacity planning.
- Story selection.
- Story point estimation.
- Sprint backlog creation.
- Task assignment.
- Sprint scope management.
- Sprint planning validation.

## 2. What Is Sprint Planning?

Sprint Planning is a Scrum event where the team decides what work it can reasonably complete during the upcoming sprint and how it will approach that work.

A sprint is a fixed-length development cycle.

For this project, we will use a two-week sprint.

Example:

| Field | Value |
|---|---|
| Jira Project | Production DevOps Platform |
| Project Key | PLAT |
| Sprint | Sprint 01 |
| Duration | 2 Weeks |
| Sprint Goal | Deliver Order Service API foundation |
| Team | Backend, DevOps and QA |
| Estimation | Story Points |

## 3. Sprint Planning Workflow

The sprint planning process follows these steps:

1. Review the prioritized backlog.
2. Confirm that selected Stories are ready.
3. Define the sprint goal.
4. Review team availability.
5. Calculate available capacity.
6. Select Stories based on priority and dependencies.
7. Discuss estimates.
8. Break down implementation work.
9. Confirm the sprint forecast.
10. Start the sprint.

## 4. Understand the Sprint Goal

A sprint goal describes the main outcome the team wants to achieve during the sprint.

Example:

**Sprint 01 — Order Service Foundation**

Sprint Goal:

Develop the initial Order Service functionality, implement order creation and validation, complete unit testing and prepare the service for deployment to AWS EKS.

The sprint goal should describe an outcome rather than simply listing Jira ticket names.

### Example Sprint Goals

| Sprint | Sprint Goal |
|---|---|
| Sprint 01 | Develop Order Service API foundation |
| Sprint 02 | Integrate Payment Service |
| Sprint 03 | Integrate Inventory Service |
| Sprint 04 | Complete Order Service deployment |
| Sprint 05 | Improve monitoring and production readiness |

These are illustrative sprint goals for the project.

## 5. Review the Product Backlog

Before sprint planning, the Product Owner and engineering team review the prioritized backlog.

Example backlog:

| Jira Issue | Work Item | Story Points | Priority |
|---|---|---:|---|
| PLAT-101 | Create Order API | 8 | Highest |
| PLAT-102 | Track Order Status | 3 | High |
| PLAT-103 | Payment Integration | 8 | High |
| PLAT-104 | Inventory Integration | 5 | High |
| PLAT-105 | Order Service EKS Deployment | 5 | High |
| PLAT-106 | Configure Health Checks | 2 | Medium |

The team checks:

- Is the requirement clear?
- Are acceptance criteria available?
- Are dependencies understood?
- Is the estimate available?
- Can the work be completed during the sprint?

Only sufficiently prepared work should be considered for selection.

## 6. Understand Team Capacity

Team capacity represents the amount of time the team has available during a sprint.

It is important because not every team member is available for every working day.

### Example Team

| Team Member | Role | Availability |
|---|---|---:|
| Member 1 | Backend Developer | 10 days |
| Member 2 | Backend Developer | 10 days |
| Member 3 | DevOps Engineer | 8 days |
| Member 4 | QA Engineer | 8 days |
| Member 5 | Technical Lead | 8 days |

Assume an eight-hour working day.

Total available person-days:

10 + 10 + 8 + 8 + 8 = 44 person-days.

Total available hours:

44 × 8 = 352 hours.

This is gross availability, not necessarily productive development capacity.

The team should also consider:

- Meetings.
- Code reviews.
- Production support.
- Leave.
- Training.
- Other operational responsibilities.

### Example Capacity Adjustment

| Activity | Hours |
|---|---:|
| Gross availability | 352 |
| Meetings and ceremonies | 40 |
| Production support | 32 |
| Leave and other planned absence | 32 |
| Remaining planned work capacity | 248 |

The figures are illustrative. Actual capacity should be calculated using the team's availability and working agreements.

### Important

Do not directly convert available hours into story points.

Story points represent relative effort and complexity. Capacity and historical team velocity are separate planning inputs.

## 7. Understand Team Velocity

Velocity is the number of story points a Scrum team completes during a sprint.

Example historical velocity:

| Sprint | Completed Story Points |
|---|---:|
| Sprint 01 | 18 |
| Sprint 02 | 21 |
| Sprint 03 | 20 |
| Sprint 04 | 22 |

Average velocity:

(18 + 21 + 20 + 22) / 4 = 20.25 story points.

The team can use historical velocity as a planning reference.

For a new team without historical data, start with a conservative forecast and adjust based on actual results.

Velocity should not be used to compare individual engineers or pressure teams to increase estimates.

## 8. Select Stories for the Sprint

Assume the team plans Sprint 01.

Sprint Goal:

Deliver the initial Order Service API foundation.

Candidate backlog:

| Issue | Story | Points |
|---|---|---:|
| PLAT-101 | Create Order API | 8 |
| PLAT-102 | Add Request Validation | 3 |
| PLAT-108 | Write Order API Unit Tests | 5 |
| PLAT-109 | Configure API Documentation | 2 |
| PLAT-110 | Prepare Deployment Configuration | 2 |

Total:

8 + 3 + 5 + 2 + 2 = 20 story points.

The team considers the scope against its availability, historical delivery and dependencies.

If the team agrees that 20 points is a realistic forecast, these items can be selected.

This is a forecast, not an unconditional guarantee.

## 9. Create a Sprint in Jira

### Step 1: Open the Jira Backlog

1. Sign in to Jira.
2. Open Production DevOps Platform.
3. Navigate to Backlog.
4. Find the sprint planning section.

### Step 2: Create a Sprint

1. Select Create sprint.
2. Jira creates a new sprint in the backlog.
3. Rename the sprint if required.
4. Select the planned Stories and Tasks.
5. Move the selected items into the sprint.

Example sprint name:

**Sprint 01 — Order Service Foundation**

### Step 3: Configure Sprint Details

Before starting the sprint, configure:

| Field | Example |
|---|---|
| Sprint Name | Sprint 01 — Order Service Foundation |
| Duration | 2 Weeks |
| Start Date | Example Monday |
| End Date | Two weeks later |
| Sprint Goal | Deliver Order Service API foundation |

The exact sprint configuration controls depend on your Jira project and board settings.

### Step 4: Review Selected Work

Verify:

- Selected work supports the sprint goal.
- Stories have acceptance criteria.
- Dependencies are understood.
- Estimates are available.
- Team availability is considered.
- The selected scope is realistic.

### Step 5: Start the Sprint

Select Start sprint and confirm the sprint details.

Jira moves the team into the active sprint view.

## 10. Example Sprint Backlog

Sprint 01:

**Sprint Goal:** Deliver the initial Order Service API foundation.

| Jira Issue | Work Item | Assignee | Points |
|---|---|---|---:|
| PLAT-101 | Create Order API | Backend Developer 1 | 8 |
| PLAT-102 | Add Request Validation | Backend Developer 2 | 3 |
| PLAT-108 | Write Unit Tests | Backend Developer 1 | 5 |
| PLAT-109 | API Documentation | Backend Developer 2 | 2 |
| PLAT-110 | Deployment Configuration | DevOps Engineer | 2 |
| | **Total** | | **20** |

The issue names, keys, assignees and estimates are illustrative.

## 11. Break Stories into Engineering Tasks

Example:

Story: PLAT-101 — Create Order API.

Engineering activities:

1. Create the application endpoint.
2. Implement request validation.
3. Configure database connectivity.
4. Implement order creation logic.
5. Add error handling.
6. Write unit tests.
7. Perform code review.

Example:

Story: PLAT-110 — Deployment Configuration.

Engineering activities:

1. Create Dockerfile.
2. Build container image.
3. Configure image scanning.
4. Prepare Kubernetes Deployment YAML.
5. Create Helm values.
6. Prepare EKS deployment configuration.

Use Jira Sub-tasks when they provide useful visibility into the work.

## 12. Sprint Execution and Daily Coordination

After starting the sprint, team members update their work as development progresses.

Example workflow:

TO DO → IN PROGRESS → CODE REVIEW → QA TESTING → DONE

The team uses the Jira active sprint board to track work.

### Daily Scrum

The team holds a short daily coordination meeting.

Typical discussion:

1. What progress has been made toward the sprint goal?
2. What will be worked on next?
3. Are there any blockers?
4. Is the sprint goal at risk?

Example:

| Team Member | Update |
|---|---|
| Backend Developer 1 | Order API implementation completed |
| Backend Developer 2 | Request validation in progress |
| DevOps Engineer | Docker build configuration completed |
| QA Engineer | Preparing API test scenarios |

The Daily Scrum is for coordination and adaptation, not merely reporting to a manager.

## 13. Manage Sprint Scope Changes

Sometimes new requirements or production issues appear during an active sprint.

Example:

The team is working on Order Service when a critical production defect is reported in Payment Service.

The team should:

1. Assess the urgency and business impact.
2. Discuss the impact on the sprint goal.
3. Coordinate with the Product Owner.
4. Decide whether existing work must be reordered or removed.
5. Update the Jira backlog and sprint scope.
6. Communicate the impact to stakeholders.

Do not silently add unlimited work to an active sprint.

If the sprint goal becomes obsolete, the Product Owner may cancel the sprint in accordance with Scrum practices.

## 14. Sprint Progress Tracking

The team can use Jira reports and board views to understand progress.

Useful Jira views and reports may include:

- Active Sprint board.
- Burndown chart.
- Sprint report.
- Velocity chart.
- Cumulative flow diagram.

### Example Sprint Progress

| Day | Remaining Story Points |
|---|---:|
| Day 1 | 20 |
| Day 3 | 17 |
| Day 5 | 14 |
| Day 7 | 9 |
| Day 9 | 4 |
| Day 10 | 0 |

This is an illustrative idealized progress example.

A burndown chart shows remaining work over time. It helps the team discuss whether progress is consistent with the sprint goal.

Do not interpret a burndown chart as proof of individual productivity.

## 15. Sprint Completion

At the end of the sprint:

1. Review completed Stories.
2. Verify acceptance criteria.
3. Confirm testing results.
4. Identify unfinished work.
5. Conduct the Sprint Review.
6. Conduct the Sprint Retrospective.
7. Review the sprint report.
8. Return unfinished work to the backlog for reassessment.

Unfinished work should not automatically be treated as completed or silently counted in the next sprint. The Product Owner and team should review and reprioritize it.

## 16. Sprint Planning Best Practices

- Define one clear sprint goal.
- Select work from a refined backlog.
- Consider team availability.
- Use historical velocity as a guide.
- Do not overcommit.
- Identify dependencies early.
- Keep Stories small enough to complete.
- Assign clear ownership.
- Track blockers promptly.
- Review sprint progress regularly.
- Communicate scope changes.
- Focus on delivering a usable outcome.

## 17. Validation Checklist

- [ ] Reviewed the product backlog.
- [ ] Confirmed Story readiness.
- [ ] Defined the sprint goal.
- [ ] Reviewed team availability.
- [ ] Considered historical velocity.
- [ ] Created a Jira sprint.
- [ ] Selected appropriate Stories and Tasks.
- [ ] Confirmed sprint estimates.
- [ ] Reviewed dependencies.
- [ ] Started the sprint.
- [ ] Updated the active sprint board.
- [ ] Reviewed sprint progress.
- [ ] Completed sprint review and retrospective.

## 18. Expected Outcome

At the end of this chapter, the engineering team should have a properly planned Jira sprint with:

- A clear sprint goal.
- A realistic work forecast.
- Selected backlog items.
- Assigned responsibilities.
- Defined estimates.
- Identified dependencies.
- A method for tracking progress.

The team is now ready to execute the planned sprint.

## 19. Next Chapter

**Chapter 05 — Development Workflow**

Topics:

1. Jira issue assignment and ownership.
2. Moving Stories through development statuses.
3. Git branching strategy.
4. Linking Jira issues with GitHub branches.
5. Commit message conventions.
6. Pull request management.
7. Code review and approval.
8. Linking development progress back to Jira.