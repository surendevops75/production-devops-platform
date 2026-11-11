# 81 — Rolling Deployment Implementation

> Production implementation of Kubernetes Rolling Updates for `order-service` using **Helm + EKS + ArgoCD + ECR + GitOps**, with health probes, graceful shutdown, PDB, autoscaling, rollback, and production validation.

---

## 1. Objective

In the previous chapters we established:

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Build + Test + Trivy
   ↓
ECR
   ↓
GitOps Repository
   ↓
ArgoCD
   ↓
EKS
```

Now we implement the first production deployment strategy:

```text
Rolling Deployment
```

The goal is:

```text
Version 1
   ↓
Running Pods
   ↓
Gradually replace
   ↓
Version 2
   ↓
No planned application downtime
```

For `order-service`, we will use:

* Kubernetes Deployment
* `RollingUpdate`
* `maxUnavailable: 0`
* `maxSurge: 1`
* Readiness probe
* Liveness probe
* Startup probe
* Graceful shutdown
* PodDisruptionBudget
* Resource requests/limits
* HPA compatibility
* ArgoCD
* Immutable ECR image
* GitOps rollback
* Jira Change Request

---

# 2. Production Architecture

The deployment path is:

```text
                    Internet
                       |
                   Route 53
                       |
                  CloudFront/WAF
                       |
                      ALB
                       |
          AWS Load Balancer Controller
                       |
                 Kubernetes Service
                       |
             +---------+---------+
             |                   |
          Pod v1              Pod v1
             |                   |
             +---------+---------+
                       |
                  RDS PostgreSQL
```

During deployment:

```text
Before:

Service
  |
  +-- Pod v1
  +-- Pod v1
  +-- Pod v1


During:

Service
  |
  +-- Pod v1
  +-- Pod v1
  +-- Pod v1
  +-- Pod v2   ← new


Then:

Service
  |
  +-- Pod v1
  +-- Pod v1
  +-- Pod v2
  +-- Pod v2


Finally:

Service
  |
  +-- Pod v2
  +-- Pod v2
  +-- Pod v2
```

The old Pods are removed only as new Pods become ready according to the Deployment strategy.

---

# 3. Why `maxUnavailable: 0`?

Our production configuration will initially use:

```yaml
maxUnavailable: 0
```

This means:

> Kubernetes should not intentionally reduce the number of available Pods below the desired replica count during the rollout.

For example:

```yaml
replicas: 3
```

We want:

```text
Desired = 3

Old:
Pod A
Pod B
Pod C
```

Kubernetes creates a new Pod before removing an old Pod.

```text
Old:
A
B
C

New:
D

Total:
A B C D
```

Once `D` becomes Ready:

```text
A B C D
```

Kubernetes can terminate one old Pod.

```text
B C D
```

Then create another:

```text
B C D E
```

And continue.

---

# 4. Helm Values

In:

```text
order-service/helm/order-service/values.yaml
```

define:

```yaml
replicaCount: 3

strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

minReadySeconds: 10

revisionHistoryLimit: 10
```

This gives us:

```text
3 replicas
1 additional Pod allowed
0 unavailable Pods allowed
10 seconds minimum ready time
10 ReplicaSets retained
```

---

# 5. Deployment Template

The Helm template:

```text
helm/order-service/templates/deployment.yaml
```

can contain:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}

spec:
  replicas: {{ .Values.replicaCount }}

  revisionHistoryLimit: {{ .Values.revisionHistoryLimit }}

  minReadySeconds: {{ .Values.minReadySeconds }}

  strategy:
    type: {{ .Values.strategy.type }}

    rollingUpdate:
      maxSurge: {{ .Values.strategy.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.strategy.rollingUpdate.maxUnavailable }}

  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}

  template:

    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}

    spec:

      terminationGracePeriodSeconds: 30

      containers:

        - name: order-service

          image: >-
            {{ .Values.image.repository }}{{- if .Values.image.digest -}}
            @{{ .Values.image.digest }}
            {{- else -}}
            :{{ .Values.image.tag }}
            {{- end }}

          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}

            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}

          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http

            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http

            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http

            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
```

---

# 6. Why Do We Need Three Probes?

This is extremely important for production deployments.

We have:

```text
Startup Probe
      ↓
Readiness Probe
      ↓
Liveness Probe
```

They have different purposes.

---

# 7. Startup Probe

The startup probe answers:

> Has the application successfully started?

Example:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: http

  periodSeconds: 5
  failureThreshold: 30
```

That allows approximately:

```text
5 × 30 = 150 seconds
```

for the application to start.

This is useful if:

* JVM startup takes time
* Spring Boot initializes many beans
* application performs startup initialization
* database connection initialization takes time

Without a startup probe, Kubernetes might kill a slow-starting application because the liveness probe starts failing too early.

---

# 8. Readiness Probe

Readiness answers:

> Can this Pod receive production traffic right now?

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: http
```

Suppose a new Pod starts:

```text
Pod v2
Status: Running
Ready: False
```

The Kubernetes Service should not treat that Pod as ready for normal traffic.

Once:

```text
readinessProbe = SUCCESS
```

the Pod becomes:

```text
Ready: True
```

and can receive traffic.

This is one of the most important mechanisms behind safe rolling deployments.

---

# 9. Liveness Probe

Liveness answers:

> Is the application still alive, or is it stuck?

Example:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: http
```

If the application becomes unhealthy repeatedly:

```text
Liveness failure
       ↓
Kubernetes restarts container
```

Important distinction:

```text
Readiness failure
    ↓
Stop sending traffic
    ↓
Do not necessarily restart container
```

while:

```text
Liveness failure
    ↓
Restart container
```

---

# 10. Spring Boot Configuration

In:

```text
src/main/resources/application-prod.yaml
```

configure:

```yaml
server:
  port: 8080
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

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

Spring Boot then exposes Kubernetes-oriented health endpoints.

Typically:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Verify locally:

```bash
curl http://localhost:8080/actuator/health/liveness
```

and:

```bash
curl http://localhost:8080/actuator/health/readiness
```

---

# 11. Graceful Shutdown

Imagine a request is currently running:

```text
Client
  |
  ↓
Pod v1
  |
  └── Processing order request
```

Kubernetes decides to terminate the Pod.

We don't want:

```text
Kubernetes SIGTERM
       ↓
Application immediately dies
       ↓
Request fails
```

Instead:

```text
SIGTERM
  ↓
Spring begins graceful shutdown
  ↓
Stop accepting new work
  ↓
Finish existing requests
  ↓
Application exits
```

We configured:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

And Kubernetes:

```yaml
terminationGracePeriodSeconds: 30
```

These values need to be chosen together with the application's real request durations.

---

# 12. PodDisruptionBudget

Create:

```text
helm/order-service/templates/pdb.yaml
```

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: {{ include "order-service.fullname" . }}

spec:
  minAvailable: 1

  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
```

With:

```text
3 Pods
```

we want at least:

```text
1 Pod
```

available during voluntary disruptions.

PDB can help during events such as:

* node draining
* cluster maintenance
* some voluntary infrastructure disruptions

Important:

> A PDB does not guarantee availability against every failure.

It does not magically protect against:

```text
Node crash
AZ outage
Application bug
Database outage
Bad deployment
```

---

# 13. Resource Requests and Limits

Example production values:

```yaml
resources:

  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

Kubernetes uses requests when scheduling Pods.

For example:

```text
Pod requires:
250m CPU
512Mi memory
```

The scheduler finds a node with enough allocatable capacity.

Limits define the upper boundaries for the container.

Memory is especially important.

If the container exceeds its memory limit:

```text
Memory limit exceeded
       ↓
OOMKilled
       ↓
Container restart
```

---

# 14. Production Environment Values

Our GitOps repository contains:

```text
gitops/
└── environments/
    └── prod/
        └── order-service/
            └── values.yaml
```

Example:

```yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
  digest: "sha256:9a7b123..."

strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

minReadySeconds: 10

revisionHistoryLimit: 10

resources:
  requests:
    cpu: "250m"
    memory: "512Mi"

  limits:
    cpu: "1000m"
    memory: "1Gi"
```

The important point:

```text
Application repository
        ↓
Helm template

GitOps repository
        ↓
Production configuration
```

---

# 15. Why Keep Strategy in GitOps?

Because production behavior should be version controlled.

For example:

```text
Git commit A

replicas: 3
maxUnavailable: 0
maxSurge: 1
```

Then later:

```text
Git commit B

replicas: 5
maxUnavailable: 0
maxSurge: 2
```

ArgoCD can show exactly what changed.

This gives:

```text
Who changed it?
When?
Why?
Which PR?
Which Jira change?
```

---

# 16. Validate the Helm Chart

From the application repository:

```bash
helm lint helm/order-service
```

Then:

```bash
helm template order-service helm/order-service \
  -f helm/order-service/values.yaml
```

For production:

```bash
helm template order-service helm/order-service \
  -f helm/order-service/values.yaml \
  -f ../gitops/environments/prod/order-service/values.yaml
```

Check the generated Deployment.

You should see:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

---

# 17. GitOps Flow

We don't manually change the production Deployment.

Instead:

```text
New application code
       ↓
GitHub
       ↓
GitHub Actions
       ↓
Build
       ↓
Test
       ↓
Trivy
       ↓
ECR
       ↓
Image digest
       ↓
GitOps PR
       ↓
Jira Change
       ↓
Approval
       ↓
Merge
       ↓
ArgoCD
       ↓
EKS
       ↓
Rolling Update
```

---

# 18. What Actually Changes in Kubernetes?

Suppose production currently runs:

```text
order-service:v1
```

with three Pods:

```text
Pod A → v1
Pod B → v1
Pod C → v1
```

GitOps changes:

```text
order-service:v2
```

ArgoCD detects:

```text
Desired state != Live state
```

ArgoCD reconciles.

Kubernetes sees a new Pod template.

Because the Pod template changed:

```text
New ReplicaSet
```

is created.

---

# 19. ReplicaSets During Rolling Update

Before:

```text
Deployment
    |
    └── ReplicaSet-v1
          |
          ├── Pod A
          ├── Pod B
          └── Pod C
```

After starting deployment:

```text
Deployment
    |
    +── ReplicaSet-v1
    |     ├── Pod A
    |     ├── Pod B
    |     └── Pod C
    |
    └── ReplicaSet-v2
          └── Pod D
```

Then:

```text
ReplicaSet-v1
    Pod A
    Pod B

ReplicaSet-v2
    Pod D
    Pod E
```

Eventually:

```text
ReplicaSet-v1
    0 Pods

ReplicaSet-v2
    Pod D
    Pod E
    Pod F
```

---

# 20. `pod-template-hash`

You may see:

```bash
kubectl get pods -n ecommerce-prod
```

Output similar to:

```text
order-service-6c8b7d9f5c-abc12
order-service-6c8b7d9f5c-def34
order-service-7f9d88c4c6-xyz12
```

The hash identifies the Pod template/ReplicaSet generation.

During deployment, you can see two hashes temporarily.

That is expected.

---

# 21. Deploy Through ArgoCD

After the GitOps PR is approved and merged:

```bash
argocd app get order-service-prod
```

Check:

```text
Sync Status: OutOfSync
```

Then:

```bash
argocd app sync order-service-prod
```

Watch:

```bash
kubectl get pods -n ecommerce-prod -w
```

And:

```bash
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

Expected:

```text
Waiting for deployment "order-service" rollout to finish...
```

Eventually:

```text
deployment "order-service" successfully rolled out
```

---

# 22. Watch the ReplicaSets

Run:

```bash
kubectl get rs -n ecommerce-prod
```

During rollout:

```text
NAME                       DESIRED   CURRENT   READY
order-service-old          2         2         2
order-service-new          1         1         1
```

Then:

```text
order-service-old          1         1         1
order-service-new          2         2         2
```

Finally:

```text
order-service-old          0         0         0
order-service-new          3         3         3
```

---

# 23. Watch the Deployment

```bash
kubectl get deployment order-service \
  -n ecommerce-prod
```

Example:

```text
NAME            READY   UP-TO-DATE   AVAILABLE
order-service   3/3     3            3
```

Meaning:

```text
READY       = 3/3
UP-TO-DATE  = 3
AVAILABLE   = 3
```

This is what you want after a successful deployment.

---

# 24. Verify the Actual Image

Don't only check:

```bash
kubectl get deployment
```

Check the actual running Pod.

First:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

You should see something like:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

This proves which image was actually pulled.

---

# 25. Verify Deployment Image

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:9a7b123...
```

This is stronger than relying only on:

```text
:latest
```

or a mutable tag.

---

# 26. Test Application Health

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl describe pod <POD_NAME> \
  -n ecommerce-prod
```

Check:

```text
Ready: True
```

You can also test from inside the cluster or through the application endpoint:

```bash
curl https://api.example.com/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

For production, health endpoints should be appropriately protected rather than unnecessarily exposed publicly.

---

# 27. Test Zero-Downtime Behavior

A simple test:

```bash
while true; do
  curl -s -o /dev/null \
    -w "%{http_code}\n" \
    https://api.example.com/api/v1/orders

  sleep 0.2
done
```

While this is running, perform the deployment.

You want to observe:

```text
200
200
200
200
200
...
```

rather than a series of:

```text
502
503
504
```

This is only a basic validation. Real production validation should also exercise representative application operations and monitor latency/error-rate SLOs.

---

# 28. What Happens If Readiness Fails?

Suppose new version:

```text
v2
```

has a startup problem.

The Pod might be:

```text
Running
Ready: False
```

Because:

```text
readinessProbe = failure
```

Kubernetes does not consider it ready.

With:

```yaml
maxUnavailable: 0
```

the rollout cannot safely remove all the healthy old capacity simply to continue.

You may see:

```text
Deployment
   |
   +-- v1 Ready
   +-- v1 Ready
   +-- v1 Ready
   |
   +-- v2 NotReady
```

This is exactly why readiness probes matter.

---

# 29. Failure Scenario — Bad Image

Suppose GitOps accidentally specifies:

```yaml
image:
  tag: "does-not-exist"
```

ArgoCD deploys it.

Kubernetes attempts:

```text
Pull image
    ↓
ECR
    ↓
Image not found
```

Pod:

```text
ImagePullBackOff
```

Check:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl describe pod <POD_NAME> \
  -n ecommerce-prod
```

Look under:

```text
Events
```

You may see image pull errors.

---

# 30. Failure Scenario — Application Crash

Suppose the image exists but the application fails during startup.

You may see:

```text
CrashLoopBackOff
```

Check:

```bash
kubectl logs <POD_NAME> \
  -n ecommerce-prod
```

Previous container:

```bash
kubectl logs <POD_NAME> \
  -n ecommerce-prod \
  --previous
```

Then:

```bash
kubectl describe pod <POD_NAME> \
  -n ecommerce-prod
```

Investigate:

```text
Environment
Events
Probes
Exit Code
Restart Count
```

---

# 31. Failure Scenario — Readiness Failure

Suppose the application starts but:

```text
/actuator/health/readiness
```

returns failure.

Then:

```text
Pod = Running
Pod = NotReady
```

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n ecommerce-prod
```

You may see:

```text
Readiness probe failed
```

Check application logs:

```bash
kubectl logs <POD_NAME> \
  -n ecommerce-prod
```

Potential causes:

```text
Database connectivity
Wrong environment variable
Secret problem
Application initialization failure
Dependency unavailable
Incorrect probe path
Port mismatch
```

---

# 32. Failure Scenario — OOMKilled

Check:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

If:

```text
OOMKilled
```

investigate:

```bash
kubectl describe pod <POD_NAME> \
  -n ecommerce-prod
```

and:

```bash
kubectl top pod -n ecommerce-prod
```

Then review:

```text
Memory request
Memory limit
JVM heap
Traffic
Application memory behavior
HPA
```

Do not simply keep increasing memory without understanding why the process needs it.

---

# 33. Rollout History

Check:

```bash
kubectl rollout history deployment/order-service \
  -n ecommerce-prod
```

You may see:

```text
REVISION
1
2
3
```

Detailed:

```bash
kubectl rollout history deployment/order-service \
  -n ecommerce-prod \
  --revision=3
```

This is useful for investigation.

---

# 34. Emergency Kubernetes Rollback

Kubernetes supports:

```bash
kubectl rollout undo deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

However, in our GitOps architecture this creates an important issue:

```text
Kubernetes live state
        ≠
Git desired state
```

ArgoCD may detect the difference and reconcile it.

Therefore:

> `kubectl rollout undo` is an emergency operational action, not the normal GitOps rollback mechanism.

---

# 35. Correct GitOps Rollback

Suppose production currently has:

```text
v2
```

and the previous version was:

```text
v1
```

Git history:

```text
Commit A → v1
Commit B → v2
```

Rollback:

```text
Git revert Commit B
        ↓
GitOps PR
        ↓
Review/approval
        ↓
Merge
        ↓
ArgoCD
        ↓
EKS
        ↓
v1
```

This gives us an auditable rollback.

---

# 36. Verify Rollback

After rollback:

```bash
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

And verify the running Pod:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-prod \
  -o jsonpath='{.status.containerStatuses[0].imageID}'
```

Then check:

```text
HTTP health
Application errors
Latency
Prometheus
Grafana
Logs
Traces
```

---

# 37. HPA and Rolling Deployment

Suppose:

```yaml
replicaCount: 3
```

and HPA scales:

```text
3 → 6
```

During deployment, `maxSurge` still applies to the Deployment's desired replica count.

For example:

```text
Desired replicas = 6
maxSurge = 1
```

Potential rollout capacity:

```text
6 old
+
1 new
=
7
```

This means rolling deployment requires enough cluster capacity.

If the cluster cannot schedule the additional Pod:

```text
New Pod
   ↓
Pending
   ↓
Insufficient CPU/memory
```

The rollout can stall.

This is why:

```text
HPA
+
Resource requests
+
Cluster autoscaling/Karpenter
+
RollingUpdate
```

must be designed together.

---

# 38. Availability Across Nodes

For production, don't put all three replicas on one node if you can avoid it.

For example:

```text
Node A
 └── order-service Pod

Node B
 └── order-service Pod

Node C
 └── order-service Pod
```

Better still, use topology spread constraints or appropriate pod anti-affinity so replicas are distributed across failure domains.

Conceptually:

```text
AZ-A
  Pod 1

AZ-B
  Pod 2

AZ-C
  Pod 3
```

This reduces the chance that a single node/AZ failure removes all application replicas.

---

# 39. Example Topology Spread

A simplified example:

```yaml
topologySpreadConstraints:

  - maxSkew: 1

    topologyKey: topology.kubernetes.io/zone

    whenUnsatisfiable: ScheduleAnyway

    labelSelector:
      matchLabels:
        app.kubernetes.io/name: order-service
```

The exact scheduling policy should be chosen based on cluster capacity and workload requirements.

For critical production workloads, you should deliberately decide whether:

```text
ScheduleAnyway
```

or:

```text
DoNotSchedule
```

is appropriate.

---

# 40. ALB During Rolling Deployment

Traffic path:

```text
Client
   ↓
ALB
   ↓
Kubernetes Service
   ↓
Ready Pods
```

During deployment:

```text
ALB
 ↓
Service
 ↓
+--------+--------+--------+
|        |        |        |
v1       v1       v2
Ready    Ready    NotReady
```

The new Pod isn't considered ready yet.

Once:

```text
v2 readiness = SUCCESS
```

it becomes eligible for Service traffic.

This is why you should not think:

> "Pod is Running, therefore traffic is safe."

Instead:

```text
Running ≠ Ready
```

---

# 41. Deployment Lifecycle

A real deployment looks like:

```text
Jira Change
    ↓
Production approval
    ↓
GitOps PR merged
    ↓
ArgoCD detects change
    ↓
Deployment updated
    ↓
New ReplicaSet
    ↓
New Pod created
    ↓
Startup probe
    ↓
Readiness probe
    ↓
Pod Ready
    ↓
Traffic eligible
    ↓
Old Pod terminated
    ↓
Graceful shutdown
    ↓
Next Pod
    ↓
Repeat
    ↓
All Pods v2
```

---

# 42. Monitoring During Deployment

During production deployment, monitor:

### Kubernetes

```bash
kubectl get pods -n ecommerce-prod
```

### Deployment

```bash
kubectl rollout status deployment/order-service \
  -n ecommerce-prod
```

### ReplicaSets

```bash
kubectl get rs -n ecommerce-prod
```

### Events

```bash
kubectl get events \
  -n ecommerce-prod \
  --sort-by=.lastTimestamp
```

### Logs

```bash
kubectl logs \
  -n ecommerce-prod \
  -l app.kubernetes.io/name=order-service \
  --tail=100
```

### Resources

```bash
kubectl top pods \
  -n ecommerce-prod
```

### ArgoCD

```bash
argocd app get order-service-prod
```

### Grafana

Monitor:

```text
Request rate
Error rate
Latency
CPU
Memory
Pod restarts
Availability
Database connections
RDS latency
```

---

# 43. Prometheus Metrics

For Spring Boot:

```text
/actuator/prometheus
```

Prometheus can collect metrics such as:

```text
HTTP request count
HTTP request duration
JVM memory
JVM CPU
GC activity
Thread count
Application health
```

During deployment, particularly watch:

```text
HTTP 5xx
Latency
Request rate
Pod restart count
JVM memory
```

A deployment should not be considered successful merely because:

```text
kubectl rollout status = successful
```

The application also needs to behave correctly.

---

# 44. Jira Change Integration

Our production deployment is tied to the Change Request.

Example:

```text
CHG-10452
```

Change:

```text
order-service
```

Image:

```text
8f31a2c
```

Digest:

```text
sha256:9a7b123...
```

Implementation:

```text
1. Confirm staging healthy
2. Confirm image digest
3. Confirm GitOps PR
4. Obtain approval
5. ArgoCD sync
6. Monitor rollout
7. Run smoke tests
8. Monitor metrics
9. Close change
```

If rollback is required:

```text
CHG-10452
     ↓
Rollback
     ↓
Git revert
     ↓
ArgoCD
     ↓
Previous image
     ↓
Validation
```

The change record should document the result.

---

# 45. Production Rolling Deployment Checklist

## Before Deployment

```text
[ ] Jira change approved
[ ] Correct service
[ ] Correct environment
[ ] Correct image tag
[ ] Correct image digest
[ ] ECR image exists
[ ] Trivy scan passed
[ ] Staging validation passed
[ ] GitOps PR approved
[ ] Database compatibility verified
[ ] Rollback version confirmed
[ ] Monitoring dashboards ready
[ ] On-call engineer available
```

---

## During Deployment

```text
[ ] ArgoCD sync started
[ ] New ReplicaSet created
[ ] New Pod scheduled
[ ] Startup probe succeeds
[ ] Readiness succeeds
[ ] Old Pods remain available
[ ] Error rate normal
[ ] Latency normal
[ ] No unexpected restarts
[ ] No OOMKilled
[ ] No ImagePullBackOff
```

---

## After Deployment

```text
[ ] All replicas Ready
[ ] Deployment successful
[ ] Correct image digest running
[ ] Smoke tests passed
[ ] Application metrics normal
[ ] Logs normal
[ ] Database connections normal
[ ] Queue backlog normal
[ ] No new critical alerts
[ ] Jira change updated
```

---

# 46. Production Failure Decision Tree

```text
Deployment started
       |
       v
New Pod created?
       |
   +---+---+
   |       |
  NO      YES
   |       |
Pending   Ready?
   |       |
Check     +---+---+
capacity  |       |
          NO      YES
          |       |
      Check probe  Traffic
      logs/events  eligible
          |
          v
     Application healthy?
          |
       +--+--+
       |     |
      NO    YES
       |     |
    rollback  continue
```

---

# 47. The Most Important Production Principle

Don't think of rolling deployment as simply:

```text
Change image
```

It is actually:

```text
Immutable artifact
       +
Health checks
       +
Graceful shutdown
       +
Capacity
       +
Traffic management
       +
Observability
       +
Rollback
       +
Change management
```

All of these must work together.

---

# 48. Final Architecture

After this chapter:

```text
                 GitHub
                    |
              GitHub Actions
                    |
          Build / Test / Trivy
                    |
                   ECR
                    |
             Immutable Digest
                    |
              GitOps PR
                    |
              Jira Change
                    |
                 Approval
                    |
                  ArgoCD
                    |
              Kubernetes
                    |
            Deployment
                    |
          RollingUpdate
          /           \
     Old Pods       New Pods
        |              |
        |         Startup Probe
        |              |
        |        Readiness Probe
        |              |
        +---------> Service
                       |
                      ALB
                       |
                    Clients
```

The key production guarantee we're building is:

```text
Old version remains available
        ↓
New version starts
        ↓
New version becomes Ready
        ↓
Traffic can reach new version
        ↓
Old version is removed
        ↓
Repeat
```

---

# 49. What We Have Implemented So Far

Our deployment strategy now has:

| Component           | Implementation        |
| ------------------- | --------------------- |
| Deployment          | Kubernetes Deployment |
| Strategy            | RollingUpdate         |
| Replicas            | 3                     |
| Max unavailable     | 0                     |
| Max surge           | 1                     |
| Startup probe       | Yes                   |
| Readiness probe     | Yes                   |
| Liveness probe      | Yes                   |
| Graceful shutdown   | Yes                   |
| PDB                 | Yes                   |
| Resource requests   | Yes                   |
| Resource limits     | Yes                   |
| HPA compatibility   | Yes                   |
| Immutable image     | Yes                   |
| ECR                 | Yes                   |
| GitOps              | Yes                   |
| ArgoCD              | Yes                   |
| Prometheus          | Yes                   |
| Grafana             | Yes                   |
| Jira Change         | Yes                   |
| Rollback            | GitOps revert         |
| Digest verification | Yes                   |

---

# 50. Next Chapter

After proving Rolling Deployment, we move to:

```text
82 — Blue-Green Deployment Implementation
```

There we will build:

```text
                 ALB
                  |
              Kubernetes
                  |
          +-------+-------+
          |               |
       BLUE v1         GREEN v2
       3 Pods           3 Pods
          |               |
          +-------+-------+
                  |
              Traffic
```

and implement the actual **Blue-Green switch and rollback**, including how the Kubernetes Service/ALB chooses Blue vs Green, how ArgoCD manages it, what happens to database changes, and how the Jira change is handled.
