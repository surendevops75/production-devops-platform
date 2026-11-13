# CHAPTER 206 — ROLLING AND CANARY DEPLOYMENT

**Filename:** `206-Rolling-Canary-Deployment.md`

---

# 1. Chapter Objective

In Chapter 205, we deployed the approved application release into Production.

Now we will learn how to release new application versions safely without causing unnecessary downtime or affecting all customers at once.

In Production environments, deploying a new version is not simply changing an image tag.

We must control:

- How many old pods are terminated.
- How many new pods are created.
- How traffic reaches new pods.
- How application readiness is verified.
- How errors and latency are monitored.
- How deployment failures are detected.
- How traffic is shifted gradually.
- How to stop or roll back a failed release.

We will implement:

1. Kubernetes Rolling Updates.
2. Rolling deployment configuration.
3. Zero-downtime deployment requirements.
4. Readiness and liveness probes.
5. Startup probes.
6. PodDisruptionBudgets.
7. Deployment revision management.
8. Rolling deployment execution.
9. Failed rollout handling.
10. Canary deployment architecture.
11. Traffic splitting.
12. AWS ALB weighted target groups.
13. Progressive traffic shifting.
14. Prometheus-based validation.
15. Argo Rollouts.
16. Automated promotion.
17. Canary rollback.
18. Production failure simulations.
19. Deployment runbooks.

# 2. Deployment Strategies Overview

There are several ways to deploy a new application version.

| Strategy | Description | Main advantage |
|---|---|---|
| Recreate | Stop old version and start new version | Simple |
| Rolling Update | Gradually replace old pods | Controlled replacement |
| Blue-Green | Maintain two environments and switch traffic | Fast traffic switch |
| Canary | Send a small percentage of traffic to new version | Limits initial exposure |

In this chapter, we focus on Rolling Updates and Canary Deployments.

# 3. Rolling Deployment Architecture

A Kubernetes Rolling Update gradually replaces old application pods with new application pods.

Example:

```text
BEFORE DEPLOYMENT

Version: v1

Pod-1 (v1)     Pod-2 (v1)     Pod-3 (v1)
     |              |              |
     +--------------+--------------+
                    |
                    v
                 Service
                    |
                    v
                  Users


DURING ROLLING UPDATE

Pod-1 (v2)     Pod-2 (v1)     Pod-3 (v1)
     |              |              |
     +--------------+--------------+
                    |
                    v
                 Service
                    |
                    v
                  Users


NEXT STEP

Pod-1 (v2)     Pod-2 (v2)     Pod-3 (v1)
     |              |              |
     +--------------+--------------+
                    |
                    v
                 Service
                    |
                    v
                  Users


AFTER DEPLOYMENT

Pod-1 (v2)     Pod-2 (v2)     Pod-3 (v2)
     |              |              |
     +--------------+--------------+
                    |
                    v
                 Service
                    |
                    v
                  Users
```

The Service routes traffic to eligible endpoints.

Kubernetes replaces pods progressively according to the deployment strategy and available capacity.

# 4. Kubernetes RollingUpdate Configuration

Create or update:

`charts/user/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: user
  namespace: production

spec:
  replicas: 3

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
      serviceAccountName: user-sa

      containers:
        - name: user

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:APPROVED_DIGEST

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: 250m
              memory: 256Mi

            limits:
              cpu: "1"
              memory: 512Mi

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080

            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080

            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3

          terminationMessagePolicy: File
```

The example assumes that the application implements these health endpoints. Use the real paths and ports from your application.

## 4.1 Understand maxSurge

```yaml
maxSurge: 1
```

This allows Kubernetes to create one additional pod above the desired replica count during the rollout.

For three replicas:

```text
Desired replicas: 3

Maximum temporary replicas: 4
```

## 4.2 Understand maxUnavailable

```yaml
maxUnavailable: 0
```

This prevents the Deployment controller from intentionally reducing the number of available replicas below the desired number during the rollout.

For three replicas:

```text
Desired available replicas: 3

Minimum available replicas during rollout: 3
```

This requires sufficient capacity to schedule the additional pod.

It does not guarantee zero downtime by itself. Application health, traffic routing, dependency compatibility, and graceful termination must also work correctly.

# 5. Understand RollingUpdate Parameters

Consider this configuration:

```yaml
replicas: 6

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1
```

The desired replica count is six.

Kubernetes may temporarily create up to eight replicas and may allow the number of available replicas to fall to five during the rollout.

| Parameter | Meaning |
|---|---|
| replicas | Desired pod count |
| maxSurge | Additional pods allowed |
| maxUnavailable | Allowed unavailable replicas |

You can configure these values as integers or percentages.

Example:

```yaml
maxSurge: 25%
maxUnavailable: 25%
```

For six replicas, Kubernetes calculates the effective values according to the Deployment rolling-update rules.

For `maxSurge`, percentage values are rounded up. For `maxUnavailable`, percentage values are rounded down.

# 6. Zero-Downtime Deployment Requirements

A rolling update does not automatically guarantee zero downtime.

We need to design the application and Kubernetes resources correctly.

Important requirements:

1. Readiness probes must represent actual traffic readiness.
2. Liveness probes must detect unrecoverable application failures.
3. Startup probes should protect slow-starting applications.
4. Enough cluster capacity must be available.
5. Graceful shutdown must be implemented.
6. Services must have healthy endpoints.
7. Database changes must remain backward compatible during the rollout.
8. Application versions must support the same shared contracts.
9. Load balancer health checks must be correct.
10. External dependencies must remain available.

# 7. Readiness Probe

A readiness probe tells Kubernetes whether a pod is ready to receive traffic.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080

  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Suppose the application starts but cannot connect to its required database.

The process may still be running, but it should not receive normal application traffic if database connectivity is essential to serving requests.

A suitable readiness endpoint can report the application as not ready until its required dependencies are available.

Check pod readiness:

```bash
kubectl get pods -n production
```

Check details:

```bash
kubectl describe pod <POD_NAME> -n production
```

Check endpoints:

```bash
kubectl get endpointslices -n production
```

A pod that is not Ready should not be included as a normal ready backend endpoint for its Service.

# 8. Liveness Probe

A liveness probe determines whether the container should be restarted.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080

  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

Example situation:

- Application starts successfully.
- An internal deadlock occurs.
- The process remains running.
- The application cannot handle requests.

A correctly designed liveness probe can detect the failure and allow Kubernetes to restart the container.

**Important:** Do not make liveness probes depend on every external service. A temporary database outage should not cause every application pod to restart repeatedly.

# 9. Startup Probe

Some applications take a long time to initialize.

For example:

- Spring Boot application.
- Java application loading dependencies.
- Application performing initialization.
- Application loading configuration.

A startup probe gives the application time to initialize before liveness and readiness checks begin.

Example:

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: 8080

  periodSeconds: 10
  failureThreshold: 30
```

This example allows approximately five minutes for startup probe failures before the container is restarted.

When a startup probe is configured, Kubernetes does not begin executing the liveness and readiness probes until startup succeeds.

# 10. Graceful Shutdown

During a rolling update, Kubernetes eventually terminates old pods.

The application should finish active requests before exiting.

Example deployment configuration:

```yaml
spec:
  terminationGracePeriodSeconds: 60

  containers:
    - name: user

      image: example-image

      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - sleep 10
```

This is an illustrative drain period, not a universal setting.

The application should:

1. Receive the termination signal.
2. Stop accepting new work.
3. Finish or safely terminate in-flight requests.
4. Close database connections.
5. Stop background workers.
6. Exit before the grace period expires.

The `preStop` hook consumes part of the termination grace period.

The application itself must implement proper SIGTERM handling. A sleep command alone does not implement graceful shutdown.

# 11. PodDisruptionBudget

A PodDisruptionBudget helps maintain a minimum availability level during supported voluntary disruptions.

Example:

`charts/user/templates/pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: user-pdb
  namespace: production

spec:
  minAvailable: 2

  selector:
    matchLabels:
      app: user
```

For three replicas, this requires at least two available pods during disruptions covered by the PDB.

Check:

```bash
kubectl get pdb -n production
```

Describe:

```bash
kubectl describe pdb user-pdb -n production
```

Important distinction:

A PDB primarily controls voluntary disruptions using the Kubernetes eviction mechanism. It does not prevent application crashes, node failures, or all Deployment rollout behavior.

Do not treat a PDB as a replacement for Deployment rolling-update settings.

# 12. Verify Deployment Strategy

Check the existing Deployment:

```bash
kubectl get deployment user \
  -n production \
  -o yaml
```

Check rollout history:

```bash
kubectl rollout history deployment/user \
  -n production
```

Check the current rollout:

```bash
kubectl rollout status deployment/user \
  -n production
```

Check events:

```bash
kubectl get events \
  -n production \
  --sort-by=.metadata.creationTimestamp
```

# 13. Execute a Rolling Deployment

Assume:

```text
Service: user
Namespace: production
Current version: v1
New version: v2
Replicas: 3
```

The new version must already be built, security-scanned, tested, and approved.

In our GitOps workflow, update the production image digest in the GitOps repository.

Example:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user
  digest: "sha256:NEW_APPROVED_DIGEST"
```

Create the GitOps pull request.

After review and approval, merge it.

Argo CD reconciles the desired state.

Monitor the rollout:

```bash
kubectl rollout status deployment/user \
  -n production \
  --timeout=300s
```

Check pods:

```bash
kubectl get pods \
  -n production \
  -l app=user \
  -o wide
```

Check ReplicaSets:

```bash
kubectl get replicasets \
  -n production \
  -l app=user
```

Check image:

```bash
kubectl get deployment user \
  -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'

echo
```

Confirm the actual running image digest.

# 14. Monitor a Rolling Deployment in Real Time

Use:

```bash
kubectl get pods \
  -n production \
  -l app=user \
  -w
```

You may observe:

```text
user-old-abc     Running
user-old-def     Running
user-old-ghi     Running

user-new-jkl     ContainerCreating
user-old-abc     Terminating
user-new-jkl     Running
user-new-jkl     Ready

user-new-mno     ContainerCreating
user-old-def     Terminating
user-new-mno     Running
user-new-mno     Ready
```

The exact sequence depends on readiness, capacity, and Deployment controller behavior.

# 15. Validate Traffic During Rolling Deployment

Check Service:

```bash
kubectl get service user -n production
```

Check EndpointSlices:

```bash
kubectl get endpointslices \
  -n production \
  -l kubernetes.io/service-name=user
```

Check the application endpoint:

```bash
curl -f https://roboshop.example.com/health
```

Check response status:

```bash
curl -s -o /dev/null \
  -w "%{http_code}\n" \
  https://roboshop.example.com/health
```

Monitor:

- HTTP 5xx errors.
- Request latency.
- Pod restarts.
- CPU usage.
- Memory usage.
- Database connections.
- Message processing errors.

A successful Kubernetes rollout is necessary but not sufficient for business-level release success.

# 16. Failed Rolling Deployment

Consider this situation:

```text
Old version: v1
New version: v2

Deployment started.

New pods are created.

New application pods fail readiness checks.

Old pods remain available.

Deployment stops progressing.
```

Check:

```bash
kubectl rollout status deployment/user \
  -n production
```

Check:

```bash
kubectl describe deployment user \
  -n production
```

Check pods:

```bash
kubectl get pods -n production
```

Check logs:

```bash
kubectl logs <NEW_POD_NAME> \
  -n production \
  --previous
```

Potential reasons:

- Invalid configuration.
- Incorrect image.
- Missing secret.
- Database connection failure.
- Application startup failure.
- Probe path mismatch.
- Insufficient resources.
- Dependency incompatibility.

# 17. Rolling Deployment Rollback

In a GitOps-managed environment, Git is the source of truth.

The preferred rollback process is:

1. Identify the failed release.
2. Identify the last known-good GitOps commit.
3. Create a controlled rollback change.
4. Restore the previously approved image digest and configuration.
5. Obtain required approval.
6. Merge the rollback change.
7. Allow Argo CD to reconcile.
8. Validate the restored version.

Kubernetes also provides rollout history:

```bash
kubectl rollout history deployment/user \
  -n production
```

Inspect a revision:

```bash
kubectl rollout history deployment/user \
  -n production \
  --revision=2
```

Direct rollback command:

```bash
kubectl rollout undo deployment/user \
  -n production
```

Use this only when authorized and coordinated with GitOps.

A direct `kubectl rollout undo` changes the live cluster without updating Git. Argo CD may subsequently reconcile it back to the Git-defined version.

For a GitOps-controlled Production environment, restore the known-good state in Git rather than leaving the live cluster and Git repository inconsistent.

# 18. Rolling Deployment Limitations

Rolling deployment is useful, but it has limitations.

Consider:

```text
Old version: v1
New version: v2
```

During the rollout, both versions may serve traffic simultaneously.

This can create problems if:

- API contracts are incompatible.
- Database schemas are changed incompatibly.
- Message formats are incompatible.
- Session formats change.
- Shared cache formats are incompatible.

Use backward-compatible deployment practices.

For database changes, a common pattern is:

```text
EXPAND
  |
  v
Deploy backward-compatible schema changes
  |
  v
Deploy new application version
  |
  v
Validate all application instances
  |
  v
MIGRATE DATA
  |
  v
REMOVE OLD SCHEMA ONLY AFTER SAFE TRANSITION
```

Do not remove a database column while old application replicas still depend on it.

# 19. Canary Deployment Architecture

A Canary Deployment introduces a new version to a small percentage of production traffic before exposing it to the entire customer base.

Example:

```text
                    USERS
                      |
                      v
                ALB / INGRESS
                      |
             +--------+--------+
             |                 |
             v                 v
         STABLE v1         CANARY v2
          90% traffic       10% traffic
             |                 |
             v                 v
          Old Pods          New Pods
             |                 |
             +--------+--------+
                      |
                      v
               Application
```

The goal is to observe the new version under real Production conditions while limiting its initial exposure.

Canary releases require careful traffic control and meaningful monitoring.

# 20. Canary Release Example

Assume:

```text
Application: User
Current version: v1
New version: v2

Total traffic: 100%

Stable version: 90%
Canary version: 10%
```

We gradually increase traffic:

| Stage | Stable v1 | Canary v2 |
|---|---:|---:|
| Initial | 100% | 0% |
| Stage 1 | 95% | 5% |
| Stage 2 | 90% | 10% |
| Stage 3 | 75% | 25% |
| Stage 4 | 50% | 50% |
| Stage 5 | 0% | 100% |

These percentages are example stages, not a mandatory schedule.

At every stage, validate:

- Error rate.
- Request latency.
- Pod health.
- CPU and memory.
- Database behavior.
- Business transaction success.
- Application logs.
- Customer-impacting metrics.

Do not advance traffic if the current stage fails its acceptance criteria.

# 21. Canary Deployment Using Separate Kubernetes Deployments

One simple approach is to create separate stable and canary Deployments.

Example stable Deployment:

`user-stable.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: user-stable
  namespace: production

spec:
  replicas: 9

  selector:
    matchLabels:
      app: user
      version: stable

  template:
    metadata:
      labels:
        app: user
        version: stable

    spec:
      serviceAccountName: user-sa

      containers:
        - name: user
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:STABLE_DIGEST
          ports:
            - containerPort: 8080
```

Example canary Deployment:

`user-canary.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: user-canary
  namespace: production

spec:
  replicas: 1

  selector:
    matchLabels:
      app: user
      version: canary

  template:
    metadata:
      labels:
        app: user
        version: canary

    spec:
      serviceAccountName: user-sa

      containers:
        - name: user
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:CANARY_DIGEST
          ports:
            - containerPort: 8080
```

This creates nine stable pods and one canary pod.

A Kubernetes Service selecting both versions:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: user
  namespace: production

spec:
  selector:
    app: user

  ports:
    - port: 8080
      targetPort: 8080
```

The Service can select pods from both Deployments.

However, this does not guarantee an exact 90/10 traffic split.

Kubernetes Service endpoint selection is not a weighted traffic-splitting mechanism. The number of pods also does not translate into an exact percentage of requests.

Use a traffic-management solution when exact or controlled percentages are required.

# 22. Canary Deployment Using AWS ALB Weighted Target Groups

AWS Application Load Balancer supports forwarding actions that distribute traffic between target groups using weights.

Example architecture:

```text
                    USERS
                      |
                      v
                 AWS ALB
                      |
             +--------+--------+
             |                 |
             v                 v
        Stable Target      Canary Target
          Group               Group
             |                 |
             v                 v
        Stable Pods         Canary Pods
          Version 1          Version 2
```

Example initial configuration:

```text
Stable Target Group: 95
Canary Target Group: 5
```

The ALB listener forwards traffic using the configured target group weights.

Important requirements:

- Both target groups must be correctly registered.
- Target group health checks must be correct.
- The listener rule must forward to the correct groups.
- The AWS Load Balancer Controller or approved infrastructure process must own the configuration.
- Traffic-shift changes must be controlled and auditable.

## 22.1 Example ALB Forward Action

Illustrative listener rule configuration:

```json
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {
        "TargetGroupArn": "STABLE_TARGET_GROUP_ARN",
        "Weight": 95
      },
      {
        "TargetGroupArn": "CANARY_TARGET_GROUP_ARN",
        "Weight": 5
      }
    ],
    "TargetGroupStickinessConfig": {
      "Enabled": false
    }
  }
}
```

This is an illustrative ALB action structure.

Use the actual target group ARNs and listener rule configuration managed by your AWS infrastructure.

Traffic weights are distribution weights, not a guarantee that every short time window will have exactly the configured percentage of requests.

Session stickiness, client behavior, retries, and traffic volume can affect observed distribution.

# 23. Verify ALB Target Group Health

Get target groups:

```bash
aws elbv2 describe-target-groups \
  --region ap-south-1
```

Check stable target group health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$STABLE_TG_ARN" \
  --region ap-south-1
```

Check canary target group health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$CANARY_TG_ARN" \
  --region ap-south-1
```

Verify:

- Target registration.
- Target health.
- Health check path.
- Port.
- Security group access.
- Availability zone coverage.

Do not shift traffic to an unhealthy canary target group.

# 24. Canary Traffic Progression

A practical example:

## Stage 0 — Baseline

```text
Stable: 100%
Canary: 0%
```

Collect baseline metrics before changing traffic.

## Stage 1 — Initial Exposure

```text
Stable: 95%
Canary: 5%
```

Validate:

- Canary pods Ready.
- ALB targets healthy.
- HTTP error rate.
- Latency.
- Application logs.

## Stage 2 — Increase Exposure

```text
Stable: 90%
Canary: 10%
```

Compare canary and stable metrics.

## Stage 3 — Medium Exposure

```text
Stable: 75%
Canary: 25%
```

Check database load and downstream service behavior.

## Stage 4 — High Exposure

```text
Stable: 50%
Canary: 50%
```

Check business transaction success and application performance.

## Stage 5 — Complete Promotion

```text
Stable: 0%
Canary: 100%
```

After successful validation, complete the promotion process and update the desired production state in GitOps.

Do not leave the stable and canary configurations unmanaged after promotion.

# 25. Canary Validation Using Prometheus

A canary must be evaluated using application-level metrics.

Useful metrics include:

- Request rate.
- HTTP 5xx rate.
- Request latency.
- Application availability.
- Database connection failures.
- Business transaction failures.

The following examples assume that the application exposes suitable Prometheus metrics and labels.

## 25.1 Request Rate

```promql
sum by (version) (
  rate(http_requests_total{
    namespace="production",
    app="user"
  }[5m])
)
```

## 25.2 HTTP 5xx Rate

```promql
sum by (version) (
  rate(http_requests_total{
    namespace="production",
    app="user",
    status=~"5.."
  }[5m])
)
```

## 25.3 HTTP Error Ratio

```promql
sum(
  rate(http_requests_total{
    namespace="production",
    app="user",
    status=~"5.."
  }[5m])
)
/
clamp_min(
  sum(
    rate(http_requests_total{
      namespace="production",
      app="user"
    }[5m])
  ),
  1
)
```

## 25.4 Request Latency

For a histogram metric:

```promql
histogram_quantile(
  0.95,
  sum by (le, version) (
    rate(http_request_duration_seconds_bucket{
      namespace="production",
      app="user"
    }[5m])
  )
)
```

The metric names and labels above are examples. Use the actual metrics exported by your application.

# 26. Stable Versus Canary Comparison

A useful canary dashboard should display both versions.

| Metric | Stable | Canary |
|---|---|---|
| Request rate | Baseline | New version |
| HTTP 5xx | Baseline | New version |
| p95 latency | Baseline | New version |
| CPU | Baseline | New version |
| Memory | Baseline | New version |
| DB failures | Baseline | New version |
| Business failures | Baseline | New version |

Example decision:

```text
Stable 5xx rate: 0.2%
Canary 5xx rate: 4.5%

Decision: STOP CANARY
```

These are illustrative values.

The actual thresholds must come from the service's SLOs, baseline, error budget, and release policy.

# 27. Automated Canary Decision Rules

Example decision policy:

```text
IF canary HTTP 5xx rate exceeds the approved threshold
    THEN stop traffic progression

IF canary latency violates the approved SLO
    THEN stop traffic progression

IF canary pods become unhealthy
    THEN stop traffic progression

IF business transaction failures increase significantly
    THEN stop traffic progression

IF all acceptance checks pass
    THEN allow the next traffic stage
```

Do not promote based on one metric alone.

Use:

- A minimum observation period.
- A minimum request volume.
- Multiple independent signals.
- A defined failure threshold.
- A defined rollback procedure.

A canary with too little traffic may not generate enough samples to establish that the release is safe.

# 28. Canary Rollback

Consider:

```text
Stable: v1
Canary: v2

Traffic:
Stable: 90%
Canary: 10%

Canary HTTP 5xx rate increases.
```

The response should be:

1. Stop traffic progression.
2. Confirm the alert and metric.
3. Shift traffic away from canary.
4. Confirm stable targets are healthy.
5. Investigate canary logs.
6. Record the release failure.
7. Revert the GitOps desired state.
8. Validate application behavior.
9. Update the change record.

For ALB-managed traffic, the traffic-shift operation should be performed through the approved controller or infrastructure workflow.

Do not make an unmanaged listener change that will be overwritten by the infrastructure controller.

After traffic is removed from canary, verify:

```bash
kubectl get pods -n production
```

Check the stable application:

```bash
kubectl get deployment user-stable \
  -n production
```

Check application endpoint:

```bash
curl -f https://roboshop.example.com/health
```

# 29. Argo Rollouts Introduction

Argo Rollouts is a Kubernetes progressive delivery controller.

It provides additional deployment strategies beyond the standard Kubernetes Deployment controller.

Capabilities include:

- Rolling updates.
- Canary deployments.
- Blue-green deployments.
- Traffic management integration.
- Analysis templates.
- Automated promotion.
- Automated rollback based on analysis.

It integrates with traffic-management providers such as supported ingress controllers and service meshes.

The exact integrations depend on the installed version and environment.

In our platform, we can use Argo Rollouts to manage progressive delivery for selected microservices.

# 30. Argo Rollouts Architecture

```text
             GitOps Repository
                    |
                    v
                 Argo CD
                    |
                    v
             Argo Rollouts
                    |
          +---------+---------+
          |                   |
          v                   v
       Stable              Canary
      ReplicaSet          ReplicaSet
          |                   |
          +---------+---------+
                    |
                    v
              Traffic Manager
                    |
                    v
               ALB / Ingress
                    |
                    v
                  Users
                    |
                    v
                Prometheus
                    |
                    v
             Analysis Results
                    |
                    v
         Promote / Abort Decision
```

Argo CD manages the desired application configuration. Argo Rollouts manages progressive delivery execution.

# 31. Install Argo Rollouts

First, review the official installation instructions and the version approved for your cluster.

Install the controller using the approved installation method.

Example using Helm:

```bash
helm repo add argo \
  https://argoproj.github.io/argo-helm

helm repo update
```

Install into a dedicated namespace:

```bash
kubectl create namespace argo-rollouts
```

Example:

```bash
helm install argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts
```

For Production, pin the chart version and use the organization's approved chart repository and installation configuration.

Verify:

```bash
kubectl get pods -n argo-rollouts
```

Check controller logs:

```bash
kubectl logs deployment/argo-rollouts \
  -n argo-rollouts
```

# 32. Install Argo Rollouts kubectl Plugin

The Argo Rollouts kubectl plugin provides commands to inspect and control Rollouts.

Install the plugin using the official installation instructions for the operating system.

Verify:

```bash
kubectl argo rollouts version
```

Useful commands:

```bash
kubectl argo rollouts list rollouts \
  -n production
```

```bash
kubectl argo rollouts get rollout user \
  -n production
```

```bash
kubectl argo rollouts history rollout user \
  -n production
```

The plugin is a management client. The Argo Rollouts controller must also be installed in the cluster.

# 33. Create an Argo Rollouts Canary Resource

Create:

`charts/user/templates/rollout.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: user
  namespace: production

spec:
  replicas: 10

  selector:
    matchLabels:
      app: user

  template:
    metadata:
      labels:
        app: user

    spec:
      serviceAccountName: user-sa

      containers:
        - name: user

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/user@sha256:NEW_APPROVED_DIGEST

          ports:
            - containerPort: 8080

  strategy:
    canary:
      maxSurge: 1
      maxUnavailable: 0

      steps:
        - setWeight: 5

        - pause:
            duration: 2m

        - setWeight: 10

        - pause:
            duration: 5m

        - setWeight: 25

        - pause:
            duration: 5m

        - setWeight: 50

        - pause:
            duration: 10m

        - setWeight: 100
```

This example demonstrates the rollout structure.

It does not yet configure an external traffic manager or automated metric analysis.

Without a supported traffic-management integration, `setWeight` may be implemented using replica scaling rather than exact request-level traffic percentages.

For accurate traffic splitting, configure the appropriate Argo Rollouts traffic-routing integration.

# 34. Configure Stable and Canary Services

For an Argo Rollouts traffic-management integration, create separate Services.

Example stable Service:

`user-stable-service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: user-stable
  namespace: production

spec:
  selector:
    app: user

  ports:
    - port: 8080
      targetPort: 8080
```

Example canary Service:

`user-canary-service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: user-canary
  namespace: production

spec:
  selector:
    app: user

  ports:
    - port: 8080
      targetPort: 8080
```

These are structural examples.

The actual stable and canary service selectors and routing integration must follow the Argo Rollouts provider configuration. Do not assume that two Services with identical selectors automatically separate stable and canary pods.

# 35. Argo Rollouts Traffic Routing

Argo Rollouts requires a supported traffic-routing provider to control request-level traffic weights.

Examples include supported integrations with:

- AWS ALB.
- NGINX Ingress.
- Istio.
- Other supported traffic-management providers.

For an AWS EKS environment, verify the exact ALB integration and controller versions before designing the production rollout.

Do not combine a standard Kubernetes Deployment and an Argo Rollout with overlapping ownership of the same application pods.

When migrating a service to Argo Rollouts, plan the resource ownership transition carefully.

# 36. Argo Rollouts AnalysisTemplate

Argo Rollouts can evaluate application metrics before allowing promotion.

Example:

`user-analysis-template.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate

metadata:
  name: user-canary-analysis
  namespace: production

spec:
  metrics:
    - name: canary-success-rate

      interval: 1m

      count: 5

      successCondition: result[0] >= 0.99

      failureLimit: 2

      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090

          query: |
            sum(
              rate(http_requests_total{
                app="user",
                version="canary",
                status!~"5.."
              }[5m])
            )
            /
            clamp_min(
              sum(
                rate(http_requests_total{
                  app="user",
                  version="canary"
                }[5m])
              ),
              1
            )
```

This example assumes:

- Prometheus is accessible from the controller.
- The application exposes the required metric.
- The labels correctly distinguish canary traffic.
- The query returns one valid numeric result.
- The threshold reflects an approved release policy.

The sample success threshold of 99% is illustrative, not a universal production SLO.

# 37. Attach Analysis to the Rollout

Example:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 5

      - pause:
          duration: 2m

      - analysis:
          templates:
            - templateName: user-canary-analysis

      - setWeight: 10

      - pause:
          duration: 5m

      - analysis:
          templates:
            - templateName: user-canary-analysis

      - setWeight: 25

      - pause:
          duration: 5m

      - setWeight: 100
```

At the configured analysis stages, the controller evaluates the metrics.

A failed analysis can prevent the rollout from progressing and may cause the rollout to abort according to the configured strategy.

# 38. Execute Argo Rollouts Deployment

After the Rollout manifest and associated resources are committed through GitOps:

```bash
kubectl get rollouts -n production
```

Check details:

```bash
kubectl argo rollouts get rollout user \
  -n production
```

Watch progression:

```bash
kubectl argo rollouts get rollout user \
  -n production \
  --watch
```

Check history:

```bash
kubectl argo rollouts history rollout user \
  -n production
```

Check ReplicaSets:

```bash
kubectl get replicasets -n production
```

# 39. Manual Canary Promotion

In a controlled rollout, an authorized operator may approve the next stage.

Inspect:

```bash
kubectl argo rollouts get rollout user \
  -n production
```

Promote:

```bash
kubectl argo rollouts promote user \
  -n production
```

This can advance the rollout past pauses.

Only promote after reviewing the required metrics and approval conditions.

# 40. Abort a Failed Canary

If a canary fails:

```bash
kubectl argo rollouts abort user \
  -n production
```

Inspect:

```bash
kubectl argo rollouts get rollout user \
  -n production
```

An abort stops the rollout progression and restores the stable ReplicaSet behavior according to the rollout strategy.

The actual traffic restoration must be verified.

For GitOps-managed Production, reconcile the Git desired state with the known-good release after aborting.

# 41. Production Canary Acceptance Criteria

Before promoting the canary, define measurable acceptance criteria.

Example:

| Validation | Example acceptance criteria |
|---|---|
| Pod readiness | All required canary replicas Ready |
| Target health | All required targets healthy |
| HTTP 5xx | Within approved SLO threshold |
| p95 latency | Within approved latency objective |
| CPU | Within capacity limit |
| Memory | Within capacity limit |
| Database | No unacceptable connection failures |
| Messaging | No unacceptable processing failures |
| Business workflow | Critical transactions successful |
| Security | No release-blocking security finding |

The values must be based on the service's production baseline and approved SLOs.

# 42. Rolling Update vs Canary Deployment

| Area | Rolling Update | Canary |
|---|---|---|
| Replacement | Gradual pod replacement | New version receives limited exposure |
| Traffic control | Service endpoints | Traffic manager or rollout controller |
| Old version | Gradually removed | Retained during evaluation |
| Metrics evaluation | Operational monitoring | Explicit release decision gates |
| Rollback | Restore previous deployment revision | Stop traffic and restore stable release |
| Complexity | Lower | Higher |
| Best use | Routine compatible releases | Higher-risk releases |

Both approaches require monitoring and a tested rollback procedure.

# 43. Real Production Scenario — Database Compatibility

Scenario:

The new Orders service changes a database column.

Old version:

```text
order_status
```

New version:

```text
status
```

The old application still expects `order_status`.

During a rolling update, old and new pods may run simultaneously.

If the database column is removed immediately, the old application may fail.

Solution:

Use an expand-and-contract migration.

```text
STEP 1
Add new column while keeping old column.

STEP 2
Deploy application compatible with both columns.

STEP 3
Migrate existing data.

STEP 4
Verify application behavior.

STEP 5
Remove old application dependency.

STEP 6
Remove old column only after all consumers have migrated.
```

Database changes must be coordinated with application rollout and rollback plans.

# 44. Real Production Scenario — Canary Latency Increase

Scenario:

```text
Stable v1 p95 latency: 180 ms
Canary v2 p95 latency: 950 ms

Canary traffic: 10%
```

Investigation:

1. Check Prometheus latency metrics.
2. Compare stable and canary resource usage.
3. Inspect application logs.
4. Check database query performance.
5. Check connection pool usage.
6. Check downstream API latency.
7. Review code changes.

Decision:

Do not increase canary traffic.

If the approved thresholds are exceeded, abort the canary and return traffic to the stable version.

# 45. Real Production Scenario — Canary Pod CrashLoopBackOff

Scenario:

```text
Stable pods: Healthy
Canary pods: CrashLoopBackOff
Traffic: 5%
```

Commands:

```bash
kubectl get pods -n production
```

```bash
kubectl describe pod <CANARY_POD> \
  -n production
```

```bash
kubectl logs <CANARY_POD> \
  -n production \
  --previous
```

Investigate:

- Application startup.
- Environment variables.
- Secrets.
- Database access.
- Container command.
- Resource limits.
- Configuration differences.

Decision:

Stop the canary rollout.

Do not promote the release until the issue is fixed and the release is retested.

# 46. Production Deployment Runbook

## Rolling Update Runbook

```text
1. Confirm production approval.
2. Verify AWS account and EKS context.
3. Verify the approved image digest.
4. Review the GitOps pull request.
5. Merge the approved change.
6. Observe Argo CD synchronization.
7. Watch the Kubernetes rollout.
8. Verify all replicas are Ready.
9. Verify Service endpoints.
10. Check application logs.
11. Check Prometheus and Grafana.
12. Run smoke tests.
13. Record evidence.
14. Complete the deployment record.
```

## Canary Runbook

```text
1. Confirm production approval.
2. Verify stable release health.
3. Verify canary image digest.
4. Deploy canary resources.
5. Verify canary pod readiness.
6. Verify canary target health.
7. Route 5% traffic to canary.
8. Observe metrics and logs.
9. Evaluate acceptance criteria.
10. Increase to 10%.
11. Evaluate acceptance criteria.
12. Increase to 25%.
13. Evaluate acceptance criteria.
14. Increase to 50%.
15. Evaluate acceptance criteria.
16. Promote to 100%.
17. Verify stable and new version state.
18. Update GitOps desired state.
19. Collect evidence.
20. Complete release handover.
```

If a stage fails, stop progression and execute the approved rollback procedure.

# 47. Deployment Validation Script

Create:

`validate-progressive-deployment.sh`

```bash
#!/bin/bash

set -euo pipefail

NAMESPACE="production"
ROLLOUT="user"
EXPECTED_CONTEXT="roboshop-production"

echo "Checking Kubernetes context"

CURRENT_CONTEXT=$(kubectl config current-context)

if [[ "$CURRENT_CONTEXT" != "$EXPECTED_CONTEXT" ]]; then
    echo "ERROR: Incorrect cluster context"
    exit 1
fi

echo "Checking rollout resource"

if ! kubectl get rollout "$ROLLOUT" \
    -n "$NAMESPACE" >/dev/null 2>&1; then
    echo "ERROR: Rollout resource not found"
    exit 1
fi

echo "Checking rollout status"

kubectl argo rollouts get rollout "$ROLLOUT" \
    -n "$NAMESPACE"

echo "Checking production pods"

kubectl get pods -n "$NAMESPACE"

echo "Checking services"

kubectl get services -n "$NAMESPACE"

echo "Checking EndpointSlices"

kubectl get endpointslices -n "$NAMESPACE"

echo "Progressive deployment inspection completed"
```

Run:

```bash
chmod +x validate-progressive-deployment.sh

./validate-progressive-deployment.sh
```

This script performs inspection. It does not automatically approve or promote a release.

# 48. Production Best Practices

1. Use immutable image digests.
2. Keep production configuration in Git.
3. Use readiness probes correctly.
4. Configure liveness probes carefully.
5. Use startup probes for slow-starting applications.
6. Implement graceful shutdown.
7. Use appropriate resource requests and limits.
8. Maintain sufficient cluster capacity.
9. Use PDBs for supported voluntary disruptions.
10. Maintain backward-compatible database migrations.
11. Define service-level acceptance criteria.
12. Monitor stable and canary versions independently.
13. Use a minimum observation window.
14. Ensure sufficient request volume for metric analysis.
15. Stop traffic progression when thresholds fail.
16. Test rollback procedures.
17. Avoid unmanaged Production changes.
18. Keep Argo CD and Argo Rollouts responsibilities clear.
19. Record release evidence.
20. Validate business workflows, not just Kubernetes health.

# 49. Chapter Completion Checklist

## Rolling Updates

- [ ] RollingUpdate strategy understood.
- [ ] maxSurge configured.
- [ ] maxUnavailable configured.
- [ ] Readiness probe configured.
- [ ] Liveness probe configured.
- [ ] Startup probe evaluated.
- [ ] Graceful shutdown implemented.
- [ ] PDB reviewed.
- [ ] Deployment history reviewed.
- [ ] Rolling rollout executed.
- [ ] Rollout status validated.
- [ ] Image digest verified.
- [ ] Application health verified.
- [ ] Rollback procedure tested.

## Canary Deployment

- [ ] Canary architecture understood.
- [ ] Stable and canary versions separated.
- [ ] Traffic manager selected.
- [ ] Traffic weights configured.
- [ ] Target group health verified.
- [ ] Prometheus metrics available.
- [ ] Stable versus canary dashboard created.
- [ ] Error-rate threshold defined.
- [ ] Latency threshold defined.
- [ ] Business acceptance criteria defined.
- [ ] Canary stages validated.
- [ ] Failed canary simulated.
- [ ] Traffic rollback verified.

## Argo Rollouts

- [ ] Controller installed.
- [ ] kubectl plugin installed.
- [ ] Rollout resource created.
- [ ] Stable and canary routing configured.
- [ ] AnalysisTemplate configured.
- [ ] Prometheus integration verified.
- [ ] Manual promotion tested.
- [ ] Abort procedure tested.
- [ ] GitOps ownership validated.

---

# 50. Chapter Summary

In this chapter, we implemented the foundation for safe progressive delivery.

We covered:

- Kubernetes Rolling Updates.
- maxSurge and maxUnavailable.
- Readiness, liveness, and startup probes.
- Graceful shutdown.
- PodDisruptionBudgets.
- Deployment rollout history.
- Rolling deployment execution.
- Failed rollout troubleshooting.
- GitOps-based rollback.
- Canary deployment architecture.
- Stable and canary versions.
- AWS ALB weighted target groups.
- Traffic progression.
- Prometheus metrics.
- Canary acceptance criteria.
- Argo Rollouts.
- AnalysisTemplate.
- Automated promotion and abort.
- Production failure scenarios.
- Deployment runbooks.

**Key principle:** A successful progressive deployment is not just about releasing a new version gradually. It is about controlling traffic, continuously evaluating application health, and having a reliable way to stop or reverse the release before customers experience a major impact.

---

# 51. Next Chapter

## CHAPTER 207 — PRODUCTION MONITORING

**Filename:** `207-Production-Monitoring.md`

Next, we will implement complete Production monitoring for our 20+ microservices platform.

Planned topics:

1. Production monitoring architecture.
2. Prometheus architecture.
3. Prometheus installation on EKS.
4. Prometheus Operator.
5. ServiceMonitor and PodMonitor.
6. Node Exporter.
7. kube-state-metrics.
8. Application metrics.
9. Spring Boot metrics.
10. Node.js application metrics.
11. Custom Prometheus metrics.
12. Grafana installation.
13. Production Grafana dashboards.
14. Kubernetes dashboards.
15. Application dashboards.
16. CPU and memory monitoring.
17. Network monitoring.
18. Database monitoring.
19. Redis monitoring.
20. RabbitMQ monitoring.
21. Kafka monitoring.
22. ELK Stack architecture.
23. Elasticsearch.
24. Logstash.
25. Kibana.
26. Filebeat and log collection.
27. Kubernetes container logs.
28. Application log parsing.
29. Centralized log correlation.
30. Alertmanager.
31. Production alert rules.
32. Alert routing.
33. Notification integration.
34. SLI and SLO implementation.
35. Error budget monitoring.
36. Golden signals.
37. Production monitoring runbooks.
38. Monitoring failure simulation.
39. Monitoring troubleshooting.
40. Complete Production monitoring validation.

**Next file: `207-Production-Monitoring.md`.**