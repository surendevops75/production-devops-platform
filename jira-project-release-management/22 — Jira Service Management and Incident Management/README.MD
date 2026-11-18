# Chapter 22 — Jira Service Management and Incident Management

## 1. Objective

Jira Service Management (JSM) provides a structured way to receive service requests, manage production incidents, coordinate technical teams, communicate with stakeholders, and track service restoration.

In this project, JSM is used as the operational service-management layer for a production platform containing 21 microservices running on AWS EKS.

The objectives are to:

- Capture incidents from monitoring alerts and user reports.
- Classify incidents based on business impact and urgency.
- Assign incidents to the correct technical team.
- Define escalation and communication procedures.
- Investigate infrastructure, Kubernetes, application, and deployment failures.
- Restore services safely.
- Record incident timelines and root-cause findings.
- Track corrective and preventive actions.

> **Important:** Jira Service Management is used for incident coordination and tracking. Prometheus, Grafana, Alertmanager, and ELK remain responsible for technical monitoring, metrics, alerting, and logs.

## 2. Production environment

The incident-management process covers the following platform components.

| Layer | Technology | Incident examples |
|---|---|---|
| Cloud | AWS | EC2, VPC, IAM, EKS, ECR, RDS |
| Containers | Docker | Image failures, container startup errors |
| Orchestration | Kubernetes / EKS | Pod crashes, scheduling failures |
| Deployment | Helm, Argo CD | Failed synchronization, unhealthy rollout |
| CI/CD | Jenkins, GitHub Actions, GitLab CI/CD | Pipeline failures |
| Monitoring | Prometheus | Metric collection and threshold breaches |
| Visualization | Grafana | Service and infrastructure dashboards |
| Alerting | Alertmanager | Alert routing and notifications |
| Logging | ELK Stack | Application and infrastructure log investigation |
| Service management | Jira Service Management | Incident records, escalation, communication |

## 3. Jira Service Management project setup

Create a dedicated JSM service project.

Example project details:

| Setting | Example |
|---|---|
| Project name | Production Platform Support |
| Project key | OPS |
| Project type | Service management |
| Project owner | Platform Engineering |
| Primary queue | Production Incidents |
| Secondary queue | Service Requests |
| Incident priority | P1, P2, P3, P4 |
| Environment | Production, Staging, Development |

The project key `OPS` is illustrative. Use the actual key configured in your Jira environment.

### Recommended service teams

Create teams or assignment groups for:

1. Platform Engineering
2. Kubernetes / EKS Operations
3. Cloud Infrastructure
4. DevOps / CI/CD
5. Database Operations
6. Application Development
7. Security Operations

The actual assignment mechanism depends on your Jira configuration and whether you use Jira Cloud or Data Center.

## 4. Incident types and request types

Use separate request types so that incidents are not mixed with routine service requests.

### Incident

Used when an existing service is unavailable, degraded, or behaving unexpectedly.

Examples:

- Production API unavailable.
- Kubernetes pods repeatedly restarting.
- Payment service returning HTTP 500.
- Database connections exhausted.
- Argo CD deployment causing application failure.
- High application latency.

### Service request

Used for standard operational requests that do not represent an active service failure.

Examples:

- Request access to a deployment dashboard.
- Request a new development namespace.
- Request a CI/CD pipeline.
- Request a deployment permission.
- Request a new monitoring dashboard.

### Problem

Used to investigate the underlying cause of one or more incidents, particularly recurring incidents.

Example:

Five separate incidents are caused by an application connection-pool configuration problem. Create a problem record to investigate and permanently address the underlying issue.

### Change

Used to record a planned modification to production infrastructure or applications.

Examples:

- EKS cluster upgrade.
- Database version upgrade.
- Production configuration change.
- Network security group modification.
- Application release.

The exact issue types and request types depend on the JSM project configuration.

## 5. Incident lifecycle

Use the following lifecycle as the operational baseline:

1. Incident detected.
2. Incident created in JSM.
3. Initial triage.
4. Severity and priority assigned.
5. Incident owner assigned.
6. Technical investigation.
7. Mitigation and recovery.
8. Service validation.
9. Incident resolution.
10. Post-incident review.
11. Corrective actions tracked.
12. Incident closure.

### Suggested statuses

| Status | Meaning |
|---|---|
| Open | Incident has been received |
| Triage | Impact and urgency are being assessed |
| In Progress | Investigation is active |
| Mitigating | Recovery actions are underway |
| Monitoring | Service appears restored and is being observed |
| Resolved | Recovery has been confirmed |
| Closed | Required documentation and closure steps are complete |

These are recommended workflow statuses, not a claim that every Jira project includes them by default. Configure the workflow to match your service-management process.

## 6. Priority and severity classification

Define severity using impact and urgency rather than relying only on the alert name.

| Priority | Example impact | Response expectation |
|---|---|---|
| P1 — Critical | Major production outage or critical business function unavailable | Immediate response and escalation |
| P2 — High | Major degradation or important service unavailable | High-priority response |
| P3 — Medium | Limited functionality affected; workaround available | Normal operational response |
| P4 — Low | Minor issue with little business impact | Planned investigation |

Response targets must be agreed with the organization and documented in its support policy. Do not treat the example priorities as guaranteed Jira or vendor SLA defaults.

### Example priority assessment

Suppose the Payment Service is unavailable in production.

- Environment: Production
- Service: Payment Service
- Customer impact: Payments cannot be completed
- Business impact: High
- Workaround: No reliable workaround
- Priority: P1

A development environment pod restart with no user impact would normally receive a lower priority.

## 7. Incident fields

Configure the incident form to collect enough information for responders to begin investigation.

Recommended fields:

| Field | Example |
|---|---|
| Summary | Payment API unavailable |
| Description | Payment requests are failing |
| Affected service | Payment Service |
| Environment | Production |
| Priority | P1 |
| Affected component | Application / EKS |
| Detection source | Alertmanager |
| First observed time | Actual timestamp |
| Customer impact | Payments cannot be completed |
| Error message | HTTP 503 |
| Dashboard URL | Relevant Grafana dashboard |
| Log reference | Relevant Kibana search |
| Deployment reference | Git commit or Argo CD application |
| Assigned team | Application Development |
| Incident owner | Assigned responder |

Avoid placing passwords, tokens, secret values, or sensitive customer information in incident descriptions.

## 8. Incident detection and creation

Incidents can originate from multiple sources.

### Source A — Monitoring alerts

Prometheus collects metrics, Alertmanager evaluates and routes alerts, and a configured integration or webhook can create a Jira incident.

Example flow:

```text
Application / Kubernetes
        |
        v
    Prometheus
        |
        v
   Alertmanager
        |
        v
Configured integration
        |
        v
Jira Service Management
        |
        v
Production Incidents queue
```

A Jira ticket is not created merely because Prometheus detects an alert. An integration must be configured, authenticated, tested, and monitored.

### Source B — Customer or support report

A user reports that the application is unavailable through the JSM portal or another approved support channel.

The support team records:

- What is failing?
- When did the failure begin?
- Which functionality is affected?
- How many users are affected?
- Is there a workaround?
- Is the issue still occurring?

### Source C — Engineering team

An engineer identifies a production failure during deployment validation, log analysis, or operational troubleshooting and creates an incident manually.

## 9. Example Alertmanager integration design

The following is a conceptual integration flow. The exact implementation depends on the selected Jira integration, middleware, or webhook service.

```text
Prometheus alert rule
        |
        v
Alertmanager receiver
        |
        v
Webhook / integration service
        |
        |-- Validate alert
        |-- Map severity to priority
        |-- Map service to assignment group
        |-- Check for duplicate alert
        |
        v
Jira Service Management API
        |
        v
Create or update incident
```

### Example alert payload

```json
{
  "alertname": "PaymentServiceDown",
  "status": "firing",
  "severity": "critical",
  "service": "payment",
  "environment": "production",
  "summary": "Payment Service is unavailable",
  "description": "Payment health check is failing",
  "dashboard": "REPLACE_WITH_GRAFANA_DASHBOARD_URL"
}
```

This is an illustrative payload, not a complete Jira API request.

### Integration requirements

- Use an approved Jira API authentication method.
- Store credentials in a secrets manager or secure integration configuration.
- Map alert labels to Jira fields.
- Validate required fields before ticket creation.
- Avoid creating a new incident for every repeated notification.
- Update an existing incident when the same alert is still firing.
- Record the alert fingerprint or another stable correlation key.
- Handle API rate limits, retries, and failed requests.
- Test the integration in a non-production project first.

## 10. Duplicate incident prevention

Repeated monitoring notifications should not create hundreds of identical Jira incidents.

Use a stable correlation key such as:

```text
environment + service + alertname + affected_component
```

Example:

```text
production-payment-PaymentServiceDown-api
```

The integration should:

1. Receive the alert.
2. Calculate or read its stable fingerprint.
3. Search for an active incident with that correlation key.
4. Update the existing incident if one is already open.
5. Create a new incident only when no matching active incident exists.
6. Record the latest alert time and relevant evidence.

Test duplicate handling carefully, including the case where an alert resolves and fires again later.

## 11. Incident assignment and escalation

Define ownership clearly.

| Incident area | Primary team | Escalation |
|---|---|---|
| EKS cluster failure | Platform Engineering | Cloud Infrastructure |
| Pod crash or application failure | Application Development | Platform Engineering |
| CI/CD pipeline failure | DevOps | Platform Engineering |
| RDS connectivity issue | Database Operations | Cloud Infrastructure |
| IAM or security issue | Security Operations | Cloud Infrastructure |
| Argo CD synchronization failure | DevOps / Platform Engineering | Application Development |

### Escalation procedure

For a P1 incident:

1. Assign an incident owner.
2. Notify the relevant on-call responders.
3. Appoint an incident commander when needed.
4. Establish an incident communication channel.
5. Record investigation updates in Jira.
6. Escalate to additional teams if the issue crosses service boundaries.
7. Provide stakeholder updates at the agreed communication interval.
8. Keep a timeline of important decisions and actions.

Do not rely on a Jira status change alone to notify an on-call engineer. Configure and test the required notification and on-call integration.

## 12. Incident roles and responsibilities

| Role | Responsibility |
|---|---|
| Incident commander | Coordinates response and priorities |
| Technical lead | Directs technical investigation |
| Service owner | Understands application behavior and impact |
| Platform engineer | Investigates EKS and infrastructure |
| Communications owner | Provides stakeholder updates |
| Scribe | Maintains the incident timeline |
| Problem owner | Tracks long-term corrective actions |

For smaller incidents, one person may perform multiple roles. For critical incidents, separate coordination from technical investigation wherever possible.

## 13. Production incident example — Payment Service unavailable

### Incident details

| Field | Value |
|---|---|
| Incident | Payment API unavailable |
| Service | Payment Service |
| Environment | Production |
| Platform | AWS EKS |
| Priority | P1 |
| Detection | Alertmanager |
| Symptom | HTTP 503 responses |
| Initial owner | Application Development |
| Supporting team | Platform Engineering |

### Step 1 — Detect the incident

Prometheus evaluates a health-related alert rule.

Illustrative PromQL:

```promql
up{job="payment-service"} == 0
```

This expression is appropriate only if the target and scrape configuration expose the expected `job` label and target.

Alertmanager routes the firing alert to the configured incident integration.

### Step 2 — Create the Jira incident

Create an incident containing:

- Affected service.
- Production environment.
- First observed time.
- Alert name and fingerprint.
- Relevant dashboard.
- Relevant logs.
- Customer impact.
- Initial priority.

### Step 3 — Check Kubernetes

```bash
kubectl get pods -n production
kubectl get deployments -n production
kubectl get events -n production --sort-by=.lastTimestamp
```

Inspect the Payment Service pod:

```bash
kubectl describe pod <payment-pod-name> -n production
```

Check logs:

```bash
kubectl logs <payment-pod-name> -n production
```

For a previous container instance:

```bash
kubectl logs <payment-pod-name> -n production --previous
```

Use the actual namespace and pod name from the cluster.

### Step 4 — Check service endpoints

```bash
kubectl get svc -n production
kubectl get endpoints -n production
kubectl get endpointslices -n production
```

Check whether the Service has healthy endpoints and whether the pod readiness state is correct.

### Step 5 — Check application and infrastructure metrics

Use Grafana and Prometheus to investigate:

- Request rate.
- Error rate.
- Response latency.
- Pod restarts.
- CPU and memory usage.
- Application health.
- Database connection metrics, if exported.

Use ELK to investigate application exceptions, connection failures, and related error messages.

### Step 6 — Review the recent deployment

Check:

- Latest Git commit.
- Pull request and approval.
- CI/CD execution result.
- Container image tag.
- Helm values changes.
- Argo CD synchronization and health status.
- Kubernetes rollout history.

Example commands:

```bash
kubectl rollout history deployment/payment-service -n production
kubectl rollout status deployment/payment-service -n production
```

### Step 7 — Restore service safely

If investigation confirms that a recent GitOps deployment caused the failure:

1. Identify the last known-good application configuration or image version.
2. Review the proposed rollback.
3. Update the GitOps repository to the known-good version.
4. Commit and review the change according to the production process.
5. Allow Argo CD to reconcile the desired state.
6. Validate pod health, endpoints, and application behavior.
7. Record the recovery action in the Jira incident.

**GitOps rule:** Argo CD uses the Git repository as the desired-state source. A manual `kubectl rollout undo` may be overwritten when Argo CD reconciles the application. For a GitOps-managed workload, use the approved Git-based rollback procedure unless your organization has a documented emergency exception.

### Step 8 — Validate recovery

Confirm:

- Payment Service pods are Ready.
- Kubernetes endpoints are healthy.
- Health checks succeed.
- HTTP 5xx errors have returned to normal.
- Request latency is acceptable.
- Payment transactions are completing successfully.
- Monitoring alerts have resolved.
- No related services are experiencing secondary failures.

### Step 9 — Update Jira

Add the incident timeline:

```text
10:05 - Alert received
10:08 - P1 incident created
10:12 - Incident owner assigned
10:18 - Kubernetes investigation started
10:25 - Recent deployment identified as a possible cause
10:32 - GitOps rollback approved and committed
10:40 - Argo CD reconciliation completed
10:45 - Application validation successful
10:55 - Monitoring confirms service recovery
```

These times are illustrative. Use actual timestamps from the incident.

## 14. Incident communication

A production incident needs technical investigation and clear communication.

### Initial notification

Include:

- Incident reference.
- Affected service.
- Current impact.
- Priority.
- Start time.
- Response owner.
- Next update time.

### Progress update

Include:

- What has been investigated.
- What is currently known.
- Current mitigation.
- Remaining impact.
- Next planned action.

### Resolution notification

Include:

- Confirmation that service has recovered.
- Recovery time.
- Validation performed.
- Any remaining limitations.
- Whether a post-incident review is required.

Keep customer-facing updates factual. Do not expose internal credentials, security-sensitive details, or unverified root-cause assumptions.

## 15. Problem management

An incident records a service disruption. A problem record investigates the underlying cause and prevents recurrence.

Example:

Three production incidents have occurred because the Payment Service intermittently exhausts its database connection pool.

Create a problem record containing:

- Related incidents.
- Affected services.
- Frequency and impact.
- Investigation findings.
- Root cause, when confirmed.
- Workaround.
- Permanent corrective actions.
- Action owners and target dates.

Link the related incidents to the problem record using Jira issue links or the configured relationship.

Do not label an assumption as a confirmed root cause.

## 16. Post-incident review and RCA

A post-incident review should be blameless and evidence-based.

Recommended structure:

### Incident summary

What happened, which service was affected, and how long the impact lasted?

### Customer and business impact

- Number or proportion of affected users, if known.
- Transactions or functionality affected.
- Regions or environments affected.
- Data integrity concerns, if any.

### Timeline

Record detection, acknowledgement, investigation, mitigation, recovery, and closure times.

### Technical findings

Include relevant:

- Kubernetes events.
- Application logs.
- Prometheus metrics.
- Grafana screenshots or dashboard references.
- Deployment history.
- Git commits.
- Argo CD synchronization events.

### Root cause

State the confirmed technical cause and supporting evidence. If the investigation is incomplete, record the cause as unknown or under investigation.

### Contributing factors

Examples:

- Missing readiness checks.
- Insufficient deployment validation.
- Incomplete alert coverage.
- Configuration drift.
- Lack of a documented rollback procedure.

### Corrective and preventive actions

| Action | Owner | Priority |
|---|---|---|
| Add deployment validation | DevOps | High |
| Improve Payment Service alerting | Platform Engineering | High |
| Add connection-pool monitoring | Application Development | Medium |
| Update rollback runbook | DevOps | Medium |

Track each action as a Jira task or other suitable work item, with an owner, due date, and verification criteria.

## 17. Change management integration

Some incidents require an emergency change to restore production.

Link the incident to the relevant change record.

Record:

- Reason for the change.
- Affected service.
- Risk and impact.
- Implementation steps.
- Validation steps.
- Rollback plan.
- Approver or authorization.
- Actual implementation time.

An incident priority does not automatically authorize a production change. Follow the organization's emergency-change policy and approval process.

## 18. Jira automation rules

Examples of useful automation:

| Rule | Trigger | Action |
|---|---|---|
| P1 notification | P1 incident created | Notify the configured response group |
| Assignment alert | Incident assigned | Notify the assignee |
| Escalation reminder | P1 remains unacknowledged | Escalate according to policy |
| Resolution update | Incident resolved | Notify the requester |
| RCA task creation | Incident reaches review stage | Create or link review work item |
| Corrective action reminder | Action due date approaches | Notify the action owner |
| Stale incident reminder | Incident has no update for a defined period | Notify the owner |

Configure the rules with appropriate permissions, conditions, and audit checks. Test them using non-production issues before enabling them for live incidents.

## 19. JQL examples

JQL searches should use the actual project key and field values configured in your Jira instance.

### Open production incidents

```jql
project = OPS
AND issuetype = Incident
AND statusCategory != Done
ORDER BY priority ASC, created ASC
```

### Critical incidents

```jql
project = OPS
AND priority = Highest
AND statusCategory != Done
ORDER BY created ASC
```

### Incidents assigned to a team member

```jql
project = OPS
AND assignee = currentUser()
AND statusCategory != Done
ORDER BY priority ASC
```

### Recently resolved incidents

```jql
project = OPS
AND issuetype = Incident
AND statusCategory = Done
AND resolved >= -7d
ORDER BY resolved DESC
```

### Incidents affecting production

```jql
project = OPS
AND "Environment" = Production
AND issuetype = Incident
ORDER BY created DESC
```

Custom field names and values must match the Jira configuration. Some JQL fields require a custom-field identifier or a different field name.

## 20. Incident dashboards and metrics

Build a JSM dashboard or reports using suitable saved filters and gadgets.

Recommended operational indicators:

| Metric | Meaning |
|---|---|
| Incident count | Number of incidents in a period |
| P1/P2 incident count | Number of high-impact incidents |
| MTTA | Mean time to acknowledge |
| MTTR | Mean time to restore or resolve, according to the defined metric |
| Recurrence rate | Repeated incidents for the same underlying problem |
| SLA compliance | Performance against configured service targets |
| Open incident backlog | Incidents not yet resolved |
| Corrective action completion | Actions completed versus actions due |

Define the start and end timestamps consistently before calculating MTTA or MTTR. For example, acknowledgement time should be measured from incident creation to the first recorded acknowledgement.

Jira reporting tracks service-management work. Prometheus and Grafana continue to track runtime behavior such as latency, error rate, CPU, memory, and availability.

## 21. Security and access control

Apply least-privilege access to the JSM project.

Recommended controls:

- Separate customer-facing access from internal responder permissions.
- Restrict sensitive incident details.
- Use approved authentication for integrations.
- Store API credentials securely.
- Review project roles and permissions.
- Preserve incident history and audit records.
- Avoid secrets and sensitive customer data in issue descriptions.
- Review access for users who change teams or leave the organization.

Do not expose internal incident comments to customers unless the communication process explicitly permits it.

## 22. Operational runbooks

Maintain runbooks for recurring production issues.

Suggested runbooks:

```text
runbooks/
├── payment-service-unavailable.md
├── high-http-5xx-errors.md
├── kubernetes-pod-crashloopbackoff.md
├── eks-node-not-ready.md
├── database-connection-exhaustion.md
├── argocd-sync-failure.md
├── ci-cd-pipeline-failure.md
└── production-rollback.md
```

Each runbook should contain:

1. Symptoms.
2. Impact assessment.
3. Required access.
4. Diagnostic commands.
5. Dashboard and log references.
6. Safe mitigation steps.
7. Validation steps.
8. Escalation contacts or teams.
9. Rollback instructions.
10. Post-incident documentation requirements.

Use placeholders for environment-specific account IDs, cluster names, namespaces, and URLs. Do not commit credentials.

## 23. Production operating procedure

Use this checklist during a production incident.

### Detection and triage

- [ ] Alert or report received.
- [ ] Incident created in JSM.
- [ ] Affected service identified.
- [ ] Production impact assessed.
- [ ] Priority assigned.
- [ ] Incident owner assigned.

### Investigation

- [ ] Kubernetes health checked.
- [ ] Application logs reviewed.
- [ ] Prometheus metrics reviewed.
- [ ] Grafana dashboard reviewed.
- [ ] ELK logs reviewed.
- [ ] Recent deployments checked.
- [ ] Database and dependency health checked where relevant.

### Recovery

- [ ] Mitigation selected.
- [ ] Required change authorization obtained.
- [ ] Recovery action recorded in Jira.
- [ ] Argo CD desired state reconciled if GitOps-managed.
- [ ] Application health validated.
- [ ] Customer impact checked.
- [ ] Monitoring alerts returned to normal.

### Closure and learning

- [ ] Resolution details recorded.
- [ ] Timeline completed.
- [ ] Stakeholders notified.
- [ ] RCA or post-incident review completed when required.
- [ ] Corrective actions created and assigned.
- [ ] Related problem and change records linked.
- [ ] Incident closed according to workflow policy.

## 24. Final architecture flow

```text
Customer / Monitoring / Engineering
                 |
                 v
       Jira Service Management
                 |
                 v
       Incident triage and priority
                 |
                 v
       Assignment and escalation
                 |
                 v
       Technical investigation
                 |
      +----------+-----------+
      |          |           |
      v          v           v
 Kubernetes   Prometheus    ELK
      |        Grafana       |
      |          |           |
      +----------+-----------+
                 |
                 v
       Mitigation and recovery
                 |
                 v
       Git / CI/CD / Argo CD
                 |
                 v
       Production validation
                 |
                 v
       Incident resolution
                 |
                 v
       RCA and corrective actions
                 |
                 v
       Reliability improvement
```

## 25. Expected outcome

After implementing this chapter, the project should have:

- A dedicated Jira Service Management project.
- Defined incident types and priorities.
- A documented incident lifecycle.
- Clear ownership and escalation responsibilities.
- A design for monitoring-to-Jira incident integration.
- A duplicate incident prevention strategy.
- A practical Kubernetes production incident simulation.
- GitOps-aware recovery procedures.
- Incident communication and RCA templates.
- JQL filters and incident reporting.
- Security and access-control practices.
- Runbooks and a production incident checklist.

**Implementation note:** The workflows, payloads, field mappings, and JQL examples in this document are reference configurations. They must be adapted and tested against the actual Jira edition, project configuration, integrations, and production environment before use.