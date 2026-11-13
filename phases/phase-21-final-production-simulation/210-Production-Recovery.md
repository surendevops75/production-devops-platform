# CHAPTER 210 — PRODUCTION RECOVERY

**Phase 21 — Final Simulation**

**File:** `210-Production-Recovery.md`

---

# 1. Objective

This chapter explains how to recover a production microservices platform after a serious service disruption, failed deployment, infrastructure issue, data problem, or dependency outage.

Production recovery is broader than restarting pods or rolling back an application. It is the controlled process of restoring the platform, its dependencies, and the affected customer workflows to an agreed operational state.

The objective is to build a repeatable recovery approach for a 20+ microservices application running on AWS EKS.

This chapter covers:

- Incident stabilization and recovery objectives
- Recovery roles and communication
- Recovery priorities and dependency mapping
- Kubernetes workload recovery
- EKS node and cluster recovery
- Networking, DNS, ingress, and load balancer recovery
- RDS, Redis, RabbitMQ, and Kafka recovery considerations
- Storage and persistent workload recovery
- Application and GitOps reconciliation
- Backup, restore, and disaster recovery
- Data integrity and reconciliation
- Observability and recovery validation
- Recovery runbooks and simulations
- Recovery metrics and evidence

**Important:** The commands and configurations are examples. Replace sample names, namespaces, regions, account IDs, endpoints, and resource identifiers with verified environment values. Recovery actions can cause data loss or service interruption. Follow the organization's incident, backup, access, and change-approval procedures.

# 2. What Is Production Recovery?

Production recovery is the process of restoring a production service or platform after an incident.

A service is not fully recovered merely because its pods are running. Recovery must consider:

1. Infrastructure availability
2. Kubernetes control and worker plane health
3. Networking and DNS
4. Application readiness
5. Database and messaging dependencies
6. Data integrity
7. Customer-facing functionality
8. Monitoring and alerting
9. Operational ownership and communication

Example:

```text
Production incident
       |
       v
Stabilize customer impact
       |
       v
Identify affected components
       |
       v
Restore critical dependencies
       |
       v
Recover Kubernetes workloads
       |
       v
Validate application services
       |
       v
Validate end-to-end workflow
       |
       v
Monitor for recurring failure
       |
       v
Confirm recovery and hand over to RCA
```

Recovery is complete only when the agreed technical and business acceptance criteria are met.

# 3. Example Production Platform

The example environment contains:

| Layer | Components |
|---|---|
| Cloud | AWS |
| Kubernetes | Amazon EKS |
| Container registry | Amazon ECR |
| Networking | VPC, subnets, security groups, ALB |
| DNS | Route 53 and CoreDNS |
| Deployment | Helm and Argo CD |
| CI/CD | Jenkins, GitHub Actions, GitLab CI/CD |
| Infrastructure as Code | Terraform |
| Databases | Amazon RDS |
| Cache | Redis |
| Messaging | RabbitMQ and Kafka |
| Monitoring | Prometheus and Grafana |
| Logging | ELK Stack |
| Application | 20+ microservices |

Example services:

- Frontend
- User
- Catalogue
- Cart
- Order
- Payment
- Inventory
- Shipping
- Notification

Actual service names and dependencies must come from the platform's service catalogue and architecture repository.

# 4. Recovery Versus Rollback Versus Disaster Recovery

These terms describe different activities.

| Activity | Meaning | Example |
|---|---|---|
| Rollback | Return a changed component to a known-good version | Restore payment `2.4.17` |
| Recovery | Restore an affected service or workflow | Restore checkout functionality |
| Disaster recovery | Restore operations after a major site, region, or platform failure | Recover into a secondary region |
| Data restore | Recover data from a backup or replica | Restore an RDS database |
| Failover | Switch workload to a healthy alternative | Promote a database standby |
| Reconciliation | Bring actual state in line with desired state | Argo CD syncs Git-defined manifests |

Chapter 209 covered production rollback. This chapter focuses on restoring the overall service and customer workflow, whether or not rollback is the chosen mitigation.

# 5. Recovery Objectives: RTO and RPO

Recovery planning needs measurable objectives.

## 5.1 Recovery Time Objective (RTO)

RTO is the target maximum duration for restoring a service after disruption.

Example:

```text
Incident begins: 10:00
Service restored: 10:45
Recovery duration: 45 minutes
Target RTO: 60 minutes
```

The example meets the 60-minute target. This does not mean all customer or data impact has been resolved.

## 5.2 Recovery Point Objective (RPO)

RPO is the maximum acceptable data loss measured in time.

Example:

```text
Last recoverable consistent data point: 09:50
Incident begins: 10:00
Potential data loss window: 10 minutes
Target RPO: 15 minutes
```

Actual RPO depends on backup frequency, replication, consistency, and restore capability. A backup timestamp alone does not prove that the restored data is consistent or usable.

## 5.3 Define objectives per service

| Service class | Example RTO | Example RPO | Notes |
|---|---:|---:|---|
| Critical checkout | 60 minutes | 5 minutes | Business-approved target required |
| Catalogue read path | 2 hours | 30 minutes | Cache may be rebuilt |
| Notifications | 4 hours | 30 minutes | Queue retention and replay matter |
| Analytics | 24 hours | 24 hours | Depends on business needs |

These values are illustrative only. Business, engineering, security, and compliance owners must agree the actual targets.

# 6. Recovery Roles and Responsibilities

| Role | Responsibility |
|---|---|
| Incident commander | Owns incident coordination and priorities |
| Platform / DevOps engineer | Restores EKS, networking, deployment, and infrastructure |
| Service owner | Validates application behavior and dependencies |
| Database owner | Owns database failover, restore, and data validation |
| Security owner | Reviews access, credentials, and security implications |
| QA / product owner | Validates critical customer journeys |
| Communications owner | Sends status updates to stakeholders |

For smaller teams, one person may hold multiple roles. Explicitly assign decision ownership during a major incident.

# 7. Recovery Priority and Dependency Mapping

Do not recover services in arbitrary order.

A typical dependency chain might be:

```text
Route 53
   |
   v
ALB / Ingress
   |
   v
Frontend
   |
   v
Order API
   |
   +------> User
   |
   +------> Cart
   |
   +------> Payment
   |           |
   |           v
   |        Payment provider
   |
   +------> Inventory
   |
   v
Database / Messaging
```

This is an illustrative dependency map. Confirm the actual call graph and failure dependencies from application configuration, service documentation, and runtime evidence.

Recovery order should normally restore foundational dependencies before dependent workloads, while avoiding unnecessary changes to healthy components.

# 8. Recovery Severity and Scope

Classify the incident before choosing a recovery path.

| Scope | Example | Recovery approach |
|---|---|---|
| Single pod | Container crash | Inspect and recover workload |
| Single service | Payment unavailable | Service-specific recovery |
| Multiple services | Shared Redis unavailable | Restore shared dependency |
| Cluster | Worker nodes unavailable | EKS and node recovery |
| Network | VPC route or DNS failure | Network recovery |
| Region | Major AWS region disruption | Disaster recovery plan |
| Data | Corruption or accidental deletion | Data recovery and reconciliation |

The scope may change as investigation proceeds. Update the incident record when new evidence changes the recovery plan.

# 9. First Response: Stabilize Before Recovering

The first task is to limit additional damage.

Actions may include:

- Stop an unsafe deployment pipeline.
- Pause automated promotion to affected environments.
- Disable a problematic feature flag.
- Reduce incoming traffic using an approved mechanism.
- Protect databases from excessive connection attempts.
- Preserve logs, events, and deployment evidence.
- Prevent repeated destructive recovery attempts.
- Communicate the incident and current customer impact.

Do not disable health checks, security controls, or network policies simply to make a dashboard appear healthy.

# 10. Initial Evidence Collection

## 10.1 Confirm the AWS identity

```bash
aws sts get-caller-identity
aws configure get region
```

Verify the account and region using the approved environment configuration. A local AWS CLI default region is not proof that the correct production region is selected.

## 10.2 Confirm Kubernetes context

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

## 10.3 Capture workload status

```bash
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A
kubectl get services -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Large clusters may produce extensive output. Narrow commands to the affected namespace after identifying the scope.

## 10.4 Capture application evidence

```bash
kubectl logs deployment/<deployment-name> \
  -n <namespace> --since=30m --all-containers=true
```

For a restarted container:

```bash
kubectl logs <pod-name> -n <namespace> \
  --previous --all-containers=true
```

Redact credentials, tokens, personal information, and customer data before sharing logs.

# 11. Kubernetes Workload Recovery

## 11.1 Pod in CrashLoopBackOff

Inspect:

```bash
kubectl get pods -n roboshop
kubectl describe pod <pod-name> -n roboshop
kubectl logs <pod-name> -n roboshop --previous
kubectl get events -n roboshop --sort-by=.metadata.creationTimestamp
```

Check:

- Application startup errors
- Missing configuration
- Secret references
- Database connectivity
- Container command and arguments
- Liveness and readiness probes
- Resource limits
- Dependency availability

Do not repeatedly delete pods without understanding why the replacement will behave differently.

## 11.2 Deployment has unavailable replicas

```bash
kubectl get deployment <deployment-name> -n roboshop
kubectl describe deployment <deployment-name> -n roboshop
kubectl get rs -n roboshop
kubectl rollout status deployment/<deployment-name> \
  -n roboshop --timeout=5m
```

Check desired, updated, ready, and available replica counts.

## 11.3 ImagePullBackOff

```bash
kubectl describe pod <pod-name> -n roboshop
kubectl get serviceaccount -n roboshop
```

Check:

- Image repository and tag or digest
- ECR repository and region
- Node or workload pull permissions
- Network access to ECR
- Image existence
- Registry authentication configuration

Do not replace a verified image with `latest` as a recovery shortcut.

## 11.4 Pending pods

```bash
kubectl describe pod <pod-name> -n roboshop
kubectl get nodes
kubectl describe nodes
kubectl get resourcequota -n roboshop
kubectl get limitrange -n roboshop
```

Investigate insufficient CPU/memory, taints, affinity, topology constraints, PVC binding, quotas, and node capacity.

# 12. EKS Worker Node Recovery

A worker node may become NotReady because of:

- EC2 instance or underlying host issue
- Kubelet failure
- Container runtime issue
- Disk or inode pressure
- Memory pressure
- CNI networking issue
- Node IAM or bootstrap problem
- Subnet or route issue

Inspect:

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -A -o wide
```

Check node conditions, events, pod placement, and capacity.

## 12.1 Cordon a node

If the node is unhealthy and the incident plan calls for preventing new scheduling:

```bash
kubectl cordon <node-name>
```

Cordon prevents new pods from being scheduled on the node. It does not evict existing pods.

## 12.2 Drain a node

Drain only after checking PodDisruptionBudgets, local storage, StatefulSets, workload redundancy, and maintenance policy:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --timeout=10m
```

Do not add `--delete-emptydir-data` or force flags without understanding the data and disruption implications.

## 12.3 Recover node capacity

Depending on the managed node group or autoscaling configuration, recovery may require:

- Replacing an unhealthy EC2 instance
- Restoring desired node-group capacity
- Correcting launch template or bootstrap configuration
- Fixing subnet IP exhaustion
- Resolving node IAM permissions
- Restoring CNI functionality

Use the approved EKS and infrastructure automation process. Avoid manually changing managed node group resources in a way that conflicts with Terraform.

# 13. EKS Control Plane and API Access Recovery

If Kubernetes API access is unavailable:

1. Confirm AWS account and region.
2. Check EKS cluster status using AWS APIs.
3. Check IAM authentication and authorization.
4. Check endpoint access configuration and network reachability.
5. Check VPN, bastion, or approved administrative path.
6. Check whether the issue affects one operator or all cluster clients.
7. Escalate AWS service-side issues through the organization's support process.

Example:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --region <region>
```

Do not recreate the cluster because a local `kubectl` command fails. First distinguish client access, IAM, network, and control-plane problems.

# 14. Kubernetes DNS Recovery

CoreDNS provides cluster DNS for service discovery.

Inspect:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get deployment coredns -n kube-system
kubectl logs deployment/coredns -n kube-system --since=20m
kubectl get services -n kube-system
```

Test DNS from an approved diagnostic pod:

```bash
nslookup kubernetes.default.svc.cluster.local
```

Then test a real service name:

```bash
nslookup payment.roboshop.svc.cluster.local
```

Check CoreDNS replicas, readiness, configuration, node connectivity, NetworkPolicy, and upstream resolver behavior.

Do not restart CoreDNS repeatedly if the underlying problem is node networking or upstream DNS.

# 15. CNI and Pod Networking Recovery

Symptoms may include:

- Pods stuck in ContainerCreating
- Sandbox creation failures
- Pod IP allocation failures
- Cross-node connectivity failures
- Node subnet IP exhaustion

Inspect:

```bash
kubectl get pods -n kube-system -o wide
kubectl get daemonsets -n kube-system
kubectl describe pod <affected-pod> -n <namespace>
kubectl describe node <node-name>
```

For the Amazon VPC CNI, inspect the `aws-node` DaemonSet and its logs using the actual pod name:

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs <aws-node-pod> -n kube-system --since=20m
```

Check subnet free IP capacity, ENI/IP limits, CNI configuration, security groups, and node health. Make changes through the approved EKS add-on or infrastructure process.

# 16. Ingress, ALB, and External Traffic Recovery

A service may be healthy inside the cluster while external users receive 502, 503, or 504 responses.

Check:

```bash
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n roboshop
kubectl get services -n roboshop
kubectl get endpoints -n roboshop
```

Inspect the AWS load balancer and target group health using approved AWS CLI permissions and the actual resource identifiers.

Validate:

- Listener and target group configuration
- Target health
- Health-check path and port
- Security group rules
- Subnet tags and routing
- TLS certificate status
- Ingress controller or AWS Load Balancer Controller health
- Service selectors and endpoints

Do not open security groups to `0.0.0.0/0` as a generic troubleshooting action.

# 17. Route 53 and External DNS Recovery

If the application is unavailable externally:

- Confirm the expected DNS record.
- Check record type and target.
- Check TTL and recent changes.
- Verify the load balancer hostname.
- Check certificate and hostname alignment.
- Test from more than one resolver or network where possible.

Example:

```bash
dig <application-domain>
nslookup <application-domain>
```

A DNS change may take time to propagate according to resolver caching and TTL. Do not repeatedly modify records without confirming the current authoritative configuration.

# 18. RDS Database Recovery

Database recovery must be owned or approved by the database team.

First identify whether the problem is:

- Database instance unavailable
- Connection limit reached
- Network or security group issue
- Storage pressure
- Long-running query or lock contention
- Authentication failure
- Replication or failover event
- Data corruption or accidental change

Application-side checks:

```bash
kubectl logs deployment/order -n roboshop --since=20m
kubectl logs deployment/payment -n roboshop --since=20m
```

Check the database endpoint, port, security group path, secret reference, connection pool behavior, and AWS RDS status.

## 18.1 RDS failover

For Multi-AZ deployments, AWS may perform failover according to the service configuration. Applications must tolerate connection interruption and reconnect safely.

Do not assume that a failover means the database endpoint or every client connection is immediately healthy.

Validate:

- RDS instance and cluster status
- Writer/reader roles where applicable
- Application reconnection
- Connection pool recovery
- Replication health
- Transaction errors
- Data consistency

## 18.2 RDS restore

A restore may create a new database instance or cluster rather than replacing the original in place. The recovery plan must account for endpoint changes, credentials, security groups, parameter groups, and application cutover.

Before restore:

1. Identify the approved recovery point.
2. Confirm RPO and expected data loss.
3. Confirm the restore target and isolation plan.
4. Obtain database and incident approval.
5. Restore into a controlled target.
6. Validate schema and data.
7. Test application connectivity.
8. Plan endpoint cutover.
9. Preserve the original source for investigation where possible.

Never overwrite or delete the original production database as an improvised first step.

# 19. Redis Recovery

Redis may be used for cache, sessions, locks, or other state. The recovery approach depends on the data's purpose.

Check:

- Redis availability
- Memory usage and eviction
- Connection count
- Network path
- Authentication and TLS
- Replication and failover state
- Persistence configuration if used
- Application cache behavior

If Redis is only a disposable cache, rebuilding it may be acceptable after confirming the application can tolerate a cold cache.

If Redis stores sessions, locks, or business-critical state, do not treat it as disposable. Assess user impact, consistency, and recovery options first.

A cold cache can create a sudden load spike on RDS and downstream services. Warm-up and traffic controls may be required.

# 20. RabbitMQ Recovery

Check:

- Broker and node availability
- Queue depth
- Unacknowledged messages
- Consumer count
- Disk and memory alarms
- Connection and channel errors
- Cluster membership and quorum state
- Dead-letter queues

Recovery priorities:

1. Restore broker availability.
2. Confirm queue and message durability settings.
3. Restore consumers gradually.
4. Observe queue drain rate.
5. Monitor downstream database and API capacity.
6. Check duplicate processing and idempotency.
7. Verify dead-letter and retry handling.

Do not purge queues to make queue-depth graphs look healthy. Messages may represent valid orders, payments, or inventory work.

# 21. Kafka Recovery

Check:

- Broker availability
- Controller and cluster health
- Partition availability
- Under-replicated partitions
- Consumer group lag
- Producer errors
- Disk capacity
- Network connectivity
- Schema compatibility

Restore consumers gradually and observe lag. A large backlog can overload downstream systems if every consumer resumes at full speed.

Do not reset offsets without a documented replay or skip procedure and business approval. Replaying messages may duplicate side effects unless consumers are idempotent.

# 22. Persistent Storage Recovery

For PVC-backed workloads:

```bash
kubectl get pvc -A
kubectl get pv
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get storageclass
```

Investigate:

- PVC binding state
- StorageClass and provisioner
- Availability Zone topology
- CSI controller and node plugin health
- Volume attachment errors
- Access mode
- Filesystem or application-level consistency

Do not delete a PVC or PV as a generic way to recover a pod. Confirm retention policy and backup status before any storage operation.

# 23. Argo CD and GitOps Recovery

Argo CD continuously compares live cluster state with desired state in Git.

Inspect:

```bash
argocd app list
argocd app get <application-name>
argocd app diff <application-name>
```

If an application is OutOfSync or Degraded:

1. Identify the intended Git revision.
2. Inspect the diff.
3. Check whether the change is expected.
4. Check cluster and dependency health.
5. Correct the Git source if it is wrong.
6. Sync only through the approved process.
7. Wait for health and sync.
8. Verify actual workload behavior.

Example, only after approval:

```bash
argocd app sync <application-name>
argocd app wait <application-name> \
  --health --sync --timeout 300
```

Do not repeatedly sync an application if the desired state is incorrect. Correct the source first.

# 24. CI/CD Pipeline Recovery

A failed pipeline can block a recovery release.

Inspect:

- Jenkins controller and agent availability
- GitHub Actions workflow status
- GitLab runner availability
- Repository access
- Artifact registry access
- Build cache
- Signing and scanning stages
- Deployment credentials
- Approval gates

Do not bypass security scans, signing, or production approvals just to accelerate recovery unless a formally approved emergency process defines the exception and its compensating controls.

If a known-good artifact already exists, use the verified immutable artifact rather than rebuilding an unverified image during the incident.

# 25. Terraform and Infrastructure Recovery

Infrastructure recovery must account for Terraform state and actual cloud resources.

Before changing infrastructure:

```bash
terraform workspace show
terraform state list
terraform plan
```

Use the correct backend, workspace, variables, and locking process.

Recovery steps:

1. Confirm account, region, workspace, and backend.
2. Inspect the failed change and state.
3. Determine whether resources were partially created or modified.
4. Review the current plan.
5. Identify destructive or replacement actions.
6. Prepare a minimal corrective change.
7. Obtain approval.
8. Apply through the approved pipeline.
9. Verify the actual AWS resources and dependent workloads.

Do not delete state files, remove resources from state, or run `terraform destroy` as a generic recovery action.

# 26. Backup and Restore Strategy

A production recovery plan should identify what is backed up, how often, where it is stored, who can restore it, and how restore success is verified.

| Component | Possible protection method | Validation |
|---|---|---|
| RDS | Automated backups and snapshots | Restore test and application query |
| EBS-backed data | Snapshots and application-aware backup | Mount and consistency test |
| S3 | Versioning, replication, lifecycle controls | Object restore test |
| Kubernetes manifests | Git repository | Re-deploy into test environment |
| Helm charts | Versioned chart repository or OCI registry | Render and install test |
| Container images | ECR retention and replication policy | Pull by digest |
| Secrets | Approved secret manager backup/recovery process | Controlled access validation |
| Messaging | Broker replication and product-specific backup | Queue/topic recovery test |

A snapshot is not a complete recovery strategy until restoration has been tested.

# 27. S3 Recovery Considerations

If application artifacts or data are stored in S3, check:

- Bucket and region
- Versioning status
- Object version history
- Lifecycle policy
- Replication status
- Encryption and KMS permissions
- Bucket policy
- Access point configuration

For versioned objects, an authorized operator can inspect versions:

```bash
aws s3api list-object-versions \
  --bucket <bucket-name> \
  --prefix <object-key>
```

Restoring an object version must follow the data owner's approval and the platform's retention policy. Verify encryption and access permissions after restore.

# 28. Disaster Recovery and Regional Recovery

Regional disaster recovery is different from a normal pod or service recovery.

A regional plan may require:

- Secondary-region VPC and subnets
- EKS cluster or cluster provisioning automation
- ECR image replication
- GitOps bootstrap
- IAM roles and policies
- Secrets and key availability
- RDS cross-region backup or replica strategy
- S3 replication
- DNS failover
- Certificates
- Messaging recovery
- External provider allowlists
- Observability and alert routing

## 28.1 Recovery sequence

```text
Declare regional disaster
          |
          v
Confirm recovery region and authority
          |
          v
Provision / validate network
          |
          v
Restore identity and security dependencies
          |
          v
Restore database and persistent data
          |
          v
Bootstrap EKS and platform add-ons
          |
          v
Restore images and GitOps
          |
          v
Deploy critical services
          |
          v
Validate dependencies and business workflow
          |
          v
Cut traffic over through approved DNS process
          |
          v
Monitor and communicate recovery
```

The actual sequence may differ depending on whether the secondary environment is warm, pilot-light, or cold.

## 28.2 DR models

| Model | Description | Trade-off |
|---|---|---|
| Backup and restore | Rebuild after failure | Lower ongoing cost, longer RTO |
| Pilot light | Core data and some infrastructure ready | Moderate cost and recovery time |
| Warm standby | Reduced-capacity environment running | Faster recovery, higher cost |
| Multi-site active/active | Multiple sites serve traffic | Highest complexity and operational cost |

Choose a model based on business-approved RTO, RPO, cost, complexity, and data residency requirements.

# 29. Data Integrity and Reconciliation

A service may become available while data remains inconsistent.

Examples:

- Payment authorized but order not created
- Order created but inventory not reserved
- Inventory reserved but payment failed
- Notification not delivered
- Duplicate event processing
- Delayed consumer processing

Create reconciliation procedures for critical business transactions.

A reconciliation process should:

1. Identify the authoritative source for each business state.
2. Find incomplete or inconsistent transactions.
3. Avoid duplicate side effects.
4. Use idempotency keys or transaction identifiers.
5. Apply approved compensating actions.
6. Record every correction.
7. Verify final business state.

Do not manually edit production rows without an approved data-repair procedure, audit trail, and validation.

# 30. Application Recovery Order

A typical recovery order might be:

1. Cloud networking and identity dependencies
2. EKS control-plane access and worker capacity
3. DNS and cluster networking
4. Persistent storage and databases
5. Redis and messaging systems
6. Internal platform services
7. Core business microservices
8. Frontend and external traffic
9. Background consumers and scheduled work
10. Non-critical analytics and reporting

This is not a universal dependency order. Use the actual application dependency map. For example, some services may need to start before a database migration or a message consumer can be safely enabled.

# 31. Gradual Traffic Restoration

After a dependency or application recovery, do not necessarily send full traffic immediately.

A controlled sequence may be:

1. Validate internal health.
2. Run approved synthetic tests.
3. Enable a small traffic percentage if the routing platform supports it.
4. Observe errors, latency, and saturation.
5. Increase traffic in planned steps.
6. Pause if service-level indicators regress.
7. Restore normal traffic only after acceptance criteria pass.

Check downstream capacity before increasing traffic. A recovered API can overwhelm a database or message broker if a large backlog or retry storm is released at once.

# 32. Monitoring During Recovery

Use Prometheus and Grafana to monitor service and infrastructure health, and ELK to investigate logs.

Important signals include:

- Request rate
- Error rate
- Latency percentiles
- Pod readiness and restart count
- Node CPU and memory pressure
- Disk and inode pressure
- Network errors
- Database connections and query latency
- Redis memory and connections
- RabbitMQ queue depth
- Kafka consumer lag
- ALB target health
- Business transaction success rate

Illustrative PromQL:

```promql
sum(rate(http_requests_total{
  service="payment",
  status=~"5.."
}[5m]))
```

Illustrative pod restart query:

```promql
increase(kube_pod_container_status_restarts_total{
  namespace="roboshop"
}[15m])
```

Metric names and labels depend on installed exporters and application instrumentation. Confirm them in Prometheus before using queries in alerts or runbooks.

# 33. Recovery Validation

Use layered validation.

## 33.1 Infrastructure validation

- AWS resources are in expected state.
- EKS API access works.
- Worker nodes are Ready.
- Required networking is functional.
- Storage is available.
- Security controls remain enabled.

## 33.2 Kubernetes validation

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get services -A
kubectl get endpoints -A
```

## 33.3 Application validation

- Pods pass readiness checks.
- Application logs show no new critical failures.
- Internal service calls work.
- Database connectivity works.
- Message producers and consumers work.
- External dependencies respond within acceptable limits.

## 33.4 Business validation

For an e-commerce workflow:

```text
Login
  |
  v
Browse product
  |
  v
Add to cart
  |
  v
Place test order
  |
  v
Authorize test payment
  |
  v
Reserve inventory
  |
  v
Confirm order
  |
  v
Verify notification
```

Use approved test accounts and non-production payment mechanisms. Do not create real customer transactions during recovery testing.

# 34. Recovery Communication

Provide concise updates containing:

- Incident identifier and severity
- Affected services and customer workflows
- Current impact
- Recovery actions completed
- Current technical state
- Remaining risks or degraded functions
- Next update time
- Owner of the next action

Example:

```text
Incident: INC-<id>
Impact: Checkout requests are failing intermittently.
Current status: Payment service and database connectivity restored.
Recovery: Workloads are healthy; controlled checkout validation is in progress.
Risk: Message backlog is still draining.
Next update: <time>
Owner: <incident commander>
```

Do not state that the incident is resolved until agreed recovery criteria pass.

# 35. Complete Production Recovery Runbook

## Step 1 — Declare incident and scope

Record the incident ID, severity, start time, customer impact, and affected services.

## Step 2 — Stabilize

Pause unsafe changes, protect data, preserve evidence, and reduce additional impact.

## Step 3 — Confirm environment

Verify AWS account, region, EKS cluster, namespace, and administrative access.

## Step 4 — Identify the failed layer

Determine whether the issue is application, Kubernetes, networking, database, messaging, storage, CI/CD, or regional infrastructure.

## Step 5 — Map dependencies

Identify the services and data stores that must recover before customer traffic can resume.

## Step 6 — Select recovery path

Choose restart, reschedule, configuration correction, rollback, failover, restore, or disaster recovery based on evidence.

## Step 7 — Obtain approval

Confirm incident authority and specialist approvals for data, infrastructure, and security actions.

## Step 8 — Recover foundational dependencies

Restore networking, cluster capacity, storage, database, and messaging as required by the dependency map.

## Step 9 — Recover application workloads

Use the approved GitOps and deployment process.

## Step 10 — Validate technical health

Check nodes, pods, deployments, endpoints, logs, metrics, and dependency connectivity.

## Step 11 — Validate data integrity

Reconcile transactions and message processing where required.

## Step 12 — Restore traffic gradually

Use an approved traffic plan and watch for saturation or regression.

## Step 13 — Validate customer journeys

Run approved end-to-end tests and confirm business success signals.

## Step 14 — Communicate and monitor

Send status updates and observe the platform for a defined stabilization window.

## Step 15 — Preserve evidence and hand off

Record actions, timestamps, recovery point, data impact, and unresolved risks. Hand root-cause investigation to the postmortem process.

# 36. Recovery Scenario A — Multiple Services Unavailable

**Situation:** Several microservices fail at approximately the same time.

Possible shared causes:

- CoreDNS
- CNI
- Node pressure
- Shared database
- Redis
- Network policy
- Common secret or identity dependency

Investigation:

```bash
kubectl get pods -A -o wide
kubectl get nodes
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get pods -n kube-system -o wide
```

Compare affected services. If services on different nodes fail simultaneously, investigate shared infrastructure and dependencies rather than restarting each service independently.

Recovery:

1. Identify the common dependency.
2. Stabilize the dependency.
3. Validate a small set of affected services.
4. Restore other services in dependency order.
5. Confirm end-to-end business workflows.

# 37. Recovery Scenario B — RDS Unavailable

**Situation:** Multiple services report database connection errors.

Investigation:

- Confirm RDS status and recent events.
- Check security group and network connectivity.
- Check database connection limits.
- Inspect application pool behavior.
- Check whether a failover occurred.
- Check application credentials and secret references.

Recovery:

1. Coordinate with the database owner.
2. Restore database availability or approved failover.
3. Validate writer endpoint and authentication.
4. Allow application pools to reconnect safely.
5. Control retry pressure.
6. Verify critical queries and transaction behavior.
7. Validate business workflows.

Do not restart every application simultaneously if doing so could create a connection storm.

# 38. Recovery Scenario C — Kafka Backlog

**Situation:** Kafka consumers are unavailable and lag increases.

Investigation:

- Check broker and partition health.
- Check consumer group membership.
- Inspect consumer errors.
- Verify schema compatibility.
- Check downstream database capacity.

Recovery:

1. Restore broker health.
2. Restore one consumer group carefully.
3. Monitor lag and processing rate.
4. Verify idempotency and duplicate handling.
5. Increase consumer capacity only when downstream dependencies can handle it.
6. Confirm business events have been processed.

# 39. Recovery Scenario D — EKS Worker Capacity Loss

**Situation:** A group of worker nodes becomes NotReady.

Investigation:

```bash
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A -o wide
```

Recovery:

1. Confirm node group and EC2 status.
2. Check subnet IP capacity.
3. Check CNI and node bootstrap.
4. Replace unhealthy nodes using the approved managed node group process.
5. Check PodDisruptionBudgets and scheduling.
6. Restore critical workloads first.
7. Confirm service capacity before normal traffic.

# 40. Recovery Scenario E — Regional Disaster

**Situation:** The primary AWS region is unavailable or cannot meet the recovery objective.

Recovery:

1. Declare disaster recovery under the approved authority.
2. Confirm the designated recovery region.
3. Verify network, identity, and encryption dependencies.
4. Restore or promote data according to the DR plan.
5. Bootstrap EKS and platform services.
6. Deploy verified images and GitOps configuration.
7. Validate critical application workflows.
8. Cut traffic over through the approved DNS process.
9. Monitor data consistency and service behavior.
10. Record actual RTO and RPO.

Do not initiate a regional cutover based only on a single failed health check.

# 41. Recovery Failure Modes

| Failure | Risk | Response |
|---|---|---|
| Wrong AWS account | Changes affect wrong environment | Stop and verify identity |
| Wrong cluster context | Recovery changes wrong cluster | Confirm context and cluster identity |
| Database restore to wrong target | Data integrity and outage risk | Stop; verify restore target |
| Stale GitOps state | Reconciliation reverses manual changes | Correct Git desired state |
| Retry storm | Dependencies become overloaded | Apply controlled retry and traffic policy |
| Cache cold start | Database load spike | Warm cache gradually |
| Consumer replay | Duplicate business side effects | Verify idempotency and offsets |
| PVC deletion | Permanent data loss | Stop destructive action |
| DNS change too early | Traffic reaches unhealthy target | Validate before cutover |
| Secrets unavailable | Applications cannot authenticate | Restore through approved secret process |

# 42. Recovery Metrics

Measure recovery performance and quality.

| Metric | Meaning |
|---|---|
| MTTD | Mean time to detect |
| MTTA | Mean time to acknowledge |
| MTTR | Mean time to restore or repair; define the organization's exact usage |
| RTO attainment | Whether recovery met the target |
| RPO attainment | Whether data loss stayed within target |
| Recovery validation time | Time to prove technical and business health |
| Data reconciliation backlog | Outstanding inconsistent transactions |
| Repeat incident rate | Whether the same failure recurs |

Do not optimize for a short recovery time at the expense of data integrity or security.

# 43. Recovery Evidence Record

Capture:

| Field | Required evidence |
|---|---|
| Incident ID | Incident/ticket reference |
| Start time | First known impact |
| Detection time | Alert or report timestamp |
| Affected scope | Services, cluster, region |
| Customer impact | Workflows and duration |
| Recovery method | Restart, rollback, failover, restore, DR |
| Recovery point | Backup/replica timestamp if applicable |
| Data impact | Confirmed or still under investigation |
| Actions | Commands, changes, approvals |
| Validation | Technical and business checks |
| RTO | Actual measured duration |
| RPO | Actual recovery point/data loss |
| Remaining risk | Known degraded functions |
| Owners | Follow-up action owners |

Store evidence in the approved incident system. Do not include credentials or sensitive customer data.

# 44. Recovery Readiness Checklist

## Platform readiness

- [ ] AWS account and region inventory is current.
- [ ] EKS cluster access procedure is documented.
- [ ] Node group recovery procedure is tested.
- [ ] Network and DNS dependencies are documented.
- [ ] Container images are retained by immutable reference.
- [ ] GitOps repository and bootstrap procedure are available.
- [ ] Helm charts and required artifacts are retained.

## Data readiness

- [ ] RDS backup and restore procedures are documented.
- [ ] Restore tests are performed regularly.
- [ ] RTO and RPO are approved.
- [ ] Redis recovery behavior is documented.
- [ ] RabbitMQ and Kafka recovery procedures are tested.
- [ ] Data reconciliation process exists.
- [ ] Encryption keys and secret recovery are addressed.

## Application readiness

- [ ] Service dependency map is current.
- [ ] Critical customer journeys are documented.
- [ ] Readiness and liveness probes are meaningful.
- [ ] Applications tolerate dependency reconnection.
- [ ] Retry and timeout behavior is bounded.
- [ ] Consumers are idempotent where required.
- [ ] Recovery validation tests are available.

## Operational readiness

- [ ] Incident roles are defined.
- [ ] Escalation contacts are current.
- [ ] Recovery runbooks are accessible during an outage.
- [ ] Required access is available through approved controls.
- [ ] DR exercises are scheduled.
- [ ] Recovery evidence is retained.
- [ ] Follow-up actions have owners and due dates.

# 45. Recovery Game Day

Perform this exercise in a non-production environment first.

**Scenario:** The order workflow is unavailable because a shared dependency has failed.

Exercise objectives:

- Detect the issue.
- Identify the shared dependency.
- Stabilize the platform.
- Restore the dependency.
- Recover affected workloads.
- Validate data and message processing.
- Restore customer traffic.
- Record actual recovery time.

Suggested sequence:

```text
Inject approved test failure
          |
          v
Observe alerts and dashboards
          |
          v
Declare incident
          |
          v
Identify common dependency
          |
          v
Restore dependency
          |
          v
Recover dependent services
          |
          v
Validate data and messaging
          |
          v
Run end-to-end test
          |
          v
Measure RTO / RPO
          |
          v
Record lessons and actions
```

Use controlled failure injection with explicit scope, owner, stop conditions, and cleanup. Never run a game day against production without formal authorization.

# 46. Interview and Practical Questions

1. What is the difference between production recovery and rollback?
2. What are RTO and RPO?
3. How do you prioritize recovery for 20+ microservices?
4. How do you identify whether an incident is application-level or infrastructure-level?
5. How do you recover a CrashLoopBackOff workload?
6. What do you check when EKS nodes become NotReady?
7. How do you troubleshoot CoreDNS failures?
8. How do you investigate CNI and pod IP allocation failures?
9. How do you recover an application that returns 503 through an ALB?
10. What are the risks of restoring an RDS database?
11. How do you handle Redis recovery when it stores sessions?
12. How do you recover RabbitMQ without losing messages?
13. How do you recover Kafka consumers without causing duplicate side effects?
14. Why should you not delete a PVC during recovery?
15. How do you restore Argo CD and GitOps reconciliation?
16. How do you recover infrastructure managed by Terraform?
17. How do you validate data integrity after an outage?
18. How do you prevent a retry storm during recovery?
19. How do you perform a regional disaster recovery?
20. How do you prove that the customer workflow has recovered?

# 47. Final Recovery Principles

- Stabilize before making broad changes.
- Verify the account, region, cluster, and namespace.
- Restore shared dependencies before dependent services when the dependency map requires it.
- Protect persistent data.
- Use GitOps as the source of truth for managed workloads.
- Restore from verified images and backups.
- Do not treat pod readiness as proof of business recovery.
- Control retries, traffic, and consumer backlog.
- Validate data consistency.
- Measure actual RTO and RPO.
- Preserve evidence for the postmortem.
- Assign owners to remaining risks and corrective actions.

# 48. Final Outcome

After completing this chapter, you should be able to coordinate a structured production recovery across AWS, EKS, application services, databases, messaging, and GitOps.

```text
INCIDENT
   |
   v
STABILIZE
   |
   v
SCOPE AND PRIORITIZE
   |
   v
RESTORE FOUNDATIONAL DEPENDENCIES
   |
   v
RECOVER PLATFORM AND APPLICATIONS
   |
   v
RECONCILE DATA AND MESSAGES
   |
   v
RESTORE TRAFFIC GRADUALLY
   |
   v
VALIDATE CUSTOMER WORKFLOWS
   |
   v
MEASURE RTO / RPO
   |
   v
COMMUNICATE AND HAND OFF TO RCA
```

**Key production principle:**

Production recovery is complete only when the platform is technically stable, critical data is consistent, dependencies are functioning, and the business workflow is verified. Restoring pods is one step in recovery—not the definition of recovery.

**End of Chapter 210 — Production Recovery.**

# CHAPTER 210 — PRODUCTION RECOVERY: IMPLEMENTATION EXPANSION
## Supplementary implementation runbooks

> This supplement expands the existing chapter. It does not replace or remove the original 48 sections. Adapt names, regions, account IDs, cluster names, storage classes, retention periods, and recovery objectives to the actual environment. Execute destructive or failover actions only with an approved incident/change record.

---

# 49. Recovery architecture and recovery boundaries

## 49.1 Define what is being recovered

A production platform is not one recoverable object. It is a dependency graph containing infrastructure, identity, networking, cluster services, application workloads, data stores, messaging systems, secrets, and external integrations.

Use separate recovery boundaries:

| Boundary | Examples | Recovery evidence |
|---|---|---|
| Cloud foundation | VPC, subnets, routes, NAT, endpoints | Terraform plan/apply and reachability checks |
| Identity | IAM roles, policies, OIDC provider, KMS grants | Authentication and authorization tests |
| EKS foundation | Cluster, node groups, add-ons | API access, nodes Ready, add-ons healthy |
| Platform services | Ingress controller, CoreDNS, metrics components | Service endpoints and health checks |
| Data | RDS, Redis, object storage, persistent volumes | Restore timestamp, integrity checks |
| Messaging | RabbitMQ, Kafka | Broker health, replication, consumer progress |
| Application | Deployments, Services, ConfigMaps | Readiness, smoke tests, business transactions |
| Edge | DNS, certificates, WAF, load balancers | External synthetic checks |

## 49.2 Dependency order

A typical recovery order is:

1. Confirm the target AWS account and region.
2. Restore or validate the cloud network.
3. Restore IAM, KMS, and cluster access.
4. Recreate or recover EKS.
5. Restore cluster add-ons and storage integration.
6. Restore secrets and configuration.
7. Restore databases and messaging.
8. Deploy platform services.
9. Deploy application workloads in dependency order.
10. Validate business transactions.
11. Shift traffic gradually.
12. Continue heightened monitoring.

This is a default order, not a universal law. If a database is intact but the cluster is unavailable, preserve the database and recover the compute plane around it. Avoid restoring healthy components unnecessarily.

## 49.3 Recovery decision record

Record the following before changing production:

```text
Incident ID:
Incident commander:
AWS account ID:
Source region:
Target region:
Cluster:
Namespace(s):
Affected services:
Last known healthy time:
Selected recovery point:
Expected RPO:
Expected RTO:
Data-loss risk:
Traffic-shift approval:
Rollback/abort condition:
Evidence location:
```

---

# 50. EKS cluster disaster recovery implementation

## 50.1 What must be recoverable

An EKS cluster recovery plan must account for more than the Kubernetes API endpoint.

- Cluster configuration and Kubernetes version.
- VPC, private/public subnets, route tables, and security groups.
- EKS control-plane logging configuration where required.
- Managed node groups or self-managed node groups.
- Cluster IAM role and node IAM roles.
- OIDC provider and IRSA trust relationships.
- EKS managed add-ons and compatible versions.
- Storage classes, CSI drivers, and volume snapshot integration.
- Ingress controller and load-balancer controller.
- CoreDNS, kube-proxy, and VPC CNI.
- Namespace-level objects and application manifests.
- Secrets and external secret references.
- Persistent data, which must be recovered separately from Kubernetes object definitions.

Kubernetes manifests do not contain the data stored inside RDS, EBS volumes, S3, Redis, or Kafka. Treat configuration recovery and data recovery as separate workstreams.

## 50.2 Capture cluster configuration before an incident

Run from an approved administrative workstation:

```bash
export AWS_REGION="ap-south-1"
export CLUSTER_NAME="production-eks"

aws sts get-caller-identity

aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --output yaml > eks-cluster-description.yaml

aws eks list-nodegroups \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --output json > eks-nodegroups.json

aws eks list-addons \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --output json > eks-addons.json

for ng in $(aws eks list-nodegroups \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'nodegroups[]' --output text); do
  aws eks describe-nodegroup \
    --cluster-name "$CLUSTER_NAME" \
    --nodegroup-name "$ng" \
    --region "$AWS_REGION" \
    --output yaml > "nodegroup-${ng}.yaml"
done
```

Protect the output. It may disclose account, network, endpoint, role, and infrastructure details. Store it in an access-controlled, encrypted location.

## 50.3 Capture Kubernetes objects

The following is an inventory aid, not a complete backup of cluster state:

```bash
mkdir -p cluster-inventory

kubectl get namespaces -o yaml > cluster-inventory/namespaces.yaml
kubectl get deployments,statefulsets,daemonsets -A -o yaml \
  > cluster-inventory/workloads.yaml
kubectl get services,ingresses -A -o yaml \
  > cluster-inventory/networking.yaml
kubectl get configmaps -A -o yaml \
  > cluster-inventory/configmaps.yaml
kubectl get pvc,pv,storageclass -A -o yaml \
  > cluster-inventory/storage-objects.yaml
kubectl get serviceaccounts,roles,rolebindings -A -o yaml \
  > cluster-inventory/rbac-namespaced.yaml
kubectl get clusterroles,clusterrolebindings -o yaml \
  > cluster-inventory/rbac-cluster.yaml
```

Do not export Secret objects into a general-purpose inventory folder. If a secure backup of Kubernetes Secrets is required, use an approved encrypted backup mechanism and tightly restrict access. Prefer external secret stores and rehydration through workload identity.

## 50.4 Recreate a cluster with infrastructure as code

Use the existing Terraform modules and the approved recovery variable set. A simplified module invocation might look like:

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version
  subnet_ids      = module.vpc.private_subnet_ids

  endpoint_private_access = true
  endpoint_public_access  = false

  node_groups = {
    system = {
      instance_types = ["m6i.large"]
      min_size       = 2
      desired_size   = 3
      max_size       = 6
    }
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Recovery    = "approved"
  }
}
```

This is an illustrative interface. It will not run unless the module exposes these exact input variables. Review the actual module contract before applying.

Recovery procedure:

1. Confirm the AWS account with `aws sts get-caller-identity`.
2. Confirm the region and target VPC.
3. Retrieve the approved Terraform state and backend configuration.
4. Run `terraform init`.
5. Run `terraform validate`.
6. Run `terraform plan -out=recovery.tfplan`.
7. Review all replacements and deletions.
8. Obtain approval for the plan.
9. Apply the reviewed plan.
10. Save the plan and apply logs as incident evidence.

```bash
terraform init
terraform validate
terraform plan -out=recovery.tfplan
terraform show recovery.tfplan
# Apply only after peer review and approval:
terraform apply recovery.tfplan
```

Never create a second cluster using a copied Terraform state file without a deliberate state-recovery procedure. State conflicts can cause resources to be managed by the wrong workspace or environment.

## 50.5 Configure kubectl and validate access

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"

kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

Expected checks:

- The current identity is the approved recovery identity.
- The API endpoint resolves and is reachable.
- Nodes join and become `Ready`.
- System pods are healthy.
- RBAC grants only the intended access.
- Private endpoint access works from the approved network path.

## 50.6 Restore managed add-ons

List add-ons and their health:

```bash
aws eks list-addons \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION"

kubectl get pods -n kube-system -o wide
kubectl get daemonsets -n kube-system
```

Install or update each add-on using a version compatible with the cluster version. Do not blindly use `latest` during a disaster recovery event. Validate the VPC CNI, CoreDNS, kube-proxy, EBS CSI driver, and any required load-balancer integration.

---

# 51. AWS backup and restore implementation

## 51.1 Backup policy design

A backup policy must define:

- Protected resource types.
- Backup frequency.
- Retention period.
- Vault and encryption key.
- Cross-account or cross-region copy requirements.
- Recovery point objectives.
- Restore testing frequency.
- Who can delete recovery points.
- Alerting for missed or failed jobs.

Example policy matrix:

| Resource | Example protection | Restore validation |
|---|---|---|
| RDS | Automated backups and PITR | Restore to temporary DB and query data |
| EBS | Scheduled snapshots | Create test volume and mount read-only |
| S3 | Versioning and replication where required | Restore an earlier object version |
| EKS configuration | Git plus protected inventory | Recreate cluster and apply manifests |
| Secrets | Secrets Manager replication/backup design | Retrieve with recovery role and validate rotation |
| Kafka/RabbitMQ | Product-specific replication and backup | Broker recovery plus message/offset validation |

A snapshot is not proven recoverable merely because the console shows it as completed. Test restore procedures and record the time taken.

## 51.2 RDS point-in-time recovery

Inspect the DB instance and backup configuration:

```bash
aws rds describe-db-instances \
  --db-instance-identifier production-db \
  --region "$AWS_REGION" \
  --query 'DBInstances[0].{Status:DBInstanceStatus,BackupRetention:BackupRetentionPeriod,LatestRestorableTime:LatestRestorableTime,MultiAZ:MultiAZ}'
```

List available DB snapshots:

```bash
aws rds describe-db-snapshots \
  --db-instance-identifier production-db \
  --region "$AWS_REGION" \
  --query 'DBSnapshots[].{ID:DBSnapshotIdentifier,Status:Status,Time:SnapshotCreateTime}'
```

For point-in-time restore, choose a recovery timestamp that is within the available restore window and before the suspected corruption or destructive event.

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier production-db \
  --target-db-instance-identifier production-db-recovery \
  --restore-time "2026-10-30T09:15:00Z" \
  --db-subnet-group-name production-db-subnets \
  --region "$AWS_REGION"
```

The timestamp above is an example only. Use the incident-approved timestamp and verify that the source has a valid restorable window. The target identifier must be unused.

After restore:

1. Wait for the new instance to become available.
2. Confirm engine version, parameter group, subnet group, security groups, and encryption.
3. Connect from a controlled recovery host.
4. Validate schemas and expected row counts.
5. Run application-level read tests.
6. Validate foreign keys and critical business invariants.
7. Confirm connection pool settings.
8. Update application configuration through the approved secret/configuration workflow.
9. Run controlled write tests.
10. Switch traffic only after approval.

Do not overwrite the original database as the first recovery action. Restore to a separate target, validate, and then make a controlled cutover decision.

## 51.3 EBS snapshot restore

Inspect snapshots:

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --region "$AWS_REGION" \
  --query 'Snapshots[].{Snapshot:SnapshotId,Volume:VolumeId,State:State,StartTime:StartTime}'
```

Create a recovery volume in the correct Availability Zone:

```bash
aws ec2 create-volume \
  --snapshot-id snap-0123456789abcdef0 \
  --availability-zone ap-south-1a \
  --volume-type gp3 \
  --region "$AWS_REGION"
```

The snapshot ID and Availability Zone are placeholders. The restored EBS volume must be attached to an instance in the same Availability Zone. For Kubernetes persistent volumes, follow the CSI driver's supported snapshot and restore workflow rather than manually attaching a volume behind Kubernetes' back.

## 51.4 S3 object recovery

Check bucket versioning:

```bash
aws s3api get-bucket-versioning \
  --bucket production-artifacts \
  --region "$AWS_REGION"
```

List versions for a known object:

```bash
aws s3api list-object-versions \
  --bucket production-artifacts \
  --prefix "releases/service-a/" \
  --region "$AWS_REGION"
```

Copy a selected version to a recovery location:

```bash
aws s3api get-object \
  --bucket production-artifacts \
  --key "releases/service-a/config.yaml" \
  --version-id "VERSION_ID" \
  /tmp/config.yaml
```

Validate the recovered object checksum and metadata before promoting it. Do not assume every S3 bucket has versioning, replication, or Object Lock enabled.

---

# 52. Cross-region disaster recovery

## 52.1 Recovery design

A cross-region recovery design needs independent decisions for:

- Network topology and CIDR planning.
- EKS cluster recreation.
- ECR image availability.
- Terraform state access.
- RDS backup or replica strategy.
- S3 replication.
- KMS key availability and grants.
- Secrets availability.
- DNS failover.
- External dependencies and allowlists.
- Capacity in the target region.

Do not assume a resource in one region automatically exists in another. Regional services and encryption keys require explicit planning.

## 52.2 ECR image availability

For a target region, verify the repository and image digest:

```bash
aws ecr describe-images \
  --repository-name service-a \
  --region "$AWS_REGION" \
  --query 'imageDetails[].{Digest:imageDigest,Tags:imageTags,PushedAt:imagePushedAt}'
```

A recovery deployment should use immutable image digests where possible. Tags such as `latest` can point to different images over time.

Example image reference:

```text
ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/service-a@sha256:IMAGE_DIGEST
```

If cross-region replication is part of the design, verify that replication rules cover all required repositories and that the target registry contains the exact digests required for recovery.

## 52.3 Route 53 failover planning

Before an incident:

- Define primary and recovery endpoints.
- Configure health checks or an external health evaluation design.
- Define TTL and expected DNS caching behavior.
- Decide whether failover is automatic or operator-approved.
- Test the failback process.
- Confirm certificates and WAF policies in the recovery region.

DNS changes are not instantaneous for every client because resolvers and applications may cache records. Measure actual client behavior during game days.

## 52.4 Controlled traffic shift

A safe traffic shift has explicit gates:

1. Recovery region infrastructure is healthy.
2. Application dependencies are healthy.
3. Data replication lag is within the approved threshold.
4. Synthetic checks pass.
5. Business transaction tests pass.
6. Security and access checks pass.
7. Incident commander approves traffic shift.
8. Traffic is moved gradually where the architecture supports it.
9. Error rate and latency are watched.
10. The team records the exact cutover time.

Do not send production traffic to a recovery region with stale or inconsistent data merely because its health endpoint returns HTTP 200.

---

# 53. Database recovery and data-integrity runbooks

## 53.1 Database recovery checklist

Before restoring:

- Identify the database and owning service.
- Confirm whether the failure is availability, corruption, accidental deletion, bad migration, or credential-related.
- Stop or restrict writes if continued writes can worsen corruption.
- Record the last known good timestamp.
- Determine the RPO and acceptable data-loss window.
- Identify dependent services and message consumers.
- Preserve logs and audit evidence.
- Select a recovery point with the application owner.

## 53.2 Postgres validation examples

Run read-only checks first:

```sql
SELECT now();
SELECT current_database();
SELECT current_user;
SELECT version();

SELECT schemaname, COUNT(*)
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY schemaname;
```

Check critical application tables:

```sql
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM payments;
SELECT MAX(created_at) FROM orders;
```

These table names are examples. Use the actual schema and verify counts against known-good evidence. Counts alone do not prove correctness; validate representative records, relationships, status transitions, and business totals.

## 53.3 MySQL validation examples

```sql
SELECT NOW();
SELECT DATABASE();
SELECT VERSION();

SELECT table_schema, COUNT(*)
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY table_schema;

SELECT COUNT(*) FROM orders;
SELECT MAX(created_at) FROM orders;
```

Run only queries appropriate to the restored schema. Avoid expensive full-table scans on a production database under recovery load.

## 53.4 Connection recovery

After a database endpoint changes:

- Update the secret or configuration source, not individual pods manually.
- Confirm DNS resolution from the application namespace.
- Confirm security group and network policy paths.
- Validate TLS settings and CA trust.
- Check connection pool maximum, minimum idle, timeout, and lifetime settings.
- Restart or roll workloads only when required.
- Watch connection count, rejected connections, and pool wait time.

Example Kubernetes check:

```bash
kubectl exec -n production deploy/orders -- \
  sh -c 'getent hosts "$DB_HOST"'
```

The container must include `getent` and the `DB_HOST` variable. If it does not, use an approved diagnostic pod in the same network and namespace.

## 53.5 Reconcile transactions after restore

A database restore may roll the database back to an earlier point while messages, external payment systems, or downstream services have progressed.

Build a reconciliation report for:

- Orders created near the recovery boundary.
- Payment authorizations and captures.
- Inventory reservations and releases.
- Shipment or notification events.
- Duplicate transaction identifiers.
- Orphaned child records.
- Events published but not committed, or committed but not published.

Do not blindly replay all events. Use idempotency keys, deduplication, business ownership, and a reviewed replay window.

---

# 54. RabbitMQ recovery implementation

## 54.1 Recovery questions

Before taking action, identify:

- Cluster topology and node count.
- Queue type: classic, quorum, or stream.
- Queue durability and message persistence settings.
- Replication and leader placement.
- Publisher confirms and consumer acknowledgements.
- Dead-letter exchange and retry policy.
- Whether messages are business-critical or reconstructable.
- Whether consumers can safely process duplicate deliveries.

## 54.2 Health checks

Use the management API or approved CLI:

```bash
rabbitmq-diagnostics status
rabbitmq-diagnostics cluster_status
rabbitmq-diagnostics check_running
rabbitmq-diagnostics check_local_alarms
```

Run commands on the relevant broker node with the required permissions.

## 54.3 Recovery sequence

1. Preserve node logs and broker status.
2. Identify whether one node or the whole cluster is affected.
3. Confirm quorum for quorum queues.
4. Avoid deleting data directories to force a node to rejoin.
5. Restore nodes using the documented RabbitMQ cluster recovery procedure for the deployed version.
6. Verify queue leaders and replica synchronization.
7. Check consumer connections and unacknowledged message counts.
8. Start consumers gradually.
9. Verify dead-letter and retry behavior.
10. Reconcile business transactions.

## 54.4 Message backlog validation

Record queue depth and consumer state before and after recovery. A queue becoming empty is not sufficient proof of success: messages may have been rejected, dead-lettered, expired, or processed incorrectly.

Use a test message with a unique correlation ID in a controlled test queue to validate publisher-to-consumer flow. Never inject test messages into a business queue without an approved test procedure.

---

# 55. Kafka recovery implementation

## 55.1 Recovery data to preserve

- Broker configuration and cluster metadata.
- Topic names, partition counts, and replication factors.
- Replication and ISR status.
- Consumer group offsets.
- ACLs and principal mappings.
- Schema registry state if used.
- Retention and compaction configuration.
- Producer idempotence and transaction configuration.
- Cross-region replication state, if configured.

## 55.2 Inspect cluster health

Use the Kafka CLI version compatible with the deployed cluster:

```bash
kafka-topics.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --describe

kafka-consumer-groups.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --all-groups \
  --describe
```

Check under-replicated partitions and unavailable partitions using the monitoring and administration mechanisms available in the deployed Kafka distribution.

## 55.3 Consumer recovery

Do not reset consumer offsets as a first response to lag. First determine whether consumers are slow, stopped, blocked on dependencies, or processing poison messages.

Offset reset is a data-processing decision. It can cause reprocessing or skipping.

Example dry-run pattern:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server "$BOOTSTRAP_SERVERS" \
  --group orders-consumer \
  --topic orders \
  --reset-offsets \
  --to-datetime "2026-10-30T09:00:00.000" \
  --dry-run
```

Review the proposed offsets with the application owner. Execute only after approval and only with the correct topic, partition, group, and time boundary.

## 55.4 Replay safety

Before replaying messages:

- Confirm the consumer's idempotency strategy.
- Confirm database uniqueness constraints.
- Confirm external side effects are protected against duplicates.
- Define the exact partition and offset range.
- Preserve the original offsets and replay evidence.
- Start with a controlled subset.
- Compare expected and actual business outcomes.

---

# 56. Secrets, IAM, KMS, and certificate recovery

## 56.1 Identity recovery checklist

Validate:

- Cluster IAM role exists and has intended trust policy.
- Node role has only required permissions.
- OIDC provider exists for the recovered EKS cluster.
- IRSA trust policies reference the correct OIDC issuer.
- Service account annotations reference the correct role ARN.
- KMS key is enabled and accessible.
- Secrets Manager or Parameter Store values are available in the target region.
- External DNS, registry, and database credentials are valid.

## 56.2 Validate IRSA

```bash
kubectl get serviceaccount -n production orders-sa -o yaml
kubectl describe pod -n production -l app=orders
```

Check the service account annotation and pod identity configuration. Validate actual AWS access from a controlled pod using the expected workload identity. Do not print credentials or tokens into logs.

## 56.3 Secret recovery

Use the approved secret store and recovery role. Do not paste secret values into incident tickets, chat, terminal recordings, or shell history.

Example metadata check:

```bash
aws secretsmanager describe-secret \
  --secret-id production/orders/database \
  --region "$AWS_REGION"
```

If restoring a prior secret version, verify which version is current and which version is safe. Rotate credentials after a suspected compromise. A restored old password may no longer match the database.

## 56.4 KMS dependency

Encrypted backups and snapshots may be unusable if the recovery identity cannot access the relevant KMS key. Test key policies, grants, and cross-account permissions before an incident. Do not disable or schedule deletion of a key during recovery.

## 56.5 Certificate recovery

Validate certificate presence, expiry, SANs, private-key access, and listener association. Reissue or reimport certificates only through the approved certificate process. After a DNS or region cutover, test the certificate from outside the cluster.

---

# 57. Recovery automation and evidence scripts

## 57.1 Read-only EKS recovery snapshot

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:-production}"
OUT="recovery-evidence-$(date -u +%Y%m%dT%H%M%SZ)"

mkdir -p "$OUT"

{
  echo "Captured at: $(date -u --iso-8601=seconds)"
  echo "Context: $(kubectl config current-context)"
  kubectl cluster-info
} > "$OUT/cluster.txt" 2>&1

kubectl get nodes -o wide > "$OUT/nodes.txt" 2>&1 || true
kubectl get pods -A -o wide > "$OUT/pods-all.txt" 2>&1 || true
kubectl get events -A --sort-by=.lastTimestamp \
  > "$OUT/events.txt" 2>&1 || true

kubectl get deployments,statefulsets,daemonsets \
  -n "$NAMESPACE" -o wide > "$OUT/workloads.txt" 2>&1 || true

kubectl get services,ingresses \
  -n "$NAMESPACE" -o wide > "$OUT/networking.txt" 2>&1 || true

echo "Evidence written to $OUT"
```

This script is read-only. It may still capture internal names and operational metadata, so store the output securely.

## 57.2 Workload readiness report

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:-production}"

kubectl get deployments -n "$NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas,AVAILABLE:.status.availableReplicas'

kubectl get statefulsets -n "$NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas'

kubectl get pods -n "$NAMESPACE" \
  --field-selector=status.phase!=Running,status.phase!=Succeeded
```

Review output manually. A pod in `Running` phase may still fail readiness or business-level health checks.

## 57.3 HTTP smoke-test script

```bash
#!/usr/bin/env bash
set -euo pipefail

URL="${1:?Usage: $0 https://service.example.com/health}"
EXPECTED="${2:-200}"

STATUS="$(curl --silent --show-error \
  --output /tmp/recovery-smoke-body.txt \
  --write-out '%{http_code}' \
  --connect-timeout 5 \
  --max-time 15 \
  "$URL")"

if [[ "$STATUS" == "$EXPECTED" ]]; then
  echo "PASS: $URL returned HTTP $STATUS"
else
  echo "FAIL: $URL returned HTTP $STATUS; expected $EXPECTED"
  exit 1
fi
```

An HTTP 200 response is only a basic smoke test. Add separate tests for database access, messaging, authentication, and critical business transactions.

## 57.4 Recovery evidence record

For each action, capture:

```text
Timestamp UTC:
Operator:
Incident/change ID:
Target account and region:
Target resource:
Command or action:
Approval reference:
Result:
Validation performed:
Evidence file:
Follow-up:
```

Never include passwords, tokens, private keys, or full secret values.

---

# 58. Recovery game-day scenarios with pass/fail criteria

## 58.1 Scenario A — EKS worker group unavailable

**Injection:** In a non-production environment, make a worker group unavailable using an approved fault-injection method.

**Expected response:**
- Detect node loss.
- Confirm workload disruption.
- Verify pod rescheduling.
- Check PodDisruptionBudgets and topology spread.
- Restore node capacity.
- Confirm application health and capacity.

**Pass criteria:**
- Critical workloads recover within the exercise RTO.
- No unapproved data loss.
- No persistent scheduling failures.
- Evidence and timeline are complete.

## 58.2 Scenario B — Bad application release

**Injection:** Deploy a deliberately faulty image to a test namespace.

**Expected response:**
- Detect readiness or error-rate regression.
- Stop further rollout.
- Identify the known-good immutable image.
- Roll back using the deployment's source-of-truth workflow.
- Validate database compatibility.
- Run smoke and business tests.

**Pass criteria:**
- Rollback is reproducible from Git or the approved release record.
- Argo CD does not immediately reapply the faulty desired state.
- Application health returns to baseline.

## 58.3 Scenario C — Database restore

**Injection:** Restore a test database to a known timestamp.

**Expected response:**
- Select the recovery point.
- Restore to a separate target.
- Validate schema and representative records.
- Reconcile dependent messages and transactions.
- Perform controlled application cutover.

**Pass criteria:**
- Restore point and actual data boundary are documented.
- Application connectivity succeeds.
- Data validation is signed off by the database and application owners.

## 58.4 Scenario D — Region unavailable

**Injection:** Simulate loss of access to the primary region without disrupting real production.

**Expected response:**
- Activate incident command.
- Confirm target-region readiness.
- Verify images, secrets, keys, and data.
- Recover platform and services in dependency order.
- Validate business flows.
- Approve and perform traffic shift.

**Pass criteria:**
- Measured RTO and RPO meet approved objectives.
- DNS and external dependencies work.
- No critical service is exposed before security validation.

## 58.5 Scenario E — Messaging backlog

**Injection:** Stop a test consumer and generate controlled test traffic.

**Expected response:**
- Detect backlog and lag.
- Identify consumer or dependency bottleneck.
- Restore consumers gradually.
- Check duplicate processing and dead-letter handling.
- Verify business outcomes.

**Pass criteria:**
- Backlog drains at a measured safe rate.
- No unapproved offset reset.
- No duplicate business transaction.

## 58.6 Game-day report template

```text
Exercise:
Date:
Environment:
Scenario owner:
Observers:
Expected RTO:
Measured RTO:
Expected RPO:
Measured RPO:
Detection time:
Decision time:
Recovery start:
Service restored:
Validation completed:
Data integrity result:
Security result:
What worked:
What failed:
Corrective actions:
Owners and due dates:
Retest date:
```

---

# 59. Recovery acceptance gates

Do not declare recovery complete until the relevant gates are satisfied.

| Gate | Required evidence |
|---|---|
| Infrastructure | Network, compute, IAM, and cluster checks pass |
| Platform | CoreDNS, CNI, ingress, storage, and controllers healthy |
| Data | Restore point and integrity checks approved |
| Messaging | Broker health, replication, lag, and replay safety validated |
| Application | Readiness, smoke tests, and critical business transactions pass |
| Security | IAM, secrets, encryption, certificates, and access reviewed |
| Traffic | Gradual shift completed and external checks pass |
| Operations | Dashboards, alerts, runbooks, and on-call ownership active |
| Incident closure | Timeline, evidence, risks, and follow-up actions recorded |

A recovery is not complete simply because pods are `Running`. The service must be safe, correct, observable, and usable by its customers.

---

# 60. Final implementation deliverables

The team should maintain these artifacts in version control or an approved secure repository:

1. Recovery architecture diagram.
2. Dependency map for all critical services.
3. Terraform recovery workspace and reviewed module versions.
4. EKS configuration and add-on compatibility record.
5. Kubernetes configuration backup and restore procedure.
6. RDS restore and validation runbook.
7. EBS and S3 restore procedures.
8. RabbitMQ and Kafka recovery runbooks.
9. Secrets, IAM, KMS, and certificate recovery procedures.
10. Cross-region recovery procedure.
11. DNS and traffic cutover procedure.
12. Read-only evidence scripts.
13. Recovery smoke tests.
14. Game-day scenarios and reports.
15. RTO/RPO measurements and approval records.
16. Post-recovery reconciliation checklist.
17. Incident communication template.
18. Corrective-action register with owners and due dates.

## Chapter 210 completion criteria

Chapter 210 can be considered implemented only when the team can demonstrate all of the following in a controlled environment:

- Recreate or recover the EKS platform from approved infrastructure definitions.
- Restore a database to a selected recovery point and validate its data.
- Recover required object and block storage.
- Restore messaging service availability without unsafe message loss or replay.
- Re-establish secrets, IAM, KMS, and certificate dependencies.
- Recover a critical application path from edge to database.
- Perform a controlled traffic shift.
- Measure actual RTO and RPO.
- Produce evidence and a post-exercise action plan.

**End of Chapter 210 implementation expansion.**
