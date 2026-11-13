# PHASE 21 — FINAL PRODUCTION SIMULATION

# Chapter 199 — GitOps Promotion



Production implementation

Chapter 199 of 212



## 1. Objective

In Chapter 198, we published the `order-service` Docker image to Amazon ECR and captured its image digest.

Now we need to deploy that exact image across our environments:

- DEV
- QA
- STAGING
- PROD

We will use GitOps with ArgoCD and Helm.

The central principle is:

Git is the source of truth for Kubernetes desired state. ArgoCD continuously compares the desired state in Git with the actual state in the EKS cluster and reconciles differences.

We will not directly execute `kubectl apply` from GitHub Actions to deploy production workloads.

Instead, the pipeline updates the GitOps repository through a controlled pull request. After the required review and approvals, the change is merged. ArgoCD detects the updated Git revision and synchronizes the cluster.

### What we will implement

1. GitOps architecture.
2. GitOps repository structure.
3. Environment separation.
4. Helm chart configuration.
5. ECR image digest verification.
6. DEV promotion.
7. QA promotion.
8. STAGING promotion.
9. Production promotion controls.
10. GitOps pull request automation.
11. ArgoCD Application configuration.
12. ArgoCD synchronization.
13. Drift detection and reconciliation.
14. Release traceability.
15. Rollback through Git history.
16. Troubleshooting and production operations.

## 2. Current release context

We must continue the same production simulation.

| Item                | Value                             |
| ------------------- | --------------------------------- |
| Jira issue          | `ECOM-4821`                       |
| Microservice        | `order-service`                   |
| Source commit       | `d4e5f6a...` (illustrative)       |
| Image tag           | Full source commit SHA            |
| AWS account         | `123456789012` (placeholder)      |
| AWS region          | `ap-south-1`                      |
| ECR repository      | `company-ecommerce/order-service` |
| Image digest        | Captured after actual ECR push    |
| Kubernetes platform | Amazon EKS                        |
| Deployment tool     | ArgoCD                            |
| Packaging           | Helm                              |
| Environments        | DEV, QA, STAGING, PROD            |

For examples, we will use this placeholder digest:

```
sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

This is not an actual digest from an ECR push. Replace it with the real digest produced in Chapter 198.

The image reference will look like:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service@sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Important: We do not rebuild the image in this chapter. We only promote the existing image digest.

## 3. GitOps architecture

[8 Infrastructure as Code Examples for Modern Platforms | CloudCops Resources](https://images.openai.com/static-rsc-4/ceoq4N87mL-xVWw8X3_dBGa6h_YSRMo6DjauLG8UM2TqVKcG9SFco9eaP2_bqN3OR3yxYPNwUGWYNqv6KPVHQG8p-BOVGpbbqIMrRN0le8lruy_l3iLGOFvmNXwL5ebtQcZqkBKP7CtYXNnmOW4EnOej0N8YP5JbvYzwNSyfl8s?purpose=inline)

[resources.cloudcops.com](https://resources.cloudcops.com/blogs/infrastructure-as-code-examples?utm_source=chatgpt.com)

Our release architecture:

```
Developer
    |
    v
Application Repository
    |
    v
GitHub Actions CI
    |
    v
Security Scanning
    |
    v
Docker Image Build
    |
    v
Amazon ECR
    |
    v
Image Digest
    |
    v
GitOps Promotion Workflow
    |
    v
GitOps Repository
    |
    v
Pull Request
    |
    v
Review and Approval
    |
    v
Merge to Environment Branch
    |
    v
ArgoCD
    |
    v
Amazon EKS
    |
    v
Kubernetes Resources
```

### 3.1 Understand the responsibilities

| Component              | Responsibility                                 |
| ---------------------- | ---------------------------------------------- |
| Application repository | Contains application source code               |
| GitHub Actions         | Builds, tests and publishes images             |
| ECR                    | Stores immutable container images              |
| GitOps repository      | Stores desired Kubernetes configuration        |
| Helm                   | Packages and configures Kubernetes resources   |
| ArgoCD                 | Reconciles Git desired state with EKS          |
| EKS                    | Runs Kubernetes workloads                      |
| Jira                   | Tracks feature and production change           |
| QA                     | Validates application behavior                 |
| SRE/Operations         | Validates production readiness and reliability |

### 3.2 Why separate application and GitOps repositories?

Application source code and deployment configuration have different responsibilities.

Application repository:

```
order-service/
```

Contains:

- Java code.
- Maven configuration.
- Unit tests.
- Dockerfile.
- CI workflows.
- Application Helm chart, if chart ownership is maintained here.

GitOps repository:

```
gitops/
```

Contains:

- Environment-specific Helm values.
- ArgoCD Applications.
- Deployment configuration.
- Environment promotion history.
- Approved image digests.

This separation allows application developers and platform engineers to work independently while maintaining a controlled deployment process.

## 4. GitOps repository structure

We will use the following structure.

```
gitops/
│
├── README.md
│
├── apps/
│   │
│   ├── order-service/
│   │   ├── application.yaml
│   │   ├── project.yaml
│   │   └── README.md
│   │
│   ├── payment-service/
│   │   └── application.yaml
│   │
│   └── inventory-service/
│       └── application.yaml
│
├── charts/
│   │
│   └── order-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values.schema.json
│       │
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── serviceaccount.yaml
│           ├── ingress.yaml
│           ├── hpa.yaml
│           ├── pdb.yaml
│           ├── configmap.yaml
│           └── _helpers.tpl
│
└── environments/
    │
    ├── dev/
    │   └── order-service/
    │       └── values.yaml
    │
    ├── qa/
    │   └── order-service/
    │       └── values.yaml
    │
    ├── staging/
    │   └── order-service/
    │       └── values.yaml
    │
    └── prod/
        └── order-service/
            └── values.yaml
```

### Why separate values files?

Each environment has different deployment requirements.

For example:

- DEV may use two replicas.
- QA may use two replicas and test configuration.
- STAGING may use production-like resource settings.
- PROD may use multiple replicas, strict availability and production autoscaling.

The application image remains the same.

Only the environment configuration changes.

## 5. Helm chart configuration

We need a Helm chart for `order-service`.

The chart contains reusable Kubernetes templates.

### 5.1 Chart.yaml

Path:

```
gitops/charts/order-service/Chart.yaml
```

```
apiVersion: v2

name: order-service

description: Helm chart for the enterprise order service

type: application

version: 1.0.0

appVersion: "1.0.0"
```

Explanation:

| Field         | Meaning                      |
| ------------- | ---------------------------- |
| `apiVersion`  | Helm chart API version       |
| `name`        | Chart name                   |
| `description` | Chart purpose                |
| `type`        | Application chart            |
| `version`     | Helm chart version           |
| `appVersion`  | Application version metadata |

The chart version and application image version are different things.

Changing the image digest does not automatically require a chart version change.

### 5.2 values.yaml

Path:

```
gitops/charts/order-service/values.yaml
```

```
replicaCount: 2

image:
  repository: ""
  digest: ""

imagePullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: false
  minAvailable: 1

probes:
  startup:
    enabled: true
    path: /actuator/health
    initialDelaySeconds: 10
    periodSeconds: 10

  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 10
    periodSeconds: 10

  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 15

env:
  SPRING_PROFILES_ACTIVE: dev

  SERVER_PORT: "8080"
```

We use an image digest rather than a mutable tag.

The actual image will be supplied by the environment-specific values file.

### 5.3 values.schema.json

Path:

```
gitops/charts/order-service/values.schema.json
```

This validates the values passed to the Helm chart.

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": [
    "image",
    "replicaCount"
  ],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "required": [
        "repository",
        "digest"
      ],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        },
        "digest": {
          "type": "string",
          "pattern": "^sha256:[a-f0-9]{64}$"
        }
      }
    }
  }
}
```

This prevents invalid image digest values from being accepted by the chart.

## 6. Create Kubernetes Deployment template

Path:

```
gitops/charts/order-service/templates/deployment.yaml
```

```
apiVersion: apps/v1

kind: Deployment

metadata:
  name: order-service

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

          imagePullPolicy: {{ .Values.imagePullPolicy }}

          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.env.SPRING_PROFILES_ACTIVE | quote }}

            - name: SERVER_PORT
              value: {{ .Values.env.SERVER_PORT | quote }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.startup.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          {{- end }}

          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          {{- end }}

          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          {{- end }}
```

### Important production considerations

This is a teaching template. A production chart should also include:

- Security context.
- Pod-level security context.
- Image pull and registry access validation.
- ConfigMap and Secret references.
- Graceful termination settings.
- Rolling update strategy.
- Topology spread constraints.
- Pod anti-affinity where appropriate.
- NetworkPolicy.
- ServiceAccount configuration.
- Monitoring annotations or ServiceMonitor where applicable.
- Optional autoscaling and disruption budget resources.

We will expand and validate these platform configurations in the relevant chapters of the larger project.

### 6.1 Image digest construction

This line is important:

```
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

Helm renders:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service@sha256:aaaaaaaa...
```

Kubernetes receives the exact image digest.

## 7. Create the Kubernetes Service template

Path:

```
gitops/charts/order-service/templates/service.yaml
```

```
apiVersion: v1

kind: Service

metadata:
  name: order-service

  labels:
    app: order-service

spec:
  type: {{ .Values.service.type }}

  selector:
    app: order-service

  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
```

This Service provides a stable internal endpoint for the order-service pods.

For example:

```
order-service.order-service.svc.cluster.local
```

The namespace in this example is `order-service`.

## 8. Create the ServiceAccount template

Path:

```
gitops/charts/order-service/templates/serviceaccount.yaml
```

```
{{- if .Values.serviceAccount.create }}

apiVersion: v1

kind: ServiceAccount

metadata:
  name: {{ .Values.serviceAccount.name }}

  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}

{{- end }}
```

The application ServiceAccount is used for the workload's Kubernetes identity.

If the application needs AWS permissions, configure EKS Pod Identity or IRSA according to the platform standard.

Do not confuse this application identity with the EKS node role used to pull the container image from ECR.

## 9. Create the HPA template

Path:

```
gitops/charts/order-service/templates/hpa.yaml
```

```
{{- if .Values.autoscaling.enabled }}

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:
  name: order-service

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  minReplicas: {{ .Values.autoscaling.minReplicas }}

  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

  metrics:
    - type: Resource

      resource:
        name: cpu

        target:
          type: Utilization

          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}

{{- end }}
```

The HPA adjusts the number of replicas based on the configured metric.

It requires the Kubernetes metrics pipeline to be available.

## 10. Create the PodDisruptionBudget template

Path:

```
gitops/charts/order-service/templates/pdb.yaml
```

```
{{- if .Values.podDisruptionBudget.enabled }}

apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:
  name: order-service

spec:
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}

  selector:
    matchLabels:
      app: order-service

{{- end }}
```

A PodDisruptionBudget helps protect application availability during supported voluntary disruptions.

It does not prevent every possible outage, and it does not replace proper replica distribution or capacity planning.

## 11. Configure environment-specific values

Now we define the deployment configuration for each environment.

The image digest remains identical across all four environments.

Only environment-specific settings change.

### 11.1 DEV values

Path:

```
gitops/environments/dev/order-service/values.yaml
```

```
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

imagePullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  enabled: false

podDisruptionBudget:
  enabled: false

env:
  SPRING_PROFILES_ACTIVE: dev
  SERVER_PORT: "8080"
```

### 11.2 QA values

Path:

```
gitops/environments/qa/order-service/values.yaml
```

```
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

imagePullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  enabled: false

podDisruptionBudget:
  enabled: false

env:
  SPRING_PROFILES_ACTIVE: qa
  SERVER_PORT: "8080"
```

### 11.3 STAGING values

Path:

```
gitops/environments/staging/order-service/values.yaml
```

```
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

imagePullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: "2"
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

env:
  SPRING_PROFILES_ACTIVE: staging
  SERVER_PORT: "8080"
```

### 11.4 PROD values

Path:

```
gitops/environments/prod/order-service/values.yaml
```

```
replicaCount: 4

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

imagePullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: order-service
  annotations: {}

resources:
  requests:
    cpu: 500m
    memory: 1Gi

  limits:
    cpu: "2"
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 15
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 3

env:
  SPRING_PROFILES_ACTIVE: prod
  SERVER_PORT: "8080"
```

These resource and replica values are illustrative starting points. Real values must be established using load testing, capacity planning and service-level objectives.

### 11.5 Compare the environments

| Setting          | DEV  | QA   | STAGING | PROD |
| ---------------- | ---- | ---- | ------- | ---- |
| Initial replicas | 2    | 2    | 3       | 4    |
| Autoscaling      | No   | No   | Yes     | Yes  |
| Maximum replicas | —    | —    | 8       | 15   |
| PDB              | No   | No   | Yes     | Yes  |
| Spring profile   | dev  | qa   | staging | prod |
| Image digest     | Same | Same | Same    | Same |

The application artifact is identical.

The runtime configuration differs.

## 12. Validate Helm before promotion

Before creating a GitOps pull request, we need to validate the chart and values.

From the GitOps repository:

```
cd gitops
```

Check the Helm version:

```
helm version
```

Lint the chart:

```
helm lint charts/order-service
```

Expected:

```
1 chart(s) linted, 0 chart(s) failed
```

Validate the DEV manifest:

```
helm template order-service \
  charts/order-service \
  --namespace order-service \
  -f environments/dev/order-service/values.yaml
```

Validate QA:

```
helm template order-service \
  charts/order-service \
  --namespace order-service \
  -f environments/qa/order-service/values.yaml
```

Validate STAGING:

```
helm template order-service \
  charts/order-service \
  --namespace order-service \
  -f environments/staging/order-service/values.yaml
```

Validate PROD:

```
helm template order-service \
  charts/order-service \
  --namespace order-service \
  -f environments/prod/order-service/values.yaml
```

Save rendered production manifests:

```
helm template order-service \
  charts/order-service \
  --namespace order-service \
  -f environments/prod/order-service/values.yaml \
  > /tmp/order-service-prod.yaml
```

Inspect the rendered image:

```
grep -n "image:" /tmp/order-service-prod.yaml
```

Expected format:

```
image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service@sha256:aaaa...
```

### 12.1 Why use Helm template?

Helm templates contain variables and conditional logic.

`helm template` renders the actual Kubernetes manifests without deploying them.

This helps us identify:

- Missing values.
- Invalid YAML.
- Incorrect image references.
- Incorrect resource configuration.
- Unexpected rendered resources.

### 12.2 Validate against the Kubernetes API

Where possible, validate the rendered manifests against the target Kubernetes API schema and cluster version.

For example, with an approved validation tool:

```
kubeconform \
  -strict \
  -summary \
  /tmp/order-service-prod.yaml
```

This checks resource schemas, but it does not prove that the application will run successfully or that all referenced cluster dependencies exist.

## 13. Configure ArgoCD Applications

Now we connect the GitOps repository to ArgoCD.

ArgoCD will watch the repository and deploy the desired configuration into EKS.

We will use one ArgoCD Application per service per environment.

Example:

```
order-service-dev
order-service-qa
order-service-staging
order-service-prod
```

### 13.1 DEV Application

Path:

```
gitops/apps/order-service/application-dev.yaml
```

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: order-service-dev

  namespace: argocd

spec:
  project: ecommerce-dev

  source:
    repoURL: https://github.com/YOUR_ORG/gitops.git

    targetRevision: main

    path: charts/order-service

    helm:
      releaseName: order-service

      valueFiles:
        - ../../environments/dev/order-service/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: order-service

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

### 13.2 QA Application

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: order-service-qa

  namespace: argocd

spec:
  project: ecommerce-qa

  source:
    repoURL: https://github.com/YOUR_ORG/gitops.git

    targetRevision: main

    path: charts/order-service

    helm:
      releaseName: order-service

      valueFiles:
        - ../../environments/qa/order-service/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: order-service

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

### 13.3 STAGING Application

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: order-service-staging

  namespace: argocd

spec:
  project: ecommerce-staging

  source:
    repoURL: https://github.com/YOUR_ORG/gitops.git

    targetRevision: main

    path: charts/order-service

    helm:
      releaseName: order-service

      valueFiles:
        - ../../environments/staging/order-service/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: order-service

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

### 13.4 PROD Application

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: order-service-prod

  namespace: argocd

spec:
  project: ecommerce-prod

  source:
    repoURL: https://github.com/YOUR_ORG/gitops.git

    targetRevision: main

    path: charts/order-service

    helm:
      releaseName: order-service

      valueFiles:
        - ../../environments/prod/order-service/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: order-service

  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Notice that the production Application does not have automated synchronization enabled.

This is intentional.

Production synchronization can be initiated under an approved release process.

### 13.5 Important Helm value-file path consideration

ArgoCD resolves Helm value-file paths relative to the chart source path in a repository.

For the example above, confirm that your ArgoCD version and repository layout support the referenced paths.

For a large production GitOps repository, an alternative is to use a multi-source ArgoCD Application or package the chart and values in a structure that avoids ambiguous relative paths.

Do not assume a Helm command that works locally will necessarily resolve identically under ArgoCD without testing the actual Application source configuration.

## 14. Create AppProjects for environment separation

ArgoCD Projects help restrict which Git repositories, clusters and namespaces an Application can use.

We should not allow every Application to deploy anywhere.

Example production project:

Path:

```
gitops/apps/order-service/project-prod.yaml
```

```
apiVersion: argoproj.io/v1alpha1

kind: AppProject

metadata:
  name: ecommerce-prod

  namespace: argocd

spec:
  description: Production applications for ecommerce

  sourceRepos:
    - https://github.com/YOUR_ORG/gitops.git

  destinations:
    - namespace: order-service
      server: https://kubernetes.default.svc

  namespaceResourceWhitelist:
    - group: ""
      kind: ConfigMap

    - group: ""
      kind: Service

    - group: ""
      kind: ServiceAccount

    - group: apps
      kind: Deployment

    - group: autoscaling
      kind: HorizontalPodAutoscaler

    - group: policy
      kind: PodDisruptionBudget

  clusterResourceWhitelist: []
```

The whitelist is illustrative and must include the resources actually required by the application.

For example, if the application also manages NetworkPolicies, the appropriate resource must be allowed.

### Why AppProjects matter

They help enforce:

- Approved source repositories.
- Approved destination clusters.
- Namespace restrictions.
- Resource restrictions.
- Application ownership boundaries.

Production access should be controlled separately from development access.

## 15. Configure ArgoCD repository access

ArgoCD needs access to the GitOps repository.

For a private GitHub repository, configure an approved repository credential mechanism.

Possible approaches include:

- GitHub App authentication.
- Dedicated deploy key.
- Approved repository credential template.

Use the least-privilege approach approved by your organization.

Do not place repository credentials in Git.

Do not use a developer's personal GitHub token as a permanent production integration credential.

For a GitHub App integration, scope access to the GitOps repository and required read operations.

ArgoCD normally needs to read the repository. It does not need write permission merely to deploy a Git revision.

## 16. Understand ArgoCD reconciliation

ArgoCD continuously compares:

```
Desired State in Git
          |
          v
       ArgoCD
          |
          v
 Actual State in EKS
```

If both are equal:

```
Synced
```

If they differ:

```
OutOfSync
```

If an Application is unhealthy:

```
Degraded
```

### 16.1 What is reconciliation?

Suppose Git declares:

```
replicaCount: 4
```

But someone manually changes the Deployment:

```
kubectl scale deployment order-service \
  --replicas=1 \
  -n order-service
```

The actual cluster now differs from Git.

If ArgoCD self-heal is enabled, ArgoCD can reconcile the workload back toward the declared state.

This is why direct manual changes should be controlled.

### 16.2 Important distinction

ArgoCD sync and Kubernetes rollout are different things.

- ArgoCD sync applies desired manifests.
- Kubernetes Deployment controller manages ReplicaSets and Pods.
- Readiness probes determine whether Pods are ready.
- Application monitoring determines whether the release behaves correctly.

A successful ArgoCD sync does not automatically prove business functionality.

## 17. Promote the image to DEV

Now we begin the actual promotion sequence.

The publishing workflow from Chapter 198 produced:

```
IMAGE_URI=<ECR repository>@<actual digest>
```

We need to update the DEV values file.

### 17.1 Create a promotion branch

In the GitOps repository:

```
git checkout main

git pull origin main

git checkout -b \
  promote/order-service-ECOM-4821-dev
```

### 17.2 Update DEV values

Update:

```
environments/dev/order-service/values.yaml
```

Example:

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Only update the image digest and any explicitly approved DEV-specific configuration.

### 17.3 Review the diff

```
git diff
```

Expected change:

```
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
-  digest: sha256:1111111111111111111111111111111111111111111111111111111111111111
+  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

The example old and new digests are illustrative.

### 17.4 Commit the promotion

```
git add environments/dev/order-service/values.yaml
```

```
git commit -m \
  "ECOM-4821 promote order-service image to DEV"
```

Push:

```
git push -u origin \
  promote/order-service-ECOM-4821-dev
```

### 17.5 Create a pull request

Example PR title:

```
ECOM-4821: Promote order-service image to DEV
```

PR description:

```
Jira Issue: ECOM-4821

Service: order-service

Environment: DEV

Source Commit: d4e5f6a...

ECR Repository:
company-ecommerce/order-service

Image Digest:
sha256:aaaaaaaa...

Change:
Promote the newly published order-service image
to the DEV environment.

Validation:
- ECR image exists
- Image digest verified
- Helm validation passed
- Security scan evidence attached

Rollback:
Revert this GitOps change to restore the
previous approved image digest.
```

### 17.6 Merge the PR

After review and required checks:

```
PR approved
     |
     v
PR merged
     |
     v
Git main updated
     |
     v
ArgoCD detects change
```

If DEV has automated sync enabled, ArgoCD should synchronize the Application according to its sync policy.

### 17.7 Check ArgoCD

```
argocd app get order-service-dev
```

Check synchronization:

```
argocd app get order-service-dev \
  -o json
```

Look for:

```
sync.status
health.status
```

Expected healthy state:

```
Synced
Healthy
```

These are expected results, not actual execution results from this conversation.

## 18. Validate the DEV deployment

Now verify the deployment in EKS.

### Check Deployment

```
kubectl get deployment order-service \
  -n order-service
```

### Check Pods

```
kubectl get pods \
  -n order-service \
  -l app=order-service
```

### Check rollout

```
kubectl rollout status deployment/order-service \
  -n order-service
```

### Check the deployed image

```
kubectl get deployment order-service \
  -n order-service \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Compare the result with the digest in Git.

### Check Service

```
kubectl get svc order-service \
  -n order-service
```

### Check Pod events

```
kubectl describe deployment order-service \
  -n order-service
```

### Check logs

```
kubectl logs deployment/order-service \
  -n order-service \
  --tail=100
```

### DEV functional validation

QA or developers should validate:

- Order creation.
- Order status retrieval.
- Order status tracking response.
- Authentication and authorization.
- Database interaction.
- Existing order APIs.
- Error handling.
- Application logs.
- No regression in existing behavior.

Only after successful validation should the image be promoted to QA.

## 19. Promote the same image to QA

The QA environment must receive the exact digest that was tested in DEV.

We do not rebuild the image.

### 19.1 Create QA promotion branch

```
git checkout main

git pull origin main

git checkout -b \
  promote/order-service-ECOM-4821-qa
```

### 19.2 Update QA values

Path:

```
environments/qa/order-service/values.yaml
```

Set the same digest:

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

### 19.3 Create PR

Title:

```
ECOM-4821: Promote order-service image to QA
```

PR must include:

- DEV deployment evidence.
- DEV functional test result.
- Image digest.
- Source commit.
- Security scan result.
- QA test scope.

### 19.4 Merge and synchronize

After approval:

```
argocd app get order-service-qa
```

Verify:

```
kubectl rollout status deployment/order-service \
  -n order-service
```

Confirm the image digest:

```
kubectl get deployment order-service \
  -n order-service \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 19.5 QA test activities

QA should perform:

| Test                  | Purpose                                     |
| --------------------- | ------------------------------------------- |
| Functional testing    | Confirm the new feature works               |
| API testing           | Validate request and response               |
| Integration testing   | Check dependent services                    |
| Database testing      | Validate order data behavior                |
| Regression testing    | Ensure existing functionality still works   |
| Negative testing      | Validate invalid inputs                     |
| Authorization testing | Ensure only permitted users can access data |
| Performance testing   | Check response times and load behavior      |

The test results should be recorded against Jira `ECOM-4821`.

## 20. Promote the same image to STAGING

STAGING should be close to production in configuration and dependencies.

### 20.1 Create branch

```
git checkout main

git pull origin main

git checkout -b \
  promote/order-service-ECOM-4821-staging
```

### 20.2 Update STAGING values

Path:

```
environments/staging/order-service/values.yaml
```

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Use the exact digest that passed DEV and QA.

### 20.3 Create STAGING PR

Title:

```
ECOM-4821: Promote order-service image to STAGING
```

Required evidence:

- DEV validation.
- QA test signoff.
- Security scan result.
- Image digest.
- Deployment configuration review.
- Database migration assessment, if applicable.
- Performance validation plan.

### 20.4 Merge and synchronize

```
argocd app get order-service-staging
```

Verify deployment:

```
kubectl rollout status deployment/order-service \
  -n order-service
```

Check resources:

```
kubectl get pods \
  -n order-service
```

### 20.5 STAGING validation

The team should perform:

- Production-like functional testing.
- End-to-end checkout and order workflows.
- Integration tests.
- Performance tests.
- Capacity checks.
- Dependency health checks.
- Failure scenario testing.
- Monitoring and alert validation.
- Rollback rehearsal where practical.

STAGING is not simply another environment where we check whether a Pod starts.

It is where we validate release readiness.

## 21. Production promotion controls

Production promotion requires stronger governance.

The image has already passed the earlier environment gates.

Now we need to ensure the production change is authorized.

### Required participants

| Team            | Responsibility                                  |
| --------------- | ----------------------------------------------- |
| Developer       | Feature implementation and technical support    |
| DevOps/Platform | Deployment configuration and release automation |
| Security        | Security findings and approved exceptions       |
| QA              | Testing signoff                                 |
| SRE/Operations  | Operational readiness                           |
| DBA/Data owner  | Database change approval if applicable          |
| Change manager  | Production change authorization                 |
| Business owner  | Business approval where required                |

### 21.1 Production release prerequisites

Before production promotion:

- DEV validation completed.
- QA signoff received.
- STAGING validation completed.
- Security requirements satisfied.
- ECR digest verified.
- Monitoring and alerts ready.
- Rollback plan documented.
- Database compatibility checked.
- Production change record created.
- Required approvals recorded.

### 21.2 Production PR

Create branch:

```
git checkout main

git pull origin main

git checkout -b \
  promote/order-service-ECOM-4821-prod
```

Update:

```
environments/prod/order-service/values.yaml
```

Example:

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/company-ecommerce/order-service
  digest: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Create the PR:

```
ECOM-4821: Promote order-service to PROD
```

Include:

```
Jira Issue: ECOM-4821

Service: order-service

Environment: PROD

Source Commit: d4e5f6a...

Image Digest:
sha256:aaaaaaaa...

DEV: PASS

QA: PASS

STAGING: PASS

Security: APPROVED

Monitoring: READY

Rollback: PREPARED

Change Record: ECOM-CHANGE-XXXX
```

The change record ID above is a placeholder for the actual Jira change record created in Chapter 203.

### 21.3 Protect the production branch

Use GitHub branch protection or repository rulesets.

Recommended controls:

- Require pull requests.
- Require code-owner review.
- Require successful validation checks.
- Require appropriate approvals.
- Restrict direct pushes.
- Restrict force pushes.
- Require up-to-date branches where appropriate.
- Use a controlled production release environment.

The PR merge is the authorization point only if your organization has explicitly designed it that way and the required change approvals are integrated into the process.

A Jira approval comment alone is not a technical deployment control.

## 22. Automate GitOps promotion from GitHub Actions

We now connect Chapter 198 ECR publishing with the GitOps repository.

The publishing workflow produces:

```
IMAGE_URI
IMAGE_DIGEST
SOURCE_COMMIT
```

The promotion workflow uses these values to create a GitOps pull request.

### 22.1 Architecture

```
ECR Publishing Job
       |
       v
Image Digest
       |
       v
GitOps Promotion Job
       |
       v
Checkout GitOps Repository
       |
       v
Update Environment Values
       |
       v
Helm Validation
       |
       v
Create Pull Request
       |
       v
Review and Approval
       |
       v
Merge
       |
       v
ArgoCD Reconciliation
```

### 22.2 Authentication

Use a GitHub App for GitOps repository updates.

The GitHub App should have only the permissions required to:

- Read the GitOps repository.
- Create a promotion branch.
- Update the approved values file.
- Create a pull request.

It should not have unrestricted administration privileges.

The app's private key must be stored in an approved secret manager or protected GitHub secret.

### 22.3 Promotion workflow example

This workflow runs after the publishing workflow and assumes that:

- The image has been published.
- The digest has been verified.
- The publishing workflow passed.
- The GitHub App is authorized for the GitOps repository.

Example:

```
name: Promote Order Service to DEV

on:
  workflow_dispatch:
    inputs:
      image_digest:
        description: ECR image digest
        required: true
        type: string

      source_commit:
        description: Source commit SHA
        required: true
        type: string

permissions:
  contents: read

jobs:

  promote:

    runs-on: ubuntu-latest

    steps:

      - name: Validate image digest
        shell: bash
        env:
          IMAGE_DIGEST: ${{ inputs.image_digest }}
          SOURCE_COMMIT: ${{ inputs.source_commit }}
        run: |
          set -euo pipefail

          if ! [[ "$IMAGE_DIGEST" =~ ^sha256:[a-f0-9]{64}$ ]]; then
            echo "Invalid image digest"
            exit 1
          fi

          if ! [[ "$SOURCE_COMMIT" =~ ^[a-f0-9]{40}$ ]]; then
            echo "Invalid source commit"
            exit 1
          fi

      - name: Create GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.GITOPS_APP_ID }}
          private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
          owner: YOUR_ORG
          repositories: gitops

      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: YOUR_ORG/gitops
          token: ${{ steps.app-token.outputs.token }}
          path: gitops

      - name: Update DEV image digest
        working-directory: gitops
        shell: bash
        env:
          IMAGE_DIGEST: ${{ inputs.image_digest }}
        run: |
          set -euo pipefail

          VALUES_FILE="environments/dev/order-service/values.yaml"

          python3 - "$VALUES_FILE" "$IMAGE_DIGEST" <<'PY'
          import sys
          import yaml

          file_path = sys.argv[1]
          digest = sys.argv[2]

          with open(file_path, "r") as f:
              data = yaml.safe_load(f)

          data["image"]["digest"] = digest

          with open(file_path, "w") as f:
              yaml.safe_dump(data, f, sort_keys=False)
          PY

      - name: Validate Helm chart
        working-directory: gitops
        run: |
          set -euo pipefail

          helm lint charts/order-service

          helm template order-service \
            charts/order-service \
            --namespace order-service \
            -f environments/dev/order-service/values.yaml \
            > /tmp/order-service-dev.yaml

      - name: Create promotion pull request
        working-directory: gitops
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          SOURCE_COMMIT: ${{ inputs.source_commit }}
          IMAGE_DIGEST: ${{ inputs.image_digest }}
        run: |
          set -euo pipefail

          BRANCH="promote/order-service-${SOURCE_COMMIT}-dev"

          git config user.name "GitHub Actions"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git checkout -b "$BRANCH"

          git add environments/dev/order-service/values.yaml

          git diff --cached --quiet && {
            echo "No promotion changes"
            exit 0
          }

          git commit -m \
            "ECOM-4821 promote order-service to DEV"

          git push -u origin "$BRANCH"

          gh pr create \
            --repo YOUR_ORG/gitops \
            --base main \
            --head "$BRANCH" \
            --title "ECOM-4821: Promote order-service to DEV" \
            --body "Source commit: ${SOURCE_COMMIT}

          Image digest: ${IMAGE_DIGEST}

          Environment: DEV

          This PR promotes the previously built and published image.
          No image rebuild is performed."
```

### 22.4 Production hardening for this workflow

This is a starting implementation, not a complete enterprise release gate.

Before production use:

1. Pin all actions to verified immutable commit SHAs.
2. Pin the Python YAML dependency and install it from an approved dependency source.
3. Verify the digest against ECR before modifying Git.
4. Verify that the supplied source commit belongs to the expected repository and approved build run.
5. Verify that the image was produced by the trusted build workflow.
6. Use protected branches and required reviews.
7. Use GitHub App permissions limited to the GitOps repository.
8. Prevent arbitrary workflow inputs from promoting unapproved images.
9. Record the actual workflow run and promotion PR.
10. Implement separate controlled workflows for QA, STAGING and PROD.

For production, do not make a freely editable workflow input the only approval mechanism.

The promotion process must be linked to the verified artifact and release evidence.

## 23. Validate the image digest before promotion

We should verify the image exists in ECR before updating GitOps.

Example:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageDigest="$IMAGE_DIGEST" \
  --region ap-south-1
```

If the image does not exist, the promotion must fail.

We can also verify that the digest belongs to the expected commit tag:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageTag="$SOURCE_COMMIT" \
  --region ap-south-1 \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Compare the returned digest with the supplied digest.

If they do not match:

```
Promotion rejected
```

This protects against promoting a digest that does not correspond to the expected source commit.

For a stronger supply-chain control, verify signed image attestations or provenance from the trusted build pipeline before promotion.

## 24. ArgoCD synchronization operations

After a GitOps PR is merged, ArgoCD detects the new Git revision.

### 24.1 List applications

```
argocd app list
```

### 24.2 Get DEV application

```
argocd app get order-service-dev
```

### 24.3 Check application status

```
argocd app get order-service-dev \
  --show-operation
```

### 24.4 Manually sync DEV if required

```
argocd app sync order-service-dev
```

This is useful when automated sync is not enabled or when an approved operator initiates synchronization.

### 24.5 Wait for healthy status

```
argocd app wait order-service-dev \
  --health \
  --sync \
  --timeout 300
```

### 24.6 Check history

```
argocd app history order-service-dev
```

This provides deployment history and revisions associated with the Application.

### 24.7 Production synchronization

For PROD, synchronization should follow the approved production process.

For example:

```
argocd app sync order-service-prod
```

The command must be run by an authorized operator or through a controlled automation path after the appropriate approvals.

Do not give the general CI publishing role permission to sync production applications.

## 25. GitOps drift detection

Drift occurs when the actual cluster configuration differs from the desired configuration in Git.

Example:

Git declares:

```
replicaCount: 4
```

Someone changes the live Deployment:

```
kubectl scale deployment order-service \
  --replicas=1 \
  -n order-service
```

Now:

```
Git desired state: 4 replicas

Kubernetes actual state: 1 replica
```

ArgoCD identifies the difference.

If self-heal is enabled, it can restore the desired state.

### 25.1 Inspect drift

```
argocd app diff order-service-dev
```

### 25.2 Check status

```
argocd app get order-service-dev
```

### 25.3 Why is drift important?

Without drift detection:

- Manual changes can go unnoticed.
- Environments can become inconsistent.
- Incident recovery becomes harder.
- Audit records may not match cluster state.

With GitOps:

- Desired state is version controlled.
- Changes have a review history.
- Drift can be detected.
- Reconciliation can restore approved configuration.

Not every field should necessarily be reconciled in exactly the same way. Some Kubernetes controllers manage fields dynamically, so configure ArgoCD diff and ignore rules carefully. Do not hide meaningful drift through broad ignore rules.

## 26. Rollback using GitOps

Suppose the new order-service release is deployed in DEV and QA.

A problem is discovered during STAGING.

We want to return to the previous known-good image.

The preferred GitOps rollback is to revert the Git change that introduced the new digest.

### 26.1 Find the promotion commit

```
git log --oneline \
  -- environments/staging/order-service/values.yaml
```

Example:

```
abc1234 ECOM-4821 promote order-service to STAGING
def5678 Previous approved order-service release
```

### 26.2 Revert the promotion

```
git revert abc1234
```

Push the revert branch:

```
git push origin HEAD
```

Create a pull request and follow the required approval process.

### 26.3 What happens next?

```
Git Revert PR
     |
     v
PR Approved
     |
     v
PR Merged
     |
     v
GitOps Desired State Updated
     |
     v
ArgoCD Detects Change
     |
     v
ArgoCD Sync
     |
     v
Previous Image Digest Restored
```

The old image must still exist in ECR.

This is why lifecycle policies must preserve rollback-required releases.

### 26.4 Important rollback distinction

GitOps rollback restores the desired Kubernetes configuration.

It does not automatically reverse database schema changes or external business side effects.

Database compatibility and migration rollback must be handled separately.

## 27. Production failure scenarios

### Scenario 1 — ArgoCD reports OutOfSync

Possible causes:

- Git changed.
- Cluster changed manually.
- Helm rendering changed.
- A controller modified managed fields.
- Resource configuration differs.

Commands:

```
argocd app get order-service-dev
```

```
argocd app diff order-service-dev
```

Fix:

- Identify the exact drift.
- Determine whether Git or cluster state is correct.
- Correct the GitOps configuration.
- Reconcile through ArgoCD.

Do not blindly force-sync production without understanding the difference.

### Scenario 2 — ArgoCD reports Degraded

Check:

```
argocd app get order-service-dev
```

Check Pods:

```
kubectl get pods -n order-service
```

Check deployment:

```
kubectl describe deployment order-service \
  -n order-service
```

Possible causes:

- Pods are not ready.
- Image pull failure.
- Failed probes.
- Application crash.
- Insufficient resources.
- Missing configuration.

Fix the underlying application or infrastructure problem.

### Scenario 3 — ImagePullBackOff

Check:

```
kubectl describe pod <pod-name> \
  -n order-service
```

Common causes:

- Incorrect ECR repository.
- Incorrect digest.
- ECR image removed.
- Node role lacks ECR pull access.
- Cross-account repository policy missing.
- Network connectivity problem.

Verify the image exists:

```
aws ecr describe-images \
  --repository-name company-ecommerce/order-service \
  --image-ids imageDigest="$IMAGE_DIGEST" \
  --region ap-south-1
```

### Scenario 4 — Helm rendering failure

Run:

```
helm lint charts/order-service
```

Then:

```
helm template order-service \
  charts/order-service \
  -f environments/dev/order-service/values.yaml
```

Possible causes:

- Incorrect indentation.
- Missing values.
- Invalid template syntax.
- Incorrect values file path.
- Invalid schema.

Fix the chart or values and validate again before merging.

### Scenario 5 — GitOps PR fails

Check:

- GitHub App permissions.
- Branch protection rules.
- Repository access.
- YAML syntax.
- Required CI checks.
- Merge conflicts.

Do not bypass mandatory review checks simply to complete a deployment.

### Scenario 6 — ArgoCD cannot access GitHub

Check:

- Repository URL.
- Credential configuration.
- GitHub App installation.
- Deploy key.
- Token expiration.
- Repository permission.

Check ArgoCD repository status through the ArgoCD interface or CLI.

### Scenario 7 — Wrong digest promoted

Suppose DEV values point to:

```
sha256:aaaaaaaa...
```

But the approved image is:

```
sha256:bbbbbbbb...
```

The promotion workflow should compare the digest with the trusted ECR publication metadata and reject the mismatch.

If already merged, revert the incorrect GitOps change through the controlled process.

### Scenario 8 — ArgoCD sync succeeds but users receive 503

This is a critical interview and production scenario.

A successful sync only means that ArgoCD applied the desired resources.

It does not mean the application is healthy from the user's perspective.

Investigate:

```
kubectl get pods -n order-service
```

```
kubectl get endpoints order-service \
  -n order-service
```

```
kubectl describe svc order-service \
  -n order-service
```

```
kubectl logs deployment/order-service \
  -n order-service \
  --tail=200
```

Check:

- Readiness probes.
- Service selectors.
- Endpoint availability.
- Ingress and load balancer.
- Application startup.
- Database connections.
- Downstream dependencies.
- Prometheus metrics and Grafana dashboards.

The response must be based on evidence, not merely the ArgoCD status.

## 28. Release traceability

We must maintain a complete chain from Jira to the deployed Kubernetes workload.

Example:

```
Jira Issue
ECOM-4821
    |
    v
Git Feature Branch
feature/ECOM-4821-order-tracking
    |
    v
Pull Request
#4821
    |
    v
Source Commit
d4e5f6a...
    |
    v
CI Workflow
Run ID: Actual run ID
    |
    v
Docker Image
order-service:<FULL_SHA>
    |
    v
ECR Digest
sha256:<ACTUAL_DIGEST>
    |
    v
GitOps Promotion PR
Actual PR number
    |
    v
GitOps Commit
Actual commit SHA
    |
    v
ArgoCD Application
order-service-dev
    |
    v
EKS Deployment
order-service
```

Repeat the environment promotion evidence for QA, STAGING and PROD.

### 28.1 Why is this important?

Imagine a production incident.

The team asks:

- Which code version is deployed?
- Which image digest is running?
- Which pull request introduced it?
- Who approved the release?
- Which test results are associated with it?
- What was the previous known-good release?

This chain allows us to answer those questions.

### 28.2 Record deployment metadata

For each environment, capture:

```
Jira ID
Service name
Source repository
Source commit SHA
CI run ID
Image tag
ECR image digest
GitOps repository
Promotion PR number
GitOps merge commit
ArgoCD Application
ArgoCD deployed revision
Kubernetes Deployment revision
Deployment timestamp
Approver
Test evidence
```

Do not invent deployment IDs or timestamps. Capture them from the actual workflow, GitHub, ArgoCD and Kubernetes.

## 29. GitOps automation for 21 microservices

We have 21 microservices.

We should use reusable Helm charts and standardized ArgoCD configuration rather than creating a completely independent deployment design for each service.

Example:

```
gitops/
│
├── apps/
│   ├── user-service/
│   ├── product-service/
│   ├── cart-service/
│   ├── order-service/
│   ├── payment-service/
│   ├── inventory-service/
│   └── ...
│
├── charts/
│   ├── user-service/
│   ├── product-service/
│   ├── cart-service/
│   ├── order-service/
│   ├── payment-service/
│   └── ...
│
└── environments/
    ├── dev/
    ├── qa/
    ├── staging/
    └── prod/
```

There are two common approaches.

### Approach A — One Helm chart per service

Useful when services have substantially different deployment requirements.

Advantages:

- Independent templates.
- Easier service-specific customization.
- Clear ownership.

Disadvantages:

- More charts to maintain.
- Potential configuration duplication.

### Approach B — Shared golden-path Helm chart

Useful when services follow a common deployment pattern.

Advantages:

- Centralized platform standards.
- Less duplication.
- Consistent security configuration.
- Easier maintenance.

Disadvantages:

- Service-specific customization requires carefully designed chart options.
- Changes to the shared chart can affect many services.

For our project, use the Phase 20 Platform Engineering golden-path work where applicable, and keep service-specific differences explicit.

### 29.1 ArgoCD ApplicationSet

An ApplicationSet can generate multiple Applications from a standardized directory structure or another supported generator.

Example directory generator:

```
apiVersion: argoproj.io/v1alpha1

kind: ApplicationSet

metadata:
  name: ecommerce-dev-services

  namespace: argocd

spec:
  generators:
    - git:
        repoURL: https://github.com/YOUR_ORG/gitops.git

        revision: main

        directories:
          - path: environments/dev/*

  template:
    metadata:
      name: '{{path.basename}}-dev'

    spec:
      project: ecommerce-dev

      source:
        repoURL: https://github.com/YOUR_ORG/gitops.git

        targetRevision: main

        path: charts/{{path.basename}}

        helm:
          valueFiles:
            - ../../environments/dev/{{path.basename}}/values.yaml

      destination:
        server: https://kubernetes.default.svc

        namespace: '{{path.basename}}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true

        syncOptions:
          - CreateNamespace=true
```

This is a pattern, not a complete drop-in configuration. The directory generator, chart locations and value-file paths must match the actual repository layout. If all services use a shared chart, change the source path accordingly.

ApplicationSet generation must also be governed so that unreviewed or unexpected directories cannot create applications with excessive privileges.

## 30. Production operating procedures

### Daily operations

The platform team should review:

- ArgoCD Application health.
- OutOfSync Applications.
- Degraded Applications.
- Failed sync operations.
- Unexpected Git changes.
- ECR image availability.
- Failed promotion workflows.

### Before a release

- Confirm the correct image digest.
- Confirm security scan evidence.
- Confirm QA signoff.
- Confirm STAGING validation.
- Confirm production change record.
- Confirm rollback image availability.
- Confirm monitoring and alert readiness.

### After a release

- Verify ArgoCD sync and health.
- Verify Kubernetes rollout.
- Confirm running image digest.
- Check application logs.
- Review Prometheus metrics.
- Review Grafana dashboards.
- Validate critical business transactions.
- Record deployment evidence.

### During an incident

- Identify the deployed digest.
- Compare it with the previous known-good digest.
- Review recent GitOps changes.
- Check application and infrastructure health.
- Follow the approved rollback procedure.
- Preserve logs, events and deployment evidence.

## 31. Complete promotion simulation

Here is the complete promotion journey for `ECOM-4821`.

## Order-service release journey

1. ECR publication

   Chapter 198

   The exact image is published to ECR and its digest is recorded.
2. DEV promotion

   First environment

   GitOps PR updates DEV values. ArgoCD deploys the approved revision. Developers validate the feature.
3. QA promotion

   Testing gate

   The same digest is deployed. QA performs functional, integration and regression tests.
4. STAGING promotion

   Release readiness

   The same digest is validated under production-like conditions.
5. PROD promotion

   Approval required

   The production GitOps change requires the organization's change and release approvals.
6. ArgoCD synchronization

   ArgoCD reconciles the approved production Git revision with EKS.

### Artifact identity across environments

| Environment | Image digest     |
| ----------- | ---------------- |
| DEV         | `sha256:aaaa...` |
| QA          | `sha256:aaaa...` |
| STAGING     | `sha256:aaaa...` |
| PROD        | `sha256:aaaa...` |

All four entries must resolve to the same actual digest for this release.

## 32. Interview questions and practical answers

Q1. What is GitOps?

GitOps is a deployment approach where the desired infrastructure and application configuration are stored in Git. A reconciliation tool such as ArgoCD compares the desired state with the actual cluster state and applies the required changes.

Q2. Why do you use ArgoCD?

We use ArgoCD to manage Kubernetes deployments through Git. It provides synchronization status, health monitoring, drift detection, revision history and controlled reconciliation.

Q3. What is the difference between CI and GitOps CD?

CI validates and builds the application artifact. GitOps CD promotes the approved artifact through environment-specific configuration in Git, and ArgoCD reconciles that configuration with Kubernetes.

Q4. Why separate application and GitOps repositories?

It separates application development from deployment configuration and provides independent review, access control and release governance.

Q5. How do you promote the same image through DEV, QA, STAGING and PROD?

We publish the image once to ECR, capture its digest and update the image digest in each environment's Helm values file. We do not rebuild the image.

Q6. What is the difference between ArgoCD Synced and Healthy?

Synced means the application's managed resources match the desired configuration in Git. Healthy indicates that the resources satisfy ArgoCD's health assessment. Both are important, but neither alone proves business functionality.

Q7. What is self-healing in ArgoCD?

When enabled, ArgoCD can automatically reconcile certain live changes back to the desired state stored in Git.

Q8. What is drift detection?

Drift detection identifies differences between the desired configuration in Git and the actual state in Kubernetes.

Q9. How do you perform a rollback in GitOps?

We revert the Git commit or submit a controlled PR restoring the previous known-good image digest and configuration. ArgoCD then reconciles the previous desired state.

Q10. Why do you use image digests instead of `latest`?

A digest identifies the exact image content. It makes releases reproducible and prevents deployments from silently changing when a tag is moved.

Q11. Can a deployment be successful if ArgoCD shows Synced?

Not necessarily. The resources may be synchronized while the application has failed readiness checks, database connectivity, downstream integration or business functionality.

Q12. How do you manage 21 microservices?

We use reusable Helm templates, standardized GitOps configuration, ArgoCD Applications or ApplicationSets, service-specific values and controlled promotion workflows.

Q13. What is an ArgoCD AppProject?

An AppProject defines boundaries for applications, such as allowed source repositories, destination clusters, namespaces and resource types.

Q14. Does ArgoCD approve Jira production changes automatically?

No. Jira approval and ArgoCD synchronization are separate concerns unless an explicit integration enforces the required approval. We integrate the change process with protected GitOps PRs and production synchronization controls.

Q15. What happens if a new image deploys successfully but order APIs start returning 503?

I check Pod readiness, Deployment rollout, Service endpoints, ingress routing, application logs, database connectivity and downstream dependencies. If the release is responsible, I follow the approved rollback procedure to restore the previous digest.

## 33. Chapter 199 completion checklist

Chapter progress

# 0/29

GitOps architecture understood

Application and GitOps repository separation established

GitOps folder structure created

Helm chart created

Deployment template created

Service template created

ServiceAccount template created

HPA and PDB templates created

Helm values schema configured

DEV values configured

QA values configured

STAGING values configured

PROD values configured

Same image digest used across environments

Helm lint and template validation defined

ArgoCD Applications configured

ArgoCD AppProjects configured

ArgoCD repository access configured

DEV promotion workflow understood

QA promotion workflow understood

STAGING promotion workflow understood

Production approval controls defined

GitHub App promotion automation designed

Image digest verification included

ArgoCD sync and reconciliation understood

Drift detection understood

GitOps rollback procedure documented

Release traceability defined

Production operating procedures documented

Reset checklist

## 34. Final production state

| Item                   | Status                            |
| ---------------------- | --------------------------------- |
| Jira issue `ECOM-4821` | Existing                          |
| Source commit          | `d4e5f6a...` illustrative         |
| CI execution           | Chapter 195                       |
| Security scanning      | Chapter 196                       |
| Docker image build     | Chapter 197                       |
| ECR publishing         | Chapter 198                       |
| Image digest           | Placeholder until actual push     |
| GitOps structure       | Defined                           |
| Helm chart             | Defined                           |
| Environment values     | Defined                           |
| DEV promotion          | Implementation defined            |
| QA promotion           | Implementation defined            |
| STAGING promotion      | Implementation defined            |
| PROD promotion         | Approval process defined          |
| ArgoCD Applications    | Defined                           |
| Actual EKS deployment  | Not executed in this conversation |

Important: The YAML and workflow examples are implementation templates. The actual deployment status must be established from the real GitHub Actions runs, merged GitOps PRs, ArgoCD Applications and EKS cluster.

# Next chapter

Chapter 200

200 — DEV Deployment

We will now focus specifically on deploying `order-service` into the DEV EKS environment.

The next chapter will cover:

1. DEV EKS cluster architecture.
2. Namespace preparation.
3. ECR image pull permissions.
4. ArgoCD DEV Application setup.
5. Helm deployment execution.
6. ConfigMaps and Secrets.
7. ServiceAccount and workload identity.
8. Deployment, Service and Ingress validation.
9. Pod readiness and startup troubleshooting.
10. Application connectivity.
11. Database and dependent microservice connectivity.
12. Prometheus and Grafana validation.
13. ELK log verification.
14. Functional smoke tests.
15. DEV deployment evidence and handoff to QA.

We will continue with the same `ECOM-4821` release and the same ECR image digest.