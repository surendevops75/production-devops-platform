# Chapter 187 — Monitoring Templates

Phase 20 — Platform Engineering

## Part 1 — Monitoring Platform Architecture and Standards

### 1. Objective

In this chapter, we will build a reusable monitoring foundation for the enterprise e-commerce platform containing 21 microservices.

The objective is not to install Prometheus and Grafana separately for every microservice. Instead, the platform team creates standardized monitoring templates that every service can use.

When a developer creates a new microservice, the service should automatically follow the organization's monitoring standards:

- Expose application metrics in a standard format.
- Provide health and readiness endpoints.
- Use consistent metric names and labels.
- Allow Prometheus to discover and scrape the service.
- Provide a standard Grafana dashboard.
- Send application logs to the centralized logging platform.
- Support troubleshooting using metrics, logs, Kubernetes events, and deployment information.
- Avoid exposing sensitive information through metrics or logs.

At the end of this chapter, all 21 microservices should follow the same monitoring integration pattern.

### 2. Monitoring architecture

Our platform uses:

- Prometheus — metrics collection and storage.
- Grafana — dashboards and visualization.
- Alertmanager — alert routing and grouping.
- Node Exporter — Linux node-level metrics.
- kube-state-metrics — Kubernetes object state metrics.
- ELK Stack — centralized log storage and analysis.
- Logstash — log processing and transformation.

We are not introducing Jaeger or OpenTelemetry as part of this implementation. They are not part of the current project stack.

[Logs - binbash Leverage™](https://images.openai.com/static-rsc-4/pF1PcapSuKvWqbOXk-vpdGjz2IWTaaw9iHT1MUcprvHQSkDizeKk86cZqPdb_t8povy5NSlEd5BxG7sJMP4JOdr-QvahyP4E0cuna4gBR5lZr_djrszr_HMiWqmkBIFJ0ppRvz3cW2LhYplZyheYoxKFTHjhXEQJYCI41i33KQE?purpose=inline)

[leverage.binbash.co](https://leverage.binbash.co/user-guide/ref-architecture-aws/features/monitoring/logs/?utm_source=chatgpt.com)

The architecture has two primary data paths.

Metrics path

```
21 Microservices
       |
       | /actuator/prometheus
       |
       v
   Prometheus
       |
       +------> Grafana
       |
       +------> Alertmanager

Kubernetes Nodes
       |
       v
  Node Exporter
       |
       v
   Prometheus

Kubernetes API
       |
       v
 kube-state-metrics
       |
       v
   Prometheus
```

Logging path

```
21 Microservices
       |
       | Structured JSON logs
       v
 Container Runtime
       |
       v
 Log Collection Agent
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

The log collection agent must be selected and configured for the cluster. For example, Fluent Bit can collect container logs and forward them to Logstash. Logstash then parses and enriches the records before sending them to Elasticsearch.

Metrics and logs serve different purposes:

| Monitoring data    | Main question it answers                  |
| ------------------ | ----------------------------------------- |
| Metrics            | Is the service healthy?                   |
| Logs               | What happened inside the service?         |
| Kubernetes events  | What happened to the workload or cluster? |
| Grafana dashboards | How is the system behaving over time?     |
| Alerts             | Which condition requires investigation?   |

### 3. Ownership and responsibilities

Monitoring is not exclusively a DevOps responsibility. Each team owns a different part of the monitoring lifecycle.

| Team                   | Responsibility                                                       |
| ---------------------- | -------------------------------------------------------------------- |
| Application developers | Expose application metrics and structured logs                       |
| Platform/DevOps        | Deploy Prometheus, Grafana, exporters and shared templates           |
| SRE/Operations         | Define service objectives, alert routing and incident response       |
| Database team          | Monitor database health, connections, storage and performance        |
| Security team          | Review access, sensitive data exposure and retention                 |
| Service owner          | Validate that dashboards and metrics represent the service correctly |

Example: Order Service

The developer implements application metrics such as:

- Number of orders created.
- Number of failed order requests.
- Order processing duration.
- Database operation duration.

The platform team provides the Prometheus configuration and reusable Grafana dashboard.

The SRE team determines which conditions should page the on-call engineer.

The database team investigates database-side problems when the application metrics show database latency.

The important distinction is that the platform team provides the monitoring mechanism, but the service owner remains responsible for the correctness of the service's metrics.

### 4. Repository structure

Create a separate repository for shared monitoring assets.

Repository:

```
company-ecommerce/platform-monitoring
```

Recommended structure:

```
platform-monitoring/
│
├── README.md
│
├── prometheus/
│   ├── values/
│   │   ├── dev-values.yaml
│   │   ├── qa-values.yaml
│   │   ├── staging-values.yaml
│   │   └── prod-values.yaml
│   │
│   ├── scrape-configs/
│   │   ├── kubernetes-pods.yaml
│   │   ├── kubernetes-services.yaml
│   │   └── external-targets.yaml
│   │
│   └── recording-rules/
│       └── service-recording-rules.yaml
│
├── exporters/
│   ├── node-exporter/
│   │   ├── values.yaml
│   │   └── README.md
│   │
│   └── kube-state-metrics/
│       ├── values.yaml
│       └── README.md
│
├── grafana/
│   ├── datasources/
│   │   └── prometheus.yaml
│   │
│   ├── dashboards/
│   │   ├── kubernetes-cluster.json
│   │   ├── kubernetes-node.json
│   │   ├── microservice-overview.json
│   │   └── order-service.json
│   │
│   └── provisioning/
│       ├── dashboards.yaml
│       └── datasources.yaml
│
├── logging/
│   ├── logstash/
│   │   ├── pipelines.yml
│   │   └── application.conf
│   │
│   └── index-management/
│       └── index-lifecycle-policy.json
│
├── templates/
│   ├── servicemonitor/
│   ├── podmonitor/
│   ├── grafana-dashboard/
│   └── monitoring-values/
│
├── scripts/
│   ├── validate-metrics.sh
│   ├── check-prometheus-targets.sh
│   └── validate-log-format.sh
│
└── docs/
    ├── monitoring-standards.md
    ├── metric-naming.md
    ├── dashboard-standards.md
    ├── logging-standards.md
    └── troubleshooting.md
```

This repository contains reusable monitoring configuration. It should not become a place where developers manually copy unrelated application configurations.

### 5. Monitoring naming standards

Before configuring Prometheus, define a naming convention.

Without naming standards, different developers may expose the same business metric using different names.

For example:

```
orders_created
order_count
total_orders
orders_total
```

These names may describe the same business activity but create unnecessary inconsistency.

Use a standard convention:

```
<application>_<operation>_<measurement>_<unit>
```

Examples:

```
orders_created_total
orders_failed_total
http_server_requests_seconds_count
http_server_requests_seconds_sum
jvm_memory_used_bytes
jvm_gc_pause_seconds_sum
```

Use Prometheus conventions:

- Counters end in `_total`.
- Durations use seconds.
- Sizes use bytes.
- Ratios are unitless.
- Gauges represent values that can increase or decrease.

Avoid putting dynamic values into metric names.

Bad:

```
orders_created_service_123
orders_created_service_124
orders_created_service_125
```

Good:

```
orders_created_total{service="order-service"}
```

The service identity belongs in a label, not in the metric name.

### 6. Standard labels

All microservices should expose a consistent set of identifying labels.

Example:

```
environment="production"
namespace="ecommerce"
service="order-service"
team="commerce"
```

Prometheus target labels can supply these values centrally.

Example target:

```
labels:
  environment: production
  namespace: ecommerce
  team: commerce
```

Avoid labels such as:

```
customer_id
user_id
order_id
request_id
session_id
email
```

These values can create extremely high cardinality and may expose sensitive information.

For request-level investigation, use logs with controlled identifiers rather than creating a new Prometheus time series for every request.

### 7. Define the service monitoring contract

Every microservice must satisfy a common monitoring contract before it is promoted to production.

Example contract for Order Service:

| Requirement         | Expected implementation                    |
| ------------------- | ------------------------------------------ |
| Application metrics | Prometheus-compatible endpoint             |
| Liveness            | `/actuator/health/liveness`                |
| Readiness           | `/actuator/health/readiness`               |
| Metrics             | `/actuator/prometheus`                     |
| Service identity    | `order-service`                            |
| Environment         | Injected through deployment configuration  |
| Logs                | Structured JSON                            |
| Dashboard           | Standard microservice dashboard            |
| Ownership           | Declared service team                      |
| Sensitive data      | No credentials or personal data in metrics |

The exact endpoints depend on the application framework. The table shows the Spring Boot implementation used in our example.

### 8. Monitoring environment separation

We have four deployment environments:

```
DEV
 |
 v
QA
 |
 v
STAGING
 |
 v
PRODUCTION
```

Monitoring configuration must be environment-aware.

For example:

```
DEV:
  lower retention
  smaller resource allocation
  fewer dashboard alerts

QA:
  test metrics and dashboard behavior
  validate application instrumentation

STAGING:
  production-like scrape configuration
  performance and load testing

PRODUCTION:
  high availability
  controlled access
  production retention
  alert routing
  capacity monitoring
```

Do not use production monitoring credentials in development or QA.

Also, do not assume that every environment needs a completely separate monitoring stack. A shared monitoring platform can be used where isolation, access control, network connectivity, and failure-domain requirements permit it. Production monitoring should not become unavailable simply because a development workload overloads a shared monitoring system.

## Part 2 — Instrumenting Microservices

### 9. Application metrics with Spring Boot

Our Order Service is a Java Spring Boot application.

Spring Boot Actuator exposes application health and operational information. Micrometer provides the metrics instrumentation, and the Prometheus registry exposes those metrics in Prometheus format.

Add dependencies to `pom.xml`:

```
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

</dependencies>
```

The actual Spring Boot and Micrometer versions must be compatible with the application's dependency-management configuration.

### 10. Configure application properties

File:

```
src/main/resources/application.yaml
```

Configuration:

```
server:
  port: 8080

spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include:
          - health
          - prometheus
          - info

  endpoint:
    health:
      probes:
        enabled: true

      show-details: never

  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${APP_ENV:dev}
      team: commerce

  prometheus:
    metrics:
      export:
        enabled: true
```

Important production considerations:

1. Do not expose every Actuator endpoint publicly.
2. Keep administrative endpoints restricted.
3. Expose metrics only through an internal service path.
4. Do not return database credentials or internal secrets in health responses.
5. Confirm that the configured endpoint is reachable from Prometheus.
6. Use Kubernetes probes for application lifecycle checks, not as a substitute for complete business monitoring.

### 11. Verify the metrics endpoint

Start the Order Service locally:

```
mvn spring-boot:run
```

Check health:

```
curl http://localhost:8080/actuator/health
```

Check Prometheus metrics:

```
curl http://localhost:8080/actuator/prometheus
```

Expected output will contain metrics in Prometheus exposition format. For example:

```
# HELP process_cpu_usage The "recent cpu usage" for the Java Virtual Machine process
# TYPE process_cpu_usage gauge
process_cpu_usage 0.12

# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="G1 Eden Space"} 1.2345678E7
```

The exact metrics depend on the JVM, application framework, Micrometer version, and instrumentation.

If the endpoint returns `404`, check:

- Is Actuator installed?
- Is the Prometheus registry dependency present?
- Is the endpoint exposed?
- Is the application using the expected port?
- Did the application start with the correct profile?

If the endpoint returns `401` or `403`, check application security configuration. Prometheus needs authorized internal access, but the endpoint should not be made publicly accessible simply to solve a scrape failure.

### 12. Create business metrics

Framework metrics tell us about HTTP requests, JVM memory, and process behavior. They do not necessarily tell us whether an important business operation is succeeding.

For Order Service, we should expose business-level metrics.

Example Java service:

```
@Service
public class OrderService {

    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;

    public OrderService(MeterRegistry registry) {

        this.orderCreatedCounter =
            Counter.builder("orders.created")
                .description("Total successfully created orders")
                .register(registry);

        this.orderFailedCounter =
            Counter.builder("orders.failed")
                .description("Total failed order creation attempts")
                .register(registry);
    }

    public void createOrder() {

        try {

            // Actual order creation logic

            orderCreatedCounter.increment();

        } catch (Exception exception) {

            orderFailedCounter.increment();

            throw exception;
        }
    }
}
```

Micrometer translates the metric names into Prometheus-compatible names. For example, the counter may appear as:

```
orders_created_total
```

This example demonstrates instrumentation only. In a real application, count an order as created only after the defined business transaction has succeeded. Decide whether retries, rejected requests, duplicate submissions, and asynchronous failures should count separately.

Do not add exception messages or order IDs as labels.

### 13. Measure business operation duration

Suppose Order Service calls Inventory Service before confirming an order.

We want to measure how long that operation takes.

Example:

```
@Service
public class InventoryClient {

    private final Timer inventoryCheckTimer;

    public InventoryClient(MeterRegistry registry) {

        this.inventoryCheckTimer =
            Timer.builder("inventory.check.duration")
                .description("Time taken to check inventory")
                .register(registry);
    }

    public void checkInventory() {

        inventoryCheckTimer.record(() -> {

            // Call Inventory Service
            // Validate inventory response

        });
    }
}
```

Prometheus exposes timer measurements as related count and sum metrics, and may expose histogram buckets if histogram publishing is enabled.

A timer helps answer:

- Is the inventory call becoming slower?
- Is the average duration increasing?
- Are a small number of calls taking unusually long?
- Did a deployment change the latency profile?

For latency percentiles, configure histogram buckets and validate the resulting time series. Do not assume that a percentile is available merely because a timer exists.

### 14. Configure histogram metrics

For a service that needs latency percentiles, configure the relevant metric:

```
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
```

The Prometheus histogram can be queried using `histogram_quantile`.

Example:

```
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

This estimates the 95th percentile request duration across the selected service series.

The query assumes the `service` label exists. If your scrape configuration uses a different label, adjust the grouping accordingly.

### 15. Standard metrics for all microservices

Each service should expose a common baseline.

| Metric category     | Example                              | Purpose             |
| ------------------- | ------------------------------------ | ------------------- |
| Request count       | `http_server_requests_seconds_count` | Request volume      |
| Request duration    | `http_server_requests_seconds_sum`   | Total request time  |
| Error count         | HTTP status labels                   | Request failures    |
| JVM memory          | `jvm_memory_used_bytes`              | Java memory usage   |
| JVM threads         | `jvm_threads_live_threads`           | Thread activity     |
| Process CPU         | `process_cpu_usage`                  | Application CPU     |
| Business success    | `orders_created_total`               | Business operations |
| Business failure    | `orders_failed_total`                | Business failures   |
| Dependency duration | `inventory_check_duration`           | Downstream latency  |

Not every service will have the same business metrics. The platform standard defines the minimum common set, while the application team defines metrics specific to its business function.

### 16. Health endpoints and Kubernetes probes

Health endpoints and Prometheus metrics have different jobs.

- Liveness tells Kubernetes whether the application process should be restarted.
- Readiness tells Kubernetes whether the application should receive traffic.
- Metrics help operators understand application behavior.

Example deployment probe configuration:

```
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

Avoid making liveness depend on every downstream dependency. If the database is temporarily unavailable, restarting every application pod may make an incident worse.

Readiness can incorporate dependency checks when that matches the application's traffic policy. The application team must decide which dependencies are essential for serving requests.

### 17. Kubernetes Service for metrics

Prometheus needs a stable Kubernetes service endpoint to scrape the application.

Example:

```
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
    monitoring: enabled
spec:
  type: ClusterIP

  selector:
    app: order-service

  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
```

The metrics endpoint is available internally through:

```
http://order-service.ecommerce.svc.cluster.local:8080/actuator/prometheus
```

This DNS name is for internal cluster communication. It should not be exposed through a public ingress just for Prometheus scraping.

### 18. Monitoring labels in the Deployment

Example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service
        monitoring: enabled
        team: commerce

    spec:
      containers:
        - name: order-service
          image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/order-service:IMAGE_TAG

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
```

The image reference is illustrative. Production GitOps should promote an immutable image digest rather than relying only on a mutable tag.

### 19. Create a reusable monitoring contract

Instead of allowing every application team to invent its own labels, define a standard contract in the platform repository.

File:

```
docs/monitoring-standards.md
```

Example:

```
# Microservice Monitoring Contract

Every production microservice must provide:

1. A Prometheus-compatible metrics endpoint.
2. Kubernetes liveness and readiness probes.
3. Standard application identity labels.
4. Structured application logs.
5. A service owner.
6. A Grafana dashboard or an approved shared dashboard.
7. A documented dependency list.
8. A documented business success and failure metric.

Required labels:

- environment
- namespace
- service
- team

Forbidden high-cardinality labels:

- user_id
- customer_id
- order_id
- request_id
- session_id

Metrics endpoint:

/actuator/prometheus

The endpoint must be accessible internally and must not expose
credentials or sensitive application information.
```

This contract becomes part of the microservice onboarding process in Chapter 192.

### 20. Practical validation script

Create:

```
scripts/validate-metrics.sh
```

```
#!/bin/bash

set -euo pipefail

SERVICE_URL="${1:-http://localhost:8080}"
METRICS_PATH="/actuator/prometheus"

echo "Checking metrics endpoint..."

HTTP_CODE=$(curl \
    --silent \
    --output /tmp/service-metrics.txt \
    --write-out "%{http_code}" \
    "${SERVICE_URL}${METRICS_PATH}")

if [ "$HTTP_CODE" != "200" ]; then
    echo "ERROR: Metrics endpoint returned HTTP ${HTTP_CODE}"
    exit 1
fi

if ! grep -q "^# TYPE " /tmp/service-metrics.txt; then
    echo "ERROR: Prometheus metric definitions not found"
    exit 1
fi

echo "Metrics endpoint is responding."

if grep -qiE \
    'password|secret|authorization|access_token' \
    /tmp/service-metrics.txt; then

    echo "WARNING: Review metrics output for possible sensitive data."
fi

echo "Metrics validation completed."
```

Run:

```
chmod +x scripts/validate-metrics.sh

./scripts/validate-metrics.sh \
  http://localhost:8080
```

The sensitive-data check is only a basic warning. It is not a substitute for a security review, and metric values or labels can contain sensitive information without matching these words.

## Part 3 — Prometheus Deployment and Kubernetes Discovery

### 21. How Prometheus discovers microservices

We have 21 microservices. We should not manually configure 21 static IP addresses in Prometheus.

Kubernetes pod IP addresses can change whenever pods are recreated, rescheduled, or scaled. Static targets would therefore require constant maintenance.

Instead, Prometheus should discover workloads through Kubernetes service discovery.

There are two common approaches:

1. Kubernetes-native scrape configuration.
2. Prometheus Operator using `ServiceMonitor` and `PodMonitor` resources.

For this project, we will use the Prometheus Operator approach because it provides reusable monitoring resources that fit naturally into our Helm and GitOps platform.

[实操案例：使用Prometheus Operator进行集群监控_prometheus operator监控已创建的容器-CSDN博客](https://images.openai.com/static-rsc-4/8bolHtH1uqsa62S6boMvTTdQLPlM90KE75oavuGTO4RkEd6UVJV5dayc2xxD7xePZt-CVLTdFlEBCn7MLpuBqyp73UXsiqC9JAyqoY-2UP60NBXxtfBqh5ngfTwcYcWE_dgVb98j4n4gI5iSXuw6r8jVdRnSp-5YnFiJ4EqLWp0?purpose=inline)

[blog.csdn.net](https://blog.csdn.net/RancherLabs/article/details/101014889?utm_source=chatgpt.com)

The discovery flow is:

```
Order Service Pods
       |
       v
Kubernetes Service
       |
       v
ServiceMonitor
       |
       v
Prometheus Operator
       |
       v
Prometheus Scrape Configuration
       |
       v
Prometheus TSDB
       |
       v
Grafana
```

The Operator watches the Kubernetes resources and generates the corresponding Prometheus scrape configuration.

### 22. Install Prometheus using Helm

We will use the community-maintained `kube-prometheus-stack` Helm chart as the deployment foundation.

It commonly packages:

- Prometheus Operator.
- Prometheus.
- Alertmanager.
- Grafana.
- Node Exporter.
- kube-state-metrics.
- Default Kubernetes monitoring resources.

The exact components enabled depend on the chart version and values.

First, create the namespace:

```
kubectl create namespace monitoring
```

Add the Helm repository:

```
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```

Update repository metadata:

```
helm repo update
```

Inspect available chart versions:

```
helm search repo prometheus-community/kube-prometheus-stack \
  --versions
```

For production, select and pin a tested chart version. Do not use an unreviewed latest version directly in the production GitOps configuration.

### 23. Create environment-specific values

File:

```
prometheus/values/prod-values.yaml
```

Illustrative starting configuration:

```
grafana:
  enabled: true

  persistence:
    enabled: true
    size: 20Gi

  resources:
    requests:
      cpu: 250m
      memory: 512Mi

    limits:
      cpu: "1"
      memory: 1Gi

prometheus:
  enabled: true

  prometheusSpec:
    replicas: 2

    retention: 15d

    retentionSize: 40GB

    scrapeInterval: 30s

    evaluationInterval: 30s

    resources:
      requests:
        cpu: "1"
        memory: 4Gi

      limits:
        cpu: "2"
        memory: 8Gi

    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 100Gi

alertmanager:
  enabled: true

  alertmanagerSpec:
    replicas: 2

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

This is a starting point, not a production sizing recommendation. Actual sizing must be based on:

- Number of active time series.
- Scrape interval.
- Number of scrape targets.
- Retention requirement.
- Query concurrency.
- Dashboard complexity.
- Alert evaluation workload.
- Storage performance.
- Availability and recovery requirements.

Also verify the values against the exact Helm chart version you pin. Helm chart values can change between releases.

### 24. Deploy the monitoring stack

For an initial non-GitOps lab deployment:

```
helm upgrade --install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version CHART_VERSION \
  --values prometheus/values/prod-values.yaml
```

Replace `CHART_VERSION` with the version approved by the platform team.

In the actual production project, the Helm release should be managed through the GitOps repository and ArgoCD, not through an engineer repeatedly running `helm upgrade` from a laptop.

Check resources:

```
kubectl get pods -n monitoring
```

Check services:

```
kubectl get svc -n monitoring
```

Check deployments:

```
kubectl get deployments -n monitoring
```

Check Prometheus custom resources:

```
kubectl get prometheus -n monitoring
```

Check ServiceMonitor resources:

```
kubectl get servicemonitor -A
```

### 25. Understand the Prometheus Operator resources

The main resources are:

| Resource       | Purpose                                              |
| -------------- | ---------------------------------------------------- |
| Prometheus     | Defines the Prometheus instance                      |
| ServiceMonitor | Selects Kubernetes Services to scrape                |
| PodMonitor     | Selects Pods directly                                |
| PrometheusRule | Defines recording and alerting rules                 |
| Alertmanager   | Defines alert routing and notification configuration |

We will focus on `ServiceMonitor` in this chapter.

Chapter 188 will cover reusable alert-rule templates and their rollout.

### 26. Create a ServiceMonitor for Order Service

File:

```
templates/servicemonitor/order-service.yaml
```

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: order-service
  namespace: ecommerce

  labels:
    release: monitoring
    team: commerce

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
      scrapeTimeout: 10s
```

Important details:

- `namespaceSelector` tells the ServiceMonitor where to find the Service.
- `selector.matchLabels` selects the Kubernetes Service by its labels.
- `port` refers to the named port on the Service, not directly to the container port number.
- `path` identifies the metrics endpoint.
- `interval` determines how frequently Prometheus scrapes the target.
- `scrapeTimeout` controls how long Prometheus waits for a scrape response.

The `release: monitoring` label is an example of a selector convention commonly used with Helm-installed Prometheus Operator stacks. Confirm that your installed Prometheus resource actually selects ServiceMonitors with this label. Otherwise, the ServiceMonitor can exist but never be discovered.

### 27. Apply and verify the ServiceMonitor

For a lab:

```
kubectl apply -f \
  templates/servicemonitor/order-service.yaml
```

Check the resource:

```
kubectl get servicemonitor \
  -n ecommerce
```

Describe it:

```
kubectl describe servicemonitor \
  order-service \
  -n ecommerce
```

Check Prometheus targets through the Prometheus UI.

Port-forward:

```
kubectl port-forward \
  -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090
```

Open:

```
http://localhost:9090/targets
```

Search for the Order Service target.

[Minibuilds | Deploying Prometheus to Kubernetes](https://images.openai.com/static-rsc-4/GKi3g3CNeWMvll2AGrGaTuwDxTM3UFxIwCBVA_KqscKeBrt-C8niPA_4YWWF06LccYnm9ZMF1Lhwvzo9ILkJroAN0uGcpD7rUuOUNmSq0B3zX3ffi7GdE_fBl33ZZfeAWBqLq88S8rPmivxEQi9gXh3vdBS18TxiQ8zzacY7Qys?purpose=inline)

[minibuilds.io](https://www.minibuilds.io/posts/prometheus-kubernetes/?utm_source=chatgpt.com)

Expected state:

```
order-service   UP
```

If the target is `DOWN`, do not immediately restart Prometheus. First inspect the target's error message and verify the service path, endpoint, labels, network connectivity, and authorization.

### 28. ServiceMonitor troubleshooting

Problem: ServiceMonitor exists but no target appears.

Check:

```
kubectl get servicemonitor \
  -n ecommerce \
  --show-labels
```

Check the Prometheus selector:

```
kubectl get prometheus \
  -n monitoring \
  -o yaml
```

Inspect:

```
serviceMonitorSelector:
```

And:

```
serviceMonitorNamespaceSelector:
```

The Prometheus resource must select the ServiceMonitor's labels and namespace.

Problem: Target appears but is DOWN.

Check the Service:

```
kubectl get svc \
  order-service \
  -n ecommerce \
  -o yaml
```

Check endpoints:

```
kubectl get endpoints \
  order-service \
  -n ecommerce
```

For newer Kubernetes troubleshooting, also inspect EndpointSlices:

```
kubectl get endpointslices \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Check the endpoint directly from a temporary diagnostic pod that has the required network access:

```
curl -v \
  http://order-service.ecommerce.svc.cluster.local:8080/actuator/prometheus
```

Possible causes:

- Service selector does not match the Pod labels.
- The Service port name does not match the ServiceMonitor endpoint port.
- Application is listening on a different port.
- Metrics endpoint is disabled.
- Application security rejects Prometheus.
- NetworkPolicy blocks traffic.
- The application pod is not ready.
- Prometheus is not selecting the ServiceMonitor.

### 29. ServiceMonitor versus PodMonitor

Use `ServiceMonitor` when your application has a Kubernetes Service that provides a stable endpoint.

Use `PodMonitor` when you intentionally want Prometheus to discover and scrape pods directly without using a Service.

For most of our HTTP microservices, prefer `ServiceMonitor`.

A PodMonitor may be useful for certain exporters or workloads where scraping individual pods is the intended model.

Do not create both resources for the same endpoint without a clear reason. Doing so may cause duplicate scraping and duplicate time series.

### 30. Reusable ServiceMonitor template

Instead of manually writing the same resource for 21 services, create a reusable Helm template.

File:

```
platform-monitoring/templates/servicemonitor.yaml
```

```
{{- if .Values.monitoring.serviceMonitor.enabled }}

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: {{ include "service.fullname" . }}
  namespace: {{ .Release.Namespace }}

  labels:
    release: {{ .Values.monitoring.serviceMonitor.release }}
    team: {{ .Values.team | quote }}

spec:
  namespaceSelector:
    matchNames:
      - {{ .Release.Namespace }}

  selector:
    matchLabels:
      app: {{ include "service.name" . }}

  endpoints:
    - port: {{ .Values.service.portName }}
      path: {{ .Values.monitoring.serviceMonitor.path }}
      interval: {{ .Values.monitoring.serviceMonitor.interval }}
      scrapeTimeout: {{ .Values.monitoring.serviceMonitor.scrapeTimeout }}

{{- end }}
```

This example assumes that the chart has the helper templates and values shown below. Integrate it into the actual shared Helm chart rather than copying it into an unrelated chart without adapting its helpers.

Values:

```
team: commerce

service:
  portName: http

monitoring:
  serviceMonitor:
    enabled: true
    release: monitoring
    path: /actuator/prometheus
    interval: 30s
    scrapeTimeout: 10s
```

A service can override the values when its endpoint differs:

```
monitoring:
  serviceMonitor:
    enabled: true
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

This makes the monitoring integration configurable without changing the reusable template.

### 31. Validate Helm output before deployment

Run:

```
helm lint ./charts/order-service
```

Render the templates:

```
helm template order-service \
  ./charts/order-service \
  --namespace ecommerce \
  --values values/prod.yaml
```

Inspect the generated ServiceMonitor:

```
helm template order-service \
  ./charts/order-service \
  --namespace ecommerce \
  --values values/prod.yaml \
  | grep -A30 "kind: ServiceMonitor"
```

For stronger validation, use Kubernetes schema validation and a test cluster that has the Prometheus Operator CRDs installed. A generic Kubernetes API server without those CRDs may reject the custom resource.

### 32. Roll out monitoring across the 21 services

The platform team should not enable monitoring for all services in one uncontrolled production change.

Use a phased rollout:

| Rollout stage | Services                                       | Validation                            |
| ------------- | ---------------------------------------------- | ------------------------------------- |
| Pilot         | Order Service                                  | Metrics endpoint and target discovery |
| Wave 1        | Auth, User, Customer, Product                  | Labels and dashboards                 |
| Wave 2        | Catalog, Inventory, Pricing, Promotion         | Business metrics                      |
| Wave 3        | Cart, Payment, Fraud, Shipping                 | Dependency metrics                    |
| Wave 4        | Delivery, Notification, Search, Recommendation | Scrape reliability                    |
| Wave 5        | Review, Tax, Invoice, Reporting                | Full monitoring coverage              |

At each stage:

1. Deploy the application instrumentation.
2. Confirm the metrics endpoint.
3. Deploy the ServiceMonitor.
4. Confirm the Prometheus target is UP.
5. Confirm the service labels.
6. Check the metrics for unexpected cardinality.
7. Validate the dashboard.
8. Record evidence in the rollout ticket.

A service should not be marked monitoring-ready merely because its ServiceMonitor YAML was merged.

### 33. Why we avoid static IP targets

Imagine Order Service has three pods:

```
order-service-7b9c   10.0.1.10
order-service-8c2d   10.0.2.20
order-service-9d3e   10.0.3.30
```

After a node replacement, Kubernetes might recreate the pods:

```
order-service-a1b2   10.0.4.40
order-service-b2c3   10.0.5.50
order-service-c3d4   10.0.6.60
```

A manually maintained Prometheus target list would become stale.

Kubernetes service discovery solves this by watching Kubernetes resources and discovering current endpoints dynamically.

### 34. Prometheus scrape interval strategy

Not every metric needs to be scraped every five seconds.

A reasonable starting strategy might be:

| Target                       | Example interval |
| ---------------------------- | ---------------- |
| Critical application metrics | 15–30 seconds    |
| General application metrics  | 30 seconds       |
| Node Exporter                | 30 seconds       |
| kube-state-metrics           | 30 seconds       |
| Infrequent external checks   | 60 seconds       |

The interval must be selected according to operational needs and ingestion capacity.

Shorter intervals increase the number of samples stored and the monitoring system's workload. They do not automatically improve incident detection if the alert evaluation and response process are not designed accordingly.

## Part 4 — Kubernetes Infrastructure Monitoring

### 35. Why application metrics are not enough

Suppose Order Service response time increases from 200 milliseconds to 3 seconds.

The application dashboard tells us that latency increased. It does not necessarily tell us why.

Possible causes include:

- Kubernetes node CPU saturation.
- Node memory pressure.
- Container CPU throttling.
- Pod restarts.
- Insufficient replicas.
- Database connection exhaustion.
- Network problems.
- A slow downstream microservice.

We therefore need infrastructure-level monitoring alongside application metrics.

Our baseline infrastructure monitoring consists of:

1. Node Exporter.
2. kube-state-metrics.
3. Kubelet and container metrics.
4. Kubernetes control-plane signals available through the EKS integration.
5. AWS service metrics where relevant.

### 36. Node Exporter architecture

Node Exporter collects operating-system and hardware-level metrics from Linux machines.

Examples:

- CPU usage.
- Memory usage.
- Disk utilization.
- Filesystem availability.
- Network traffic.
- Disk I/O.
- System load.
- Filesystem inode usage.

Architecture:

```
EKS Worker Node 1
    |
    +---- Node Exporter
    |
    +---- Kubernetes Pods

EKS Worker Node 2
    |
    +---- Node Exporter
    |
    +---- Kubernetes Pods

EKS Worker Node 3
    |
    +---- Node Exporter
    |
    +---- Kubernetes Pods

          |
          v
      Prometheus
          |
          v
        Grafana
```

Node Exporter normally runs as a DaemonSet so that one instance runs on each eligible Linux worker node.

### 37. Deploy Node Exporter

If we use `kube-prometheus-stack`, Node Exporter is commonly included and enabled by default.

Check:

```
kubectl get daemonset -n monitoring
```

Find the Node Exporter DaemonSet:

```
kubectl get daemonset \
  -n monitoring \
  | grep node-exporter
```

Check pods:

```
kubectl get pods \
  -n monitoring \
  -o wide \
  | grep node-exporter
```

Expected:

```
node-exporter-xxxxx   1/1   Running
node-exporter-yyyyy   1/1   Running
node-exporter-zzzzz   1/1   Running
```

The number of instances should correspond to the eligible nodes selected by the DaemonSet. Taints, tolerations, node selectors, operating-system type, and architecture can affect placement.

Do not install a second independent Node Exporter DaemonSet if the Helm stack already manages one.

### 38. Verify Node Exporter metrics

Port-forward a Node Exporter pod:

```
kubectl port-forward \
  -n monitoring \
  pod/NODE_EXPORTER_POD \
  9100:9100
```

Check:

```
curl http://localhost:9100/metrics
```

You should see metrics such as:

```
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_memory_MemTotal_bytes
node_filesystem_avail_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
node_network_transmit_bytes_total
node_load1
```

These metrics allow Prometheus to calculate node-level resource utilization.

### 39. Important Node Exporter queries

CPU usage percentage

```
100 *
(
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Memory usage percentage

```
100 *
(
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```

Filesystem usage percentage

```
100 *
(
  1 -
  (
    node_filesystem_avail_bytes{
      fstype!~"tmpfs|overlay",
      mountpoint!="/boot"
    }
    /
    node_filesystem_size_bytes{
      fstype!~"tmpfs|overlay",
      mountpoint!="/boot"
    }
  )
)
```

Review the filesystem filters for your actual EKS node operating system and mount layout. Do not assume every filesystem represents a useful persistent capacity measure.

Network receive rate

```
rate(node_network_receive_bytes_total[5m])
```

Network transmit rate

```
rate(node_network_transmit_bytes_total[5m])
```

System load

```
node_load1
```

Load average must be interpreted in relation to the number of CPU cores. A load of 4 means something different on a two-core node than on a 16-core node.

### 40. kube-state-metrics

Node Exporter tells us about the Linux machine.

It does not tell us the complete state of Kubernetes objects.

For example:

- How many pods are desired?
- How many pods are ready?
- How many replicas are unavailable?
- Which deployments have failed to roll out?
- Which pods are waiting or restarting?
- Which PersistentVolumeClaims are pending?

That information comes from Kubernetes object state.

`kube-state-metrics` watches Kubernetes API objects and exposes their state as Prometheus metrics.

Examples:

```
kube_pod_status_phase
kube_pod_container_status_restarts_total
kube_deployment_spec_replicas
kube_deployment_status_replicas_available
kube_node_status_condition
kube_persistentvolumeclaim_status_phase
```

Check its deployment:

```
kubectl get deployment \
  -n monitoring \
  | grep kube-state-metrics
```

Check its service:

```
kubectl get svc \
  -n monitoring \
  | grep kube-state-metrics
```

Check pods:

```
kubectl get pods \
  -n monitoring \
  | grep kube-state-metrics
```

### 41. Kubernetes workload metrics

Pods not running

```
kube_pod_status_phase{
  namespace="ecommerce",
  phase!="Running"
} == 1
```

This is a basic inspection query. It may include pods in legitimate transitional states, so production alerts need more careful conditions and durations.

Container restart count increase

```
increase(
  kube_pod_container_status_restarts_total{
    namespace="ecommerce"
  }[15m]
)
```

Deployment desired replicas

```
kube_deployment_spec_replicas{
  namespace="ecommerce"
}
```

Deployment available replicas

```
kube_deployment_status_replicas_available{
  namespace="ecommerce"
}
```

PersistentVolumeClaims pending

```
kube_persistentvolumeclaim_status_phase{
  namespace="ecommerce",
  phase="Pending"
} == 1
```

These queries are useful for dashboards and troubleshooting. Alert rules should account for rollout windows, expected scaling behavior, and transient states.

### 42. Kubelet and container metrics

Kubernetes nodes expose additional metrics through the kubelet.

Depending on the cluster configuration, Prometheus can collect:

- Container CPU usage.
- Container memory working set.
- Container filesystem usage.
- Pod-level resource usage.
- Kubelet operation metrics.

Common metric names include:

```
container_cpu_usage_seconds_total
container_memory_working_set_bytes
kubelet_running_pods
kubelet_running_containers
```

Metric availability varies with Kubernetes, kubelet, container runtime, and monitoring configuration.

For example, container metrics may come from the kubelet's `/metrics/cadvisor` endpoint. Ensure that the relevant endpoint is enabled, discoverable, and securely accessible to Prometheus.

Do not expose kubelet metrics publicly.

### 43. Identify CPU throttling

A pod can have low apparent CPU utilization and still experience CPU throttling because of its configured CPU limit.

Useful metric:

```
container_cpu_cfs_throttled_seconds_total
```

A basic throttling ratio can be calculated using available CFS counters:

```
100 *
(
  rate(
    container_cpu_cfs_throttled_periods_total{
      namespace="ecommerce"
    }[5m]
  )
  /
  rate(
    container_cpu_cfs_periods_total{
      namespace="ecommerce"
    }[5m]
  )
)
```

This query calculates the percentage of observed CFS periods that experienced throttling. It is not a direct percentage of CPU capacity consumed.

Check the actual metric availability and labels in your environment before using it in a dashboard.

### 44. Monitor EKS control-plane activity

EKS control-plane monitoring needs special attention because the control plane is AWS-managed.

You cannot install Node Exporter or kube-state-metrics inside the managed Kubernetes API server.

Instead, use the monitoring options provided by AWS and the EKS service:

- EKS control-plane logging.
- CloudWatch metrics and log groups.
- Kubernetes API server metrics where supported through AWS integrations.
- AWS service health and infrastructure signals.
- Kubernetes API object state collected by in-cluster monitoring.

Enable the relevant EKS control-plane logs through the EKS cluster configuration.

Example AWS CLI:

```
aws eks update-cluster-config \
  --region ap-south-1 \
  --name ecommerce-prod \
  --logging '{
    "clusterLogging": [
      {
        "types": [
          "api",
          "audit",
          "authenticator",
          "controllerManager",
          "scheduler"
        ],
        "enabled": true
      }
    ]
  }'
```

This command changes the EKS cluster logging configuration. Review the change in the production change-management process before executing it.

Verify:

```
aws eks describe-cluster \
  --region ap-south-1 \
  --name ecommerce-prod \
  --query 'cluster.logging'
```

The logs are useful for investigating:

- Kubernetes API authentication failures.
- Authorization problems.
- Control-plane operations.
- Scheduling issues.
- Kubernetes API activity.
- Changes made by users and automation.

CloudWatch is an AWS-managed destination for these control-plane logs. It is not a replacement for the Prometheus and Grafana monitoring platform.

### 45. RDS monitoring integration

Our microservices may use Amazon RDS. Database monitoring should combine application-side and database-side information.

Application-side metrics:

```
database connection pool active
database connection pool idle
database connection timeout count
database query duration
```

RDS-side metrics can include:

- CPU utilization.
- Free storage.
- Freeable memory.
- Database connections.
- Read IOPS.
- Write IOPS.
- Read latency.
- Write latency.
- Disk queue depth.

The platform team should integrate the relevant AWS metrics into Grafana using an approved AWS metrics datasource or monitoring integration.

Do not assume that Prometheus automatically scrapes RDS just because it runs inside EKS. RDS is an AWS-managed service and needs a separate metrics integration.

### 46. Database connection pool dashboard

For Spring Boot applications using HikariCP, inspect the application metrics for connection-pool utilization.

Depending on the Micrometer configuration, metrics may include:

```
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_max
hikaricp_connections_min
```

A useful dashboard should display:

- Active connections.
- Idle connections.
- Pending connection requests.
- Maximum pool size.
- Connection acquisition latency.
- Connection timeout failures.

Example:

```
hikaricp_connections_active{
  service="order-service"
}
```

Pending connections:

```
hikaricp_connections_pending{
  service="order-service"
}
```

Connection pool utilization:

```
100 *
(
  hikaricp_connections_active
  /
  hikaricp_connections_max
)
```

Confirm that the labels and metric names match the actual application's exported metrics.

A high connection-pool utilization percentage is an investigation signal, not proof that the database itself is overloaded. Check the database's connection count, CPU, query latency, and connection limits before changing pool sizes.

### 47. Messaging monitoring

The project has discussed Kafka and RabbitMQ as messaging options. Do not deploy monitoring for both merely because both are possible technologies.

Monitor the messaging system actually selected for each workload.

For Kafka, useful areas include:

- Broker availability.
- Under-replicated partitions.
- Offline partitions.
- Consumer lag.
- Request latency.
- Disk utilization.
- Network throughput.

For RabbitMQ, useful areas include:

- Queue depth.
- Message publish and delivery rates.
- Unacknowledged messages.
- Consumer count.
- Connection count.
- Memory and disk alarms.
- Node availability.

For either technology, application-level metrics should also show whether the service is successfully publishing and consuming messages.

The platform team provides the monitoring integration. The messaging owner defines broker-specific operational thresholds.

### 48. Node monitoring validation checklist

After deploying infrastructure monitoring, verify:

```
kubectl get daemonset -n monitoring

kubectl get pods -n monitoring -o wide

kubectl get servicemonitor -A

kubectl get prometheus -n monitoring
```

Then verify in Prometheus:

- Node Exporter targets are UP.
- kube-state-metrics targets are UP.
- Kubelet targets are UP where configured.
- Node CPU metrics are available.
- Node memory metrics are available.
- Kubernetes deployment metrics are available.
- Pod restart metrics are available.
- No unexpected duplicate scrape targets exist.

Record the results in the platform monitoring rollout ticket.

## Part 5 — Prometheus Production Operations

### 49. Understand Prometheus storage

Prometheus stores time-series data in its local Time Series Database (TSDB).

A time series is identified by:

- Metric name.
- Label set.

For example:

```
http_requests_total{
  service="order-service",
  environment="production",
  status="200"
}
```

If any label value changes, Prometheus may create a different time series.

This is why high-cardinality labels are dangerous.

### 50. Estimate time-series growth

Suppose the platform has:

- 21 microservices.
- 3 replicas per service.
- 100 active time series per pod.

A simple estimate:

```
21 services × 3 pods × 100 series

= 6,300 active time series
```

This is only an illustration. Real series counts can be much higher because of:

- HTTP route labels.
- HTTP status labels.
- JVM metrics.
- Database pool metrics.
- Business metrics.
- Kubernetes metadata.
- Histograms and their buckets.
- Infrastructure exporters.

A histogram with many buckets produces multiple series for each label combination.

Measure actual cardinality rather than relying on a theoretical estimate.

### 51. Why high cardinality is dangerous

Consider this metric:

```
orders_created_total{
  service="order-service",
  customer_id="C10001"
}
```

If the application serves one million customers, the customer ID label may create a huge number of time series.

Consequences:

- Higher memory usage.
- Increased disk usage.
- Longer query execution.
- Higher CPU usage.
- Slower Prometheus startup and recovery.
- Increased monitoring infrastructure cost.

Use low-cardinality dimensions:

```
service
environment
namespace
team
operation
status
```

Use logs for individual customer or order investigation, subject to privacy and access controls.

### 52. Prometheus retention

Retention determines how long Prometheus keeps local metrics.

Example:

```
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: 40GB
```

A 15-day retention period may be sufficient for short-term operational troubleshooting, but business and compliance requirements may need longer retention.

Decide retention based on:

- Incident investigation requirements.
- Historical trend analysis.
- Storage capacity.
- Compliance requirements.
- Backup and recovery objectives.
- Cost.

Do not increase retention without measuring the resulting disk requirement.

### 53. Persistent storage

Prometheus needs persistent storage if metrics must survive pod recreation.

For EKS, a common approach is to use EBS-backed persistent volumes through an approved StorageClass.

Example:

```
storageSpec:
  volumeClaimTemplate:
    spec:
      storageClassName: gp3
      accessModes:
        - ReadWriteOnce

      resources:
        requests:
          storage: 100Gi
```

This configuration requests a 100 GiB volume. It does not guarantee that 100 GiB is sufficient.

Before selecting storage capacity, measure ingestion rate and expected retention. Also confirm:

- The EBS CSI driver is installed and working.
- The StorageClass exists.
- The EBS volume type and performance are appropriate.
- The node and volume availability-zone placement is compatible.
- Snapshot and recovery procedures are documented.
- PVC expansion is supported and tested.

### 54. Prometheus replicas and high availability

For production, consider running multiple Prometheus replicas.

However, two Prometheus replicas do not automatically create a single shared TSDB.

Each replica generally scrapes and stores its own copy of the metrics. Querying, deduplication, and long-term storage require additional architecture decisions.

A simplified model:

```
                 Kubernetes Targets
                         |
              +----------+----------+
              |                     |
              v                     v
       Prometheus A          Prometheus B
              |                     |
              v                     v
        Local TSDB A           Local TSDB B
              |                     |
              +----------+----------+
                         |
                         v
                    Grafana
```

For more demanding requirements, evaluate a long-term metrics architecture such as remote write to an appropriate metrics backend.

Do not describe replicas alone as complete disaster recovery. Define:

- What happens if one replica fails?
- How are queries served?
- How is data deduplicated, if applicable?
- How is data recovered after storage loss?
- What is the acceptable data-loss window?

### 55. Prometheus resource monitoring

Prometheus itself must be monitored.

Useful areas:

- Process CPU and memory.
- TSDB head series.
- Sample ingestion rate.
- Scrape duration.
- Scrape failures.
- Rule evaluation duration.
- Query execution duration.
- Storage availability.

Useful Prometheus self-metrics include:

```
prometheus_tsdb_head_series
prometheus_tsdb_head_samples_appended_total
prometheus_target_scrapes_exceeded_sample_limit_total
prometheus_rule_evaluation_failures_total
prometheus_engine_query_duration_seconds
```

Metric availability depends on the Prometheus version and configuration.

Do not wait for Prometheus to become unavailable before checking its own capacity.

### 56. Prometheus capacity investigation

Suppose Prometheus restarts frequently.

Start with:

```
kubectl get pods -n monitoring
```

Inspect the pod:

```
kubectl describe pod \
  PROMETHEUS_POD \
  -n monitoring
```

Check previous logs:

```
kubectl logs \
  PROMETHEUS_POD \
  -n monitoring \
  --previous
```

Check resource usage:

```
kubectl top pod \
  -n monitoring
```

Check the PVC:

```
kubectl get pvc \
  -n monitoring
```

Investigate:

1. Was the container OOM-killed?
2. Is the PVC full?
3. Did active series suddenly increase?
4. Did a new service introduce high-cardinality labels?
5. Did histogram bucket count increase?
6. Are expensive dashboard queries running too frequently?
7. Did the scrape interval change?
8. Did a large number of targets become available simultaneously?

A capacity incident should be solved by identifying the growth source, not merely increasing memory limits without investigation.

### 57. Grafana dashboard queries and performance

A dashboard can overload Prometheus if it executes many expensive queries at short intervals.

For example, this query may be expensive across a large number of series:

```
sum by (service, route, status) (
  rate(http_server_requests_seconds_count[1m])
)
```

Potential improvements:

- Reduce unnecessary label dimensions.
- Use a suitable rate window.
- Avoid querying every unique route if it is not operationally useful.
- Use recording rules for repeated expensive calculations.
- Avoid excessive dashboard refresh frequency.
- Limit dashboard time ranges when appropriate.

Chapter 188 will introduce reusable alert rules. Recording rules may also be used to precompute common expressions for dashboards.

### 58. Prometheus operational checklist

Before production approval, verify:

```
[ ] Prometheus targets are discovered correctly
[ ] All expected targets are UP
[ ] Scrape failures are investigated
[ ] Retention is documented
[ ] PVC capacity is appropriate
[ ] Resource requests and limits are tested
[ ] High-cardinality metrics are reviewed
[ ] Prometheus self-monitoring is enabled
[ ] Backup and recovery requirements are documented
[ ] Access to Prometheus is restricted
[ ] Production dashboards do not overload Prometheus
[ ] Prometheus upgrades are tested before production rollout
```

## Part 6 — Grafana Dashboards as Code

### 59. Why dashboards must be managed as code

Imagine a developer manually creates an Order Service dashboard in Grafana.

Another developer creates a similar dashboard for Payment Service.

A third engineer creates a different dashboard for Inventory Service.

Over time:

- Dashboard names become inconsistent.
- Different teams use different metric definitions.
- Some dashboards contain outdated queries.
- Production and staging dashboards behave differently.
- Recreating a dashboard after Grafana recovery becomes difficult.

The platform team should therefore maintain dashboards in Git.

The deployment process should provision dashboards automatically.

### 60. Grafana dashboard structure

Our repository:

```
platform-monitoring/
└── grafana/
    ├── datasources/
    │   └── prometheus.yaml
    │
    ├── dashboards/
    │   ├── kubernetes-cluster.json
    │   ├── kubernetes-node.json
    │   ├── microservice-overview.json
    │   ├── order-service.json
    │   ├── payment-service.json
    │   └── database-overview.json
    │
    └── provisioning/
        ├── dashboards.yaml
        └── datasources.yaml
```

Use a standard dashboard layout:

1. Service health.
2. Request volume.
3. Error rate.
4. Request latency.
5. CPU and memory.
6. Pod replicas and restarts.
7. Dependency health.
8. Business metrics.

### 61. Configure Prometheus as a Grafana datasource

File:

```
grafana/provisioning/datasources.yaml
```

Example:

```
apiVersion: 1

datasources:

  - name: Prometheus
    type: prometheus

    access: proxy

    url: http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090

    isDefault: true

    editable: false

    jsonData:
      timeInterval: 30s
```

The URL is an example based on a common Helm service naming pattern. Confirm the actual Prometheus service name and namespace in your cluster.

Provisioning means Grafana creates or updates the datasource from configuration rather than requiring an administrator to create it manually.

### 62. Configure dashboard provisioning

File:

```
grafana/provisioning/dashboards.yaml
```

```
apiVersion: 1

providers:

  - name: ecommerce-dashboards

    orgId: 1

    folder: Ecommerce Platform

    type: file

    disableDeletion: false

    editable: false

    options:
      path: /var/lib/grafana/dashboards
```

The dashboard JSON files must be mounted into the configured directory.

In production, protect the Git repository and require review for dashboard changes. If dashboards are provisioned from files, do not rely on manual edits in Grafana as the permanent source of truth.

### 63. Standard microservice dashboard

Every service should have the same basic dashboard structure.

Example: Order Service.

[Monitoring and Distributed Tracing with Grafana, Prometheus, Loki & Tempo · Cozi](https://images.openai.com/static-rsc-4/bBv6i6gC2oP8MX11A7H_fz9DGpxOxSTuiw1Y78iNsZeQtidD2XBanWjOx7scaHGWtmN0A0-BmeuInKrYu6fM7w8eBB_8tmP-H017xSA3Y9ytcGEDgcfRQ4hgYsc4U_u1zLOzcP3T4vE8uEdPEB0WIl0z62zq7DdIYfGB85pFeF8?purpose=inline)

[cozi.dev](https://cozi.dev/posts/monitoring-and-distributed-tracing-with-grafana-prometheus-loki-and-tempo/?utm_source=chatgpt.com)

Panel 1 — Request rate

```
sum(
  rate(
    http_server_requests_seconds_count{
      service="order-service"
    }[5m]
  )
)
```

Panel 2 — HTTP error rate

```
100 *
(
  sum(
    rate(
      http_server_requests_seconds_count{
        service="order-service",
        status=~"5.."
      }[5m]
    )
  )
  /
  clamp_min(
    sum(
      rate(
        http_server_requests_seconds_count{
          service="order-service"
        }[5m]
      )
    ),
    0.001
  )
)
```

This query calculates the percentage of server-side 5xx responses among the observed requests. It does not include every possible business failure, such as an order rejected because inventory is unavailable.

Panel 3 — 95th percentile latency

```
histogram_quantile(
  0.95,
  sum by (le) (
    rate(
      http_server_requests_seconds_bucket{
        service="order-service"
      }[5m]
    )
  )
)
```

This requires histogram bucket metrics.

Panel 4 — CPU usage

For application container CPU:

```
sum by (pod) (
  rate(
    container_cpu_usage_seconds_total{
      namespace="ecommerce",
      container="order-service"
    }[5m]
  )
)
```

Panel 5 — Memory usage

```
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="ecommerce",
    container="order-service"
  }
)
```

Panel 6 — Pod restarts

```
sum by (pod) (
  increase(
    kube_pod_container_status_restarts_total{
      namespace="ecommerce",
      container="order-service"
    }[15m]
  )
)
```

Verify the exact labels on your exported metrics. Kubernetes and container-runtime metric labels can differ across versions and configurations.

### 64. Dashboard variables

We do not want to create a completely different dashboard for every environment.

Use Grafana variables.

Example variables:

```
environment
namespace
service
```

Then the dashboard can be reused for:

```
DEV
QA
STAGING
PRODUCTION
```

For example, use a service variable in PromQL:

```
sum(
  rate(
    http_server_requests_seconds_count{
      service=~"$service",
      environment=~"$environment"
    }[5m]
  )
)
```

This works only if the relevant labels are consistently available on the metric series.

### 65. Dashboard access control

Production dashboards may expose internal infrastructure information.

Use appropriate access controls:

- Developers can view approved service dashboards.
- Platform engineers can manage shared dashboards.
- SREs can view operational and incident dashboards.
- Administrators manage Grafana configuration.
- Database teams access relevant database dashboards.

Do not give every developer administrator access to the production Grafana instance.

Use the organization's approved authentication integration and group-based authorization where available.

## Part 7 — Centralized Logging Integration

### 66. Why metrics and logs must work together

Consider this production incident:

```
Order Service response time increased.
```

Prometheus tells us:

```
Request latency increased from 200ms to 3s.
```

Grafana shows:

```
Order Service error rate increased.
```

But these metrics do not necessarily show the exact application failure.

We need logs to investigate the request processing path.

For this project, the logging stack is:

```
Application
    |
    v
Container stdout/stderr
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

The log collection agent can be Fluent Bit deployed as a DaemonSet. It collects container logs from the nodes and forwards them to Logstash.

Logstash parses and enriches the events before indexing them in Elasticsearch.

### 67. Structured JSON logging

Do not produce inconsistent unstructured log messages.

Bad:

```
Order created successfully
```

Better:

```
{
  "timestamp": "2026-10-31T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "environment": "production",
  "message": "Order created successfully",
  "operation": "create-order",
  "order_reference": "ORD-REDACTED"
}
```

The reference above is illustrative. Use a non-sensitive identifier or a properly controlled pseudonymous reference.

Standard log fields:

| Field                   | Purpose                         |
| ----------------------- | ------------------------------- |
| timestamp               | Event time                      |
| level                   | Severity                        |
| service                 | Application identity            |
| environment             | Deployment environment          |
| message                 | Human-readable description      |
| operation               | Business or technical operation |
| trace or correlation ID | Request investigation           |
| version                 | Application release             |

A correlation ID can help connect related logs across services without creating a Prometheus label for every request.

Do not log:

- Passwords.
- Access tokens.
- Authorization headers.
- Database credentials.
- Payment card data.
- Unnecessary personal information.
- Full sensitive request or response bodies.

### 68. Example Logstash pipeline

File:

```
logging/logstash/application.conf
```

Example:

```
input {

  beats {
    port => 5044
  }

}

filter {

  json {
    source => "message"
    skip_on_invalid_json => true
  }

  mutate {
    lowercase => [
      "service",
      "environment"
    ]
  }

  if [level] == "ERROR" {
    mutate {
      add_tag => ["application_error"]
    }
  }

}

output {

  elasticsearch {

    hosts => [
      "http://elasticsearch:9200"
    ]

    index => "ecommerce-application-%{+YYYY.MM.dd}"

  }

  stdout {
    codec => rubydebug
  }

}
```

This is a basic demonstration pipeline. A production pipeline needs additional configuration for:

- TLS and authentication.
- Elasticsearch credentials through a secure secret mechanism.
- Schema validation.
- Invalid JSON handling.
- Field normalization.
- Sensitive-field redaction.
- Retry and backpressure behavior.
- Index lifecycle management.
- Dead-letter handling where appropriate.
- Removal of the debug `stdout` output.

Also, this example assumes a Beats-compatible input. Configure the chosen log collector to use the corresponding protocol and destination.

### 69. Configure application logging

For Spring Boot, use a JSON logging encoder compatible with the application's logging framework, or configure the application's approved logging library to produce JSON.

The application should write logs to standard output and standard error.

Kubernetes and the container runtime capture these streams. The log collector reads the container log files from the node.

Do not make each microservice connect directly to Elasticsearch using its own credentials. Centralized collection provides better control over buffering, parsing, security, and retention.

### 70. Log correlation during an incident

Example request flow:

```
Customer
   |
   v
API Gateway
   |
   v
Order Service
   |
   +----> Inventory Service
   |
   +----> Payment Service
   |
   v
Order Database
```

Suppose a customer reports that an order failed.

The investigation process is:

1. Find the request using the approved correlation ID.
2. Search Order Service logs in Kibana.
3. Identify the downstream service call.
4. Search Inventory or Payment Service logs using the same correlation ID, if propagated.
5. Compare the log timestamps with the Prometheus latency graphs.
6. Check Kubernetes events and pod restarts.
7. Determine whether the problem is application, dependency, database, or infrastructure related.

This approach provides request-level investigation without creating high-cardinality Prometheus labels.

### 71. Log retention and index lifecycle

Do not retain all application logs indefinitely without a defined business or compliance requirement.

Define retention by log category.

For example:

| Log category                 | Example policy                                |
| ---------------------------- | --------------------------------------------- |
| Application operational logs | 30 days                                       |
| Security audit logs          | Based on security policy                      |
| Debug logs                   | Short retention                               |
| Payment-related audit events | Based on regulatory and internal requirements |
| Infrastructure logs          | Based on operational requirements             |

These are illustrative retention categories, not approved retention periods. The security and compliance teams must define the actual policy.

Use Elasticsearch index lifecycle management to control rollover, retention, and deletion.

### 72. Monitoring platform security

Monitoring systems often contain sensitive operational information.

Apply these controls:

Network access

- Keep Prometheus and Grafana internal.
- Restrict access to exporter endpoints.
- Restrict access to Logstash inputs.
- Use NetworkPolicies where supported.
- Validate required traffic before enabling default-deny policies.

Authentication and authorization

- Use approved authentication for Grafana.
- Restrict production access by team.
- Protect Elasticsearch and Kibana.
- Do not expose internal dashboards through public ingress without an approved access design.

Secrets

- Do not commit datasource passwords.
- Do not store Elasticsearch credentials in plain YAML.
- Use the approved secret-management integration.
- Rotate credentials according to policy.

Data protection

- Redact sensitive application fields before indexing.
- Restrict access to security audit logs.
- Define retention and deletion requirements.
- Audit administrative access.

### 73. Chapter 187 versus Chapter 188

These two chapters have separate responsibilities.

| Chapter                    | Responsibility                                                                                             |
| -------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 187 — Monitoring Templates | Install monitoring infrastructure, expose metrics, discover services, create dashboards and integrate logs |
| 188 — Alert Rule Templates | Define reusable alert conditions, severity, thresholds, routing and alert testing                          |

For example:

Chapter 187 gives us the ability to see that Order Service latency is increasing.

Chapter 188 defines when that latency should generate a warning or page an on-call engineer.

## Part 8 — Production Troubleshooting Runbooks

### 74. Runbook: Prometheus target is DOWN

Symptoms

- Order Service target shows DOWN.
- Grafana panels show missing data.
- Prometheus reports scrape errors.

Step 1 — Check the target

Open:

```
http://localhost:9090/targets
```

Read the exact scrape error.

Step 2 — Check ServiceMonitor

```
kubectl get servicemonitor \
  -n ecommerce \
  order-service \
  -o yaml
```

Step 3 — Check Service

```
kubectl get svc \
  -n ecommerce \
  order-service \
  -o yaml
```

Step 4 — Check EndpointSlices

```
kubectl get endpointslices \
  -n ecommerce \
  -l kubernetes.io/service-name=order-service
```

Step 5 — Check application

```
kubectl get pods \
  -n ecommerce \
  -l app=order-service
```

Step 6 — Test endpoint

From a diagnostic pod with network access:

```
curl -v \
  http://order-service.ecommerce.svc.cluster.local:8080/actuator/prometheus
```

Possible causes

- Wrong endpoint path.
- Incorrect ServiceMonitor label selector.
- Wrong named Service port.
- Application not listening.
- Security rejection.
- NetworkPolicy blocking traffic.
- No ready endpoints.

Resolution

Correct the specific configuration issue, deploy through GitOps, and verify that the target becomes UP.

Do not bypass security controls by making the metrics endpoint publicly accessible.

### 75. Runbook: Grafana dashboard shows no data

Symptoms

- Grafana opens successfully.
- Dashboard panels display `No data`.
- Prometheus may still be running.

Check:

1. Is the Prometheus datasource healthy?
2. Is the dashboard querying the correct metric?
3. Are the environment and service variables correct?
4. Does the service label exist?
5. Is the selected time range correct?
6. Does the Prometheus query return results directly?

Example direct query:

```
up{
  service="order-service"
}
```

If the query returns no data, investigate target discovery and labels.

If Prometheus returns data but Grafana does not, investigate the datasource, query, permissions, and dashboard variables.

### 76. Runbook: Prometheus pod is OOMKilled

Check:

```
kubectl describe pod \
  PROMETHEUS_POD \
  -n monitoring
```

Look for:

```
Reason: OOMKilled
```

Then investigate:

- Memory limit.
- Active time series.
- Scrape volume.
- Histogram bucket growth.
- Query load.
- Recent monitoring configuration changes.

Use Prometheus self-metrics and resource metrics to identify the cause.

Temporary mitigation may include increasing memory after capacity review. Permanent remediation should address the source of the growth.

### 77. Runbook: Node Exporter target missing

Check:

```
kubectl get daemonset \
  -n monitoring
```

Check:

```
kubectl get pods \
  -n monitoring \
  -o wide
```

If a node has no Node Exporter pod, inspect:

```
kubectl describe daemonset \
  NODE_EXPORTER_DAEMONSET \
  -n monitoring
```

Check:

- Node selectors.
- Taints and tolerations.
- Pod scheduling events.
- DaemonSet update status.
- Node architecture and operating system.
- Prometheus scrape discovery.

If the pod exists but the target is missing, inspect the ServiceMonitor or PodMonitor selection and the Prometheus target configuration.

### 78. Runbook: Logs are missing in Kibana

Start with the application:

```
kubectl logs \
  -n ecommerce \
  ORDER_SERVICE_POD \
  --tail=100
```

If logs exist in Kubernetes but not Kibana, investigate:

1. Log collector pod health.
2. Collector input configuration.
3. Collector output configuration.
4. Network access to Logstash.
5. Logstash pipeline status.
6. JSON parsing failures.
7. Elasticsearch indexing failures.
8. Index naming and Kibana data view.
9. Retention or lifecycle deletion.

Check Logstash logs:

```
kubectl logs \
  -n logging \
  LOGSTASH_POD \
  --tail=200
```

Check Elasticsearch health using the approved authenticated endpoint.

Do not assume that a missing Kibana result means the application did not generate the log.

### 79. Runbook: High node memory utilization

Check:

```
kubectl top nodes
```

Then:

```
kubectl top pods \
  -n ecommerce \
  --sort-by=memory
```

Compare:

- Node memory utilization.
- Pod working-set memory.
- Pod requests and limits.
- Container restart history.
- JVM heap metrics.
- Node scheduling capacity.

If a single application pod is consuming too much memory, inspect the application.

If many pods are consuming more memory than expected, investigate workload growth and node capacity.

Do not simply increase node size without identifying whether the problem is an application memory leak, incorrect requests, workload growth, or scheduling imbalance.

### 80. Monitoring rollout completion checklist

Before declaring Chapter 187 complete:

Application integration

```
[ ] All 21 services have a documented metrics endpoint
[ ] Required application metrics are exposed
[ ] Liveness and readiness endpoints are configured
[ ] Standard service labels are present
[ ] Business metrics are reviewed by service owners
[ ] Sensitive values are not exposed
```

Prometheus

```
[ ] Prometheus is deployed through approved Helm configuration
[ ] Service discovery works
[ ] All expected targets are UP
[ ] Node Exporter is collecting node metrics
[ ] kube-state-metrics is collecting Kubernetes object metrics
[ ] Kubelet metrics are configured where required
[ ] Retention and storage are documented
[ ] Capacity is reviewed
```

Grafana

```
[ ] Prometheus datasource is provisioned
[ ] Dashboards are stored in Git
[ ] Service dashboard is standardized
[ ] Environment variables work
[ ] Dashboard access is restricted
```

Logging

```
[ ] Application logs are structured
[ ] Logs reach Logstash
[ ] Logs are indexed in Elasticsearch
[ ] Kibana searches work
[ ] Sensitive fields are redacted
[ ] Retention is defined
```

Operations

```
[ ] Troubleshooting runbooks are available
[ ] Monitoring platform health is monitored
[ ] Deployment and rollback procedures are documented
[ ] Production owners are identified
[ ] Monitoring rollout evidence is recorded
```

### 81. Practical implementation exercise

Implement monitoring for Order Service first.

Task 1 — Instrument the application

- Add Spring Boot Actuator.
- Add Micrometer Prometheus registry.
- Expose `/actuator/prometheus`.
- Add business metrics for successful and failed order creation.
- Add a timer for a downstream inventory check.

Task 2 — Deploy monitoring

- Install the approved `kube-prometheus-stack` chart.
- Configure persistent storage.
- Verify Node Exporter.
- Verify kube-state-metrics.
- Configure Prometheus Operator discovery.

Task 3 — Connect Order Service

- Add monitoring labels.
- Create the ServiceMonitor.
- Confirm the target is UP.
- Validate the metrics in Prometheus.

Task 4 — Create Grafana dashboards

- Provision Prometheus datasource.
- Create a standard microservice dashboard.
- Add request rate.
- Add error rate.
- Add p95 latency.
- Add CPU and memory.
- Add pod restarts.
- Add business operation metrics.

Task 5 — Integrate logging

- Produce structured JSON logs.
- Collect container logs.
- Forward them to Logstash.
- Verify Elasticsearch indexing.
- Search the Order Service logs in Kibana.

Task 6 — Simulate an issue

For a controlled test environment:

- Generate application traffic.
- Introduce a temporary application delay.
- Observe the latency panel.
- Inspect application logs.
- Identify the affected downstream dependency.
- Restore the normal application behavior.

Do not introduce artificial failures into production as part of this exercise.

### 82. Definition of Done

Chapter 187 completion criteria

Chapter 187 is complete when the platform has a reusable monitoring implementation that can be applied consistently to all 21 microservices.

The platform must provide:

1. Standard application instrumentation.
2. Kubernetes service discovery.
3. Prometheus deployment and persistent storage.
4. Node and Kubernetes infrastructure monitoring.
5. Grafana dashboards provisioned from Git.
6. Centralized structured logging.
7. Security and retention standards.
8. Troubleshooting procedures.
9. A tested monitoring onboarding path for new services.

Next chapter — Chapter 188: Alert Rule Templates

In Chapter 188, we will build reusable Prometheus alert rules for the 21 microservices, including CPU, memory, pod restarts, HTTP 5xx errors, latency, database connection exhaustion, node health, and production service availability.