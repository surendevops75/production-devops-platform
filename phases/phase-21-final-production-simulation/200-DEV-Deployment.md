# CHAPTER 200 — DEV DEPLOYMENT



Production-style implementation

AWS EKS + ArgoCD + Helm



Filename: `200-DEV-Deployment.md`

Project: RoboShop microservices\
Work item: `ECOM-4821`\
Microservice: `order-service`\
Deployment environment: DEV\
Image source: The exact ECR image digest published in Chapter 198.

## 200.1 — Objective

In Chapter 198, we implemented the artifact handoff between the build and publish jobs. The Docker image was built once, transferred between jobs, verified, and published to Amazon ECR.

Now we will deploy that exact image into the DEV EKS environment.

Our goal is to implement a complete deployment process:

1. Prepare the DEV EKS namespace.
2. Configure ECR image-pull permissions.
3. Configure Kubernetes ServiceAccount and workload identity where required.
4. Configure ConfigMaps and Secrets.
5. Configure Helm deployment.
6. Create an ArgoCD Application.
7. Deploy `order-service` into DEV.
8. Validate Deployment, Pods, Service and Ingress.
9. Verify database and microservice connectivity.
10. Validate Prometheus, Grafana and ELK integration.
11. Execute smoke tests.
12. Collect deployment evidence and hand over to QA.

Important: We will deploy using the ECR image digest, not rebuild the Docker image or use a mutable `latest` tag.

## 200.2 — DEV deployment architecture

We will follow this deployment architecture:

GitHub Actions

Chapter 198 — Build, verify and publish

Amazon ECR

order-service image identified by digest

GitOps Repository

DEV Helm values and Kubernetes configuration

ArgoCD

Detects Git changes and reconciles DEV

AWS EKS — DEV Cluster

Namespace: roboshop-dev

- order-service Deployment
- order-service Pods
- order-service Service
- order-service Ingress
- ConfigMap and Secret
- ServiceAccount

RDS / Dependencies

Database and service connectivity

Observability

Prometheus, Grafana, ELK

### Component responsibilities

| Component         | Responsibility                             |
| ----------------- | ------------------------------------------ |
| GitHub Actions    | Build and publish the image                |
| ECR               | Store the container image                  |
| GitOps repository | Maintain desired deployment configuration  |
| ArgoCD            | Synchronize Git configuration with EKS     |
| Helm              | Package and configure Kubernetes resources |
| EKS               | Run the application                        |
| Prometheus        | Collect metrics                            |
| Grafana           | Visualize metrics                          |
| ELK               | Collect and search application logs        |

## 200.3 — Understand the deployment ticket

Our deployment is associated with:

```
Ticket       : ECOM-4821
Application  : RoboShop
Service      : order-service
Environment  : DEV
Cluster      : roboshop-dev-eks
Namespace    : roboshop-dev
```

Before starting, we need the following information from the release process.

| Required information | Source                           |
| -------------------- | -------------------------------- |
| Commit SHA           | GitHub Actions                   |
| ECR repository       | Chapter 198                      |
| ECR image digest     | Publish job output               |
| Helm chart           | Application repository           |
| DEV values file      | GitOps repository                |
| DEV EKS cluster      | AWS                              |
| Database endpoint    | DEV infrastructure configuration |
| Dependency endpoints | DEV service configuration        |

Deployment rule: The image digest approved for this deployment must match the digest published by the preceding CI/CD workflow.

Do not substitute a newly built image or a different tag.

## 200.4 — Prepare the GitOps repository

We maintain application source code and deployment configuration separately.

Example repository structure:

```
roboshop-gitops/
│
├── environments/
│   │
│   └── dev/
│       │
│       ├── applications/
│       │   └── order-service.yaml
│       │
│       └── order-service/
│           ├── values.yaml
│           └── secrets-reference.yaml
│
└── charts/
    │
    └── order-service/
        │
        ├── Chart.yaml
        │
        ├── values.yaml
        │
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            ├── configmap.yaml
            └── serviceaccount.yaml
```

This repository contains the desired state of our DEV environment.

When developers update the DEV values file and push it to Git, ArgoCD detects the changes and reconciles the cluster.

### Why separate application and GitOps repositories?

Application repository:

- Application source code.
- Dockerfile.
- Unit tests.
- CI/CD workflow.
- Build scripts.

GitOps repository:

- Helm values.
- Environment-specific configurations.
- ArgoCD Applications.
- Deployment configuration.

This separation helps us manage application releases and infrastructure configuration independently.

## 200.5 — Connect to the DEV EKS cluster

First, identify the AWS account and region.

```
aws sts get-caller-identity
```

Expected example:

```
{
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/devops-user"
}
```

Update your kubeconfig:

```
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name roboshop-dev-eks
```

Verify the cluster:

```
kubectl cluster-info
```

Check the nodes:

```
kubectl get nodes -o wide
```

Check existing namespaces:

```
kubectl get namespaces
```

Check your current context:

```
kubectl config current-context
```

### Production validation

Before making changes, confirm:

- You are connected to the DEV cluster.
- Your AWS account is correct.
- Your Kubernetes permissions are appropriate.
- You are not connected to the production cluster.

A wrong Kubernetes context can result in deployment to the wrong environment.

## 200.6 — Prepare the DEV namespace

We will deploy the service into:

```
roboshop-dev
```

Create the namespace:

```
kubectl create namespace roboshop-dev
```

If it already exists, do not recreate it.

Verify:

```
kubectl get namespace roboshop-dev
```

Add environment labels:

```
kubectl label namespace roboshop-dev \
  environment=dev \
  application=roboshop \
  --overwrite
```

Verify:

```
kubectl get namespace roboshop-dev --show-labels
```

### Configure resource quotas

File:

```
namespace-quota.yaml
```

```
apiVersion: v1
kind: ResourceQuota

metadata:
  name: roboshop-dev-quota
  namespace: roboshop-dev

spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi

    limits.cpu: "16"
    limits.memory: 32Gi

    pods: "50"
    services: "30"
```

Apply:

```
kubectl apply -f namespace-quota.yaml
```

Verify:

```
kubectl describe resourcequota \
  -n roboshop-dev
```

A resource quota prevents workloads in the namespace from consuming resources beyond the configured limits.

The actual quota must be sized according to the DEV cluster's available capacity.

## 200.7 — Configure ECR image-pull permissions

This is an important part of the deployment.

The Kubernetes node needs permission to retrieve the image from ECR.

First, understand the difference between image-pull authentication and application AWS permissions.

| Requirement                        | Typical identity                                |
| ---------------------------------- | ----------------------------------------------- |
| Pull image from ECR                | EKS node IAM role or Fargate pod execution role |
| Application access to AWS services | IRSA or EKS Pod Identity                        |
| Kubernetes API access              | Kubernetes ServiceAccount and RBAC              |

For a standard EKS cluster using EC2 managed nodes, the node IAM role normally provides the ECR image-pull permissions.

### 200.7.1 — Identify the node group

Run:

```
aws eks list-nodegroups \
  --cluster-name roboshop-dev-eks \
  --region ap-south-1
```

Describe the node group:

```
aws eks describe-nodegroup \
  --cluster-name roboshop-dev-eks \
  --nodegroup-name roboshop-dev-ng \
  --region ap-south-1
```

Find the node IAM role ARN:

```
nodeRole:
arn:aws:iam::123456789012:role/roboshop-dev-node-role
```

### 200.7.2 — Verify ECR permissions

The node role generally needs:

```
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:GetDownloadUrlForLayer
ecr:BatchGetImage
```

For EC2-backed EKS nodes, AWS's managed `AmazonEC2ContainerRegistryReadOnly` policy can provide the standard read-only permissions.

Alternatively, configure a restricted customer-managed policy.

Example:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/roboshop/order-service"
    }
  ]
}
```

The exact repository ARN and AWS account must match your environment.

### 200.7.3 — Verify ECR repository access

```
aws ecr describe-repositories \
  --repository-names roboshop/order-service \
  --region ap-south-1
```

Important: Do not create a Kubernetes `imagePullSecret` unnecessarily when the standard EKS node IAM role already handles ECR authentication. Fargate and other cluster configurations may have different identity requirements.

## 200.8 — Configure the order-service image

We need the image repository and digest from the publishing workflow.

Example:

```
Repository:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/order-service

Digest:
sha256:REPLACE_WITH_ACTUAL_64_CHARACTER_DIGEST
```

Our final image reference will be:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/order-service@sha256:REPLACE_WITH_ACTUAL_64_CHARACTER_DIGEST
```

We will use this image reference throughout the deployment.

### Why use the digest?

Imagine two deployments:

```
Deployment A:
order-service:latest

Deployment B:
order-service@sha256:abc123...
```

The `latest` tag may point to a different image later.

The digest identifies a specific image manifest, making the deployment reference immutable.

## 200.9 — Configure the Helm chart

Helm allows us to package reusable Kubernetes resources and supply environment-specific values.

### 200.9.1 — Chart.yaml

File:

```
charts/order-service/Chart.yaml
```

```
apiVersion: v2

name: order-service

description: RoboShop Order Service Helm Chart

type: application

version: 1.0.0

appVersion: "1.0.0"
```

### 200.9.2 — Default values.yaml

File:

```
charts/order-service/values.yaml
```

```
replicaCount: 2

image:
  repository: ""
  digest: ""

containerPort: 8080

service:
  type: ClusterIP
  port: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

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
    initialDelaySeconds: 20
    periodSeconds: 20

config:
  NODE_ENV: production
  PORT: "8080"
  LOG_LEVEL: info

secret:
  existingSecret: order-service-secret

ingress:
  enabled: false
  className: alb
  host: ""
```

These are example settings. Confirm that the actual order-service application implements the configured health endpoints and uses the specified environment variables.

### 200.9.3 — DEV values.yaml

File:

```
environments/dev/order-service/values.yaml
```

```
replicaCount: 2

image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/order-service"

  digest: "sha256:REPLACE_WITH_ACTUAL_64_CHARACTER_DIGEST"

service:
  type: ClusterIP
  port: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service

  annotations:
    # Add the approved IRSA role annotation here
    # only if the application needs AWS permissions.
    {}

config:
  NODE_ENV: production
  PORT: "8080"
  LOG_LEVEL: info

secret:
  existingSecret: order-service-secret

ingress:
  enabled: true
  className: alb
  host: order-dev.example.com
```

Replace the placeholder digest with the real digest from Chapter 198.

Do not commit real credentials or database passwords to this repository.

## 200.10 — Create the Kubernetes ServiceAccount

A ServiceAccount gives a Kubernetes workload an identity within the cluster.

File:

```
charts/order-service/templates/serviceaccount.yaml
```

```
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount

metadata:
  name: {{ .Values.serviceAccount.name }}
  namespace: {{ .Release.Namespace }}

  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}

automountServiceAccountToken: false
{{- end }}
```

### When do we need IRSA or EKS Pod Identity?

Suppose `order-service` needs to read a message from an AWS SQS queue.

We should not give broad AWS permissions to every worker node simply because one application needs them.

Instead, associate the application's ServiceAccount with a restricted IAM role using IRSA or EKS Pod Identity.

For IRSA, the ServiceAccount can have:

```
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/order-service-dev-role
```

Use the correct role ARN and configure the corresponding IAM trust relationship.

If the application does not need to access AWS APIs, it does not need an application-specific IAM role.

Remember: ECR image pulling and application AWS permissions are two separate concerns.

## 200.11 — Configure ConfigMaps

A ConfigMap stores non-sensitive application configuration.

File:

```
charts/order-service/templates/configmap.yaml
```

```
apiVersion: v1
kind: ConfigMap

metadata:
  name: order-service-config
  namespace: {{ .Release.Namespace }}

data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

Example rendered ConfigMap:

```
apiVersion: v1
kind: ConfigMap

metadata:
  name: order-service-config
  namespace: roboshop-dev

data:
  NODE_ENV: "production"
  PORT: "8080"
  LOG_LEVEL: "info"
```

Check it after deployment:

```
kubectl get configmap \
  -n roboshop-dev
```

Inspect:

```
kubectl describe configmap order-service-config \
  -n roboshop-dev
```

### What belongs in ConfigMaps?

Examples:

- Application port.
- Environment name.
- Log level.
- Non-sensitive service endpoints.
- Feature flags.

Never store passwords, API keys or database credentials in ConfigMaps.

## 200.12 — Configure Secrets

A Secret stores sensitive configuration, such as database credentials.

For a demonstration, we can create a Kubernetes Secret from the command line.

```
kubectl create secret generic order-service-secret \
  --namespace roboshop-dev \
  --from-literal=DB_USERNAME=order_user \
  --from-literal=DB_PASSWORD='REPLACE_WITH_DEV_PASSWORD' \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

Verify:

```
kubectl get secrets \
  -n roboshop-dev
```

Describe:

```
kubectl describe secret order-service-secret \
  -n roboshop-dev
```

Kubernetes Secrets are not automatically encrypted merely because they are stored as Secret resources. Configure encryption at rest, restrict RBAC, and avoid exposing their values in logs or Git.

For a mature production implementation, consider integrating AWS Secrets Manager with an approved secrets synchronization solution.

### How does the application receive these values?

We will inject them into the container using `envFrom`.

The deployment template will reference:

```
envFrom:
  - configMapRef:
      name: order-service-config

  - secretRef:
      name: order-service-secret
```

The application can then read the variables using its runtime environment.

For example, in Node.js:

```
const dbUsername = process.env.DB_USERNAME;
const dbPassword = process.env.DB_PASSWORD;
```

## 200.13 — Create the Kubernetes Deployment

This is the main workload resource.

File:

```
charts/order-service/templates/deployment.yaml
```

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

  namespace: {{ .Release.Namespace }}

  labels:
    app: order-service

spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}

      containers:
        - name: order-service

          image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"

          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}

          envFrom:
            - configMapRef:
                name: order-service-config

            - secretRef:
                name: {{ .Values.secret.existingSecret }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http

            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}

          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http

            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}

          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            capabilities:
              drop:
                - ALL
```

### Understand the important fields

Replicas

```
replicas: 2
```

We request two application Pods.

Image

```
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

This ensures that Kubernetes pulls the exact image identified by the ECR digest.

ServiceAccount

```
serviceAccountName: order-service
```

The Pod uses the intended Kubernetes identity.

Readiness probe

The readiness probe checks whether the application is ready to receive traffic.

If readiness fails, Kubernetes removes that Pod from normal Service endpoint selection.

Liveness probe

The liveness probe checks whether the container should be restarted because it is unhealthy.

The two probes have different purposes. Do not make a readiness failure automatically restart an otherwise healthy application.

Security context

The example requires a non-root image. Confirm that the actual Docker image runs correctly as a non-root user before applying this setting.

## 200.14 — Create the Kubernetes Service

The Service provides a stable internal endpoint for the application.

File:

```
charts/order-service/templates/service.yaml
```

```
apiVersion: v1
kind: Service

metadata:
  name: order-service

  namespace: {{ .Release.Namespace }}

  labels:
    app: order-service

spec:
  type: {{ .Values.service.type }}

  selector:
    app: order-service

  ports:
    - name: http
      port: {{ .Values.service.port }}

      targetPort: http

      protocol: TCP
```

After deployment:

```
kubectl get services \
  -n roboshop-dev
```

Expected:

```
NAME            TYPE        CLUSTER-IP
order-service   ClusterIP   10.x.x.x
```

Other microservices inside the cluster can communicate with the order-service using Kubernetes DNS.

Example:

```
http://order-service.roboshop-dev.svc.cluster.local:8080
```

This assumes the application listens on port `8080`.

## 200.15 — Configure Ingress

If external or shared DEV access is required, we can expose the Service using an AWS Application Load Balancer through the AWS Load Balancer Controller.

File:

```
charts/order-service/templates/ingress.yaml
```

```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: order-service

  namespace: {{ .Release.Namespace }}

  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health/ready

spec:
  ingressClassName: {{ .Values.ingress.className }}

  rules:
    - host: {{ .Values.ingress.host }}

      http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: order-service

                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

This example uses an internal ALB.

Before applying it, confirm:

- AWS Load Balancer Controller is installed.
- Its IAM permissions are configured.
- The cluster networking and subnets are suitable.
- The target health-check endpoint exists.
- The required DNS record is configured.

An Ingress object alone does not guarantee that an ALB will be provisioned.

## 200.16 — Validate Helm before deployment

Before allowing ArgoCD to deploy, validate the chart.

Run:

```
helm lint charts/order-service
```

Expected:

```
1 chart(s) linted, 0 chart(s) failed
```

Render the Kubernetes resources:

```
helm template order-service \
  charts/order-service \
  --namespace roboshop-dev \
  -f environments/dev/order-service/values.yaml
```

Check the rendered image:

```
helm template order-service \
  charts/order-service \
  --namespace roboshop-dev \
  -f environments/dev/order-service/values.yaml |
  grep -n "image:"
```

Expected format:

```
image: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/order-service@sha256:..."
```

Also verify that:

- The digest is not a placeholder.
- The ConfigMap name is correct.
- The Secret already exists.
- The ServiceAccount is correct.
- The container port matches the application.
- The health-check endpoints exist.
- The requested resources fit the DEV quota.

Do not deploy a chart that contains unresolved placeholders.

## 200.17 — Create the ArgoCD Application

Now we configure ArgoCD to manage the DEV deployment.

File:

```
environments/dev/applications/order-service.yaml
```

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-dev

  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_ORG/roboshop-gitops.git

    targetRevision: main

    path: charts/order-service

    helm:
      valueFiles:
        - ../../environments/dev/order-service/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: roboshop-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
```

Replace the repository URL with the actual GitOps repository.

The relative Helm values-file path must be validated against the repository layout and the Helm/ArgoCD version in use. If the repository layout or ArgoCD source configuration does not support the relative reference, use an ArgoCD multi-source Application or place the values file within the chart source.

### What does ArgoCD do?

ArgoCD continuously compares:

```
Desired state:
GitOps repository

Actual state:
Kubernetes cluster
```

If the two differ, ArgoCD identifies the difference and reconciles the cluster according to the configured sync policy.

### What is selfHeal?

```
selfHeal: true
```

If someone manually changes a managed resource in the cluster, ArgoCD can restore the Git-defined configuration.

### What is prune?

```
prune: true
```

Resources removed from the Git-managed application may be deleted during synchronization.

This is why we must review Git changes carefully before merging them.

## 200.18 — Deploy through ArgoCD

First, confirm ArgoCD is running:

```
kubectl get pods -n argocd
```

Apply the Application:

```
kubectl apply -f \
  environments/dev/applications/order-service.yaml
```

Check the Application:

```
kubectl get applications -n argocd
```

Describe:

```
kubectl describe application order-service-dev \
  -n argocd
```

Check ArgoCD synchronization:

```
argocd app get order-service-dev
```

If necessary, initiate a manual sync:

```
argocd app sync order-service-dev
```

Monitor:

```
argocd app wait order-service-dev \
  --sync \
  --health \
  --timeout 300
```

### Expected deployment state

order-service-dev

Expected

Sync status

Synced

Health status

Healthy

Namespace

roboshop-dev

Replicas

2 desired / 2 ready

## 200.19 — Validate Kubernetes resources

After synchronization, we must validate every important resource.

### 200.19.1 — Check Deployment

```
kubectl get deployment order-service \
  -n roboshop-dev
```

Expected:

```
NAME            READY   UP-TO-DATE   AVAILABLE
order-service   2/2     2            2
```

Describe:

```
kubectl describe deployment order-service \
  -n roboshop-dev
```

### 200.19.2 — Check Pods

```
kubectl get pods \
  -n roboshop-dev \
  -o wide
```

Expected:

```
NAME                             READY   STATUS
order-service-xxxxxxxxxx-abc12   1/1     Running
order-service-xxxxxxxxxx-def34   1/1     Running
```

### 200.19.3 — Check ReplicaSets

```
kubectl get replicasets \
  -n roboshop-dev
```

The ReplicaSet maintains the requested number of Pods for the Deployment.

### 200.19.4 — Check Service

```
kubectl get service order-service \
  -n roboshop-dev
```

### 200.19.5 — Check Service endpoints

```
kubectl get endpointslice \
  -n roboshop-dev
```

Or:

```
kubectl get endpoints order-service \
  -n roboshop-dev
```

Depending on the Kubernetes version and resource configuration, EndpointSlices are the preferred API.

We should see the IP addresses and ports of ready application Pods.

If there are no endpoints, investigate the Service selector and Pod readiness.

### 200.19.6 — Check Ingress

```
kubectl get ingress \
  -n roboshop-dev
```

Describe:

```
kubectl describe ingress order-service \
  -n roboshop-dev
```

Check whether the ALB address has been provisioned.

## 200.20 — Validate application health

A Pod can be running while the application is not functioning correctly.

Therefore, we must check application health separately.

### Check application logs

```
kubectl logs deployment/order-service \
  -n roboshop-dev \
  --all-pods=true
```

Follow logs:

```
kubectl logs -f deployment/order-service \
  -n roboshop-dev
```

Check previous container logs if a container restarted:

```
kubectl logs <pod-name> \
  -n roboshop-dev \
  --previous
```

### Test readiness endpoint

First, forward a local port:

```
kubectl port-forward \
  service/order-service \
  8080:8080 \
  -n roboshop-dev
```

In another terminal:

```
curl -i http://localhost:8080/health/ready
```

Test liveness:

```
curl -i http://localhost:8080/health/live
```

The actual endpoints must match the application implementation.

A successful response should indicate that the application is functioning according to its health-check contract.

## 200.21 — Validate database connectivity

Our order-service may need to communicate with a database.

For example:

```
order-service
     |
     v
DEV RDS endpoint
     |
     v
Database
```

Before testing, verify the database configuration.

### Step 1: Check environment variables

Do not print database passwords.

```
kubectl exec deployment/order-service \
  -n roboshop-dev \
  -- printenv DB_USERNAME
```

Check that the required variables exist without displaying secrets.

### Step 2: Verify the database endpoint

Confirm that the application is configured with the correct DEV database endpoint.

### Step 3: Check DNS resolution

From a diagnostic Pod in the namespace:

```
kubectl run network-debug \
  --image=busybox:1.36 \
  --restart=Never \
  -n roboshop-dev \
  --command -- sleep 3600
```

Test DNS:

```
kubectl exec network-debug \
  -n roboshop-dev \
  -- nslookup <DEV_RDS_ENDPOINT>
```

### Step 4: Test the database port

For MySQL, the default port is `3306`.

For PostgreSQL, the default port is `5432`.

Example:

```
kubectl exec network-debug \
  -n roboshop-dev \
  -- nc -zv <DEV_RDS_ENDPOINT> 3306
```

Use the port configured for the actual database.

### Step 5: Check network configuration

If connectivity fails, investigate:

- RDS security group inbound rules.
- EKS worker node or Pod security groups.
- VPC routing.
- Network ACLs.
- DNS resolution.
- Database availability.
- Kubernetes NetworkPolicies, if configured.

Do not open database ports to `0.0.0.0/0` to resolve connectivity problems.

### Step 6: Verify application database connection

Review application logs for:

```
Database connection established
```

Or the actual successful connection message emitted by the application.

A successful TCP connection does not prove that the application can authenticate or execute queries. Verify the actual application-level database operation as well.

Remove the temporary diagnostic Pod when testing is complete:

```
kubectl delete pod network-debug \
  -n roboshop-dev
```

## 200.22 — Validate microservice connectivity

In a microservices architecture, order-service may communicate with:

- User service.
- Product Catalog service.
- Cart service.
- Payment service.
- Inventory service.
- Notification service.

Example request flow:

```
Client
  |
  v
Order Service
  |
  +----> User Service
  |
  +----> Cart Service
  |
  +----> Product Catalog
  |
  +----> Payment Service
  |
  +----> Inventory Service
  |
  +----> Notification Service
```

Check Service discovery:

```
kubectl get services \
  -n roboshop-dev
```

Check DNS for a dependency:

```
kubectl exec network-debug \
  -n roboshop-dev \
  -- nslookup user-service.roboshop-dev.svc.cluster.local
```

Test a dependency endpoint from the diagnostic Pod if the application exposes an appropriate internal health endpoint.

If a dependency is deployed in a different namespace, use its actual namespace in the DNS name.

### What if the dependency is down?

Do not automatically restart the order-service.

First determine:

1. Is the dependency Pod running?
2. Is the dependency Service present?
3. Are there ready endpoints?
4. Does DNS resolve?
5. Is the network path available?
6. Is the dependency application healthy?
7. Does order-service have suitable retry and timeout handling?

This prevents unnecessary restarts and helps identify the actual failing component.

## 200.23 — Integrate Prometheus and Grafana

We already use Prometheus and Grafana in our observability stack.

The DEV deployment must be monitored.

### Step 1: Confirm Prometheus is running

```
kubectl get pods -A | grep prometheus
```

### Step 2: Expose application metrics

If order-service exposes Prometheus metrics, confirm the metrics endpoint exists.

Example:

```
/metrics
```

The application must actually expose this endpoint; Kubernetes cannot generate application metrics automatically.

### Step 3: Create a ServiceMonitor if using Prometheus Operator

Example:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: order-service
  namespace: roboshop-dev

  labels:
    release: prometheus

spec:
  namespaceSelector:
    matchNames:
      - roboshop-dev

  selector:
    matchLabels:
      app: order-service

  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

The `release` label must match the Prometheus Operator installation's ServiceMonitor selector.

Apply:

```
kubectl apply -f order-service-servicemonitor.yaml
```

### Step 4: Verify Prometheus targets

Open the Prometheus UI and check the Targets page.

Confirm:

```
order-service
State: UP
```

### Step 5: Validate Grafana

Create or update a DEV dashboard showing:

| Metric               | Purpose                   |
| -------------------- | ------------------------- |
| Pod CPU              | CPU consumption           |
| Pod memory           | Memory consumption        |
| Pod restarts         | Container instability     |
| HTTP request rate    | Application traffic       |
| HTTP error rate      | Application failures      |
| Request latency      | Application response time |
| Replica availability | Workload health           |

Application-specific metrics must be instrumented and exposed before they can be displayed.

## 200.24 — Integrate ELK logging

Our observability stack also includes ELK.

The application should write structured logs to standard output and standard error.

Example JSON log:

```
{
  "timestamp": "2026-11-01T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "environment": "dev",
  "message": "Order processing started",
  "orderId": "ORD-1001"
}
```

Kubernetes container logs can then be collected by the configured log shipper.

### Validate logs

```
kubectl logs deployment/order-service \
  -n roboshop-dev \
  --tail=100
```

Open Kibana and search:

```
service : "order-service"
```

Filter by:

```
environment : "dev"
```

Check:

- Application startup logs.
- Database connection logs.
- Dependency communication logs.
- HTTP errors.
- Exception traces.
- Request correlation information, if implemented.

Do not log passwords, access tokens, session cookies or other sensitive information.

## 200.25 — Execute smoke tests

Once the deployment and dependencies are healthy, we execute smoke tests.

A smoke test checks whether the most important application functions are working after deployment.

Example test script:

File:

```
scripts/smoke-test.sh
```

```
#!/bin/bash

set -euo pipefail

BASE_URL="${BASE_URL:?BASE_URL is required}"

echo "Starting order-service smoke tests"

echo "Checking readiness endpoint"

HTTP_STATUS=$(curl -sS \
  -o /tmp/order-health.json \
  -w "%{http_code}" \
  "${BASE_URL}/health/ready")

if [ "${HTTP_STATUS}" != "200" ]; then
    echo "Readiness check failed"
    cat /tmp/order-health.json
    exit 1
fi

echo "Readiness check passed"

echo "Checking application response"

HTTP_STATUS=$(curl -sS \
  -o /tmp/order-response.json \
  -w "%{http_code}" \
  "${BASE_URL}/api/orders")

if [ "${HTTP_STATUS}" -ge 500 ]; then
    echo "Order API returned a server error"
    cat /tmp/order-response.json
    exit 1
fi

echo "Order API returned HTTP ${HTTP_STATUS}"

echo "Smoke tests completed"
```

This example assumes the endpoint exists and allows a GET request. Replace it with the actual order-service API contract.

For APIs requiring authentication, use an approved test identity and secure credential handling.

Do not create real orders, initiate payments, or modify business data unless the DEV test procedure explicitly permits it.

### Smoke-test checklist

| Test                  | Expected result        |
| --------------------- | ---------------------- |
| Application readiness | HTTP 200               |
| Application liveness  | HTTP 200               |
| Service DNS           | Resolves               |
| Database connection   | Successful             |
| Dependency connection | Successful             |
| Metrics endpoint      | Available              |
| Logs                  | Visible in ELK         |
| API response          | Matches API contract   |
| Pod replicas          | Desired replicas ready |

## 200.26 — Troubleshooting production-style deployment issues

A DevOps engineer must understand how to troubleshoot common Kubernetes deployment failures.

### Scenario 1 — ImagePullBackOff

Check:

```
kubectl describe pod <pod-name> \
  -n roboshop-dev
```

Possible reasons:

- Incorrect ECR repository.
- Incorrect image digest.
- ECR repository missing.
- Node role lacks ECR permissions.
- Network connectivity issue.
- Image is in a different AWS account without the required access.

Check the image reference:

```
kubectl get deployment order-service \
  -n roboshop-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Verify the digest in ECR.

### Scenario 2 — CrashLoopBackOff

Check:

```
kubectl logs <pod-name> \
  -n roboshop-dev \
  --previous
```

Investigate:

- Application startup failure.
- Missing environment variables.
- Incorrect database credentials.
- Database connection failure.
- Invalid configuration.
- Application process exiting.

Do not increase the restart count or continuously delete Pods without understanding the root cause.

### Scenario 3 — CreateContainerConfigError

Check:

```
kubectl describe pod <pod-name> \
  -n roboshop-dev
```

Common causes:

- ConfigMap missing.
- Secret missing.
- Incorrect environment variable reference.
- Invalid volume reference.

Verify:

```
kubectl get configmaps -n roboshop-dev

kubectl get secrets -n roboshop-dev
```

### Scenario 4 — Readiness probe failure

Check:

```
kubectl describe pod <pod-name> \
  -n roboshop-dev
```

Verify:

- Health endpoint.
- Container port.
- Application startup time.
- Database readiness, if the readiness contract includes it.
- Probe timing.

Do not simply disable readiness probes to make the Deployment appear healthy.

### Scenario 5 — Service returns 503

Investigate:

1. Are Pods ready?
2. Does the Service selector match the Pod labels?
3. Are EndpointSlices populated?
4. Is the application listening on the expected port?
5. Is the Ingress target group healthy?
6. Does the load balancer health check use the correct path?

Useful commands:

```
kubectl get pods -n roboshop-dev

kubectl get svc -n roboshop-dev

kubectl get endpointslice -n roboshop-dev

kubectl describe ingress order-service \
  -n roboshop-dev
```

### Scenario 6 — ArgoCD OutOfSync

Check:

```
argocd app get order-service-dev
```

Possible causes:

- Manual cluster changes.
- Incorrect Helm values.
- Git commit not pushed.
- Wrong branch.
- Incorrect source path.
- ArgoCD rendering error.

Compare the desired state and actual state before synchronizing.

### Scenario 7 — ArgoCD Degraded

Check the application health:

```
argocd app get order-service-dev
```

Then inspect Kubernetes resources:

```
kubectl get all \
  -n roboshop-dev
```

Investigate the resource causing the degraded health status rather than assuming that ArgoCD itself is broken.

### Scenario 8 — Database connection timeout

Check:

- RDS availability.
- DNS resolution.
- Security group rules.
- Route tables.
- Database port.
- Network ACLs.
- Database connection pool configuration.
- Application connection timeout.

Use logs and network tests to distinguish DNS, network, authentication and database errors.

## 200.27 — Rollback procedure

Suppose the newly deployed order-service version causes errors.

We need to restore the previous known-good version.

Because this is a GitOps-managed deployment, the preferred rollback is through Git.

### Step 1 — Identify the previous image digest

Review the previous successful DEV deployment or release record.

Example:

```
Current digest:
sha256:NEW_IMAGE_DIGEST

Previous digest:
sha256:PREVIOUS_IMAGE_DIGEST
```

### Step 2 — Update the GitOps values file

Change:

```
image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/order-service"
  digest: "sha256:PREVIOUS_IMAGE_DIGEST"
```

### Step 3 — Commit the rollback

```
git add environments/dev/order-service/values.yaml

git commit -m "Rollback order-service DEV - ECOM-4821"

git push origin main
```

### Step 4 — Allow ArgoCD to synchronize

```
argocd app get order-service-dev
```

If required:

```
argocd app sync order-service-dev
```

### Step 5 — Verify the restored version

```
kubectl get pods \
  -n roboshop-dev
```

Check the actual deployed image:

```
kubectl get deployment order-service \
  -n roboshop-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check application health and smoke tests again.

Important: Rolling back the container image does not automatically reverse database schema changes. Database migrations need a separate, backward-compatible rollback strategy.

## 200.28 — Deployment evidence collection

For every deployment, we should retain evidence.

Create a directory:

```
mkdir -p evidence/ECOM-4821
```

Capture Deployment status:

```
kubectl get deployment order-service \
  -n roboshop-dev \
  -o yaml \
  > evidence/ECOM-4821/deployment.yaml
```

Capture Pod status:

```
kubectl get pods \
  -n roboshop-dev \
  -o wide \
  > evidence/ECOM-4821/pods.txt
```

Capture Service:

```
kubectl get svc order-service \
  -n roboshop-dev \
  -o yaml \
  > evidence/ECOM-4821/service.yaml
```

Capture Ingress:

```
kubectl get ingress order-service \
  -n roboshop-dev \
  -o yaml \
  > evidence/ECOM-4821/ingress.yaml
```

Capture ArgoCD status:

```
argocd app get order-service-dev \
  > evidence/ECOM-4821/argocd-status.txt
```

Capture application logs:

```
kubectl logs deployment/order-service \
  -n roboshop-dev \
  --tail=200 \
  > evidence/ECOM-4821/application.log
```

Capture the deployed image:

```
kubectl get deployment order-service \
  -n roboshop-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}' \
  > evidence/ECOM-4821/image-reference.txt
```

Do not store credentials or unredacted secrets in deployment evidence.

## 200.29 — QA handover

After deployment validation, we hand over the DEV environment to the QA team.

Example handover information:

| Field                 | Value         |
| --------------------- | ------------- |
| Ticket                | ECOM-4821     |
| Application           | RoboShop      |
| Microservice          | order-service |
| Environment           | DEV           |
| Namespace             | roboshop-dev  |
| Deployment            | Successful    |
| ArgoCD sync           | Synced        |
| ArgoCD health         | Healthy       |
| Image                 | ECR digest    |
| Replicas              | 2/2 ready     |
| Smoke tests           | Passed        |
| Database connectivity | Verified      |
| Logging               | Verified      |
| Metrics               | Verified      |

### QA handover message

Hi QA Team,

The `order-service` deployment for **ECOM-4821** has been completed successfully in the DEV EKS environment.

**Deployment details:**

- Environment: DEV
- Namespace: `roboshop-dev`
- Application: `order-service`
- Deployment status: Successful
- ArgoCD status: Synced and Healthy
- Pod status: 2/2 replicas ready
- Image: Published ECR image digest

Application health checks and initial smoke tests have been completed. Database and dependency connectivity have also been verified.

The application is ready for QA validation.

Please proceed with the planned test cases and share any issues found.

## 200.30 — Final DEV deployment checklist

CHAPTER 200 — Deployment progress

# 0%

0 of 28 tasks completed

Confirm ECOM-4821 release details

Verify the DEV AWS account and EKS cluster

Prepare roboshop-dev namespace

Configure resource quota

Verify ECR node image-pull permissions

Confirm the exact ECR image digest

Prepare the order-service Helm chart

Configure ServiceAccount and workload identity if needed

Configure ConfigMap

Create DEV Secret securely

Configure Deployment and probes

Configure Kubernetes Service

Configure Ingress and ALB if required

Run Helm lint and template validation

Create ArgoCD Application

Synchronize the DEV deployment

Validate Pods and Deployment

Validate Service and EndpointSlices

Validate Ingress and ALB health

Test application readiness and liveness

Verify database connectivity

Verify microservice connectivity

Verify Prometheus metrics

Verify Grafana dashboard

Verify ELK logs

Execute smoke tests

Capture deployment evidence

Complete QA handover

## 200.31 — Expected outcome

After completing this chapter, you should be able to independently deploy an application into a DEV EKS environment using a GitOps workflow.

You should understand:

- How EKS retrieves images from ECR.
- How to configure Kubernetes namespaces and resources.
- How Helm manages reusable deployment configuration.
- How ConfigMaps and Secrets provide application configuration.
- How ServiceAccounts and workload identity work.
- How ArgoCD synchronizes Git configuration with EKS.
- How to troubleshoot common deployment failures.
- How to validate database and microservice connectivity.
- How to verify Prometheus, Grafana and ELK integration.
- How to execute smoke tests, collect evidence and hand over to QA.

Key production principle: The same image digest that passed the CI/CD process must be deployed into DEV. We should never silently rebuild or replace the artifact during deployment.

Chapter sequence note: The next planned chapter is CHAPTER 201. The exact sequence and filename should be preserved from the project plan.