# 63-Helm-Implementation.md

# Helm Implementation — Production-Grade Kubernetes Packaging

We now have:

```text
GitHub
   ↓
GitHub Actions
   ↓
Build / Test / Security
   ↓
Docker
   ↓
Trivy
   ↓
ECR
```

The next step is to define **how `order-service` runs inside Kubernetes**.

We will package the application using Helm.

The target architecture becomes:

```text
GitHub
   ↓
GitHub Actions
   ↓
ECR
   ↓
GitOps Repository
   ↓
ArgoCD
   ↓
Helm
   ↓
EKS
   ↓
order-service
```

---

# 1. Why Helm?

Without Helm, we could maintain many raw Kubernetes YAML files:

```text
deployment.yaml
service.yaml
configmap.yaml
serviceaccount.yaml
hpa.yaml
pdb.yaml
ingress.yaml
```

For 21 microservices and multiple environments, this becomes difficult to maintain.

Helm allows us to create a reusable package:

```text
Helm Chart
    |
    +--> DEV values
    +--> QA values
    +--> STAGING values
    +--> PROD values
```

The Kubernetes manifests remain consistent while environment-specific configuration changes through values.

---

# 2. Helm's Role in Our Architecture

Helm is responsible for **Kubernetes packaging and templating**.

It is not our GitOps controller.

The responsibilities are:

| Component  | Responsibility                      |
| ---------- | ----------------------------------- |
| Docker     | Build container                     |
| ECR        | Store image                         |
| Helm       | Package/render Kubernetes manifests |
| GitOps     | Store desired deployment state      |
| ArgoCD     | Reconcile Git state to EKS          |
| Kubernetes | Run workload                        |

Therefore:

```text
Docker
  ↓
ECR
  ↓
Helm
  ↓
GitOps
  ↓
ArgoCD
  ↓
EKS
```

---

# 3. Helm Chart Structure

Inside `order-service`:

```text
order-service/
│
├── src/
├── pom.xml
├── Dockerfile
│
└── helm/
    └── order-service/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-dev.yaml
        ├── values-qa.yaml
        ├── values-staging.yaml
        ├── values-prod.yaml
        │
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── serviceaccount.yaml
            ├── configmap.yaml
            ├── ingress.yaml
            ├── hpa.yaml
            └── pdb.yaml
```

---

# 4. Create the Chart

From the repository root:

```bash
mkdir -p helm
cd helm
```

Create:

```bash
helm create order-service
```

Helm generates many example files.

For our production implementation, remove the unnecessary examples and keep the structure we actually use.

---

# 5. Final Chart Structure

```text
helm/order-service/
│
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-qa.yaml
├── values-staging.yaml
├── values-prod.yaml
│
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── configmap.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    └── pdb.yaml
```

---

# 6. `Chart.yaml`

```yaml
apiVersion: v2

name: order-service

description: Helm chart for the Order Service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 7. Helm Chart Version vs Application Version

These are different.

### Chart version

```text
0.1.0
```

means:

```text
Helm package version
```

### Application version

```text
1.0.0
```

means:

```text
Order Service application version
```

Our actual Docker artifact is identified by:

```text
Git SHA
```

for example:

```text
a91c2f7d1234
```

So:

```text
Chart version
    ↓
Helm packaging

Git SHA
    ↓
Application artifact
```

---

# 8. Base `values.yaml`

Create:

```text
helm/order-service/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

containerPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

serviceAccount:
  create: true
  name: order-service

config:
  springProfile: prod

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10

  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

ingress:
  enabled: false
  className: alb
  host: order.example.com
```

This is our baseline.

---

# 9. Why `ClusterIP`?

The application service is:

```yaml
service:
  type: ClusterIP
```

This means:

```text
Internet
   X
   |
   X
Order Service
```

The service isn't directly exposed to the Internet.

Instead:

```text
Internet
   |
   v
ALB
   |
   v
Ingress
   |
   v
ClusterIP Service
   |
   v
Order Pods
```

---

# 10. Deployment

Create:

```text
templates/deployment.yaml
```

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: {{ include "order-service.fullname" . }}

  labels:
    app.kubernetes.io/name: {{ include "order-service.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "order-service.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}

  template:

    metadata:
      labels:
        app.kubernetes.io/name: {{ include "order-service.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}

    spec:

      serviceAccountName: {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      containers:

        - name: order-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.config.springProfile | quote }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http

            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http

            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /actuator/health
              port: http

            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 30
```

---

# 11. Why Three Probes?

This is important for production Kubernetes.

We use:

```text
startupProbe
readinessProbe
livenessProbe
```

They answer different questions.

---

# 12. Startup Probe

Question:

> Has the application successfully started?

```text
Container starts
      |
      v
Startup Probe
      |
      +---- failure
      |
      +---- success
```

Useful for Java applications that need time to initialize.

Without a startup probe, Kubernetes may consider a slow-starting application unhealthy too quickly.

---

# 13. Readiness Probe

Question:

> Can this pod receive traffic?

```text
Pod
 |
 +--> Ready
       |
       v
Service sends traffic
```

If readiness fails:

```text
Pod
 |
 X
Service traffic
```

but Kubernetes doesn't necessarily restart the container.

This is extremely important during deployments.

---

# 14. Liveness Probe

Question:

> Is the application still alive?

If liveness fails repeatedly:

```text
Pod
 |
 v
Kubernetes
 |
 v
Container restart
```

Do not make liveness checks overly aggressive.

A badly designed liveness probe can cause:

```text
Healthy application
      |
      v
Temporary dependency problem
      |
      v
Liveness failure
      |
      v
Restart
      |
      v
More load
      |
      v
More failures
```

This can become a restart storm.

---

# 15. Spring Boot Health Endpoints

Our application already has:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus

  endpoint:
    health:
      probes:
        enabled: true
```

This gives:

```text
/actuator/health
/actuator/health/liveness
/actuator/health/readiness
/actuator/prometheus
```

Therefore Kubernetes can use Spring Boot's health mechanisms.

---

# 16. Service Account

Create:

```text
templates/serviceaccount.yaml
```

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:
  name: {{ .Values.serviceAccount.name }}

  labels:
    app.kubernetes.io/name: {{ include "order-service.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

This service account becomes important later for:

```text
EKS Pod Identity
```

---

# 17. Why Every Application Gets Its Own Service Account

Avoid:

```text
All applications
      |
      v
default ServiceAccount
```

Instead:

```text
order-service
    |
    v
order-service ServiceAccount

payment-service
    |
    v
payment-service ServiceAccount
```

This allows workload identity to be scoped independently.

---

# 18. Pod Identity Flow

Eventually:

```text
Order Pod
   |
   v
order-service ServiceAccount
   |
   v
EKS Pod Identity Association
   |
   v
Order Service IAM Role
   |
   +--> SQS
   +--> Secrets Manager
   +--> S3
```

Payment service gets a different identity.

This is least privilege.

---

# 19. Service

Create:

```text
templates/service.yaml
```

```yaml
apiVersion: v1

kind: Service

metadata:
  name: {{ include "order-service.fullname" . }}

  labels:
    app.kubernetes.io/name: {{ include "order-service.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}

spec:

  type: {{ .Values.service.type }}

  selector:
    app.kubernetes.io/name: {{ include "order-service.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}

  ports:

    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
```

---

# 20. Service Traffic Flow

Our current flow:

```text
Client
  |
  v
ALB
  |
  v
Ingress
  |
  v
ClusterIP Service
  |
  v
Pod
  |
  v
8080
```

The Service selects pods through:

```yaml
selector:
  app.kubernetes.io/name: order-service
```

---

# 21. Why We Don't Use Pod IPs

Never configure:

```text
http://10.0.12.42:8080
```

as the permanent application endpoint.

Pods can be recreated:

```text
Old Pod
10.0.12.42
    |
    X
Deleted

New Pod
10.0.13.55
```

Service provides stable discovery:

```text
order-service
      |
      v
Current healthy pods
```

---

# 22. ConfigMap

Create:

```text
templates/configmap.yaml
```

```yaml
apiVersion: v1

kind: ConfigMap

metadata:
  name: {{ include "order-service.fullname" . }}-config

data:

  SPRING_PROFILES_ACTIVE: {{ .Values.config.springProfile | quote }}
```

But remember:

> ConfigMap is for configuration, not secrets.

Do not put:

```text
DB_PASSWORD
AWS_SECRET_ACCESS_KEY
API_TOKEN
```

into ConfigMap.

---

# 23. Production Configuration

Our application should receive:

```text
non-secret configuration
        ↓
ConfigMap
```

and:

```text
secret values
        ↓
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
```

We will implement that later.

---

# 24. HPA

Create:

```text
templates/hpa.yaml
```

```yaml
{{- if .Values.hpa.enabled }}

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:
  name: {{ include "order-service.fullname" . }}

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: {{ include "order-service.fullname" . }}

  minReplicas: {{ .Values.hpa.minReplicas }}

  maxReplicas: {{ .Values.hpa.maxReplicas }}

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.targetCPUUtilizationPercentage }}

{{- end }}
```

---

# 25. HPA Flow

Example:

```text
Order Service
     |
     v
CPU increases
     |
     v
Metrics Server
     |
     v
HPA
     |
     v
Increase replicas
```

Example:

```text
2 Pods
  |
  | CPU > 70%
  v
4 Pods
  |
  v
8 Pods
```

up to:

```text
maxReplicas: 10
```

---

# 26. HPA Does Not Create Nodes

Important distinction.

Suppose:

```text
HPA
2 Pods
   ↓
10 Pods
```

but the cluster has no capacity.

Then:

```text
10 desired Pods
       |
       v
Node capacity insufficient
       |
       v
Some Pods Pending
```

The solution may involve:

```text
Cluster Autoscaler
```

or:

```text
Karpenter
```

which we cover later.

---

# 27. Resource Requests

Our deployment has:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

This means:

```text
CPU = 0.25 CPU
Memory = 512 MiB
```

Kubernetes uses requests for scheduling.

---

# 28. Resource Limits

We also define:

```yaml
limits:
  cpu: 1000m
  memory: 1Gi
```

This gives a maximum resource boundary.

Be careful with memory limits.

If a Java process exceeds its container memory limit:

```text
Container
   |
   v
Memory limit exceeded
   |
   v
OOMKilled
```

---

# 29. Java Container Memory

Java applications need careful memory sizing.

Suppose:

```text
Container memory limit
1 GiB
```

The JVM cannot safely assume it can use all of that for heap.

We should account for:

```text
JVM heap
Metaspace
thread stacks
native memory
class metadata
direct buffers
```

Later we can tune:

```text
-Xms
-Xmx
```

or use JVM container-aware defaults.

---

# 30. Pod Disruption Budget

Create:

```text
templates/pdb.yaml
```

```yaml
{{- if .Values.pdb.enabled }}

apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:
  name: {{ include "order-service.fullname" . }}

spec:

  minAvailable: {{ .Values.pdb.minAvailable }}

  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "order-service.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}

{{- end }}
```

---

# 31. Why PDB?

Suppose:

```text
3 Order Service pods
```

and we perform node maintenance.

A PDB helps Kubernetes disruption mechanisms preserve the desired level of availability.

Example:

```text
minAvailable: 2
```

means we should maintain at least two available replicas during supported voluntary disruption scenarios.

PDB does not protect against every failure.

It does not guarantee availability during:

```text
node crash
AZ outage
application crash
OOM
```

---

# 32. Ingress

For our AWS architecture:

```text
Internet
   |
   v
ALB
```

we can define:

```text
templates/ingress.yaml
```

```yaml
{{- if .Values.ingress.enabled }}

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:
  name: {{ include "order-service.fullname" . }}

  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip

spec:

  ingressClassName: {{ .Values.ingress.className }}

  rules:

    - host: {{ .Values.ingress.host }}

      http:

        paths:

          - path: /api/v1/orders

            pathType: Prefix

            backend:

              service:
                name: {{ include "order-service.fullname" . }}

                port:
                  number: {{ .Values.service.port }}

{{- end }}
```

---

# 33. ALB Target Type

We use:

```text
target-type: ip
```

which allows the ALB to route traffic directly to pod IP targets.

Conceptually:

```text
Internet
   |
   v
ALB
   |
   +--> Pod IP
   +--> Pod IP
   +--> Pod IP
```

This is different from routing only to EC2 node ports.

---

# 34. ALB Health Checks

The ALB controller can configure target health checks.

Our application must provide a meaningful health endpoint.

For example:

```text
/actuator/health
```

The ALB should only send traffic to healthy targets according to the configured health-check behavior.

---

# 35. Ingress Should Be Disabled Initially

Our base values contain:

```yaml
ingress:
  enabled: false
```

DEV can enable it:

```yaml
ingress:
  enabled: true
```

This prevents accidentally creating an Internet-facing ALB merely because someone installed the chart.

---

# 36. Development Values

Create:

```text
values-dev.yaml
```

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

resources:
  requests:
    cpu: 200m
    memory: 384Mi

  limits:
    cpu: 750m
    memory: 768Mi

config:
  springProfile: dev

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 4
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

ingress:
  enabled: true
  className: alb
  host: order-dev.example.com
```

---

# 37. QA Values

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

config:
  springProfile: qa

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

ingress:
  enabled: true
  className: alb
  host: order-qa.example.com
```

---

# 38. Staging Values

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"

resources:
  requests:
    cpu: 300m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

config:
  springProfile: staging

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 8
  targetCPUUtilizationPercentage: 65

pdb:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  className: alb
  host: order-staging.example.com
```

---

# 39. Production Values

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 500m
    memory: 768Mi

  limits:
    cpu: 1500m
    memory: 1536Mi

config:
  springProfile: prod

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 65

pdb:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  className: alb
  host: order.example.com
```

These are starting values, not universal production numbers. Real values should come from load testing and observed resource usage.

---

# 40. Important: Don't Store Secrets in Values

Never:

```yaml
database:
  username: postgres
  password: SuperSecret123
```

in:

```text
values-prod.yaml
```

Even if the Git repository is private.

Git history persists.

Instead:

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
Pod
```

---

# 41. Database Configuration

The production application needs:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

We should eventually configure:

```yaml
envFrom:
  - secretRef:
      name: order-service-db
```

But the actual Secret should be generated/synchronized through our secrets architecture.

We'll implement that in the dedicated secrets/GitOps stage.

---

# 42. Environment-Specific Values

Our structure is:

```text
values.yaml
     |
     +--> common defaults

values-dev.yaml
     |
     +--> DEV overrides

values-qa.yaml
     |
     +--> QA overrides

values-staging.yaml
     |
     +--> STAGING overrides

values-prod.yaml
     |
     +--> PROD overrides
```

Helm merges values.

---

# 43. Helm Rendering

Before deploying anything, render the chart locally.

From:

```text
helm/order-service
```

run:

```bash
helm template order-service .
```

This generates the final Kubernetes YAML.

This is extremely useful.

You can inspect:

```text
Deployment
Service
ServiceAccount
ConfigMap
HPA
PDB
Ingress
```

without changing the cluster.

---

# 44. Render DEV

```bash
helm template order-service . \
  -f values-dev.yaml
```

To save:

```bash
helm template order-service . \
  -f values-dev.yaml \
  > rendered-dev.yaml
```

Inspect:

```bash
less rendered-dev.yaml
```

---

# 45. Helm Lint

Run:

```bash
helm lint .
```

Expected:

```text
1 chart(s) linted, 0 chart(s) failed
```

This catches many chart mistakes before GitOps.

---

# 46. Helm Dependency

If the chart later uses dependencies:

```text
PostgreSQL
Redis
external chart
```

Helm can manage chart dependencies.

But for our application chart:

```text
order-service
```

we should avoid bundling unrelated infrastructure.

For example, don't put:

```text
PostgreSQL
Redis
Kafka
```

inside the Order Service chart just because the application uses them.

Those are platform/data services with independent lifecycles.

---

# 47. Application Chart vs Platform Chart

### Application chart

```text
order-service
```

contains:

```text
Deployment
Service
Ingress
HPA
PDB
ServiceAccount
ConfigMap
```

### Platform

contains:

```text
ArgoCD
Prometheus
Grafana
ALB Controller
External Secrets
```

### Infrastructure

contains:

```text
VPC
EKS
RDS
ECR
SQS
IAM
```

These should have separate ownership/lifecycles.

---

# 48. Helm Release

When deployed, Helm creates a release such as:

```text
order-service
```

in namespace:

```text
production
```

Conceptually:

```text
Helm Release
    |
    +--> Deployment
    +--> Service
    +--> HPA
    +--> PDB
    +--> ServiceAccount
```

ArgoCD will later manage the desired state rather than developers manually running Helm commands against production.

---

# 49. Namespace

We should deploy applications into dedicated namespaces.

Example:

```text
production
```

Then:

```text
production
├── order-service
├── payment-service
├── inventory-service
└── ...
```

Platform components remain separate:

```text
argocd
monitoring
logging
external-secrets
```

---

# 50. Namespace Isolation

Namespaces help with:

```text
RBAC
resource quotas
network policies
organization
observability
```

But a namespace is **not a complete security boundary**.

We still need:

```text
IAM
RBAC
NetworkPolicy
Security Groups
Pod Security controls
Secrets management
```

---

# 51. Labels

Use consistent labels:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/instance: order-service
```

We can also add:

```text
app.kubernetes.io/version
app.kubernetes.io/component
app.kubernetes.io/part-of
app.kubernetes.io/managed-by
```

These labels help with:

```text
monitoring
troubleshooting
selection
cost attribution
Kubernetes queries
```

---

# 52. Deployment Version Labels

We can expose the image version:

```yaml
metadata:
  labels:
    app.kubernetes.io/version: {{ .Values.image.tag | quote }}
```

Then:

```bash
kubectl get pods --show-labels
```

helps identify which image version is running.

---

# 53. Graceful Shutdown

Our Deployment has:

```yaml
terminationGracePeriodSeconds: 30
```

When Kubernetes terminates a pod:

```text
SIGTERM
   |
   v
Application shutdown
   |
   v
Finish active requests
   |
   v
Container exits
```

Spring Boot supports graceful shutdown when configured appropriately.

This is important during:

```text
rolling deployment
node drain
scale down
```

---

# 54. Rolling Deployment

We configured:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Suppose:

```text
Current:
Pod A
Pod B
Pod C
```

New version:

```text
Pod A
Pod B
Pod C
Pod D (new)
```

Once the new pod becomes Ready:

```text
old pod removed
```

This reduces downtime.

---

# 55. Why Readiness Matters During Rolling Updates

Suppose new version starts:

```text
New Pod
   |
   v
Application still initializing
```

Readiness:

```text
FAIL
```

Therefore:

```text
Service
   |
   X
New Pod
```

Traffic remains on healthy pods.

Once:

```text
Readiness = PASS
```

the pod becomes an eligible traffic target.

---

# 56. Pod Anti-Affinity

For production, we should avoid putting all replicas on the same node.

We can later add:

```yaml
affinity:
  podAntiAffinity:
```

or topology spread constraints.

Conceptually:

```text
AZ-A
  |
  +--> order pod

AZ-B
  |
  +--> order pod

AZ-C
  |
  +--> order pod
```

This improves resilience.

We will add topology rules when we finalize production scheduling.

---

# 57. Topology Spread

A more explicit pattern is:

```yaml
topologySpreadConstraints:
  - maxSkew: 1

    topologyKey: topology.kubernetes.io/zone

    whenUnsatisfiable: ScheduleAnyway

    labelSelector:
      matchLabels:
        app.kubernetes.io/name: order-service
```

For critical production workloads, the exact policy should be selected based on capacity and availability requirements.

---

# 58. Security Context

Production containers should not normally run as root.

Our Dockerfile already has:

```dockerfile
USER 10001
```

We should reinforce that at Kubernetes level.

For example:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

And:

```yaml
containerSecurityContext:
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

The exact settings depend on application behavior.

---

# 59. Recommended Security Context

Add under the pod/container configuration:

```yaml
securityContext:
  runAsNonRoot: true

containers:
  - name: order-service

    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

If the application needs filesystem writes, provide a specific writable volume instead of making the whole root filesystem writable.

---

# 60. Read-Only Root Filesystem

A hardened container can use:

```yaml
readOnlyRootFilesystem: true
```

But first test the application.

If Java/Spring needs writable temporary space:

```text
/tmp
```

provide:

```yaml
emptyDir: {}
```

and mount it explicitly.

Don't enable security settings blindly without testing the workload.

---

# 61. Helm Validation

Run:

```bash
helm lint helm/order-service
```

Then:

```bash
helm template \
  order-service \
  helm/order-service \
  -f helm/order-service/values-dev.yaml
```

Then inspect:

```bash
kubectl apply \
  --dry-run=client \
  -f rendered-dev.yaml
```

This is local validation.

---

# 62. Kubernetes Server-Side Validation

Once connected to EKS:

```bash
kubectl apply \
  --dry-run=server \
  -f rendered-dev.yaml
```

This asks the Kubernetes API server to validate the manifests without actually applying them.

This is useful before GitOps deployment.

---

# 63. Helm Package

We can package the chart:

```bash
helm package helm/order-service
```

Output:

```text
order-service-0.1.0.tgz
```

But in our GitOps architecture, we will normally store the chart in Git rather than treating the `.tgz` as the primary artifact.

---

# 64. Why Helm Charts Live in Git

Our GitOps model is:

```text
Git
 |
 +--> Helm chart
 |
 +--> environment values
 |
 v
ArgoCD
```

This gives us:

```text
Version history
Pull requests
Code review
Rollback
Audit trail
```

---

# 65. GitOps Repository Boundary

We have two possible locations:

### Application repository

```text
order-service/
└── helm/
```

contains the reusable application chart.

### GitOps repository

```text
gitops/
└── environments/
```

contains environment-specific desired state.

This gives:

```text
Application Repo
       |
       +--> Source
       +--> Dockerfile
       +--> Helm chart

GitOps Repo
       |
       +--> DEV deployment
       +--> QA deployment
       +--> STAGING deployment
       +--> PROD deployment
```

---

# 66. Why Separate GitOps?

Because application code and production deployment approval are different concerns.

For example:

```text
Developer merges code
       |
       v
CI builds image
       |
       v
ECR
```

This does not automatically mean:

```text
PROD
```

should change.

Production promotion can require:

```text
GitOps PR
   |
   v
Review
   |
   v
Approval
   |
   v
ArgoCD
```

---

# 67. Image Update

Suppose CI creates:

```text
order-service:a91c2f7d1234
```

GitOps should update:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: a91c2f7d1234
```

Then:

```text
GitOps commit
      |
      v
ArgoCD
      |
      v
Helm
      |
      v
EKS
```

---

# 68. Do Not Put the Git SHA in the Dockerfile

Avoid:

```dockerfile
ARG VERSION
```

unless needed for application metadata.

The image identity should be determined by the CI build/tagging process.

The same image can then be promoted across environments without rebuilding.

---

# 69. Production Image Pinning

Our chart currently uses:

```yaml
image:
  repository: ...
  tag: ...
```

A stronger production model can pin the digest:

```yaml
image:
  repository: ...
  digest: sha256:abc123...
```

Then the deployment references:

```text
repository@sha256:abc123...
```

rather than only:

```text
repository:tag
```

This provides stronger protection against tag ambiguity.

---

# 70. Chart Improvement for Digest

We can eventually change the template to support:

```yaml
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

when a digest is supplied.

For now, Git SHA tags are sufficient for the initial implementation.

We will introduce digest pinning when we implement the complete GitOps promotion flow.

---

# 71. Final Helm Flow

The Helm implementation now looks like:

```text
order-service
       |
       v
Helm Chart
       |
       +--> Deployment
       +--> Service
       +--> ServiceAccount
       +--> ConfigMap
       +--> Ingress
       +--> HPA
       +--> PDB
       |
       v
GitOps Repository
       |
       v
ArgoCD
       |
       v
EKS
```

---

# 72. What Helm Does NOT Do

Helm does not:

* build Docker images
* scan Docker images
* store images
* create EKS
* create VPCs
* replace Terraform
* replace ArgoCD
* replace IAM
* replace AWS Secrets Manager

Our separation remains:

```text
Terraform
   |
   +--> AWS infrastructure

GitHub Actions
   |
   +--> CI

ECR
   |
   +--> Container artifacts

Helm
   |
   +--> Kubernetes packaging

GitOps
   |
   +--> Desired state

ArgoCD
   |
   +--> Reconciliation

EKS
   |
   +--> Runtime
```

---

# 73. Current Project Progress

```text
53 Repository Setup
        ↓
54 Order Service
        ↓
55 Local Docker
        ↓
56 Terraform State
        ↓
57 Terraform VPC
        ↓
58 Terraform IAM
        ↓
59 Terraform ECR
        ↓
60 Terraform EKS
        ↓
61 EKS Add-ons
        ↓
62 GitHub Actions CI
        ↓
63 Helm Implementation
```

We now have:

```text
Source Code
    ↓
CI
    ↓
ECR
    ↓
Helm
```

The remaining missing piece in the delivery chain is **GitOps + ArgoCD**.

---

# 74. Next Chapter

## `64-GitOps-Implementation.md`

We will create the actual GitOps repository:

```text
gitops/
│
├── applications/
│
├── environments/
│   ├── dev/
│   │   └── order-service/
│   ├── qa/
│   │   └── order-service/
│   ├── staging/
│   │   └── order-service/
│   └── prod/
│       └── order-service/
│
└── argocd/
    ├── projects/
    ├── applications/
    └── appsets/
```

Then we'll implement the actual promotion path:

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions
   ↓
ECR
   ↓
GitOps PR
   ↓
DEV
   ↓
QA
   ↓
STAGING
   ↓
PROD
   ↓
ArgoCD
   ↓
EKS
```

Most importantly, we'll show **exactly how CI updates the GitOps repository with the new image tag/digest without giving CI direct production `kubectl` access**.
