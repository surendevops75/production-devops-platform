# 149 — ALB 5xx Production Incident

> **Phase 18 — Incident Management**
>
> This chapter covers a production-grade **AWS Application Load Balancer (ALB) 5xx incident** in our EKS e-commerce platform. The focus is on identifying **where the 5xx response originated**, distinguishing ALB-generated failures from application-generated failures, tracing the request through Kubernetes, and restoring service without blindly restarting Pods.

---

## 1. What Is an ALB 5xx Incident?

A 5xx response means a server-side failure occurred somewhere in the request path.

Our architecture is:

```text
Internet
   |
   v
Route 53
   |
   v
CloudFront / WAF
   |
   v
ALB
   |
   v
AWS Load Balancer Controller
   |
   v
Kubernetes Service
   |
   v
Pods
   |
   +--> Redis
   +--> RDS
   +--> SQS/Kafka
   +--> External APIs
```

The critical question is:

> **Who generated the 5xx response?**

It could be:

```text
ALB
 |
 +--> Target connection failure
 +--> Target timeout
 +--> No healthy targets
 +--> Target reset
 |
Application
 |
 +--> Spring Boot 500
 +--> Dependency failure
 +--> Database error
 +--> Redis failure
 +--> Unhandled exception
```

These are very different incidents.

---

# 2. 5xx Does Not Automatically Mean ALB Is Broken

Suppose Grafana shows:

```text
HTTP 5xx = 15%
```

It is incorrect to immediately conclude:

> "ALB is failing."

The actual path may be:

```text
Client
  |
  v
ALB
  |
  v
order-service
  |
  v
RDS connection failure
  |
  v
Spring Boot returns 500
```

The ALB simply forwarded the request.

Therefore:

```text
ALB 5xx metric
```

and:

```text
Application HTTP 5xx metric
```

must be investigated together.

---

# 3. Types of ALB 5xx

Common ALB-related failure categories include:

```text
HTTP 500
HTTP 502
HTTP 503
HTTP 504
```

The exact meaning depends on where the response was generated and what happened between the ALB and target.

A useful starting mental model:

| Status | Typical investigation direction |
|---|---|
| 500 | Application/target-generated server error |
| 502 | Bad/invalid response, target connection/reset, protocol issue |
| 503 | No usable target / target availability problem |
| 504 | Target response timeout |

These are **investigation starting points**, not absolute root-cause definitions.

---

# 4. Production Incident Scenario

Suppose customers report:

> "Checkout is failing."

Grafana shows:

```text
order-service
5xx: 22%
```

ALB shows:

```text
HTTPCode_ELB_5XX_Count: low
HTTPCode_Target_5XX_Count: high
```

This is a major clue.

It suggests:

```text
ALB
 |
 +--> target
       |
       +--> order-service
               |
               +--> application returning 500
```

Now investigate the application and its dependencies.

---

# 5. First Principle

Never troubleshoot an ALB 5xx incident by simply doing:

```bash
kubectl delete pods --all
```

That can:

- destroy evidence
- increase traffic pressure on remaining Pods
- create more failures
- cause dependency connection spikes
- hide the actual root cause

Instead:

```text
Detect
  ↓
Classify
  ↓
Locate failure layer
  ↓
Correlate
  ↓
Mitigate
  ↓
Validate
  ↓
RCA
```

---

# 6. Production Incident Flow

```text
Alert
  ↓
Confirm customer impact
  ↓
Check ALB 5xx
  ↓
Separate ELB 5xx vs Target 5xx
  ↓
Check target health
  ↓
Check Kubernetes Service
  ↓
Check EndpointSlices
  ↓
Check Pods
  ↓
Check application 5xx
  ↓
Check logs
  ↓
Check traces
  ↓
Check dependencies
  ↓
Check recent deployment
  ↓
Mitigate
  ↓
Validate
  ↓
RCA
```

---

# 7. Step 1 — Confirm Customer Impact

Start with:

```bash
kubectl get pods -n ecommerce
```

Then:

```bash
kubectl get svc -n ecommerce
```

Then:

```bash
kubectl get endpointslice -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Check whether healthy Pods are actually behind the Service.

---

# 8. Check the ALB

AWS CLI:

```bash
aws elbv2 describe-load-balancers \
  --region ap-south-1
```

Find the ALB associated with the Kubernetes Ingress.

Then inspect listeners:

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <alb-arn> \
  --region ap-south-1
```

---

# 9. Find the Target Group

Get target groups:

```bash
aws elbv2 describe-target-groups \
  --load-balancer-arn <alb-arn> \
  --region ap-south-1
```

Then inspect target health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --region ap-south-1
```

Example:

```text
Target
10.0.12.31
State:
unhealthy
```

Now we know the ALB cannot consider that target healthy.

---

# 10. Target Health Is Critical

A target may be:

```text
healthy
unhealthy
initial
draining
```

If all targets are unhealthy:

```text
ALB
 |
 +--> no healthy targets
 |
 +--> customer failures
```

This is fundamentally different from:

```text
ALB
 |
 +--> healthy targets
       |
       +--> application returns 500
```

---

# 11. Check Kubernetes Service

```bash
kubectl get svc order-service -n ecommerce
```

Example:

```text
NAME            TYPE       CLUSTER-IP
order-service   ClusterIP  10.100.42.17
```

Then:

```bash
kubectl describe svc order-service -n ecommerce
```

Check:

```text
Selector
Port
TargetPort
Endpoints
```

A Service with no endpoints is a major clue.

---

# 12. Check EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Example healthy result:

```text
10.0.12.31
10.0.13.45
10.0.11.88
```

If there are no usable endpoints:

```text
Service
  |
  +--> no ready Pods
```

ALB traffic cannot reach the application.

---

# 13. Check Pods

```bash
kubectl get pods -n ecommerce \
  -l app=order-service
```

Look for:

```text
Running
Ready
CrashLoopBackOff
NotReady
Pending
Terminating
```

Example:

```text
order-service-abc    1/1 Running
order-service-def    0/1 Running
order-service-ghi    0/1 CrashLoopBackOff
```

The important metric is not simply:

```text
STATUS = Running
```

but:

```text
READY = 1/1
```

---

# 14. Readiness Is Critical

A Pod can be:

```text
Running
```

but:

```text
NotReady
```

For example:

```text
1/1 Running
```

means ready.

But:

```text
0/1 Running
```

means the container is running but not currently considered ready.

Readiness controls whether traffic should be sent to the workload.

---

# 15. Check Readiness Probe

```bash
kubectl describe pod <pod> -n ecommerce
```

Look for:

```text
Readiness:
```

Example:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

If the application becomes unhealthy:

```text
Pod
 ↓
Readiness fails
 ↓
Pod removed from ready endpoints
```

This can eventually cause:

```text
ALB
 ↓
503
```

if no healthy targets remain.

---

# 16. Important Probe Principle

Do not make readiness unnecessarily dependent on every external system.

Bad:

```text
readiness =
application
+
RDS
+
Redis
+
Kafka
+
Payment API
+
SQS
```

If Redis temporarily fails:

```text
Redis failure
 ↓
Readiness fails
 ↓
All Pods removed
 ↓
ALB has no targets
 ↓
503
```

This can turn a dependency problem into a complete application outage.

---

# 17. Check ALB Target 5xx vs ALB 5xx

ALB CloudWatch metrics are extremely useful.

Conceptually distinguish:

```text
HTTPCode_ELB_5XX_Count
```

from:

```text
HTTPCode_Target_5XX_Count
```

The first points toward failures generated by the load balancer.

The second points toward responses generated by targets.

This distinction dramatically narrows the investigation.

---

# 18. Example — Target 5xx High

Suppose:

```text
HTTPCode_ELB_5XX_Count     = 2
HTTPCode_Target_5XX_Count  = 1,500
```

This strongly suggests:

```text
ALB
  |
  v
order-service
  |
  +--> returning HTTP 500
```

Investigate:

```text
application logs
exceptions
database
Redis
external APIs
recent deployment
```

---

# 19. Example — ELB 5xx High

Suppose:

```text
HTTPCode_ELB_5XX_Count     = 1,200
HTTPCode_Target_5XX_Count  = 20
```

Now investigate ALB-side behavior and connectivity more aggressively:

```text
listener/routing
target connectivity
timeouts
target resets
load balancer behavior
networking
configuration
```

Do not immediately blame Spring Boot.

---

# 20. Check Application Metrics

Our Spring Boot services expose Prometheus metrics.

Conceptually:

```text
HTTP server requests
HTTP status
latency
request rate
```

Look at:

```text
5xx rate
P95
P99
request rate
active requests
```

Example:

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
```

The exact metric names depend on the Micrometer/Spring Boot version and configuration.

---

# 21. Golden Signals

For the affected service:

```text
Traffic
Errors
Latency
Saturation
```

Example:

```text
Traffic   → 1,500 RPS
Errors    → 25%
Latency   → P99 8 seconds
CPU       → 90%
```

This tells us the incident is broader than an isolated ALB configuration issue.

---

# 22. Check Application Logs

```bash
kubectl logs deployment/order-service -n ecommerce
```

Or:

```bash
kubectl logs <pod-name> -n ecommerce
```

Search for:

```text
ERROR
Exception
Timeout
Connection refused
Connection timeout
SQL
Redis
Kafka
OutOfMemory
```

Because our logging architecture is centralized, also search Kibana using:

```text
service.name = order-service
environment = production
status >= 500
```

---

# 23. Use Correlation ID

Our application architecture includes:

```text
X-Correlation-ID
```

Example:

```text
Client
 |
 | X-Correlation-ID: 8c123...
 v
ALB
 |
 v
order-service
 |
 +--> customer-service
 +--> pricing-service
 +--> inventory-service
```

Search that ID in ELK.

This allows you to follow one failed request.

---

# 24. Use Jaeger

If the request reaches the application, find the trace.

Architecture:

```text
Request
  |
  v
ALB
  |
  v
order-service
  |
  +--> customer-service
  +--> pricing-service
  +--> inventory-service
  +--> RDS
  +--> Redis
```

A trace might show:

```text
order-service      40ms
customer-service   30ms
pricing-service    50ms
inventory-service  5.2s
RDS                5.0s
```

Now the 5xx is probably downstream-induced.

---

# 25. Check RDS

If order-service is returning:

```text
500
```

because PostgreSQL connections are exhausted:

```text
order-service
    |
    v
RDS
    |
    X
connection refused/timeout
```

Check:

```text
RDS connections
CPU
memory
storage
locks
slow queries
events
```

Also check Hikari metrics.

---

# 26. Check Redis

If Redis is failing:

```text
order-service
    |
    v
Redis
    |
    X
timeout
```

The application may return:

```text
500
```

or:

```text
503
```

depending on application error handling.

This demonstrates why:

```text
ALB 5xx
```

doesn't automatically mean:

```text
ALB problem
```

---

# 27. Check Kafka/SQS

Suppose an order API synchronously waits for a downstream operation.

If:

```text
Kafka
```

or:

```text
SQS-dependent processing
```

becomes unhealthy or delayed, the API may eventually time out.

Then:

```text
dependency latency
 ↓
application timeout
 ↓
HTTP 5xx
 ↓
ALB Target 5xx
```

---

# 28. Check External APIs

Our services may depend on:

```text
Payment provider
Tax provider
Shipping provider
Notification provider
Fraud provider
```

If payment provider latency increases:

```text
payment-service
     ↓
timeout
     ↓
order-service
     ↓
HTTP 500/502/504
     ↓
ALB target 5xx
```

Distributed tracing should make this visible.

---

# 29. ALB 502 Investigation

A 502 should trigger investigation around:

```text
ALB → target
```

Potential areas:

```text
Target reset connection
Invalid target response
Protocol mismatch
Application/container termination
Network connectivity
Target port
```

Check:

```bash
kubectl get svc order-service -n ecommerce
```

and:

```bash
kubectl describe svc order-service -n ecommerce
```

Also inspect the Ingress configuration.

---

# 30. ALB 503 Investigation

A 503 often requires checking target availability.

Start with:

```bash
kubectl get pods -n ecommerce
```

then:

```bash
kubectl get endpointslice -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

and AWS target health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --region ap-south-1
```

Potential causes:

```text
No healthy targets
Readiness failures
Pods unavailable
Incorrect target registration
Deployment outage
Target group configuration
```

---

# 31. ALB 504 Investigation

504 usually makes us investigate timeout behavior.

Potential chain:

```text
Client
  |
  v
ALB
  |
  v
order-service
  |
  v
RDS
  |
  X
slow query
```

or:

```text
order-service
  |
  v
payment-service
  |
  X
external API timeout
```

Check:

```text
ALB timeout configuration
application request timeout
HTTP client timeout
database query latency
downstream service latency
external API latency
```

---

# 32. Timeout Budget

A production system needs a consistent timeout strategy.

Example:

```text
Client timeout
     >
ALB timeout
     >
Application request timeout
     >
Downstream timeout
```

The exact numbers must be designed and tested.

Do not randomly set every timeout to:

```text
60 seconds
```

because this can cause huge numbers of concurrent requests waiting on slow dependencies.

---

# 33. Retry Storm

Suppose:

```text
RDS becomes slow
```

Application requests time out.

The application retries each request three times.

Then:

```text
1 request
 ↓
3 attempts
```

At scale:

```text
1,000 requests/s
 ↓
3,000 database attempts/s
```

The database becomes even slower.

Then:

```text
slow RDS
 ↓
timeouts
 ↓
retries
 ↓
more load
 ↓
more timeouts
 ↓
5xx
```

This is a retry storm.

---

# 34. ALB + HPA Interaction

Suppose ALB traffic increases:

```text
1,000 RPS
 ↓
CPU increases
 ↓
HPA scales
 ↓
Pods increase
```

This sounds good.

But if the bottleneck is RDS:

```text
1,000 RPS
 ↓
HPA
 ↓
10 → 30 Pods
 ↓
more DB connections
 ↓
RDS connection exhaustion
 ↓
5xx increases
```

Autoscaling can make the incident worse.

---

# 35. Check HPA

```bash
kubectl get hpa -n ecommerce
```

Then:

```bash
kubectl describe hpa order-service -n ecommerce
```

Check:

```text
current replicas
desired replicas
CPU
memory
custom metrics
maxReplicas
```

Ask:

> Did scaling help, or did scaling amplify downstream pressure?

---

# 36. Check Deployment

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

Then:

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

Check whether the 5xx spike started immediately after a deployment.

---

# 37. Deployment Correlation

Example:

```text
10:00
order-service 1.8.0

10:15
deploy 1.9.0

10:17
5xx rises from 0.2% → 18%

10:18
P99 rises

10:20
ALB target 5xx alert
```

This is strong evidence that the deployment may be related.

Still investigate logs/traces before declaring root cause.

---

# 38. GitOps Correlation

Check:

```text
Git commit
   ↓
GitOps change
   ↓
ArgoCD sync
   ↓
Deployment
   ↓
ALB 5xx
```

ArgoCD:

```bash
argocd app get order-service
```

Check:

```text
sync status
health
revision
recent operation
```

---

# 39. Example Root Cause — Bad Readiness Probe

Suppose version `2.2.0` changes:

```text
/actuator/health/readiness
```

and the endpoint now requires Redis.

Redis becomes temporarily unavailable.

Then:

```text
Redis failure
 ↓
readiness failure
 ↓
Pods removed from endpoints
 ↓
ALB has no healthy targets
 ↓
503
```

The root cause isn't necessarily:

```text
ALB
```

It may be:

```text
bad readiness dependency design
```

---

# 40. Example Root Cause — Application 500

Suppose:

```text
HTTPCode_Target_5XX_Count
```

is high.

Logs show:

```text
DataAccessException
connection pool exhausted
```

RDS metrics show:

```text
connections near max
```

Then:

```text
RDS connection exhaustion
 ↓
order-service exceptions
 ↓
HTTP 500
 ↓
ALB target 5xx
```

The ALB is functioning correctly.

---

# 41. Example Root Cause — No Healthy Targets

Suppose:

```text
ALB → 503
```

and:

```text
Target health:
all unhealthy
```

Kubernetes:

```text
Pods:
0 Ready
```

Then:

```text
ALB 503
```

is the symptom of application target unavailability.

Investigate why Pods aren't Ready.

---

# 42. Example Root Cause — Wrong Target Port

Suppose Service:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

but the application actually listens on:

```text
9090
```

Then:

```text
ALB
 ↓
target
 ↓
connection failure
```

This can cause target health failures and downstream 5xx behavior.

Check:

```bash
kubectl describe svc order-service -n ecommerce
```

and:

```bash
kubectl describe pod <pod> -n ecommerce
```

---

# 43. Check Ingress

Our ALB is generally managed by the AWS Load Balancer Controller.

Check:

```bash
kubectl get ingress -n ecommerce
```

Then:

```bash
kubectl describe ingress <ingress-name> -n ecommerce
```

Review:

```text
ALB address
listeners
rules
backend service
annotations
target type
health check configuration
```

---

# 44. AWS Load Balancer Controller

Architecture:

```text
Kubernetes Ingress
       |
       v
AWS Load Balancer Controller
       |
       v
AWS ALB
       |
       v
Target Group
       |
       v
Kubernetes workload
```

If an Ingress change causes unexpected behavior, inspect controller logs:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller
```

Use the appropriate deployment name/namespace for your installation.

---

# 45. Target Type

The AWS Load Balancer Controller can use different target models.

A common EKS setup uses:

```text
IP targets
```

which allows the ALB target group to point directly to Pod IPs.

Architecture:

```text
ALB
 |
 v
Target Group
 |
 +--> Pod IP
 +--> Pod IP
 +--> Pod IP
```

This makes EndpointSlice/readiness behavior especially important.

---

# 46. NetworkPolicy Consideration

Suppose:

```text
Pod
```

is healthy but network traffic is blocked by:

```text
NetworkPolicy
```

or another network/security control.

Then:

```text
ALB
 ↓
target
 X
Pod
```

Investigate:

```text
NetworkPolicy
security groups
pod networking
service routing
```

Don't assume application code is responsible.

---

# 47. Security Groups

For AWS networking, inspect the relevant security groups.

Architecture:

```text
Internet
  ↓
ALB SG
  ↓
EKS workload/node networking
  ↓
Pod
```

The exact path depends on the target mode and EKS networking architecture.

Security group rules must permit the required traffic.

Avoid:

```text
0.0.0.0/0
```

as a blanket internal access solution.

---

# 48. Check ALB Access Logs

If enabled, ALB access logs are valuable.

They can help determine:

```text
request
client
path
status
target status
latency
```

This allows correlation between:

```text
ALB
```

and:

```text
application
```

Access logs should be stored and retained according to the platform's logging policy.

---

# 49. Correlating ALB and Application Logs

A useful investigation chain:

```text
ALB access log
      ↓
timestamp
      ↓
request path
      ↓
status
      ↓
correlation ID
      ↓
application log
      ↓
trace ID
      ↓
Jaeger
      ↓
dependency
```

This turns:

```text
"Checkout returns 500"
```

into:

```text
"Checkout 500 because inventory-service waited 5s for
RDS connection and order-service timeout was exceeded."
```

---

# 50. Customer Impact Classification

Not every ALB 5xx affects the entire platform.

Example:

```text
catalog-service → healthy
search-service  → healthy
order-service   → 5xx
payment-service → healthy
```

Customer impact:

```text
Browsing works
Search works
Checkout fails
```

This is a **partial outage**.

Incident communication should say exactly that.

---

# 51. Immediate Mitigation Options

Depending on root cause:

### Bad deployment

Rollback.

### Insufficient Pods

Scale appropriately.

### Node failure

Replace/rebalance nodes.

### Readiness problem

Correct configuration/probe.

### Database saturation

Protect database and reduce request pressure.

### External dependency failure

Circuit breaker/degraded mode.

### Bad ALB configuration

Restore known-good Ingress configuration.

### Network issue

Restore required connectivity.

---

# 52. Do Not Automatically Scale to Maximum

Suppose:

```text
maxReplicas = 100
```

and:

```text
RDS is saturated
```

Scaling:

```text
10 Pods → 100 Pods
```

may create:

```text
more connections
more queries
more retries
more CPU
more memory
more 5xx
```

Scale based on the bottleneck.

---

# 53. Controlled Rollback

If the incident began immediately after deployment:

```text
GitOps
   |
   +--> revert commit
   |
   v
ArgoCD
   |
   v
known-good deployment
```

Monitor:

```text
5xx
latency
target health
Pod readiness
business transactions
```

---

# 54. Validate ALB Target Health

After mitigation:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --region ap-south-1
```

You want:

```text
healthy
healthy
healthy
```

rather than:

```text
unhealthy
unhealthy
```

---

# 55. Validate Kubernetes

```bash
kubectl get pods -n ecommerce
```

Then:

```bash
kubectl get endpointslice -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Verify:

```text
Ready Pods
+
Ready endpoints
```

---

# 56. Validate HTTP

Test through the real application path.

For example:

```text
GET /api/v1/products
POST /api/v1/orders
```

Check:

```text
2xx
latency
error rate
```

Do not consider:

```text
curl localhost
```

alone sufficient.

You need to validate:

```text
Client
 ↓
CloudFront/WAF
 ↓
ALB
 ↓
EKS
 ↓
Service
 ↓
Pod
```

where appropriate.

---

# 57. Validate Business Flow

For checkout:

```text
Login
 ↓
Catalog
 ↓
Cart
 ↓
Order
 ↓
Inventory
 ↓
Fraud
 ↓
Payment
 ↓
Invoice
 ↓
Notification
```

Check that the complete workflow works.

---

# 58. Observability Dashboard

Our ALB dashboard should contain:

```text
Request count
HTTPCode_ELB_5XX_Count
HTTPCode_Target_5XX_Count
HTTPCode_ELB_4XX_Count
Target response time
Healthy target count
Unhealthy target count
Rejected connections
Active connections
Processed bytes
```

Application dashboard:

```text
Application 5xx
P50
P95
P99
RPS
CPU
Memory
Pod restarts
HPA replicas
RDS connections
Redis health
Kafka lag
SQS backlog
```

---

# 59. The Most Important Comparison

During an ALB incident, compare:

```text
ALB 5xx
        vs
Application 5xx
```

### Case A

```text
ALB 5xx      low
Target 5xx   high
App 5xx      high
```

Likely:

```text
Application/dependency issue
```

### Case B

```text
ELB 5xx      high
Target 5xx   low
App 5xx      low
```

Investigate:

```text
ALB/network/target connection path
```

### Case C

```text
503 high
Healthy targets = 0
```

Investigate:

```text
Pods
readiness
EndpointSlices
target health
deployment
```

---

# 60. Production Incident Timeline

Example:

```text
10:00
order-service 2.3.0 deployed

10:03
P95 increases

10:05
Target 5xx rises

10:06
ALB 5xx alert fires

10:07
Incident declared

10:09
Target health checked

10:11
Pods healthy

10:13
Application logs show DB timeout

10:15
RDS connections near maximum

10:18
HPA reduced/guarded to prevent further DB pressure

10:20
Deployment rolled back

10:25
5xx falls to baseline

10:30
Business transaction validated

11:00
RCA started
```

---

# 61. Root Cause Example

Suppose version `2.3.0` changed:

```text
order creation
```

from:

```text
1 database query
```

to:

```text
12 database queries
```

At normal traffic:

```text
works
```

At peak traffic:

```text
RPS increases
 ↓
DB queries increase
 ↓
RDS connection pool saturates
 ↓
request latency increases
 ↓
timeouts
 ↓
HTTP 500
 ↓
ALB Target 5xx
```

The ALB is not the root cause.

The root cause is:

> **Application/database query amplification introduced by deployment 2.3.0.**

---

# 62. Another Root Cause — Health Check Misconfiguration

Deployment changes:

```text
health endpoint:
```

from:

```text
/actuator/health/readiness
```

to:

```text
/health
```

but ALB target group still checks:

```text
/actuator/health
```

The target becomes unhealthy.

Result:

```text
Healthy targets = 0
        ↓
ALB 503
```

This is a configuration contract failure between:

```text
ALB
+
Kubernetes
+
application
```

---

# 63. Another Root Cause — Slow Application

Suppose:

```text
ALB timeout = 30 seconds
```

Application:

```text
request takes 45 seconds
```

Then:

```text
Request
 ↓
ALB waits
 ↓
30 sec
 ↓
timeout
 ↓
504
```

Fixing the timeout blindly to:

```text
120 seconds
```

may hide the performance problem and increase resource pressure.

Investigate:

```text
Why does the request take 45 seconds?
```

---

# 64. Another Root Cause — Retry Amplification

Example:

```text
Payment API latency
       ↓
payment-service retries
       ↓
order-service retries
       ↓
client retries
```

One user action may generate:

```text
1 → 3 → 9 requests
```

This can create a large spike.

The solution involves:

```text
timeout
backoff
jitter
idempotency
circuit breaker
retry budget
```

not simply increasing ALB capacity.

---

# 65. ALB + Circuit Breaker

For critical external dependencies:

```text
order-service
      |
      v
payment-service
      |
      v
Payment Provider
```

If the payment provider fails:

```text
circuit opens
```

rather than allowing every request to wait and fail.

This protects:

```text
order-service
ALB
EKS
RDS
```

from cascading failure.

---

# 66. GitOps Principle

If the ALB configuration is managed through Kubernetes Ingress:

```text
Git
 ↓
Helm
 ↓
GitOps
 ↓
ArgoCD
 ↓
Ingress
 ↓
AWS Load Balancer Controller
 ↓
ALB
```

Do not manually change the ALB and forget to update Git.

Otherwise:

```text
AWS manual change
       ↓
works temporarily
       ↓
ArgoCD/controller reconciliation
       ↓
configuration changes again
```

Always restore the desired state through the proper source of truth.

---

# 67. Incident Severity

Example classification:

### SEV-1

```text
Entire checkout unavailable
Large customer impact
```

### SEV-2

```text
Major service partially unavailable
```

### SEV-3

```text
Limited functionality degradation
```

### SEV-4

```text
Minor issue/no significant customer impact
```

Actual severity should follow the organization's incident policy.

---

# 68. Communication During Incident

Example:

```text
[SEV-2] Checkout API degradation

Impact:
Customers are experiencing failures while placing orders.

Current symptoms:
Order API HTTP 5xx increased to 18%.

Investigation:
ALB target 5xx is elevated.
Targets are healthy.
RDS connection utilization is high.

Mitigation:
Traffic pressure reduced and known-good application version restored.

Status:
Error rate is returning to baseline.
Business validation in progress.
```

Avoid:

> "ALB is broken."

until evidence supports that conclusion.

---

# 69. RCA Template

```markdown
# ALB 5xx Incident RCA

## Incident

ALB 5xx / Order API degradation

## Severity

SEV-2

## Customer Impact

Describe affected functionality.

## Detection

Prometheus/Grafana/ALB alert.

## Timeline

- Deployment
- 5xx increase
- Incident declaration
- Investigation
- Mitigation
- Recovery

## Metrics

### ALB

- ELB 5xx
- Target 5xx
- Healthy targets
- Target response time

### Application

- RPS
- 5xx
- P95
- P99

### Dependencies

- RDS
- Redis
- Kafka/SQS
- External APIs

## Root Cause

Actual technical cause.

## Contributing Factors

List contributing issues.

## Mitigation

What restored service?

## Permanent Fix

What prevents recurrence?

## RTO

Measured recovery time.

## Customer Impact

Quantified if possible.

## Corrective Actions

- [ ] Fix application
- [ ] Improve alert
- [ ] Improve dashboard
- [ ] Add test
- [ ] Update runbook
- [ ] Add dependency protection
```

---

# 70. Production Prevention Checklist

### ALB

- [ ] Target health monitored
- [ ] ELB 5xx monitored
- [ ] Target 5xx monitored
- [ ] Target response time monitored
- [ ] Access logs enabled where required
- [ ] Health checks correctly configured
- [ ] Timeout values reviewed

### Kubernetes

- [ ] Readiness probes configured
- [ ] Liveness probes separated from readiness
- [ ] EndpointSlices monitored
- [ ] HPA configured
- [ ] PDB configured
- [ ] Topology spread configured
- [ ] Deployment rollout strategy tested

### Application

- [ ] HTTP 5xx metrics
- [ ] Structured logging
- [ ] Correlation IDs
- [ ] Distributed tracing
- [ ] Timeout policies
- [ ] Retry policies
- [ ] Circuit breakers where appropriate
- [ ] Idempotency for critical operations

### Dependencies

- [ ] RDS connections monitored
- [ ] Redis monitored
- [ ] Kafka lag monitored
- [ ] SQS backlog monitored
- [ ] External API latency monitored

### GitOps

- [ ] Deployment history available
- [ ] ArgoCD health monitored
- [ ] Rollback tested
- [ ] Ingress changes reviewed
- [ ] No unmanaged production drift

---

# 71. Production Troubleshooting Commands

### Pods

```bash
kubectl get pods -n ecommerce -o wide
```

```bash
kubectl describe pod <pod> -n ecommerce
```

```bash
kubectl logs <pod> -n ecommerce
```

---

### Deployment

```bash
kubectl rollout status deployment/order-service -n ecommerce
```

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

---

### Service

```bash
kubectl get svc order-service -n ecommerce
```

```bash
kubectl describe svc order-service -n ecommerce
```

---

### EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

---

### Ingress

```bash
kubectl get ingress -n ecommerce
```

```bash
kubectl describe ingress <ingress-name> -n ecommerce
```

---

### HPA

```bash
kubectl get hpa -n ecommerce
```

```bash
kubectl describe hpa order-service -n ecommerce
```

---

### AWS Target Health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --region ap-south-1
```

---

### ArgoCD

```bash
argocd app get order-service
```

---

# 72. Final Decision Tree

```text
ALB 5xx
   |
   v
Is ELB 5xx high or Target 5xx high?
   |
   +-------------------------+
   |                         |
   v                         v
ELB 5xx                  Target 5xx
   |                         |
   v                         v
ALB/target path          Application
network/config           /dependency
   |                         |
   |                         v
   |                    Check Pods
   |                         |
   |                    Check readiness
   |                         |
   |                    Check EndpointSlices
   |                         |
   |                    Check application 5xx
   |                         |
   |                    Check logs
   |                         |
   |                    Check traces
   |                         |
   |                    Check RDS/Redis/Kafka/etc.
   |
   +------------+
                |
                v
          Recent deployment?
                |
          +-----+-----+
          |           |
         Yes          No
          |           |
          v           v
       Rollback?   Continue
          |
          v
      Validate
          |
          v
  Business transaction
          |
          v
         RCA
```

---

# 73. Final Production Mental Model

When you hear:

> **"ALB is returning 5xx."**

Think:

```text
Don't blame ALB immediately.
```

Instead:

```text
ALB 5xx
   |
   +--> ELB-generated?
   |
   +--> Target-generated?
           |
           +--> Pod healthy?
           |
           +--> Ready?
           |
           +--> Endpoint exists?
           |
           +--> Application returning 5xx?
           |
           +--> Dependency failing?
           |
           +--> Recent deployment?
```

Then correlate:

```text
ALB
 ↓
Kubernetes
 ↓
Application
 ↓
Logs
 ↓
Traces
 ↓
Dependencies
```

---

# 74. The Complete Incident Chain

```text
Customer
   |
   v
CloudFront / WAF
   |
   v
ALB
   |
   +--> ELB 5xx?
   |
   v
Target Group
   |
   +--> Healthy?
   |
   v
Kubernetes Service
   |
   +--> EndpointSlice?
   |
   v
Pod
   |
   +--> Readiness?
   |
   v
Application
   |
   +--> 5xx?
   |
   v
Dependencies
   |
   +--> RDS
   +--> Redis
   +--> Kafka
   +--> SQS
   +--> External API
```

This is the troubleshooting path you should mentally follow during a real production incident.

---

# 75. Key Rules to Remember

1. **ALB 5xx does not automatically mean ALB is the root cause.**
2. Separate `HTTPCode_ELB_5XX_Count` from `HTTPCode_Target_5XX_Count`.
3. Check ALB target health.
4. Check Kubernetes Pods.
5. Check Pod readiness.
6. Check EndpointSlices.
7. Check application-level 5xx.
8. Check logs and traces.
9. Check RDS/Redis/messaging/external dependencies.
10. Correlate incidents with recent deployments.
11. Do not blindly scale when a database is the bottleneck.
12. Do not blindly restart all Pods.
13. Use GitOps for deployment/configuration rollback.
14. Validate the complete customer request path after recovery.
15. Measure customer impact.
16. Record the actual root cause, not merely "ALB 5xx."
17. Protect against retry storms and cascading failures.
18. Keep ALB, Kubernetes, application, and dependency dashboards connected.
19. Test rollback and health-check behavior before production incidents.
20. The correct mental model is:

```text
ALB 5xx
   ↓
Where was the response generated?
   ↓
ALB or Target?
   ↓
If Target:
    Pod → Application → Dependency
   ↓
Root Cause
   ↓
Mitigate
   ↓
Validate
   ↓
RCA
```

---

# 76. Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ✅
147 — Disk Full                    ✅
148 — ImagePullBackOff             ✅
149 — ALB 5xx                      ✅
150 — Service Connectivity Failure ⏭️
151 — DNS Failure                  ⏭️
152 — RDS Connection Exhaustion    ⏭️
153 — Redis Failure                ⏭️
154 — SQS Backlog                  ⏭️
155 — Kafka Consumer Lag            ⏭️
156 — Node Failure                 ⏭️
157 — AZ Failure                   ⏭️
158 — ArgoCD Drift                 ⏭️
159 — Bad Production Deployment    ⏭️
160 — Security Incident            ⏭️
```

**Next: `150 — Service Connectivity Failure Production Incident`**.