# PHASE 20 — PLATFORM ENGINEERING

# Chapter 182 — Reusable Helm Templates

Production implementation guide · AWS EKS · Helm · GitOps · Argo CD · 21 microservices

This chapter builds a reusable Helm chart for the enterprise e-commerce platform. The goal is to avoid maintaining 21 separate copies of Deployment, Service, HPA, ServiceAccount, and other Kubernetes manifests.

The implementation is structured as a production engineering chapter, not just a collection of YAML files. It covers repository design, chart architecture, template implementation, configuration contracts, security, validation, release management, GitOps integration, troubleshooting, upgrades, and operational readiness.

Delivery plan: This chapter is split into four substantial parts to preserve detail and keep each part usable. This is Part 1 of 4.

## 1. Chapter objectives

By the end of this chapter, you should be able to:

- Explain why an organization needs reusable Helm charts.
- Design a shared chart for 21 microservices without hiding service-specific requirements.
- Separate reusable platform defaults from application-specific configuration.
- Create a versioned Helm chart with templates and a validated values schema.
- Deploy applications using immutable container image references.
- Configure health probes, resources, security contexts, ServiceAccounts, Services, HPA, and PDB.
- Support Spring Boot, Node.js, Python, and Go applications.
- Validate rendered Kubernetes manifests before deployment.
- Publish versioned charts to Amazon ECR.
- Integrate Helm with Argo CD and a GitOps promotion workflow.
- Diagnose template rendering, scheduling, probe, rollout, and autoscaling failures.
- Manage chart upgrades without unexpectedly breaking existing services.

## 2. Project architecture and boundaries

### 2.1 Platform context

Our e-commerce platform contains 21 microservices:

| Chapter | Microservice           |
| ------- | ---------------------- |
| 161     | Auth Service           |
| 162     | User Service           |
| 163     | Customer Service       |
| 164     | Catalog Service        |
| 165     | Product Service        |
| 166     | Inventory Service      |
| 167     | Pricing Service        |
| 168     | Promotion Service      |
| 169     | Cart Service           |
| 170     | Order Service          |
| 171     | Payment Service        |
| 172     | Fraud Service          |
| 173     | Shipping Service       |
| 174     | Delivery Service       |
| 175     | Notification Service   |
| 176     | Search Service         |
| 177     | Recommendation Service |
| 178     | Review Service         |
| 179     | Tax Service            |
| 180     | Invoice Service        |
| 181     | Reporting Service      |

Each application has different business logic, runtime requirements, resource needs, and scaling behavior. However, most services share common deployment concerns:

- Run a container image.
- Set a namespace and application labels.
- Configure replicas and rollout strategy.
- Define container ports.
- Set CPU and memory requests and limits.
- Configure liveness, readiness, and optionally startup probes.
- Apply security contexts.
- Use a ServiceAccount.
- Expose an internal Kubernetes Service.
- Configure autoscaling when appropriate.
- Support environment-specific values.
- Produce consistent labels and metadata for monitoring and operations.

The reusable chart standardizes these common concerns without forcing every microservice to use identical settings.

### 2.2 Target deployment flow

Application repository

Java / Node.js / Python / Go source code

CI pipeline

Tests · SAST · dependency scan · image scan · build

Amazon ECR

Immutable image digest and verified release artifact

GitOps repository

Chart version + environment-specific values + image digest

Argo CD → Amazon EKS

Reconciliation · rollout · health status

The chart is a reusable deployment package. It is not responsible for compiling Java, building a Docker image, creating the EKS cluster, or deciding whether a business release is approved.

### 2.3 Responsibilities and ownership

| Responsibility                        | Primary owner                            | Reason                                                    |
| ------------------------------------- | ---------------------------------------- | --------------------------------------------------------- |
| Business API implementation           | Application team                         | Owns application behavior                                 |
| Application port and health endpoints | Application team                         | Must match the actual application                         |
| Reusable Helm chart                   | Platform engineering                     | Standardizes workload deployment                          |
| CPU and memory defaults               | Platform + application team              | Platform provides guardrails; teams validate actual needs |
| EKS cluster and node infrastructure   | Cloud/platform engineering               | Owns cluster and infrastructure lifecycle                 |
| Production image promotion            | Release process and authorized approvers | Controls what version reaches production                  |
| Secrets and IAM access                | Security/platform/application owners     | Requires least privilege and explicit access boundaries   |
| GitOps reconciliation                 | Platform engineering                     | Maintains declared versus actual state                    |
| Business-level smoke tests            | Application/QA teams                     | Verifies application behavior                             |
| Cluster-level rollout health          | Platform/SRE teams                       | Verifies deployment and platform behavior                 |

These are default ownership boundaries. Individual organizations may assign them differently.

### 2.4 What belongs in the shared chart?

Good candidates for the shared chart:

- Deployment.
- ClusterIP Service.
- ServiceAccount.
- HPA.
- PDB.
- Common labels and selectors.
- Security-context defaults.
- Standard probes.
- Resource requests and limits.
- Optional non-secret ConfigMap.
- Optional lifecycle hooks.
- Optional scheduling and topology-spread settings.

Keep separate or optional when appropriate:

- Ingress and external exposure.
- NetworkPolicy.
- ServiceMonitor or PodMonitor.
- Database infrastructure.
- Kafka and RabbitMQ infrastructure.
- AWS IAM role creation.
- External Secrets integration.
- CronJobs and one-off migration jobs.
- Stateful workloads that need different lifecycle semantics.

Those capabilities may be provided by separate charts or platform templates in later chapters. Do not turn one chart into an uncontrolled collection of unrelated features.

## 3. Repository architecture

### 3.1 Recommended GitHub organization

Example organization:

```
company-ecommerce/
├── auth-service/
├── user-service/
├── customer-service/
├── catalog-service/
├── product-service/
├── inventory-service/
├── pricing-service/
├── promotion-service/
├── cart-service/
├── order-service/
├── payment-service/
├── fraud-service/
├── shipping-service/
├── delivery-service/
├── notification-service/
├── search-service/
├── recommendation-service/
├── review-service/
├── tax-service/
├── invoice-service/
├── reporting-service/
├── platform-helm-charts/
├── platform-github-actions/
├── platform-terraform/
└── ecommerce-gitops/
```

The exact repository count can differ. The important boundary is that shared platform templates have a controlled release process, while application source code and environment promotion remain independently manageable.

### 3.2 Shared Helm repository

Use this repository for the chart implementation:

```
platform-helm-charts/
├── charts/
│   └── microservice/
│       ├── Chart.yaml
│       ├── Chart.lock
│       ├── values.yaml
│       ├── values.schema.json
│       ├── .helmignore
│       ├── README.md
│       ├── CHANGELOG.md
│       ├── templates/
│       │   ├── _helpers.tpl
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── serviceaccount.yaml
│       │   ├── hpa.yaml
│       │   ├── pdb.yaml
│       │   ├── configmap.yaml
│       │   └── NOTES.txt
│       └── tests/
│           ├── values-minimal.yaml
│           ├── values-production.yaml
│           └── expected-resources.md
├── examples/
│   ├── order-service/
│   │   ├── values-dev.yaml
│   │   ├── values-qa.yaml
│   │   └── values-prod.yaml
│   └── inventory-service/
│       ├── values-dev.yaml
│       └── values-prod.yaml
├── scripts/
│   ├── validate-chart.sh
│   ├── validate-release-values.sh
│   └── render-all-examples.sh
├── .github/
│   └── workflows/
│       └── validate-chart.yaml
└── docs/
    ├── chart-contract.md
    ├── upgrade-policy.md
    └── troubleshooting.md
```

### 3.3 What belongs in each repository?

| Repository                | Contents                                                |
| ------------------------- | ------------------------------------------------------- |
| `order-service`           | Source code, tests, Dockerfile, application-specific CI |
| `platform-helm-charts`    | Reusable Helm chart source and chart release workflow   |
| `ecommerce-gitops`        | Environment declarations, chart versions, image digests |
| `platform-terraform`      | EKS, VPC, IAM, ECR, and other infrastructure modules    |
| `platform-github-actions` | Reusable CI workflows and approved pipeline components  |

Do not copy the entire shared chart into each application repository. That creates 21 versions of the same deployment logic, making fixes difficult to roll out consistently.

### 3.4 GitOps repository layout

A simple, explicit layout for environment promotion:

```
ecommerce-gitops/
├── apps/
│   ├── order-service/
│   │   ├── base/
│   │   │   ├── application.yaml
│   │   │   └── values-common.yaml
│   │   └── environments/
│   │       ├── dev/
│   │       │   └── values.yaml
│   │       ├── qa/
│   │       │   └── values.yaml
│   │       ├── staging/
│   │       │   └── values.yaml
│   │       └── prod/
│   │           └── values.yaml
│   └── inventory-service/
│       ├── base/
│       └── environments/
├── clusters/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
├── policies/
└── README.md
```

Environment values should contain deployment configuration, not secret credentials.

For example, the GitOps repository may specify the external secret name and a non-secret database hostname, while the password itself is retrieved through an approved secrets-management mechanism.

### 3.5 Source of truth

Use this ownership model:

- Application repository: source code and application tests.
- CI pipeline: validates source, builds the image, scans it, and publishes the approved artifact.
- ECR: stores the container image.
- Helm chart repository or OCI registry: stores the versioned deployment chart.
- GitOps repository: declares which chart and image digest belong in each environment.
- Argo CD: reconciles the declared state into the cluster.
- EKS: runs the resulting workloads.

A CI job should not silently modify production by running an unreviewed `kubectl apply`. In this design, CI publishes artifacts and opens or updates a GitOps change; Argo CD performs reconciliation.

## 4. Helm fundamentals for this project

### 4.1 What is a Helm chart?

A Helm chart is a versioned package of Kubernetes resource templates and default configuration.

A chart contains:

- Metadata describing the chart.
- Default values.
- Templates that generate Kubernetes manifests.
- A schema that validates values.
- Optional documentation and tests.

Helm renders templates using supplied values. The resulting YAML is then processed by Kubernetes tooling or the deployment controller.

### 4.2 Template versus values

A template describes the reusable structure.

Example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservice.fullname" . }}
```

The values file supplies the configuration:

```
nameOverride: order-service

replicaCount: 2
```

The chart combines the template and values to produce the final manifest.

### 4.3 Why not use one Deployment YAML per service?

Without a shared chart, teams may independently implement:

- Labels.
- Probe paths.
- Security settings.
- Resource requests.
- Rolling-update settings.
- Service selectors.
- HPA behavior.

That leads to inconsistent defaults and repeated fixes.

A shared chart makes the common contract explicit. It does not mean all 21 services should have the same replica count, CPU limit, probe path, or scaling target.

### 4.4 Helm values precedence

Helm generally resolves values from lower to higher precedence:

1. Chart defaults in `values.yaml`.
2. Parent-chart values, if used as a dependency.
3. User-supplied values files, in the order specified.
4. Command-line `--set` or equivalent overrides.

For example:

```
helm template order-service ./charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml \
  -f examples/order-service/values-prod.yaml
```

The later values file overrides earlier values for keys it defines. This command is shown only to explain precedence; do not combine development and production values like this in a real deployment.

For a development render, use the development file. For production, use the production file.

### 4.5 Values are an API contract

Treat the chart's values as a supported interface between platform engineering and application teams.

For example:

```
replicaCount: 3

image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service"
  digest: "sha256:REPLACE_WITH_REAL_64_CHARACTER_DIGEST"

service:
  port: 8080
```

The chart author owns the meaning of these keys. Renaming `image.digest` to `image.sha` can break every service consuming the chart.

Therefore:

- Document every supported value.
- Validate values with a schema.
- Keep defaults safe.
- Version breaking changes.
- Test upgrades from existing releases.
- Provide migration notes when changing the chart contract.

## 5. Chart versioning strategy

### 5.1 `Chart.yaml`

Create `charts/microservice/Chart.yaml`:

```
apiVersion: v2
name: microservice
description: Reusable deployment chart for stateless e-commerce microservices
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - kubernetes
  - microservices
  - platform-engineering
maintainers:
  - name: Platform Engineering
```

`version` is the Helm chart version. It changes when the chart package changes.

`appVersion` is informational application-version metadata. It is not a substitute for the actual container image digest.

### 5.2 Semantic versioning

Use semantic versioning for the chart:

| Change                                   | Example                                                 | Version guidance |
| ---------------------------------------- | ------------------------------------------------------- | ---------------- |
| Backward-compatible template fix         | Correct an optional annotation                          | Patch: `1.0.1`   |
| Backward-compatible new optional setting | Add an optional ConfigMap                               | Minor: `1.1.0`   |
| Breaking values contract                 | Rename a required value or change behavior incompatibly | Major: `2.0.0`   |

A chart patch version can change without rebuilding application images.

An application image can change without changing the chart version, provided the new image remains compatible with the chart's contract.

### 5.3 Release compatibility rules

Before releasing a chart version:

- Render every maintained example.
- Validate against the Kubernetes API version used by the target EKS clusters.
- Check that generated selectors remain stable.
- Confirm probes match the application's real endpoints.
- Verify that security defaults are compatible with supported images.
- Run upgrade tests against representative releases.
- Review changes to default values.
- Publish a changelog.
- Obtain the platform team's required review and approval.

Do not assume that a chart that installs successfully into an empty namespace is automatically safe to upgrade in production.

## 6. Create the initial chart

### 6.1 Install and verify Helm

On a trusted developer workstation or CI runner:

```
helm version
helm help
helm lint --help
helm template --help
```

The actual Helm version should be pinned or controlled in CI. The commands above verify availability; they do not install Helm.

### 6.2 Create the chart directory

```
mkdir -p platform-helm-charts/charts/microservice/templates
mkdir -p platform-helm-charts/charts/microservice/tests
mkdir -p platform-helm-charts/examples/order-service
mkdir -p platform-helm-charts/scripts
```

Change into the repository:

```
cd platform-helm-charts
```

Create the chart files listed in the repository structure.

### 6.3 Establish a clean Git history

```
git init
git add charts/microservice
git status
git commit -m "feat(helm): add reusable microservice chart foundation"
```

In a shared repository, branch protections, pull-request reviews, and required CI checks should be configured before production releases depend on the chart.

## 7. Default values: the common configuration contract

Create `charts/microservice/values.yaml`:

```
# Human-readable service name.
nameOverride: ""

# Override the generated resource fullname only when needed.
fullnameOverride: ""

# Workload replica count when autoscaling is disabled.
replicaCount: 2

image:
  # Set an approved repository in the service/environment values.
  repository: ""

  # Use a digest for immutable production releases.
  digest: ""

  # Tag support is intended for controlled development workflows.
  # Production release validation must reject tag-only references.
  tag: ""

  pullPolicy: IfNotPresent

imagePullSecrets: []

serviceAccount:
  create: true
  name: ""
  annotations: {}
  automountServiceAccountToken: false

podAnnotations: {}
podLabels: {}

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  capabilities:
    drop:
      - ALL

containerPort: 8080

service:
  type: ClusterIP
  port: 8080
  targetPort: http
  annotations: {}

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

probes:
  startup:
    enabled: false
    path: /actuator/health
    initialDelaySeconds: 0
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 30

  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 2
    failureThreshold: 3

  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 3

terminationGracePeriodSeconds: 30

lifecycle: {}

env: []
envFrom: []

configMap:
  enabled: false
  data: {}

volumes: []
volumeMounts: []

nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []

deployment:
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 600
  maxSurge: 25%
  maxUnavailable: 0

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: null

pdb:
  enabled: false
  minAvailable: 1
  maxUnavailable: null

podDisruptionBudget:
  # Compatibility alias is intentionally not provided.
  # Use the "pdb" section consistently.

serviceMonitor:
  enabled: false
```

### 7.1 Why are these defaults conservative?

- The Service defaults to `ClusterIP`, avoiding accidental external exposure.
- The ServiceAccount token is not automatically mounted by default.
- The container drops Linux capabilities and disallows privilege escalation.
- The root filesystem is read-only by default.
- Autoscaling is disabled until a service explicitly configures it.
- PDB creation is disabled by default so a single-replica service is not unintentionally protected by an overly restrictive disruption policy.
- Production image digests are intended to be set through environment values.
- Probe defaults are suitable only for applications that implement those paths; each service must override them when necessary.

These settings are starting points, not universal truths. Validate them against each application's runtime and operational requirements.

### 7.2 Important image compatibility requirement

The example security context uses UID and GID `10001`. The container image must be built to run correctly with those IDs.

If the application needs to write to `/tmp`, a writable mounted volume may be required because the root filesystem is read-only.

Do not remove security controls automatically when a pod fails. First determine which filesystem path, permission, or process behavior is incompatible.

### 7.3 Example development values

Create `examples/order-service/values-dev.yaml`:

```
nameOverride: order-service

replicaCount: 1

image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service"
  tag: "dev-placeholder"
  digest: ""

containerPort: 8080

service:
  port: 8080
  targetPort: http

probes:
  startup:
    enabled: true
    path: /actuator/health
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 30

  liveness:
    enabled: true
    path: /actuator/health/liveness

  readiness:
    enabled: true
    path: /actuator/health/readiness

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

pdb:
  enabled: false
```

The image tag above is a placeholder for illustrating development configuration. Replace it with an actual built image tag before deploying. Production policy should require an immutable digest rather than accepting a mutable tag.

### 7.4 Example production values

Create `examples/order-service/values-prod.yaml`:

```
nameOverride: order-service

replicaCount: 3

image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service"
  digest: "sha256:REPLACE_WITH_REAL_64_CHARACTER_DIGEST"
  tag: ""

containerPort: 8080

service:
  type: ClusterIP
  port: 8080
  targetPort: http

probes:
  startup:
    enabled: true
    path: /actuator/health
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 30

  liveness:
    enabled: true
    path: /actuator/health/liveness
    periodSeconds: 10
    timeoutSeconds: 2
    failureThreshold: 3

  readiness:
    enabled: true
    path: /actuator/health/readiness
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 3

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

pdb:
  enabled: true
  minAvailable: 2

deployment:
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  maxSurge: 25%
  maxUnavailable: 0
```

The digest is deliberately a placeholder and is not deployable. The release pipeline must reject it until the real digest is supplied. Replica counts, resource settings, and scaling thresholds must be based on load testing and production behavior, not copied blindly to all services.

## 8. Template helpers

Create `charts/microservice/templates/_helpers.tpl`.

Helpers centralize naming and labels so every resource uses consistent values. A selector label must remain stable across chart upgrades; changing it can make a Deployment upgrade fail.

```
{{/*
Return the chart name, with an optional override.
*/}}
{{- define "microservice.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Return a stable resource fullname.
*/}}
{{- define "microservice.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{/*
Return the chart name and chart version label.
Replace "+" in chart versions because Kubernetes label values
cannot contain all SemVer characters.
*/}}
{{- define "microservice.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Common labels applied to resources.
*/}}
{{- define "microservice.labels" -}}
helm.sh/chart: {{ include "microservice.chart" . }}
{{ include "microservice.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- with .Values.commonLabels }}
{{ toYaml . }}
{{- end }}
{{- end -}}

{{/*
Selector labels must remain stable during upgrades.
Do not put chart version or image digest in these labels.
*/}}
{{- define "microservice.selectorLabels" -}}
app.kubernetes.io/name: {{ include "microservice.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/*
Return the effective ServiceAccount name.
*/}}
{{- define "microservice.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
{{- default (include "microservice.fullname" .) .Values.serviceAccount.name -}}
{{- else -}}
{{- default "default" .Values.serviceAccount.name -}}
{{- end -}}
{{- end -}}

{{/*
Return the image reference.
A digest is preferred when supplied.
The CI release validation must reject a missing production digest.
*/}}
{{- define "microservice.image" -}}
{{- if .Values.image.digest -}}
{{- printf "%s@%s" .Values.image.repository .Values.image.digest -}}
{{- else -}}
{{- printf "%s:%s" .Values.image.repository (default "latest" .Values.image.tag) -}}
{{- end -}}
{{- end -}}
```

### 8.1 Add the remaining default values

Add these keys to `values.yaml` if they are not already present:

```
commonLabels: {}
commonAnnotations: {}

podAnnotations: {}
podLabels: {}
```

### 8.2 Why selector labels matter

A Deployment uses a selector to identify the Pods it manages.

For example:

```
selector:
  matchLabels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/instance: order-service
```

The Pod template must contain the same labels.

Do not include the chart version in selector labels. A chart upgrade that changes a selector can make an immutable Deployment selector incompatible with the existing Deployment.

Use annotations for metadata that changes frequently, such as a configuration checksum.

### 8.3 Validate naming behavior

Render the chart with a known release name:

```
helm template order-service ./charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml
```

Inspect resource names and labels:

```
helm template order-service ./charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml \
  --debug
```

Verify that:

- Names are within Kubernetes naming limits.
- Names do not change unexpectedly between releases.
- The Deployment selector matches Pod labels.
- The Service selector matches Pod labels.
- Two releases in the same namespace do not unintentionally collide.

## 9. ServiceAccount template

Create `templates/serviceaccount.yaml`:

```
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "microservice.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}
{{- end }}
```

### 9.1 Why use a dedicated ServiceAccount?

Kubernetes assigns a ServiceAccount to a Pod. It provides an identity that can be used for Kubernetes API authorization and, when configured, workload identity integrations.

Do not give every microservice broad Kubernetes permissions simply because it runs in the cluster.

A service that only calls a database may not need Kubernetes API access at all. In that case, token automounting should generally remain disabled.

### 9.2 Example: Order Service

If the Order Service needs access to an AWS resource, the platform team should configure the appropriate AWS workload identity mechanism for the cluster and workload. The exact annotations and trust policy depend on whether the cluster uses EKS Pod Identity or IAM Roles for Service Accounts (IRSA).

Example values for an IRSA-based configuration:

```
serviceAccount:
  create: true
  name: order-service
  automountServiceAccountToken: false
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/order-service-role"
```

This example illustrates the chart interface only. The IAM role, trust relationship, permissions, and identity mechanism must be provisioned and verified separately.

Important: The `eks.amazonaws.com/role-arn` annotation is specific to IRSA. Do not assume it configures EKS Pod Identity.

### 9.3 Using an existing ServiceAccount

Some platform teams manage ServiceAccounts outside the application chart.

Configure:

```
serviceAccount:
  create: false
  name: order-service
  automountServiceAccountToken: false
```

The named ServiceAccount must already exist in the target namespace.

Do not set `create: false` with an empty name unless the cluster's default ServiceAccount is deliberately intended.

## 10. Kubernetes Service template

Create `templates/service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "microservice.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
  selector:
    {{- include "microservice.selectorLabels" . | nindent 4 }}
```

### 10.1 Why use a ClusterIP Service?

A `ClusterIP` Service exposes the application through a stable internal address inside the cluster.

For example:

```
order-service.ecommerce-prod.svc.cluster.local
```

The exact DNS name depends on the Service name, namespace, and cluster DNS domain.

A Service selects Pods through labels. Kubernetes networking then directs traffic to eligible endpoints.

The Service is not a replacement for an external load balancer or an ingress controller. External exposure should be explicitly designed in the relevant networking layer.

### 10.2 Port mapping example

Suppose the Order Service listens on TCP port `8080`.

The chart configuration is:

```
containerPort: 8080

service:
  port: 8080
  targetPort: http
```

The Deployment will define a named container port `http` mapped to `8080`. The Service's `targetPort: http` refers to that named container port.

If the application listens on port `8081` but the chart still targets `8080`, the Service can exist successfully while requests fail. Kubernetes does not verify that the application is listening on the configured port.

### 10.3 Service validation checklist

- The container is listening on the configured port.
- `service.port` matches the port expected by internal clients.
- `service.targetPort` resolves to a real container port.
- Service selectors match the Pod template labels.
- The Service has endpoints after Pods become ready.
- The Service is not exposed externally by accident.

Useful commands:

```
kubectl get svc -n ecommerce-prod
kubectl describe svc order-service -n ecommerce-prod
kubectl get endpoints order-service -n ecommerce-prod
kubectl get endpointslices -n ecommerce-prod
```

EndpointSlices are the modern API used to represent Service endpoints. A Service with no ready endpoints usually indicates that no eligible Pods match its selector or that its Pods are not ready.

## 11. Deployment template

Create `templates/deployment.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservice.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
  {{- with .Values.commonAnnotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  revisionHistoryLimit: {{ .Values.deployment.revisionHistoryLimit }}
  progressDeadlineSeconds: {{ .Values.deployment.progressDeadlineSeconds }}
  selector:
    matchLabels:
      {{- include "microservice.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.deployment.maxSurge }}
      maxUnavailable: {{ .Values.deployment.maxUnavailable }}
  template:
    metadata:
      labels:
        {{- include "microservice.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      annotations:
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      serviceAccountName: {{ include "microservice.serviceAccountName" . }}
      automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}

      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      containers:
        - name: {{ include "microservice.name" . }}
          image: {{ include "microservice.image" . | quote }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}

          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP

          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.startup.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.startup.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
          {{- end }}

          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          {{- end }}

          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
          {{- end }}

          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          {{- with .Values.lifecycle }}
          lifecycle:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}

      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 11.1 Deployment behavior

This template produces a Deployment that manages Pods and performs rolling updates.

When autoscaling is enabled, the template omits `spec.replicas`. This avoids repeatedly writing a fixed replica count from Helm while the HPA is managing the replica count.

When autoscaling is disabled, the chart sets replicas from `replicaCount`.

### 11.2 Why `maxUnavailable: 0`?

With a compatible number of replicas and sufficient cluster capacity, `maxUnavailable: 0` tells the Deployment controller not to intentionally reduce availability during a rolling update. It must bring up replacement capacity before removing old Pods.

This can increase temporary resource consumption because old and new Pods may coexist.

It also does not guarantee zero downtime. Application startup failures, readiness failures, capacity constraints, dependency outages, or incompatible releases can still cause disruption.

### 11.3 Startup, liveness, and readiness probes

These probes serve different purposes:

- Startup probe: allows a slow-starting application time to initialize before liveness and readiness checks become active.
- Liveness probe: detects an application process that needs restarting according to the chosen health contract.
- Readiness probe: determines whether a Pod should receive normal Service traffic.

For Spring Boot, the application team may expose dedicated health groups through Actuator. The paths must exist in the actual application configuration.

For Node.js, Python, and Go services, the application team must provide equivalent endpoints or configure another supported probe mechanism.

Do not make liveness depend directly on every downstream service. A temporary database outage should not automatically restart every application Pod if the application process itself remains healthy. Readiness may incorporate dependency availability when that matches the application's intended traffic policy.

### 11.4 Graceful termination

When a Pod is terminated, Kubernetes initiates the shutdown lifecycle and eventually sends a termination signal to the container process. The grace period allows the application to stop accepting work, finish in-flight requests, and close resources.

The application team must implement appropriate shutdown behavior. A `terminationGracePeriodSeconds` value alone does not create graceful shutdown.

For services that process messages, shutdown behavior should also consider:

- Stopping new message intake.
- Finishing or safely abandoning in-flight work.
- Committing offsets or acknowledgments only when appropriate.
- Avoiding duplicate business side effects.
- Releasing database connections.

### 11.5 Environment variables

Example values:

```
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "production"

  - name: SERVER_PORT
    value: "8080"

  - name: LOG_LEVEL
    value: "INFO"
```

For an environment-specific value sourced from a Kubernetes Secret:

```
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: password
```

These references do not create the Secret. A separate approved process must create and maintain it.

Avoid storing actual passwords, tokens, or private keys in chart defaults or committed GitOps values.

### 11.6 `envFrom`

Example:

```
envFrom:
  - configMapRef:
      name: order-service-runtime

  - secretRef:
      name: order-service-runtime-secret
```

Use `envFrom` when importing all suitable keys from a ConfigMap or Secret. Use `env` with `valueFrom` when a service needs precise control over which key maps to which variable.

A missing referenced Secret or ConfigMap can prevent a container from starting correctly. Check events and Pod descriptions during troubleshooting.

### 11.7 Writable temporary storage

If the image requires `/tmp` but the root filesystem is read-only, configure an explicit writable volume.

Values:

```
volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 128Mi

volumeMounts:
  - name: tmp
    mountPath: /tmp
```

This creates ephemeral storage tied to the Pod lifecycle. It is not persistent storage and must not be used for data that must survive Pod replacement.

### 11.8 Topology spread example

A production service may need to distribute replicas across Availability Zones or nodes.

Example values:

```
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/instance: order-service
```

The selector must match the actual Pod labels produced by the chart.

`DoNotSchedule` can leave Pods pending if the cluster does not have enough eligible topology domains or capacity. Use it only when that strict behavior is intended and tested.

### 11.9 Security context considerations

The default container security context:

- Disallows privilege escalation.
- Drops all Linux capabilities.
- Requires a non-root process.
- Uses a read-only root filesystem.

These controls reduce risk but can expose incompatible assumptions in application images.

Before rollout, validate:

- The image starts with the configured UID/GID.
- The process does not require root privileges.
- The application can write to only its approved writable paths.
- Required ports and filesystem permissions work.
- The application does not depend on adding Linux capabilities that are intentionally dropped.

Do not weaken the default security context without documenting the reason, obtaining the appropriate approval, and adding a test or policy exception where required.

## 12. Horizontal Pod Autoscaler

Create `templates/hpa.yaml`:

```
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "microservice.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "microservice.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 12.1 How HPA works

The Horizontal Pod Autoscaler adjusts the replica count of a supported workload according to configured metrics.

For CPU utilization, the metric is calculated relative to the CPU requests configured for the containers. If the CPU request is missing, CPU utilization-based scaling may not work as expected.

Example:

```
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

If the configured CPU target is 70%, the HPA attempts to keep average CPU utilization around the target when metrics and workload conditions permit. It does not guarantee that CPU usage will never exceed 70%.

### 12.2 Configure the HPA per service

For the Order Service:

```
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

For a lightweight service in a non-production environment:

```
autoscaling:
  enabled: false

replicaCount: 1
```

For a CPU-heavy service, the CPU threshold may need to be lower. For a service dominated by downstream latency, adding replicas might not solve the bottleneck.

The correct target must be established through load testing, production observations, and capacity planning.

### 12.3 Important HPA template correction

The template above must not produce an empty `metrics` list. If both metric targets are `null` or unset, the HPA manifest is invalid or unusable for the intended scaling configuration.

Use the schema and release validation to require at least one supported metric whenever `autoscaling.enabled` is `true`. The CI checks in Section 19 will enforce that contract.

### 12.4 CPU-based scaling is not queue-based scaling

For a Kafka consumer, CPU utilization may not reflect processing backlog. A consumer may be blocked on database calls while CPU remains low.

For message-processing workloads, use an appropriate metric such as consumer lag when supported by the selected autoscaling solution. KEDA is one option for scaling workloads based on event-source metrics.

The shared chart should not silently assume that every workload is HTTP request-driven. The chart can expose the common HPA pattern, while specialized worker charts or additional templates handle event-driven scaling.

### 12.5 Diagnose HPA problems

```
kubectl get hpa -n ecommerce-prod
kubectl describe hpa order-service -n ecommerce-prod
kubectl top pods -n ecommerce-prod
kubectl get deployment order-service -n ecommerce-prod
```

Look for:

- Missing CPU or memory requests.
- Metrics unavailable from the cluster metrics pipeline.
- HPA target misconfiguration.
- `minReplicas` greater than `maxReplicas`.
- Replica count already at the maximum.
- Pods failing to schedule due to insufficient capacity.
- Application saturation caused by a downstream dependency.
- A workload that needs queue-lag or custom-metric scaling rather than CPU scaling.

## 13. PodDisruptionBudget

Create `templates/pdb.yaml`:

```
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "microservice.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
spec:
  {{- if and (hasKey .Values.pdb "minAvailable") (ne .Values.pdb.minAvailable nil) }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- else if and (hasKey .Values.pdb "maxUnavailable") (ne .Values.pdb.maxUnavailable nil) }}
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  {{- else }}
  {{- fail "pdb.enabled requires either pdb.minAvailable or pdb.maxUnavailable" }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "microservice.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 13.1 Why use a PDB?

A PodDisruptionBudget limits certain voluntary disruptions, such as those caused by supported node-drain operations. It helps protect availability during maintenance.

A PDB does not prevent every possible outage. It does not protect against all involuntary failures, application crashes, or a bad deployment.

### 13.2 Example for a three-replica service

```
replicaCount: 3

pdb:
  enabled: true
  minAvailable: 2
  maxUnavailable: null
```

This requests that at least two matching Pods remain available during supported voluntary disruptions.

The PDB selector must match the intended Pods. An incorrect selector can cause the PDB to protect the wrong workload or no workload at all.

### 13.3 Why not enable the PDB everywhere?

A single-replica service with `minAvailable: 1` may prevent a node drain from proceeding because no matching Pod can be disrupted while preserving one available replica.

This can obstruct maintenance without providing true high availability.

For a service that needs to remain available during node maintenance, first ensure that:

- It has enough replicas.
- Replicas can be scheduled on separate eligible nodes or zones.
- The application can tolerate concurrent instances.
- Readiness probes correctly reflect service health.
- Cluster capacity exists to replace disrupted Pods.

Then choose a PDB that supports the maintenance objective.

## 14. Optional ConfigMap template

Create `templates/configmap.yaml`:

```
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "microservice.fullname" . }}-runtime
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
data:
  {{- toYaml .Values.configMap.data | nindent 2 }}
{{- end }}
```

Example values:

```
configMap:
  enabled: true
  data:
    LOG_LEVEL: "INFO"
    FEATURE_RECOMMENDATIONS_ENABLED: "true"
    REQUEST_TIMEOUT_MS: "2500"
```

### 14.1 What belongs in a ConfigMap?

ConfigMaps are appropriate for non-confidential configuration.

Examples:

- Log level.
- Feature flags that are safe to expose.
- Non-secret application settings.
- Timeout values.
- Runtime mode identifiers.

Do not store database passwords, private keys, access tokens, or other credentials in ConfigMaps.

### 14.2 Reference the generated ConfigMap

The Deployment can consume the chart-created ConfigMap:

```
envFrom:
  - configMapRef:
      name: order-service-runtime
```

The name must match the rendered ConfigMap name.

### 14.3 Configuration update behavior

Changing a ConfigMap does not automatically guarantee that an application process will reload its environment variables. Environment variables are established when the container starts.

If an application needs a restart after configuration changes, implement a controlled rollout mechanism. One common approach is to add a checksum annotation to the Pod template based on the rendered ConfigMap content. A content change then changes the Pod template and triggers a Deployment rollout.

If the chart adds that pattern, test it carefully and keep the checksum annotation stable when the ConfigMap content has not changed.

For configuration mounted as files, the application must still support reading updated file content. Kubernetes updating a projected file does not guarantee the application will reload it.

## 15. Helm chart schema

Create `charts/microservice/values.schema.json`.

The schema validates supported values before a release is rendered. It is a first line of defense, not a replacement for policy checks or Kubernetes API validation.

```
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "Reusable Microservice Helm Chart Values",
  "type": "object",
  "properties": {
    "nameOverride": {
      "type": "string",
      "maxLength": 63
    },
    "fullnameOverride": {
      "type": "string",
      "maxLength": 63
    },
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": {
          "type": "string"
        },
        "digest": {
          "type": "string",
          "pattern": "^(|sha256:[a-f0-9]{64})$"
        },
        "tag": {
          "type": "string"
        },
        "pullPolicy": {
          "type": "string",
          "enum": [
            "Always",
            "IfNotPresent",
            "Never"
          ]
        }
      },
      "required": [
        "repository",
        "digest",
        "tag",
        "pullPolicy"
      ]
    },
    "containerPort": {
      "type": "integer",
      "minimum": 1,
      "maximum": 65535
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": [
            "ClusterIP",
            "NodePort",
            "LoadBalancer"
          ]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        },
        "targetPort": {
          "oneOf": [
            {
              "type": "integer",
              "minimum": 1,
              "maximum": 65535
            },
            {
              "type": "string",
              "minLength": 1
            }
          ]
        },
        "annotations": {
          "type": "object"
        }
      },
      "required": [
        "type",
        "port",
        "targetPort"
      ]
    },
    "resources": {
      "type": "object",
      "properties": {
        "requests": {
          "type": "object",
          "required": [
            "cpu",
            "memory"
          ],
          "properties": {
            "cpu": {
              "type": "string",
              "minLength": 1
            },
            "memory": {
              "type": "string",
              "minLength": 1
            }
          }
        },
        "limits": {
          "type": "object",
          "properties": {
            "cpu": {
              "type": "string",
              "minLength": 1
            },
            "memory": {
              "type": "string",
              "minLength": 1
            }
          }
        }
      },
      "required": [
        "requests",
        "limits"
      ]
    },
    "serviceAccount": {
      "type": "object",
      "properties": {
        "create": {
          "type": "boolean"
        },
        "name": {
          "type": "string"
        },
        "automountServiceAccountToken": {
          "type": "boolean"
        },
        "annotations": {
          "type": "object"
        }
      },
      "required": [
        "create",
        "automountServiceAccountToken"
      ]
    },
    "autoscaling": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "minReplicas": {
          "type": "integer",
          "minimum": 1
        },
        "maxReplicas": {
          "type": "integer",
          "minimum": 1
        },
        "targetCPUUtilizationPercentage": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1,
          "maximum": 100
        },
        "targetMemoryUtilizationPercentage": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1,
          "maximum": 100
        }
      },
      "required": [
        "enabled",
        "minReplicas",
        "maxReplicas"
      ]
    },
    "pdb": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "minAvailable": {
          "type": [
            "integer",
            "string",
            "null"
          ],
          "minimum": 0
        },
        "maxUnavailable": {
          "type": [
            "integer",
            "string",
            "null"
          ],
          "minimum": 0
        }
      },
      "required": [
        "enabled"
      ]
    },
    "probes": {
      "type": "object",
      "properties": {
        "startup": {
          "$ref": "#/definitions/probe"
        },
        "liveness": {
          "$ref": "#/definitions/probe"
        },
        "readiness": {
          "$ref": "#/definitions/probe"
        }
      }
    }
  },
  "definitions": {
    "probe": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "path": {
          "type": "string",
          "minLength": 1
        },
        "initialDelaySeconds": {
          "type": "integer",
          "minimum": 0
        },
        "periodSeconds": {
          "type": "integer",
          "minimum": 1
        },
        "timeoutSeconds": {
          "type": "integer",
          "minimum": 1
        },
        "failureThreshold": {
          "type": "integer",
          "minimum": 1
        }
      },
      "required": [
        "enabled",
        "path"
      ]
    }
  }
}
```

### 15.1 Schema limitations

The schema above validates many types and ranges, but it does not enforce every cross-field rule.

Examples that still need explicit validation:

- `image.repository` must not be empty for a release.
- Production must specify a real image digest.
- A tag and digest must not both be set.
- Autoscaling minimum replicas must not exceed maximum replicas.
- At least one HPA metric must be configured when autoscaling is enabled.
- Exactly one of `pdb.minAvailable` and `pdb.maxUnavailable` must be configured when the PDB is enabled.
- The selected image must actually support the configured security context.
- A production Service must not be changed to `LoadBalancer` without an approved exposure design.

Some cross-field rules can be expressed in a richer schema, depending on the supported JSON Schema draft and Helm version. Others are clearer and easier to audit in dedicated release-validation scripts and policy tests.

### 15.2 Keep schema and values synchronized

Whenever a new values key is added:

1. Add it to `values.yaml`.
2. Implement it in the relevant template.
3. Add schema validation.
4. Document its purpose.
5. Add at least one example.
6. Add a rendering test.
7. Confirm that existing service values still work.

A value that exists in `values.yaml` but is never consumed by a template can mislead application teams into believing it has an effect.

## 16. Helm ignore file and chart documentation

Create `.helmignore`:

```
.git/
.gitignore
.github/
.idea/
.vscode/
*.swp
*.tmp
*.bak
.DS_Store
tests/
docs/
```

Only exclude paths that should not be shipped in the chart package. Check the generated chart archive before changing this file, especially if tests or documentation are intentionally packaged for consumers.

Create `templates/NOTES.txt`:

```
Release: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

Service:
  Name: {{ include "microservice.fullname" . }}
  Type: {{ .Values.service.type }}
  Port: {{ .Values.service.port }}

Deployment:
  Name: {{ include "microservice.fullname" . }}

{{- if .Values.autoscaling.enabled }}
Autoscaling:
  Minimum replicas: {{ .Values.autoscaling.minReplicas }}
  Maximum replicas: {{ .Values.autoscaling.maxReplicas }}
{{- else }}
Replicas:
  {{ .Values.replicaCount }}
{{- end }}

Inspect the deployment:
  kubectl get deployment {{ include "microservice.fullname" . }} -n {{ .Release.Namespace }}

Inspect the Service:
  kubectl get service {{ include "microservice.fullname" . }} -n {{ .Release.Namespace }}

Inspect Pods:
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}
```

Create `README.md` inside the chart directory with:

- Supported chart version.
- Supported Kubernetes versions.
- Required values.
- Image reference rules.
- Probe requirements.
- Resource configuration guidance.
- ServiceAccount behavior.
- Autoscaling behavior.
- PDB limitations.
- Secret-handling rules.
- Upgrade and rollback notes.
- Examples for development and production.
- Known limitations and exceptions.

## 17. Release validation: rules that templates alone cannot enforce

Create `scripts/validate-release-values.sh`:

```
#!/usr/bin/env bash

set -euo pipefail

VALUES_FILE="${1:?Usage: validate-release-values.sh <values-file> [environment]}"
ENVIRONMENT="${2:-dev}"

fail() {
  echo "ERROR: $*" >&2
  exit 1
}

require_command() {
  command -v "$1" >/dev/null 2>&1 ||
    fail "Required command not found: $1"
}

require_command yq

REPOSITORY="$(yq -r '.image.repository // ""' "$VALUES_FILE")"
DIGEST="$(yq -r '.image.digest // ""' "$VALUES_FILE")"
TAG="$(yq -r '.image.tag // ""' "$VALUES_FILE")"

AUTOSCALING_ENABLED="$(yq -r '.autoscaling.enabled // false' "$VALUES_FILE")"
MIN_REPLICAS="$(yq -r '.autoscaling.minReplicas // 1' "$VALUES_FILE")"
MAX_REPLICAS="$(yq -r '.autoscaling.maxReplicas // 1' "$VALUES_FILE")"
CPU_TARGET="$(yq -r '.autoscaling.targetCPUUtilizationPercentage // ""' "$VALUES_FILE")"
MEMORY_TARGET="$(yq -r '.autoscaling.targetMemoryUtilizationPercentage // ""' "$VALUES_FILE")"

PDB_ENABLED="$(yq -r '.pdb.enabled // false' "$VALUES_FILE")"
PDB_MIN="$(yq -r '.pdb.minAvailable // ""' "$VALUES_FILE")"
PDB_MAX="$(yq -r '.pdb.maxUnavailable // ""' "$VALUES_FILE")"

[[ -n "$REPOSITORY" ]] ||
  fail "image.repository must be set"

if [[ -n "$DIGEST" && -n "$TAG" ]]; then
  fail "Set either image.digest or image.tag, not both"
fi

if [[ "$ENVIRONMENT" == "prod" ]]; then
  [[ "$DIGEST" =~ ^sha256:[a-f0-9]{64}$ ]] ||
    fail "Production requires a real sha256 image digest"

  [[ -z "$TAG" ]] ||
    fail "Production must not use image.tag"
fi

if [[ -n "$DIGEST" ]]; then
  [[ "$DIGEST" =~ ^sha256:[a-f0-9]{64}$ ]] ||
    fail "image.digest must be sha256 followed by 64 lowercase hex characters"
elif [[ -z "$TAG" ]]; then
  fail "Either image.digest or image.tag must be set"
fi

if [[ "$AUTOSCALING_ENABLED" == "true" ]]; then
  [[ "$MIN_REPLICAS" =~ ^[0-9]+$ ]] ||
    fail "autoscaling.minReplicas must be an integer"

  [[ "$MAX_REPLICAS" =~ ^[0-9]+$ ]] ||
    fail "autoscaling.maxReplicas must be an integer"

  (( MIN_REPLICAS >= 1 )) ||
    fail "autoscaling.minReplicas must be at least 1"

  (( MAX_REPLICAS >= MIN_REPLICAS )) ||
    fail "autoscaling.maxReplicas must be >= minReplicas"

  [[ -n "$CPU_TARGET" || -n "$MEMORY_TARGET" ]] ||
    fail "Autoscaling requires a CPU target, a memory target, or both"
fi

if [[ "$PDB_ENABLED" == "true" ]]; then
  if [[ -n "$PDB_MIN" && -n "$PDB_MAX" ]]; then
    fail "Set either pdb.minAvailable or pdb.maxUnavailable, not both"
  fi

  if [[ -z "$PDB_MIN" && -z "$PDB_MAX" ]]; then
    fail "PDB enabled but neither pdb.minAvailable nor pdb.maxUnavailable is set"
  fi
fi

echo "Release values validation passed for environment: $ENVIRONMENT"
```

### 17.1 Script assumptions

This script assumes a compatible `yq` implementation that supports expressions such as `yq -r '.image.repository' file.yaml`. Pin the exact `yq` implementation and version in CI; different tools use the same command name.

This script is a minimum contract check, not a complete policy engine. It does not prove that:

- The image exists in ECR.
- The digest belongs to the expected repository.
- The image was scanned or signed.
- The cluster can pull the image.
- The application implements its probe paths.
- The generated manifests pass Kubernetes admission policies.

Those checks belong in the wider release pipeline.

### 17.2 Validate both development and production examples

```
chmod +x scripts/validate-release-values.sh

./scripts/validate-release-values.sh \
  examples/order-service/values-dev.yaml dev

./scripts/validate-release-values.sh \
  examples/order-service/values-prod.yaml prod
```

The production example will fail until the placeholder digest is replaced by a valid digest. That is intentional: the sample demonstrates the required format, not a deployable image.

## 18. Render and inspect generated manifests

Before running any deployment, render the chart locally.

### 18.1 Basic lint

```
helm lint charts/microservice
```

This checks chart structure and some common chart issues. It does not replace Kubernetes API validation, security-policy evaluation, or runtime testing.

### 18.2 Render development manifests

```
helm template order-service charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml \
  --output-dir /tmp/order-service-rendered
```

Inspect the output:

```
find /tmp/order-service-rendered -type f -maxdepth 3 -print
```

The output directory and file layout can vary with Helm version and chart structure. Inspect the generated files rather than assuming a particular filename.

### 18.3 Render production manifests

Use the actual release digest before validating production:

```
helm template order-service charts/microservice \
  --namespace ecommerce-prod \
  -f examples/order-service/values-prod.yaml \
  --output-dir /tmp/order-service-prod-rendered
```

A successful render does not prove that the image exists or that the Kubernetes resources will run. It proves only that Helm can render the supplied values and templates.

### 18.4 Check the rendered image

```
grep -R "image:" /tmp/order-service-prod-rendered
```

For a production release, the resulting image reference should use the expected registry, repository, and immutable digest.

### 18.5 Inspect resource selectors

```
grep -R -n "selector:" /tmp/order-service-rendered
grep -R -n "app.kubernetes.io/name" /tmp/order-service-rendered
grep -R -n "app.kubernetes.io/instance" /tmp/order-service-rendered
```

Confirm that the Deployment selector, Pod labels, and Service selector agree.

### 18.6 Inspect security configuration

```
grep -R -n "runAsNonRoot" /tmp/order-service-rendered
grep -R -n "allowPrivilegeEscalation" /tmp/order-service-rendered
grep -R -n "readOnlyRootFilesystem" /tmp/order-service-rendered
grep -R -n "capabilities" /tmp/order-service-rendered
```

These checks help reviewers inspect generated output. They are not substitutes for policy-as-code checks that fail automatically when an unsafe configuration is introduced.

## 19. Complete chart validation pipeline

The chart repository needs its own CI pipeline. An application pipeline cannot guarantee the shared chart is safe for all 21 services.

Create `.github/workflows/validate-chart.yaml`:

```
name: Validate Reusable Helm Chart

on:
  pull_request:
    paths:
      - "charts/microservice/**"
      - "examples/**"
      - "scripts/**"
      - ".github/workflows/validate-chart.yaml"
  push:
    branches:
      - main
    paths:
      - "charts/microservice/**"
      - "examples/**"
      - "scripts/**"
      - ".github/workflows/validate-chart.yaml"

permissions:
  contents: read

jobs:
  validate:
    name: Lint, Render and Validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.16.4

      - name: Install yq
        run: |
          set -euo pipefail
          YQ_VERSION="v4.44.3"
          curl -fsSL \
            "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" \
            -o /tmp/yq
          chmod +x /tmp/yq
          sudo install /tmp/yq /usr/local/bin/yq
          yq --version

      - name: Lint chart
        run: |
          set -euo pipefail
          helm lint charts/microservice

      - name: Validate development values
        run: |
          ./scripts/validate-release-values.sh \
            examples/order-service/values-dev.yaml dev

      - name: Validate production values
        run: |
          # The repository example must contain a real digest
          # if this step is enabled as a strict release gate.
          ./scripts/validate-release-values.sh \
            examples/order-service/values-prod.yaml prod

      - name: Render development chart
        run: |
          set -euo pipefail
          helm template order-service charts/microservice \
            --namespace ecommerce-dev \
            --values examples/order-service/values-dev.yaml \
            --output-dir /tmp/order-service-dev

      - name: Render production chart
        run: |
          set -euo pipefail
          helm template order-service charts/microservice \
            --namespace ecommerce-prod \
            --values examples/order-service/values-prod.yaml \
            --output-dir /tmp/order-service-prod

      - name: Upload rendered manifests
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: rendered-helm-manifests
          path: |
            /tmp/order-service-dev
            /tmp/order-service-prod
          if-no-files-found: warn
```

### 19.1 Important: the production example is intentionally strict

The production example contains a placeholder digest. With the strict validation step above, CI should fail until the example is updated.

For a working pull-request pipeline, choose one of these approaches:

1. Maintain a known, valid, approved test-image digest for the production fixture.
2. Use a dedicated test values file with a syntactically valid digest for rendering-only tests, and separately test the real release validation.
3. Split chart validation from release validation so that a template change does not require publishing a new application image.

Do not weaken production validation simply to make a placeholder example pass.

A well-designed pipeline separates chart correctness from release artifact correctness.

### 19.2 Pinning GitHub Actions

The workflow uses version tags for readability. Organizations with stricter supply-chain requirements should pin third-party actions to reviewed full commit SHAs and update them through a controlled process.

Also pin and verify external binaries. Downloading a binary over HTTPS does not by itself prove that the binary is the intended artifact.

### 19.3 Extend validation beyond Helm lint

A production chart pipeline should progressively add:

- JSON/YAML syntax validation.
- Helm schema validation.
- Kubernetes schema validation using a tool such as `kubeconform`.
- Policy checks using Conftest/OPA, Kyverno CLI, or an equivalent approved policy engine.
- Unit tests for rendered templates.
- Upgrade tests.
- Secret scanning.
- Dependency and artifact checks.
- Release artifact integrity checks.
- A protected chart release workflow.

Each check should have a clear failure message and an owner responsible for resolving it.

## 20. Kubernetes API schema validation

Helm lint is useful but does not prove that every generated object is valid for the target Kubernetes version.

A tool such as `kubeconform` can validate rendered YAML against Kubernetes schemas.

Example:

```
helm template order-service charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml \
  > /tmp/order-service.yaml
```

Then:

```
kubeconform \
  -strict \
  -summary \
  /tmp/order-service.yaml
```

The tool must be installed and version-pinned in CI. Configure the schemas and Kubernetes version to match the cluster versions supported by the platform.

### 20.1 What schema validation can catch

Examples include:

- Invalid Kubernetes field names.
- Incorrect field types.
- Unsupported resource fields.
- Invalid structures in HPA or PDB objects.
- Incorrect object shapes.

### 20.2 What schema validation cannot catch

It generally cannot establish that:

- A referenced Secret exists.
- An ECR image can be pulled.
- A probe endpoint returns a healthy response.
- A Pod can fit on the available nodes.
- An IAM trust policy is correct.
- A database is reachable.
- A release meets business acceptance criteria.

Those require additional validation.

## 21. Policy-as-code checks

A reusable chart must enforce organizational guardrails without pretending that a rendered manifest alone proves runtime security.

Example policy requirements:

| Policy               | Example rule                                                |
| -------------------- | ----------------------------------------------------------- |
| Image immutability   | Production workload images must use digests                 |
| Non-root execution   | Workload containers must not run as root                    |
| Privilege escalation | Containers must disallow privilege escalation               |
| Linux capabilities   | Drop unnecessary capabilities                               |
| Resource requests    | Workloads must define CPU and memory requests               |
| External exposure    | LoadBalancer Services require explicit approval             |
| Secret handling      | No literal passwords or tokens in values                    |
| Probe contract       | HTTP services must define appropriate health checks         |
| Ownership            | Resources must carry required application and team metadata |

A policy rule should be designed around the actual organization standard. Some workloads, such as specialized agents or infrastructure components, may require different controls and explicit exceptions.

### 21.1 Example Conftest workflow step

Assume the organization maintains reviewed policies in `policies/`:

```
helm template order-service charts/microservice \
  --namespace ecommerce-prod \
  -f examples/order-service/values-prod.yaml \
  > /tmp/order-service.yaml

conftest test /tmp/order-service.yaml \
  --policy policies/
```

This command assumes Conftest is installed and the policy directory contains applicable rules. The pipeline must fail if the policy engine reports a violation.

Do not create policies that only print warnings while allowing the production deployment to proceed when the rule is meant to be mandatory.

### 21.2 Policy exceptions

An exception should record:

- The workload.
- The policy being waived.
- The technical reason.
- The risk accepted.
- The compensating control.
- The approving owner.
- The expiration or review date.

Avoid permanent exceptions that silently spread from one service to all others.

## 22. Chart unit tests

Rendering a chart once is not enough. The platform team should verify specific expected behavior.

A chart unit-testing framework, such as `helm-unittest`, can test rendered resources against assertions.

Example test intent:

```
suite: deployment defaults
templates:
  - deployment.yaml

tests:
  - it: creates a Deployment
    asserts:
      - isKind:
          of: Deployment

  - it: uses the configured replica count when autoscaling is disabled
    set:
      replicaCount: 3
      autoscaling.enabled: false
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: omits fixed replicas when autoscaling is enabled
    set:
      autoscaling.enabled: true
      autoscaling.minReplicas: 2
      autoscaling.maxReplicas: 8
      autoscaling.targetCPUUtilizationPercentage: 70
    asserts:
      - notExists:
          path: spec.replicas

  - it: uses the non-root container security context
    asserts:
      - equal:
          path: spec.template.spec.containers[0].securityContext.runAsNonRoot
          value: true
      - equal:
          path: spec.template.spec.containers[0].securityContext.allowPrivilegeEscalation
          value: false
```

This is an illustrative suite. Before using it, confirm the test framework version and assertion syntax, then run the tests against the exact chart version in the repository.

### 22.1 Required test scenarios

At minimum, test:

1. Default values.
2. Development values.
3. Production values with a valid fixture digest.
4. Autoscaling enabled.
5. Autoscaling disabled.
6. ServiceAccount creation enabled.
7. Existing ServiceAccount selected.
8. Startup probe enabled.
9. Optional ConfigMap enabled.
10. Writable temporary volume configured.
11. PDB enabled.
12. PDB disabled.
13. Custom annotations.
14. Custom labels.
15. Topology-spread constraints.
16. Invalid values rejected.
17. Image reference validation.
18. Stable selectors across chart versions.

## 23. Testing chart upgrades

A chart can render correctly from scratch and still break an existing installation.

Use a test cluster or isolated test namespace to validate upgrade behavior.

A typical test sequence is:

```
helm upgrade --install order-service \
  ./charts/microservice \
  --namespace helm-upgrade-test \
  --create-namespace \
  --values examples/order-service/values-dev.yaml \
  --wait \
  --timeout 5m
```

After the initial release, introduce a chart change and run:

```
helm upgrade order-service \
  ./charts/microservice \
  --namespace helm-upgrade-test \
  --values examples/order-service/values-dev.yaml \
  --wait \
  --timeout 5m
```

Inspect the resulting release:

```
helm status order-service -n helm-upgrade-test
helm history order-service -n helm-upgrade-test
kubectl rollout status deployment/order-service -n helm-upgrade-test
```

The actual Deployment name depends on the configured release name and chart naming rules.

### 23.1 Upgrade test cases

Test changes such as:

- Adding an optional field.
- Changing a default value.
- Changing a Service annotation.
- Adding a new volume.
- Changing HPA configuration.
- Updating probe defaults.
- Introducing a new required value.
- Renaming a values key.
- Changing generated resource names.
- Changing labels or selectors.

For every breaking change, define a migration procedure and an upgrade path. If a breaking change is necessary, a new major chart version may be appropriate.

### 23.2 Rollback tests

A chart rollback is not guaranteed to restore every external side effect.

Test whether the release can be rolled back to a previous chart revision and whether the application remains compatible with the previous configuration and image.

Pay special attention to:

- Database schema changes.
- Message formats.
- External configuration.
- IAM policy changes.
- Persistent data changes.
- Kubernetes resources that are not safely reversible.

Helm can restore Kubernetes release state, but it cannot automatically reverse every database migration or business operation.

## 24. Package and publish the chart

After chart validation passes, package the chart:

```
helm package charts/microservice --destination dist/
```

Inspect the generated artifact:

```
ls -lh dist/
```

The filename includes the chart name and version, for example:

```
microservice-1.0.0.tgz
```

Do not overwrite an already published chart version. Publish a new version for a changed artifact.

### 24.1 Publish to Amazon ECR as an OCI artifact

Helm supports OCI registries. The organization must create the ECR repository and configure permissions before publishing.

Example variables:

```
export AWS_REGION="ap-south-1"
export AWS_ACCOUNT_ID="123456789012"
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
export CHART_REPOSITORY="helm"
```

Authenticate to ECR:

```
aws ecr get-login-password --region "$AWS_REGION" \
  | helm registry login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"
```

Push the packaged chart:

```
helm push dist/microservice-1.0.0.tgz \
  "oci://${ECR_REGISTRY}/${CHART_REPOSITORY}"
```

The command assumes the relevant ECR repository exists and the caller has the necessary permissions. The chart is published under the configured OCI repository namespace. Confirm the exact artifact path and retrieval behavior with the registry and Helm versions used by the organization.

### 24.2 Required ECR permissions

The publishing identity needs permissions appropriate to authenticate and push the chart artifact. Depending on the repository and organization configuration, this includes authorization-token access and the relevant image-layer and image-manifest upload operations.

Do not grant the chart publishing workflow broad administrator permissions just to make the push succeed.

### 24.3 Chart release identity

A chart release should be traceable to:

- Git commit.
- Pull request and reviewers.
- Chart version.
- CI run.
- Validation results.
- Published artifact digest or equivalent registry identity.
- Release timestamp.
- Release notes.

Where the organization's supply-chain tooling supports it, generate provenance or attestations and verify them before consuming the artifact.

## 25. Chart consumption options

There are two common ways to consume a reusable chart.

### Option A: Chart source lives in Git

Argo CD reads a chart directly from a Git repository, often alongside the application's values. This is straightforward for Git-based chart sources, although it may mean consuming a chart commit rather than a separately published chart package.

### Option B: Chart is published as an OCI artifact

Argo CD retrieves a versioned chart from a registry such as ECR. This supports explicit chart artifact releases, but repository authentication and OCI source configuration must be set up correctly.

Do not mix the two approaches in the same configuration without a clear reason.

For this project, a published OCI chart provides a clear separation between the chart release and the GitOps environment promotion. The examples below assume that the ECR repository, chart artifact, and Argo CD repository credentials have been configured for the installed Argo CD version.

## 26. Argo CD integration: recommended GitOps flow

The deployment flow is:

1. An application developer opens a pull request.
2. CI tests the application and scans dependencies.
3. CI builds and scans a container image.
4. The approved image is published to ECR.
5. The image digest is recorded.
6. The GitOps change references the approved chart version and image digest.
7. Review and approval rules are applied.
8. Argo CD reconciles the GitOps declaration.
9. Kubernetes performs the rollout.
10. Monitoring and release checks validate the result.

Argo CD does not need to rebuild the application. It applies the desired deployment state.

### 26.1 OCI chart and separate values repository

For multi-source Argo CD Applications, use a configuration supported by the exact Argo CD release installed in the cluster. OCI Helm source handling and multi-source values behavior can differ between versions.

A common pattern is to define:

- One source for the OCI Helm chart.
- Another source for the Git repository containing environment values.
- A `ref` alias for the values source.
- A chart source `helm.valueFiles` entry referencing the values repository alias.

The exact Application manifest should be validated against the deployed Argo CD version before production use. Do not copy a multi-source example into production without testing authentication, path resolution, chart version resolution, and sync behavior.

### 26.2 Alternative: Git-based chart consumption

If the chart and environment values are stored in Git, the Application can point at the chart path and supply values from the repository.

This is often simpler to bootstrap, but chart versioning and promotion then depend on Git references and repository conventions. Protect the relevant branches and ensure production tracks an approved, immutable commit or release tag according to the organization's policy.

### 26.3 GitOps promotion rules

A safe promotion policy should ensure:

- Development, QA, staging, and production values are explicit.
- Production never silently tracks an unreviewed development branch.
- Image digests are promoted rather than rebuilt separately for each environment.
- Chart versions are explicit.
- Changes are reviewed according to environment risk.
- Rollback procedures are documented and tested.
- Argo CD health status and application-level checks are part of release verification.

The same image digest should normally move through environments when the goal is to promote the exact artifact that was tested. Environment-specific configuration can change, but the artifact identity should remain traceable.

## 27. Environment-specific configuration for the 21 microservices

The reusable chart should provide consistent deployment mechanics while allowing each service to declare its own configuration.

The following are starting-point examples. They are not production sizing recommendations; actual values must be validated through testing and observed workload behavior.

| Service        | Example runtime port | Probe family                | Important scaling consideration                           |
| -------------- | -------------------- | --------------------------- | --------------------------------------------------------- |
| Auth           | 8080                 | Application health endpoint | Authentication latency and identity-provider dependencies |
| User           | 8080                 | Application health endpoint | Database connection limits                                |
| Customer       | 8080                 | Application health endpoint | Database and downstream API latency                       |
| Catalog        | 8080                 | Application health endpoint | Read-heavy request traffic                                |
| Product        | 8080                 | Application health endpoint | Database and cache utilization                            |
| Inventory      | 8080                 | Application health endpoint | Consistency and concurrent updates                        |
| Pricing        | 8080                 | Application health endpoint | CPU demand and rule evaluation                            |
| Promotion      | 8080                 | Application health endpoint | Traffic spikes during campaigns                           |
| Cart           | 8080                 | Application health endpoint | Latency and cache/database access                         |
| Order          | 8080                 | Application health endpoint | Database capacity and event processing                    |
| Payment        | 8080                 | Application health endpoint | External payment-provider latency                         |
| Fraud          | 8080                 | Application health endpoint | CPU-intensive scoring or model execution                  |
| Shipping       | 8080                 | Application health endpoint | External carrier API latency                              |
| Delivery       | 8080                 | Application health endpoint | Event processing and delivery updates                     |
| Notification   | 8080                 | Application health endpoint | Queue depth and provider rate limits                      |
| Search         | 8080                 | Application health endpoint | Search backend capacity and query cost                    |
| Recommendation | 8080                 | Application health endpoint | CPU/memory usage and inference workload                   |
| Review         | 8080                 | Application health endpoint | Database and read/write mix                               |
| Tax            | 8080                 | Application health endpoint | External tax API latency and calculation load             |
| Invoice        | 8080                 | Application health endpoint | Document generation and downstream dependencies           |
| Reporting      | 8080                 | Application health endpoint | Heavy queries, batch processing, and memory               |

These port numbers are illustrative. Confirm the actual port and health contract for each application before deploying it.

### 27.1 Spring Boot example

A Spring Boot service may use:

```
containerPort: 8080

probes:
  startup:
    enabled: true
    path: /actuator/health
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 30

  liveness:
    enabled: true
    path: /actuator/health/liveness
    periodSeconds: 10
    timeoutSeconds: 2
    failureThreshold: 3

  readiness:
    enabled: true
    path: /actuator/health/readiness
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 3
```

The Java developer must add and configure the required Actuator endpoints. The chart cannot create an endpoint inside the application.

### 27.2 Node.js example

A Node.js application might implement `/health/live` and `/health/ready`.

Its values could be:

```
containerPort: 3000

service:
  port: 8080
  targetPort: http

probes:
  startup:
    enabled: false

  liveness:
    enabled: true
    path: /health/live

  readiness:
    enabled: true
    path: /health/ready
```

The Deployment's named `http` port will use `containerPort: 3000`, while the Service remains reachable on port `8080`.

This is valid when the application listens on port `3000` and the Service maps traffic to that port. The application must actually implement the configured endpoints.

### 27.3 Python example

A Python application might expose:

```
containerPort: 8000

probes:
  startup:
    enabled: false

  liveness:
    enabled: true
    path: /health/live

  readiness:
    enabled: true
    path: /health/ready

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

For CPU-bound Python work, scaling behavior depends on the runtime, concurrency model, workload, and process configuration. Do not choose replica counts from language alone.

### 27.4 Go example

A Go service may use:

```
containerPort: 8080

probes:
  startup:
    enabled: false

  liveness:
    enabled: true
    path: /healthz

  readiness:
    enabled: true
    path: /readyz

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

A small binary does not automatically mean low memory consumption or high throughput. Measure actual behavior.

## 28. Advanced chart safeguards

The chart has a usable foundation, but the platform team should not release it as a universal production chart until several safeguards are implemented and tested.

### 28.1 Require an explicit image

The current image helper has a development fallback of `latest` if neither digest nor tag is configured. That fallback is convenient for initial exploration but undesirable for a production contract.

A stronger implementation is to fail rendering when no image reference is provided.

Replace the image helper with:

```
{{- define "microservice.image" -}}
{{- $repository := required "image.repository is required" .Values.image.repository -}}
{{- if .Values.image.digest -}}
{{- printf "%s@%s" $repository .Values.image.digest -}}
{{- else if .Values.image.tag -}}
{{- printf "%s:%s" $repository .Values.image.tag -}}
{{- else -}}
{{- fail "Set image.digest or image.tag" -}}
{{- end -}}
{{- end -}}
```

The release validation script must still enforce the stricter production rule: a valid digest is required, and tag-only production releases are rejected.

### 28.2 Avoid duplicate configuration ownership

The ServiceAccount token setting appears in both the ServiceAccount and Pod configuration. This is intentional in the example, but the values contract must define the desired behavior clearly.

The platform team should choose a consistent approach:

- Use one values field to control token automounting for the Pod.
- Set ServiceAccount automounting according to the platform's chosen default.
- Avoid creating confusing configurations in which the two settings disagree.

The Pod-level setting can override the ServiceAccount-level default. Document the effective behavior rather than assuming both fields always have the same result.

### 28.3 Validate lifecycle configuration

The Deployment template renders `.Values.lifecycle` directly. This allows application teams to specify container lifecycle hooks, but it also means arbitrary hook configuration can enter a production workload.

Validate lifecycle hooks against the Kubernetes schema and platform policies. Prefer application-level graceful shutdown support over adding unnecessary shell commands to every container.

### 28.4 Protect stable resource identity

Before releasing a chart update, compare rendered manifests against the previous chart version.

Flag unexpected changes to:

- Deployment names.
- Service names.
- Selector labels.
- ServiceAccount names.
- ConfigMap names.
- Namespace assumptions.
- Resource ownership labels.

Some changes are intentional, but they should be reviewed as migration changes rather than treated as ordinary template edits.

## 29. Production deployment checklist

Before a service consumes the reusable chart in production, verify the following.

### Application readiness

- Application listens on the declared port.
- Startup, liveness, and readiness endpoints exist and return expected status codes.
- Graceful termination is implemented.
- The image runs with the configured UID/GID.
- The application supports a read-only root filesystem or declares required writable mounts.
- Database and message-processing connection limits have been reviewed.
- Application configuration is externalized appropriately.
- Secrets are retrieved using the approved mechanism.

### Chart readiness

- Chart metadata and version are correct.
- Values schema is synchronized with the templates.
- Names and selectors are stable.
- Image reference validation is enforced.
- Resource requests and limits are set.
- Probe behavior is configurable.
- HPA is disabled unless the service has an appropriate scaling configuration.
- PDB is disabled unless its disruption behavior is understood.
- Optional resources render correctly.
- Security policy checks pass.
- Upgrade tests pass.

### Release readiness

- CI tests and security checks pass.
- Image is published to the approved registry.
- Image digest is recorded and verified.
- Chart artifact is versioned and published.
- GitOps configuration references the intended chart and image.
- Required reviewers approve the change.
- Rollback instructions are available.
- Monitoring and release verification are ready.

## 30. Troubleshooting playbook

### Scenario 1 — Helm template fails

Symptoms

```
Error: template: ... executing ...
```

Diagnosis

```
helm lint charts/microservice --debug
```

Then render the exact service values:

```
helm template order-service charts/microservice \
  --namespace ecommerce-dev \
  -f examples/order-service/values-dev.yaml \
  --debug
```

Investigate

- Missing required values.
- Incorrect indentation in a template.
- A helper name that does not exist.
- Invalid values types.
- An invalid schema.
- Incorrect use of `with`, `if`, or `toYaml`.

Do not debug with an unrelated values file. Reproduce the failure using the same chart version and configuration as the failed pipeline.

### Scenario 2 — Deployment exists but Pods do not start

Commands:

```
kubectl get deployment order-service -n ecommerce-prod
kubectl get pods -n ecommerce-prod -o wide
kubectl describe deployment order-service -n ecommerce-prod
kubectl get events -n ecommerce-prod --sort-by=.metadata.creationTimestamp
```

Possible causes include:

- Image pull failures.
- Invalid command or arguments.
- Missing environment variables.
- Missing Secrets or ConfigMaps.
- Filesystem permissions.
- Security-context incompatibility.
- Resource requests that cannot fit on available nodes.
- Scheduling constraints that cannot be satisfied.

The `kubectl describe pod` output and namespace events usually provide the next useful clue.

### Scenario 3 — `ImagePullBackOff`

Check:

```
kubectl describe pod <pod-name> -n ecommerce-prod
```

Look for errors such as:

- Image not found.
- Unauthorized registry access.
- Incorrect image digest.
- Incorrect repository path.
- Missing registry credentials.
- Node or workload identity configuration problems.

Verify the exact image reference in the rendered manifest and confirm that the expected artifact exists in ECR.

Do not solve registry authentication problems by granting every workload broad AWS permissions.

### Scenario 4 — Pod is running but readiness fails

Check:

```
kubectl describe pod <pod-name> -n ecommerce-prod
kubectl logs <pod-name> -n ecommerce-prod
kubectl logs <pod-name> -n ecommerce-prod --previous
```

Investigate:

- Incorrect readiness path.
- Wrong application port.
- Slow application initialization.
- Failed database initialization.
- Missing runtime configuration.
- Dependency failures.
- Incorrect readiness semantics.

A Running Pod is not necessarily a Ready Pod. The Service normally routes traffic only to eligible ready endpoints.

### Scenario 5 — Service has no endpoints

Check:

```
kubectl get svc order-service -n ecommerce-prod -o yaml
kubectl get pods -n ecommerce-prod --show-labels
kubectl get endpointslices -n ecommerce-prod
```

Compare the Service selector with the actual Pod labels.

If selectors match, check readiness and whether the Pods are in the correct namespace.

Do not assume that recreating the Service will fix a selector mismatch caused by the Helm templates.

### Scenario 6 — HPA does not scale

Check:

```
kubectl describe hpa order-service -n ecommerce-prod
kubectl top pods -n ecommerce-prod
kubectl get deployment order-service -n ecommerce-prod
```

Investigate metrics availability, CPU requests, scaling targets, replica limits, and pending Pods.

If the workload is queue-driven, CPU may be the wrong scaling signal. Inspect queue depth or consumer lag and evaluate a suitable event-driven scaling design.

### Scenario 7 — Upgrade is blocked by a PDB

Check:

```
kubectl get pdb -n ecommerce-prod
kubectl describe pdb order-service -n ecommerce-prod
kubectl get pods -n ecommerce-prod -o wide
```

Determine whether the PDB is protecting the intended workload and whether enough replicas are available.

Do not disable a PDB immediately as a generic workaround. Understand whether the disruption is necessary, whether the workload has sufficient redundancy, and what risk the change introduces.

### Scenario 8 — Pods fail after enabling the read-only filesystem

Check application logs and the container's exit reason.

Typical causes include an application trying to write to:

- `/tmp`.
- An application log directory.
- A cache directory.
- A runtime-generated file path.
- A certificate or temporary working directory.

Mount writable storage only at the required locations, using the smallest suitable scope. Avoid changing the root filesystem back to writable without understanding why the application needs write access.

### Scenario 9 — Deployment rollout times out

Check:

```
kubectl rollout status deployment/order-service \
  -n ecommerce-prod \
  --timeout=5m

kubectl describe deployment order-service -n ecommerce-prod

kubectl get rs -n ecommerce-prod
kubectl get pods -n ecommerce-prod -o wide
```

Investigate readiness failures, image pull errors, insufficient cluster capacity, quota limits, restrictive scheduling rules, and application startup failures.

A timeout is a symptom, not a root cause.

### Scenario 10 — New chart version breaks several services

First establish whether the change affected a shared default, template, helper, schema, or resource name.

Then:

1. Identify the exact chart versions used by affected services.
2. Compare the previous and new rendered manifests.
3. Reproduce the failure in a test namespace.
4. Determine whether the issue is in the chart or an application-specific override.
5. Roll back or pin the chart version if that is the safest immediate mitigation.
6. Add a regression test.
7. Publish a corrected chart version.
8. Document the compatibility issue.

Do not independently patch 21 application manifests to compensate for a shared chart regression unless a temporary service-specific exception is explicitly required.

## 31. Rollback and recovery

A production rollback must restore a known-good application deployment while preserving the intended configuration and data safety.

### 31.1 Helm-managed release rollback

For a release managed directly by Helm:

```
helm history order-service -n ecommerce-prod
helm rollback order-service <REVISION> -n ecommerce-prod
```

Use the actual revision from the release history. Verify the resulting Deployment, Pods, and application health.

### 31.2 GitOps-managed rollback

For the Argo CD model used in this project, the preferred rollback path is normally a reviewed Git revert or a new GitOps commit that restores the last known-good chart and image references.

Argo CD then reconciles the restored desired state.

Manual cluster changes may be overwritten by Argo CD, depending on its sync configuration. Use the repository as the source of truth and document any emergency exception.

### 31.3 Verify the rollback

```
kubectl rollout status deployment/order-service -n ecommerce-prod
kubectl get pods -n ecommerce-prod -o wide
kubectl get svc order-service -n ecommerce-prod
```

Then verify:

- Application health.
- Business-level smoke tests.
- Error rate and latency.
- Dependency connectivity.
- Queue processing, if relevant.
- Database compatibility.
- Expected image digest.
- Argo CD sync and health status.

A Kubernetes rollout can succeed while the application is functionally broken. Include application-level verification in the release procedure.

### 31.4 Rollback limitations

A chart rollback does not necessarily reverse:

- Database migrations.
- Messages already published.
- External payment or shipping operations.
- Schema changes in external systems.
- Persistent volume data modifications.
- IAM or external infrastructure changes made outside the release.
- Side effects already performed by the application.

For a stateful change, define the recovery strategy before deployment. Application rollback compatibility should be considered when designing database migrations and event schemas.

## 32. Operational ownership and maintenance

The shared chart becomes a platform product. It needs a defined support and maintenance process.

### 32.1 Platform engineering responsibilities

The platform team should:

- Maintain chart templates and the supported values schema.
- Publish chart versions.
- Maintain validation workflows.
- Define security defaults.
- Maintain documentation and examples.
- Track supported Kubernetes and Helm versions.
- Review breaking changes.
- Maintain upgrade and rollback guidance.
- Help application teams diagnose chart-level issues.

### 32.2 Application team responsibilities

Each application team should:

- Maintain service-specific values.
- Ensure health endpoints behave correctly.
- Set realistic resource requests and limits.
- Validate the image against the chart's security defaults.
- Test service-specific rollout behavior.
- Validate dependencies and business smoke tests.
- Report chart compatibility issues with reproducible evidence.

### 32.3 Change management

A shared chart change should include:

- Problem statement.
- Proposed behavior.
- Affected templates and values.
- Backward-compatibility assessment.
- Security impact.
- Tests and rendered-manifest comparisons.
- Upgrade or migration guidance.
- Release notes.
- Rollback plan.

A chart that is used by 21 services needs stronger change control than an experimental chart used by one developer.

## 33. Practical implementation exercise

Complete the following exercise in the chart repository.

### Task A — Build the chart

- Create `Chart.yaml`.
- Create `values.yaml`.
- Create `values.schema.json`.
- Create `_helpers.tpl`.
- Create Deployment and Service templates.
- Create ServiceAccount, HPA, and PDB templates.
- Add optional ConfigMap support.
- Add chart documentation.

### Task B — Configure the Order Service

- Set the actual image repository.
- Set the actual application port.
- Configure startup, liveness, and readiness probes.
- Set resource requests and limits.
- Configure a dedicated ServiceAccount.
- Configure the production image digest.
- Enable autoscaling only after confirming that the metric and resource settings are appropriate.
- Configure a PDB only after confirming replica availability and maintenance requirements.

### Task C — Validate the chart

- Run `helm lint`.
- Render development values.
- Render production values.
- Run release-value validation.
- Validate generated YAML against the target Kubernetes schema.
- Run policy checks.
- Run chart unit tests.
- Compare the rendered manifests against the expected output.
- Test an upgrade in an isolated namespace.

### Task D — Publish and consume

- Package the chart.
- Publish a new immutable chart version to ECR.
- Verify registry access.
- Update the GitOps declaration.
- Verify Argo CD can retrieve the chart.
- Promote the image digest through the intended environments.
- Verify rollout and application health.
- Record the chart version and release evidence.

## 34. Chapter 182 completion criteria

Chapter 182 is complete when the platform team can demonstrate all of the following:

1. The reusable chart renders a valid Deployment and Service.
2. The chart produces stable resource names and selectors.
3. ServiceAccount behavior is explicit.
4. Probe, security-context, resource, and scheduling settings are configurable.
5. HPA and PDB are optional and correctly validated.
6. Production image references require a real immutable digest.
7. CI validates templates, values, and generated manifests.
8. Chart upgrades are tested against an existing release.
9. The chart can be packaged and published to the approved registry.
10. GitOps can reference the chart and an approved image digest.
11. Rollback behavior is documented and tested.
12. Application-specific requirements are not hidden behind platform defaults.

## 35. Final platform engineering principles

Remember these distinctions:

- Helm templates standardize resource generation.
- Values files provide service- and environment-specific configuration.
- The schema and validation scripts define which configurations are supported.
- CI validates the chart and produces approved artifacts.
- ECR stores container images and can store OCI chart artifacts.
- The GitOps repository declares which artifacts and configuration belong in each environment.
- Argo CD reconciles the declared state.
- Kubernetes schedules and runs the workloads.
- Application teams remain responsible for application correctness.
- Platform engineering owns the reusable deployment contract and its operational lifecycle.

The objective is not to make every microservice identical. The objective is to make every microservice follow a consistent, testable, secure, and maintainable deployment process while preserving the differences that are genuinely required by each application.

Next chapter — Chapter 183: Reusable GitHub Actions.

That chapter will establish reusable pipeline components for the same 21 microservices, including checkout, build, test, security scanning, Docker image creation, ECR publishing, image-digest output, and the boundary between CI artifact production and GitOps deployment.