# Chapter 188 — Alert Rule Templates

Phase 20 — Platform Engineering

## Part 1 — Alerting Architecture and Standards

### 1. Objective

In Chapter 187, we built the monitoring foundation:

- Prometheus collects metrics.
- Grafana displays dashboards.
- Node Exporter exposes Linux node metrics.
- kube-state-metrics exposes Kubernetes object state.
- Application services expose business and technical metrics.
- ELK centralizes application logs.

Now we need to answer a different question:

When should the monitoring platform notify an engineer that something needs attention?

That is the purpose of alerting.

We will build reusable alert rules for the 21 microservices so that every service follows consistent operational standards.

The alerting system must:

1. Detect service availability problems.
2. Detect elevated HTTP error rates.
3. Detect latency degradation.
4. Detect CPU and memory pressure.
5. Detect pod restart and deployment failures.
6. Detect database connection exhaustion.
7. Detect node and storage problems.
8. Route notifications to the correct team.
9. Avoid duplicate and noisy alerts.
10. Support incident investigation and recovery.

The goal is not to generate an alert for every metric that changes. The goal is to notify the right engineer when a meaningful operational condition occurs.

### 2. Alerting architecture

Our architecture:

[SLAs That Matter: How to Define SLOs for Research Platforms | Biotech AI Blog](https://images.openai.com/static-rsc-4/2uEHWEycrRoPARvrdeSbjld9rFt_OZ8eEeljd5ZaoiL25Wssod7RJ-QGjw3FrR-xIEk1-f-9BdTB6c_XMjUgHC8daq03bJ_j5TmdFDHlHvgdSi5RppHXieDJ7LGQkYQmblj2uUnoe_EN4BMXm4H5oliGYw_Ankbz-Xh6YXGSxSg?purpose=inline)

[biotechai.ch](https://www.biotechai.ch/blog/slas-that-matter.html?utm_source=chatgpt.com)

```
21 Microservices
       |
       v
   Prometheus
       |
       v
  Alert Rules
       |
       v
  Alertmanager
       |
       +------------------+
       |                  |
       v                  v
   SRE Team          Platform Team
       |                  |
       v                  v
   On-call           Infrastructure
   Engineer          Investigation

       |
       +------------------+
       |                  |
       v                  v
 Application Team     Database Team
```

The components have separate responsibilities:

| Component                | Responsibility                                   |
| ------------------------ | ------------------------------------------------ |
| Prometheus               | Evaluates alert expressions                      |
| PrometheusRule           | Stores reusable alert definitions                |
| Alertmanager             | Groups, deduplicates, silences and routes alerts |
| Grafana                  | Displays alert status and dashboards             |
| Notification integration | Delivers notifications to approved channels      |
| On-call process          | Ensures someone investigates and responds        |

Prometheus detects a condition. Alertmanager manages the notification. The on-call process manages the human response.

### 3. Alert ownership

Alert ownership must be clear before production rollout.

| Alert category           | Primary owner                   | Escalation        |
| ------------------------ | ------------------------------- | ----------------- |
| Application availability | Service owner                   | SRE               |
| HTTP 5xx errors          | Application team                | SRE               |
| Application latency      | Application team                | SRE               |
| CPU and memory           | Platform team                   | SRE               |
| Kubernetes node health   | Platform team                   | SRE               |
| Deployment availability  | Platform team and service owner | SRE               |
| Database connections     | Database team                   | Application team  |
| RDS storage and CPU      | Database team                   | Platform team     |
| Kafka/RabbitMQ health    | Messaging owner                 | Platform team     |
| Security-related alerts  | Security team                   | Incident response |

The service owner must be declared in the service's monitoring metadata.

For example:

```
service:
  name: order-service
  team: commerce
  escalation_team: sre
  criticality: high
```

This metadata helps standardize alert routing. It does not replace a documented on-call schedule.

### 4. Repository structure

We will extend the existing platform monitoring repository.

```
platform-monitoring/
│
├── prometheus/
│   ├── rules/
│   │   ├── application/
│   │   │   ├── availability.yaml
│   │   │   ├── http-errors.yaml
│   │   │   ├── latency.yaml
│   │   │   └── business.yaml
│   │   │
│   │   ├── kubernetes/
│   │   │   ├── pod-health.yaml
│   │   │   ├── deployment-health.yaml
│   │   │   ├── node-health.yaml
│   │   │   └── storage.yaml
│   │   │
│   │   ├── infrastructure/
│   │   │   ├── prometheus-health.yaml
│   │   │   └── exporter-health.yaml
│   │   │
│   │   ├── database/
│   │   │   ├── connection-pool.yaml
│   │   │   └── database-health.yaml
│   │   │
│   │   └── messaging/
│   │       ├── kafka.yaml
│   │       └── rabbitmq.yaml
│   │
│   └── values/
│       ├── dev-values.yaml
│       ├── qa-values.yaml
│       ├── staging-values.yaml
│       └── prod-values.yaml
│
├── alertmanager/
│   ├── alertmanager.yaml
│   ├── templates/
│   │   └── notification.tmpl
│   └── routing/
│       └── team-routing.yaml
│
├── tests/
│   ├── rules/
│   ├── fixtures/
│   └── test-alerts.sh
│
├── docs/
│   ├── alert-standards.md
│   ├── severity-policy.md
│   ├── routing-policy.md
│   └── incident-runbook.md
│
└── README.md
```

The repository stores reusable alert definitions and routing configuration. Service-specific overrides should be limited and reviewed.

### 5. Alert naming standards

Use a consistent alert naming convention:

```
<service_or_platform>_<condition>_<severity>
```

Examples:

```
OrderServiceHighErrorRate
OrderServiceHighLatency
OrderServiceDown
KubernetesPodCrashLooping
KubernetesNodeNotReady
PrometheusTargetMissing
DatabaseConnectionPoolExhausted
```

The name should describe the condition, not the person or team receiving the alert.

### 6. Severity standards

We will use four severity levels.

| Severity | Meaning                                    | Response                                 |
| -------- | ------------------------------------------ | ---------------------------------------- |
| Critical | Major production impact or imminent outage | Page on-call immediately                 |
| Warning  | Degradation requiring investigation        | Notify responsible team                  |
| Info     | Operational event or low-risk condition    | Dashboard or low-priority notification   |
| None     | Not a notification severity                | Used for grouping or internal logic only |

Examples:

Critical

- Production service unavailable.
- Sustained high 5xx rate affecting customers.
- Production database unavailable.
- Critical node or cluster failure affecting workloads.

Warning

- CPU utilization is elevated.
- Memory utilization is approaching a limit.
- Pod restarts are increasing.
- Database connection pool is approaching exhaustion.

Info

- A non-production deployment has completed.
- A capacity trend requires review.
- A non-urgent resource threshold has been crossed.

Severity must reflect business impact, not merely the size of a metric.

### 7. Alert rule structure

A Prometheus alert rule usually contains:

- `alert`: alert name.
- `expr`: PromQL expression.
- `for`: duration the condition must remain true.
- `labels`: severity, team, service and other routing metadata.
- `annotations`: description, summary and runbook reference.

Example:

```
groups:
  - name: order-service-alerts

    rules:
      - alert: OrderServiceHighErrorRate

        expr: |
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
          ) > 0.05

        for: 5m

        labels:
          severity: critical
          team: commerce
          service: order-service

        annotations:
          summary: "Order Service has a high HTTP 5xx rate"

          description: |
            More than 5% of observed Order Service HTTP requests
            are returning 5xx responses for at least 5 minutes.

          runbook: "docs/runbooks/order-service-high-error-rate.md"
```

This example uses a 5% error threshold for demonstration. A production threshold should be based on the service's traffic, customer impact, SLO, and error budget.

### 8. Understand the `for` duration

Suppose an alert expression becomes true at 10:00.

Configuration:

```
for: 5m
```

Prometheus waits until the condition remains true for five minutes before firing the alert.

Example:

```
10:00  Error rate becomes 8%
10:01  Error rate is 7%
10:02  Error rate is 6%
10:03  Error rate is 8%
10:04  Error rate is 7%
10:05  Alert becomes firing
```

If the condition returns to normal before the `for` duration completes, the alert returns to inactive.

This helps prevent notifications for short-lived spikes.

However, a long `for` duration can delay detection of a serious outage. Choose the duration according to the impact and detection objective.

### 9. Alert states

Prometheus alert rules generally move through these states:

```
          Condition false
                |
                v
             INACTIVE
                |
                | Expression becomes true
                v
             PENDING
                |
                | `for` duration completed
                v
             FIRING
                |
                | Condition becomes false
                v
             INACTIVE
```

A pending alert is not yet firing.

This distinction matters when investigating why a condition appears in Prometheus but has not generated a notification.

### 10. Alert design principles

Follow these standards:

1. Alert on symptoms that affect users or critical operations.
2. Use a clear owner for every actionable alert.
3. Include a useful description and runbook.
4. Avoid high-cardinality labels.
5. Avoid duplicate alerts for the same underlying failure.
6. Use `for` durations to filter transient conditions.
7. Separate warning and critical thresholds.
8. Test alert expressions before production deployment.
9. Route production alerts differently from development alerts.
10. Review alert noise after incidents and deployments.

An alert that repeatedly fires but never leads to an action is a candidate for redesign.

## Part 2 — Application Alert Templates

### 11. Service availability alert

First, we need to detect whether Prometheus can scrape the application.

Prometheus automatically creates the `up` metric for each scrape target.

Values:

```
up = 1   Target scrape succeeded
up = 0   Target scrape failed
```

For Order Service:

```
up{
  namespace="ecommerce",
  service="order-service"
} == 0
```

Alert definition:

```
groups:
  - name: application-availability

    rules:
      - alert: OrderServiceTargetDown

        expr: |
          up{
            namespace="ecommerce",
            service="order-service"
          } == 0

        for: 2m

        labels:
          severity: critical
          team: commerce
          service: order-service

        annotations:
          summary: "Order Service metrics target is unavailable"

          description: |
            Prometheus has been unable to scrape Order Service
            for at least 2 minutes.

          runbook: "docs/runbooks/order-service-down.md"
```

Important distinction:

A failed Prometheus scrape does not always prove that customers cannot access the application. It can also mean the metrics endpoint is unavailable while the main HTTP endpoint still works.

For customer-facing availability, combine monitoring with appropriate synthetic checks or service-level indicators. Do not treat one metrics scrape as a complete customer availability measurement.

### 12. Generalized service availability template

We do not want to create a different YAML expression for every microservice.

Use a generalized rule:

```
groups:
  - name: microservice-availability

    rules:

      - alert: MicroserviceMetricsTargetDown

        expr: |
          up{
            namespace="ecommerce",
            environment="production"
          } == 0

        for: 2m

        labels:
          severity: critical
          team: platform

        annotations:
          summary: "A production microservice metrics target is down"

          description: |
            Prometheus cannot scrape target
            {{ $labels.service }}.

          runbook: "docs/runbooks/metrics-target-down.md"
```

This rule assumes the scrape target has the `environment` and `service` labels.

If you do not attach those labels to targets, the expression will not match. Confirm actual labels through the Prometheus query interface before deploying the rule.

For team ownership, a generalized rule can initially route to the platform team. A stronger design attaches the actual service owner to the target or uses carefully designed routing rules.

### 13. HTTP 5xx error-rate alert

Suppose Order Service normally processes 1,000 requests per minute.

Suddenly, 100 requests fail with HTTP 500 or 503.

The error rate is:

\\[ \text{Error Rate} = \frac{\text{5xx Requests}}{\text{Total Requests}} \times 100 \\]

Example:

```
Total requests = 1,000
5xx requests   = 100

Error rate = 10%
```

PromQL:

```
100 *
(
  sum by (service) (
    rate(
      http_server_requests_seconds_count{
        namespace="ecommerce",
        status=~"5.."
      }[5m]
    )
  )
  /
  clamp_min(
    sum by (service) (
      rate(
        http_server_requests_seconds_count{
          namespace="ecommerce"
        }[5m]
      )
    ),
    0.001
  )
)
```

Reusable alert:

```
groups:
  - name: application-http-errors

    rules:

      - alert: MicroserviceHigh5xxErrorRate

        expr: |
          100 *
          (
            sum by (service) (
              rate(
                http_server_requests_seconds_count{
                  namespace="ecommerce",
                  status=~"5.."
                }[5m]
              )
            )
            /
            clamp_min(
              sum by (service) (
                rate(
                  http_server_requests_seconds_count{
                    namespace="ecommerce"
                  }[5m]
                )
              ),
              0.001
            )
          ) > 5

        for: 5m

        labels:
          severity: critical
          team: commerce

        annotations:
          summary: "High HTTP 5xx rate for {{ $labels.service }}"

          description: |
            Service {{ $labels.service }} has a 5xx error
            rate greater than 5% for at least 5 minutes.

          runbook: "docs/runbooks/high-5xx-rate.md"
```

This is a service-wide threshold template. In production, you may need different thresholds for different criticality classes or traffic patterns.

### 14. Low-traffic protection

A common mistake is to alert on an error percentage when a service has almost no traffic.

Example:

```
1 request
1 failed request

Error rate = 100%
```

That may not represent a serious incident.

Use a minimum request-rate condition when appropriate.

Example:

```
(
  100 *
  (
    sum by (service) (
      rate(
        http_server_requests_seconds_count{
          namespace="ecommerce",
          status=~"5.."
        }[5m]
      )
    )
    /
    clamp_min(
      sum by (service) (
        rate(
          http_server_requests_seconds_count{
            namespace="ecommerce"
          }[5m]
        )
      ),
      0.001
    )
  ) > 5
)
and
(
  sum by (service) (
    rate(
      http_server_requests_seconds_count{
        namespace="ecommerce"
      }[5m]
    )
  ) > 1
)
```

This example requires more than one request per second before the error-rate alert can fire.

The minimum traffic threshold should be selected for each service. A low-volume but critical payment operation may need a different detection strategy.

### 15. HTTP latency alert

Latency is the time taken to process a request.

For an e-commerce application, slow responses can affect:

- Product browsing.
- Cart updates.
- Checkout.
- Order creation.
- Payment confirmation.
- Search.

A useful indicator is p95 latency.

Example:

```
histogram_quantile(
  0.95,
  sum by (service, le) (
    rate(
      http_server_requests_seconds_bucket{
        namespace="ecommerce"
      }[5m]
    )
  )
)
```

This query returns the estimated p95 request latency by service.

Alert example:

```
groups:
  - name: application-latency

    rules:

      - alert: MicroserviceHighP95Latency

        expr: |
          histogram_quantile(
            0.95,
            sum by (service, le) (
              rate(
                http_server_requests_seconds_bucket{
                  namespace="ecommerce"
                }[5m]
              )
            )
          ) > 1

        for: 10m

        labels:
          severity: warning
          team: commerce

        annotations:
          summary: "High p95 latency for {{ $labels.service }}"

          description: |
            The estimated p95 request latency for
            {{ $labels.service }} has exceeded 1 second
            for at least 10 minutes.

          runbook: "docs/runbooks/high-latency.md"
```

The threshold of one second is illustrative. Each service should have a latency objective based on its user experience and business function.

### 16. Critical latency alert

A warning may indicate that latency is elevated.

A critical alert should indicate a more serious and sustained impact.

Example:

```
- alert: MicroserviceCriticalP95Latency

  expr: |
    histogram_quantile(
      0.95,
      sum by (service, le) (
        rate(
          http_server_requests_seconds_bucket{
            namespace="ecommerce"
          }[5m]
        )
      )
    ) > 3

  for: 5m

  labels:
    severity: critical
    team: commerce

  annotations:
    summary: "Critical latency for {{ $labels.service }}"

    description: |
      The estimated p95 request latency has exceeded
      3 seconds for at least 5 minutes.
```

Before deploying both warning and critical rules, verify that the thresholds and durations are sufficiently separated to avoid notification storms.

### 17. Request traffic unexpectedly stops

A service may remain healthy but suddenly stop receiving traffic.

For example, Search Service may receive requests normally, then its request rate drops to zero.

A possible query:

```
sum by (service) (
  rate(
    http_server_requests_seconds_count{
      namespace="ecommerce"
    }[10m]
  )
) == 0
```

However, zero traffic is not necessarily a failure. A service may be idle overnight or may only receive event-driven requests.

For important customer-facing services, compare the current traffic against expected traffic patterns or use synthetic checks.

Do not apply a zero-traffic alert indiscriminately to all 21 services.

### 18. Business operation failure alert

Technical HTTP success does not always mean business success.

Consider this situation:

```
HTTP response: 200 OK
Order status: FAILED
```

The application may return a valid HTTP response while the business operation fails.

Therefore, Order Service should expose a business failure metric.

Example metric:

```
orders_failed_total
```

PromQL:

```
sum(
  rate(
    orders_failed_total{
      service="order-service",
      environment="production"
    }[5m]
  )
)
```

An example alert:

```
- alert: OrderCreationFailureRateHigh

  expr: |
    (
      sum(
        rate(
          orders_failed_total{
            service="order-service",
            environment="production"
          }[5m]
        )
      )
      /
      clamp_min(
        sum(
          rate(
            orders_created_total{
              service="order-service",
              environment="production"
            }[5m]
          )
        )
        +
        sum(
          rate(
            orders_failed_total{
              service="order-service",
              environment="production"
            }[5m]
          )
        ),
        0.001
      )
    ) > 0.05

  for: 5m

  labels:
    severity: critical
    team: commerce
    service: order-service

  annotations:
    summary: "Order creation failure rate is high"

    description: |
      More than 5% of observed order creation attempts
      are failing.
```

This assumes that successful and failed counters represent mutually exclusive outcomes from the same population of order creation attempts.

If the application counts retries, duplicates, or asynchronous failures differently, the metric design must be corrected before using this ratio.

### 19. Database connection pool alert

Suppose Order Service uses HikariCP.

Its maximum pool size is 30.

At a particular moment:

```
Active connections = 29
Idle connections  = 1
Maximum pool size  = 30
```

The pool is almost exhausted.

Metric:

```
hikaricp_connections_active
```

Metric:

```
hikaricp_connections_max
```

PromQL:

```
100 *
(
  hikaricp_connections_active{
    service="order-service"
  }
  /
  clamp_min(
    hikaricp_connections_max{
      service="order-service"
    },
    1
  )
)
```

Alert:

```
- alert: DatabaseConnectionPoolNearlyExhausted

  expr: |
    100 *
    (
      hikaricp_connections_active{
        namespace="ecommerce"
      }
      /
      clamp_min(
        hikaricp_connections_max{
          namespace="ecommerce"
        },
        1
      )
    ) > 90

  for: 5m

  labels:
    severity: warning
    team: commerce

  annotations:
    summary: "Database connection pool is nearly exhausted"

    description: |
      Connection pool utilization for
      {{ $labels.service }} has exceeded 90%.
```

Also monitor pending connections:

```
hikaricp_connections_pending{
  namespace="ecommerce"
} > 0
```

Pending connections indicate that application threads are waiting for a connection.

A connection pool alert should trigger investigation of:

- Database response time.
- Connection leaks.
- Long-running queries.
- Pool size.
- Number of application replicas.
- Database maximum connections.

Increasing the pool size without checking the database's total connection budget can make the incident worse.

### 20. Database connection timeout alert

If the application exports a connection timeout counter, monitor its increase.

The exact metric name depends on the Micrometer and HikariCP versions and configuration.

Example:

```
increase(
  hikaricp_connections_timeout_total{
    namespace="ecommerce"
  }[5m]
) > 0
```

Check the exported metric before using this rule. If your application exposes a different name, update the expression.

A timeout is often more actionable than a pool-utilization warning because it indicates that an application operation actually failed to acquire a connection.

### 21. Downstream dependency failure

Suppose Order Service depends on:

- Inventory Service.
- Payment Service.
- Shipping Service.

The application should expose dependency-specific metrics.

Examples:

```
inventory_client_requests_total
payment_client_requests_total
shipping_client_requests_total
```

The labels can include a controlled dependency name:

```
dependency="inventory-service"
```

Avoid dynamic URL or request ID labels.

Example query:

```
sum by (service, dependency) (
  rate(
    downstream_requests_failed_total{
      environment="production"
    }[5m]
  )
)
```

Alert:

```
- alert: DownstreamDependencyFailuresHigh

  expr: |
    sum by (service, dependency) (
      rate(
        downstream_requests_failed_total{
          environment="production"
        }[5m]
      )
    ) > 1

  for: 5m

  labels:
    severity: warning
    team: commerce

  annotations:
    summary: "Downstream dependency failures detected"

    description: |
      {{ $labels.service }} is experiencing elevated
      failures while calling {{ $labels.dependency }}.
```

This is an example of a reusable dependency alert. A production implementation should distinguish retryable failures, expected business rejections, and actual dependency availability failures.

### 22. Application alert checklist

For each microservice, validate:

```
[ ] Metrics endpoint is available
[ ] Service labels are correct
[ ] Request counter exists
[ ] HTTP status labels exist
[ ] Histogram buckets exist if latency alerts use them
[ ] Business success/failure metrics are defined
[ ] Dependency metrics are defined where relevant
[ ] Database connection metrics are available where relevant
[ ] Alert thresholds have a documented owner
[ ] Each actionable alert has a runbook
```

## Part 3 — Kubernetes and Infrastructure Alert Templates

### 23. Pod CrashLoopBackOff alert

A pod may repeatedly start, fail, and restart.

Example:

```
Order Service Pod
       |
       v
Container starts
       |
       v
Application crashes
       |
       v
Kubernetes restarts container
       |
       v
Application crashes again
       |
       v
CrashLoopBackOff
```

This can happen because of:

- Invalid application configuration.
- Missing environment variables.
- Incorrect secret references.
- Database connection failures during startup.
- Application exceptions.
- Incorrect container command.
- Insufficient memory.
- Failed startup probes.

Prometheus metric:

```
kube_pod_container_status_waiting_reason
```

Reusable rule:

```
groups:
  - name: kubernetes-pod-health

    rules:

      - alert: KubernetesPodCrashLooping

        expr: |
          max by (namespace, pod, container) (
            kube_pod_container_status_waiting_reason{
              namespace="ecommerce",
              reason="CrashLoopBackOff"
            }
          ) == 1

        for: 5m

        labels:
          severity: warning
          team: platform

        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

          description: |
            Container {{ $labels.container }} in pod
            {{ $labels.pod }} has been in CrashLoopBackOff
            for at least 5 minutes.

          runbook: "docs/runbooks/pod-crashloop.md"
```

For a production-critical service, you may classify a sustained crash loop as critical when it reduces available replicas below the required capacity.

### 24. Pod restart alert

A container may restart repeatedly without remaining in CrashLoopBackOff.

Use the restart counter:

```
increase(
  kube_pod_container_status_restarts_total{
    namespace="ecommerce"
  }[15m]
)
```

Reusable rule:

```
- alert: KubernetesContainerRestartingFrequently

  expr: |
    increase(
      kube_pod_container_status_restarts_total{
        namespace="ecommerce"
      }[15m]
    ) > 3

  for: 5m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Container restarts detected"

    description: |
      Container {{ $labels.container }} in pod
      {{ $labels.pod }} has restarted more than
      3 times in the last 15 minutes.
```

This rule needs testing against the actual metric labels and the lifecycle of short-lived pods. A deployment rollout can legitimately create new pods, and the restart counter behavior should be interpreted accordingly.

### 25. Pod not ready alert

A pod may exist but fail readiness checks.

Example:

```
kube_pod_status_ready{
  namespace="ecommerce",
  condition="false"
} == 1
```

Alert:

```
- alert: KubernetesPodNotReady

  expr: |
    kube_pod_status_ready{
      namespace="ecommerce",
      condition="false"
    } == 1

  for: 10m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Pod {{ $labels.pod }} is not ready"

    description: |
      Pod {{ $labels.pod }} in namespace
      {{ $labels.namespace }} has remained unready
      for at least 10 minutes.
```

Do not treat every unready pod as a production incident. During a rolling deployment, old and new pods may temporarily change readiness.

For stronger detection, compare available replicas with desired replicas and consider the rollout state.

### 26. Deployment replicas unavailable

Suppose the Order Service Deployment expects three replicas:

```
spec:
  replicas: 3
```

But only one replica is available.

The application may still be responding, but it has reduced capacity and resilience.

PromQL:

```
kube_deployment_spec_replicas{
  namespace="ecommerce"
}
>
kube_deployment_status_replicas_available{
  namespace="ecommerce"
}
```

Reusable alert:

```
- alert: KubernetesDeploymentReplicasUnavailable

  expr: |
    (
      kube_deployment_spec_replicas{
        namespace="ecommerce"
      }
      -
      kube_deployment_status_replicas_available{
        namespace="ecommerce"
      }
    ) > 0

  for: 10m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Deployment replicas are unavailable"

    description: |
      Deployment {{ $labels.deployment }} in namespace
      {{ $labels.namespace }} has fewer available replicas
      than desired.
```

For critical production services, create a separate critical rule when no replicas are available or when the service is below its minimum required capacity.

### 27. Deployment rollout stuck

A deployment can become stuck because:

- New pods cannot be scheduled.
- Image pull fails.
- Readiness probes fail.
- The new application version crashes.
- PVC mounting fails.
- Resource requests cannot be satisfied.

A simple metric-based check can detect deployments whose updated replicas do not match the desired replicas.

Example:

```
kube_deployment_spec_replicas{
  namespace="ecommerce"
}
>
kube_deployment_status_replicas_updated{
  namespace="ecommerce"
}
```

Alert:

```
- alert: KubernetesDeploymentRolloutIncomplete

  expr: |
    (
      kube_deployment_spec_replicas{
        namespace="ecommerce"
      }
      -
      kube_deployment_status_replicas_updated{
        namespace="ecommerce"
      }
    ) > 0

  for: 15m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Deployment rollout is incomplete"

    description: |
      Deployment {{ $labels.deployment }} in namespace
      {{ $labels.namespace }} has not updated all desired
      replicas within the expected time.
```

This is a basic rollout signal. For production, combine it with deployment progress information and account for intentionally paused or slowly progressing rollouts.

### 28. Node NotReady alert

An EKS worker node can become NotReady.

Possible reasons:

- Node networking failure.
- Kubelet unavailable.
- Node memory pressure.
- Disk pressure.
- Instance failure.
- Node termination.
- CNI problems.

Metric:

```
kube_node_status_condition
```

Alert:

```
groups:
  - name: kubernetes-node-health

    rules:

      - alert: KubernetesNodeNotReady

        expr: |
          kube_node_status_condition{
            condition="Ready",
            status="true"
          } == 0

        for: 5m

        labels:
          severity: critical
          team: platform

        annotations:
          summary: "Kubernetes node is not ready"

          description: |
            Node {{ $labels.node }} is not reporting
            Ready status.

          runbook: "docs/runbooks/node-not-ready.md"
```

Validate the metric labels and output in your cluster before using this exact expression. kube-state-metrics exposes condition series for multiple statuses; the rule should select the correct series and avoid unintended duplicate matches.

### 29. Node CPU utilization alert

CPU usage can increase because of:

- High request volume.
- CPU-intensive application operations.
- Inefficient code.
- Excessive background processing.
- A noisy neighbor workload.
- Incorrect pod requests and limits.

Query:

```
100 *
(
  1 -
  avg by (instance) (
    rate(
      node_cpu_seconds_total{
        mode="idle"
      }[5m]
    )
  )
)
```

Alert:

```
- alert: KubernetesNodeHighCPU

  expr: |
    100 *
    (
      1 -
      avg by (instance) (
        rate(
          node_cpu_seconds_total{
            mode="idle"
          }[5m]
        )
      )
    ) > 85

  for: 10m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Node CPU utilization is high"

    description: |
      Node {{ $labels.instance }} has CPU utilization
      above 85% for at least 10 minutes.
```

A CPU threshold alone does not prove that workloads are unhealthy. Check scheduling capacity, CPU throttling, request volume, and application latency before taking action.

### 30. Node memory utilization alert

Memory pressure can cause:

- Pod eviction.
- Container OOM kills.
- Application restarts.
- Scheduling failures.
- Node instability.

PromQL:

```
100 *
(
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
) > 90
```

Alert:

```
- alert: KubernetesNodeHighMemory

  expr: |
    100 *
    (
      1 -
      (
        node_memory_MemAvailable_bytes
        /
        node_memory_MemTotal_bytes
      )
    ) > 90

  for: 10m

  labels:
    severity: critical
    team: platform

  annotations:
    summary: "Node memory utilization is critical"

    description: |
      Node {{ $labels.instance }} has memory utilization
      above 90% for at least 10 minutes.
```

The threshold is illustrative. Use the node's actual memory pressure and eviction behavior when defining the production threshold.

### 31. Node disk utilization alert

A node's filesystem may fill up because of:

- Container logs.
- Image layers.
- Temporary files.
- Container runtime data.
- Application-generated files.
- Uncontrolled local storage.

PromQL:

```
100 *
(
  1 -
  (
    node_filesystem_avail_bytes{
      fstype!~"tmpfs|overlay",
      mountpoint="/"
    }
    /
    node_filesystem_size_bytes{
      fstype!~"tmpfs|overlay",
      mountpoint="/"
    }
  )
) > 85
```

Alert:

```
- alert: KubernetesNodeDiskUsageHigh

  expr: |
    100 *
    (
      1 -
      (
        node_filesystem_avail_bytes{
          fstype!~"tmpfs|overlay",
          mountpoint="/"
        }
        /
        node_filesystem_size_bytes{
          fstype!~"tmpfs|overlay",
          mountpoint="/"
        }
      )
    ) > 85

  for: 10m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Node root filesystem usage is high"

    description: |
      Node {{ $labels.instance }} root filesystem
      utilization has exceeded 85%.
```

Filesystem mount paths vary. Verify the root filesystem mountpoint and exporter labels on the actual EKS node image.

### 32. Persistent volume capacity alert

Persistent volumes can fill up and affect applications.

Example query:

```
100 *
(
  1 -
  (
    kubelet_volume_stats_available_bytes{
      namespace="ecommerce"
    }
    /
    kubelet_volume_stats_capacity_bytes{
      namespace="ecommerce"
    }
  )
) > 85
```

Alert:

```
- alert: KubernetesPersistentVolumeUsageHigh

  expr: |
    100 *
    (
      1 -
      (
        kubelet_volume_stats_available_bytes{
          namespace="ecommerce"
        }
        /
        kubelet_volume_stats_capacity_bytes{
          namespace="ecommerce"
        }
      )
    ) > 85

  for: 10m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Persistent volume usage is high"

    description: |
      A persistent volume used by namespace
      {{ $labels.namespace }} has exceeded 85% utilization.
```

This depends on kubelet volume statistics being available for the storage type and volume configuration.

For EBS-backed volumes, also monitor AWS storage capacity and performance using the relevant AWS metrics integration.

### 33. Node pressure conditions

Kubernetes reports node conditions such as:

- MemoryPressure.
- DiskPressure.
- PIDPressure.

Example memory-pressure query:

```
kube_node_status_condition{
  condition="MemoryPressure",
  status="true"
} == 1
```

Example disk-pressure query:

```
kube_node_status_condition{
  condition="DiskPressure",
  status="true"
} == 1
```

These conditions are useful because they describe Kubernetes' assessment of the node, not merely a generic utilization percentage.

Use separate alerts for these conditions and assign appropriate severity based on their impact.

### 34. Alerting for node exporter and kube-state-metrics

Monitoring infrastructure itself must be monitored.

If Node Exporter stops exposing metrics, the platform loses node visibility.

Example:

```
- alert: NodeExporterTargetMissing

  expr: |
    up{
      job=~".*node-exporter.*"
    } == 0

  for: 5m

  labels:
    severity: warning
    team: platform

  annotations:
    summary: "Node Exporter target is unavailable"

    description: |
      Prometheus cannot scrape a Node Exporter target.
```

For kube-state-metrics:

```
- alert: KubeStateMetricsTargetMissing

  expr: |
    up{
      job=~".*kube-state-metrics.*"
    } == 0

  for: 5m

  labels:
    severity: critical
    team: platform

  annotations:
    summary: "kube-state-metrics target is unavailable"

    description: |
      Prometheus cannot scrape kube-state-metrics.
      Kubernetes object-state visibility may be incomplete.
```

Job labels vary by chart version and configuration. Inspect `up` series in Prometheus and adapt these selectors to the actual jobs.

### 35. Infrastructure alert checklist

```
[ ] CrashLoopBackOff is detected
[ ] Container restarts are detected
[ ] Unready pods are detected
[ ] Deployment replica availability is monitored
[ ] Stuck rollouts are detected
[ ] Node Ready status is monitored
[ ] Node CPU utilization is monitored
[ ] Node memory utilization is monitored
[ ] Node filesystem usage is monitored
[ ] Persistent volume usage is monitored
[ ] Node pressure conditions are monitored
[ ] Node Exporter availability is monitored
[ ] kube-state-metrics availability is monitored
```

## Part 4 — Prometheus Rule Groups and Reusable Templates

### 36. Why group alert rules

Prometheus allows related rules to be organized into rule groups.

Example:

```
groups:
  - name: application-alerts
    rules:
      - alert: MicroserviceHigh5xxErrorRate
        # Rule configuration

      - alert: MicroserviceHighP95Latency
        # Rule configuration

  - name: kubernetes-alerts
    rules:
      - alert: KubernetesPodCrashLooping
        # Rule configuration

      - alert: KubernetesNodeNotReady
        # Rule configuration
```

Grouping helps:

- Organize rules by function.
- Review related alerts together.
- Manage evaluation intervals.
- Apply consistent labels and conventions.
- Separate application and infrastructure responsibilities.

A group can have a configured evaluation interval. Choose it intentionally; a shorter interval increases rule evaluation workload.

### 37. Configure evaluation interval

Example:

```
groups:
  - name: application-alerts

    interval: 30s

    rules:
      - alert: MicroserviceHigh5xxErrorRate
        # Rule configuration
```

This means Prometheus evaluates the group's rules every 30 seconds.

The `for` duration and evaluation interval work together. A rule configured with `for: 5m` does not necessarily fire at exactly five minutes; it fires after the condition has remained true across the required evaluations.

### 38. Create a shared rule file

File:

```
prometheus/rules/application/http-errors.yaml
```

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: ecommerce-application-http-alerts
  namespace: monitoring

  labels:
    release: monitoring
    team: platform

spec:
  groups:

    - name: ecommerce-http-errors

      interval: 30s

      rules:

        - alert: MicroserviceHigh5xxErrorRate

          expr: |
            (
              100 *
              sum by (service) (
                rate(
                  http_server_requests_seconds_count{
                    namespace="ecommerce",
                    environment="production",
                    status=~"5.."
                  }[5m]
                )
              )
              /
              clamp_min(
                sum by (service) (
                  rate(
                    http_server_requests_seconds_count{
                      namespace="ecommerce",
                      environment="production"
                    }[5m]
                  )
                ),
                0.001
              )
            ) > 5

          for: 5m

          labels:
            severity: critical
            team: commerce

          annotations:
            summary: "High 5xx error rate: {{ $labels.service }}"

            description: |
              Service {{ $labels.service }} has a 5xx error
              rate greater than 5% for at least 5 minutes.
```

This is a Kubernetes custom resource, not an ordinary Prometheus configuration file. It requires the Prometheus Operator CRD.

### 39. Deploy the PrometheusRule

For a lab:

```
kubectl apply -f \
  prometheus/rules/application/http-errors.yaml
```

Verify:

```
kubectl get prometheusrule \
  -n monitoring
```

Inspect:

```
kubectl describe prometheusrule \
  ecommerce-application-http-alerts \
  -n monitoring
```

Confirm that the Prometheus resource selects the rule's namespace and labels.

A PrometheusRule resource that exists in Kubernetes is not necessarily loaded by Prometheus. Its selectors and namespace selection must match the Prometheus configuration.

### 40. Reusable Helm alert template

If we want environment-specific thresholds, we can build a Helm template.

File:

```
platform-monitoring/templates/application-alerts.yaml
```

```
{{- if .Values.alerts.application.enabled }}

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: {{ .Release.Name }}-application-alerts
  namespace: {{ .Release.Namespace }}

  labels:
    release: {{ .Values.alerts.prometheusRelease }}
    team: platform

spec:
  groups:

    - name: ecommerce-application-alerts

      interval: {{ .Values.alerts.evaluationInterval }}

      rules:

        - alert: MicroserviceHigh5xxErrorRate

          expr: |
            (
              100 *
              sum by (service) (
                rate(
                  http_server_requests_seconds_count{
                    namespace="{{ .Values.alerts.namespace }}",
                    environment="{{ .Values.alerts.environment }}",
                    status=~"5.."
                  }[5m]
                )
              )
              /
              clamp_min(
                sum by (service) (
                  rate(
                    http_server_requests_seconds_count{
                      namespace="{{ .Values.alerts.namespace }}",
                      environment="{{ .Values.alerts.environment }}"
                    }[5m]
                  )
                ),
                0.001
              )
            ) > {{ .Values.alerts.application.errorRateThreshold }}

          for: {{ .Values.alerts.application.errorRateFor }}

          labels:
            severity: {{ .Values.alerts.application.errorRateSeverity }}
            team: commerce

          annotations:
            summary: "High 5xx rate: {{ "{{ $labels.service }}" }}"

            description: |
              HTTP 5xx error rate exceeded
              {{ .Values.alerts.application.errorRateThreshold }}%.
{{- end }}
```

The nested template expression is intentional: Helm must preserve Prometheus' `$labels.service` expression rather than evaluating it as a Helm variable.

Values:

```
alerts:
  prometheusRelease: monitoring
  evaluationInterval: 30s
  namespace: ecommerce
  environment: production

  application:
    enabled: true
    errorRateThreshold: 5
    errorRateFor: 5m
    errorRateSeverity: critical
```

This example demonstrates a configurable template. Before production use, validate the rendered YAML and PromQL, and ensure the label names match the actual metric series.

### 41. Environment-specific thresholds

Example development values:

```
alerts:
  environment: dev

  application:
    enabled: true
    errorRateThreshold: 20
    errorRateFor: 10m
    errorRateSeverity: warning
```

Example production values:

```
alerts:
  environment: production

  application:
    enabled: true
    errorRateThreshold: 5
    errorRateFor: 5m
    errorRateSeverity: critical
```

These thresholds are illustrative, not recommended universal values.

The important design is that the same reusable template can be configured differently for different environments without maintaining unrelated copies of the entire alert definition.

### 42. Avoid duplicate alert definitions

Do not define the same alert in all of these places simultaneously:

- Prometheus configuration.
- Helm chart.
- Grafana-managed alert rules.
- A separate manually applied PrometheusRule.

Choose one authoritative source for each alert.

For this project, use PrometheusRule resources managed through the platform monitoring repository and GitOps.

Grafana can still display alert status and dashboards, but avoid creating duplicate alerting logic without a clear ownership model.

### 43. Validate rules before deployment

Prometheus provides a rule validation utility.

If the `promtool` binary is available:

```
promtool check rules \
  prometheus/rules/application/http-errors.yaml
```

For a standalone Prometheus rule file, this validates the rule syntax.

For a Kubernetes `PrometheusRule` custom resource, first render or extract its `spec.groups` content into a Prometheus rule file for `promtool` validation. Then validate the Kubernetes resource against the installed CRD schema.

Also validate the actual PromQL against real metric names and labels in a test Prometheus environment. Syntax validation cannot prove that a query matches any series.

### 44. Test rule behavior

Prometheus supports rule unit tests through `promtool test rules`.

Example test file:

```
rule_files:
  - http-errors.rules.yaml

evaluation_interval: 1m

tests:

  - interval: 1m

    input_series:

      - series: 'http_server_requests_seconds_count{service="order-service",namespace="ecommerce",environment="production",status="200"}'
        values: '0+100x10'

      - series: 'http_server_requests_seconds_count{service="order-service",namespace="ecommerce",environment="production",status="500"}'
        values: '0+20x10'

    alert_rule_test:

      - eval_time: 10m

        alertname: MicroserviceHigh5xxErrorRate

        exp_alerts:
          - exp_labels:
              service: order-service
              severity: critical
              team: commerce
```

This is an illustrative fixture. The actual test must match the exact rule file and expected labels, including the effects of the `for` duration. If the rule includes additional labels or annotations, ensure the test's expected result matches the rendered rule.

Run:

```
promtool test rules \
  prometheus/rules/tests/http-errors.test.yaml
```

A rule that parses successfully can still be logically wrong. Unit tests should verify both firing and non-firing scenarios.

## Part 5 — Alertmanager Configuration and Notification Routing

### 45. Why Alertmanager is required

Prometheus evaluates alert rules and sends firing alerts to Alertmanager.

Alertmanager handles:

- Grouping related alerts.
- Deduplicating repeated alerts.
- Routing alerts to teams.
- Applying notification delays.
- Managing silences.
- Suppressing lower-priority alerts when a higher-priority incident is active.
- Sending notifications to configured integrations.

Example:

```
Prometheus
    |
    |  Order Service High 5xx
    |  Order Service High Latency
    |  Order Service Database Errors
    |
    v
Alertmanager
    |
    | Group related alerts
    | Identify service owner
    |
    v
Commerce On-call
```

Without grouping, one incident can generate dozens of separate notifications.

### 46. Alert grouping

Suppose Order Service experiences a database outage.

Several alerts may fire:

```
OrderServiceHighLatency
OrderServiceHigh5xxErrorRate
DatabaseConnectionPoolNearlyExhausted
OrderServiceBusinessFailures
```

These alerts may all have the same underlying cause.

We should group related alerts using labels such as:

```
group_by:
  - alertname
  - namespace
  - service
```

However, grouping only by `alertname` can still produce multiple notifications for the same incident when different alert names fire.

A useful production grouping strategy is to group by service and namespace, with additional grouping dimensions selected according to notification needs.

### 47. Alertmanager routing configuration

File:

```
alertmanager/alertmanager.yaml
```

Example:

```
global:
  resolve_timeout: 5m

route:

  receiver: platform-team

  group_by:
    - namespace
    - service

  group_wait: 30s

  group_interval: 5m

  repeat_interval: 4h

  routes:

    - matchers:
        - severity="critical"
        - team="commerce"

      receiver: commerce-oncall

      continue: false

    - matchers:
        - severity="critical"
        - team="platform"

      receiver: platform-oncall

      continue: false

    - matchers:
        - team="database"

      receiver: database-team

      continue: false

    - matchers:
        - severity="warning"

      receiver: engineering-alerts

      continue: false

receivers:

  - name: platform-team

  - name: commerce-oncall

  - name: platform-oncall

  - name: database-team

  - name: engineering-alerts
```

This illustrates routing structure. The receivers are placeholders and do not send notifications until an actual integration is configured.

The order of child routes matters. Alertmanager evaluates routes according to its routing-tree rules. Review the route matching behavior and test it with representative alerts before enabling production notifications.

### 48. Configure an email receiver

A basic email receiver example:

```
receivers:

  - name: commerce-oncall

    email_configs:

      - to: commerce-oncall@example.com

        from: alerts@example.com

        smarthost: smtp.example.com:587

        auth_username: alerts@example.com

        auth_password: SMTP_PASSWORD

        send_resolved: true
```

Do not commit the actual SMTP password to Git.

Use the chart's supported secret configuration or the approved secret-management integration. Confirm how the selected Alertmanager Helm chart expects SMTP credentials to be supplied.

The address and SMTP host above are placeholders.

### 49. Configure a chat notification receiver

If the organization uses Slack, Microsoft Teams, or another approved notification service, configure the corresponding Alertmanager integration or an approved intermediary.

Example using a generic webhook:

```
receivers:

  - name: commerce-oncall

    webhook_configs:

      - url: https://notification-gateway.example.com/alertmanager

        send_resolved: true

        max_alerts: 20
```

This is a generic webhook example. The endpoint must implement the Alertmanager webhook payload contract or use a compatible notification gateway.

Do not assume that any arbitrary chat webhook accepts Alertmanager's native payload.

### 50. Notification content

An alert notification should answer:

1. What happened?
2. Which service is affected?
3. Which environment is affected?
4. How severe is it?
5. When did it start?
6. Where is the runbook?
7. Which team owns it?

Example notification:

```
ALERT: Order Service High 5xx Rate

Severity: CRITICAL

Environment: production
Namespace: ecommerce
Service: order-service

Condition:
HTTP 5xx error rate exceeded 5%.

Duration:
5 minutes.

Owner:
Commerce Team

Runbook:
docs/runbooks/high-5xx-rate.md
```

Avoid notifications containing unnecessary metric dumps or sensitive request data.

### 51. Alertmanager notification template

File:

```
alertmanager/templates/notification.tmpl
```

Example:

```
{{ define "ecommerce.alert.title" }}
[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
{{ end }}

{{ define "ecommerce.alert.message" }}

Status: {{ .Status }}

{{ range .Alerts }}

Alert: {{ .Labels.alertname }}
Severity: {{ .Labels.severity }}

Service: {{ .Labels.service }}
Namespace: {{ .Labels.namespace }}
Environment: {{ .Labels.environment }}

Summary:
{{ .Annotations.summary }}

Description:
{{ .Annotations.description }}

Started:
{{ .StartsAt }}

{{ if .Annotations.runbook }}
Runbook:
{{ .Annotations.runbook }}
{{ end }}

{{ end }}

{{ end }}
```

Configure the template file through the Alertmanager Helm chart or its supported configuration mechanism.

Template functions and available fields should be validated against the Alertmanager version in use.

### 52. Inhibition rules

Inhibition prevents a lower-priority alert from notifying independently when a more important alert already explains the same incident.

Example:

```
Node NotReady
    |
    +----> Pod NotReady
    |
    +----> Application Target Down
```

If an entire node becomes unavailable, notifying separately for every pod on that node can create unnecessary noise.

An inhibition rule can suppress the lower-level alerts when the node-level alert is active.

Example:

```
inhibit_rules:

  - source_matchers:
      - alertname="KubernetesNodeNotReady"
      - severity="critical"

    target_matchers:
      - alertname="KubernetesPodNotReady"

    equal:
      - node
```

This is illustrative. It works only if both alert types carry a consistent `node` label.

A pod alert may not naturally contain the node label. If necessary, attach the node identity through the appropriate metric join or alert-label strategy.

Do not create inhibition rules that hide independent failures. For example, a node outage should not automatically suppress a database outage that may affect other nodes.

### 53. Silences

A silence temporarily prevents matching alerts from generating notifications.

Use cases:

- Approved maintenance.
- Planned cluster upgrades.
- A known non-production test.
- A controlled production change with an agreed notification policy.

Example:

```
Silence matchers:

environment = production
service     = order-service

Start:
Approved maintenance start time

End:
Approved maintenance end time

Reason:
Order Service maintenance window

Created by:
Authorized operator
```

Silencing is not the same as fixing the problem.

Use:

- A clear reason.
- A specific set of matchers.
- An explicit expiration time.
- An authorized operator.
- A documented maintenance or incident reference.

Avoid broad silences such as:

```
severity = critical
```

A broad silence can hide unrelated production incidents.

### 54. Group wait, group interval and repeat interval

These settings control notification behavior.

| Setting           | Meaning                                                            |
| ----------------- | ------------------------------------------------------------------ |
| `group_wait`      | Wait before sending the first notification for a new group         |
| `group_interval`  | Wait before sending updates for an existing group                  |
| `repeat_interval` | Wait before repeating a notification for an unresolved alert group |

Example:

```
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

These values are examples. A critical page may require a different routing and notification policy from a warning notification.

A short repeat interval can create alert fatigue. An excessively long interval can cause an unresolved issue to be forgotten. Define the policy with the on-call team.

### 55. Alert notification lifecycle

```
Prometheus detects condition
          |
          v
      Alert fires
          |
          v
     Alertmanager
          |
          v
     Group alerts
          |
          v
    Route to owner
          |
          v
  Engineer receives alert
          |
          v
   Engineer investigates
          |
          v
  Condition is resolved
          |
          v
 Prometheus sends resolved
          |
          v
 Alertmanager notifies owner
```

A notification being delivered does not prove that the engineer acknowledged it.

For critical alerts, the organization needs an on-call acknowledgement and escalation process outside the basic Prometheus rule itself.

## Part 6 — Alert Testing and Operational Validation

### 56. Why alert testing is mandatory

A rule may be syntactically correct but operationally useless.

Examples:

- The query matches no metrics.
- The metric label is wrong.
- The threshold is too high.
- The threshold is too low.
- The `for` duration delays an urgent notification.
- The notification route does not match.
- The receiver is not configured.
- The runbook does not exist.

Every production alert must be tested before it is enabled.

### 57. Test Prometheus expressions manually

Open Prometheus:

```
http://localhost:9090
```

Test the availability query:

```
up{
  namespace="ecommerce",
  service="order-service"
}
```

Test the error rate:

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

Test latency:

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

Check the returned value and labels.

If the query returns no data, investigate the metric instrumentation before changing the alert threshold.

### 58. Use Prometheus alert status

Prometheus provides an Alerts page:

```
http://localhost:9090/alerts
```

You can inspect:

- Inactive alerts.
- Pending alerts.
- Firing alerts.
- Labels.
- Annotations.
- Evaluation status.

[Alertmanager报警模块集成钉钉/企业微信机器人 - 浮\~生 - 博客园](https://images.openai.com/static-rsc-4/HlTIYOpM6HuH6r5iK2Cqfj8VMsQDJxCBIvVbIPnuZu1QVhoi3EPlB3h7DV6nbvg4DVTPnvznNBP4ZLhSb5PZfQY_16HPGclMIRntNCFXyMu7AFieKwqeh5jzjBCYiDIJkKNo8HHk50a2h5Tk6lW3JgA99zMDhw-pW-rj8dh5ok8?purpose=inline)

[cnblogs.com](https://www.cnblogs.com/gk888/p/16110565.html?utm_source=chatgpt.com)

This is useful when a query is true but a notification has not been delivered.

The troubleshooting path is:

```
Does the expression return data?
             |
             v
       Is condition true?
             |
             v
      Is alert pending?
             |
             v
      Has `for` elapsed?
             |
             v
      Is alert firing?
             |
             v
      Did Alertmanager receive it?
             |
             v
      Did route match receiver?
             |
             v
      Was notification delivered?
```

### 59. Test rule syntax

Use `promtool`:

```
promtool check rules \
  prometheus/rules/application/http-errors.rules.yaml
```

For the PrometheusRule resource, extract the `spec.groups` content into a standard rules file before running this command.

Validate rendered Helm output:

```
helm template monitoring-alerts \
  ./charts/monitoring-alerts \
  --namespace monitoring \
  --values values/prod-values.yaml
```

Inspect the generated resource and check that:

- YAML indentation is correct.
- The CRD API version is correct.
- Rule names are unique.
- PromQL expressions are valid.
- Labels are present.
- Annotations are useful.
- Helm has not evaluated Prometheus template expressions.

### 60. Test alert rules with fixtures

Create:

```
tests/rules/http-errors.rules.yaml
```

A simplified standalone rule for testing:

```
groups:
  - name: test-http-errors

    rules:
      - alert: TestHighErrorRate

        expr: |
          (
            sum by (service) (
              rate(
                http_requests_total{
                  status=~"5.."
                }[5m]
              )
            )
            /
            clamp_min(
              sum by (service) (
                rate(http_requests_total[5m])
              ),
              0.001
            )
          ) > 0.05

        for: 2m

        labels:
          severity: critical

        annotations:
          summary: "Test service error rate is high"
```

Test file:

```
tests/rules/http-errors.test.yaml
```

```
rule_files:
  - http-errors.rules.yaml

evaluation_interval: 1m

tests:

  - interval: 1m

    input_series:

      - series: 'http_requests_total{service="order-service",status="200"}'
        values: '0+100x10'

      - series: 'http_requests_total{service="order-service",status="500"}'
        values: '0+20x10'

    alert_rule_test:

      - eval_time: 5m

        alertname: TestHighErrorRate

        exp_alerts:
          - exp_labels:
              service: order-service
              severity: critical
```

Run:

```
promtool test rules \
  tests/rules/http-errors.test.yaml
```

This fixture is designed to illustrate the test structure. Verify its timing and expected labels against the exact Prometheus version and rule behavior in your CI environment. Add a non-firing fixture as well.

### 61. Test the notification route

Prometheus rule testing does not prove that Alertmanager can send notifications.

Test Alertmanager separately.

Verify:

```
kubectl get pods \
  -n monitoring \
  | grep alertmanager
```

Inspect logs:

```
kubectl logs \
  ALERTMANAGER_POD \
  -n monitoring \
  --tail=200
```

Check Alertmanager status and active silences through its UI or approved API access.

For a safe test:

1. Create a test alert with a dedicated test label.
2. Route it to a test receiver.
3. Verify the notification arrives.
4. Verify the alert resolves.
5. Verify the resolved notification arrives if configured.
6. Remove the test alert.
7. Record the test evidence.

Do not test notification delivery by intentionally breaking a production service.

### 62. Test a complete alert lifecycle

Example controlled test in QA:

Initial state

```
Order Service:
  HTTP 5xx rate = 0%
  Availability  = UP
```

Generate controlled failures

Use a test-only endpoint or a controlled QA fault-injection mechanism to produce server errors.

Expected result

```
HTTP 5xx rate exceeds configured threshold.
```

Prometheus

```
Alert state: PENDING
```

After the configured `for` duration:

```
Alert state: FIRING
```

Alertmanager

```
Receiver: commerce-oncall-test
```

Notification

```
Order Service error-rate alert received.
```

Recovery

Stop the test failure and confirm:

```
Alert state: INACTIVE
```

This validates the complete detection and notification path.

### 63. Alert test cases

| Test                            | Expected result                            |
| ------------------------------- | ------------------------------------------ |
| Service target goes down        | Availability alert                         |
| 5xx rate exceeds threshold      | Error-rate alert                           |
| p95 latency remains high        | Latency alert                              |
| Pod enters CrashLoopBackOff     | Pod health alert                           |
| Deployment loses replicas       | Deployment alert                           |
| Node becomes NotReady           | Node alert                                 |
| Disk usage exceeds threshold    | Disk alert                                 |
| Database pool becomes saturated | Connection pool alert                      |
| Alert resolves                  | Resolved notification                      |
| Alert is silenced               | Matching notification suppressed           |
| Unrelated alert fires           | Not hidden by an incorrect inhibition rule |

### 64. Alert noise review

After deployment, review:

- Number of alerts fired.
- Number of duplicate notifications.
- Alerts without an owner.
- Alerts without runbooks.
- Alerts that repeatedly resolve and fire.
- Alerts that never lead to an action.
- Alerts that fire during normal deployment activity.
- Alerts that detect symptoms too late.

An alert should have an operational purpose.

If an alert does not lead to investigation or action, review whether it should be:

- Changed to a dashboard-only signal.
- Given a different threshold.
- Given a different `for` duration.
- Grouped with another alert.
- Suppressed under a valid higher-level condition.
- Removed.

## Part 7 — Production Alert Governance and Incident Response

### 65. Alert ownership metadata

Every production alert should identify the responsible team.

Example:

```
labels:
  severity: critical
  team: commerce
  service: order-service
  environment: production
```

A missing team label creates routing ambiguity.

Before accepting a new alert, require:

```
[ ] Alert name
[ ] Severity
[ ] Service or platform owner
[ ] Environment
[ ] Useful summary
[ ] Actionable description
[ ] Runbook reference
[ ] Tested PromQL
[ ] Tested notification route
```

The platform team owns the shared contract. The application team owns the meaning of application-specific metrics and thresholds.

### 66. Alert routing by team

For the 21 microservices, establish an ownership inventory.

| Service                | Owning team       | Example alert receiver    |
| ---------------------- | ----------------- | ------------------------- |
| Auth Service           | Identity          | Identity on-call          |
| User Service           | Customer Platform | Customer Platform on-call |
| Customer Service       | Customer Platform | Customer Platform on-call |
| Catalog Service        | Commerce          | Commerce on-call          |
| Product Service        | Commerce          | Commerce on-call          |
| Inventory Service      | Supply Chain      | Supply Chain on-call      |
| Pricing Service        | Commerce          | Commerce on-call          |
| Promotion Service      | Commerce          | Commerce on-call          |
| Cart Service           | Commerce          | Commerce on-call          |
| Order Service          | Commerce          | Commerce on-call          |
| Payment Service        | Payments          | Payments on-call          |
| Fraud Service          | Risk              | Risk on-call              |
| Shipping Service       | Fulfillment       | Fulfillment on-call       |
| Delivery Service       | Fulfillment       | Fulfillment on-call       |
| Notification Service   | Communications    | Communications on-call    |
| Search Service         | Discovery         | Discovery on-call         |
| Recommendation Service | Discovery         | Discovery on-call         |
| Review Service         | Customer Platform | Customer Platform on-call |
| Tax Service            | Finance           | Finance on-call           |
| Invoice Service        | Finance           | Finance on-call           |
| Reporting Service      | Analytics         | Analytics on-call         |

This is an illustrative ownership model. The organization must confirm actual team ownership before enabling routing.

### 67. Production alert priority

Not every production alert needs an immediate page.

Use the following operational distinction:

| Condition                            | Suggested handling                       |
| ------------------------------------ | ---------------------------------------- |
| Entire payment service unavailable   | Critical page                            |
| Sustained checkout failures          | Critical page                            |
| Order Service p95 latency elevated   | Warning or critical based on SLO impact  |
| One pod restarts once                | Dashboard or warning depending on impact |
| Node disk approaching capacity       | Warning                                  |
| Node filesystem exhausted            | Critical if workloads are affected       |
| Development environment target down  | Development notification                 |
| Non-production deployment incomplete | Development team notification            |

The same metric can require different severities depending on customer impact.

For example, 90% CPU utilization on a production node that is still meeting all service objectives may be a warning. The same CPU pressure combined with rising latency, failed requests, and unavailable replicas may justify a critical incident.

### 68. Alert suppression during deployment

Suppose the GitOps pipeline deploys a new version of Order Service.

During the rollout:

```
Old pods terminate.
New pods start.
Readiness checks run.
Traffic moves to new pods.
```

Temporary pod state changes are expected.

If alerts are configured without considering deployment behavior, a normal rollout can create unnecessary notifications.

Possible controls:

- Use suitable `for` durations.
- Compare desired and available replicas.
- Exclude intentionally terminated workloads where appropriate.
- Avoid alerting on every transient pod transition.
- Keep critical availability alerts active.
- Use a narrowly scoped maintenance silence only when approved.

Do not suppress all production alerts during deployments. A deployment can cause a genuine outage, and the monitoring system must remain capable of detecting it.

### 69. Alert fatigue

Alert fatigue occurs when engineers receive too many low-value notifications.

Example:

```
09:00  Pod restart warning
09:01  Pod not ready warning
09:02  High CPU warning
09:03  High latency warning
09:04  HTTP 5xx warning
09:05  Database pool warning
09:06  Deployment replica warning
```

These may all be symptoms of one underlying database problem.

The solution is not simply to disable alerts.

Instead:

1. Identify the common root cause.
2. Group related notifications.
3. Define a primary actionable alert.
4. Use inhibition only for genuinely dependent symptoms.
5. Improve alert thresholds and durations.
6. Improve runbooks.
7. Review alert quality after incidents.

### 70. Incident acknowledgement and escalation

Alertmanager sends notifications, but it is not a complete incident-management system.

A production organization also needs:

- On-call schedules.
- Acknowledgement tracking.
- Escalation policies.
- Incident ownership.
- Incident communication.
- Resolution tracking.
- Post-incident review.

For a critical alert:

```
Alert fired
    |
    v
Primary on-call notified
    |
    v
Engineer acknowledges
    |
    +----> Investigation
    |
    +----> Mitigation
    |
    +----> Recovery
    |
    v
Incident resolved
    |
    v
Post-incident review
```

If the engineer does not acknowledge within the defined response window, the incident process should escalate to the secondary on-call or incident commander.

The actual escalation times must come from the organization's incident policy.

### 71. Runbook standard

Every actionable alert should link to a runbook.

Recommended runbook structure:

```
# Alert Name

## 1. Meaning

Explain what the alert detects.

## 2. Customer impact

Explain possible user or business impact.

## 3. Initial checks

List the first commands and dashboard panels.

## 4. Investigation

Describe the likely causes and diagnostic steps.

## 5. Mitigation

List approved temporary mitigation options.

## 6. Recovery verification

Explain how to confirm service recovery.

## 7. Escalation

Identify the responsible team.

## 8. Evidence

Record logs, metrics, deployment version,
incident time and actions taken.
```

A runbook should not instruct an engineer to blindly restart pods or increase resources without investigating the underlying cause.

### 72. Example runbook — Order Service high 5xx rate

File:

```
docs/runbooks/high-5xx-rate.md
```

```
# Order Service High 5xx Rate

## Alert

MicroserviceHigh5xxErrorRate

## Meaning

Order Service is returning an elevated percentage
of HTTP 5xx responses.

## Customer impact

Customers may be unable to create orders,
view order status or complete checkout.

## Initial checks

1. Open the Order Service Grafana dashboard.
2. Review HTTP 5xx rate.
3. Review request volume.
4. Review p95 latency.
5. Check available replicas.
6. Review recent deployment activity.

## Kubernetes checks

kubectl get pods -n ecommerce \
  -l app=order-service

kubectl get deployment \
  order-service \
  -n ecommerce

kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp

## Application logs

kubectl logs \
  -n ecommerce \
  ORDER_SERVICE_POD \
  --since=15m

## Database checks

1. Review HikariCP active connections.
2. Review pending connections.
3. Review connection timeout metrics.
4. Check RDS connection count.
5. Check database CPU and latency.

## Dependency checks

1. Check Inventory Service availability.
2. Check Payment Service availability.
3. Review downstream failure metrics.
4. Search related application logs.

## Deployment checks

Review the last GitOps synchronization
and application image digest.

## Mitigation

Use the approved rollback or recovery
procedure if the incident is linked
to a recent deployment.

Do not manually change production
resources without the required approval.

## Recovery verification

1. HTTP 5xx rate returns to normal.
2. p95 latency returns to the service objective.
3. Order creation succeeds.
4. Replicas are healthy.
5. Database connections are stable.
6. Alert resolves.

## Escalation

Commerce on-call and SRE.
```

### 73. Runbook — Node NotReady

File:

```
docs/runbooks/node-not-ready.md
```

```
# Kubernetes Node NotReady

## Meaning

An EKS worker node is not reporting
Ready status to the Kubernetes API.

## Possible impact

- Pods may become unavailable.
- Workloads may be rescheduled.
- Capacity may decrease.
- Persistent volumes may require recovery handling.

## Initial checks

kubectl get nodes

kubectl describe node NODE_NAME

kubectl get pods -A -o wide

## Inspect node conditions

Review:

- Ready
- MemoryPressure
- DiskPressure
- PIDPressure

## Inspect affected workloads

Identify pods previously scheduled
on the affected node.

## AWS investigation

Check:

- EC2 instance state.
- Auto Scaling group activity.
- EKS managed node group status.
- VPC networking.
- Relevant node system logs.

## Mitigation

Follow the approved node replacement
or managed node group recovery procedure.

Do not terminate an instance blindly
if workloads or storage require special handling.

## Recovery verification

- Node becomes Ready.
- Required DaemonSets are running.
- Workloads become available.
- Prometheus targets recover.
- Application SLOs return to normal.

## Escalation

Platform on-call and SRE.
```

### 74. Change management for alert rules

Alert rules are production configuration.

They can affect:

- On-call notifications.
- Incident detection.
- Escalation.
- Operational workload.
- Service-level reporting.

Therefore, alert-rule changes must use a review process.

Example:

```
Developer or SRE creates PR
           |
           v
Rule syntax validation
           |
           v
PromQL unit tests
           |
           v
Ownership review
           |
           v
Platform review
           |
           v
Merge to monitoring repository
           |
           v
GitOps deployment to DEV
           |
           v
QA alert testing
           |
           v
STAGING validation
           |
           v
Production approval
           |
           v
Production rule deployment
```

Critical alert changes should have an explicit rollback plan.

### 75. Change review checklist

Reviewers should check:

```
[ ] Does the expression match real metrics?
[ ] Are all labels valid?
[ ] Is the threshold justified?
[ ] Is the `for` duration appropriate?
[ ] Is the severity correct?
[ ] Is the owner correct?
[ ] Does the alert have a runbook?
[ ] Does the notification route match?
[ ] Are duplicate alerts avoided?
[ ] Are inhibition rules safe?
[ ] Are firing and non-firing cases tested?
[ ] Is the change reversible?
```

### 76. Audit and operational evidence

For production alert changes, preserve:

- Pull request number.
- Approvals.
- Prometheus rule validation results.
- Unit-test results.
- Deployment commit.
- ArgoCD synchronization status.
- Production verification.
- Notification test evidence.
- Rollback record, if applicable.

This makes it possible to answer:

- Who changed the alert?
- Why was the threshold changed?
- Who approved it?
- Which environment received the change?
- Did the notification path work?
- Was the change rolled back?

## Part 8 — CI/CD and GitOps Integration

### 77. Why alert rules belong in GitOps

We already have:

```
company-ecommerce/platform-monitoring
```

The repository is the source of truth for shared monitoring rules.

An engineer should not manually modify a production alert through the Prometheus UI and leave Git unaware of the change.

The GitOps process should deploy approved rule changes.

### 78. GitOps repository layout

Example:

```
company-ecommerce/
│
├── platform-monitoring/
│   ├── prometheus/
│   │   └── rules/
│   └── alertmanager/
│
└── gitops/
    ├── applications/
    │   ├── monitoring-dev.yaml
    │   ├── monitoring-qa.yaml
    │   ├── monitoring-staging.yaml
    │   └── monitoring-prod.yaml
    │
    └── environments/
        ├── dev/
        │   └── monitoring/
        ├── qa/
        │   └── monitoring/
        ├── staging/
        │   └── monitoring/
        └── prod/
            └── monitoring/
```

The monitoring repository holds the reusable rule definitions.

The GitOps repository defines how those assets are deployed to each environment.

### 79. ArgoCD application example

An illustrative ArgoCD Application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: monitoring-prod
  namespace: argocd

spec:
  project: platform

  source:
    repoURL: https://github.com/company-ecommerce/platform-monitoring.git
    targetRevision: main
    path: prometheus/rules

  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring

  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

This example assumes the referenced path contains deployable Kubernetes manifests. If the directory contains only source files or Helm templates, configure the appropriate Kustomize, Helm, or directory rendering mechanism instead.

For production, pin an approved revision or use a controlled promotion process rather than allowing an unrestricted branch change to deploy immediately.

### 80. Monitoring rule validation workflow

Create:

```
.github/workflows/validate-monitoring-rules.yml
```

Example:

```
name: Validate Monitoring Rules

on:
  pull_request:
    paths:
      - "prometheus/rules/**"
      - "alertmanager/**"
      - "tests/**"

permissions:
  contents: read

jobs:

  validate:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install Prometheus tools
        run: |
          echo "Install a pinned promtool version here"

      - name: Validate standalone rule files
        run: |
          find tests/rules -name "*.rules.yaml" \
            -print0 |
          while IFS= read -r -d '' file; do
            promtool check rules "$file"
          done

      - name: Run alert rule tests
        run: |
          find tests/rules -name "*.test.yaml" \
            -print0 |
          while IFS= read -r -d '' file; do
            promtool test rules "$file"
          done
```

This workflow illustrates the validation stages. Before production use:

- Pin the action to a reviewed full commit SHA.
- Install a pinned Prometheus tooling version.
- Ensure the file paths match the actual test repository structure.
- Fail the workflow if no expected rule files are found.
- Validate Kubernetes custom resources against the correct CRDs.
- Test Helm-rendered output.
- Add policy checks for mandatory labels and annotations.

The example is not a complete production workflow until those controls are implemented.

### 81. Promotion process

Alert configuration should be promoted through environments.

```
PR
 |
 v
CI validation
 |
 v
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

At each stage:

DEV

- Check that the rule loads.
- Validate the expression against available metrics.
- Confirm alert state behavior.

QA

- Simulate a controlled failure.
- Verify the alert fires.
- Verify the notification route.

STAGING

- Test with production-like metrics.
- Check alert noise.
- Review team ownership.

PRODUCTION

- Obtain approval.
- Deploy through ArgoCD.
- Confirm the PrometheusRule is loaded.
- Verify routing.
- Record deployment evidence.

### 82. Rollback alert changes

Suppose a new alert rule creates thousands of notifications.

The recovery process should be:

1. Identify the offending rule and deployment commit.
2. Confirm whether it is a rule problem or a metric-label problem.
3. Stop notification noise using a narrowly scoped, time-limited operational control if needed.
4. Revert the faulty Git commit.
5. Allow ArgoCD to reconcile the approved version.
6. Confirm the old rule is restored.
7. Confirm notifications behave normally.
8. Remove temporary silences.
9. Record the incident and corrective action.

Do not manually delete the PrometheusRule from the cluster while leaving the same resource in Git. GitOps may recreate it.

### 83. Production safeguards

For alert configuration:

- Protect the default branch.
- Require code-owner review.
- Require successful validation checks.
- Use approved production promotion.
- Avoid unreviewed direct production edits.
- Restrict Alertmanager administrative access.
- Review silence creation permissions.
- Monitor the monitoring platform itself.

The monitoring platform must not become an ungoverned path for changing production operations.

## Part 9 — Platform-Wide Alert Coverage

### 84. Standard alert coverage for all 21 services

Not every service needs identical business alerts, but every production service should have a common technical baseline.

| Service                | Baseline alert coverage                 | Additional service-specific signals                |
| ---------------------- | --------------------------------------- | -------------------------------------------------- |
| Auth Service           | Availability, 5xx, latency, CPU, memory | Authentication failures and token operation errors |
| User Service           | Availability, 5xx, latency, resources   | User profile operation failures                    |
| Customer Service       | Availability, 5xx, latency, resources   | Customer registration and update failures          |
| Catalog Service        | Availability, 5xx, latency, resources   | Catalog retrieval and synchronization failures     |
| Product Service        | Availability, 5xx, latency, resources   | Product update and retrieval failures              |
| Inventory Service      | Availability, 5xx, latency, resources   | Stock reservation and update failures              |
| Pricing Service        | Availability, 5xx, latency, resources   | Price calculation failures                         |
| Promotion Service      | Availability, 5xx, latency, resources   | Promotion evaluation failures                      |
| Cart Service           | Availability, 5xx, latency, resources   | Cart update and persistence failures               |
| Order Service          | Availability, 5xx, latency, resources   | Order creation and processing failures             |
| Payment Service        | Availability, 5xx, latency, resources   | Payment authorization and settlement failures      |
| Fraud Service          | Availability, 5xx, latency, resources   | Fraud evaluation failures and processing backlog   |
| Shipping Service       | Availability, 5xx, latency, resources   | Shipment creation failures                         |
| Delivery Service       | Availability, 5xx, latency, resources   | Delivery status update failures                    |
| Notification Service   | Availability, 5xx, latency, resources   | Notification delivery failures and queue backlog   |
| Search Service         | Availability, 5xx, latency, resources   | Search query latency and indexing failures         |
| Recommendation Service | Availability, 5xx, latency, resources   | Recommendation generation failures                 |
| Review Service         | Availability, 5xx, latency, resources   | Review submission failures                         |
| Tax Service            | Availability, 5xx, latency, resources   | Tax calculation failures                           |
| Invoice Service        | Availability, 5xx, latency, resources   | Invoice generation failures                        |
| Reporting Service      | Availability, 5xx, latency, resources   | Report generation and job failures                 |

The baseline should be shared through reusable templates.

Service-specific signals should be created with the relevant application owners. Do not invent a metric simply because the service name suggests one; confirm that the application actually emits it.

### 85. Platform-level alert inventory

In addition to application alerts, the platform needs alerts for its own components.

| Component          | Example alert                                 |
| ------------------ | --------------------------------------------- |
| Prometheus         | Scrape failure, high memory, storage pressure |
| Alertmanager       | Notification delivery failure                 |
| Node Exporter      | Target unavailable                            |
| kube-state-metrics | Target unavailable                            |
| Grafana            | Datasource unavailable                        |
| Elasticsearch      | Cluster health degraded                       |
| Logstash           | Pipeline failure or backlog                   |
| EKS                | Node NotReady, node capacity issues           |
| Persistent storage | PVC pending or volume capacity pressure       |

Monitoring systems need their own monitoring. Otherwise, the organization may lose visibility without knowing that monitoring itself has failed.

### 86. Alert coverage matrix

Use a central matrix to track rollout.

| Service        | Metrics | Availability | Error rate | Latency | Resource | Business |
| -------------- | ------- | ------------ | ---------- | ------- | -------- | -------- |
| Auth           | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| User           | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Customer       | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Catalog        | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Product        | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Inventory      | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Pricing        | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Promotion      | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Cart           | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Order          | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Payment        | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Fraud          | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Shipping       | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Delivery       | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Notification   | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Search         | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Recommendation | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Review         | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Tax            | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Invoice        | Pending | Pending      | Pending    | Pending | Pending  | Pending  |
| Reporting      | Pending | Pending      | Pending    | Pending | Pending  | Pending  |

Update each cell only after testing and recording evidence. A merged PR is not sufficient evidence that an alert works in production.

### 87. Service-level objectives and alerting

A production alerting system should eventually be tied to service-level objectives (SLOs).

For example, Order Service might define:

```
Availability objective:
99.9%

Latency objective:
95% of eligible requests below the
approved latency threshold.

Measurement window:
30 days.
```

These values are examples, not approved objectives for the project.

An SLO defines the expected service quality. An alert should detect when the service is at risk of violating that objective or is already experiencing unacceptable impact.

A basic error-budget concept:

\\[ \text{Error Budget} = 1 - \text{SLO} \\]

For a 99.9% availability objective:

\\[ 1 - 0.999 = 0.001 \\]

That means 0.1% of eligible measurement events may be unsuccessful within the defined measurement model.

A mature SLO design must specify:

- Eligible requests.
- Excluded requests.
- Measurement window.
- Error classification.
- Latency threshold.
- Data source.
- Treatment of low traffic.
- How the error budget is consumed.

Do not create SLO alerts until the measurement model is agreed upon with the service owner.

### 88. Alert review cadence

A useful governance process:

| Frequency                    | Activity                                 |
| ---------------------------- | ---------------------------------------- |
| Daily                        | Review unresolved critical alerts        |
| Weekly                       | Review alert noise and repeated warnings |
| Monthly                      | Review thresholds and ownership          |
| Quarterly                    | Review service objectives and escalation |
| After incidents              | Review missed alerts and false positives |
| After major platform changes | Revalidate affected alert rules          |

Alert quality is not a one-time configuration task. Application behavior and infrastructure change continuously.

## Part 10 — Practical Production Simulation

### 89. Scenario: Order Service starts returning 503 errors

This simulation connects monitoring, alerting, and incident response.

Initial condition:

```
Order Service replicas: 3
HTTP error rate: 0%
p95 latency: 200ms
Database pool utilization: 35%
```

At 10:00:

```
HTTP 503 responses increase.
Error rate reaches 12%.
p95 latency reaches 2.5 seconds.
Database pool utilization reaches 95%.
```

Prometheus evaluates:

```
MicroserviceHigh5xxErrorRate
DatabaseConnectionPoolNearlyExhausted
MicroserviceHighP95Latency
```

After their configured `for` durations, the relevant alerts fire.

Alertmanager groups them by service and namespace and routes the notification to the Commerce on-call team.

### 90. Engineer investigation

The on-call engineer receives:

```
CRITICAL: Order Service High 5xx Rate

Environment: production
Namespace: ecommerce
Service: order-service

Observed:
HTTP 5xx error rate: 12%

Related signals:
p95 latency elevated
Database connection pool nearly exhausted

Runbook:
high-5xx-rate.md
```

The engineer starts with:

```
kubectl get pods \
  -n ecommerce \
  -l app=order-service
```

Then:

```
kubectl get deployment \
  order-service \
  -n ecommerce
```

Check logs:

```
kubectl logs \
  -n ecommerce \
  ORDER_SERVICE_POD \
  --since=15m
```

Check the HikariCP dashboard.

The evidence suggests that database connection acquisition is timing out.

The engineer escalates to the database owner and investigates:

- RDS connection count.
- Database CPU.
- Database query latency.
- Long-running transactions.
- Connection pool utilization.
- Recent application changes.

### 91. Identify the root cause

Suppose the investigation discovers:

- A recent application release introduced a slow query.
- The query holds connections longer than expected.
- Connection pool utilization increased.
- Requests began timing out.
- HTTP 503 responses increased.

The alerting platform has successfully identified the impact and directed the investigation.

It has not automatically identified the root cause. That requires correlation between metrics, logs, database information, and deployment history.

### 92. Mitigation

The responsible teams decide on an approved mitigation.

Possible options, depending on evidence:

- Roll back the affected application release.
- Correct the query.
- Reduce unnecessary concurrent database work.
- Address a database bottleneck.
- Apply a previously tested temporary configuration change.

Do not blindly increase HikariCP maximum pool size. The database may already be at its connection limit.

If rollback is selected, use the approved GitOps promotion and rollback process.

### 93. Recovery verification

After mitigation:

```
HTTP 5xx rate: 0.2%
p95 latency: 250ms
Database pool utilization: 40%
Order creation: Successful
```

Verify:

- Prometheus metrics return to normal.
- The relevant alerts resolve.
- Alertmanager sends the resolved notification.
- The service owner confirms business functionality.
- The database owner confirms stable connections.
- No other microservice has been affected.

Record the incident evidence for the later RCA chapter.

### 94. Lessons from the simulation

The simulation demonstrates why the monitoring and alerting layers must work together.

| Layer                       | Contribution                              |
| --------------------------- | ----------------------------------------- |
| Application instrumentation | Exposes request and business metrics      |
| Prometheus                  | Detects error-rate and latency changes    |
| Grafana                     | Shows the incident timeline               |
| Alertmanager                | Routes grouped notifications              |
| ELK                         | Provides application error details        |
| Database metrics            | Reveal connection and query problems      |
| GitOps                      | Provides controlled rollback              |
| Runbook                     | Guides the engineer through investigation |

## Part 11 — Definition of Done

### 95. Alert template completion

Chapter 188 completion criteria

The chapter is complete when the platform has:

Application alerts

```
[ ] Service availability
[ ] HTTP 5xx error rate
[ ] p95 latency
[ ] Business operation failures
[ ] Database connection pool utilization
[ ] Database connection timeouts
[ ] Downstream dependency failures
```

Kubernetes alerts

```
[ ] CrashLoopBackOff
[ ] Container restarts
[ ] Pod readiness
[ ] Deployment replica availability
[ ] Incomplete rollout
[ ] Node NotReady
[ ] Node CPU utilization
[ ] Node memory utilization
[ ] Node disk utilization
[ ] Persistent volume capacity
[ ] Node pressure conditions
```

Alertmanager

```
[ ] Team routing
[ ] Severity routing
[ ] Notification integration
[ ] Grouping
[ ] Deduplication
[ ] Inhibition
[ ] Silences
[ ] Resolved notifications
```

Testing and governance

```
[ ] PromQL expressions validated
[ ] Rule syntax validated
[ ] Unit tests implemented
[ ] Notification delivery tested
[ ] Runbooks linked
[ ] Ownership labels verified
[ ] GitHub Actions validation enabled
[ ] GitOps deployment configured
[ ] Production change approval defined
[ ] Rollback procedure documented
[ ] Alert review cadence established
```

### 96. Final platform principle

A production alert should not merely say:

```
CPU is high.
```

It should help an engineer understand:

```
Which service is affected?
What is the customer impact?
How long has the condition existed?
Which team owns the problem?
What evidence should be checked?
What is the approved recovery procedure?
```

That is the difference between collecting alerts and operating a reliable production alerting platform.

Next chapter — Chapter 189: NetworkPolicy Templates

We will build reusable Kubernetes NetworkPolicy templates for the 21 microservices, including default-deny ingress and egress, service-to-service communication, DNS access, database connectivity, monitoring access, and controlled rollout through GitOps.