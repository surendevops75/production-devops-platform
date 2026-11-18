# Chapter 23 — Jira Metrics, KPIs and Delivery Analytics

## 1. Objective

The purpose of delivery analytics is to understand how efficiently engineering teams deliver software, how frequently defects occur, how reliably releases are completed, and where work gets delayed.

For the production platform containing 21 microservices on AWS EKS, Jira provides visibility into:

- Backlog and sprint performance.
- Development and testing progress.
- Work-item completion.
- Defect trends.
- Release readiness.
- Deployment and rollback records.
- Production incidents.
- Corrective action completion.

The objective is not to measure developers by the number of tickets they close. It is to identify delivery bottlenecks, improve predictability, and make better engineering decisions.

## 2. Metrics architecture

The analytics process has four stages:

1. Collect Jira project and issue data.
2. Create saved JQL filters.
3. Display metrics using Jira dashboards and reports.
4. Review trends and take improvement actions.

### Data sources

| Source | Data collected |
|---|---|
| Jira Software | Epics, stories, tasks, bugs, sprints |
| Jira Service Management | Incidents, priorities, SLA records |
| GitHub | Pull requests, reviews, commits |
| Jenkins | Pipeline execution records |
| GitHub Actions | Workflow execution records |
| GitLab CI/CD | Pipeline execution records |
| Argo CD | Deployment synchronization and health |
| Prometheus | Application and infrastructure metrics |
| Grafana | Runtime dashboards |
| ELK | Application and infrastructure logs |

**Important:** Jira does not automatically become a complete analytics platform for every connected tool. Integrations, API access, data mappings, and reporting pipelines may be needed to combine information from different systems.

## 3. Define KPI ownership

Before creating dashboards, define who owns each metric.

| Metric area | Owner |
|---|---|
| Sprint performance | Scrum Master / Engineering Lead |
| Backlog health | Product Owner |
| Development flow | Engineering Lead |
| Defect trends | QA Lead |
| Release performance | Release Manager |
| CI/CD performance | DevOps Engineer |
| Production incidents | Operations / Incident Manager |
| Runtime reliability | Platform Engineering |

Avoid creating metrics without a clear purpose, owner, and agreed definition.

## 4. Sprint performance metrics

Sprint metrics help teams understand how much planned work is completed and whether sprint commitments are realistic.

### 4.1 Sprint velocity

Velocity measures the number of story points completed during a sprint.

Example:

| Sprint | Planned points | Completed points |
|---|---:|---:|
| Sprint 1 | 40 | 34 |
| Sprint 2 | 38 | 36 |
| Sprint 3 | 42 | 39 |
| Sprint 4 | 40 | 37 |

Average velocity:

\[
\frac{34+36+39+37}{4}=36.5
\]

The average completed velocity is 36.5 story points per sprint.

Use velocity for team-level planning. Do not compare individual developers using story points or treat velocity as a productivity score.

### 4.2 Sprint completion percentage

\[
\text{Completion \%} =
\frac{\text{Completed committed points}}
{\text{Committed points}}\times100
\]

Example:

- Committed points: 40
- Completed committed points: 36

\[
\frac{36}{40}\times100=90\%
\]

The sprint completion percentage is 90%.

Define how the team treats scope changes during a sprint. Otherwise, completion percentages can be misleading.

### 4.3 Sprint carryover

Carryover represents work that was committed but not completed within the sprint.

Example:

| Sprint | Incomplete committed points |
|---|---:|
| Sprint 1 | 6 |
| Sprint 2 | 2 |
| Sprint 3 | 3 |

Investigate recurring carryover. Possible reasons include:

- Stories were too large.
- Dependencies were not identified.
- Requirements changed.
- Testing took longer than expected.
- Production incidents interrupted planned work.

### 4.4 Burndown chart

A sprint burndown chart compares remaining work against the sprint timeline.

Use it to identify:

- Work that is not progressing.
- Scope increases.
- Testing delays.
- Late sprint completion.
- Unexpected blockers.

A burndown chart is a planning aid, not proof that the team is delivering high-quality software.

## 5. Backlog health metrics

Backlog metrics help Product Owners and engineering teams understand whether upcoming work is ready.

Recommended indicators:

| Metric | Purpose |
|---|---|
| Total open backlog | Understand remaining work |
| Unestimated issues | Identify planning gaps |
| Blocked issues | Find dependencies |
| Aging backlog items | Identify stale work |
| Unassigned issues | Find ownership gaps |
| High-priority backlog | Track urgent work |
| Ready-for-development items | Assess sprint readiness |

### Example JQL — Unassigned issues

```jql
project = SHOP
AND assignee IS EMPTY
AND statusCategory != Done
ORDER BY priority DESC
```

### Example JQL — Blocked issues

```jql
project = SHOP
AND status = Blocked
ORDER BY updated ASC
```

The `Blocked` status must exist in the project's workflow for this query to work.

### Example JQL — Unestimated stories

```jql
project = SHOP
AND issuetype = Story
AND "Story Points" IS EMPTY
AND statusCategory != Done
```

The story-point field name can differ by Jira configuration.

### Example JQL — Old unresolved issues

```jql
project = SHOP
AND statusCategory != Done
AND created <= -30d
ORDER BY created ASC
```

Use this filter to review older work that may no longer be relevant or may need refinement.

## 6. Development flow metrics

These metrics measure how work moves from development to completion.

### 6.1 Lead time

Lead time measures the elapsed time from a defined request or work-start event to delivery.

For engineering analytics, define the starting event explicitly.

Example:

- Story created: 1 November.
- Story delivered: 12 November.
- Lead time: 11 calendar days.

Different organizations define lead time differently. Keep the definition consistent across reports.

### 6.2 Cycle time

Cycle time measures how long an item takes from entering active work until completion.

Example:

- Story moved to In Progress: 5 November.
- Story moved to Done: 9 November.
- Cycle time: 4 calendar days.

Lead time and cycle time are related but not interchangeable.

### 6.3 Throughput

Throughput measures the number of work items completed in a defined period.

Example:

| Week | Completed issues |
|---|---:|
| Week 1 | 12 |
| Week 2 | 15 |
| Week 3 | 11 |
| Week 4 | 16 |

Average weekly throughput:

\[
\frac{12+15+11+16}{4}=13.5
\]

Average throughput is 13.5 completed issues per week.

Use comparable issue types and scope when interpreting this metric.

### 6.4 Work in progress

Work in progress (WIP) is the number of items currently being worked on.

Excessive WIP can cause:

- Context switching.
- Longer cycle times.
- Delayed code reviews.
- Testing bottlenecks.
- Incomplete work across multiple services.

Example:

If the team has 15 issues in progress but only 5 engineers actively working on them, investigate whether work is being started faster than it can be completed.

### 6.5 Aging work items

Aging work items are active issues that have remained in progress for longer than expected.

Review these issues during daily stand-ups and delivery reviews.

Potential causes:

- External dependencies.
- Unclear acceptance criteria.
- Technical complexity.
- Missing environment access.
- Delayed code reviews.
- QA resource constraints.

## 7. Quality metrics

Quality metrics show whether software delivery is producing stable and testable releases.

### 7.1 Defect count

Track bugs by:

- Service.
- Severity.
- Environment.
- Release.
- Sprint.
- Root cause category.

Example:

| Service | Open bugs | Critical bugs |
|---|---:|---:|
| User | 4 | 0 |
| Product Catalog | 6 | 1 |
| Cart | 3 | 0 |
| Orders | 5 | 1 |
| Payment | 7 | 2 |
| Inventory | 2 | 0 |
| Notification | 3 | 0 |

These are illustrative figures, not actual production measurements.

### 7.2 Defect reopen rate

\[
\text{Reopen Rate} =
\frac{\text{Reopened defects}}
{\text{Resolved defects}}\times100
\]

Example:

- Resolved defects: 50
- Reopened defects: 5

\[
\frac{5}{50}\times100=10\%
\]

A high reopen rate may indicate incomplete fixes, insufficient test coverage, or unclear acceptance criteria.

### 7.3 Escaped defects

Escaped defects are defects discovered after the intended testing stage, such as in UAT or production.

Track:

- Defects found in development.
- Defects found in QA.
- Defects found in UAT.
- Defects found in production.

A defect found in production may have greater business impact than one found during development. Track severity and customer impact, not only raw defect count.

### 7.4 Defect aging

Measure how long bugs remain unresolved.

Example:

| Age | Interpretation |
|---|---|
| 0–2 days | Recently reported |
| 3–7 days | Needs regular review |
| 8–14 days | Investigate delay |
| More than 14 days | Escalate for ownership or prioritization review |

These are example reporting bands. Set actual targets according to your team's support and delivery policy.

## 8. Release performance metrics

Release metrics help understand how reliably software changes move into production.

Recommended metrics:

| Metric | Meaning |
|---|---|
| Deployment frequency | How often production deployments occur |
| Release lead time | Time from agreed development milestone to production |
| Change failure rate | Proportion of deployments that cause a failure requiring intervention |
| Rollback rate | Proportion of deployments requiring rollback |
| Release success rate | Proportion of releases meeting the defined success criteria |
| Release duration | Time taken to complete the deployment process |

### Example deployment frequency

If the team performs 12 production deployments in 30 days:

\[
\text{Deployment Frequency} =
\frac{12}{30}=0.4
\]

This is 0.4 deployments per day, or approximately 2.8 deployments per week.

### Example change failure rate

Suppose:

- Production deployments: 20.
- Deployments resulting in a production failure requiring remediation: 2.

\[
\text{Change Failure Rate} =
\frac{2}{20}\times100=10\%
\]

Define what qualifies as a deployment-related failure and how multiple failed changes are counted. Avoid mixing unrelated production incidents into this metric.

### Example rollback rate

Suppose:

- Production deployments: 20.
- Deployments requiring rollback: 1.

\[
\text{Rollback Rate} =
\frac{1}{20}\times100=5\%
\]

Rollback rate and change failure rate are not the same. Some failures are corrected forward without rollback.

## 9. CI/CD pipeline metrics

Jira can link development issues to pipeline records, but detailed pipeline measurements should normally come from the CI/CD systems themselves.

### Recommended CI/CD metrics

| Metric | Source |
|---|---|
| Pipeline success rate | Jenkins / GitHub Actions / GitLab |
| Pipeline failure rate | CI/CD system |
| Average build duration | CI/CD system |
| Test execution duration | CI/CD system |
| Security scan failures | SonarQube / Trivy / Veracode |
| Artifact publishing success | ECR and pipeline logs |
| Deployment duration | CI/CD and Argo CD |
| Failed deployment count | Argo CD / deployment records |

### Example CI/CD flow

```text
Jira Story
    |
    v
GitHub Pull Request
    |
    v
Jenkins / GitHub Actions / GitLab
    |
    +--> Unit Tests
    |
    +--> SonarQube
    |
    +--> Trivy
    |
    +--> Build Docker Image
    |
    +--> Push to ECR
    |
    v
Argo CD
    |
    v
AWS EKS
    |
    v
Deployment Result
```

Link Jira issues to pull requests and commits using the configured development integration. Use a consistent issue-key convention in branch names and commit messages.

Example:

```text
SHOP-245-payment-service-connection-fix
```

A Jira dashboard may show linked development information depending on the integration and Jira edition. For full historical pipeline analytics, use CI/CD APIs or a reporting data pipeline where required.

## 10. Incident management metrics

Incident metrics help operations teams understand service interruptions and response effectiveness.

### 10.1 Mean Time to Acknowledge (MTTA)

\[
MTTA =
\frac{\sum \text{Acknowledgement Time}}
{\text{Number of Incidents}}
\]

Example:

| Incident | Acknowledgement time |
|---|---:|
| Incident 1 | 5 minutes |
| Incident 2 | 8 minutes |
| Incident 3 | 11 minutes |

\[
MTTA = \frac{5+8+11}{3}=8
\]

Average acknowledgement time is 8 minutes.

### 10.2 Mean Time to Restore (MTTR)

For this project, define MTTR as mean time to restore service.

\[
MTTR =
\frac{\sum \text{Time to Restore}}
{\text{Number of Restored Incidents}}
\]

Example:

| Incident | Time to restore |
|---|---:|
| Incident 1 | 30 minutes |
| Incident 2 | 45 minutes |
| Incident 3 | 60 minutes |

\[
MTTR = \frac{30+45+60}{3}=45
\]

Average restoration time is 45 minutes.

Some organizations use MTTR to mean repair, recovery, or resolution time. Document the exact definition used by your organization.

### 10.3 Incident recurrence

Track how often the same underlying problem causes another incident.

A recurring Payment Service database connection issue should be linked to the same problem record where appropriate.

### 10.4 Corrective action completion

Track:

- Actions created after incidents.
- Actions completed.
- Actions overdue.
- Actions verified as effective.
- Repeated incidents after the action was implemented.

Closing an RCA task does not automatically prove that the corrective action prevented recurrence.

## 11. Jira dashboard design

Create separate dashboards for different audiences rather than placing every available gadget on one dashboard.

### Dashboard A — Engineering Delivery

Audience: Engineering Lead and Scrum Master.

Recommended gadgets and reports:

- Sprint burndown.
- Sprint velocity.
- Created versus resolved issues.
- Filter results for blocked work.
- Open work by status.
- Aging issues.
- Sprint completion trend.

### Dashboard B — Quality and Testing

Audience: QA Lead and Development Lead.

Recommended reports:

- Open bugs by priority.
- Bugs created versus resolved.
- Reopened defects.
- Bugs by service.
- UAT issues.
- Defect aging.
- Release-blocking defects.

### Dashboard C — Release Management

Audience: Release Manager and DevOps Lead.

Recommended reports:

- Planned releases.
- Release readiness.
- Issues by release version.
- Outstanding release blockers.
- Change approvals.
- Deployment issue links.
- Rollback records.

### Dashboard D — Incident Management

Audience: Operations and Platform Engineering.

Recommended reports:

- Open P1/P2 incidents.
- Incidents by service.
- Incident priority distribution.
- MTTA and MTTR.
- SLA performance.
- Recurring incidents.
- Open corrective actions.

### Dashboard E — Management Overview

Audience: Engineering Management.

Recommended reports:

- Delivery throughput trend.
- Sprint completion trend.
- Release frequency.
- Defect trend.
- Incident trend.
- High-priority backlog.
- Corrective action completion.

Use the reporting and dashboard features available in your Jira edition. Some gadgets and reports differ between Jira Cloud and Data Center.

## 12. Saved JQL filters

Create reusable filters before adding gadgets to dashboards.

The following examples use `SHOP` as an illustrative Jira project key.

### Filter 1 — Open issues

```jql
project = SHOP
AND statusCategory != Done
ORDER BY priority DESC, created ASC
```

### Filter 2 — Open production bugs

```jql
project = SHOP
AND issuetype = Bug
AND environment = Production
AND statusCategory != Done
ORDER BY priority DESC
```

The `environment` field may be populated as free text or represented through a custom field. Adjust the query to match your configuration.

### Filter 3 — Recently completed work

```jql
project = SHOP
AND statusCategory = Done
AND resolved >= -30d
ORDER BY resolved DESC
```

### Filter 4 — Issues assigned to the current user

```jql
project = SHOP
AND assignee = currentUser()
AND statusCategory != Done
ORDER BY priority DESC
```

### Filter 5 — Release blockers

```jql
project = SHOP
AND fixVersion = "2.0.0"
AND priority in (Highest, High)
AND statusCategory != Done
ORDER BY priority ASC
```

Replace the example release version with the actual Jira version.

### Filter 6 — Unresolved incidents

```jql
project = OPS
AND issuetype = Incident
AND statusCategory != Done
ORDER BY priority ASC, created ASC
```

### Filter 7 — Recently resolved incidents

```jql
project = OPS
AND issuetype = Incident
AND resolved >= -30d
ORDER BY resolved DESC
```

Save each filter with a meaningful name and share it only with the appropriate Jira users or groups.

## 13. Multi-microservice delivery analytics

For a 21-microservice platform, do not analyze only the overall project total. A platform-wide average can hide problems in individual services.

Create Jira components for the services and platform areas.

Known service examples:

- User
- Product Catalog
- Cart
- Orders
- Payment
- Inventory
- Notification

Create components for the remaining services using their actual repository and service names. Do not use placeholder names as production service identities.

### Recommended service-level metrics

| Area | Example metric |
|---|---|
| Development | Open stories by service |
| Quality | Open and reopened bugs by service |
| Sprint | Completed versus committed work by service |
| Release | Release blockers by service |
| CI/CD | Pipeline success rate by repository |
| Deployment | Deployment result by service |
| Incidents | P1/P2 incidents by service |
| Reliability | Availability and error rate from monitoring |

Use consistent service identifiers across Jira, GitHub repositories, CI/CD pipelines, ECR image names, Argo CD applications, and monitoring labels.

This makes cross-system reporting easier.

## 14. Jira and Prometheus/Grafana reporting separation

Jira and observability systems answer different questions.

| Question | Primary system |
|---|---|
| How many stories were completed? | Jira |
| Why is a sprint delayed? | Jira |
| How many bugs are open? | Jira |
| How many releases were planned? | Jira |
| How long did a pipeline take? | Jenkins / GitHub Actions / GitLab |
| Is the Payment API responding? | Prometheus / Grafana |
| What is the current HTTP 5xx rate? | Prometheus / Grafana |
| Which pod is restarting? | Kubernetes metrics and events |
| What exception occurred? | ELK |
| Is the Argo CD application healthy? | Argo CD |

Do not use a Jira issue count as a replacement for service availability, latency, or error-rate monitoring.

## 15. Example management KPI scorecard

The following is an illustrative scorecard. These are example values, not actual platform measurements or recommended universal targets.

| KPI | Example result | Review |
|---|---:|---|
| Sprint completion | 90% | Review carryover |
| Average cycle time | 4 days | Compare trend |
| Weekly throughput | 13.5 issues | Review work mix |
| Reopen rate | 10% | Investigate quality |
| Deployment frequency | 2.8 per week | Review service-level distribution |
| Change failure rate | 10% | Review failed changes |
| Rollback rate | 5% | Review rollback causes |
| MTTA | 8 minutes | Review acknowledgement |
| MTTR | 45 minutes | Review recovery bottlenecks |

A scorecard should show the metric definition, reporting period, source, owner, trend, and any important limitations.

## 16. KPI review cadence

Use different review intervals for different metrics.

| Cadence | Review |
|---|---|
| Daily | Blocked work, aging issues, critical incidents |
| Sprint review | Velocity, completion, carryover, defects |
| Weekly | Throughput, cycle time, backlog health |
| Release review | Deployment results, change failures, rollback trends |
| Monthly | Incident trends, quality, reliability, delivery trends |
| Quarterly | KPI definitions, process improvements, strategic delivery performance |

Do not change KPI definitions frequently. If a definition changes, document the change so historical comparisons remain understandable.

## 17. Delivery analytics improvement process

Use a repeatable improvement cycle.

### Step 1 — Identify the bottleneck

Example: Stories remain in Code Review for an average of three days.

### Step 2 — Investigate the cause

Possible reasons:

- Too few reviewers.
- Large pull requests.
- Missing automated checks.
- Unclear ownership.
- Cross-service dependencies.

### Step 3 — Agree an improvement

Examples:

- Introduce smaller pull requests.
- Define reviewer ownership.
- Add automated quality checks.
- Set review expectations.
- Improve developer documentation.

### Step 4 — Track the change

Create a Jira improvement task with an owner and target date.

### Step 5 — Compare before and after

Review the same metric over comparable periods.

### Step 6 — Confirm the result

Check whether cycle time improved without increasing defects or production incidents.

## 18. Governance and data quality

Poor data produces misleading dashboards.

Apply these controls:

- Define issue types and workflows consistently.
- Require relevant service components.
- Maintain meaningful priorities.
- Use consistent release versions.
- Record actual sprint commitments.
- Keep issue statuses current.
- Avoid duplicate Jira issues.
- Maintain reliable incident timestamps.
- Use consistent service identifiers.
- Restrict dashboard access appropriately.
- Document metric formulas and exclusions.

Avoid using raw ticket counts as individual performance rankings. Metrics should support team improvement and delivery decisions, not encourage ticket splitting or unnecessary status changes.

## 19. Implementation checklist

### Jira configuration

- [ ] Configure project components.
- [ ] Configure issue types.
- [ ] Confirm workflows and statuses.
- [ ] Configure sprint and release versions.
- [ ] Standardize priorities.
- [ ] Define required issue fields.

### Reporting

- [ ] Create delivery JQL filters.
- [ ] Create quality JQL filters.
- [ ] Create release JQL filters.
- [ ] Create incident JQL filters.
- [ ] Build engineering dashboard.
- [ ] Build QA dashboard.
- [ ] Build release dashboard.
- [ ] Build incident dashboard.
- [ ] Build management dashboard.

### Data and integrations

- [ ] Link Jira with GitHub.
- [ ] Validate issue-key references in branches and commits.
- [ ] Validate CI/CD links.
- [ ] Confirm release records are maintained.
- [ ] Confirm incident timestamps and ownership.
- [ ] Define how external CI/CD and monitoring data will be reported.
- [ ] Validate metric calculations against source records.

### Governance

- [ ] Assign metric owners.
- [ ] Document KPI definitions.
- [ ] Agree review cadence.
- [ ] Restrict dashboard sharing.
- [ ] Review data quality.
- [ ] Track improvement actions.

## 20. Expected outcome

After completing this chapter, the project will have a practical delivery analytics model covering:

- Sprint planning and completion.
- Backlog health.
- Lead time and cycle time.
- Throughput and work in progress.
- Defect and quality trends.
- Release performance.
- CI/CD pipeline performance.
- Production incident metrics.
- Service-level delivery reporting.
- Jira dashboard design.
- KPI governance and continuous improvement.

The result is a measurable engineering delivery process in which Jira provides workflow visibility and the existing CI/CD and observability tools provide their respective technical measurements.

**Implementation note:** The examples in this chapter are reference configurations. Validate JQL field names, dashboard gadgets, Jira edition capabilities, and integration data before using them in a live environment.