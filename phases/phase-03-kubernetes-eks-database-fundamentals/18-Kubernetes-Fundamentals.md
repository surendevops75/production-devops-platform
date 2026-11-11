# 18-Kubernetes-Fundamentals.md

# Kubernetes Fundamentals — Production DevOps Perspective

Until now, we have built the application and delivery side:

```text
Developer
   ↓
GitHub
   ↓
CI
   ↓
Build / Test / Security
   ↓
Docker Image
   ↓
ECR
   ↓
Artifact Promotion
```

Now we enter the **runtime platform**.

Our application eventually runs here:

```text
                         AWS
                          │
                          ▼
                       VPC
                          │
                          ▼
                         EKS
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
       Node 1           Node 2           Node 3
          │               │                │
       ┌──┴──┐         ┌──┴──┐          ┌──┴──┐
       ▼     ▼         ▼     ▼          ▼     ▼
     Pod    Pod       Pod    Pod        Pod    Pod
```

Kubernetes is responsible for keeping these workloads running according to the desired state.

---

# 1. What Kubernetes Actually Does

At a high level:

> **Kubernetes continuously compares the desired state with the current state and takes actions to make the current state match the desired state.**

Suppose we declare:

```yaml
replicas: 3
```

Kubernetes understands:

```text
Desired:
3 Pods
```

If currently:

```text
Running:
2 Pods
```

Kubernetes works toward:

```text
2 → 3
```

If a pod crashes:

```text
3 Pods
   ↓
1 crashes
   ↓
2 running
   ↓
Kubernetes creates replacement
   ↓
3 running
```

This reconciliation model is fundamental.

---

# 2. Kubernetes Is Not the Application

Our architecture is:

```text
Kubernetes
     │
     ├── Runs containers
     ├── Schedules workloads
     ├── Provides networking
     ├── Handles service discovery
     ├── Performs health checks
     ├── Maintains desired replicas
     └── Provides deployment primitives
```

The application itself is still:

```text
order-service
payment-service
catalog-service
```

Kubernetes provides the runtime platform.

---

# 3. Production EKS Architecture

Our target AWS architecture:

```text
                         AWS
                          │
                    ┌─────┴─────┐
                    │    VPC    │
                    └─────┬─────┘
                          │
                         EKS
                          │
             ┌────────────┴────────────┐
             │                         │
        Control Plane              Data Plane
        managed by AWS             worker capacity
             │                         │
             │                 ┌───────┼───────┐
             │                 ▼       ▼       ▼
             │               Node    Node    Node
             │                 │       │       │
             │                Pods    Pods    Pods
             │
             └── Kubernetes API
```

Later, the EKS chapter will go deeply into exactly what AWS manages and what our DevOps/platform team manages.

---

# 4. Kubernetes Objects

The most important objects for our application are:

```text
Namespace
   │
   ├── Deployment
   │      │
   │      └── ReplicaSet
   │             │
   │             └── Pods
   │
   ├── Service
   │
   ├── ConfigMap
   │
   ├── Secret
   │
   ├── ServiceAccount
   │
   ├── HPA
   │
   ├── PDB
   │
   └── NetworkPolicy
```

Ingress/load-balancing resources sit in front of Services.

---

# 5. Namespace

A namespace provides logical separation inside a Kubernetes cluster.

Our environments might be:

```text
EKS Cluster
│
├── dev
├── qa
├── staging
├── production
└── platform
```

For example:

```text
production
```

contains:

```text
production
├── order-service
├── payment-service
├── catalog-service
├── inventory-service
└── ...
```

---

# 6. Why Namespaces?

Without namespaces:

```text
Cluster
├── order-service
├── payment-service
├── catalog-service
├── dev-order-service
├── qa-order-service
└── ...
```

This becomes difficult to manage.

With namespaces:

```text
dev/
├── order-service
└── payment-service

qa/
├── order-service
└── payment-service

production/
├── order-service
└── payment-service
```

The same application name can exist in different namespaces.

---

# 7. Namespace Is Not a Security Boundary by Itself

This is important.

Don't assume:

```text
namespace = complete isolation
```

Namespaces provide logical organization.

Security additionally requires:

```text
RBAC
NetworkPolicies
IAM
ResourceQuotas
Pod Security controls
Admission policies
```

For example:

```text
production
     │
     ├── RBAC
     ├── NetworkPolicy
     └── ResourceQuota
```

---

# 8. Production Namespace Strategy

For our project:

```text
production
```

will contain production application workloads.

Platform services may have dedicated namespaces:

```text
argocd
monitoring
logging
tracing
ingress-system
```

Example:

```text
EKS
│
├── production
├── monitoring
├── logging
├── tracing
├── argocd
└── ingress-system
```

This keeps platform components separate from business workloads.

---

# 9. Pod

The smallest deployable Kubernetes unit is the **Pod**.

A Pod usually contains one application container in our architecture.

Example:

```text
Pod
└── order-service container
```

But Kubernetes technically allows multiple containers in one Pod.

For example:

```text
Pod
├── application container
└── sidecar container
```

This is useful for certain architectures, but we should not automatically put every tool into a sidecar.

---

# 10. Pod Is Not a VM

A common beginner misunderstanding:

```text
Pod = virtual machine
```

Incorrect.

Conceptually:

```text
VM
 └── Operating system
      └── Application

Pod
 └── Container(s)
```

Containers share the underlying node's kernel.

---

# 11. Pod Lifecycle

A pod may move through states such as:

```text
Pending
   ↓
Running
   ↓
Succeeded / Failed
```

For a long-running service:

```text
Pending
   ↓
Container starts
   ↓
Readiness check
   ↓
Ready
   ↓
Serving traffic
```

If startup fails:

```text
Pending
   ↓
Container fails
   ↓
Restart
   ↓
CrashLoopBackOff
```

---

# 12. CrashLoopBackOff

This is one of the most common production issues.

Example:

```text
order-service Pod
       ↓
Container starts
       ↓
Application crashes
       ↓
Kubernetes restarts
       ↓
Application crashes
       ↓
Kubernetes restarts
       ↓
Backoff increases
       ↓
CrashLoopBackOff
```

Troubleshooting:

```bash
kubectl get pods -n production
```

Then:

```bash
kubectl describe pod <pod-name> -n production
```

Then:

```bash
kubectl logs <pod-name> -n production
```

If the container restarted:

```bash
kubectl logs <pod-name> \
  -n production \
  --previous
```

`--previous` is extremely useful when the current container has already restarted.

---

# 13. Deployment

We don't normally create individual application Pods manually.

Instead:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service
  namespace: production

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f3a21c9
```

---

# 14. Why Deployment?

Suppose we need:

```text
3 replicas
```

We don't want an engineer manually creating:

```text
pod-1
pod-2
pod-3
```

Instead:

```text
Deployment
   ↓
"I want 3 replicas"
```

Kubernetes handles the workload lifecycle.

---

# 15. Deployment → ReplicaSet → Pods

This hierarchy is important.

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ├── Pod
    ├── Pod
    └── Pod
```

The Deployment manages the desired rollout state.

The ReplicaSet maintains the requested number of matching Pods.

---

# 16. Why ReplicaSet Exists

Suppose:

```text
Deployment
replicas: 3
```

ReplicaSet ensures:

```text
Desired = 3
Current = 3
```

If:

```text
Current = 2
```

it creates another Pod.

If:

```text
Current = 4
```

it removes an excess Pod as needed.

---

# 17. Rolling Update

Suppose production is running:

```text
Version A
├── Pod A
├── Pod A
└── Pod A
```

We deploy version B.

Kubernetes can perform a rolling update:

```text
A A A
  ↓
A A B
  ↓
A B B
  ↓
B B B
```

This avoids taking all instances down simultaneously.

---

# 18. Desired State

The Deployment says:

```yaml
replicas: 3
```

and:

```yaml
image: order-service:8f3a21c9
```

This is the desired state.

Kubernetes continuously works toward it.

This is why Kubernetes is called a **declarative system**.

---

# 19. Declarative vs Imperative

### Imperative

You tell the system:

```text
"Start this container."
```

### Declarative

You tell Kubernetes:

```text
"I want 3 replicas of this application running."
```

Kubernetes figures out the actions required.

Our GitOps architecture strongly prefers declarative configuration.

---

# 20. Service

Pods are temporary.

Their IP addresses can change.

Suppose:

```text
order-service
├── Pod A → 10.0.1.10
├── Pod B → 10.0.1.11
└── Pod C → 10.0.1.12
```

Pod A crashes.

Replacement:

```text
Pod D → 10.0.1.20
```

Clients should not need to know this.

That's where a Kubernetes Service comes in.

---

# 21. Service

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service
  namespace: production

spec:
  selector:
    app: order-service

  ports:
    - name: http
      port: 8080
      targetPort: http

  type: ClusterIP
```

The Service provides a stable network identity.

---

# 22. Service Flow

```text
Other Service
      │
      ▼
order-service:8080
      │
      ▼
 Kubernetes Service
      │
      ├── Pod A
      ├── Pod B
      └── Pod C
```

The Service selects Pods using:

```yaml
selector:
  app: order-service
```

---

# 23. ClusterIP

Our internal microservices generally use:

```text
type: ClusterIP
```

This means:

> Accessible internally within the Kubernetes networking environment.

For example:

```text
payment-service
       │
       ▼
order-service.production.svc.cluster.local
       │
       ▼
Order Pods
```

---

# 24. Kubernetes DNS

Kubernetes provides service discovery.

A Service can be addressed through DNS.

For example:

```text
order-service.production.svc.cluster.local
```

Conceptually:

```text
order-service
     ↓
production namespace
     ↓
svc
     ↓
cluster.local
```

Applications therefore don't need to hardcode individual Pod IPs.

---

# 25. Microservice Communication

Our platform:

```text
payment-service
      │
      ▼
order-service
      │
      ▼
inventory-service
```

The application should generally communicate using stable service names:

```text
http://order-service:8080
```

when appropriate within the same namespace.

Across namespaces:

```text
http://order-service.production.svc.cluster.local:8080
```

---

# 26. Service Types

Important Service types:

```text
ClusterIP
NodePort
LoadBalancer
```

For our architecture:

```text
Internal microservices
        ↓
ClusterIP
```

External traffic is handled through our ingress/load-balancing architecture rather than exposing every microservice individually.

---

# 27. Why Not Give Every Service a Public IP?

Bad:

```text
Internet
 │
 ├── order-service
 ├── payment-service
 ├── inventory-service
 ├── catalog-service
 └── ...
```

This increases:

* attack surface
* complexity
* cost
* security exposure

Better:

```text
Internet
   ↓
CloudFront / WAF
   ↓
ALB
   ↓
Ingress
   ↓
Services
   ↓
Pods
```

Only required entry points are externally exposed.

---

# 28. ConfigMap

Applications need configuration.

Example:

```text
LOG_LEVEL=INFO
SPRING_PROFILES_ACTIVE=prod
```

A ConfigMap can hold non-sensitive configuration.

Example:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: order-service-config
  namespace: production

data:
  LOG_LEVEL: "INFO"
  SPRING_PROFILES_ACTIVE: "prod"
```

---

# 29. What Should NOT Go Into ConfigMap?

Never put sensitive credentials into ConfigMap.

Bad:

```yaml
data:
  DB_PASSWORD: "super-secret-password"
```

Instead use an appropriate secret-management mechanism.

For our AWS architecture:

```text
AWS Secrets Manager
       ↓
EKS
       ↓
Application
```

We'll cover this in detail later.

---

# 30. Kubernetes Secret

Kubernetes has a Secret object:

```yaml
apiVersion: v1
kind: Secret
```

But remember:

> A Kubernetes Secret is not automatically equivalent to a fully managed enterprise secret-management system.

For our production architecture, we want AWS Secrets Manager integrated appropriately with EKS.

---

# 31. ServiceAccount

Every Pod runs under a Kubernetes identity represented by a ServiceAccount.

Example:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: order-service
  namespace: production
```

Then:

```yaml
spec:
  serviceAccountName: order-service
```

This becomes especially important for AWS permissions.

---

# 32. Pod → AWS

Our Order Service might need:

```text
SQS
Secrets Manager
S3
```

We do not want:

```text
Pod
 ↓
AWS Access Key
 ↓
Secret stored in environment
```

Instead:

```text
Order Pod
    │
    ▼
Kubernetes ServiceAccount / EKS identity mechanism
    │
    ▼
AWS IAM Role
    │
    ├── SQS
    ├── Secrets Manager
    └── S3
```

For modern EKS architectures, we'll use EKS Pod Identity where appropriate.

---

# 33. Resource Requests

Every production application should have appropriate resource requests.

Example:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

This tells Kubernetes approximately:

```text
"I need at least this much schedulable capacity."
```

---

# 34. Resource Limits

Example:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 1Gi
```

This establishes a maximum boundary for the container.

Memory limits are especially important because memory exhaustion can result in:

```text
OOMKilled
```

---

# 35. CPU vs Memory

CPU:

```text
250m = 0.25 CPU
```

Memory:

```text
512Mi
```

For example:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 500m
    memory: 1Gi
```

These values are examples.

We must determine actual values from:

* application behavior
* load testing
* production metrics
* JVM characteristics
* traffic patterns

---

# 36. Why Requests Matter for Scheduling

Suppose a node has:

```text
4 CPU
```

and several Pods request:

```text
1 CPU
```

Kubernetes uses requests when deciding whether the Pod can fit.

Conceptually:

```text
Node
4 CPU
│
├── Pod A → 1 CPU request
├── Pod B → 1 CPU request
├── Pod C → 1 CPU request
└── Pod D → 1 CPU request
```

Now there is no additional requested CPU capacity.

---

# 37. Why Limits Matter

Suppose:

```text
request = 250m
limit = 500m
```

The application can normally consume its requested resources and may use more CPU when available, subject to its limit and node conditions.

Memory behaves differently from CPU when a container approaches its limit.

This is why blindly copying:

```text
requests = limits
```

or:

```text
limits = huge
```

is not a production strategy.

---

# 38. Probes

Production applications need health checks.

Our Spring Boot application exposes:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Kubernetes can use these.

---

# 39. Liveness Probe

Liveness answers:

> Is this container alive enough to continue running?

Example:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: http

  initialDelaySeconds: 30
  periodSeconds: 20
```

If the container repeatedly fails its liveness check, Kubernetes may restart it.

---

# 40. Readiness Probe

Readiness answers:

> Should this Pod receive traffic right now?

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: http

  initialDelaySeconds: 10
  periodSeconds: 10
```

This is extremely important during deployments.

---

# 41. Liveness vs Readiness

Think:

```text
Liveness
   ↓
"Should Kubernetes restart me?"
```

Readiness:

```text
"Should traffic be sent to me?"
```

Example:

```text
Application starting
      ↓
Pod alive
      ↓
Readiness = false
      ↓
No application traffic
      ↓
Application initializes
      ↓
Readiness = true
      ↓
Traffic allowed
```

---

# 42. Startup Probe

For slow-starting applications, a startup probe can be useful.

Conceptually:

```text
Startup probe
      ↓
Application initialization
      ↓
Successful startup
      ↓
Liveness/readiness take over
```

This prevents an application that needs substantial startup time from being killed prematurely by a liveness probe.

This is particularly relevant for some JVM applications.

---

# 43. Deployment Without Readiness

Imagine:

```text
Old version
A A A
```

New version starts:

```text
A A B
```

But B hasn't finished initializing.

Without proper readiness handling, traffic could reach B too early.

With readiness:

```text
A A B
    │
    └── Not Ready
```

Traffic remains with healthy ready Pods.

---

# 44. Horizontal Pod Autoscaler

Our application may receive variable traffic.

For example:

```text
Normal:
3 Pods

Peak:
10 Pods
```

HPA can scale the Deployment.

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: order-service
  namespace: production

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  minReplicas: 3
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

We'll study autoscaling in much greater detail later.

---

# 45. Pod Disruption Budget

Production applications should tolerate planned disruptions.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: order-service
  namespace: production

spec:
  minAvailable: 2

  selector:
    matchLabels:
      app: order-service
```

This helps protect availability during certain voluntary disruptions.

---

# 46. PDB Does Not Protect Against Everything

Important distinction.

PDB helps with:

```text
voluntary disruptions
```

such as certain maintenance/drain operations.

It doesn't magically protect against:

```text
node crash
AWS outage
application crash
OOMKill
```

Those require other resilience mechanisms.

---

# 47. Node

Pods run on worker capacity.

Conceptually:

```text
EKS
│
├── Node 1
│    ├── Pod
│    └── Pod
│
├── Node 2
│    ├── Pod
│    └── Pod
│
└── Node 3
     ├── Pod
     └── Pod
```

In modern EKS environments, worker capacity may be managed through:

* managed node groups
* Karpenter
* other supported capacity mechanisms

We'll cover this in the EKS chapter.

---

# 48. Scheduler

The Kubernetes scheduler decides where unscheduled Pods should run.

Conceptually:

```text
New Pod
   │
   ▼
Scheduler
   │
   ├── CPU available?
   ├── Memory available?
   ├── Node constraints?
   ├── Taints/tolerations?
   ├── Affinity?
   └── topology constraints?
   │
   ▼
Selected Node
```

---

# 49. Why a Pod May Stay Pending

Production troubleshooting often starts with:

```bash
kubectl get pods -n production
```

You see:

```text
order-service-abc123   Pending
```

Then:

```bash
kubectl describe pod order-service-abc123 -n production
```

Events may reveal:

```text
Insufficient cpu
```

or:

```text
Insufficient memory
```

or:

```text
No nodes match selector
```

or:

```text
untolerated taint
```

The solution isn't always "restart the Pod."

---

# 50. Node Affinity

We may want workloads on specific nodes.

For example:

```text
General workloads
        ↓
general node pool

High-memory workloads
        ↓
memory-optimized node pool
```

Affinity/selector mechanisms help express placement requirements.

---

# 51. Taints and Tolerations

Suppose we have dedicated nodes:

```text
payment workloads
```

We can taint nodes so ordinary workloads don't land there.

Conceptually:

```text
Node
taint:
dedicated=payments
```

Only Pods with an appropriate toleration can use it.

This is useful for workload isolation.

---

# 52. Availability Zones

Our production EKS cluster spans multiple AZs.

Example:

```text
Region: ap-south-1

AZ-A
 ├── Node
 └── Pods

AZ-B
 ├── Node
 └── Pods

AZ-C
 ├── Node
 └── Pods
```

We don't want all replicas in one AZ.

---

# 53. Why Multiple AZs?

Suppose:

```text
order-service
Pod A → AZ-A
Pod B → AZ-B
Pod C → AZ-C
```

If AZ-A has a problem:

```text
Pod A → unavailable
```

we still have:

```text
Pod B
Pod C
```

This improves resilience.

---

# 54. Topology Spread

Modern Kubernetes provides mechanisms to distribute replicas across topology domains.

Conceptually:

```text
order-service
│
├── AZ-A → 1 Pod
├── AZ-B → 1 Pod
└── AZ-C → 1 Pod
```

This is preferable to accidentally scheduling all replicas onto one node/AZ.

---

# 55. NetworkPolicy

By default, Kubernetes networking can allow broad connectivity depending on the CNI and policies configured.

Production should explicitly define required communication.

Example architecture:

```text
Internet
   ↓
Ingress
   ↓
order-service
   ↓
inventory-service
```

Maybe:

```text
payment-service
```

doesn't need to talk directly to:

```text
reporting-service
```

NetworkPolicy can restrict unnecessary communication.

---

# 56. Zero-Trust-Oriented Microservice Networking

Instead of:

```text
Every service
   ↓
Every other service
```

we want:

```text
order-service
   ├── payment-service → ALLOW
   ├── inventory-service → ALLOW
   └── reporting-service → DENY
```

This reduces lateral movement risk.

---

# 57. Ingress

External traffic needs an entry point.

Our architecture:

```text
Internet
   │
   ▼
Route 53
   │
   ▼
CloudFront / WAF
   │
   ▼
AWS Load Balancer
   │
   ▼
Kubernetes Ingress
   │
   ▼
Service
   │
   ▼
Pods
```

The exact AWS implementation will be covered in the EKS/ingress chapters.

---

# 58. ConfigMap + Secret + ServiceAccount

A real application Pod might conceptually look like:

```yaml
spec:
  serviceAccountName: order-service

  containers:
    - name: order-service
      image: ...

      envFrom:
        - configMapRef:
            name: order-service-config

      resources:
        requests:
          cpu: 250m
          memory: 512Mi

        limits:
          cpu: 500m
          memory: 1Gi
```

Secrets should be integrated through the chosen production secret-management mechanism.

---

# 59. Production Deployment Object

Our eventual Helm-generated Deployment will look roughly like:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service
  namespace: production

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      serviceAccountName: order-service

      containers:
        - name: order-service

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f3a21c9

          ports:
            - name: http
              containerPort: 8080

          resources:
            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: 500m
              memory: 1Gi

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
```

This is an intentionally simplified production foundation.

---

# 60. What Actually Happens During Deployment?

Suppose GitOps changes:

```text
image digest:
OLD
```

to:

```text
image digest:
NEW
```

ArgoCD notices:

```text
Git desired state
       ↓
ArgoCD
       ↓
Kubernetes API
```

The Deployment changes.

Kubernetes creates new Pods.

Then:

```text
New Pod
   ↓
Container starts
   ↓
Startup
   ↓
Readiness probe
   ↓
Ready
   ↓
Service includes Pod
   ↓
Traffic
```

Old Pods are gradually removed according to the rollout strategy.

---

# 61. Kubernetes API Server

The API server is the main interface to the Kubernetes control plane.

Commands such as:

```bash
kubectl get pods
```

ultimately communicate with the Kubernetes API.

Conceptually:

```text
kubectl
   │
   ▼
API Server
   │
   ├── Deployment
   ├── Service
   ├── Pod
   ├── ConfigMap
   └── ...
```

In EKS, AWS manages the Kubernetes control plane infrastructure.

---

# 62. `kubectl`

`kubectl` is the primary Kubernetes CLI.

Common commands:

```bash
kubectl get pods -n production
```

```bash
kubectl get deployments -n production
```

```bash
kubectl get services -n production
```

```bash
kubectl get nodes
```

```bash
kubectl describe pod <pod> -n production
```

```bash
kubectl logs <pod> -n production
```

---

# 63. First Production Troubleshooting Command

When someone says:

> "Order service is down."

Don't immediately restart it.

Start:

```bash
kubectl get pods -n production
```

Then look for:

```text
Running
Pending
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
OOMKilled
NotReady
```

Each tells a different story.

---

# 64. ImagePullBackOff

Example:

```text
order-service-xyz   ImagePullBackOff
```

Possible causes:

```text
Wrong image name
Wrong tag
Image doesn't exist
ECR access problem
IAM problem
Network problem
Registry authentication issue
```

Check:

```bash
kubectl describe pod <pod> -n production
```

Look at Events.

---

# 65. OOMKilled

If:

```text
Reason: OOMKilled
```

the container exceeded its memory limit or otherwise encountered memory pressure conditions leading to termination.

Investigate:

```text
Memory usage
Heap
JVM settings
Container limit
Traffic
Application behavior
Leaks
```

Don't simply increase memory indefinitely.

---

# 66. Service Has No Traffic

Suppose:

```text
order-service
```

exists, but requests fail.

Check:

```bash
kubectl get svc order-service -n production
```

Then endpoints:

```bash
kubectl get endpoints order-service -n production
```

or the appropriate EndpointSlice resources.

If the Service has no healthy backend endpoints, investigate:

```text
Selector
Labels
Readiness
Pod state
```

---

# 67. Selector Mismatch

Service:

```yaml
selector:
  app: order-service
```

Pod:

```yaml
labels:
  app: order
```

These don't match.

Result:

```text
Service
   ↓
No matching Pods
```

Application may be running, but traffic never reaches it.

This is a classic Kubernetes troubleshooting issue.

---

# 68. Readiness Failure

Suppose:

```text
Pod = Running
```

but:

```text
Ready = 0/1
```

This means the container process can be running while Kubernetes considers it unsuitable for traffic.

Check:

```bash
kubectl describe pod <pod> -n production
```

Then inspect the readiness probe.

Possible cause:

```text
Application
   ↓
DB connection failed
   ↓
Readiness endpoint unhealthy
   ↓
Pod not ready
```

---

# 69. Running ≠ Ready

This distinction is critical.

```text
Running
```

means the container is running.

```text
Ready
```

means it is currently considered eligible to receive traffic.

Therefore:

```text
1/1 Running
```

is different from:

```text
0/1 Running
```

and the latter may indicate readiness failure.

---

# 70. Production Request Path

Our eventual request path:

```text
User
 │
 ▼
Route 53
 │
 ▼
CloudFront
 │
 ▼
WAF
 │
 ▼
ALB
 │
 ▼
Ingress
 │
 ▼
Service
 │
 ▼
Pod
 │
 ▼
Spring Boot
 │
 ▼
Database / Redis / SQS
```

This is the chain we'll troubleshoot later layer by layer.

---

# 71. Kubernetes and Observability

Kubernetes itself doesn't replace observability.

Our production platform will eventually include:

```text
Metrics
 ├── Prometheus
 └── Grafana

Logs
 ├── Fluent Bit
 ├── Logstash
 ├── Elasticsearch
 └── Kibana

Traces
 ├── OpenTelemetry
 └── Jaeger
```

So when an application fails:

```text
Kubernetes
   +
Metrics
   +
Logs
   +
Traces
```

give us the complete picture.

---

# 72. Kubernetes Doesn't Mean Everything Is Automatically Reliable

A common misconception:

> "We use Kubernetes, therefore our application is highly available."

Not automatically.

You need:

```text
Multiple replicas
+
Multiple AZs
+
Readiness probes
+
Resource requests
+
PDB
+
Autoscaling
+
Proper deployment strategy
+
Application resilience
+
Database HA
+
Observability
```

Kubernetes is a platform, not a magic reliability switch.

---

# 73. Production Checklist for a Microservice

For `order-service`:

```text
[ ] Deployment
[ ] Multiple replicas
[ ] Service
[ ] Readiness probe
[ ] Liveness probe
[ ] Startup probe if required
[ ] Resource requests
[ ] Resource limits
[ ] ServiceAccount
[ ] IAM integration
[ ] ConfigMap
[ ] Secret integration
[ ] PDB
[ ] NetworkPolicy
[ ] HPA
[ ] Multi-AZ placement
[ ] Pod topology considerations
[ ] Logging
[ ] Metrics
[ ] Tracing
[ ] SecurityContext
```

---

# 74. Complete Order-Service Runtime

Our service will eventually look like:

```text
                         EKS
                          │
                    production NS
                          │
                 ┌────────┴────────┐
                 │   Deployment    │
                 └────────┬────────┘
                          │
                    ReplicaSet
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
          Pod A         Pod B         Pod C
            │             │             │
       order-service order-service order-service
            │             │             │
            └─────────────┼─────────────┘
                          │
                       Service
                          │
                          ▼
                    Internal traffic

External:
Internet
   ↓
Route 53
   ↓
CloudFront/WAF
   ↓
ALB
   ↓
Ingress
   ↓
Service
```

---

# 75. The Most Important Kubernetes Mental Model

Memorize the **relationship**, not individual YAML fields:

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ▼
Pods
    │
    ▼
Containers
```

and:

```text
Service
    │
    ▼
selects Pods
```

and:

```text
Ingress
    │
    ▼
Service
    │
    ▼
Pods
```

and:

```text
HPA
    │
    ▼
Deployment
    │
    ▼
more/fewer Pods
```

and:

```text
PDB
    │
    ▼
protects availability during certain voluntary disruptions
```

---

# 76. Production Request + Deployment Model

Put everything together:

```text
                     GIT
                      │
                      ▼
                  GitOps Repo
                      │
                      ▼
                    ArgoCD
                      │
                      ▼
                 Kubernetes API
                      │
                      ▼
                 Deployment
                      │
                      ▼
                  ReplicaSet
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
            Pod      Pod      Pod
             │        │        │
             └────────┼────────┘
                      │
                   Service
                      │
                      ▼
                   Ingress
                      │
                      ▼
                     ALB
                      │
                      ▼
                 WAF/CloudFront
                      │
                      ▼
                    Users
```

This is the runtime foundation for everything coming next.

---

# 77. What We Have Learned

At this point, you should understand:

```text
Pod
 ↓
smallest deployable workload unit

Deployment
 ↓
manages application rollout and desired replicas

ReplicaSet
 ↓
maintains Pod count

Service
 ↓
stable network identity + load balancing to matching Pods

Namespace
 ↓
logical isolation/grouping

ConfigMap
 ↓
non-sensitive configuration

Secret
 ↓
sensitive configuration mechanism
   (with production secret-management integration)

ServiceAccount
 ↓
workload identity

HPA
 ↓
horizontal scaling

PDB
 ↓
availability during voluntary disruption

Ingress
 ↓
HTTP entry/routing toward Services
```

---

# 78. Production Principle

Don't think:

> "Kubernetes runs my Docker image."

Think:

> **"Kubernetes manages the desired state, lifecycle, scheduling, networking, health, and scaling of my workloads across a distributed cluster."**

Then EKS adds AWS-specific capabilities around that Kubernetes platform.

---

# 79. Next Chapter

## `19-EKS.md`

Now we'll move from generic Kubernetes into **actual AWS EKS production architecture**.

We'll cover:

```text
AWS VPC
   │
   ├── Public Subnets
   └── Private Subnets
          │
          ▼
         EKS
          │
          ├── AWS-managed Control Plane
          │
          └── Data Plane
               ├── Managed Node Groups
               ├── Karpenter
               ├── EC2 instances
               └── Pods
```

And we'll go into the production details you specifically asked for:

* how EKS is actually provisioned
* what AWS manages vs what DevOps manages
* EKS cluster creation with Terraform
* private/public endpoint decisions
* IAM and EKS access
* node groups
* Karpenter
* instance sizing
* AMIs
* bootstrap
* security groups
* pod networking
* AWS VPC CNI
* CoreDNS
* kube-proxy
* EBS CSI
* load balancer integration
* EKS upgrades
* multi-AZ architecture
* production hardening
* monitoring
* troubleshooting nodes and Pods
* and **exactly where each component is installed/running in a real production EKS environment**.
