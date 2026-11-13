# CHAPTER 209 — PRODUCTION ROLLBACK

**Phase 21 — Final Simulation**

**File:** `209-Production-Rollback.md`

---

# 1. Objective

This chapter explains how to safely roll back a production release in a 20+ microservices application running on AWS EKS.

A production rollback is a controlled change that returns an affected component to a previously validated state when a new release causes unacceptable customer impact.

The objective is not simply to run `kubectl rollout undo`. A production engineer must identify the failed change, select a compatible known-good version, understand data and dependency compatibility, choose the correct rollback mechanism, preserve the GitOps source of truth, and verify the complete customer journey.

We will cover:

- Rollback decision-making during an incident
- Rollback versus roll-forward
- Known-good release identification
- Immutable image tags and digests
- Kubernetes Deployment rollback
- Helm release rollback
- Git revert and GitOps rollback with Argo CD
- Rollback through environment overlays
- Coordinated rollback across microservices
- Database and schema compatibility
- Configuration and secret rollback
- Feature-flag rollback
- Terraform and infrastructure rollback precautions
- Stateful workload and messaging considerations
- Rollback approval and audit evidence
- Monitoring and business validation
- Failure scenarios and a complete production simulation

**Important:** These are implementation examples. Replace sample names, namespaces, AWS account IDs, regions, repository paths, image digests, and revision numbers with verified environment values. Never execute a rollback in production until the target, compatibility, approval, and stop conditions have been checked.

# 2. Production Environment

## 2.1 Application architecture

The example platform uses:

- AWS EKS for Kubernetes workloads
- Amazon ECR for container images
- Helm for packaging Kubernetes applications
- Argo CD for GitOps deployment
- GitHub or GitLab repositories for application and deployment source
- Jenkins, GitHub Actions, or GitLab CI for build and validation pipelines
- Amazon RDS for relational database workloads
- Redis for caching
- RabbitMQ and/or Kafka for asynchronous messaging
- Prometheus and Grafana for metrics and dashboards
- ELK Stack for centralized logs
- AWS ALB and Route 53 for external application access

```text
Developer
   |
   v
Application Git Repository
   |
   v
CI Pipeline
   |-- Unit / integration tests
   |-- SonarQube
   |-- Trivy
   |-- Build image
   |-- Push immutable image to ECR
   |
   v
GitOps Repository
   |-- Helm chart / values
   |-- Environment configuration
   |-- Image digest
   |
   v
Argo CD
   |
   v
AWS EKS
   |-- Frontend
   |-- User
   |-- Catalogue
   |-- Cart
   |-- Order
   |-- Payment
   |-- Inventory
   |-- Notification
   |-- Other microservices
   |
   +--> RDS
   +--> Redis
   +--> RabbitMQ / Kafka
   +--> ALB
   |
   v
Prometheus / Grafana / ELK
```

## 2.2 Example release state

Assume the payment service was promoted to production:

| Item | Value |
|---|---|
| Service | `payment` |
| Namespace | `roboshop` |
| Previous known-good release | `2.4.17` |
| New release | `2.5.0` |
| New image | `payment:2.5.0` |
| Image registry | Amazon ECR |
| Deployment controller | Argo CD |
| Packaging | Helm |
| Symptom | Payment API error rate increased |
| Customer impact | Checkout failures |

The values are illustrative. Confirm the actual release metadata before taking action.

# 3. What Is a Production Rollback?

A rollback returns a changed component to a previously validated release or configuration.

Example:

```text
Before deployment:
payment 2.4.17
       |
       v
Deploy new release
       |
       v
payment 2.5.0
       |
       v
Error rate increases
       |
       v
Confirm release is responsible
       |
       v
Select known-good 2.4.17
       |
       v
Rollback through approved workflow
       |
       v
Validate checkout and payment
```

Rollback is a mitigation. It does not automatically prove the root cause is fixed.

The incident may remain open while engineers investigate the defect, check data integrity, and plan a corrected release.

# 4. Rollback, Roll Forward, and Recovery

These actions are related but not interchangeable.

| Action | Meaning | Example |
|---|---|---|
| Rollback | Return to a previous known-good release | Payment `2.5.0` to `2.4.17` |
| Roll forward | Deploy a new corrective release | Payment `2.5.1` |
| Configuration revert | Restore a previous safe setting | Restore a connection pool value |
| Feature disablement | Turn off a problematic capability | Disable new checkout flow |
| Recovery | Restore the service and business workflow | Re-establish successful checkout |

Use rollback when the old release is compatible with current APIs, data, configuration, and dependencies.

Use roll forward when reverting would be unsafe, for example after an incompatible database migration or when the previous version contains a serious security vulnerability.

# 5. When Should We Roll Back?

Consider rollback when:

- Customer-facing errors began immediately after a release.
- The affected service or configuration change is identified.
- The previous release is known to be healthy.
- The previous image is still available and its digest is known.
- API and database compatibility have been checked.
- The rollback has a bounded blast radius.
- The rollback action is approved under the incident or emergency-change process.
- There is a clear validation plan and stop condition.

Do not roll back only because a dashboard is red. First establish whether the release is causally related.

# 6. When Should We Not Roll Back?

Pause and assess before rollback when:

- A destructive or non-backward-compatible schema migration has run.
- The previous application version cannot read current records.
- A message format has changed and old consumers cannot process new messages.
- A secret or credential has been rotated and the old credential has been revoked.
- The old version contains a known critical security vulnerability.
- The previous image or chart cannot be verified.
- The failure is caused by an external dependency or infrastructure issue unrelated to the release.
- Rollback would increase the incident's blast radius.
- The previous version is not compatible with current configuration or API contracts.

In these cases, consider a targeted mitigation, feature disablement, configuration correction, or small forward fix.

# 7. Rollback Decision Checklist

Before execution, answer:

1. Which customer workflow is affected?
2. Which service or component changed?
3. What is the exact deployment timestamp?
4. Which Git commit introduced the change?
5. Which image digest is currently deployed?
6. Which previous version was healthy?
7. Is that previous image still available?
8. Did the release modify a database schema?
9. Did it change an API or event/message contract?
10. Did it change environment variables or ConfigMaps?
11. Did it change Secrets or identity permissions?
12. Does the old version have sufficient cluster capacity?
13. Is the rollback action approved?
14. How will recovery be measured?
15. What condition will stop the rollback?

If a critical compatibility question is unknown, do not assume the rollback is safe.

# 8. Rollback Architecture and Ownership

```text
Incident Commander
       |
       +--> Service Owner
       |       |
       |       +--> Identify known-good release
       |       +--> Check application compatibility
       |
       +--> Platform / DevOps Engineer
       |       |
       |       +--> Verify cluster and namespace
       |       +--> Execute approved deployment rollback
       |       +--> Verify Argo CD / Kubernetes state
       |
       +--> Database Owner
       |       |
       |       +--> Check schema and data compatibility
       |
       +--> QA / Product Owner
       |       |
       |       +--> Verify customer workflow
       |
       +--> Communications Owner
               |
               +--> Record timeline and status
```

One person may hold multiple roles in a small team, but the decision, execution, and validation responsibilities must be clear.

# 9. Pre-Rollback Evidence Collection

Collect evidence before changing the cluster, where time and incident impact allow.

## 9.1 Confirm the Kubernetes context

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Confirm the AWS account and region using the approved account identity process. Do not rely only on a context name.

## 9.2 Capture deployment state

```bash
kubectl get deployment payment -n roboshop -o wide
kubectl describe deployment payment -n roboshop
kubectl rollout history deployment/payment -n roboshop
kubectl get rs -n roboshop -l app=payment
kubectl get pods -n roboshop -l app=payment -o wide
```

## 9.3 Capture the current image

```bash
kubectl get deployment payment -n roboshop \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Capture the live pod image IDs as well:

```bash
kubectl get pods -n roboshop -l app=payment \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{range .status.containerStatuses[*]}{.image}{"  "}{.imageID}{"\n"}{end}{end}'
```

## 9.4 Capture recent events and logs

```bash
kubectl get events -n roboshop \
  --sort-by=.metadata.creationTimestamp | tail -50

kubectl logs deployment/payment -n roboshop \
  --since=30m --all-containers=true
```

For a restarted container, inspect previous logs:

```bash
kubectl logs <pod-name> -n roboshop \
  --previous --all-containers=true
```

Use the actual pod name. Redact tokens, credentials, customer information, and other sensitive values before sharing evidence.

# 10. Identify the Known-Good Release

A rollback target must be evidence-based.

Use multiple sources:

- Git commit history
- GitOps repository history
- Argo CD application history
- Helm release history
- Kubernetes rollout history
- CI pipeline records
- ECR image metadata and digest
- Release approval records
- Previous monitoring and test results

Example:

```text
Release       Git revision       Image
2.4.16        9c2f...             sha256:aaa...
2.4.17        2bd1...             sha256:bbb...  <- known good
2.5.0         7a91...             sha256:ccc...  <- current failure
```

Do not select a release merely because its version number is lower. Verify that it actually passed production validation.

# 11. Immutable Image Rollback

## 11.1 Why immutable references matter

Avoid relying on mutable tags such as:

```text
latest
stable
production
```

A mutable tag may point to different image content over time.

Prefer a versioned tag and, where supported by the deployment process, an image digest:

```text
<account-id>.dkr.ecr.<region>.amazonaws.com/payment:2.4.17
```

or:

```text
<account-id>.dkr.ecr.<region>.amazonaws.com/payment@sha256:<verified-digest>
```

The digest must be copied from trusted release metadata, not guessed or typed from memory.

## 11.2 Verify the image in ECR

Example:

```bash
aws sts get-caller-identity

aws ecr describe-images \
  --repository-name payment \
  --region <region> \
  --image-ids imageTag=2.4.17
```

Verify the account, region, repository, tag, digest, scan status, and release record.

Do not delete the failed image or release evidence during an incident. It may be needed for investigation.

# 12. Kubernetes Deployment Rollback

Kubernetes Deployments maintain rollout revisions while the relevant ReplicaSets remain available.

## 12.1 Inspect rollout history

```bash
kubectl rollout history deployment/payment -n roboshop
```

Inspect a specific revision:

```bash
kubectl rollout history deployment/payment \
  --revision=4 -n roboshop
```

Revision numbers are cluster-specific. Never assume revision `4` is the correct release.

## 12.2 Perform an emergency rollout undo

Only when the deployment is not being continuously reconciled by GitOps, or when this action is explicitly part of the approved emergency procedure:

```bash
kubectl rollout undo deployment/payment -n roboshop
```

Watch the rollout:

```bash
kubectl rollout status deployment/payment \
  -n roboshop --timeout=5m
```

Inspect the result:

```bash
kubectl get deployment payment -n roboshop
kubectl get pods -n roboshop -l app=payment -o wide
kubectl get rs -n roboshop -l app=payment
```

## 12.3 Roll back to a specific revision

```bash
kubectl rollout undo deployment/payment \
  --to-revision=<verified-revision> \
  -n roboshop
```

Use a revision only after matching it to the known-good ReplicaSet and release record.

## 12.4 Important limitation

A Kubernetes rollout undo restores a prior Deployment pod template. It does not automatically roll back:

- Database schema or data
- External resources
- Secrets stored outside the Deployment
- ConfigMaps that are independently managed
- Message contents already published
- Side effects already performed by the application
- Other microservices
- Infrastructure provisioned by Terraform

A Deployment rollback is not a complete application recovery plan.

# 13. Helm Rollback

Helm stores release revisions and can restore a previous chart release.

## 13.1 Inspect release history

```bash
helm list -n roboshop
helm status payment -n roboshop
helm history payment -n roboshop
```

Review the chart version, values, revision, and status.

## 13.2 Inspect the target revision

```bash
helm get values payment -n roboshop --revision <revision>
helm get manifest payment -n roboshop --revision <revision>
```

Confirm that the target revision corresponds to the known-good release.

## 13.3 Execute Helm rollback

```bash
helm rollback payment <verified-revision> \
  -n roboshop --wait --timeout 5m
```

Verify:

```bash
helm status payment -n roboshop
helm history payment -n roboshop
kubectl rollout status deployment/payment -n roboshop
```

## 13.4 Helm and GitOps ownership

If Argo CD manages the Helm release, a direct `helm rollback` can create drift. Argo CD may later reconcile the cluster to the Git-defined state.

For a permanent GitOps rollback, change the desired state in Git and let Argo CD reconcile it. Use direct Helm rollback only if the incident procedure explicitly authorizes it and the follow-up Git correction is assigned.

# 14. GitOps Rollback with Argo CD

For this platform, Git should normally remain the source of truth for persistent production configuration.

## 14.1 Normal GitOps flow

```text
Application release
       |
       v
CI builds and tests image
       |
       v
Image pushed to ECR
       |
       v
GitOps values updated
       |
       v
Pull request reviewed
       |
       v
Production approval
       |
       v
Argo CD syncs desired state
       |
       v
EKS runs approved version
```

## 14.2 Inspect Argo CD state

```bash
argocd app get roboshop-prod
argocd app history roboshop-prod
argocd app diff roboshop-prod
```

Confirm:

- Application and destination cluster
- Namespace
- Current Git revision
- Sync status
- Health status
- Image reference
- Pending or failed operations

## 14.3 Identify the bad Git commit

```bash
git log --oneline --decorate -20
```

For a specific deployment values file:

```bash
git log --oneline -- \
  environments/prod/payment/values.yaml
```

Inspect the candidate commit:

```bash
git show <bad-commit>
```

Confirm it changed the affected service and introduced the failing release.

## 14.4 Revert the production change

On a controlled working branch:

```bash
git checkout main
git pull --ff-only origin main
git checkout -b rollback/payment-2.5.0
git revert <bad-commit>
```

If the commit contains unrelated changes, do not blindly revert the entire commit. Create a reviewed corrective commit that changes only the affected production setting.

Inspect the diff:

```bash
git diff --stat
git diff
git status
```

Push the branch:

```bash
git push -u origin rollback/payment-2.5.0
```

Open a pull request or merge request according to the emergency-change policy. Obtain the required approval and merge through the protected workflow.

Do not force-push or rewrite shared production history as a routine rollback method.

## 14.5 Reconcile through Argo CD

After the approved Git change is merged:

```bash
argocd app get roboshop-prod
argocd app diff roboshop-prod
```

If automated sync is not enabled and the change is approved:

```bash
argocd app sync roboshop-prod
```

Monitor:

```bash
argocd app wait roboshop-prod \
  --health --sync --timeout 300
```

Then verify the actual EKS image and workload state. Argo CD reporting Synced and Healthy is necessary but not sufficient to prove customer recovery.

# 15. GitOps Values Example

An illustrative Helm values file:

```yaml
# environments/prod/payment/values.yaml
image:
  repository: <account-id>.dkr.ecr.<region>.amazonaws.com/payment
  tag: "2.4.17"
  # Prefer a verified immutable digest when supported by the chart.
  # digest: "sha256:<verified-digest>"

replicaCount: 6

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "768Mi"

autoscaling:
  enabled: true
  minReplicas: 6
  maxReplicas: 20
```

The values are examples, not production sizing recommendations. Preserve the approved resource and autoscaling settings from the known-good configuration. Do not accidentally revert unrelated tuning or security changes.

After the Git change, verify that the chart renders the expected image and settings.

# 16. Rollback Through Environment Overlays

A repository may separate environments:

```text
gitops-repo/
├── applications/
│   └── payment/
├── charts/
│   └── payment/
└── environments/
    ├── dev/
    │   └── payment/
    ├── qa/
    │   └── payment/
    ├── staging/
    │   └── payment/
    └── prod/
        └── payment/
            └── values.yaml
```

A production rollback must modify the production desired state, not accidentally change development or staging.

Before merging:

```bash
git diff origin/main...HEAD -- \
  environments/prod/payment/
```

Review the exact file and environment path. Confirm that the same image digest is not unintentionally promoted or reverted in another environment.

# 17. Rollback with Kustomize

If the application uses Kustomize rather than Helm, the production overlay may pin an image:

```yaml
# environments/prod/payment/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: roboshop

resources:
  - ../../base/payment

images:
  - name: payment
    newName: <account-id>.dkr.ecr.<region>.amazonaws.com/payment
    newTag: "2.4.17"
```

Validate the rendered output:

```bash
kubectl kustomize environments/prod/payment
```

Review the rendered manifests and Git diff before merging. Argo CD should reconcile the approved overlay.

# 18. Rollback Across Multiple Microservices

Some releases contain coordinated API or event-contract changes.

Example:

```text
Frontend v3
   |
   v
Order v3
   |
   v
Payment v3
   |
   v
Inventory v3
```

If the new versions depend on one another, rolling back only one service may break the workflow.

## 18.1 Build a compatibility matrix

| Caller / Producer | Target / Consumer | Compatibility question |
|---|---|---|
| Frontend | Order API | Can old frontend call current API? |
| Order | Payment API | Can old order service use current payment API? |
| Payment | Inventory API | Are request and response contracts compatible? |
| Producer | Kafka consumer | Can old consumer read newly published events? |
| Publisher | RabbitMQ consumer | Are routing key and payload compatible? |

## 18.2 Prefer backward-compatible releases

A safer release sequence is:

1. Add new fields or endpoints without removing old ones.
2. Deploy consumers that can handle both formats.
3. Deploy producers using the new format.
4. Observe traffic and compatibility.
5. Remove old behavior only after all consumers have migrated.

This reduces the chance that rollback requires a simultaneous downgrade of many services.

## 18.3 Targeted versus coordinated rollback

Prefer a targeted rollback when one service is independently compatible.

Use a coordinated rollback only when contract dependencies require it. Record the exact service versions and order of operations.

Do not roll back every microservice as a default response to one service failure.

# 19. Database Schema Compatibility

Database changes are one of the most important rollback risks.

## 19.1 Expand-and-contract migration

Prefer a backward-compatible sequence:

```text
EXPAND
  |
  v
Add nullable column / new table / compatible index
  |
  v
Deploy application that supports old and new schema
  |
  v
Backfill and validate data
  |
  v
Switch application reads/writes
  |
  v
Observe
  |
  v
CONTRACT
  |
  v
Remove obsolete schema only after rollback window closes
```

## 19.2 Example

Suppose release `2.5.0` adds a new column `payment_reference`.

A risky migration immediately renames or drops a column used by version `2.4.17`. The old application may fail after rollback.

A safer migration adds the new column while retaining the old column. Both application versions can operate during the transition, and cleanup happens in a later approved release.

## 19.3 Database rollback decision table

| Migration type | Rollback risk | Recommended response |
|---|---|---|
| Add nullable column | Usually lower | Verify old code ignores it |
| Add compatible index | Usually lower | Check lock and performance impact |
| Add new table | Depends on usage | Keep table; assess writes |
| Backfill data | Medium | Validate data and idempotency |
| Rename column | High | Use compatibility layer or forward fix |
| Drop column | Very high | Restore may require backup or forward fix |
| Change data type | High | Verify old application compatibility |
| Destructive transformation | Very high | Do not assume application rollback is safe |

Never run a down migration automatically during an incident. A schema downgrade can destroy valid production data.

# 20. Database Rollback Runbook

Before application rollback:

1. Identify migrations included in the release.
2. Identify which migrations actually ran in production.
3. Check whether writes occurred using the new schema.
4. Check whether old application code can read the current schema.
5. Check whether any data conversion was destructive.
6. Consult the database owner.
7. Choose application rollback, forward fix, or a separately approved database recovery plan.

Useful evidence depends on the database and migration tool. Examples include migration history, schema version, application logs, query errors, lock metrics, and database backups.

Do not print credentials or connection strings into incident logs.

# 21. Configuration Rollback

A release can fail because of configuration rather than application code.

Examples:

- Incorrect database endpoint
- Wrong service URL
- Invalid timeout
- Incorrect connection pool size
- Bad feature setting
- Wrong queue name
- Incorrect TLS configuration

For ConfigMaps:

```bash
kubectl get configmap -n roboshop
kubectl describe configmap payment-config -n roboshop
```

For a GitOps-managed ConfigMap, revert the configuration in Git and reconcile through Argo CD.

Be careful: a ConfigMap mounted as a volume and a ConfigMap consumed as environment variables may update differently. Many applications require a pod restart to load changed environment variables. Follow the application's documented reload behavior.

Do not dump Secret values to terminal output or incident chat. Verify Secret metadata and version identifiers without exposing plaintext.

# 22. Secret and Credential Rollback

Secret rollback is not the same as restoring an old Kubernetes Secret object.

A credential may have been:

- Rotated at the source system
- Revoked
- Expired
- Removed from an allowlist
- Replaced in a database or external service
- Distributed to only some consumers

Before reverting a secret reference, confirm that the underlying credential is still valid and authorized.

Never restore an old password merely because the previous application version used it. Coordinate with the credential owner and use the approved secret-management workflow.

# 23. Feature-Flag Rollback

Feature flags can provide a lower-risk mitigation than a complete application rollback.

Example:

```text
New checkout implementation
        |
        v
Feature flag enabled
        |
        v
Error rate increases
        |
        v
Disable feature flag
        |
        v
Old checkout path serves traffic
```

A flag rollback is safe only if the fallback path is still supported and tested.

Record:

- Flag name and environment
- Previous and current values
- Owner and approver
- Expected customer impact
- Validation signals
- Expiry or follow-up date

Avoid permanent emergency flags with no owner or cleanup plan.

# 24. Terraform and Infrastructure Rollback

Terraform changes infrastructure state and may affect resources shared by many services. It is not an application rollback command.

Examples of risky changes:

- Security group rules
- Route tables
- EKS node groups
- IAM policies
- RDS configuration
- Load balancer resources
- VPC endpoints
- Autoscaling configuration

## 24.1 Inspect before changing infrastructure

```bash
terraform workspace show
terraform state list
terraform plan
```

Run these commands from the correct repository, backend, workspace, and approved environment. Use the project's normal backend and locking process.

## 24.2 Do not blindly apply an old plan

An old saved plan can be stale. A previous Terraform state or configuration snapshot may no longer match the live infrastructure.

Instead:

1. Identify the exact infrastructure change.
2. Inspect the current state and configuration.
3. Review the relevant Git commit.
4. Create a controlled corrective change.
5. Run a fresh plan.
6. Review destructive actions carefully.
7. Obtain required approval.
8. Apply through the approved pipeline.
9. Validate dependent services.

Never use `terraform destroy` as a generic rollback strategy.

# 25. Stateful Workloads and Persistent Data

Rolling back a stateless Deployment is different from restoring stateful systems.

Before changing a StatefulSet or persistent workload, check:

- PersistentVolumeClaim binding
- Storage class and volume topology
- Data format compatibility
- StatefulSet update strategy
- Replication health
- Backup and restore point
- Application-level recovery procedure

Do not delete PVCs or PVs to force a rollback. Deleting a workload does not mean its data is disposable.

# 26. RabbitMQ and Kafka Compatibility

Messages already published cannot be removed simply by rolling back a producer.

## 26.1 RabbitMQ

Check:

- Queue depth
- Unacknowledged message count
- Dead-letter queue
- Consumer availability
- Message schema
- Retry behavior
- Idempotency

A previous consumer must be able to process messages published by the new producer. If not, a forward fix or compatibility adapter may be safer.

Do not purge queues during rollback unless data loss is explicitly approved and the business owner understands the impact.

## 26.2 Kafka

Check:

- Topic and partition compatibility
- Consumer group lag
- Event schema
- Offset behavior
- Duplicate processing risk
- Consumer idempotency
- Dead-letter or retry topics

Do not reset consumer offsets as a generic rollback action. Offset changes can cause message loss or replay and must follow a separately approved recovery procedure.

# 27. Capacity and Scheduling Checks

A known-good release may need more resources or may have different scheduling constraints.

Before rollback, check:

```bash
kubectl get nodes
kubectl describe nodes
kubectl get pods -n roboshop -o wide
kubectl get resourcequota -n roboshop
kubectl get limitrange -n roboshop
kubectl get hpa -n roboshop
```

Check whether the old version can be scheduled and whether cluster autoscaling can provide capacity in time.

Do not scale replicas blindly. More pods can increase pressure on RDS connection limits, Redis, downstream APIs, or message brokers.

# 28. Rollback Verification Strategy

Verify at multiple layers.

## 28.1 Kubernetes

```bash
kubectl get deployment payment -n roboshop
kubectl rollout status deployment/payment -n roboshop
kubectl get pods -n roboshop -l app=payment
kubectl get endpoints payment -n roboshop
```

Check desired and available replicas, readiness, restarts, image reference, and service endpoints.

## 28.2 Argo CD

```bash
argocd app get roboshop-prod
argocd app diff roboshop-prod
```

Confirm the application is Synced to the intended Git revision and Healthy. Investigate unexpected drift rather than blindly syncing.

## 28.3 Application logs

```bash
kubectl logs deployment/payment -n roboshop \
  --since=15m --all-containers=true
```

Search ELK for the relevant service, release, request ID, and error pattern. Confirm that the new release's failure signature has stopped and that no new error pattern has appeared.

## 28.4 Prometheus and Grafana

Use the existing service instrumentation and dashboard definitions. Example signals:

- Request rate
- HTTP 5xx rate
- Request latency, including p95 and p99
- Pod restarts
- CPU and memory utilization
- Readiness and availability
- Database connection usage
- Queue depth and consumer lag

Illustrative PromQL only; metric names and labels depend on the application's instrumentation:

```promql
sum(rate(http_requests_total{
  service="payment",
  status=~"5.."
}[5m]))
/
clamp_min(
  sum(rate(http_requests_total{
    service="payment"
  }[5m])),
  1
)
```

Do not copy this query unchanged if your metric labels differ.

## 28.5 Business validation

A successful Kubernetes rollout does not prove checkout works.

Validate the customer journey:

```text
Login
  |
  v
Browse catalogue
  |
  v
Add item to cart
  |
  v
Start checkout
  |
  v
Authorize payment
  |
  v
Reserve inventory
  |
  v
Create order
  |
  v
Confirm order
  |
  v
Send notification
```

Use approved test accounts and test payment mechanisms. Do not create real charges or modify real customer orders during a simulation.

# 29. Rollback Stop Conditions

Stop and escalate if:

- The wrong AWS account, cluster, namespace, or application is selected.
- The target version cannot be verified.
- The old version fails compatibility checks.
- Database or message data may be corrupted.
- The rollback increases error rate or customer impact.
- Pods cannot become Ready within the agreed timeout.
- Argo CD repeatedly reverts the emergency action.
- The rollback causes dependency saturation.
- The validation workflow cannot establish business recovery.

A stop condition is not a failure of the engineer. It is a guardrail against turning a contained incident into a larger one.

# 30. Complete Production Rollback Runbook

## Step 1 — Declare and coordinate

Record the incident ID, severity, affected service, customer impact, incident commander, service owner, and change approver.

## Step 2 — Confirm the release

Identify the current image digest, Git commit, Helm revision, Argo CD revision, and deployment timestamp.

## Step 3 — Identify the known-good target

Use release records and monitoring evidence. Confirm the image and chart are still available.

## Step 4 — Check compatibility

Review database migrations, API contracts, message formats, configuration, secrets, and external dependencies.

## Step 5 — Choose the rollback mechanism

| Deployment model | Preferred permanent action |
|---|---|
| Argo CD + Helm | Revert Git desired state and let Argo CD reconcile |
| Argo CD + Kustomize | Revert the production overlay in Git |
| Direct Helm-managed release | Approved Helm rollback |
| Direct Kubernetes Deployment | Approved rollout undo or reviewed image update |
| Terraform-managed infrastructure | Reviewed corrective Terraform change |

## Step 6 — Obtain approval

Follow the incident emergency-change process. Record who approved the action and why.

## Step 7 — Execute

Use the selected mechanism. Do not combine multiple rollback mechanisms simultaneously unless the runbook explicitly requires it.

## Step 8 — Verify technical health

Check rollout, replicas, readiness, endpoints, image digest, logs, metrics, and GitOps state.

## Step 9 — Verify customer workflow

Run the approved end-to-end checkout test and inspect business-level success signals.

## Step 10 — Communicate

Report what changed, what version is active, what is healthy, what remains degraded, and what investigation continues.

## Step 11 — Preserve evidence

Keep the bad release, logs, deployment history, Git commit, image digest, and timeline for RCA.

## Step 12 — Close rollback activity

Document the final state and hand unresolved root-cause work to Chapter 211 — RCA Postmortem. Service recovery and incident closure are separate decisions.

# 31. Production Rollback Scenario — Payment Service

## 31.1 Incident

A production deployment promoted payment `2.5.0`. Within minutes:

- Payment API 5xx rate increased.
- Checkout success rate decreased.
- Payment pods remained Ready.
- CPU and memory were within expected limits.
- Logs showed a new payment-provider timeout pattern.

This is an example scenario, not a claim about a real incident.

## 31.2 Initial investigation

```bash
kubectl config current-context
kubectl get deployment payment -n roboshop -o wide
kubectl get pods -n roboshop -l app=payment
kubectl logs deployment/payment -n roboshop --since=20m
kubectl rollout history deployment/payment -n roboshop
```

Compare the incident start time with the deployment time. Confirm whether the provider issue is new, whether the provider itself is degraded, and whether the previous release used a compatible integration.

## 31.3 Decision

The team confirms:

- Release `2.5.0` introduced a changed timeout/retry behavior.
- Release `2.4.17` was healthy.
- No incompatible database migration was run.
- The previous release supports the current API and message formats.
- The previous image digest is available.
- The rollback is approved.

The team selects the GitOps rollback rather than a direct cluster-only change.

## 31.4 Execute GitOps rollback

```bash
git checkout main
git pull --ff-only origin main
git checkout -b rollback/payment-2.5.0
git log --oneline -- environments/prod/payment/values.yaml
```

Identify and inspect the release commit:

```bash
git show <bad-commit>
```

Revert the isolated release commit:

```bash
git revert <bad-commit>
git diff
git push -u origin rollback/payment-2.5.0
```

Open the emergency pull request, obtain approval, and merge according to policy.

## 31.5 Reconcile

```bash
argocd app get roboshop-prod
argocd app diff roboshop-prod
argocd app sync roboshop-prod
argocd app wait roboshop-prod --health --sync --timeout 300
```

Only run the sync command when manual sync is required and approved.

## 31.6 Verify

```bash
kubectl rollout status deployment/payment -n roboshop
kubectl get pods -n roboshop -l app=payment
kubectl get deployment payment -n roboshop \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Then check metrics, ELK logs, payment-provider connectivity, and the approved checkout test.

## 31.7 Result criteria

Rollback is considered technically successful when the intended release is active, pods are Ready, endpoints are available, Argo CD is Synced and Healthy, and error/latency signals return to the agreed range.

The customer workflow must also succeed. If payment errors remain, do not declare recovery solely because the old image is running. Continue incident investigation.

# 32. Rollback Failure Scenarios

## Scenario A — Argo CD reverts a manual rollback

**Symptom:** An engineer runs `kubectl rollout undo`, but the new image returns.

**Cause:** Argo CD reconciles the cluster to the Git desired state.

**Response:** Stop competing changes. Revert the production GitOps configuration through the approved workflow and observe Argo CD reconciliation.

## Scenario B — Old pods cannot start

**Symptom:** Previous release pods enter CrashLoopBackOff.

**Possible causes:**

- Incompatible schema
- Missing environment variable
- Revoked credential
- Incompatible ConfigMap
- Dependency contract change

**Response:** Capture logs and events, stop repeated blind restarts, and assess whether a forward fix is safer.

## Scenario C — Old image is unavailable

**Symptom:** ImagePullBackOff after selecting the previous tag.

**Possible causes:**

- Image was deleted
- Wrong ECR repository or region
- IAM pull permission failure
- Incorrect tag or digest

**Response:** Verify the exact image reference and ECR metadata. Do not substitute `latest` or an unverified image.

## Scenario D — Rollback causes database errors

**Symptom:** Old application version starts but database queries fail.

**Cause:** Current schema or data is incompatible with old code.

**Response:** Stop further rollback attempts. Engage the database and service owners. Prefer a compatible forward fix or an approved data recovery plan.

## Scenario E — Pods are Pending

**Symptom:** Old release cannot schedule.

**Possible causes:** Insufficient CPU/memory, node affinity, taints, quotas, or unavailable node capacity.

**Response:** Inspect scheduling events and capacity. Do not blindly increase replicas or remove scheduling constraints.

## Scenario F — Error rate remains high

**Symptom:** Rollback completes but customer errors continue.

**Possible causes:** External provider outage, shared dependency failure, bad configuration, persistent data issue, or a second independent fault.

**Response:** Treat rollback as one mitigation, not proof of root cause. Continue the incident investigation.

# 33. Rollback Anti-Patterns

Avoid:

- Rolling back without confirming customer impact.
- Selecting a version by memory rather than release evidence.
- Using mutable image tags.
- Running `kubectl rollout undo` against a GitOps-managed workload and ignoring reconciliation.
- Running `helm rollback` while Argo CD owns the release without a coordinated plan.
- Reverting database schema automatically.
- Restoring revoked credentials.
- Purging RabbitMQ queues or resetting Kafka offsets casually.
- Rolling back every microservice for a single-service failure.
- Applying an old Terraform plan without a fresh review.
- Deleting PVCs to force recovery.
- Force-pushing shared production Git history.
- Declaring recovery based only on pod readiness.
- Removing logs or failed images before RCA.
- Leaving temporary flags and emergency configuration changes undocumented.

# 34. Rollback Evidence and Audit Record

Record at minimum:

| Field | Example |
|---|---|
| Incident ID | INC-<id> |
| Environment | production |
| Service | payment |
| Current release | 2.5.0 |
| Target release | 2.4.17 |
| Current image digest | Verified digest |
| Target image digest | Verified digest |
| Bad Git commit | Commit SHA |
| Rollback Git commit | Revert SHA |
| Argo CD application | roboshop-prod |
| Approval | Approver / ticket reference |
| Start and end time | Incident timestamps |
| Validation | Technical and business checks |
| Result | Successful / partial / failed |
| Follow-up | RCA and corrective action owner |

Do not include passwords, access tokens, private keys, or customer payment data in the record.

# 35. Production Rollback Checklist

## Before rollback

- [ ] Correct AWS account and region confirmed.
- [ ] Correct EKS cluster and namespace confirmed.
- [ ] Customer impact and affected workflow recorded.
- [ ] Incident commander and service owner identified.
- [ ] Current release and image digest recorded.
- [ ] Known-good release verified.
- [ ] Previous image exists in ECR.
- [ ] Database compatibility checked.
- [ ] API and message compatibility checked.
- [ ] Configuration and secret changes reviewed.
- [ ] Capacity checked.
- [ ] Rollback method selected.
- [ ] Approval recorded.
- [ ] Validation and stop conditions agreed.

## During rollback

- [ ] Only the approved service/environment changed.
- [ ] GitOps ownership respected.
- [ ] No competing rollback mechanism running.
- [ ] Rollout progress monitored.
- [ ] Events and logs captured.
- [ ] Incident timeline updated.

## After rollback

- [ ] Expected image/version is active.
- [ ] Deployment rollout completed.
- [ ] Pods are Ready.
- [ ] Service endpoints are available.
- [ ] Argo CD is Synced and Healthy where applicable.
- [ ] Error rate and latency checked.
- [ ] Database and messaging health checked.
- [ ] Business workflow tested.
- [ ] Customer/stakeholder status communicated.
- [ ] Failed release evidence preserved.
- [ ] Follow-up RCA assigned.
- [ ] Temporary changes documented and given an owner.

# 36. Rollback Readiness — What Must Exist Before Production?

A mature platform should maintain:

- Immutable container images and digests
- Retained known-good release artifacts
- Version-controlled deployment manifests
- Protected GitOps branches
- Required production approvals
- Argo CD application history
- Helm release history
- Kubernetes rollout history
- Database migration compatibility policy
- API and event schema compatibility policy
- Tested feature-flag fallback
- Backup and restore procedures
- Monitoring dashboards and actionable alerts
- A rollback runbook per critical service
- Regular rollback game days

A rollback plan that has never been tested is an assumption, not a reliable production capability.

# 37. Rollback Game Day

Run this exercise in a non-production environment first.

**Scenario:** Payment `2.5.0` increases checkout errors. The team must restore the previous compatible release.

Measure:

- Time to detect the regression
- Time to identify the release
- Time to approve the rollback
- Time to complete reconciliation
- Time to restore successful checkout
- Error rate during rollback
- Any unexpected data or dependency issue

Game-day sequence:

```text
Inject approved test regression
          |
          v
Detect and declare incident
          |
          v
Identify bad release
          |
          v
Find known-good version
          |
          v
Check schema/API/message compatibility
          |
          v
Approve rollback
          |
          v
Revert GitOps desired state
          |
          v
Argo CD reconciles EKS
          |
          v
Validate service and checkout
          |
          v
Record timeline and lessons
```

Do not run failure injection or rollback drills against production without explicit authorization and an approved change window.

# 38. Expected Engineer Knowledge

After completing this chapter, you should be able to explain:

1. What is a production rollback?
2. What is the difference between rollback and roll forward?
3. How do you identify a known-good release?
4. Why are immutable image digests important?
5. How do you inspect Kubernetes rollout history?
6. How do you perform a controlled Deployment rollback?
7. How do you inspect and roll back a Helm release?
8. Why can a direct Kubernetes rollback be temporary under Argo CD?
9. How do you revert a GitOps change safely?
10. Why is `git revert` preferred over rewriting shared production history?
11. How do you verify an Argo CD rollback?
12. What is the expand-and-contract database migration strategy?
13. Why can a database migration make rollback unsafe?
14. How do you handle secret rotation during rollback?
15. When is a feature-flag rollback preferable?
16. Why is Terraform not an application rollback mechanism?
17. How do you handle RabbitMQ and Kafka compatibility?
18. How do you coordinate rollback across dependent microservices?
19. How do you verify customer recovery?
20. What evidence should be preserved for RCA?

# 39. Final Outcome

By completing this chapter, you should be able to perform a controlled rollback using the appropriate deployment mechanism and validate both platform health and customer behavior.

```text
PRODUCTION REGRESSION
        |
        v
CONFIRM CUSTOMER IMPACT
        |
        v
IDENTIFY THE CHANGE
        |
        v
SELECT KNOWN-GOOD RELEASE
        |
        v
CHECK DATA / API / MESSAGE COMPATIBILITY
        |
        v
OBTAIN APPROVAL
        |
        v
EXECUTE CONTROLLED ROLLBACK
        |
        v
VERIFY KUBERNETES AND GITOPS
        |
        v
VERIFY BUSINESS WORKFLOW
        |
        v
COMMUNICATE AND PRESERVE EVIDENCE
        |
        v
HAND OFF TO RCA
```

**Key production principle:**

A rollback is safe only when the target version is known, compatibility is understood, the change is approved, and recovery is verified. In a GitOps-managed EKS platform, restore the intended state in Git and let Argo CD reconcile it; do not leave the cluster and Git describing different production states.

**End of Chapter 209 — Production Rollback.**
