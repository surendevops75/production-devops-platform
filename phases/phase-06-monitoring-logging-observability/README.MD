# Phase 06 — Monitoring, Logging & Observability

## Overview

This phase covers monitoring, logging, and observability fundamentals for the microservices platform.

It introduces Prometheus for metrics collection, Grafana for visualization, Alertmanager for alert handling, Node Exporter and kube-state-metrics for infrastructure and Kubernetes monitoring, and Fluent Bit for log collection.

It also covers Logstash, Elasticsearch, and Kibana for centralized logging, along with OpenTelemetry and Jaeger as additional observability topics.

The objective is to understand how to collect metrics, monitor infrastructure and Kubernetes workloads, visualize system health, configure alerts, and centralize application logs.

## Objectives

- Understand monitoring and observability fundamentals.
- Learn Prometheus architecture and metrics collection.
- Create monitoring dashboards using Grafana.
- Understand alert management using Alertmanager.
- Monitor Linux infrastructure using Node Exporter.
- Monitor Kubernetes resources using kube-state-metrics.
- Understand log collection using Fluent Bit.
- Learn log processing and centralized storage using Logstash and Elasticsearch.
- Visualize and analyze logs using Kibana.
- Understand OpenTelemetry and Jaeger fundamentals.

## Chapter Index

| Chapter | Documentation |
|---|---|
| 30 | [Prometheus](30-Prometheus.md) |
| 31 | [Grafana](31-Grafana.md) |
| 32 | [Alertmanager](32-Alertmanager.md) |
| 33 | [Node Exporter](33-Node-Exporter.md) |
| 34 | [kube-state-metrics](34-kube-state-metrics.md) |
| 35 | [Fluent Bit](35-Fluent-Bit.md) |
| 36 | [Logstash](36-Logstash.md) |
| 37 | [Elasticsearch](37-Elasticsearch.md) |
| 38 | [Kibana](38-Kibana.md) |
| 39 | [OpenTelemetry](39-OpenTelemetry.md) |
| 40 | [Jaeger](40-Jaeger.md) |

## Topics Covered

### 1. Prometheus
- Prometheus architecture.
- Metrics collection and scraping.
- Targets and exporters.
- PromQL fundamentals.
- Monitoring infrastructure and applications.

### 2. Grafana
- Grafana architecture.
- Prometheus data source integration.
- Dashboard creation.
- Metrics visualization.
- Monitoring application and infrastructure health.

### 3. Alertmanager
- Alert routing and management.
- Alert rules and notification handling.
- Grouping and silencing alerts.
- Integrating Alertmanager with Prometheus.

### 4. Node Exporter
- Linux server metrics collection.
- CPU, memory, disk, and filesystem monitoring.
- Integrating Node Exporter with Prometheus.

### 5. kube-state-metrics
- Kubernetes object state metrics.
- Monitoring Pods, Deployments, and Nodes.
- Collecting Kubernetes resource information.
- Integrating kube-state-metrics with Prometheus.

### 6. Fluent Bit
- Log collection architecture.
- Collecting container and application logs.
- Log forwarding and processing.
- Integrating Fluent Bit with centralized logging systems.

### 7. Logstash
- Log ingestion and processing.
- Input, filter, and output plugins.
- Log parsing and transformation.
- Forwarding processed logs to Elasticsearch.

### 8. Elasticsearch
- Centralized log storage.
- Indexing and searching logs.
- Elasticsearch architecture.
- Log data management.

### 9. Kibana
- Kibana architecture.
- Elasticsearch integration.
- Log visualization and dashboards.
- Searching and analyzing application logs.

### 10. OpenTelemetry
- Observability concepts.
- Telemetry collection fundamentals.
- OpenTelemetry architecture.
- Understanding instrumentation and telemetry pipelines.

### 11. Jaeger
- Distributed tracing fundamentals.
- Trace and span concepts.
- Understanding service request paths.
- Analyzing latency across distributed services.

## Expected Outcome

After completing this phase, you should understand how monitoring and centralized logging support a microservices environment.

You should be able to explain how Prometheus collects metrics, Grafana visualizes them, Alertmanager handles alerts, exporters expose infrastructure and Kubernetes metrics, and the ELK components support centralized log processing and analysis.

You should also understand the purpose of OpenTelemetry and Jaeger in distributed observability.

These concepts prepare the project for practical observability implementation and production monitoring.

## Completion Checklist

- [ ] Prometheus fundamentals documented.
- [ ] Grafana dashboards and integration documented.
- [ ] Alertmanager configuration and alert handling documented.
- [ ] Node Exporter monitoring documented.
- [ ] kube-state-metrics integration documented.
- [ ] Fluent Bit log collection documented.
- [ ] Logstash processing documented.
- [ ] Elasticsearch storage and indexing documented.
- [ ] Kibana log visualization documented.
- [ ] OpenTelemetry fundamentals documented.
- [ ] Jaeger fundamentals documented.
- [ ] All eleven chapter files organized in this phase folder.