# CHAPTER 211 — RCA POSTMORTEM
## Production-grade incident investigation, root cause analysis, and post-incident learning

**Platform context:** AWS, EKS, Kubernetes, 20+ microservices, Docker, ECR, Helm, Argo CD, Terraform, Jenkins, GitHub Actions, GitLab CI/CD, RDS, Redis, RabbitMQ, Kafka, Prometheus, Grafana, and ELK.

**Purpose:** Build a repeatable, blameless, evidence-led process to understand production incidents, explain customer impact, identify technical and organizational contributing factors, and deliver verified corrective actions.

> This chapter is an implementation guide. Replace example service names, namespaces, thresholds, timestamps, account IDs, and commands with values from the target environment. Do not paste credentials, tokens, personal customer data, or secret values into incident records.

---

## 1. Learning objectives

By completing this chapter, the engineer should be able to:

1. Explain the difference between symptom, trigger, root cause, and contributing factor.
2. Establish a reliable incident timeline using UTC timestamps.
3. Collect and preserve evidence from Kubernetes, AWS, CI/CD, databases, messaging, monitoring, and logs.
4. Write a blameless and technically defensible RCA.
5. Distinguish direct technical causes from systemic weaknesses.
6. Quantify customer impact and service degradation.
7. Identify detection, response, mitigation, and recovery gaps.
8. Convert findings into corrective and preventive actions with owners and due dates.
9. Prioritize actions by risk reduction rather than convenience.
10. Validate that actions are completed and effective.
11. Facilitate a productive post-incident review.
12. Communicate findings to engineering, management, security, and affected stakeholders.

## 2. What an RCA is — and what it is not

Root cause analysis (RCA) is a structured investigation into why an incident occurred and why the system allowed it to produce a particular impact.

An RCA is not:

- A list of commands executed during an outage.
- A way to identify and blame an individual.
- A guess based on the first visible error.
- A document that says only “human error”.
- A promise that the same incident can never happen again.
- A replacement for incident response or service restoration.
- A collection of every log line generated during the incident.

A useful RCA explains the system conditions that made the incident possible, the event that triggered it, how the impact spread, how the team detected and mitigated it, and what will change.

## 3. Core terminology

| Term | Meaning | Example |
|---|---|---|
| Incident | Unplanned service disruption or degradation | Checkout requests fail |
| Symptom | Observable behavior | HTTP 503 responses |
| Trigger | Event that starts the failure sequence | New release changes a timeout |
| Direct cause | Immediate technical mechanism | Requests exceed upstream timeout |
| Root cause | Underlying condition that made the failure possible | No representative load test for dependency latency |
| Contributing factor | Additional condition that increased likelihood or impact | Retry policy amplifies traffic |
| Detection gap | Why the issue was not detected earlier | Alert measures pod health, not checkout success |
| Mitigation | Action that reduces ongoing impact | Roll back the release |
| Recovery | Return to a safe, validated service state | Restore healthy traffic and verify orders |
| Corrective action | Change addressing a specific failure mechanism | Add bounded retry and timeout policy |
| Preventive action | System improvement reducing recurrence likelihood or impact | Add dependency failure testing |
| Residual risk | Risk remaining after actions | External provider can still fail |

Avoid using “root cause” as a synonym for the first error message. A crash may be a symptom of a deeper dependency, configuration, capacity, or deployment problem.

## 4. RCA principles

### 4.1 Blameless, not accountability-free

A blameless review examines why an action made sense to a person working with the information, tools, procedures, and time pressure available at that moment. It still records decisions, responsibilities, and follow-up actions.

Use language such as:

- “The deployment process allowed an unreviewed configuration change.”
- “The alert did not include the customer-facing failure signal.”
- “The runbook did not describe this recovery path.”
- “The service retried indefinitely and increased downstream load.”

Avoid:

- “The engineer was careless.”
- “The operator broke production.”
- “The team should have known.”
- “Someone forgot to check.”

### 4.2 Evidence before conclusion

Every important conclusion should link to evidence: logs, metrics, deployment records, Kubernetes events, AWS events, database records, configuration history, or a reproducible test.

### 4.3 Separate facts from hypotheses

Mark statements as:
- **Confirmed:** directly supported by evidence.
- **Likely:** supported by several observations but not fully proven.
- **Unknown:** evidence is unavailable or insufficient.
- **Disproved:** evidence contradicts the hypothesis.

Do not turn an unverified theory into a confirmed root cause.

### 4.4 Focus on system improvement

A good action changes a system, guardrail, test, alert, runbook, architecture, or operational process. “Be more careful” is not a durable action.

---

## 5. RCA workflow from incident to closure

The end-to-end workflow:

1. Open an incident record.
2. Confirm incident scope and severity.
3. Stabilize the service using the incident response process.
4. Preserve time-sensitive evidence.
5. Build a UTC timeline.
6. Quantify customer and business impact.
7. Identify the trigger and failure mechanism.
8. Test competing hypotheses.
9. Identify contributing conditions and control gaps.
10. Review detection, response, mitigation, and recovery.
11. Draft the RCA.
12. Conduct a blameless review with relevant owners.
13. Create prioritized corrective actions.
14. Assign accountable owners and due dates.
15. Verify implementation and effectiveness.
16. Publish the approved report.
17. Track residual risks and close the incident review.

Do not delay urgent mitigation to complete an RCA. Incident response restores service; RCA follows with a disciplined investigation.

## 6. Incident record and evidence workspace

Create a consistent incident workspace:

```text
incidents/
└── INC-2026-1042/
    ├── README.md
    ├── timeline.md
    ├── impact.md
    ├── hypotheses.md
    ├── evidence-index.md
    ├── logs/
    ├── metrics/
    ├── deployment/
    ├── kubernetes/
    ├── aws/
    ├── database/
    ├── messaging/
    ├── customer-communications/
    ├── rca-draft.md
    └── actions.csv
```

Use an approved access-controlled repository or incident management platform. Restrict evidence access according to its sensitivity and retention requirements.

### 6.1 Incident README template

```markdown
# INC-2026-1042 — Checkout API elevated errors

- Status: Investigating / Review / Actions open / Closed
- Severity: SEV-2
- Service owner: Commerce Platform
- Incident commander: <role or name>
- Start time UTC: YYYY-MM-DDTHH:MM:SSZ
- End time UTC: YYYY-MM-DDTHH:MM:SSZ
- Customer impact: <verified summary>
- Trigger: <confirmed event or unknown>
- Mitigation: <what restored service>
- RCA owner: <owner>
- Review date: <date>
- Evidence location: <restricted location>
```

Do not put secret values or unnecessary customer identifiers in the README.

## 7. Severity and scope

Use the organization's severity policy. Example classification:

| Severity | Example impact |
|---|---|
| SEV-1 | Broad critical service outage, major data integrity or security impact |
| SEV-2 | Significant customer-facing degradation or critical workflow failure |
| SEV-3 | Limited impact, partial degradation, or workaround available |
| SEV-4 | Low-impact issue with no material customer disruption |

Severity should be based on impact, not on how difficult the technical problem was to solve.

Record:
- Affected services and endpoints.
- Affected regions, clusters, namespaces, and tenants.
- Number or proportion of affected requests or customers where reliably known.
- Start and end of customer impact.
- Whether data was lost, duplicated, delayed, or exposed.
- Workarounds and degraded modes.
- Third-party dependencies involved.

## 8. Build an accurate UTC timeline

A timeline is the backbone of an RCA. Use UTC consistently and preserve the original timezone when a source uses local time.

### 8.1 Timeline fields

| Field | Description |
|---|---|
| Timestamp UTC | Normalized event time |
| Source | Alert, log, deployment, operator note, AWS event |
| Event | What happened |
| Evidence | Link or evidence identifier |
| Confidence | Confirmed, likely, unknown |
| Impact | Customer or system effect |
| Actor type | Automation, system, operator, external provider |

### 8.2 Example timeline

| UTC | Event |
|---|---|
| 09:02 | Release pipeline begins deployment of checkout-api |
| 09:06 | Deployment completes; new ReplicaSet becomes available |
| 09:08 | Checkout p95 latency begins increasing |
| 09:10 | HTTP 5xx alert fires |
| 09:12 | Incident declared SEV-2 |
| 09:16 | Investigation identifies elevated calls to payment dependency |
| 09:22 | Rollback approved and initiated |
| 09:27 | Previous application version becomes ready |
| 09:31 | Error rate returns to baseline |
| 09:45 | Synthetic checkout test passes |
| 10:05 | Incident downgraded and customer impact ends |

This is an illustrative timeline, not evidence of an actual event. Use source timestamps to build the real one.

### 8.3 Normalize timestamps

Linux UTC time:

```bash
date -u --iso-8601=seconds
timedatectl status
```

Kubernetes event timestamps:

```bash
kubectl get events -A \
  --sort-by=.metadata.creationTimestamp
```

Pod details:

```bash
kubectl get pod -n production <pod-name> -o json \
  | jq '{created: .metadata.creationTimestamp,
         startTime: .status.startTime,
         conditions: .status.conditions}'
```

Do not rely only on the time at which an engineer copied a log. Use the timestamp recorded by the source system and account for clock skew.

---

## 9. Evidence collection and preservation

### 9.1 Evidence handling rules

- Collect evidence as early as practical.
- Prefer read-only commands.
- Record command, operator, timestamp, account, region, cluster, and namespace.
- Preserve raw evidence before filtering or summarizing.
- Do not overwrite original exports.
- Protect sensitive data.
- Record gaps and unavailable sources.
- Follow security/legal retention requirements for suspected data exposure or malicious activity.

### 9.2 Evidence index

```markdown
| ID | Source | Time range UTC | Description | Location | Confidence |
|---|---|---|---|---|---|
| E-001 | Grafana | 09:00–10:00 | Checkout error rate | restricted link | Confirmed |
| E-002 | Argo CD | 09:00–09:30 | Sync and revision history | restricted link | Confirmed |
| E-003 | Kubernetes | 09:00–10:00 | Pod events and restarts | restricted link | Confirmed |
| E-004 | ELK | 09:00–10:00 | Application exception sample | restricted link | Confirmed |
```

### 9.3 Kubernetes evidence

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get events -A --sort-by=.lastTimestamp
```

For a specific workload:

```bash
kubectl describe deployment checkout-api -n production
kubectl rollout history deployment/checkout-api -n production
kubectl get rs -n production -l app=checkout-api -o wide
kubectl get pods -n production -l app=checkout-api -o wide
```

Container logs:

```bash
kubectl logs -n production deploy/checkout-api \
  --since=60m --all-containers=true
```

Previous container logs after a restart:

```bash
kubectl logs -n production <pod-name> \
  --previous --all-containers=true
```

Capture relevant logs promptly. Previous-container logs may disappear after pod replacement.

### 9.4 AWS evidence

Confirm account and region before querying:

```bash
aws sts get-caller-identity
aws configure get region
```

EKS cluster description:

```bash
aws eks describe-cluster \
  --name production-eks \
  --region ap-south-1
```

Node group information:

```bash
aws eks list-nodegroups \
  --cluster-name production-eks \
  --region ap-south-1
```

RDS events:

```bash
aws rds describe-events \
  --source-type db-instance \
  --duration 1440 \
  --region ap-south-1
```

CloudTrail lookup for relevant management events, where available and authorized:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateService \
  --start-time "2026-10-30T08:00:00Z" \
  --end-time "2026-10-30T11:00:00Z" \
  --region ap-south-1
```

Adjust the event name and time window. CloudTrail event history and configured trails have different retention and coverage; absence of an event in one query does not prove that no change occurred.

### 9.5 CI/CD and GitOps evidence

Collect:
- Pipeline run ID and timestamps.
- Commit SHA and branch.
- Build artifact digest.
- Image tag and digest.
- Deployment environment and approver.
- Helm release revision.
- Argo CD application history and sync status.
- Terraform plan/apply logs for infrastructure changes.
- Scanner output relevant to the incident.
- Manual changes made outside the pipeline.

Useful commands:

```bash
kubectl rollout history deployment/checkout-api -n production
helm history checkout-api -n production
argocd app get checkout-api
argocd app history checkout-api
```

Use the Argo CD CLI only where installed and authenticated. Preserve the Git commit and image digest; mutable tags alone are insufficient evidence of exactly what ran.

### 9.6 Database and messaging evidence

Database evidence may include:
- RDS events and failover timestamps.
- Engine logs and relevant error excerpts.
- Connection count and connection errors.
- Slow query samples.
- Lock waits and deadlocks.
- Backup and restore point metadata.
- Replication lag.

Messaging evidence may include:
- RabbitMQ node and queue state.
- Queue depth and unacknowledged messages.
- Consumer connection changes.
- Kafka broker and partition health.
- Under-replicated partitions.
- Consumer group lag and offset changes.
- Dead-letter and retry activity.

Do not export complete production tables or message payloads unless explicitly authorized and necessary. Prefer minimal, redacted, targeted evidence.

---

## 10. Quantify customer impact

A technically interesting failure is not automatically a high-impact incident. Quantify what customers experienced.

### 10.1 Impact dimensions

- Availability: failed requests or unavailable workflows.
- Latency: p50, p95, and p99 changes.
- Correctness: wrong, duplicated, missing, or stale data.
- Freshness: delay in processing events or updates.
- Security: unauthorized access or exposure.
- Scope: services, regions, customer groups, and duration.
- Workaround: whether customers could complete the workflow another way.

### 10.2 Example calculations

If 12,000 requests were received and 1,080 failed:

\[
\text{Error rate} = \frac{1080}{12000}\times 100 = 9\%
\]

If the service was degraded for 24 minutes in a 30-day month:

\[
\text{Time-based availability} =
\frac{30\times24\times60-24}{30\times24\times60}\times100
\]

Be explicit about the measurement method. Request-based success rate and time-based availability are not interchangeable.

### 10.3 Impact statement template

```text
Between <start UTC> and <end UTC>, <service/workflow> experienced
<verified failure mode>. <measured proportion/count> of requests
were affected. Customers experienced <observable impact>.
<Other workflows> remained operational / were also affected.
We found <confirmed data impact or no evidence of data impact>.
The issue was mitigated by <action>.
```

Avoid unsupported claims such as “all users were affected” or “no data was lost” unless evidence supports them.

---

## 11. Form hypotheses before selecting a root cause

Create a hypothesis list and test each one.

| Hypothesis | Evidence needed | Result |
|---|---|---|
| New application release introduced regression | Commit, image digest, before/after metrics | Confirmed / rejected |
| Database connection pool exhausted | Pool metrics, DB connections, logs | Confirmed / rejected |
| Node pressure caused pod disruption | Node conditions, events, kubelet evidence | Confirmed / rejected |
| Payment provider latency increased | Dependency latency, provider status, traces if available | Confirmed / rejected |
| Network policy blocked service traffic | Policy history, connectivity tests | Confirmed / rejected |

Use the following structure:

```text
Hypothesis:
Why it is plausible:
Evidence that would support it:
Evidence that would disprove it:
Evidence collected:
Conclusion:
Confidence:
Remaining uncertainty:
```

Do not stop investigating after finding one plausible explanation. Multiple conditions often combine to create an incident.

---

## 12. Root cause analysis methods

### 12.1 Five Whys

Ask why repeatedly, but stop when the answer reaches a system condition that can be addressed.

Example:

1. Why did checkout requests fail?  
   Because checkout-api timed out while calling payment-api.
2. Why did payment-api respond slowly?  
   Because its database connection pool was saturated.
3. Why was the pool saturated?  
   Because requests were retried aggressively during database latency.
4. Why did retries amplify the load?  
   Because the retry policy had no bounded backoff and no retry budget.
5. Why was this policy deployed?  
   Because integration tests did not simulate dependency slowdown and the release gate did not validate retry behavior.

Potential root and contributing conditions: unsafe retry configuration, insufficient dependency-failure testing, and release gates that did not validate the behavior.

Five Whys is a prompt for investigation, not proof. Validate each answer with evidence.

### 12.2 Fishbone analysis

Group possible causes:

- People and roles: unclear ownership, insufficient training, handoff gaps.
- Process: missing approval, incomplete release checklist, weak change control.
- Technology: timeout, retry, resource limit, dependency failure.
- Configuration: incorrect environment variable, secret, policy, or Helm value.
- Measurement: missing SLI, alert, dashboard, or log context.
- Environment: region, AZ, capacity, DNS, external provider.

Use categories to broaden investigation, not to assign blame.

### 12.3 Change analysis

Compare the last known healthy state with the incident state:

- Application commit.
- Container image digest.
- Helm values.
- Kubernetes manifests.
- Terraform changes.
- IAM policy changes.
- Network policy changes.
- Database schema changes.
- Secret version changes.
- External dependency changes.

A change near the incident time is a lead, not proof of causation.

### 12.4 Fault-tree reasoning

Start with the top-level failure and decompose it:

```text
Checkout unavailable
  OR
  ├── Checkout API unavailable
  │   ├── Pods not ready
  │   ├── No schedulable nodes
  │   └── Startup dependency failure
  ├── Request path unavailable
  │   ├── DNS failure
  │   ├── Ingress or ALB failure
  │   └── Network policy/security group issue
  └── Critical dependency unavailable
      ├── Database connection failure
      ├── Payment provider timeout
      └── Messaging dependency failure
```

Continue decomposing until the investigation reaches testable mechanisms.

---

## 13. Worked RCA: checkout outage after a release

### 13.1 Scenario

A new checkout-api release is deployed to an EKS production cluster. Shortly afterward, checkout latency and HTTP 5xx errors increase. The service depends on payment-api, which uses RDS. The rollback restores service.

This is a teaching scenario. The timings and metrics below are illustrative.

### 13.2 Architecture

```text
Customer
   |
Route 53
   |
ALB / Ingress
   |
checkout-api (EKS)
   |
payment-api (EKS)
   |
RDS
```

### 13.3 Observations

- New checkout-api image deployed.
- Pod readiness initially succeeds.
- Checkout p95 latency rises after rollout.
- Payment API latency rises.
- Database connection usage approaches configured capacity.
- Retry volume increases.
- Rollback reduces request amplification and latency.
- No confirmed evidence of permanent order data loss in this example.

### 13.4 Investigation

1. Compare the deployment timestamp with the first latency increase.
2. Verify image digest and commit SHA.
3. Compare retry and timeout configuration between releases.
4. Inspect payment-api request volume.
5. Inspect RDS connection count and connection errors.
6. Compare successful and failed request samples.
7. Reproduce dependency latency in a test environment.
8. Verify that the previous version does not exhibit the same retry amplification.
9. Confirm that rollback restores baseline metrics.

### 13.5 Example conclusion

**Trigger:** Release of a checkout-api version with a more aggressive retry configuration.

**Direct failure mechanism:** During payment-api and database latency, checkout-api retried requests without sufficient bounded backoff. This increased downstream request volume and worsened connection pressure.

**Contributing factors:**
- Integration tests did not simulate slow downstream dependencies.
- Retry policy lacked a clear retry budget.
- Alerting detected elevated HTTP errors but did not alert on retry amplification or dependency saturation early enough.
- Release validation focused on pod readiness and basic endpoint health.

**Not established:** Do not claim a database outage or permanent data loss unless database and transaction evidence confirms it.

### 13.6 Corrective actions

| Action | Priority | Owner role | Verification |
|---|---|---|---|
| Add bounded exponential backoff and jitter | P1 | Application owner | Unit/integration test |
| Add retry budget and timeout hierarchy | P1 | Platform/application | Load test |
| Add dependency latency alert | P1 | Observability owner | Alert test |
| Add fault-injection test for payment latency | P1 | QA/application | Test report |
| Add release gate for retry behavior | P2 | CI/CD owner | Pipeline evidence |
| Document safe rollback and data checks | P2 | Service owner | Game-day result |

---

## 14. Worked RCA: Kubernetes node pressure

### 14.1 Scenario

Several pods are evicted from a worker node. Customer requests fail because replicas are concentrated on a small number of nodes.

### 14.2 Evidence

```bash
kubectl get nodes
kubectl describe node <node-name>
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl top nodes
kubectl top pods -A
```

Look for:
- `MemoryPressure` or `DiskPressure`.
- Eviction events.
- Pods with high memory use.
- Missing or unrealistic resource requests.
- Insufficient node group capacity.
- Pod topology and anti-affinity settings.
- Cluster autoscaler or node provisioning errors.

### 14.3 Example analysis

The direct cause may be node memory pressure. The deeper causes may include missing memory requests, an unbounded memory workload, insufficient capacity headroom, and no alert on eviction trends.

Do not conclude “Kubernetes failed” when the cluster behaved according to configured resource and scheduling constraints.

### 14.4 Actions

- Define realistic CPU and memory requests and limits.
- Add workload memory trend alerts.
- Validate capacity and autoscaling limits.
- Use topology spread constraints for critical replicas.
- Test node loss and eviction behavior.
- Review PodDisruptionBudgets.
- Add resource sizing to service onboarding.

---

## 15. Worked RCA: Terraform infrastructure change

### 15.1 Scenario

A Terraform change modifies a security group or subnet route. Application pods can no longer reach a required dependency.

### 15.2 Evidence

- Pull request and reviewed diff.
- Terraform plan and apply logs.
- State version and workspace.
- AWS resource configuration before and after.
- VPC flow logs if enabled.
- Application connection errors.
- Security group and route table relationships.

Commands:

```bash
terraform plan
terraform show
terraform state list
```

Run these from the correct repository, workspace, and backend configuration. Avoid changing state during evidence collection.

AWS inspection examples:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-0123456789abcdef0 \
  --region ap-south-1

aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-0123456789abcdef0 \
  --region ap-south-1
```

### 15.3 Example conclusion

A reviewed change removed an egress rule required by a service dependency. The change passed because validation checked Terraform syntax and plan approval but did not run a connectivity test for critical service paths.

### 15.4 Actions

- Add automated policy checks for required network paths.
- Add integration connectivity tests.
- Require service-owner review for shared network modules.
- Document dependency ownership.
- Add post-apply validation and safe rollback criteria.

---

## 16. Worked RCA: Argo CD desired-state drift

### 16.1 Scenario

An operator manually changes a Kubernetes Deployment. Argo CD later reconciles the resource back to the Git-defined state, causing unexpected behavior.

### 16.2 Evidence

```bash
argocd app get checkout-api
argocd app history checkout-api
kubectl get deployment checkout-api -n production -o yaml
kubectl rollout history deployment/checkout-api -n production
```

Compare:
- Git commit.
- Argo CD sync operation.
- Live object.
- Helm values and rendered manifests.
- Manual change record.
- Application behavior before and after reconciliation.

### 16.3 Analysis

The immediate event is reconciliation. The systemic weakness may be an undocumented emergency change process or a missing mechanism to commit emergency changes back to Git.

### 16.4 Actions

- Define break-glass procedure.
- Record emergency changes.
- Reconcile approved changes into Git immediately.
- Add drift alerts and ownership.
- Train responders on Argo CD behavior.
- Test emergency rollback and reconciliation in a non-production environment.

---

## 17. Worked RCA: Kafka consumer lag

### 17.1 Scenario

Orders are accepted, but downstream inventory updates are delayed because Kafka consumer lag grows.

### 17.2 Evidence

Collect:
- Consumer group lag over time.
- Partition distribution.
- Consumer restarts and deployment changes.
- Processing latency.
- Broker health and ISR state.
- Database dependency latency.
- Poison-message and dead-letter counts.

Example:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --group inventory-consumer \
  --describe
```

### 17.3 Analysis questions

- Did event production increase?
- Did consumer throughput fall?
- Did one partition become a hotspot?
- Are consumers blocked on database connections?
- Are retries causing repeated processing?
- Is a poison message blocking progress?
- Did a deployment reduce consumer concurrency?
- Is the lag measured in messages, bytes, or time?

### 17.4 Actions

- Alert on lag and estimated processing delay.
- Define safe concurrency and partition strategy.
- Add poison-message handling.
- Add idempotent processing.
- Test replay and offset recovery.
- Document business reconciliation after delayed processing.

---

## 18. Detection analysis

Ask:

- Which signal first showed the problem?
- How long between impact start and alert?
- Was the alert actionable?
- Did it identify the affected service and dependency?
- Was the alert routed to the correct on-call owner?
- Did dashboards show the customer-facing symptom?
- Were alerts noisy, duplicated, or suppressed?
- Could a synthetic test have detected the issue earlier?

### 18.1 Detection metrics

| Metric | Definition |
|---|---|
| Time to detect (TTD) | Impact start to first reliable detection |
| Time to acknowledge (TTA) | Alert to responder acknowledgement |
| Time to mitigate (TTM) | Impact start or incident declaration to mitigation |
| Time to recover (TTR) | Impact start to validated recovery |
| Alert precision | Proportion of alerts that represent actionable conditions |
| Alert coverage | Critical failure modes with a useful detection signal |

Define measurement boundaries consistently. Different organizations may use different MTTR definitions; state which one is used.

### 18.2 Alert design review

For each alert, record:
- User or service impact represented.
- Metric and threshold.
- Evaluation window.
- Severity.
- Routing destination.
- Runbook link.
- Required action.
- Test method.

An alert without an owner or response path is an operational gap.

---

## 19. Response and mitigation analysis

Review the response without judging individuals:

- Was an incident commander assigned?
- Were roles clear?
- Was the severity appropriate?
- Were customer communications timely?
- Were changes controlled?
- Were risky actions reviewed?
- Was the rollback path known?
- Were dependencies and owners identified?
- Were decisions and timestamps recorded?
- Did responders have the required access?
- Were runbooks accurate and available?

Identify process or tooling changes that would make the next response safer and faster.

## 20. Recovery analysis

Recovery is not complete when a dashboard turns green.

Verify:
- Customer-facing synthetic tests pass.
- Critical business transactions succeed.
- Database consistency checks pass.
- Messaging backlog is controlled.
- No unexpected duplicate processing exists.
- Error rate and latency return to acceptable levels.
- Autoscaling and capacity are stable.
- Alerts and dashboards are operational.
- Security controls remain enabled.
- The incident commander approves closure.

Record any degraded mode or residual risk.

---

## 21. Corrective and preventive action design

### 21.1 Good action characteristics

An action should be:
- Specific.
- Measurable.
- Owned by one accountable role or person.
- Time-bound.
- Verifiable.
- Linked to a finding.
- Prioritized by risk reduction.
- Small enough to track but meaningful enough to change the system.

Weak: “Improve monitoring.”

Strong: “Create an alert when checkout payment dependency p95 latency exceeds the agreed service threshold for five minutes; route it to the Commerce on-call rotation; validate it with a controlled latency test.”

### 21.2 Action categories

| Category | Examples |
|---|---|
| Code | Fix retry logic, memory leak, race condition |
| Configuration | Correct timeout, limits, pool size |
| Architecture | Bulkhead, queue isolation, redundancy |
| Testing | Load, chaos, dependency failure, migration tests |
| CI/CD | Immutable artifacts, validation gates, progressive delivery |
| Security | Least privilege, secret rotation, policy checks |
| Observability | SLI, alert, dashboard, log context |
| Operations | Runbook, ownership, game day, escalation |
| Capacity | Headroom, autoscaling, quotas, partition planning |
| Data | Backup, restore, reconciliation, idempotency |

### 21.3 Prioritization

Use a risk-based approach:

- P0: urgent action needed to prevent imminent severe harm or ongoing exposure.
- P1: high-risk recurrence reduction for critical services.
- P2: meaningful reliability improvement with a planned delivery date.
- P3: lower-risk hygiene or longer-term improvement.

The organization’s formal priority definitions take precedence.

### 21.4 Action tracker

```csv
Action_ID,Incident_ID,Finding,Action,Priority,Owner,Due_Date,Status,Verification_Evidence,Effectiveness_Review
ACT-001,INC-2026-1042,Unbounded retries,Implement bounded retry with jitter,P1,Checkout owner,YYYY-MM-DD,Open,,Pending
ACT-002,INC-2026-1042,Missing dependency alert,Add payment latency alert,P1,Observability owner,YYYY-MM-DD,Open,,Pending
ACT-003,INC-2026-1042,No latency fault test,Add dependency fault test,P2,QA owner,YYYY-MM-DD,Open,,Pending
```

A due date and owner must be real commitments, not placeholders, before the review is closed.

---

## 22. Verify actions and measure effectiveness

Completion is not proof of effectiveness.

For each action:
1. Confirm implementation.
2. Link the pull request, configuration change, test, or runbook.
3. Review the change with the service owner.
4. Test the expected behavior.
5. Validate production telemetry after rollout.
6. Check for unintended side effects.
7. Revisit the action after an agreed observation period.
8. Close only when the acceptance criteria are met.

Example: adding a retry alert is complete only after the alert is tested, routed to the correct team, linked to a runbook, and shown to detect the intended condition.

### 22.1 Effectiveness measures

- Recurrence of the same failure mode.
- Change failure rate.
- Detection time.
- Mitigation and recovery time.
- Percentage of critical services with tested runbooks.
- Percentage of critical backups with successful restore tests.
- Percentage of actions completed on time.
- Alert actionability.
- Error-budget impact.

Do not optimize a metric in isolation. A shorter recovery time is not useful if the recovery procedure increases data loss or security risk.

---

## 23. Post-incident review meeting

### 23.1 Participants

Invite only people needed to explain or improve the system:
- Incident commander.
- Service owner.
- Responders.
- Platform/SRE/DevOps representative.
- Database or messaging owner.
- Security representative when relevant.
- Product or business stakeholder for impact validation.
- Facilitator and note taker.

### 23.2 Preparation

Before the meeting:
- Share the draft timeline.
- Share impact statement.
- Link evidence.
- List confirmed facts and open questions.
- Draft findings and actions.
- Give participants time to review.

### 23.3 Suggested 60-minute agenda

| Time | Topic |
|---|---|
| 0–5 min | Purpose and blameless ground rules |
| 5–15 min | Customer impact and timeline |
| 15–30 min | Technical failure mechanism |
| 30–40 min | Detection, response, and recovery |
| 40–50 min | Corrective actions and priorities |
| 50–55 min | Owners and due dates |
| 55–60 min | Open questions and approval path |

### 23.4 Facilitator questions

- What did customers experience?
- What changed before the incident?
- What signals were available at the time?
- What made the failure possible?
- Which assumptions were incorrect?
- What made diagnosis difficult?
- What helped responders restore service?
- What should be easier next time?
- Which actions reduce the greatest risk?

Do not turn the meeting into a debate about who made a decision. Reconstruct the conditions and information available at the time.

---

## 24. RCA report template

Use this as the complete report structure:

```markdown
# Root Cause Analysis — <Incident ID>

## 1. Executive summary
<Short factual description of what happened and how it was resolved.>

## 2. Incident metadata
- Incident ID:
- Severity:
- Service:
- Environment:
- Region / cluster:
- Start time UTC:
- End time UTC:
- RCA owner:
- Review date:

## 3. Customer impact
- Affected workflows:
- Affected requests/customers:
- Duration:
- Availability/latency impact:
- Data integrity impact:
- Security impact:
- Workaround:

## 4. What happened
<Concise narrative of the failure sequence.>

## 5. Detection
<How and when the incident was detected.>

## 6. Timeline
| UTC | Event | Evidence |
|---|---|---|

## 7. Technical architecture
<Relevant request and dependency flow.>

## 8. Trigger
<Specific event that initiated the failure sequence.>

## 9. Direct cause
<Immediate technical mechanism.>

## 10. Root cause
<Underlying system condition supported by evidence.>

## 11. Contributing factors
<List with evidence.>

## 12. Hypotheses considered
| Hypothesis | Evidence | Conclusion |
|---|---|---|

## 13. Mitigation and recovery
<What was done, by whom/role, and how recovery was validated.>

## 14. What went well
<Controls and actions that helped.>

## 15. What made the incident harder
<System and process gaps, stated without blame.>

## 16. Corrective and preventive actions
| ID | Action | Priority | Owner | Due date | Verification |
|---|---|---|---|---|---|

## 17. Residual risk
<What remains possible and how it is managed.>

## 18. Lessons learned
<Specific learning that changes engineering or operations.>

## 19. Evidence index
<Links to protected evidence.>

## 20. Approval and review
<Service owner, incident commander, security/data owner where relevant.>
```

---

## 25. Executive summary writing

Executives need impact, cause, response, and risk—not every command.

A concise structure:

1. What service was affected?
2. What did customers experience?
3. When did it start and end?
4. What triggered the incident?
5. What restored service?
6. What changes will reduce recurrence?
7. Is any risk still open?

Example:

> On <date>, checkout requests experienced elevated failures for approximately <duration>. The issue followed a checkout-api release that increased downstream request volume during payment dependency latency. The team rolled back the release and validated the checkout flow. The investigation identified gaps in retry controls, dependency-failure testing, and alert coverage. Corrective actions have owners and target dates. No claim about data integrity is made until reconciliation evidence is complete.

---

## 26. Communication templates

### 26.1 Internal review invitation

```text
Subject: Post-incident review — <Incident ID>

We will review the <service> incident from <date/time UTC>.

The objective is to establish a shared, evidence-based timeline, understand
the technical and contributing conditions, review detection and recovery,
and agree on measurable improvements.

Please review the incident timeline and evidence before the meeting.
The review is blameless and focused on improving the system.
```

### 26.2 Stakeholder update

```text
Incident: <ID>
Service: <service>
Status: Resolved / Monitoring / Actions in progress
Customer impact: <verified facts>
Start and end: <UTC timestamps>
Mitigation: <summary>
Data/security status: <confirmed status or investigation ongoing>
Next update: <time or channel>
```

### 26.3 Action overdue reminder

```text
Action: <action ID and title>
Incident: <incident ID>
Owner: <owner>
Due date: <date>
Current status: <status>
Risk if delayed: <specific risk>
Requested update: revised delivery date and blocker, if any
```

---

## 27. Security and data incidents

Security incidents require coordination with the security incident response process. An RCA must not interfere with containment, evidence preservation, legal obligations, or notification requirements.

For suspected compromise:
- Preserve logs and audit trails.
- Restrict evidence access.
- Avoid destructive cleanup before evidence preservation.
- Coordinate with security and legal owners.
- Record credential and key rotation.
- Record affected identities and resources.
- Identify data access or exfiltration evidence.
- Keep sensitive details out of broadly accessible reports.
- Separate confirmed facts from forensic hypotheses.

Do not publish exploit details, customer identifiers, secret material, or sensitive indicators in a general engineering postmortem.

---

## 28. Special considerations for distributed microservices

In a 20+ microservice platform, incidents often cross service boundaries.

### 28.1 Build a dependency map

Example:

```text
frontend
  └── gateway
       ├── user
       ├── catalog
       ├── cart
       └── orders
            ├── payment
            ├── inventory
            └── notification
```

The actual service graph must come from the application architecture and runtime evidence.

For each dependency, record:
- Owner.
- Criticality.
- Timeout.
- Retry policy.
- Circuit breaker or bulkhead behavior, if implemented.
- Health signal.
- Data ownership.
- Recovery order.
- Fallback behavior.

### 28.2 Correlation across services

Use consistent request IDs and correlation IDs in application logs. Ensure IDs are propagated across HTTP and messaging boundaries where appropriate. Avoid logging access tokens, payment details, or sensitive personal information.

Example structured log:

```json
{
  "timestamp": "2026-10-30T09:10:00Z",
  "level": "ERROR",
  "service": "checkout-api",
  "environment": "production",
  "request_id": "example-request-id",
  "dependency": "payment-api",
  "operation": "authorize",
  "duration_ms": 5000,
  "error_type": "UPSTREAM_TIMEOUT"
}
```

This is a schema example, not a real incident log.

### 28.3 Avoid cascading failure

Investigate:
- Retry amplification.
- Timeout mismatch between services.
- Thread or connection pool exhaustion.
- Queue buildup.
- Synchronous dependency chains.
- Lack of load shedding.
- Shared infrastructure bottlenecks.
- Insufficient isolation between critical and non-critical workloads.

A service can be healthy at the pod level while the end-to-end business transaction is failing.

---

## 29. Observability evidence with Prometheus, Grafana, and ELK

The platform in this guide uses Prometheus, Grafana, and ELK. Do not assume Jaeger or OpenTelemetry is deployed.

### 29.1 Prometheus metrics

Useful signals include:
- Request rate.
- Error rate.
- Request duration.
- Pod restarts.
- CPU and memory usage.
- Node pressure.
- HPA desired/current replicas.
- Database connections.
- Queue depth and consumer lag.
- Ingress response codes.

Example PromQL patterns; metric names depend on instrumentation:

```promql
sum(rate(http_requests_total{service="checkout-api",status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="checkout-api"}[5m]))
```

Latency percentile, when a histogram is correctly instrumented:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="checkout-api"}[5m])
  )
)
```

Pod restart increase:

```promql
increase(kube_pod_container_status_restarts_total{
  namespace="production"
}[15m])
```

Do not assume these exact metric names exist. Confirm them in Prometheus before using them in an RCA.

### 29.2 Grafana evidence

Export or link:
- Dashboard name and UID.
- Time range and timezone.
- Query or panel.
- Relevant annotations.
- Screenshot only when necessary.
- Dashboard variable selections.

Preserve the query and time range so another engineer can reproduce the view.

### 29.3 ELK evidence

Search by:
- UTC time window.
- Service name.
- Environment.
- Request/correlation ID.
- Error type.
- Deployment version.
- Dependency.

Redact sensitive values before attaching excerpts to a broadly accessible RCA.

---

## 30. CI/CD and release engineering findings

Review whether the release process had:
- Immutable image identity.
- Build provenance.
- Automated tests.
- SonarQube quality gates.
- Trivy image or filesystem scanning.
- Veracode checks where required by policy.
- Deployment approval.
- Environment-specific configuration validation.
- Progressive rollout or canary where supported.
- Automated smoke tests.
- Rollback instructions.
- Database migration compatibility checks.

A scanner passing does not prove runtime reliability. Security, code quality, functional correctness, and operational readiness are different controls.

### 30.1 Release evidence record

```text
Pipeline:
Run ID:
Repository:
Commit SHA:
Build artifact:
Image digest:
Scanner results:
Test results:
Approval:
Deployment target:
Helm revision:
Argo CD sync:
Start/end UTC:
Post-deployment validation:
```

### 30.2 Change failure analysis

Classify the change:
- Application code.
- Container base image.
- Helm values or Kubernetes manifests.
- Terraform infrastructure.
- IAM or security policy.
- Database schema.
- Secret or certificate.
- External dependency.

Then test whether the change caused the incident or merely coincided with it.

---

## 31. Database migration and schema-change RCA

A deployment may fail because application code and database schema are incompatible.

Review:
- Migration version and execution time.
- Lock duration and blocked queries.
- Backward compatibility.
- Expand-and-contract approach.
- Old and new application versions running simultaneously.
- Rollback behavior.
- Data backfill status.
- Replication lag.
- Connection pool behavior.

Prefer compatible migration sequences:
1. Add backward-compatible schema.
2. Deploy code that can use old and new schema.
3. Backfill data safely.
4. Validate.
5. Move reads/writes.
6. Remove obsolete schema only after old code is retired.

Do not assume reverting an application image reverses a database migration.

---

## 32. RCA quality review checklist

Before publication, confirm:

- [ ] Incident scope and severity are clear.
- [ ] Customer impact is quantified or uncertainty is stated.
- [ ] All timeline entries use normalized UTC.
- [ ] Important claims link to evidence.
- [ ] Trigger is distinguished from root cause.
- [ ] Direct cause and contributing factors are separate.
- [ ] Competing hypotheses are documented.
- [ ] Unknowns are explicitly identified.
- [ ] Detection and response are reviewed.
- [ ] Recovery validation is recorded.
- [ ] Wording is blameless.
- [ ] Actions are specific and measurable.
- [ ] Every action has an owner and due date.
- [ ] Verification criteria are defined.
- [ ] Sensitive information is protected.
- [ ] Residual risks are visible.
- [ ] Service owner has reviewed the report.

---

## 33. Common RCA mistakes

| Mistake | Why it is weak | Better approach |
|---|---|---|
| “Human error” | Stops investigation at the person | Examine controls, information, workflow, and interface |
| “Network issue” | Too broad to test | Identify DNS, routing, policy, security group, or CNI mechanism |
| Timeline based on memory | Can be inaccurate | Use source timestamps |
| No customer impact | Hides business consequence | Quantify affected workflows and duration |
| One hypothesis only | Encourages confirmation bias | Test alternatives |
| Every action is P3 | Fails to manage risk | Prioritize by severity and recurrence risk |
| Action says “monitor closely” | Not measurable | Define metric, threshold, owner, and test |
| Close when PR merges | Does not prove effectiveness | Validate behavior and observe production |
| Blame-focused language | Discourages honest reporting | Describe system conditions |
| Claim no data loss without checks | Can be false reassurance | Reconcile data and state evidence |
| Copy all logs into report | Creates noise and exposure risk | Link restricted, targeted evidence |
| RCA treated as incident response | Delays restoration | Restore first; investigate with preserved evidence |

---

## 34. RCA metrics and program health

Measure the quality of the learning system, not just the number of reports.

Possible measures:
- Percentage of SEV-1/SEV-2 incidents with completed reviews.
- Median time from incident resolution to review.
- Percentage of actions with owners and due dates.
- Percentage of actions verified for effectiveness.
- Percentage of overdue high-priority actions.
- Repeat incidents by failure mode.
- Detection and recovery time trends.
- Percentage of critical services with tested recovery procedures.
- Percentage of critical dependencies with named owners.

Avoid targets that encourage teams to suppress incident reporting or downgrade severity.

---

## 35. Operational ownership model

| Role | Responsibility |
|---|---|
| Incident commander | Coordinates response and closure |
| RCA facilitator | Leads evidence-based review |
| Service owner | Owns service findings and actions |
| Platform owner | Owns shared infrastructure findings |
| Security owner | Reviews security implications |
| Data owner | Validates data integrity and reconciliation |
| Action owner | Delivers assigned improvement |
| Reviewer | Challenges assumptions and verifies quality |

One person may hold multiple roles in a small team, but responsibilities must remain explicit.

---

## 36. Practical exercise: investigate a 503 incident

### 36.1 Scenario

Users receive HTTP 503 from the checkout endpoint. The Deployment reports available replicas, but the application is not completing requests.

### 36.2 Investigation sequence

1. Confirm the customer-facing symptom.
2. Check ALB target health and ingress events.
3. Check Service selectors and endpoints.
4. Check pod readiness and container logs.
5. Check dependency DNS and connectivity.
6. Check database connection errors.
7. Compare the latest deployment and configuration.
8. Form at least three hypotheses.
9. Preserve evidence.
10. Identify the failure mechanism.
11. Restore service using the approved runbook.
12. Validate business transactions.
13. Draft the impact statement and timeline.
14. Propose actions with owners and tests.

Commands:

```bash
kubectl get ingress -n production
kubectl describe ingress checkout -n production
kubectl get svc checkout -n production -o yaml
kubectl get endpoints checkout -n production -o yaml
kubectl get pods -n production -l app=checkout-api -o wide
kubectl describe pods -n production -l app=checkout-api
kubectl logs -n production deploy/checkout-api --since=30m
```

### 36.3 Required exercise output

- Incident summary.
- Timeline.
- Evidence index.
- Three or more hypotheses.
- Confirmed direct cause.
- Root cause and contributing factors.
- Customer impact.
- Mitigation and recovery validation.
- At least three measurable actions.
- A completed quality checklist.

---

## 37. Practical exercise: bad Terraform change

### Scenario

A shared Terraform module changes a security group rule. A service in EKS loses access to RDS.

### Tasks

1. Identify the exact infrastructure change.
2. Compare before and after security group rules.
3. Identify source and destination security groups.
4. Verify routes and network ACLs.
5. Correlate the change with application errors.
6. Determine whether the change is causal.
7. Restore connectivity through the approved change process.
8. Add a test that would have caught the issue.
9. Define a peer-review requirement for shared network modules.

Do not apply an unreviewed emergency Terraform change or edit state to make the incident disappear.

---

## 38. Practical exercise: duplicate order processing

### Scenario

A messaging consumer restarts during a dependency outage. Some orders appear to have been processed twice.

### Tasks

1. Identify the message key and correlation ID.
2. Check consumer group offset changes.
3. Inspect acknowledgement and retry behavior.
4. Identify whether the handler is idempotent.
5. Check database uniqueness constraints.
6. Reconcile order, payment, and inventory records.
7. Determine customer impact.
8. Prevent duplicate side effects.
9. Add a replay test and duplicate-processing alert.

Do not replay production messages until the replay window, idempotency behavior, and business approval are clear.

---

## 39. Practical exercise: write an RCA from evidence

Use the following fictional evidence:

```text
09:02 UTC — Deployment started.
09:06 UTC — New ReplicaSet available.
09:08 UTC — Checkout p95 latency increased.
09:10 UTC — HTTP 5xx alert fired.
09:12 UTC — Incident declared.
09:16 UTC — Payment dependency latency elevated.
09:18 UTC — Database connection utilization elevated.
09:22 UTC — Rollback started.
09:27 UTC — Previous ReplicaSet available.
09:31 UTC — Error rate returned to baseline.
09:45 UTC — Synthetic checkout passed.
```

Write:
- A two-sentence executive summary.
- A customer impact statement.
- A timeline with evidence references.
- A trigger statement.
- A direct cause statement.
- A root cause statement.
- Three contributing factors.
- Five actions with owners, priorities, and verification.

Important: the supplied timeline alone does not prove the retry configuration caused the incident. State that as a hypothesis unless additional configuration and request evidence is supplied.

---

## 40. Interview preparation: RCA and postmortem

### Q1. What is the difference between root cause and trigger?

**Answer:** The trigger is the event that starts the failure sequence. The root cause is the underlying system condition that allowed the trigger to produce the incident impact. A deployment may be the trigger, while missing compatibility tests or unsafe retry behavior may be root or contributing conditions.

### Q2. How do you conduct an RCA for a production Kubernetes incident?

**Answer:** I first ensure the service is stable and evidence is preserved. I collect deployment history, pod events, logs, metrics, node conditions, ingress and service state, and dependency health. I build a UTC timeline, quantify impact, test hypotheses, identify the failure mechanism and contributing factors, then create measurable actions with owners and verification criteria.

### Q3. How do you keep an RCA blameless?

**Answer:** I describe system behavior and operating conditions rather than judging individuals. I ask what information and controls were available at the time, why the process made the action possible, and how the system can be made safer.

### Q4. What if the root cause is unknown?

**Answer:** I state what is confirmed, what remains unknown, and which evidence is missing. I do not invent certainty. I create actions to improve observability or reproduce the failure and keep the investigation open if the risk warrants it.

### Q5. How do you prioritize corrective actions?

**Answer:** I prioritize by customer impact, severity, recurrence likelihood, blast radius, and risk reduction. Each action needs an owner, due date, acceptance criteria, and evidence of effectiveness.

### Q6. What is the difference between MTTR and TTR?

**Answer:** Organizations use MTTR to mean different things, such as mean time to repair, restore, or recover. I define the exact start and end boundaries in the report and use them consistently.

### Q7. Why is “human error” not a sufficient root cause?

**Answer:** It does not explain why the system allowed one action to cause an incident. I investigate workflow design, safeguards, permissions, interface feedback, training, automation, and time pressure.

### Q8. How do you prove that a corrective action worked?

**Answer:** I define acceptance criteria before implementation, verify the change, run a controlled test, inspect production signals after rollout, and review effectiveness over an agreed period.

### Q9. How do you investigate a cascading failure across microservices?

**Answer:** I map the request and event dependency paths, correlate timestamps and request IDs, compare latency and error signals across services, inspect retries, timeouts, pools, queues, and shared dependencies, and identify where load amplification began.

### Q10. What should be included in an executive RCA?

**Answer:** Customer impact, duration, trigger, confirmed cause, mitigation, current risk, and prioritized improvements. Detailed logs and commands belong in linked technical evidence.

---

## 41. Interview scenario: deployment succeeded but customers see 503

A successful Kubernetes rollout does not prove that the application is usable. Investigate the full request path:

1. DNS resolves to the intended endpoint.
2. ALB listener and target group are healthy.
3. Ingress routes to the correct Service.
4. Service selectors match ready pods.
5. EndpointSlices contain expected endpoints.
6. Readiness probes represent real application readiness.
7. Application can reach required dependencies.
8. Authentication and authorization work.
9. Critical business transaction succeeds.

Then compare deployment revision and configuration against the last known healthy state. Preserve evidence before rollback, and validate the rollback at the business-transaction level.

## 42. Interview scenario: Terraform apply partially failed

Do not immediately rerun apply or manually delete resources.

1. Preserve full error output.
2. Confirm account, region, workspace, and backend.
3. Inspect state and actual cloud resources.
4. Determine whether resources were created but not recorded.
5. Review dependency and provider errors.
6. Run a fresh plan.
7. Reconcile state only through a reviewed procedure.
8. Apply a reviewed plan after the state and real infrastructure are understood.

The RCA should examine why the change was not safely recoverable and whether plan review, module testing, state protection, and staged rollout were adequate.

## 43. Interview scenario: CI/CD pipeline caused an incident

Investigate:
- Source commit and pull request.
- Pipeline run and stages.
- Build artifact identity.
- Security and quality checks.
- Deployment strategy.
- Environment-specific values.
- Approval records.
- Smoke tests and rollback.
- Whether the pipeline allowed a known failure mode.

Actions may include immutable artifact promotion, automated tests, policy gates, canary deployment, and post-deployment validation.

## 44. Interview scenario: monitoring did not detect the issue

Determine whether:
- The metric existed.
- The metric represented customer impact.
- The threshold and window were appropriate.
- The alert was enabled and routed.
- The responder had a runbook.
- The dashboard showed dependency health.
- The signal was tested.

Do not simply lower every alert threshold. Improve signal quality and response usefulness.

---

## 45. Governance and review lifecycle

Suggested lifecycle:

```text
Incident resolved
      |
Evidence preserved
      |
RCA owner assigned
      |
Timeline and impact validated
      |
Technical investigation
      |
Blameless review
      |
Actions approved
      |
Owners and due dates assigned
      |
Implementation
      |
Effectiveness verification
      |
Residual risk accepted or mitigated
      |
RCA closed
```

Define service-level expectations for review completion according to incident severity and organizational policy. Do not rush a complex investigation merely to meet a generic document deadline.

## 46. Knowledge sharing and confidentiality

Share lessons that help other teams:
- Failure pattern.
- Detection signal.
- Safe mitigation.
- Preventive design.
- Reusable test or runbook.

Before publishing outside the incident team:
- Remove customer identifiers.
- Remove credentials and tokens.
- Remove private IPs or infrastructure details when inappropriate.
- Remove exploit-enabling details from public documents.
- Obtain security and communications approval where required.

A public incident summary may be shorter and less technical than the internal RCA.

## 47. Reusable RCA checklist for every critical service

Before an incident:
- [ ] Service owner and dependencies are documented.
- [ ] SLOs and customer-facing signals exist.
- [ ] Dashboards and alerts are tested.
- [ ] Deployment revisions and image digests are retained.
- [ ] Runbooks are accessible.
- [ ] Backup and recovery procedures are tested.
- [ ] Emergency access is controlled.
- [ ] Evidence retention is understood.

After an incident:
- [ ] Incident ID and severity recorded.
- [ ] Impact window established.
- [ ] Evidence preserved.
- [ ] UTC timeline created.
- [ ] Trigger and root cause distinguished.
- [ ] Hypotheses tested.
- [ ] Detection and recovery reviewed.
- [ ] Actions prioritized and assigned.
- [ ] Verification completed.
- [ ] Lessons shared safely.

---

## 48. Final chapter implementation project

Create a complete RCA package for one controlled incident in a non-production environment.

### Required implementation

1. Deploy a sample microservice to a test EKS namespace.
2. Record baseline request rate, latency, error rate, pod health, and dependency health.
3. Introduce one controlled failure, such as an invalid configuration or dependency delay.
4. Detect the issue using the available Prometheus/Grafana/ELK setup.
5. Follow the incident response runbook and restore service.
6. Preserve Kubernetes, deployment, and monitoring evidence.
7. Build a UTC timeline.
8. Quantify impact using test traffic.
9. Form and test multiple hypotheses.
10. Write the RCA using the report template.
11. Conduct a blameless review.
12. Create corrective actions.
13. Implement at least one action.
14. Re-run the same failure exercise.
15. Compare the result and record whether the action reduced impact or detection/recovery time.

### Required deliverables

```text
211-RCA-Postmortem/
├── README.md
├── incident-summary.md
├── timeline.md
├── impact-analysis.md
├── hypotheses.md
├── evidence-index.md
├── rca-report.md
├── actions.csv
├── review-notes.md
├── validation-results.md
└── lessons-learned.md
```

### Completion criteria

The chapter is complete when you can:

- Explain an incident without confusing symptom, trigger, and root cause.
- Build a timestamped evidence-based timeline.
- Explain customer impact with defensible measurements.
- Investigate Kubernetes, AWS, CI/CD, database, and messaging failure modes.
- Produce a blameless RCA that separates facts from hypotheses.
- Create actions with measurable acceptance criteria.
- Verify action effectiveness through testing and operational evidence.
- Present the findings clearly to both engineers and stakeholders.

---

## 49. Production RCA quick-reference

### First questions

1. What failed from the customer's perspective?
2. When did impact begin and end?
3. What changed before the failure?
4. What evidence confirms the sequence?
5. What was the direct failure mechanism?
6. Which system conditions contributed?
7. Why did detection not happen earlier?
8. What restored service?
9. How was recovery validated?
10. What measurable changes will reduce risk?

### Essential commands

```bash
# Identity and context
aws sts get-caller-identity
kubectl config current-context

# Kubernetes state
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp

# Workload history
kubectl rollout history deployment/<name> -n <namespace>
kubectl describe deployment <name> -n <namespace>

# Logs
kubectl logs -n <namespace> deploy/<name> --since=60m
kubectl logs -n <namespace> <pod> --previous

# Helm and GitOps
helm history <release> -n <namespace>
argocd app get <application>
argocd app history <application>

# Terraform evidence (from correct workspace)
terraform show
terraform state list
terraform plan

# AWS resource inspection
aws eks describe-cluster --name <cluster> --region <region>
aws rds describe-events --source-type db-instance --duration 1440 --region <region>
```

Commands with angle-bracket placeholders must be replaced. Some commands require installed CLIs and appropriate permissions. Use read-only access for evidence gathering wherever possible.

## 50. Chapter conclusion

A strong postmortem turns a production failure into a concrete improvement in engineering and operations. The objective is not to produce a perfect narrative or identify a person to blame. It is to understand the system, preserve facts, explain customer impact, identify contributing conditions, and verify that corrective actions make the platform safer.

For the 20+ microservices EKS platform, every significant incident should improve at least one of the following: architecture, application behavior, deployment safety, security controls, observability, capacity planning, recovery procedures, or operational readiness.

**Chapter 211 — RCA Postmortem complete.**
