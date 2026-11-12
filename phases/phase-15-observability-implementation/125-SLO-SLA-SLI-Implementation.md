# 125. SLO / SLA / SLI Implementation

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Kubernetes:** Amazon EKS
> **Application:** Java + Spring Boot
> **Services:** 21 microservices
> **Metrics:** Prometheus
> **Dashboards:** Grafana
> **Logs:** ELK
> **Traces:** OpenTelemetry + Jaeger v2
> **Alerting:** Alertmanager
> **Deployment:** ArgoCD + GitOps

---

## 1. Objective

Until now, we built observability:

```text
Metrics
Logs
Traces
```

But simply having dashboards does not answer the most important production question:

> **How reliable does our application need to be?**

For example:

```text
Is 99% availability acceptable?
Is 99.9% acceptable?
Is 99.99% required?
How much downtime is allowed?
How much latency is acceptable?
When should SRE be paged?
When should deployments be stopped?
```

This is where:

```text
SLI
SLO
SLA
Error Budget
Burn Rate
```

come into the architecture.

---

# 2. SLI

**SLI = Service Level Indicator**

An SLI is a **measurement of actual service behavior**.

Examples:

```text
Availability
Latency
Error rate
Throughput
Success rate
Freshness
Queue processing delay
```

For an API:

```text
Successful requests
-------------------
Total valid requests
```

might be our availability SLI.

---

# 3. SLO

**SLO = Service Level Objective**

An SLO defines the target we want the service to achieve.

Example:

```text
99.9% successful requests
```

over:

```text
30 days
```

So:

```text
SLI = what actually happened

SLO = what we target
```

---

# 4. SLA

**SLA = Service Level Agreement**

An SLA is a contractual/business commitment between parties.

For example:

```text
Customer contract:
99.9% monthly availability
```

An SLA can include:

* service availability
* support response
* incident response
* credits/penalties
* exclusions
* measurement rules

An SLA is therefore different from an engineering SLO.

---

# 5. Simple Difference

Remember this:

```text
SLI
 ↓
Measurement

SLO
 ↓
Engineering target

SLA
 ↓
Contractual commitment
```

Example:

```text
SLI:
99.94% availability

SLO:
99.90%

SLA:
99.90%
```

These numbers are only an example; our actual targets should be defined from business requirements.

---

# 6. Error Budget

If our SLO is:

```text
99.9%
```

then the allowed unreliability is:

```text
0.1%
```

That allowed amount is the:

```text
Error Budget
```

Conceptually:

```text
100%
 |
 |--------------------|
 |     SLO 99.9%      |
 |--------------------|
 |
 |   Error Budget     |
 |       0.1%         |
 |
 0%
```

---

# 7. Why Error Budgets Matter

Without an error budget:

```text
Developer:
"Let's deploy this risky feature."
```

SRE:

```text
"We already had several incidents."
```

There may be no objective operational mechanism.

With an error budget:

```text
SLO
 ↓
Error Budget
 ↓
Budget remaining
 ↓
Deployment/reliability decision
```

The engineering organization can establish rules such as:

```text
Healthy budget
→ normal release velocity

Budget heavily consumed
→ increased reliability work

Budget exhausted
→ require reliability remediation before risky releases
```

These are **organizational policies**, not universal technical rules.

---

# 8. Availability SLI

For our ecommerce APIs:

```text
Good requests
-------------
Eligible requests
```

Example:

```text
99,900 successful requests
--------------------------
100,000 total eligible requests
```

would produce:

```text
99.9%
```

---

# 9. What Counts as Failure?

This is extremely important.

Suppose:

```text
HTTP 200 → success
HTTP 201 → success
HTTP 400 → client error
HTTP 401 → authentication error
HTTP 404 → resource not found
HTTP 429 → rate limited
HTTP 500 → server failure
HTTP 502 → gateway failure
HTTP 503 → service unavailable
```

We should not automatically classify every non-2xx response as a service failure.

For example:

```text
POST /orders
400 Invalid request
```

may be a valid application response rather than an infrastructure failure.

Therefore the SLI definition must specify **eligible requests and failure criteria**.

---

# 10. Example Order Service SLI

For:

```text
order-service
```

we might define:

```text
Availability SLI =
successful eligible HTTP requests
/
total eligible HTTP requests
```

Possible server-side failures:

```text
5xx
timeouts
connection failures
```

while some expected 4xx responses may be excluded depending on the business definition.

---

# 11. Latency SLI

Availability is not enough.

Imagine:

```text
99.99% requests succeed
```

but:

```text
95% take 10 seconds
```

The system is technically available but practically unpleasant.

So we need latency.

Example SLO:

```text
99% of eligible requests complete within 500 ms
```

This is a latency SLO.

---

# 12. Latency Percentiles

We commonly use:

```text
P50
P90
P95
P99
```

Example:

```text
P50 = 120 ms
P95 = 450 ms
P99 = 900 ms
```

Meaning approximately:

```text
50% → ≤ 120 ms
95% → ≤ 450 ms
99% → ≤ 900 ms
```

The exact interpretation depends on the histogram/query window and implementation.

---

# 13. Why P99 Matters

Average latency can hide problems.

Example:

```text
Requests:
100 ms
100 ms
100 ms
100 ms
10 sec
```

Average:

```text
2.08 sec
```

But most requests were fast.

Conversely, a high percentile exposes tail latency.

For customer-facing APIs, tail latency can be particularly important.

---

# 14. Availability + Latency

Our SLOs might therefore include:

```text
Availability:
99.9%

Latency:
99% under 500 ms
```

Now we measure two dimensions:

```text
Reliability
+
Performance
```

---

# 15. Business-Critical Services

Not every microservice necessarily needs identical SLOs.

Our 21 services have different business roles.

For example:

```text
payment-service
order-service
inventory-service
```

may have different criticality from:

```text
reporting-service
recommendation-service
```

The SLO framework should reflect actual business requirements.

---

# 16. Example SLO Classification

This is an **example design**, not a universal ranking:

### Customer transaction path

```text
order-service
payment-service
inventory-service
```

Focus heavily on:

```text
availability
latency
correctness
```

### Customer experience services

```text
catalog-service
search-service
pricing-service
```

Focus on:

```text
availability
latency
freshness
```

### Background services

```text
reporting-service
invoice-service
notification-service
```

May use:

```text
processing success
queue delay
completion time
```

rather than only synchronous HTTP availability.

---

# 17. SLI Types for Our Platform

We can define several SLI categories.

```text
1. Availability
2. Latency
3. Error rate
4. Throughput
5. Queue freshness
6. Processing success
7. Data freshness
8. Dependency success
```

---

# 18. Availability SLI

Formula:

```text
Successful eligible requests
--------------------------------
Total eligible requests
```

Prometheus can calculate this using request counters.

For example:

```text
http_requests_total
```

with labels such as:

```text
service
route
method
status
```

---

# 19. Prometheus Example

Conceptually:

```promql
sum(rate(http_server_requests_seconds_count{
  service="order-service",
  status=~"2.."
}[5m]))
/
sum(rate(http_server_requests_seconds_count{
  service="order-service"
}[5m]))
```

The exact metric name depends on our Spring Boot/OpenTelemetry/Micrometer instrumentation.

The important principle is:

> **Standardize the metric schema before building SLO queries.**

---

# 20. Better Production SLI Design

Avoid putting too many dimensions into the SLI.

Bad:

```text
service
route
method
status
customer_id
order_id
region
pod
node
```

This creates unnecessary cardinality.

Instead, SLI metrics should use controlled dimensions such as:

```text
service
route
method
status_class
environment
```

---

# 21. Error Rate SLI

Another useful SLI:

```text
5xx requests
------------
eligible requests
```

Example:

```text
100 server errors
-----------------
100,000 requests
```

gives:

```text
0.1%
```

error rate.

Availability would therefore be approximately:

```text
99.9%
```

assuming the definitions align.

---

# 22. Latency SLI with Histograms

For latency, Prometheus histograms are particularly useful.

Example:

```text
http_server_requests_seconds_bucket
```

Then we can calculate:

```text
P95
P99
```

using:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_server_requests_seconds_bucket{
      service="order-service"
    }[5m])
  )
)
```

Again, actual metric names depend on our instrumentation.

---

# 23. SLO Example

Suppose:

```text
Service:
order-service

Availability SLO:
99.9%

Latency SLO:
99% < 500 ms

Window:
30 days
```

Then we can calculate:

```text
Availability achieved:
99.96%

Latency achieved:
99.2%
```

The service currently meets both example targets.

---

# 24. Error Budget Calculation

For:

```text
99.9% SLO
```

error budget:

```text
100% - 99.9%
=
0.1%
```

For a 30-day month, this corresponds to roughly:

```text
43 minutes 12 seconds
```

of total unavailability if the SLO is defined as a pure availability target over the full month.

This is a mathematical budget, not automatically an SLA commitment.

---

# 25. Different SLOs Create Different Budgets

For:

```text
99%
```

budget:

```text
1%
```

For:

```text
99.9%
```

budget:

```text
0.1%
```

For:

```text
99.99%
```

budget:

```text
0.01%
```

A small increase in the number of nines significantly reduces allowable failure time.

---

# 26. Why 100% Is Usually a Poor Engineering Target

A theoretical:

```text
100% availability
```

leaves:

```text
0
```

error budget.

That means there is no allowance for:

```text
deployments
maintenance
unexpected failures
dependency outages
```

Real systems have failure modes.

SLOs allow us to explicitly manage acceptable unreliability.

---

# 27. Error Budget Remaining

Suppose our monthly budget is:

```text
43m 12s
```

and incidents consumed:

```text
15m
```

Remaining:

```text
28m 12s
```

We can visualize:

```text
Error Budget
100% |████████████████████|
     |████████████        |
     |████████            |
  0% +--------------------+
```

The exact visualization belongs in Grafana.

---

# 28. Error Budget Policy

Our organization can define policies.

Example:

```text
Budget > 50%
→ normal release process

Budget 20–50%
→ reliability review

Budget < 20%
→ prioritize reliability work

Budget exhausted
→ require explicit reliability review before selected risky changes
```

These are **example policies**.

A real organization should agree on the thresholds.

---

# 29. Burn Rate

Error budget tells us:

> How much budget have we consumed?

Burn rate tells us:

> **How quickly are we consuming it?**

Suppose:

```text
monthly error budget = 0.1%
```

and we're consuming it much faster than expected.

That indicates a serious reliability problem.

---

# 30. Example Burn Rate

Normal:

```text
Budget consumption:
slow
```

Incident:

```text
Budget consumption:
very fast
```

Conceptually:

```text
Error Budget
|
|████████████████
|██████████
|████
|██
+--------------------> Time
```

A steep drop indicates high burn.

---

# 31. Why Burn Rate Is Useful

Suppose our SLO is:

```text
99.9%
```

An incident producing:

```text
0.2% failures
```

for a short period may consume a large portion of the monthly budget.

Burn-rate alerting can identify this quickly.

---

# 32. Multi-Window Burn Rate

A common production technique is to use:

```text
short window
+
long window
```

For example:

```text
5-minute window
+
1-hour window
```

or:

```text
30-minute window
+
6-hour window
```

The exact windows depend on the SLO and operational requirements.

The purpose is to detect both:

```text
fast severe incidents
```

and:

```text
slower sustained degradation
```

---

# 33. SLO Architecture

Our platform becomes:

```text
Application
    |
    v
Prometheus metrics
    |
    v
SLI calculations
    |
    v
SLO
    |
    +----> Error Budget
    |
    +----> Burn Rate
    |
    +----> Alertmanager
    |
    +----> Grafana
```

---

# 34. SLO Dashboard

Grafana should show:

```text
Service
SLO
Current SLI
Target
Error Budget
Budget Remaining
Burn Rate
```

Example:

```text
order-service

Availability
Current: 99.96%
Target: 99.90%

Error Budget
Remaining: 64%

Latency
Current: 99.2% < 500ms
Target: 99%
```

---

# 35. SLO Dashboard for 21 Services

We can create a variable:

```text
service
```

Then select:

```text
order-service
```

or:

```text
payment-service
```

or:

```text
inventory-service
```

or:

```text
all
```

The dashboard can display:

```text
Service | Availability | Latency | Budget | Burn Rate
```

---

# 36. SLOs Should Not Be Based Only on CPU

This is important.

CPU:

```text
85%
```

does not automatically mean:

```text
service reliability = bad
```

Likewise:

```text
CPU = 30%
```

does not mean:

```text
service = healthy
```

SLOs should measure **customer-visible or business-relevant service behavior**.

---

# 37. Example

Suppose:

```text
CPU = 95%
```

but:

```text
availability = 99.99%
latency = within target
```

The service might still be meeting its SLO.

Conversely:

```text
CPU = 30%
```

but:

```text
HTTP 5xx = 10%
```

means customers are experiencing failures.

Therefore:

```text
Infrastructure metrics
```

and:

```text
Service-level objectives
```

serve different purposes.

---

# 38. SLOs for Asynchronous Services

Not every service is HTTP.

Our platform contains:

```text
SQS consumers
Kafka consumers
RabbitMQ consumers
```

For these, useful SLIs include:

```text
message processing success
consumer lag
queue age
processing latency
DLQ rate
```

---

# 39. SQS SLO Example

For:

```text
notification-service
```

we might define:

```text
99% of notification messages
processed within 60 seconds
```

The SLI becomes:

```text
messages processed within target
----------------------------------
eligible messages
```

This is more meaningful than:

```text
HTTP availability
```

for a background worker.

---

# 40. Kafka SLO Example

For:

```text
inventory-service
```

as a Kafka consumer:

```text
99% of messages processed within X seconds
```

or a lag objective:

```text
consumer lag remains below agreed threshold
```

But lag should be tied to actual customer/business impact rather than treating a single lag number as universally bad.

---

# 41. Freshness SLI

For:

```text
catalog-service
search-service
recommendation-service
```

data freshness may matter.

Example:

```text
99% of catalog updates visible
within 2 minutes
```

This becomes:

```text
Freshness SLI
```

rather than simple API availability.

---

# 42. Correctness

Availability alone does not guarantee correctness.

Imagine:

```text
POST /orders
HTTP 200
```

but the order is created incorrectly.

The request is technically successful.

The business outcome is not.

Therefore critical workflows may require additional business indicators.

Example:

```text
payment authorization success
inventory reservation success
order completion success
```

These should be defined carefully because business correctness is usually more complex than a single HTTP metric.

---

# 43. Order Journey SLO

Our critical order path:

```text
Customer
   ↓
order-service
   ↓
payment-service
   ↓
fraud-service
   ↓
inventory-service
   ↓
shipping-service
   ↓
notification
```

We can define a business-level objective around the complete workflow.

Example:

```text
99% of eligible orders
reach a successful or clearly resolved state
within 30 seconds
```

This is more meaningful than measuring each service independently.

---

# 44. Service SLO vs Journey SLO

We therefore have:

```text
Service SLOs
```

and:

```text
Business Journey SLOs
```

Example:

```text
order-service
99.9% availability

payment-service
99.95% availability

Order completion journey
99% successful within target
```

The business journey can still fail even when individual services meet their own SLOs.

---

# 45. Dependency Impact

Suppose:

```text
order-service
```

has:

```text
99.99% availability
```

but:

```text
payment-service
```

has:

```text
99.5%
```

The order journey can be affected by the dependency.

Therefore SLO design should consider:

```text
dependencies
timeouts
retries
fallbacks
business workflows
```

---

# 46. Don't Create Too Many SLOs

A common mistake is:

```text
Every endpoint
Every pod
Every database query
Every dependency
Every queue
```

gets its own SLO.

This creates:

```text
SLO explosion
```

Instead, start with the most important user journeys and services.

---

# 47. SLO Ownership

Example:

```text
order-service
Owner: Backend Team

payment-service
Owner: Payment Team

EKS platform
Owner: Platform Team

Observability
Owner: SRE/Platform

Order journey
Shared ownership
```

Ownership must be explicit.

---

# 48. SLO Repository Structure

We can keep SLO definitions in Git.

Example:

```text
platform/
└── slo/
    ├── services/
    │   ├── order-service.yaml
    │   ├── payment-service.yaml
    │   ├── inventory-service.yaml
    │   └── ...
    │
    ├── recording-rules/
    │   ├── availability.yaml
    │   └── latency.yaml
    │
    ├── alerts/
    │   └── burn-rate.yaml
    │
    └── README.md
```

This makes SLOs:

```text
version-controlled
reviewable
auditable
repeatable
```

---

# 49. Example SLO Definition

Conceptual YAML:

```yaml
service: order-service

slos:
  - name: availability
    target: 99.9%
    window: 30d

  - name: latency
    target: 99%
    threshold: 500ms
    window: 30d
```

This is a platform configuration model.

The actual Prometheus recording rules and alert rules implement it.

---

# 50. Recording Rules

Instead of repeatedly executing expensive PromQL queries, we can use recording rules.

Conceptually:

```yaml
groups:
  - name: order-service-slo
    rules:

      - record: order_service:availability_ratio
        expr: |
          ...
```

Then Grafana and Alertmanager can query the precomputed series.

---

# 51. Why Recording Rules Matter

Suppose we have:

```text
21 services
×
multiple routes
×
multiple windows
×
multiple SLO calculations
```

Running huge queries every time Grafana loads can be expensive.

Recording rules provide:

```text
raw metrics
    ↓
precomputed SLI
    ↓
SLO dashboard/alerts
```

---

# 52. Burn Rate Alert

Conceptually:

```yaml
groups:
  - name: slo-burn-rate
    rules:

      - alert: OrderServiceHighErrorBudgetBurn
        expr: |
          ...
        for: 5m

        labels:
          severity: critical
          service: order-service

        annotations:
          summary: "Order service is consuming error budget rapidly"
```

The exact expression depends on the SLO definition.

---

# 53. Alert Severity

Example:

```text
critical
warning
info
```

For SLOs:

```text
Critical:
rapid budget consumption

Warning:
sustained elevated burn

Info:
budget trending downward
```

The exact routing policy should be agreed with SRE/operations.

---

# 54. Alert Routing

Our architecture:

```text
Prometheus
     |
     v
PrometheusRule
     |
     v
Alertmanager
     |
     +---- Slack
     |
     +---- PagerDuty
     |
     +---- Email
```

This is already covered in Chapter 120.

Now SLO alerts become another input.

---

# 55. SLO vs Infrastructure Alerts

Infrastructure alert:

```text
Node CPU > 90%
```

SLO alert:

```text
Order availability SLO burn rate is excessive
```

The second is much closer to customer impact.

We still need infrastructure alerts because they can provide early warning and help diagnose causes.

---

# 56. Alert Hierarchy

Our platform can use:

```text
Business SLO
      |
      v
Service SLO
      |
      v
Golden Signals
      |
      v
Infrastructure
      |
      v
Raw system metrics
```

This helps prioritize alerts around impact.

---

# 57. Golden Signals

For services:

```text
Traffic
Errors
Latency
Saturation
```

These complement SLOs.

Example:

```text
Traffic ↑
Errors ↑
Latency ↑
CPU ↑
```

could indicate a traffic-driven saturation incident.

---

# 58. SLO and Deployments

Suppose:

```text
Error budget:
80% remaining
```

Normal deployment process can continue.

After a deployment:

```text
Error rate ↑
Latency ↑
Budget burn ↑
```

The deployment system and release process can use these signals for automated or human-controlled rollback decisions.

---

# 59. GitOps Deployment + SLO

Our flow becomes:

```text
Developer
    ↓
PR
    ↓
CI
    ↓
Security
    ↓
ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
EKS
    ↓
Prometheus
    ↓
SLO
    ↓
Error Budget
    ↓
Release decision
```

This connects delivery with reliability.

---

# 60. Canary Deployment

For canary:

```text
95% → current version
5%  → new version
```

We compare:

```text
availability
latency
errors
business metrics
```

If the new version consumes error budget disproportionately, the release process can stop or roll back according to our deployment policy.

---

# 61. Example Canary Analysis

Old version:

```text
5xx = 0.1%
P95 = 300 ms
```

New version:

```text
5xx = 3.2%
P95 = 1.8 sec
```

The new version is showing degraded service behavior.

Our release process should detect this through metrics/SLO signals before increasing traffic further.

---

# 62. SLO During Incidents

Suppose:

```text
10:00
deployment starts

10:05
5xx increases

10:07
burn rate becomes critical

10:08
rollback

10:12
error rate returns to normal
```

The SLO dashboard tells us:

```text
how much budget was consumed
```

and incident management records:

```text
why it happened
```

---

# 63. SLO + RCA

After the incident:

```text
Incident
   ↓
Timeline
   ↓
Root cause
   ↓
Customer impact
   ↓
Error budget consumed
   ↓
Corrective action
```

Example corrective actions:

```text
increase test coverage
improve timeout
fix database query
add capacity
improve canary
add alert
improve rollback
```

---

# 64. SLO + Capacity Planning

Suppose traffic grows:

```text
10k req/s
→
50k req/s
→
100k req/s
```

Our SLOs remain:

```text
availability
latency
```

Capacity planning asks:

```text
Can infrastructure maintain those SLOs?
```

Therefore:

```text
SLO
 ↓
Capacity requirement
 ↓
Scaling
```

This will connect directly to Chapters:

```text
127 Load Testing
128 Performance Testing
129 Kubernetes Capacity Planning
130 21-Microservice Scaling
```

---

# 65. SLO + Autoscaling

HPA normally reacts to:

```text
CPU
memory
custom metrics
```

KEDA can react to:

```text
SQS queue
Kafka lag
```

SLOs measure:

```text
customer/service outcome
```

These are complementary.

```text
HPA/KEDA
   ↓
capacity adjustment

SLO
   ↓
reliability outcome
```

---

# 66. SLO for RDS

RDS itself may have:

```text
CPU
connections
IOPS
latency
storage
```

But our application SLO might be:

```text
order API availability
order API latency
```

RDS metrics help explain why the SLO is being violated.

---

# 67. SLO for Redis

Redis:

```text
cache hit rate
latency
connections
memory
evictions
```

can explain application behavior.

But the application SLO remains:

```text
customer-facing outcome
```

unless cache freshness/latency itself is a defined service objective.

---

# 68. SLO for Kafka

Kafka:

```text
consumer lag
producer errors
broker health
ISR
throughput
```

help explain asynchronous SLO violations.

For example:

```text
Kafka lag ↑
     ↓
notification delay ↑
     ↓
notification SLO violation
```

---

# 69. SLO for SQS

Example:

```text
Oldest message age
```

can become an important SLI for asynchronous workloads.

Example objective:

```text
99% of eligible messages processed within 60 seconds
```

This is more meaningful than simply:

```text
queue contains messages
```

---

# 70. Business SLO Example

For our ecommerce platform:

```text
Order Creation SLO

99% of eligible orders should reach
a successful or clearly resolved state
within 30 seconds.
```

Supporting service SLOs:

```text
order-service
payment-service
inventory-service
```

Then:

```text
Order Journey
      |
      +-- order-service
      +-- payment-service
      +-- fraud-service
      +-- inventory-service
```

---

# 71. Error Budget Dashboard

Grafana should show:

```text
                    SLO STATUS

Order Service
Availability:       99.94%
Target:             99.90%

Latency:
99% < 500ms

Error Budget:
72% remaining

Burn Rate:
1.2x
```

Again, these numbers are example values.

---

# 72. Service Overview Dashboard

For all services:

```text
+------------------+--------+---------+---------+
| Service          | Avail  | Latency | Budget  |
+------------------+--------+---------+---------+
| order-service    | 99.94% | 420ms  | 72%     |
| payment-service  | 99.97% | 380ms  | 81%     |
| inventory-service| 99.91% | 490ms  | 54%     |
| catalog-service  | 99.99% | 180ms  | 96%     |
+------------------+--------+---------+---------+
```

This is an operational dashboard, not a ranking of services; it is showing each service against its own defined objectives.

---

# 73. SLO Documentation

Each SLO should document:

```text
Name
Owner
Purpose
SLI definition
SLO target
Measurement window
Eligible traffic
Exclusions
Error budget policy
Alert thresholds
Dashboard
Runbook
```

Example:

```text
SLO:
Order API Availability

Owner:
Backend Team

Target:
99.9%

Window:
30 days

Runbook:
order-service availability incident
```

---

# 74. SLO Review Process

SLOs should not be permanently frozen.

Review periodically:

```text
Are customers actually impacted?
Is the target achievable?
Is the target too loose?
Are we measuring the right thing?
Are alerts actionable?
Is the error budget policy useful?
```

Changes should go through normal engineering governance.

---

# 75. Common SLO Mistakes

### Mistake 1

Using:

```text
CPU < 80%
```

as the main SLO.

### Mistake 2

Creating hundreds of SLOs.

### Mistake 3

Using average latency instead of tail latency.

### Mistake 4

Counting all 4xx responses as server failures without defining eligibility.

### Mistake 5

Ignoring asynchronous workloads.

### Mistake 6

No ownership.

### Mistake 7

No error-budget policy.

### Mistake 8

No runbook.

### Mistake 9

No business journey SLO.

### Mistake 10

Alerting on every small SLO fluctuation.

---

# 76. Production SLO Architecture

Our complete flow:

```text
                         CUSTOMER
                            |
                            v
                     E-Commerce APIs
                            |
                            v
                      Spring Boot
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          Metrics         Logs          Traces
             |              |              |
        Prometheus          ELK        OpenTelemetry
             |                           |
             v                           v
        SLI Recording               Collector
             |                           |
             v                           v
            SLO                       Jaeger
             |                           |
       +-----+------+                    |
       |            |                    |
       v            v                    |
 Error Budget   Burn Rate                |
       |            |                    |
       +-----+------+--------------------+
             |
             v
        Alertmanager
             |
       +-----+------+
       |            |
       v            v
    PagerDuty     Slack
```

---

# 77. Complete Reliability Model

Our project now has:

```text
SLI
 ↓
SLO
 ↓
Error Budget
 ↓
Burn Rate
 ↓
Alert
 ↓
Incident
 ↓
RCA
 ↓
Corrective Action
 ↓
Improved Reliability
```

This creates an actual reliability engineering feedback loop.

---

# 78. Production Checklist

```text
[ ] Define business-critical journeys
[ ] Define service-level objectives
[ ] Define SLIs
[ ] Define eligible traffic
[ ] Define failure criteria
[ ] Define latency thresholds
[ ] Define measurement windows
[ ] Create Prometheus recording rules
[ ] Create SLO dashboards
[ ] Calculate error budgets
[ ] Calculate burn rates
[ ] Configure burn-rate alerts
[ ] Configure Alertmanager routing
[ ] Create SLO runbooks
[ ] Assign service ownership
[ ] Define error-budget policy
[ ] Include asynchronous workloads
[ ] Include business journey SLOs
[ ] Test SLO alerts
[ ] Review SLOs periodically
```

---

# 79. Chapter Completion Checklist

```text
[ ] Understand SLI
[ ] Understand SLO
[ ] Understand SLA
[ ] Understand error budget
[ ] Understand burn rate
[ ] Understand availability SLI
[ ] Understand latency SLI
[ ] Understand percentiles
[ ] Understand error-rate SLI
[ ] Understand asynchronous SLOs
[ ] Understand freshness SLI
[ ] Understand business journey SLO
[ ] Understand Prometheus SLI calculations
[ ] Understand recording rules
[ ] Understand burn-rate alerts
[ ] Understand SLO dashboards
[ ] Understand SLO ownership
[ ] Understand SLO deployment impact
[ ] Understand SLO during incidents
[ ] Understand error-budget policies
```

---

# 80. Where We Are Now

Our platform has progressed from basic monitoring to a complete reliability model:

```text
                         PLATFORM
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
      Metrics             Logs               Traces
        |                   |                   |
   Prometheus              ELK          OpenTelemetry
        |                                      |
        |                                      v
        |                                    Jaeger
        |                                      |
        +------------------+-------------------+
                           |
                           v
                         SLOs
                           |
                +----------+----------+
                |                     |
                v                     v
          Error Budget            Burn Rate
                |                     |
                +----------+----------+
                           |
                           v
                      Alertmanager
                           |
                           v
                        Incident
                           |
                           v
                          RCA
```

### Next chapter

# **126 — Production Observability Runbooks**

There we will turn everything we've built into **actual production troubleshooting runbooks** for:

```text
High API latency
High 5xx
Pod CrashLoopBackOff
OOMKilled
RDS connection exhaustion
Redis failure
SQS backlog
Kafka consumer lag
Jaeger/OTel failure
Prometheus failure
ELK/logging failure
ALB 5xx
Node failure
AZ failure
```

Each runbook will follow the practical production sequence:

```text
Alert
 ↓
Confirm
 ↓
Scope
 ↓
Investigate
 ↓
Correlate Metrics + Logs + Jaeger
 ↓
Mitigate
 ↓
Verify
 ↓
Rollback/Recovery if required
 ↓
RCA
 ↓
Prevent recurrence
```
