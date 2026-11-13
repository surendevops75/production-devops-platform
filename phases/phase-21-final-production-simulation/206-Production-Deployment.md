# CHAPTER 203 — PRODUCTION DEPLOYMENT

**Filename:** `203-Production-Deployment.md`

---

# 1. Chapter Objective

The objective of this chapter is to deploy the approved RoboShop application into Production using a controlled, secure, and repeatable deployment process.

Production deployment is different from DEV, QA, and Staging.

A mistake in Production can affect real users, business transactions, revenue, and application availability.

Therefore, every production deployment must have:

- Approved release artifacts.
- Verified image digests.
- Infrastructure readiness.
- Change approval.
- Security validation.
- Database migration planning.
- Deployment strategy.
- Health checks.
- Monitoring and alerting.
- Rollback procedures.
- Incident response procedures.
- Deployment evidence.

## 1.1 Production Deployment Flow

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
   +--> Build Docker Image
   +--> Push to ECR
   |
   v
Immutable Image Digest
   |
   v
DEV
   |
   v
QA
   |
   v
STAGING
   |
   v
Staging Approval
   |
   v
Production Change Request
   |
   v
Release Approval
   |
   v
Production GitOps Promotion PR
   |
   v
Git Review and Approval
   |
   v
Argo CD
   |
   v
AWS EKS PRODUCTION
   |
   +--> Deployment
   +--> Health Validation
   +--> Traffic Validation
   +--> Monitoring
   +--> Smoke Tests
   |
   v
Production Sign-Off
```

## 1.2 Core Production Principle

Never rebuild an application image separately for Production.

Promote the same image digest that successfully passed QA and Staging.

```text
Build Once
   |
   v
Scan Once
   |
   v
Test
   |
   v
Publish Immutable Artifact
   |
   v
Promote Same Artifact
   |
   +--> DEV
   +--> QA
   +--> STAGING
   +--> PRODUCTION
```

---

# 2. Production Architecture

The Production environment should provide appropriate isolation, availability, scalability, and security.

## 2.1 Production Components

| Layer | Technology |
|---|---|
| Cloud | AWS |
| Kubernetes | Amazon EKS |
| Container Registry | Amazon ECR |
| Infrastructure as Code | Terraform |
| Deployment | Helm |
| GitOps | Argo CD |
| CI/CD | Jenkins / GitHub Actions / GitLab CI |
| Secrets | AWS Secrets Manager |
| Identity | IAM, IRSA or EKS Pod Identity |
| Database | Amazon RDS |
| Cache | Redis |
| Messaging | RabbitMQ / Kafka as implemented |
| Load Balancing | AWS ALB |
| DNS | Route 53 |
| TLS | AWS ACM |
| Monitoring | Prometheus |
| Dashboards | Grafana |
| Logging | ELK Stack |
| Security | SonarQube, Trivy, Veracode |
| Performance Testing | k6 |

## 2.2 Production Environment Layout

```text
AWS PRODUCTION ACCOUNT
|
+-- VPC
|   |
|   +-- Public Subnets
|   |   +-- Application Load Balancer
|   |
|   +-- Private Subnets
|       +-- EKS Worker Nodes
|       +-- Application Pods
|       +-- Internal Services
|
+-- EKS Production Cluster
|   |
|   +-- production namespace
|       |
|       +-- frontend
|       +-- user
|       +-- catalog
|       +-- cart
|       +-- orders
|       +-- payment
|       +-- inventory
|       +-- notification
|       +-- remaining approved services
|
+-- RDS
|
+-- Redis
|
+-- Messaging Infrastructure
|
+-- ECR
|
+-- Secrets Manager
|
+-- Prometheus
|
+-- Grafana
|
+-- ELK
```

For stronger isolation, use a separate AWS account and EKS cluster for Production.

Production workloads should not share credentials, databases, or administrative permissions with lower environments.

---

# 3. Production Release Governance

Before deploying, establish the release record.

Example release:

```text
Release ID: roboshop-2026.11.03
Environment: Production
Source: Approved Staging Release
Deployment Method: GitOps
Deployment Strategy: Rolling Update
Approval: Required
Rollback: Prepared
```

## 3.1 Required Approvals

| Role | Responsibility |
|---|---|
| Application Owner | Application release approval |
| QA Owner | Test completion approval |
| DevOps Owner | Deployment readiness |
| Security Owner | Security approval |
| Database Owner | Database migration approval |
| Change Manager | Production change approval |

Actual approval roles depend on organizational policy.

## 3.2 Production Change Request

A production change request should contain:

- Change description.
- Business reason.
- Release ID.
- Application version.
- Git commit.
- Image digests.
- Services affected.
- Database changes.
- Expected impact.
- Deployment window.
- Deployment procedure.
- Validation steps.
- Rollback procedure.
- Responsible engineers.
- Approval records.

## 3.3 Change Risk Classification

Example:

| Risk | Example |
|---|---|
| Low | Configuration-only change |
| Medium | Application deployment |
| High | Database schema migration |
| Critical | Network or cluster architecture change |

Risk classification must follow the organization's change-management policy.

---

# 4. Production Readiness Checklist

Do not start deployment until the required readiness checks are completed.

## 4.1 Application Readiness

- [ ] Staging approval completed.
- [ ] Regression tests passed.
- [ ] Integration tests passed.
- [ ] Performance tests reviewed.
- [ ] Security scans reviewed.
- [ ] No unresolved critical defects.
- [ ] Application configuration reviewed.
- [ ] Image digests verified.
- [ ] Database compatibility verified.
- [ ] Rollback compatibility reviewed.

## 4.2 Infrastructure Readiness

- [ ] Correct AWS account.
- [ ] Correct EKS cluster.
- [ ] Worker nodes healthy.
- [ ] Required EKS add-ons healthy.
- [ ] Load balancer operational.
- [ ] DNS configured.
- [ ] TLS certificate valid.
- [ ] Database available.
- [ ] Redis available.
- [ ] Messaging infrastructure healthy.
- [ ] Storage operational.
- [ ] Network policies reviewed.

## 4.3 Operational Readiness

- [ ] Prometheus operational.
- [ ] Grafana dashboards available.
- [ ] ELK ingestion working.
- [ ] Alerts configured.
- [ ] On-call team identified.
- [ ] Incident escalation documented.
- [ ] Backup and recovery validated.
- [ ] Rollback tested or rehearsed.
- [ ] Deployment evidence location prepared.

---

# 5. Verify Production AWS Account

Use the approved Production deployment identity.

```bash
aws sts get-caller-identity
```

Confirm:

- AWS account ID.
- IAM role or user.
- Intended Production account.

Set variables:

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=roboshop-production-eks
```

Update kubeconfig:

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

Check context:

```bash
kubectl config current-context
```

Check cluster:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes -o wide
```

**Critical control:** Verify the account ID and cluster name independently before executing any production modification.

For sensitive environments, use a separate deployment role and a controlled deployment workstation or CI runner.

---

# 6. Production Infrastructure Validation

## 6.1 Check EKS Nodes

```bash
kubectl get nodes -o wide
```

Check node utilization:

```bash
kubectl top nodes
```

Inspect node details:

```bash
kubectl describe node <NODE_NAME>
```

Validate:

- Node status.
- CPU capacity.
- Memory capacity.
- Pod capacity.
- Node conditions.
- Taints.
- Labels.
- Availability Zone distribution.

## 6.2 Check System Components

```bash
kubectl get pods -n kube-system
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Check AWS Load Balancer Controller:

```bash
kubectl get deployment \
  aws-load-balancer-controller \
  -n kube-system
```

Check cluster add-ons:

```bash
aws eks list-addons \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

## 6.3 Validate Node Distribution

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Production workloads should be distributed across suitable Availability Zones.

Do not assume that having several nodes automatically provides high availability. Verify actual pod placement and failure-domain distribution.

---

# 7. Production Namespace and Policies

Check the Production namespace:

```bash
kubectl get namespace production
```

If it is not managed through GitOps, create it through the approved infrastructure process:

```bash
kubectl create namespace production \
  --dry-run=client -o yaml | kubectl apply -f -
```

Apply labels:

```bash
kubectl label namespace production \
  environment=production \
  application=roboshop \
  --overwrite
```

Check:

```bash
kubectl get namespace production --show-labels
```

## 7.1 ResourceQuota

Example:

`production-resource-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-resource-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    pods: "300"
    services: "150"
    persistentvolumeclaims: "50"
```

Apply:

```bash
kubectl apply -f production-resource-quota.yaml
```

These values are illustrative. Derive the actual quota from capacity planning and workload requirements.

## 7.2 LimitRange

`production-limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limit-range
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 50m
        memory: 64Mi
```

Apply:

```bash
kubectl apply -f production-limit-range.yaml
```

---

# 8. Production Access Control

Production permissions must be more restrictive than lower environments.

Separate:

- Application developers.
- Read-only support engineers.
- Deployment operators.
- Cluster administrators.
- Security administrators.

Example read-only application role:

`production-readonly-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: production-readonly
  namespace: production
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - services
      - events
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch
```

Example RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: production-readonly-binding
  namespace: production
subjects:
  - kind: Group
    name: production-readonly
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: production-readonly
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f production-readonly-role.yaml
kubectl apply -f production-readonly-binding.yaml
```

Validate:

```bash
kubectl auth can-i get pods \
  -n production \
  --as-group=production-readonly
```

Use EKS access entries or the organization's configured identity integration to map actual identities into Kubernetes groups.

Do not give every engineer cluster-admin access.

---

# 9. Production Secrets and Workload Identity

Use AWS Secrets Manager and the approved secrets delivery integration.

Example secret names:

```text
/roboshop/production/user/database
/roboshop/production/catalog/database
/roboshop/production/orders/database
/roboshop/production/payment/provider
/roboshop/production/rabbitmq/credentials
```

Production secrets must be different from QA and Staging secrets.

For application workloads, use IRSA or EKS Pod Identity.

Verify:

- Correct IAM role.
- Correct trust policy.
- Correct service account.
- Minimum required AWS permissions.
- No hardcoded access keys.
- Secret rotation strategy.
- Access auditability.

Example ServiceAccount using IRSA:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-service-account
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/roboshop-production-user-role
```

Apply through the approved GitOps configuration.

Never copy Production credentials into a lower environment or Git repository.

---

# 10. Production Database Readiness

Database readiness is one of the most important parts of production deployment.

Before application deployment:

1. Verify database availability.
2. Verify backup status.
3. Verify recovery procedures.
4. Verify connection limits.
5. Verify database storage.
6. Verify security groups.
7. Verify TLS.
8. Verify application credentials.
9. Review pending schema changes.
10. Approve the migration plan.

## 10.1 Check RDS

```bash
aws rds describe-db-instances \
  --region "$AWS_REGION" \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,MultiAZ,Endpoint.Address]'
```

Review:

- Multi-AZ configuration where required.
- Backup retention.
- Storage capacity.
- Monitoring.
- Parameter groups.
- Maintenance window.
- Encryption.
- Network access.

## 10.2 Check Backups

Review the automated backup configuration and latest successful recovery point using the approved RDS or AWS Backup procedures.

A backup is not sufficient evidence of recoverability. Periodically test restoring into an isolated environment.

## 10.3 Database Migration Strategy

Use backward-compatible database migrations.

Example:

```text
Old Application
      |
      v
Old Database Schema
      |
      v
Expand Migration
      |
      v
Compatible Old + New Schema
      |
      v
Deploy New Application
      |
      v
Validate
      |
      v
Contract Migration Later
```

### Expand-and-Contract Example

Suppose the application needs to rename a database field.

Unsafe approach:

1. Rename the column immediately.
2. Deploy the new application.
3. Old application instances fail during rollout.

Safer approach:

1. Add the new column.
2. Deploy code compatible with both columns.
3. Backfill data.
4. Verify data consistency.
5. Switch application reads and writes.
6. Remove the old column in a later approved change.

Do not run destructive migrations automatically as an unreviewed side effect of every application deployment.

---

# 11. Production Messaging and Cache Readiness

Validate the actual messaging architecture.

## 11.1 Redis

Verify:

- Endpoint.
- Authentication.
- TLS.
- Failover.
- Memory.
- Eviction policy.
- Connection limits.
- Application connectivity.

## 11.2 RabbitMQ

Verify:

- Broker cluster health.
- Queue health.
- Exchange configuration.
- Bindings.
- Consumer health.
- Dead-letter queues.
- Authentication.
- Persistence and recovery.

## 11.3 Kafka

Verify:

- Broker health.
- Topic configuration.
- Partition count.
- Replication factor.
- Consumer groups.
- Retention.
- Authentication.
- TLS.
- Consumer lag.

## 11.4 Messaging Deployment Safety

Application deployments should tolerate:

- Duplicate messages.
- Temporary broker outages.
- Consumer restarts.
- Delayed message processing.
- Retry behavior.
- Dead-letter routing.

Use idempotent business operations where required.

---

# 12. Verify Production Release Artifacts

Before deploying, identify the approved release.

Example:

```yaml
release:
  id: roboshop-2026.11.03
  environment: production

  source:
    repository: roboshop-microservices
    commit: APPROVED_COMMIT_SHA

  services:
    user:
      image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user
      digest: sha256:REAL_DIGEST

    catalog:
      image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/roboshop/catalog
      digest: sha256:REAL_DIGEST

  approvals:
    qa: approved
    staging: approved
    security: approved
    change: approved
```

Do not use placeholder values in an actual release manifest.

## 12.1 Verify ECR Digest

```bash
aws ecr describe-images \
  --repository-name roboshop/user \
  --region "$AWS_REGION" \
  --query 'imageDetails[*].[imageDigest,imageTags]'
```

Confirm the digest matches the approved Staging artifact.

## 12.2 Verify Image Provenance

Where supported, verify:

- Build provenance.
- Artifact signature.
- Image scan results.
- Source commit.
- Build pipeline identity.
- Approved registry.

If image signing is part of the organization’s supply-chain policy, enforce signature verification before deployment.

---

# 13. Production Deployment Strategy

Choose a deployment strategy based on application architecture, risk, and availability requirements.

Common approaches:

| Strategy | Description | Consideration |
|---|---|---|
| Rolling | Gradually replaces pods | Simple, but old and new versions coexist |
| Blue-Green | Runs two environments and switches traffic | Requires additional capacity |
| Canary | Sends limited traffic to new version | Requires traffic control and metrics |
| Recreate | Stops old version before new version | Causes downtime |

For this chapter, use Rolling Update as the baseline deployment method.

Canary and Blue-Green deployment are covered as additional strategies later in the chapter.

---

# 14. Configure Rolling Updates

Example Deployment:

`user-production-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user
  namespace: production
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: user

  template:
    metadata:
      labels:
        app: user

    spec:
      serviceAccountName: user-service-account

      containers:
        - name: user

          image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:REAL_DIGEST

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: 250m
              memory: 256Mi

            limits:
              cpu: 1000m
              memory: 1Gi

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 10
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            periodSeconds: 20
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health/startup
              port: 8080
            periodSeconds: 5
            failureThreshold: 30
```

Replace the example image digest and probe paths with the actual approved release values.

## 14.1 Understand maxSurge

```yaml
maxSurge: 1
```

Kubernetes may temporarily create one additional pod beyond the desired replica count.

For four replicas, the Deployment may temporarily run five pods.

## 14.2 Understand maxUnavailable

```yaml
maxUnavailable: 0
```

Kubernetes should not intentionally reduce the available replica count below the desired count during the rolling update.

This requires sufficient cluster capacity and correctly functioning readiness probes.

## 14.3 Verify

```bash
kubectl rollout status deployment/user \
  -n production \
  --timeout=600s
```

Check:

```bash
kubectl get pods -n production \
  -l app=user -o wide
```

---

# 15. Readiness, Liveness, and Startup Probes

These probes serve different purposes.

| Probe | Purpose |
|---|---|
| Startup | Determines whether application startup completed |
| Readiness | Determines whether the pod can receive traffic |
| Liveness | Determines whether the container should be restarted |

## 15.1 Readiness Failure

When readiness fails:

- Pod remains running.
- Pod is removed from ready Service endpoints.
- Traffic should stop being routed to that pod.
- Kubernetes continues checking readiness.

## 15.2 Liveness Failure

When liveness fails repeatedly:

- Kubernetes restarts the container.
- The container may enter a restart loop if the failure continues.

## 15.3 Startup Probe

Startup probes protect slow-starting applications from premature liveness restarts.

Use probe settings based on measured startup time.

Do not use an external database dependency as a liveness condition without careful design. A temporary database outage should not cause every application container to restart continuously.

---

# 16. Configure Pod Distribution

Production replicas should be distributed across nodes and Availability Zones.

Example topology spread constraint:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: user
```

Also consider node-level anti-affinity when appropriate.

Validate placement:

```bash
kubectl get pods -n production \
  -l app=user \
  -o wide
```

Review:

- Node placement.
- Availability Zone.
- Resource availability.
- Replica distribution.

If the cluster lacks enough capacity or eligible zones, strict spread rules may leave pods Pending. Validate capacity before rollout.

---

# 17. Configure Production HPA

Example:

`user-production-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user

  minReplicas: 4
  maxReplicas: 20

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

Apply through GitOps.

Validate:

```bash
kubectl get hpa -n production
```

```bash
kubectl describe hpa user-hpa \
  -n production
```

Check resource metrics:

```bash
kubectl top pods -n production
```

Set the minimum and maximum replica counts based on capacity planning, load testing, and business requirements.

---

# 18. Configure Pod Disruption Budgets

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-pdb
  namespace: production
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: user
```

Apply:

```bash
kubectl apply -f user-pdb.yaml
```

Validate:

```bash
kubectl get pdb -n production
```

A PDB helps control voluntary disruptions, such as node maintenance.

It does not guarantee availability during hardware failures, application crashes, or other involuntary disruptions.

---

# 19. Production GitOps Promotion

All production deployment configuration should be reviewed and stored in Git.

Example repository:

```text
roboshop-gitops/
|
+-- applications/
|   |
|   +-- dev/
|   +-- qa/
|   +-- staging/
|   +-- production/
|       |
|       +-- user.yaml
|       +-- catalog.yaml
|       +-- cart.yaml
|       +-- orders.yaml
|       +-- payment.yaml
|       +-- inventory.yaml
|       +-- notification.yaml
|
+-- projects/
|
+-- policies/
```

## 19.1 Production Promotion PR

The promotion PR should update only the approved production release configuration.

Review:

- Image digests.
- Helm values.
- Resource limits.
- Replica counts.
- Environment configuration.
- ServiceAccount references.
- Secrets references.
- Ingress changes.
- Database migration references.
- Network policies.

## 19.2 Branch Protection

Require:

- Pull request review.
- Required CI checks.
- Restricted merge permissions.
- Protected production configuration.
- Auditable approvals.

Avoid direct pushes to the production deployment branch.

---

# 20. Configure Argo CD Production Project

Example:

`roboshop-production-project.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: roboshop-production
  namespace: argocd
spec:
  description: RoboShop Production

  sourceRepos:
    - https://github.com/ORG/roboshop-helm.git

  destinations:
    - namespace: production
      server: https://kubernetes.default.svc

  namespaceResourceWhitelist:
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: ""
      kind: Service
    - group: apps
      kind: Deployment
    - group: autoscaling
      kind: HorizontalPodAutoscaler
    - group: networking.k8s.io
      kind: NetworkPolicy
```

Add only the resource types required by the actual application.

Do not allow unrestricted cluster-scoped resources unless explicitly approved.

Apply:

```bash
kubectl apply -f roboshop-production-project.yaml
```

## 20.1 Production Application

Example:

`user-production-application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: roboshop-user-production
  namespace: argocd
spec:
  project: roboshop-production

  source:
    repoURL: https://github.com/ORG/roboshop-helm.git
    targetRevision: main
    path: charts/user

    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

For production, use a controlled sync process that requires the appropriate release approval.

Do not enable automatic sync merely for convenience if the organization's release policy requires manual approval.

Apply:

```bash
kubectl apply -f user-production-application.yaml
```

---

# 21. Execute Production Deployment

Once the production promotion PR is approved and merged:

1. Confirm the approved Git revision.
2. Confirm the production image digests.
3. Confirm the deployment window.
4. Confirm on-call coverage.
5. Confirm rollback readiness.
6. Trigger the approved Argo CD sync process.

Check application status:

```bash
argocd app get roboshop-user-production
```

Trigger sync only through the approved deployment process:

```bash
argocd app sync roboshop-user-production
```

Wait for health:

```bash
argocd app wait roboshop-user-production \
  --sync \
  --health \
  --timeout 600
```

Verify Deployment:

```bash
kubectl get deployment user \
  -n production
```

Verify pods:

```bash
kubectl get pods \
  -n production \
  -l app=user
```

Verify image:

```bash
kubectl get deployment user \
  -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Repeat for every approved microservice.

For a 20+ microservice application, use a controlled release inventory and service dependency order. Do not deploy all services blindly in parallel if there are database, event-contract, or API compatibility dependencies.

---

# 22. Deploy in Controlled Batches

For a large microservices platform, use batches.

Example:

```text
BATCH 1 — FOUNDATION
  |
  +--> Shared configuration
  +--> Service accounts
  +--> Required infrastructure integrations

BATCH 2 — CORE SERVICES
  |
  +--> User
  +--> Catalog
  +--> Cart

BATCH 3 — TRANSACTION SERVICES
  |
  +--> Orders
  +--> Payment
  +--> Inventory

BATCH 4 — ASYNC SERVICES
  |
  +--> Notification
  +--> Other consumers

BATCH 5 — USER EXPERIENCE
  |
  +--> Frontend
  +--> API routing
```

This is an example sequence, not a guaranteed dependency graph. Derive the real deployment order from the service architecture and compatibility requirements.

After each batch:

- Verify Argo CD health.
- Verify pod readiness.
- Verify logs.
- Verify service endpoints.
- Run relevant smoke tests.
- Review alerts.
- Obtain the required go/no-go decision.

---

# 23. Zero-Downtime Deployment Considerations

A rolling update alone does not guarantee zero downtime.

Zero-downtime behavior depends on:

- Multiple healthy replicas.
- Correct readiness probes.
- Sufficient cluster capacity.
- Service endpoint updates.
- Graceful shutdown.
- Connection draining.
- Backward-compatible application changes.
- Database compatibility.
- Load balancer health checks.

## 23.1 Graceful Shutdown

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 60

  containers:
    - name: user
      image: REPLACE_WITH_APPROVED_IMAGE

      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - sleep 10
```

This example allows a short delay before container termination. It is not a substitute for application-level graceful shutdown.

The application should:

1. Stop accepting new work.
2. Finish or safely cancel in-flight requests.
3. Stop consuming messages when appropriate.
4. Close database connections.
5. Exit within the termination grace period.

The delay should be aligned with actual load balancer deregistration, endpoint propagation, and application shutdown behavior.

## 23.2 Connection Draining

Verify the load balancer target group configuration.

Ensure old pods are not terminated before traffic has been drained appropriately.

Test long-running requests and WebSocket connections if the application uses them.

---

# 24. Canary Deployment Strategy

Canary deployment sends a small percentage of traffic to a new application version before full rollout.

Example:

```text
Existing Version: 100%
       |
       v
Deploy New Version
       |
       v
Canary Traffic: 5%
       |
       v
Validate Metrics
       |
       +--> Errors Increased?
       |       |
       |       +--> YES: Stop and Roll Back
       |       |
       |       +--> NO
       |
       v
Canary Traffic: 25%
       |
       v
Validate Again
       |
       v
Canary Traffic: 50%
       |
       v
Validate Again
       |
       v
Canary Traffic: 100%
```

Canary traffic shifting requires a compatible traffic-management mechanism, such as an appropriately configured AWS load balancer integration or service-mesh rollout controller.

Simply deploying two Kubernetes Deployments does not automatically create percentage-based traffic splitting.

## 24.1 Canary Validation Metrics

Monitor:

- HTTP 5xx rate.
- p95 latency.
- p99 latency.
- CPU and memory.
- Pod restarts.
- Business transaction success rate.
- Database errors.
- Message processing errors.

Define stop thresholds before deployment.

## 24.2 Canary Rollback

If the new version violates the approved thresholds:

1. Stop traffic shifting.
2. Route traffic back to the stable version.
3. Confirm error rates return to baseline.
4. Preserve logs and metrics.
5. Open an incident or defect.
6. Investigate before retrying.

---

# 25. Blue-Green Deployment Strategy

Blue-Green deployment maintains two versions.

```text
              Production Traffic
                      |
                      v
                 Load Balancer
                      |
               Traffic Switch
                 /        \
                /          \
          BLUE VERSION   GREEN VERSION
          Current         New Release
```

Deployment steps:

1. Keep Blue serving Production traffic.
2. Deploy Green with the approved image.
3. Validate Green internally.
4. Run smoke and integration tests.
5. Verify database compatibility.
6. Switch traffic to Green.
7. Monitor Production metrics.
8. Keep Blue available for the agreed rollback window.
9. Retire Blue after successful sign-off.

Blue-Green requires additional capacity and careful handling of sessions, asynchronous processing, and shared database state.

Do not switch traffic if Green has not passed readiness and application-level validation.

---

# 26. Production Ingress and DNS Validation

Check ingress:

```bash
kubectl get ingress -n production
```

Describe:

```bash
kubectl describe ingress \
  roboshop-production-ingress \
  -n production
```

Check ALB health:

- Target group health.
- Listener configuration.
- Security groups.
- TLS certificate.
- Routing rules.
- Health-check path.

Check DNS:

```bash
dig roboshop.example.com
```

Check HTTPS:

```bash
curl -I https://roboshop.example.com
```

Check response:

```bash
curl -sS \
  -o /dev/null \
  -w 'HTTP %{http_code} | Total %{time_total}s\n' \
  https://roboshop.example.com/
```

Verify the expected hostname and certificate. Do not disable TLS validation to make a failing test pass.

---

# 27. Production Smoke Tests

Run a limited, approved smoke-test suite after each deployment batch.

Example:

```bash
#!/bin/bash

set -euo pipefail

BASE_URL="https://roboshop.example.com"

echo "Checking frontend..."

curl --fail --silent --show-error \
  "$BASE_URL/"

echo "Checking product API..."

curl --fail --silent --show-error \
  "$BASE_URL/api/products"

echo "Production smoke tests passed."
```

Adapt endpoints to the real application.

Smoke tests should avoid:

- Creating unintended customer transactions.
- Triggering real payments.
- Sending real customer notifications.
- Deleting business data.
- Performing destructive operations.

Use approved test accounts and safe test workflows.

---

# 28. Production Business Transaction Validation

Validate critical business workflows using safe test transactions.

Example:

```text
Customer Authentication
        |
        v
Product Search
        |
        v
Cart Creation
        |
        v
Order Creation
        |
        v
Payment Authorization
        |
        v
Inventory Reservation
        |
        v
Order Confirmation
        |
        v
Notification
```

Verify:

- Correct response.
- Correct order status.
- Correct inventory state.
- Correct payment test result.
- Correct event processing.
- No duplicate business operation.
- No unexpected database changes.

Business transaction validation is more meaningful than simply checking whether a pod is Running.

---

# 29. Production Monitoring

Use Prometheus and Grafana for operational visibility.

## 29.1 Kubernetes Monitoring

Monitor:

- Node availability.
- Node CPU.
- Node memory.
- Pod CPU.
- Pod memory.
- Pod restarts.
- Pending pods.
- Deployment availability.
- HPA behavior.
- Persistent volume health.

## 29.2 Application Monitoring

Monitor:

- Request rate.
- Error rate.
- Latency.
- HTTP status distribution.
- Application exceptions.
- Database connection pool.
- Cache hit ratio.
- Messaging backlog.
- Consumer failures.

## 29.3 Example PromQL

Pod CPU usage:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="production",
    container!=""
  }[5m])
)
```

Pod memory:

```promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="production",
    container!=""
  }
)
```

Restart count:

```promql
sum by (pod) (
  increase(kube_pod_container_status_restarts_total{
    namespace="production"
  }[15m])
)
```

Deployment availability:

```promql
kube_deployment_status_replicas_available{
  namespace="production"
}
```

The exact metrics depend on the installed Prometheus exporters and metric labels.

---

# 30. Production Logging

Use ELK Stack for centralized application and infrastructure logs.

Validate:

- Logstash receives application logs.
- Logs are parsed correctly.
- Elasticsearch indexing succeeds.
- Kibana dashboards are available.
- Environment labels identify Production.
- Correlation IDs are preserved.
- Timestamps are correct.
- Sensitive values are masked.

Example log:

```json
{
  "timestamp": "2026-11-03T12:00:00Z",
  "level": "ERROR",
  "service": "orders",
  "environment": "production",
  "message": "Payment request failed",
  "correlation_id": "example-correlation-id",
  "error_code": "PAYMENT_PROVIDER_TIMEOUT"
}
```

Never expose passwords, tokens, payment details, or personal customer information in logs.

---

# 31. Production Alerting

Configure and validate alerts.

| Alert | Suggested condition |
|---|---|
| ServiceDown | Required service unavailable |
| High5xxRate | HTTP 5xx rate exceeds approved threshold |
| HighLatency | p95 latency exceeds SLO |
| PodCrashLoop | Repeated container restarts |
| NodeNotReady | Production node unavailable |
| HighCPU | Sustained CPU saturation |
| HighMemory | Memory pressure |
| DatabaseConnectionsHigh | Connection pool nearing limit |
| DatabaseUnavailable | Database health failure |
| QueueBacklogHigh | Messaging backlog increasing |
| ConsumerLagHigh | Kafka consumer lag exceeds threshold |
| DeploymentUnavailable | Required replicas unavailable |

Define thresholds based on actual service-level objectives and historical baselines.

Every critical alert must have:

- Owner.
- Severity.
- Description.
- Runbook.
- Escalation path.
- Notification channel.

---

# 32. Production Incident Response

If a deployment causes an incident, follow a structured response process.

## 32.1 Incident Workflow

```text
Alert Triggered
      |
      v
Incident Declared
      |
      v
Assign Incident Commander
      |
      v
Assess Customer Impact
      |
      v
Stabilize Service
      |
      +--> Rollback
      |
      +--> Traffic Shift
      |
      +--> Disable Faulty Feature
      |
      v
Validate Recovery
      |
      v
Communicate Status
      |
      v
Collect Evidence
      |
      v
Root Cause Analysis
      |
      v
Preventive Actions
```

## 32.2 First Response Checklist

1. Identify affected services.
2. Identify deployment time.
3. Compare error rates before and after deployment.
4. Review Argo CD changes.
5. Inspect Kubernetes events.
6. Review application logs.
7. Check database health.
8. Check messaging health.
9. Assess customer impact.
10. Decide whether rollback is required.

Avoid making multiple untracked changes during an incident.

Record every mitigation action.

---

# 33. Production Rollback Strategy

Rollback must be planned before deployment.

For GitOps, the preferred method is to restore the last known-good Git configuration.

## 33.1 Identify the Current Release

```bash
argocd app get roboshop-user-production
```

Review history:

```bash
argocd app history roboshop-user-production
```

## 33.2 Identify the Previous Approved Digest

Check the release manifest and Git history.

Confirm that the previous image is still available in ECR and that it is compatible with the current database schema.

## 33.3 Revert the Production Promotion Commit

```bash
git revert <PRODUCTION_PROMOTION_COMMIT>
```

Create and approve the rollback PR according to emergency change policy.

## 33.4 Sync the Restored State

```bash
argocd app sync roboshop-user-production
```

Wait for health:

```bash
argocd app wait roboshop-user-production \
  --sync \
  --health \
  --timeout 600
```

## 33.5 Validate

```bash
kubectl rollout status deployment/user \
  -n production
```

```bash
kubectl get pods -n production
```

Verify:

- Previous image digest.
- Healthy replicas.
- Service endpoints.
- Ingress health.
- Error rate.
- Application smoke tests.
- Business transaction results.

## 33.6 Emergency Rollback with Kubernetes

If the approved emergency procedure permits it, a Deployment can be rolled back:

```bash
kubectl rollout history deployment/user \
  -n production
```

```bash
kubectl rollout undo deployment/user \
  -n production
```

This changes the live Kubernetes object.

Argo CD may restore the Git-defined state afterward. Therefore, emergency rollback must also reconcile Git to the approved desired state.

Do not rely on `kubectl rollout undo` as the only GitOps rollback procedure.

## 33.7 Database Rollback Warning

Application rollback and database rollback are separate operations.

If a migration is backward compatible, the old application may continue to operate.

If a migration is destructive or incompatible, restoring the old image may not restore application functionality.

Database restoration must follow the approved recovery plan and must account for data created after the backup.

---

# 34. Production Backup and Disaster Recovery

Production recovery must be planned independently of application deployment.

Review:

- RDS automated backups.
- Point-in-time recovery.
- Snapshot retention.
- Cross-region backup requirements.
- EBS volume backup requirements.
- Kubernetes configuration stored in Git.
- Terraform state backup and access controls.
- ECR image retention.
- Secrets recovery.
- DNS recovery.
- Messaging recovery.
- Recovery Point Objective (RPO).
- Recovery Time Objective (RTO).

## 34.1 RPO

Recovery Point Objective describes the maximum acceptable amount of data loss measured in time.

## 34.2 RTO

Recovery Time Objective describes the target time to restore service.

Example targets must be agreed by the business and validated through recovery exercises.

Do not claim an RTO or RPO has been achieved without testing it.

---

# 35. Production Validation Script

Create:

`validate-production.sh`

```bash
#!/bin/bash

set -uo pipefail

NAMESPACE="production"
FAILED=0

check() {
    local description="$1"
    shift

    echo
    echo "Checking: $description"

    if "$@"; then
        echo "PASS: $description"
    else
        echo "FAIL: $description"
        FAILED=1
    fi
}

check "Production namespace" \
    kubectl get namespace "$NAMESPACE"

check "Deployments" \
    kubectl get deployments -n "$NAMESPACE"

check "Pods" \
    kubectl get pods -n "$NAMESPACE"

check "Services" \
    kubectl get services -n "$NAMESPACE"

check "Ingress" \
    kubectl get ingress -n "$NAMESPACE"

check "HPA" \
    kubectl get hpa -n "$NAMESPACE"

check "PDB" \
    kubectl get pdb -n "$NAMESPACE"

echo
echo "Deployment summary:"
kubectl get deployments -n "$NAMESPACE"

echo
echo "Pod summary:"
kubectl get pods -n "$NAMESPACE"

echo
echo "Recent events:"
kubectl get events -n "$NAMESPACE" \
    --sort-by='.lastTimestamp' | tail -20

if [ "$FAILED" -eq 0 ]; then
    echo
    echo "Basic Production checks completed."
    exit 0
else
    echo
    echo "Production checks reported failures."
    exit 1
fi
```

Run:

```bash
chmod +x validate-production.sh
./validate-production.sh
```

This is a basic resource validation script. A production release gate must additionally verify:

- Deployment availability.
- Image digests.
- Application health.
- Service endpoints.
- Ingress health.
- Security checks.
- Business smoke tests.
- Monitoring and alerts.

---

# 36. Production Deployment Evidence

Create a release-specific evidence directory.

```text
production-evidence/
|
+-- release-metadata.yaml
+-- change-request.txt
+-- approvals/
+-- git-commit.txt
+-- image-digests.txt
+-- infrastructure-validation/
+-- argocd-status/
+-- deployment-status/
+-- pod-status/
+-- ingress-validation/
+-- smoke-tests/
+-- integration-tests/
+-- security-reports/
+-- performance-reports/
+-- monitoring/
+-- incident-records/
+-- rollback-record/
+-- final-signoff/
```

Capture deployment status:

```bash
mkdir -p production-evidence
```

```bash
kubectl get deployments -n production -o wide \
  > production-evidence/deployment-status.txt
```

```bash
kubectl get pods -n production -o wide \
  > production-evidence/pod-status.txt
```

```bash
kubectl get services -n production \
  > production-evidence/service-status.txt
```

```bash
kubectl get ingress -n production \
  > production-evidence/ingress-status.txt
```

```bash
kubectl get events -n production \
  --sort-by='.lastTimestamp' \
  > production-evidence/events.txt
```

```bash
argocd app list \
  > production-evidence/argocd-status.txt
```

Store evidence securely and ensure it contains no secrets or sensitive customer information.

---

# 37. Production Go/No-Go Decision

Before declaring the release successful, review the release criteria.

## Infrastructure

- [ ] Production cluster healthy.
- [ ] Nodes healthy.
- [ ] Required add-ons healthy.
- [ ] Networking operational.
- [ ] Load balancer healthy.
- [ ] Database healthy.
- [ ] Redis healthy.
- [ ] Messaging healthy.

## Deployment

- [ ] Approved image digests deployed.
- [ ] All required services available.
- [ ] Argo CD synchronized.
- [ ] No unexpected drift.
- [ ] Readiness checks passing.
- [ ] No abnormal restart patterns.

## Application

- [ ] Frontend accessible.
- [ ] Authentication works.
- [ ] Product APIs work.
- [ ] Cart workflow works.
- [ ] Order workflow works.
- [ ] Payment integration works in the approved mode.
- [ ] Inventory updates work.
- [ ] Notifications work.

## Operations

- [ ] Prometheus targets healthy.
- [ ] Grafana dashboards available.
- [ ] ELK logs available.
- [ ] Critical alerts tested.
- [ ] On-call team available.
- [ ] Incident escalation confirmed.
- [ ] Rollback plan available.

## Approval

- [ ] QA approval.
- [ ] Security approval.
- [ ] Application owner approval.
- [ ] DevOps approval.
- [ ] Change approval.
- [ ] Production sign-off.

---

# 38. Production Sign-Off Record

Example:

```yaml
release:
  id: roboshop-2026.11.03
  environment: production

deployment:
  strategy: rolling-update
  status: successful

validation:
  infrastructure: passed
  application: passed
  security: passed
  smoke_tests: passed
  integration_tests: passed
  monitoring: passed
  rollback_readiness: verified

approvals:
  application_owner: approved
  qa_owner: approved
  devops_owner: approved
  security_owner: approved
  change_manager: approved

final_status: production-approved
```

The actual release record should contain approver identities, timestamps, evidence references, and approved exceptions.

---

# 39. Production Deployment Troubleshooting

## 39.1 Pods in ImagePullBackOff

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

Investigate:

- Image reference.
- ECR repository.
- Digest.
- IAM permissions.
- Network access.
- Registry availability.

## 39.2 CrashLoopBackOff

```bash
kubectl logs <POD_NAME> \
  -n production
```

```bash
kubectl logs <POD_NAME> \
  -n production \
  --previous
```

Investigate:

- Application startup.
- Missing configuration.
- Secrets.
- Database connectivity.
- Memory limits.
- Health probes.

## 39.3 Deployment Stuck

```bash
kubectl describe deployment user \
  -n production
```

```bash
kubectl get replicasets \
  -n production
```

```bash
kubectl get pods \
  -n production \
  -l app=user
```

Possible causes:

- Insufficient capacity.
- Readiness failures.
- Image pull errors.
- Quota restrictions.
- Scheduling constraints.

## 39.4 Production 503

```bash
kubectl get endpointslices \
  -n production
```

```bash
kubectl get pods \
  -n production
```

```bash
kubectl describe ingress \
  roboshop-production-ingress \
  -n production
```

Investigate:

- Missing ready endpoints.
- Target group health.
- Service selectors.
- Backend ports.
- Application health.
- Load balancer configuration.

## 39.5 Argo CD OutOfSync

```bash
argocd app get roboshop-user-production
```

Review:

- Git revision.
- Helm rendering.
- Live resource differences.
- Permissions.
- Sync errors.
- Unexpected manual changes.

Do not automatically overwrite live changes without understanding their impact.

---

# 40. Real Production Incident Example

## Incident: Production Deployment Causes HTTP 503

### Situation

A new User Service version is deployed.

Argo CD reports that the application is synchronized, but users begin receiving HTTP 503 errors.

### Step 1: Identify the Impact

Prometheus shows increased HTTP 5xx errors.

Grafana shows:

- Increased error rate.
- Reduced ready replicas.
- Increased pod restarts.

### Step 2: Check Kubernetes

```bash
kubectl get pods -n production \
  -l app=user
```

Some pods are not Ready.

### Step 3: Check Events

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

The readiness probe is failing.

### Step 4: Check Logs

```bash
kubectl logs <POD_NAME> \
  -n production
```

The logs show an application startup error related to an incorrect database configuration.

### Step 5: Stop Further Rollout

Pause the rollout through the approved deployment process.

Do not continue promoting the faulty release.

### Step 6: Restore Previous Configuration

Revert the production Git promotion commit to the last known-good image digest and configuration.

### Step 7: Validate

Confirm:

- Pods become Ready.
- Service endpoints recover.
- ALB targets become healthy.
- HTTP 5xx rate returns to baseline.
- Smoke tests pass.

### Step 8: RCA

Example root cause:

The new application version expected a configuration property that was missing from the Production ConfigMap.

### Preventive Actions

- Add configuration validation to CI.
- Validate environment-specific Helm values.
- Add startup configuration checks.
- Run a production-like configuration test in Staging.
- Add a release gate for required environment variables.
- Improve deployment health monitoring.

This is an illustrative incident. Actual root cause must be established using incident evidence.

---

# 41. Final Production Deployment Workflow

```text
1. Staging Approval
       |
       v
2. Create Production Change Request
       |
       v
3. Review Release Manifest
       |
       v
4. Verify Image Digests
       |
       v
5. Validate Production Infrastructure
       |
       v
6. Verify Database and Messaging
       |
       v
7. Approve Deployment Window
       |
       v
8. Create Production GitOps PR
       |
       v
9. Review and Merge
       |
       v
10. Argo CD Sync
       |
       v
11. Deploy in Controlled Batches
       |
       v
12. Validate Pods and Services
       |
       v
13. Run Smoke Tests
       |
       v
14. Validate Business Transactions
       |
       v
15. Monitor Production Metrics
       |
       v
16. Confirm Stability
       |
       v
17. Record Production Sign-Off
```

---

# 42. Key Production Engineering Principles

1. Build once and promote the same immutable artifact.
2. Keep Production isolated from lower environments.
3. Use least-privilege IAM and Kubernetes RBAC.
4. Store secrets outside Git.
5. Use backward-compatible database migrations.
6. Deploy through reviewed GitOps changes.
7. Use readiness probes correctly.
8. Distribute replicas across failure domains.
9. Ensure graceful application shutdown.
10. Monitor business transactions, not just pod status.
11. Define deployment stop conditions in advance.
12. Keep rollback procedures tested and documented.
13. Record production changes and evidence.
14. Avoid manual, untracked Production changes.
15. Treat incidents as opportunities for systematic improvement.

---

# 43. Chapter Completion Checklist

- [ ] Production architecture reviewed.
- [ ] Production AWS account verified.
- [ ] EKS cluster validated.
- [ ] Production namespace configured.
- [ ] ResourceQuota and LimitRange configured.
- [ ] RBAC reviewed.
- [ ] Workload identity validated.
- [ ] Secrets integration validated.
- [ ] Database readiness verified.
- [ ] Messaging and Redis validated.
- [ ] Release image digests verified.
- [ ] Production change approved.
- [ ] Argo CD project configured.
- [ ] Production Application configured.
- [ ] Rolling update validated.
- [ ] Readiness and liveness probes validated.
- [ ] Pod distribution validated.
- [ ] HPA and PDB configured.
- [ ] Ingress and DNS validated.
- [ ] Smoke tests passed.
- [ ] Business workflows passed.
- [ ] Prometheus and Grafana validated.
- [ ] ELK logging validated.
- [ ] Alerts validated.
- [ ] Rollback procedure reviewed.
- [ ] Production sign-off recorded.

---

# 44. Chapter Summary

This chapter covered the Production deployment of the RoboShop microservices platform.

The implementation included:

- Production architecture and isolation.
- Release governance.
- AWS EKS validation.
- Kubernetes access control.
- IAM and workload identity.
- Production secrets management.
- Database and messaging readiness.
- Immutable ECR artifact promotion.
- Rolling update deployment.
- Readiness, liveness, and startup probes.
- Pod distribution and high availability.
- HPA and PDB.
- Argo CD GitOps promotion.
- Controlled microservice deployment batches.
- Zero-downtime deployment considerations.
- Canary and Blue-Green strategies.
- Ingress and DNS validation.
- Production smoke and business tests.
- Prometheus, Grafana, and ELK monitoring.
- Alerting and incident response.
- Rollback and disaster recovery.
- Evidence collection and Production sign-off.

**Final principle:** A Production deployment is successful only when the application is available, business workflows are functioning, security requirements are met, observability is working, and the team can recover safely if something goes wrong.

---

# 45. Next Chapter

**CHAPTER 204 — PRODUCTION RELEASE MANAGEMENT AND OPERATIONS**

The next chapter should cover:

- Release versioning and release calendars.
- Production deployment windows.
- Change-management workflows.
- Release freeze.
- Hotfix releases.
- Emergency deployment procedures.
- Feature flags.
- Progressive delivery.
- Post-deployment verification.
- Production support handover.
- Operational runbooks.
- Release metrics.
- Deployment frequency.
- Change failure rate.
- Mean time to recovery.
- Continuous improvement.


# CHAPTER 205 — PRODUCTION DEPLOYMENT

**Filename:** `205-Production-Deployment.md`

---

# 1. Chapter Objective

In the previous chapters, we completed the release preparation and approval process.

| Chapter | Activity |
|---|---|
| 199 | GitOps Promotion |
| 200 | DEV Deployment |
| 201 | QA Deployment |
| 202 | Staging Deployment |
| 203 | Jira Production Change |
| 204 | Production Approval |
| 205 | Production Deployment |

Now we will deploy the approved application release into the Production environment.

We will implement a practical deployment process for our RoboShop microservices platform running on AWS EKS.

The deployment process must ensure:

- Only an approved release reaches Production.
- The same tested container images are promoted.
- Image tags and digests are verified.
- Production configuration is correct.
- Secrets are retrieved securely.
- Database connectivity is verified.
- Redis and messaging dependencies are ready.
- Kubernetes resources are healthy.
- Argo CD deploys the approved GitOps revision.
- Application functionality is validated.
- Monitoring and logging are operational.
- Deployment evidence is retained.

**Important:** This chapter focuses on the standard Production deployment. Rolling and canary deployment strategies are covered separately in Chapter 206.

# 2. Production Deployment Architecture

Our Production deployment uses AWS EKS, Amazon ECR, Helm, Argo CD, and GitOps.

```text
                 DEVELOPER
                     |
                     v
                Git Repository
                     |
                     v
                 CI Pipeline
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Build      SonarQube    Trivy
          |          |          |
          +----------+----------+
                     |
                     v
                Docker Image
                     |
                     v
                 Amazon ECR
                     |
                     v
             DEV Deployment
                     |
                     v
              QA Deployment
                     |
                     v
           Staging Deployment
                     |
                     v
             Jira Change
                     |
                     v
          Production Approval
                     |
                     v
           GitOps Production PR
                     |
                     v
              GitOps Repository
                     |
                     v
                  Argo CD
                     |
                     v
              Production EKS
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Services    Database   Messaging
          |          |          |
          +----------+----------+
                     |
                     v
               ALB / Ingress
                     |
                     v
                   Users
                     |
                     v
              Monitoring Stack
          Prometheus / Grafana / ELK
```

# 3. Production Environment Architecture

We maintain separate environments.

```text
AWS Account
    |
    +-------------------------+
    |                         |
    v                         v
Non-Production Account    Production Account
    |                         |
    +-- DEV EKS               +-- Production EKS
    +-- QA EKS                +-- Production VPC
    +-- Staging EKS           +-- Production RDS
                              +-- Production Redis
                              +-- Production Messaging
                              +-- Production ALB
                              +-- Production DNS
```

This is an example architecture.

Production may use a dedicated AWS account, while DEV, QA, and Staging may share a non-production account or use separate accounts.

The exact account separation must follow the organization's security and operational policies.

## 3.1 Example Production Configuration

```yaml
environment: production

aws:
  region: ap-south-1
  account_id: "123456789012"

eks:
  cluster_name: roboshop-production-eks

namespace: production

registry:
  url: 123456789012.dkr.ecr.ap-south-1.amazonaws.com

gitops:
  repository: roboshop-gitops
  branch: main

deployment:
  method: argocd
  strategy: rolling-update
```

Use your actual AWS account, cluster, namespace, repository, and region.

Do not commit sensitive credentials into this file.

# 4. Production GitOps Repository Structure

Our application source code and Production deployment configuration are maintained separately.

Example GitOps repository:

```text
roboshop-gitops/
│
├── applications/
│   ├── dev/
│   │   ├── user.yaml
│   │   ├── cart.yaml
│   │   └── catalog.yaml
│   │
│   ├── qa/
│   │   ├── user.yaml
│   │   ├── cart.yaml
│   │   └── catalog.yaml
│   │
│   ├── staging/
│   │   ├── user.yaml
│   │   ├── cart.yaml
│   │   └── catalog.yaml
│   │
│   └── production/
│       ├── user.yaml
│       ├── cart.yaml
│       ├── catalog.yaml
│       ├── orders.yaml
│       ├── payment.yaml
│       └── root-app.yaml
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── production/
│       ├── user-values.yaml
│       ├── cart-values.yaml
│       ├── catalog-values.yaml
│       ├── orders-values.yaml
│       ├── payment-values.yaml
│       └── global-values.yaml
│
├── charts/
│   ├── user/
│   ├── cart/
│   ├── catalog/
│   ├── orders/
│   └── payment/
│
└── README.md
```

The actual repository should include all deployed microservices and shared platform components.

Do not assume that every microservice has the same values or dependencies.

# 5. Production Deployment Prerequisites

Before deploying, verify the following.

## 5.1 Change Approval

Verify:

- Jira change is approved.
- Production deployment window is valid.
- Required approvers have approved the release.
- No outstanding approval conditions remain.
- The deployment owner is assigned.

Example:

```text
Jira Change: ROBOSHOP-450
Release: roboshop-2026.11.03
Environment: Production
Approval: Approved
Deployment Window: Confirmed
```

## 5.2 Release Artifact

Verify:

- The release was deployed successfully in Staging.
- The image digest is recorded.
- The image belongs to the approved release.
- Security evidence is available.
- The image has not been rebuilt or replaced after approval.

## 5.3 Operational Readiness

Verify:

- Monitoring is available.
- Alerting is configured.
- Logging is operational.
- On-call engineers are available.
- Rollback procedures are documented.
- Required dependencies are healthy.

# 6. Production AWS Account Verification

First, confirm the AWS identity.

```bash
aws sts get-caller-identity
```

Expected output format:

```json
{
  "UserId": "AIDAXXXXXXXXXXXXX",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/ProductionDeploymentRole/session"
}
```

The example account and identity are placeholders.

Validate the account:

```bash
EXPECTED_ACCOUNT="123456789012"

ACTUAL_ACCOUNT=$(aws sts get-caller-identity \
  --query Account \
  --output text)

if [ "$ACTUAL_ACCOUNT" != "$EXPECTED_ACCOUNT" ]; then
    echo "ERROR: Incorrect AWS account"
    exit 1
fi

echo "Production AWS account verified"
```

Never continue if the identity check fails.

## 6.1 Verify AWS Region

```bash
aws configure get region
```

Or explicitly set the region in deployment commands:

```bash
export AWS_REGION="ap-south-1"
```

Verify:

```bash
aws ec2 describe-availability-zones \
  --region "$AWS_REGION"
```

# 7. Production EKS Cluster Access

Configure the Production cluster context.

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name roboshop-production-eks \
  --alias roboshop-production
```

Verify the current context:

```bash
kubectl config current-context
```

Expected:

```text
roboshop-production
```

Verify cluster connectivity:

```bash
kubectl cluster-info
```

Verify nodes:

```bash
kubectl get nodes -o wide
```

Expected node status:

```text
NAME                         STATUS   ROLES    AGE
ip-10-0-1-101.ec2.internal   Ready    <none>   45d
ip-10-0-2-102.ec2.internal   Ready    <none>   45d
ip-10-0-3-103.ec2.internal   Ready    <none>   45d
```

These are illustrative values.

## 7.1 Add a Production Context Safety Check

Create:

`production-context-check.sh`

```bash
#!/bin/bash

set -euo pipefail

EXPECTED_CONTEXT="roboshop-production"

CURRENT_CONTEXT=$(kubectl config current-context)

if [[ "$CURRENT_CONTEXT" != "$EXPECTED_CONTEXT" ]]; then
    echo "ERROR: Wrong Kubernetes context"
    echo "Expected: $EXPECTED_CONTEXT"
    echo "Current:  $CURRENT_CONTEXT"
    exit 1
fi

echo "Production Kubernetes context verified"
```

Run:

```bash
chmod +x production-context-check.sh

./production-context-check.sh
```

This check is a safety measure, not a substitute for checking the AWS account and cluster identity.

# 8. Production Namespace Validation

Check the Production namespace:

```bash
kubectl get namespace production
```

If it does not exist, create it only through the approved infrastructure or GitOps process.

Inspect namespace configuration:

```bash
kubectl describe namespace production
```

Verify resource quotas:

```bash
kubectl get resourcequota -n production
```

Verify limit ranges:

```bash
kubectl get limitrange -n production
```

Verify network policies:

```bash
kubectl get networkpolicy -n production
```

Verify service accounts:

```bash
kubectl get serviceaccounts -n production
```

The namespace must have the required policies and controls before application deployment.

# 9. Production Node and Capacity Validation

Check node resources:

```bash
kubectl describe nodes
```

Check current pod distribution:

```bash
kubectl get pods -n production -o wide
```

Check resource requests and limits:

```bash
kubectl top nodes

kubectl top pods -n production
```

`kubectl top` requires Metrics Server or a compatible metrics API.

Check pending pods:

```bash
kubectl get pods -n production \
  --field-selector=status.phase=Pending
```

Check node conditions:

```bash
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

Before deployment, review:

- Available CPU.
- Available memory.
- Node group capacity.
- Pod scheduling capacity.
- Availability zone distribution.
- Cluster autoscaler or Karpenter readiness, if used.
- Pod disruption budgets.
- Storage capacity.

Do not assume that a healthy node status means sufficient capacity for the release.

# 10. IAM and IRSA Verification

Production workloads should use restricted AWS permissions.

For example:

- User service may need access to a particular secret.
- Notification service may need access to an approved messaging resource.
- Application services should not receive unrestricted AWS permissions.

Inspect service accounts:

```bash
kubectl get serviceaccounts -n production
```

Inspect a service account:

```bash
kubectl describe serviceaccount user-sa \
  -n production
```

For IRSA, verify the role annotation:

```bash
kubectl get serviceaccount user-sa \
  -n production \
  -o yaml
```

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/roboshop-production-user-role
```

The IAM role must trust the correct EKS OIDC provider and service account identity.

Verify the IAM role trust policy and permissions using the organization's approved AWS review process.

Do not attach administrator permissions to application service accounts.

# 11. Production ECR Image Verification

The same tested image should be promoted from Staging to Production.

Do not rebuild the image for Production.

Example image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:APPROVED_IMAGE_DIGEST
```

Verify the ECR repository:

```bash
aws ecr describe-repositories \
  --repository-names roboshop/user \
  --region ap-south-1
```

Verify the image digest:

```bash
aws ecr describe-images \
  --repository-name roboshop/user \
  --region ap-south-1 \
  --query 'imageDetails[*].[imageTags,imageDigest,imagePushedAt]' \
  --output table
```

Confirm the approved digest exists.

## 11.1 Image Digest Verification Script

Create:

`verify-ecr-image.sh`

```bash
#!/bin/bash

set -euo pipefail

AWS_REGION="ap-south-1"
REPOSITORY="roboshop/user"
EXPECTED_DIGEST="${1:?Usage: $0 sha256:APPROVED_DIGEST}"

ACTUAL_DIGEST=$(aws ecr describe-images \
    --repository-name "$REPOSITORY" \
    --region "$AWS_REGION" \
    --query "imageDetails[?imageDigest=='${EXPECTED_DIGEST}'].imageDigest | [0]" \
    --output text)

if [[ "$ACTUAL_DIGEST" != "$EXPECTED_DIGEST" ]]; then
    echo "ERROR: Approved image digest not found"
    exit 1
fi

echo "Approved image digest verified: $ACTUAL_DIGEST"
```

Run:

```bash
chmod +x verify-ecr-image.sh

./verify-ecr-image.sh sha256:APPROVED_IMAGE_DIGEST
```

Replace the placeholder with the actual digest recorded in the approved change.

# 12. Production Configuration Management

Production configuration must be separate from DEV, QA, and Staging.

Example:

`environments/production/user-values.yaml`

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user
  digest: "sha256:APPROVED_IMAGE_DIGEST"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

probes:
  readiness:
    path: /health
    port: 8080
  liveness:
    path: /health
    port: 8080
```

The chart must support the `digest` field and render the image reference as an immutable digest.

Example template expression:

```yaml
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

The actual health endpoint, port, resource sizing, and replica counts must match the application and its load requirements.

# 13. Production Secrets Management

Do not store plaintext passwords, API tokens, database credentials, or private keys in Git.

Use the organization's approved secrets solution.

For example:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Application Pod
```

Example ExternalSecret:

`external-secrets/user-secret.yaml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: user-secrets
  namespace: production
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: production-secret-store
    kind: ClusterSecretStore

  target:
    name: user-secrets
    creationPolicy: Owner

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/roboshop/user
        property: DB_PASSWORD
```

This example assumes that the External Secrets Operator and the referenced ClusterSecretStore have already been configured.

Verify:

```bash
kubectl get externalsecrets -n production

kubectl describe externalsecret user-secrets \
  -n production

kubectl get secrets -n production
```

Do not print secret values in terminal output, pipeline logs, or deployment evidence.

# 14. Production Database Readiness

Before deploying an application release, verify its database dependencies.

For RoboShop, services may use different databases depending on their implementation.

Example dependency map:

| Service | Database or storage |
|---|---|
| User | MongoDB or configured user database |
| Catalog | Catalog database |
| Cart | Redis |
| Orders | Orders database |
| Payment | Payment data store |
| Inventory | Inventory database |

This is an illustrative mapping. Use the actual database architecture from the project.

## 14.1 Verify RDS Status

```bash
aws rds describe-db-instances \
  --region ap-south-1 \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table
```

Expected database status:

```text
available
```

## 14.2 Verify Database Network Connectivity

Run connectivity tests from an authorized diagnostic pod or application environment.

For example:

```bash
nc -vz production-db.example.internal 5432
```

For MySQL:

```bash
nc -vz production-mysql.example.internal 3306
```

A successful TCP connection does not prove that authentication, database permissions, or SQL operations are working.

## 14.3 Verify Migration Status

Check the application's migration process and migration history.

For a Flyway-managed application, an authorized deployment job can inspect migration status:

```bash
flyway info
```

Do not run production migrations blindly from an engineer's workstation.

The migration execution method must be controlled, logged, and approved.

# 15. Redis and Messaging Readiness

Verify Redis:

```bash
kubectl get pods -n production

kubectl get services -n production
```

If Redis is managed outside Kubernetes, verify its endpoint and network connectivity.

Example:

```bash
nc -vz production-redis.example.internal 6379
```

Verify messaging dependencies.

For RabbitMQ, check the configured cluster health and application connectivity.

For Kafka, verify:

- Broker availability.
- Bootstrap endpoints.
- Topic existence.
- ACLs.
- Consumer group configuration.
- Network access.
- TLS configuration, if enabled.

Example TCP check:

```bash
nc -vz kafka-1.example.internal 9092
```

A TCP check alone does not validate Kafka authentication, topic permissions, or message production and consumption.

# 16. Production Helm Validation

Before deployment, validate the chart.

```bash
helm lint ./charts/user
```

Render the manifests:

```bash
helm template roboshop-user ./charts/user \
  --namespace production \
  -f environments/production/user-values.yaml
```

Review the rendered resources.

Check:

- Image digest.
- Replica count.
- Environment variables.
- Secret references.
- Service ports.
- Probes.
- Resource requests.
- Resource limits.
- Security context.
- Service account.
- Autoscaling.
- Network policies.

Where supported by the installed Helm version, validate against the Kubernetes API:

```bash
helm template roboshop-user ./charts/user \
  --namespace production \
  -f environments/production/user-values.yaml \
  | kubectl apply --dry-run=server -f -
```

This is a validation operation, not a deployment.

# 17. Production GitOps Promotion

The Production configuration should be updated through a reviewed GitOps pull request.

Example:

```text
roboshop-gitops/
└── environments/
    └── production/
        └── user-values.yaml
```

Update only the approved image digest.

Example Git diff:

```diff
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user
-  digest: "sha256:PREVIOUS_DIGEST"
+  digest: "sha256:APPROVED_IMAGE_DIGEST"
```

Verify the diff:

```bash
git diff
```

Commit:

```bash
git add environments/production/user-values.yaml

git commit \
  -m "ROBOSHOP-450 promote user service to production"
```

Push the branch:

```bash
git push origin feature/ROBOSHOP-450-production-release
```

Open a pull request.

Required checks:

- Jira change key is present.
- Approved image digest matches.
- Helm rendering passes.
- Security checks pass.
- Required reviewers approve.
- Branch protection requirements pass.
- No unrelated changes are included.

After approval, merge according to the Production GitOps policy.

# 18. Argo CD Production Application

Argo CD observes the Production GitOps repository and reconciles the cluster with the desired state.

Example Application:

`applications/production/user.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: roboshop-user-production
  namespace: argocd

spec:
  project: roboshop-production

  source:
    repoURL: https://github.com/example/roboshop-gitops.git
    targetRevision: main
    path: charts/user

    helm:
      valueFiles:
        - ../../environments/production/user-values.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: false
      selfHeal: true

    syncOptions:
      - CreateNamespace=false
```

This is an illustrative manifest. Confirm the actual Helm chart path and relative values-file configuration in the repository. If necessary, use a multi-source Argo CD Application or another repository layout supported by the installed Argo CD version.

## 18.1 Why Disable Automated Pruning?

In this example:

```yaml
prune: false
```

Argo CD will not automatically delete resources removed from Git.

This is a conservative example for Production.

If the organization enables pruning, it should be deliberately configured with suitable review and safeguards.

## 18.2 Verify Argo CD Application

```bash
argocd app get roboshop-user-production
```

Check:

- Sync status.
- Health status.
- Repository.
- Target revision.
- Destination namespace.
- Last operation.
- Conditions.

Expected healthy state:

```text
Sync Status: Synced
Health Status: Healthy
```

# 19. Production Deployment Execution

After the Production GitOps PR has been approved and merged, Argo CD can reconcile the application.

For a controlled manual synchronization process:

```bash
argocd app sync roboshop-user-production
```

Wait for the application:

```bash
argocd app wait roboshop-user-production \
  --sync \
  --health \
  --timeout 600
```

Inspect:

```bash
argocd app get roboshop-user-production
```

The GitOps merge and Argo CD synchronization should be recorded against the Jira change.

Do not use an unrestricted `kubectl apply` from an engineer's workstation as a replacement for the GitOps deployment process.

# 20. Kubernetes Deployment Verification

Check deployments:

```bash
kubectl get deployments -n production
```

Check pods:

```bash
kubectl get pods -n production -o wide
```

Check replica status:

```bash
kubectl get deployments -n production \
  -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,AVAILABLE:.status.availableReplicas
```

Check rollout:

```bash
kubectl rollout status deployment/user \
  -n production \
  --timeout=300s
```

Check ReplicaSets:

```bash
kubectl get replicasets -n production
```

Check recent events:

```bash
kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp
```

Expected outcome:

- Required replicas are available.
- Pods are Ready.
- No unexpected restarts.
- No CrashLoopBackOff.
- No ImagePullBackOff.
- No scheduling failures.
- No unresolved mount failures.

# 21. Verify Running Image Digests

Verify the image used by the deployment:

```bash
kubectl get deployment user \
  -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'

echo
```

Verify the actual pod image:

```bash
kubectl get pods -n production \
  -l app=user \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.containerStatuses[*]}{.imageID}{" "}{end}{"\n"}{end}'
```

Compare the deployed image with the approved release digest.

This is important because a deployment being Ready does not prove that the correct artifact was deployed.

# 22. Production Service Verification

Check Services:

```bash
kubectl get services -n production
```

Check EndpointSlices:

```bash
kubectl get endpointslices -n production
```

Inspect a specific service:

```bash
kubectl describe service user \
  -n production
```

Verify:

- Service selector matches pod labels.
- Target port is correct.
- EndpointSlices contain expected endpoints.
- Application pods are Ready.
- Network policies allow the intended traffic.

A Service without healthy endpoints may result in application connection failures even when the Service object exists.

# 23. Production Ingress and ALB Verification

If AWS Load Balancer Controller manages the Production ingress, verify its resources.

```bash
kubectl get ingress -n production

kubectl describe ingress user-ingress \
  -n production
```

Check the AWS load balancer:

```bash
aws elbv2 describe-load-balancers \
  --region ap-south-1
```

Verify:

- ALB exists.
- Listener configuration is correct.
- Target groups exist.
- Targets are healthy.
- Security groups permit intended traffic.
- TLS certificates are valid.
- Health check paths are correct.

Check target health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --region ap-south-1
```

Do not assume the load balancer is healthy simply because an Ingress resource exists.

# 24. Production DNS and TLS Verification

Verify DNS:

```bash
dig +short roboshop.example.com
```

Verify TLS:

```bash
openssl s_client \
  -connect roboshop.example.com:443 \
  -servername roboshop.example.com
```

Check certificate details:

```bash
echo | openssl s_client \
  -connect roboshop.example.com:443 \
  -servername roboshop.example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Confirm:

- Correct DNS record.
- Correct ALB target.
- Valid certificate.
- Correct hostname.
- Certificate validity period.
- Correct TLS listener.

# 25. Production Application Smoke Testing

After deployment, validate the application's basic functionality.

Example:

```bash
curl -f https://roboshop.example.com/health
```

Validate application-specific endpoints:

```bash
curl -f https://roboshop.example.com/api/catalog
```

Use actual Production endpoint paths and approved test data.

A smoke test should confirm:

- Application responds.
- Health checks succeed.
- Important APIs work.
- Authentication works.
- Critical dependencies are accessible.
- No unexpected 5xx responses occur.

Do not use destructive test operations against real customer data.

# 26. End-to-End Business Workflow Testing

Validate the complete business flow using an approved Production smoke-test account and safe test data.

Example:

```text
              Production User
                    |
                    v
               User Login
                    |
                    v
              Product Catalog
                    |
                    v
                Add Cart
                    |
                    v
              Place Test Order
                    |
                    v
             Payment Validation
                    |
                    v
             Inventory Validation
                    |
                    v
            Notification Validation
```

If real payment processing is involved, use the organization's approved non-chargeable test mechanism or another safe verification process.

Verify:

- User authentication.
- Product retrieval.
- Cart operations.
- Order creation.
- Payment integration.
- Inventory consistency.
- Notification processing.

Do not declare the release successful based solely on pod health.

# 27. Production Application Logs

Inspect logs:

```bash
kubectl logs deployment/user \
  -n production \
  --tail=100
```

For multiple replicas, inspect the individual pods:

```bash
kubectl get pods -n production -l app=user
```

Then:

```bash
kubectl logs <POD_NAME> \
  -n production \
  --tail=100
```

Follow logs:

```bash
kubectl logs -f deployment/user \
  -n production
```

Check for:

- Database connection errors.
- Redis connection errors.
- Messaging errors.
- Authentication failures.
- Application exceptions.
- Timeout errors.
- Unexpected restarts.

Never include credentials, tokens, or sensitive customer data in troubleshooting evidence.

# 28. Production Monitoring Validation

Our monitoring stack uses:

- Prometheus.
- Grafana.
- ELK Stack.

## 28.1 Prometheus

Verify targets:

```text
Prometheus UI
    |
    v
Status
    |
    v
Targets
```

Check that the relevant application and infrastructure targets are healthy.

Example PromQL:

```promql
up
```

Check CPU usage:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="production",
    container!=""
  }[5m])
)
```

Check memory usage:

```promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="production",
    container!=""
  }
)
```

Check restart counts:

```promql
increase(kube_pod_container_status_restarts_total{
  namespace="production"
}[15m])
```

Use the metrics exposed by the actual Prometheus scrape configuration.

## 28.2 Grafana

Review:

- Kubernetes cluster dashboard.
- Node dashboard.
- Namespace dashboard.
- Pod resource dashboard.
- Application dashboard.
- HTTP error dashboard.
- Latency dashboard.
- Database dashboard.
- Messaging dashboard.

Compare the current behavior with the baseline.

## 28.3 ELK Stack

Review application logs in Kibana.

Useful filters:

```text
kubernetes.namespace : "production"
```

```text
kubernetes.labels.app : "user"
```

Look for:

- ERROR.
- FATAL.
- Database connection failures.
- Timeout exceptions.
- Failed requests.
- Messaging failures.

Verify that production logs are arriving without exposing sensitive information.

# 29. Production Alert Validation

Confirm that critical alerts are functioning.

Examples:

| Alert | Condition |
|---|---|
| PodUnavailable | Expected replicas unavailable |
| PodCrashLoop | Container repeatedly restarts |
| HighCPU | CPU above configured threshold |
| HighMemory | Memory above configured threshold |
| Application5xx | Elevated server errors |
| HighLatency | Request latency exceeds SLO |
| NodeNotReady | Kubernetes node becomes unhealthy |
| TargetUnhealthy | ALB target health fails |

Example PromQL for unavailable deployment replicas:

```promql
kube_deployment_status_replicas_available{
  namespace="production"
}
<
kube_deployment_spec_replicas{
  namespace="production"
}
```

The alert rule must handle deployments with zero desired replicas and other special cases according to the monitoring design.

Do not intentionally trigger a customer-impacting alert in Production just to test notification delivery. Use an approved synthetic test or controlled alert-testing method.

# 30. Production Deployment Validation Script

Create:

`production-deployment-validation.sh`

```bash
#!/bin/bash

set -euo pipefail

NAMESPACE="production"
DEPLOYMENT="user"
EXPECTED_CONTEXT="roboshop-production"

echo "Starting Production validation"

CURRENT_CONTEXT=$(kubectl config current-context)

if [[ "$CURRENT_CONTEXT" != "$EXPECTED_CONTEXT" ]]; then
    echo "ERROR: Wrong Kubernetes context"
    exit 1
fi

echo "Kubernetes context verified"

if ! kubectl get namespace "$NAMESPACE" >/dev/null 2>&1; then
    echo "ERROR: Production namespace not found"
    exit 1
fi

echo "Production namespace verified"

if ! kubectl get deployment "$DEPLOYMENT" \
    -n "$NAMESPACE" >/dev/null 2>&1; then
    echo "ERROR: Deployment not found"
    exit 1
fi

echo "Deployment exists"

kubectl rollout status deployment/"$DEPLOYMENT" \
    -n "$NAMESPACE" \
    --timeout=300s

READY=$(kubectl get deployment "$DEPLOYMENT" \
    -n "$NAMESPACE" \
    -o jsonpath='{.status.readyReplicas}')

DESIRED=$(kubectl get deployment "$DEPLOYMENT" \
    -n "$NAMESPACE" \
    -o jsonpath='{.spec.replicas}')

READY=${READY:-0}
DESIRED=${DESIRED:-0}

if [[ "$READY" != "$DESIRED" ]]; then
    echo "ERROR: Deployment replicas are not fully ready"
    echo "Ready: $READY"
    echo "Desired: $DESIRED"
    exit 1
fi

echo "Deployment replicas verified"

kubectl get pods -n "$NAMESPACE"

kubectl get services -n "$NAMESPACE"

kubectl get endpointslices -n "$NAMESPACE"

echo "Production validation completed"
```

Run:

```bash
chmod +x production-deployment-validation.sh

./production-deployment-validation.sh
```

This script validates one deployment and its basic Kubernetes state. Extend it to validate all required services, image digests, external dependencies, and application-specific smoke tests before using it as a complete release gate.

# 31. Multi-Microservice Production Validation

Our RoboShop platform contains more than 20 microservices.

Validating only the User service is insufficient.

Create an inventory of the actual Production deployments.

Example:

```bash
kubectl get deployments -n production
```

A validation script can check every expected deployment.

Create:

`validate-all-production-deployments.sh`

```bash
#!/bin/bash

set -euo pipefail

NAMESPACE="production"

DEPLOYMENTS=(
    user
    catalog
    cart
    orders
    payment
    inventory
    notification
)

FAILED=0

for DEPLOYMENT in "${DEPLOYMENTS[@]}"; do

    echo "--------------------------------"
    echo "Checking: $DEPLOYMENT"
    echo "--------------------------------"

    if ! kubectl get deployment "$DEPLOYMENT" \
        -n "$NAMESPACE" >/dev/null 2>&1; then

        echo "FAILED: Deployment not found"
        FAILED=1
        continue
    fi

    if ! kubectl rollout status deployment/"$DEPLOYMENT" \
        -n "$NAMESPACE" \
        --timeout=180s; then

        echo "FAILED: Rollout verification"
        FAILED=1
        continue
    fi

    echo "PASSED: $DEPLOYMENT"

done

if [[ "$FAILED" -ne 0 ]]; then
    echo "Production validation FAILED"
    exit 1
fi

echo "All listed Production deployments passed"
```

The list above is illustrative.

Replace it with the complete deployment inventory from the actual 20+ microservice architecture.

For a larger platform, maintain this inventory in version control and validate it against the approved release manifest.

# 32. Production Deployment Failure Handling

If deployment validation fails, do not automatically assume that a rollback is always the safest action.

First determine the failure type.

## Scenario 1: ImagePullBackOff

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

Possible causes:

- Wrong ECR image URI.
- Missing image digest.
- ECR access failure.
- Incorrect IAM configuration.
- Registry connectivity issue.

## Scenario 2: CrashLoopBackOff

Check:

```bash
kubectl logs <POD_NAME> \
  -n production \
  --previous
```

Possible causes:

- Application startup failure.
- Invalid configuration.
- Database connectivity failure.
- Missing secrets.
- Incompatible application version.

## Scenario 3: Pods Pending

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

Possible causes:

- Insufficient CPU.
- Insufficient memory.
- Node selector mismatch.
- Taints and tolerations.
- Persistent volume binding failure.

## Scenario 4: Service Has No Endpoints

Check:

```bash
kubectl get endpointslices \
  -n production

kubectl describe service user \
  -n production
```

Possible causes:

- Selector mismatch.
- Pods not Ready.
- Incorrect labels.
- Readiness probe failure.

## Scenario 5: HTTP 503

Investigate:

1. ALB target health.
2. Ingress configuration.
3. Service endpoints.
4. Pod readiness.
5. Application logs.
6. Network policies.
7. Backend connectivity.

Do not blindly restart all application pods.

# 33. Production Deployment Evidence Collection

Maintain evidence for every Production release.

Example directory:

```text
release-evidence/
└── roboshop-2026.11.03/
    ├── jira-change.txt
    ├── approval-record.txt
    ├── release-manifest.json
    ├── image-digests.txt
    ├── gitops-commit.txt
    ├── argocd-status.txt
    ├── deployment-status.txt
    ├── pod-status.txt
    ├── service-status.txt
    ├── smoke-test-results.txt
    ├── monitoring-validation.txt
    └── deployment-summary.md
```

Example collection commands:

```bash
mkdir -p release-evidence/roboshop-2026.11.03

kubectl get deployments -n production -o wide \
  > release-evidence/roboshop-2026.11.03/deployment-status.txt

kubectl get pods -n production -o wide \
  > release-evidence/roboshop-2026.11.03/pod-status.txt

kubectl get services -n production \
  > release-evidence/roboshop-2026.11.03/service-status.txt

kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp \
  > release-evidence/roboshop-2026.11.03/events.txt

argocd app get roboshop-user-production \
  > release-evidence/roboshop-2026.11.03/argocd-status.txt
```

Add the remaining service application statuses and test results.

Store evidence in the organization's approved artifact or audit storage.

Do not store credentials or secret values in the evidence directory.

# 34. Production Deployment Completion Record

After validation, update the Jira change record.

Example:

```text
PRODUCTION DEPLOYMENT COMPLETION

Change:
ROBOSHOP-450

Release:
roboshop-2026.11.03

Environment:
Production

Deployment Method:
Argo CD / GitOps

Deployment Status:
Successful

Image Digests:
Verified

Kubernetes Workloads:
Healthy

Application Smoke Tests:
Passed

Business Workflow Tests:
Passed

Monitoring:
Verified

Logging:
Verified

Known Issues:
Documented

Evidence:
Attached

Deployment Owner:
Assigned

Completion Status:
Ready for Post-Deployment Monitoring
```

Do not mark the deployment successful if critical validation is still incomplete.

# 35. Production Handover

Once the release is deployed, hand over the release information to the operations and support teams.

Handover information:

- Jira change key.
- Release identifier.
- Deployment time.
- GitOps commit.
- Container image digests.
- Services changed.
- Database changes.
- Configuration changes.
- Known issues.
- Monitoring dashboards.
- Relevant alerts.
- Rollback reference.
- On-call contact.
- Support owner.

The release should have an identified operational owner.

# 36. Production Deployment Security Best Practices

Follow these controls:

1. Use a dedicated Production AWS account where required.
2. Restrict Production EKS access.
3. Use IAM roles instead of long-lived credentials.
4. Use IRSA for workload-level AWS access.
5. Use immutable image digests.
6. Separate application source and GitOps repositories.
7. Protect Production Git branches.
8. Require code and configuration reviews.
9. Use approved secrets management.
10. Restrict direct Kubernetes write access.
11. Audit Argo CD operations.
12. Maintain a deployment evidence trail.
13. Apply least privilege to CI/CD identities.
14. Protect production environment variables.
15. Validate the exact approved release before deployment.

# 37. End-to-End Production Deployment Flow

Let's connect the entire process.

```text
1. Developer Completes Application Changes
                |
                v
2. CI Pipeline Builds Image
                |
                v
3. SonarQube and Trivy Validation
                |
                v
4. Image Published to ECR
                |
                v
5. DEV Deployment
                |
                v
6. QA Deployment
                |
                v
7. Staging Deployment
                |
                v
8. Jira Production Change Created
                |
                v
9. Production Approval Completed
                |
                v
10. Production GitOps PR Approved
                |
                v
11. AWS Account and EKS Verified
                |
                v
12. Infrastructure and Dependencies Verified
                |
                v
13. Image Digest Verified
                |
                v
14. Argo CD Synchronizes Production
                |
                v
15. Kubernetes Rollout Verified
                |
                v
16. Application Smoke Tests
                |
                v
17. Business Workflow Tests
                |
                v
18. Prometheus / Grafana / ELK Verified
                |
                v
19. Evidence Collected
                |
                v
20. Operations Handover
```

# 38. Chapter Completion Checklist

## AWS and EKS

- [ ] Production AWS account verified.
- [ ] AWS region verified.
- [ ] EKS context verified.
- [ ] Cluster connectivity verified.
- [ ] Node health verified.
- [ ] Capacity reviewed.
- [ ] Namespace verified.
- [ ] Resource quotas reviewed.
- [ ] IAM and IRSA verified.

## Artifacts and Configuration

- [ ] Approved ECR image exists.
- [ ] Image digest verified.
- [ ] Production Helm values reviewed.
- [ ] Configuration reviewed.
- [ ] Secrets integration verified.
- [ ] Database readiness verified.
- [ ] Redis readiness verified.
- [ ] Messaging readiness verified.

## GitOps and Deployment

- [ ] Production GitOps PR approved.
- [ ] Jira key linked to the change.
- [ ] Approved image digest committed.
- [ ] Argo CD application verified.
- [ ] Argo CD synchronization completed.
- [ ] Deployment rollout completed.
- [ ] All required microservices validated.

## Application and Operations

- [ ] Services and EndpointSlices verified.
- [ ] ALB targets verified.
- [ ] DNS verified.
- [ ] TLS verified.
- [ ] Smoke tests passed.
- [ ] Business workflows validated.
- [ ] Application logs reviewed.
- [ ] Prometheus targets verified.
- [ ] Grafana dashboards reviewed.
- [ ] ELK logs reviewed.
- [ ] Alerts verified.
- [ ] Evidence collected.
- [ ] Jira updated.
- [ ] Operations handover completed.

---

# 39. Chapter Summary

In this chapter, we implemented the standard Production deployment process for our RoboShop microservices platform.

We covered:

1. Production deployment architecture.
2. Production environment configuration.
3. GitOps repository structure.
4. Deployment prerequisites.
5. AWS account verification.
6. EKS cluster access.
7. Namespace validation.
8. Node and capacity validation.
9. IAM and IRSA.
10. ECR image verification.
11. Production configuration.
12. Secrets management.
13. Database readiness.
14. Redis and messaging readiness.
15. Helm validation.
16. GitOps promotion.
17. Argo CD application configuration.
18. Production synchronization.
19. Kubernetes rollout validation.
20. Image digest verification.
21. Service and EndpointSlice validation.
22. ALB and Ingress verification.
23. DNS and TLS validation.
24. Application smoke testing.
25. Business workflow testing.
26. Production logs.
27. Prometheus, Grafana, and ELK validation.
28. Alert validation.
29. Deployment validation scripts.
30. Multi-microservice validation.
31. Deployment failure handling.
32. Evidence collection.
33. Production handover.
34. Security best practices.

**Key principle:** Production deployment is not complete when Kubernetes reports that pods are Running. It is complete only after the approved artifact is deployed, workloads are healthy, application functionality is verified, dependencies are working, and operational readiness is confirmed.

---

# 40. Next Chapter

## CHAPTER 206 — ROLLING AND CANARY DEPLOYMENT

**Filename:** `206-Rolling-Canary-Deployment.md`

Next, we will implement advanced Production deployment strategies.

Planned topics:

1. Rolling deployment architecture.
2. Kubernetes RollingUpdate strategy.
3. maxSurge and maxUnavailable.
4. Zero-downtime deployment requirements.
5. Readiness and liveness probes.
6. PodDisruptionBudgets.
7. Deployment revision management.
8. Rolling deployment execution.
9. Rolling deployment validation.
10. Failed rollout handling.
11. Canary deployment architecture.
12. Canary traffic management.
13. ALB weighted target groups.
14. Progressive traffic shifting.
15. 5%, 10%, 25%, 50%, and 100% traffic stages.
16. Prometheus-based canary metrics.
17. Error rate and latency analysis.
18. Automated canary promotion.
19. Canary rollback.
20. Argo Rollouts integration.
21. Production deployment comparisons.
22. Failure simulations.
23. Real-world deployment scenarios.
24. Production deployment runbooks.
25. Final validation and evidence collection.

**Next file: `206-Rolling-Canary-Deployment.md`.**