# 28-Helm.md

````markdown
# 28 — Helm

## 1. Overview

Helm is the package manager and templating system commonly used to manage Kubernetes application manifests.

In our enterprise e-commerce platform, Helm sits between the application deployment definition and Kubernetes.

The overall flow is:

Developer
    |
    v
Application Repository
    |
    v
CI
    |
    v
ECR
    |
    v
GitOps Repository
    |
    v
Helm Chart + Environment Values
    |
    v
ArgoCD
    |
    v
Kubernetes API
    |
    v
EKS


Helm answers:

> How do we package a Kubernetes application so that the same deployment definition can be reused across environments?

---

# 2. Why Helm?

Without Helm, a microservice might need separate Kubernetes manifests:

```text
deployment-dev.yaml
deployment-qa.yaml
deployment-staging.yaml
deployment-prod.yaml
````

For 21 microservices, this quickly becomes difficult to maintain.

Example:

```text
21 services
x
4 environments
x
multiple Kubernetes resources
```

This can create hundreds of YAML files.

Helm allows us to define reusable templates.

For example:

```text
One Deployment template
        |
        +--> DEV values
        +--> QA values
        +--> STAGING values
        +--> PROD values
```

The same template can produce different manifests.

---

# 3. Helm in Our Architecture

Our architecture is:

```text
Application Repository
        |
        v
       CI
        |
        v
       ECR
        |
        v
   GitOps Repository
        |
        +--> Helm Chart
        |
        +--> Environment Values
        |
        v
      ArgoCD
        |
        v
       EKS
```

Helm is not responsible for building the Docker image.

Helm is not responsible for creating the AWS VPC.

Helm is responsible for rendering and packaging Kubernetes application configuration.

---

# 4. Helm vs Kubernetes

Kubernetes understands:

```yaml
apiVersion: apps/v1
kind: Deployment
```

Helm adds templating.

Instead of hardcoding:

```yaml
replicas: 3
```

we can use:

```yaml
replicas: {{ .Values.replicaCount }}
```

Then:

```yaml
replicaCount: 3
```

can come from `values.yaml`.

---

# 5. Basic Helm Structure

A Helm chart typically looks like:

```text
order-service/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── ingress.yaml
│   ├── networkpolicy.yaml
│   └── _helpers.tpl
└── .helmignore
```

---

# 6. Chart.yaml

`Chart.yaml` contains chart metadata.

Example:

```yaml
apiVersion: v2

name: order-service

description: Helm chart for Order Service

type: application

version: 1.0.0

appVersion: "1.6.0"
```

Important distinction:

```text
version
```

is the Helm chart version.

```text
appVersion
```

represents the application version.

They are not necessarily the same.

---

# 7. Chart Version

Suppose we change only the Helm template.

Example:

```text
Chart version:
1.0.0 -> 1.0.1
```

The application itself might still be:

```text
order-service 1.6.0
```

Therefore:

```text
Chart:
1.0.1

Application:
1.6.0
```

This distinction is important for release management.

---

# 8. values.yaml

`values.yaml` contains default configuration.

Example:

```yaml
replicaCount: 2

image:
  repository: ""
  tag: ""
  digest: ""

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi

  limits:
    cpu: 1
    memory: 1Gi
```

The chart templates consume these values.

---

# 9. Environment Values

We can override defaults.

Example:

```text
values.yaml
values-dev.yaml
values-qa.yaml
values-staging.yaml
values-prod.yaml
```

Production:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: 2
    memory: 2Gi
```

Development:

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi

  limits:
    cpu: 500m
    memory: 512Mi
```

Same template.

Different values.

---

# 10. Helm Values Precedence

When multiple values sources are supplied, later/higher-precedence values override earlier values.

Conceptually:

```text
Chart defaults
      |
      v
values.yaml
      |
      v
environment values
      |
      v
explicit overrides
```

For example:

```bash
helm template order-service ./order-service \
  -f values.yaml \
  -f values-prod.yaml
```

If both files contain:

```yaml
replicaCount:
```

the later value wins.

This is useful for environment-specific configuration.

---

# 11. Deployment Template

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}

  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}

    spec:
      containers:
        - name: order-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:
            - containerPort: {{ .Values.service.port }}
```

Helm replaces the template expressions with values.

---

# 12. Understanding `.Values`

When we write:

```text
.Values.replicaCount
```

Helm looks inside:

```yaml
replicaCount: 3
```

Therefore:

```yaml
replicas: {{ .Values.replicaCount }}
```

becomes:

```yaml
replicas: 3
```

---

# 13. Understanding `.`

The dot:

```text
.
```

represents the current Helm context.

Common objects include:

```text
.Values
.Release
.Chart
.Capabilities
.Files
```

Examples:

```text
.Values.image.repository
.Chart.Name
.Release.Name
.Release.Namespace
```

---

# 14. Release Information

Helm exposes release metadata.

Example:

```yaml
metadata:
  labels:
    release: {{ .Release.Name }}
    namespace: {{ .Release.Namespace }}
```

If the release is:

```text
order-service-prod
```

the rendered result contains that value.

---

# 15. `_helpers.tpl`

The `_helpers.tpl` file contains reusable named templates.

Example:

```gotemplate
{{- define "order-service.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}
```

Another:

```gotemplate
{{- define "order-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "order-service.name" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

This avoids repeating naming logic throughout templates.

---

# 16. Labels

A production chart should use consistent Kubernetes labels.

Example:

```gotemplate
labels:
  {{- include "order-service.labels" . | nindent 4 }}
```

Helper:

```gotemplate
{{- define "order-service.labels" -}}
helm.sh/chart: {{ include "order-service.chart" . }}
{{ include "order-service.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

Additional organizational labels can be included where appropriate.

---

# 17. Selector Labels

Selectors are extremely important.

Example:

```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: order-service
```

Pod:

```yaml
labels:
  app.kubernetes.io/name: order-service
```

These must match.

Otherwise:

```text
Deployment
     |
     X
Pods not selected
```

or:

```text
Service
     |
     X
Pods not selected
```

can occur.

---

# 18. Service Template

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  type: {{ .Values.service.type }}

  selector:
    {{- include "order-service.selectorLabels" . | nindent 4 }}

  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
```

Values:

```yaml
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
```

---

# 19. Why ClusterIP?

Internal microservices normally use:

```text
ClusterIP
```

Example:

```text
order-service.production.svc.cluster.local
```

External traffic should normally reach:

```text
Internet
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
Ingress
   |
   v
Service
   |
   v
Pods
```

Do not expose every microservice using:

```text
LoadBalancer
```

unless there is a specific requirement.

---

# 20. Container Image

A simple chart may use:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "1.6.0"
```

Template:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

But for production, digest pinning is preferable.

---

# 21. Digest-Based Image

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

Template:

```gotemplate
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

Rendered result:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:abc123...
```

This gives stronger artifact immutability.

---

# 22. Avoid `latest`

Do not use:

```yaml
image:
  tag: latest
```

Why?

Because:

```text
latest
```

does not uniquely identify an immutable artifact.

Today:

```text
latest -> image A
```

Tomorrow:

```text
latest -> image B
```

Production behavior becomes difficult to reproduce.

Use:

```text
Git SHA
version
digest
```

Preferably:

```text
image digest
```

for controlled production deployments.

---

# 23. Environment Variables

Application configuration can be templated.

Example:

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: {{ .Values.environment | quote }}
```

Values:

```yaml
environment: production
```

Result:

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "production"
```

---

# 24. ConfigMap

Example:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: {{ include "order-service.fullname" . }}-config

data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
```

Values:

```yaml
config:
  logLevel: INFO
```

Do not put passwords or credentials in ConfigMaps.

---

# 25. Secrets

Do not do this:

```yaml
data:
  DB_PASSWORD: "production-password"
```

Instead reference a Kubernetes Secret created through the approved secrets workflow.

Example:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: DB_PASSWORD
```

Or use an ExternalSecret:

```text
Helm
  |
  v
ExternalSecret
  |
  v
AWS Secrets Manager
```

Secrets management is covered in:

```text
26-Secrets-Management.md
```

---

# 26. Resource Requests and Limits

Helm should make resources configurable.

Example:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: 2
    memory: 2Gi
```

Template:

```gotemplate
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

This allows different environments to have different resource profiles.

---

# 27. Why Requests Matter

Kubernetes uses requests for scheduling.

If:

```text
CPU request = 500m
Memory request = 512Mi
```

the scheduler looks for a node with sufficient allocatable capacity.

Therefore incorrect requests can cause:

```text
Pending Pods
```

even when the application itself is healthy.

---

# 28. Why Limits Matter

Limits restrict container resource consumption.

Example:

```yaml
limits:
  cpu: 2
  memory: 2Gi
```

Memory limit breaches can result in:

```text
OOMKilled
```

Therefore limits must be based on observed application behavior, not arbitrary numbers.

---

# 29. Production Probes

Helm should support:

```text
startupProbe
readinessProbe
livenessProbe
```

Example:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: http
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: http
  periodSeconds: 20
  timeoutSeconds: 2
  failureThreshold: 3
```

For Spring Boot, these endpoints should be configured appropriately.

---

# 30. Startup vs Readiness vs Liveness

### Startup

Answers:

> Has the application finished starting?

Useful for slow-starting applications.

### Readiness

Answers:

> Can this Pod receive traffic?

If readiness fails:

```text
Pod remains running
but is removed from Service endpoints
```

### Liveness

Answers:

> Is the application still functioning sufficiently to remain alive?

If liveness repeatedly fails:

```text
Kubelet restarts the container
```

Misconfigured liveness probes can cause restart loops.

---

# 31. HPA Template

Example:

```yaml
{{- if .Values.autoscaling.enabled }}

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "order-service.fullname" . }}

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

Values:

```yaml
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

---

# 32. PDB Template

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}

  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
```

Production:

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

PDB helps protect availability during voluntary disruptions.

---

# 33. Ingress Template

An application can optionally expose HTTP through an Ingress.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  ingressClassName: alb

  rules:
    - host: orders.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "order-service.fullname" . }}
                port:
                  number: 8080
```

In our AWS architecture, the AWS Load Balancer Controller manages the ALB integration.

Not every internal microservice needs an Ingress.

---

# 34. ServiceAccount

Example:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: {{ include "order-service.fullname" . }}
```

The ServiceAccount can be associated with the application's AWS workload identity mechanism.

For our AWS architecture:

```text
Pod
 |
 v
EKS workload identity
 |
 v
IAM role
 |
 +--> SQS
 +--> Secrets Manager
 +--> S3
```

No AWS access keys should be embedded in Helm values.

---

# 35. NetworkPolicy

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: order-service

  policyTypes:
    - Ingress
    - Egress
```

A production NetworkPolicy should explicitly allow only required communication.

For example:

```text
order-service
    |
    +--> PostgreSQL
    +--> Redis
    +--> SQS
    +--> notification-service
```

The exact implementation depends on the network architecture and CNI capabilities.

---

# 36. Conditional Templates

Helm supports conditionals.

Example:

```gotemplate
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

Values:

```yaml
ingress:
  enabled: false
```

DEV:

```text
Ingress disabled
```

Production:

```yaml
ingress:
  enabled: true
```

This is useful when environments have different requirements.

---

# 37. `with`

Helm supports `with`.

Example:

```gotemplate
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

If:

```yaml
nodeSelector:
  workload: application
```

is provided, it gets rendered.

If it is absent, the block is omitted.

---

# 38. `range`

`range` can iterate over lists.

Example:

```yaml
env:
{{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

Values:

```yaml
extraEnv:
  - name: LOG_LEVEL
    value: INFO

  - name: FEATURE_X
    value: enabled
```

This produces multiple environment variables.

Be careful with complex templating; overly clever Helm charts become difficult to maintain.

---

# 39. `default`

Example:

```gotemplate
{{ .Values.service.port | default 8080 }}
```

If the value is missing:

```text
8080
```

is used.

However, defaults should not hide configuration mistakes for critical production settings.

---

# 40. `required`

For critical values:

```gotemplate
{{ required "image.repository is required" .Values.image.repository }}
```

If missing, Helm fails rather than generating a broken manifest.

This is useful for values that should never be omitted.

---

# 41. `quote`

Use:

```gotemplate
{{ .Values.environment | quote }}
```

Result:

```yaml
value: "production"
```

Quoting can prevent YAML type interpretation problems.

For example:

```yaml
version: "1.0"
```

is safer than relying on YAML to infer the intended type.

---

# 42. `toYaml`

Useful for rendering nested objects.

Example:

```gotemplate
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

Values:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

The whole structure is inserted correctly.

---

# 43. `nindent`

Indentation matters in YAML.

Example:

```gotemplate
{{- toYaml .Values.resources | nindent 12 }}
```

means:

```text
Convert object to YAML
        |
        v
Indent it by 12 spaces
```

Many Helm template errors are indentation errors.

---

# 44. Helm Lint

Before deployment:

```bash
helm lint ./order-service
```

Example result:

```text
==> Linting ./order-service
1 chart(s) linted, 0 chart(s) failed
```

This should be part of GitOps CI.

---

# 45. Helm Template

Render templates without installing.

```bash
helm template order-service ./order-service
```

With production values:

```bash
helm template order-service ./order-service \
  -f values.yaml \
  -f values-prod.yaml
```

This is one of the most useful Helm troubleshooting commands.

---

# 46. Inspect Rendered YAML

Save output:

```bash
helm template order-service ./order-service \
  -f values-prod.yaml \
  > rendered.yaml
```

Then inspect:

```bash
less rendered.yaml
```

Check:

```text
Deployment
Service
HPA
PDB
Ingress
ServiceAccount
ConfigMap
NetworkPolicy
```

before applying anything.

---

# 47. Helm Install

Standalone Helm deployment:

```bash
helm install order-service ./order-service \
  -n production \
  --create-namespace
```

This installs a Helm release.

But in our production GitOps model, ArgoCD is normally responsible for managing the Helm release rather than engineers manually running `helm install` against production.

---

# 48. Helm Upgrade

Standalone:

```bash
helm upgrade order-service ./order-service \
  -n production \
  -f values-prod.yaml
```

Again, in our GitOps architecture:

```text
Git change
   |
   v
ArgoCD
   |
   v
Helm rendering
   |
   v
Kubernetes
```

rather than:

```text
Engineer laptop
   |
   v
helm upgrade
   |
   v
Production
```

---

# 49. Helm Rollback

Traditional Helm supports:

```bash
helm history order-service -n production
```

and:

```bash
helm rollback order-service 3 -n production
```

However, with GitOps, the preferred production rollback is normally:

```text
Git revert
   |
   v
ArgoCD
   |
   v
previous desired state
```

This maintains Git as the source of truth.

---

# 50. Helm Release State

Helm tracks release information in Kubernetes.

ArgoCD can use Helm as a rendering/deployment mechanism.

Important distinction:

```text
Helm
=
package manager/template engine

ArgoCD
=
GitOps reconciliation controller
```

They are complementary.

---

# 51. Helm + ArgoCD

Example ArgoCD source:

```yaml
spec:
  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    path: environments/prod/order-service
    helm:
      valueFiles:
        - values.yaml
```

Conceptually:

```text
GitOps repo
     |
     v
Helm values
     |
     v
Helm chart
     |
     v
Rendered Kubernetes manifests
     |
     v
ArgoCD
     |
     v
EKS
```

The full ArgoCD implementation comes in Chapter 29.

---

# 52. Helm Dependencies

Charts can depend on other charts.

Example:

```yaml
dependencies:
  - name: redis
    version: 20.x.x
    repository: ...
```

But be careful.

For production microservices, blindly embedding infrastructure dependencies inside application charts can create ownership problems.

For example, we generally do not want:

```text
order-service Helm release
    |
    +--> creates PostgreSQL
    +--> creates Redis
    +--> creates Kafka
```

unless that architecture is intentionally designed.

Infrastructure/platform components should have clear ownership.

---

# 53. Application Chart vs Platform Chart

Application chart:

```text
order-service
```

manages:

```text
Deployment
Service
HPA
PDB
ConfigMap
ServiceAccount
NetworkPolicy
```

Platform charts might manage:

```text
Prometheus
Grafana
External Secrets
Fluent Bit
ArgoCD
Ingress controller
```

This separation makes operations easier.

---

# 54. OCI Helm Charts

Helm charts can also be stored in OCI-compatible registries.

Conceptually:

```text
Helm Chart
    |
    v
OCI Registry
```

AWS environments can use Amazon ECR for OCI artifacts where appropriate.

However, for our GitOps model, keeping the chart source in Git provides strong version-control and review visibility.

The exact artifact strategy depends on platform standards.

---

# 55. Helm Chart Versioning

Example:

```text
order-service chart
1.0.0
```

A template-only change:

```text
1.0.1
```

A larger backward-compatible feature:

```text
1.1.0
```

A breaking chart interface change:

```text
2.0.0
```

Follow a consistent chart-versioning policy.

---

# 56. Application Versioning

Application:

```text
1.6.0
```

Chart:

```text
1.2.0
```

Image:

```text
order-service@sha256:abc123
```

These are three separate identities:

```text
Application version
Chart version
Artifact digest
```

Do not confuse them.

---

# 57. GitOps Values Example

Production:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...

replicaCount: 3

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: 2
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

This becomes the desired production configuration.

---

# 58. Helm Validation in GitOps CI

A GitOps PR should run:

```bash
helm lint ./applications/order-service

helm template order-service \
  ./applications/order-service \
  -f environments/prod/order-service/values.yaml
```

Additional checks:

```text
YAML validation
Policy validation
Secret detection
Kubernetes schema validation
```

The goal is:

```text
Bad manifest
    |
    X
Git merge
```

rather than discovering the problem after ArgoCD tries to deploy it.

---

# 59. Helm Schema

A chart can provide:

```text
values.schema.json
```

This can validate values.

Example:

```text
replicaCount must be integer
service.port must be integer
image.repository must be string
```

This helps catch configuration mistakes early.

---

# 60. Example Schema Concept

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object"
    }
  }
}
```

A malformed value can then fail validation before deployment.

---

# 61. Common Helm Mistake: Hardcoded Names

Bad:

```yaml
metadata:
  name: order-service
```

This can create naming conflicts.

Prefer helper-based naming:

```gotemplate
metadata:
  name: {{ include "order-service.fullname" . }}
```

This allows different releases/environments to use predictable names.

---

# 62. Common Helm Mistake: Hardcoded Namespace

Avoid putting:

```yaml
metadata:
  namespace: production
```

inside reusable application templates unless there is a specific reason.

ArgoCD/Kubernetes should determine the target namespace.

This makes the same chart reusable across:

```text
dev
qa
staging
prod
```

---

# 63. Common Helm Mistake: Environment Logic Everywhere

Avoid:

```gotemplate
{{ if eq .Values.environment "prod" }}
...
{{ else if eq .Values.environment "qa" }}
...
{{ else if eq .Values.environment "dev" }}
...
{{ end }}
```

repeated throughout every template.

Prefer:

```text
same template
+
different values
```

This keeps the chart maintainable.

---

# 64. Common Helm Mistake: Too Many Values

Bad chart:

```text
200+ configuration parameters
```

where almost nobody knows what they mean.

Good chart:

```text
well-documented
sensible defaults
important overrides exposed
dangerous settings controlled
```

A Helm chart is an interface.

Its values are configuration APIs.

---

# 65. Common Helm Mistake: Hidden Defaults

Suppose production requires:

```text
database URL
```

Do not silently default it to:

```text
localhost
```

because the deployment may appear successful but fail at runtime.

For critical configuration:

```gotemplate
required
```

is often better.

---

# 66. Common Helm Mistake: Secrets in Values

Never:

```yaml
database:
  password: mypassword
```

Even if:

```text
values-prod.yaml
```

is in a private repository.

Use:

```text
Secrets Manager
External Secrets
Kubernetes Secret references
```

instead.

---

# 67. Common Helm Mistake: Using `lookup` Excessively

Helm has a `lookup` function that can inspect Kubernetes resources.

Although powerful, excessive use can make rendering dependent on live cluster state.

GitOps benefits from deterministic rendering.

Prefer:

```text
Git values
+
Helm templates
=
predictable manifests
```

rather than hidden dependencies on current cluster state.

---

# 68. Common Helm Mistake: Installing Infrastructure Through App Charts

Avoid coupling:

```text
order-service
```

to:

```text
PostgreSQL
Redis
Kafka
Elasticsearch
```

unless intentional.

For our platform:

```text
RDS
ElastiCache
SQS
SNS
```

are AWS infrastructure.

They are managed separately.

The application chart only consumes them.

---

# 69. Helm and RDS

The application chart might contain:

```yaml
env:
  - name: DB_URL
    valueFrom:
      secretKeyRef:
        name: order-service-db
        key: DB_URL
```

It does not create the RDS database.

Architecture:

```text
Terraform
   |
   v
RDS
```

and:

```text
Helm
   |
   v
Order Service
```

This maintains clean ownership.

---

# 70. Helm and Redis

Similarly:

```text
Terraform
   |
   v
ElastiCache
```

Application:

```text
Helm
   |
   v
REDIS_HOST
```

The Helm chart does not need to create Redis for every application.

---

# 71. Helm and SQS

Infrastructure:

```text
Terraform
   |
   v
SQS Queue
```

Application:

```text
Helm
   |
   v
AWS IAM workload identity
   |
   v
SQS
```

The queue name/ARN can be configuration.

The secret credentials should not be stored in Helm.

---

# 72. Helm and Observability

The chart can expose:

```yaml
metrics:
  enabled: true
```

and potentially configure:

```text
Prometheus annotations
ServiceMonitor
OpenTelemetry environment variables
```

depending on the observability architecture.

Example:

```yaml
metrics:
  enabled: true
  port: 8080
  path: /actuator/prometheus
```

For our Spring Boot services:

```text
/actuator/prometheus
```

can expose application metrics.

---

# 73. Helm and OpenTelemetry

Later we can configure:

```yaml
otel:
  enabled: true
  endpoint: http://otel-collector.monitoring:4317
```

The application receives:

```text
OTEL_EXPORTER_OTLP_ENDPOINT
```

through Kubernetes configuration.

Again:

```text
Helm
=
configuration delivery

OpenTelemetry
=
telemetry system
```

---

# 74. Production Helm Chart Quality

A production-ready chart should provide:

```text
[ ] Consistent naming
[ ] Standard labels
[ ] Resource requests
[ ] Resource limits
[ ] Startup probe
[ ] Readiness probe
[ ] Liveness probe
[ ] HPA
[ ] PDB
[ ] ServiceAccount
[ ] SecurityContext
[ ] NetworkPolicy where appropriate
[ ] ConfigMap support
[ ] Secret references
[ ] Image digest support
[ ] Environment overrides
[ ] Validation
[ ] Helm lint
[ ] Deterministic rendering
```

---

# 75. SecurityContext

Example:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

Container-level:

```yaml
securityContext:
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

These settings should be tested against the application because some applications legitimately require writable filesystem locations.

Do not blindly enable settings without validating the workload.

---

# 76. Pod Security and Helm

Helm should make secure defaults easy.

Example:

```yaml
securityContext:
  runAsNonRoot: true

containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

Then the platform can enforce stronger policy through:

```text
Kyverno
OPA Gatekeeper
Pod Security Admission
```

where appropriate.

---

# 77. Node Scheduling

Some workloads may require dedicated nodes.

Values:

```yaml
nodeSelector:
  workload: application
```

Or:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: application
    effect: NoSchedule
```

For production, this can be used to separate:

```text
system workloads
application workloads
stateful workloads
observability workloads
```

according to capacity and operational requirements.

---

# 78. Topology Spread

A production chart can support:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: order-service
```

This helps distribute replicas across failure domains.

For a 3-AZ EKS cluster:

```text
AZ-A -> Pod
AZ-B -> Pod
AZ-C -> Pod
```

is generally preferable to:

```text
AZ-A -> Pod
AZ-A -> Pod
AZ-A -> Pod
```

for critical workloads, subject to capacity and scheduling constraints.

---

# 79. Anti-Affinity

Another option:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: order-service
```

This encourages replicas onto different nodes.

Use hard constraints only when the availability requirement justifies the scheduling risk.

---

# 80. Helm Debugging

When something is wrong:

```bash
helm lint ./chart
```

Then:

```bash
helm template release ./chart -f values-prod.yaml
```

Then:

```bash
helm template release ./chart \
  -f values-prod.yaml \
  --debug
```

Inspect the rendered YAML.

Then validate Kubernetes objects.

Only after rendering is correct should you investigate:

```text
ArgoCD
Kubernetes
Pods
Application
```

---

# 81. ArgoCD Shows ComparisonError

If ArgoCD reports a Helm rendering error:

```text
ComparisonError
```

do not immediately investigate Pods.

First reproduce:

```bash
helm template order-service ./chart \
  -f values-prod.yaml
```

Common causes:

```text
Missing values
Invalid YAML
Template syntax error
Wrong indentation
Invalid Kubernetes API version
Wrong value type
```

Fix the GitOps PR first.

---

# 82. ArgoCD Syncs but Pod Fails

This means Helm successfully generated Kubernetes resources.

Example:

```text
Helm rendering:
PASS

ArgoCD sync:
PASS

Pod:
CrashLoopBackOff
```

Now Helm is probably not the primary problem.

Investigate:

```bash
kubectl logs
kubectl describe pod
kubectl get events
```

Possible application issues:

```text
Database connection
Redis connection
Missing secret
Bad configuration
Java startup failure
Dependency unavailable
```

---

# 83. Helm and Immutable Artifacts

Our final release chain is:

```text
Git commit
   |
   v
CI
   |
   v
Docker build
   |
   v
ECR
   |
   v
Image digest
   |
   v
GitOps values
   |
   v
Helm
   |
   v
ArgoCD
   |
   v
EKS
```

The Helm chart does not rebuild the application.

It only references the already-built artifact.

---

# 84. Full Order-Service Example

Application CI produces:

```text
order-service
commit = 8a73b91
```

ECR:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

Digest:

```text
sha256:abc123...
```

GitOps:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: sha256:abc123...
```

Helm renders:

```yaml
containers:
  - name: order-service
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:abc123...
```

ArgoCD applies the desired state.

Kubernetes starts the Pod.

---

# 85. Production Deployment Chain

```text
                    SOURCE
                      |
                      v
              Application Git
                      |
                      v
                     CI
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
     Test          SonarQube       Security
       |                             |
       +--------------+--------------+
                      |
                      v
                Docker Build
                      |
                      v
                     ECR
                      |
                      v
                Image Digest
                      |
                      v
                GitOps PR
                      |
                      v
                Helm Values
                      |
                      v
                   ArgoCD
                      |
                      v
                     EKS
                      |
                      v
                  Kubernetes
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
      RDS           Redis           SQS
```

This is the deployment architecture we are building throughout the project.

---

# 86. Helm Repository Strategy for Our Project

For this project, keep application deployment definitions logically separated from application source.

A practical structure is:

```text
gitops/
├── applications/
│   ├── order-service/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │
│   ├── payment-service/
│   └── catalog-service/
│
└── environments/
    ├── dev/
    │   ├── order-service/
    │   │   └── values.yaml
    │   ├── payment-service/
    │   │   └── values.yaml
    │   └── catalog-service/
    │       └── values.yaml
    │
    ├── qa/
    ├── staging/
    └── prod/
```

Another valid enterprise model is to maintain reusable charts in a dedicated platform/chart repository and have GitOps consume versioned chart artifacts.

The correct choice depends on team ownership and release-management requirements.

---

# 87. Do Not Duplicate Complete Charts Per Environment

Avoid:

```text
prod/order-service/deployment.yaml
qa/order-service/deployment.yaml
dev/order-service/deployment.yaml
```

with nearly identical content.

Instead:

```text
One chart
+
Environment values
```

Example:

```text
Chart
 |
 +--> dev values
 +--> qa values
 +--> staging values
 +--> prod values
```

This reduces configuration duplication.

---

# 88. When Environment-Specific Templates Are Justified

Sometimes environments genuinely differ.

For example:

```text
DEV:
Ingress disabled

PROD:
ALB enabled
```

This can be controlled through values.

But if production requires an entirely different architecture, separate templates or charts may be justified.

Do not force everything into one gigantic conditional template.

---

# 89. Chart Testing

A mature Helm workflow should test:

```text
Helm syntax
Rendered YAML
Kubernetes API compatibility
Policy compliance
Expected resource configuration
```

For critical charts, automated tests can verify:

```text
Deployment exists
Service exists
Probes exist
Resources exist
SecurityContext exists
HPA exists
```

The goal is to catch deployment defects before production.

---

# 90. Helm Upgrade Compatibility

A chart update can break an application even if the application image is unchanged.

Example:

```text
Chart 1.3.0
   |
   v
Chart 1.4.0
```

Possible changes:

```text
Service port
labels
selectors
Ingress
security context
environment variables
volume mounts
API versions
```

Therefore chart changes need the same review discipline as application changes.

---

# 91. Helm and Kubernetes API Versions

Always use supported API versions.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
```

Avoid obsolete APIs.

Before upgrading EKS/Kubernetes, inspect:

```text
deprecated APIs
removed APIs
CRD compatibility
Ingress API
HPA API
PodDisruptionBudget API
```

A Helm chart that works on one Kubernetes version may fail after an upgrade if it uses removed APIs.

---

# 92. Helm and EKS Upgrades

EKS upgrade process:

```text
EKS version
   |
   v
Kubernetes API compatibility
   |
   v
Helm charts
   |
   v
CRDs/controllers
   |
   v
Application workloads
```

Before upgrading production:

```text
[ ] Test charts against target Kubernetes version
[ ] Validate API versions
[ ] Validate CRDs
[ ] Validate ingress controller
[ ] Validate autoscaling
[ ] Validate security policies
[ ] Validate observability
```

Helm becomes part of the upgrade validation process.

---

# 93. Helm Production Checklist

```text
[ ] Chart.yaml valid
[ ] Chart version controlled
[ ] Application version tracked
[ ] Image digest supported
[ ] No latest tag
[ ] No secrets in values
[ ] Resource requests defined
[ ] Resource limits defined
[ ] Startup probe defined where needed
[ ] Readiness probe defined
[ ] Liveness probe defined
[ ] HPA configured where appropriate
[ ] PDB configured where appropriate
[ ] ServiceAccount configured
[ ] SecurityContext configured
[ ] NetworkPolicy reviewed
[ ] Topology rules reviewed
[ ] Helm lint passes
[ ] Helm template passes
[ ] Policy validation passes
[ ] Kubernetes API compatibility verified
[ ] ArgoCD integration tested
```

---

# 94. Final Mental Model

The most important thing to understand is:

```text
Helm does NOT replace Kubernetes.

Helm packages/templates Kubernetes resources.
```

And:

```text
ArgoCD does NOT replace Helm.

ArgoCD can use Helm to render and deploy desired state.
```

And:

```text
GitOps does NOT replace CI.

CI builds and validates the artifact.
GitOps promotes and deploys the artifact.
```

And:

```text
Helm does NOT replace Terraform.

Terraform manages AWS infrastructure.
Helm manages Kubernetes application resources.
```

Therefore:

```text
Terraform
    |
    v
AWS Infrastructure
    |
    v
EKS

CI
    |
    v
ECR

GitOps
    |
    v
Helm
    |
    v
ArgoCD
    |
    v
Kubernetes
```

---

# 95. Complete Project Mental Model

For our Enterprise E-Commerce Platform:

```text
                        GitHub
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
 Application Repo   Infrastructure Repo   Platform Repo
        |                  |                  |
        v                  v                  v
       CI              Terraform          Platform Git
        |                  |                  |
        v                  v                  |
       ECR              AWS                  |
        |                                     |
        +------------------+------------------+
                           |
                           v
                     GitOps Repository
                           |
                           v
                         Helm
                           |
                           v
                        ArgoCD
                           |
                           v
                         EKS
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   21 Microservices      Platform       Observability
          |                |                |
          v                v                v
     RDS/Redis          Ingress         Prometheus
     SQS/SNS            Secrets         Grafana
                                        ELK
                                        OTel
                                        Jaeger
```

The key production rule remains:

> Build once, scan once, publish one immutable artifact, and promote that exact artifact through environments using GitOps.

---

# 96. Next Chapter

The next chapter is:

```text
29-ArgoCD.md
```

It will move from Helm packaging into the actual GitOps deployment controller.

We will cover:

```text
ArgoCD architecture
ArgoCD components
API server
Repo server
Application controller
Redis
Dex/SSO
ArgoCD installation on EKS
Helm installation
Terraform + ArgoCD bootstrap
Projects
Applications
ApplicationSets
Repositories
Clusters
RBAC
SSO
GitHub integration
GitLab integration
Sync
Auto-sync
Self-heal
Prune
Sync waves
Hooks
Health checks
Drift detection
Multi-cluster
Multi-account
DEV/QA/STAGING/PROD
Production approvals
Rollback
ArgoCD CLI
ArgoCD UI
Troubleshooting
Security hardening
High availability
Backup/DR
Production architecture
```

The key architecture we will implement is:

```text
GitOps Repository
       |
       v
     ArgoCD
       |
       +------------+
       |            |
       v            v
   DEV EKS       PROD EKS
       |            |
       v            v
    Helm         Helm
       |            |
       v            v
 Kubernetes    Kubernetes
```

That is where the GitOps workflow becomes an actual production deployment system.

```
```
