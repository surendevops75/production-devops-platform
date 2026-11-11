# 30-Prometheus.md

````markdown
# 30 — Prometheus

## 1. Overview

Prometheus is the primary metrics collection and time-series monitoring system for our Kubernetes platform.

Our architecture is:

```text
                    EKS
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
Microservices     Nodes         Kubernetes
       |             |             |
       v             v             v
 Micrometer     Node Exporter  kube-state-metrics
       |             |             |
       +-------------+-------------+
                     |
                     v
                Prometheus
                     |
                     v
                  Grafana
                     |
                     v
               Alertmanager
````

Prometheus answers questions such as:

```text
How much traffic is the application receiving?
How many requests are failing?
What is the request latency?
How much CPU is being consumed?
How much memory is being consumed?
How many Pods are running?
How many Pods are pending?
How many replicas are unavailable?
How many database connections exist?
Is the HPA scaling?
Is the application returning errors?
```

Prometheus is therefore one of the core components of our observability platform.

---

# 2. Monitoring vs Observability

Monitoring generally answers:

```text
Is something wrong?
```

Observability helps answer:

```text
Why is it wrong?
```

For example:

```text
Prometheus:
HTTP 500 rate = 8%
```

This tells us something is wrong.

Then:

```text
Grafana:
Error rate increased at 10:32
```

Then:

```text
Logs:
Database connection timeout
```

Then:

```text
Jaeger:
order-service -> payment-service -> payment DB
```

Together:

```text
Metrics
+
Logs
+
Traces
=
Observability
```

Prometheus provides the metrics foundation.

---

# 3. Why Prometheus?

Kubernetes environments generate large amounts of operational data.

We need to monitor:

```text
Applications
Pods
Deployments
Nodes
Namespaces
Services
Ingress
HPA
ArgoCD
Kubernetes controllers
```

Prometheus provides:

```text
Metrics collection
Time-series storage
PromQL
Alerting rules
Service discovery
Metric relabeling
Recording rules
```

---

# 4. Prometheus in Our Project

Our Enterprise E-Commerce Platform contains:

```text
21 microservices
```

Each service can expose application metrics.

Example:

```text
order-service
payment-service
catalog-service
inventory-service
shipping-service
...
```

The architecture is:

```text
Spring Boot
    |
    v
Micrometer
    |
    v
/actuator/prometheus
    |
    v
Prometheus
```

Prometheus scrapes the metrics endpoint.

---

# 5. Prometheus Pull Model

Prometheus primarily uses a pull model.

```text
Prometheus
    |
    | HTTP GET /metrics
    v
Application
```

For example:

```text
Prometheus -> order-service:8080/actuator/prometheus
```

The application returns:

```text
http_server_requests_seconds_count{...} 12345
```

Prometheus stores the resulting time series.

---

# 6. Pull vs Push

Traditional monitoring systems may push metrics:

```text
Application
    |
    v
Monitoring system
```

Prometheus normally pulls:

```text
Prometheus
    |
    v
Application
```

Advantages include:

```text
Service discovery
Centralized scrape control
Easy health detection
Clear scrape intervals
```

Prometheus also has mechanisms for special cases where direct scraping is unsuitable.

---

# 7. Prometheus Architecture

A typical Prometheus installation contains:

```text
Prometheus Server
    |
    +--> Service Discovery
    |
    +--> Scraping
    |
    +--> TSDB
    |
    +--> PromQL
    |
    +--> Recording Rules
    |
    +--> Alerting Rules
    |
    v
Alertmanager
```

Prometheus itself is not the complete visualization stack.

Typically:

```text
Prometheus -> Grafana
Prometheus -> Alertmanager
```

---

# 8. Prometheus Components

Important concepts:

```text
Prometheus Server
Prometheus TSDB
Service Discovery
Scrape Configuration
Exporters
PromQL
Recording Rules
Alerting Rules
Alertmanager
```

In Kubernetes, additional components commonly provide metrics:

```text
kube-state-metrics
Node Exporter
Application exporters
```

---

# 9. Prometheus in EKS

A production architecture could look like:

```text
AWS
 |
 v
EKS
 |
 +-------------------------------+
 |                               |
 v                               v
Application Pods              Kubernetes Nodes
 |                               |
 v                               v
Micrometer                  Node Exporter
 |
 v
Application Metrics
 |
 +---------------+---------------+
                 |
                 v
             Prometheus
                 |
        +--------+--------+
        |                 |
        v                 v
     Grafana         Alertmanager
```

---

# 10. Where Prometheus Runs

Prometheus normally runs inside the Kubernetes monitoring/platform layer.

Example:

```text
EKS
│
├── argocd
│
├── production
│
├── monitoring
│   ├── prometheus
│   ├── grafana
│   └── alertmanager
│
├── logging
│
└── tracing
```

Prometheus should not be deployed separately inside every application namespace.

Use a centralized monitoring architecture.

---

# 11. Production Installation

A common Kubernetes approach is the Prometheus Operator ecosystem.

A widely used deployment bundle is:

```text
kube-prometheus-stack
```

It can provide:

```text
Prometheus
Alertmanager
Grafana
Prometheus Operator
Node Exporter
kube-state-metrics
```

The exact components and versions depend on the selected chart/version.

For production:

```text
Pin chart version
Review release notes
Test upgrade
Manage through GitOps
```

Do not blindly install the latest chart directly into production.

---

# 12. Helm Installation Model

For learning:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

Example installation:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace
```

For production, the preferred architecture is:

```text
GitOps
   |
   v
Helm chart version
   |
   v
ArgoCD
   |
   v
EKS
```

rather than:

```text
Engineer laptop
   |
   v
helm install
   |
   v
Production
```

---

# 13. Prometheus Operator

Prometheus Operator simplifies Kubernetes-native Prometheus management.

It introduces Kubernetes custom resources such as:

```text
Prometheus
ServiceMonitor
PodMonitor
PrometheusRule
Alertmanager
```

Instead of manually maintaining every scrape configuration, we can declare monitoring intent using Kubernetes resources.

---

# 14. ServiceMonitor

A `ServiceMonitor` tells Prometheus how to scrape a Kubernetes Service.

Conceptually:

```text
Service
   |
   v
ServiceMonitor
   |
   v
Prometheus
```

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: order-service
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: order-service

  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
```

The exact namespace and selector behavior depends on Prometheus Operator configuration.

---

# 15. PodMonitor

A `PodMonitor` can scrape Pods directly.

Conceptually:

```text
Pod
 |
 v
PodMonitor
 |
 v
Prometheus
```

Use `ServiceMonitor` when the application is naturally represented by a Kubernetes Service.

Use `PodMonitor` when direct Pod discovery is more appropriate.

Do not create both unnecessarily.

---

# 16. ServiceMonitor Labels

Prometheus Operator needs to discover the `ServiceMonitor`.

Depending on the Prometheus configuration, a label selector may be required.

Example:

```yaml
metadata:
  labels:
    release: monitoring
```

The Prometheus resource may be configured to select:

```text
release=monitoring
```

Therefore troubleshooting must include:

```text
ServiceMonitor exists
+
Prometheus selects ServiceMonitor
+
Service selector matches
+
endpoint exists
```

---

# 17. Spring Boot Metrics

Our services use:

```text
Micrometer
```

and:

```text
micrometer-registry-prometheus
```

Example Maven dependency:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Spring Boot Actuator exposes:

```text
/actuator/prometheus
```

---

# 18. Spring Boot Configuration

Example:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,info

  endpoint:
    health:
      probes:
        enabled: true
```

Then:

```text
http://order-service:8080/actuator/prometheus
```

returns Prometheus-compatible metrics.

---

# 19. Example Metrics

You may see:

```text
jvm_memory_used_bytes
jvm_threads_live_threads
process_cpu_usage
http_server_requests_seconds_count
http_server_requests_seconds_sum
http_server_requests_seconds_max
```

The exact metric names depend on Spring Boot and Micrometer versions/configuration.

---

# 20. Business Metrics

Infrastructure metrics are not enough.

We should expose business-relevant metrics.

For example:

```text
orders_created_total
orders_failed_total
payments_success_total
payments_failed_total
cart_items_added_total
checkout_success_total
```

Then Prometheus can answer:

```text
How many orders are being created?
```

not just:

```text
How much CPU is being used?
```

---

# 21. Counter

A counter represents a value that generally increases over time.

Examples:

```text
requests_total
orders_created_total
payment_failures_total
```

Conceptually:

```text
0
|
+--> 1
|
+--> 2
|
+--> 3
|
+--> 4
```

A counter may reset when the application restarts.

PromQL functions such as:

```text
rate()
increase()
```

are commonly used with counters.

---

# 22. Gauge

A gauge can increase or decrease.

Examples:

```text
current_active_orders
queue_depth
memory_usage
active_connections
```

Conceptually:

```text
100
 |
 v
80
 |
 v
120
 |
 v
70
```

Gauges represent current state.

---

# 23. Histogram

Histograms are extremely important for latency monitoring.

A request latency distribution might be:

```text
<50ms
<100ms
<250ms
<500ms
<1s
<2s
```

Prometheus can store histogram bucket counts.

This allows us to calculate latency quantiles using PromQL.

Example:

```text
histogram_quantile()
```

is commonly used to estimate percentiles from histogram buckets.

---

# 24. Why Histograms Matter

Average latency can hide serious problems.

Example:

```text
99 requests = 50ms
1 request  = 10 seconds
```

Average:

```text
149.5ms
```

might look acceptable.

But:

```text
P99
```

could reveal a serious latency issue.

Therefore production monitoring should track:

```text
P50
P90
P95
P99
```

where useful.

---

# 25. Summary

A Summary metric also provides statistical information.

However, unlike histograms, summary quantiles are calculated on the client side.

This creates limitations for aggregating quantiles across multiple instances.

For Kubernetes applications with many replicas:

```text
Histogram
```

is often preferred for aggregatable latency analysis.

---

# 26. Labels

Prometheus metrics use labels.

Example:

```text
http_requests_total{
  service="order-service",
  method="GET",
  status="200"
}
```

Labels allow filtering.

Example:

```text
service="order-service"
```

or:

```text
status="500"
```

Labels are extremely powerful.

They are also one of the biggest causes of Prometheus scalability problems when used incorrectly.

---

# 27. Cardinality

Cardinality means the number of unique label combinations.

Good:

```text
service
method
status
```

Dangerous:

```text
user_id
request_id
session_id
transaction_id
email
```

Example:

```text
http_requests_total{user_id="12345"}
```

If millions of users exist:

```text
Millions of time series
```

can be created.

This can consume large amounts of:

```text
CPU
Memory
Disk
```

---

# 28. High-Cardinality Example

Bad:

```text
request_duration_seconds{
  customer_id="123456",
  request_id="a8f7..."
}
```

Every request may generate a new label combination.

Better:

```text
request_duration_seconds{
  service="order-service",
  route="/api/v1/orders",
  method="POST",
  status="201"
}
```

Use bounded dimensions.

---

# 29. Metric Naming

Prefer consistent names.

Examples:

```text
orders_created_total
orders_failed_total
payment_requests_total
```

Use units where appropriate:

```text
request_duration_seconds
memory_usage_bytes
```

Avoid ambiguous names such as:

```text
time
count
value
```

Good metric naming makes PromQL easier to understand.

---

# 30. RED Method

For request-driven services, monitor:

```text
R = Rate
E = Errors
D = Duration
```

Example:

```text
Rate:
requests/sec

Errors:
5xx/sec

Duration:
P95 latency
```

For `order-service`:

```text
order requests/sec
order 5xx rate
order request latency
```

This gives a strong high-level service view.

---

# 31. USE Method

For infrastructure resources:

```text
U = Utilization
S = Saturation
E = Errors
```

For nodes:

```text
CPU utilization
memory utilization
CPU throttling
disk pressure
network errors
```

RED is often useful for services.

USE is often useful for infrastructure.

---

# 32. Four Golden Signals

Another common framework:

```text
Latency
Traffic
Errors
Saturation
```

For `order-service`:

```text
Latency:
P95/P99

Traffic:
requests/sec

Errors:
5xx %

Saturation:
CPU/memory/thread/connection pressure
```

These should become core Grafana dashboards.

---

# 33. PromQL

PromQL is Prometheus Query Language.

Example:

```promql
up
```

returns the health of scrape targets.

Example:

```promql
up{job="order-service"}
```

filters by job.

---

# 34. Request Rate

Example:

```promql
rate(http_server_requests_seconds_count[5m])
```

This calculates the per-second rate over the last five minutes.

For production dashboards, filter by:

```text
service
route
method
status
```

as appropriate.

---

# 35. Error Rate

Example:

```promql
sum(
  rate(http_server_requests_seconds_count{
    status=~"5.."
  }[5m])
)
```

This gives the rate of HTTP 5xx responses.

For service-specific monitoring:

```promql
sum(
  rate(http_server_requests_seconds_count{
    service="order-service",
    status=~"5.."
  }[5m])
)
```

Metric labels vary by application instrumentation.

---

# 36. Error Percentage

A useful expression:

```promql
100 *
sum(rate(http_server_requests_seconds_count{
  service="order-service",
  status=~"5.."
}[5m]))
/
sum(rate(http_server_requests_seconds_count{
  service="order-service"
}[5m]))
```

This gives:

```text
5xx percentage
```

subject to the exact metric labels exposed by the application.

---

# 37. Request Latency

For histogram metrics:

```promql
histogram_quantile(
  0.95,
  sum(
    rate(http_server_requests_seconds_bucket{
      service="order-service"
    }[5m])
  ) by (le)
)
```

This calculates an approximate P95 latency.

The exact query must match the metric's label structure.

---

# 38. CPU Usage

For containers:

```promql
sum(
  rate(container_cpu_usage_seconds_total[5m])
) by (pod)
```

This helps identify CPU-heavy Pods.

For production, filter out irrelevant containers/series such as infrastructure artifacts where appropriate.

---

# 39. Memory Usage

Example:

```promql
container_memory_working_set_bytes
```

This shows container working-set memory.

Filter by namespace:

```promql
container_memory_working_set_bytes{
  namespace="production"
}
```

Then aggregate by Pod or workload.

---

# 40. CPU Requests

You can compare usage against requested CPU.

Conceptually:

```promql
CPU usage
/
CPU request
```

This helps determine whether:

```text
requests are too high
```

or:

```text
requests are too low
```

which affects both scheduling and cost.

---

# 41. Pod Availability

kube-state-metrics exposes Kubernetes object state.

For example:

```promql
kube_deployment_status_replicas_available
```

and:

```promql
kube_deployment_spec_replicas
```

can be compared.

Conceptually:

```text
Desired replicas = 3
Available replicas = 2
```

This indicates reduced availability.

---

# 42. kube-state-metrics

`kube-state-metrics` exposes Kubernetes object state as metrics.

Examples:

```text
Deployments
Pods
DaemonSets
StatefulSets
Jobs
CronJobs
Services
Nodes
Namespaces
```

Important distinction:

```text
kube-state-metrics
=
Kubernetes object state

Node Exporter
=
Operating system/node metrics
```

They complement each other.

---

# 43. Node Exporter

Node Exporter exposes host-level metrics.

Examples:

```text
CPU
Memory
Filesystem
Network
Load
Disk
```

Architecture:

```text
EC2 Node
   |
   v
Node Exporter
   |
   v
Prometheus
```

In Kubernetes, Node Exporter is commonly deployed as a DaemonSet so that each node has an exporter.

---

# 44. Node Monitoring

Important node metrics:

```text
CPU utilization
Memory utilization
Filesystem usage
Disk I/O
Network traffic
Network errors
Load
File descriptors
```

For EKS, also monitor:

```text
Node readiness
Pod capacity
IP capacity
CPU capacity
Memory capacity
```

Especially important with VPC CNI.

---

# 45. EKS VPC CNI Monitoring

AWS VPC CNI allocates IP addresses to Pods.

Potential problem:

```text
Node has CPU available
but no Pod IP available
```

Result:

```text
Pod Pending
```

Therefore monitoring should include:

```text
Pod IP availability
ENI/IP allocation
Subnet IP exhaustion
CNI errors
```

This is an example of why infrastructure-specific monitoring matters.

---

# 46. Kubernetes API Metrics

The Kubernetes control plane exposes metrics.

Depending on EKS configuration and accessible endpoints, important areas include:

```text
API request latency
API request rate
API errors
Controller behavior
Scheduler behavior
```

AWS-managed control-plane visibility and metrics access should be designed according to the EKS monitoring options in use.

Do not assume every control-plane metric is directly available from inside the worker nodes.

---

# 47. ArgoCD Metrics

ArgoCD exposes metrics for monitoring its own operation.

Useful areas include:

```text
Application sync status
Application health
Reconciliation
Controller performance
Repository operations
```

This lets us monitor the GitOps control plane.

Example architecture:

```text
ArgoCD
   |
   v
Prometheus
   |
   v
Grafana
```

---

# 48. Prometheus Targets

Prometheus maintains scrape targets.

Conceptually:

```text
Prometheus
 |
 +--> order-service
 +--> payment-service
 +--> catalog-service
 +--> node-exporter
 +--> kube-state-metrics
 +--> ArgoCD
```

A target can be:

```text
UP
```

or:

```text
DOWN
```

A `DOWN` target does not necessarily mean the application itself is down.

It may mean:

```text
metrics endpoint unavailable
DNS problem
network policy
wrong port
wrong path
TLS problem
```

---

# 49. Target Troubleshooting

If a target is down:

```text
1. Is ServiceMonitor selected?
2. Does Service exist?
3. Does Service selector match Pods?
4. Is endpoint port correct?
5. Is metrics path correct?
6. Is Pod listening on the port?
7. Is NetworkPolicy blocking traffic?
8. Is TLS configured correctly?
9. Can Prometheus reach the target?
```

This is the standard scrape troubleshooting chain.

---

# 50. ServiceMonitor Troubleshooting

Suppose:

```text
order-service
```

is not visible in Prometheus.

Check:

```bash
kubectl get servicemonitor -A
```

Then:

```bash
kubectl describe servicemonitor order-service -n monitoring
```

Check the Prometheus configuration and selectors.

Then:

```bash
kubectl get svc -n production
```

Then:

```bash
kubectl get endpoints -n production
```

or the appropriate EndpointSlice resources.

If the Service has no endpoints:

```text
Service selector
       |
       X
Pods
```

is likely wrong.

---

# 51. Metrics Endpoint Testing

From a controlled environment:

```bash
curl http://<service>:8080/actuator/prometheus
```

You should see:

```text
# HELP ...
# TYPE ...
metric_name ...
```

If the endpoint fails:

```text
Prometheus cannot scrape it
```

Fix the application or Service first.

---

# 52. Prometheus Scrape Interval

Example:

```yaml
interval: 30s
```

This means Prometheus attempts to scrape the target approximately every 30 seconds.

Short intervals:

```text
10s
15s
```

provide more frequent observations but increase:

```text
scrape load
storage
network traffic
Prometheus resource usage
```

Long intervals:

```text
60s
```

reduce cost but can miss short-lived changes.

Choose according to monitoring requirements.

---

# 53. Scrape Timeout

A scrape timeout controls how long Prometheus waits.

Conceptually:

```text
scrape interval = 30s
scrape timeout = 10s
```

If the application takes too long:

```text
timeout
```

Prometheus records a scrape failure.

The timeout should normally be shorter than the scrape interval.

---

# 54. Metric Relabeling

Prometheus supports relabeling.

It can:

```text
modify labels
drop labels
keep labels
drop metrics
rewrite target metadata
```

This is useful for controlling:

```text
cardinality
naming
organization
target selection
```

Use relabeling carefully.

An incorrect rule can silently remove important metrics.

---

# 55. Metric Relabeling for Cardinality

Suppose an exporter produces unnecessary high-cardinality metrics.

A relabeling rule can drop them.

Conceptually:

```text
Exporter
   |
   v
Metric relabeling
   |
   +--> useful metrics
   |
   X--> unwanted metrics
```

This can reduce Prometheus resource consumption.

But always validate what is being dropped.

---

# 56. Recording Rules

PromQL queries can become expensive.

Example:

```text
complex P95 query
+
21 services
+
many routes
+
many labels
```

Running the query repeatedly from Grafana can consume significant Prometheus resources.

Recording rules precompute frequently used queries.

Conceptually:

```text
Raw metrics
    |
    v
Recording rule
    |
    v
Precomputed time series
    |
    v
Grafana
```

---

# 57. Recording Rule Example

Conceptually:

```yaml
groups:
  - name: ecommerce-recording-rules

    rules:
      - record: service:http_requests_per_second
        expr: |
          sum(
            rate(http_server_requests_seconds_count[5m])
          ) by (service)
```

Then Grafana can query:

```promql
service:http_requests_per_second
```

instead of recalculating the expensive expression every time.

---

# 58. Alerting Rules

Prometheus can evaluate alerting expressions.

Example:

```yaml
groups:
  - name: order-service

    rules:
      - alert: OrderServiceHighErrorRate

        expr: |
          (
            sum(rate(http_server_requests_seconds_count{
              service="order-service",
              status=~"5.."
            }[5m]))
            /
            sum(rate(http_server_requests_seconds_count{
              service="order-service"
            }[5m]))
          ) > 0.05

        for: 10m

        labels:
          severity: critical

        annotations:
          summary: Order service error rate is high
```

The exact metric names and labels depend on the application's instrumentation.

---

# 59. `for`

The:

```yaml
for: 10m
```

means the alert condition must remain true for approximately that duration before firing.

This helps reduce noise.

Without a duration:

```text
Short spike
   |
   v
Alert
```

With:

```text
Condition > threshold
       |
       v
10 minutes
       |
       v
Alert
```

Use durations appropriate to the symptom.

---

# 60. Alert Labels

Useful labels:

```yaml
labels:
  severity: critical
  team: backend-team
  service: order-service
  environment: production
```

These allow Alertmanager to route alerts.

For example:

```text
production
+
critical
+
backend-team
```

can route to the backend on-call channel.

---

# 61. Alert Annotations

Useful annotations:

```yaml
annotations:
  summary: Order service error rate is high
  description: HTTP 5xx rate exceeded 5% for 10 minutes
  runbook_url: ...
```

A good alert should tell the responder:

```text
What happened?
How serious?
Which service?
Which environment?
What should I investigate?
```

Avoid alerts that simply say:

```text
Something is wrong.
```

---

# 62. Alertmanager

Prometheus detects alert conditions.

Alertmanager handles:

```text
Grouping
Deduplication
Routing
Silencing
Inhibition
Notifications
```

Architecture:

```text
Prometheus
    |
    | Alert
    v
Alertmanager
    |
    +--> Slack
    +--> Email
    +--> PagerDuty
    +--> Webhook
```

This separation is important.

---

# 63. Alertmanager Routing

Example:

```text
severity=critical
       |
       v
PagerDuty / on-call

severity=warning
       |
       v
Slack

team=database
       |
       v
DBA/SRE channel
```

Routing should reflect operational ownership.

---

# 64. Alert Grouping

Suppose:

```text
order-service
payment-service
catalog-service
```

all become unavailable because the EKS cluster has a networking issue.

Without grouping:

```text
100 alerts
```

With grouping:

```text
EKS networking problem
+
affected services
```

This reduces alert fatigue.

---

# 65. Alert Fatigue

Bad monitoring:

```text
CPU > 70%
Memory > 70%
Pod restart
Pod restart
Pod restart
CPU > 70%
Disk > 70%
...
```

Thousands of alerts can train engineers to ignore alerts.

Good monitoring focuses on:

```text
User impact
Service availability
SLO violations
Actionable infrastructure failures
```

---

# 66. SLO Metrics

Suppose `order-service` has an SLO:

```text
99.9% successful requests
```

We can calculate:

```text
good requests
/
total requests
```

Then:

```text
Error budget
```

can be tracked.

Prometheus provides the raw metrics needed for this.

---

# 67. Availability Example

Suppose:

```text
Total requests = 1,000,000
Failed requests = 500
```

Availability:

```text
999,500 / 1,000,000
=
99.95%
```

This can be compared with:

```text
SLO = 99.9%
```

The service is currently above the SLO.

---

# 68. Saturation Monitoring

Monitor saturation such as:

```text
CPU throttling
Memory pressure
Node capacity
Connection pools
Thread pools
RDS connections
Redis connections
SQS backlog
Kafka consumer lag
```

Prometheus can collect many of these signals.

This is where metrics become useful for capacity planning, not only incident detection.

---

# 69. SQS Monitoring

Our messaging architecture uses SQS/SNS.

Useful SQS metrics include:

```text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
ApproximateAgeOfOldestMessage
```

AWS publishes these metrics through CloudWatch.

Prometheus does not automatically know about every AWS metric.

An AWS exporter/integration may be used when those metrics need to be brought into Prometheus/Grafana.

Alternatively, Grafana can query CloudWatch directly depending on the observability architecture.

---

# 70. RDS Monitoring

RDS exposes important CloudWatch metrics.

Examples:

```text
CPUUtilization
DatabaseConnections
FreeStorageSpace
FreeableMemory
ReadLatency
WriteLatency
ReadIOPS
WriteIOPS
```

These can be visualized in CloudWatch/Grafana.

If we need them in Prometheus:

```text
AWS metrics
   |
   v
AWS metrics exporter/integration
   |
   v
Prometheus
```

Do not assume Prometheus automatically collects all AWS service metrics.

---

# 71. Redis Monitoring

For ElastiCache:

```text
CPU
Memory
Evictions
Connections
Commands
Replication
Latency
```

AWS provides CloudWatch metrics.

Again:

```text
ElastiCache
   |
   v
CloudWatch
```

is the native AWS monitoring path.

Prometheus can be integrated when centralized metric querying is required.

---

# 72. Prometheus Storage

Prometheus stores time-series data locally in its TSDB.

Conceptually:

```text
Scraped metrics
      |
      v
Prometheus TSDB
      |
      +--> blocks
      +--> WAL
```

The local TSDB is optimized for time-series workloads.

---

# 73. WAL

Prometheus uses a Write-Ahead Log.

Conceptually:

```text
Incoming sample
      |
      v
WAL
      |
      v
TSDB
```

The WAL helps provide durability for recent samples and recovery after restart.

This is one reason Prometheus storage should use reliable persistent storage in production.

---

# 74. Persistent Volume

A production Prometheus instance should normally have persistent storage.

Example:

```text
Prometheus
    |
    v
PVC
    |
    v
EBS
```

For EKS:

```text
Prometheus
   |
   v
EBS CSI
   |
   v
gp3 EBS volume
```

The storage architecture must account for EBS being AZ-scoped.

---

# 75. Prometheus and EBS Availability

EBS volumes are associated with an Availability Zone.

Therefore:

```text
Prometheus Pod
      |
      v
EBS volume
```

must be scheduled in a compatible AZ.

Kubernetes storage topology and the EBS CSI driver handle this relationship.

Do not assume a single EBS-backed Prometheus can freely move across AZs without storage considerations.

---

# 76. Retention

Prometheus retention can be configured by:

```text
time
size
```

Example concept:

```text
15 days
```

or:

```text
50GB
```

Longer retention means:

```text
More storage
More resource requirements
More operational cost
```

Do not automatically keep years of raw Prometheus data locally.

---

# 77. Long-Term Metrics

For long-term storage, organizations may use systems such as:

```text
Thanos
Cortex
Grafana Mimir
Amazon Managed Service for Prometheus
```

The correct choice depends on requirements.

Conceptually:

```text
Prometheus
    |
    v
Remote Write
    |
    v
Long-term metrics backend
```

Prometheus can then remain focused on local operational monitoring while historical data is stored elsewhere.

---

# 78. High Availability

A single Prometheus instance is a potential monitoring failure point.

A production architecture may use:

```text
Prometheus A
Prometheus B
```

scraping the same targets.

Conceptually:

```text
Targets
  |
  +---------> Prometheus A
  |
  +---------> Prometheus B
```

A downstream system can deduplicate the resulting series if required.

HA design should be selected based on monitoring criticality and scale.

---

# 79. Why Monitoring Must Survive Application Failure

Imagine:

```text
order-service
    |
    X
down
```

We still need:

```text
Prometheus
Grafana
Alertmanager
```

to remain operational.

Therefore monitoring infrastructure should not be dependent on the application it monitors.

Example:

```text
production
 |
 +--> applications

monitoring
 |
 +--> Prometheus
 +--> Grafana
 +--> Alertmanager
```

Separate namespace and appropriate capacity help.

---

# 80. Prometheus Resource Sizing

Prometheus resource requirements depend heavily on:

```text
Number of targets
Scrape interval
Number of metrics
Active time series
Label cardinality
Retention
Query load
Recording rules
Remote write
```

The number of Pods alone is not enough to size Prometheus.

For example:

```text
100 Pods
```

can generate far more metrics than:

```text
500 Pods
```

if the first workload has extremely high metric cardinality.

---

# 81. Cardinality Is Often the Real Problem

Suppose:

```text
21 services
x
1000 time series/service
```

approximately:

```text
21,000 time series
```

This may be manageable.

But:

```text
21 services
x
100,000 time series/service
```

becomes:

```text
2.1 million time series
```

This significantly changes resource requirements.

Therefore monitor:

```text
active series
samples/sec
scrape duration
Prometheus memory
disk usage
query latency
```

---

# 82. Prometheus Self-Monitoring

Prometheus should monitor itself.

Important metrics include:

```text
prometheus_tsdb_head_series
prometheus_tsdb_head_samples_appended_total
prometheus_target_scrapes_exceeded_sample_limit_total
prometheus_rule_evaluation_failures_total
prometheus_engine_query_duration_seconds
```

The exact available metrics depend on Prometheus version.

Monitor:

```text
Prometheus health
Prometheus capacity
Prometheus performance
```

---

# 83. Prometheus Alert Examples

Important alerts include:

```text
PrometheusDown
TargetDown
HighScrapeDuration
ScrapeErrors
HighCardinality
PrometheusStorageLow
RuleEvaluationFailures
RemoteWriteFailures
```

The exact thresholds should be based on production behavior.

---

# 84. Target Down Alert

Conceptually:

```promql
up == 0
```

But this can be noisy.

A production alert should include:

```text
job
service
environment
target
```

and a duration.

Example:

```yaml
for: 5m
```

This avoids paging on a momentary scrape failure.

---

# 85. Prometheus Query Performance

Bad dashboard:

```text
Very expensive query
+
1 second refresh
+
large time range
```

This can overload Prometheus.

Avoid:

```text
Every panel
   |
   v
Complex raw PromQL
```

Prefer:

```text
Recording rules
+
reasonable dashboard refresh
+
bounded queries
```

---

# 86. Dashboard Query Design

A dashboard should use filters:

```text
Environment
Service
Namespace
Pod
Status
Route
```

Example:

```text
environment="production"
service="order-service"
```

This avoids querying the entire platform unnecessarily.

---

# 87. Prometheus Security

Prometheus contains operational information.

Protect:

```text
Prometheus UI
Prometheus API
Metrics endpoints
Remote-write endpoints
```

Do not expose Prometheus directly to the public internet.

Preferred:

```text
Engineer
   |
   v
Internal access
   |
   v
Grafana
   |
   v
Prometheus
```

Users normally interact with Grafana rather than directly accessing Prometheus.

---

# 88. Metrics Endpoint Security

Application metrics can reveal:

```text
service names
internal endpoints
database pool state
JVM information
request patterns
infrastructure details
```

Therefore metrics endpoints should not automatically be publicly exposed.

Example:

```text
/actuator/prometheus
```

should be reachable by the monitoring system, not necessarily by the internet.

---

# 89. NetworkPolicy

Prometheus needs access to metrics endpoints.

A NetworkPolicy can allow:

```text
Prometheus
   |
   v
Application Pods :8080
```

while preventing unrelated workloads from accessing metrics.

Example conceptual policy:

```text
allow ingress to metrics port
source = monitoring namespace
```

Network policy must be aligned with the EKS networking implementation.

---

# 90. Service Discovery

Kubernetes is highly dynamic.

Pods can be:

```text
created
destroyed
rescheduled
scaled
```

Static IP-based monitoring would be impractical.

Prometheus integrates with Kubernetes service discovery.

Conceptually:

```text
Kubernetes API
     |
     v
Prometheus
     |
     +--> discover Pods
     +--> discover Services
     +--> discover Nodes
```

This is one of the major reasons Prometheus works well with Kubernetes.

---

# 91. Dynamic Pods

Suppose:

```text
order-service
3 Pods
```

HPA scales:

```text
3 -> 7
```

Prometheus should automatically discover the new Pods.

When Pods disappear:

```text
7 -> 3
```

their scrape targets should disappear as appropriate.

This is why Kubernetes service discovery is critical.

---

# 92. HPA and Prometheus

HPA can consume metrics through Kubernetes metrics APIs or custom/external metrics depending on architecture.

Prometheus itself is not automatically the HPA source in every setup.

A common architecture for custom metrics is:

```text
Application
   |
   v
Prometheus
   |
   v
Prometheus Adapter
   |
   v
Kubernetes HPA
```

For CPU/memory-based HPA:

```text
Kubernetes Metrics Server
```

is typically involved instead.

This distinction is important.

---

# 93. Metrics Server vs Prometheus

Do not confuse:

```text
Metrics Server
```

with:

```text
Prometheus
```

Metrics Server is primarily used for Kubernetes resource metrics such as:

```text
CPU
Memory
```

and supports functionality such as:

```bash
kubectl top pods
```

Prometheus is a broader time-series monitoring system.

Therefore:

```text
Metrics Server
=
Kubernetes resource metrics

Prometheus
=
Monitoring/observability metrics
```

---

# 94. `kubectl top`

Example:

```bash
kubectl top pods -n production
```

or:

```bash
kubectl top nodes
```

This is useful for immediate resource inspection.

But it is not a replacement for Prometheus.

Prometheus provides:

```text
Historical data
Queries
Dashboards
Alerts
Longer-term analysis
```

---

# 95. Prometheus + Grafana

Prometheus provides the data.

Grafana visualizes it.

```text
Prometheus
    |
    v
Grafana
```

Example Grafana dashboard:

```text
Order Service
--------------------------------
Request Rate       850 req/s
Error Rate         0.3%
P95 Latency        210 ms
P99 Latency        490 ms
CPU                52%
Memory             61%
Pods               5/5
```

Grafana will be covered in:

```text
31-Grafana.md
```

---

# 96. Prometheus + Alertmanager

Prometheus:

```text
Condition detected
```

Alertmanager:

```text
Notification management
```

Example:

```text
Prometheus
   |
   | High error rate
   v
Alertmanager
   |
   +--> SRE
   +--> Slack
   +--> PagerDuty
```

Alertmanager is not responsible for calculating the metric itself.

---

# 97. Prometheus + ELK

Prometheus handles metrics.

ELK handles logs.

```text
Metrics:
Prometheus

Logs:
Fluent Bit
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

Do not try to store all logs inside Prometheus.

Likewise, do not use Elasticsearch as a direct replacement for Prometheus time-series monitoring.

---

# 98. Prometheus + OpenTelemetry

OpenTelemetry provides:

```text
Metrics
Logs
Traces
```

Prometheus remains an important metrics system in many Kubernetes environments.

For our platform:

```text
Application
    |
    +--> Micrometer -> Prometheus
    |
    +--> OpenTelemetry -> Collector -> Jaeger
    |
    +--> stdout -> Fluent Bit -> ELK
```

This gives separate but correlated observability signals.

---

# 99. Observability Correlation

Suppose:

```text
order-service
HTTP 500 = 7%
```

Prometheus identifies:

```text
Error rate increased
```

Grafana shows:

```text
Spike started at 10:32
```

ELK shows:

```text
Database timeout exceptions
```

Jaeger shows:

```text
order-service
    |
    v
payment-service
    |
    v
database
```

Now we can investigate:

```text
Metric
   |
   v
Log
   |
   v
Trace
   |
   v
Root cause
```

This is the observability architecture we will build.

---

# 100. Production Prometheus Architecture

Our target platform:

```text
                         EKS
                          |
        +-----------------+------------------+
        |                 |                  |
        v                 v                  v
Applications          Nodes            Kubernetes
        |                 |                  |
        v                 v                  v
   Micrometer       Node Exporter     kube-state-metrics
        |                 |                  |
        +-----------------+------------------+
                          |
                          v
                     Prometheus
                          |
             +------------+------------+
             |                         |
             v                         v
          Grafana                 Alertmanager
             |                         |
             v                         v
       Dashboards                 Notifications
```

AWS services:

```text
RDS
Redis
SQS
SNS
ALB
EKS
```

can additionally provide CloudWatch metrics and can be integrated into the centralized observability model where useful.

---

# 101. Production Prometheus Checklist

```text
[ ] Prometheus deployed in monitoring namespace
[ ] Version pinned
[ ] Helm chart version pinned
[ ] Managed through GitOps
[ ] Persistent storage configured
[ ] Storage sized appropriately
[ ] Retention defined
[ ] Service discovery configured
[ ] ServiceMonitors configured
[ ] PodMonitors used where appropriate
[ ] Node Exporter configured
[ ] kube-state-metrics configured
[ ] Application metrics enabled
[ ] Cardinality reviewed
[ ] Resource requests configured
[ ] Resource limits configured
[ ] Network access restricted
[ ] UI access protected
[ ] Recording rules configured
[ ] Alerting rules configured
[ ] Alertmanager integrated
[ ] Grafana integrated
[ ] Prometheus self-monitoring enabled
[ ] Backup/DR strategy defined where required
[ ] Long-term storage strategy defined where required
```

---

# 102. Prometheus Troubleshooting Checklist

If metrics are missing:

```text
1. Is Prometheus running?
2. Is the target discovered?
3. Is the target UP?
4. Is ServiceMonitor selected?
5. Is PodMonitor selected?
6. Does Service selector match Pods?
7. Does the metrics endpoint work?
8. Is the port correct?
9. Is the path correct?
10. Is NetworkPolicy blocking traffic?
11. Is TLS configured correctly?
12. Are labels correct?
13. Is the metric actually exported?
14. Is the metric being dropped by relabeling?
15. Is cardinality/resource pressure causing problems?
```

---

# 103. Example End-to-End Troubleshooting

Problem:

```text
Grafana:
No order-service metrics
```

Step 1:

```bash
kubectl get pods -n monitoring
```

Check Prometheus.

Step 2:

Check targets in Prometheus.

If:

```text
order-service = DOWN
```

continue.

Step 3:

```bash
kubectl get servicemonitor -A
```

Step 4:

```bash
kubectl get svc -n production
```

Step 5:

Check endpoints.

Step 6:

Test:

```bash
curl http://order-service:8080/actuator/prometheus
```

Step 7:

Check NetworkPolicy.

Step 8:

Check Prometheus logs.

This prevents randomly modifying dashboards.

---

# 104. Example: High Error Rate

Alert:

```text
OrderServiceHighErrorRate
```

First:

```text
Grafana
```

Check:

```text
Error rate
Request rate
Latency
Pod count
CPU
Memory
```

Then:

```text
ELK
```

Search:

```text
order-service
HTTP 500
```

Then:

```text
Jaeger
```

Check request traces.

Then:

```text
RDS
Redis
SQS
```

Check dependency health.

The investigation becomes:

```text
Prometheus
    |
    v
What changed?
    |
    v
ELK
    |
    v
What error?
    |
    v
Jaeger
    |
    v
Where in request path?
    |
    v
Dependency
    |
    v
Root cause
```

---

# 105. Low Traffic

For low traffic:

```text
Few Pods
Low request rate
Low metric volume
```

Prometheus requirements are smaller.

Example:

```text
3 replicas
30s scrape interval
15-day retention
```

This is suitable for a smaller environment subject to actual metric volume.

---

# 106. Medium Traffic

For medium traffic:

```text
More Pods
More metrics
More queries
More dashboards
```

We may need:

```text
More Prometheus CPU
More memory
More storage
Cardinality control
Recording rules
```

Example:

```text
10 replicas
15-30s scrape
30-day retention
```

The actual values must be derived from observed workload.

---

# 107. High Traffic

For high traffic:

```text
Many services
Many replicas
Large metric volume
High query volume
Large time-series count
```

Consider:

```text
Prometheus HA
Remote write
Thanos/Mimir/Cortex
Managed Prometheus
Metric sharding
Cardinality control
Recording rules
Dedicated monitoring nodes
```

Do not simply keep increasing one Prometheus instance forever.

---

# 108. Dedicated Monitoring Nodes

For production, monitoring workloads can be isolated.

Example:

```text
Node Group
   |
   +--> monitoring=true
```

Prometheus/Grafana/Alertmanager can use:

```text
nodeSelector
tolerations
affinity
topology spread
```

This prevents application workloads from competing heavily with monitoring infrastructure.

---

# 109. Monitoring Capacity Planning

Monitor Prometheus itself.

Important dimensions:

```text
Active time series
Samples ingested/sec
Scrape duration
Storage growth
Query rate
Query latency
Rule evaluation duration
Memory
CPU
```

Example:

```text
Active series:
1.2 million

Samples/sec:
80,000

Storage growth:
X GB/day
```

These numbers are much more useful for sizing than:

```text
We have 100 Pods.
```

---

# 110. Production Anti-Patterns

Avoid:

```text
Using Prometheus as log storage
Using high-cardinality labels
No persistence
No retention policy
No monitoring of Prometheus itself
Public Prometheus endpoint
One giant unbounded query
1-second dashboard refresh everywhere
Scraping every endpoint blindly
Alerting on every tiny CPU spike
No alert ownership
No runbooks
```

---

# 111. Prometheus Design Principles

Remember:

```text
Metrics should be:
Useful
Queryable
Bounded
Actionable
Reliable
```

The objective is not:

```text
Collect every possible metric.
```

The objective is:

```text
Collect the metrics required to understand and operate the platform.
```

---

# 112. Final Mental Model

The complete metrics architecture is:

```text
                    APPLICATIONS
                         |
                    Micrometer
                         |
                         v
              /actuator/prometheus
                         |
                         v
                       Prometheus
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Grafana        Alertmanager     Recording Rules
        |                |
        v                v
   Dashboards        Notifications
```

Kubernetes:

```text
Kubernetes Nodes
      |
      v
Node Exporter
      |
      v
Prometheus

Kubernetes API/Object State
      |
      v
kube-state-metrics
      |
      v
Prometheus
```

AWS:

```text
RDS
Redis
SQS
SNS
ALB
EKS
 |
 v
CloudWatch
 |
 v
Grafana / AWS integration
```

---

# 113. Complete Observability Architecture

Eventually our project will have:

```text
                         EKS
                          |
        +-----------------+------------------+
        |                 |                  |
        v                 v                  v
 Applications          Nodes            Kubernetes
        |                 |                  |
        |                 v                  v
        |            Node Exporter     kube-state-metrics
        |                 |                  |
        v                 +--------+---------+
   Micrometer                      |
        |                          |
        +--------------------------+
                   |
                   v
              Prometheus
                   |
          +--------+--------+
          |                 |
          v                 v
       Grafana         Alertmanager
```

Logs:

```text
Application
    |
    v
stdout/stderr
    |
    v
Fluent Bit
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

Traces:

```text
Application
    |
    v
OpenTelemetry SDK
    |
    v
OTel Collector
    |
    v
Jaeger
```

Together:

```text
                Observability
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
     Metrics        Logs         Traces
       |             |             |
       v             v             v
  Prometheus         ELK         Jaeger
       |             |             |
       +-------------+-------------+
                     |
                     v
                   SRE
```

---

# 114. Next Chapter

## 31 — Grafana

The next chapter will build the visualization layer:

```text
Prometheus
     |
     v
Grafana
```

We will cover:

```text
Grafana architecture
Installation on EKS
Helm deployment
Production HA
Data sources
Prometheus integration
CloudWatch integration
Dashboards
Variables
Templating
Panels
Time-series panels
Stat panels
Tables
Heatmaps
Latency dashboards
RED dashboards
USE dashboards
Kubernetes dashboards
EKS dashboards
Node dashboards
Application dashboards
RDS dashboards
Redis dashboards
SQS dashboards
ArgoCD dashboards
Alert visualization
Dashboard provisioning
ConfigMaps
GitOps-managed dashboards
Folder organization
RBAC
SSO
Secrets
Datasource provisioning
Production security
Dashboard performance
Troubleshooting
```

Target architecture:

```text
                    Prometheus
                        |
                        v
                     Grafana
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
 Application        Kubernetes        AWS
 Dashboards         Dashboards      Dashboards
        |               |               |
        +---------------+---------------+
                        |
                        v
                       SRE
```

The objective is to move from:

```text
Raw metrics
```

to:

```text
Operational dashboards
+
Actionable alerts
+
Production visibility
```

```
```
