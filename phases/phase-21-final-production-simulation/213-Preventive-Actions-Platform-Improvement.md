No problem. I'll paste the actual Chapter 212 content here so you can copy it directly into a Markdown file.

The complete chapter is 2,825 lines, so it won't fit reliably in one chat message. I'll provide it in consecutive parts, keeping the original section numbering and content in order. This is Part 1.

# CHAPTER 212 — PREVENTIVE ACTIONS & PLATFORM IMPROVEMENT

## Turning incidents and operational risks into measurable engineering improvements

Platform context: AWS, EKS, Kubernetes, 20+ microservices, Docker, ECR, Helm, Argo CD, Terraform, Jenkins, GitHub Actions, GitLab CI/CD, RDS, Redis, RabbitMQ, Kafka, Prometheus, Grafana, and ELK.

Purpose: Establish a practical, risk-based improvement system that prevents recurring incidents, reduces blast radius, improves deployment safety, strengthens observability, and makes the platform easier to operate and recover.

> Adapt names, thresholds, accounts, regions, namespaces, ownership, and approval rules to the target environment. Commands and configuration are examples until validated against the actual modules, cluster version, and organizational standards. Test changes in non-production before production rollout.

## 1. Learning objectives

By completing this chapter, you should be able to:

1. Convert RCA findings into prioritized engineering work.
2. Distinguish corrective, preventive, detective, resilience, and recovery actions.
3. Build a platform improvement backlog linked to evidence and risk.
4. Prioritize work using customer impact, likelihood, blast radius, and effort.
5. Improve Kubernetes workload reliability and resource management.
6. Improve EKS infrastructure, node capacity, networking, and cluster operations.
7. Strengthen CI/CD, GitOps, Terraform, and release safety.
8. Improve application resilience across microservice dependencies.
9. Improve database, cache, and messaging reliability.
10. Improve Prometheus, Grafana, and ELK observability.
11. Improve security, identity, secrets, and supply-chain controls.
12. Define measurable acceptance criteria and verify effectiveness.
13. Build a continuous improvement operating model.
14. Run platform improvement reviews and reliability game days.

## 2. Why preventive improvement matters

Restoring a service ends the immediate incident. It does not automatically reduce the chance of recurrence.

A platform that repeatedly fixes symptoms accumulates operational risk:

- The same deployment failure returns.
- Teams manually restart pods without removing the underlying fault.
- Alerts detect incidents after customers are affected.
- Backups exist but restores are not tested.
- Shared infrastructure creates a large blast radius.
- Retries amplify dependency failures.
- Terraform changes are difficult to reverse safely.
- Teams lack clear ownership for cross-service dependencies.
- Runbooks become stale.
- Corrective actions remain open indefinitely.

Preventive improvement turns incident learning into system changes that can be tested and measured.

## 3. Improvement action types

| Action type   | Purpose                                          | Example                           |
| ------------- | ------------------------------------------------ | --------------------------------- |
| Corrective    | Fix the specific defect that caused the incident | Correct a faulty timeout          |
| Preventive    | Reduce the chance of recurrence                  | Add dependency failure tests      |
| Detective     | Detect the failure earlier                       | Alert on checkout error rate      |
| Resilience    | Reduce impact when failure occurs                | Add bounded retries and bulkheads |
| Recovery      | Restore service or data safely                   | Test RDS point-in-time restore    |
| Security      | Prevent unauthorized access or exposure          | Tighten IAM permissions           |
| Operational   | Make response safer and faster                   | Update and test the runbook       |
| Architectural | Remove systemic weakness                         | Isolate critical workloads        |

One incident may require several action types. Adding an alert alone does not prevent the failure; it may only improve detection.

## 4. Platform improvement lifecycle

```
Incident / Risk / Review finding
             |
             v
       Record evidence
             |
             v
       Define failure mode
             |
             v
       Assess customer risk
             |
             v
       Select improvement type
             |
             v
       Prioritize and assign
             |
             v
       Design and review change
             |
             v
       Implement in stages
             |
             v
       Validate acceptance criteria
             |
             v
       Observe effectiveness
             |
             v
       Close or create follow-up
```

An action should not be closed simply because a ticket is marked Done. Close it after acceptance criteria and effectiveness checks are satisfied.

## 5. Establish platform reliability objectives

### 5.1 Define service criticality

| Tier   | Example                                                              | Expected engineering treatment                                 |
| ------ | -------------------------------------------------------------------- | -------------------------------------------------------------- |
| Tier 0 | Identity, cluster access, critical shared control-plane dependencies | Strong access controls, tested recovery, high availability     |
| Tier 1 | Checkout, orders, payments                                           | Explicit SLOs, capacity headroom, tested rollback and recovery |
| Tier 2 | Catalog, user profile, inventory updates                             | Defined SLOs, dependency controls, operational runbooks        |
| Tier 3 | Internal reporting or non-critical asynchronous jobs                 | Controlled degraded mode and documented recovery               |

The exact tiers and service membership must be agreed by service owners and business stakeholders.

### 5.2 Define SLIs

Common service-level indicators:

- Availability or successful request ratio.
- Latency percentiles.
- Request throughput.
- Correctness of business transactions.
- Queue processing delay.
- Data freshness.
- Recovery time.
- Dependency success rate.

Example request success ratio:

\\[ \text{Success ratio} = \frac{\text{successful eligible requests}} {\text{total eligible requests}} \times 100 \\]

Define which responses count as successful, which traffic is excluded, and how synthetic checks are treated.

### 5.3 Define SLOs and error budgets

An SLO is a target for an SLI over a defined window. A service might target 99.9% successful eligible requests over a rolling 30-day window. This is illustrative, not a universal recommendation.

\\[ \text{Error budget} = 1 - \text{SLO target} \\]

For a 99.9% target, the allowed unsuccessful proportion is 0.1% over the same measurement window.

Use error-budget policy to inform release risk, reliability investment, and change restrictions. An SLO does not replace customer-specific contractual commitments.

## 6. Create a platform improvement register

Every improvement should have a traceable record.

### 6.1 Required fields

| Field               | Description                                                |
| ------------------- | ---------------------------------------------------------- |
| Improvement ID      | Unique identifier                                          |
| Source              | RCA, game day, audit, review, risk assessment              |
| Service/platform    | Affected component                                         |
| Failure mode        | What can go wrong                                          |
| Customer impact     | Consequence                                                |
| Evidence            | Logs, metrics, incident, test                              |
| Action type         | Corrective, preventive, detective, resilience, recovery    |
| Priority            | Risk-based priority                                        |
| Owner               | Accountable role/person                                    |
| Due date            | Agreed target                                              |
| Acceptance criteria | How completion is proven                                   |
| Verification        | Test or operational evidence                               |
| Status              | Proposed, approved, in progress, blocked, verified, closed |
| Residual risk       | What remains after implementation                          |

### 6.2 Example register

```
Improvement_ID,Source,Component,Failure_Mode,Action,Type,Priority,Owner,Due_Date,Acceptance_Criteria,Status
PI-001,INC-1042,checkout-api,Retry amplification,Implement bounded retries and jitter,Resilience,P1,Checkout owner,YYYY-MM-DD,Fault test passes without retry storm,Proposed
PI-002,Game-day,EKS,Worker capacity exhaustion,Validate node headroom and autoscaling,Preventive,P1,Platform owner,YYYY-MM-DD,Node-loss exercise passes,Proposed
PI-003,INC-1045,RDS,Restore procedure untested,Run point-in-time restore exercise,Recovery,P1,Database owner,YYYY-MM-DD,Restore meets approved RTO/RPO,Proposed
PI-004,Review,Observability,Customer impact detected late,Add checkout SLI alert,Detective,P1,Observability owner,YYYY-MM-DD,Alert test routes correctly,Proposed
```

Replace placeholders with real owners and dates before approving the backlog.

## 7. Prioritize improvements by risk

Evaluate:

- Customer impact.
- Probability or frequency.
- Blast radius.
- Detectability.
- Recovery difficulty.
- Data integrity or security consequence.
- Existing compensating controls.

| Impact   | Likelihood | Example priority |
| -------- | ---------- | ---------------- |
| Critical | High       | P0 / urgent      |
| Critical | Medium     | P1               |
| High     | High       | P1               |
| High     | Medium     | P2               |
| Moderate | Medium     | P2               |
| Low      | Low        | P3               |

A simple internal prioritization score may be:

\\[ \text{Risk score} = \text{Impact} \times \text{Likelihood} \times \text{Exposure} \\]

Use consistent scales such as 1–5. This is a prioritization aid, not a mathematically precise probability.

- P0: Immediate action for active severe risk, security exposure, or imminent critical failure.
- P1: High risk to critical customer workflows or recurring severe incidents.
- P2: Material reliability or operational improvement.
- P3: Lower-risk hygiene and long-term maintainability.

Group related findings into coherent improvements. Split work when separate owners, delivery stages, or verification methods are required.

## 8. Kubernetes workload reliability baseline

Every production workload should have an explicit operational baseline:

- Multiple replicas where availability requires them.
- Readiness, liveness, and startup probes appropriate to application behavior.
- CPU and memory requests.
- Appropriate limits where required.
- PodDisruptionBudget for critical replicated workloads.
- Topology spread or anti-affinity where appropriate.
- Graceful termination and shutdown handling.
- Security context and non-root execution where supported.
- ServiceAccount and least-privilege access.
- Image pinned by immutable digest or controlled release identity.
- Resource quota and namespace boundaries.
- Deployment strategy and rollback instructions.

### 8.1 Example Deployment baseline

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
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
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      terminationGracePeriodSeconds: 45
      serviceAccountName: checkout-api-sa
      containers:
        - name: checkout-api
          image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/checkout-api@sha256:REPLACE_WITH_DIGEST
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 15
            timeoutSeconds: 2
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
```

This is a template. The image digest, probe endpoints, UID, filesystem needs, ports, and resource values must match the real application. Do not set a read-only root filesystem if the application requires writable paths unless those paths are explicitly mounted.

### 8.2 Probe design

- Startup probe protects slow-starting applications from premature liveness restarts.
- Readiness probe controls whether a pod should receive traffic.
- Liveness probe detects a process that is alive but irrecoverably unhealthy.

Do not make liveness depend on every external dependency. A temporary database outage should not necessarily cause every application pod to restart and increase recovery pressure.

### 8.3 PodDisruptionBudget

```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: checkout-api
```

Ensure the PDB is compatible with replica count and maintenance requirements. A PDB can block voluntary disruptions; it does not protect against every involuntary failure.

### 8.4 Topology spread

```
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: checkout-api
```

Use zone spread only when the cluster has appropriate node capacity and the workload can tolerate the resulting placement. Validate behavior during zone capacity loss.

## 9. Resource requests, limits, and capacity

Missing or inaccurate requests can cause overpacking, CPU contention, scheduling failures, memory pressure, eviction, and unpredictable autoscaling. Limits that are too low can cause throttling or OOM kills.

Inspect current usage:

```
kubectl top nodes
kubectl top pods -A
kubectl describe node <node-name>
kubectl get pods -A -o wide
```

Use historical Prometheus metrics as well. A single point-in-time sample is not enough for capacity planning.

Improvement procedure:

1. Collect representative usage over normal and peak periods.
2. Identify CPU throttling and memory working-set trends.
3. Inspect OOMKilled and eviction events.
4. Compare actual usage with requests and limits.
5. Check node allocatable capacity.
6. Review workload scheduling constraints.
7. Adjust requests in a test environment.
8. Run load and failure tests.
9. Roll out gradually.
10. Observe saturation, latency, and cost.

Quick inventory:

```
kubectl get pods -n production \
  -o custom-columns='NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory,CPU_LIMIT:.spec.containers[*].resources.limits.cpu,MEM_LIMIT:.spec.containers[*].resources.limits.memory'
```

For a large platform, use a structured inventory or policy tool. This command is not a complete resource audit.

## 10. Horizontal Pod Autoscaler improvements

Example HPA:

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 3
  maxReplicas: 12
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

Before enabling or changing HPA:

- Ensure resource requests are defined.
- Confirm metrics-server or the required metrics source works.
- Confirm the node group can supply additional capacity.
- Check startup time and warm-up behavior.
- Validate downstream database and messaging capacity.
- Set sensible minimum and maximum replicas.
- Test scale-up and scale-down behavior.

Scaling application replicas without scaling or protecting downstream dependencies can make an incident worse.

## 11. Cluster autoscaling and node group resilience

Review minimum, desired, and maximum node group capacity; instance type availability; AZ distribution; Pod scheduling constraints; DaemonSet overhead; system workload reservations; autoscaler behavior; quotas; and EC2 capacity constraints.

```
aws eks list-nodegroups \
  --cluster-name production-eks \
  --region ap-south-1

aws eks describe-nodegroup \
  --cluster-name production-eks \
  --nodegroup-name system \
  --region ap-south-1
```

Do not assume autoscaling can create capacity instantly. Model provisioning time and maintain appropriate headroom for critical services.

## 12. Namespace, quota, and workload isolation

Example ResourceQuota:

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

Example LimitRange:

```
apiVersion: v1
kind: LimitRange
metadata:
  name: production-defaults
  namespace: production
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
```

These values are examples. Quotas should reflect namespace ownership and real capacity. Defaults can hide poor application sizing if not reviewed.

## 13. Network and DNS improvements

Review CoreDNS replica count and resources, DNS latency and errors, VPC CNI health, subnet IP headroom, security groups, network policies, service selectors, EndpointSlices, ingress, ALB health, Route 53 records, and TTL.

```
kubectl get pods -n kube-system -o wide
kubectl get deployment coredns -n kube-system
kubectl get daemonset aws-node -n kube-system
kubectl get endpointslices -A
kubectl get networkpolicies -A
```

Preventive improvements:

- Alert on CoreDNS errors and saturation.
- Monitor subnet free IP capacity.
- Test service-to-service connectivity after network changes.
- Review network policies with service owners.
- Validate ALB target health after releases.
- Keep a tested DNS and ingress recovery runbook.

Do not make broad network policy changes without testing required application paths.

## 14. Kubernetes upgrade and maintenance safety

1. Check EKS and add-on compatibility.
2. Review deprecated API usage.
3. Test application manifests against the target version.
4. Validate node AMI and runtime compatibility.
5. Review disruption budgets and capacity.
6. Upgrade a non-production cluster.
7. Run smoke and integration tests.
8. Upgrade production using an approved plan.
9. Drain nodes safely.
10. Validate workloads, storage, ingress, and monitoring.
11. Preserve supported recovery options.
12. Record results.

Do not assume an EKS control-plane upgrade can be reversed to an earlier Kubernetes version. Design forward recovery and workload compatibility.

## 15. Microservice dependency reliability

In a 20+ microservice environment, one service failure can affect several dependent services.

Example order flow:

```
                 Customer
                    |
                    v
               API Gateway
                    |
                    v
               User Service
                    |
                    v
               Cart Service
                    |
                    v
              Order Service
               /        \
              v          v
       Payment Service  Inventory Service
              |          |
              v          v
          Payment DB   Inventory DB
              |
              v
        Notification Service
```

A failure in Payment Service may affect order completion. A failure in Notification Service should not necessarily prevent an order from being placed.

### 15.1 Build a dependency inventory

For every service, record:

| Field                   | Example                            |
| ----------------------- | ---------------------------------- |
| Service                 | order-service                      |
| Owner                   | Order team                         |
| Criticality             | Tier 1                             |
| Upstream dependencies   | cart-service, user-service         |
| Downstream dependencies | payment-service, inventory-service |
| Database                | orders-db                          |
| Messaging               | RabbitMQ                           |
| Cache                   | Redis                              |
| Timeout                 | Defined per operation              |
| Retry policy            | Bounded, operation-specific        |
| Failure behavior        | Reject, queue, or degrade          |
| Dashboard               | Grafana service dashboard          |
| Runbook                 | Order service failure runbook      |

### 15.2 Identify critical and non-critical dependencies

Classify dependencies by business requirement.

Critical dependencies:

- A payment authorization required before an order can be confirmed.
- Inventory reservation required to prevent overselling.
- Authentication required before protected operations.

Potentially non-critical dependencies:

- Email notification.
- Analytics event delivery.
- Non-essential recommendation data.
- Background reporting.

The classification must reflect actual business rules. Do not make a dependency asynchronous merely to improve availability if doing so breaks transaction correctness.

### 15.3 Prevent cascading failures

A cascading failure can happen when:

1. Payment Service becomes slow.
2. Order Service waits for payment responses.
3. Order Service request threads become occupied.
4. API Gateway requests begin timing out.
5. Customers retry.
6. The retry traffic increases pressure on the already unhealthy services.

Preventive controls include:

- Connection and request timeouts.
- Bounded retries.
- Exponential backoff and jitter.
- Circuit breakers.
- Bulkheads.
- Concurrency limits.
- Queue-based decoupling where business semantics allow.
- Graceful degradation.
- Load shedding.
- Idempotency.

## 16. Bounded retries and exponential backoff

Retries are useful for temporary failures. Uncontrolled retries can amplify an outage.

### 16.1 Unsafe retry example

```
while True:    try:        response = call_payment_service()        break    except Exception:        continue
```

This is unsafe because it:

- Has no retry limit.
- Has no delay.
- Can overload the dependency.
- Can occupy application threads indefinitely.
- Does not distinguish retryable from permanent errors.

### 16.2 Safer retry example

```
import randomimport timeMAX_ATTEMPTS = 3BASE_DELAY = 0.5MAX_DELAY = 3.0def call_with_retry(operation):    for attempt in range(MAX_ATTEMPTS):        try:            return operation()        except TemporaryServiceError:            if attempt == MAX_ATTEMPTS - 1:                raise            delay = min(                BASE_DELAY * (2 ** attempt),                MAX_DELAY            )            delay += random.uniform(0, 0.2)            time.sleep(delay)
```

This is an illustrative pattern. Define `TemporaryServiceError` based on the actual client library and error classification.

### 16.3 Retry policy checklist

- Set a maximum attempt count.
- Use exponential backoff.
- Add jitter.
- Define an overall request deadline.
- Retry only appropriate transient failures.
- Do not retry validation or authorization errors.
- Avoid retrying non-idempotent operations without an idempotency mechanism.
- Prevent retries at multiple layers from multiplying unexpectedly.
- Monitor retry counts and retry exhaustion.

### 16.4 Example timeout budget

Suppose an API request has a total deadline of 5 seconds.

```
API Gateway deadline:       5.0 seconds
Order Service budget:       4.5 seconds
Payment call timeout:       1.5 seconds
Inventory call timeout:     1.0 seconds
Internal processing:       remaining budget
```

The actual budgets must account for sequential versus parallel calls, network overhead, and response processing. Do not configure each downstream call with a timeout longer than the remaining upstream deadline.

## 17. Circuit breakers and bulkheads

### 17.1 Circuit breaker

A circuit breaker temporarily stops calls to a failing dependency.

Typical states:

```
          CLOSED
             |
      Failure threshold
             |
             v
           OPEN
             |
       Wait duration
             |
             v
         HALF-OPEN
          /      \
     Success    Failure
        |          |
        v          v
      CLOSED      OPEN
```

- Closed: Calls flow normally.
- Open: Calls fail fast or use a fallback.
- Half-open: A limited number of test calls are allowed.

A circuit breaker should be configured with:

- Failure-rate or consecutive-failure threshold.
- Minimum request volume.
- Open-state duration.
- Half-open probe limit.
- Dependency-specific timeout.
- Safe fallback behavior.

Do not use one circuit breaker configuration blindly for every service.

### 17.2 Bulkheads

Bulkheads isolate resources so one dependency cannot consume all application capacity.

Example:

```
Order Service
     |
     +---- Payment thread pool
     |
     +---- Inventory thread pool
     |
     +---- Notification worker pool
```

If Notification Service is slow, notification tasks should not consume every thread required by order processing.

Possible controls:

- Separate worker pools.
- Separate connection pools.
- Per-dependency concurrency limits.
- Bounded queues.
- Queue overflow policies.
- Independent resource requests for critical workloads.

### 17.3 Validation

Simulate a slow dependency in a non-production environment and verify:

1. The affected calls time out within their budget.
2. The circuit breaker opens when its configured conditions are met.
3. Unrelated operations continue.
4. Thread and connection pools remain within safe limits.
5. The service recovers when the dependency returns.
6. Metrics and logs clearly identify the event.

## 18. Graceful degradation

Graceful degradation means preserving the most important business functionality when a non-critical component is unavailable.

Example:

| Component unavailable  | Possible behavior                                                                    |
| ---------------------- | ------------------------------------------------------------------------------------ |
| Recommendation service | Show products without recommendations                                                |
| Email notification     | Accept order and queue notification                                                  |
| Analytics              | Continue transaction and buffer event                                                |
| Product image service  | Show product details without image                                                   |
| Payment provider       | Do not confirm payment; show controlled failure or approved alternate payment flow   |
| Inventory service      | Follow approved reservation or availability policy; do not claim stock is guaranteed |

### 18.1 Example order workflow

```
Customer submits order
          |
          v
      Validate cart
          |
          v
   Reserve inventory
          |
          v
   Authorize payment
          |
          v
     Create order
          |
          v
   Publish order event
          |
          +------------------+
          |                  |
          v                  v
    Notification         Analytics
          |                  |
          v                  v
     Email/SMS          Reporting
```

If analytics is unavailable, the order flow may continue while analytics events are retried or buffered.

If payment authorization fails, the system must not falsely confirm a paid order.

### 18.2 Degradation requirements

Document:

- Which functions are essential.
- Which functions may be disabled.
- Whether data can be buffered.
- Maximum acceptable buffering time.
- How delayed operations are replayed.
- How duplicate processing is prevented.
- How customers are informed.
- How the system returns to normal operation.

## 19. Idempotency and duplicate request protection

An idempotent operation produces the same intended business outcome when the same request is processed more than once.

This is important because retries, network timeouts, message redelivery, and client resubmission can cause duplicate requests.

### 19.1 Example payment problem

```
Customer clicks Pay
       |
       v
Payment request sent
       |
       v
Payment provider charges customer
       |
       v
Response is lost
       |
       v
Customer retries
       |
       v
Second charge may occur
```

### 19.2 Idempotency key

A client can send a unique key for a logical payment operation.

```
POST /payments
Idempotency-Key: order-123-payment-attempt-1
Content-Type: application/json
```

The payment service should:

1. Receive the key.
2. Check whether it has already processed the key.
3. Return the previously stored result if it has.
4. Otherwise process the operation.
5. Store the result associated with the key.
6. Return the result to the caller.

### 19.3 Important design details

- Scope keys to the correct customer or account.
- Store a request fingerprint to detect reuse with different payloads.
- Define key expiration and retention.
- Handle concurrent requests with database constraints or atomic operations.
- Make result storage durable enough for the business requirement.
- Coordinate with external payment provider idempotency features where available.

Do not rely only on an in-memory map. A pod restart or rescheduling event could lose the record.

### 19.4 Practical test

Send the same payment request multiple times with the same idempotency key.

Expected result:

- One logical payment operation.
- Repeated requests return the same outcome.
- No duplicate charge.
- A different payload using the same key is rejected or handled according to the documented contract.

## 20. Service ownership and dependency mapping

A platform improvement cannot be sustained if nobody owns the affected service.

Define clear ownership for:

- Application code.
- Kubernetes deployment manifests.
- Helm charts.
- CI/CD pipelines.
- Terraform modules.
- Database schemas.
- Messaging resources.
- Dashboards and alerts.
- Operational runbooks.
- Incident response.
- Recovery and backup verification.

### 20.1 Example ownership matrix

| Component          | Primary owner               | Supporting owner     |
| ------------------ | --------------------------- | -------------------- |
| EKS cluster        | Platform team               | Cloud infrastructure |
| Terraform modules  | Infrastructure team         | Platform team        |
| checkout-api       | Checkout team               | Platform team        |
| payment-service    | Payments team               | Database team        |
| RDS                | Database team               | Cloud infrastructure |
| RabbitMQ           | Messaging/platform team     | Application teams    |
| Kafka              | Messaging/platform team     | Application teams    |
| Prometheus         | Observability/platform team | Service owners       |
| Grafana dashboards | Service owners              | Observability team   |
| ELK pipeline       | Observability/platform team | Application teams    |
| Argo CD            | Platform team               | Application teams    |

### 20.2 Ownership rules

Each production service should have:

- A named primary team.
- A backup escalation path.
- A repository location.
- A deployment owner.
- A dashboard.
- An alert route.
- A runbook.
- A documented dependency list.
- A recovery procedure.

Review ownership after team restructuring and service transfers.

## 21. RDS reliability improvements

Databases are critical shared dependencies. A database problem can affect multiple microservices even when the Kubernetes cluster is healthy.

### 21.1 Preventive review

Review:

- Instance class and capacity.
- CPU and memory trends.
- Storage utilization and growth.
- Connection counts.
- Connection pool configuration.
- Query latency.
- Lock waits and deadlocks.
- Backup retention.
- Point-in-time recovery configuration.
- Multi-AZ configuration where required.
- Parameter and option groups.
- Maintenance windows.
- Encryption and network access.
- Database user privileges.

### 21.2 Connection pool management

If 20 services each create excessive connections, RDS can reach its connection limit.

Example:

```
20 microservices
       |
       v
3 replicas per service
       |
       v
5 connections per pod
       |
       v
20 x 3 x 5 = 300 connections
```

This is only a simple capacity illustration. Real connection usage depends on application configuration, background workers, multiple data sources, and traffic.

Calculate the connection budget across all services, including administrative and monitoring connections.

### 21.3 Example HikariCP configuration

```
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=3000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

These values are examples, not universal production defaults. Tune them based on workload, database limits, query duration, and replica count.

### 21.4 RDS preventive checks

```
aws rds describe-db-instances \
  --db-instance-identifier production-db \
  --region ap-south-1
```

Check backup configuration:

```
aws rds describe-db-instances \
  --db-instance-identifier production-db \
  --query 'DBInstances[0].[BackupRetentionPeriod,PreferredBackupWindow,MultiAZ]' \
  --region ap-south-1
```

### 21.5 Database improvement acceptance criteria

- Connection utilization remains within an agreed safe threshold.
- Slow queries have owners and remediation plans.
- Backup configuration meets the approved policy.
- Restore testing demonstrates the required recovery objective.
- Database access is least privilege.
- Schema changes support the deployed application versions.
- Monitoring detects connection exhaustion and latency growth.

## 22. Redis reliability improvements

Redis may be used for caching, sessions, rate limiting, distributed locks, or other application functions. The failure impact depends on how it is used.

### 22.1 Preventive review

Review:

- Memory utilization.
- Eviction policy.
- Key expiration.
- Hot keys.
- Connection count.
- Command latency.
- Persistence requirements.
- Replication and failover behavior.
- Network access and authentication.
- Client timeout and retry configuration.

### 22.2 Cache stampede

A cache stampede occurs when many requests try to rebuild the same expired or missing cache entry simultaneously.

Example:

```
Product cache expires
        |
        v
1,000 requests arrive
        |
        v
1,000 cache misses
        |
        v
1,000 database queries
        |
        v
Database overload
```

Preventive techniques:

- Add randomized TTL variation.
- Use request coalescing or single-flight behavior.
- Apply controlled refresh-ahead where appropriate.
- Protect the database with concurrency limits.
- Use safe stale data only when business rules permit it.

### 22.3 Example cache TTL variation

```
import randombase_ttl = 300actual_ttl = base_ttl + random.randint(0, 60)
```

### 22.4 Failure behavior

For every Redis-backed feature, decide:

- Can the application continue without the cache?
- Is stale data acceptable?
- Does the feature require strong consistency?
- Can a cache miss overload the database?
- Are sessions or locks stored in Redis?
- What happens during failover?

Do not assume Redis is always disposable. Its business role determines the recovery requirements.

## 23. RabbitMQ reliability improvements

RabbitMQ provides messaging and work-queue capabilities. Poor queue management can cause growing backlogs, duplicate processing, and delayed business operations.

### 23.1 Preventive review

Review:

- Queue depth.
- Consumer count.
- Consumer processing rate.
- Unacknowledged messages.
- Dead-letter queues.
- Message TTL.
- Persistent message requirements.
- Publisher confirms.
- Consumer acknowledgements.
- Retry and redelivery behavior.
- Disk and memory alarms.
- Cluster and node health.

### 23.2 Queue backlog

```
Producers publish messages
          |
          v
      Main Queue
          |
          v
    Consumer workers
          |
          v
     Business logic
```

If producers publish faster than consumers process, queue depth grows.

Monitor:

- Ready messages.
- Unacknowledged messages.
- Publish rate.
- Delivery rate.
- Acknowledgement rate.
- Consumer availability.
- Oldest message age.

### 23.3 Dead-letter handling

Failed messages should not retry forever.

A safer design:

```
Main Queue
    |
    v
Consumer
    |
    +---- Success ----> Acknowledge
    |
    +---- Retryable --> Controlled retry
    |
    +---- Permanent --> Dead-letter queue
```

Use bounded retries and preserve enough metadata to diagnose failures.

### 23.4 Example RabbitMQ inspection

```
rabbitmqctl list_queues \
  name messages_ready messages_unacknowledged consumers
```

Use the commands supported by the deployed RabbitMQ version and deployment model.

### 23.5 Acceptance criteria

- Queue growth triggers an actionable alert.
- Consumer failure does not silently lose required messages.
- Poison messages are isolated.
- Retry behavior is bounded.
- Duplicate delivery is handled safely.
- Queue recovery is tested.

## 24. Kafka reliability improvements

Kafka is commonly used for durable event streams and asynchronous communication. Preventive improvements should focus on event correctness, consumer lag, replication, and safe replay.

### 24.1 Preventive review

Review:

- Broker health.
- Under-replicated partitions.
- Offline partitions.
- Partition distribution.
- Replication factor.
- Producer acknowledgements.
- Consumer lag.
- Consumer group stability.
- Retention and storage.
- Message size.
- Schema compatibility.
- Reprocessing and replay procedures.

### 24.2 Consumer lag

Consumer lag is the difference between the latest available offset and the consumer's committed or processed position, depending on the monitoring definition.

```
Producer
   |
   v
Kafka Topic
   |
   v
Partition 0
   |
   v
Consumer Group
   |
   v
Application processing
```

If producers publish faster than consumers process, lag increases.

### 24.3 Example consumer group inspection

```
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group order-consumer-group
```

### 24.4 Preventive controls

- Alert on sustained consumer lag.
- Review partition key distribution.
- Avoid hot partitions.
- Make consumers idempotent.
- Handle rebalances correctly.
- Monitor broker disk capacity.
- Test consumer restart and replay.
- Define retention based on recovery and replay needs.
- Use compatible schema evolution rules.

### 24.5 Message processing correctness

A consumer may process an event and fail before committing its offset. The event can be delivered again.

Therefore:

- Design processing to tolerate duplicate delivery.
- Use event IDs or business keys.
- Commit offsets at the correct point in the processing workflow.
- Use transactional patterns where required and supported.
- Define how failed events are retried or quarantined.

Do not claim exactly-once business outcomes merely because Kafka supports specific delivery or transactional features. End-to-end correctness depends on the consumer, external database, and side effects.

## 25. CI/CD pipeline reliability baseline

A production CI/CD pipeline must do more than build and deploy an application. It should provide repeatable, traceable, secure, and recoverable software delivery.

For a 20+ microservices platform, each service should follow a consistent pipeline structure while allowing service-specific testing and deployment requirements.

### 25.1 Recommended pipeline flow

```
Developer pushes code
         |
         v
     Git repository
         |
         v
   Pull request checks
         |
         v
    Source validation
         |
         v
       Unit tests
         |
         v
     SonarQube scan
         |
         v
    Dependency checks
         |
         v
      Docker build
         |
         v
      Trivy scan
         |
         v
      Image push
         |
         v
  Deployment manifest update
         |
         v
       Argo CD
         |
         v
   Kubernetes deployment
         |
         v
     Smoke testing
         |
         v
    Release verification
```

The exact sequence depends on the organization's security policy. For example, dependency scanning may happen before image building, and image scanning may happen before pushing to a production repository.

### 25.2 Pipeline design principles

- Use repeatable build steps.
- Pin tool and action versions.
- Use immutable image tags or digests.
- Separate build and deployment responsibilities.
- Store secrets outside source code.
- Require review for production changes.
- Enforce security gates based on policy.
- Preserve build and deployment evidence.
- Make failed deployments visible.
- Support safe rollback.
- Avoid unnecessary rebuilds of unchanged components.

### 25.3 Per-service pipeline ownership

For each microservice, define:

| Item                 | Example              |
| -------------------- | -------------------- |
| Repository           | checkout-service     |
| Build tool           | Maven                |
| Runtime              | Java                 |
| Dockerfile           | Dockerfile           |
| Image repository     | ECR checkout-service |
| Scan                 | SonarQube and Trivy  |
| Deployment package   | Helm chart           |
| GitOps repository    | platform-gitops      |
| Kubernetes namespace | production           |
| Deployment owner     | Checkout team        |

The service repository should not silently deploy unrelated services.

## 26. Pipeline stage design and failure handling

A pipeline should stop when a mandatory quality or security condition fails.

### 26.1 Example stages

| Stage           | Purpose                            | Failure behavior                     |
| --------------- | ---------------------------------- | ------------------------------------ |
| Checkout        | Retrieve source                    | Stop on failure                      |
| Validate        | Check formatting and configuration | Stop if policy fails                 |
| Unit test       | Validate code behavior             | Stop if required tests fail          |
| SonarQube       | Code quality analysis              | Enforce configured quality gate      |
| Dependency scan | Identify vulnerable dependencies   | Apply security policy                |
| Docker build    | Produce image                      | Stop on build failure                |
| Trivy           | Scan image and configuration       | Apply vulnerability policy           |
| Image push      | Publish artifact                   | Stop on push failure                 |
| GitOps update   | Update desired release             | Require authorized change            |
| Argo CD sync    | Reconcile cluster state            | Monitor and verify                   |
| Smoke test      | Validate deployed behavior         | Trigger recovery procedure if needed |

### 26.2 Example Jenkins pipeline

```
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'checkout-service'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                  docker build \
                    -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Image Scan') {
            steps {
                sh '''
                  trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ${ECR_REPOSITORY}:${IMAGE_TAG}
                '''
            }
        }

        stage('Publish') {
            steps {
                echo 'Authenticate to ECR and push approved image'
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: '**/target/surefire-reports/*.xml'
        }

        failure {
            echo 'Pipeline failed. Review logs and failed stage.'
        }
    }
}
```

This is a structural example, not a ready-to-run production pipeline. Configure Jenkins credentials, ECR authentication, SonarQube quality-gate waiting, artifact naming, and registry publishing before use. Do not store AWS access keys directly in the Jenkinsfile.

### 26.3 Pipeline failure categories

Classify failures so teams can identify recurring delivery problems:

- Source checkout failure.
- Build dependency download failure.
- Unit test failure.
- Quality gate failure.
- Registry authentication failure.
- Image scan failure.
- Artifact publication failure.
- GitOps repository update failure.
- Argo CD reconciliation failure.
- Kubernetes scheduling or startup failure.
- Post-deployment smoke-test failure.

Track failure rate by category and service.

## 27. Immutable artifacts and build provenance

An immutable artifact is built once and promoted through environments rather than rebuilt differently for each environment.

### 27.1 Unsafe approach

```
Build image for development
          |
          v
Rebuild for staging
          |
          v
Rebuild for production
```

Each build could contain different dependencies or source changes.

### 27.2 Safer approach

```
Source commit
     |
     v
Single build
     |
     v
Image digest
     |
     +----> Development
     |
     +----> Staging
     |
     +----> Production
```

The same approved image digest should be promoted through the delivery process.

### 27.3 Image tagging

A Git commit-based tag can help trace a release:

```
IMAGE_TAG="$(git rev-parse --short HEAD)"
```

A stronger deployment reference uses the image digest:

```
ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/checkout-service@sha256:...
```

Tags can be moved unless registry policy prevents it. A digest identifies image content.

### 27.4 Provenance information

Capture:

- Repository URL.
- Commit SHA.
- Build ID.
- Build timestamp.
- Builder identity.
- Dependency versions.
- Container base image.
- Scan results.
- Artifact digest.
- Approval identity.
- Deployment environment.
- Release change reference.

Use signed provenance or attestations when supported by the delivery platform and organizational policy.

## 28. Deployment safety improvements

Deployment risk should be reduced before production traffic reaches a new version.

### 28.1 Rolling deployment

Kubernetes replaces old pods with new pods gradually.

Advantages:

- Straightforward deployment model.
- No separate environment required.
- Supports readiness-based rollout progression.

Risks:

- Old and new application versions may run simultaneously.
- Database schema compatibility is required.
- A rollout can stall due to scheduling, readiness, or image issues.

### 28.2 Canary deployment

A small portion of traffic is sent to the new version before wider rollout.

```
Incoming traffic
       |
       v
 Traffic router
    /       \
   v         v
Stable     Canary
 90%        10%
```

Increase traffic only when agreed metrics remain healthy.

### 28.3 Blue-green deployment

Two application environments are maintained:

```
Blue: Current version
Green: New version

       |
       v
Traffic switch
       |
       v
Green becomes active
```

Blue-green deployment requires capacity for both environments and a strategy for shared database and external side effects.

### 28.4 Pre-deployment checklist

- Correct source commit approved.
- Tests passed.
- Security policy passed.
- Image digest recorded.
- Helm values reviewed.
- Database compatibility checked.
- Required capacity available.
- Readiness and startup probes validated.
- Rollback or forward-recovery plan documented.
- Monitoring dashboards ready.
- Deployment owner available.
- Change window and communication completed where required.

### 28.5 Post-deployment validation

```
kubectl rollout status deployment/checkout-api \
  -n production \
  --timeout=300s

kubectl get pods \
  -n production \
  -l app=checkout-api

kubectl get endpointslices \
  -n production

kubectl get events \
  -n production \
  --sort-by=.lastTimestamp
```

A successful Kubernetes rollout does not prove the business workflow is healthy. Run application-level smoke tests and inspect customer-facing SLIs.

## 29. Argo CD and GitOps reliability

Argo CD continuously compares the desired configuration stored in Git with the live Kubernetes resources.

### 29.1 Desired state workflow

```
Developer change
       |
       v
GitOps repository
       |
       v
Pull request review
       |
       v
Merge approved change
       |
       v
Argo CD detects change
       |
       v
Reconciliation / sync
       |
       v
Kubernetes resources
       |
       v
Health verification
```

### 29.2 Preventive controls

- Protect production branches.
- Require review for critical changes.
- Restrict who can modify Argo CD applications.
- Use separate projects and permissions.
- Keep application configuration in version control.
- Review drift rather than blindly overriding it.
- Pin chart and dependency versions.
- Validate manifests before merge.
- Record sync failures and health status.
- Test recovery from incorrect desired state.

### 29.3 Example Argo CD Application

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout-api
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://git.example.com/platform/gitops.git
    targetRevision: main
    path: apps/checkout-api/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

This is an example. Review pruning, self-healing, sync windows, and project permissions carefully. Automated pruning can delete resources that are no longer present in Git.

### 29.4 Drift management

Drift occurs when live cluster state differs from the desired state.

Possible causes:

- Manual `kubectl` changes.
- Controller-managed fields.
- Emergency changes.
- Incorrect Helm values.
- Changes from another deployment system.

Review drift before reconciliation. For an emergency manual change, record it and update Git so that the approved desired state is preserved.

### 29.5 Validation commands

```
argocd app list

argocd app get checkout-api

argocd app diff checkout-api

argocd app history checkout-api
```

Confirm CLI context and permissions before running commands against production.

## 30. Terraform infrastructure reliability

Terraform changes can affect VPCs, EKS, IAM, RDS, networking, and shared platform infrastructure. A poorly reviewed change can have a large blast radius.

### 30.1 Preventive workflow

```
Terraform code change
         |
         v
       Pull request
         |
         v
   Formatting and validation
         |
         v
     Security review
         |
         v
     Terraform plan
         |
         v
   Plan review and approval
         |
         v
      Terraform apply
         |
         v
  Infrastructure verification
         |
         v
   Drift and state review
```

### 30.2 Recommended commands

```
terraform fmt -check -recursive

terraform init

terraform validate

terraform plan -out=tfplan

terraform show tfplan
```

Apply only the reviewed plan through an approved process:

```
terraform apply tfplan
```

The saved plan must be protected because it may contain sensitive values.

### 30.3 State management

Use a remote backend with controlled access, versioning, encryption, and state recovery procedures.

Example S3 backend:

```
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "production/eks/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Backend support and locking behavior depend on the installed Terraform version and backend configuration. Validate the exact configuration against the version in use. Do not assume that a lock configuration supported by one version works with every older version.

### 30.4 Terraform preventive checks

- Restrict state access.
- Enable bucket versioning.
- Protect state encryption keys.
- Avoid concurrent applies to the same state.
- Review resource replacement actions.
- Separate production and non-production state.
- Use modules with documented inputs and outputs.
- Pin provider versions.
- Review IAM and security group changes.
- Test module changes in non-production.
- Maintain a recovery process for state corruption.

### 30.5 Dangerous plan review

Pay special attention to:

- Resource replacement.
- Database deletion.
- Subnet or route changes.
- Security group rule changes.
- IAM policy replacement.
- EKS node group replacement.
- Persistent storage changes.
- Load balancer and DNS changes.

A plan is not safe merely because Terraform generated it successfully.

## 31. DevSecOps preventive controls

Security controls should be part of the software delivery process rather than a final manual inspection.

### 31.1 Recommended security stages

```
Source code
    |
    v
Static code analysis
  SonarQube
    |
    v
Dependency checks
    |
    v
Container build
    |
    v
Container image scan
    Trivy
    |
    v
Configuration and IaC checks
    |
    v
Artifact approval
    |
    v
Deployment
    |
    v
Runtime security monitoring
```

Veracode can be included according to the organization's application security program and licensing.

### 31.2 SonarQube

Review:

- Bugs.
- Vulnerabilities.
- Code smells.
- Coverage.
- Duplicated code.
- Quality gate status.

Example:

```
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=checkout-service \
  -Dsonar.host.url="$SONAR_HOST_URL"
```

Authenticate through a securely managed token or credential mechanism. Do not place tokens in source code or shell history.

### 31.3 Trivy

Example image scan:

```
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/checkout-service:RELEASE_TAG
```

Review:

- Vulnerability severity.
- Fix availability.
- Base image.
- Package versions.
- Exception approvals.
- Scan database freshness.
- False positives and accepted risk.

### 31.4 Security gate policy

Define:

- Which findings block releases.
- Which findings require approval.
- Who may accept risk.
- How exceptions expire.
- How remediation is tracked.
- How evidence is retained.

Do not set a blanket policy that ignores every vulnerability without considering exploitability, exposure, and business risk.

### 31.5 Veracode

Where Veracode is used:

- Define the application profile.
- Configure the appropriate scan type.
- Integrate scan results with the release process.
- Set quality or policy gates.
- Track findings to remediation.
- Retain evidence for audits.

The exact integration depends on the Veracode product and the organization's license.

## 32. Observability baseline

Observability helps engineering teams understand what is happening inside an application and its infrastructure by examining its outputs.

For this platform, the primary observability stack is:

- Prometheus: Metrics collection and alert evaluation.
- Grafana: Dashboards and visualization.
- ELK Stack: Log collection, processing, indexing, and search.

The goal is not to collect every possible metric or log. The goal is to collect the evidence needed to detect failures, understand their impact, and make operational decisions.

### 32.1 Three important observability signals

| Signal  | What it tells us                          | Example                             |
| ------- | ----------------------------------------- | ----------------------------------- |
| Metrics | Numeric behavior over time                | CPU usage, latency, error rate      |
| Logs    | Individual events and application details | Payment timeout, database exception |
| Traces  | Request path across services              | End-to-end request timing           |

This implementation uses Prometheus, Grafana, and ELK. Distributed tracing is not part of the configured stack in this chapter.

### 32.2 Recommended monitoring architecture

```
                EKS Cluster
                    |
          +---------+---------+
          |                   |
          v                   v
    Application Pods       Worker Nodes
          |                   |
          v                   v
    Application metrics   Node Exporter
          |                   |
          +---------+---------+
                    |
                    v
                Prometheus
                    |
          +---------+---------+
          |                   |
          v                   v
      Alert Rules          Grafana
          |                   |
          v                   v
    Alertmanager          Dashboards
          |
          v
   Notification channel


    Application container logs
                 |
                 v
             Log agent
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

The exact log agent and deployment model must be defined separately. The architecture above shows the intended flow, not a complete installation configuration.

### 32.3 Define monitoring ownership

For each critical service, identify:

- Service owner.
- Metrics owner.
- Dashboard owner.
- Alert owner.
- Log pipeline owner.
- Escalation contact.
- Runbook location.

An alert without an owner and response procedure is an incomplete operational control.

## 33. Prometheus alert improvements

Alerts should identify meaningful risks and give responders enough information to take action.

### 33.1 Recommended alert categories

| Category       | Example alert                    |
| -------------- | -------------------------------- |
| Availability   | Service has no healthy endpoints |
| Latency        | High request latency             |
| Errors         | HTTP 5xx rate elevated           |
| Saturation     | CPU or memory pressure           |
| Kubernetes     | Pod restart rate increasing      |
| Scheduling     | Pods remain Pending              |
| Database       | Connection utilization high      |
| Messaging      | RabbitMQ queue backlog growing   |
| Kafka          | Consumer lag increasing          |
| Storage        | Persistent volume capacity low   |
| Infrastructure | Node NotReady                    |

### 33.2 Example Prometheus alert rules

```
groups:
  - name: application-reliability
    rules:

      - alert: HighPodRestartRate
        expr: |
          increase(
            kube_pod_container_status_restarts_total{
              namespace="production"
            }[15m]
          ) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod restart rate is high"
          description: "A production pod has restarted repeatedly."

      - alert: PodNotReady
        expr: |
          kube_pod_status_ready{
            namespace="production",
            condition="true"
          } == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Production pod is not ready"
          description: "Check pod events, probes, scheduling and logs."
```

These rules require compatible metric names and labels from the installed kube-state-metrics version. Validate the queries in Prometheus before enabling them.

### 33.3 Example service error-rate alert

If the application exports a counter named `http_requests_total` with `status` and `service` labels:

```
- alert: HighServiceErrorRate
  expr: |
    (
      sum by (service) (
        rate(http_requests_total{
          status=~"5.."
        }[5m])
      )
      /
      clamp_min(
        sum by (service) (
          rate(http_requests_total[5m])
        ),
        0.001
      )
    ) > 0.05
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Service error rate is above 5%"
    description: "Investigate application errors and dependencies."
```

This example assumes the application exposes these exact metric and label names. Production alerting should also account for traffic volume and minimum request thresholds to avoid noisy alerts on low-traffic services.

### 33.4 Alert quality checklist

- Alert represents a meaningful failure or risk.
- Threshold has a documented reason.
- Evaluation window is appropriate.
- Alert has a clear severity.
- Owner and escalation path exist.
- Runbook link is available.
- Duplicate alerts are controlled.
- Recovery and resolution behavior are understood.
- Alert has been tested.
- Alert noise is reviewed regularly.

## 34. Grafana dashboard improvements

A dashboard should help an engineer answer operational questions quickly.

### 34.1 Recommended dashboard levels

Platform overview

- Cluster health.
- Node readiness.
- Node CPU and memory.
- Pending pods.
- Namespace resource usage.
- Cluster capacity.
- Critical alerts.

Service dashboard

- Request rate.
- Error rate.
- Latency percentiles.
- Pod replicas.
- Restarts.
- CPU and memory.
- Dependency health.

Database dashboard

- Connections.
- CPU.
- Storage.
- Query latency.
- Lock waits.
- Replication or failover health where applicable.

Messaging dashboard

- RabbitMQ queue depth.
- Unacknowledged messages.
- Consumer count.
- Kafka consumer lag.
- Broker health.
- Disk capacity.

### 34.2 Example dashboard layout

```
+------------------------------------------------------+
|                PLATFORM OVERVIEW                     |
+----------------------+-------------------------------+
| Cluster Status       | Node Readiness                |
+----------------------+-------------------------------+
| CPU Utilization      | Memory Utilization            |
+----------------------+-------------------------------+
| Pending Pods         | Pod Restarts                  |
+----------------------+-------------------------------+
| Service Error Rate   | Request Latency               |
+----------------------+-------------------------------+
| Database Health      | Messaging Health              |
+----------------------+-------------------------------+
| Active Alerts        | Recent Incidents              |
+----------------------+-------------------------------+
```

### 34.3 Dashboard design rules

- Use consistent service and environment labels.
- Include time-range controls.
- Show units and meaningful legends.
- Avoid excessive panels.
- Use thresholds based on service objectives.
- Provide links to relevant runbooks.
- Include the last deployment or release identifier where available.
- Avoid high-cardinality labels that increase metric storage costs.

### 34.4 Dashboard validation

For each critical panel:

1. Confirm the query returns data.
2. Confirm labels identify the correct service and environment.
3. Compare results with a known test.
4. Check behavior during a controlled failure.
5. Confirm empty or missing data is not shown as healthy.
6. Verify permissions and access.

## 35. ELK log pipeline improvements

Logs are valuable for diagnosing application errors, dependency failures, and unexpected behavior.

### 35.1 Recommended logging architecture

```
Application containers
         |
         v
Container log files
         |
         v
Log collection agent
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

### 35.2 Structured logging

Prefer structured JSON logs instead of inconsistent free-text messages.

Example:

```
{
  "timestamp": "2026-11-04T10:30:00Z",
  "level": "ERROR",
  "service": "checkout-api",
  "environment": "production",
  "request_id": "req-12345",
  "message": "Payment dependency timeout",
  "dependency": "payment-service",
  "duration_ms": 3000
}
```

The values are illustrative. Use the actual request correlation mechanism and ensure sensitive information is not logged.

### 35.3 Log quality standards

Each log event should include appropriate fields:

- Timestamp.
- Severity.
- Service name.
- Environment.
- Application version.
- Request or correlation ID.
- Event name.
- Error category.
- Relevant diagnostic context.

Never log passwords, access tokens, private keys, payment card data, or unnecessary personal information.

### 35.4 Log retention

Define retention based on:

- Operational troubleshooting requirements.
- Security and audit requirements.
- Legal or regulatory obligations.
- Storage capacity and cost.
- Data classification.

Separate hot searchable data from longer-term archival data where appropriate.

### 35.5 Log pipeline failure controls

Monitor:

- Collection agent health.
- Logstash input and output rates.
- Processing errors.
- Elasticsearch cluster health.
- Disk watermark status.
- Index growth.
- Ingestion delay.
- Dropped or rejected events.

A service may be healthy while its logging pipeline is failing. Treat observability infrastructure as production infrastructure.

## 36. Alert quality and incident detection

An alerting system should distinguish between symptoms, causes, and business impact.

Example:

| Alert                     | Meaning                               | Response                                     |
| ------------------------- | ------------------------------------- | -------------------------------------------- |
| High CPU                  | Resource pressure                     | Check load, throttling, and recent changes   |
| Pod restart               | Container repeatedly terminates       | Inspect previous logs and events             |
| High 5xx rate             | Requests are failing                  | Check application and dependency errors      |
| High latency              | Requests are slow                     | Examine saturation and dependency latency    |
| Queue backlog             | Processing is falling behind          | Inspect consumers and message rates          |
| Database connections high | Connection capacity is under pressure | Review pool configuration and database usage |

### 36.1 Symptoms versus causes

A high CPU alert may be a symptom of:

- Increased traffic.
- Infinite loop.
- Retry storm.
- Expensive query.
- Memory pressure causing extra work.
- Unexpected background processing.

Do not automatically increase CPU limits without finding the underlying cause.

### 36.2 Alert lifecycle

```
Metric crosses threshold
          |
          v
Prometheus evaluates rule
          |
          v
Alertmanager receives alert
          |
          v
Grouping and routing
          |
          v
Responder notified
          |
          v
Runbook investigation
          |
          v
Incident resolved
          |
          v
Alert reviewed for quality
```

### 36.3 Monthly alert review

Review:

- Alerts that fired.
- Alerts that were ignored.
- False positives.
- Duplicate alerts.
- Alerts without action.
- Incidents without corresponding alerts.
- Missing ownership.
- Thresholds that no longer match workload behavior.

## 37. IAM and IRSA preventive improvements

Kubernetes workloads should not use broad, shared AWS permissions.

Use IAM Roles for Service Accounts (IRSA) where applicable, or the supported EKS Pod Identity approach selected by the platform.

### 37.1 Example access design

```
checkout-api Pod
      |
      v
Kubernetes ServiceAccount
      |
      v
IAM role association
      |
      v
Least-privilege IAM policy
      |
      v
Specific AWS resource access
```

### 37.2 Example ServiceAccount

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: checkout-api-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/checkout-api-role
```

### 37.3 Example IAM policy concept

A service that only needs to read selected S3 objects should not receive administrator permissions.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::company-checkout-config/production/*"
      ]
    }
  ]
}
```

This policy is only an example. Add required permissions based on the actual application behavior and resource configuration.

### 37.4 Preventive checks

- Review trust policy conditions.
- Restrict role assumption to intended service accounts.
- Remove unused permissions.
- Avoid wildcard actions and resources where possible.
- Review cross-account access.
- Audit credential use.
- Test access from the intended pod.
- Confirm that unrelated pods cannot assume the role.

### 37.5 Validation

```
kubectl get serviceaccount checkout-api-sa \
  -n production \
  -o yaml
```

From an appropriately configured test pod, inspect the AWS caller identity:

```
aws sts get-caller-identity
```

Confirm that the returned role is the intended workload role. Never print or expose temporary credentials.

## 38. Secret management improvements

Secrets include:

- Database credentials.
- API tokens.
- Signing keys.
- TLS private keys.
- Messaging credentials.
- Third-party service credentials.

### 38.1 Unsafe practices

Avoid:

- Committing secrets to Git.
- Hardcoding credentials in Dockerfiles.
- Passing credentials in command-line arguments.
- Printing secrets in pipeline logs.
- Sharing production secrets across unrelated services.
- Using one permanent credential for every environment.

### 38.2 Recommended design

```
Approved secret store
         |
         v
Controlled workload identity
         |
         v
Secret retrieval
         |
         v
Application configuration
         |
         v
Runtime use
```

Use a managed secret store such as AWS Secrets Manager or an approved external secrets integration.

### 38.3 Rotation procedure

1. Identify the secret and all consumers.
2. Confirm whether overlapping credentials are supported.
3. Create or enable the replacement credential.
4. Update the secret store.
5. Refresh or restart workloads safely.
6. Verify all consumers.
7. Revoke the old credential.
8. Check logs and authentication failures.
9. Record completion evidence.

### 38.4 Rotation acceptance criteria

- The application uses the new credential.
- No unexpected authentication errors occur.
- Old credentials are revoked where appropriate.
- Secret values are not exposed in logs.
- The rotation is documented and repeatable.

## 39. Kubernetes security improvements

Security should be applied consistently to namespaces, pods, workloads, and cluster access.

### 39.1 Pod security controls

Review:

- Non-root execution.
- Read-only root filesystem where practical.
- Dropped Linux capabilities.
- No privileged containers unless explicitly approved.
- No host PID or host network unless required.
- Restricted hostPath usage.
- Seccomp profile.
- ServiceAccount token mounting.
- Image provenance and scanning.

### 39.2 Example security context

```
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

Container-level and pod-level security contexts have different fields. Place each setting at the correct level and test compatibility with the application.

### 39.3 Network isolation

Use Kubernetes NetworkPolicies where supported by the installed network plugin.

Example policy allowing ingress only from a specific namespace:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: checkout-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: checkout-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              access: checkout
      ports:
        - protocol: TCP
          port: 8080
```

This is an example ingress policy. Confirm namespace labels, required sources, egress dependencies, and CNI enforcement before applying it.

### 39.4 Access review

```
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterroles
kubectl get clusterrolebindings
```

Review:

- Excessive cluster-admin access.
- Unused service accounts.
- Broad namespace permissions.
- Human access to production.
- Service accounts with unnecessary token mounting.
- Emergency access procedures.

## 40. Software supply-chain improvements

A secure deployment process needs to establish where an artifact came from and whether it was approved.

### 40.1 Supply-chain control points

```
Source repository
       |
       v
Protected branch
       |
       v
Reviewed code
       |
       v
Controlled build
       |
       v
Dependency and image scans
       |
       v
Artifact digest
       |
       v
Approval and provenance
       |
       v
Production deployment
```

### 40.2 Preventive controls

- Protect release branches.
- Require pull request review.
- Pin dependencies.
- Use trusted base images.
- Scan dependencies and images.
- Restrict registry write permissions.
- Preserve build logs.
- Record artifact digests.
- Verify artifacts before deployment.
- Restrict production deployment permissions.
- Review third-party CI/CD actions and plugins.

### 40.3 Image lifecycle

Define:

- Image owner.
- Supported base image.
- Patch schedule.
- Vulnerability policy.
- Retention policy.
- Immutable release reference.
- Approval process.
- Retirement process.

A vulnerability scan performed once at build time does not guarantee the image remains secure. New vulnerabilities may be discovered after the release.

## 41. Automation candidates

Automation should remove repetitive work without removing necessary human approval.

| Activity             | Automation opportunity                | Safety control                 |
| -------------------- | ------------------------------------- | ------------------------------ |
| Deployment preflight | Validate configuration before release | Fail closed on required checks |
| Backup verification  | Check backup metadata                 | Restore test remains necessary |
| Evidence capture     | Collect read-only diagnostic data     | Restrict data access           |
| Resource audit       | Identify missing requests and limits  | Human review before changes    |
| Certificate expiry   | Alert before expiration               | Validate renewal               |
| Queue backlog        | Detect sustained growth               | Alert and runbook              |
| Access review        | Identify excessive permissions        | Owner approval                 |
| Drift detection      | Report Terraform or GitOps drift      | Review before reconciliation   |
| Runbook validation   | Test command and link availability    | Avoid destructive execution    |

### 41.1 Automation safety levels

Level 1 — Read-only: Collect status, logs, metrics, and configuration.

Level 2 — Recommendation: Generate suggested changes for review.

Level 3 — Approved execution: Perform changes after explicit approval.

Level 4 — Controlled remediation: Automatically remediate only narrow, well-tested, reversible conditions with guardrails.

Start with read-only automation. Promote workflows only after testing their failure behavior and access boundaries.

## 42. Deployment preflight script

This script checks basic Kubernetes deployment conditions before a release. It does not perform a deployment.

```
#!/bin/bash

set -u

NAMESPACE="${1:-production}"
DEPLOYMENT="${2:-checkout-api}"

echo "=================================="
echo "Production Deployment Preflight"
echo "=================================="

echo "Namespace: $NAMESPACE"
echo "Deployment: $DEPLOYMENT"

if ! kubectl get namespace "$NAMESPACE" >/dev/null 2>&1; then
    echo "ERROR: Namespace does not exist"
    exit 1
fi

if ! kubectl get deployment "$DEPLOYMENT" \
    -n "$NAMESPACE" >/dev/null 2>&1; then
    echo "ERROR: Deployment does not exist"
    exit 1
fi

echo
echo "Checking deployment rollout status..."

if ! kubectl rollout status deployment/"$DEPLOYMENT" \
    -n "$NAMESPACE" \
    --timeout=30s; then
    echo "WARNING: Current deployment is not healthy"
    exit 1
fi

echo
echo "Checking pods..."

kubectl get pods \
    -n "$NAMESPACE" \
    -l "app=$DEPLOYMENT"

echo
echo "Checking recent events..."

kubectl get events \
    -n "$NAMESPACE" \
    --sort-by=.lastTimestamp | tail -20

echo
echo "Preflight checks completed."
```

### 42.1 Script limitations

- The label selector assumes pods use `app=DEPLOYMENT_NAME`.
- The rollout status check examines the current deployment, not a proposed new release.
- It does not check business transactions.
- It does not check database compatibility.
- It does not validate image provenance.
- It does not approve or execute a release.

Add environment-specific checks before treating this as a production release gate.

## 43. Backup metadata verification script

A successful backup job does not automatically prove that data can be recovered.

A preventive backup process should verify:

- Backup job completion.
- Backup timestamp.
- Backup retention.
- Backup encryption.
- Backup destination.
- Backup size and metadata.
- Recovery-point objectives.
- Restore-test evidence.

### 43.1 Example RDS backup metadata check

```
#!/bin/bash

set -euo pipefail

DB_INSTANCE="${1:?Usage: $0 <db-instance-id>}"
REGION="${AWS_REGION:-ap-south-1}"

echo "Checking RDS backup configuration..."

aws rds describe-db-instances \
  --db-instance-identifier "$DB_INSTANCE" \
  --region "$REGION" \
  --query 'DBInstances[0].{
    Identifier:DBInstanceIdentifier,
    BackupRetentionDays:BackupRetentionPeriod,
    LatestRestorableTime:LatestRestorableTime,
    StorageEncrypted:StorageEncrypted,
    MultiAZ:MultiAZ
  }' \
  --output table

echo
echo "Checking recent automated backups..."

aws rds describe-db-instance-automated-backups \
  --db-instance-identifier "$DB_INSTANCE" \
  --region "$REGION" \
  --query 'DBInstanceAutomatedBackups[*].{
    Identifier:DBInstanceIdentifier,
    Status:Status,
    Retention:BackupRetentionPeriod,
    RestoreWindow:RestoreWindow
  }' \
  --output table
```

### 43.2 Example S3 backup metadata check

```
#!/bin/bash

set -euo pipefail

BUCKET="${1:?Usage: $0 <bucket-name>}"
PREFIX="${2:-backups/}"
REGION="${AWS_REGION:-ap-south-1}"

echo "Checking S3 backup objects..."

aws s3api list-objects-v2 \
  --bucket "$BUCKET" \
  --prefix "$PREFIX" \
  --region "$REGION" \
  --query 'Contents[*].{
    Key:Key,
    Size:Size,
    LastModified:LastModified
  }' \
  --output table
```

### 43.3 Important limitations

These scripts inspect metadata. They do not prove that:

- A database backup can be restored.
- An object is not corrupted.
- An application can use restored data.
- Encryption keys are available.
- Cross-account recovery works.
- The restored environment meets the required RTO and RPO.

A restore exercise is mandatory for proving recoverability.

### 43.4 Backup verification acceptance criteria

- Backup configuration meets approved retention requirements.
- Backup jobs complete successfully.
- Recovery points are available within the required window.
- Encryption and access controls are verified.
- Restore tests are performed on a defined schedule.
- Application-level validation succeeds after restoration.
- Evidence is recorded and reviewed.

## 44. Production evidence capture script

During incidents, reliability reviews, and preventive audits, engineers need consistent evidence.

A read-only evidence collection script reduces the risk of missing basic diagnostic information.

### 44.1 Evidence capture script

```
#!/bin/bash

set -euo pipefail

NAMESPACE="${1:-production}"
OUTPUT_DIR="${2:-evidence-$(date +%Y%m%d-%H%M%S)}"

mkdir -p "$OUTPUT_DIR"

echo "Collecting Kubernetes evidence..."
echo "Namespace: $NAMESPACE"
echo "Output: $OUTPUT_DIR"

kubectl get nodes -o wide \
  > "$OUTPUT_DIR/nodes.txt"

kubectl get pods -n "$NAMESPACE" -o wide \
  > "$OUTPUT_DIR/pods.txt"

kubectl get deployments -n "$NAMESPACE" \
  > "$OUTPUT_DIR/deployments.txt"

kubectl get services -n "$NAMESPACE" \
  > "$OUTPUT_DIR/services.txt"

kubectl get endpointslices -n "$NAMESPACE" \
  > "$OUTPUT_DIR/endpointslices.txt"

kubectl get events -n "$NAMESPACE" \
  --sort-by=.lastTimestamp \
  > "$OUTPUT_DIR/events.txt"

kubectl get hpa -n "$NAMESPACE" \
  > "$OUTPUT_DIR/hpa.txt"

kubectl get pdb -n "$NAMESPACE" \
  > "$OUTPUT_DIR/pdb.txt"

kubectl top pods -n "$NAMESPACE" \
  > "$OUTPUT_DIR/pod-resource-usage.txt" 2>&1 || true

kubectl top nodes \
  > "$OUTPUT_DIR/node-resource-usage.txt" 2>&1 || true

echo "Evidence collection completed."
echo "Files saved in: $OUTPUT_DIR"
```

### 44.2 Evidence handling requirements

Before collecting production evidence:

- Confirm the active Kubernetes context.
- Confirm the namespace.
- Avoid collecting secrets.
- Restrict access to output files.
- Redact customer and personal data.
- Record the collection time.
- Preserve original evidence where required.
- Follow the organization's retention policy.

### 44.3 Verify the context first

```
kubectl config current-context
kubectl get namespaces
```

Never assume that the active context points to the intended cluster.

## 45. Weekly reliability review

A weekly review gives platform and service owners a regular opportunity to detect emerging problems before they become incidents.

### 45.1 Weekly review agenda

| Topic               | Questions                                              |
| ------------------- | ------------------------------------------------------ |
| Incidents           | Which incidents occurred this week?                    |
| Availability        | Did critical services meet their objectives?           |
| Latency             | Did latency increase?                                  |
| Errors              | Which services had elevated error rates?               |
| Kubernetes          | Were there restarts, evictions, or pending pods?       |
| Capacity            | Is there sufficient CPU, memory, and IP capacity?      |
| Database            | Are connections, storage, or query latency increasing? |
| Messaging           | Are queue depth and consumer lag healthy?              |
| Deployments         | Which releases failed or required rollback?            |
| Security            | Were critical findings discovered?                     |
| Backups             | Were backup jobs successful?                           |
| Improvement backlog | Which actions are overdue or blocked?                  |

### 45.2 Example weekly report

```
WEEKLY PLATFORM RELIABILITY REVIEW

Review period:
Platform:
Cluster:
Prepared by:

1. Service health
   - Critical service availability:
   - Error-rate trends:
   - Latency trends:

2. Kubernetes health
   - Node readiness:
   - Pod restart trends:
   - Pending pods:
   - Resource pressure:

3. Database health
   - Connection utilization:
   - Storage:
   - Slow queries:
   - Backup status:

4. Messaging health
   - RabbitMQ queue depth:
   - Kafka consumer lag:
   - Consumer availability:

5. Delivery health
   - Deployment count:
   - Failed deployments:
   - Rollbacks:
   - Pipeline failures:

6. Security
   - Critical findings:
   - Overdue remediation:
   - Access review findings:

7. Improvement actions
   - Completed:
   - In progress:
   - Blocked:
   - Overdue:

8. Decisions and owners
   - Decision:
   - Owner:
   - Target date:
```

### 45.3 Weekly review outputs

The review should produce:

- Updated improvement register.
- New risk records.
- Assigned action owners.
- Escalations for overdue high-risk items.
- Agreed changes to monitoring or runbooks.
- Follow-up items for the next review.

## 46. Monthly platform reliability review

The monthly review should examine longer-term patterns rather than individual alerts.

### 46.1 Monthly metrics

| Metric                 | What it measures                                     |
| ---------------------- | ---------------------------------------------------- |
| Availability           | Service availability against target                  |
| Error rate             | Failed requests                                      |
| P95/P99 latency        | Tail response time                                   |
| Deployment frequency   | How often releases occur                             |
| Change failure rate    | Releases causing incidents or remediation            |
| MTTR                   | Time to restore service, using a defined calculation |
| Alert noise            | Unactionable or duplicate alerts                     |
| Backup success rate    | Successful backup operations                         |
| Restore-test success   | Recovery exercises that passed                       |
| Vulnerability backlog  | Outstanding security findings                        |
| Improvement completion | Verified actions completed                           |

Define each metric consistently before comparing months. For example, change failure rate should use an agreed definition of what constitutes a failed change.

### 46.2 Monthly questions

1. Are incidents recurring?
2. Are the same services repeatedly failing?
3. Are deployments becoming safer?
4. Are alerting improvements reducing detection time?
5. Are capacity trends creating future risks?
6. Are backup and restore tests meeting objectives?
7. Are security findings being resolved on time?
8. Are critical improvements blocked by dependencies?
9. Are platform changes reducing operational effort?
10. Which risks require leadership attention?

### 46.3 Monthly report example

```
MONTHLY PLATFORM IMPROVEMENT REPORT

Month:
Platform:
Owner:

Reliability:
- Availability:
- Error rate:
- P95 latency:
- P99 latency:

Incident management:
- Total incidents:
- Critical incidents:
- Recurring incidents:
- Mean time to detect:
- Mean time to restore:

Delivery:
- Deployment frequency:
- Change failure rate:
- Rollback count:
- Pipeline failure rate:

Infrastructure:
- Cluster capacity:
- Node utilization:
- Database capacity:
- Messaging backlog:

Security:
- Critical vulnerabilities:
- High vulnerabilities:
- Overdue actions:

Improvement:
- Actions opened:
- Actions completed:
- Actions overdue:
- Actions blocked:

Top risks:
1.
2.
3.

Next month's priorities:
1.
2.
3.
```

## 47. Quarterly reliability game day

A game day is a controlled exercise that tests whether the platform behaves as expected when a component fails.

The purpose is not to cause disruption. It is to validate resilience, monitoring, response procedures, and recovery readiness.

### 47.1 Game day rules

Before running an exercise:

- Obtain required approval.
- Define the test scope.
- Identify affected services.
- Set customer-impact limits.
- Prepare an abort condition.
- Confirm responder availability.
- Verify backup and recovery readiness.
- Notify relevant teams.
- Record baseline metrics.
- Ensure the test environment is appropriate.

Do not execute destructive production experiments without explicit authorization and a tested containment plan.

### 47.2 Example game day schedule

| Time  | Activity                       |
| ----- | ------------------------------ |
| 09:00 | Briefing and safety checks     |
| 09:15 | Capture baseline metrics       |
| 09:30 | Execute controlled failure     |
| 09:45 | Observe detection and response |
| 10:00 | Restore normal conditions      |
| 10:15 | Validate service behavior      |
| 10:30 | Review findings                |
| 11:00 | Assign improvement actions     |

### 47.3 Game day success criteria

- Expected alert fires.
- Responder identifies the correct failure.
- Blast radius stays within the approved boundary.
- Recovery follows the documented procedure.
- Customer-facing service remains within agreed limits, or the expected degraded behavior occurs.
- Evidence is recorded.
- Findings become owned improvement actions.

## 48. Improvement ownership model

An improvement register needs clear accountability.

### 48.1 Example RACI

RACI means Responsible, Accountable, Consulted, and Informed.

| Activity                        | Platform | Service owner | Security | Database | Engineering lead |
| ------------------------------- | -------- | ------------- | -------- | -------- | ---------------- |
| EKS capacity review             | R        | C             | I        | I        | A                |
| Application timeout improvement | C        | R             | I        | C        | A                |
| IAM policy review               | R        | C             | A/R      | I        | I                |
| RDS recovery exercise           | C        | I             | I        | R        | A                |
| CI/CD security gates            | R        | C             | A/C      | I        | I                |
| Service SLO definition          | C        | R             | I        | C        | A                |
| Incident follow-up              | R        | R             | C        | C        | A                |

This is an illustrative model. Assign one accountable owner per activity and adapt the matrix to the organization's actual roles.

### 48.2 Ownership rules

- Every action has one accountable owner.
- Supporting teams are named.
- Due dates are agreed.
- Dependencies are visible.
- Blocked actions have an escalation path.
- Closure requires evidence.
- Residual risk is documented.

## 49. Improvement completion criteria

An action is complete only when its intended result has been demonstrated.

### 49.1 Completion checklist

- Root risk or failure mode is clearly described.
- Change design has been reviewed.
- Required approvals are complete.
- Implementation is complete.
- Tests have passed.
- Security implications have been reviewed.
- Monitoring has been updated.
- Runbooks have been updated.
- Ownership is documented.
- Acceptance criteria are satisfied.
- Effectiveness has been checked.
- Residual risks are documented.
- Evidence is attached to the improvement record.

### 49.2 Example

Finding: Checkout requests time out during temporary Payment Service latency.

Weak completion statement:

```
Added retry logic.
Ticket closed.
```

Strong completion statement:

```
Implemented bounded retries with exponential backoff and jitter.

Added a payment dependency timeout.

Added a circuit breaker and dependency-specific concurrency limit.

Tested Payment Service latency and failure in non-production.

Verified that retry traffic stays within the configured limit.

Verified that unrelated checkout operations remain responsive.

Updated Grafana panels and the checkout runbook.

Reviewed the result with the service owner.

Residual risk: payment-provider outage still prevents payment authorization.
```

The second statement explains what changed and how the outcome was validated.

## 50. Practical project — Preventive improvement for 20+ microservices

This project applies the chapter to a realistic EKS-based microservices environment.

### 50.1 Project objective

Build a preventive improvement program that:

- Reduces repeated production incidents.
- Improves deployment reliability.
- Strengthens Kubernetes workloads.
- Improves dependency handling.
- Improves observability.
- Strengthens security.
- Tests backup and recovery.
- Establishes measurable ownership.

### 50.2 Example service inventory

| Service            | Criticality | Main dependency            | Improvement focus                 |
| ------------------ | ----------- | -------------------------- | --------------------------------- |
| User               | High        | User database              | Availability and authentication   |
| Product Catalog    | Medium      | Product database           | Cache resilience                  |
| Cart               | High        | Redis                      | Cache and session behavior        |
| Order              | Critical    | Payment, inventory         | Idempotency and consistency       |
| Payment            | Critical    | Payment provider, RDS      | Timeouts and duplicate protection |
| Inventory          | Critical    | Inventory database         | Reservation consistency           |
| Notification       | Medium      | RabbitMQ                   | Retry and dead-letter handling    |
| Search             | Medium      | Search backend             | Degraded operation                |
| Shipping           | High        | External shipping provider | Dependency timeouts               |
| Reporting          | Low         | Kafka                      | Lag and replay                    |
| Remaining services | Per owner   | Per dependency             | Individual risk assessment        |

This is a sample inventory. Replace it with the actual service list and dependency map.

### 50.3 Project implementation stages

Stage 1 — Baseline

- Inventory all services.
- Identify owners.
- Map dependencies.
- Record current SLIs and SLOs.
- Collect incident history.
- Review current alerts.

Stage 2 — Risk assessment

- Identify recurring failures.
- Identify shared dependency risks.
- Review resource configuration.
- Review backup and restore coverage.
- Review security findings.
- Prioritize the improvement backlog.

Stage 3 — Kubernetes improvements

- Add or correct probes.
- Review requests and limits.
- Review PDBs.
- Improve topology spread.
- Review HPA.
- Validate node capacity.
- Review quotas and network policies.

Stage 4 — Application resilience

- Review timeouts.
- Introduce bounded retries.
- Add circuit breakers where appropriate.
- Implement idempotency.
- Review asynchronous workflows.
- Define degraded modes.

Stage 5 — Database and messaging

- Review RDS connection budgets.
- Test restore procedures.
- Review Redis cache behavior.
- Review RabbitMQ backlog and retry handling.
- Review Kafka lag and consumer behavior.

Stage 6 — Delivery and security

- Improve pipeline gates.
- Promote immutable artifacts.
- Validate GitOps ownership.
- Review Terraform safety.
- Improve IAM and IRSA.
- Review secrets and vulnerability remediation.

Stage 7 — Observability

- Build service dashboards.
- Improve SLI-based alerting.
- Review log quality.
- Test alert routing.
- Document runbooks.

Stage 8 — Validation

- Run game days.
- Verify acceptance criteria.
- Measure improvement.
- Record residual risk.
- Review outcomes with owners.

### 50.4 Expected project deliverables

```
platform-improvement/
|
+-- inventory/
|   +-- service-inventory.md
|   +-- dependency-map.md
|   +-- ownership-matrix.md
|
+-- reliability/
|   +-- sli-slo-register.md
|   +-- reliability-baseline.md
|   +-- capacity-review.md
|
+-- kubernetes/
|   +-- workload-baseline.yaml
|   +-- pdb.yaml
|   +-- hpa.yaml
|   +-- resource-quota.yaml
|
+-- observability/
|   +-- prometheus-alerts.yaml
|   +-- grafana-dashboard-requirements.md
|   +-- elk-logging-standard.md
|
+-- security/
|   +-- iam-review.md
|   +-- secrets-rotation.md
|   +-- vulnerability-policy.md
|
+-- automation/
|   +-- deployment-preflight.sh
|   +-- backup-verification.sh
|   +-- evidence-capture.sh
|
+-- governance/
|   +-- improvement-register.csv
|   +-- weekly-review.md
|   +-- monthly-review.md
|   +-- game-day-plan.md
|
+-- evidence/
    +-- test-results/
    +-- restore-exercises/
    +-- game-day-reports/
```
## 51. Interview questions and answers

### Q1. What is the difference between corrective and preventive actions?

Answer:

Corrective action fixes an existing problem or defect. Preventive action reduces the likelihood of a similar failure happening again.

For example, restarting a failed pod is a corrective response. Identifying why it repeatedly crashes, correcting the application issue, and adding appropriate monitoring are preventive improvements.

### Q2. How do you convert an RCA finding into an engineering task?

Answer:

I first identify the failure mode and its customer impact. Then I collect supporting evidence, determine the underlying risk, and define a measurable action.

I assign an owner, priority, due date, and acceptance criteria. After implementation, I validate the result through testing and monitoring before closing the action.

### Q3. How do you prioritize platform improvements?

Answer:

I consider customer impact, likelihood of occurrence, blast radius, security implications, detectability, recovery difficulty, and implementation effort.

Critical risks affecting payment, checkout, identity, or data integrity receive higher priority than minor operational improvements.

### Q4. How do you prevent Kubernetes pods from repeatedly restarting?

Answer:

I inspect pod events, previous container logs, exit codes, resource usage, and probe configuration.

I check for:

- OOMKilled events.
- Application exceptions.
- Incorrect startup or liveness probes.
- CPU or memory pressure.
- Dependency timeouts.
- Configuration errors.

Then I correct the underlying problem, validate the fix, and add monitoring to detect recurrence.

### Q5. How do you prevent resource-related incidents in EKS?

Answer:

I review CPU and memory requests and limits, node capacity, pod distribution, autoscaling, namespace quotas, and historical utilization.

I also monitor pending pods, node pressure, OOM kills, CPU throttling, and subnet IP availability.

I validate capacity during normal traffic, peak traffic, and node failure scenarios.

### Q6. How do you improve microservice resilience?

Answer:

I use dependency-specific timeouts, bounded retries, exponential backoff, circuit breakers, bulkheads, idempotency, and graceful degradation.

I also identify critical and non-critical dependencies. Non-critical operations such as analytics or notifications can often be processed asynchronously, while payment and inventory operations must preserve business consistency.

### Q7. Why can retries make a production incident worse?

Answer:

When a dependency becomes slow, retries create additional requests. If many services retry simultaneously, traffic increases against an already unhealthy dependency.

I use bounded retries, backoff, jitter, timeouts, and circuit breakers to control retry traffic.

### Q8. How do you prevent duplicate payment processing?

Answer:

I use an idempotency key for each logical payment operation.

The payment service stores the result associated with the key. When the same request is received again, it returns the previous result rather than performing another payment operation.

I also handle concurrent requests and ensure that the idempotency record is stored durably.

### Q9. How do you prevent RDS connection exhaustion?

Answer:

I calculate the maximum potential connections across all service replicas and connection pools.

I review HikariCP settings, database connection limits, query duration, connection leaks, and traffic patterns.

I configure connection timeouts and appropriate pool sizes, and monitor connection utilization.

### Q10. How do you prevent RabbitMQ queue backlog?

Answer:

I monitor ready messages, unacknowledged messages, consumer count, publish rate, and processing rate.

I investigate slow consumers, failed dependencies, and insufficient worker capacity.

I use controlled retries and dead-letter queues for messages that repeatedly fail.

### Q11. How do you prevent Kafka consumer lag?

Answer:

I monitor consumer lag, processing rate, partition distribution, consumer health, and broker capacity.

I investigate slow processing, hot partitions, consumer failures, and rebalancing.

I scale consumers where appropriate, while respecting partition parallelism and downstream capacity.

### Q12. How do you improve CI/CD reliability?

Answer:

I standardize pipeline stages for source validation, unit testing, SonarQube analysis, dependency checks, Docker builds, Trivy scans, and artifact publication.

I use immutable image references, secure credentials, protected branches, approval gates, and deployment verification.

I also track pipeline failures by stage and improve recurring failure areas.

### Q13. Why is building an image once important?

Answer:

Building once and promoting the same image digest across environments ensures that the artifact tested in staging is the same artifact deployed in production.

It improves traceability and reduces environment-specific build differences.

### Q14. How do you prevent Terraform infrastructure mistakes?

Answer:

I use pull request reviews, formatting, validation, security checks, and saved Terraform plans.

I review destructive changes, resource replacement, IAM modifications, networking, and database changes carefully.

I also use protected remote state, locking, versioning, encryption, and controlled apply permissions.

### Q15. How do you manage GitOps drift?

Answer:

I compare the live cluster state with the desired state stored in Git.

I identify whether drift came from manual changes, controllers, or another deployment process. I review the difference, update the approved desired state, and then reconcile using Argo CD.

I avoid blindly applying changes that could delete important resources.

### Q16. How do you improve monitoring and alerting?

Answer:

I create service-level dashboards and alerts around availability, latency, error rate, saturation, and important dependencies.

I use Prometheus for metrics, Grafana for dashboards, and ELK for log analysis.

I also ensure every critical alert has an owner, severity, runbook, and tested notification route.

### Q17. How do you verify that a backup is useful?

Answer:

I verify backup completion, retention, encryption, timestamps, and recovery-point availability.

However, metadata alone does not prove recoverability. I perform restore exercises and validate the restored application and data against approved RTO and RPO requirements.

### Q18. How do you measure whether a preventive action worked?

Answer:

I define acceptance criteria before implementation.

After the change, I run the required tests and monitor relevant metrics. For example, if an action addresses recurring pod crashes, I verify that the crash condition no longer occurs under the tested workload and that the relevant alert behaves correctly.

I also check for unintended side effects.

### Q19. What is a reliability game day?

Answer:

A game day is a controlled exercise in which teams simulate selected failure conditions to test detection, response, resilience, and recovery.

Examples include worker-node loss, dependency failure, database failover, and message backlog.

The exercise should have approval, scope, safety boundaries, an abort procedure, and documented findings.

### Q20. What is the difference between monitoring and observability?

Answer:

Monitoring tells us whether predefined conditions are healthy or unhealthy. Observability helps us investigate system behavior using available evidence such as metrics and logs.

Monitoring can identify a high error rate. Observability helps engineers investigate why it happened.

### Q21. How do you reduce alert fatigue?

Answer:

I review alerts for duplicates, false positives, missing ownership, and lack of actionable information.

I align thresholds with service objectives, use suitable evaluation windows, group related alerts, and provide runbook links.

I also investigate incidents that were not detected by existing alerts.

### Q22. What is an SLO and why is it useful?

Answer:

A Service Level Objective is a defined reliability target measured using a Service Level Indicator over a specified time window.

For example, a checkout service may target 99.9% successful eligible requests over 30 days.

SLOs help teams understand reliability performance and make informed release and improvement decisions.

### Q23. How do you implement least privilege for EKS workloads?

Answer:

I assign a dedicated Kubernetes ServiceAccount to the workload and associate it with a narrowly scoped IAM role using IRSA or the selected EKS workload identity mechanism.

I restrict permissions to required actions and resources, validate the role assumed by the pod, and periodically review access.

### Q24. How do you prevent secrets from leaking through CI/CD?

Answer:

I store credentials in an approved secret-management system or secured pipeline credentials store.

I avoid hardcoding secrets, printing them in logs, or committing them to Git. I restrict access, rotate credentials, and review pipeline permissions.

### Q25. What is your approach to platform improvement?

Answer:

I follow a continuous improvement cycle:

1. Identify incidents and operational risks.
2. Collect evidence.
3. Prioritize by impact and likelihood.
4. Assign ownership.
5. Design the improvement.
6. Implement through controlled changes.
7. Validate acceptance criteria.
8. Monitor effectiveness.
9. Document residual risk.
10. Review results regularly.

## 52. Common platform improvement mistakes

| Mistake                                       | Why it is harmful                            | Better approach                        |
| --------------------------------------------- | -------------------------------------------- | -------------------------------------- |
| Restarting pods without investigation         | Root cause remains                           | Investigate events, logs, and metrics  |
| Increasing replicas without dependency review | Can overload RDS or messaging                | Review downstream capacity             |
| Unlimited retries                             | Creates retry storms                         | Use bounded retries and jitter         |
| Incorrect liveness probes                     | Can cause unnecessary restarts               | Separate readiness and liveness        |
| No resource requests                          | Scheduling and capacity become unpredictable | Define and tune requests               |
| HPA without node capacity                     | Pods remain Pending                          | Validate node autoscaling              |
| No PDB                                        | Maintenance may disrupt too many replicas    | Define disruption budgets              |
| One shared IAM role for all services          | Excessive access and large blast radius      | Dedicated least-privilege roles        |
| Hardcoded secrets                             | Credential exposure                          | Use managed secret storage             |
| Mutable production image tags                 | Weak release traceability                    | Deploy immutable digests               |
| Unreviewed Terraform apply                    | Infrastructure destruction risk              | Review saved plans                     |
| Blind Argo CD self-healing                    | Can overwrite emergency changes              | Review drift and update Git            |
| Backup metadata treated as proof              | Restore may still fail                       | Perform restore tests                  |
| Alerting without ownership                    | Alerts are ignored                           | Define owner and runbook               |
| Too many dashboards                           | Important signals become difficult to find   | Design around operational questions    |
| Logging sensitive information                 | Privacy and security exposure                | Apply structured logging and redaction |
| Closing actions without evidence              | Risks may remain unresolved                  | Verify acceptance criteria             |
| No follow-up on game days                     | Findings are lost                            | Create owned improvement actions       |

## 53. Production readiness checklist

Use this checklist before approving a critical service or platform improvement for production.

### 53.1 Application readiness

- Application owner identified.
- Service criticality defined.
- Dependencies documented.
- Timeouts configured.
- Retry policy bounded.
- Idempotency considered for important operations.
- Graceful shutdown implemented.
- Graceful degradation defined.
- Health endpoints validated.
- Error handling reviewed.

### 53.2 Kubernetes readiness

- Deployment replicas reviewed.
- Readiness probe configured.
- Liveness probe configured.
- Startup probe configured where necessary.
- CPU requests defined.
- Memory requests defined.
- Limits reviewed.
- PDB configured where appropriate.
- Topology spread reviewed.
- HPA validated.
- Node capacity checked.
- Namespace quotas reviewed.
- ServiceAccount permissions reviewed.
- Network policies validated.

### 53.3 Database readiness

- Connection pool configured.
- Connection budget calculated.
- Slow queries reviewed.
- Storage capacity monitored.
- Backup retention configured.
- Restore test completed.
- Database permissions reviewed.
- Schema compatibility checked.
- Failover behavior documented.

### 53.4 Messaging readiness

- RabbitMQ queue depth monitored.
- Consumer count monitored.
- Retry policy bounded.
- Dead-letter handling configured.
- Kafka consumer lag monitored.
- Kafka partition distribution reviewed.
- Duplicate message processing considered.
- Message retention reviewed.
- Replay procedure documented.

### 53.5 CI/CD and GitOps readiness

- Branch protection configured.
- Pull request review required.
- Unit tests enabled.
- SonarQube quality gate configured.
- Trivy scan configured.
- Dependency checks reviewed.
- Immutable artifact reference recorded.
- Pipeline credentials protected.
- GitOps repository reviewed.
- Argo CD permissions restricted.
- Deployment validation completed.
- Rollback or forward-recovery plan documented.

### 53.6 Security readiness

- IAM least privilege reviewed.
- IRSA or workload identity validated.
- Secrets stored securely.
- Secret rotation procedure tested.
- Container security context reviewed.
- Image vulnerability policy applied.
- Network access restricted.
- Production access reviewed.
- Audit evidence retained.

### 53.7 Observability readiness

- Prometheus metrics available.
- Grafana dashboards available.
- ELK logs searchable.
- Alert thresholds reviewed.
- Alert owner defined.
- Runbook linked.
- Notification routing tested.
- Service-level objectives defined.
- Missing telemetry identified.
- Observability pipeline health monitored.

### 53.8 Operational readiness

- Incident runbook available.
- Rollback procedure available.
- Recovery procedure available.
- On-call ownership defined.
- Escalation contacts documented.
- Game day performed where appropriate.
- Improvement actions assigned.
- Residual risk documented.
- Acceptance criteria satisfied.

## 54. Final outcome

After completing this chapter, the engineer should be able to operate a continuous platform improvement process rather than relying only on reactive incident response.

The expected outcomes are:

1. Incidents generate measurable engineering improvements.
2. Improvement work is prioritized by customer and platform risk.
3. Kubernetes workloads have consistent reliability baselines.
4. EKS capacity and networking risks are reviewed.
5. Microservice dependencies are documented and tested.
6. Retries, timeouts, circuit breakers, and idempotency reduce failure amplification.
7. RDS, Redis, RabbitMQ, and Kafka have defined reliability controls.
8. CI/CD and GitOps delivery become more predictable.
9. Terraform changes are reviewed and controlled.
10. SonarQube, Trivy, and other approved security controls are integrated into delivery.
11. Prometheus, Grafana, and ELK provide useful operational evidence.
12. IAM, workload identity, and secret management follow least-privilege principles.
13. Backups are verified through actual restore exercises.
14. Weekly, monthly, and quarterly reviews keep improvement work moving.
15. Every completed action has measurable acceptance evidence.

Final principle:

> A mature DevOps platform does not simply recover from failures. It learns from failures, reduces recurring risks, validates improvements, and continuously becomes safer to operate.