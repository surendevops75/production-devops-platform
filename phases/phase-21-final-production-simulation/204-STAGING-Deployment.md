# CHAPTER 202 — STAGING DEPLOYMENT

**Filename:** `202-STAGING-Deployment.md`

---

# 1. Chapter Objective

The objective of this chapter is to implement a complete Staging deployment workflow for the RoboShop microservices platform.

Staging is the final pre-production environment where application changes are validated under production-like conditions.

The deployment process must ensure:

- The same application artifact tested in QA is promoted to Staging.
- No image is rebuilt during environment promotion.
- Container images are referenced using immutable SHA256 digests.
- Infrastructure is isolated from DEV and QA.
- Kubernetes configurations are environment-specific.
- Secrets are securely retrieved.
- Database and messaging connectivity is validated.
- Argo CD manages deployments through GitOps.
- Security, integration, performance, and regression tests are executed.
- Monitoring and logging are functional.
- Rollback procedures are documented and tested.
- Deployment evidence is collected before production approval.

## 1.1 Environment Promotion Flow

```text
Developer
   |
   v
Git Repository
   |
   v
CI Pipeline
   |
   +--> Build
   |
   +--> Unit Tests
   |
   +--> SonarQube
   |
   +--> Trivy
   |
   +--> Docker Build
   |
   +--> Push to Amazon ECR
   |
   v
Immutable Image Digest
   |
   v
DEV Deployment
   |
   v
QA Deployment
   |
   v
QA Approval
   |
   v
STAGING Promotion PR
   |
   v
Git Review and Approval
   |
   v
Argo CD
   |
   v
AWS EKS STAGING
   |
   +--> Smoke Tests
   +--> Integration Tests
   +--> Regression Tests
   +--> Security Validation
   +--> Performance Tests
   +--> Observability Validation
   |
   v
STAGING SIGN-OFF
   |
   v
Production Deployment Approval
```

---

# 2. Staging Environment Architecture

Staging should closely resemble Production in terms of application configuration, Kubernetes deployment structure, networking, security controls, and integrations.

The main differences should be capacity, cost, and intentionally isolated test data.

## 2.1 Staging Infrastructure Components

| Component | Staging implementation |
|---|---|
| Cloud | AWS |
| Kubernetes | Amazon EKS |
| Container registry | Amazon ECR |
| Infrastructure | Terraform |
| Deployment | Helm |
| GitOps | Argo CD |
| CI/CD | Jenkins / GitLab CI / GitHub Actions |
| Secrets | AWS Secrets Manager |
| Identity | IAM Roles for Service Accounts or EKS Pod Identity |
| Database | Amazon RDS |
| Cache | Redis |
| Messaging | RabbitMQ / Kafka, according to application design |
| Ingress | AWS Load Balancer Controller |
| TLS | ACM |
| DNS | Route 53 |
| Monitoring | Prometheus and Grafana |
| Logging | ELK Stack |
| Security | SonarQube, Trivy, Veracode where applicable |
| Performance | k6 |

## 2.2 Environment Isolation

Use separate namespaces at minimum:

```text
DEV EKS
  |
  +-- dev
       |
       +-- user
       +-- catalog
       +-- cart
       +-- orders
       +-- payment
       +-- inventory
       +-- notification

QA EKS
  |
  +-- qa
       |
       +-- user
       +-- catalog
       +-- cart
       +-- orders
       +-- payment
       +-- inventory
       +-- notification

STAGING EKS
  |
  +-- staging
       |
       +-- user
       +-- catalog
       +-- cart
       +-- orders
       +-- payment
       +-- inventory
       +-- notification

PRODUCTION EKS
  |
  +-- production
       |
       +-- user
       +-- catalog
       +-- cart
       +-- orders
       +-- payment
       +-- inventory
       +-- notification
```

For stronger isolation, use separate EKS clusters or AWS accounts for each environment. The choice depends on security requirements, cost, operational maturity, and organizational standards.

Do not assume a namespace alone provides complete infrastructure isolation.

---

# 3. Repository Structure

Maintain infrastructure, application, Helm, and GitOps configuration in clearly separated repositories.

Example:

```text
roboshop/
|
+-- roboshop-infrastructure/
|   |
|   +-- terraform/
|       |
|       +-- modules/
|       |   +-- vpc/
|       |   +-- eks/
|       |   +-- iam/
|       |   +-- rds/
|       |   +-- redis/
|       |   +-- security-groups/
|       |
|       +-- environments/
|           +-- dev/
|           +-- qa/
|           +-- staging/
|           +-- production/
|
+-- roboshop-microservices/
|   |
|   +-- user/
|   +-- catalog/
|   +-- cart/
|   +-- orders/
|   +-- payment/
|   +-- inventory/
|   +-- notification/
|   +-- Dockerfiles/
|
+-- roboshop-helm/
|   |
|   +-- charts/
|   |   +-- user/
|   |   +-- catalog/
|   |   +-- cart/
|   |   +-- orders/
|   |   +-- payment/
|   |   +-- inventory/
|   |   +-- notification/
|   |
|   +-- environments/
|       +-- dev/
|       +-- qa/
|       +-- staging/
|       +-- production/
|
+-- roboshop-gitops/
    |
    +-- applications/
    |   +-- dev/
    |   +-- qa/
    |   +-- staging/
    |   +-- production/
    |
    +-- projects/
    +-- application-sets/
    +-- policies/
```

For a 20+ microservice application, maintain a service inventory and generate repetitive configuration consistently rather than manually copying unverified configurations.

---

# 4. Pre-Deployment Checklist

Before starting the Staging deployment, verify the QA environment has completed its validation and approval process.

## 4.1 QA Exit Criteria

- [ ] QA deployment completed successfully.
- [ ] Application smoke tests passed.
- [ ] Integration tests passed.
- [ ] Regression tests passed.
- [ ] Security checks passed or have approved exceptions.
- [ ] Database migrations validated.
- [ ] Messaging workflows validated.
- [ ] No unresolved critical defects.
- [ ] Application image digests recorded.
- [ ] QA test evidence collected.
- [ ] QA owner approved the release.

## 4.2 Release Metadata

Create a release manifest.

Example:

`release-metadata.yaml`

```yaml
release:
  application: roboshop
  release_id: roboshop-2026.11.01
  source_environment: qa
  target_environment: staging

  git:
    repository: roboshop-microservices
    commit: REPLACE_WITH_APPROVED_COMMIT

  images:
    user:
      repository: REPLACE_WITH_ECR_USER_URI
      digest: sha256:REPLACE_WITH_REAL_DIGEST

    catalog:
      repository: REPLACE_WITH_ECR_CATALOG_URI
      digest: sha256:REPLACE_WITH_REAL_DIGEST

    cart:
      repository: REPLACE_WITH_ECR_CART_URI
      digest: sha256:REPLACE_WITH_REAL_DIGEST

  approval:
    qa_approved: true
    staging_approved: false
```

This is an example structure. Populate it from actual CI pipeline outputs and release approvals.

Do not invent image digests or use placeholders in a real deployment.

---

# 5. Connect to AWS

Use the appropriate AWS account and credentials for the Staging environment.

```bash
aws sts get-caller-identity
```

Expected output should identify the intended AWS account and IAM principal.

Set the region:

```bash
export AWS_REGION=ap-south-1
```

Use your actual AWS region.

Configure the staging cluster:

```bash
export CLUSTER_NAME=roboshop-staging-eks
```

Update kubeconfig:

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

Verify the Kubernetes context:

```bash
kubectl config current-context
```

Verify the cluster:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Check namespaces:

```bash
kubectl get namespaces
```

**Important:** Never assume that a successful AWS CLI authentication means you are connected to the correct EKS cluster.

Verify the account, region, cluster name, and Kubernetes context before making changes.

---

# 6. Validate Staging Infrastructure

## 6.1 Validate EKS Nodes

```bash
kubectl get nodes
```

Check node readiness:

```bash
kubectl get nodes \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type
```

Inspect a node:

```bash
kubectl describe node <NODE_NAME>
```

Check resource availability:

```bash
kubectl top nodes
```

If metrics are unavailable, verify the cluster metrics-server installation.

## 6.2 Validate System Pods

```bash
kubectl get pods -n kube-system
```

Check important components:

```bash
kubectl get deployment -n kube-system
```

Validate CoreDNS:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Validate the AWS Load Balancer Controller:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

Validate EBS CSI:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-ebs-csi-driver
```

The exact labels depend on the installed chart version.

## 6.3 Validate Cluster Add-ons

Check the EKS add-ons:

```bash
aws eks list-addons \
  --cluster-name "$CLUSTER_NAME" \
  --region "$AWS_REGION"
```

Check add-on versions and status:

```bash
aws eks describe-addon \
  --cluster-name "$CLUSTER_NAME" \
  --addon-name vpc-cni \
  --region "$AWS_REGION"
```

Repeat for the add-ons installed in the cluster.

---

# 7. Create the Staging Namespace

```bash
kubectl create namespace staging \
  --dry-run=client -o yaml | kubectl apply -f -
```

Validate:

```bash
kubectl get namespace staging
```

Apply environment labels:

```bash
kubectl label namespace staging \
  environment=staging \
  application=roboshop \
  --overwrite
```

Check labels:

```bash
kubectl get namespace staging --show-labels
```

---

# 8. Configure Resource Quotas

ResourceQuota limits aggregate resource consumption within a namespace.

Create:

`staging-resource-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-resource-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "150"
    services: "100"
    persistentvolumeclaims: "30"
```

Apply:

```bash
kubectl apply -f staging-resource-quota.yaml
```

Validate:

```bash
kubectl describe resourcequota \
  -n staging
```

These values are examples. Calculate the actual limits from the number of services, replicas, node capacity, and expected workload.

---

# 9. Configure LimitRange

LimitRange provides default and maximum resource settings for containers in a namespace.

Create:

`staging-limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limit-range
  namespace: staging
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
        cpu: "2"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
```

Apply:

```bash
kubectl apply -f staging-limit-range.yaml
```

Validate:

```bash
kubectl describe limitrange \
  -n staging
```

Remember that these defaults are applied only when container resource values are omitted. Explicitly configure resources in Helm values for every service.

---

# 10. Configure RBAC

Staging access should follow least privilege.

Create:

`staging-developer-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: staging-developer
  namespace: staging
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - services
      - configmaps
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

Create a RoleBinding for the approved group:

`staging-developer-binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: staging-developer-binding
  namespace: staging
subjects:
  - kind: Group
    name: staging-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: staging-developer
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f staging-developer-role.yaml
kubectl apply -f staging-developer-binding.yaml
```

Check access:

```bash
kubectl auth can-i get pods \
  -n staging \
  --as-group=staging-developers
```

The actual identity integration depends on the EKS access configuration and identity provider. Kubernetes RBAC objects alone do not create AWS or human identities.

---

# 11. Configure Application Identity and AWS Access

Applications should not use hardcoded AWS access keys.

Use EKS Pod Identity or IAM Roles for Service Accounts (IRSA), depending on the cluster's identity configuration.

Example service account:

`user-service-account.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-service-account
  namespace: staging
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/roboshop-staging-user-role
```

Apply:

```bash
kubectl apply -f user-service-account.yaml
```

For IRSA, ensure:

1. The cluster has an IAM OIDC provider configured.
2. The IAM role trust policy allows the intended service account.
3. The role has only the required AWS permissions.
4. The deployment references the correct service account.

Example deployment setting:

```yaml
spec:
  template:
    spec:
      serviceAccountName: user-service-account
```

For EKS Pod Identity, create the required association and use the corresponding service account. Do not assume the IRSA annotation is required for Pod Identity.

Verify the workload identity through an approved test container and AWS STS identity check.

---

# 12. Configure Secrets Management

Do not commit passwords, database connection strings, API tokens, or private keys to Git.

Use AWS Secrets Manager with an approved external-secrets integration or another organizationally approved secrets delivery mechanism.

Example secret naming:

```text
/roboshop/staging/user/database
/roboshop/staging/catalog/database
/roboshop/staging/orders/database
/roboshop/staging/payment/provider
/roboshop/staging/rabbitmq/credentials
```

Example ExternalSecret:

`staging-user-external-secret.yaml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: user-service-secret
  namespace: staging
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: staging-aws-secrets-manager
    kind: ClusterSecretStore

  target:
    name: user-service-secret
    creationPolicy: Owner

  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /roboshop/staging/user/database
        property: username

    - secretKey: DB_PASSWORD
      remoteRef:
        key: /roboshop/staging/user/database
        property: password
```

Apply:

```bash
kubectl apply -f staging-user-external-secret.yaml
```

Validate:

```bash
kubectl get externalsecret -n staging
```

```bash
kubectl describe externalsecret \
  user-service-secret \
  -n staging
```

Verify the generated Kubernetes Secret exists:

```bash
kubectl get secret user-service-secret \
  -n staging
```

Do not print secret values in terminal logs, CI output, or troubleshooting evidence.

The API version and fields must match the installed External Secrets Operator version.

---

# 13. Configure Application Configuration

Use environment-specific ConfigMaps for non-sensitive configuration.

Example:

`staging-user-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: staging
data:
  APP_ENV: staging
  LOG_LEVEL: info
  DB_HOST: staging-user-db.example.internal
  DB_PORT: "3306"
  DB_NAME: users
  REDIS_HOST: staging-redis.example.internal
  REDIS_PORT: "6379"
```

Apply:

```bash
kubectl apply -f staging-user-configmap.yaml
```

Validate:

```bash
kubectl describe configmap \
  user-service-config \
  -n staging
```

Use the actual database engine, port, and application configuration. Do not copy this example into a service that uses a different database or cache design.

---

# 14. Validate Database Connectivity

The Staging environment must use Staging database endpoints and credentials.

Do not point Staging at Production databases.

## 14.1 Validate RDS

Check the RDS instance:

```bash
aws rds describe-db-instances \
  --region "$AWS_REGION" \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]'
```

Validate:

- Database is available.
- Correct VPC and subnet groups are used.
- Security groups allow the required application traffic.
- Database credentials are retrieved securely.
- Database backups and retention are configured.
- TLS is enabled where required.
- Connection limits are sufficient.

## 14.2 Validate Network Connectivity

From an approved diagnostic pod:

```bash
kubectl run network-test \
  --namespace staging \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Check DNS:

```bash
kubectl exec -n staging network-test -- \
  nslookup staging-user-db.example.internal
```

Check TCP connectivity:

```bash
kubectl exec -n staging network-test -- \
  nc -zvw5 staging-user-db.example.internal 3306
```

Use the database's actual port.

Remove the diagnostic pod:

```bash
kubectl delete pod network-test \
  -n staging
```

A successful TCP connection does not prove that application authentication, SQL queries, schema, or transactions are working.

## 14.3 Database Migration Validation

Before running migrations:

1. Identify the migration version.
2. Confirm that the QA migration completed.
3. Confirm the migration is compatible with the application version.
4. Verify backups and recovery options.
5. Review whether the migration is backward compatible.
6. Obtain the required approval.
7. Run the migration using a controlled Job or migration pipeline.
8. Validate schema and data integrity.

Avoid destructive schema changes during routine application deployment.

Use an expand-and-contract strategy for database changes that need compatibility between old and new application versions.

---

# 15. Validate Redis Connectivity

Check the Redis endpoint and security configuration.

```bash
kubectl run redis-test \
  --namespace staging \
  --image=redis:7 \
  --restart=Never \
  -- sleep 3600
```

Check connectivity:

```bash
kubectl exec -n staging redis-test -- \
  redis-cli -h staging-redis.example.internal \
  -p 6379 PING
```

Expected response:

```text
PONG
```

For authenticated Redis, use a secure secret-injection method. Do not expose passwords in command-line arguments or shell history.

Remove the test pod:

```bash
kubectl delete pod redis-test \
  -n staging
```

---

# 16. Validate Messaging Infrastructure

RoboShop may use RabbitMQ, Kafka, or both, depending on the implemented service architecture.

Do not create unnecessary messaging infrastructure simply because a technology appears in a reference architecture.

## 16.1 RabbitMQ

Validate:

- Broker health.
- DNS resolution.
- Network connectivity.
- Credentials.
- Virtual host permissions.
- Exchanges.
- Queues.
- Bindings.
- Dead-letter configuration.
- Consumer connectivity.

For AMQP, the usual TCP port is 5672. Use the actual configured endpoint and TLS settings.

## 16.2 Kafka

Validate:

- Broker availability.
- Bootstrap endpoint.
- TLS and authentication.
- Topic existence.
- Producer permissions.
- Consumer permissions.
- Consumer group configuration.
- Retention policy.
- Partition count.
- Replication configuration.

The common Kafka client port is 9092, but the actual endpoint may differ.

## 16.3 End-to-End Messaging Test

Execute an application-level workflow:

```text
Create Order
    |
    v
Orders Service
    |
    v
Publish Order Event
    |
    v
Message Broker
    |
    v
Inventory Consumer
    |
    v
Update Inventory
    |
    v
Notification Consumer
    |
    v
Send Notification
```

Verify:

- Message is published.
- Message is consumed.
- Consumer processing succeeds.
- Failed processing is observable.
- Retry behavior works.
- Duplicate delivery does not cause incorrect business results.
- Dead-letter handling works when implemented.

---

# 17. Validate Helm Charts

Before deploying, validate the chart structure.

Example:

```text
roboshop-helm/
|
+-- charts/
    |
    +-- user/
        |
        +-- Chart.yaml
        +-- values.yaml
        +-- values-staging.yaml
        +-- templates/
            +-- deployment.yaml
            +-- service.yaml
            +-- ingress.yaml
            +-- hpa.yaml
            +-- serviceaccount.yaml
            +-- servicemonitor.yaml
```

Validate chart metadata:

```bash
helm lint charts/user
```

Render Staging manifests:

```bash
helm template user charts/user \
  --namespace staging \
  -f charts/user/values.yaml \
  -f charts/user/values-staging.yaml
```

Save rendered output for inspection:

```bash
helm template user charts/user \
  --namespace staging \
  -f charts/user/values.yaml \
  -f charts/user/values-staging.yaml \
  > /tmp/user-staging-rendered.yaml
```

Inspect:

```bash
less /tmp/user-staging-rendered.yaml
```

Check for:

- Correct image repository.
- Immutable image digest.
- Correct namespace.
- Correct replica count.
- CPU and memory requests.
- CPU and memory limits.
- Liveness probe.
- Readiness probe.
- Startup probe where needed.
- Correct ConfigMap references.
- Correct Secret references.
- Correct ServiceAccount.
- Correct Service ports.
- Correct labels and selectors.
- Security context.
- NetworkPolicy compatibility.

If using a Kubernetes schema validation tool, validate the rendered manifests against the actual target Kubernetes version.

---

# 18. Configure Staging Helm Values

Example:

`charts/user/values-staging.yaml`

```yaml
replicaCount: 2

image:
  repository: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user
  digest: sha256:REPLACE_WITH_REAL_DIGEST
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

probes:
  readiness:
    path: /health/ready
    initialDelaySeconds: 10
    periodSeconds: 10

  liveness:
    path: /health/live
    initialDelaySeconds: 30
    periodSeconds: 20

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 70

environment:
  APP_ENV: staging
  LOG_LEVEL: info
```

The chart template must support the `digest` field and construct the container image as:

```text
repository@digest
```

For example:

```yaml
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

Use health-check paths implemented by the actual application. The sample paths above are not universal.

---

# 19. Verify Immutable Image Promotion

This is one of the most important controls in environment promotion.

Do not rebuild the application when promoting from QA to Staging.

The same tested image digest should be used.

## 19.1 Get Image Digest from ECR

```bash
aws ecr describe-images \
  --repository-name roboshop/user \
  --region "$AWS_REGION" \
  --query 'imageDetails[*].[imageDigest,imageTags]'
```

Identify the digest associated with the approved release.

## 19.2 Confirm QA Image

Inspect the QA deployment:

```bash
kubectl get deployment user \
  -n qa \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Record the actual deployed image reference.

## 19.3 Compare Staging Image

After deployment:

```bash
kubectl get deployment user \
  -n staging \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Compare the image digest against the QA release manifest.

Do not compare only image tags.

Tags can be moved or overwritten. A digest identifies the image content.

Also verify the running pod's image ID:

```bash
kubectl get pods -n staging \
  -l app=user \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.containerStatuses[*].imageID}{"\n"}{end}'
```

---

# 20. Configure Argo CD for Staging

Argo CD should deploy the approved configuration from Git.

The deployment process should not rely on manual `helm upgrade` commands for routine GitOps-managed releases.

## 20.1 Validate Argo CD

```bash
kubectl get pods -n argocd
```

Check applications:

```bash
argocd app list
```

Check projects:

```bash
argocd proj list
```

## 20.2 Create an AppProject

Example:

`roboshop-staging-project.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: roboshop-staging
  namespace: argocd
spec:
  description: RoboShop Staging Environment

  sourceRepos:
    - https://github.com/ORG/roboshop-helm.git

  destinations:
    - namespace: staging
      server: https://kubernetes.default.svc

  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"

  clusterResourceWhitelist: []
```

Apply:

```bash
kubectl apply -f roboshop-staging-project.yaml
```

Review resource permissions carefully. The example permits namespace-scoped resource types; production AppProjects should use narrower allowlists wherever practical.

## 20.3 Create Argo CD Application

Example:

`user-staging-application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: roboshop-user-staging
  namespace: argocd
spec:
  project: roboshop-staging

  source:
    repoURL: https://github.com/ORG/roboshop-helm.git
    targetRevision: main
    path: charts/user

    helm:
      valueFiles:
        - values.yaml
        - values-staging.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: staging

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Apply:

```bash
kubectl apply -f user-staging-application.yaml
```

For controlled release promotion, consider disabling automatic sync or using an approval-controlled sync workflow. The choice should match the organization's release policy.

`prune: true` can delete resources removed from Git. Review its impact before enabling it.

## 20.4 Validate Application

```bash
argocd app get roboshop-user-staging
```

Check synchronization:

```bash
argocd app wait roboshop-user-staging \
  --sync \
  --health \
  --timeout 600
```

Check resources:

```bash
argocd app resources roboshop-user-staging
```

---

# 21. Deploy All Microservices

Repeat the same controlled deployment process for every approved microservice.

Example service inventory:

| Service | Deployment validation |
|---|---|
| User | API and database connectivity |
| Catalog | Product API and database |
| Cart | Cart API and Redis |
| Orders | Order creation and messaging |
| Payment | Payment integration and security |
| Inventory | Stock update and messaging |
| Notification | Event consumption and delivery |
| Shipping | Shipment workflow |
| Frontend | UI and backend connectivity |

The complete 20+ service inventory must come from the actual RoboShop repository. Do not assume that this example is the complete list.

Use a release manifest to ensure every service has an approved image digest.

## 21.1 Check Deployments

```bash
kubectl get deployments -n staging
```

Check pods:

```bash
kubectl get pods -n staging -o wide
```

Check services:

```bash
kubectl get services -n staging
```

Check ReplicaSets:

```bash
kubectl get replicasets -n staging
```

Check events:

```bash
kubectl get events -n staging \
  --sort-by='.lastTimestamp'
```

## 21.2 Check Deployment Availability

```bash
kubectl rollout status deployment/user \
  -n staging \
  --timeout=300s
```

Repeat for the deployed services.

## 21.3 Check Pod Logs

```bash
kubectl logs deployment/user \
  -n staging \
  --all-containers=true \
  --tail=100
```

For a previous container instance:

```bash
kubectl logs <POD_NAME> \
  -n staging \
  --previous
```

---

# 22. Configure Services and Internal DNS

Validate that every service has the correct Service object.

```bash
kubectl get services -n staging
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user
  namespace: staging
spec:
  selector:
    app: user
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Check endpoints:

```bash
kubectl get endpointslices \
  -n staging
```

Validate DNS from an application diagnostic pod:

```bash
kubectl exec -n staging <POD_NAME> -- \
  nslookup user.staging.svc.cluster.local
```

Service DNS and port configuration must match the actual application architecture.

A Service with no ready endpoints can indicate:

- Incorrect selector.
- Pod not ready.
- Wrong namespace.
- Missing labels.
- Readiness probe failure.

---

# 23. Configure Network Policies

Network policies should restrict unnecessary communication between workloads.

Example policy:

`user-network-policy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: user-network-policy
  namespace: staging
spec:
  podSelector:
    matchLabels:
      app: user

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: staging
      ports:
        - protocol: TCP
          port: 8080
```

Apply:

```bash
kubectl apply -f user-network-policy.yaml
```

Important:

- This example only restricts ingress.
- It does not define egress rules.
- Verify that the cluster networking implementation enforces Kubernetes NetworkPolicy.
- Ensure DNS, database, Redis, broker, and required service traffic remains functional.
- Do not apply default-deny rules without a tested allowlist.

---

# 24. Configure Ingress and TLS

Staging should use a dedicated hostname.

Example:

```text
staging.roboshop.example.com
```

Create an Ingress:

`staging-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: roboshop-staging-ingress
  namespace: staging
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: REPLACE_WITH_STAGING_ACM_CERTIFICATE_ARN
    alb.ingress.kubernetes.io/ssl-redirect: "443"

spec:
  ingressClassName: alb

  rules:
    - host: staging.roboshop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f staging-ingress.yaml
```

Validate:

```bash
kubectl get ingress -n staging
```

Describe:

```bash
kubectl describe ingress \
  roboshop-staging-ingress \
  -n staging
```

Validate:

- Correct ALB is provisioned.
- Correct target groups exist.
- Targets are healthy.
- ACM certificate is valid.
- Security groups permit required traffic.
- DNS resolves to the correct ALB.
- HTTP redirects to HTTPS.
- Backend Service and port are correct.

For a private Staging environment, use an internal ALB and appropriate private DNS.

---

# 25. Configure Route 53

Create or update the Staging DNS record after the ALB is provisioned.

Example DNS name:

```text
staging.roboshop.example.com
```

Validate resolution:

```bash
nslookup staging.roboshop.example.com
```

Or:

```bash
dig staging.roboshop.example.com
```

Validate HTTPS:

```bash
curl -I https://staging.roboshop.example.com
```

Validate the application response:

```bash
curl -sS \
  -o /dev/null \
  -w '%{http_code}\n' \
  https://staging.roboshop.example.com/
```

A successful HTTP response alone does not prove every backend workflow is healthy.

---

# 26. Configure HPA

Horizontal Pod Autoscaler adjusts the number of replicas based on configured metrics.

Example:

`user-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-hpa
  namespace: staging
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user

  minReplicas: 2
  maxReplicas: 6

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Apply:

```bash
kubectl apply -f user-hpa.yaml
```

Validate:

```bash
kubectl get hpa -n staging
```

Describe:

```bash
kubectl describe hpa user-hpa \
  -n staging
```

Check whether the metrics API is available:

```bash
kubectl top pods -n staging
```

Do not enable CPU-based HPA without appropriate CPU requests.

---

# 27. Configure Pod Disruption Budgets

PodDisruptionBudget helps protect application availability during voluntary disruptions.

Example:

`user-pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-pdb
  namespace: staging
spec:
  minAvailable: 1
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
kubectl get pdb -n staging
```

Ensure the replica count and disruption budget are compatible. A PDB can block maintenance if configured too strictly.

---

# 28. Security Validation

Security validation must happen before Staging sign-off.

## 28.1 SonarQube

Review the approved CI scan report.

Validate:

- New bugs.
- New vulnerabilities.
- Code smells.
- Security hotspots.
- Coverage.
- Quality gate.

Do not treat a successful deployment as proof that source-code security checks passed.

## 28.2 Trivy Image Scan

Scan the exact image promoted to Staging.

Example:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:REAL_DIGEST
```

Check:

- OS packages.
- Application dependencies.
- Critical vulnerabilities.
- High vulnerabilities.
- Fixed-version availability.
- Approved exceptions.

## 28.3 Kubernetes Configuration Scan

Example:

```bash
trivy config roboshop-helm/
```

Review the findings against the actual rendered Staging manifests as well as source templates.

## 28.4 Runtime Security Checks

Verify:

- Containers do not run as root unless justified.
- Privilege escalation is disabled.
- Capabilities are dropped where possible.
- Read-only root filesystems are used where compatible.
- Secrets are not embedded in images.
- Images use approved registries.
- Kubernetes RBAC follows least privilege.
- Network access is restricted.
- EKS control-plane and workload access follow organizational policy.

Example security context:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

Some applications need writable temporary directories or specific capabilities. Validate application compatibility before enforcing these settings.

---

# 29. Application Smoke Testing

Smoke tests confirm the basic application functions after deployment.

## 29.1 Frontend Test

```bash
curl -f https://staging.roboshop.example.com/
```

## 29.2 Health Endpoint Test

Use the real application health endpoint:

```bash
curl -f \
  https://staging.roboshop.example.com/health
```

## 29.3 API Test

Example:

```bash
curl -f \
  https://staging.roboshop.example.com/api/products
```

The API path and expected response must match the application implementation.

## 29.4 Authentication Test

Validate:

- Login.
- Token generation.
- Token expiry.
- Unauthorized access rejection.
- Session behavior.

## 29.5 Order Workflow Test

Execute a controlled Staging transaction:

```text
User Login
   |
   v
Browse Products
   |
   v
Add to Cart
   |
   v
Create Order
   |
   v
Process Payment
   |
   v
Update Inventory
   |
   v
Send Notification
```

Verify the state transitions in every participating service.

Use test accounts and test payment integrations. Never use real customer or payment data in routine Staging tests.

---

# 30. Integration Testing

Integration tests validate communication between microservices and infrastructure.

Examples:

| Integration | Validation |
|---|---|
| Frontend → User | Authentication |
| Frontend → Catalog | Product listing |
| Frontend → Cart | Cart operations |
| Orders → Payment | Payment request |
| Orders → Inventory | Stock validation |
| Orders → Messaging | Event publishing |
| Messaging → Notification | Event consumption |
| Services → Database | Queries and transactions |
| Services → Redis | Cache operations |

For each integration, capture:

- Request.
- Response.
- HTTP status or message acknowledgement.
- Correlation ID.
- Logs.
- Database or messaging side effects.
- Failure behavior.

Do not validate only HTTP 200 responses. Verify the actual business result.

---

# 31. Regression Testing

Run the approved regression suite.

Suggested coverage:

1. User registration.
2. User login.
3. Product search.
4. Product details.
5. Add to cart.
6. Update cart.
7. Remove cart item.
8. Checkout.
9. Payment success.
10. Payment failure.
11. Inventory update.
12. Order status.
13. Notification.
14. Retry behavior.
15. Unauthorized access.
16. Invalid input.
17. Service dependency failure.

Record test execution, failures, defect IDs, and approvals.

---

# 32. Performance Testing with k6

Staging should undergo controlled performance testing.

Do not run an uncontrolled load test against shared infrastructure.

## 32.1 Install k6

Use the official installation instructions for the operating system and approved version.

Verify:

```bash
k6 version
```

## 32.2 Create a Test Script

`staging-load-test.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 30 },
    { duration: '3m', target: 50 },
    { duration: '2m', target: 0 },
  ],

  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};

export default function () {
  const response = http.get(
    'https://staging.roboshop.example.com/'
  );

  check(response, {
    'status is 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

## 32.3 Execute

```bash
k6 run staging-load-test.js
```

The thresholds and traffic profile are examples, not universal service-level objectives.

## 32.4 Analyze

Review:

- Request rate.
- Error rate.
- p50 latency.
- p95 latency.
- p99 latency.
- CPU utilization.
- Memory utilization.
- Pod restarts.
- HPA scaling.
- Database connections.
- Redis latency.
- Messaging backlog.

If performance degrades, identify the bottleneck before increasing replicas or infrastructure size.

---

# 33. Observability Validation

Use the existing monitoring and logging architecture.

The observability stack for this project is:

- Prometheus.
- Grafana.
- ELK Stack.

## 33.1 Prometheus

Check Prometheus:

```bash
kubectl get pods -A | grep prometheus
```

Verify targets through the Prometheus UI.

Confirm:

- Node exporter targets.
- Kubernetes targets.
- Application metrics.
- ServiceMonitor selection.
- Scrape intervals.
- Target health.

Example application metrics:

```text
http_requests_total
http_request_duration_seconds
process_cpu_seconds_total
process_resident_memory_bytes
```

Actual metric names depend on the application's instrumentation.

## 33.2 Grafana

Verify dashboards for:

- Cluster health.
- Node CPU and memory.
- Pod CPU and memory.
- Deployment replicas.
- Pod restarts.
- Application request rate.
- Error rate.
- Response latency.
- Database health.
- Messaging health.

## 33.3 ELK Stack

Verify that Staging application logs reach the logging pipeline.

Check:

- Logstash input.
- Log parsing.
- Timestamp normalization.
- Service name.
- Environment label.
- Log level.
- Correlation ID.
- Error stack traces.
- Elasticsearch indexing.
- Kibana visibility.

Every Staging log should be distinguishable from QA and Production logs.

Example structured log:

```json
{
  "timestamp": "2026-11-03T12:00:00Z",
  "level": "INFO",
  "service": "orders",
  "environment": "staging",
  "message": "Order created successfully",
  "correlation_id": "example-correlation-id"
}
```

Do not log credentials, access tokens, payment details, or other sensitive data.

---

# 34. Alert Validation

Verify that important alert rules are active and routed correctly.

Suggested alerts:

| Alert | Condition |
|---|---|
| PodCrashLooping | Container repeatedly restarts |
| DeploymentUnavailable | Desired replicas unavailable |
| HighCPUUsage | CPU exceeds approved threshold |
| HighMemoryUsage | Memory exceeds approved threshold |
| HighHTTP5xx | Error rate exceeds threshold |
| HighLatency | p95 latency exceeds threshold |
| NodeNotReady | Kubernetes node unavailable |
| DatabaseConnectionsHigh | Connection usage near limit |
| MessageBacklogHigh | Consumer backlog exceeds threshold |

Do not rely on alerts that have never been tested.

Perform controlled tests and verify notification delivery to the approved Staging channel.

---

# 35. Failure Testing

Test recovery behavior using controlled failure scenarios.

## 35.1 Delete a Test Pod

```bash
kubectl delete pod <TEST_POD_NAME> \
  -n staging
```

Verify that the Deployment recreates it.

## 35.2 Simulate Application Failure

Use an approved test deployment or controlled test endpoint.

Verify:

- Readiness probe failure removes the pod from ready endpoints.
- Liveness probe behavior is correct.
- Restart count is observable.
- Alerts are generated when thresholds are met.

## 35.3 Test Dependency Failure

In a controlled test environment, simulate loss of a non-critical dependency.

Verify:

- Timeouts.
- Retry limits.
- Circuit breaker behavior, if implemented.
- Graceful degradation.
- Error logging.
- Recovery after the dependency returns.

Do not intentionally disrupt shared databases, production integrations, or critical infrastructure.

---

# 36. Rollback Strategy

A Staging rollback should restore a known-good release configuration.

For a GitOps deployment, the preferred rollback is to revert the Git promotion commit or submit a controlled PR restoring the previous image digest and configuration.

## 36.1 Identify Previous Release

```bash
argocd app history roboshop-user-staging
```

Review the Git history and identify the previously approved release commit.

## 36.2 Revert the Promotion Commit

```bash
git revert <PROMOTION_COMMIT>
```

Push the change through the approved pull request process.

## 36.3 Verify Argo CD

```bash
argocd app get roboshop-user-staging
```

Wait for the restored desired state:

```bash
argocd app wait roboshop-user-staging \
  --sync \
  --health \
  --timeout 600
```

## 36.4 Validate the Rollback

```bash
kubectl rollout status deployment/user \
  -n staging
```

```bash
kubectl get pods -n staging
```

Check:

- Expected image digest restored.
- Application healthy.
- Service endpoints available.
- Smoke tests pass.
- Error rate returns to acceptable levels.

**Database warning:** Rolling back an application image does not automatically roll back a database migration. Database recovery must follow the separately approved migration and backup strategy.

---

# 37. Troubleshooting

## 37.1 Pods in ImagePullBackOff

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n staging
```

Possible causes:

- Incorrect ECR repository.
- Incorrect image digest.
- ECR permissions.
- Network connectivity.
- Invalid image reference.

Verify the EKS node or workload image-pull permissions according to the configured architecture.

## 37.2 CrashLoopBackOff

Check logs:

```bash
kubectl logs <POD_NAME> \
  -n staging
```

Previous logs:

```bash
kubectl logs <POD_NAME> \
  -n staging \
  --previous
```

Inspect:

```bash
kubectl describe pod <POD_NAME> \
  -n staging
```

Investigate:

- Application startup failure.
- Invalid configuration.
- Missing Secret.
- Database connection failure.
- Incorrect command or arguments.
- Memory limits.
- Failed health probes.

## 37.3 Pending Pods

```bash
kubectl describe pod <POD_NAME> \
  -n staging
```

Check:

- Node capacity.
- Resource requests.
- Taints and tolerations.
- Node selectors.
- Affinity.
- Persistent volume claims.
- ResourceQuota.

## 37.4 Service Returns 503

Check:

```bash
kubectl get svc -n staging
```

```bash
kubectl get endpointslices -n staging
```

```bash
kubectl get pods -n staging
```

Common causes:

- No ready pods.
- Incorrect Service selector.
- Incorrect target port.
- Readiness probe failure.
- Ingress target health failure.

## 37.5 Argo CD OutOfSync

```bash
argocd app get roboshop-user-staging
```

Inspect:

- Git revision.
- Helm values.
- Rendered manifests.
- Cluster drift.
- Missing resources.
- Permission errors.

Do not blindly sync an application without reviewing unexpected resource differences.

## 37.6 Database Connection Failure

Check:

- DNS.
- Security groups.
- Network ACLs.
- Route tables.
- Database endpoint.
- Credentials.
- TLS configuration.
- Connection pool limits.
- Database availability.

## 37.7 High Latency

Correlate:

- Application logs.
- Prometheus metrics.
- Grafana dashboards.
- Database metrics.
- Redis latency.
- Messaging backlog.
- Kubernetes resource utilization.

Do not immediately increase replica counts without finding the bottleneck.

---

# 38. Staging Validation Script

Create:

`validate-staging.sh`

```bash
#!/bin/bash

set -uo pipefail

NAMESPACE="staging"
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

check "Staging namespace exists" \
    kubectl get namespace "$NAMESPACE"

check "Deployments available" \
    kubectl get deployments -n "$NAMESPACE"

check "Pods listed" \
    kubectl get pods -n "$NAMESPACE"

check "Services listed" \
    kubectl get services -n "$NAMESPACE"

check "Resource quota exists" \
    kubectl get resourcequota -n "$NAMESPACE"

check "LimitRange exists" \
    kubectl get limitrange -n "$NAMESPACE"

echo
echo "Deployment status summary:"
kubectl get deployments -n "$NAMESPACE"

echo
echo "Pod status summary:"
kubectl get pods -n "$NAMESPACE"

echo
echo "Recent events:"
kubectl get events -n "$NAMESPACE" \
    --sort-by='.lastTimestamp' | tail -20

if [ "$FAILED" -eq 0 ]; then
    echo
    echo "Basic Staging validation completed."
    exit 0
else
    echo
    echo "Staging validation reported failures."
    exit 1
fi
```

Run:

```bash
chmod +x validate-staging.sh
./validate-staging.sh
```

This script checks basic Kubernetes resources. It does not replace application tests, security validation, image-digest verification, or release approval.

---

# 39. Deployment Evidence Collection

Create an evidence directory for every release.

Example:

```text
staging-evidence/
|
+-- release-metadata.yaml
+-- git-commit.txt
+-- image-digests.txt
+-- argocd-status.txt
+-- deployment-status.txt
+-- pod-status.txt
+-- service-status.txt
+-- ingress-status.txt
+-- security-scan-results/
+-- smoke-test-results/
+-- integration-test-results/
+-- regression-test-results/
+-- performance-test-results/
+-- monitoring-evidence/
+-- approval-records/
```

Collect basic Kubernetes evidence:

```bash
mkdir -p staging-evidence
```

```bash
kubectl get deployments -n staging -o wide \
  > staging-evidence/deployment-status.txt
```

```bash
kubectl get pods -n staging -o wide \
  > staging-evidence/pod-status.txt
```

```bash
kubectl get services -n staging \
  > staging-evidence/service-status.txt
```

```bash
kubectl get ingress -n staging \
  > staging-evidence/ingress-status.txt
```

```bash
kubectl get events -n staging \
  --sort-by='.lastTimestamp' \
  > staging-evidence/events.txt
```

```bash
argocd app list \
  > staging-evidence/argocd-status.txt
```

Protect evidence files. Do not include secrets, access tokens, or sensitive customer information.

---

# 40. Staging Acceptance Criteria

The release is ready for Staging sign-off only when the agreed acceptance criteria are satisfied.

## Infrastructure

- [ ] Correct AWS account and EKS cluster.
- [ ] Nodes are healthy.
- [ ] Required EKS add-ons are operational.
- [ ] Namespace and quotas configured.
- [ ] RBAC validated.
- [ ] Workload identity validated.
- [ ] Network connectivity verified.

## Application

- [ ] All approved microservices deployed.
- [ ] Expected image digests deployed.
- [ ] Deployments available.
- [ ] Pods healthy.
- [ ] Services have ready endpoints.
- [ ] Internal DNS works.
- [ ] Ingress and TLS work.
- [ ] Database connectivity verified.
- [ ] Redis connectivity verified.
- [ ] Messaging workflows verified.

## Security

- [ ] SonarQube quality gate reviewed.
- [ ] Trivy image scan reviewed.
- [ ] Kubernetes configuration scan reviewed.
- [ ] Secrets not committed to Git.
- [ ] RBAC and IAM permissions reviewed.
- [ ] Security exceptions approved.

## Testing

- [ ] Smoke tests passed.
- [ ] Integration tests passed.
- [ ] Regression tests passed.
- [ ] Performance tests completed.
- [ ] Failure tests completed.
- [ ] No unresolved release-blocking defects.

## Observability

- [ ] Prometheus targets healthy.
- [ ] Grafana dashboards available.
- [ ] Application logs indexed in ELK.
- [ ] Important alerts active.
- [ ] Notification delivery tested.

## Release Management

- [ ] Git promotion reviewed.
- [ ] Argo CD sync successful.
- [ ] Rollback procedure documented.
- [ ] Evidence collected.
- [ ] Staging approval recorded.

---

# 41. Staging Sign-Off

Create a release approval record.

Example:

```yaml
release:
  name: roboshop-2026.11.01
  environment: staging

validation:
  infrastructure: passed
  application: passed
  security: passed
  integration: passed
  regression: passed
  performance: passed
  observability: passed
  rollback: verified

approvals:
  qa_owner: APPROVED
  devops_owner: APPROVED
  security_owner: APPROVED
  application_owner: PENDING

production_eligibility: false
```

This is a sample record. The actual approval system should identify the responsible people, timestamps, evidence links, and any approved exceptions.

Do not mark a release as production-ready while required approvals remain pending.

---

# 42. What Happens After Staging?

After Staging approval:

1. Freeze the approved release candidate.
2. Record the final Git commit.
3. Record every service's image digest.
4. Confirm the database migration plan.
5. Confirm the production infrastructure readiness.
6. Review security exceptions.
7. Review performance results.
8. Review rollback procedures.
9. Obtain production change approval.
10. Promote the approved release to Production.

The Production deployment must use the approved artifact. Do not rebuild the application just to deploy it to Production.

The next chapter should cover Production Deployment, including release controls, production change approval, deployment strategy, zero-downtime rollout, live validation, incident response, and production rollback.

---

# 43. Final Chapter Summary

In this chapter, the RoboShop application was prepared for a production-like Staging environment.

The implementation covered:

- AWS EKS Staging access and infrastructure validation.
- Kubernetes namespace isolation.
- ResourceQuota and LimitRange.
- RBAC and workload identity.
- AWS Secrets Manager integration.
- Environment-specific ConfigMaps.
- Database, Redis, RabbitMQ, and Kafka connectivity.
- Helm chart validation.
- Immutable ECR image promotion.
- Argo CD AppProject and Application configuration.
- Deployment of the approved microservices.
- Kubernetes Services and internal DNS.
- NetworkPolicy.
- ALB Ingress, ACM, and Route 53.
- HPA and PodDisruptionBudget.
- SonarQube and Trivy security checks.
- Smoke, integration, and regression testing.
- k6 performance testing.
- Prometheus, Grafana, and ELK validation.
- Alert and failure testing.
- GitOps rollback.
- Troubleshooting.
- Automated validation.
- Evidence collection.
- Staging sign-off.

**Core DevOps principle:**

Staging is not simply another Kubernetes namespace. It is the final controlled validation environment before Production. Every release must be traceable, tested, secure, observable, and recoverable.