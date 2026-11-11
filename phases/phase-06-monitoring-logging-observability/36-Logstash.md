# 36 — Logstash

> Production-focused DevOps notes for running Logstash in an AWS EKS-based microservices platform and integrating it with Fluent Bit and Elasticsearch.

---

## 1. What is Logstash?

Logstash is a server-side data processing pipeline used to:

* Receive logs/events
* Parse unstructured data
* Transform fields
* Normalize different log formats
* Enrich events
* Route events
* Filter unwanted data
* Send events to Elasticsearch or other destinations

In our project, Logstash is the **central processing layer** between Fluent Bit and Elasticsearch.

```text
21 Microservices
       │
       ▼
stdout / stderr
       │
       ▼
Kubernetes container logs
       │
       ▼
Fluent Bit
       │
       ▼
Logstash
       │
       ├── Parse
       ├── Filter
       ├── Normalize
       ├── Enrich
       └── Route
       │
       ▼
Elasticsearch
       │
       ▼
Kibana
```

---

# 2. Why Do We Need Logstash?

A common question is:

> If Fluent Bit can send logs directly to Elasticsearch, why use Logstash?

That's correct.

This architecture is technically possible:

```text
Fluent Bit
     │
     ▼
Elasticsearch
     │
     ▼
Kibana
```

For many environments, this is actually preferable because it is simpler.

However, our project intentionally uses:

```text
Fluent Bit
     │
     ▼
Logstash
     │
     ▼
Elasticsearch
```

because Logstash provides a centralized processing layer.

For example:

```text
payment-service
    │
    ├── parse JSON
    ├── normalize fields
    ├── remove sensitive fields
    ├── add environment
    ├── add service metadata
    └── route to payment index
```

This becomes particularly useful when many applications produce different log formats.

---

# 3. Logstash's Position in Production

Logstash should normally **not run as a sidecar inside every application Pod**.

Bad design:

```text
order Pod
 ├── application
 └── Logstash

payment Pod
 ├── application
 └── Logstash

catalog Pod
 ├── application
 └── Logstash
```

This creates unnecessary:

* CPU consumption
* Memory consumption
* operational complexity
* configuration duplication

Instead:

```text
Node 1
 ├── Application Pods
 └── Fluent Bit

Node 2
 ├── Application Pods
 └── Fluent Bit

Node 3
 ├── Application Pods
 └── Fluent Bit
          │
          ▼
     Logstash Service
       ├── Pod 1
       └── Pod 2
          │
          ▼
     Elasticsearch
```

Fluent Bit handles node-level collection.

Logstash handles centralized processing.

---

# 4. Logstash in EKS

Our production placement:

```text
EKS
│
├── production
│   ├── order-service
│   ├── payment-service
│   ├── catalog-service
│   └── ...
│
├── logging
│   ├── Fluent Bit
│   ├── Logstash
│   ├── Elasticsearch
│   └── Kibana
│
├── monitoring
│   ├── Prometheus
│   ├── Grafana
│   └── Alertmanager
│
└── tracing
    ├── OpenTelemetry
    └── Jaeger
```

Logstash belongs in the **logging/platform layer**, not in the application namespace.

---

# 5. Namespace

Create a dedicated namespace:

```bash
kubectl create namespace logging
```

Production:

```text
logging
 ├── fluent-bit
 ├── logstash
 ├── elasticsearch
 └── kibana
```

The exact namespace separation can vary between organizations.

Some organizations separate:

```text
logging
 ├── fluent-bit
 └── logstash

elasticsearch
 └── Elasticsearch

kibana
 └── Kibana
```

The important point is to separate the logging platform from business workloads.

---

# 6. Logstash Architecture

Logstash follows:

```text
INPUT
  │
  ▼
FILTER
  │
  ▼
OUTPUT
```

For our architecture:

```text
Fluent Bit
     │
     ▼
Input
     │
     ▼
JSON Parsing
     │
     ▼
Field Normalization
     │
     ▼
Filtering
     │
     ▼
Routing
     │
     ▼
Elasticsearch Output
```

---

# 7. Basic Pipeline

A Logstash pipeline looks like:

```text
input {
    ...
}

filter {
    ...
}

output {
    ...
}
```

Example:

```conf
input {
  http {
    port => 8080
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "application-logs-%{+YYYY.MM.dd}"
  }
}
```

This is only an illustrative example.

The exact input plugin must match the transport configured in Fluent Bit.

---

# 8. Fluent Bit → Logstash Transport

There are multiple ways to connect Fluent Bit and Logstash.

For example:

```text
Fluent Bit
   │
   ├── HTTP
   │
   ├── Forward
   │
   ├── Beats-compatible transport
   │
   └── other supported protocols
```

The important production rule is:

> **Do not assume that every Fluent Bit output plugin is compatible with every Logstash input plugin.**

The exact configuration depends on:

* Fluent Bit version
* Logstash version
* installed plugins
* protocol
* TLS requirements
* authentication requirements

Always validate the chosen Fluent Bit output and Logstash input combination against the versions being deployed.

---

# 9. Example HTTP Architecture

For an illustrative HTTP design:

```text
Fluent Bit
    │
    │ HTTP
    ▼
Logstash Service
    │
    ▼
Logstash HTTP Input
```

Logstash:

```conf
input {
  http {
    port => 8080
  }
}
```

Kubernetes:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: logging
spec:
  selector:
    app: logstash
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Fluent Bit can then target:

```text
http://logstash.logging.svc.cluster.local:8080
```

For production, use TLS and authentication where supported by the selected transport.

---

# 10. Logstash Deployment or StatefulSet?

This is an important architectural decision.

For a stateless processing layer:

```text
Deployment
```

is normally appropriate.

Why?

Logstash processing Pods generally do not need stable identities.

Example:

```text
Logstash Deployment
       │
       ├── Pod 1
       ├── Pod 2
       └── Pod 3
```

Kubernetes can replace Pods without requiring persistent identity.

Therefore:

> **Logstash is commonly deployed as a Deployment rather than a StatefulSet when persistent queues are not tied to stable Pod identity.**

---

# 11. When Persistence Changes the Design

Logstash can use **Persistent Queue (PQ)** storage.

Conceptually:

```text
Fluent Bit
    │
    ▼
Logstash
    │
    ├── Memory
    │
    └── Persistent Queue
            │
            ▼
      Elasticsearch
```

Persistent queues allow events to survive certain Logstash process failures.

However:

> Persistent Queue does not magically make Logstash highly available.

You still need:

* multiple replicas
* correct storage
* pod distribution
* capacity planning
* Elasticsearch availability
* monitoring

---

# 12. High Availability

Production example:

```text
                    ┌── Logstash Pod 1
Fluent Bit ────────►│
                    ├── Logstash Pod 2
                    │
                    └── Logstash Pod 3
                              │
                              ▼
                       Elasticsearch
```

Recommended considerations:

```text
replicas: 2+
```

The exact number depends on:

* log volume
* event size
* filter complexity
* Elasticsearch throughput
* CPU
* memory
* peak traffic

Do not blindly choose three replicas simply because the cluster has three AZs.

---

# 13. Pod Distribution

We don't want:

```text
Node 1
 ├── Logstash-1
 ├── Logstash-2
 └── Logstash-3
```

If Node 1 fails:

```text
ALL LOGSTASH PODS
       ↓
     DOWN
```

Instead:

```text
AZ-A
 └── Logstash-1

AZ-B
 └── Logstash-2

AZ-C
 └── Logstash-3
```

Use:

* topology spread constraints
* pod anti-affinity
* PodDisruptionBudget

depending on the deployment model.

---

# 14. PodDisruptionBudget

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: logstash
  namespace: logging
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: logstash
```

This protects against excessive **voluntary disruption**.

For example:

```text
Node maintenance
       │
       ▼
Pod eviction
       │
       ▼
PDB prevents too many Logstash replicas
from being voluntarily disrupted simultaneously
```

---

# 15. Logstash Resources

Logstash is JVM-based.

It can consume significant:

* CPU
* memory
* heap
* disk

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

These are starting examples only.

Real production sizing should come from:

```text
events/sec
+
average event size
+
filter complexity
+
Elasticsearch throughput
+
peak traffic
```

---

# 16. JVM Heap

Logstash runs on the JVM.

Heap configuration is therefore important.

Conceptually:

```text
Container Memory
       │
       ├── JVM Heap
       ├── JVM Native Memory
       ├── Plugins
       ├── Buffers
       └── OS/container overhead
```

Do not allocate:

```text
Container = 4 GiB
Heap = 4 GiB
```

That leaves no room for non-heap memory.

A common starting principle is to leave meaningful headroom outside the JVM heap and then tune using actual metrics.

Example:

```text
Container memory = 4 GiB

JVM heap ≈ 2 GiB
Remaining ≈ 2 GiB
for non-heap/container overhead
```

The correct value depends on workload and Logstash/JVM version.

---

# 17. Pipeline Workers

Logstash can process events using pipeline workers.

Conceptually:

```text
             ┌── Worker 1
Input ───────┼── Worker 2
             ├── Worker 3
             └── Worker 4
```

More workers do not automatically mean more performance.

If the bottleneck is Elasticsearch:

```text
More Logstash workers
        ↓
More events
        ↓
Elasticsearch overloaded
```

Therefore tune the complete pipeline.

---

# 18. Pipeline Configuration

A production pipeline can be separated into configuration files.

Example:

```text
/etc/logstash/
├── pipelines.yml
└── pipeline/
    ├── application.conf
    ├── audit.conf
    └── infrastructure.conf
```

Conceptually:

```yaml
- pipeline.id: application
  path.config: "/usr/share/logstash/pipeline/application.conf"

- pipeline.id: audit
  path.config: "/usr/share/logstash/pipeline/audit.conf"
```

This allows different streams to have different processing pipelines.

---

# 19. Why Multiple Pipelines?

Imagine our platform receives:

```text
Application logs
Audit logs
Security logs
Infrastructure logs
```

Instead of:

```text
Everything
   ↓
One huge pipeline
```

we can use:

```text
Application logs ──► application pipeline
Audit logs ────────► audit pipeline
Security logs ─────► security pipeline
Infrastructure ────► infrastructure pipeline
```

This makes large environments easier to operate.

---

# 20. JSON Application Logs

Our Spring Boot applications should preferably produce structured JSON logs.

Instead of:

```text
2026-09-10 INFO Order 123 created
```

produce something conceptually like:

```json
{
  "timestamp": "2026-09-10T10:30:00Z",
  "level": "INFO",
  "service": "order-service",
  "environment": "production",
  "trace_id": "abc123",
  "order_id": "12345",
  "message": "Order created"
}
```

This is much easier to search.

Kibana can query:

```text
service:order-service
```

or:

```text
level:ERROR
```

or:

```text
order_id:12345
```

---

# 21. JSON Parsing

If Fluent Bit forwards the application message as a JSON string, Logstash can parse it.

Example:

```conf
filter {
  json {
    source => "message"
  }
}
```

Before:

```text
message = "{\"level\":\"ERROR\",\"service\":\"order-service\"}"
```

After parsing:

```text
level = ERROR
service = order-service
```

---

# 22. Field Normalization

Different services may use:

```text
serviceName
service_name
application
app
```

This makes searching painful.

Logstash can normalize them into:

```text
service.name
```

Conceptually:

```conf
mutate {
  rename => {
    "serviceName" => "[service][name]"
  }
}
```

Now Kibana can use a consistent field.

---

# 23. Environment Enrichment

We can add:

```text
environment=production
cluster=eks-prod
region=ap-south-1
```

Example:

```conf
mutate {
  add_field => {
    "environment" => "production"
    "cluster" => "eks-prod"
    "region" => "ap-south-1"
  }
}
```

However, environment-specific configuration should preferably be supplied through deployment configuration rather than hard-coded into reusable pipelines.

---

# 24. Kubernetes Metadata

Fluent Bit should normally enrich records with Kubernetes metadata.

Example:

```text
namespace = production
pod = order-service-xxxx
container = order-service
node = ip-10-0-x-x
```

Then Logstash can use these fields.

For example:

```text
namespace=production
service=order-service
```

can determine the Elasticsearch destination.

---

# 25. Routing Logs

Suppose we want:

```text
order-service
payment-service
catalog-service
```

to different indexes.

Conceptually:

```conf
output {
  if [service][name] == "order-service" {
    elasticsearch {
      index => "order-service-%{+YYYY.MM.dd}"
    }
  } else if [service][name] == "payment-service" {
    elasticsearch {
      index => "payment-service-%{+YYYY.MM.dd}"
    }
  }
}
```

For 21 microservices, manually creating large conditional blocks can become difficult.

A better production design is to use a consistent index/data-stream naming strategy.

---

# 26. Better Index Strategy

Instead of:

```text
order-index
payment-index
catalog-index
...
```

consider a consistent pattern such as:

```text
logs-production-order-service
logs-production-payment-service
logs-production-catalog-service
```

or an Elastic data-stream-oriented naming strategy.

The exact naming convention should be standardized across the organization.

---

# 27. Don't Create Unlimited Indexes

A common mistake:

```text
Every Pod
    ↓
New Elasticsearch index
```

This can create huge numbers of shards and metadata overhead.

For example:

```text
21 services
×
multiple environments
×
daily indexes
×
many shards
```

can grow rapidly.

Index strategy must consider:

* document volume
* retention
* shard size
* search patterns
* rollover
* lifecycle policies

---

# 28. Filtering Logs

Not every log is valuable.

Example:

```text
health check
health check
health check
health check
health check
```

Millions of repetitive health-check logs can generate unnecessary cost.

Logstash can filter unwanted events.

Conceptually:

```conf
if [message] =~ /health-check/ {
    drop { }
}
```

But filtering must be done carefully.

Don't delete security or audit information simply because it appears noisy.

---

# 29. Sensitive Data

This is extremely important.

Never allow:

```text
password
JWT token
credit card
secret
AWS credentials
database password
```

to appear in logs.

Bad:

```json
{
  "username": "user1",
  "password": "MyPassword123"
}
```

Better:

```json
{
  "username": "user1",
  "password": "[REDACTED]"
}
```

Ideally, sensitive data should never be logged in the first place.

Logstash redaction should be considered a **secondary safety control**, not the primary control.

---

# 30. Grok

Grok is useful when logs are unstructured.

Example:

```text
2026-09-10 10:30:00 ERROR order-service Order 123 failed
```

Grok can extract:

```text
timestamp
level
service
order_id
message
```

Conceptually:

```conf
grok {
  match => {
    "message" =>
    "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{DATA:service} Order %{NUMBER:order_id} %{GREEDYDATA:message}"
  }
}
```

But:

> Prefer structured JSON logs over parsing application text with Grok whenever you control the application.

JSON is usually more reliable than complex regular expressions.

---

# 31. Multiline Logs

Java applications can generate:

```text
Exception
    at com.company.order.OrderService...
    at com.company.order.OrderController...
    at ...
```

If every line becomes a separate event:

```text
Event 1: Exception
Event 2: at ...
Event 3: at ...
Event 4: at ...
```

Kibana becomes difficult to use.

The preferred approach is generally to handle multiline aggregation as close to collection as practical.

For Kubernetes logs:

```text
Application
     ↓
stdout
     ↓
container runtime
     ↓
Fluent Bit
     ↓
Logstash
```

Fluent Bit can perform multiline handling.

Logstash can also handle multiline in some designs, but doing it centrally can increase processing complexity.

Therefore:

> **Prefer application/collector-side multiline handling where possible; use Logstash multiline processing when there is a specific reason and the chosen transport preserves the required event boundaries.**

---

# 32. Backpressure

This is one of the most important production concepts.

Suppose:

```text
Applications
   ↓
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
```

Elasticsearch becomes slow.

Then:

```text
Elasticsearch
      ↓
slow
      ↓
Logstash output slows
      ↓
Logstash queue grows
      ↓
Logstash memory/disk increases
      ↓
Fluent Bit buffers grow
      ↓
Node disk pressure
```

This is a complete pipeline problem.

Do not troubleshoot only Elasticsearch.

---

# 33. Persistent Queues

Logstash Persistent Queue provides disk-backed buffering.

Conceptually:

```text
             ┌── memory
Input ───────┤
             └── persistent queue
                     │
                     ▼
                Elasticsearch
```

If Elasticsearch temporarily becomes unavailable:

```text
Elasticsearch DOWN
       │
       ▼
Logstash
       │
       ▼
Persistent Queue
       │
       ▼
wait
```

Once Elasticsearch recovers:

```text
Persistent Queue
       │
       ▼
Logstash
       │
       ▼
Elasticsearch
```

This can protect against short outages.

---

# 34. Persistent Queue Is Not Infinite

Never think:

> "We enabled persistent queues, so logs cannot be lost."

Incorrect.

Disk is finite.

Example:

```text
PQ capacity = 100 GB
Incoming logs = 50 GB/hour
Elasticsearch unavailable = 4 hours
```

Required:

```text
50 × 4 = 200 GB
```

But available:

```text
100 GB
```

The queue eventually fills.

Therefore calculate:

```text
Required buffer capacity
=
Peak log rate × expected outage duration
```

and add operational headroom.

---

# 35. Logstash Disk

If persistent queues are enabled, storage becomes important.

For example:

```yaml
volumeMounts:
  - name: logstash-data
    mountPath: /usr/share/logstash/data
```

and:

```yaml
volumes:
  - name: logstash-data
    persistentVolumeClaim:
      claimName: logstash-pq
```

The exact storage architecture depends on:

* number of replicas
* queue strategy
* failure model
* storage performance
* whether queues need to survive Pod replacement

---

# 36. Stateful Persistent Queue Consideration

If a Logstash instance owns a local persistent queue:

```text
Logstash Pod
     │
     └── PVC
```

and the Pod moves:

```text
Node A
  ↓
Node B
```

the storage must still be attachable and the workload must be designed appropriately.

This is one reason persistent queue deployments need more careful architecture than a purely stateless Deployment.

Don't add PVCs without understanding the failure behavior.

---

# 37. Logstash → Elasticsearch

The output layer typically uses Elasticsearch output functionality.

Conceptually:

```conf
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

Production configuration should also consider:

```text
TLS
authentication
CA certificates
index/data stream strategy
bulk behavior
timeouts
retry behavior
```

---

# 38. Bulk Processing

Sending:

```text
1 event
1 request
1 event
1 request
1 event
1 request
```

is inefficient.

Logstash and Elasticsearch integrations support batching/bulk behavior.

Conceptually:

```text
Logstash
 │
 ├── event
 ├── event
 ├── event
 ├── event
 └── event
       │
       ▼
   Bulk request
       │
       ▼
 Elasticsearch
```

This improves throughput but requires tuning.

---

# 39. Elasticsearch Bottleneck

Suppose:

```text
Fluent Bit
   ↓
10,000 events/sec
   ↓
Logstash
   ↓
Elasticsearch
```

If Elasticsearch supports only:

```text
5,000 events/sec
```

then increasing Logstash replicas does not solve the underlying problem.

You need to investigate:

* Elasticsearch CPU
* JVM heap
* disk I/O
* shard count
* indexing rate
* refresh interval
* mapping complexity
* storage performance
* ingestion architecture

---

# 40. TLS

Production logging traffic should be protected.

```text
Fluent Bit
    │
    │ TLS
    ▼
Logstash
    │
    │ TLS
    ▼
Elasticsearch
```

Avoid:

```text
HTTP plaintext
```

for sensitive production logging networks unless there is a deliberately secured alternative architecture.

---

# 41. Authentication

Depending on the chosen stack/version:

```text
Fluent Bit
   │
   ├── authentication
   ▼
Logstash
   │
   ├── authentication
   ▼
Elasticsearch
```

Credentials/certificates should be supplied through:

* Kubernetes Secrets
* External Secrets Operator
* AWS Secrets Manager
* appropriate identity mechanisms

Do not place credentials directly into Git.

---

# 42. Example Kubernetes Secret Pattern

GitOps should contain a reference rather than a plaintext password.

Conceptually:

```text
Git
 │
 └── ExternalSecret definition
          │
          ▼
AWS Secrets Manager
          │
          ▼
Kubernetes Secret
          │
          ▼
Logstash
```

This follows the same secrets architecture used by our application workloads.

---

# 43. NetworkPolicy

Logging should not be reachable by arbitrary workloads.

Example conceptual policy:

```text
production workloads
       │
       ▼
Logstash : approved port

Other namespaces
       X
       │
       ▼
Logstash
```

NetworkPolicy should restrict:

* who can send logs
* who can access Elasticsearch
* who can access Kibana

---

# 44. Logstash Service

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: logging
spec:
  selector:
    app: logstash
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Fluent Bit uses:

```text
logstash.logging.svc.cluster.local
```

Kubernetes DNS resolves this Service to the Logstash Pods.

---

# 45. Deployment Example

Simplified example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: logging
spec:
  replicas: 3

  selector:
    matchLabels:
      app: logstash

  template:
    metadata:
      labels:
        app: logstash

    spec:
      containers:
        - name: logstash
          image: docker.elastic.co/logstash/logstash:<approved-version>

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

This is intentionally simplified.

Production deployment should additionally handle:

* configuration
* secrets
* health checks
* securityContext
* topology
* PDB
* storage if required
* TLS
* JVM settings
* monitoring

---

# 46. ConfigMap for Pipeline Configuration

Pipeline configuration can be managed as code.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipeline
  namespace: logging
data:
  application.conf: |
    input {
      http {
        port => 8080
      }
    }

    filter {
      json {
        source => "message"
      }
    }

    output {
      elasticsearch {
        hosts => ["https://elasticsearch:9200"]
      }
    }
```

For production:

```text
Git
 ↓
GitOps
 ↓
ConfigMap
 ↓
Logstash
```

---

# 47. Configuration Change Flow

Do not manually edit:

```bash
kubectl edit configmap logstash
```

in production as your normal process.

Instead:

```text
Developer/Platform Engineer
          │
          ▼
Git commit
          │
          ▼
Pull Request
          │
          ▼
Review
          │
          ▼
Merge
          │
          ▼
ArgoCD
          │
          ▼
EKS
          │
          ▼
Logstash configuration
```

This gives us:

* version control
* review
* audit trail
* rollback

---

# 48. Helm

A production Logstash deployment can be packaged through Helm.

Example:

```text
helm/
└── logstash/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    ├── values-qa.yaml
    ├── values-prod.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── configmap.yaml
        ├── serviceaccount.yaml
        └── pdb.yaml
```

ArgoCD manages the desired state.

---

# 49. GitOps Structure

Our GitOps repository can contain:

```text
gitops/
├── applications/
│   └── logging/
│       └── logstash.yaml
│
└── environments/
    ├── dev/
    │   └── logging/
    │       └── logstash/
    │
    ├── qa/
    │   └── logging/
    │
    └── prod/
        └── logging/
            └── logstash/
```

The exact structure can vary, but the principle remains:

```text
Git = desired state
ArgoCD = reconciler
EKS = actual state
```

---

# 50. Health Checks

Logstash should have appropriate health/readiness checks based on the deployed version and enabled monitoring/API endpoints.

Conceptually:

```text
Kubernetes
    │
    ▼
Is Logstash ready?
    │
 ┌──┴──┐
YES    NO
 │      │
Service  remove from traffic
```

Readiness is especially important during:

* startup
* configuration reload
* rolling deployment
* JVM pressure
* dependency failures

---

# 51. Rolling Update

Suppose:

```text
Logstash v1
 replicas = 3
```

We deploy:

```text
Logstash v2
```

Kubernetes should perform a controlled rollout.

```text
v1 ── v1 ── v1

       ↓

v2 ── v1 ── v1

       ↓

v2 ── v2 ── v1

       ↓

v2 ── v2 ── v2
```

But logging pipelines need special attention during upgrades because temporary connection failures can create backpressure.

---

# 52. Logstash Monitoring

Logstash itself must be monitored.

Important categories:

```text
JVM
CPU
Memory
Pipeline events
Queue
Output failures
Throughput
Latency
```

Track:

```text
events in
events out
events filtered
queue size
pipeline worker utilization
JVM heap
GC
CPU
memory
```

---

# 53. Monitoring Flow

Our monitoring architecture becomes:

```text
Logstash
   │
   ▼
Metrics
   │
   ▼
Prometheus
   │
   ▼
Grafana
```

Logs from Logstash itself can also flow into:

```text
Fluent Bit
   ↓
Logstash
   ↓
Elasticsearch
```

Be careful to avoid accidental logging loops.

---

# 54. The Logging Loop Problem

Bad design:

```text
Logstash
   ↓
writes logs
   ↓
Fluent Bit collects them
   ↓
Logstash
   ↓
writes logs
   ↓
Fluent Bit
```

This can become:

```text
infinite logging loop
```

Therefore exclude logging-platform logs from inappropriate collection paths or configure collection carefully.

---

# 55. Common Failure #1 — Fluent Bit Cannot Connect

Symptoms:

```text
Fluent Bit
   ↓
connection refused
```

Check:

```bash
kubectl get svc -n logging
```

Then:

```bash
kubectl get pods -n logging -o wide
```

Check endpoints:

```bash
kubectl get endpoints -n logging logstash
```

If no endpoints:

```text
Service selector
      ↓
doesn't match Pods
```

---

# 56. Common Failure #2 — Logstash Pod CrashLoopBackOff

Check:

```bash
kubectl get pods -n logging
```

Then:

```bash
kubectl logs -n logging deployment/logstash
```

Look for:

```text
pipeline configuration error
plugin configuration error
JVM failure
memory issue
permission issue
```

Also inspect:

```bash
kubectl describe pod -n logging <pod>
```

---

# 57. Common Failure #3 — Pipeline Syntax Error

Example:

```text
Expected one of ...
```

Usually means:

```text
Logstash configuration syntax
        ↓
invalid
```

Validate configuration before production rollout.

Never treat Logstash configuration as ordinary text.

It is executable pipeline configuration.

---

# 58. Common Failure #4 — Elasticsearch Is Down

Flow:

```text
Elasticsearch DOWN
      ↓
Logstash output retries
      ↓
Queue increases
      ↓
Fluent Bit buffers increase
```

Check:

```bash
kubectl get pods -n logging
```

Then Elasticsearch health.

The correct response is not immediately:

```bash
kubectl restart deployment/logstash
```

First determine:

```text
Why can't Logstash deliver events?
```

---

# 59. Common Failure #5 — Logstash OOMKilled

Check:

```bash
kubectl get pods -n logging
```

If:

```text
OOMKilled
```

investigate:

```text
event volume
large messages
Grok complexity
pipeline workers
JVM heap
persistent queue
container memory limit
```

Increasing memory may help, but it should not replace root-cause analysis.

---

# 60. Common Failure #6 — Logs Are Delayed

Suppose:

```text
Application
  ↓
Fluent Bit
  ↓
Logstash
  ↓
Elasticsearch
```

Kibana shows the event five minutes late.

Don't automatically blame Logstash.

Check each layer:

```text
1. Application generated log?
2. Container log exists?
3. Fluent Bit collected it?
4. Fluent Bit forwarded it?
5. Logstash received it?
6. Logstash processed it?
7. Elasticsearch indexed it?
8. Kibana query/index pattern correct?
```

---

# 61. Common Failure #7 — Logs Disappear During Elasticsearch Outage

Investigate:

```text
Fluent Bit buffering
Logstash queue
Persistent Queue capacity
Elasticsearch outage duration
disk capacity
retry behavior
```

Calculate:

```text
peak logs/sec
×
average event bytes
×
outage duration
```

Then compare with:

```text
available buffer capacity
```

---

# 62. Common Failure #8 — Elasticsearch Rejects Documents

Possible causes:

```text
mapping conflict
invalid field type
oversized document
index blocked
disk watermark
authentication failure
TLS problem
shard failure
```

Logstash may show the output error.

Trace:

```text
Logstash
   ↓
Elasticsearch response
   ↓
specific rejection
```

Fix the destination problem rather than endlessly restarting Logstash.

---

# 63. Common Failure #9 — Mapping Conflicts

Suppose:

```text
order_id = "12345"
```

in one event:

```text
string
```

and:

```text
order_id = 12345
```

in another:

```text
number
```

Elasticsearch mapping can become problematic.

This is why:

> **Field schemas must be standardized across microservices.**

---

# 64. Schema Standardization

Define common fields:

```text
@timestamp
log.level
service.name
service.version
deployment.environment
trace.id
span.id
kubernetes.namespace
kubernetes.pod.name
message
```

Then application-specific fields:

```text
order.id
customer.id
payment.id
```

This makes:

```text
Kibana
Grafana
OpenTelemetry
Elasticsearch
```

much easier to integrate.

---

# 65. Log Volume Planning

Suppose:

```text
21 services
```

Average:

```text
100 events/sec/service
```

Then:

```text
21 × 100
= 2,100 events/sec
```

If average event size is:

```text
1 KB
```

Raw data:

```text
2,100 KB/sec
≈ 2.1 MB/sec
```

Per day:

```text
2.1 × 86,400
≈ 181 GB/day
```

This is **before** considering:

* Elasticsearch indexing overhead
* replicas
* metadata
* compression
* bursts
* multiple environments

This is why logging requires capacity planning.

---

# 66. High-Traffic Scenario

Suppose traffic increases 10×.

```text
2,100 events/sec
      ↓
21,000 events/sec
```

If Logstash remains:

```text
2 replicas
```

it may become a bottleneck.

Scaling may require:

```text
2 Logstash Pods
       ↓
4 Logstash Pods
       ↓
6 Logstash Pods
```

But only after measuring:

```text
CPU
pipeline throughput
queue depth
Elasticsearch capacity
```

---

# 67. Logstash Scaling Is Not Application HPA

Application:

```text
CPU > 70%
    ↓
HPA
    ↓
more Pods
```

Logstash scaling should consider:

```text
event throughput
pipeline utilization
queue depth
CPU
memory
output latency
```

CPU alone may not be enough.

A custom metric such as event throughput or queue depth can be more meaningful depending on the architecture.

---

# 68. Dedicated Nodes

Logging can be resource-intensive.

We don't necessarily want:

```text
Logstash
   ↓
competing with
   ↓
payment-service
```

on the same nodes.

We can create dedicated capacity:

```text
Application nodes
      │
      └── business workloads

Logging nodes
      │
      ├── Logstash
      └── Elasticsearch
```

Use:

* node labels
* taints
* tolerations
* affinity
* topology spread

where justified.

---

# 69. Failure Domains

A production logging architecture should consider:

```text
Node failure
Pod failure
AZ failure
EKS failure
Elasticsearch failure
network failure
storage failure
```

Example:

```text
AZ-A
 └── Logstash

AZ-B
 └── Logstash

AZ-C
 └── Logstash
```

Then:

```text
AZ-A failure
     ↓
AZ-B + AZ-C
     ↓
logging continues
```

---

# 70. Fluent Bit vs Logstash

| Capability              | Fluent Bit        | Logstash            |
| ----------------------- | ----------------- | ------------------- |
| Lightweight             | Excellent         | Heavier             |
| Node-level collection   | Excellent         | Poor fit            |
| DaemonSet               | Excellent         | Usually unnecessary |
| Parsing                 | Good              | Excellent           |
| Complex transformations | Moderate          | Strong              |
| JVM dependency          | No                | Yes                 |
| Central processing      | Possible          | Excellent           |
| Kubernetes metadata     | Excellent         | Depends on input    |
| Persistent queue        | Buffering options | Strong PQ support   |
| Resource footprint      | Low               | Higher              |

The common architecture is:

```text
Fluent Bit = collection
Logstash = processing
Elasticsearch = storage/search
Kibana = visualization
```

---

# 71. Why Not Logstash as DaemonSet?

Technically possible, but usually inefficient.

If every node runs:

```text
Logstash
```

then:

```text
100 nodes
=
100 Logstash instances
```

This can dramatically increase resource consumption.

Fluent Bit is designed for lightweight node-level collection.

Therefore:

```text
DaemonSet → Fluent Bit
Deployment → Logstash
```

is a common architecture.

---

# 72. Production Installation Strategy

Do not install Logstash manually on an EKS node.

Avoid:

```bash
ssh node
yum install logstash
```

Instead:

```text
Terraform
    │
    └── EKS/platform infrastructure

Helm
    │
    └── Logstash package/config

GitOps
    │
    └── desired state

ArgoCD
    │
    ▼
EKS
```

This gives repeatability.

---

# 73. Tool Placement

| Tool           | Production location                           |
| -------------- | --------------------------------------------- |
| Fluent Bit     | EKS DaemonSet                                 |
| Logstash       | EKS logging namespace                         |
| Elasticsearch  | EKS logging platform / managed Elastic option |
| Kibana         | EKS logging platform / managed Elastic option |
| Prometheus     | EKS monitoring                                |
| Grafana        | EKS monitoring                                |
| ArgoCD         | EKS platform                                  |
| Terraform      | CI/control environment                        |
| Trivy          | CI runner                                     |
| SonarQube      | Central platform/SaaS                         |
| GitHub Actions | CI runner                                     |
| ECR            | AWS managed                                   |
| RDS            | AWS managed                                   |
| ElastiCache    | AWS managed                                   |

This distinction is important.

---

# 74. Security Checklist

Logstash production should have:

```text
[ ] Non-root container where supported
[ ] Read-only filesystem where practical
[ ] TLS
[ ] Authentication
[ ] Kubernetes Secrets / External Secrets
[ ] NetworkPolicy
[ ] Least-privilege RBAC
[ ] Resource requests
[ ] Resource limits
[ ] PDB
[ ] Pod distribution
[ ] No sensitive data logging
[ ] Audit trail
```

---

# 75. Operational Checklist

```text
[ ] Fluent Bit can reach Logstash
[ ] Logstash Pods are distributed
[ ] Service has healthy endpoints
[ ] Pipeline configuration validated
[ ] Elasticsearch connectivity tested
[ ] TLS verified
[ ] Authentication verified
[ ] Queue monitored
[ ] JVM monitored
[ ] CPU/memory monitored
[ ] Event throughput monitored
[ ] Output failures monitored
[ ] Disk capacity monitored
[ ] Backup/retention strategy defined
[ ] Log retention defined
[ ] Index/data-stream strategy defined
[ ] Runbooks available
```

---

# 76. Real Production Troubleshooting Method

When logs disappear:

### Layer 1 — Application

```text
Did the application generate the log?
```

### Layer 2 — Kubernetes

```text
Does the container log exist?
```

### Layer 3 — Fluent Bit

```text
Did Fluent Bit collect and forward it?
```

### Layer 4 — Logstash

```text
Did Logstash receive and process it?
```

### Layer 5 — Elasticsearch

```text
Did Elasticsearch accept and index it?
```

### Layer 6 — Kibana

```text
Are we querying the correct index/data stream/time range?
```

This is much better than randomly restarting Pods.

---

# 77. Complete Production Logging Architecture

Our architecture now becomes:

```text
                         USERS
                           │
                           ▼
                    Route 53 / CloudFront
                           │
                           ▼
                         ALB
                           │
                           ▼
                          EKS
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    order-service    payment-service   catalog-service
          │                │                │
          └────────────────┼────────────────┘
                           │
                     stdout/stderr
                           │
                           ▼
                 Kubernetes container logs
                           │
                           ▼
                    Fluent Bit DaemonSet
                           │
                           ▼
                    Logstash Service
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
             Pipeline             Persistent
             Processing             Queue
                │                     │
                └──────────┬──────────┘
                           ▼
                    Elasticsearch
                           │
                           ▼
                         Kibana
```

---

# 78. How This Fits the Entire Observability Stack

We now have:

```text
                    EKS APPLICATIONS
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
       Metrics           Logs            Traces
          │                │                │
          ▼                ▼                ▼
    Prometheus         Fluent Bit      OpenTelemetry
          │                │                │
          │                ▼                ▼
          │            Logstash           Jaeger
          │                │                │
          ▼                ▼                ▼
       Grafana       Elasticsearch       Grafana
          │                │
          ▼                ▼
    Alertmanager          Kibana
```

This gives three major observability signals:

```text
Metrics → What is happening?
Logs    → What happened?
Traces  → Where did it happen?
```

---

# 79. Example: Payment Failure

Imagine a customer reports:

> "My payment failed."

We can investigate using all three signals.

### Metrics

Prometheus:

```text
payment_request_error_rate ↑
```

### Logs

Kibana:

```text
service.name:payment-service
level:ERROR
```

Find:

```text
Payment gateway timeout
```

### Trace

Jaeger:

```text
order-service
      ↓
payment-service
      ↓
external-payment-provider
```

Trace shows:

```text
payment-service → gateway
        4.8 seconds
```

Now we have:

```text
Metrics
   +
Logs
   +
Traces
```

instead of guessing.

---

# 80. Final Mental Model

Remember the responsibility of each component:

```text
Fluent Bit
    =
COLLECT

Logstash
    =
PROCESS

Elasticsearch
    =
STORE + SEARCH

Kibana
    =
VISUALIZE
```

And the production flow:

```text
Application
    ↓
stdout/stderr
    ↓
Fluent Bit
    ↓
Logstash
    ↓
Elasticsearch
    ↓
Kibana
```

The most important operational lesson is:

> **Logging is a pipeline, not a single tool.**

When something breaks, trace the event through every layer.

---

## 81. Production Design Summary

For our Enterprise E-Commerce Platform:

```text
21 Microservices
       │
       ▼
stdout/stderr
       │
       ▼
Fluent Bit DaemonSet
       │
       ▼
Logstash Deployment
       │
       ├── JSON parsing
       ├── normalization
       ├── enrichment
       ├── filtering
       ├── routing
       └── buffering
       │
       ▼
Elasticsearch
       │
       ▼
Kibana
```

With:

```text
GitOps
  ↓
Helm
  ↓
ArgoCD
  ↓
EKS
```

and:

```text
Prometheus
   ↓
Grafana
   ↓
Alertmanager
```

for Logstash/platform monitoring.

---

# 82. What Comes Next

The next component in the logging stack is:

```text
36 Logstash
      ↓
37 Elasticsearch
      ↓
38 Kibana
      ↓
39 OpenTelemetry
      ↓
40 Jaeger
```

**Next file: `37-Elasticsearch.md`**

That chapter should go much deeper into the part that actually stores our logs: **Elasticsearch architecture, nodes, shards, replicas, mappings, indexing, ILM/data streams, EBS storage, ECK on EKS, TLS, security, scaling, snapshots, retention, and production failure scenarios.**
