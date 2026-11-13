# 201 — QA Deployment

## Production DevOps Capstone — RoboShop Microservices on AWS EKS

**Environment:** QA  
**Deployment:** GitOps with Argo CD and Helm  
**Cloud:** AWS  
**Container platform:** Amazon EKS  
**Registry:** Amazon ECR  
**Packaging:** Helm  
**Monitoring:** Prometheus and Grafana  
**Logging:** ELK Stack  
**Ingress:** AWS Load Balancer Controller / ALB  
**Secrets:** AWS Secrets Manager with the approved external-secrets integration  
**Deployment source of truth:** GitOps repository

---

# 1. Chapter Objectives

This chapter implements the complete QA deployment lifecycle for the RoboShop microservices platform.

By completing this chapter, you should be able to:

1. Understand the role of QA in a production deployment lifecycle.
2. Promote the same immutable image digest from DEV to QA.
3. Verify that the artifact passed DEV validation.
4. Validate ECR image availability.
5. Prepare the QA EKS cluster.
6. Configure the QA Kubernetes context.
7. Verify cluster access and worker-node health.
8. Establish QA namespace isolation.
9. Configure ResourceQuota and LimitRange.
10. Configure Kubernetes RBAC.
11. Configure ServiceAccounts and workload identity.
12. Configure image-pull permissions.
13. Configure QA secrets and ConfigMaps.
14. Configure database connectivity.
15. Configure Redis and messaging connectivity where required.
16. Configure QA Helm values.
17. Validate Helm templates.
18. Configure Argo CD Applications.
19. Configure AppProjects and deployment permissions.
20. Deploy applications through GitOps.
21. Validate Deployments, Pods, Services and EndpointSlices.
22. Validate probes, resources and scheduling.
23. Configure internal service communication.
24. Configure ALB Ingress and QA DNS.
25. Validate TLS and application endpoints.
26. Perform integration testing.
27. Perform API and end-to-end smoke testing.
28. Perform security validation.
29. Validate Prometheus and Grafana.
30. Validate centralized logging through ELK.
31. Validate application performance.
32. Test rollback and recovery.
33. Collect release evidence.
34. Approve the artifact for staging or production promotion.

---

# 2. DEV-to-QA Deployment Architecture

The deployment lifecycle is:

```text
Developer
    |
    v
Application Repository
    |
    v
CI Pipeline
    |
    +--> Unit Tests
    |
    +--> Maven / npm Build
    |
    +--> SonarQube
    |
    +--> Trivy
    |
    +--> Veracode
    |
    +--> Docker Build
    |
    v
Amazon ECR
    |
    v
Immutable Image Digest
    |
    v
DEV GitOps
    |
    v
DEV Argo CD
    |
    v
DEV EKS
    |
    v
DEV Validation
    |
    v
QA Promotion Pull Request
    |
    v
QA GitOps
    |
    v
QA Argo CD
    |
    v
QA EKS
    |
    +--> Integration Testing
    |
    +--> API Testing
    |
    +--> Security Testing
    |
    +--> Performance Testing
    |
    +--> Observability Validation
    |
    v
QA Approval
    |
    v
STAGING Promotion
```

Important principle:

```text
DEV IMAGE = QA IMAGE = STAGING IMAGE = PROD IMAGE
```

The image digest must remain identical throughout the promotion lifecycle.

Environment-specific configuration changes, but the application artifact does not.

---

# 3. QA Environment Responsibilities

DEV and QA serve different purposes.

| Area | DEV | QA |
|---|---|---|
| Main purpose | Developer feedback | Quality validation |
| Deployment | Frequent | Controlled |
| Image | Immutable digest | Same immutable digest |
| Testing | Unit and basic smoke tests | Integration and E2E |
| Configuration | Development configuration | QA configuration |
| Database | DEV database | QA database |
| Test data | Development data | Controlled QA data |
| Access | Developers | QA and engineering teams |
| Security | Development checks | Broader validation |
| Release decision | DEV validation | QA sign-off |

QA should resemble production architecturally, while using controlled capacity and isolated data.

---

# 4. QA Environment Architecture

Reference architecture:

```text
                         AWS ACCOUNT
                              |
                              v
                         AWS REGION
                              |
                              v
                         QA VPC
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          AZ-A              AZ-B              AZ-C
             |                |                |
       Private Subnet   Private Subnet   Private Subnet
             |                |                |
             +----------------+----------------+
                              |
                              v
                           QA EKS
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
         System Nodes    Application Nodes   QA Workloads
                              |
                              v
                        QA Namespace
                              |
       +----------+-----------+----------+-----------+
       |          |           |          |           |
       v          v           v          v           v
    Frontend   Catalogue     Cart       User       Orders
       |          |           |          |           |
       +----------+-----------+----------+-----------+
                              |
                              v
                      Internal Services
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          Database          Redis          Messaging
                              |
                              v
                         QA ALB
                              |
                              v
                         QA DNS
                              |
                              v
                         QA Testers
```

This is a logical reference architecture. The actual microservice-to-database and messaging dependencies must follow the application's documented architecture.

---

# 5. Repository Structure

The QA deployment uses the existing application, infrastructure and GitOps repositories.

## 5.1 Application repository

Example:

```text
roboshop-cart/
|
+-- src/
+-- tests/
+-- Dockerfile
+-- pom.xml
+-- Jenkinsfile
+-- .github/
|   +-- workflows/
+-- .gitlab-ci.yml
+-- README.md
```

The application repository owns:

- Application source.
- Unit tests.
- Integration-test source.
- Dockerfile.
- Build configuration.
- CI pipeline.
- Application documentation.

## 5.2 Infrastructure repository

```text
roboshop-infra/
|
+-- modules/
|   +-- vpc/
|   +-- eks/
|   +-- iam/
|   +-- ecr/
|   +-- rds/
|   +-- security/
|
+-- environments/
    +-- dev/
    +-- qa/
    +-- staging/
    +-- prod/
```

Terraform owns AWS infrastructure.

## 5.3 GitOps repository

```text
roboshop-gitops/
|
+-- README.md
|
+-- argocd/
|   +-- projects/
|   |   +-- roboshop-dev.yaml
|   |   +-- roboshop-qa.yaml
|   |   +-- roboshop-prod.yaml
|   |
|   +-- applications/
|   |   +-- dev/
|   |   +-- qa/
|   |   +-- prod/
|   |
|   +-- applicationsets/
|
+-- helm/
|   +-- roboshop/
|       +-- frontend/
|       +-- catalogue/
|       +-- cart/
|       +-- user/
|       +-- orders/
|       +-- payment/
|       +-- shipping/
|       +-- inventory/
|       +-- notification/
|
+-- environments/
|   +-- dev/
|   +-- qa/
|   +-- staging/
|   +-- prod/
|
+-- platform/
|   +-- namespaces/
|   +-- rbac/
|   +-- ingress/
|   +-- network-policies/
|   +-- secrets/
|   +-- resource-policies/
|
+-- monitoring/
|   +-- prometheus-rules/
|   +-- dashboards/
|   +-- alertmanager/
|
+-- scripts/
    +-- validate.sh
    +-- smoke-test.sh
    +-- rollback-check.sh
```

Do not duplicate complete Helm charts for each environment.

Keep shared templates and separate environment-specific values.

---

# 6. QA Deployment Prerequisites

Before starting, confirm the following.

## 6.1 DEV prerequisites

- DEV deployment completed.
- DEV Argo CD application is Synced.
- DEV application is Healthy.
- Smoke tests passed.
- Required integration tests passed.
- Image digest is recorded.
- Security scan results are available.
- No unresolved release-blocking vulnerabilities.
- DEV deployment evidence is available.

## 6.2 AWS prerequisites

- QA AWS account or approved environment is available.
- QA VPC exists.
- Private subnets are configured.
- EKS cluster is running.
- Worker nodes are Ready.
- ECR repositories are available.
- Required IAM roles exist.
- Required network routes are configured.
- Database endpoints are available.
- Required secrets are present.

## 6.3 GitOps prerequisites

- QA directory exists.
- QA Helm values exist.
- Argo CD can access the GitOps repository.
- QA AppProject exists.
- QA Application or ApplicationSet configuration exists.
- Branch protection is configured.
- Required reviewers are available.

## 6.4 Tool verification

```bash
aws --version
kubectl version --client
helm version
argocd version
git --version
docker --version
```

---

# 7. Verify the DEV Release Artifact

Do not start QA deployment before verifying the DEV release.

Capture the following:

```text
Application name
Source commit
CI pipeline ID
Docker image tag
ECR repository
ECR digest
DEV GitOps commit
DEV Argo CD revision
DEV deployment status
DEV test results
Security scan results
```

Example release metadata:

```yaml
release:
  application: cart
  environment: dev

  source:
    repository: roboshop-cart
    commit: "<SOURCE_COMMIT>"

  image:
    repository: "<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/roboshop/cart"
    tag: "<GIT_SHA>"
    digest: "sha256:<ACTUAL_DIGEST>"

  validation:
    unitTests: passed
    integrationTests: passed
    trivy: passed
    sonarqube: passed
    devSmokeTests: passed
```

This is an illustrative release record. Populate it from actual CI results.

## 7.1 Retrieve image digest

```bash
aws ecr describe-images \
  --repository-name roboshop/cart \
  --image-ids imageTag=<GIT_SHA> \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Save the returned digest.

Example format:

```text
sha256:0123456789abcdef...
```

Do not copy an illustrative digest into the real configuration.

## 7.2 Verify the image exists

```bash
aws ecr describe-images \
  --repository-name roboshop/cart \
  --image-ids imageDigest=<IMAGE_DIGEST>
```

## 7.3 Verify the image manifest

```bash
aws ecr batch-get-image \
  --repository-name roboshop/cart \
  --image-ids imageDigest=<IMAGE_DIGEST>
```

Confirm the expected artifact exists.

## 7.4 Important promotion rule

Do not use:

```yaml
image:
  tag: latest
```

Do not rebuild the image during QA deployment.

Use:

```text
repository@sha256:<ACTUAL_DIGEST>
```

The digest identifies the exact artifact.

---

# 8. Connect to QA EKS

Configure the AWS CLI credentials using the approved IAM identity.

Verify the active identity:

```bash
aws sts get-caller-identity
```

Update kubeconfig:

```bash
aws eks update-kubeconfig \
  --region <AWS_REGION> \
  --name <QA_CLUSTER_NAME> \
  --alias roboshop-qa
```

Check contexts:

```bash
kubectl config get-contexts
```

Switch to QA:

```bash
kubectl config use-context roboshop-qa
```

Verify the context:

```bash
kubectl config current-context
```

Expected:

```text
roboshop-qa
```

Verify cluster access:

```bash
kubectl get nodes
kubectl get namespaces
kubectl cluster-info
```

Check node details:

```bash
kubectl get nodes -o wide
kubectl describe nodes
```

Check node capacity:

```bash
kubectl top nodes
```

If metrics are unavailable, verify that the Metrics Server or approved metrics provider is installed and healthy.

**Safety rule:** Never assume the active kubectl context is QA. Verify it before applying or inspecting environment-specific resources.

---

# 9. QA Namespace Creation

Use a dedicated namespace.

File:

`platform/namespaces/qa/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: roboshop-qa
  labels:
    environment: qa
    application: roboshop
    managed-by: argocd
```

Apply through the approved GitOps platform configuration.

For a controlled bootstrap only:

```bash
kubectl apply -f platform/namespaces/qa/namespace.yaml
```

Verify:

```bash
kubectl get namespace roboshop-qa
kubectl describe namespace roboshop-qa
```

The namespace must not be shared with DEV or PROD.

---

# 10. ResourceQuota

File:

`platform/namespaces/qa/resource-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: roboshop-qa-quota
  namespace: roboshop-qa
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "150"
    services: "60"
    persistentvolumeclaims: "30"
    secrets: "100"
    configmaps: "150"
```

These values are example capacity limits, not universal production values.

Validate actual cluster capacity and expected QA test load before adoption.

Apply and verify:

```bash
kubectl apply -f platform/namespaces/qa/resource-quota.yaml

kubectl describe resourcequota \
  roboshop-qa-quota \
  -n roboshop-qa
```

Monitor:

```bash
kubectl get resourcequota -n roboshop-qa
```

---

# 11. LimitRange

File:

`platform/namespaces/qa/limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: roboshop-qa-limits
  namespace: roboshop-qa
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 10m
        memory: 32Mi
      max:
        cpu: "4"
        memory: 8Gi
```

Verify:

```bash
kubectl get limitrange -n roboshop-qa
kubectl describe limitrange roboshop-qa-limits -n roboshop-qa
```

---

# 12. QA RBAC

QA testers generally need read access to application resources and logs.

Developers should not automatically receive cluster administrator permissions.

File:

`platform/rbac/qa/qa-readonly-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: qa-readonly
  namespace: roboshop-qa
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - services
      - endpoints
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

  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
      - networkpolicies
    verbs:
      - get
      - list
      - watch
```

File:

`platform/rbac/qa/qa-readonly-binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: qa-readonly-binding
  namespace: roboshop-qa
subjects:
  - kind: Group
    name: roboshop-qa-testers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: qa-readonly
  apiGroup: rbac.authorization.k8s.io
```

The group must exist in the configured EKS authentication and identity-provider integration.

Verify:

```bash
kubectl auth can-i get pods \
  -n roboshop-qa \
  --as-group=roboshop-qa-testers

kubectl auth can-i delete deployments \
  -n roboshop-qa \
  --as-group=roboshop-qa-testers
```

Validate with an actual authenticated QA user as well.

---

# 13. ServiceAccount and Workload Identity

Applications should use dedicated Kubernetes ServiceAccounts.

Do not distribute AWS access keys through environment variables or container images.

File:

`helm/roboshop/cart/templates/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cart
  namespace: {{ .Release.Namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.serviceAccount.roleArn | quote }}
automountServiceAccountToken: false
```

Example QA values:

```yaml
serviceAccount:
  create: true
  name: cart
  roleArn: arn:aws:iam::<ACCOUNT_ID>:role/roboshop-qa-cart-role
```

This example uses the EKS IRSA annotation pattern.

If the cluster uses EKS Pod Identity, implement the corresponding Pod Identity association instead. Do not configure both mechanisms blindly.

## 13.1 IAM trust policy

For IRSA, the trust relationship must identify:

- Correct AWS account.
- Correct OIDC provider.
- Correct namespace.
- Correct ServiceAccount.
- Correct audience.

Example trust policy structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com",
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:roboshop-qa:cart"
        }
      }
    }
  ]
}
```

The role's permissions must be limited to the exact AWS resources required by the service.

---

# 14. ECR Image Access

EKS worker nodes need permission to pull private ECR images.

For the normal EKS EC2 worker-node model, validate the node IAM role's ECR pull permissions.

For example, the role may require the AWS-managed `AmazonEC2ContainerRegistryPullOnly` policy, or a reviewed equivalent least-privilege policy.

Do not give the application ServiceAccount broad ECR permissions merely because the Pod cannot pull an image.

Verify the image:

```bash
aws ecr describe-images \
  --repository-name roboshop/cart \
  --image-ids imageDigest=<IMAGE_DIGEST>
```

If the image is cross-account, configure the required ECR repository policy and IAM permissions.

---

# 15. QA Configuration Management

Application configuration should be separate from sensitive credentials.

Example:

`helm/roboshop/cart/values-qa.yaml`

```yaml
environment: qa

replicaCount: 2

image:
  repository: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/roboshop/cart
  digest: sha256:<ACTUAL_DIGEST>
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

config:
  LOG_LEVEL: INFO
  ENVIRONMENT: qa
```

The port and application-specific settings must match the actual Cart service.

## 15.1 ConfigMap

File:

`helm/roboshop/cart/templates/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cart-config
  namespace: {{ .Release.Namespace }}
data:
  ENVIRONMENT: {{ .Values.environment | quote }}
  LOG_LEVEL: {{ .Values.config.LOG_LEVEL | quote }}
```

Do not put passwords, tokens or private keys in ConfigMaps.

---

# 16. QA Secrets

Use AWS Secrets Manager and the approved External Secrets integration.

Example ExternalSecret:

`helm/roboshop/cart/templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cart-secrets
  namespace: {{ .Release.Namespace }}
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: roboshop-qa-secret-store
    kind: ClusterSecretStore

  target:
    name: cart-secrets
    creationPolicy: Owner

  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: roboshop/qa/cart
        property: databasePassword
```

The exact API version, SecretStore name and remote secret structure must match the installed External Secrets Operator configuration.

Verify:

```bash
kubectl get externalsecrets -n roboshop-qa

kubectl describe externalsecret \
  cart-secrets \
  -n roboshop-qa

kubectl get secret cart-secrets -n roboshop-qa
```

Do not print secret contents during validation.

---

# 17. Database Connectivity

QA must use QA databases.

Never accidentally connect QA workloads to production databases.

Document the dependencies of every service.

Example dependency inventory:

| Service | Dependency | QA endpoint |
|---|---|---|
| Cart | Redis, if required | QA Redis endpoint |
| Catalogue | Its configured database | QA database endpoint |
| User | Its configured database | QA database endpoint |
| Orders | Its configured database | QA database endpoint |
| Payment | Its configured database or external provider sandbox | QA endpoint |
| Shipping | Its configured database or dependency | QA endpoint |
| Notification | Its configured messaging provider | QA endpoint |

This is an example mapping. Use the actual application dependency inventory rather than assuming all services share the same database.

## 17.1 Network validation

Check DNS resolution from a temporary diagnostic Pod:

```bash
kubectl run dns-check \
  -n roboshop-qa \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Then:

```bash
kubectl exec -n roboshop-qa dns-check -- \
  nslookup <QA_DATABASE_HOST>
```

Check TCP connectivity using an approved diagnostic image that contains the necessary tools:

```bash
kubectl exec -n roboshop-qa <DIAGNOSTIC_POD> -- \
  nc -vz <QA_DATABASE_HOST> <DATABASE_PORT>
```

A successful TCP connection does not prove authentication or application-level correctness.

## 17.2 Connection pool validation

For Java Spring Boot applications, validate the application's actual datasource and HikariCP configuration.

Example:

```yaml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

These are illustrative settings. Calculate the maximum connection budget across all replicas and compare it with the database connection limit.

## 17.3 Database migration strategy

QA deployment must validate database migrations.

A safe sequence is:

```text
Review migration
      |
      v
Backup / recovery readiness
      |
      v
Run migration
      |
      v
Verify schema
      |
      v
Deploy compatible application
      |
      v
Integration tests
```

Use backward-compatible expand-and-contract migrations where application versions may overlap.

Do not assume a Kubernetes Deployment rollback automatically reverses database schema changes.

---

# 18. Redis and Messaging Dependencies

Where the actual application architecture requires Redis:

- Use the QA Redis endpoint.
- Verify DNS.
- Verify network access.
- Verify authentication.
- Validate TLS if required.
- Test application operations.
- Monitor connection failures and latency.

Where the application requires RabbitMQ or Kafka:

- Use isolated QA brokers or approved QA namespaces.
- Use separate credentials.
- Use QA-specific exchanges, queues or topics.
- Validate producer and consumer connectivity.
- Test retry and dead-letter behavior.
- Validate message processing and duplicate handling.

For Kafka, validate consumer lag and partition assignments.

For RabbitMQ, validate queue depth, consumer count and unacknowledged messages.

Do not use production queues for QA testing.

---

# 19. Helm Chart Validation

Before Argo CD deployment, validate the chart.

```bash
helm lint helm/roboshop/cart
```

Render the QA manifests:

```bash
helm template cart \
  helm/roboshop/cart \
  --namespace roboshop-qa \
  -f helm/roboshop/cart/values.yaml \
  -f helm/roboshop/cart/values-qa.yaml
```

Validate Kubernetes schemas using the approved schema-validation tool.

Inspect:

- Image digest.
- Namespace.
- ServiceAccount.
- Resource requests and limits.
- Probes.
- ConfigMap references.
- Secret references.
- Service ports.
- NetworkPolicy.
- Ingress configuration.
- Autoscaling settings.
- Pod security context.

Check that no DEV or PROD endpoints appear in the rendered QA configuration.

---

# 20. QA Helm Values for Multiple Services

Keep a separate QA values file for each application where the charts are independent.

Example:

```text
environments/qa/
|
+-- frontend-values.yaml
+-- catalogue-values.yaml
+-- cart-values.yaml
+-- user-values.yaml
+-- orders-values.yaml
+-- payment-values.yaml
+-- shipping-values.yaml
+-- inventory-values.yaml
+-- notification-values.yaml
```

For the remaining microservices, create corresponding values files from the actual service inventory.

Each values file should define:

```text
Environment
Image repository
Image digest
Replica count
Resource requests
Resource limits
Service port
Health probes
Config references
Secret references
ServiceAccount
Autoscaling
Network policy requirements
Monitoring labels
```

Do not copy arbitrary values from DEV. Review QA capacity and test requirements.

---

# 21. Argo CD QA AppProject

File:

`argocd/projects/roboshop-qa.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: roboshop-qa
  namespace: argocd
spec:
  description: RoboShop QA environment

  sourceRepos:
    - https://github.com/<ORG>/roboshop-gitops.git

  destinations:
    - name: roboshop-qa-cluster
      namespace: roboshop-qa

  namespaceResourceWhitelist:
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: ""
      kind: Service
    - group: ""
      kind: ServiceAccount
    - group: apps
      kind: Deployment
    - group: apps
      kind: StatefulSet
    - group: networking.k8s.io
      kind: Ingress
    - group: networking.k8s.io
      kind: NetworkPolicy
    - group: autoscaling
      kind: HorizontalPodAutoscaler
    - group: policy
      kind: PodDisruptionBudget
    - group: external-secrets.io
      kind: ExternalSecret

  clusterResourceWhitelist: []
```

Add only the resource kinds required by the real application.

If a cluster-scoped resource is required, manage it through a separate platform-owned project with controlled permissions.

---

# 22. Argo CD QA Application

Example:

`argocd/applications/qa/cart.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cart-qa
  namespace: argocd
spec:
  project: roboshop-qa

  source:
    repoURL: https://github.com/<ORG>/roboshop-gitops.git
    targetRevision: main
    path: helm/roboshop/cart
    helm:
      valueFiles:
        - values.yaml
        - values-qa.yaml

  destination:
    name: roboshop-qa-cluster
    namespace: roboshop-qa

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
```

Important:

- Register the QA cluster with Argo CD first.
- Ensure the destination name matches the registered cluster.
- Ensure the repository path and Helm value files actually exist.
- Ensure the namespace is created through the approved platform bootstrap process.
- Confirm the installed Argo CD version supports the configured fields.

QA auto-sync is a policy choice. If QA testing requires a controlled release window, use manual or approval-controlled sync instead.

---

# 23. ApplicationSet for QA

For many microservices, an ApplicationSet can reduce repetitive Application definitions.

Example concept:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: roboshop-qa-services
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - service: cart
            chartPath: helm/roboshop/cart
          - service: catalogue
            chartPath: helm/roboshop/catalogue
          - service: user
            chartPath: helm/roboshop/user
          - service: orders
            chartPath: helm/roboshop/orders
          - service: payment
            chartPath: helm/roboshop/payment
          - service: shipping
            chartPath: helm/roboshop/shipping

  template:
    metadata:
      name: '{{service}}-qa'

    spec:
      project: roboshop-qa

      source:
        repoURL: https://github.com/<ORG>/roboshop-gitops.git
        targetRevision: main
        path: '{{chartPath}}'
        helm:
          valueFiles:
            - values.yaml
            - values-qa.yaml

      destination:
        name: roboshop-qa-cluster
        namespace: roboshop-qa

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This is an example generator. Extend it to the complete approved service inventory and verify every chart path.

Do not deploy two competing Argo CD Applications to the same resources.

---

# 24. QA Argo CD RBAC

QA deployment permissions must be isolated from DEV and PROD.

Conceptual RBAC:

```csv
g, roboshop-qa-testers, role:qa-readonly
g, roboshop-qa-deployers, role:qa-deployer

p, role:qa-readonly, applications, get, roboshop-qa/*, allow
p, role:qa-readonly, projects, get, *, allow

p, role:qa-deployer, applications, get, roboshop-qa/*, allow
p, role:qa-deployer, applications, sync, roboshop-qa/*, allow
p, role:qa-deployer, projects, get, *, allow
```

Use the actual Argo CD application naming conventions and group integration.

Verify:

- QA testers cannot sync PROD.
- QA deployers cannot change platform-wide resources.
- QA users cannot administer Argo CD.
- Platform administrators retain controlled break-glass access.

---

# 25. QA Promotion Pull Request

The promotion process begins with a GitOps pull request.

The PR should show:

```text
Application: cart

Source environment: DEV
Target environment: QA

Old QA image digest:
sha256:<OLD_DIGEST>

New QA image digest:
sha256:<DEV_VALIDATED_DIGEST>

Source commit:
<SOURCE_COMMIT>

CI pipeline:
<PIPELINE_ID>

DEV deployment:
SUCCESS

Security validation:
PASSED

QA configuration review:
PENDING
```

Reviewers should verify:

1. Exact image digest.
2. Correct ECR repository.
3. Correct QA namespace.
4. Correct database endpoints.
5. Correct secret references.
6. Correct replica count.
7. Resource configuration.
8. Network policies.
9. Ingress and DNS.
10. Test readiness.

After approval:

```bash
git checkout main
git pull

git checkout -b promote/cart-to-qa

git add environments/qa/

git commit -m "promote cart image to QA"

git push origin promote/cart-to-qa
```

Open a pull request and follow repository approval policies.

Do not directly push protected environment changes.

---

# 26. Argo CD Synchronization

After the approved GitOps change is merged, Argo CD reconciles the QA application.

Check:

```bash
argocd app list
```

Inspect:

```bash
argocd app get cart-qa
```

Review the diff:

```bash
argocd app diff cart-qa
```

If QA uses manual synchronization:

```bash
argocd app sync cart-qa
```

Wait:

```bash
argocd app wait cart-qa \
  --sync \
  --health
```

Check status:

```bash
argocd app get cart-qa
```

Expected:

```text
Sync Status: Synced
Health Status: Healthy
```

An Argo CD Healthy status is not proof of business functionality. Continue with application and integration tests.

---

# 27. Kubernetes Deployment Validation

Verify:

```bash
kubectl get deployments -n roboshop-qa
kubectl get replicasets -n roboshop-qa
kubectl get pods -n roboshop-qa -o wide
kubectl get services -n roboshop-qa
kubectl get endpointslices -n roboshop-qa
```

Check Deployment:

```bash
kubectl describe deployment cart -n roboshop-qa
```

Check Pods:

```bash
kubectl describe pods -n roboshop-qa
```

Check events:

```bash
kubectl get events \
  -n roboshop-qa \
  --sort-by=.lastTimestamp
```

Validate image digest in the actual Pod specification:

```bash
kubectl get deployment cart \
  -n roboshop-qa \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Check actual running image IDs:

```bash
kubectl get pods \
  -n roboshop-qa \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.imageID}{"\n"}{end}{end}'
```

Compare the image identity with the promoted release digest.

---

# 28. Pod Health and Probes

Verify all application containers define appropriate health checks.

Example:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  periodSeconds: 20
  failureThreshold: 3
```

The endpoint names above are examples. Use endpoints implemented by the actual application.

Understand the difference:

- Startup probe: application has completed startup.
- Readiness probe: application can receive traffic.
- Liveness probe: application process should continue running.

A temporary database outage should not automatically cause a liveness restart storm.

Check:

```bash
kubectl get pods -n roboshop-qa
kubectl describe pod <POD_NAME> -n roboshop-qa
kubectl logs <POD_NAME> -n roboshop-qa
kubectl logs <POD_NAME> -n roboshop-qa --previous
```

---

# 29. Service and Internal DNS Validation

Verify:

```bash
kubectl get services -n roboshop-qa
kubectl get endpointslices -n roboshop-qa
```

Example Kubernetes DNS:

```text
cart.roboshop-qa.svc.cluster.local
```

From a diagnostic Pod:

```bash
nslookup cart.roboshop-qa.svc.cluster.local
```

Validate:

- Service selector matches Pod labels.
- Service port matches targetPort.
- Pods are Ready.
- EndpointSlices contain the expected endpoints.
- NetworkPolicy allows required traffic.
- Application listens on the expected interface and port.

Do not confuse successful DNS resolution with successful application connectivity.

---

# 30. NetworkPolicy Validation

Apply default-deny ingress only after defining the required communication matrix.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cart-ingress
  namespace: roboshop-qa
spec:
  podSelector:
    matchLabels:
      app: cart

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: roboshop-qa
      ports:
        - protocol: TCP
          port: 8080
```

This example allows ingress from Pods in the QA namespace.

It is not a complete production policy.

Add explicit rules for:

- Ingress controller traffic.
- Required application dependencies.
- DNS.
- Monitoring scrapes.
- Logging agents.
- Database connectivity.
- Messaging.
- External service access.

NetworkPolicy enforcement depends on the installed CNI and its supported features. Validate the actual EKS networking implementation.

---

# 31. QA ALB Ingress

QA should have its own DNS name and external access policy.

Example:

```text
qa.example.com
```

File:

`environments/qa/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: roboshop-qa
  namespace: roboshop-qa
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: <QA_ACM_CERTIFICATE_ARN>
spec:
  ingressClassName: alb
  rules:
    - host: qa.example.com
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

This example assumes an internet-facing QA environment.

If QA is internal-only, use an internal ALB and approved private DNS instead.

Validate the certificate, target type, health-check path, security groups and listener configuration.

Verify:

```bash
kubectl get ingress -n roboshop-qa
kubectl describe ingress roboshop-qa -n roboshop-qa
```

Check AWS target-group health and ALB listener rules.

---

# 32. Route 53 and TLS Validation

Create or update the QA DNS record using the approved DNS process.

Example:

```text
qa.example.com
    |
    v
QA ALB
```

Validate DNS:

```bash
nslookup qa.example.com
```

Validate HTTPS:

```bash
curl -I https://qa.example.com
```

Validate certificate details:

```bash
openssl s_client \
  -connect qa.example.com:443 \
  -servername qa.example.com
```

Check:

- Certificate validity.
- Certificate hostname.
- Expiration.
- Certificate chain.
- HTTPS redirect.
- ALB listener.
- Backend target health.

---

# 33. Integration Testing

QA's main purpose is to validate interactions between components.

The integration test plan should cover:

```text
Frontend
   |
   v
Catalogue
   |
   v
Cart
   |
   v
User
   |
   v
Orders
   |
   v
Payment
   |
   v
Shipping
   |
   v
Notification
```

This is a logical business flow. Validate the actual application API and dependency sequence before implementing the tests.

Test categories:

1. Service-to-service communication.
2. Database reads and writes.
3. Redis operations.
4. Message publishing.
5. Message consumption.
6. API authentication.
7. Authorization.
8. Error handling.
9. Retry behavior.
10. Idempotency.
11. Timeout behavior.
12. Dependency failure handling.

Do not use production customer data for QA.

---

# 34. API Testing

Example API checks:

```bash
curl -f https://qa.example.com/health
```

For actual business APIs, use the application's documented endpoints.

Example:

```bash
curl -i \
  -H "Content-Type: application/json" \
  https://qa.example.com/<API_PATH>
```

For authenticated endpoints:

```bash
curl -i \
  -H "Authorization: Bearer <QA_TEST_TOKEN>" \
  https://qa.example.com/<API_PATH>
```

Never commit test tokens into Git.

Test:

- HTTP status codes.
- Response schemas.
- Authentication.
- Authorization.
- Validation errors.
- Duplicate requests.
- Timeout handling.
- Invalid input.
- Rate limits.
- Dependency failures.

---

# 35. End-to-End Smoke Test

File:

`scripts/qa-smoke-test.sh`

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

BASE_URL="${BASE_URL:-https://qa.example.com}"

check_endpoint() {
    local path="$1"
    local expected_code="${2:-200}"

    local actual_code

    actual_code=$(
        curl -sS \
          -o /tmp/qa-response.txt \
          -w "%{http_code}" \
          --connect-timeout 5 \
          --max-time 20 \
          "${BASE_URL}${path}"
    )

    if [[ "$actual_code" != "$expected_code" ]]; then
        echo "FAILED: ${path}"
        echo "Expected: ${expected_code}"
        echo "Actual: ${actual_code}"
        cat /tmp/qa-response.txt
        return 1
    fi

    echo "PASSED: ${path}"
}

echo "Starting QA smoke tests"

check_endpoint "/health" 200

echo "QA smoke tests completed"
```

Make executable:

```bash
chmod +x scripts/qa-smoke-test.sh
```

Run:

```bash
BASE_URL=https://qa.example.com \
  ./scripts/qa-smoke-test.sh
```

Add actual service-specific checks based on the application API contract.

Do not treat a single health endpoint as a complete end-to-end test.

---

# 36. Database Integration Tests

Validate:

- Database DNS resolution.
- Network connectivity.
- Authentication.
- Connection pool behavior.
- Schema version.
- Migration completion.
- Read operations.
- Write operations.
- Transaction rollback.
- Connection recovery.
- Timeout handling.

Example test sequence:

```text
Application starts
      |
      v
Database connection succeeds
      |
      v
Schema version validated
      |
      v
Test record inserted
      |
      v
Test record read
      |
      v
Transaction committed
      |
      v
Cleanup completed
```

Use controlled test records and avoid destructive tests against shared QA data.

---

# 37. Messaging Integration Tests

For RabbitMQ:

- Verify broker connectivity.
- Verify credentials.
- Verify exchange.
- Verify queue binding.
- Publish a test message.
- Verify consumer processing.
- Verify retry behavior.
- Verify dead-letter handling.

For Kafka:

- Verify bootstrap connectivity.
- Verify authentication.
- Verify topic existence.
- Produce a test event.
- Consume the event.
- Verify partition assignment.
- Check consumer lag.
- Test retry and duplicate processing.

Example:

```text
Test Producer
     |
     v
QA Topic / Queue
     |
     v
QA Consumer
     |
     v
Expected Result
```

Keep test messages isolated from production traffic.

---

# 38. Security Validation

QA deployment should validate security controls before promotion.

## 38.1 Image security

Confirm:

- Image came from approved ECR.
- Digest matches DEV.
- Trivy results are available.
- Critical findings follow the organization's policy.
- Base image is approved.
- No secrets are embedded in the image.

## 38.2 Kubernetes security

Inspect:

```bash
kubectl get pods -n roboshop-qa -o yaml
```

Validate:

- Containers do not run as root where avoidable.
- Privilege escalation is disabled.
- Capabilities are dropped.
- Read-only root filesystem is used where compatible.
- ServiceAccount permissions are least privilege.
- Secret access is restricted.
- Network policies are enforced.
- Resource requests and limits are defined.

Example security context:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

Validate compatibility with the application image before enforcement.

## 38.3 Vulnerability scanning

Use the approved Trivy scanning pipeline.

Example:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  <ECR_IMAGE>@sha256:<ACTUAL_DIGEST>
```

Use the same image digest that QA is deploying.

## 38.4 Secret validation

Check that:

- No credentials are committed to Git.
- No passwords appear in application logs.
- Secret references point to QA secrets.
- IAM access is restricted.
- Rotation behavior is documented.

---

# 39. Prometheus Validation

QA services should expose metrics according to the monitoring architecture.

Check ServiceMonitor resources:

```bash
kubectl get servicemonitor -A
```

Check Prometheus targets in the Prometheus UI.

Validate:

- Target discovered.
- Target is UP.
- Scrape interval is appropriate.
- Metrics endpoint responds.
- Service labels match the ServiceMonitor selector.
- Namespace selection is correct.

If the environment does not use Prometheus Operator, validate the equivalent scrape configuration instead.

---

# 40. Grafana Validation

Create or reuse QA dashboards.

Recommended panels:

```text
QA Pod CPU
QA Pod Memory
Container Restarts
Deployment Replicas
HTTP Request Rate
HTTP Error Rate
Request Latency
Database Connection Pool
Redis Connections
Queue Depth
Kafka Consumer Lag
Argo CD Sync Status
```

Use actual application metrics and exporter metrics.

Avoid inventing metric names. Confirm the exported metrics before building PromQL panels.

---

# 41. ELK Logging Validation

Validate the full log path:

```text
Application Container
       |
       v
stdout / stderr
       |
       v
Log Collector
       |
       v
Logstash
       |
       v
Elasticsearch
       |
       v
Kibana
```

Check:

- QA application logs arrive.
- Environment label is `qa`.
- Service name is present.
- Timestamp is correct.
- Log level is parsed.
- Exceptions are searchable.
- Sensitive values are not logged.
- Retention policy is appropriate.

Example fields:

```json
{
  "environment": "qa",
  "service": "cart",
  "level": "INFO",
  "message": "Cart request processed",
  "trace_id": "<OPTIONAL_CORRELATION_ID>",
  "timestamp": "<UTC_TIMESTAMP>"
}
```

Do not add Jaeger or OpenTelemetry to this capstone's established observability stack.

---

# 42. Performance Testing

QA should validate application behavior under expected test traffic.

Use the approved load-testing tool, such as k6.

Test:

1. Baseline response time.
2. Expected request rate.
3. Increasing concurrency.
4. Sustained load.
5. Short traffic spike.
6. Database saturation behavior.
7. Queue processing behavior.
8. Recovery after load.

Example k6 script:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '2m',
};

export default function () {
  const response = http.get(
    'https://qa.example.com/health'
  );

  check(response, {
    'status is 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

Run:

```bash
k6 run qa-load-test.js
```

This is a small baseline test, not a full production capacity test.

Do not run unrestricted load tests against shared databases or external payment systems.

Record:

- Virtual users.
- Request rate.
- p50 latency.
- p95 latency.
- p99 latency.
- Error rate.
- CPU.
- Memory.
- Database connections.
- Queue depth.
- Pod restarts.

Define acceptance thresholds with the application and QA owners before testing.

---

# 43. Autoscaling Validation

If HPA is enabled in QA:

```bash
kubectl get hpa -n roboshop-qa
kubectl describe hpa cart -n roboshop-qa
```

Check:

```bash
kubectl top pods -n roboshop-qa
```

Validate:

- Metrics are available.
- HPA target is correct.
- Minimum replicas are respected.
- Maximum replicas are respected.
- Scaling occurs under load.
- Scaling stabilizes after load.
- Downstream dependencies can handle increased traffic.

Do not assume HPA will work simply because the YAML exists.

---

# 44. Failure Testing

QA should verify expected failure behavior.

Examples:

| Scenario | Expected behavior |
|---|---|
| Pod terminates | Replacement Pod starts |
| Worker node fails | Workload reschedules where capacity exists |
| Database unavailable | Application handles dependency failure |
| Redis unavailable | Defined retry/error behavior |
| Messaging unavailable | Defined retry or queue behavior |
| Invalid credentials | Access is rejected safely |
| Image unavailable | Deployment does not become healthy |
| Readiness fails | Pod is removed from ready endpoints |
| Application crashes | Kubernetes restarts it according to policy |
| ALB target unhealthy | Traffic is not routed to unhealthy targets |

Perform controlled tests only.

Do not inject failures into shared infrastructure without an approved test window and recovery plan.

---

# 45. QA Deployment Rollback

Rollback must be tested before the release is approved.

## 45.1 GitOps rollback

Identify the previous known-good QA GitOps commit:

```bash
git log --oneline environments/qa/
```

Revert the bad promotion:

```bash
git revert <BAD_PROMOTION_COMMIT>
git push
```

Argo CD reconciles the previous desired state.

## 45.2 Argo CD history

```bash
argocd app history cart-qa
```

Inspect:

```bash
argocd app get cart-qa
```

Git should remain the source of truth for the normal GitOps rollback process.

## 45.3 Rollback validation

After rollback:

```bash
kubectl rollout status deployment/cart \
  -n roboshop-qa

kubectl get pods -n roboshop-qa

kubectl get endpointslices -n roboshop-qa
```

Run the QA smoke tests again.

Validate:

- Previous image digest.
- Deployment health.
- Database compatibility.
- API response.
- Error rate.
- Logs.
- Business transaction behavior.

Remember: rolling back application code does not automatically reverse database migrations.

---

# 46. QA Troubleshooting

## 46.1 ImagePullBackOff

Commands:

```bash
kubectl describe pod <POD_NAME> -n roboshop-qa
kubectl get events -n roboshop-qa
```

Investigate:

- ECR repository.
- Image digest.
- Worker-node IAM permissions.
- Cross-account repository policy.
- Network access.
- Image architecture.

## 46.2 CrashLoopBackOff

Commands:

```bash
kubectl logs <POD_NAME> -n roboshop-qa
kubectl logs <POD_NAME> -n roboshop-qa --previous
kubectl describe pod <POD_NAME> -n roboshop-qa
```

Investigate:

- Application startup error.
- Missing configuration.
- Secret references.
- Database connection.
- Invalid arguments.
- Probe configuration.
- Memory limits.

## 46.3 Pending Pods

Commands:

```bash
kubectl describe pod <POD_NAME> -n roboshop-qa
kubectl describe nodes
kubectl get events -n roboshop-qa
```

Investigate:

- Insufficient CPU.
- Insufficient memory.
- Node selectors.
- Taints and tolerations.
- ResourceQuota.
- PVC binding.
- AZ capacity.

## 46.4 Argo CD OutOfSync

Commands:

```bash
argocd app get cart-qa
argocd app diff cart-qa
```

Investigate:

- Wrong repository path.
- Invalid Helm values.
- Invalid chart.
- Missing CRD.
- Destination mismatch.
- RBAC restrictions.
- Admission policy rejection.

## 46.5 Argo CD Degraded

Investigate:

```bash
kubectl get deployments -n roboshop-qa
kubectl get pods -n roboshop-qa
kubectl get events -n roboshop-qa
```

Review the unhealthy resource and its events.

## 46.6 ALB 502

Investigate:

```bash
kubectl get ingress -n roboshop-qa
kubectl get services -n roboshop-qa
kubectl get endpointslices -n roboshop-qa
kubectl get pods -n roboshop-qa
```

Check:

- ALB target health.
- Target group port.
- Service targetPort.
- Application listener.
- Security groups.
- Health-check path.
- Pod readiness.

## 46.7 Database Connection Failure

Investigate:

- Database endpoint.
- DNS.
- Security groups.
- Network ACLs.
- NetworkPolicy.
- Credentials.
- TLS.
- Connection pool.
- Database capacity.

## 46.8 ExternalSecret Not Ready

Commands:

```bash
kubectl describe externalsecret cart-secrets \
  -n roboshop-qa

kubectl get events -n roboshop-qa
```

Check:

- SecretStore health.
- IAM permissions.
- Secret ARN/name.
- Secret property.
- KMS access.
- External Secrets controller.

Never print actual secret values while troubleshooting.

---

# 47. Automated QA Deployment Validation

Create:

`scripts/validate-qa-deployment.sh`

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

NAMESPACE="roboshop-qa"

echo "Checking Kubernetes context"

CONTEXT="$(kubectl config current-context)"

if [[ "$CONTEXT" != "roboshop-qa" ]]; then
    echo "ERROR: Expected roboshop-qa context"
    exit 1
fi

echo "Checking namespace"

kubectl get namespace "$NAMESPACE"

echo "Checking deployments"

kubectl get deployments -n "$NAMESPACE"

echo "Checking pods"

kubectl get pods -n "$NAMESPACE"

echo "Checking services"

kubectl get services -n "$NAMESPACE"

echo "Checking unhealthy pods"

UNHEALTHY="$(
  kubectl get pods -n "$NAMESPACE" --no-headers |
  awk '$3 != "Running" && $3 != "Completed" {print}'
)"

if [[ -n "$UNHEALTHY" ]]; then
    echo "ERROR: Unhealthy Pods found"
    echo "$UNHEALTHY"
    exit 1
fi

echo "Checking deployment readiness"

kubectl get deployments -n "$NAMESPACE" \
  -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas

echo "QA Kubernetes validation completed"
```

Make executable:

```bash
chmod +x scripts/validate-qa-deployment.sh
```

Run:

```bash
./scripts/validate-qa-deployment.sh
```

Extend this script to validate all required deployments, exact replica counts, image digests and endpoint availability.

Do not use Pod phase alone as the final health signal. Check readiness and application behavior.

---

# 48. QA Release Evidence

Store evidence for each release.

Recommended structure:

```text
release-evidence/
|
+-- QA/
    |
    +-- <RELEASE_ID>/
        |
        +-- release-metadata.yaml
        +-- ci-pipeline.txt
        +-- image-digest.txt
        +-- trivy-report.json
        +-- sonarqube-report.txt
        +-- argocd-status.txt
        +-- deployment-status.txt
        +-- integration-test-results/
        +-- api-test-results/
        +-- performance-test-results/
        +-- security-test-results/
        +-- smoke-test-results.txt
        +-- grafana-screenshots/
        +-- kibana-query-results/
        +-- approval-record.txt
```

Do not store secrets, access tokens or sensitive customer information in evidence.

---

# 49. QA Acceptance Criteria

The QA deployment is ready for sign-off only when all required checks pass.

| Category | Acceptance criteria |
|---|---|
| Artifact | Same digest as DEV |
| ECR | Image available |
| EKS | Correct cluster |
| Namespace | QA isolation confirmed |
| Argo CD | Synced and Healthy |
| Deployment | Expected replicas ready |
| Pods | No unexplained failures |
| Services | Expected endpoints available |
| Configuration | QA endpoints used |
| Secrets | QA secrets resolved |
| Database | Required operations validated |
| Messaging | Required event flows validated |
| Ingress | Correct ALB and DNS |
| TLS | Valid certificate |
| API | Contract tests passed |
| Integration | Required tests passed |
| Security | Release policy passed |
| Monitoring | Prometheus targets healthy |
| Dashboards | QA metrics visible |
| Logging | QA logs searchable in ELK |
| Performance | Agreed thresholds met |
| Rollback | Procedure validated |
| Evidence | Results retained |
| Approval | QA owner signs off |

A failed mandatory criterion blocks promotion.

---

# 50. QA Sign-Off

Example:

```yaml
release:
  application: roboshop
  environment: qa
  releaseId: "<RELEASE_ID>"

artifact:
  repository: "<ECR_REPOSITORY>"
  digest: "sha256:<ACTUAL_DIGEST>"
  sourceCommit: "<SOURCE_COMMIT>"

deployment:
  cluster: "<QA_CLUSTER_NAME>"
  namespace: roboshop-qa
  argocdApplication: "<APPLICATION_NAME>"

validation:
  deployment: passed
  apiTests: passed
  integrationTests: passed
  securityTests: passed
  smokeTests: passed
  performanceTests: passed
  observability: passed
  rollbackTest: passed

approval:
  qaStatus: approved
  approvedBy: "<QA_OWNER>"
  approvalReference: "<APPROVAL_REFERENCE>"
```

Use the actual release evidence and approval process.

---

# 51. QA-to-Staging Promotion

After QA approval:

```text
QA VALIDATED
     |
     v
Release Evidence
     |
     v
QA Approval
     |
     v
Staging Promotion PR
     |
     v
Review and Approval
     |
     v
Staging GitOps
     |
     v
Argo CD
     |
     v
Staging EKS
```

The staging image must use the exact same digest.

Do not rebuild the image after QA approval.

Only the environment-specific configuration changes.

---

# 52. Final QA Deployment Flow

```text
1. Verify DEV release.
2. Capture image digest.
3. Verify ECR artifact.
4. Verify QA AWS identity.
5. Connect to QA EKS.
6. Verify cluster health.
7. Verify QA namespace.
8. Verify quotas and limits.
9. Verify RBAC.
10. Verify workload identity.
11. Verify ECR access.
12. Verify QA secrets.
13. Verify QA database connectivity.
14. Verify Redis and messaging dependencies.
15. Validate Helm values.
16. Render Helm manifests.
17. Validate Kubernetes resources.
18. Review QA AppProject.
19. Review Argo CD Application.
20. Create promotion PR.
21. Review and merge approved PR.
22. Wait for Argo CD reconciliation.
23. Verify application health.
24. Verify Services and EndpointSlices.
25. Verify ingress and DNS.
26. Verify TLS.
27. Run API tests.
28. Run integration tests.
29. Run end-to-end tests.
30. Run security validation.
31. Run performance tests.
32. Verify Prometheus.
33. Verify Grafana.
34. Verify ELK.
35. Validate failure behavior.
36. Validate rollback.
37. Collect evidence.
38. Obtain QA sign-off.
39. Promote the same digest to staging.
```

---

# 53. Chapter Completion Checklist

## Infrastructure

- [ ] QA EKS cluster verified
- [ ] AWS identity verified
- [ ] Worker nodes healthy
- [ ] Networking verified
- [ ] ECR access verified

## Kubernetes

- [ ] QA namespace created
- [ ] ResourceQuota configured
- [ ] LimitRange configured
- [ ] RBAC configured
- [ ] ServiceAccounts configured
- [ ] Workload identity verified
- [ ] NetworkPolicy validated

## Application

- [ ] Same DEV image digest promoted
- [ ] Helm templates validated
- [ ] ConfigMaps configured
- [ ] Secrets resolved
- [ ] Database connectivity verified
- [ ] Messaging dependencies verified
- [ ] Deployments healthy
- [ ] Services healthy
- [ ] Probes verified

## GitOps

- [ ] QA AppProject configured
- [ ] QA Application configured
- [ ] ApplicationSet validated where used
- [ ] Promotion PR approved
- [ ] Argo CD Synced
- [ ] Argo CD Healthy

## Networking

- [ ] ALB created
- [ ] Target groups healthy
- [ ] QA DNS configured
- [ ] TLS validated
- [ ] Application endpoints accessible

## Testing

- [ ] API tests passed
- [ ] Integration tests passed
- [ ] End-to-end tests passed
- [ ] Database tests passed
- [ ] Messaging tests passed
- [ ] Security tests passed
- [ ] Performance tests passed
- [ ] Failure tests completed

## Observability

- [ ] Prometheus targets healthy
- [ ] Grafana dashboards available
- [ ] ELK logs searchable
- [ ] Application errors reviewed
- [ ] Resource utilization reviewed

## Release

- [ ] Rollback validated
- [ ] Evidence collected
- [ ] QA approval received
- [ ] Staging promotion prepared

---

# 54. Important Production Lessons

1. QA must deploy the exact artifact validated in DEV.
2. A successful Kubernetes deployment does not prove application correctness.
3. A healthy Argo CD Application does not prove business functionality.
4. QA databases and messaging systems must be isolated from production.
5. Secrets must not be committed to Git.
6. Image tags alone are weaker than immutable digest references.
7. NetworkPolicy must be validated against actual CNI enforcement.
8. Database migrations require their own recovery strategy.
9. Performance testing must protect shared dependencies.
10. Rollback must be tested, not merely documented.
11. Monitoring and logging must be verified after deployment.
12. QA approval must be supported by recorded evidence.
13. The same image digest should move through the promotion lifecycle.
14. Environment-specific configuration must be reviewed independently.
15. Git remains the desired-state source of truth.

---

# 55. Final Chapter Outcome

After completing Chapter 201, the RoboShop application should be deployed and validated in the QA environment.

The completed lifecycle is:

```text
Application Source
       |
       v
CI Pipeline
       |
       v
Security Validation
       |
       v
ECR Immutable Image
       |
       v
DEV EKS
       |
       v
DEV Validation
       |
       v
QA Promotion
       |
       v
QA EKS
       |
       v
Integration Validation
       |
       v
Security and Performance Testing
       |
       v
QA Approval
       |
       v
STAGING Promotion
```

**The key production principle:**

Build once, validate in DEV, promote the same immutable artifact to QA, validate the complete application behavior, and only then authorize the next environment promotion.

The next chapter should implement **STAGING Deployment**, building on the exact artifact approved by QA.