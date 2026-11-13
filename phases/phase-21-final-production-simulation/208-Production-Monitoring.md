# CHAPTER 207 — PRODUCTION MONITORING

**Filename:** `207-Production-Monitoring.md`

---

# 1. Chapter Objective

In the previous chapters, we implemented Production deployment strategies, including rolling updates and canary deployments.

Now we need to monitor our Production environment continuously.

Deploying an application successfully does not mean that the application will remain healthy.

Production issues can occur because of:

- High CPU utilization.
- Memory exhaustion.
- Kubernetes node failures.
- Pod crashes.
- Application errors.
- Database connection exhaustion.
- Redis failures.
- RabbitMQ queue buildup.
- Kafka consumer lag.
- Network latency.
- Disk pressure.
- Increased application response time.
- Deployment failures.
- Infrastructure capacity problems.

Our goal is to build a complete Production monitoring system.

We will implement:

1. Production monitoring architecture.
2. Prometheus architecture.
3. Prometheus installation on EKS.
4. Prometheus Operator.
5. ServiceMonitor.
6. PodMonitor.
7. Node Exporter.
8. kube-state-metrics.
9. Application metrics.
10. Spring Boot metrics.
11. Node.js application metrics.
12. Custom Prometheus metrics.
13. Grafana installation.
14. Production Grafana dashboards.
15. Kubernetes dashboards.
16. Application dashboards.
17. CPU and memory monitoring.
18. Network monitoring.
19. Database monitoring.
20. Redis monitoring.
21. RabbitMQ monitoring.
22. Kafka monitoring.
23. ELK Stack architecture.
24. Elasticsearch.
25. Logstash.
26. Kibana.
27. Filebeat and log collection.
28. Kubernetes container logs.
29. Application log parsing.
30. Centralized log correlation.
31. Alertmanager.
32. Production alert rules.
33. Alert routing.
34. Notification integration.
35. SLI and SLO implementation.
36. Error budget monitoring.
37. Golden signals.
38. Production monitoring runbooks.
39. Monitoring failure simulations.
40. Monitoring troubleshooting.
41. Complete Production monitoring validation.

# 2. Production Environment Architecture

Our application consists of more than 20 microservices running on AWS EKS.

The monitoring platform must collect infrastructure, Kubernetes, application, database, and messaging metrics.

Our monitoring tools:

| Component | Tool |
|---|---|
| Metrics collection | Prometheus |
| Metrics visualization | Grafana |
| Kubernetes metrics | kube-state-metrics |
| Node metrics | Node Exporter |
| Alert evaluation | Prometheus |
| Alert routing | Alertmanager |
| Log processing | Logstash |
| Log storage and search | Elasticsearch |
| Log visualization | Kibana |
| Load testing | k6 |

We are not introducing Jaeger or OpenTelemetry in this implementation.

# 3. Complete Production Monitoring Architecture

```text
                     USERS
                       |
                       v
                  AWS ALB
                       |
                       v
                    INGRESS
                       |
                       v
                 AWS EKS CLUSTER
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
     USER POD       CART POD       ORDER POD
        |              |              |
        v              v              v
    APP METRICS    APP METRICS    APP METRICS
        |              |              |
        +--------------+--------------+
                       |
                       v
                  PROMETHEUS
                       |
              +--------+--------+
              |                 |
              v                 v
           GRAFANA          ALERTMANAGER
              |                 |
              v                 v
        DASHBOARDS          NOTIFICATIONS


                KUBERNETES METRICS

     NODE EXPORTER       KUBE-STATE-METRICS
           |                    |
           +---------+----------+
                     |
                     v
                 PROMETHEUS


                   LOG PIPELINE

       APPLICATION CONTAINER LOGS
                     |
                     v
                  FILEBEAT
                     |
                     v
                  LOGSTASH
                     |
                     v
                ELASTICSEARCH
                     |
                     v
                   KIBANA
```

# 4. Monitoring Responsibilities

Different tools monitor different parts of the environment.

## 4.1 Prometheus

Prometheus collects and stores time-series metrics.

Examples:

- CPU utilization.
- Memory utilization.
- HTTP request count.
- HTTP error count.
- Application response time.
- Pod restart count.
- Node availability.
- Database connection count.

Prometheus uses a pull-based scraping model for normal target collection.

It periodically requests metrics from configured targets.

## 4.2 Grafana

Grafana connects to Prometheus and visualizes metrics.

Examples:

- Kubernetes cluster dashboard.
- Node health dashboard.
- Application performance dashboard.
- Database dashboard.
- Redis dashboard.
- RabbitMQ dashboard.
- Kafka dashboard.

## 4.3 Node Exporter

Node Exporter exposes Linux host-level metrics.

Examples:

- CPU.
- Memory.
- Disk.
- Filesystem.
- Network interfaces.

## 4.4 kube-state-metrics

kube-state-metrics exposes Kubernetes object state.

Examples:

- Deployment desired replicas.
- Deployment available replicas.
- Pod status.
- Pod readiness.
- Job status.
- Node conditions.
- PersistentVolumeClaim status.

Important distinction:

Node Exporter monitors operating-system-level metrics. kube-state-metrics monitors Kubernetes object state. They solve different problems.

## 4.5 ELK Stack

The ELK Stack collects, processes, stores, and visualizes logs.

- Elasticsearch: Stores and searches log data.
- Logstash: Processes and transforms log events.
- Kibana: Provides log exploration and dashboards.

# 5. Create Monitoring Namespace

Create a dedicated namespace:

```bash
kubectl create namespace monitoring
```

Verify:

```bash
kubectl get namespaces
```

Expected:

```text
monitoring
```

All monitoring components will be deployed into this namespace unless a component specifically requires another namespace.

# 6. Verify EKS Cluster

Before installation, confirm that we are connected to the correct Production cluster.

```bash
aws sts get-caller-identity
```

Check Kubernetes context:

```bash
kubectl config current-context
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Check cluster resources:

```bash
kubectl get pods -A
```

Never install monitoring components into Production without verifying the AWS account and Kubernetes context.

# 7. Prometheus Architecture

Prometheus contains several important components.

```text
             PROMETHEUS SERVER
                    |
       +------------+------------+
       |            |            |
       v            v            v
   RETRIEVAL     STORAGE       QUERY
       |            |            |
       v            v            v
   SCRAPING      TSDB         PROMQL
       |
       v
   METRICS TARGETS
```

Important components:

| Component | Responsibility |
|---|---|
| Prometheus Server | Main monitoring server |
| Retrieval | Scrapes configured targets |
| TSDB | Stores time-series metrics |
| PromQL | Queries metrics |
| Service Discovery | Discovers monitoring targets |
| Alerting Rules | Evaluates alert conditions |
| Alertmanager | Routes notifications |

# 8. Understand Prometheus Pull Model

Suppose our User microservice exposes metrics at:

```text
http://user-service:8080/metrics
```

Prometheus periodically requests:

```http
GET /metrics
```

The application returns:

```text
http_requests_total{method="GET",status="200"} 1250
http_requests_total{method="POST",status="201"} 350
```

Prometheus stores the samples as time-series data.

We can query the data using PromQL.

Example:

```promql
http_requests_total
```

Prometheus normally scrapes targets at configured intervals.

Example:

```yaml
scrape_interval: 30s
```

This means Prometheus attempts to collect metrics every 30 seconds.

# 9. Install Prometheus Operator Using kube-prometheus-stack

For our EKS environment, we will use the kube-prometheus-stack Helm chart.

It provides an integrated monitoring installation containing:

- Prometheus Operator.
- Prometheus.
- Alertmanager.
- Grafana.
- Node Exporter.
- kube-state-metrics.
- Prometheus rules.
- ServiceMonitor CRDs.

The exact components and chart defaults depend on the pinned chart version.

## 9.1 Add Helm Repository

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Search:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

For Production, select and pin an approved chart version.

## 9.2 Create Values File

Create:

```text
monitoring/
├── prometheus/
│   └── values-production.yaml
```

Example:

`monitoring/prometheus/values-production.yaml`

```yaml
prometheus:
  prometheusSpec:
    replicas: 2

    retention: 15d

    retentionSize: 40GB

    scrapeInterval: 30s

    evaluationInterval: 30s

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
              storage: 50Gi

    serviceMonitorSelectorNilUsesHelmValues: false

    podMonitorSelectorNilUsesHelmValues: false

    ruleSelectorNilUsesHelmValues: false

grafana:
  enabled: true

  adminPassword: CHANGE_ME_BEFORE_DEPLOYMENT

  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi

  resources:
    requests:
      cpu: 250m
      memory: 512Mi

    limits:
      cpu: "1"
      memory: 1Gi

alertmanager:
  enabled: true

  alertmanagerSpec:
    replicas: 2

    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi

prometheusOperator:
  enabled: true

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

Important Production considerations:

- Verify the chart version and values schema.
- Do not store Grafana passwords in Git.
- Use Kubernetes Secrets or an approved external secrets solution.
- Verify the EKS StorageClass exists.
- Configure anti-affinity and topology spread where appropriate.
- Review the chart's default selectors and service discovery.
- Configure backup and recovery for persistent data.
- Two replicas do not automatically provide fully independent, deduplicated Prometheus HA storage.

The values above are a starting configuration, not a final capacity design.

## 9.3 Install the Chart

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version <APPROVED_CHART_VERSION> \
  -f monitoring/prometheus/values-production.yaml
```

If the release already exists, use a controlled Helm upgrade:

```bash
helm upgrade monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version <APPROVED_CHART_VERSION> \
  -f monitoring/prometheus/values-production.yaml
```

In our GitOps implementation, commit the Helm configuration to Git and let Argo CD manage the release instead of running unmanaged Helm changes.

# 10. Verify Prometheus Installation

Check pods:

```bash
kubectl get pods -n monitoring
```

Check services:

```bash
kubectl get services -n monitoring
```

Check StatefulSets:

```bash
kubectl get statefulsets -n monitoring
```

Check PersistentVolumeClaims:

```bash
kubectl get pvc -n monitoring
```

Check CRDs:

```bash
kubectl get crds | grep monitoring.coreos.com
```

Expected CRDs include resources such as:

```text
servicemonitors.monitoring.coreos.com
podmonitors.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
alertmanagers.monitoring.coreos.com
```

# 11. Access Prometheus UI

Find the Prometheus service:

```bash
kubectl get svc -n monitoring
```

Forward the port:

```bash
kubectl port-forward \
  -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090
```

Open:

```text
http://localhost:9090
```

Check:

- Status.
- Targets.
- Rules.
- Graph.
- Alerts.
- Configuration.

Do not expose the Prometheus UI publicly without an approved authentication and network security design.

# 12. Prometheus Targets

Open:

```text
Status → Targets
```

Prometheus displays:

- Target name.
- Scrape URL.
- Scrape interval.
- Last scrape.
- Scrape duration.
- Health status.

Example:

```text
Target                         State

node-exporter                  UP
kube-state-metrics             UP
prometheus                     UP
application-user               UP
application-cart               UP
application-orders             UP
```

A target marked DOWN indicates a scrape failure.

Investigate:

```bash
kubectl get endpoints -n monitoring
```

Check ServiceMonitor resources:

```bash
kubectl get servicemonitors -A
```

Check Prometheus logs:

```bash
kubectl logs \
  -n monitoring \
  statefulset/monitoring-kube-prometheus-prometheus
```

# 13. Node Exporter

Node Exporter collects host-level Linux metrics.

In Kubernetes, it is normally deployed as a DaemonSet so that an exporter runs on each eligible node.

Architecture:

```text
             EKS CLUSTER

      NODE 1       NODE 2       NODE 3
        |            |            |
        v            v            v
    EXPORTER     EXPORTER     EXPORTER
        |            |            |
        +------------+------------+
                     |
                     v
                 PROMETHEUS
                     |
                     v
                  GRAFANA
```

The kube-prometheus-stack chart can install Node Exporter.

Verify:

```bash
kubectl get daemonsets -n monitoring
```

Check pods:

```bash
kubectl get pods -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter
```

Check service:

```bash
kubectl get svc -n monitoring \
  | grep node-exporter
```

Node Exporter commonly exposes metrics on port 9100.

# 14. Node Exporter Metrics

Useful metrics:

## CPU

```promql
node_cpu_seconds_total
```

CPU utilization percentage, excluding idle time:

```promql
100 *
(
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{
      mode="idle"
    }[5m])
  )
)
```

## Memory

Memory utilization percentage:

```promql
100 *
(
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

## Disk

Filesystem usage percentage:

```promql
100 *
(
  1 -
  node_filesystem_avail_bytes{
    fstype!~"tmpfs|overlay"
  }
  /
  node_filesystem_size_bytes{
    fstype!~"tmpfs|overlay"
  }
)
```

## Network

Receive bytes per second:

```promql
rate(node_network_receive_bytes_total[5m])
```

Transmit bytes per second:

```promql
rate(node_network_transmit_bytes_total[5m])
```

Exclude loopback and virtual interfaces as appropriate for your environment.

# 15. kube-state-metrics

kube-state-metrics monitors the state of Kubernetes objects.

It reads Kubernetes API objects and exposes their state as Prometheus metrics.

Examples:

- Deployment replicas.
- Pod status.
- Node status.
- Job status.
- StatefulSet replicas.
- PersistentVolumeClaim status.

Verify:

```bash
kubectl get deployment -n monitoring \
  | grep kube-state-metrics
```

Check pods:

```bash
kubectl get pods -n monitoring \
  -l app.kubernetes.io/name=kube-state-metrics
```

Example queries:

Deployment desired replicas:

```promql
kube_deployment_spec_replicas
```

Available replicas:

```promql
kube_deployment_status_replicas_available
```

Unavailable replicas:

```promql
kube_deployment_spec_replicas
-
kube_deployment_status_replicas_available
```

Pod container restart count:

```promql
kube_pod_container_status_restarts_total
```

Pending pods:

```promql
kube_pod_status_phase{
  phase="Pending"
} == 1
```

# 16. ServiceMonitor

A ServiceMonitor tells Prometheus Operator how to discover and scrape metrics exposed through Kubernetes Services.

Example application Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: user-service
  namespace: production

  labels:
    app: user

spec:
  selector:
    app: user

  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
```

Example ServiceMonitor:

`user-servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: user-servicemonitor
  namespace: production

  labels:
    release: monitoring

spec:
  namespaceSelector:
    matchNames:
      - production

  selector:
    matchLabels:
      app: user

  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Important:

- The ServiceMonitor selector matches Service labels.
- The endpoint port refers to the Service port name.
- The path must exist in the application.
- The Prometheus resource must select this ServiceMonitor.
- Namespace selection must allow Prometheus to discover it.
- Network policies must permit scraping.

Apply:

```bash
kubectl apply -f user-servicemonitor.yaml
```

Verify:

```bash
kubectl get servicemonitor -n production
```

Check Prometheus Targets.

# 17. PodMonitor

A PodMonitor allows Prometheus Operator to scrape pods directly without requiring a Service.

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor

metadata:
  name: user-podmonitor
  namespace: production

  labels:
    release: monitoring

spec:
  namespaceSelector:
    matchNames:
      - production

  selector:
    matchLabels:
      app: user

  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Use PodMonitor when direct pod discovery is appropriate.

Prefer ServiceMonitor when your monitoring design uses Services as the discovery interface.

Do not create duplicate scraping configurations for the same target unless intentional.

# 18. Application Metrics

Infrastructure metrics tell us whether the cluster is healthy.

Application metrics tell us whether the business application is functioning correctly.

For our microservices platform, we need to monitor:

- Total requests.
- Successful requests.
- Failed requests.
- Request latency.
- Active requests.
- Database connections.
- Database pool usage.
- Cache hit and miss counts.
- Message processing.
- Queue depth.
- Application exceptions.

Example metric names:

```text
http_requests_total
http_request_duration_seconds
http_requests_in_progress
database_connections_active
database_connections_idle
```

Metric names depend on the application instrumentation.

# 19. Spring Boot Application Metrics

For Spring Boot applications, Micrometer and Spring Boot Actuator can expose application metrics.

Add the required dependencies to the Maven project.

Example `pom.xml` dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Configure:

`application.yml`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,metrics

  endpoint:
    health:
      probes:
        enabled: true

  prometheus:
    metrics:
      export:
        enabled: true
```

The exact configuration may vary by Spring Boot version.

Verify the endpoint:

```bash
curl http://localhost:8080/actuator/prometheus
```

Expected metrics include JVM, HTTP server, and application instrumentation metrics where enabled.

Do not expose all Actuator endpoints publicly.

Restrict management endpoints through network controls and authentication.

# 20. Node.js Application Metrics

For Node.js services, use an appropriate Prometheus client library.

Install:

```bash
npm install prom-client
```

Example:

`metrics.js`

```javascript
const client = require('prom-client');

const register = new client.Registry();

client.collectDefaultMetrics({
  register
});

const httpRequests = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']
});

register.registerMetric(httpRequests);

const requestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2, 5]
});

register.registerMetric(requestDuration);

module.exports = {
  register,
  httpRequests,
  requestDuration
};
```

Integrate metrics into the application:

```javascript
const express = require('express');

const {
  register,
  httpRequests,
  requestDuration
} = require('./metrics');

const app = express();

app.use((req, res, next) => {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const duration =
      Number(process.hrtime.bigint() - start) / 1e9;

    const route = req.route?.path || 'unmatched';

    httpRequests.inc({
      method: req.method,
      route,
      status: String(res.statusCode)
    });

    requestDuration.observe({
      method: req.method,
      route
    }, duration);
  });

  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);

  res.end(await register.metrics());
});

app.listen(8080);
```

In Production, ensure:

- Metrics middleware is registered correctly.
- Route labels use normalized route templates.
- Sensitive data is not included in labels.
- Cardinality remains controlled.
- The metrics endpoint is protected from public access.

# 21. Custom Prometheus Metrics

We can expose custom business metrics.

Example:

```text
orders_created_total
payments_processed_total
inventory_updates_total
```

For example, a business metric could track completed orders.

A metric must have a clearly defined meaning and update behavior.

Avoid labels containing:

- User IDs.
- Order IDs.
- Email addresses.
- Session IDs.
- Raw URLs with unique identifiers.

These create high cardinality and can cause excessive Prometheus memory consumption.

# 22. Prometheus Query Language

PromQL is used to query Prometheus metrics.

## 22.1 Instant Query

```promql
up
```

Returns current target status.

## 22.2 Rate

```promql
rate(http_requests_total[5m])
```

Calculates the per-second average rate of increase over five minutes.

## 22.3 Increase

```promql
increase(http_requests_total[1h])
```

Calculates the increase over one hour.

## 22.4 Aggregation

```promql
sum by (namespace) (
  rate(http_requests_total[5m])
)
```

## 22.5 Top CPU-Consuming Pods

```promql
topk(
  10,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{
      container!="",
      container!="POD"
    }[5m])
  )
)
```

The query may need filtering for the namespaces and containers relevant to the environment.

# 23. Grafana Installation

Grafana is included in the kube-prometheus-stack installation.

Verify:

```bash
kubectl get pods -n monitoring \
  -l app.kubernetes.io/name=grafana
```

Find the service:

```bash
kubectl get svc -n monitoring \
  | grep grafana
```

Forward the port:

```bash
kubectl port-forward \
  -n monitoring \
  svc/monitoring-grafana \
  3000:80
```

Open:

```text
http://localhost:3000
```

Retrieve the initial administrator password from the Kubernetes Secret created by the chart.

Example:

```bash
kubectl get secret monitoring-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' \
  | base64 -d

echo
```

The Secret name can differ depending on the Helm release.

Do not place credentials in source control or shell history.

# 24. Configure Prometheus Data Source

Grafana needs a Prometheus data source.

When both are installed using kube-prometheus-stack, the chart normally configures the Prometheus data source automatically.

Verify:

```text
Grafana
  |
  v
Connections
  |
  v
Data Sources
  |
  v
Prometheus
```

Example in-cluster URL:

```text
http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090
```

The exact Service name depends on the Helm release.

Click Save and Test.

# 25. Grafana Production Dashboards

We need dashboards for different operational responsibilities.

Recommended dashboards:

| Dashboard | Purpose |
|---|---|
| EKS Cluster Overview | Overall cluster health |
| Kubernetes Nodes | CPU, memory, disk, network |
| Kubernetes Pods | Pod readiness and restarts |
| Deployments | Desired and available replicas |
| Application Overview | Request rate and errors |
| API Latency | p50, p95, p99 latency |
| Database | Connections and query health |
| Redis | Memory and cache performance |
| RabbitMQ | Queue depth and consumers |
| Kafka | Consumer lag and broker health |
| Alert Overview | Active and firing alerts |

Use dashboards that match the actual metrics and labels exposed by our platform.

# 26. Kubernetes Cluster Dashboard

Useful PromQL queries:

Node CPU utilization:

```promql
100 *
(
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{
      mode="idle"
    }[5m])
  )
)
```

Node memory utilization:

```promql
100 *
(
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

Node count:

```promql
count(kube_node_info)
```

Ready nodes:

```promql
sum(
  kube_node_status_condition{
    condition="Ready",
    status="true"
  }
)
```

Pending pods:

```promql
sum(
  kube_pod_status_phase{
    phase="Pending"
  } == 1
)
```

# 27. Pod Monitoring Dashboard

Pod restart count:

```promql
sum by (namespace, pod) (
  increase(kube_pod_container_status_restarts_total[1h])
)
```

Pods not Ready:

```promql
sum by (namespace) (
  kube_pod_status_ready{
    condition="false"
  } == 1
)
```

Container CPU:

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{
    container!="",
    container!="POD"
  }[5m])
)
```

Container memory:

```promql
container_memory_working_set_bytes{
  container!="",
  container!="POD"
}
```

# 28. Application Performance Dashboard

Monitor:

- Request rate.
- Error rate.
- p50 latency.
- p95 latency.
- p99 latency.
- Active requests.
- Request methods.
- Response codes.

Request rate:

```promql
sum by (app) (
  rate(http_requests_total[5m])
)
```

HTTP 5xx rate:

```promql
sum by (app) (
  rate(http_requests_total{
    status=~"5.."
  }[5m])
)
```

Error ratio:

```promql
sum(
  rate(http_requests_total{
    status=~"5.."
  }[5m])
)
/
clamp_min(
  sum(
    rate(http_requests_total[5m])
  ),
  1
)
```

Latency:

```promql
histogram_quantile(
  0.95,
  sum by (le, app) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

These queries assume the example metric names and labels are present.

# 29. Database Monitoring

Our microservices platform uses databases and caches as shared dependencies.

Database monitoring must include:

- Connection count.
- Active connections.
- Idle connections.
- Connection pool saturation.
- Query latency.
- Database CPU.
- Memory.
- Storage.
- Replication health where applicable.
- Connection errors.

For AWS RDS, use an approved database exporter or AWS-managed monitoring integration.

Do not expose database credentials in exporter configuration stored in Git.

Use Secrets or an approved secrets manager.

For application-level connection pool monitoring, expose metrics from the connection pool.

For HikariCP, Spring Boot Actuator and Micrometer can expose pool metrics when the relevant instrumentation is enabled.

Example:

```promql
hikaricp_connections_active
```

```promql
hikaricp_connections_pending
```

```promql
hikaricp_connections_max
```

The exact metric names depend on the Micrometer version and instrumentation.

# 30. Redis Monitoring

Redis monitoring should include:

- Memory usage.
- Connected clients.
- Commands per second.
- Cache hit ratio.
- Cache miss ratio.
- Evictions.
- Keyspace information.
- Replication health.

A Redis exporter can expose Redis metrics to Prometheus.

Example queries, depending on exporter version:

```promql
redis_connected_clients
```

```promql
redis_memory_used_bytes
```

```promql
rate(redis_commands_processed_total[5m])
```

Cache hit ratio:

```promql
rate(redis_keyspace_hits_total[5m])
/
clamp_min(
  rate(redis_keyspace_hits_total[5m])
  +
  rate(redis_keyspace_misses_total[5m]),
  1
)
```

# 31. RabbitMQ Monitoring

Monitor:

- Queue depth.
- Ready messages.
- Unacknowledged messages.
- Consumer count.
- Publish rate.
- Delivery rate.
- Node health.
- Memory alarms.
- Disk alarms.

Use RabbitMQ's supported Prometheus metrics endpoint or an approved exporter.

Example metrics may include:

```text
rabbitmq_queue_messages_ready
rabbitmq_queue_messages_unacked
rabbitmq_queue_consumers
```

Check queue growth:

```promql
rabbitmq_queue_messages_ready
```

A growing queue can indicate that consumers are slower than producers.

Investigate:

- Consumer availability.
- Processing failures.
- Message retries.
- Dead-letter queues.
- Database performance.
- Application resource saturation.

# 32. Kafka Monitoring

Kafka monitoring should include:

- Broker availability.
- Under-replicated partitions.
- Offline partitions.
- Consumer lag.
- Request latency.
- Network throughput.
- Disk utilization.
- Replication health.

Consumer lag is especially important.

It represents the difference between the latest available offset and the consumer's committed or current processing offset, depending on the monitoring metric.

Example:

```text
Topic: orders
Consumer group: orders-service

Latest offset: 10000
Committed offset: 8500

Lag: 1500
```

A growing lag indicates that consumers may not be keeping up with incoming messages.

Use Kafka's supported metrics or an approved exporter.

# 33. ELK Stack Architecture

Now we implement centralized logging.

```text
            EKS CLUSTER
                 |
       +---------+---------+
       |         |         |
       v         v         v
    USER POD  CART POD  ORDER POD
       |         |         |
       v         v         v
   CONTAINER LOGS / STDOUT
       |         |         |
       +---------+---------+
                 |
                 v
              FILEBEAT
                 |
                 v
              LOGSTASH
                 |
                 v
           ELASTICSEARCH
                 |
                 v
               KIBANA
```

The goal is to search logs from all microservices in one place.

# 34. Understand Container Logging

Applications running in Kubernetes should generally write logs to standard output and standard error.

Example:

```text
2026-11-03T10:15:30Z INFO User service started
2026-11-03T10:15:35Z INFO Request received
2026-11-03T10:15:36Z ERROR Database connection failed
```

Kubernetes container runtimes capture these logs.

We can inspect them using:

```bash
kubectl logs <POD_NAME> -n production
```

For multiple containers:

```bash
kubectl logs <POD_NAME> \
  -c <CONTAINER_NAME> \
  -n production
```

For previous container instances:

```bash
kubectl logs <POD_NAME> \
  --previous \
  -n production
```

Centralized logging helps us search across services without manually accessing each pod.

# 35. Elasticsearch

Elasticsearch stores indexed log data.

Example document:

```json
{
  "@timestamp": "2026-11-03T10:15:36Z",
  "service": "user",
  "environment": "production",
  "level": "ERROR",
  "message": "Database connection failed",
  "namespace": "production",
  "pod": "user-7d6f8b"
}
```

Production considerations:

- Storage sizing.
- Index lifecycle management.
- Replicas.
- Snapshots.
- Access control.
- Encryption.
- Retention.
- Cluster health.
- JVM memory.
- Disk watermarks.

For production, deploy Elasticsearch using a supported and maintained operator or managed service architecture.

Do not run a single-node Elasticsearch deployment as a highly available Production logging platform.

# 36. Logstash

Logstash receives, parses, transforms, and forwards logs.

Example pipeline:

`logstash/pipeline/logstash.conf`

```text
input {
  beats {
    port => 5044
  }
}

filter {
  if [message] =~ "ERROR" {
    mutate {
      add_field => {
        "severity" => "high"
      }
    }
  }

  if [message] =~ "WARN" {
    mutate {
      add_field => {
        "severity" => "medium"
      }
    }
  }

  mutate {
    add_field => {
      "environment" => "production"
    }
  }
}

output {
  elasticsearch {
    hosts => [
      "http://elasticsearch:9200"
    ]

    index => "roboshop-logs-%{+YYYY.MM.dd}"
  }
}
```

This is a basic parsing example.

In Production:

- Configure authentication.
- Enable TLS.
- Avoid hardcoded credentials.
- Validate input formats.
- Parse JSON application logs where possible.
- Configure persistent queues when required.
- Configure retry and dead-letter handling.
- Apply index lifecycle policies.
- Avoid unbounded field creation.

# 37. Filebeat

Filebeat collects logs and forwards them to Logstash.

In Kubernetes, Filebeat is commonly deployed as a DaemonSet.

It reads container log files from the worker nodes.

Example configuration:

`filebeat.yml`

```yaml
filebeat.inputs:
  - type: filestream

    id: kubernetes-container-logs

    paths:
      - /var/log/containers/*.log

    parsers:
      - container:
          stream: all
          format: auto

output.logstash:
  hosts:
    - "logstash.monitoring.svc:5044"
```

The actual Kubernetes metadata enrichment, container runtime handling, and mount configuration must be added for the selected Filebeat version and deployment method.

For Production, use the official version-matched Filebeat Kubernetes manifests or Helm chart and configure the required hostPath mounts, permissions, and metadata processors.

# 38. Deploy Filebeat

Filebeat can be deployed as a DaemonSet.

Conceptual architecture:

```text
NODE 1
  |
  +-- USER POD LOGS
  +-- CART POD LOGS
  |
  +-- FILEBEAT

NODE 2
  |
  +-- ORDER POD LOGS
  +-- PAYMENT POD LOGS
  |
  +-- FILEBEAT

NODE 3
  |
  +-- INVENTORY POD LOGS
  +-- NOTIFICATION POD LOGS
  |
  +-- FILEBEAT
```

Verify:

```bash
kubectl get daemonsets -n monitoring
```

Check:

```bash
kubectl get pods -n monitoring \
  -l app=filebeat
```

Check logs:

```bash
kubectl logs daemonset/filebeat \
  -n monitoring
```

The exact resource name depends on the installed release.

# 39. Kibana

Kibana provides log search and visualization.

Access Kibana using the approved internal endpoint or port forwarding.

Example:

```bash
kubectl port-forward \
  -n monitoring \
  svc/kibana \
  5601:5601
```

Open:

```text
http://localhost:5601
```

Create a data view matching:

```text
roboshop-logs-*
```

Select the timestamp field:

```text
@timestamp
```

Search examples:

Find errors:

```text
level: ERROR
```

Find User service errors:

```text
service: user AND level: ERROR
```

Find database failures:

```text
message: "Database connection failed"
```

Find Production logs:

```text
environment: production
```

The search syntax depends on the configured Kibana data view and field mappings.

# 40. Structured Logging

In Production, structured JSON logging is preferable to unstructured messages.

Example:

```json
{
  "timestamp": "2026-11-03T10:15:36Z",
  "level": "ERROR",
  "service": "orders",
  "environment": "production",
  "message": "Payment processing failed",
  "request_id": "example-request-id",
  "trace_id": null
}
```

This example uses a request correlation identifier.

We are not implementing distributed tracing in this chapter.

Avoid logging:

- Passwords.
- Access tokens.
- Secret keys.
- Database credentials.
- Payment card details.
- Sensitive personal data.

# 41. Correlating Logs and Metrics

Suppose a customer reports that an order failed.

We should be able to investigate:

```text
1. Find the request in application logs.
2. Identify the affected microservice.
3. Check its HTTP error metrics.
4. Check pod restart history.
5. Check database connection metrics.
6. Check RabbitMQ or Kafka processing metrics.
7. Identify the failure.
```

Use common labels and identifiers where appropriate:

- Service name.
- Environment.
- Namespace.
- Pod.
- Application version.
- Request ID.

Avoid using unique request IDs as Prometheus labels because this can create high-cardinality time series.

Request IDs are appropriate in logs, not usually as metric labels.

# 42. Alertmanager

Prometheus evaluates alerting rules.

When an alert fires, Prometheus sends it to Alertmanager.

Alertmanager handles:

- Grouping.
- Deduplication.
- Routing.
- Silencing.
- Inhibition.
- Notification delivery.

Architecture:

```text
            PROMETHEUS
                 |
                 v
            ALERT RULES
                 |
                 v
            ALERTMANAGER
                 |
          +------+------+
          |             |
          v             v
        EMAIL        CHAT TOOL
```

# 43. Create Production Alert Rules

Create:

`monitoring/alerts/production-alerts.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: production-alerts
  namespace: monitoring

  labels:
    release: monitoring

spec:
  groups:
    - name: infrastructure-alerts

      rules:
        - alert: NodeHighCPU

          expr: |
            100 *
            (
              1 -
              avg by (instance) (
                rate(node_cpu_seconds_total{
                  mode="idle"
                }[5m])
              )
            ) > 85

          for: 10m

          labels:
            severity: warning

          annotations:
            summary: "High CPU utilization on node"
            description: "Node CPU utilization is above 85% for 10 minutes."

        - alert: NodeHighMemory

          expr: |
            100 *
            (
              1 -
              node_memory_MemAvailable_bytes
              /
              node_memory_MemTotal_bytes
            ) > 90

          for: 10m

          labels:
            severity: critical

          annotations:
            summary: "High memory utilization"
            description: "Node memory utilization is above 90%."

        - alert: PodRestartIncrease

          expr: |
            increase(
              kube_pod_container_status_restarts_total[15m]
            ) > 3

          for: 5m

          labels:
            severity: warning

          annotations:
            summary: "Container restart increase detected"
            description: "A container restarted more than three times in 15 minutes."

        - alert: DeploymentUnavailableReplicas

          expr: |
            kube_deployment_spec_replicas
            -
            kube_deployment_status_replicas_available > 0

          for: 10m

          labels:
            severity: critical

          annotations:
            summary: "Deployment replicas unavailable"
            description: "A deployment has fewer available replicas than desired."

        - alert: PrometheusTargetDown

          expr: up == 0

          for: 5m

          labels:
            severity: warning

          annotations:
            summary: "Monitoring target is down"
            description: "A Prometheus scrape target has been unavailable for five minutes."
```

Apply:

```bash
kubectl apply -f \
  monitoring/alerts/production-alerts.yaml
```

Verify:

```bash
kubectl get prometheusrules -n monitoring
```

The Prometheus instance must select the rule resource through its configured rule selectors.

Validate expressions against the actual metrics and labels in your environment.

# 44. Alertmanager Configuration

Alertmanager can route alerts by severity and ownership.

Example:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default-receiver

  group_by:
    - alertname
    - namespace

  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - matchers:
        - severity="critical"

      receiver: critical-receiver

receivers:
  - name: default-receiver

    email_configs:
      - to: devops@example.com
        from: alerts@example.com
        smarthost: smtp.example.com:587
        auth_username: alerts@example.com
        auth_password: REPLACE_WITH_SECRET
        require_tls: true

  - name: critical-receiver

    email_configs:
      - to: oncall@example.com
        from: alerts@example.com
        smarthost: smtp.example.com:587
        auth_username: alerts@example.com
        auth_password: REPLACE_WITH_SECRET
        require_tls: true
```

This is an illustrative configuration.

For Production:

- Store SMTP credentials securely.
- Use a supported Alertmanager configuration mechanism.
- Validate configuration before applying.
- Configure notification delivery and retries.
- Define ownership and escalation policies.
- Avoid routing every warning directly to the on-call engineer.

# 45. Alert Severity

Use consistent severity levels.

| Severity | Meaning | Example |
|---|---|---|
| Critical | Immediate customer or service impact | Production service unavailable |
| Warning | Potential or developing issue | CPU above threshold |
| Info | Operational information | Deployment completed |

An alert should have:

- Clear name.
- Description.
- Impact.
- Severity.
- Owner.
- Runbook.
- Action required.

Avoid alerts that fire without a clear operational action.

# 46. Golden Signals

The four golden signals are:

1. Latency.
2. Traffic.
3. Errors.
4. Saturation.

## Latency

How long does a request take?

Example:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

## Traffic

How much demand is reaching the application?

```promql
sum(
  rate(http_requests_total[5m])
)
```

## Errors

How many requests fail?

```promql
sum(
  rate(http_requests_total{
    status=~"5.."
  }[5m])
)
```

## Saturation

How close is the service to its capacity?

Examples:

- CPU utilization.
- Memory utilization.
- Database connection pool usage.
- Queue depth.
- Kafka consumer lag.

# 47. SLI and SLO Implementation

An SLI is a measured indicator of service performance.

An SLO is the target for that indicator.

Example:

```text
Service: User API

Availability SLI:
Successful eligible requests / Total eligible requests

SLO:
99.9% over the agreed measurement window
```

For latency:

```text
Latency SLI:
Percentage of eligible requests completed
within the approved latency threshold.
```

Do not treat an HTTP 200 response alone as proof of business success.

The SLI must represent the service behavior that matters to customers.

# 48. Error Budget

Suppose an availability SLO is 99.9%.

The allowed error budget is 0.1% over the same measurement window.

For a 30-day month:

```text
30 days × 24 hours × 60 minutes = 43,200 minutes

0.1% of 43,200 minutes = 43.2 minutes
```

This is an illustrative time-based approximation.

For request-based availability SLOs, calculate the error budget using eligible request counts rather than simply converting it to minutes.

When the error budget is consumed too quickly, the organization may:

- Reduce risky releases.
- Improve reliability work.
- Fix recurring incidents.
- Review capacity.
- Improve deployment safeguards.

# 49. Production Monitoring Runbook

## High CPU

Check:

```bash
kubectl top nodes
```

```bash
kubectl top pods -n production
```

Investigate:

- CPU-intensive application code.
- Traffic spikes.
- Unexpected loops.
- Resource requests and limits.
- HPA behavior.
- Node capacity.

## High Memory

Check:

```bash
kubectl top pods -n production
```

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

Investigate:

- Memory leaks.
- JVM heap.
- Container memory limits.
- Large payloads.
- Cache growth.
- OOMKilled events.

## Pod Restarts

Check:

```bash
kubectl get pods -n production
```

Check:

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

Check previous logs:

```bash
kubectl logs <POD_NAME> \
  --previous \
  -n production
```

## High API Latency

Investigate:

1. Request volume.
2. Application CPU.
3. Application memory.
4. Database latency.
5. Connection pool usage.
6. Cache behavior.
7. Downstream service latency.
8. Network conditions.

## High Queue Depth

Investigate:

1. Producer rate.
2. Consumer count.
3. Consumer errors.
4. Processing time.
5. Database bottlenecks.
6. Retry behavior.
7. Dead-letter queue growth.

# 50. Monitoring Failure Simulation

We must validate the monitoring platform itself.

## Scenario 1 — CPU Alert

Generate controlled CPU load in a test environment.

Do not run resource-exhaustion tests on Production without explicit approval and capacity safeguards.

Example test workload:

```bash
kubectl run cpu-test \
  --image=busybox \
  --restart=Never \
  --namespace=testing \
  -- sh -c 'while true; do :; done'
```

Observe:

```bash
kubectl top pods -n testing
```

Check Prometheus and Grafana.

Clean up:

```bash
kubectl delete pod cpu-test -n testing
```

## Scenario 2 — Pod Restart Alert

In a test namespace, deploy a controlled test container that exits repeatedly.

Observe:

```bash
kubectl get pods -n testing -w
```

Check restart metrics.

## Scenario 3 — Prometheus Target Down

In a test environment, temporarily remove or disable a test scrape target.

Observe:

- Prometheus Targets.
- `up` metric.
- Alert firing.
- Alertmanager notification.

Restore the target after testing.

## Scenario 4 — Log Pipeline Failure

In a test environment:

1. Generate test logs.
2. Temporarily stop the Logstash test instance.
3. Observe Filebeat output errors.
4. Restore Logstash.
5. Verify whether buffered events are delivered.
6. Check for lost events.

The expected behavior depends on the configured buffering and retry settings.

# 51. Monitoring Troubleshooting

## Prometheus Target DOWN

Possible reasons:

- ServiceMonitor selector mismatch.
- Wrong Service label.
- Wrong port name.
- Incorrect metrics path.
- Application endpoint unavailable.
- NetworkPolicy blocking access.
- TLS configuration mismatch.
- Service has no endpoints.

Commands:

```bash
kubectl get servicemonitors -A
```

```bash
kubectl get services -n production
```

```bash
kubectl get endpointslices -n production
```

## Grafana No Data

Possible reasons:

- Incorrect PromQL.
- Prometheus target down.
- Incorrect time range.
- Wrong data source.
- Metric name mismatch.
- Label mismatch.

## Elasticsearch Disk Watermark

Possible reasons:

- Excessive log ingestion.
- Retention policy not configured.
- Old indices not deleted.
- Insufficient storage.

Investigate disk usage and index lifecycle policies before deleting data.

## Filebeat Not Sending Logs

Check:

```bash
kubectl logs daemonset/filebeat \
  -n monitoring
```

Investigate:

- Log file permissions.
- HostPath mounts.
- Logstash connectivity.
- TLS certificates.
- Authentication.
- Configuration errors.
- Network policies.

# 52. Monitoring Security

Monitoring systems contain sensitive operational information.

Apply:

1. Kubernetes RBAC.
2. Restricted access to Prometheus.
3. Grafana authentication.
4. TLS for service communication.
5. Secure Elasticsearch access.
6. Log masking.
7. Secret management.
8. Network policies.
9. Audit logging.
10. Data retention policies.

Do not expose monitoring services publicly without an approved security architecture.

# 53. Production Capacity and Retention

Monitoring systems themselves require resources.

Prometheus storage depends on:

- Active time series.
- Scrape interval.
- Retention period.
- Label cardinality.
- Sample ingestion rate.

Elasticsearch storage depends on:

- Log ingestion volume.
- Replication.
- Retention.
- Index size.
- Compression.
- Query workload.

Before finalizing Production capacity, measure actual ingestion and growth.

Do not blindly use the example storage values in this chapter.

# 54. Monitoring Git Repository Structure

Maintain monitoring configurations in Git.

Example:

```text
roboshop-platform/
|
├── monitoring/
|
|   ├── prometheus/
|   |   ├── values-production.yaml
|   |   ├── values-staging.yaml
|   |   └── servicemonitors/
|   |
|   ├── grafana/
|   |   ├── dashboards/
|   |   └── datasources/
|   |
|   ├── alerts/
|   |   ├── infrastructure-alerts.yaml
|   |   ├── application-alerts.yaml
|   |   └── database-alerts.yaml
|   |
|   ├── alertmanager/
|   |   └── values-production.yaml
|   |
|   ├── elasticsearch/
|   |
|   ├── logstash/
|   |   └── pipeline/
|   |
|   ├── filebeat/
|   |   └── values-production.yaml
|   |
|   └── README.md
```

Store credentials outside Git.

Use Argo CD to reconcile the monitoring configuration into the cluster.

# 55. Production Monitoring Validation Checklist

## Infrastructure

- [ ] Node Exporter running.
- [ ] kube-state-metrics running.
- [ ] Node CPU metrics available.
- [ ] Node memory metrics available.
- [ ] Disk metrics available.
- [ ] Network metrics available.

## Prometheus

- [ ] Prometheus installed.
- [ ] Prometheus Operator installed.
- [ ] Persistent storage configured.
- [ ] Targets discovered.
- [ ] ServiceMonitor validated.
- [ ] PodMonitor validated.
- [ ] PromQL queries tested.
- [ ] Retention configured.
- [ ] Resource usage reviewed.

## Application

- [ ] Spring Boot metrics exposed.
- [ ] Node.js metrics exposed.
- [ ] HTTP request metrics available.
- [ ] Error metrics available.
- [ ] Latency metrics available.
- [ ] Custom business metrics validated.
- [ ] High-cardinality labels reviewed.

## Grafana

- [ ] Grafana installed.
- [ ] Prometheus data source connected.
- [ ] Cluster dashboard created.
- [ ] Node dashboard created.
- [ ] Pod dashboard created.
- [ ] Application dashboard created.
- [ ] Database dashboard created.
- [ ] Messaging dashboard created.

## Logging

- [ ] Elasticsearch available.
- [ ] Logstash available.
- [ ] Filebeat available.
- [ ] Container logs collected.
- [ ] Application logs parsed.
- [ ] Kibana data view created.
- [ ] Search validated.
- [ ] Retention policy configured.
- [ ] Sensitive data protected.

## Alerting

- [ ] Alertmanager configured.
- [ ] Production alert rules created.
- [ ] Alert routing configured.
- [ ] Notifications tested.
- [ ] Severity definitions documented.
- [ ] Runbooks available.
- [ ] Alert ownership defined.

## Reliability

- [ ] SLI defined.
- [ ] SLO defined.
- [ ] Error budget calculated.
- [ ] Golden signals monitored.
- [ ] Monitoring failure tested.
- [ ] Recovery procedures documented.

# 56. Final Production Monitoring Architecture

```text
                  AWS EKS
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
  APPLICATION     KUBERNETES      NODES
    METRICS          STATE       METRICS
       |             |             |
       v             v             v
       +-------------+-------------+
                     |
                     v
                 PROMETHEUS
                     |
             +-------+-------+
             |               |
             v               v
          GRAFANA        ALERTMANAGER
             |               |
             v               v
         DASHBOARDS      NOTIFICATIONS


                LOG COLLECTION
                     |
                     v
                  FILEBEAT
                     |
                     v
                  LOGSTASH
                     |
                     v
               ELASTICSEARCH
                     |
                     v
                   KIBANA
```

# 57. Chapter Summary

In this chapter, we implemented the Production monitoring design for our 20+ microservices platform.

We covered:

- Production monitoring architecture.
- Prometheus.
- Prometheus Operator.
- kube-prometheus-stack.
- Node Exporter.
- kube-state-metrics.
- ServiceMonitor.
- PodMonitor.
- Spring Boot metrics.
- Node.js metrics.
- Custom metrics.
- PromQL.
- Grafana.
- Kubernetes dashboards.
- Application dashboards.
- Database monitoring.
- Redis monitoring.
- RabbitMQ monitoring.
- Kafka monitoring.
- Elasticsearch.
- Logstash.
- Filebeat.
- Kibana.
- Centralized logging.
- Alertmanager.
- Production alert rules.
- Golden signals.
- SLI and SLO.
- Error budgets.
- Monitoring runbooks.
- Failure simulations.
- Monitoring troubleshooting.
- Production security.
- Monitoring validation.

**Key principle:** Production monitoring must tell us not only whether infrastructure is running, but whether customers can successfully use our application.

---

# 58. Next Chapter

## CHAPTER 208 — PRODUCTION ALERTING AND INCIDENT RESPONSE

**Filename:** `208-Production-Alerting-Incident-Response.md`

Planned topics:

1. Production alerting architecture.
2. Alert strategy and ownership.
3. Alert severity and prioritization.
4. Prometheus alert rules.
5. Alertmanager routing.
6. Alert grouping and deduplication.
7. Inhibition and silencing.
8. Email and chat notifications.
9. On-call alert handling.
10. Alert fatigue reduction.
11. Incident classification.
12. Incident severity levels.
13. Incident response lifecycle.
14. Incident command and ownership.
15. Incident communication.
16. Production troubleshooting workflow.
17. Kubernetes incident scenarios.
18. AWS infrastructure incidents.
19. Database incidents.
20. Messaging incidents.
21. Application performance incidents.
22. Deployment failure incidents.
23. Incident timelines.
24. Root Cause Analysis.
25. Corrective and preventive actions.
26. Post-incident reviews.
27. Incident runbooks.
28. Incident simulations.
29. Recovery validation.
30. Production incident closure.

**Next file: `208-Production-Alerting-Incident-Response.md`.**