# 118-Prometheus-Production-Implementation.md

````markdown
# 118-Prometheus-Production-Implementation.md

# Prometheus Production Implementation on AWS EKS

**Project:** Enterprise E-Commerce Platform  
**Platform:** AWS EKS  
**Monitoring:** Prometheus  
**Visualization:** Grafana  
**Alerting:** Alertmanager  
**Deployment:** Helm + ArgoCD  
**Infrastructure:** Terraform  
**Applications:** 21 Microservices  
**Region:** `ap-south-1`

---

# 1. Objective

In the previous chapters we built the complete event-driven architecture.

Now we begin the **production observability implementation**.

The first component is:

```text
Prometheus
````

Prometheus will collect metrics from:

* Kubernetes
* EKS nodes
* 21 microservices
* Kafka
* RDS
* Redis
* KEDA
* Kubernetes controllers
* Application infrastructure
* Platform components

The architecture becomes:

```text
Applications
     |
     | /actuator/prometheus
     v
Prometheus
     |
     +--> Kubernetes metrics
     +--> Node metrics
     +--> Kafka metrics
     +--> RDS metrics
     +--> Redis metrics
     |
     v
Grafana
     |
     v
Alertmanager
```

---

# 2. What Is Prometheus?

Prometheus is a metrics collection and monitoring system.

It stores time-series data.

Example:

```text
http_requests_total{service="order-service",status="200"}
```

Prometheus stores the value over time.

Example:

```text
10:00 -> 10,000
10:01 -> 10,250
10:02 -> 10,700
```

We can calculate:

```text
requests/sec
error rate
latency
CPU usage
memory usage
pod count
Kafka lag
database connections
```

---

# 3. Logs vs Metrics vs Traces

These are different observability signals.

## Metrics

Answer:

```text
"What is happening?"
```

Example:

```text
CPU = 72%
HTTP error rate = 2%
Kafka lag = 1,200
```

---

## Logs

Answer:

```text
"What exactly happened?"
```

Example:

```text
ERROR Payment provider timeout
orderId=ORD-1001
```

---

## Traces

Answer:

```text
"Where did the request spend time?"
```

Example:

```text
Order API
   |
   +--> Pricing 120ms
   |
   +--> Tax 80ms
   |
   +--> Payment 900ms
```

We will implement the other observability components in later chapters.

---

# 4. Prometheus Architecture

Our production architecture:

```text
                    EKS
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
  Applications    Node Exporter  kube-state-metrics
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
```

---

# 5. Metrics Sources

Prometheus can scrape multiple sources.

## Application metrics

```text
order-service
payment-service
inventory-service
...
```

Spring Boot exposes:

```text
/actuator/prometheus
```

---

## Node metrics

Node Exporter provides:

```text
CPU
Memory
Disk
Network
Filesystem
Load
```

---

## Kubernetes metrics

`kube-state-metrics` provides Kubernetes object state such as:

```text
Deployment replicas
Pod status
DaemonSet state
StatefulSet state
Job state
Node conditions
```

---

## Infrastructure metrics

AWS services can expose metrics through CloudWatch and exporters/integrations.

Examples:

```text
RDS
MSK
ElastiCache
ALB
NAT Gateway
```

We should not blindly install an exporter for every AWS service.

Use the appropriate AWS integration for the metrics required by the platform.

---

# 6. Production Monitoring Namespace

Create:

```text
monitoring
```

Namespace:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: monitoring
```

In our GitOps repository:

```text
gitops/
└── infrastructure/
    └── monitoring/
        ├── namespace.yaml
        ├── prometheus/
        ├── grafana/
        └── alertmanager/
```

---

# 7. Helm-Based Installation

For a Kubernetes environment, we will use the Prometheus community Helm stack.

The commonly used chart is:

```text
kube-prometheus-stack
```

It packages:

```text
Prometheus
Alertmanager
Grafana
kube-state-metrics
Prometheus Operator
```

The exact chart version should be pinned and tested against our EKS/Kubernetes version.

Do not use:

```text
helm install ... latest
```

for production.

---

# 8. Repository Structure

Our GitOps repository becomes:

```text
gitops/
|
├── infrastructure/
│   └── monitoring/
│       ├── namespace.yaml
│       ├── prometheus/
│       │   ├── values-prod.yaml
│       │   └── application.yaml
│       |
│       ├── grafana/
│       └── alertmanager/
│
└── apps/
    └── services/
        ├── order-service/
        ├── payment-service/
        ├── inventory-service/
        └── ...
```

ArgoCD manages the installation.

---

# 9. Prometheus Helm Values

Example production values:

```yaml
prometheus:
  prometheusSpec:

    replicas: 2

    retention: 15d

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: "2"
        memory: 4Gi

    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes:
            - ReadWriteOnce

          resources:
            requests:
              storage: 100Gi

    serviceMonitorSelectorNilUsesHelmValues: false

    podMonitorSelectorNilUsesHelmValues: false
```

These values are starting points.

Production sizing depends on:

```text
Number of targets
Scrape interval
Number of metrics
Metric cardinality
Retention
Query workload
Remote storage
```

---

# 10. Prometheus Persistence

Do not run production Prometheus with:

```text
emptyDir
```

because Prometheus data disappears when the pod is recreated.

Use persistent storage.

Architecture:

```text
Prometheus Pod
      |
      v
PVC
      |
      v
EBS gp3
```

Example:

```text
100Gi
```

is only an example.

Calculate required storage from actual ingestion rate and retention.

---

# 11. Prometheus High Availability

Production:

```text
Prometheus-0
Prometheus-1
```

Both scrape the targets.

However:

```text
2 replicas
!=
automatically deduplicated HA
```

You must consider how dashboards and long-term storage handle duplicate samples.

For larger environments, evaluate:

```text
Thanos
Cortex
Grafana Mimir
Amazon Managed Service for Prometheus
```

depending on organizational requirements.

For our initial EKS platform, two Prometheus replicas provide monitoring redundancy, while the long-term architecture can evolve separately.

---

# 12. Prometheus Operator

The Prometheus Operator allows us to define monitoring configuration using Kubernetes resources.

Important CRDs include:

```text
Prometheus
ServiceMonitor
PodMonitor
PrometheusRule
Alertmanager
```

Example:

```text
ServiceMonitor
      |
      v
Prometheus Operator
      |
      v
Prometheus configuration
```

---

# 13. ServiceMonitor

A `ServiceMonitor` tells Prometheus Operator which Kubernetes Service to scrape.

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: order-service
  namespace: monitoring

spec:
  namespaceSelector:
    matchNames:
      - ecommerce

  selector:
    matchLabels:
      app: order-service

  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
```

The exact label and port names must match the Service deployed by the application Helm chart.

---

# 14. Application Service

Order Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: order-service
  namespace: ecommerce

  labels:
    app: order-service

spec:
  selector:
    app: order-service

  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Now Prometheus can discover the service.

---

# 15. Spring Boot Metrics Endpoint

Our Order Service already exposes:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

Prometheus endpoint:

```text
/actuator/prometheus
```

Test inside the cluster:

```bash
kubectl run curl-test \
  --rm \
  -it \
  --image=curlimages/curl \
  -n ecommerce \
  -- \
  curl http://order-service:8080/actuator/prometheus
```

Expected output:

```text
# HELP ...
# TYPE ...
jvm_memory_used_bytes ...
http_server_requests_seconds_count ...
```

---

# 16. Why We Do Not Expose `/actuator/prometheus` Publicly

Bad:

```text
Internet
   |
   v
ALB
   |
   v
/actuator/prometheus
```

Prometheus metrics should not normally be publicly exposed.

Better:

```text
Prometheus
    |
    v
Internal Kubernetes Service
    |
    v
Application
```

Use:

```text
ClusterIP
```

for the application service unless another exposure mechanism is required.

---

# 17. Application Metrics

Spring Boot provides JVM and HTTP metrics.

Examples:

```text
jvm_memory_used_bytes
jvm_threads_live_threads
process_cpu_usage
http_server_requests_seconds_count
http_server_requests_seconds_sum
```

These allow us to calculate:

```text
Request rate
Error rate
Latency
JVM memory
CPU
```

---

# 18. RED Method

For services, use:

```text
R = Rate
E = Errors
D = Duration
```

Example:

```text
Request Rate
     |
     v
HTTP requests/sec

Error Rate
     |
     v
HTTP 5xx %

Duration
     |
     v
Latency
```

For Order Service:

```text
Requests/sec
5xx rate
p95 latency
p99 latency
```

are more useful than simply watching CPU.

---

# 19. USE Method

For infrastructure:

```text
U = Utilization
S = Saturation
E = Errors
```

Example node:

```text
CPU utilization
Memory utilization
Disk saturation
Network errors
```

This complements application-level RED metrics.

---

# 20. Node Exporter

Node Exporter runs as a DaemonSet.

Architecture:

```text
Node 1
 |
 +--> node-exporter

Node 2
 |
 +--> node-exporter

Node 3
 |
 +--> node-exporter
```

Each node gets one exporter pod.

It exposes host metrics.

---

# 21. Node Exporter Metrics

Examples:

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_memory_MemTotal_bytes
node_filesystem_avail_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
node_network_transmit_bytes_total
```

We can build:

```text
CPU %
Memory %
Disk %
Network throughput
```

---

# 22. Important Node Exporter Detail

Node Exporter is installed on:

```text
Kubernetes worker nodes
```

not inside every application container.

Example:

```text
EC2 Node
 |
 +-- kubelet
 +-- container runtime
 +-- order-service pod
 +-- payment-service pod
 +-- inventory-service pod
 +-- node-exporter
```

One exporter collects host-level metrics for the node.

---

# 23. kube-state-metrics

`kube-state-metrics` exposes Kubernetes object state.

Example:

```text
Deployment desired replicas
Deployment available replicas
Pod phase
Pod restart count
Node conditions
DaemonSet desired/current pods
```

Useful metrics include:

```text
kube_deployment_spec_replicas
kube_deployment_status_replicas_available
kube_pod_status_phase
kube_pod_container_status_restarts_total
```

---

# 24. Kubernetes Monitoring Architecture

```text
                   Kubernetes API
                        |
                        v
               kube-state-metrics
                        |
                        v
Prometheus <-------- Metrics
   ^
   |
   +------ node-exporter
   |
   +------ application /metrics
   |
   +------ platform exporters
```

---

# 25. Prometheus Scrape Flow

Every 30 seconds, for example:

```text
Prometheus
    |
    v
ServiceMonitor
    |
    v
order-service:8080
    |
    v
/actuator/prometheus
```

Prometheus receives:

```text
metric_name
labels
value
timestamp
```

---

# 26. Labels

Example:

```text
http_server_requests_seconds_count{
  application="order-service",
  method="GET",
  status="200",
  uri="/api/v1/orders"
}
```

Labels allow us to filter.

Example:

```promql
http_server_requests_seconds_count{
  application="order-service"
}
```

---

# 27. Label Cardinality

This is extremely important.

Bad label:

```text
userId="987654321"
```

If millions of users exist:

```text
millions of time series
```

This creates high cardinality.

Avoid putting unbounded values into metric labels.

Bad:

```text
orderId
customerId
requestId
traceId
```

as Prometheus labels.

These belong in logs/traces.

---

# 28. Good Labels

Good:

```text
service
namespace
method
status
route
environment
```

Example:

```text
service="order-service"
environment="prod"
method="GET"
status="200"
route="/api/v1/orders/{id}"
```

The route should be a normalized template rather than a unique URL containing IDs.

---

# 29. Prometheus Query Language

PromQL is used to query metrics.

Example:

```promql
up
```

Shows whether targets are reachable.

---

# 30. Check Order Service

```promql
up{job="order-service"}
```

Expected:

```text
1
```

means target is up.

```text
0
```

means target is down/unavailable from Prometheus's perspective.

---

# 31. Request Rate

Example:

```promql
rate(
  http_server_requests_seconds_count{
    application="order-service"
  }[5m]
)
```

This calculates approximate requests per second over five minutes.

---

# 32. HTTP 5xx Rate

Example:

```promql
sum(
  rate(
    http_server_requests_seconds_count{
      application="order-service",
      status=~"5.."
    }[5m]
  )
)
```

This gives the rate of HTTP 5xx responses.

---

# 33. Total Request Rate

```promql
sum(
  rate(
    http_server_requests_seconds_count{
      application="order-service"
    }[5m]
  )
)
```

---

# 34. Error Percentage

```promql
100 *
(
  sum(
    rate(
      http_server_requests_seconds_count{
        application="order-service",
        status=~"5.."
      }[5m]
    )
  )
  /
  sum(
    rate(
      http_server_requests_seconds_count{
        application="order-service"
      }[5m]
    )
  )
)
```

This produces:

```text
5xx %
```

---

# 35. Latency

Spring Boot exposes histogram metrics depending on configuration.

Example:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(
      http_server_requests_seconds_bucket{
        application="order-service"
      }[5m]
    )
  )
)
```

This estimates p95 latency.

Similarly:

```promql
histogram_quantile(
  0.99,
  sum by (le) (
    rate(
      http_server_requests_seconds_bucket{
        application="order-service"
      }[5m]
    )
  )
)
```

gives p99.

---

# 36. CPU Usage

Container CPU:

```promql
sum by (pod) (
  rate(
    container_cpu_usage_seconds_total{
      namespace="ecommerce"
    }[5m]
  )
)
```

For production dashboards, filter out infrastructure containers and ensure the metric source/version matches the installed Kubernetes metrics pipeline.

---

# 37. Memory Usage

Example:

```promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="ecommerce"
  }
)
```

This helps identify memory-heavy pods.

---

# 38. Pod Restarts

```promql
sum by (pod) (
  increase(
    kube_pod_container_status_restarts_total{
      namespace="ecommerce"
    }[15m]
  )
)
```

This helps identify:

```text
CrashLoopBackOff
OOMKilled
Application crashes
Probe failures
```

---

# 39. Deployment Availability

```promql
kube_deployment_status_replicas_available{
  namespace="ecommerce"
}
```

Compare with:

```promql
kube_deployment_spec_replicas{
  namespace="ecommerce"
}
```

If:

```text
Desired = 5
Available = 3
```

we have a deployment availability problem.

---

# 40. PrometheusRule

We can store alert rules as Kubernetes resources.

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: ecommerce-application-rules
  namespace: monitoring

spec:
  groups:
    - name: ecommerce.application

      rules:
        - alert: OrderServiceDown

          expr: |
            up{job="order-service"} == 0

          for: 5m

          labels:
            severity: critical
            service: order-service

          annotations:
            summary: "Order Service is unavailable"
            description: "Prometheus cannot scrape order-service for 5 minutes."
```

Alert routing will be implemented in the Alertmanager chapter.

---

# 41. Recording Rules

Complex PromQL queries can be expensive to calculate repeatedly.

Instead:

```text
Raw Metrics
     |
     v
Recording Rule
     |
     v
Precomputed Time Series
```

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: ecommerce-recording-rules
  namespace: monitoring

spec:
  groups:
    - name: ecommerce.recording

      rules:
        - record: service:http_requests_per_second:rate5m

          expr: |
            sum by (service) (
              rate(
                http_server_requests_seconds_count[5m]
              )
            )
```

Then Grafana can query:

```promql
service:http_requests_per_second:rate5m
```

instead of repeatedly executing the full query.

---

# 42. Prometheus Resource Planning

Prometheus resources depend heavily on:

```text
Number of targets
Scrape interval
Metrics per target
Cardinality
Retention
Query volume
Recording rules
```

Starting configuration:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 2Gi

  limits:
    cpu: "2"
    memory: 4Gi
```

Do not treat these as universal production values.

Measure actual Prometheus ingestion and memory usage.

---

# 43. Scrape Interval

Example:

```yaml
interval: 30s
```

Means Prometheus scrapes every 30 seconds.

Shorter:

```text
10s
```

means:

```text
More data
More network
More CPU
More storage
```

Longer:

```text
60s
```

means:

```text
Lower resource usage
Lower resolution
```

Use different intervals where justified.

Critical application metrics may need more frequent scraping than slow-changing infrastructure state.

---

# 44. Retention

Example:

```yaml
retention: 15d
```

This means local Prometheus retains approximately 15 days of data, subject to storage and retention configuration.

Possible architecture:

```text
Prometheus
 |
 +--> 15 days local
 |
 +--> Long-term storage
```

For long-term metrics, consider:

```text
Amazon Managed Service for Prometheus
Thanos
Grafana Mimir
Cortex
```

depending on platform requirements.

---

# 45. Prometheus Storage Calculation

Conceptually:

```text
Storage
≈
Samples per second
×
Bytes per sample
×
Retention duration
```

But actual disk consumption depends on:

```text
Compression
WAL
Series churn
Metadata
Compaction
Head block
```

Therefore:

```text
Measure actual ingestion
```

before finalizing production volume size.

---

# 46. Prometheus Targets

Check targets from Prometheus UI:

```text
Status
  |
  v
Targets
```

You should see:

```text
order-service
payment-service
inventory-service
...
```

Each target:

```text
UP
```

or:

```text
DOWN
```

---

# 47. Target DOWN Troubleshooting

If:

```text
order-service = DOWN
```

check:

```text
1. Service exists
2. Service labels
3. ServiceMonitor labels
4. Service port name
5. Target port
6. /actuator/prometheus
7. NetworkPolicy
8. Prometheus RBAC
9. Prometheus target configuration
10. Application logs
```

---

# 48. Kubernetes Service Discovery

Our target chain:

```text
ServiceMonitor
      |
      v
Service
      |
      v
Endpoints / EndpointSlices
      |
      v
Pod
      |
      v
/actuator/prometheus
```

If the Service has no endpoints:

```text
Service
   |
   X
No matching pods
```

Prometheus cannot scrape the application.

---

# 49. Common ServiceMonitor Mistake

Service:

```yaml
labels:
  app: order-service
```

ServiceMonitor:

```yaml
selector:
  matchLabels:
    app: payment-service
```

Result:

```text
No target
```

Always verify labels.

---

# 50. Port Name Mistake

Service:

```yaml
ports:
  - name: http
    port: 8080
```

ServiceMonitor:

```yaml
endpoints:
  - port: metrics
```

Result:

```text
No matching port
```

Use:

```yaml
port: http
```

or create a dedicated metrics port.

---

# 51. Dedicated Metrics Port

An application can expose metrics separately.

Example:

```text
Application
 |
 +--> 8080 application
 |
 +--> 8081 metrics
```

Service:

```yaml
ports:
  - name: http
    port: 8080
    targetPort: 8080

  - name: metrics
    port: 8081
    targetPort: 8081
```

This can provide stronger network separation.

However, our current Spring Boot implementation exposes Actuator on the application port, so we can start with:

```text
8080/actuator/prometheus
```

and harden further if required.

---

# 52. NetworkPolicy

Suppose we apply:

```text
default deny
```

to `ecommerce`.

Prometheus needs permission to scrape application pods.

Example concept:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-prometheus
  namespace: ecommerce

spec:
  podSelector:
    matchLabels:
      metrics: "enabled"

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring

      ports:
        - protocol: TCP
          port: 8080
```

The exact selector and port must match our deployed application.

---

# 53. Prometheus RBAC

Prometheus needs permission to discover Kubernetes targets.

The Prometheus Operator manages the required Kubernetes resources when correctly configured.

The important principle is:

```text
Prometheus
    |
    v
Kubernetes API
    |
    +--> Pods
    +--> Services
    +--> Endpoints
    +--> Nodes
    +--> ServiceMonitors
```

Use least privilege where custom RBAC is required.

---

# 54. Monitoring the 21 Microservices

Our target list:

```text
order-service
payment-service
inventory-service
auth-service
user-service
customer-service
catalog-service
product-service
pricing-service
promotion-service
cart-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

Each should expose standardized metrics.

---

# 55. Standard Application Metrics

Every service should provide:

```text
HTTP request count
HTTP request latency
HTTP error count
JVM memory
JVM threads
CPU
Database connection pool
Kafka producer metrics
Kafka consumer metrics
Application-specific business metrics
```

For Java/Spring Boot services, Micrometer provides the common metrics foundation.

---

# 56. Standard Labels

Use:

```text
service
namespace
environment
instance
pod
method
route
status
```

Example:

```text
service="order-service"
namespace="ecommerce"
environment="prod"
```

Avoid:

```text
orderId
customerId
traceId
requestId
```

as Prometheus labels.

---

# 57. Business Metrics

Technical metrics are not enough.

Order Service should expose:

```text
orders_created_total
orders_cancelled_total
orders_completed_total
orders_failed_total
```

Payment:

```text
payments_authorized_total
payments_failed_total
payments_refunded_total
```

Inventory:

```text
inventory_reservations_total
inventory_reservation_failures_total
```

These tell us what the business is experiencing.

---

# 58. Example Custom Metric

Java:

```java
Counter ordersCreated = Counter.builder("orders.created")
        .description("Number of orders created")
        .tag("service", "order-service")
        .register(meterRegistry);
```

Then:

```java
ordersCreated.increment();
```

Prometheus receives a metric derived from the Micrometer name.

Use stable low-cardinality tags.

---

# 59. Business Metric Example

```text
orders_created_total
```

Query:

```promql
rate(
  orders_created_total[5m]
)
```

This gives approximate orders per second.

Daily volume can be calculated using:

```promql
increase(
  orders_created_total[24h]
)
```

---

# 60. Why Business Metrics Matter

Suppose:

```text
CPU = 30%
Memory = 40%
Pods = healthy
```

Everything looks normal.

But:

```text
orders_created_total
```

drops from:

```text
100/min
```

to:

```text
0/min
```

The application may be technically healthy but functionally broken.

This is why production observability needs business metrics.

---

# 61. Kafka Metrics

For Kafka we want:

```text
Consumer lag
Producer errors
Consumer errors
Records consumed
Records produced
Broker health
Under-replicated partitions
Offline partitions
```

Kafka monitoring is covered by the metrics integrations available for our selected MSK architecture.

---

# 62. RDS Metrics

RDS monitoring should include:

```text
CPU
FreeableMemory
DatabaseConnections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
FreeStorageSpace
NetworkThroughput
```

These are AWS service metrics and are typically available through CloudWatch.

Prometheus can consume selected AWS metrics through the chosen AWS monitoring integration.

Do not install a random exporter simply because the metric exists in CloudWatch.

---

# 63. Redis Metrics

Monitor:

```text
Memory
CPU
Connections
Cache hits
Cache misses
Evictions
Replication health
Network
Commands
```

Again, use the supported ElastiCache/Prometheus integration appropriate to the deployment.

---

# 64. ALB Metrics

Important:

```text
RequestCount
HTTPCode_ELB_5XX_Count
HTTPCode_Target_5XX_Count
TargetResponseTime
HealthyHostCount
UnHealthyHostCount
```

These are critical when troubleshooting:

```text
Users receiving 5xx
```

---

# 65. EKS Metrics

Monitor:

```text
Node CPU
Node memory
Node filesystem
Pod CPU
Pod memory
Pod restarts
Pod availability
Deployment availability
DaemonSet health
StatefulSet health
```

---

# 66. Prometheus Self-Monitoring

Prometheus must monitor itself.

Important metrics:

```text
prometheus_tsdb_head_series
prometheus_tsdb_head_samples_appended_total
prometheus_target_scrapes_exceeded_sample_limit_total
prometheus_rule_evaluation_failures_total
prometheus_engine_query_duration_seconds
```

Also monitor:

```text
Disk
Memory
CPU
WAL
Scrape failures
```

---

# 67. Prometheus Scrape Failure

Alert concept:

```promql
up == 0
```

But don't create one generic critical alert for every target.

Different targets have different business criticality.

For example:

```text
order-service down
```

may be critical.

While:

```text
recommendation-service down
```

may be lower severity.

Alert severity should reflect actual business impact.

---

# 68. Example Service Availability Rule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: order-service-alerts
  namespace: monitoring

spec:
  groups:
    - name: order-service

      rules:
        - alert: OrderServiceDown

          expr: |
            up{
              job="order-service"
            } == 0

          for: 5m

          labels:
            severity: critical
            service: order-service

          annotations:
            summary: "Order Service is down"

            description: |
              Prometheus has been unable to scrape
              order-service for more than 5 minutes.
```

---

# 69. CPU Alert

Example:

```yaml
- alert: OrderServiceHighCPU

  expr: |
    (
      sum by (pod) (
        rate(
          container_cpu_usage_seconds_total{
            namespace="ecommerce",
            pod=~"order-service-.*"
          }[5m]
        )
      )
    ) > 0.8

  for: 10m

  labels:
    severity: warning
    service: order-service

  annotations:
    summary: "Order Service CPU is high"
```

The expression should ideally be compared with CPU requests or limits rather than a raw absolute value when building a reusable platform rule.

---

# 70. Memory Alert

Example concept:

```promql
container_memory_working_set_bytes
```

compared with:

```promql
kube_pod_container_resource_limits
```

The goal is to identify:

```text
Memory usage / memory limit
```

rather than simply saying:

```text
Memory > 1Gi
```

because different services have different resource limits.

---

# 71. Prometheus Alert Philosophy

Avoid alerting on everything.

Bad:

```text
CPU 60%
```

every five minutes.

This creates:

```text
Alert fatigue
```

Better:

```text
CPU saturation
+
sustained duration
+
service impact
```

Example:

```text
CPU > 90%
for 10 minutes
AND
request latency increasing
```

This is much more actionable.

---

# 72. Alert Labels

Standardize:

```yaml
labels:
  severity: critical
  team: platform
  service: order-service
  environment: prod
```

This helps Alertmanager route alerts.

Later:

```text
critical -> PagerDuty / on-call
warning -> Slack / email
info -> dashboard
```

---

# 73. PrometheusRule Repository

Recommended:

```text
gitops/
└── monitoring/
    ├── prometheus/
    │   ├── servicemonitors/
    │   ├── rules/
    │   │   ├── platform-rules.yaml
    │   │   ├── application-rules.yaml
    │   │   ├── kafka-rules.yaml
    │   │   └── database-rules.yaml
    │   └── values-prod.yaml
```

Rules are reviewed like application code.

---

# 74. GitOps Flow

```text
Developer
   |
   v
Git PR
   |
   v
Review
   |
   v
Merge
   |
   v
ArgoCD
   |
   v
Prometheus Operator
   |
   v
PrometheusRule
   |
   v
Prometheus
```

This prevents manual production monitoring changes from becoming invisible configuration drift.

---

# 75. ArgoCD Application

Conceptual:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: monitoring

spec:
  project: platform

  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    path: infrastructure/monitoring/prometheus

  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring

  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

For production monitoring infrastructure, automatic pruning should be evaluated carefully.

---

# 76. Monitoring Namespace Resource Management

Prometheus itself consumes resources.

Therefore configure:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 2Gi

  limits:
    cpu: "2"
    memory: 4Gi
```

Also monitor:

```text
Prometheus CPU
Prometheus memory
Prometheus disk
Prometheus query latency
Prometheus series count
```

---

# 77. Prometheus OOM

Symptoms:

```text
Prometheus pod
   |
   X OOMKilled
```

Possible causes:

```text
High cardinality
Too many series
Long retention
Large queries
Insufficient memory
Metric churn
```

Do not immediately increase memory.

First investigate:

```promql
prometheus_tsdb_head_series
```

and inspect expensive metrics.

---

# 78. High Cardinality Incident

Suppose developer adds:

```text
orderId
```

as a metric label.

Traffic:

```text
1 million orders
```

Potentially:

```text
1 million unique label values
```

Prometheus memory increases significantly.

Symptoms:

```text
Prometheus memory high
Queries slow
Compactions slow
OOMKilled
```

Solution:

```text
Remove high-cardinality label
```

Use logs/traces for individual IDs.

---

# 79. Metric Naming Convention

Prefer:

```text
orders_created_total
orders_failed_total
payment_requests_total
inventory_reservations_total
```

For durations:

```text
payment_processing_duration_seconds
```

For gauges:

```text
inventory_available_items
```

Avoid inconsistent names across services.

---

# 80. Platform Metric Standards

Every service should provide:

```text
HTTP
JVM
Process
Database
Messaging
Business
```

metrics where applicable.

Create a standard service template.

Example:

```text
service-template/
|
├── src/
├── Dockerfile
├── helm/
├── .github/
└── monitoring/
    ├── servicemonitor.yaml
    └── prometheusrule.yaml
```

This allows every new service to become observable by default.

---

# 81. Standard Service Helm Values

Example:

```yaml
monitoring:
  enabled: true

  serviceMonitor:
    enabled: true

    interval: 30s

    path: /actuator/prometheus

    labels:
      release: kube-prometheus-stack
```

The actual selector labels must match the installed Prometheus Operator configuration.

---

# 82. Service Template

Helm template:

```yaml
{{- if .Values.monitoring.serviceMonitor.enabled }}

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: {{ include "service.fullname" . }}

spec:
  selector:
    matchLabels:
      app: {{ include "service.name" . }}

  endpoints:
    - port: http
      path: {{ .Values.monitoring.serviceMonitor.path }}
      interval: {{ .Values.monitoring.serviceMonitor.interval }}

{{- end }}
```

Now every service can enable monitoring through values.

---

# 83. Why This Matters for 21 Services

Without reusable templates:

```text
21 services
×
manual monitoring configuration
```

creates inconsistency.

With templates:

```text
Standard Helm Chart
       |
       +--> Service
       +--> ServiceMonitor
       +--> Alerts
       +--> Dashboard labels
```

Every new service follows the same monitoring standard.

This becomes important in Phase 19 when we onboard all 21 services systematically.

---

# 84. Prometheus Security

Prometheus contains infrastructure information.

Protect:

```text
Prometheus UI
Prometheus API
Metrics endpoints
Remote write endpoints
```

Do not expose Prometheus publicly without a deliberate authentication and network security design.

Prefer:

```text
Internal ClusterIP
+
private access
+
identity-aware access where needed
```

---

# 85. NetworkPolicy for Monitoring

Example concept:

```text
monitoring namespace
        |
        | scrape
        v
ecommerce namespace
```

NetworkPolicy should allow:

```text
Prometheus -> application metrics
```

while preventing unnecessary access.

---

# 86. Prometheus + Secrets

Prometheus should not need application secrets simply to scrape `/actuator/prometheus`.

Metrics endpoints should not expose:

```text
DB password
AWS credentials
API tokens
JWT secrets
```

Review metric labels and metric values carefully.

---

# 87. What Prometheus Should Not Store

Do not use Prometheus as:

```text
Log database
Trace database
Business transaction database
Customer data store
```

Prometheus stores metrics.

Use:

```text
ELK -> logs
Tracing backend -> traces
RDS -> business data
S3 -> objects
```

---

# 88. Production Monitoring Layers

Our observability architecture:

```text
                   Users
                     |
                     v
                  ALB/WAF
                     |
                     v
                    EKS
                     |
        +------------+------------+
        |            |            |
        v            v            v
     Metrics        Logs        Traces
        |            |            |
        v            v            v
  Prometheus         ELK       Tracing
        |                         |
        +------------+------------+
                     |
                     v
                  Grafana
                     |
                     v
                Alertmanager
```

---

# 89. Monitoring Data Flow

Application:

```text
Order Service
     |
     +--> /actuator/prometheus
             |
             v
         Prometheus
             |
             v
           Grafana
```

Logs:

```text
Order Service
     |
     v
stdout
     |
     v
Fluent Bit
     |
     v
Logstash
     |
     v
Elasticsearch
```

Traces:

```text
Order Service
     |
     v
OpenTelemetry
     |
     v
Tracing backend
```

Each signal has a different purpose.

---

# 90. Production Prometheus Checklist

```text
Infrastructure
[ ] Monitoring namespace
[ ] Helm chart version pinned
[ ] Prometheus resources defined
[ ] Persistent storage configured
[ ] StorageClass validated
[ ] Retention configured
[ ] HA requirements assessed

Kubernetes
[ ] kube-state-metrics
[ ] node-exporter
[ ] Pod discovery
[ ] Service discovery
[ ] RBAC
[ ] NetworkPolicy

Applications
[ ] 21 services expose metrics
[ ] ServiceMonitor configured
[ ] Standard labels
[ ] Business metrics
[ ] JVM metrics
[ ] HTTP metrics
[ ] Database metrics
[ ] Kafka metrics

Security
[ ] Prometheus private
[ ] Metrics endpoints protected
[ ] No secrets in metrics
[ ] NetworkPolicy
[ ] RBAC

Operations
[ ] Target monitoring
[ ] Cardinality monitoring
[ ] Prometheus resource monitoring
[ ] Disk monitoring
[ ] Scrape failure alerts
[ ] Recording rules
[ ] Alert rules

GitOps
[ ] Prometheus values in Git
[ ] ServiceMonitors in Git
[ ] PrometheusRules in Git
[ ] ArgoCD manages monitoring
[ ] Production changes reviewed
```

---

# 91. Production Troubleshooting Runbook

## Problem

```text
Order Service metrics missing
```

### Step 1

Check application:

```bash
kubectl get pods -n ecommerce
```

### Step 2

Check Service:

```bash
kubectl get svc order-service -n ecommerce
```

### Step 3

Check endpoint:

```bash
kubectl run curl-test \
  --rm \
  -it \
  --image=curlimages/curl \
  -n ecommerce \
  -- \
  curl http://order-service:8080/actuator/prometheus
```

### Step 4

Check ServiceMonitor:

```bash
kubectl get servicemonitor \
  -n monitoring
```

### Step 5

Describe:

```bash
kubectl describe servicemonitor \
  order-service \
  -n monitoring
```

### Step 6

Check Prometheus target.

### Step 7

Check NetworkPolicy.

### Step 8

Check Prometheus logs:

```bash
kubectl logs \
  -n monitoring \
  <prometheus-pod>
```

---

# 92. Problem: Prometheus OOMKilled

Check:

```bash
kubectl get pod \
  -n monitoring
```

Then:

```bash
kubectl describe pod \
  -n monitoring \
  <prometheus-pod>
```

Look for:

```text
OOMKilled
```

Then investigate:

```promql
prometheus_tsdb_head_series
```

and:

```text
High cardinality metrics
Large queries
Retention
Scrape targets
Metric churn
```

Only then decide whether to increase resources.

---

# 93. Problem: Prometheus Disk Full

Symptoms:

```text
PVC usage high
WAL pressure
Prometheus instability
```

Investigate:

```text
Retention
Ingestion rate
Series count
Compaction
Unexpected metric growth
```

Possible solutions:

```text
Reduce unnecessary metrics
Reduce retention
Increase volume
Move long-term storage
Remove high-cardinality metrics
```

Do not simply increase disk without understanding why usage increased.

---

# 94. Problem: Target UP but Application Is Broken

This is important.

Prometheus says:

```text
up = 1
```

but customers report:

```text
Orders failing
```

Why?

Because:

```text
up = scrape endpoint reachable
```

It does not mean:

```text
business workflow healthy
```

Therefore we need:

```text
Technical metrics
+
Business metrics
```

Example:

```promql
rate(orders_created_total[5m])
```

may show:

```text
0
```

while:

```text
up{job="order-service"}
```

is:

```text
1
```

This indicates the service is reachable but may not be functioning correctly.

---

# 95. Monitoring Golden Signals

For every critical service:

```text
Traffic
Errors
Latency
Saturation
```

Example Order Service dashboard:

```text
Requests/sec
5xx %
p95 latency
p99 latency
CPU
Memory
DB connections
Kafka lag
Pod restarts
Orders/min
```

This should be the baseline dashboard.

---

# 96. Prometheus Production Architecture

Final architecture:

```text
                         AWS EKS
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
 Application Pods     Node Exporter      kube-state-metrics
        |                   |                   |
        +-------------------+-------------------+
                            |
                            v
                       Prometheus
                       /         \
                      /           \
                     v             v
              Recording Rules   Alert Rules
                     |             |
                     +------+------+
                            |
                            v
                         Grafana
                            |
                            v
                       Alertmanager
```

Infrastructure metrics:

```text
RDS
MSK
Redis
ALB
AWS infrastructure
```

are integrated through appropriate AWS metrics pipelines.

---

# 97. Production Design Principle

Prometheus should answer:

```text
"What is happening?"
```

Grafana should answer:

```text
"Can I understand it visually?"
```

Alertmanager should answer:

```text
"Who needs to know?"
```

ELK should answer:

```text
"What exactly happened?"
```

Tracing should answer:

```text
"Where did the request spend time?"
```

Together:

```text
Metrics
+
Logs
+
Traces
```

form our production observability platform.

---

# 98. Hands-On Implementation Order

Implement in this sequence:

```text
1. Create monitoring namespace

2. Install kube-prometheus-stack

3. Configure Prometheus resources

4. Configure persistent storage

5. Enable kube-state-metrics

6. Enable node-exporter

7. Verify Prometheus targets

8. Configure Order Service metrics

9. Create Order Service ServiceMonitor

10. Verify /actuator/prometheus

11. Verify Prometheus target

12. Add remaining services

13. Add business metrics

14. Add recording rules

15. Add PrometheusRules

16. Build Grafana dashboards

17. Configure Alertmanager

18. Test alerts

19. Perform failure simulations

20. Document runbooks
```

---

# 99. Final Prometheus Validation

Run:

```bash
kubectl get pods -n monitoring
```

Expected components include the Prometheus Operator stack components appropriate to the selected chart configuration.

Check:

```bash
kubectl get servicemonitor -A
```

Check:

```bash
kubectl get prometheusrule -A
```

Check application:

```bash
kubectl get pods -n ecommerce
```

Check metrics:

```bash
kubectl run curl-test \
  --rm \
  -it \
  --image=curlimages/curl \
  -n ecommerce \
  -- \
  curl http://order-service:8080/actuator/prometheus
```

Then verify in Prometheus:

```text
Status
  |
  +--> Targets
  |
  +--> Rules
  |
  +--> Service Discovery
```

Finally query:

```promql
up
```

Then:

```promql
up{namespace="ecommerce"}
```

Then:

```promql
rate(
  http_server_requests_seconds_count[5m]
)
```

---

# 100. Final Architecture

Our production monitoring flow is now:

```text
                    21 Microservices
                           |
                           v
                 /actuator/prometheus
                           |
                           v
                      Prometheus
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
       Recording       Alert Rules    Time Series
         Rules             |             |
                           v             v
                      Alertmanager     Grafana
                           |             |
                           v             v
                         On-call      Dashboards
```

Infrastructure:

```text
EKS Nodes
   |
   v
Node Exporter
   |
   v
Prometheus

Kubernetes
   |
   v
kube-state-metrics
   |
   v
Prometheus

AWS Services
   |
   v
CloudWatch / AWS metrics integration
   |
   v
Prometheus / Grafana
```

---

# 101. What We Have Achieved

We now have the foundation for production metrics monitoring across the platform.

We can monitor:

```text
21 microservices
EKS nodes
Kubernetes workloads
Deployments
Pods
CPU
Memory
Disk
HTTP traffic
HTTP errors
HTTP latency
JVM
Database connections
Kafka
Redis
RDS
ALB
Business metrics
```

We also established:

```text
ServiceMonitor
PrometheusRule
Recording Rules
Resource planning
Persistent storage
High availability considerations
Cardinality management
NetworkPolicy
RBAC
GitOps
Troubleshooting
Monitoring standards
```

---

# 102. Next Chapter

## 119-Grafana-Production-Dashboards.md

The next chapter will build the actual production dashboards on top of Prometheus.

We will create dashboards for:

```text
1. Executive / Business Overview

2. EKS Cluster

3. Kubernetes Nodes

4. Kubernetes Pods

5. 21 Microservices

6. Order Service

7. Payment Service

8. Inventory Service

9. Kafka

10. RDS PostgreSQL

11. Redis

12. ALB

13. SQS

14. Application Golden Signals

15. SLO / SLA

16. Production Incident Dashboard
```

We will also design:

```text
RED dashboards
USE dashboards
Business dashboards
Kafka lag dashboards
Database dashboards
Kubernetes capacity dashboards
Service dependency dashboards
```

and connect them to the alerting architecture.

```
```
