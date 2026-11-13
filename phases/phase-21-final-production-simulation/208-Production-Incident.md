# CHAPTER 208 — PRODUCTION INCIDENT

**Phase 21 — Final Simulation**

**File:** `208-Production-Incident.md`

---

# 1. Objective

This chapter simulates a real production incident in a 20+ microservices application running on AWS EKS.

The objective is to understand how a DevOps / Platform Engineer responds when production users experience application failures.

We will cover:

- Incident detection
- Alert investigation
- Incident severity classification
- Initial impact assessment
- Kubernetes troubleshooting
- Application troubleshooting
- Database connectivity failures
- Redis failures
- RabbitMQ / Kafka messaging failures
- Ingress and ALB troubleshooting
- Resource exhaustion
- Service recovery
- Rollback procedures
- Communication and escalation
- Root Cause Analysis (RCA)
- Corrective and preventive actions
- Incident closure

**Important:** This is a controlled simulation. Do not execute failure-injection commands against a real production cluster without authorization, an approved change window, and a tested recovery plan.

# 2. Production Environment

## 2.1 Architecture

The application contains multiple microservices deployed on AWS EKS.

```text
                         USERS
                           |
                           v
                    Route 53 DNS
                           |
                           v
                    AWS ALB / Ingress
                           |
                           v
                    Kubernetes Services
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
   Frontend             Catalogue            User
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                    Cart / Checkout
                           |
                           v
                     Order Service
                           |
              +------------+------------+
              |            |            |
              v            v            v
           Payment     Inventory    Notification
              |            |            |
              v            v            v
             RDS          RDS       RabbitMQ/Kafka

       Supporting infrastructure:
       - AWS EKS
       - Amazon ECR
       - AWS RDS
       - Redis
       - RabbitMQ / Kafka
       - Argo CD
       - Helm
       - Prometheus
       - Grafana
       - Elasticsearch
       - Logstash
       - Kibana
       - Alertmanager
```

## 2.2 Infrastructure Components

| Component | Responsibility |
|---|---|
| AWS EKS | Kubernetes platform |
| AWS ALB | External traffic entry |
| Route 53 | DNS resolution |
| ECR | Container image registry |
| RDS | Relational database |
| Redis | Cache |
| RabbitMQ | Message broker |
| Kafka | Event streaming |
| Argo CD | GitOps deployment |
| Helm | Kubernetes packaging |
| Prometheus | Metrics collection |
| Grafana | Dashboards |
| Alertmanager | Alert routing |
| Elasticsearch | Log storage and search |
| Logstash | Log processing |
| Kibana | Log visualization |

# 3. Incident Scenario

## 3.1 Incident Description

At 10:15 AM, users report that they cannot complete orders.

The frontend is accessible, but checkout requests are failing.

Some users receive:

```text
HTTP 500 Internal Server Error
HTTP 502 Bad Gateway
HTTP 503 Service Unavailable
```

The operations team receives alerts from Prometheus and Alertmanager.

## 3.2 Incident Details

| Attribute | Value |
|---|---|
| Incident ID | INC-2026-001 |
| Environment | Production |
| Platform | AWS EKS |
| Application | RoboShop |
| Affected functionality | Checkout and order processing |
| Detection | Prometheus / Grafana alerts |
| Initial severity | SEV-2, subject to impact assessment |
| Incident start | 10:15 AM |
| Incident commander | Assigned engineer |
| Primary responders | DevOps, Application, Database teams |

The severity is provisional. The incident commander should classify it using actual customer impact and the organization's incident policy.

## 3.3 Expected Business Impact

- Users cannot complete checkout.
- Order creation requests fail.
- Payment processing may be interrupted.
- Inventory updates may be delayed.
- Notification events may remain unprocessed.
- Revenue-impacting transactions may be affected.

Do not assume that every downstream service is failing. Confirm the actual impact using metrics, logs, traces if available from existing application instrumentation, and service-level checks.

# 4. Incident Response Roles

A production incident needs clear ownership.

| Role | Responsibility |
|---|---|
| Incident Commander | Coordinates the incident |
| Technical Lead | Directs technical investigation |
| DevOps Engineer | Investigates EKS and infrastructure |
| Application Engineer | Investigates application behavior |
| Database Engineer | Investigates database health |
| Communications Lead | Sends stakeholder updates |
| Scribe | Maintains incident timeline |

One person may perform multiple roles in a small team, but incident coordination and technical execution should remain clearly identified.

# 5. Incident Detection

## 5.1 Prometheus Alerts

Example alert rules:

```yaml
groups:
  - name: production-application-alerts
    rules:

      - alert: HighHTTP5xxRate
        expr: |
          (
            sum(rate(http_requests_total{
              environment="production",
              status=~"5.."
            }[5m]))
            /
            clamp_min(
              sum(rate(http_requests_total{
                environment="production"
              }[5m])),
              1
            )
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP 5xx error rate"
          description: "Production HTTP 5xx rate exceeded 5%."

      - alert: HighPodRestartRate
        expr: |
          increase(kube_pod_container_status_restarts_total{
            namespace="production"
          }[10m]) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Production container restarts detected"

      - alert: HighPodCPUUsage
        expr: |
          (
            sum by (namespace, pod) (
              rate(container_cpu_usage_seconds_total{
                namespace="production",
                container!="",
                container!="POD"
              }[5m])
            )
            /
            sum by (namespace, pod) (
              kube_pod_container_resource_limits{
                namespace="production",
                resource="cpu",
                unit="core"
              }
            )
          ) > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Production pod CPU usage is high"
```

**Implementation notes:**

- `http_requests_total` must be exposed by your application instrumentation.
- The HTTP status label may be named differently in your application.
- Verify that kube-state-metrics exposes the resource metrics used by the rules.
- Exclude infrastructure and pause containers from application-level calculations.
- Handle missing or zero resource limits appropriately.

## 5.2 Alertmanager Notification

Example configuration:

```yaml
route:
  receiver: default-team

  routes:
    - matchers:
        - severity="critical"
      receiver: production-oncall
      continue: false

    - matchers:
        - severity="warning"
      receiver: devops-team

receivers:
  - name: default-team
    # Configure an approved notification integration.

  - name: production-oncall
    # Configure an approved on-call notification integration.

  - name: devops-team
    # Configure an approved team notification integration.
```

The receiver integrations are intentionally omitted. Configure them with your organization's approved notification service and credentials.

# 6. Initial Incident Response

## 6.1 Acknowledge the Alert

The on-call engineer should:

1. Acknowledge the alert.
2. Record the incident start time.
3. Create or update the incident ticket.
4. Identify affected services.
5. Notify the incident commander.
6. Start the incident timeline.

Example:

```text
INCIDENT: INC-2026-001

Time: 10:15 AM

Issue:
Production checkout requests are failing.

Initial symptoms:
- HTTP 5xx errors
- Checkout failures
- Order processing delays

Environment:
AWS EKS - Production

Status:
Investigating
```

## 6.2 Confirm the Impact

Check:

- Is the frontend accessible?
- Are product catalogue requests working?
- Are login requests working?
- Are checkout requests failing?
- Are order requests failing?
- Are payment requests failing?
- Are failures affecting all users or a subset?
- Did the issue start after a deployment?

Avoid restarting services immediately. First collect enough evidence to identify the failure domain.

# 7. Kubernetes Investigation

## 7.1 Connect to the Production Cluster

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name roboshop-prod
```

Verify:

```bash
kubectl config current-context

kubectl get nodes

kubectl get namespaces
```

Check production workloads:

```bash
kubectl get pods -n production -o wide

kubectl get deployments -n production

kubectl get services -n production

kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp
```

## 7.2 Identify Unhealthy Pods

```bash
kubectl get pods -n production
```

Example:

```text
NAME                              READY   STATUS             RESTARTS
frontend-7c9d8b6f5d-abc12          1/1     Running            0
catalogue-6d8c7f9c5d-def34        1/1     Running            0
cart-5d7c8f9d6b-ghi56              1/1     Running            0
order-7f9d8c6b5d-jkl78             0/1     CrashLoopBackOff   8
payment-6c8d7f9c5d-mno90           1/1     Running            0
```

The order service is unhealthy.

Investigate the pod:

```bash
kubectl describe pod \
  order-7f9d8c6b5d-jkl78 \
  -n production
```

Check logs:

```bash
kubectl logs \
  order-7f9d8c6b5d-jkl78 \
  -n production
```

Check previous container logs:

```bash
kubectl logs \
  order-7f9d8c6b5d-jkl78 \
  -n production \
  --previous
```

Check the deployment:

```bash
kubectl describe deployment order -n production
```

## 7.3 Investigate CrashLoopBackOff

Common causes:

- Application startup exception
- Invalid environment variables
- Database connection failure
- Redis connection failure
- Missing Kubernetes Secret
- Incorrect configuration
- Failed startup probe
- Out-of-memory termination
- Incompatible application image

Do not assume CrashLoopBackOff is caused by insufficient memory. Read the container's previous logs and pod events.

## 7.4 Check OOMKilled

```bash
kubectl get pods -n production -o json
```

For a particular pod:

```bash
kubectl describe pod <pod-name> -n production
```

Look for:

```text
Last State:     Terminated
Reason:         OOMKilled
Exit Code:      137
```

If confirmed, inspect:

- Container memory usage
- Memory requests and limits
- Application memory behavior
- Recent traffic changes
- JVM heap settings, if applicable
- Memory leaks
- Node memory pressure

Do not simply increase limits without understanding node capacity and the application memory profile.

# 8. Application Log Investigation

## 8.1 Search Logs in Kibana

Select:

- Data view: production application logs
- Environment: production
- Time range: incident start time to current time

Example query:

```text
environment: "production"
AND service.name: "order"
AND log.level: "ERROR"
```

Search for database failures:

```text
message: "connection refused"
OR message: "connection timeout"
OR message: "too many connections"
```

Search for Redis failures:

```text
message: "Redis connection"
OR message: "Redis timeout"
```

Search for message broker failures:

```text
message: "RabbitMQ"
OR message: "Kafka"
OR message: "broker unavailable"
```

The exact fields depend on your Logstash pipeline and application log format.

## 8.2 Check Logs Directly

```bash
kubectl logs \
  deployment/order \
  -n production \
  --since=30m
```

Follow logs:

```bash
kubectl logs \
  deployment/order \
  -n production \
  -f
```

For multiple replicas, inspect individual pods or use your centralized logging system. A deployment-level log command may not show every replica's logs.

# 9. Database Investigation

## 9.1 Identify Database Errors

Example application log:

```text
ERROR: Unable to acquire database connection

Connection timed out

SQLState: 08001
```

Possible causes:

- RDS unavailable
- Network connectivity problem
- Security group restriction
- Incorrect database endpoint
- DNS resolution problem
- Database connection limit reached
- Connection pool exhaustion
- Incorrect credentials
- Database overload

## 9.2 Check Database Connectivity

From an approved diagnostic pod with the necessary client installed:

```bash
kubectl exec -it <diagnostic-pod> \
  -n production -- sh
```

Resolve the database hostname:

```bash
nslookup <rds-endpoint>
```

Check the port:

```bash
nc -vz <rds-endpoint> 5432
```

For MySQL, the typical port is 3306.

For PostgreSQL, the typical port is 5432.

A successful TCP connection does not prove that authentication or SQL queries will succeed.

## 9.3 Check RDS Health

Inspect:

- RDS availability
- CPU utilization
- Freeable memory
- Free storage
- Database connections
- Read/write latency
- Database logs
- Recent parameter changes
- Failover events

Use the approved AWS console or CLI access for your environment.

## 9.4 Check Connection Pool Exhaustion

Example Spring Boot configuration:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

Investigate:

- Active connections
- Idle connections
- Pending connection requests
- Connection acquisition time
- Query execution time
- Database maximum connections
- Total possible connections across all replicas

For example, if 10 replicas each allow 20 connections, the application could attempt to open 200 connections. Account for every service sharing the database before changing pool sizes.

# 10. Redis Investigation

Check Redis service:

```bash
kubectl get svc -n production | grep redis
```

Check Redis pods:

```bash
kubectl get pods -n production | grep redis
```

Inspect logs:

```bash
kubectl logs deployment/redis -n production
```

Check connectivity from an approved diagnostic pod:

```bash
nc -vz redis 6379
```

If `redis-cli` is available and approved:

```bash
redis-cli -h redis -p 6379 PING
```

Expected response:

```text
PONG
```

Investigate:

- Redis availability
- Memory usage
- Evictions
- Connection count
- Network connectivity
- Authentication
- Latency
- Cache hit ratio

Do not flush Redis as an incident response shortcut. That can cause a cache stampede and increase database load.

# 11. RabbitMQ and Kafka Investigation

## 11.1 RabbitMQ

Check pods:

```bash
kubectl get pods -n production | grep rabbitmq
```

Check service:

```bash
kubectl get svc -n production | grep rabbitmq
```

Inspect logs:

```bash
kubectl logs deployment/rabbitmq -n production
```

Investigate:

- Queue depth
- Unacknowledged messages
- Consumer count
- Connection count
- Channel count
- Memory alarms
- Disk alarms
- Consumer failures
- Message publishing failures

A growing queue may indicate that consumers are slower than producers or that consumers are unavailable.

## 11.2 Kafka

Check Kafka workloads:

```bash
kubectl get pods -n production | grep kafka
```

Investigate:

- Broker availability
- Under-replicated partitions
- Offline partitions
- Consumer lag
- Producer errors
- Network connectivity
- Disk capacity
- Broker logs

If Kafka consumer lag increases, determine whether the cause is:

- Consumer downtime
- Slow message processing
- Insufficient consumer capacity
- Partition imbalance
- Broker performance
- Downstream database latency

Do not reset consumer offsets without understanding the impact on message processing and data consistency.

# 12. Ingress and ALB Investigation

If users receive 502 or 503 responses, investigate the traffic path.

```text
User
 |
 v
Route 53
 |
 v
AWS ALB
 |
 v
Ingress Controller / Target Group
 |
 v
Kubernetes Service
 |
 v
Application Pod
```

Check ingress:

```bash
kubectl get ingress -n production

kubectl describe ingress <ingress-name> \
  -n production
```

Check services:

```bash
kubectl get svc -n production
```

Check endpoints:

```bash
kubectl get endpoints -n production
```

Where EndpointSlices are used:

```bash
kubectl get endpointslices -n production
```

Check ALB target health using the AWS console or approved AWS CLI access.

Investigate:

- Unhealthy targets
- Target group health checks
- Incorrect service port
- Incorrect target port
- Application readiness
- Security group rules
- Network ACLs
- Ingress routing
- TLS configuration
- ALB listener rules

A 503 response can indicate no healthy targets or an application-level failure. Confirm the source of the response before making changes.

# 13. Root Cause Identification

After investigating the application logs, Kubernetes events, and database connectivity, the team identifies this simulated failure:

**The order service cannot acquire database connections because the database connection pool is exhausted.**

Supporting evidence:

1. Order service pods are restarting or returning errors.
2. Application logs show connection acquisition timeouts.
3. Database connectivity is available.
4. RDS is reachable.
5. Database connection counts are elevated.
6. Connection pool metrics show pending requests.
7. Other services sharing the database are contributing to connection pressure.

The initial incident hypothesis is now supported by evidence.

The team should still verify whether connection leaks, slow queries, excessive replica counts, or an inappropriate pool configuration caused the exhaustion.

# 14. Immediate Mitigation

The incident commander approves a mitigation based on the evidence.

Possible actions:

- Reduce unnecessary database connection pressure.
- Stop or reduce noncritical background workloads if approved.
- Scale application replicas only after checking total connection capacity.
- Roll back a recent application release if it introduced the issue.
- Correct an approved connection pool configuration.
- Temporarily limit nonessential traffic if business and application teams approve.

Example: inspect current deployment configuration.

```bash
kubectl get deployment order \
  -n production -o yaml
```

Check current replicas:

```bash
kubectl get deployment order -n production
```

If the incident is linked to a recent release, review rollout history:

```bash
kubectl rollout history deployment/order \
  -n production
```

Do not blindly scale the order service. More replicas can increase the total number of database connections and make the incident worse.

# 15. Rollback Procedure

If the team confirms that the latest deployment introduced the failure, use the approved rollback procedure.

Check rollout history:

```bash
kubectl rollout history deployment/order \
  -n production
```

Review the relevant revision:

```bash
kubectl rollout history deployment/order \
  -n production --revision=2
```

For a deployment managed directly through Kubernetes:

```bash
kubectl rollout undo deployment/order \
  -n production
```

Wait for rollout completion:

```bash
kubectl rollout status deployment/order \
  -n production
```

**GitOps consideration:**

If Argo CD manages this deployment, a direct `kubectl rollout undo` may be reverted by Argo CD reconciliation.

For a GitOps-managed workload:

1. Identify the last known-good Git commit.
2. Revert the faulty application or Helm configuration change.
3. Push the approved change.
4. Review the resulting GitOps sync.
5. Confirm the intended image digest and configuration.
6. Verify the live workload.
7. Confirm that the incident symptoms have stopped.

Do not disable Argo CD reconciliation as a routine rollback method.

# 16. Recovery Validation

After mitigation, verify the complete customer request path.

## 16.1 Kubernetes Health

```bash
kubectl get pods -n production

kubectl get deployments -n production

kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp
```

Expected:

- Pods are Ready.
- Deployments have the desired available replicas.
- No unexpected restarts.
- No new critical warning events.

## 16.2 Application Health

```bash
curl -i https://<production-domain>/health
```

Use the actual health endpoint exposed by the application.

Check:

- Login
- Catalogue
- Cart
- Checkout
- Order creation
- Payment integration
- Inventory updates
- Notification processing

Use approved test accounts and test transactions. Do not create real customer payments during an incident test.

## 16.3 Database Health

Verify:

- Connection count has returned to a safe level.
- Connection pool pending requests have cleared.
- Query latency has recovered.
- Database errors have stopped.
- No unexpected transaction backlog remains.

## 16.4 Messaging Health

Verify:

- RabbitMQ queue depth is recovering.
- Unacknowledged messages are decreasing.
- Kafka consumer lag is recovering.
- No unexpected duplicate processing is occurring.
- Dead-letter queues are reviewed.

## 16.5 Monitoring Health

Check Grafana:

- HTTP error rate
- Request latency
- Request throughput
- Pod CPU and memory
- Pod restart count
- Database connections
- Database latency
- Redis health
- Message queue depth
- Kafka consumer lag

The incident is not resolved simply because pods are Running. Confirm that users can successfully complete the affected business workflow.

# 17. Incident Communication

## 17.1 Initial Update

```text
INCIDENT UPDATE — INC-2026-001

Time: 10:30 AM

Issue:
Production checkout requests are failing.

Impact:
Order processing is affected.

Current investigation:
The team is investigating order service database
connection failures.

Status:
Investigation in progress.

Next update:
As per the incident communication schedule.
```

## 17.2 Recovery Update

```text
INCIDENT UPDATE — INC-2026-001

Time: 11:05 AM

Issue:
Production checkout failures.

Action taken:
The team applied the approved mitigation and
restored database connection availability.

Current status:
Checkout requests are succeeding in validation.

Monitoring:
The team is monitoring application errors,
database connections, and order processing.
```

## 17.3 Resolution Update

```text
INCIDENT RESOLVED — INC-2026-001

Time: 11:30 AM

Impact:
Production checkout and order processing
were affected.

Resolution:
The database connection exhaustion issue
was mitigated.

Validation:
- Order service is healthy.
- Checkout requests are succeeding.
- Database connection pressure has reduced.
- Application error rate has recovered.

Follow-up:
A formal RCA and preventive action plan
will be completed.
```

# 18. Incident Timeline

Maintain an accurate timeline using actual timestamps.

| Time | Activity |
|---|---|
| 10:15 AM | First alert received |
| 10:18 AM | Incident acknowledged |
| 10:22 AM | Production impact confirmed |
| 10:25 AM | Technical investigation started |
| 10:35 AM | Order service errors identified |
| 10:45 AM | Database connection exhaustion identified |
| 10:55 AM | Mitigation approved |
| 11:05 AM | Mitigation applied |
| 11:15 AM | Recovery validation started |
| 11:30 AM | Incident resolved |

These timestamps are illustrative. Record actual timestamps during a real incident.

# 19. Root Cause Analysis (RCA)

## 19.1 RCA Summary

**Incident:** Production checkout failure

**Root cause:** Database connection pool exhaustion in the order processing path.

**Customer impact:** Checkout and order processing failures.

**Detection:** Prometheus alerting and customer reports.

**Resolution:** Approved mitigation restored database connection availability.

## 19.2 Five Whys Analysis

**Why 1: Why did checkout fail?**

Because the order service could not complete database operations.

**Why 2: Why could the order service not complete database operations?**

Because database connection acquisition requests were timing out.

**Why 3: Why were connection requests timing out?**

Because available database connections were exhausted.

**Why 4: Why were database connections exhausted?**

Because the combined application connection demand exceeded the available database connection capacity.

**Why 5: Why was the connection demand not controlled?**

Because the total connection budget across replicas and services was not adequately validated against the database capacity, and connection pressure alerts were insufficient.

The final RCA must be supported by application metrics, database metrics, configuration history, and logs. The five-whys example is a simulated hypothesis, not proof of a real root cause.

## 19.3 Corrective Actions

| Action | Priority | Owner |
|---|---|---|
| Validate connection pool configuration | High | Application team |
| Review database connection capacity | High | Database team |
| Add connection pool saturation alerts | High | DevOps team |
| Review slow SQL queries | High | Database team |
| Review application connection lifecycle | High | Application team |
| Load test checkout workflow | Medium | QA team |
| Document database connection budget | Medium | Platform team |

# 20. Preventive Monitoring

Add monitoring for:

- Database active connections
- Database connection pool usage
- Pending connection acquisition
- Connection acquisition latency
- Query latency
- Database CPU and memory
- HTTP 5xx rate
- Checkout success rate
- Order processing latency

Example PromQL for application-exposed HikariCP metrics:

```promql
hikaricp_connections_active
```

```promql
hikaricp_connections_pending
```

```promql
hikaricp_connections_timeout_total
```

Metric names and labels depend on the Spring Boot and Micrometer instrumentation version. Confirm the actual `/actuator/prometheus` output before creating production alerts.

# 21. Incident Simulation Exercise

Perform the following exercise in a nonproduction environment.

## Exercise A — Pod Failure

Objective: Learn how to identify an unhealthy pod.

```bash
kubectl get pods -n qa
```

Inspect a selected test pod:

```bash
kubectl describe pod <pod-name> -n qa
```

Review events and logs.

Recovery: Use the approved deployment or GitOps process.

## Exercise B — Application Failure

Objective: Learn to investigate an application returning HTTP 500.

Check:

```bash
kubectl logs deployment/<deployment-name> \
  -n qa --since=15m
```

Check service health:

```bash
kubectl get svc -n qa

kubectl get endpoints -n qa
```

Recovery: Identify and correct the application or configuration problem.

## Exercise C — Database Connectivity Failure

Objective: Understand how to isolate database connectivity issues.

From an approved diagnostic pod:

```bash
nslookup <database-endpoint>

nc -vz <database-endpoint> 5432
```

Recovery: Validate DNS, routing, security groups, database availability, and credentials.

## Exercise D — High CPU

Objective: Identify resource pressure.

```bash
kubectl top nodes

kubectl top pods -n qa
```

Inspect:

```bash
kubectl describe pod <pod-name> -n qa
```

Recovery: Identify the workload causing pressure and apply an approved capacity or application fix.

## Exercise E — Message Queue Backlog

Objective: Investigate asynchronous processing delays.

Check:

- Queue depth
- Consumer availability
- Unacknowledged messages
- Consumer processing rate
- Downstream service health

Recovery: Restore consumers or correct the downstream dependency without discarding messages.

# 22. Production Incident Troubleshooting Reference

| Symptom | First investigation | Possible cause |
|---|---|---|
| CrashLoopBackOff | Previous container logs | Startup failure |
| OOMKilled | Pod description and memory metrics | Memory exhaustion |
| HTTP 502 | ALB target health and application logs | Upstream failure |
| HTTP 503 | Service endpoints and target health | No healthy targets |
| HTTP 500 | Application logs | Application exception |
| Database timeout | Pool and RDS metrics | Connection exhaustion |
| Redis timeout | Redis health and connectivity | Cache failure |
| Queue backlog | Consumer and queue metrics | Consumer bottleneck |
| Kafka lag | Consumer group metrics | Processing delay |
| DNS failure | DNS lookup and CoreDNS health | Name resolution problem |
| High latency | Application and dependency metrics | Slow downstream dependency |

# 23. Production Incident Closure Checklist

## Detection and coordination

- [ ] Alert acknowledged.
- [ ] Incident ID created.
- [ ] Severity assessed.
- [ ] Incident commander assigned.
- [ ] Impact confirmed.
- [ ] Stakeholders informed.

## Investigation

- [ ] Kubernetes health checked.
- [ ] Application logs reviewed.
- [ ] Database health checked.
- [ ] Redis health checked.
- [ ] Messaging infrastructure checked.
- [ ] Ingress and ALB checked.
- [ ] Recent deployments reviewed.
- [ ] Root cause supported by evidence.

## Recovery

- [ ] Mitigation approved.
- [ ] Recovery action completed.
- [ ] Application health verified.
- [ ] Customer workflow tested.
- [ ] Database health verified.
- [ ] Messaging backlog reviewed.
- [ ] Monitoring confirms recovery.

## Closure

- [ ] Incident timeline completed.
- [ ] RCA documented.
- [ ] Corrective actions assigned.
- [ ] Preventive actions assigned.
- [ ] Runbooks updated.
- [ ] Stakeholders informed.
- [ ] Incident formally closed.

# 24. Expected Engineer Knowledge

After completing this chapter, you should be able to explain:

1. How do you respond when production checkout stops working?
2. How do you identify the affected microservice?
3. How do you investigate CrashLoopBackOff?
4. How do you investigate OOMKilled?
5. How do you identify database connection exhaustion?
6. How do you investigate Redis connectivity?
7. How do you troubleshoot RabbitMQ queue backlogs?
8. How do you troubleshoot Kafka consumer lag?
9. How do you investigate ALB 502 and 503 errors?
10. When should you roll back a production deployment?
11. How do you handle a GitOps-managed rollback?
12. How do you validate application recovery?
13. How do you communicate with stakeholders?
14. How do you prepare an RCA?
15. How do you prevent the same incident from happening again?

# 25. Final Outcome

By completing this chapter, you will have practiced the production incident lifecycle:

```text
ALERT
  |
  v
ACKNOWLEDGE
  |
  v
ASSESS IMPACT
  |
  v
INVESTIGATE
  |
  v
IDENTIFY ROOT CAUSE
  |
  v
APPROVE MITIGATION
  |
  v
RECOVER SERVICE
  |
  v
VALIDATE
  |
  v
COMMUNICATE
  |
  v
RCA
  |
  v
PREVENT RECURRENCE
  |
  v
CLOSE INCIDENT
```

**Key production principle:**

Do not restart, scale, delete, or roll back resources blindly during a production incident. First establish the impact, collect evidence, identify the failure domain, obtain the required approval, and validate the recovery.

**End of Chapter 208 — Production Incident.**

# 30. Decision Trees for Common Symptoms

## 30.1 HTTP 5xx

```text
HTTP 5xx observed
        |
        v
Generated by ALB or application?
        |
   +----+----+
   |         |
  ALB     Application
   |         |
Target      Application logs
health      and request IDs
   |         |
Healthy     Dependency errors?
targets?        |
   |       +---+---+
  No      Yes      No
   |       |       |
Ingress/  Isolate  Inspect code,
service   failing  config and
endpoints dependency resource pressure
```

## 30.2 Pod Not Ready

```text
Pod not Ready
     |
Container running?
   |       |
  No      Yes
   |       |
Inspect   Inspect readiness
events    probe result
and logs      |
          +---+---+
          |       |
        Fails   Passes
          |       |
Check path,   Check endpoint
port, app     registration,
startup       selector and target port
```

## 30.3 Database Failure

```text
Database request failing
          |
DNS resolution succeeds?
      |          |
     No         Yes
      |          |
DNS path     TCP connection works?
                 |
             +---+---+
             |       |
            No      Yes
             |       |
Network/SG/route  Authentication works?
                       |
                  +----+----+
                  |         |
                 No        Yes
                  |         |
             Credentials   Pool, query,
             / policy      lock, capacity
```

## 30.4 Queue Backlog

```text
Queue depth increasing
          |
Consumers connected?
       |        |
      No       Yes
       |        |
Restore       Processing rate
consumers     lower than publish rate?
                  |
              +---+---+
              |       |
             Yes      No
              |       |
       Inspect slow   Check metrics,
       processing     routing and age
```

# 31. Recovery and Rollback Decision Framework

## 31.1 When to Roll Back

Rollback is appropriate when a recent release is strongly correlated with the incident, the previous version is known healthy, schema and message compatibility are confirmed, and the rollback is tested and approved.

Rollback may be unsafe after destructive schema changes, when the previous version cannot read current data, when the fault is infrastructure/dependency-related, when rollback reintroduces a security vulnerability, or when irreversible side effects have occurred.

## 31.2 Roll Forward

A forward fix may be safer after a non-backward-compatible migration or when the old version cannot safely process current data. Keep the change small, reviewed, tested, and approved.

## 31.3 Mitigation Guardrails

Before changing production, record expected benefit, side effects, data-loss risk, security impact, reversibility, approver, validation method, and stop condition. Afterward compare before/after metrics and verify no new failure was introduced.

# 32. Customer Impact and Service-Level Indicators

## 32.1 Define the Customer Journey

A checkout journey may include authentication, catalogue retrieval, cart update, checkout validation, order creation, payment authorization, inventory reservation, order confirmation, and notification delivery. A pod can be healthy while the complete journey is failing.

## 32.2 Example SLIs

| SLI | Measurement |
|---|---|
| Availability | Successful eligible requests / total eligible requests |
| Latency | Requests below agreed latency threshold |
| Checkout success | Successful valid checkouts / valid checkout attempts |
| Order completion | Orders reaching agreed completed state |
| Queue freshness | Age of oldest eligible message |
| Data correctness | Reconciled records / records checked |

Define exclusions for health checks, retries, synthetic traffic, and invalid user requests. Do not invent an SLO during an incident; use the approved objective and measurement window.

# 33. Load and Capacity Evidence

Compare request rate, error rate, p50/p95/p99 latency, pod and node CPU/memory, HPA desired/current replicas, pending pods, database connections, database CPU/storage, Redis hit ratio, queue depth, and consumer lag. Use matching time windows and timezone.

## 33.1 Controlled k6 Validation

Use QA or an explicitly approved production profile.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 2,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<1000'],
  },
};

export default function () {
  const response = http.get('https://qa.example.invalid/health');
  check(response, {
    'health endpoint returns 200': (r) => r.status === 200,
  });
  sleep(1);
}
```

Replace the example host with the approved QA endpoint. Do not send synthetic payment traffic to production without explicit approval and isolation.

## 33.2 Load Test Guardrails

Agree maximum request rate, test accounts/data, downstream quotas, stop conditions, monitoring, on-call notification, and immediate stop mechanism before starting. Watch database and broker capacity.

# 34. Incident Automation and Operational Scripts

## 34.1 Read-Only Cluster Snapshot

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

NAMESPACE="${1:-production}"
OUT_DIR="${2:-incident-snapshot-$(date -u +%Y%m%dT%H%M%SZ)}"
mkdir -p "$OUT_DIR"

capture() {
  local name="$1"
  shift
  echo "Collecting: $name"
  "$@" > "$OUT_DIR/$name" 2>&1 || {
    echo "Collection failed: $name" >> "$OUT_DIR/collection-errors.txt"
  }
}

capture context.txt kubectl config current-context
capture nodes.txt kubectl get nodes -o wide
capture pods.txt kubectl get pods -n "$NAMESPACE" -o wide
capture deployments.txt kubectl get deployments -n "$NAMESPACE"
capture services.txt kubectl get services -n "$NAMESPACE"
capture endpointslices.txt kubectl get endpointslices -n "$NAMESPACE"
capture events.txt kubectl get events -n "$NAMESPACE" \
  --sort-by=.metadata.creationTimestamp

echo "Snapshot saved in: $OUT_DIR"
```

This script is read-only. Use an identity with only required read permissions and review evidence before attaching it to a ticket.

## 34.2 Deployment Health Check

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

NAMESPACE="${1:-production}"
DEPLOYMENT="${2:?Usage: $0 <namespace> <deployment>}"

kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE"
kubectl rollout status "deployment/$DEPLOYMENT" \
  -n "$NAMESPACE" --timeout=120s
kubectl get pods -n "$NAMESPACE" -l "app=$DEPLOYMENT" -o wide
```

Confirm the application's actual pod label before relying on the selector.

## 34.3 Endpoint Availability Check

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SERVICE="${1:?Usage: $0 <service> <namespace>}"
NAMESPACE="${2:?Usage: $0 <service> <namespace>}"

kubectl get service "$SERVICE" -n "$NAMESPACE"
kubectl get endpoints "$SERVICE" -n "$NAMESPACE"
kubectl get endpointslices -n "$NAMESPACE" \
  -l "kubernetes.io/service-name=$SERVICE"
```

An endpoint does not prove business functionality; follow with an application request.

## 34.4 Timestamp Helper

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
printf 'UTC: '; date -u -Is
printf 'Local: '; date -Is
printf 'Epoch: '; date +%s
```

Use UTC in shared timelines unless the organization specifies otherwise. Always label timezone.

## 34.5 Script Safety

Use read-only identities for collection, quote variables, avoid secret output, avoid destructive commands, capture failures, include timestamps, test in QA, and version-control scripts.

# 35. Post-Incident Review

## 35.1 Blameless Review

Focus on how system design and process allowed the incident. Prefer evidence-based findings such as "deployment validation did not calculate aggregate database connection demand" over personal blame.

Blameless review does not remove accountability; it improves controls, design, and operational learning.

## 35.2 Review Agenda

1. Confirm incident scope.
2. Review customer impact.
3. Walk through the timeline.
4. Separate facts from assumptions.
5. Identify contributing conditions.
6. Evaluate detection and mitigation.
7. Review communication and handoffs.
8. Agree corrective actions.
9. Assign owners and due dates.

## 35.3 Action Item Quality

Each action needs a specific deliverable, named owner, due date, priority, measurable completion criterion, and verification method. "Improve monitoring" is vague. "Alert when order-service connection acquisition p95 exceeds the approved threshold for ten minutes and validate in QA" is actionable.

## 35.4 Track Completion

Track open and overdue actions, accepted risks, deferred actions and approvers, completion evidence, and an effectiveness review. Creating actions is not the same as completing them.

# 36. Production Readiness Review

## 36.1 Application

- [ ] Health endpoints documented.
- [ ] Readiness and liveness probes tested.
- [ ] Graceful shutdown configured.
- [ ] Requests and limits measured.
- [ ] Timeouts explicit.
- [ ] Retries bounded.
- [ ] Idempotency implemented where required.
- [ ] Logs include correlation identifiers.
- [ ] Service indicators exposed.

## 36.2 Platform

- [ ] Nodes span intended Availability Zones.
- [ ] Cluster capacity has headroom.
- [ ] Pod disruption budgets are appropriate.
- [ ] Network policies reviewed.
- [ ] Service accounts use least privilege.
- [ ] Ingress and target health checks tested.
- [ ] Storage recovery documented.
- [ ] Autoscaling limits reviewed.

## 36.3 Data and Messaging

- [ ] Database backups verified.
- [ ] Restore procedure tested.
- [ ] Connection budget documented.
- [ ] Schema migration backward compatible.
- [ ] Queue retry and DLQ policy documented.
- [ ] Consumer idempotency tested.
- [ ] Kafka offset recovery documented.
- [ ] Data reconciliation process exists.

## 36.4 Observability

- [ ] Critical service dashboards exist.
- [ ] Alert rules have owners.
- [ ] Notification delivery tested.
- [ ] Logs searchable.
- [ ] Retention appropriate.
- [ ] Monitoring fallback documented.
- [ ] Runbooks linked from alerts.
- [ ] SLOs approved and measurable.

# 37. Final Simulation — Multi-Service Checkout Outage

## 37.1 Objective

In QA, simulate checkout disruption caused by database connection pressure and a downstream queue backlog. Demonstrate declaration, coordination, evidence collection, dependency isolation, mitigation, validation, and RCA.

## 37.2 Preconditions

Use QA only, an approved test window, disposable test data, a test payment provider or mock, dedicated test queue, known-good image digest, tested rollback, enabled monitoring, named controller/responders, and documented stop conditions.

## 37.3 Exercise Stages

1. Record baseline request rate, error rate, latency, pod health, DB connections, and queue depth.
2. Introduce the approved QA-only database pressure condition.
3. Confirm alerts fire and on-call acknowledges.
4. Investigate logs, pool metrics, DB metrics, and queue health.
5. Apply the pre-approved mitigation and record approver.
6. Verify pool recovery, order processing, queue drain, and successful test checkout.
7. Conduct review and assign actions.

## 37.4 Stop Conditions

Stop if shared services are affected, data integrity is uncertain, test data reaches production, unexpected external payments occur, the controller cannot restore the environment, monitoring is unavailable, or the approved window ends.

## 37.5 Scorecard

| Capability | Evidence | Result |
|---|---|---|
| Detection | Alert timestamp | Pass / Fail |
| Acknowledgement | On-call record | Pass / Fail |
| Impact assessment | Affected journey | Pass / Fail |
| Investigation | Evidence bundle | Pass / Fail |
| Mitigation | Approved change record | Pass / Fail |
| Recovery | Synthetic journey | Pass / Fail |
| Communication | Timed updates | Pass / Fail |
| RCA | Evidence-based analysis | Pass / Fail |

Close only after the fault is removed, QA is restored, test data reconciled, queues are in expected state, no unexpected workload remains, evidence is stored, and follow-up owners are assigned.

# 38. Production Incident Command Reference

## 38.1 Cluster and Node

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

## 38.2 Workloads

```bash
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get replicasets -A
kubectl get daemonsets -A
kubectl get statefulsets -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --since=30m
kubectl logs <pod-name> -n <namespace> --previous
```

## 38.3 Networking and Storage

```bash
kubectl get svc -A
kubectl get ingress -A
kubectl get endpoints -A
kubectl get endpointslices -A
kubectl get networkpolicy -A
kubectl get pvc -A
kubectl get pv
kubectl get storageclass
```

## 38.4 Deployment and AWS

```bash
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl get rs -n <namespace>
aws sts get-caller-identity
aws eks describe-cluster --name roboshop-prod --region ap-south-1
aws eks list-nodegroups --cluster-name roboshop-prod --region ap-south-1
```

Use approved IAM permissions and review output for sensitive infrastructure details.

# 39. Incident Anti-Patterns

Avoid restarting every pod without evidence; scaling without checking dependencies; deleting pods before capturing logs; flushing Redis; purging queues; resetting Kafka offsets casually; running destructive SQL; increasing pools without aggregate capacity calculations; disabling security controls; granting broad IAM; unrecorded manual changes; fighting GitOps; rolling back incompatible schema changes; retrying payments without idempotency; declaring recovery because pods are Running; closing before customer validation; silencing all alerts; sharing secrets; treating hypotheses as facts; or closing RCA without owners.

# 40. Final Chapter Deliverables

```text
docs/
└── operations/
    └── incidents/
        ├── incident-response-policy.md
        ├── severity-matrix.md
        ├── incident-commander-runbook.md
        ├── production-incident-template.md
        ├── incident-handoff-template.md
        ├── production-checkout-outage.md
        ├── database-connection-exhaustion.md
        ├── kubernetes-node-failure.md
        ├── ingress-502-503.md
        ├── rabbitmq-backlog.md
        ├── kafka-consumer-lag.md
        ├── redis-availability.md
        ├── argocd-sync-failure.md
        ├── terraform-partial-apply.md
        ├── security-incident-response.md
        ├── regional-disaster-recovery.md
        ├── post-incident-review.md
        └── recovery-validation-checklist.md
```

These are recommended artifacts. Create only files actually implemented and validated; do not describe placeholder dashboards or rules as deployed controls.

# 41. Final Knowledge Validation

1. Who can declare an incident?
2. Who owns technical investigation?
3. What belongs in the incident timeline?
4. What must a shift handoff contain?
5. When should severity increase?
6. How do you distinguish scheduling failure from startup failure?
7. How do you investigate a NotReady node?
8. What evidence should be captured before restarting a pod?
9. How do readiness and liveness probes differ?
10. How do you investigate a Service with no endpoints?
11. How do you distinguish IAM authorization from network failure?
12. How do you investigate unhealthy ALB targets?
13. What causes EKS pod IP exhaustion?
14. How do you investigate a Route 53 change?
15. What must be checked before regional failover?
16. How do you calculate aggregate DB connection demand?
17. Why can increasing replicas worsen DB exhaustion?
18. How do you investigate a slow query?
19. Why is blind payment retry dangerous?
20. How do you recover a message backlog safely?
21. Why should Kafka offsets not be reset casually?
22. How do you prevent duplicate message processing?
23. How do you recover from partial Terraform apply?
24. How do you investigate Argo CD drift?
25. What is the safe rollback path for GitOps?
26. What happens after a secret is exposed?
27. How do you handle scanner outage without silently bypassing gates?
28. What proves a customer journey has recovered?
29. What makes an RCA evidence-based?
30. How do you turn findings into measurable actions?

# 42. Chapter Completion Checklist

- [ ] Incident command roles understood.
- [ ] Severity policy reviewed.
- [ ] Evidence collection practiced.
- [ ] Request path mapped.
- [ ] Kubernetes failure scenarios practiced in QA.
- [ ] DNS and networking scenarios practiced.
- [ ] Database scenarios reviewed.
- [ ] Redis and messaging scenarios reviewed.
- [ ] ALB and ingress scenarios reviewed.
- [ ] Argo CD and Helm recovery practiced.
- [ ] Terraform recovery reviewed.
- [ ] Security incident escalation understood.
- [ ] Monitoring failure scenarios reviewed.
- [ ] Incident scripts tested in QA.
- [ ] Final multi-service simulation completed.
- [ ] RCA completed with evidence.
- [ ] Corrective actions assigned and tracked.
- [ ] Runbooks reviewed by service owners.

**End of expanded Chapter 208 — Production Incident.**

## 29.1 Scenario — DNS resolution failure

**Symptom:** Pods cannot resolve internal service names or external dependency hostnames.

**Investigation priorities:**
- CoreDNS health, service name, namespace, DNS suffix, VPC resolver, private hosted zones, and DNS egress policy.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pods -n kube-system -o wide; kubectl get svc -n kube-system; kubectl get endpointslices -n kube-system; nslookup order.production.svc.cluster.local
```

**Mitigation:** Restore the resolver or revert the incorrect DNS configuration through approved change control.

**Recovery validation:** Resolve names from multiple namespaces and complete an application request.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.2 Scenario — CoreDNS unavailable

**Symptom:** Workloads report intermittent DNS failures across namespaces.

**Investigation priorities:**
- CoreDNS replicas, scheduling, resource pressure, configuration, upstream resolver, and node networking.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get deployment coredns -n kube-system; kubectl describe deployment coredns -n kube-system; kubectl logs -n kube-system -l k8s-app=kube-dns --since=30m
```

**Mitigation:** Restore replica availability and correct the proven configuration or capacity issue. Do not delete all replicas simultaneously.

**Recovery validation:** Verify DNS from multiple nodes and application namespaces.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.3 Scenario — EKS API access failure

**Symptom:** kubectl times out, returns Unauthorized, or returns Forbidden.

**Investigation priorities:**
- Separate endpoint connectivity, IAM authentication, EKS access mapping, Kubernetes RBAC, and API readiness.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
aws sts get-caller-identity; aws eks describe-cluster --name roboshop-prod --region ap-south-1; kubectl cluster-info; kubectl get --raw=/readyz
```

**Mitigation:** Restore the approved network path or narrowly scoped identity mapping; never grant cluster-admin as a shortcut.

**Recovery validation:** Run read-only commands using the intended operator identity.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.4 Scenario — Node NotReady

**Symptom:** Worker nodes become NotReady and workloads are disrupted.

**Investigation priorities:**
- Node conditions, kubelet, runtime, disk/memory pressure, CNI, EC2 status checks, and node-group events.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get nodes -o wide; kubectl describe node <node>; kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
```

**Mitigation:** Protect critical workloads and use the node repair or replacement runbook after checking capacity, disruption budgets, and storage.

**Recovery validation:** Confirm replacement capacity and workload readiness across Availability Zones.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.5 Scenario — Node DiskPressure

**Symptom:** Pods are evicted or cannot be scheduled because node storage is exhausted.

**Investigation priorities:**
- Filesystem usage, inode exhaustion, container images, logs, ephemeral storage, and runtime directories.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe node <node>; df -h; df -i; sudo journalctl --disk-usage
```

**Mitigation:** Use supported log/image cleanup or node replacement procedures. Never manually delete container runtime internals.

**Recovery validation:** Confirm disk and inode headroom and verify workloads recover.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.6 Scenario — Node MemoryPressure

**Symptom:** Pods are evicted or suffer OOM events on a worker node.

**Investigation priorities:**
- Working set, requests/limits, daemonsets, system reservations, and workload growth.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe node <node>; kubectl top node <node>; kubectl top pods -A --sort-by=memory
```

**Mitigation:** Reduce noncritical load or add approved capacity; check remaining-node headroom before rescheduling.

**Recovery validation:** Verify pressure clears without transferring the problem to another node.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.7 Scenario — CNI or pod networking failure

**Symptom:** Pods run but cannot communicate across nodes or obtain addresses.

**Investigation priorities:**
- aws-node health, subnet IP capacity, ENI limits, security groups, routes, network ACLs, and policies.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pods -n kube-system -o wide; kubectl get ds -n kube-system; kubectl describe node <node>
```

**Mitigation:** Follow the EKS networking runbook. Do not restart CNI across all nodes simultaneously.

**Recovery validation:** Test pod-to-pod, pod-to-service, DNS, and dependency connectivity.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.8 Scenario — ImagePullBackOff

**Symptom:** A workload cannot pull its image from ECR.

**Investigation priorities:**
- Image name/digest, repository existence, node IAM role, repository policy, network access, and image architecture.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe pod <pod> -n production; kubectl get deploy <name> -n production -o yaml; aws ecr describe-images --repository-name <repo> --region ap-south-1
```

**Mitigation:** Restore the correct immutable image reference or least-privilege ECR permission. Do not use an unverified latest tag.

**Recovery validation:** Confirm all replicas run the expected image digest.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.9 Scenario — Readiness probe failure

**Symptom:** Containers run but are not Ready and are removed from Service endpoints.

**Investigation priorities:**
- Probe path, port, scheme, startup duration, authentication, dependency checks, and readiness events.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe pod <pod> -n production; kubectl get endpointslices -n production; kubectl logs <pod> -n production
```

**Mitigation:** Correct the probe or application readiness behavior based on evidence; do not weaken checks just to make pods Ready.

**Recovery validation:** Confirm ready endpoints and execute the customer-facing request path.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.10 Scenario — Liveness probe restart loop

**Symptom:** Containers repeatedly restart despite apparently successful startup.

**Investigation priorities:**
- Probe timeout/frequency, slow dependencies, blocked threads, startup duration, and previous logs.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe pod <pod> -n production; kubectl logs <pod> -n production --previous; kubectl get events -n production
```

**Mitigation:** Use a startup probe where appropriate and tune liveness only after measuring startup and request behavior.

**Recovery validation:** Verify restart count stabilizes under sustained traffic.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.11 Scenario — HPA cannot scale

**Symptom:** Traffic rises but replica count does not increase.

**Investigation priorities:**
- Metrics API, resource requests, HPA target, max replicas, pending pods, and node capacity.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get hpa -n production; kubectl describe hpa <name> -n production; kubectl get apiservice; kubectl top pods -n production
```

**Mitigation:** Restore metrics or approved capacity. Check database and broker connection budgets before raising replica limits.

**Recovery validation:** Generate controlled QA load and verify desired/current replicas converge.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.12 Scenario — Cluster autoscaler failure

**Symptom:** Pods remain Pending although HPA requests more replicas.

**Investigation priorities:**
- Node-group maximum, EC2 capacity, subnet IPs, instance constraints, taints, affinity, PVC zones, and autoscaler logs.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe pod <pending-pod> -n production; kubectl get events -A --sort-by=.metadata.creationTimestamp; kubectl get nodes
```

**Mitigation:** Correct the actual capacity or scheduling constraint; do not remove topology rules blindly.

**Recovery validation:** Verify pods schedule and zone distribution remains healthy.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.13 Scenario — Service has no endpoints

**Symptom:** A Service exists but has no ready backends.

**Investigation priorities:**
- Service selectors, pod labels, readiness, target port, EndpointSlices, and namespace.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get svc <service> -n production -o yaml; kubectl get pods -n production --show-labels; kubectl get endpointslices -n production
```

**Mitigation:** Correct selector, label, port, or readiness through source control.

**Recovery validation:** Confirm endpoint addresses and make an in-cluster request.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.14 Scenario — NetworkPolicy blocks traffic

**Symptom:** DNS works but a workload cannot connect to a dependency port.

**Investigation priorities:**
- Ingress/egress rules, pod selectors, namespace selectors, ports, and DNS egress.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get networkpolicy -A; kubectl describe networkpolicy <policy> -n production
```

**Mitigation:** Apply the smallest approved policy change; do not delete all NetworkPolicies.

**Recovery validation:** Test intended allowed flow and an unrelated denied flow.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.15 Scenario — ALB target group unhealthy

**Symptom:** Users receive 502/503 and target health reports unhealthy.

**Investigation priorities:**
- Target type, health path, success matcher, target port, security groups, pod readiness, and ingress annotations.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl describe ingress <ingress> -n production; kubectl get svc -n production; kubectl get endpointslices -n production
```

**Mitigation:** Correct target registration, health check, service mapping, or application readiness through approved change control.

**Recovery validation:** Confirm healthy targets and test the external request path.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.16 Scenario — TLS certificate expired

**Symptom:** Clients report certificate validity or TLS handshake errors.

**Investigation priorities:**
- Certificate dates, chain, hostname, listener attachment, renewal controller, and secret version.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
openssl s_client -connect <host>:443 -servername <host> </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```

**Mitigation:** Renew and attach the correct certificate through approved certificate management; never expose private keys.

**Recovery validation:** Validate certificate chain, hostname, expiry, and application requests.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.17 Scenario — Route 53 record incorrect

**Symptom:** Users resolve an old endpoint or different endpoints across resolvers.

**Investigation priorities:**
- Record type, alias target, routing policy, health check, TTL, and recent change history.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
dig <hostname>; dig <hostname> A; dig <hostname> CNAME
```

**Mitigation:** Restore the intended record through change control and account for resolver caching.

**Recovery validation:** Verify authoritative answer and multiple recursive resolvers.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.18 Scenario — RDS failover

**Symptom:** Connections drop during database failover or maintenance.

**Investigation priorities:**
- RDS events, endpoint resolution, pool recovery, transaction retry behavior, and replication state.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
nslookup <rds-endpoint>; inspect RDS events and application connection errors
```

**Mitigation:** Allow supported endpoint reconnection and bounded retry with jitter; avoid hardcoded instance IPs.

**Recovery validation:** Verify reads, writes, transaction integrity, and pool recovery.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.19 Scenario — RDS storage exhaustion

**Symptom:** Writes fail or database availability degrades as free storage falls.

**Investigation priorities:**
- Free storage trend, log growth, temporary files, table growth, and storage autoscaling.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review RDS free storage, database logs, growth trend, and recent schema changes
```

**Mitigation:** Use database-specific cleanup or approved storage scaling; never delete database files manually.

**Recovery validation:** Confirm storage headroom and successful write transactions.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.20 Scenario — Slow database queries

**Symptom:** Application latency increases while database connections remain available.

**Investigation priorities:**
- Query latency, execution plans, locks, long transactions, indexes, statistics, and recent releases.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review approved query performance tooling, slow query logs, and application query timings
```

**Mitigation:** Tune the identified query or reduce load with database-owner approval; do not kill sessions blindly.

**Recovery validation:** Compare query and customer request latency before and after.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.21 Scenario — Database deadlock

**Symptom:** Some transactions fail with deadlock errors while other traffic continues.

**Investigation priorities:**
- Deadlock diagnostics, transaction order, lock duration, missing indexes, and retry behavior.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review database deadlock logs and correlate transaction/request identifiers
```

**Mitigation:** Correct transaction ordering or indexing; use bounded retries only for safe idempotent operations.

**Recovery validation:** Run concurrent transaction tests and verify no duplicate business effects.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.22 Scenario — Credential rotation failure

**Symptom:** Applications receive authentication failures after a database credential change.

**Investigation priorities:**
- Secret version, rollout status, stale connections, user state, and secret delivery mechanism.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect secret version metadata without printing values; review DB authentication errors and pod rollout timestamps
```

**Mitigation:** Complete staged rotation and restart only workloads that require it after confirming the new secret is available.

**Recovery validation:** Verify every replica authenticates and no secret value appears in logs.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.23 Scenario — Redis memory eviction

**Symptom:** Cache hit ratio falls and database load rises.

**Investigation priorities:**
- Memory, maxmemory policy, eviction counters, key sizes, and request growth.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Use approved Redis INFO metrics and application cache metrics
```

**Mitigation:** Restore capacity or tune cache policy based on workload; never FLUSHDB or FLUSHALL.

**Recovery validation:** Verify hit ratio, eviction rate, and database load recover.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.24 Scenario — Redis failover

**Symptom:** Clients receive resets or writes fail during primary change.

**Investigation priorities:**
- Managed service events or cluster topology, endpoint use, DNS caching, reconnect logic, and write routing.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Check Redis service events, pod/service state, client errors, and endpoint resolution
```

**Mitigation:** Restore topology-aware reconnect behavior; never hardcode a pod or primary IP.

**Recovery validation:** Verify writes reach the current primary and replicas are healthy.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.25 Scenario — RabbitMQ disk alarm

**Symptom:** Broker blocks publishers or reports a disk alarm.

**Investigation priorities:**
- Disk free space, persistent messages, queue growth, consumer rate, and node health.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect RabbitMQ alarms, queue metrics, pod events, and broker logs
```

**Mitigation:** Restore disk headroom or consumer capacity; do not purge queues without business approval.

**Recovery validation:** Verify publisher flow resumes and queues drain safely.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.26 Scenario — RabbitMQ consumer failure

**Symptom:** Queue depth grows because consumers are disconnected or failing.

**Investigation priorities:**
- Consumer readiness, connection count, unacknowledged messages, retries, and downstream health.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect RabbitMQ queue/consumer metrics and consumer application logs
```

**Mitigation:** Restore consumers and correct the failure; acknowledge only after successful business processing.

**Recovery validation:** Verify queue age and depth trend down without duplicate effects.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.27 Scenario — Kafka broker unavailable

**Symptom:** Producers or consumers report broker connection errors.

**Investigation priorities:**
- Broker readiness, listeners, advertised listeners, DNS, storage, quorum/controller health, and network policy.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect broker/operator status, logs, and producer/consumer errors
```

**Mitigation:** Follow the deployed Kafka distribution recovery procedure; do not assume all topologies are identical.

**Recovery validation:** Verify metadata refresh, producer delivery, and consumer group recovery.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.28 Scenario — Kafka consumer lag

**Symptom:** Events are produced but downstream processing falls behind.

**Investigation priorities:**
- Consumer group lag, partition assignment, processing latency, poison messages, and downstream dependency health.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect consumer group lag and consumer application metrics
```

**Mitigation:** Restore consumers or processing capacity only after checking partition count, ordering, and downstream capacity.

**Recovery validation:** Verify lag declines and oldest-message age returns to target.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.29 Scenario — Duplicate message processing

**Symptom:** Orders, notifications, or inventory updates occur more than once.

**Investigation priorities:**
- Producer retry, redelivery, ack timing, offset commits, and transaction boundaries.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Correlate business idempotency keys with broker metadata and application logs
```

**Mitigation:** Implement idempotency/deduplication at the business boundary; broker semantics alone do not guarantee business exactly-once.

**Recovery validation:** Replay a controlled duplicate in QA and verify one business effect.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.30 Scenario — Dead-letter queue growth

**Symptom:** Messages repeatedly fail and accumulate in a DLQ.

**Investigation priorities:**
- Exception type, retry count, schema version, message age, and downstream availability.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect an approved sample of DLQ metadata and consumer error logs
```

**Mitigation:** Classify transient/permanent failures and replay only after validating idempotency and downstream capacity.

**Recovery validation:** Verify DLQ trend and replay outcomes.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.31 Scenario — Argo CD OutOfSync

**Symptom:** Live cluster differs from Git desired state.

**Investigation priorities:**
- Recent commit, Helm values, manual changes, admission mutations, ignored diffs, and secret rendering.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
argocd app get <app>; argocd app diff <app>; argocd app history <app>
```

**Mitigation:** Correct desired state in Git or revert unintended changes; understand diff before sync.

**Recovery validation:** Confirm desired revision, sync status, health, and live image digest.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.32 Scenario — Argo CD sync failure

**Symptom:** A committed release does not become healthy.

**Investigation priorities:**
- Manifest validation, missing CRDs, quotas, RBAC, image pull, admission, hooks, and resource ordering.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
argocd app get <app>; kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

**Mitigation:** Fix source-controlled manifests or dependency ordering and retry reviewed sync.

**Recovery validation:** Confirm resources are healthy at the expected revision.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.33 Scenario — Helm upgrade failure

**Symptom:** Upgrade fails or leaves resources partially updated.

**Investigation priorities:**
- Release history, hooks, immutable fields, timeout, readiness, and rendered manifest.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
helm list -n production; helm status <release> -n production; helm history <release> -n production
```

**Mitigation:** Correct chart/values in Git for GitOps-owned releases; avoid competing direct rollback operations.

**Recovery validation:** Verify release revision and live resource state.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.34 Scenario — Terraform partial apply

**Symptom:** Infrastructure changes stop after some resources are created or modified.

**Investigation priorities:**
- CI log, backend/workspace, lock, state versus cloud reality, provider error, and out-of-band changes.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
terraform plan; inspect backend/workspace and cloud inventory with approved credentials
```

**Mitigation:** Preserve state and logs, reconcile actual resources, and use reviewed state recovery only when necessary.

**Recovery validation:** A reviewed plan shows only intended changes before next apply.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.35 Scenario — Terraform state lock contention

**Symptom:** A pipeline reports that state is locked.

**Investigation priorities:**
- Active operation, lock owner, backend health, and whether a prior process terminated.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect backend lock metadata and pipeline execution history
```

**Mitigation:** Never force-unlock while an apply may be active; use supported recovery with approval.

**Recovery validation:** Run a read-only plan and confirm no concurrent state operation.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.36 Scenario — CI/CD pipeline failure

**Symptom:** A release stops at checkout, tests, scanning, build, push, manifest update, sync, or smoke test.

**Investigation priorities:**
- Failed stage, commit SHA, image digest, pipeline ID, tool version, and dependency outage.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect pipeline job logs and artifact metadata
```

**Mitigation:** Fix the failed stage; do not bypass security gates or promote an unverified image.

**Recovery validation:** Run pipeline and confirm the same immutable digest is promoted.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.37 Scenario — Scanner outage

**Symptom:** Trivy, SonarQube, or Veracode cannot complete a required scan.

**Investigation priorities:**
- Scanner availability, credentials, network, project config, and service status.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect scanner health and pipeline integration logs without exposing tokens
```

**Mitigation:** Use only the formal time-bound security exception process with approver and compensating controls.

**Recovery validation:** Verify scanner recovery and complete deferred scans.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.38 Scenario — Secret exposure

**Symptom:** Credential is committed, printed, or included in an artifact.

**Investigation priorities:**
- Credential scope, exposure location, access logs, affected systems, and artifact access.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Preserve evidence; review repository, CI logs, and cloud audit records under security procedures
```

**Mitigation:** Treat as compromised, revoke/rotate, restrict artifacts, investigate misuse, and remove secret from active paths.

**Recovery validation:** Verify new credential use and confirm old credential is invalid.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.39 Scenario — Image supply-chain concern

**Symptom:** A deployed image contains a critical vulnerability or suspicious component.

**Investigation priorities:**
- Exact image digest, provenance, scanner result, runtime exposure, and patched image availability.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.containerStatuses[*]}{.imageID}{" "}{end}{"\n"}{end}'
```

**Mitigation:** Escalate suspected compromise; rebuild from trusted inputs, scan, test, and promote the same immutable digest.

**Recovery validation:** Verify every replica runs the approved replacement digest.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.40 Scenario — Quota exhaustion

**Symptom:** New pods cannot be created because namespace quota is consumed.

**Investigation priorities:**
- CPU/memory requests, pod count, PVC quota, terminating pods, and LimitRange defaults.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get resourcequota -n production; kubectl describe resourcequota -n production; kubectl get limitrange -n production
```

**Mitigation:** Adjust workload or quota only after confirming cluster capacity and tenant policy.

**Recovery validation:** Confirm scheduling and namespace quota headroom.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.41 Scenario — PersistentVolume mount failure

**Symptom:** Pods remain Pending or show volume attach/mount errors.

**Investigation priorities:**
- PVC/PV state, StorageClass, access mode, CSI driver, zone constraints, and cloud volume state.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pvc -n production; kubectl get pv; kubectl describe pod <pod> -n production
```

**Mitigation:** Follow CSI/storage recovery; never delete PVC/PV without data-loss and reclaim-policy review.

**Recovery validation:** Verify mount, filesystem, and application data integrity.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.42 Scenario — Availability Zone disruption

**Symptom:** Nodes or dependencies in one AZ become unavailable.

**Investigation priorities:**
- Subnet, node, load balancer target, zonal storage, and remaining-zone capacity.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get nodes -L topology.kubernetes.io/zone; kubectl get pods -A -o wide; review cloud health events
```

**Mitigation:** Use approved workload redistribution or DR procedures; do not detach stateful volumes casually.

**Recovery validation:** Verify critical replicas span healthy zones and storage is consistent.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.43 Scenario — Regional dependency failure

**Symptom:** A regional service or dependency becomes unavailable.

**Investigation priorities:**
- Regional/global dependencies, replication lag, DNS failover, image/secret availability, and write ownership.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review regional health, replication status, DNS policy, and recovery-region capacity
```

**Mitigation:** Follow DR plan; failover requires data consistency, write fencing, and explicit approval.

**Recovery validation:** Measure actual RTO/RPO and validate customer workflows.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.44 Scenario — Traffic spike

**Symptom:** Request volume rises and latency/errors increase.

**Investigation priorities:**
- Route-level request rate, client segment, HPA, node capacity, DB connections, cache hit ratio, and queue depth.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review Prometheus request metrics, HPA, kubectl top, DB metrics, and broker metrics
```

**Mitigation:** Use approved rate limits, admission control, caching, or capacity scaling with downstream budget checks.

**Recovery validation:** Verify legitimate traffic succeeds and dependencies remain safe.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.45 Scenario — Retry storm

**Symptom:** A slow dependency receives amplified traffic from retries.

**Investigation priorities:**
- Retry rate, timeout hierarchy, nested retries, queue depth, and dependency request volume.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Correlate client retry metrics with downstream request rate and error rate
```

**Mitigation:** Use bounded exponential backoff with jitter and circuit breaking where appropriate.

**Recovery validation:** Confirm retry amplification drops and dependency recovers.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.46 Scenario — Cascading microservice failure

**Symptom:** One slow downstream service causes upstream failures.

**Investigation priorities:**
- First latency change, dependency graph, thread/connection pools, queues, timeouts, and retries.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Compare service latency/error metrics and correlated request logs
```

**Mitigation:** Isolate initiating dependency and protect critical paths; avoid restarting all services.

**Recovery validation:** Verify downstream recovery and upstream backlog drains.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.47 Scenario — Thread pool exhaustion

**Symptom:** Requests wait despite moderate CPU usage.

**Investigation priorities:**
- Active/queued threads, blocked thread stacks, downstream latency, and application pool settings.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect application thread metrics and approved thread dumps
```

**Mitigation:** Correct blocking dependency or pool sizing based on measurements; avoid arbitrary thread increases.

**Recovery validation:** Verify queue wait and request latency recover under controlled load.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.48 Scenario — File descriptor exhaustion

**Symptom:** Application cannot open sockets or files.

**Investigation priorities:**
- Process limits, open descriptor count, socket churn, and leaked resources.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
ulimit -n; cat /proc/<pid>/limits; ls /proc/<pid>/fd | wc -l
```

**Mitigation:** Fix resource leaks and adjust limits only after measuring process and node capacity.

**Recovery validation:** Verify descriptor count stabilizes under sustained traffic.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.49 Scenario — Clock skew

**Symptom:** Authentication, TLS, event ordering, or log correlation fails.

**Investigation priorities:**
- Node time sync, container timezone assumptions, and external system timestamps.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
date -u -Is; compare timestamps across nodes and dependencies
```

**Mitigation:** Restore approved time synchronization; do not manually change production clocks.

**Recovery validation:** Confirm timestamps align and authentication succeeds.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.50 Scenario — Log pipeline backpressure

**Symptom:** Recent application logs are missing or delayed in Kibana.

**Investigation priorities:**
- Shipper health, Logstash queues, Elasticsearch ingest, disk watermarks, and index errors.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pods -A | grep -E 'filebeat|logstash|elasticsearch'; inspect component logs and queue metrics
```

**Mitigation:** Restore pipeline capacity without dropping buffered data; use alternate evidence meanwhile.

**Recovery validation:** Verify log freshness and ingestion rate recover.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.51 Scenario — Elasticsearch disk watermark

**Symptom:** Elasticsearch stops allocating shards or rejects writes.

**Investigation priorities:**
- Disk use, shard allocation, index growth, retention, and snapshot availability.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect cluster health, node disk metrics, and shard allocation explanation
```

**Mitigation:** Restore headroom and apply approved retention; do not delete indices blindly.

**Recovery validation:** Verify shard allocation and indexing recover.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.52 Scenario — Prometheus scrape failure

**Symptom:** Dashboards show missing or stale metrics.

**Investigation priorities:**
- Targets, scrape errors, ServiceMonitor/PodMonitor selectors, labels, endpoints, auth, and policy.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get servicemonitor -A; kubectl get podmonitor -A; kubectl get endpointslices -A
```

**Mitigation:** Restore the broken discovery or scrape stage; distinguish instrumentation from collection failure.

**Recovery validation:** Verify target health and metric freshness.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.53 Scenario — Grafana unavailable

**Symptom:** Responders cannot access dashboards.

**Investigation priorities:**
- Pod health, service endpoints, ingress, persistence, and authentication provider.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get pods -A | grep grafana; kubectl get svc -A | grep grafana; inspect logs
```

**Mitigation:** Use Prometheus or approved fallback views while restoring Grafana.

**Recovery validation:** Confirm dashboard access and data freshness.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.54 Scenario — Alertmanager notification failure

**Symptom:** Alerts fire but on-call is not notified.

**Investigation priorities:**
- Routes, receiver config, integration status, silences, grouping, and notification errors.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect Alertmanager status and logs; run an approved test notification
```

**Mitigation:** Restore correct receiver and routing; protect webhook credentials.

**Recovery validation:** Confirm test alerts reach intended on-call destination.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.55 Scenario — Alert storm

**Symptom:** One failure generates excessive notifications.

**Investigation priorities:**
- Duplicate rules, per-pod alerting, missing grouping, absent inhibition, and short pending duration.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review firing alerts grouped by cluster, namespace, and service
```

**Mitigation:** Apply scoped, time-limited silences with owner/expiry; retain independent critical alerts.

**Recovery validation:** Verify notification volume normalizes without hiding new failures.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.56 Scenario — Monitoring data loss

**Symptom:** Metrics or logs are missing for an incident window.

**Investigation priorities:**
- Collection, transport, retention, storage failure, and scrape/drop errors.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Compare Prometheus targets, shipper logs, cloud metrics, and application evidence
```

**Mitigation:** Record observability gap and use independent evidence; do not present estimates as exact measurements.

**Recovery validation:** Verify collection resumes and retention/backup controls are reviewed.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.57 Scenario — Partial rollout availability

**Symptom:** Some pods run new version while others run old version.

**Investigation priorities:**
- ReplicaSets, pod image digests, readiness, rollout strategy, and schema/message compatibility.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl rollout status deployment/<name> -n production; kubectl get rs -n production; kubectl get pods -n production -o wide
```

**Mitigation:** Pause or roll back only if old version is compatible with current schema and messages.

**Recovery validation:** Verify all replicas converge on approved digest and business requests succeed.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.58 Scenario — Database migration failure

**Symptom:** Application deployment succeeds but requests fail after schema change.

**Investigation priorities:**
- Migration version, applied statements, locks, compatibility, and reversibility.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review migration job logs, DB migration table, locks, and application errors
```

**Mitigation:** Use expand-and-contract; do not run destructive rollback SQL without a tested recovery plan.

**Recovery validation:** Verify application compatibility and data integrity.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.59 Scenario — Data corruption

**Symptom:** Orders, inventory, or payment records are inconsistent.

**Investigation priorities:**
- Affected time window, transaction/audit logs, duplicate events, and write path.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Preserve secure identifiers and transaction evidence; engage data owner
```

**Mitigation:** Escalate, isolate faulty writes only with approval, and use audited reconciliation.

**Recovery validation:** Reconcile affected records and obtain business/data-owner signoff.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.60 Scenario — Payment provider failure

**Symptom:** Payment requests time out or return provider errors.

**Investigation priorities:**
- Provider status, transaction reference, authorization state, idempotency key, and response loss.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Correlate provider reference and application request logs; use provider reconciliation process
```

**Mitigation:** Do not assume timeout means failure; reconcile before retrying to avoid duplicate charges.

**Recovery validation:** Confirm each test transaction has one correct final state.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.61 Scenario — Inventory inconsistency

**Symptom:** Available stock differs from confirmed orders.

**Investigation priorities:**
- Reservation/decrement/cancel events, duplicate messages, transaction boundary, and compensation logic.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Compare order, inventory, and event records using approved read-only queries
```

**Mitigation:** Use audited reconciliation and business approval; do not bulk-edit stock manually.

**Recovery validation:** Reconcile stock against source-of-truth records.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.62 Scenario — Notification backlog

**Symptom:** Checkout succeeds but notifications are delayed.

**Investigation priorities:**
- Consumer health, queue age/depth, provider rate limit, retry backlog, and downstream errors.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect queue metrics, consumer logs, and provider responses
```

**Mitigation:** Restore consumers with bounded retries and protect external provider limits.

**Recovery validation:** Verify oldest-message age and delivery outcomes.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.63 Scenario — External API throttling

**Symptom:** Dependency returns HTTP 429 or throttling responses.

**Investigation priorities:**
- Quota scope, response headers, request rate, retry policy, and client identity.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Inspect status distribution and provider quota metrics
```

**Mitigation:** Use bounded backoff, request shaping, and safe caching; do not evade provider limits.

**Recovery validation:** Verify request rate remains within quota and errors decline.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.64 Scenario — Security group regression

**Symptom:** A network rule change blocks application-to-dependency traffic.

**Investigation priorities:**
- Change record, affected SG rules, source/destination references, and audit evidence.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review approved cloud audit records and current security group rules
```

**Mitigation:** Restore last approved rule or add narrowest required rule; do not open broad CIDRs.

**Recovery validation:** Test intended connection and confirm unrelated paths remain blocked.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.65 Scenario — IAM permission regression

**Symptom:** Workload loses access to S3, SQS, Secrets Manager, or another AWS API.

**Investigation priorities:**
- Workload identity, trust policy, identity association, IAM policy, resource policy, and denial event.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
aws sts get-caller-identity; review role association and approved CloudTrail denial evidence
```

**Mitigation:** Restore only required least-privilege permission; never attach AdministratorAccess as workaround.

**Recovery validation:** Verify intended role and required actions.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.66 Scenario — IRSA or Pod Identity failure

**Symptom:** Pod cannot obtain expected AWS role credentials.

**Investigation priorities:**
- Service account, trust relationship, audience/association, projected token, and identity configuration.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
kubectl get sa <service-account> -n production -o yaml; inspect workload identity configuration
```

**Mitigation:** Repair identity through GitOps; never inject static AWS keys.

**Recovery validation:** Verify intended role identity from a controlled test workload.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.67 Scenario — Unexpected public exposure

**Symptom:** Internal or administrative endpoint is publicly reachable.

**Investigation priorities:**
- Load balancer, ingress, DNS, security group, endpoint authentication, and access logs.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review ingress, ALB listeners, DNS, SG rules, and access logs
```

**Mitigation:** Treat as security incident; restrict exposure through approved emergency change while preserving evidence.

**Recovery validation:** Security owner confirms exposure is closed and access reviewed.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

## 29.68 Scenario — Regional DR exercise failure

**Symptom:** Recovery region cannot serve required business functions.

**Investigation priorities:**
- Replication lag, RTO/RPO, DNS, write fencing, secrets, images, and regional capacity.
- Establish the first timestamp at which the symptom appeared.
- Compare the affected workload with a healthy peer in the same environment.
- Check recent deployments, configuration changes, and infrastructure events.
- Preserve evidence before changing state.

**Commands and evidence:**
```bash
Review DR exercise logs, replication status, and recovery region validation
```

**Mitigation:** Follow DR plan and obtain failover authority; DNS change alone is not complete recovery.

**Recovery validation:** Measure achieved RTO/RPO and verify full business journey.

**Escalation:** Involve the service owner and the relevant platform, database, networking, security, or cloud owner when evidence crosses team boundaries. Record the decision maker and approval.

**Prevention:** Add an actionable alert or runbook step for the failure signal, test the recovery path in QA, and assign an owner to verify the improvement.

